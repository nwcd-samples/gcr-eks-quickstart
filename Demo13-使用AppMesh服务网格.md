# Demo13-使用App Mesh服务网格

--
#### Contributor: Kunyao Han
--

在本次Deam中，我们将使用以下产品目录应用程序部署示例向您介绍以下流行的 App Mesh 用例：

- 使用AWS Fargate在Amazon EKS中部署一个基于微服务的应用程序
- 配置一个App Mesh虚拟网关，将流量路由到应用服务
- 启用App Mesh的可观察性功能，包括为Fargate、Amazon Cloudwatch Container Insights和AWS X-Ray追踪提供日志记录。

## 1. EKS Fargate和可观测性设置

在本章中，我们将在你现有的EKS集群eksworkshop-eksctl中执行以下任务。

- 创建Fargate配置文件
-  启用OIDC提供者
- 为应用程序的部署创建命名空间
- 为应用程序命名空间prodcatalog-ns创建IRSA（服务账户的IAM角色）
- 启用日志和指标的可观察性

### 1.1 前置条件

a. 已经创建好了一个EKS集群
<br>b. 需要具备对 EKS, AWS App Mesh, Amazon Cloudwatch, AWS X-Ray访问权限的 console credentials
<br>c. 对 AWS_REGION 和 ACCOUNT_ID 进行检查

```	
test -n "$AWS_REGION" && echo AWS_REGION is "$AWS_REGION" || echo AWS_REGION is not set
test -n "$ACCOUNT_ID" && echo ACCOUNT_ID is "$ACCOUNT_ID" || echo ACCOUNT_ID is not set
```	
如果没有设置，使用如下命令配置：

```	
export ACCOUNT_ID=<your_account_id>
export AWS_REGION=<your_aws_region>
```	
d.下载git相关文件到本地

```	
git clone https://github.com/aws-containers/eks-app-mesh-polyglot-demo.git
cd eks-app-mesh-polyglot-demo
```	

### 1.2 创建Fargate Profile

#### 1.2.1 创建IAM OIDC provider

注意： region和cluser需要依据实际情况进行修改

```	
eksctl utils associate-iam-oidc-provider \
    --region cn-northwest-1 \
    --cluster eks0714 \
    --approve
```	

#### 1.2.2 创建 Namespace，IAM Role 和 ServiceAccount

```	
kubectl create namespace prodcatalog-ns

cd eks-app-mesh-polyglot-demo
aws iam create-policy \
    --policy-name ProdEnvoyNamespaceIAMPolicy \
    --policy-document file://deployment/envoy-iam-policy.json

eksctl create iamserviceaccount --cluster eks0714 \
  --namespace prodcatalog-ns \
  --name prodcatalog-envoy-proxies \
  --attach-policy-arn arn:aws:iam::$ACCOUNT_ID:policy/ProdEnvoyNamespaceIAMPolicy \
  --override-existing-serviceaccounts \
  --approve 
```	
输出显示：

```	
namespace/prodcatalog-ns created

{
    "Policy": {
        "PolicyName": "ProdEnvoyNamespaceIAMPolicy",
        "PolicyId": "ANPAXCZ56Y5GJWS5NCPKF",
        "Arn": "arn:aws-cn:iam::487071860556:policy/ProdEnvoyNamespaceIAMPolicy",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-07-14T06:25:00+00:00",
        "UpdateDate": "2022-07-14T06:25:00+00:00"
    }
}

2022-07-14 14:26:05 [ℹ]  eksctl version 0.90.0
2022-07-14 14:26:05 [ℹ]  using region cn-northwest-1
2022-07-14 14:26:06 [ℹ]  1 existing iamserviceaccount(s) (karpenter/karpenter) will be excluded
2022-07-14 14:26:06 [ℹ]  1 iamserviceaccount (prodcatalog-ns/prodcatalog-envoy-proxies) was included (based on the include/exclude rules)
2022-07-14 14:26:06 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2022-07-14 14:26:06 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "prodcatalog-ns/prodcatalog-envoy-proxies",
        create serviceaccount "prodcatalog-ns/prodcatalog-envoy-proxies",
    } }2022-07-14 14:26:06 [ℹ]  building iamserviceaccount stack "eksctl-eks0714-addon-iamserviceaccount-prodcatalog-ns-prodcatalog-envoy-proxies"
2022-07-14 14:26:06 [ℹ]  deploying stack "eksctl-eks0714-addon-iamserviceaccount-prodcatalog-ns-prodcatalog-envoy-proxies"
2022-07-14 14:26:06 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-addon-iamserviceaccount-prodcatalog-ns-prodcatalog-envoy-proxies"
2022-07-14 14:26:23 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-addon-iamserviceaccount-prodcatalog-ns-prodcatalog-envoy-proxies"
2022-07-14 14:26:23 [ℹ]  created serviceaccount "prodcatalog-ns/prodcatalog-envoy-proxies"
```	
检查sa与role关联情况：

```	
kubectl describe sa prodcatalog-envoy-proxies -n prodcatalog-ns
```	

输出显示如下：

```	
Name:                prodcatalog-envoy-proxies
Namespace:           prodcatalog-ns
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws-cn:iam::487071860556:role/eksctl-eks0714-addon-iamserviceaccount-prodc-Role1-1PGZAK1HGD3NG
Image pull secrets:  <none>
Mountable secrets:   prodcatalog-envoy-proxies-token-l9jhh
Tokens:              prodcatalog-envoy-proxies-token-l9jhh
Events:              <none>
```	

#### 1.2.3 创建Fargate profile

