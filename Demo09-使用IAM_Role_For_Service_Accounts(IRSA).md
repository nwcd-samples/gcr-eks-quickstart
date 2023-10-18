# Demo09-使用IAM Role for Service Account(IRSA)
--
#### Contributor: Yi Zheng
#### 更新时间: 2023-10-18
#### 基于EKS版本: EKS 1.27
--

## 1. 先决条件  
1.1 准备实验环境：参考Demo 01  
1.2 使用eksctl创建集群：参考Demo 02，不要执行 4. 镜像处置(针对中国区)

## 2. 配置IAM Role, ServiceAccount
使用eksctl 创建service account  

```
# 执行以下命令创建OpenID Connect (OIDC) 身份提供商
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}

#创建serviceaccount s3-echoer with IAM role
eksctl create iamserviceaccount --name s3-echoer --namespace default \
    --cluster ${CLUSTER_NAME} --attach-policy-arn arn:aws-cn:iam::aws:policy/AmazonS3FullAccess \
    --approve --override-existing-serviceaccounts --region ${AWS_REGION}
```
## 3 部署测试访问S3的应用
使用已有s3 bucket或创建s3 bucket, 请确保bucket名字唯一才能创建成功.

```
# 设置环境变量TARGET_BUCKET,Pod访问的S3 bucket
TARGET_BUCKET=eksworkshop-irsa-2022-renew
if [ $(aws s3 ls | grep $TARGET_BUCKET | wc -l) -eq 0 ]; then
    aws s3api create-bucket  --bucket $TARGET_BUCKET  --create-bucket-configuration LocationConstraint=$AWS_REGION  --region $AWS_REGION
else
    echo "S3 bucket $TARGET_BUCKET existed, skip creation"
fi

# 部署Job
cat <<EoF> ~/job-s3.yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: list-bucket
spec:
  template:
    spec:
      serviceAccountName: s3-echoer
      containers:
      - name: list-bucket
        image: public.ecr.aws/docker/library/amazonlinux:2
        command:
        - "sh"
        - "-c"
        - "yum install pip -y && pip install awscli && aws s3 ls"
      restartPolicy: Never
EoF

kubectl apply -f ~/job-s3.yaml

# 验证
kubectl logs [pod name]

# 参考输出
2022-07-12 23:35:06 eksworkshop-irsa-2022-renew
```

## 4. 清理环境
```
kubectl delete -f ~/job-s3.yaml
```
