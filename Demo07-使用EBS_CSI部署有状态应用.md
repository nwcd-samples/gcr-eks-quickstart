# Demo07-使用EBS CSI部署有状态应用
--
#### Contributor: Tao Dai
#### 更新时间: 2024-12-31
#### 基于EKS版本: EKS 1.31

### 先决条件
为集群创建IAM OIDC提供商

```
eksctl utils associate-iam-oidc-provider --cluster test --approve
```
### 1.配置EBS CSI所需IAM权限

使用eksctl创建EBS CSI使用的IAM Service Account

```
export CLUSTER_NAME=test

eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster $CLUSTER_NAME \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws-cn:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --approve
```
### 2.部署EBS CSI Driver插件

2.1 使用eksctl部署EBS CSI Driver插件
<br>1234567890替换成当前AWS中国区域账号ID

```
eksctl create addon \
	--name aws-ebs-csi-driver \
	--cluster $CLUSTER_NAME \
	--service-account-role-arn arn:aws-cn:iam::1234567890:role/AmazonEKS_EBS_CSI_DriverRole \
	--force
```
2.2 查看EBS CSI Driver插件

```
eksctl get addon --name aws-ebs-csi-driver --cluster test
NAME                    VERSION                 STATUS  ISSUES  IAMROLE                                                         UPDATE AVAILABLE        CONFIGURATION VALUES    POD IDENTITY ASSOCIATION ROLES
aws-ebs-csi-driver      v1.37.0-eksbuild.1      ACTIVE  0       arn:aws-cn:iam::150430853770:role/AmazonEKS_EBS_CSI_DriverRole
```

### 3.部署示例应用程序进行验证

3.1 将EBS CSI驱动程序Github项目克隆到本地

```
sudo yum install git -y
git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
```

3.2 进入dynamic-provisioning目录

```
cd aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning
```

3.3 部署StorageClass, PV以及示例应用

```
kubectl apply -f manifests/
```

3.4 查看StorageClass, PV

```
kubectl get sc,pv | grep ebs
storageclass.storage.k8s.io/ebs-sc   ebs.csi.aws.com         Delete          WaitForFirstConsumer   false                  5s
storageclass.storage.k8s.io/gp2      kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  90m
```

3.5 删除旧的默认StorageClass

```
kubectl delete sc gp2
```

3.6 设置ebs-sc为新的默认StorageClass

```
kubectl patch sc ebs-sc -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

3.7 查看并验证Pod中是否已将数据写入EBS卷

```
kubectl exec -it app -- cat /data/out.txt
Tue Dec 31 08:43:17 UTC 2024
Tue Dec 31 08:43:22 UTC 2024
Tue Dec 31 08:43:27 UTC 2024
Tue Dec 31 08:43:32 UTC 2024
Tue Dec 31 08:43:37 UTC 2024
Tue Dec 31 08:43:42 UTC 2024
```
### 4.清理环境

```
# 删除示例应用程序
kubectl delete -f manifests/

# 删除EBS CSI Driver插件
eksctl delete addon --cluster test --name aws-ebs-csi-driver --preserve

# 删除IAM Serive Account
eksctl delete iamserviceaccount \
  --name ebs-csi-controller-sa \
  --cluster $CLUSTER_NAME
```
