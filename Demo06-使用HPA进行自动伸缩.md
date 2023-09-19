# Demo06-使用HPA进行自动伸缩
--
#### Contributor: Tao Dai
#### 更新时间: 2023-09-19
#### 基于EKS版本: EKS 1.27
--

Metrics Server是Kubernetes内置自动缩放pipelines的可扩展、高效的容器资源指标来源,这些指标将推动部署的扩展行为,我们将使用 Kubernetes Metrics Server部署指标服务器。

## 1. 部署 Metrics Server

### 1.1 创建Metrics Server

a. 下载metrics-server安装文件到本地：

```
wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.4/components.yaml
```

b. 使用Kubernetes mutating admission webhook自动更换Kubernetes Pod的容器镜像

```
kubectl apply -f https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml
```
注：如果不成功，可以在浏览器直接输入

```
https://raw.githubusercontent.com/nwcdlabs/container-mirror/master/webhook/mutating-webhook.yaml
```
将yaml文件的内容复制保存到本地，生成mutating-webhook.yaml；

c. 将Image更新为Public ECR中1.27支持的最新版本：
<br>public.ecr.aws/eks-distro/kubernetes-sigs/metrics-server:v0.6.4-eks-1-27-latest
<br>并安装metrics-server
```
kubectl apply -f components.yaml
```
输出显示：

```
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

### 1.2 验证metrics-server APIService 的状态（可能需要等几分钟）

```
sudo yum install epel-release -y
sudo yum install jq -y

kubectl get apiservice v1beta1.metrics.k8s.io -o json | jq '.status'
```
输出显示：

```
{
  "conditions": [
    {
      "lastTransitionTime": "2023-09-19T12:46:19Z",
      "message": "all checks passed",
      "reason": "Passed",
      "status": "True",
      "type": "Available"
    }
  ]
}
```

## 2. 使用 HPA 扩展应用

### 2.1 部署测试App

```
kubectl create deployment php-apache --image=k8s.gcr.io/hpa-example:latest
kubectl set resources deploy php-apache --requests=cpu=200m
kubectl expose deploy php-apache --port 80
```

输出显示：

```
deployment.apps/php-apache created
deployment.apps/php-apache resource requirements updated
service/php-apache exposed
```

查看部署应用状态

```
kubectl get pod -l app=php-apache
```

输出显示：

```
NAME                          READY   STATUS    RESTARTS   AGE
php-apache-5b8f57879d-sfngd   1/1     Running   0          44s
```

2.2 创建 HPA 资源

当所分配的pod CPU资源使用超过50%时，进行HPA自动扩容

```
kubectl autoscale deployment php-apache `#The target average CPU utilization` \
    --cpu-percent=50 \
    --min=1 `#The lower limit for the number of pods that can be set by the autoscaler` \
    --max=10 `#The upper limit for the number of pods that can be set by the autoscaler` 
```

输出显示：

```
horizontalpodautoscaler.autoscaling/php-apache autoscaled
```

查看hpa状态：

```
kubectl get hpa
```
输出显示：

```
NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         10        1          45s
```

## 3. 产生负载触发扩容

打开一个新的terminal窗口，创建一个load-generator的pod

```
kubectl run -i --tty load-generator --image=busybox /bin/sh
```

在容器环境下执行如下命令：

```
while true; do wget -q -O - http://php-apache; done
```

输出显示：

```
If you don't see a command prompt, try pressing enter.
/ # while true; do wget -q -O - http://php-apache; done
OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!OK!
```

在之前的terminal下观察hpa的状态：

```
kubectl get hpa -w
```

输出显示：

```
kubectl get hpa -w
NAME         REFERENCE               TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   <unknown>/50%   1         10        1          2m2s
php-apache   Deployment/php-apache   481%/50%        1         10        1          2m30s
php-apache   Deployment/php-apache   467%/50%        1         10        4          2m45s
php-apache   Deployment/php-apache   163%/50%        1         10        8          3m
php-apache   Deployment/php-apache   59%/50%         1         10        10         3m15s
php-apache   Deployment/php-apache   47%/50%         1         10        10         3m30s
```

查看pod的情况

```
kubectl get pod
```

输出显示：

```
NAME                          READY   STATUS    RESTARTS   AGE
load-generator                1/1     Running   0          2m38s
php-apache-5b8f57879d-252lb   1/1     Running   0          94s
php-apache-5b8f57879d-4jgfj   1/1     Running   0          63s
php-apache-5b8f57879d-6cvg4   1/1     Running   0          79s
php-apache-5b8f57879d-d6b8s   1/1     Running   0          94s
php-apache-5b8f57879d-j79qq   1/1     Running   0          79s
php-apache-5b8f57879d-jf6zs   1/1     Running   0          63s
php-apache-5b8f57879d-lrr78   1/1     Running   0          94s
php-apache-5b8f57879d-nn8b6   1/1     Running   0          79s
php-apache-5b8f57879d-sfngd   1/1     Running   0          11m
php-apache-5b8f57879d-xvffh   1/1     Running   0          79s
```

停止load test 使用 Ctrl + C，退出load test 使用 Ctrl + D，hpa会慢慢将pod的数量降至最小数量1。

## 4. 清理环境

```
# 删除 hpa和服务
kubectl delete hpa,svc php-apache

# 删除php-apache部署
kubectl delete deployment php-apache

# 删除 load-generator pod
kubectl delete pod load-generator

# 删除metris-server
kubectl delete -f components.yaml
```
