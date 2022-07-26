# Demo09-使用IAM Role for Service Account(IRSA)
--
#### Contributor: Yi Zheng
--

## 1. 先决条件  
1.1 准备实验环境：参考Demo 01  
1.2 使用eksctl创建集群：参考Demo 02  

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
        image: amazonlinux:2018.03
        command:
        - "sh"
        - "-c"
        - "yum install unzip -y && curl 'https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip' -o 'awscliv2.zip' && unzip awscliv2.zip && ./aws/install && aws s3 ls"
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