```	
cd eks-app-mesh-polyglot-demo
envsubst < ./deployment/clusterconfig.yaml | eksctl create fargateprofile -f -
```	
输出显示：

```	
2022-07-14 14:36:27 [ℹ]  eksctl version 0.90.0
2022-07-14 14:36:27 [ℹ]  using region cn-northwest-1
2022-07-14 14:36:28 [ℹ]  deploying stack "eksctl-eks0714-fargate"
2022-07-14 14:36:28 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-fargate"
2022-07-14 14:36:47 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-fargate"
2022-07-14 14:36:47 [ℹ]  creating Fargate profile "fargate-productcatalog" on EKS cluster "eks0714"
2022-07-14 14:38:57 [ℹ]  created Fargate profile "fargate-productcatalog" on EKS cluster "eks0714"
```	

检查fargate profile：

```	

2022-07-14 14:40:19 [ℹ]  eksctl version 0.90.0
2022-07-14 14:40:19 [ℹ]  using region cn-northwest-1
- name: fargate-productcatalog
  podExecutionRoleARN: arn:aws-cn:iam::487071860556:role/eksctl-eks0714-fargate-FargatePodExecutionRole-TKUI24OPEQE2
  selectors:
  - labels:
      app: prodcatalog
    namespace: prodcatalog-ns
  status: ACTIVE
  subnets:
  - subnet-0321df0f0ae09292f
  - subnet-0debabfbcf7387bfd
  - subnet-00ff18067fe034e2d
```	

### 1.3 可观测性设置

#### 1.3.1 启用 Amazon Cloudwatch Container Insights

a. 创建 IAM role for the cloudwatch-agent service account

```	
eksctl create iamserviceaccount \
  --cluster eks0714 \
  --namespace amazon-cloudwatch \
  --name cloudwatch-agent \
  --attach-policy-arn  arn:aws-cn:iam::aws:policy/CloudWatchAgentServerPolicy \
  --override-existing-serviceaccounts \
  --approve
```	
输出显示：

```	
2022-07-14 14:44:02 [ℹ]  eksctl version 0.90.0
2022-07-14 14:44:02 [ℹ]  using region cn-northwest-1
2022-07-14 14:44:03 [ℹ]  2 existing iamserviceaccount(s) (karpenter/karpenter,prodcatalog-ns/prodcatalog-envoy-proxies) will be excluded
2022-07-14 14:44:03 [ℹ]  1 iamserviceaccount (amazon-cloudwatch/cloudwatch-agent) was included (based on the include/exclude rules)
2022-07-14 14:44:03 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2022-07-14 14:44:03 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "amazon-cloudwatch/cloudwatch-agent",
        create serviceaccount "amazon-cloudwatch/cloudwatch-agent",
    } }2022-07-14 14:44:03 [ℹ]  building iamserviceaccount stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-cloudwatch-agent"
2022-07-14 14:44:03 [ℹ]  deploying stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-cloudwatch-agent"
2022-07-14 14:44:03 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-cloudwatch-agent"
2022-07-14 14:44:20 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-cloudwatch-agent"
2022-07-14 14:44:37 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-cloudwatch-agent"
2022-07-14 14:44:38 [ℹ]  created namespace "amazon-cloudwatch"
2022-07-14 14:44:38 [ℹ]  created serviceaccount "amazon-cloudwatch/cloudwatch-agent"
```	

b. Create an IAM role for the fluent service account

```	
eksctl create iamserviceaccount \
  --cluster eks0714 \
  --namespace amazon-cloudwatch \
  --name fluentd \
  --attach-policy-arn  arn:aws-cn:iam::aws:policy/CloudWatchAgentServerPolicy \
  --override-existing-serviceaccounts \
  --approve
```	
输出显示：

```	
2022-07-14 14:47:15 [ℹ]  eksctl version 0.90.0
2022-07-14 14:47:15 [ℹ]  using region cn-northwest-1
2022-07-14 14:47:16 [ℹ]  3 existing iamserviceaccount(s) (amazon-cloudwatch/cloudwatch-agent,karpenter/karpenter,prodcatalog-ns/prodcatalog-envoy-proxies) will be excluded
2022-07-14 14:47:16 [ℹ]  1 iamserviceaccount (amazon-cloudwatch/fluentd) was included (based on the include/exclude rules)
2022-07-14 14:47:16 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2022-07-14 14:47:16 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "amazon-cloudwatch/fluentd",
        create serviceaccount "amazon-cloudwatch/fluentd",
    } }2022-07-14 14:47:16 [ℹ]  building iamserviceaccount stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-fluentd"
2022-07-14 14:47:16 [ℹ]  deploying stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-fluentd"
2022-07-14 14:47:16 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-fluentd"
2022-07-14 14:47:35 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-fluentd"
2022-07-14 14:47:36 [ℹ]  created serviceaccount "amazon-cloudwatch/fluentd"
```	
c. 部署 Container Insights

下载 cwagent-fluentd-quickstart.yaml 

```	
https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluentd-quickstart.yaml 
```	
将yaml文件内容保存到本地，使用同样的名称cwagent-fluentd-quickstart.yaml，并将yaml文件中{{cluster_name}}替换为创建的集群名称，本次demo中集群的名字为eks0714；将{{region_name}}替换成本次测试的region，本次demo中region为cn-northwest-1 

创建fluentd

```	
kubectl -f cwagent-fluentd-quickstart.yaml 

```	
输出显示：

