# Demo13-使用App Mesh服务网格

--
#### Contributor: Tao Dai
#### 更新时间: 2023-10-25
#### 基于EKS版本: EKS 1.27
--

在本次Demo中，我们将使用以下产品目录应用程序部署示例向您介绍以下流行的 App Mesh 用例：

- 部署App Mesh控制器
- 部署一个简单的Nginx应用服务
- 配置一个App Mesh虚拟路由器，将流量路由到应用服务

## 1. 安装App Mesh控制器

### 1.1 前置条件

a.设置环境变量

```	
export CLUSTER_NAME=test
export AWS_REGION=cn-northwest-1
sudo yum install jq -y
```	
b.创建IAM OIDC provider

注意： region和cluser需要依据实际情况进行修改

```	
eksctl utils associate-iam-oidc-provider \
    --region cn-northwest-1 \
    --cluster test \
    --approve
```	

### 1.2 安装App Mesh Controller

a.安装前检查

```	
curl -o pre_upgrade_check.sh https://raw.githubusercontent.com/aws/eks-charts/master/stable/appmesh-controller/upgrade/pre_upgrade_check.sh
sh ./pre_upgrade_check.sh

```	
输出显示：

```	
kubectl installation check: PASSED!
jq installation check: PASSED!
kubectl context check: PASSED!
App Mesh CRD check: PASSED!
Controller version check: PASSED!
Injector check for namespace appmesh-inject: PASSED!
Injector check for namespace appmesh-system: PASSED!

Your cluster is ready for upgrade. Please proceed to the installation instructions
```	
b.安装App Mesh Controller：

```	
helm repo add eks https://aws.github.io/eks-charts
kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"
kubectl create ns appmesh-system

eksctl create iamserviceaccount \
    --cluster $CLUSTER_NAME \
    --namespace appmesh-system \
    --name appmesh-controller \
    --attach-policy-arn  arn:aws-cn:iam::aws:policy/AWSCloudMapFullAccess,arn:aws-cn:iam::aws:policy/AWSAppMeshFullAccess \
    --override-existing-serviceaccounts \
    --approve \

helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=$AWS_REGION \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller \
    --set sidecar.image.repository=public.ecr.aws/appmesh/aws-appmesh-envoy \
    --set init.image.repository=919830735681.dkr.ecr.cn-northwest-1.amazonaws.com.cn/aws-appmesh-proxy-route-manager \
     --set image.repository=public.ecr.aws/appmesh/appmesh-controller
```	

c.确认App Mesh Controller版本,需要大于1.4.0：

```	
kubectl get deployment appmesh-controller \
    -n appmesh-system \
    -o json  | jq -r ".spec.template.spec.containers[].image" | cut -f2 -d ':'
```	

## 2 创建App Mesh资源

```
# 创建Namespace	
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: my-apps
  labels:
    mesh: my-mesh
    appmesh.k8s.aws/sidecarInjectorWebhook: enabled
EOF

# 创建Mesh
cat <<EOF | kubectl apply -f -
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: my-mesh
spec:
  namespaceSelector:
    matchLabels:
      mesh: my-mesh
EOF

# 创建VirtualNode
cat <<EOF | kubectl apply -f -
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualNode
metadata:
  name: my-service-a
  namespace: my-apps
spec:
  podSelector:
    matchLabels:
      app: my-app-1
  listeners:
    - portMapping:
        port: 80
        protocol: http
  serviceDiscovery:
    dns:
      hostname: my-service-a.my-apps.svc.cluster.local
EOF

# 创建VirtualRouter
cat <<EOF | kubectl apply -f -
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualRouter
metadata:
  namespace: my-apps
  name: my-service-a-virtual-router
spec:
  listeners:
    - portMapping:
        port: 80
        protocol: http
  routes:
    - name: my-service-a-route
      httpRoute:
        match:
          prefix: /
        action:
          weightedTargets:
            - virtualNodeRef:
                name: my-service-a
              weight: 1
EOF

# 创建VirtualService
cat <<EOF | kubectl apply -f -
apiVersion: appmesh.k8s.aws/v1beta2
kind: VirtualService
metadata:
  name: my-service-a
  namespace: my-apps
spec:
  awsName: my-service-a.my-apps.svc.cluster.local
  provider:
    virtualRouter:
      virtualRouterRef:
        name: my-service-a-virtual-router
EOF
```	

