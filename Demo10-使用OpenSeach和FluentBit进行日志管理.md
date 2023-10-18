# Demo10-使用OpenSeach和Fluent bit进行日志管理
--
#### Contributor: Yi Zheng
#### 更新时间: 2023-10-18
#### 基于EKS版本: EKS 1.27
--

## 1. 先决条件  
### 1.1 准备实验环境：参考Demo 01  
### 1.2 使用eksctl创建集群：参考Demo 02
### 1.3 设置环境变量
  
```
AWS_REGION=cn-northwest-1
AWS_DEFAULT_REGION=cn-northwest-1
CLUSTER_NAME=eksworkshop

AWS_Region=cn-northwest-1
ACCOUNT_ID=<>
```

## 2. 配置Fluent Bit使用的IRSA
```
# 执行以下命令创建OpenID Connect (OIDC) 身份提供商
eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}
    
# 执行以下命令新建IAM Policy
mkdir -p ~/environment/logging/
export ES_DOMAIN_NAME="eksworkshop-logging"
cat <<EoF > ~/environment/logging/fluent-bit-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "es:ESHttp*"
            ],
            "Resource": "arn:aws-cn:es:${AWS_REGION}:${ACCOUNT_ID}:domain/${ES_DOMAIN_NAME}",
            "Effect": "Allow"
        }
    ]
}
EoF

aws iam create-policy   \
  --policy-name fluent-bit-policy \
  --policy-document file://~/environment/logging/fluent-bit-policy.json
```

## 3. 创建Fluent Bit使用的Service Account
```
kubectl create namespace logging

eksctl create iamserviceaccount \
    --region ${AWS_REGION} \
    --name fluent-bit \
    --namespace logging \
    --cluster eksworkshop \
    --attach-policy-arn "arn:aws-cn:iam::${ACCOUNT_ID}:policy/fluent-bit-policy" \
    --approve \
    --override-existing-serviceaccounts

kubectl -n logging describe sa fluent-bit
# 参考输出
Name:                fluent-bit
Namespace:           logging
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws-cn:iam::332433839685:role/eksctl-eksworkshop-addon-iamserviceaccount-l-Role1-1O4FLW0SEVFKY
Image pull secrets:  <none>
Mountable secrets:   fluent-bit-token-qcnts
Tokens:              fluent-bit-token-qcnts
Events:              <none>
```

## 4. 部署Amazon Opensearch Domain
```
# 设置变量
export ES_DOMAIN_NAME="eksworkshop-logging"
export ES_VERSION="OpenSearch_1.0"
export ES_DOMAIN_USER="eksworkshop"
export ES_DOMAIN_PASSWORD="ezQZsym3UUQSlj8F_Ek1$"

# 下载并更新文件
curl -sS https://archive.eksworkshop.com/intermediate/230_logging/deploy.files/es_domain.json \
  | envsubst > ~/environment/logging/es_domain.json
sed -i '18,20d' ~/environment/logging/es_domain.json
sed -i 's/ESHttp//' ~/environment/logging/es_domain.json
sed -i 's@arn:aws:es:::domain/eksworkshop-logging/@@' ~/environment/logging/es_domain.json
  
# 创建集群，请等待创建完成
aws opensearch create-domain \
  --cli-input-json  file://~/environment/logging/es_domain.json

while true
do
   ES_ENDPOINT=$(aws opensearch describe-domain --domain-name ${ES_DOMAIN_NAME} --region ${AWS_REGION} --output text --query "DomainStatus.Endpoint")
   if [ $ES_ENDPOINT == "None" ]; then
      echo "Still provisioning"
      sleep 10;
      continue
   else
      echo "Provisioning finished"
      break
   fi
done
```

## 5. 将IAM Role映射到用户
```
# 安装jq
sudo yum install jq -y

# 获取Fluent Bit Role ARN
export FLUENTBIT_ROLE=$(eksctl get iamserviceaccount --cluster eksworkshop --namespace logging --region ${AWS_REGION} -o json | jq '.[].status.roleARN' -r)

# 获取Amazon OpenSearch Endpoint
export ES_ENDPOINT=$(aws opensearch describe-domain --domain-name ${ES_DOMAIN_NAME} --region ${AWS_REGION} --output text --query "DomainStatus.Endpoint")

# 更新Elasticsearch内部数据库
curl -sS -u "${ES_DOMAIN_USER}:${ES_DOMAIN_PASSWORD}" \
    -X PATCH \
    https://${ES_ENDPOINT}/_opendistro/_security/api/rolesmapping/all_access?pretty \
    -H 'Content-Type: application/json' \
    -d'
[
  {
    "op": "add", "path": "/backend_roles", "value": ["'${FLUENTBIT_ROLE}'"]
  }
]
'

# 参考输出：
{
  "status" : "OK",
  "message" : "'all_access' updated."
}
```

