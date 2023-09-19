# Demo04-使用Cluster Autoscaler进行自动伸缩
--
#### Contributor: Tao Dai
#### 更新时间: 2023-09-19
#### 基于EKS版本: EKS 1.27
--

Cluster Autoscaler（CA）集群自动伸缩与Auto Scaling groups集成，CA给客户提供了四种部署的方式：

- One Auto Scaling group
- Multiple Auto Scaling groups
- Auto-Discovery
- Control-plane Node setup

Auto-Discovery 是推荐的集群自动伸缩配置方法，CA将会通过确定ASG加载模版里的实例类型的CPU、内存、GPU资源使用，从而进行集群的扩所容。

## 1.配置ASG

创建并配置ASG的核心参数，最小，最大和期望的实例数量：

### 1.1 检查目前集群ASG的状态

```
aws autoscaling \
    describe-auto-scaling-groups \
    --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='test']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
    --output table
```
注意： cluster-name需要根据自己创建的集群名称进行调整，本次测试案例里eks集群名字是"test"

输出显示：

```
-----------------------------------------------------------------------
|                      DescribeAutoScalingGroups                      |
+------------------------------------------------------+----+----+----+
|  eks-nodegroup-0ec0ebf0-fa82-89de-7f5f-f4dbdd42bee4  |  1 |  2 |  1 |
+------------------------------------------------------+----+----+----+

目前最大值是2
```

### 1.2 修改最大值参数，修改为4

1 检查ASG名称

```
export ASG_NAME=$(aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='test']].AutoScalingGroupName" --output text)
```

2 调整最大值为 4

```
aws autoscaling \
    update-auto-scaling-group \
    --auto-scaling-group-name ${ASG_NAME} \
    --min-size 1 \
    --desired-capacity 2 \
    --max-size 4
```
3  检查新的数值

```
aws autoscaling \
    describe-auto-scaling-groups \
    --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='test']].[AutoScalingGroupName, MinSize, MaxSize,DesiredCapacity]" \
    --output table
```

输出显示：

```
-----------------------------------------------------------------------
|                      DescribeAutoScalingGroups                      |
+------------------------------------------------------+----+----+----+
|  eks-nodegroup-0ec0ebf0-fa82-89de-7f5f-f4dbdd42bee4  |  1 |  4 |  1 |
+------------------------------------------------------+----+----+----+

目前最大的数值为4
```

## 2. IAM roles for service accounts

### 2.1 为集群service account 开启IAM roles（IRSA）

```
eksctl utils associate-iam-oidc-provider \
    --cluster test \
    --approve
```

输出显示：

```
2022-07-11 09:54:47 [ℹ]  eksctl version 0.90.0
2022-07-11 09:54:47 [ℹ]  using region cn-northwest-1
2022-07-11 09:54:48 [ℹ]  will create IAM Open ID Connect provider for cluster "test" in "cn-northwest-1"
2022-07-11 09:54:48 [✔]  created IAM Open ID Connect provider for cluster "test" in "cn-northwest-1"
```

### 2.2 创建service account的IAM policy，用于CA pod与ASG进行交互

```
mkdir ~/cluster-autoscaler

cat <<EoF > ~/cluster-autoscaler/k8s-asg-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
EoF

aws iam create-policy   \
  --policy-name k8s-asg-policy \
  --policy-document file://~/cluster-autoscaler/k8s-asg-policy.json
```

输出显示：

```
{
    "Policy": {
        "PolicyName": "k8s-asg-policy",
        "PolicyId": "ANPAXCZ56Y5GOG3ZVSGPJ",
        "Arn": "arn:aws-cn:iam::1234567890:policy/k8s-asg-policy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-07-11T01:58:19+00:00",
        "UpdateDate": "2022-07-11T01:58:19+00:00"
    }
}
```
### 2.3 为在kube-system namespace内的cluster-autoscaler Service Account创建IAM role

```
eksctl create iamserviceaccount \
    --name cluster-autoscaler \
    --namespace kube-system \
    --cluster eks0707 \
    --attach-policy-arn "arn:aws-cn:iam::1234567890:policy/k8s-asg-policy" \
    --approve \
    --override-existing-serviceaccounts
```

