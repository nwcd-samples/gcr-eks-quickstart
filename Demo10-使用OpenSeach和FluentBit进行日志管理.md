# Demo10-使用OpenSeach和Fluent bit进行日志管理
--
#### Contributor: Yi Zheng
--

## 1. 先决条件  
### 1.1 准备实验环境：参考Demo 01  
### 1.2 使用eksctl创建集群：参考Demo 02
### 1.3 设置环境变量
  
```
AWS_REGION=cn-northwest-1
AWS_Region=cn-northwest-1
AWS_DEFAULT_REGION=cn-northwest-1  
CLUSTER_NAME=eksworkshop  
ACCOUNT_ID=<>
```

## 2. 配置Fluent Bit使用的IRSA
```
# 执行以下命令创建OpenID Connect (OIDC) 身份提供商
eksctl utils associate-iam-oidc-provider \
    --cluster eksworkshop \
    --approve
    
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
curl -sS https://www.eksworkshop.com/intermediate/230_logging/deploy.files/es_domain.json \
  | envsubst > ~/environment/logging/es_domain.json
sed -i '18,20d' ~/environment/logging/es_domain.json
sed -i 's/ESHttp//' ~/environment/logging/es_domain.json
sed -i 's@arn:aws:es:::domain/eksworkshop-logging/@@' ~/environment/logging/es_domain.json
  
# 创建集群，请等待创建完成
aws opensearch create-domain \
  --cli-input-json  file://~/environment/logging/es_domain.json
```

## 5. 将IAM Role映射到用户
```
# 获取Fluent Bit Role ARN
export FLUENTBIT_ROLE=$(eksctl get iamserviceaccount --cluster eksworkshop --namespace logging -o json | jq '.[].status.roleARN' -r)

# 获取Amazon OpenSearch Endpoint
export ES_ENDPOINT=$(aws opensearch describe-domain --domain-name ${ES_DOMAIN_NAME} --output text --query "DomainStatus.Endpoint")

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

curl -Ss https://www.eksworkshop.com/intermediate/230_logging/deploy.files/fluentbit.yaml \
    | envsubst > ~/environment/logging/fluentbit.yaml

# 部署fluentbit pod    
kubectl apply -f ~/environment/logging/fluentbit.yaml
kubectl --namespace=logging get pods

# 参考输出：
NAME               READY   STATUS    RESTARTS   AGE
fluent-bit-9kqwf   1/1     Running   0          16s
fluent-bit-svrw2   1/1     Running   0          15s
```

## 7. Opensearch Dashboards
```
echo "OpenSearch Dashboards URL: https://${ES_ENDPOINT}/_dashboards/
OpenSearch Dashboards user: ${ES_DOMAIN_USER}
OpenSearch Dashboards password: ${ES_DOMAIN_PASSWORD}"
```
7.1 登录OpenSearch Dashboards
![](https://www.eksworkshop.com/images/logging/opensearch_01.png)
7.2 选择Explore on my own
![](https://www.eksworkshop.com/images/logging/opensearch_02.png)
7.3 选择Confirm
![](https://www.eksworkshop.com/images/logging/opensearch_03.png)
7.4 选择OpenSearch Dashboards
![](https://www.eksworkshop.com/images/logging/opensearch_04.png)
7.5 选择Add your data
![](https://www.eksworkshop.com/images/logging/opensearch_05.png)
7.6 选择Create index pattern
![](https://www.eksworkshop.com/images/logging/opensearch_06.png)
7.7 选择Create index pattern
![](https://www.eksworkshop.com/images/logging/opensearch_06.png)
7.8 写入*fluent-bit*
![](https://www.eksworkshop.com/images/logging/opensearch_07.png)
7.9 选择@timestamp
![](https://www.eksworkshop.com/images/logging/opensearch_08.png)
7.10 选择Discover查看数据
![](https://www.eksworkshop.com/images/logging/opensearch_09.png)

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