```	
namespace/amazon-cloudwatch created
serviceaccount/cloudwatch-agent created
clusterrole.rbac.authorization.k8s.io/cloudwatch-agent-role created
clusterrolebinding.rbac.authorization.k8s.io/cloudwatch-agent-role-binding created
configmap/cwagentconfig created
daemonset.apps/cloudwatch-agent created
configmap/cluster-info created
serviceaccount/fluentd created
clusterrole.rbac.authorization.k8s.io/fluentd-role created
clusterrolebinding.rbac.authorization.k8s.io/fluentd-role-binding created
configmap/fluentd-config created
daemonset.apps/fluentd-cloudwatch created
```	
检查相关daemonsets

```	
kubectl -n amazon-cloudwatch get daemonsets
```	
输出显示：

```	

NAME                 DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
cloudwatch-agent     1         1         1       1            1           <none>          102s
fluentd-cloudwatch   1         1         1       1            1           <none>          100s
```	

### 1.3.2 在 CloudWatch 中开启 Prometheus Metrics 


a. 创建 IAM role for the prometheus service account

```	
eksctl create iamserviceaccount \
  --cluster eks0714 \
  --namespace amazon-cloudwatch \
  --name cwagent-prometheus \
  --attach-policy-arn  arn:aws-cn:iam::aws:policy/CloudWatchAgentServerPolicy \
  --override-existing-serviceaccounts \
  --approve
```	
输出显示：

```	
2022-07-14 15:19:17 [ℹ]  eksctl version 0.90.0
2022-07-14 15:19:17 [ℹ]  using region cn-northwest-1
2022-07-14 15:19:18 [ℹ]  4 existing iamserviceaccount(s) (amazon-cloudwatch/cloudwatch-agent,amazon-cloudwatch/fluentd,karpenter/karpenter,prodcatalog-ns/prodcatalog-envoy-proxies) will be excluded
2022-07-14 15:19:18 [ℹ]  1 iamserviceaccount (amazon-cloudwatch/cwagent-prometheus) was included (based on the include/exclude rules)
2022-07-14 15:19:18 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2022-07-14 15:19:18 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "amazon-cloudwatch/cwagent-prometheus",
        create serviceaccount "amazon-cloudwatch/cwagent-prometheus",
    } }2022-07-14 15:19:18 [ℹ]  building iamserviceaccount stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-cwagent-prometheus"
2022-07-14 15:19:18 [ℹ]  deploying stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-cwagent-prometheus"
2022-07-14 15:19:18 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-cwagent-prometheus"
2022-07-14 15:19:34 [ℹ]  waiting for CloudFormation stack "eksctl-eks0714-addon-iamserviceaccount-amazon-cloudwatch-cwagent-prometheus"
2022-07-14 15:19:35 [ℹ]  created serviceaccount "amazon-cloudwatch/cwagent-prometheus"
```	

b. 安装 Prometheus Agent


下载prometheus-eks.yaml

```	
https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/service/cwagent-prometheus/prometheus-eks.yaml
```	
将yaml文件内容保存到本地，使用同样的名称prometheus-eks.yaml

```	
kubectl apply -f prometheus-eks.yaml
```	
输出显示：

```	
namespace/amazon-cloudwatch unchanged
configmap/prometheus-cwagentconfig created
configmap/prometheus-config created
Warning: resource serviceaccounts/cwagent-prometheus is missing the kubectl.kubernetes.io/last-applied-configuration annotation which is required by kubectl apply. kubectl apply should only be used on resources created declaratively by either kubectl create --save-config or kubectl apply. The missing annotation will be patched automatically.
serviceaccount/cwagent-prometheus configured
clusterrole.rbac.authorization.k8s.io/cwagent-prometheus-role created
clusterrolebinding.rbac.authorization.k8s.io/cwagent-prometheus-role-binding created
deployment.apps/cwagent-prometheus created
```	

c. 确认 Agent 运行正常 


```	
kubectl get pod -l "app=cwagent-prometheus" -n amazon-cloudwatch

NAME                                  READY   STATUS    RESTARTS   AGE
cwagent-prometheus-7c8db687db-x5x2t   1/1     Running   0          117s
```	

### 1.3.3 启用 Fargate 日志

a. 创建 Fluent Bit 专用的 aws-observability namespace 和 ConfigMap 

```	
cd eks-app-mesh-polyglot-demo
envsubst < ./deployment/fluentbit-config.yaml | kubectl apply -f -
```	
输出显示：

```	
namespace/aws-observability created
configmap/aws-logging created
```	
b.  确认 Fluent Bit ConfigMap 部署成功

```	
kubectl -n aws-observability get cm
```

输出显示：

```
NAME               DATA   AGE
aws-logging        1      12s
kube-root-ca.crt   1      12s
```	

c. 赋予 Fluent Bit 写入 CloudWatch 的权限

下载permissions.json

```	
https://raw.githubusercontent.com/aws-samples/amazon-eks-fluent-logging-examples/mainline/examples/fargate/cloudwatchlogs/permissions.json
```	

将yaml文件内容保存到本地，使用同样的名称 permissions.json

d. 创建 IAM Policy

```	
aws iam create-policy \
        --policy-name FluentBitEKSFargate \
        --policy-document file://permissions.json 
```	
输出显示：

```	
{
    "Policy": {
        "PolicyName": "FluentBitEKSFargate",
        "PolicyId": "ANPAXCZ56Y5GF7OZM7BKZ",
        "Arn": "arn:aws-cn:iam::487071860556:policy/FluentBitEKSFargate",
        "Path": "/",
        "DefaultVersionId": "v1",
        "AttachmentCount": 0,
        "PermissionsBoundaryUsageCount": 0,
        "IsAttachable": true,
        "CreateDate": "2022-07-15T02:31:12+00:00",
        "UpdateDate": "2022-07-15T02:31:12+00:00"
    }
}
```	

