# Demo05-使用Karpenter进行自动伸缩
--
#### Contributor: Yi Zheng
#### 更新时间: 2023-10-25
#### 基于EKS版本: EKS 1.27
--

## 0. 先决条件  
0.1 准备实验环境：参考Demo 01  
0.2 使用eksctl创建集群：参考Demo 02，不要执行 4. 镜像处置(针对中国区)  
0.3 设置环境变量
```
AWS_REGION=cn-northwest-1
AWS_DEFAULT_REGION=cn-northwest-1
CLUSTER_NAME=eksworkshop
```

## 1. 创建环境

### 1.1 前置条件

```
export CLUSTER_NAME=$(eksctl get clusters -o json | jq -r '.[0].Name')
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=cn-northwest-1
```
### 1.2 标记子网

```
SUBNET_IDS=$(aws cloudformation describe-stacks \
    --stack-name eksctl-${CLUSTER_NAME}-cluster \
    --query 'Stacks[].Outputs[?OutputKey==`SubnetsPrivate`].OutputValue' \
    --output text)
aws ec2 create-tags \
    --resources $(echo $SUBNET_IDS | tr ',' '\n') \
    --tags Key="kubernetes.io/cluster/${CLUSTER_NAME}",Value=
```

如果子网信息tag不成功，可以尝试手工将subnet-id进行输入

```
aws ec2 create-tags \
    --resources $(echo subnet-0321df0f0ae09292f,subnet-00ff18067fe034e2d,subnet-0debabfbcf7387bfd | tr ',' '\n') \
    --tags Key="kubernetes.io/cluster/${CLUSTER_NAME}",Value=
```

检查子网信息

```
VALIDATION_SUBNETS_IDS=$(aws ec2 describe-subnets --filters Name=tag:"kubernetes.io/cluster/${CLUSTER_NAME}",Values= --query "Subnets[].SubnetId" --output text | sed 's/\t/,/')
echo "$SUBNET_IDS == $VALIDATION_SUBNETS_IDS"
```

输出显示：

```
subnet-0321df0f0ae09292f,subnet-00ff18067fe034e2d,subnet-0debabfbcf7387bfd == subnet-0321df0f0ae09292f,subnet-0debabfbcf7387bfd	subnet-00ff18067fe034e2d
```
### 1.3 创建karpenter节点IAM Role和实例profile

```
export TEMPOUT=$(mktemp)
export KARPENTER_VERSION=v0.31.1

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
Successfully created/updated stack - Karpenter-eks0707
```

给实例授权使其可以使用之前创建的profile来连接eks集群。

```
eksctl create iamidentitymapping \
  --username system:node:{{EC2PrivateDNSName}} \
  --cluster  ${CLUSTER_NAME} \
  --arn arn:aws-cn:iam::${ACCOUNT_ID}:role/KarpenterNodeRole-${CLUSTER_NAME} \
  --group system:bootstrappers \
  --group system:nodes \
  --region ${AWS_REGION}
```

输出显示：

```
2022-07-11 17:45:35 [ℹ]  eksctl version 0.90.0
2022-07-11 17:45:35 [ℹ]  using region cn-northwest-1
2022-07-11 17:45:36 [ℹ]  adding identity "arn:aws:iam::487071860556:role/KarpenterNodeRole-eks0707" to auth ConfigMap
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
export ACCOUNT_ID=<> # <>--->替换成Account ID

eksctl create iamserviceaccount \
  --cluster $CLUSTER_NAME --name karpenter --namespace karpenter \
  --attach-policy-arn arn:aws-cn:iam::$ACCOUNT_ID:policy/KarpenterControllerPolicy-$CLUSTER_NAME \
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
```

### 2.2 使用Helm Chart 安装Karpenter

```
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm upgrade --install karpenter karpenter/karpenter --namespace karpenter \
  --create-namespace --set serviceAccount.create=false --version 0.4.3 \
  --set controller.clusterName=${CLUSTER_NAME} \
  --set controller.clusterEndpoint=$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output json) \
  --set defaultProvisioner.create=false \
  --wait # for the defaulting webhook to install before creating a Provisioner
```

输出显示：

