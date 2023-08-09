# Demo12-使用CodePipeline进行CICD
--
#### Contributor: Zhengyu Ren
#### 更新时间: 2023-08-09
#### 基于EKS版本: EKS 1.27
--
## 1. 创建IAM 角色
在AWS CodePipeline中, 我们将使用AWS CodeBuild部署示例Kubernetes服务。
<br>这需要一个能够与EKS集群交互的AWS身份和访问管理 (IAM) 角色。
<br>在此步骤中, 我们将创建一个IAM角色, 并添加一个内联策略, 我们将在CodeBuild阶段使用该策略, 通过kubectl与EKS集群交互。

```
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION="cn-northwest-1"

TRUST="{ \"Version\": \"2012-10-17\", \"Statement\": [ { \"Effect\": \"Allow\", \"Principal\": { \"AWS\": \"arn:aws-cn:iam::${ACCOUNT_ID}:root\" }, \"Action\": \"sts:AssumeRole\" } ] }"

cat << EOF >  iam-role-policy
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "eks:Describe*",
            "Resource": "*"
        }
    ]
}
EOF

aws iam create-role --role-name EksWorkshopCodeBuildKubectlRole \
--assume-role-policy-document "$TRUST" --output text --query 'Role.Arn'

aws iam put-role-policy --role-name EksWorkshopCodeBuildKubectlRole \
--policy-name eks-describe --policy-document file://iam-role-policy
```

## 2. 修改AWS-Auth Configmap
现在我们已经创建了IAM角色, 我们将该角色添加到EKS集群的aws-auth ConfigMap中。
<br>一旦ConfigMap包含此新角色, 管道CodeBuild阶段的kubectl将能够通过IAM角色与EKS集群交互。

```
ROLE="    - rolearn: arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuildKubectlRole\n      username: build\n      groups:\n        - system:masters"

kubectl get -n kube-system configmap/aws-auth -o yaml \
 | awk "/mapRoles: \|/{print;print \"$ROLE\";next}1" > aws-auth-patch.yml

kubectl patch configmap/aws-auth -n kube-system \
--patch "$(cat aws-auth-patch.yml)"
```
## 3. 下载示例代码并上传到CodeCommit

