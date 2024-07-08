# Demo05-使用Karpenter进行自动伸缩
--
#### Contributor: Yi Zheng
#### 更新时间: 2024-07-08
#### 基于EKS版本: EKS 1.30
--

## 0. 先决条件  
使用eksctl创建集群：
```
export AWS_REGION=cn-northwest-1
export AWS_DEFAULT_REGION=cn-northwest-1
export CLUSTER_NAME=eksworkshop
eksctl create cluster --name=${CLUSTER_NAME} --node-type t3.medium --managed --region=${AWS_REGION} --version 1.30 -N 1
```

## 1. 创建环境

### 1.1 前置条件

```
export KARPENTER_NAMESPACE="kube-system"
export KARPENTER_VERSION="0.37.0"
export K8S_VERSION="1.30"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export TEMPOUT=$(mktemp)

export AWS_PARTITION="aws-cn" # if you are not using standard partitions, you may need to configure to aws-cn / aws-us-gov
export ARM_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2-arm64/recommended/image_id --query Parameter.Value --output text)"
export AMD_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2/recommended/image_id --query Parameter.Value --output text)"
export GPU_AMI_ID="$(aws ssm get-parameter --name /aws/service/eks/optimized-ami/${K8S_VERSION}/amazon-linux-2-gpu/recommended/image_id --query Parameter.Value --output text)"

```
### 1.2 标记子网和安全组

```
SUBNET_IDS=$(aws cloudformation describe-stacks \
    --stack-name eksctl-${CLUSTER_NAME}-cluster \
    --query 'Stacks[].Outputs[?OutputKey==`SubnetsPrivate`].OutputValue' \
    --output text)
aws ec2 create-tags \
    --resources $(echo $SUBNET_IDS | tr ',' '\n') \
    --tags Key="karpenter.sh/discovery",Value="${CLUSTER_NAME}"

Security_group_ids=$(aws cloudformation describe-stacks \
    --stack-name eksctl-${CLUSTER_NAME}-cluster \
    --query 'Stacks[].Outputs[?OutputKey==`ClusterSecurityGroupId`].OutputValue' \
    --output text)
aws ec2 create-tags \
    --resources $(echo $Security_group_ids | tr ',' '\n') \
    --tags Key="karpenter.sh/discovery",Value="${CLUSTER_NAME}"
```
### 1.3 创建karpenter节点IAM Role和实例profile

```
curl -fsSL https://raw.githubusercontent.com/aws/karpenter-provider-aws/v"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml  > "${TEMPOUT}" \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```

给实例授权使其可以使用之前创建的profile来连接eks集群。

```
eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --arn arn:aws-cn:iam::${ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME} \
  --group system:bootstrappers \
  --group system:nodes \
  --cluster ${CLUSTER_NAME}
```

检查 AWS auth map

```
kubectl describe configmap -n kube-system aws-auth
```

输出显示：

```
Name:         aws-auth
Namespace:    kube-system
Labels:       <none>
Annotations:  <none>

Data
====
mapRoles:
----
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws-cn:iam::487071860556:role/eksctl-eks0714-nodegroup-nodegrou-NodeInstanceRole-1CE70C5WOR3Y
  username: system:node:{{EC2PrivateDNSName}}
- groups:
  - system:bootstrappers
  - system:nodes
  rolearn: arn:aws-cn:iam::487071860556:role/KarpenterNodeRole-eks0714
  username: system:node:{{EC2PrivateDNSName}}

mapUsers:
----
[]


BinaryData
====

Events:  <none>
```

### 1.4 创建 KarpenterController IAM Role

创建IAM OIDC Identity Provider for the cluster

```
eksctl utils associate-iam-oidc-provider --cluster ${CLUSTER_NAME} --approve --region ${AWS_REGION}
```

创建AWS IAM Role, Kubernetes service account, 并使用 IAM Roles for Service Accounts (IRSA)

```
eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME --name karpenter --namespace karpenter \
  --attach-policy-arn arn:${AWS_PARTITION}:iam::${AWS_ACCOUNT_ID}:policy/KarpenterControllerPolicy-${CLUSTER_NAME} \
  --approve \
  --region ${AWS_REGION}
```

检查service account

```
kubectl get serviceaccounts --namespace karpenter
```

## 2. 安装 KARPENTER

### 2.1 前提条件
已经安装 Helm，Windows和Macos下安装helm

```
https://helm.sh/zh/docs/intro/install/

~]$ helm version
version.BuildInfo{Version:"v3.12.3", GitCommit:"3a31588ad33fe3b89af5a2a54ee1d25bfe6eaa5e", GitTreeState:"clean", GoVersion:"go1.20.7"}
```

### 2.2 使用Helm Chart 安装Karpenter

```
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
echo $CLUSTER_ENDPOINT

aws iam create-service-linked-role --aws-service-name spot.amazonaws.com || true

helm registry logout public.ecr.aws

helm upgrade --install karpenter oci://public.ecr.aws/karpenter/karpenter --version ${KARPENTER_VERSION} --namespace karpenter --create-namespace \
  --set serviceAccount.create=false \
  --set serviceAccount.name=karpenter \
  --set settings.clusterName=${CLUSTER_NAME} \
  --set settings.interruptionQueue=${CLUSTER_NAME} \
  --wait
```

输出显示：