e. 将 Policy 绑定到 Pod Execution Role

```	
export PodRole=$(aws eks describe-fargate-profile --cluster-name eks0714 --fargate-profile-name fargate-productcatalog --query 'fargateProfile.podExecutionRoleArn' | sed -n 's/^.*role\/\(.*\)".*$/\1/ p')

aws iam attach-role-policy \
        --policy-arn arn:aws-cn:iam::${ACCOUNT_ID}:policy/FluentBitEKSFargate \
        --role-name ${PodRole}
	
echo $PodRole
```	


### 1.3.4 开启 Amazon EKS 控制层面日志

```	
eksctl utils update-cluster-logging \
    --enable-types all \
    --region ${AWS_REGION} \
    --cluster eks0714 \
    --approve
```	
输出显示：

```	
2022-07-15 10:35:34 [ℹ]  eksctl version 0.90.0
2022-07-15 10:35:34 [ℹ]  using region cn-northwest-1
2022-07-15 10:35:35 [ℹ]  will update CloudWatch logging for cluster "eks0714" in "cn-northwest-1" (enable types: api, audit, authenticator, controllerManager, scheduler & no types to disable)
2022-07-15 10:35:35 [ℹ]  waiting for requested "LoggingUpdate" in cluster "eks0714" to succeed
2022-07-15 10:35:52 [ℹ]  waiting for requested "LoggingUpdate" in cluster "eks0714" to succeed
2022-07-15 10:36:09 [ℹ]  waiting for requested "LoggingUpdate" in cluster "eks0714" to succeed
2022-07-15 10:36:09 [✔]  configured CloudWatch logging for cluster "eks0714" in "cn-northwest-1" (enabled types: api, audit, authenticator, controllerManager, scheduler & no types disabled)
```	

## 2. 部署产品目录应用程序

要了解AWS App Mesh，最好还要了解运行在它上面的任何应用程序。在这一章中，我们将引导你完成应用程序设置和部署的以下部分。

- 讨论应用架构
- 构建应用服务容器镜像
- 将容器镜像推送到亚马逊ECR
- 将应用服务部署到亚马逊EKS集群中，最初没有AWS App Mesh
- 测试服务之间的连接性

### 2.1 创建产品目录应用程序

#### 2.1.1 构建应用服务 

a 构建并推送容器镜像到ECR

```	
cd eks-app-mesh-polyglot-demo

aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com.cn
PROJECT_NAME=eks-app-mesh-demo
export APP_VERSION=1.0
```	

注意，需要安装docker在本地

```	
for app in catalog_detail product_catalog frontend_node; do
  aws ecr describe-repositories --repository-name $PROJECT_NAME/$app >/dev/null 2>&1 || \
  aws ecr create-repository --repository-name $PROJECT_NAME/$app >/dev/null
  TARGET=$ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com.cn/$PROJECT_NAME/$app:$APP_VERSION
  docker build -t $TARGET apps/$app
  docker push $TARGET
done
```	
将 base_app.ymal 文件中image信息进行修改，注意{ACCOUNT_ID}和{AWS_REGION}和{APP_VERSION} 按照demo环境进行替换：

```	
"${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/eks-app-mesh-demo/frontend_node:${APP_VERSION}"
替换为：
487071860556.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks-app-mesh-demo/frontend_node:1.0

"${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/eks-app-mesh-demo/catalog_detail:${APP_VERSION}"
替换为：
487071860556.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks-app-mesh-demo/catalog_detail:1.0

"${ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/eks-app-mesh-demo/product_catalog:${APP_VERSION}"
替换为：
487071860556.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks-app-mesh-demo/product_catalog:1.0
```	

b. 将应用部署到 EKS

```	
envsubst < ./deployment/base_app.yaml | kubectl apply -f -
```	
输出显示：

```	
deployment.apps/frontend-node created
service/frontend-node created
deployment.apps/proddetail created
service/proddetail created
deployment.apps/prodcatalog created
service/prodcatalog created
```	
c. 获取部署细节

```	
kubectl get deployment,pods,svc -n prodcatalog-ns -o wide
```	
输出显示：

```	
NAME                            READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS      IMAGES                                                                                       SELECTOR
deployment.apps/frontend-node   1/1     1            1           2m29s   frontend-node   487071860556.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks-app-mesh-demo/frontend_node:1.0     app=frontend-node
deployment.apps/prodcatalog     1/1     1            1           2m28s   prodcatalog     487071860556.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks-app-mesh-demo/product_catalog:1.0   app=prodcatalog
deployment.apps/proddetail      1/1     1            1           2m28s   proddetail      487071860556.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks-app-mesh-demo/catalog_detail:1.0    app=proddetail

NAME                                READY   STATUS    RESTARTS   AGE     IP                NODE                                                        NOMINATED NODE   READINESS GATES
pod/frontend-node-b565b984b-tqtfg   1/1     Running   0          2m29s   192.168.156.106   ip-192-168-143-99.cn-northwest-1.compute.internal           <none>           <none>
pod/prodcatalog-7b9bc46784-pn6hj    1/1     Running   0          2m28s   192.168.148.80    fargate-ip-192-168-148-80.cn-northwest-1.compute.internal   <none>           <none>
pod/proddetail-7869897696-2qhc5     1/1     Running   0          2m28s   192.168.148.76    ip-192-168-143-99.cn-northwest-1.compute.internal           <none>           <none>

NAME                    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE     SELECTOR
service/frontend-node   ClusterIP   10.100.229.175   <none>        9000/TCP   2m29s   app=frontend-node
service/prodcatalog     ClusterIP   10.100.199.8     <none>        5000/TCP   2m27s   app=prodcatalog
service/proddetail      ClusterIP   10.100.17.48     <none>        3000/TCP   2m28s   app=proddetail
```	

