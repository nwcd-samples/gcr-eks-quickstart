# Demo02-使用eksctl创建集群
--
#### Contributor: Zhengyu Ren
--
## 1. 使用eksctl创建EKS集群
该操作需要10~15分钟, 该操作将会创建一个使用t3. medium的纳管节点组.

### 1.1 环境变量
```
#环境变量
#CLUSTER_NAME 集群名称
#AWS_REGION cn-northwest-1：宁夏区； cn-north-1：北京区

AWS_REGION=cn-northwest-1
AWS_DEFAULT_REGION=cn-northwest-1
CLUSTER_NAME=eksworkshop
```

### 1.2 集群创建
```
#参数说明
#--node-type 工作节点类型 默认为m5.large
#--nodes 工作节点数量 默认为2

eksctl create cluster --name=${CLUSTER_NAME} --node-type t3.medium --managed --alb-ingress-access --region=${AWS_REGION}
```

参考输出

```
2022-07-10 20:12:28 [ℹ]  eksctl version 0.105.0-dev+aa76f1d4.2022-07-08T14:38:11Z
2022-07-10 20:12:28 [ℹ]  using region cn-northwest-1
2022-07-10 20:12:28 [ℹ]  setting availability zones to [cn-northwest-1c cn-northwest-1b cn-northwest-1a]
2022-07-10 20:12:28 [ℹ]  subnets for cn-northwest-1c - public:192.168.0.0/19 private:192.168.96.0/19
2022-07-10 20:12:28 [ℹ]  subnets for cn-northwest-1b - public:192.168.32.0/19 private:192.168.128.0/19
2022-07-10 20:12:28 [ℹ]  subnets for cn-northwest-1a - public:192.168.64.0/19 private:192.168.160.0/19
2022-07-10 20:12:28 [ℹ]  nodegroup "ng-2e67ae8c" will use "" [AmazonLinux2/1.22]
2022-07-10 20:12:28 [ℹ]  using Kubernetes version 1.22
2022-07-10 20:12:28 [ℹ]  creating EKS cluster "eksworkshop" in "cn-northwest-1" region with managed nodes
2022-07-10 20:12:28 [ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
2022-07-10 20:12:28 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=cn-northwest-1 --cluster=eksworkshop'
2022-07-10 20:12:28 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "cn-northwest-1"
2022-07-10 20:12:28 [ℹ]  CloudWatch logging will not be enabled for cluster "eksworkshop" in "cn-northwest-1"
2022-07-10 20:12:28 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=cn-northwest-1 --cluster=eksworkshop'
2022-07-10 20:12:28 [ℹ]
2 sequential tasks: { create cluster control plane "eksworkshop",
    2 sequential sub-tasks: {
        wait for control plane to become ready,
        create managed nodegroup "ng-2e67ae8c",
    }
}
2022-07-10 20:12:28 [ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
2022-07-10 20:12:30 [ℹ]  deploying stack "eksctl-eksworkshop-cluster"
2022-07-10 20:13:00 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2022-07-10 20:23:37 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-ng-2e67ae8c"
2022-07-10 20:23:37 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-2e67ae8c"
2022-07-10 20:23:37 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-2e67ae8c"
2022-07-10 20:24:08 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-2e67ae8c"
2022-07-10 20:24:47 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-2e67ae8c"
2022-07-10 20:26:00 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-2e67ae8c"
2022-07-10 20:26:55 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-2e67ae8c"
2022-07-10 20:26:55 [ℹ]  waiting for the control plane availability...
2022-07-10 20:26:56 [✔]  saved kubeconfig as "/Users/zeno/.kube/config"
2022-07-10 20:26:56 [ℹ]  no tasks
2022-07-10 20:26:56 [✔]  all EKS cluster resources for "eksworkshop" have been created
2022-07-10 20:26:56 [ℹ]  nodegroup "ng-2e67ae8c" has 2 node(s)
2022-07-10 20:26:56 [ℹ]  node "ip-192-168-25-85.cn-northwest-1.compute.internal" is ready
2022-07-10 20:26:56 [ℹ]  node "ip-192-168-51-110.cn-northwest-1.compute.internal" is ready
2022-07-10 20:26:56 [ℹ]  waiting for at least 2 node(s) to become ready in "ng-2e67ae8c"
2022-07-10 20:26:56 [ℹ]  nodegroup "ng-2e67ae8c" has 2 node(s)
2022-07-10 20:26:56 [ℹ]  node "ip-192-168-25-85.cn-northwest-1.compute.internal" is ready
2022-07-10 20:26:56 [ℹ]  node "ip-192-168-51-110.cn-northwest-1.compute.internal" is ready
2022-07-10 20:26:57 [ℹ]  kubectl command should work with "/Users/zeno/.kube/config", try 'kubectl get nodes'
2022-07-10 20:26:57 [✔]  EKS cluster "eksworkshop" in "cn-northwest-1" region is ready
```

集群创建完成后, 查看EKS集群节点

```
kubectl get node
```

参考输出

```
NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-25-85.cn-northwest-1.compute.internal    Ready    <none>   87m   v1.22.9-eks-810597c
ip-192-168-51-110.cn-northwest-1.compute.internal   Ready    <none>   87m   v1.22.9-eks-810597c
```

## 2. 部署一个Nginx测试EKS集群基本功能
参考如下nginx-nlb.yaml创建一个nginx pod, 并通过Loadbalancer类型对外暴露.
**特别提示80/443 在AWS China Region需要完成备案流程，请联系你的商务经理确保已开通，或者自行更改nginx-nlb.yaml的端口**

### 2.1 nginx-nlb.yaml 参考文件
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80