```
Release "karpenter" does not exist. Installing it now.
NAME: karpenter
LAST DEPLOYED: Mon Jul  8 11:23:38 2024
NAMESPACE: karpenter
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

检查 karpenter controller和webhook

```
kubectl get pods --namespace karpenter
```

```
kubectl get deployment -n karpenter
```

## 3. 设置 PROVISIONER

### 3.1 设置默认的 CRD Provisioner

```
cat <<EOF | envsubst | kubectl apply -f -
apiVersion: karpenter.sh/v1beta1
kind: NodePool
metadata:
  name: default
spec:
  template:
    spec:
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["amd64"]
        - key: kubernetes.io/os
          operator: In
          values: ["linux"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["spot"]
        - key: karpenter.k8s.aws/instance-category
          operator: In
          values: ["c", "m", "r"]
        - key: karpenter.k8s.aws/instance-generation
          operator: Gt
          values: ["2"]
      nodeClassRef:
        apiVersion: karpenter.k8s.aws/v1beta1
        kind: EC2NodeClass
        name: default
  limits:
    cpu: 1000
  disruption:
    consolidationPolicy: WhenUnderutilized
    expireAfter: 720h # 30 * 24h = 720h
---
apiVersion: karpenter.k8s.aws/v1beta1
kind: EC2NodeClass
metadata:
  name: default
spec:
  amiFamily: AL2 # Amazon Linux 2
  role: "KarpenterNodeRole-${CLUSTER_NAME}" # replace with your cluster name
  subnetSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  securityGroupSelectorTerms:
    - tags:
        karpenter.sh/discovery: "${CLUSTER_NAME}" # replace with your cluster name
  amiSelectorTerms:
    - id: "${ARM_AMI_ID}"
    - id: "${AMD_AMI_ID}"
#   - id: "${GPU_AMI_ID}" # <- GPU Optimized AMD AMI 
#   - name: "amazon-eks-node-${K8S_VERSION}-*" # <- automatically upgrade when a new AL2 EKS Optimized AMI is released. This is unsafe for production workloads. Validate AMIs in lower environments before deploying them to production.
EOF
```

### 3.2 手动修改Service Account IAM Role

```
kubectl describe sa  karpenter -n karpenter | egrep Annotations
Annotations:         eks.amazonaws.com/role-arn: arn:aws-cn:iam::332433839685:role/eksctl-eksworkshop-addon-iamserviceaccount-ka-Role1-cxMapWHVHZGj

# 删除arn:aws-cn:iam::332433839685:role/eksctl-eksworkshop-addon-iamserviceaccount-ka-Role1-cxMapWHVHZGj中的如下部分
# 原始策略
{
    "Sid": "AllowPassingInstanceRole",
    "Effect": "Allow",
    "Resource": "arn:aws-cn:iam::332433839685:role/KarpenterNodeRole-eksworkshop",
    "Action": "iam:PassRole",
    "Condition": {
        "StringEquals": {
            "iam:PassedToService": "ec2.amazonaws.com"
        }
    }
},
# 修改策略
{
    "Sid": "AllowPassingInstanceRole",
    "Effect": "Allow",
    "Resource": "arn:aws-cn:iam::332433839685:role/KarpenterNodeRole-eksworkshop",
    "Action": "iam:PassRole"
},
```

## 4. 自动节点配置

### 4.1 自动节点配置

```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: inflate
spec:
  replicas: 0
  selector:
    matchLabels:
      app: inflate
  template:
    metadata:
      labels:
        app: inflate
    spec:
      terminationGracePeriodSeconds: 0
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.7
          resources:
            requests:
              cpu: 1
EOF

kubectl scale deployment inflate --replicas 5
kubectl logs -f -n "${KARPENTER_NAMESPACE}" -l app.kubernetes.io/name=karpenter -c controller
```

### 4.2 将pod的数量缩减至0观察

```
kubectl scale deployment inflate --replicas 0
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
2023-10-29T13:44:51.387Z        INFO    controller.deprovisioning       deprovisioning via consolidation delete, terminating 1 machines ip-192-168-98-76.cn-northwest-1.compute.internal/c4.xlarge/spot   {"commit": "61b3e1e-dirty"}
2023-10-29T13:44:51.463Z        INFO    controller.termination  cordoned node   {"commit": "61b3e1e-dirty", "node": "ip-192-168-98-76.cn-northwest-1.compute.internal"}
2023-10-29T13:44:51.826Z        INFO    controller.termination  deleted node    {"commit": "61b3e1e-dirty", "node": "ip-192-168-98-76.cn-northwest-1.compute.internal"}
2023-10-29T13:44:52.110Z        INFO    controller.machine.termination  deleted machine {"commit": "61b3e1e-dirty", "machine": "default-bmr6l", "provisioner": "default", "node": "ip-192-168-98-76.cn-northwest-1.compute.internal", "provider-id": "aws:///cn-northwest-1c/i-0b7b012bb90671250"}
```

## 5. 清除环境

```
# 删除karpanter
helm uninstall karpenter --namespace karpenter

# 删除service account
eksctl delete iamserviceaccount --cluster ${CLUSTER_NAME} --name karpenter --namespace karpenter

# 删除 cloudformation stack
aws cloudformation delete-stack --stack-name Karpenter-${CLUSTER_NAME}

# 检查launch-templates
aws ec2 describe-launch-templates \
    | jq -r ".LaunchTemplates[].LaunchTemplateName" \
    | grep -i Karpenter-${CLUSTER_NAME} \
    | xargs -I{} aws ec2 delete-launch-template --launch-template-name {}
```