d. 确认 Fargate Pod 使用了指定的 Service Account role

```	
export BE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=prodcatalog -o jsonpath='{.items[].metadata.name}') 

kubectl describe pod ${BE_POD_NAME} -n  prodcatalog-ns | grep 'AWS_ROLE_ARN\|AWS_WEB_IDENTITY_TOKEN_FILE\|serviceaccount' 
```	
输出显示：

```	
AWS_ROLE_ARN: arn:aws-cn:iam::487071860556:role/eksctl-eks0714-addon-iamserviceaccount-prodc-Role1-1PGZAK1HGD3NG
AWS_WEB_IDENTITY_TOKEN_FILE:  /var/run/secrets/eks.amazonaws.com/serviceaccount/token
/var/run/secrets/eks.amazonaws.com/serviceaccount from aws-iam-token (ro)
/var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-mgff9 (ro)
```	 

e. 确认 Fargate Pod Logging 已经启用

```	 
kubectl describe pod ${BE_POD_NAME} -n  prodcatalog-ns | grep LoggingEnabled

Logging: LoggingEnabled
Normal  LoggingEnabled  5m33s  fargate-scheduler  Successfully enabled logging for pod
```	 

### 2.2 测试应用

测试 Fargate 和 Nodegroup Pod之间的连通性

a. 测试 ported Product Catalog App 是否运行正常

```	 
export FE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=frontend-node -o jsonpath='{.items[].metadata.name}') 

kubectl -n prodcatalog-ns exec -it ${FE_POD_NAME} -c frontend-node bash

root@frontend-node-b565b984b-tqtfg:/usr/src/app#
```	 


b. curl Fargate prodcatalog backend endpoint


```	 
curl http://prodcatalog.prodcatalog-ns.svc.cluster.local:5000/products/ 

{
    "products": {},
    "details": {
        "version": "1",
        "vendors": [
            "ABC.com"
        ]
    }
}
```	 

c. 测试 Fargate service prodcatalog 与 Nodegroup service proddetail 的连通性

```	 
export BE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=prodcatalog -o jsonpath='{.items[].metadata.name}') 

kubectl -n prodcatalog-ns exec -it ${BE_POD_NAME} -c prodcatalog bash

root@prodcatalog-7b9bc46784-pn6hj:/app#

curl http://proddetail.prodcatalog-ns.svc.cluster.local:3000/catalogDetail

{"version":"1","vendors":["ABC.com"]}
```	 

# 3. 安装 App Mesh 

## 3.1 安装 AWS App Mesh Controller

a. 安装 App Mesh Helm Chart

```	 
helm repo add eks https://aws.github.io/eks-charts
```	 
输出显示：

```	 
"eks" has been added to your repositories
```	 

b. 安装 App Mesh Controller

b.1 创建 namespace appmesh-system, 启用 OIDC, 创建 IRSA (IAM for Service Account) for AWS App Mesh installation

```	 
# 创建 namespace
kubectl create ns appmesh-system

# 安装 App Mesh CRDs
kubectl apply -k "github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"
(下载到本地目录，crds文件夹下，包含crds.yaml和kustomization.yaml)

kubectl apply -k app-mash-crds/

# 创建 EKS 集群使用的 OIDC identity provider
eksctl utils associate-iam-oidc-provider \
  --cluster eks0714 \
  --approve

# 下载 AWS App Mesh Kubernetes Controller IAM policy
curl -o controller-iam-policy.json https://raw.githubusercontent.com/aws/aws-app-mesh-controller-for-k8s/master/config/iam/controller-iam-policy.json

# 创建 AWSAppMeshK8sControllerIAMPolicy IAM Policy
aws iam create-policy \
    --policy-name AWSAppMeshK8sControllerIAMPolicy \
    --policy-document file://controller-iam-policy.json

# 创建 IAM role for appmesh-controller service account
eksctl create iamserviceaccount --cluster eks0714 \
    --namespace appmesh-system \
    --name appmesh-controller \
    --attach-policy-arn arn:aws-cn:iam::$ACCOUNT_ID:policy/AWSAppMeshK8sControllerIAMPolicy  \
    --override-existing-serviceaccounts \
    --approve
```	 

b.2 helm repo update

```	 
helm repo update
```	 
输出显示：

```	 
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "karpenter" chart repository
...Successfully got an update from the "eks" chart repository
...Successfully got an update from the "projectcalico" chart repository
Update Complete. ⎈Happy Helming!⎈
```	 

b.3 安装 App Mesh Controller

```	 
helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=$AWS_REGION \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller \
    --set tracing.enabled=true \
    --set tracing.provider=x-ray \
    --set sidecar.image.repository=919830735681.dkr.ecr.cn-northwest-1.amazonaws.com.cn/aws-appmesh-envoy:v1.22.2.0-prod \
    --set init.image.repository=919830735681.dkr.ecr.cn-northwest-1.amazonaws.com.cn/aws-appmesh-proxy-route-manager \
    --set image.repository=919830735681.dkr.ecr.cn-northwest-1.amazonaws.com.cn/amazon/appmesh-controller
```	 
输出显示：

```	 
Release "appmesh-controller" does not exist. Installing it now.
NAME: appmesh-controller
LAST DEPLOYED: Sat Jul 16 10:43:48 2022
NAMESPACE: appmesh-system
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
AWS App Mesh controller installed!
```	 

确认 controller version 大于 v1.0.0.