## 6. 部署Fluent Bit
```
cd ~/environment/logging

# 获取Amazon OpenSearch Endpoint
export ES_ENDPOINT=$(aws es describe-elasticsearch-domain --domain-name ${ES_DOMAIN_NAME} --output text --query "DomainStatus.Endpoint")
export AWS_REGION=cn-northwest-1

cd ~/environment/logging/
wget https://archive.eksworkshop.com/intermediate/230_logging/deploy.files/fluentbit.yaml
envsubst < ~/environment/logging/fluentbit.yaml > ~/environment/logging/fluentbit_new.yaml

# 部署fluentbit pod    
kubectl apply -f ~/environment/logging/fluentbit.yaml
kubectl --namespace=logging get pods

# 参考输出：
NAME               READY   STATUS    RESTARTS   AGE
fluent-bit-9kqwf   1/1     Running   0          16s
fluent-bit-svrw2   1/1     Running   0          15s

# 如果出现如下报错，需要修改fluentbit.yaml文件，并在output-elasticsearch.conf段落里设置AWS_Region的值
[2022/12/02 11:22:22] [  Error] File output-elasticsearch.conf
[2022/12/02 11:22:22] [  Error] Error in line 8: Key has an empty value
```

## 7. Opensearch Dashboards
```
echo "OpenSearch Dashboards URL: https://${ES_ENDPOINT}/_dashboards/
OpenSearch Dashboards user: ${ES_DOMAIN_USER}
OpenSearch Dashboards password: ${ES_DOMAIN_PASSWORD}"
```
7.1 登录OpenSearch Dashboards
![](https://archive.eksworkshop.com/images/logging/opensearch_01.png)
7.2 选择Explore on my own
![](https://archive.eksworkshop.com/images/logging/opensearch_02.png)
7.3 选择Confirm
![](https://archive.eksworkshop.com/images/logging/opensearch_03.png)
7.4 选择OpenSearch Dashboards
![](https://archive.eksworkshop.com/images/logging/opensearch_04.png)
7.5 选择Add your data
![](https://archive.eksworkshop.com/images/logging/opensearch_05.png)
7.6 选择Create index pattern
![](https://archive.eksworkshop.com/images/logging/opensearch_06.png)
7.7 选择Create index pattern
![](https://archive.eksworkshop.com/images/logging/opensearch_06.png)
7.8 写入*fluent-bit*
![](https://archive.eksworkshop.com/images/logging/opensearch_07.png)
7.9 选择@timestamp
![](https://archive.eksworkshop.com/images/logging/opensearch_08.png)
7.10 选择Discover查看数据
![](https://archive.eksworkshop.com/images/logging/opensearch_09.png)

## 8. 清理环境
```
cd  ~/environment/

# 删除Fluent bit
kubectl delete -f ~/environment/logging/fluentbit.yaml

# 删除OpenSearch Domain
aws opensearch delete-domain \
    --domain-name ${ES_DOMAIN_NAME}

# 删除IAM Service Account
eksctl delete iamserviceaccount \
    --name fluent-bit \
    --namespace logging \
    --cluster eksworkshop \
    --wait

# 删除IAM Policy
aws iam delete-policy   \
  --policy-arn "arn:aws-cn:iam::${ACCOUNT_ID}:policy/fluent-bit-policy"

# 删除namespace
kubectl delete namespace logging

# 删除文件夹
rm -rf ~/environment/logging

# unset环境变量
unset ES_DOMAIN_NAME
unset ES_VERSION
unset ES_DOMAIN_USER
unset ES_DOMAIN_PASSWORD
unset FLUENTBIT_ROLE
unset ES_ENDPOINT
unset AWS_REGION
unset AWS_Region
unset AWS_DEFAULT_REGION
unset CLUSTER_NAME 
unset ACCOUNT_ID
```