输出显示：

```
2022-07-11 10:26:01 [ℹ]  eksctl version 0.157.0
2022-07-11 10:26:01 [ℹ]  using region cn-northwest-1
2022-07-11 10:26:02 [ℹ]  1 iamserviceaccount (kube-system/cluster-autoscaler) was included (based on the include/exclude rules)
2022-07-11 10:26:02 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2022-07-11 10:26:02 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "kube-system/cluster-autoscaler",
        create serviceaccount "kube-system/cluster-autoscaler",
    } }2022-07-11 10:26:02 [ℹ]  building iamserviceaccount stack "eksctl-test-addon-iamserviceaccount-kube-system-cluster-autoscaler"
2022-07-11 10:26:03 [ℹ]  deploying stack "eksctl-test-addon-iamserviceaccount-kube-system-cluster-autoscaler"
2022-07-11 10:26:03 [ℹ]  waiting for CloudFormation stack "eksctl-test-addon-iamserviceaccount-kube-system-cluster-autoscaler"
2022-07-11 10:26:21 [ℹ]  waiting for CloudFormation stack "eksctl-test-addon-iamserviceaccount-kube-system-cluster-autoscaler"
2022-07-11 10:26:23 [ℹ]  created serviceaccount "kube-system/cluster-autoscaler"
```
查看sa信息

```
kubectl -n kube-system describe sa cluster-autoscaler
```
输出显示：

```
Name:                cluster-autoscaler
Namespace:           kube-system
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws-cn:iam::1234567890:role/eksctl-test-addon-iamserviceaccount-kube-Role1-17T56JSDQ7VAL
Image pull secrets:  <none>
Mountable secrets:   cluster-autoscaler-token-rjfxh
Tokens:              cluster-autoscaler-token-rjfxh
Events:              <none>
```


## 3. 部署Cluster Autoscaler (CA)

### 3.1 使用如下命令部署CA到测试集群

```
wget https://www.eksworkshop.com/beginner/080_scaling/deploy_ca.files/cluster-autoscaler-autodiscover.yaml
```
修改 cluster-autoscaler-autodiscover.yaml,将ekscluster name(eksworkshop-eksctl)替换成创建的ekscluster name(test),并更新Image版本

```
sed -i 's/eksworkshop-eksctl/test/g' cluster-autoscaler-autodiscover.yaml
sed -i 's/v1.20.3/v1.21.0/g' cluster-autoscaler-autodiscover.yaml
```
创建CA

```
kubectl apply -f cluster-autoscaler-autodiscover.yaml
```

输出显示：

```
clusterrole.rbac.authorization.k8s.io/cluster-autoscaler created
role.rbac.authorization.k8s.io/cluster-autoscaler created
clusterrolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
rolebinding.rbac.authorization.k8s.io/cluster-autoscaler created
deployment.apps/cluster-autoscaler created
```

### 3.2 规避CA pod被驱逐，在deployments里增加 cluster-autoscaler.kubernetes.io/safe-to-evict annotation

```
kubectl -n kube-system \
    annotate deployment.apps/cluster-autoscaler \
    cluster-autoscaler.kubernetes.io/safe-to-evict="false"
```

### 3.3 观察CA日志

```
kubectl -n kube-system logs -f deployment/cluster-autoscaler
```

## 4. 使用CA扩展集群

### 4.1 部署测试app

```
cat <<EoF> ~/cluster-autoscaler/nginx.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-to-scaleout
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        service: nginx
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx-to-scaleout
        resources:
          limits:
            cpu: 500m
            memory: 512Mi
          requests:
            cpu: 500m
            memory: 512Mi
EoF

kubectl apply -f ~/cluster-autoscaler/nginx.yaml
kubectl get deployment/nginx-to-scaleout
```
### 4.2 扩展ReplicaSet

```
kubectl scale --replicas=10 deployment/nginx-to-scaleout
```

### 4.3 观察Pod状态

```
kubectl get pods -l app=nginx -o wide --watch
```
输出显示：

