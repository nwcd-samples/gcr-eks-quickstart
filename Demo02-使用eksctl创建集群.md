# Demo02-使用eksctl创建集群
--
#### Contributor: Zhengyu Ren
#### 更新时间: 2024-11-04
#### 基于EKS版本: EKS 1.31
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
#--version 这里以1.27版本为例

$ eksctl create cluster --name=${CLUSTER_NAME} --node-type t3.medium --managed --alb-ingress-access --region=${AWS_REGION} --version 1.31
```

参考输出

```
2024-11-04 10:04:01 [ℹ]  eksctl version 0.193.0-dev+19cb88bf5.2024-10-18T12:39:39Z
2024-11-04 10:04:01 [ℹ]  using region cn-northwest-1
2024-11-04 10:04:04 [ℹ]  setting availability zones to [cn-northwest-1a cn-northwest-1b cn-northwest-1c]
2024-11-04 10:04:04 [ℹ]  subnets for cn-northwest-1a - public:192.168.0.0/19 private:192.168.96.0/19
2024-11-04 10:04:04 [ℹ]  subnets for cn-northwest-1b - public:192.168.32.0/19 private:192.168.128.0/19
2024-11-04 10:04:04 [ℹ]  subnets for cn-northwest-1c - public:192.168.64.0/19 private:192.168.160.0/19
2024-11-04 10:04:04 [ℹ]  nodegroup "ng-0ec6130f" will use "" [AmazonLinux2/1.31]
2024-11-04 10:04:04 [ℹ]  using Kubernetes version 1.31
2024-11-04 10:04:04 [ℹ]  creating EKS cluster "eksworkshop" in "cn-northwest-1" region with managed nodes
2024-11-04 10:04:04 [ℹ]  will create 2 separate CloudFormation stacks for cluster itself and the initial managed nodegroup
2024-11-04 10:04:04 [ℹ]  if you encounter any issues, check CloudFormation console or try 'eksctl utils describe-stacks --region=cn-northwest-1 --cluster=eksworkshop'
2024-11-04 10:04:04 [ℹ]  Kubernetes API endpoint access will use default of {publicAccess=true, privateAccess=false} for cluster "eksworkshop" in "cn-northwest-1"
2024-11-04 10:04:04 [ℹ]  CloudWatch logging will not be enabled for cluster "eksworkshop" in "cn-northwest-1"
2024-11-04 10:04:04 [ℹ]  you can enable it with 'eksctl utils update-cluster-logging --enable-types={SPECIFY-YOUR-LOG-TYPES-HERE (e.g. all)} --region=cn-northwest-1 --cluster=eksworkshop'
2024-11-04 10:04:04 [ℹ]  default addons vpc-cni, kube-proxy, coredns were not specified, will install them as EKS addons
2024-11-04 10:04:04 [ℹ]
2 sequential tasks: { create cluster control plane "eksworkshop",
    2 sequential sub-tasks: {
        2 sequential sub-tasks: {
            1 task: { create addons },
            wait for control plane to become ready,
        },
        create managed nodegroup "ng-0ec6130f",
    }
}
2024-11-04 10:04:04 [ℹ]  building cluster stack "eksctl-eksworkshop-cluster"
2024-11-04 10:04:06 [ℹ]  deploying stack "eksctl-eksworkshop-cluster"
2024-11-04 10:04:36 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2024-11-04 10:05:06 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2024-11-04 10:06:07 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2024-11-04 10:07:08 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2024-11-04 10:08:08 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2024-11-04 10:09:10 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2024-11-04 10:10:11 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2024-11-04 10:11:12 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-cluster"
2024-11-04 10:11:16 [ℹ]  creating addon
2024-11-04 10:11:16 [ℹ]  successfully created addon
2024-11-04 10:11:17 [ℹ]  creating addon
2024-11-04 10:11:17 [ℹ]  successfully created addon
2024-11-04 10:11:18 [ℹ]  creating addon
2024-11-04 10:11:18 [ℹ]  successfully created addon
2024-11-04 10:13:23 [ℹ]  building managed nodegroup stack "eksctl-eksworkshop-nodegroup-ng-0ec6130f"
2024-11-04 10:13:24 [ℹ]  deploying stack "eksctl-eksworkshop-nodegroup-ng-0ec6130f"
2024-11-04 10:13:24 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-0ec6130f"
2024-11-04 10:13:55 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-0ec6130f"
2024-11-04 10:14:48 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-0ec6130f"
2024-11-04 10:16:29 [ℹ]  waiting for CloudFormation stack "eksctl-eksworkshop-nodegroup-ng-0ec6130f"
2024-11-04 10:16:29 [ℹ]  waiting for the control plane to become ready
2024-11-04 10:16:30 [✔]  saved kubeconfig as "/Users/zeno/.kube/config"
2024-11-04 10:16:30 [ℹ]  no tasks
2024-11-04 10:16:30 [✔]  all EKS cluster resources for "eksworkshop" have been created
2024-11-04 10:16:30 [✔]  created 0 nodegroup(s) in cluster "eksworkshop"
2024-11-04 10:16:30 [ℹ]  nodegroup "ng-0ec6130f" has 2 node(s)
2024-11-04 10:16:30 [ℹ]  node "ip-192-168-19-153.cn-northwest-1.compute.internal" is ready
2024-11-04 10:16:30 [ℹ]  node "ip-192-168-34-136.cn-northwest-1.compute.internal" is ready
2024-11-04 10:16:30 [ℹ]  waiting for at least 2 node(s) to become ready in "ng-0ec6130f"
2024-11-04 10:16:30 [ℹ]  nodegroup "ng-0ec6130f" has 2 node(s)
2024-11-04 10:16:30 [ℹ]  node "ip-192-168-19-153.cn-northwest-1.compute.internal" is ready
2024-11-04 10:16:30 [ℹ]  node "ip-192-168-34-136.cn-northwest-1.compute.internal" is ready
2024-11-04 10:16:30 [✔]  created 1 managed nodegroup(s) in cluster "eksworkshop"
2024-11-04 10:16:31 [ℹ]  kubectl command should work with "/Users/zeno/.kube/config", try 'kubectl get nodes'
2024-11-04 10:16:31 [✔]  EKS cluster "eksworkshop" in "cn-northwest-1" region is ready
```

集群创建完成后, 查看EKS集群节点

```
$ kubectl get node
```

参考输出

```
$ kubectl get node
NAME                                                STATUS   ROLES    AGE     VERSION
ip-192-168-19-153.cn-northwest-1.compute.internal   Ready    <none>   2m49s   v1.31.0-eks-a737599
ip-192-168-34-136.cn-northwest-1.compute.internal   Ready    <none>   2m45s   v1.31.0-eks-a737599
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
      - image: public.ecr.aws/nginx/nginx:latest
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
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5487657745-h7q4k   1/1     Running   0          20s