### 3.1 下载示例资源
示例仓库: [https://github.com/rnzsgh/eks-workshop-sample-api-service-go.git](https://github.com/rnzsgh/eks-workshop-sample-api-service-go.git)

```
git clone https://github.com/rnzsgh/eks-workshop-sample-api-service-go.git
```


### 3.2 创建ECR
```
aws ecr create-repository --repository-name eksworkshop-codepipeline
```

### 3.3 创建CodeCommit
```
aws codecommit create-repository --repository-name \
eks-workshop-sample-api-service-go --repository-description "EKS Workshop CodeCommit"
```

### 3.4 上传下载的github的示例到CodeCommit
```
cd eks-workshop-sample-api-service-go

# 初始化
git init

# 添加跟踪
git add .

# 提交本地版本库
git commit -m 'codepipeline'

# 建立远程连接
git remote add orgin \
--no-tags https://git-codecommit.cn-northwest-1.amazonaws.com.cn/v1/repos/eks-workshop-sample-api-service-go

# 分支合并至main
git branch -m main

# 查看状态
git status

# 推送
git push orgin main
```

控制台中CodeCommit的界面可以看到已经成功上传完成


## 4. 创建CodeBuild

### 4.1 生成CodeBuild所需的IAM Policy文件

```
cat << EOF > code_build.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "ecr:ListTagsForResource",
                "ecr:UploadLayerPart",
                "ecr:ListImages",
                "codebuild:CreateReport",
                "logs:CreateLogStream",
                "iam:PassRole",
                "codebuild:UpdateReport",
                "ecr:CompleteLayerUpload",
                "codebuild:BatchPutCodeCoverages",
                "ecr:DescribeRepositories",
                "ecr:BatchCheckLayerAvailability",
                "ecr:GetLifecyclePolicy",
                "ecr:DescribeImageScanFindings",
                "ecr:GetLifecyclePolicyPreview",
                "ecr:GetDownloadUrlForLayer",
                "ecr:GetAuthorizationToken",
                "logs:CreateLogGroup",
                "logs:PutLogEvents",
                "ecr:PutImage",
                "codebuild:CreateReportGroup",
                "s3:*",
                "sts:AssumeRole",
                "ecr:BatchGetImage",
                "ecr:DescribeImages",
                "ecr:InitiateLayerUpload",
                "codebuild:BatchPutTestCases",
                "ecr:GetRepositoryPolicy"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "codecommit:GitPull",
            "Resource": "*"
        }
    ]
}
EOF
```

### 4.2 创建CodeBuild

```
# 配置CodeBuild的信任策略
CODEBUILD_TRUST="{\"Version\": \"2012-10-17\",\"Statement\": [{\"Effect\": \"Allow\",\"Principal\": {\"Service\": \"codebuild.amazonaws.com\"},\"Action\": \"sts:AssumeRole\"}]}"

# 配置CodeBuild的角色
aws iam create-role --role-name EksWorkshopCodeBuild \
--assume-role-policy-document "$CODEBUILD_TRUST" \
--output text --query 'Role.Arn'

# 配置CodeBuild对应的IAM策略
aws iam put-role-policy --role-name EksWorkshopCodeBuild \
 --policy-name codebuild --policy-document file://code_build.json

# 如下service-role需要改为创建的角色ARN
aws codebuild create-project \
--name "eksworkshop-codebuild" \
 --source "{\"type\": \"CODECOMMIT\",\"location\": \"https://git-codecommit.cn-northwest-1.amazonaws.com.cn/v1/repos/eks-workshop-sample-api-service-go\"}" \
 --artifacts {"\"type\": \"NO_ARTIFACTS\""} \
 --environment "{\"type\": \"LINUX_CONTAINER\",\"image\": \"aws/codebuild/standard:6.0\",\"computeType\": \"BUILD_GENERAL1_SMALL\",\"environmentVariables\": [{\"name\": \"REPOSITORY_URI\",\"value\": \"${ACCOUNT_ID}.dkr.ecr.cn-northwest-1.amazonaws.com.cn/eksworkshop-codepipeline\",\"type\": \"PLAINTEXT\"},{\"name\": \"REPOSITORY_NAME\",\"value\": \"eks-workshop-sample-api-service-go\",\"type\": \"PLAINTEXT\"},{\"name\": \"REPOSITORY_BRANCH\",\"value\": \"main\",\"type\": \"PLAINTEXT\"},{\"name\": \"EKS_CLUSTER_NAME\",\"value\": \"eksworkshop\",\"type\": \"PLAINTEXT\"},{\"name\": \"EKS_KUBECTL_ROLE_ARN\",\"value\": \"arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuildKubectlRole\",\"type\": \"PLAINTEXT\"}],\"privilegedMode\": true}" \
 --service-role "arn:aws-cn:iam::${ACCOUNT_ID}:role/EksWorkshopCodeBuild"
```


## 5. 创建CodePipeline

```
# 创建CodePipeline使用的IAM Policy文件
cat << EOF > iam_codepipeline_policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "codebuild:BatchGetBuildBatches",
                "appconfig:StartDeployment",
                "codedeploy:CreateDeployment",
                "states:DescribeStateMachine",
                "rds:*",
                "codedeploy:GetApplicationRevision",
                "codedeploy:GetDeploymentConfig",
                "iam:CreateRole",
                "iam:AttachRolePolicy",
                "cloudformation:CreateChangeSet",
                "sqs:*",
                "autoscaling:*",
                "codebuild:BatchGetBuilds",
                "cloudformation:DeleteChangeSet",
                "iam:PassRole",
                "cloudformation:UpdateStack",
                "cloudformation:DescribeChangeSet",
                "codedeploy:GetApplication",
                "cloudformation:ExecuteChangeSet",
                "sns:*",
                "cloudformation:SetStackPolicy",
                "lambda:ListFunctions",
                "codedeploy:RegisterApplicationRevision",
                "s3:*",
                "lambda:InvokeFunction",
                "appconfig:GetDeployment",
                "cloudformation:*",
                "elasticloadbalancing:*",
                "appconfig:StopDeployment",
                "cloudformation:DescribeStacks",
                "iam:CreatePolicy",
                "elasticbeanstalk:*",
                "sts:AssumeRole",
                "states:DescribeExecution",
                "cloudwatch:*",
                "cloudformation:CreateStack",
                "codebuild:StartBuildBatch",
                "cloudformation:DeleteStack",
                "codedeploy:GetDeployment",
                "ecr:*",
                "ecs:*",
                "states:StartExecution",
                "ec2:*",
                "codebuild:StartBuild",
                "cloudformation:ValidateTemplate",
                "s3:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "codecommit:UploadArchive",
                "codecommit:GetCommit",
                "codecommit:GetUploadArchiveStatus",
                "codecommit:GetRepository",
                "codecommit:GetBranch",
                "codecommit:CancelUploadArchive"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor2",
            "Effect": "Allow",
            "Action": [
                "servicecatalog:ListProvisioningArtifacts",
                "servicecatalog:DeleteProvisioningArtifact",
                "servicecatalog:UpdateProduct",
                "servicecatalog:DescribeProvisioningArtifact",
                "servicecatalog:CreateProvisioningArtifact"
            ],
            "Resource": "*"
        }
    ]
}
EOF

# 创建S3存储桶
**如下<S3_BUKCET>请自行设定一个名称**
aws s3api create-bucket --bucket <S3_BUCKET> \
 --region cn-northwest-1 \
 --create-bucket-configuration LocationConstraint=cn-northwest-1

# 创建CodePipeline IAM角色的信任策略
CODEPIPILINE_TRUST='{"Version": "2012-10-17","Statement": [{"Effect": "Allow","Principal": {"Service": "codepipeline.amazonaws.com"},"Action": "sts:AssumeRole"}]}'

# 创建CodePipeline IAM角色
aws iam create-role --role-name EksWorkshopCodePipleLine \
--assume-role-policy-document "$CODEPIPILINE_TRUST" \
--output text --query 'Role.Arn'

# 绑定IAM Policy
aws iam put-role-policy --role-name EksWorkshopCodePipleLine \
--policy-name codepipeline \
--policy-document file://iam_codepipeline_policy.json

# 创建CodePipeline的定义文件
# 需要根据初始创建的S3桶更新如下Location-<S3_BUCKET>; 以及ACCOUNT_ID为当前账户信息
# 需要将RoleARN改为测试环境的ARN
cat << EOF > codepipeline.json
{
  "pipeline": {
    "name": "eksworkshop-codepipleline",
    "roleArn": "arn:aws-cn:iam::<ACCOUNT_ID>:role/EksWorkshopCodePipleLine",
    "artifactStore": {
      "type": "S3",
      "location": "<S3_BUCKET>"
    },
    "stages": [
      {
        "name": "Source",
        "actions": [
          {
            "name": "Source",
            "actionTypeId": {
              "category": "Source",
              "owner": "AWS",
              "provider": "CodeCommit",
              "version": "1"
            },
            "runOrder": 1,
            "configuration": {
              "BranchName": "main",
              "OutputArtifactFormat": "CODE_ZIP",
              "PollForSourceChanges": "true",
              "RepositoryName": "eks-workshop-sample-api-service-go"
            },
            "outputArtifacts": [
              {
                "name": "SourceArtifact"
              }
            ],
            "inputArtifacts": [],
            "region": "cn-northwest-1",
            "namespace": "SourceVariables"
          }
        ]
      },
      {
        "name": "Build",
        "actions": [
          {
            "name": "Build",
            "actionTypeId": {
              "category": "Build",
              "owner": "AWS",
              "provider": "CodeBuild",
              "version": "1"
            },
            "runOrder": 1,
            "configuration": {
              "ProjectName": "eksworkshop-codebuild"
            },
            "outputArtifacts": [
              {
                "name": "BuildArtifact"
              }
            ],
            "inputArtifacts": [
              {
                "name": "SourceArtifact"
              }
            ],
            "region": "cn-northwest-1",
            "namespace": "BuildVariables"
          }
        ]
      }
    ],
    "version": 1
  }
}
EOF

# 创建pipeline
aws codepipeline create-pipeline --cli-input-json file://codepipeline.json
```
**此时可以登陆控制台或CLI查看CodePipeline的情况, 正常情况下会有一个正常执行中的操作**

**由于网络问题可能出现报错在CodeBuild节点, 选择重试即可**
**受制于网络情况, 有可能需要多次尝试**


## 6. 触发新发布
修改main.go文件的18行内容

```
res := &response{Message: "Hello World"}
---> 修改为
res := &response{Message: "Hello World from Zeno"}
```

提交同步

```
cd eks-workshop-sample-api-service-go

git add .

git commit -m 'change message text'

git push orgin main
```

## 7. 清理环境
```
# 删除Pipeline
aws codepipeline delete-pipeline \
--name $(aws codepipeline list-pipelines --query 'pipelines[].name' --output text)

# 删除CodeBuild项目
aws codebuild delete-project \
--name $(aws codebuild list-projects --query 'projects' --output text)

# 删除CodeCommit Repository
aws codecommit delete-repository \
--repository-name $(aws codecommit list-repositories --query 'repositories[].repositoryName' --output text)

# 删除ECR Repository
aws ecr delete-repository \
--repository-name eksworkshop-codepipeline --force

# 删除S3存储桶
aws s3 rm s3://<S3_BUCKET> --recursive
aws s3api delete-bucket --region cn-northwest-1 --bucket <S3_BUCKET>

# 删除IAM Role关联的Inline Policy
aws iam delete-role-policy --role-name EksWorkshopCodeBuild --policy-name codebuild
aws iam delete-role-policy --role-name EksWorkshopCodeBuildKubectlRole --policy-name eks-describe
aws iam delete-role-policy --role-name EksWorkshopCodePipleLine --policy-name codepipeline

# 删除IAM Role
aws iam delete-role --role-name EksWorkshopCodeBuild
aws iam delete-role --role-name EksWorkshopCodeBuildKubectlRole
aws iam delete-role --role-name EksWorkshopCodePipleLine
```