```
NAME                                 READY   STATUS    RESTARTS   AGE
nginx-to-scaleout-6fcd49fb84-78tv2   1/1     Running   0          21s
nginx-to-scaleout-6fcd49fb84-8n5tz   1/1     Running   0          21s
nginx-to-scaleout-6fcd49fb84-b6kpf   1/1     Running   0          21s
nginx-to-scaleout-6fcd49fb84-ghcrk   1/1     Running   0          21s
nginx-to-scaleout-6fcd49fb84-gk4r9   0/1     Pending   0          21s
nginx-to-scaleout-6fcd49fb84-hhs5z   1/1     Running   0          21s
nginx-to-scaleout-6fcd49fb84-kcl5t   1/1     Running   0          4m48s
nginx-to-scaleout-6fcd49fb84-x5nwg   1/1     Running   0          21s
nginx-to-scaleout-6fcd49fb84-z7plg   1/1     Running   0          21s
nginx-to-scaleout-6fcd49fb84-z8ltm   1/1     Running   0          21s
```
### 4.4 观察cluster-autoscaler logs & nodes

```
kubectl -n kube-system logs -f deployment/cluster-autoscaler
```

输出显示：

```
I0711 07:51:04.747329       1 flags.go:52] FLAG: --new-pod-scale-up-delay="0s"
I0711 07:51:04.747577       1 flags.go:52] FLAG: --scale-up-from-zero="true"
I0711 07:51:44.281156       1 scale_up.go:586] Final scale-up plan: [{eks-nodegroup-0ec0ebf0-fa82-89de-7f5f-f4dbdd42bee4 3->4 (max: 8)}]
I0711 07:51:44.428350       1 event_sink_logging_wrapper.go:48] Event(v1.ObjectReference{Kind:"Pod", Namespace:"default", Name:"nginx-to-scaleout-6fcd49fb84-mxflx", UID:"38d06f84-bba8-49d3-b7a5-736f7487f8f1", APIVersion:"v1", ResourceVersion:"986308", FieldPath:""}): type: 'Normal' reason: 'TriggeredScaleUp' pod triggered scale-up: [{eks-nodegroup-0ec0ebf0-fa82-89de-7f5f-f4dbdd42bee4 2->4 (max: 8)}]
```

```
kubectl get nodes
```

输出显示：

```
NAME                                                STATUS   ROLES    AGE    VERSION
ip-192-168-105-113.cn-northwest-1.compute.internal   Ready    <none>   40m   v1.27.4-eks-8ccc7ba
ip-192-168-157-4.cn-northwest-1.compute.internal     Ready    <none>   93s   v1.27.4-eks-8ccc7ba
ip-192-168-157-50.cn-northwest-1.compute.internal    Ready    <none>   93s   v1.27.4-eks-8ccc7ba
ip-192-168-172-39.cn-northwest-1.compute.internal    Ready    <none>   40m   v1.27.4-eks-8ccc7ba
```
## 5. 清除环境

```
# 删除示例应用
kubectl delete -f ~/cluster-autoscaler/nginx.yaml

# 删除cluster-autoscaler
kubectl delete -f ~/cluster-autoscaler/cluster-autoscaler-autodiscover.yaml

# 删除IAM Service Account
eksctl delete iamserviceaccount \
  --name cluster-autoscaler \
  --namespace kube-system \
  --cluster test \
  --wait

# 删除IAM Policy
aws iam delete-policy \
  --policy-arn arn:aws-cn:iam::1234567890:policy/k8s-asg-policy

export ASG_NAME=$(aws autoscaling describe-auto-scaling-groups --query "AutoScalingGroups[? Tags[? (Key=='eks:cluster-name') && Value=='eks0707']].AutoScalingGroupName" --output text)

# 恢复ASG实例规模
aws autoscaling \
  update-auto-scaling-group \
  --auto-scaling-group-name ${ASG_NAME} \
  --min-size 1 \
  --desired-capacity 1 \
  --max-size 4

# 删除文件夹
rm -rf ~/cluster-autoscaler

# unset变量
unset ASG_NAME
```