---
apiVersion: v1
kind: Service
metadata:
  name: "service-nginx"
  annotations:
        service.beta.kubernetes.io/aws-load-balancer-type: nlb
spec:
  selector:
    app: nginx
  type: LoadBalancer
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
```

### 2.2 部署nginx-nlb Pod
```
kubectl apply -f nginx-nlb.yaml 

## Check deployment status
kubectl get pods
kubectl get deployment nginx-deployment 

## Get the external access 确保 EXTERNAL-IP是一个有效的AWS Network Load Balancer的地址
kubectl get service service-nginx -o wide 

## 下面可以试着访问这个地址
ELB=$(kubectl get service service-nginx -o json | jq -r '.status.loadBalancer.ingress[].hostname')
curl -m3 -v $ELB
```

参考输出

```bash
$ kubectl apply -f nginx-nlb.yaml
deployment.apps/nginx-deployment created
service/service-nginx created

$ kubectl get pods
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-7848d4b86f-xlkg4   0/1     ContainerCreating   0          14s
(base) ~/Downloads$ kubectl get pods
NAME                                READY   STATUS              RESTARTS   AGE
nginx-deployment-7848d4b86f-xlkg4   0/1     ContainerCreating   0          15s
(base) ~/Downloads$ kubectl get pods
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-7848d4b86f-xlkg4   1/1     Running   0          17s

$ kubectl get deployment nginx-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           30s

$ kubectl get service service-nginx -o wide
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                             PORT(S)        AGE   SELECTOR
service-nginx   LoadBalancer   10.100.243.82   a4705ea1dbea943e6863ff87a396b7cb-89a0f084e135db56.elb.cn-northwest-1.amazonaws.com.cn   80:30104/TCP   35s   app=nginx
$ ELB=$(kubectl get service service-nginx -o json | jq -r '.status.loadBalancer.ingress[].hostname')

$ curl -m3 -v $ELB
*   Trying 52.83.120.194:80...
* Connected to 52.83.120.194 (52.83.120.194) port 80 (#0)
> GET / HTTP/1.1
> Host: 52.83.120.194
> User-Agent: curl/7.82.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Server: nginx/1.23.0
< Date: Sun, 10 Jul 2022 14:02:42 GMT
< Content-Type: text/html
< Content-Length: 615
< Last-Modified: Tue, 21 Jun 2022 14:25:37 GMT
< Connection: keep-alive
< ETag: "62b1d4e1-267"
< Accept-Ranges: bytes
<
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
* Connection #0 to host 52.83.120.194 left intact
```

### 2.3 清理环境
```
kubectl delete -f nginx-nlb.yaml 
```

参考输出

```
$ kubectl delete -f nginx-nlb.yaml
deployment.apps "nginx-deployment" deleted
service "service-nginx" deleted
```

## 3. 扩展集群节点
前面步骤通过eksctl创建了一个2节点的集群，下面扩展集群节点到3.

```
#获取节点组信息
NODE_GROUP=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].Name')
eksctl scale nodegroup --cluster=${CLUSTER_NAME} --nodes=3 --name=${NODE_GROUP} --nodes-max=3 --region=${AWS_REGION}
```

参考输出

```
$ eksctl scale nodegroup --cluster=${CLUSTER_NAME} --nodes=3 --name=${NODE_GROUP} --nodes-max=3 --region=${AWS_REGION}
2022-07-10 22:07:41 [ℹ]  scaling nodegroup "ng-2e67ae8c" in cluster eksworkshop
2022-07-10 22:07:42 [ℹ]  waiting for scaling of nodegroup "ng-2e67ae8c" to complete
2022-07-10 22:08:13 [ℹ]  nodegroup successfully scaled
```

检查结果

```
eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION}

kubectl get node
```

参考输出

```
$ eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION}
CLUSTER		NODEGROUP	STATUS	CREATED			MIN SIZE	MAX SIZE	DESIRED CAPACITY	INSTANCE TYPE	IMAGE ID	ASG NAME						TYPE
eksworkshop	ng-2e67ae8c	ACTIVE	2022-07-10T12:23:57Z	2		3		3			t3.medium	AL2_x86_64	eks-ng-2e67ae8c-6ec0f408-7d9a-e4fd-8fa5-4773be1dfb4b	managed

$ kubectl get node
NAME                                                STATUS     ROLES    AGE    VERSION
ip-192-168-25-85.cn-northwest-1.compute.internal    Ready      <none>   103m   v1.22.9-eks-810597c
ip-192-168-51-110.cn-northwest-1.compute.internal   Ready      <none>   103m   v1.22.9-eks-810597c
ip-192-168-66-163.cn-northwest-1.compute.internal   NotReady   <none>   4s     v1.22.9-eks-810597c
```

## 4. 镜像处置(针对中国区)
由于防火墙或安全限制，海外gcr.io, quay.io的镜像可能无法下载，为了不手动修改原始yaml文件的镜像路径，可以使用 [amazon-api-gateway-mutating-webhook-for-k8](https://github.com/aws-samples/amazon-api-gateway-mutating-webhook-for-k8) 项目实现镜像自动映射, 本workshop所需要的镜像已经由nwcdlabs/container-mirror准备好了，直接部署MutatingWebhookConfiguration即可。

**注意：China region EKS service 有2个官方的账号id**
cn-northwest-1   961992271922
cn-north-1           91830976355
因此如果你需要下载EKS 官方的镜像，需要正确使用上面的两个id

```
#例如宁夏区
#aws-efs-csi-driver
961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks/aws-efs-csi-driver

#aws-ebs-csi-driver
961992271922.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eks/aws-ebs-csi-driver
```