$ kubectl get deployment nginx-deployment
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   1/1     1            1           36m

$ kubectl get service service-nginx -o wide
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                                             PORT(S)        AGE   SELECTOR
service-nginx   LoadBalancer   10.100.120.22   xxxx.elb.cn-northwest-1.amazonaws.com.cn   80:32678/TCP   38m   app=nginx

$ ELB=$(kubectl get service service-nginx -o json | jq -r '.status.loadBalancer.ingress[].hostname')

$ curl -m3 -v $ELB
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
```

### 2.3 清理环境
```
$ kubectl delete -f nginx-nlb.yaml 
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
$ NODE_GROUP=$(eksctl get nodegroup --cluster ${CLUSTER_NAME} --region=${AWS_REGION} -o json | jq -r '.[].Name')
$ eksctl scale nodegroup --cluster=${CLUSTER_NAME} --nodes=3 --name=${NODE_GROUP} --nodes-max=3 --region=${AWS_REGION}
```

参考输出

```
$ eksctl scale nodegroup --cluster=${CLUSTER_NAME} --nodes=3 --name=${NODE_GROUP} --nodes-max=3 --region=${AWS_REGION}
2024-11-04 10:59:28 [ℹ]  scaling nodegroup "ng-0ec6130f" in cluster eksworkshop
2024-11-04 10:59:30 [ℹ]  initiated scaling of nodegroup
2024-11-04 10:59:30 [ℹ]  to see the status of the scaling run `eksctl get nodegroup --cluster eksworkshop --region cn-northwest-1 --name ng-0ec6130f`
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
eksworkshop	ng-0ec6130f	ACTIVE	2024-11-04T02:13:46Z	2		3		3			t3.medium	AL2_x86_64	eks-ng-0ec6130f-72c97a79-8bf5-6027-ef56-a69801bea12a	managed

$ kubectl get node
NAME                                                STATUS   ROLES    AGE    VERSION
ip-192-168-19-153.cn-northwest-1.compute.internal   Ready    <none>   48m    v1.31.0-eks-a737599
ip-192-168-34-136.cn-northwest-1.compute.internal   Ready    <none>   48m    v1.31.0-eks-a737599
ip-192-168-87-190.cn-northwest-1.compute.internal   Ready    <none>   3m9s   v1.31.0-eks-a737599
```

## 4. 镜像处置(针对中国区)
由于防火墙或安全限制，海外gcr.io，quay.io的镜像可能无法下载，为了不手动修改原始yaml文件的镜像路径，可以使用 [Kubernetes mutating admission webhook](https://github.com/nwcdlabs/container-mirror/blob/master/webhook/README.md) 项目实现镜像自动映射，直接参照如下命令部署即可

```
$ kubectl apply -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml
```
