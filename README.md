# ECR existing images replication 

在 DR 或 Multi-Region 应用部署等场景下，ECR 中的镜像需要满足一次构建后进行 Cross-Region/Account 复制的需求，目前 ECR 原生支持在开启 Replication Feature 之后，新 Push 的镜像，将会自动复制到已配置的目标区域或账户中，但是对已存在的镜像，在不重新 Push 的情况下，不会被自动复制。 

本方案将介绍如何借助 CodeBuild ，针对 ECR 中已存在的镜像，进行按需复制，方案主要优势：
 - CodeBuild 提供托管的容器运行环境，不需要单独维护一套计算资源用来容器镜像复制
 - 运维简单，只需要创建一个 CodeBuild Project
 - 扩展性好，需要复制大量的容器镜像时，CodeBuild原生支持任务队列机制
 - 原生支持 Cross-region/account 复制

## Architecture

![Architecture](/images/arch.png)

该方案部署在 Source Account：

1. 准备需要复制的 ECR repo 和 Container image tag 列表
2. 调用 CodeBuild API，把需要复制的 ECR repo 和 image tag 作为环境变量带入 CodeBuild API
3. CodeBuild 获取到需要复制的 repo name 和 image tag 后，登录到 source ECR，通过 Docker Pull 拉取需要复制的镜像
4. CodeBuild 通过 STS Assume Role 切换权限到 Target 账户
5. 登录到 Target 账户 ECR，查看 Repo 是否存在，如果不存在则创建新的 Repo，然后将 CodeBuild 本地运行环境的镜像 Push 到 Target Repo 中 

CodeBuild 中的 buildspec.yaml 如下，主要在 build commands 中实现上述过程。

```
version: 0.2

#env:
  #variables:
     # key: "value"
     # key: "value"
  #parameter-store:
     # key: "value"
     # key: "value"
  #secrets-manager:
     # key: secret-id:json-key:version-stage:version-id
     # key: secret-id:json-key:version-stage:version-id
  #exported-variables:
     # - variable
     # - variable
  #git-credential-helper: yes
#batch:
  #fast-fail: true
  #build-list:
  #build-matrix:
  #build-graph:
phases:
  install:
    #If you use the Ubuntu standard image 2.0 or later, you must specify runtime-versions.
    #If you specify runtime-versions and use an image other than Ubuntu standard image 2.0, the build fails.
    runtime-versions:
      docker: 19
      # name: version
      # name: version
    #commands:
      # - command
      # - command
  #pre_build:
    #commands:
      # - command
      # - command
  build:
    commands:
      - |
        ##==Pull image from source repo==
        
        echo "login source registry"
        aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $src_account.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
        srcImage="$src_account.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO_NAME:$ECR_REPO_TAG"
        echo "pull image"
        docker pull $srcImage
        
        ##==Assume role==
        echo "Assuming role $dest_role"
        sts=$(aws sts assume-role \
          --role-arn "$dest_role" \
          --role-session-name target_profile \
          --query 'Credentials.[AccessKeyId,SecretAccessKey,SessionToken]' \
          --output text)
        echo "Converting sts to array"
        sts=($sts)
        echo "AWS_ACCESS_KEY_ID is ${sts[0]}"
        aws configure set aws_access_key_id ${sts[0]} --profile target_profile
        aws configure set aws_secret_access_key ${sts[1]} --profile target_profile
        aws configure set aws_session_token ${sts[2]} --profile target_profile
        echo "credentials stored in the profile named target_profile"
        
        ##==Check if repo exists in target account target region==
        if ! repoMsg=$(aws ecr describe-repositories --repository-names $ECR_REPO_NAME --region $dest_region --profile target_profile 2>&1); then
          echo -n "$ECR_REPO_NAME does not exists in ECR@$dest_region, creating... "
          aws ecr create-repository --repository-name $ECR_REPO_NAME --region $dest_region --profile target_profile > /dev/null
          echo "done."
        fi

        ##==Push image to target repo==
        targetImage="$dest_account.dkr.ecr.$dest_region.amazonaws.com/$ECR_REPO_NAME:$ECR_REPO_TAG"
        docker tag $srcImage $targetImage
        echo "login target registry in $dest_account"
        
        aws ecr get-login-password --region $dest_region --profile target_profile | docker login --username AWS --password-stdin $dest_account.dkr.ecr.$dest_region.amazonaws.com
        echo "push image to target account $dest_account target region $dest_region"
        docker push $targetImage
      # - command
      # - command
  #post_build:
    #commands:
      # - command
      # - command
#reports:
  #report-name-or-arn:
    #files:
      # - location
      # - location
    #base-directory: location
    #discard-paths: yes
    #file-format: JunitXml | CucumberJson
#artifacts:
  #files:
    # - location
    # - location
  #name: $(date +%Y-%m-%d)
  #discard-paths: yes
  #base-directory: location
#cache:
  #paths:
    # - paths
```

注意： CodeBuild Project 配置中使用了环境变量，用来标识 Source/Target 的一些参数，请酌情替换
 - dest_region： 需要复制到的目标区域 ID
 - dest_account： 需要复制到的目标账户 ID
 - dest_role： 目标账户的一个IAM Role ARN，要具备读取和创建 Repo，以及 Push 镜像的权限，同时 Trust Policy Assume Role 需要允许 Source 账户中的 CodeBuild Role
 - src_accont： 源账户 ID  
```
            "environment": {
                "type": "LINUX_CONTAINER",
                "image": "aws/codebuild/amazonlinux2-x86_64-standard:3.0",
                "computeType": "BUILD_GENERAL1_SMALL",
                "environmentVariables": [
                    {
                        "name": "dest_region",
                        "value": <Target_Region>,
                        "type": "PLAINTEXT"
                    },
                    {
                        "name": "dest_account",
                        "value": <Target_Account_ID>,
                        "type": "PLAINTEXT"
                    },
                    {
                        "name": "dest_role",
                        "value": <Target_Role_ARN>,
                        "type": "PLAINTEXT"
                    },
                    {
                        "name": "src_account",
                        "value": <Source_Account_ID>,
                        "type": "PLAINTEXT"
                    }
                ],
                "privilegedMode": true,
                "imagePullCredentialsType": "CODEBUILD"
            },
```

在调用 CodeBuild 时，API Request 中将包含```ECR_REPO_NAME```, ```ECR_REPO_TAG```两项输入，作为环境变量被 CodeBuild 使用。
```
response = client.start_build(
    projectName='ecr-replication-build',
    environmentVariablesOverride=[
        {
            'name': 'ECR_REPO_NAME',
            'value': 'nginx',
            'type': 'PLAINTEXT'
        },
        {
            'name': 'ECR_REPO_TAG',
            'value': '123',
            'type': 'PLAINTEXT'
        },
    ]
)
```

