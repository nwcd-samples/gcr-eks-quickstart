# Demo14-使用Istio服务网格
--
#### Contirbutor: Yi Zheng
--

## 1. 先决条件  
1.1 准备实验环境：参考 Demo 01  
1.2 使用eksctl创建集群：参考 Demo 02，不要执行 4. 镜像处置(针对中国区)
1.3 设置环境变量
```
AWS_REGION=cn-northwest-1
AWS_DEFAULT_REGION=cn-northwest-1
CLUSTER_NAME=eksworkshop
```

## 2. 服务网格介绍
服务网格用来描述组成应用程序的微服务网络以及它们之间的交互。随着服务网格的规模和复杂性不断的增长，它将会变得越来越难以理解和管理。它的需求包括服务发现、负载均衡、故障恢复、度量和监控等。服务网格通常还有更复杂的运维需求，比如 A/B 测试、金丝雀发布、速率限制、访问控制和端到端认证。
Istio 是一个完全开源的服务网格，作为透明的一层接入到现有的分布式应用程序里。它也是一个平台，拥有可以集成任何日志、遥测和策略系统的 API 接口。 Istio 允许您连接、保护、控制和观察服务。
在本节中，我们将学习使用 Istio 来构建服务网格，控制服务之间的流量和 API 调用过程。
[官方文档](https://istio.io/)

## 3. 实验目的
使用 Istio 在 Kubernetes 集群中实施服务网格，实现服务之间的流量管理

## 4. 部署 Istio
### 4.1 下载 istioctl

```bash
# 下载 istioctl，本 Workshop 使用 1.12.9
# https://github.com/istio/istio/releases/
mkdir istio && cd istio
echo 'export ISTIO_VERSION="1.12.9"' >> ~/.bash_profile
source ~/.bash_profile

# 下载并安装 istioctl
curl -L https://istio.io/downloadIstio | sh -
sudo cp -v istio-1.12.9/bin/istioctl /usr/local/bin/

# 验证
istioctl version --remote=false
```

### 4.2 部署 istio

```
# 删除webhook
kubectl delete -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml

# 部署 istio
# 使用 --set profile=demo 部署所有模块
istioctl manifest apply --set profile=demo

# 验证，等待 istio pod 全部处于 READY 状态
kubectl -n istio-system get svc
kubectl -n istio-system get pods
```

## 5. 部署 Bookinfo 示例应用

```
# 启用 Istio sidecar 自动注入
kubectl label namespace default istio-injection=enabled

# 删除 image mirror webhook 并重新部署，以便自动映射 sidecar image 为中国区镜像
kubectl apply -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml

# 部署 bookinfo 示例应用
cd ~/istio/istio-1.12.9/
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml

# 等待所有 pod 正常运行
kubectl get services
kubectl get pods

# 验证 Bookinfo 应用
kubectl exec "$(kubectl get pod -l app=ratings -o jsonpath='{.items[0].metadata.name}')" -c ratings -- curl -sS productpage:9080/productpage | grep -o "<title>.*</title>"

# 为 bookinfo 部署 gateway
kubectl apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

## 6. 验证访问 bookinfo 页面

```
# 使用如下命令得到 nodeport 值
export INGRESS_PORT=$(kubectl -n istio-system get service istio-ingressgateway -o jsonpath='{.spec.ports[?(@.name=="http2")].nodePort}')

# 配置 nlb：此处省略
# 需要注意的点：1，选择的子网要和工作节点所在子网一致 2，目标组中的端口使用$INGRESS_PORT

# 访问 bookinfo 页面
http://{nlb address}/productpage
```

## 7. 配置流量管理策略
### 7.1 为 bookinfo 中的所有服务创建默认 destination rules

```
kubectl apply -f samples/bookinfo/networking/destination-rule-all.yaml

# 查看 destination rules
kubectl get destinationrules -n bookinfo
NAME          HOST          AGE
details       details       3h3m
productpage   productpage   3h3m
ratings       ratings       3h3m
reviews       reviews       3h3m

# 刷新 bookinfo 页面，可以看到随机显示三个版本的 Book Reviews：无星、黑色星形评价、红色星形评价
```

### 7.2 创建 Virtual Service，将所有流量指向 reviews:v1

```
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml 

# 刷新 bookinfo 页面，可以看到当前只显示一个版本的 Book Reviews：无星
```

### 7.3 修改 Virtual Service，将用户 jason 的流量指向 reviews:v2，其他用户仍然指向 reviews:v1

```
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-test-v2.yaml 

# 刷新 bookinfo 页面，点击 Sign in 并以 jason 登录可以显示 reviews:v2（黑色星形评价），登出之后或者以其他用户登录仍然显示 reviews:v1（无星）
```

### 7.4 修改 Virtual Service，为用户 jason 的流量注入 7s 的延迟

```
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-delay.yaml 

# 刷新 bookinfo 页面，点击 Sign in 并以 jason 登录可以看到获取 review 超时，登出之后或者以其他用户登录仍然显示 reviews:v1（无星）
# productpage 和 reviews 服务间的超时总时间为 6s（3s + 1次重试）
```

### 7.5 修改 Virtual Service，对用户 jason 的流量做 HTTP abort

```
kubectl apply -f samples/bookinfo/networking/virtual-service-ratings-test-abort.yaml 

# 刷新 bookinfo 页面，点击 Sign in 并以 jason 登录可以看到页面立即显示 Ratings service is currently unavailable，登出之后或者以其他用户登录仍然显示 reviews:v1（无星）
```

### 7.6 修改 Virtual Service，实现流量灰度迁移

```
# 修改 Virtual Service，将所有流量指向 reviews:v1
kubectl apply -f samples/bookinfo/networking/virtual-service-all-v1.yaml -n bookinfo
# 刷新 bookinfo 页面，可以看到当前只显示一个版本的 Book Reviews：无星

# 修改 Virtual Service，将 50% 流量指向 reviews:v3
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-50-v3.yaml -n bookinfo
# 刷新 bookinfo 页面，可以看到有一半的概率显示 reviews:v1（无星） 和 reviews:v3（红色星形评价）

# 修改 Virtual Service，将所有流量指向 reviews:v3
kubectl apply -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml -n bookinfo
# 刷新 bookinfo 页面，可以看到当前只显示一个版本的 Book Reviews：reviews:v3（红色星形评价）
```

## 8. 清理环境

```
# 删除所有相关资源
kubectl delete -f samples/bookinfo/networking/virtual-service-reviews-v3.yaml -n bookinfo
kubectl delete -f samples/bookinfo/networking/destination-rule-all.yaml
删除 nlb
kubectl delete -f samples/bookinfo/networking/bookinfo-gateway.yaml
kubectl delete -f samples/bookinfo/platform/kube/bookinfo.yaml
kubectl delete -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml

# 使用 istioctl 删除所有 rbac 权限、istio-system namespace 及相关资源
istioctl manifest generate --set profile=demo | kubectl delete -f -

# 删除安装时下载的 istio 目录，清除 ~/.bash_profile
cd ~
rm -rf istio
sed -i '/ISTIO_VERSION/d' ~/.bash_profile
unset ISTIO_VERSION
```
 