```	 
kubectl get deployment appmesh-controller \
    -n appmesh-system \
    -o json  | jq -r ".spec.template.spec.containers[].image" | cut -f2 -d ':'

1.5.0
```	 

确认所有 App Mesh CRDs 已经创建

```	 
kubectl get crds | grep appmesh
```	 
输出显示：

```	 
gatewayroutes.appmesh.k8s.aws                2022-07-15T08:30:45Z
meshes.appmesh.k8s.aws                       2022-07-15T08:30:45Z
virtualgateways.appmesh.k8s.aws              2022-07-15T08:30:46Z
virtualnodes.appmesh.k8s.aws                 2022-07-15T08:30:46Z
virtualrouters.appmesh.k8s.aws               2022-07-15T08:30:46Z
virtualservices.appmesh.k8s.aws              2022-07-15T08:30:47Z
```	 

查看 appmesh-system Namespace 中的所有资源

```	 
kubectl -n appmesh-system get all  
```	 
输出显示：

```	         
NAME                                      READY   STATUS    RESTARTS   AGE
pod/appmesh-controller-7f87d6d775-z7nwn   1/1     Running   0          2m1s

NAME                                         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/appmesh-controller-webhook-service   ClusterIP   10.100.246.143   <none>        443/TCP   96m

NAME                                 READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/appmesh-controller   1/1     1            1           17m

NAME                                            DESIRED   CURRENT   READY   AGE
replicaset.apps/appmesh-controller-5887db4656   0         0         0       14m
replicaset.apps/appmesh-controller-75b98655ff   0         0         0       17m
replicaset.apps/appmesh-controller-7f87d6d775   1         1         1       2m2s
```	 

## 4. 将产品目录部署到App Mesh
  
### 4.1 创建 Meshed Application

a. 创建 Mesh Object

```	 
cd  eks-app-mesh-polyglot-demo/deployment
kubectl apply -f mesh.yaml  
```	 
输出显示：

```	 
namespace/prodcatalog-ns configured
mesh.appmesh.k8s.aws/prodcatalog-mesh created
```	 
b. 确认 Mesh object 和 Namespace 已经创建

```	 
kubectl describe namespace prodcatalog-ns

Name:         prodcatalog-ns
Labels:       appmesh.k8s.aws/sidecarInjectorWebhook=enabled
              gateway=ingress-gw
              kubernetes.io/metadata.name=prodcatalog-ns
              mesh=prodcatalog-mesh
Annotations:  <none>
Status:       Active


kubectl describe mesh prodcatalog-mesh

Status:
  Conditions:
    Last Transition Time:  2022-07-16T04:47:53Z
    Status:                True
    Type:                  MeshActive
  Mesh ARN:                arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh
  Observed Generation:     1
```	 
c. 创建 Service 使用的 App Mesh 资源

创建 App Mesh 资源

```	 
cd  eks-app-mesh-polyglot-demo/deployment
kubectl apply -f meshed_app.yaml 
```	 
输出显示：

```	 
virtualnode.appmesh.k8s.aws/prodcatalog created
virtualservice.appmesh.k8s.aws/prodcatalog created
virtualservice.appmesh.k8s.aws/proddetail created
virtualrouter.appmesh.k8s.aws/proddetail-router created
virtualrouter.appmesh.k8s.aws/prodcatalog-router created
virtualnode.appmesh.k8s.aws/proddetail-v1 created
virtualnode.appmesh.k8s.aws/frontend-node created
virtualservice.appmesh.k8s.aws/frontend-node created
```
查看 App Mesh 资源

```	 
kubectl get virtualnode,virtualservice,virtualrouter -n prodcatalog-ns
```	 
输出显示：

```	 
NAME                                        ARN                                                                                                             AGE
virtualnode.appmesh.k8s.aws/frontend-node   arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualNode/frontend-node_prodcatalog-ns   51s
virtualnode.appmesh.k8s.aws/prodcatalog     arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualNode/prodcatalog_prodcatalog-ns     53s
virtualnode.appmesh.k8s.aws/proddetail-v1   arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualNode/proddetail-v1_prodcatalog-ns   51s

NAME                                           ARN                                                                                                                                  AGE
virtualservice.appmesh.k8s.aws/frontend-node   arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualService/frontend-node.prodcatalog-ns.svc.cluster.local   51s
virtualservice.appmesh.k8s.aws/prodcatalog     arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualService/prodcatalog.prodcatalog-ns.svc.cluster.local     52s
virtualservice.appmesh.k8s.aws/proddetail      arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualService/proddetail.prodcatalog-ns.svc.cluster.local      52s

NAME                                               ARN                                                                                                                    AGE
virtualrouter.appmesh.k8s.aws/prodcatalog-router   arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualRouter/prodcatalog-router_prodcatalog-ns   52s
virtualrouter.appmesh.k8s.aws/proddetail-router    arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualRouter/proddetail-router_prodcatalog-ns    52s
```	 
### 4.3 Sidecar 注入

a. Sidecar 注入之前

```	 
kubectl get pods -n prodcatalog-ns -o wide
```	 
输出显示

```	 
NAME                            READY   STATUS    RESTARTS   AGE   IP                NODE                                                        NOMINATED NODE   READINESS GATES
frontend-node-b565b984b-tqtfg   1/1     Running   0          25h   192.168.156.106   ip-192-168-143-99.cn-northwest-1.compute.internal           <none>           <none>
prodcatalog-7b9bc46784-pn6hj    1/1     Running   0          25h   192.168.148.80    fargate-ip-192-168-148-80.cn-northwest-1.compute.internal   <none>           <none>
proddetail-7869897696-2qhc5     1/1     Running   0          25h   192.168.148.76    ip-192-168-143-99.cn-northwest-1.compute.internal           <none>           <none>
```	 