```
Release "karpenter" has been upgraded. Happy Helming!
NAME: karpenter
LAST DEPLOYED: Wed Jul 13 14:55:34 2022
NAMESPACE: karpenter
STATUS: deployed
REVISION: 2
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
  labels:
    intent: apps
  provider:
    instanceProfile: KarpenterNodeInstanceProfile-${CLUSTER_NAME}
    tags:
      accountingEC2Tag: KarpenterDevEnvironmentEC2
  ttlSecondsAfterEmpty: 30
EOF
```

### 3.2 显示 Karpenter 日志

```
kubectl logs -f deployment/karpenter-controller -n karpenter
```

## 4. 自动节点配置

### 4.1 自动节点配置

```
cat <<EOF > inflate.yaml
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
      nodeSelector:
        intent: apps
      containers:
        - name: inflate
          image: public.ecr.aws/eks-distro/kubernetes/pause:3.2
          resources:
            requests:
              cpu: 1
              memory: 1.5Gi
EOF
kubectl apply -f inflate.yaml
```

### 4.2 扩展测试及输出检查

a. 增加inflate deployment数量

```
kubectl scale deployment inflate --replicas 1
```

```
kubectl get deployment inflate
```

b. 检查新的pod是否部署在指定label的node上，即由karpenter新创建的node上：

```
kubectl get node --selector=intent=apps --show-labels
```

c. 检查新node的实例类型及相关node创建的日志：

```
echo type: $(kubectl describe node --selector=intent=apps | grep "beta.kubernetes.io/instance-type" )
```

日志输出：

```
2022-07-14T03:43:07.931Z	INFO	controller.allocation.provisioner/default	Found 1 provisionable pods	{"commit": "dc47849"}
2022-07-14T03:43:08.051Z	INFO	controller.allocation.provisioner/default	Computed packing for 1 pod(s) with instance type option(s) [c5d.large c5.large c4.large t3.medium c5a.large t3a.medium t3a.large m5a.large m5d.large m5.large t3.large m4.large c5d.xlarge c5a.xlarge c5.xlarge c4.xlarge r4.large r5d.large r5.large r5a.large]	{"commit": "dc47849"}
2022-07-14T03:43:09.880Z	INFO	controller.allocation.provisioner/default	Launched instance: i-0cd61546e0c5ec3d7, hostname: ip-192-168-186-250.cn-northwest-1.compute.internal, type: t3a.medium, zone: cn-northwest-1b, capacityType: on-demand	{"commit": "dc47849"}
2022-07-14T03:43:09.905Z	INFO	controller.allocation.provisioner/default	Bound 1 pod(s) to node ip-192-168-186-250.cn-northwest-1.compute.internal	{"commit": "dc47849"}
```

d. 检查新node的属性和标签

```
kubectl describe node --selector=intent=apps

Labels:             beta.kubernetes.io/arch=amd64
                    beta.kubernetes.io/instance-type=t3a.medium
                    beta.kubernetes.io/os=linux
                    failure-domain.beta.kubernetes.io/region=cn-northwest-1
                    failure-domain.beta.kubernetes.io/zone=cn-northwest-1b
                    intent=apps
                    k8s.io/cloud-provider-aws=ad3888def509b2b87abf9037440a6f32
                    karpenter.sh/capacity-type=on-demand
                    karpenter.sh/provisioner-name=default
```

e. 观察将scale pod的数量增加到6的现象

```
kubectl scale deployment inflate --replicas 6
```

日志输出：

```
2022-07-14T03:57:44.930Z	INFO	controller.allocation.provisioner/default	Found 5 provisionable pods	{"commit": "dc47849"}
2022-07-14T03:57:44.938Z	INFO	controller.allocation.provisioner/default	Computed packing for 5 pod(s) with instance type option(s) [c4.2xlarge c5d.2xlarge c5.2xlarge c5a.2xlarge t3.2xlarge m5d.2xlarge m4.2xlarge m5a.2xlarge t3a.2xlarge m5.2xlarge c4.4xlarge c5d.4xlarge c5.4xlarge c5a.4xlarge r4.2xlarge r5d.2xlarge r5.2xlarge r5a.2xlarge m5.4xlarge m5d.4xlarge]	{"commit": "dc47849"}
2022-07-14T03:57:46.639Z	INFO	controller.allocation.provisioner/default	Launched instance: i-03ed4fca6f5ffbcc2, hostname: ip-192-168-158-198.cn-northwest-1.compute.internal, type: t3a.2xlarge, zone: cn-northwest-1a, capacityType: on-demand	{"commit": "dc47849"}
2022-07-14T03:57:46.668Z	INFO	controller.allocation.provisioner/default	Bound 5 pod(s) to node ip-192-168-158-198.cn-northwest-1.compute.internal	{"commit": "dc47849"}
```

