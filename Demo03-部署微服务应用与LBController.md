# Demo03-部署微服务应用与Amazon Load Balancer Controller
--
#### Contributor: Zhengyu Ren
--

## 1. 部署微服务应用
### 1.1 获取实验样例
```
#Ruby FrontEnd
git clone https://github.com/brentley/ecsdemo-frontend.git

#NodeJS Backend and crystal backend
git clone https://github.com/brentley/ecsdemo-nodejs.git
git clone https://github.com/brentley/ecsdemo-crystal.git
```

### 1.2 部署后台应用
```
# 部署nodejs应用
kubectl apply -f ecsdemo-nodejs/kubernetes/deployment.yaml
kubectl apply -f ecsdemo-nodejs/kubernetes/service.yaml

# 检查部署是否正确
kubectl get deployment ecsdemo-nodejs

# 部署crystal应用
kubectl apply -f ecsdemo-crystal/kubernetes/deployment.yaml
kubectl apply -f ecsdemo-crystal/kubernetes/service.yaml

# 检查部署是否正确
kubectl get deployment ecsdemo-crystal
```

### 1.3 部署前台应用
```
AWS_REGION='cn-northwest-1'

#检查ELB Service Role以及在您的账号下创建, 如果没有创建, 请参考AWS文档进行创建
aws iam get-role --role-name "AWSServiceRoleForElasticLoadBalancing" --region ${AWS_REGION} || aws iam create-service-linked-role --aws-service-name "elasticloadbalancing.amazonaws.com" --region ${AWS_REGION}

#部署
kubectl apply -f ecsdemo-frontend/kubernetes/deployment.yaml
kubectl apply -f ecsdemo-frontend/kubernetes/service.yaml
kubectl get deployment ecsdemo-frontend

# 检查状态
kubectl get service ecsdemo-frontend -o wide

# 访问前端服务
ELB=$(kubectl get service ecsdemo-frontend -o json | jq -r '.status.loadBalancer.ingress[].hostname')
echo ${ELB}

# ELB创建至Active状态通常需要数分钟时间, 建议等待3-5mins后执行后续操作
# 浏览器访问或者通过curl命令进行验证
curl -m3 -v $ELB
```

### 1.4 微服务部署扩展
通过观察发现集群并不是跨多节点的高可用的架构, 因此计划对部署进行扩展.

```
# 每一个微服务目前都只有一个部署单元
kubectl get deployments
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# ecsdemo-crystal    1/1     1            1           19m
# ecsdemo-frontend   1/1     1            1           7m51s
# ecsdemo-nodejs     1/1     1            1           24m

# scale 到3个replicas
kubectl scale deployment ecsdemo-nodejs --replicas=3
kubectl scale deployment ecsdemo-crystal --replicas=3
kubectl scale deployment ecsdemo-frontend --replicas=3

kubectl get deployments
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# ecsdemo-crystal    3/3     3            3           21m
# ecsdemo-frontend   3/3     3            3           9m51s
# ecsdemo-nodejs     3/3     3            3           26m
```

### 1.5 清除资源
```
kubectl delete -f ecsdemo-frontend/kubernetes/service.yaml
kubectl delete -f ecsdemo-frontend/kubernetes/deployment.yaml

kubectl delete -f ecsdemo-crystal/kubernetes/service.yaml
kubectl delete -f ecsdemo-crystal/kubernetes/deployment.yaml

kubectl delete -f ecsdemo-nodejs/kubernetes/service.yaml
kubectl delete -f ecsdemo-nodejs/kubernetes/deployment.yaml
```