## 3 创建应用服务

### 3.1 创建Service Account

```	
cat <<EOF >> proxy-auth.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "appmesh:StreamAggregatedResources",
            "Resource": [
                "arn:aws-cn:appmesh:cn-northwest-1:111122223333:mesh/my-mesh/virtualNode/my-service-a_my-apps"
            ]
        }
    ]
}
EOF

aws iam create-policy --policy-name AppMeshProxyPolicy --policy-document file://proxy-auth.json

eksctl create iamserviceaccount \
    --cluster $CLUSTER_NAME \
    --namespace my-apps \
    --name my-service-a \
    --attach-policy-arn arn:aws-cn:iam::111122223333:policy/AppMeshProxyPolicy \
    --override-existing-serviceaccounts \
    --approve
```	

### 3.1 创建Nginx应用服务

```	
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: my-service-a
  namespace: my-apps
  labels:
    app: my-app-1
spec:
  selector:
    app: my-app-1
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-service-a
  namespace: my-apps
  labels:
    app: my-app-1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app-1
  template:
    metadata:
      labels:
        app: my-app-1
    spec:
      serviceAccountName: my-service-a
      containers:
      - name: nginx
        image: nginx:1.19.0
        ports:
        - containerPort: 80
EOF
```	
查看应用Pod，确认是否启用App Mesh：

```	
kubectl -n my-apps describe pod my-service-a-558dbd84f-sqdjh
```	
输出显示