从日志可以看到，新增加的pod会绑定在新的node上，检查pod和node的信息也可以看到：

```
kubectl get pod -owide
NAME                      READY   STATUS    RESTARTS   AGE     IP                NODE                                                 NOMINATED NODE   READINESS GATES
inflate-b9d769f59-26mhx   1/1     Running   0          7m30s   192.168.159.152   ip-192-168-158-198.cn-northwest-1.compute.internal   <none>           <none>
inflate-b9d769f59-85mc8   1/1     Running   0          7m30s   192.168.147.0     ip-192-168-158-198.cn-northwest-1.compute.internal   <none>           <none>
inflate-b9d769f59-gx4zs   1/1     Running   0          22m     192.168.189.210   ip-192-168-186-250.cn-northwest-1.compute.internal   <none>           <none>
inflate-b9d769f59-p7spt   1/1     Running   0          7m30s   192.168.151.201   ip-192-168-158-198.cn-northwest-1.compute.internal   <none>           <none>
inflate-b9d769f59-vq2f8   1/1     Running   0          7m30s   192.168.130.196   ip-192-168-158-198.cn-northwest-1.compute.internal   <none>           <none>
inflate-b9d769f59-zc4nz   1/1     Running   0          7m30s   192.168.133.192   ip-192-168-158-198.cn-northwest-1.compute.internal   <none>           <none>

kubectl get node

NAME                                                 STATUS   ROLES    AGE     VERSION
ip-192-168-143-99.cn-northwest-1.compute.internal    Ready    <none>   90m     v1.22.9-eks-810597c
ip-192-168-158-198.cn-northwest-1.compute.internal   Ready    <none>   7m47s   v1.22.9-eks-810597c
ip-192-168-186-250.cn-northwest-1.compute.internal   Ready    <none>   22m     v1.22.9-eks-810597c
```

f. 将pod的数量缩减至0观察

```
kubectl scale deployment inflate --replicas 0
```

因为配置了 ttlSecondsAfterEmpty=30 seconds，所以karpenter将会讲空的node进行终止操作，日志如下：

```
2022-07-14T04:09:12.986Z	INFO	controller.Node	Added TTL to empty node ip-192-168-186-250.cn-northwest-1.compute.internal	{"commit": "dc47849"}
2022-07-14T04:09:15.420Z	INFO	controller.Node	Added TTL to empty node ip-192-168-158-198.cn-northwest-1.compute.internal	{"commit": "dc47849"}
2022-07-14T04:09:43.006Z	INFO	controller.Node	Triggering termination after 30s for empty node ip-192-168-186-250.cn-northwest-1.compute.internal	{"commit": "dc47849"}
2022-07-14T04:09:43.033Z	INFO	controller.Termination	Cordoned node ip-192-168-186-250.cn-northwest-1.compute.internal	{"commit": "dc47849"}
2022-07-14T04:09:43.218Z	INFO	controller.Termination	Deleted node ip-192-168-186-250.cn-northwest-1.compute.internal	{"commit": "dc47849"}
2022-07-14T04:09:45.435Z	INFO	controller.Node	Triggering termination after 30s for empty node ip-192-168-158-198.cn-northwest-1.compute.internal	{"commit": "dc47849"}
2022-07-14T04:09:45.464Z	INFO	controller.Termination	Cordoned node ip-192-168-158-198.cn-northwest-1.compute.internal	{"commit": "dc47849"}
2022-07-14T04:09:45.651Z	INFO	controller.Termination	Deleted node ip-192-168-158-198.cn-northwest-1.compute.internal	{"commit": "dc47849"}
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
