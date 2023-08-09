# Demo01-准备实验环境

--
#### Contributer: Zhengyu Ren
#### 更新时间: 2023-08-09
#### 基于EKS版本: EKS 1.27
--


本次Workshop中所需要的软件环境为AWS CLI, eksctl, kubectl; 且需要EKS相关IAM 权限.

## 1. AWS CLI
参考链接: [AWS CLI](https://aws.amazon.com/cli/)

### 1.1 Linux 系统
如您使用AWS Amazon Linux 操作系统, AWS CLI已预装.
如使用非Amazon Linux系统, 安装方式如下:
* 需要 Python 2.6.5 或更高版本
* 使用 pip 安装

```
pip install awscli
```
检查版本:

```
$ aws --version
aws-cli/2.13.7 Python/3.11.4 Darwin/22.5.0 source/arm64 prompt/off
```

### 1.2 Windows系统
Windows 系统下可以根据系统情况, 通过下载[64位](https://s3.amazonaws.com/aws-cli/AWSCLI64.msi) 或 [32位](https://s3.amazonaws.com/aws-cli/AWSCLI32.msi)安装包进行安装.

下载完成后, 双击即可进行安装.

检查版本:

```
>aws --version
aws-cli/2.13.7 Python/3.11.4 Windows/10 exe/AMD64 prompt/off
```

### 1.3 配置AWS CLI 权限
* 如您在AWS EC2中已经配置了IAM Role, 则可直接通过IAM Role获得相关权限.
	[参考文档](https://docs.amazonaws.cn/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)

* 如您希望使用IAM User进行操作, 请参考如下内容

```
#配置aws cli的用户权限
#具体区域设定以所需创建集群区域设置.
#北京区为cn-north-1; 宁夏区为cn-northwest-1
$aws configure
AWS Access Key ID :
AWS Secret Access Key :
Default region name:
Default output format [None]:
```

### 1.4 测试权限

```
aws sts get-caller-identity
# 如果可以正常返回以下内容(包含account id),则表示已经正确设置角色权限
{
    "Account": "<your account id, etc.11111111>", 
    "UserId": "AIDAIG42GHSYU2TYCMCZW", 
    "Arn": "arn:aws-cn:iam::<your account id, etc.11111111>:user/<iam user>"
}
```


## 2. eksctl
参考链接: [eksctl](https://eksctl.io/)

### 2.1 Mac系统安装
* 直接下载安装方式

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

* Brew安装

```
brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

### 2.2 Linux系统安装

```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

### 2.3 Windows系统安装
直接下载安装包后安装
<br>[https://github.com/weaveworks/eksctl/releases](https://github.com/weaveworks/eksctl/releases)

### 2.4 版本确认
```
$eksctl version
0.105.0-dev+aa76f1d4.2022-07-08T14:38:11Z
```

## 3.kubectl
**建议保持与集群版本一致的kubectl, 避免由于K8S版本更新较快导致的后续实验报错**
**如K8S针对 v1alpha和v1beta的deprecated**


参考链接: [kubectl](https://kubernetes.io/docs/tasks/tools/)

### 3.1 Mac系统安装
参考链接: [https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/)

* 直接下载

```
#Intel CPU
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/amd64/kubectl"

#Apple Sillicon
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/darwin/arm64/kubectl"

#调整权限, 并放置到推荐路径
chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl
sudo chown root: /usr/local/bin/kubectl

#查询版本
kubectl version --client
```

* 通过Brew安装

```
brew install kubectl
```

### 3.2 Linux系统安装
参考链接: [https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

* 直接下载

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"

sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

### 3.3 Windows系统安装
参考链接: [https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)

直接下载安装即可

## 4. EKS IAM 所需权限
参考文档:
<br>[https://docs.amazonaws.cn/eks/latest/userguide/security-iam.html](https://docs.amazonaws.cn/eks/latest/userguide/security-iam.html)
<br>[https://github.com/toreydai/eksctl-min-iam-policies-cn](https://github.com/toreydai/eksctl-min-iam-policies-cn)

* AmazonEC2FullAccess (AWS Managed Policy)
* EksAllAccess
* IamLimitedAccess