```
Name:             my-service-a-558dbd84f-sqdjh
Namespace:        my-apps
Priority:         0
Service Account:  my-service-a
Node:             ip-192-168-97-129.cn-northwest-1.compute.internal/192.168.97.129
Start Time:       Wed, 25 Oct 2023 09:32:55 +0000
Labels:           app=my-app-1
                  pod-template-hash=558dbd84f
Annotations:      <none>
Status:           Running
IP:               192.168.112.79
IPs:
  IP:           192.168.112.79
Controlled By:  ReplicaSet/my-service-a-558dbd84f
Init Containers:
  proxyinit:
    Container ID:   containerd://a7b7fbe36dc4a4625f4fbd9a691c963ffc392625fcdcc6469450d9a4ded81acd
    Image:          919830735681.dkr.ecr.cn-northwest-1.amazonaws.com.cn/aws-appmesh-proxy-route-manager:v7-prod
    Image ID:       919830735681.dkr.ecr.cn-northwest-1.amazonaws.com.cn/aws-appmesh-proxy-route-manager@sha256:36d444b57dc253ea9e76cd6a6c5525416c4f9012c5d3a836ea2da91ab2bb28ff
    Port:           <none>
    Host Port:      <none>
    State:          Terminated
      Reason:       Completed
      Exit Code:    0
      Started:      Wed, 25 Oct 2023 09:32:58 +0000
      Finished:     Wed, 25 Oct 2023 09:32:58 +0000
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:     10m
      memory:  32Mi
    Environment:
      APPMESH_START_ENABLED:         1
      APPMESH_IGNORE_UID:            1337
      APPMESH_ENVOY_INGRESS_PORT:    15000
      APPMESH_ENVOY_EGRESS_PORT:     15001
      APPMESH_APP_PORTS:             80
      APPMESH_EGRESS_IGNORED_IP:     169.254.169.254
      APPMESH_EGRESS_IGNORED_PORTS:  22
      APPMESH_ENABLE_IPV6:           1
      AWS_STS_REGIONAL_ENDPOINTS:    regional
      AWS_DEFAULT_REGION:            cn-northwest-1
      AWS_REGION:                    cn-northwest-1
      AWS_ROLE_ARN:                  arn:aws-cn:iam::111122223333:role/eksctl-test-addon-iamserviceaccount-my-apps-m-Role1-IROXtlxy098D
      AWS_WEB_IDENTITY_TOKEN_FILE:   /var/run/secrets/eks.amazonaws.com/serviceaccount/token
    Mounts:
      /var/run/secrets/eks.amazonaws.com/serviceaccount from aws-iam-token (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-srv9n (ro)
Containers:
  nginx:
    Container ID:   containerd://e2efa6bf0b62b20f2fcf7fae06e49503f668365e21426f90d406772b702243a3
    Image:          nginx:1.19.0
    Image ID:       docker.io/library/nginx@sha256:21f32f6c08406306d822a0e6e8b7dc81f53f336570e852e25fbe1e3e3d0d0133
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 25 Oct 2023 09:33:10 +0000
    Ready:          True
    Restart Count:  0
    Environment:
      AWS_STS_REGIONAL_ENDPOINTS:   regional
      AWS_DEFAULT_REGION:           cn-northwest-1
      AWS_REGION:                   cn-northwest-1
      AWS_ROLE_ARN:                 arn:aws-cn:iam::111122223333:role/eksctl-test-addon-iamserviceaccount-my-apps-m-Role1-IROXtlxy098D
      AWS_WEB_IDENTITY_TOKEN_FILE:  /var/run/secrets/eks.amazonaws.com/serviceaccount/token
    Mounts:
      /var/run/secrets/eks.amazonaws.com/serviceaccount from aws-iam-token (ro)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-srv9n (ro)
  envoy:
    Container ID:   containerd://7ac3f33e862615f15e3828f4f5dea7966b2e84bee90700b0cd796e7bfb73558e
    Image:          public.ecr.aws/appmesh/aws-appmesh-envoy:v1.27.0.0-prod
    Image ID:       public.ecr.aws/appmesh/aws-appmesh-envoy@sha256:e962021573bec151ba648cd20d9fa898abda97ece28074749c1c83d078ccbbe4
    Port:           9901/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 25 Oct 2023 09:33:49 +0000
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:      10m
      memory:   32Mi
    Readiness:  exec [sh -c curl -s http://localhost:9901/server_info | grep state | grep -q LIVE] delay=1s timeout=1s period=10s #success=1 #failure=3
    Environment:
      APPMESH_FIPS_ENDPOINT:                         0
      APPMESH_PREVIEW:                               0
      ENVOY_LOG_LEVEL:                               info
      ENVOY_ADMIN_ACCESS_LOG_FILE:                   /tmp/envoy_admin_access.log
      APPMESH_PLATFORM_K8S_VERSION:                  v1.28.2-eks-f8587cb
      APPMESH_PLATFORM_APP_MESH_CONTROLLER_VERSION:  v1.12.3-dirty
      APPMESH_VIRTUAL_NODE_NAME:                     mesh/my-mesh/virtualNode/my-service-a_my-apps

Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  109s  default-scheduler  Successfully assigned my-apps/my-service-a-558dbd84f-sqdjh to ip-192-168-97-129.cn-northwest-1.compute.internal
  Normal  Pulling    108s  kubelet            Pulling image "919830735681.dkr.ecr.cn-northwest-1.amazonaws.com.cn/aws-appmesh-proxy-route-manager:v7-prod"
  Normal  Pulled     107s  kubelet            Successfully pulled image "919830735681.dkr.ecr.cn-northwest-1.amazonaws.com.cn/aws-appmesh-proxy-route-manager:v7-prod" in 1.362s (1.362s including waiting)
  Normal  Created    107s  kubelet            Created container proxyinit
  Normal  Started    106s  kubelet            Started container proxyinit
  Normal  Pulling    105s  kubelet            Pulling image "nginx:1.19.0"
  Normal  Pulled     95s   kubelet            Successfully pulled image "nginx:1.19.0" in 10.621s (10.621s including waiting)
  Normal  Created    95s   kubelet            Created container nginx
  Normal  Started    94s   kubelet            Started container nginx
  Normal  Pulling    94s   kubelet            Pulling image "public.ecr.aws/appmesh/aws-appmesh-envoy:v1.27.0.0-prod"
  Normal  Pulled     55s   kubelet            Successfully pulled image "public.ecr.aws/appmesh/aws-appmesh-envoy:v1.27.0.0-prod" in 39.054s (39.054s including waiting)
  Normal  Created    55s   kubelet            Created container envoy
  Normal  Started    55s   kubelet            Started container envoy
```	
## 4 清理环境

```
# 删除应用服务
kubectl delete namespace my-apps

# 删除Mesh
kubectl delete mesh my-mesh

# 卸载App Mesh Controller
helm delete appmesh-controller -n appmesh-system
```