重启deployment，自动注入sidecar

```	 
kubectl -n prodcatalog-ns rollout restart deployment prodcatalog
kubectl -n prodcatalog-ns rollout restart deployment proddetail 
kubectl -n prodcatalog-ns rollout restart deployment frontend-node
```	 
b. Sidecar注入之后检查

```	 
kubectl get pods -n prodcatalog-ns -o wide
```	 
输出显示：

```	 
NAME                             READY   STATUS    RESTARTS   AGE    IP                NODE                                                         NOMINATED NODE   READINESS GATES
frontend-node-7676ff5849-z75l5   3/3     Running   0          2m6s   192.168.26.239    ip-192-168-13-85.cn-northwest-1.compute.internal             <none>           <none>
prodcatalog-74f654d9f9-h8b7s     3/3     Running   0          2m8s   192.168.189.210   fargate-ip-192-168-189-210.cn-northwest-1.compute.internal   <none>           <none>
proddetail-bffcc8f65-f6s94       3/3     Running   0          2m7s   192.168.184.200   ip-192-168-185-174.cn-northwest-1.compute.internal           <none>           <none>
```	 

c. 从 Pod 中获取运行的容器

```	 
POD=$(kubectl -n prodcatalog-ns get pods -o jsonpath='{.items[0].metadata.name}')
kubectl -n prodcatalog-ns get pods ${POD} -o jsonpath='{.spec.containers[*].name}'; echo
```	 
输出显示：

```	 
frontend-node envoy xray-daemon
```	 

### 4.4 测试 Application

a. 测试 Product Catalog App 是否按照预期运行

```	 
export FE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=frontend-node -o jsonpath='{.items[].metadata.name}') 

kubectl -n prodcatalog-ns exec -it ${FE_POD_NAME} -c frontend-node bash

root@frontend-node-7676ff5849-z75l5:/usr/src/app#
 
curl -v http://prodcatalog.prodcatalog-ns.svc.cluster.local:5000/products/
```	 
输出显示：

```	
*   Trying 10.100.199.8...
* TCP_NODELAY set
* Expire in 200 ms for 4 (transfer 0x5615a3b840f0)
* Connected to prodcatalog.prodcatalog-ns.svc.cluster.local (10.100.199.8) port 5000 (#0)
> GET /products/ HTTP/1.1
> Host: prodcatalog.prodcatalog-ns.svc.cluster.local:5000
> User-Agent: curl/7.64.0
> Accept: */*
>
< HTTP/1.1 200 OK
< content-type: application/json
< content-length: 124
< x-amzn-trace-id: Root=1-62d256a7-75dd70f3b9587e485635a615
< access-control-allow-origin: *
< server: envoy
< date: Sat, 16 Jul 2022 06:11:51 GMT
< x-envoy-upstream-service-time: 17
<
{
    "products": {},
    "details": {
        "version": "1",
        "vendors": [
            "ABC.com"
        ]
    }
}
* Connection #0 to host prodcatalog.prodcatalog-ns.svc.cluster.local left intact
```	 

b. 测试从 Fargate service prodcatalog 到 Nodegroup service proddetail 的连通性

```	 
export BE_POD_NAME=$(kubectl get pods -n prodcatalog-ns -l app=prodcatalog -o jsonpath='{.items[].metadata.name}') 

kubectl -n prodcatalog-ns exec -it ${BE_POD_NAME} -c prodcatalog bash

root@prodcatalog-74f654d9f9-h8b7s:/app#

curl -v http://proddetail.prodcatalog-ns.svc.cluster.local:3000/catalogDetail
```	 
输出显示：

```	 
*   Trying 10.100.17.48:3000...
* Connected to proddetail.prodcatalog-ns.svc.cluster.local (10.100.17.48) port 3000 (#0)
> GET /catalogDetail HTTP/1.1
> Host: proddetail.prodcatalog-ns.svc.cluster.local:3000
> User-Agent: curl/7.74.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: application/json; charset=utf-8
< content-length: 37
< etag: W/"25-+DP7kANx3olb0HJqt5zDWgaO2Gg"
< date: Sat, 16 Jul 2022 06:14:13 GMT
< x-envoy-upstream-service-time: 2
< server: envoy
<
* Connection #0 to host proddetail.prodcatalog-ns.svc.cluster.local left intact
```	 

# 5. 安装 Virtual Gateway 

## 5.1 添加 App Mesh Virtual Gateway

a. 创建 Virtual Gateway

```	 
cd eks-app-mesh-polyglot-demo/deployment
kubectl apply -f virtual_gateway.yaml 
```	 
输出显示：

```	 
virtualgateway.appmesh.k8s.aws/ingress-gw created
gatewayroute.appmesh.k8s.aws/gateway-route-frontend created
service/ingress-gw created
deployment.apps/ingress-gw created
```	 
b. 查看ingress

```	 
kubectl get all  -n prodcatalog-ns -o wide | grep ingress
```	 
输出显示：

