# Demo06-使用EBS CSI部署有状态应用
--
#### Contributor: Tao Dai
--
### 更新20221204：建议使用EKS Add-ons安装

### 先决条件
为集群创建IAM OIDC提供商

```
eksctl utils associate-iam-oidc-provider --cluster prod --approve
```
### 1.配置EBS CSI所需IAM权限


1.1 创建IAM策略

a.将IAM Policy文件下载到本地

```
curl -o example-iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-ebs-csi-driver/master/docs/example-iam-policy.json
```
b.更新IAM Policy文件以便中国区使用

```
sed -i 's/arn:aws/arn:aws-cn/g' example-iam-policy.json

```
c.创建IAM Policy

```
POLICY_ARN=$(aws iam create-policy \
    --policy-name AmazonEKS_EBS_CSI_Driver_Policy \
    --policy-document file://example-iam-policy.json \
    --query 'Policy.Arn' \
    --output text)
```

1.2 创建EBS CSI使用的IAM Service Account

a.使用eksctl创建IAM Service Account

```
CLUSTER_NAME=prod

eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster $CLUSTER_NAME  \
    --attach-policy-arn $POLICY_ARN \
    --approve \
    --override-existing-serviceaccounts
```
b.获取EBS CSI IAM Role的ARN

```
ROLE_ARN=$(aws cloudformation describe-stacks \
    --stack-name eksctl-$CLUSTER_NAME-addon-iamserviceaccount-kube-system-ebs-csi-controller-sa \
    --query='Stacks[].Outputs[?OutputKey==`Role1`].OutputValue' \
    --output text)
```
### 2.部署EBS CSI驱动程序

2.1 中国区使用国内容器镜像站

```
kubectl apply -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml
```
2.2 部署EBS CSI驱动程序

a.部署EBS CSI

```
kubectl apply -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.6"
```
b.对service account添加注解

```
kubectl annotate serviceaccount ebs-csi-controller-sa \
    -n kube-system eks.amazonaws.com.cn/role-arn=$ROLE_ARN
```

c.删除驱动程序Pod，以使其按照新的IAM Role重新部署

```
kubectl delete pods \
    -n kube-system \
    -l=app=ebs-csi-controller
```

### 3.部署示例应用程序进行验证

3.1 将EBS CSI驱动程序Github项目克隆到本地

```
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
storageclass.storage.k8s.io/ebs-sc (default)   ebs.csi.aws.com   Delete          WaitForFirstConsumer   false                  3h37m
persistentvolume/pvc-d36f352d-db1b-4130-88cf-7e8ce9944bba   4Gi        RWO            Delete           Bound    default/ebs-claim   ebs-sc                  3h37m
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
Wed Jul 6 07:17:31 UTC 2022
Wed Jul 6 07:17:36 UTC 2022
Wed Jul 6 07:17:41 UTC 2022
Wed Jul 6 07:17:46 UTC 2022
Wed Jul 6 07:17:51 UTC 2022
```
### 4.清理环境

```
# 删除示例应用程序
kubectl delete -f manifests/

# 删除EBS CSI驱动程序
kubectl delete -k "github.com/kubernetes-sigs/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/?ref=release-1.6"

# 删除IAM Serive Account
eksctl delete iamserviceaccount \
  --name ebs-csi-controller-sa \
  --cluster $CLUSTER_NAME

# 删除IAM Policy
aws iam delete-policy --policy-arn $POLICY_ARN
```


