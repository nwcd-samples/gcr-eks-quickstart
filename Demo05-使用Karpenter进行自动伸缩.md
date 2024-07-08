# Demo05-使用Karpenter进行自动伸缩
--
#### Contributor: Yi Zheng
#### 更新时间: 2023-10-29
#### 基于EKS版本: EKS 1.27
--

## 0. 先决条件  
使用eksctl创建集群：
```
~]$ eksctl create cluster --name=${CLUSTER_NAME} --node-type t3.medium --managed --region=${AWS_REGION} --version 1.30 -N 1
```

## 1. 创建环境

### 1.1 前置条件

```
export KARPENTER_VERSION=v0.31.1
export CLUSTER_NAME=$(eksctl get clusters -o json | jq -r '.[0].Name')
export AWS_DEFAULT_REGION="cn-northwest-1"
export AWS_REGION="cn-northwest-1"
export AWS_ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export ACCOUNT_ID="$(aws sts get-caller-identity --query Account --output text)"
export TEMPOUT=$(mktemp)
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
curl -fsSL https://raw.githubusercontent.com/aws/karpenter/"${KARPENTER_VERSION}"/website/content/en/preview/getting-started/getting-started-with-karpenter/cloudformation.yaml  > $TEMPOUT \
&& aws cloudformation deploy \
  --stack-name "Karpenter-${CLUSTER_NAME}" \
  --template-file "${TEMPOUT}" \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides "ClusterName=${CLUSTER_NAME}"
```

输出显示：

```
Waiting for changeset to be created..
Waiting for stack create/update to complete
Successfully created/updated stack - Karpenter-eksworkshop
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

输出显示：

```
2023-10-29 08:46:23 [ℹ]  checking arn arn:aws-cn:iam::332433839685:role/KarpenterNodeRole-eksworkshop against entries in the auth ConfigMap
2023-10-29 08:46:23 [ℹ]  adding identity "arn:aws-cn:iam::332433839685:role/KarpenterNodeRole-eksworkshop" to auth ConfigMap
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

输出显示：

```
2022-07-11 17:53:42 [ℹ]  eksctl version 0.90.0
2022-07-11 17:53:42 [ℹ]  using region cn-northwest-1
2022-07-11 17:53:42 [ℹ]  IAM Open ID Connect provider is already associated with cluster "eks0707" in "cn-northwest-1"
```

创建AWS IAM Role, Kubernetes service account, 并使用 IAM Roles for Service Accounts (IRSA)

```
eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME --name karpenter --namespace karpenter \
  --attach-policy-arn arn:aws-cn:iam::aws:policy/AdministratorAccess \
  --approve \
  --region ${AWS_REGION}
```
输出显示：

```
2022-07-11 17:54:37 [ℹ]  eksctl version 0.90.0
2022-07-11 17:54:37 [ℹ]  using region cn-northwest-1
2022-07-11 17:54:39 [ℹ]  1 iamserviceaccount (karpenter/karpenter) was included (based on the include/exclude rules)
2022-07-11 17:54:39 [!]  serviceaccounts that exist in Kubernetes will be excluded, use --override-existing-serviceaccounts to override
2022-07-11 17:54:39 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "karpenter/karpenter",
        create serviceaccount "karpenter/karpenter",
    } }2022-07-11 17:54:39 [ℹ]  building iamserviceaccount stack "eksctl-eks0707-addon-iamserviceaccount-karpenter-karpenter"
2022-07-11 17:54:39 [ℹ]  deploying stack "eksctl-eks0707-addon-iamserviceaccount-karpenter-karpenter"
2022-07-11 17:54:39 [ℹ]  waiting for CloudFormation stack "eksctl-eks0707-addon-iamserviceaccount-karpenter-karpenter"
2022-07-11 17:54:55 [ℹ]  waiting for CloudFormation stack "eksctl-eks0707-addon-iamserviceaccount-karpenter-karpenter"
2022-07-11 17:54:56 [ℹ]  created namespace "karpenter"
2022-07-11 17:54:56 [ℹ]  created serviceaccount "karpenter/karpenter"
```

检查service account

```
kubectl get serviceaccounts --namespace karpenter
```
输出显示：

```
NAME        SECRETS   AGE
default     1         44s
karpenter   1         44s
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
  --set settings.aws.clusterName=${CLUSTER_NAME} \
  --set settings.aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --set settings.aws.interruptionQueueName=${CLUSTER_NAME} \
  --wait
```

输出显示：

```
Release "karpenter" does not exist. Installing it now.
Pulled: public.ecr.aws/karpenter/karpenter:v0.31.1
Digest: sha256:0f0f4f12d688c62d6ba02ce2b73580c80f24767769ae0db21f17c027ee4dd711
NAME: karpenter
LAST DEPLOYED: Sun Oct 29 07:55:55 2023
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
cat <<EOF | kubectl apply -f -
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  limits:
    resources:
      cpu: 1000
  providerRef:
    name: default
  consolidation: 
    enabled: true
---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
  securityGroupSelector:
    karpenter.sh/discovery: ${CLUSTER_NAME}
EOF

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
kubectl logs -f -n karpenter -l app.kubernetes.io/name=karpenter -c controller
2023-10-29T13:36:07.995Z        INFO    controller      Starting informers...   {"commit": "61b3e1e-dirty"}
2023-10-29T13:37:59.196Z        INFO    controller.machine.lifecycle    launched machine        {"commit": "61b3e1e-dirty", "machine": "default-bmr6l", "provisioner": "default", "provider-id": "aws:///cn-northwest-1c/i-0b7b012bb90671250", "instance-type": "c4.xlarge", "zone": "cn-northwest-1c", "capacity-type": "spot", "allocatable": {"cpu":"3920m","ephemeral-storage":"17Gi","memory":"6111Mi","pods":"58"}}
2023-10-29T13:38:23.236Z        DEBUG   controller.machine.lifecycle    registered machine      {"commit": "61b3e1e-dirty", "machine": "default-bmr6l", "provisioner": "default", "provider-id": "aws:///cn-northwest-1c/i-0b7b012bb90671250", "node": "ip-192-168-98-76.cn-northwest-1.compute.internal"}
2023-10-29T13:38:34.435Z        DEBUG   controller.machine.lifecycle    initialized machine     {"commit": "61b3e1e-dirty", "machine": "default-bmr6l", "provisioner": "default", "provider-id": "aws:///cn-northwest-1c/i-0b7b012bb90671250", "node": "ip-192-168-98-76.cn-northwest-1.compute.internal"}
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