```	 
pod/ingress-gw-8764db7df-9dq2s       2/2     Running   0          2m24s   192.168.9.224     ip-192-168-13-85.cn-northwest-1.compute.internal             <none>           <none>
service/ingress-gw      LoadBalancer   10.100.214.15    a42ab8618c1144cf899c47e5fa9ab723-561c245813cac465.elb.cn-northwest-1.amazonaws.com.cn   80:31375/TCP   2m25s   app=ingress-gw
deployment.apps/ingress-gw      1/1     1            1           2m25s   envoy           ${ENVOY_IMAGE}                                                                               app=ingress-gw
replicaset.apps/ingress-gw-8764db7df       1         1         1       2m25s   envoy           ${ENVOY_IMAGE}                                                                               app=ingress-gw,pod-template-hash=8764db7df
virtualgateway.appmesh.k8s.aws/ingress-gw   arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualGateway/ingress-gw_prodcatalog-ns   2m27s
gatewayroute.appmesh.k8s.aws/gateway-route-frontend   arn:aws-cn:appmesh:cn-northwest-1:487071860556:mesh/prodcatalog-mesh/virtualGateway/ingress-gw_prodcatalog-ns/gatewayRoute/gateway-route-frontend_prodcatalog-ns   2m27s

```	 
注意： 如果elb一直处于pending状态，请检查子网标签是否缺失

https://docs.aws.amazon.com/zh_cn/eks/latest/userguide/alb-ingress.html 
https://aws.amazon.com/cn/premiumsupport/knowledge-center/eks-vpc-subnet-discovery/


## 5.2 测试 Virtual Gateway

### a. 验证elb是否生效，并保留其url

```	 
export LB_NAME=$(kubectl get svc ingress-gw -n prodcatalog-ns -o jsonpath="{.status.loadBalancer.ingress[*].hostname}") 

curl -v --silent ${LB_NAME} | grep x-envoy

echo $LB_NAME
```	 
输出显示：

```	 
*   Trying 69.230.200.127:80...
* Connected to a42ab8618c1144cf899c47e5fa9ab723-561c245813cac465.elb.cn-northwest-1.amazonaws.com.cn (69.230.200.127) port 80 (#0)
> GET / HTTP/1.1
> Host: a42ab8618c1144cf899c47e5fa9ab723-561c245813cac465.elb.cn-northwest-1.amazonaws.com.cn
> User-Agent: curl/7.79.1
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< x-powered-by: Express
< content-type: text/html; charset=utf-8
< content-length: 1195
< etag: W/"4ab-ju0cYuWnpkHio52kIUHS0XrmIdU"
< date: Sat, 16 Jul 2022 07:18:29 GMT
< x-envoy-upstream-service-time: 24
< server: envoy
<
{ [1195 bytes data]
* Connection #0 to host a42ab8618c1144cf899c47e5fa9ab723-561c245813cac465.elb.cn-northwest-1.amazonaws.com.cn left intact

echo $LB_NAME
a42ab8618c1144cf899c47e5fa9ab723-561c245813cac465.elb.cn-northwest-1.amazonaws.com.cn
```	 
b. 对业务进行验证

浏览器打开 

```	 
a42ab8618c1144cf899c47e5fa9ab723-561c245813cac465.elb.cn-northwest-1.amazonaws.com.cn
```	 
输入ID=1的Table并添加，新的产品 Table生成。

Product Catalog version 1 has vendors ABC.com

# 6 观测

请参考链接进行观察测试：

```	 
https://www.eksworkshop.com/advanced/330_servicemesh_using_appmesh/integrate_observability/
```	 

观察内容包含：

- Container Insights
- Cloudwatch Container logs
- Prometheus App Mesh Metrics
- Fargate Container logs
- AWS X-Ray Tracing

# 7 清理环境

```	 
# 删除 Product Catalog apps
kubectl delete namespace prodcatalog-ns

# 删除 ECR images
aws ecr delete-repository --repository-name eks-app-mesh-demo/catalog_detail --force
aws ecr delete-repository --repository-name eks-app-mesh-demo/frontend_node --force
aws ecr delete-repository --repository-name eks-app-mesh-demo/product_catalog --force

# 关闭 Amazon EKS 控制层面日志
eksctl utils update-cluster-logging --disable-types all \
    --region ${AWS_REGION} \
    --cluster eksworkshop-eksctl \
    --approve

# 删除 Cloudwatch namespace
kubectl delete namespace amazon-cloudwatch

# 删除 Observability namespace
kubectl delete namespace aws-observability

# 删除 Product Catalog mesh
kubectl delete meshes prodcatalog-mesh

# 卸载 Helm Charts
helm -n appmesh-system delete appmesh-controller

# 删除 AWS App Mesh CRDs
for i in $(kubectl get crd | grep appmesh | cut -d" " -f1) ; do
kubectl delete crd $i
done

# 删除 AppMesh Controller service account
eksctl delete iamserviceaccount  --cluster eksworkshop-eksctl --namespace appmesh-system --name appmesh-controller

# 删除 App Mesh namespace
kubectl delete namespace appmesh-system

# 删除 Fargate Logging Policy
export PodRole=$(aws eks describe-fargate-profile --cluster-name eksworkshop-eksctl --fargate-profile-name fargate-productcatalog --query 'fargateProfile.podExecutionRoleArn' | sed -n 's/^.*role\/\(.*\)".*$/\1/ p')
aws iam detach-role-policy \
        --policy-arn arn:aws:iam::${ACCOUNT_ID}:policy/FluentBitEKSFargate \
        --role-name ${PodRole}
aws iam delete-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/FluentBitEKSFargate

# 删除 Fargate profile
eksctl delete fargateprofile \
  --name fargate-productcatalog \
  --cluster eksworkshop-eksctl

# 删除 IAM Policy & IRSA
eksctl delete iamserviceaccount --cluster eksworkshop-eksctl   --namespace prodcatalog-ns --name prodcatalog-envoy-proxies
aws iam delete-policy --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/ProdEnvoyNamespaceIAMPolicy
```	 