## 2. 使用Amazon Load Balancer Controller
参考文档:
<br>[https://docs.amazonaws.cn/eks/latest/userguide/aws-load-balancer-controller.html](https://docs.amazonaws.cn/eks/latest/userguide/aws-load-balancer-controller.html)
<br>[https://aws.amazon.com/cn/blogs/china/releasing-k8s-service-on-aws-eks-platform-part-1/](https://aws.amazon.com/cn/blogs/china/releasing-k8s-service-on-aws-eks-platform-part-1/)

### 2.1 创建Amazon Load Balancer Controller所需要的IAM policy , EKS OIDC provider, service account

#### 2.1.1 创建EKS OIDC Provider (这个操作每个集群只需要做一次）
```
AWS_REGION='cn-northwest-1'
CLUSTER_NAME='eksworkshop'

eksctl utils associate-iam-oidc-provider --cluster=${CLUSTER_NAME} --approve --region ${AWS_REGION}

2022-07-11 18:00:37 [ℹ]  will create IAM Open ID Connect provider for cluster "eksworkshop" in "cn-northwest-1"
2022-07-11 18:00:37 [✔]  created IAM Open ID Connect provider for cluster "eksworkshop" in "cn-northwest-1"
```

#### 2.1.2 创建所需要的IAM policy
```
#下载针对中国区的策略
curl -o iam_policy_cn.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.4.0/docs/install/iam_policy_cn.json

#创建IAM策略
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_cn.json --region ${AWS_REGION}

#获取Policy ARN
POLICY_NAME=$(aws iam list-policies | egrep 'AWSLoadBalancerControllerIAMPolicy'|grep 'Arn'|awk '{print $2}'|awk -F ',' '{print$1}')
```

#### 2.1.3 使用上述创建的IAM Policy ARN来创建Service Account
```
eksctl create iamserviceaccount \
  --cluster=${CLUSTER_NAME} \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=${POLICY_NAME} \
  --override-existing-serviceaccounts \
  --approve

2022-07-11 18:45:44 [ℹ]  1 existing iamserviceaccount(s) (kube-system/alb-ingress-controller) will be excluded
2022-07-11 18:45:44 [ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included (based on the include/exclude rules)
2022-07-11 18:45:44 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set
2022-07-11 18:45:44 [ℹ]  1 task: {
    2 sequential sub-tasks: {
        create IAM role for serviceaccount "kube-system/aws-load-balancer-controller",
        create serviceaccount "kube-system/aws-load-balancer-controller",
    } }2022-07-11 18:45:44 [ℹ]  building iamserviceaccount stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2022-07-11 18:45:44 [ℹ]  deploying stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2022-07-11 18:45:44 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2022-07-11 18:46:15 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-addon-iamserviceaccount-kube-system-aws-load-balancer-controller"
2022-07-11 18:46:15 [ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"
```

### 2.2 部署Amazon Load Balancer Controller
使用 Helm V3 或更高版本或通过应用 Kubernetes 清单来安装 Amazon Load Balancer Controller.

#### 2.2.1 Helm部署
1.添加 eks-charts Repo

```
helm repo add eks https://aws.github.io/eks-charts
```

2.更新您的Repo, 以确保正确的部署进行后续步骤

```
helm repo update
```

3.利用Helm进行部署

Helm安装参考文档: [https://docs.amazonaws.cn/eks/latest/userguide/helm.html](https://docs.amazonaws.cn/eks/latest/userguide/helm.html)

```
#北京区Controller Image地址
918309763551.dkr.ecr.cn-north-1.amazonaws.com.cn/amazon/aws-load-balancer-controller:2.4.0

#宁夏区Controller Image地址
961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn/amazon/aws-load-balancer-controller:2.4.0

AWS_REGION='cn-northwest-1'
CLUSTER_NAME='eksworkshop'

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=${CLUSTER_NAME} \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set enableShield=false \
  --set enableWaf=false \
  --set enableWafv2=false \
  --set region=${AWS_REGION} \
  --set vpcId=<Your-VPC-ID> \
  --set image.repository=961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn/amazon/aws-load-balancer-controller

#NAME: aws-load-balancer-controller
#LAST DEPLOYED: Mon Jul 11 18:31:27 2022
#NAMESPACE: kube-system
#STATUS: deployed
#REVISION: 1
#TEST SUITE: None
#NOTES:
#AWS Load Balancer controller installed!

#验证控制器是否已安装
kubectl get deployment -n kube-system aws-load-balancer-controller

#NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
#aws-load-balancer-controller   2/2     2            2           20m
```

### 2.3 使用Amazon Load Balancer Controller

#### 2.3.1 nginx-1.yaml

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-1
  labels:
    app: nginx-1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-1
  template:
    metadata:
      labels:
        app: nginx-1
    spec:
      containers:
      - name: nginx-1
        image: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: "nginx-1"
spec:
  selector:
    app: nginx-1
  type: ClusterIP
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
 annotations:
   alb.ingress.kubernetes.io/scheme: internet-facing
   alb.ingress.kubernetes.io/target-type: ip
   kubernetes.io/ingress.class: alb
 creationTimestamp: null
 labels:
   app: nginx
 name: "nginx-1"
 namespace: default
spec:
 rules:
 - http:
     paths:
     - backend:
         service:
           name: "nginx-1"
           port:
             number: 80
       path: /
       pathType: Prefix
```


#### 2.3.2 为nginx service创建ingress

```
kubectl apply -f nginx-1.yaml
```
验证

```
ALB=$(kubectl get ingress -o json | jq -r '.items[0].status.loadBalancer.ingress[].hostname')

curl -m3 -v $ALB

#如果遇到问题，请查看日志
kubectl logs -n kube-system $(kubectl get po -n kube-system | egrep -o alb-ingress[a-zA-Z0-9-]+)
```

#### 2.3.3 清理环境

```
kubectl delete -f nginx-1.yaml
```
