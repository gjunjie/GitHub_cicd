# RAG App - AWS App Runner 部署项目

本项目演示如何使用 Terraform 在 AWS 上创建基础设施，并通过 GitHub Actions 自动部署 RAG (Retrieval-Augmented Generation) 应用到 AWS App Runner。

## 📋 前置要求

1. **AWS 账户** - 确保您有 AWS 账户并配置了 AWS CLI
2. **Terraform** - 安装 Terraform (版本 >= 1.0)
3. **GitHub 账户** - 用于 GitHub Actions CI/CD
4. **OpenAI API Key** - 用于 RAG 应用

## 🚀 快速开始

### 步骤 1: 配置 AWS 凭证

确保您的 AWS CLI 已配置：

```bash
aws configure
```

### 步骤 2: 配置 Terraform 变量

复制示例变量文件并填入您的信息：

```bash
cp terraform.tfvars.example terraform.tfvars
```

编辑 `terraform.tfvars` 文件：

```hcl
github_org_or_user = "your-github-username"
github_repo_name = "your-repo-name"
openai_api_key = "sk-your-openai-api-key-here"
manage_apprunner_via_terraform = false
```

**重要**: `terraform.tfvars` 包含敏感信息，已在 `.gitignore` 中，请勿提交到 Git。

### 步骤 3: 初始化 Terraform

```bash
terraform init
```

### 步骤 4: 创建 AWS 资源

运行 Terraform 来创建所有必需的 AWS 资源：

```bash
terraform apply
```

或者使用自动批准：

```bash
terraform apply -auto-approve
```

这将创建以下 AWS 资源：

- ✅ **OIDC Provider** - GitHub Actions 身份认证
- ✅ **IAM Role** - GitHub Actions 部署角色 (`github-actions-deploy-role`)
- ✅ **ECR Repository** - Docker 镜像仓库 (`bee-edu-rag-app`)
- ✅ **Secrets Manager** - 存储 OpenAI API Key (`bee-edu-openai-key-secret`)
- ✅ **IAM Roles** - App Runner 服务角色和实例角色
- ✅ **App Runner Service** (可选) - 如果 `manage_apprunner_via_terraform = true`

### 步骤 5: 保存 Terraform 输出

Terraform 应用成功后，会输出以下重要信息，请保存这些值（这些值将用于配置 GitHub Secrets）：

```
aws_region = "us-east-1"
github_actions_role_arn = "arn:aws:iam::ACCOUNT_ID:role/github-actions-deploy-role"
ecr_repository_name = "bee-edu-rag-app"
apprunner_service_arn = "arn:aws:apprunner:us-east-1:ACCOUNT_ID:service/..." (可能为 null)
```

**注意：** 如果 `apprunner_service_arn` 输出为 `null`（当 `manage_apprunner_via_terraform = false` 时），这是正常的。App Runner 服务将由 GitHub Actions 创建。

### 步骤 6: 配置 GitHub Secrets

**学生必须在 GitHub 仓库的 Settings > Secrets > Actions 中，配置以下4个Secrets（所有值均来自 terraform apply 的 outputs）：**

1. **AWS_REGION**
   - 值：`us-east-1`（或您使用的 AWS 区域）
   - 说明：AWS 区域标识符

2. **ECR_REPOSITORY**
   - 值：来自 Terraform output `ecr_repository_name`
   - 说明：ECR 镜像仓库名称，用于存储 Docker 镜像

3. **APP_RUNNER_ARN**
   - 值：来自 Terraform output `apprunner_service_arn`
   - 说明：App Runner 服务的 ARN，用于部署和更新应用

4. **AWS_IAM_ROLE_TO_ASSUME**
   - 值：来自 Terraform output `github_actions_role_arn`
   - 说明：GitHub Actions 用于访问 AWS 的 IAM 角色 ARN

**配置步骤：**
1. 在 GitHub 仓库页面，点击 **Settings** → **Secrets and variables** → **Actions**
2. 点击 **New repository secret**
3. 依次添加上述4个 Secrets，名称和值必须完全匹配
4. 所有值都可以从 `terraform apply` 成功后的输出中获取

**重要提示：**
- 如果 `apprunner_service_arn` 输出为 `null`（当 `manage_apprunner_via_terraform = false` 时），您需要先通过 GitHub Actions 创建 App Runner 服务，然后手动获取 ARN 并添加到 Secrets 中
- 确保所有 Secrets 的值与 Terraform 输出完全一致，包括大小写和特殊字符

### 步骤 7: 准备 FAISS 索引（本地）

在本地构建 FAISS 向量索引：

```bash
export OPENAI_API_KEY="your-openai-api-key"
python ingest.py
```

这将创建 `faiss_index/` 目录，包含向量索引文件。

### 步骤 8: 设置 GitHub Actions 工作流

创建 `.github/workflows/deploy.yml` 文件（如果还没有）：

```yaml
name: Deploy to AWS App Runner

on:
  push:
    branches:
      - main

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: ${{ secrets.ECR_REPOSITORY }}

jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_IAM_ROLE_TO_ASSUME }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build and push Docker image
        run: |
          docker build -t $ECR_REPOSITORY:${{ github.sha }} .
          docker tag $ECR_REPOSITORY:${{ github.sha }} $ECR_REPOSITORY:latest
          docker push $ECR_REPOSITORY:${{ github.sha }}
          docker push $ECR_REPOSITORY:latest

      - name: Deploy to App Runner
        uses: awslabs/amazon-app-runner-deploy@main
        with:
          service: bee-edu-rag-service
          image: ${{ steps.ecr-login.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          region: ${{ env.AWS_REGION }}
          access-role-arn: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/bee-edu-apprunner-role
          instance-role-arn: arn:aws:iam::${{ secrets.AWS_ACCOUNT_ID }}:role/bee-edu-apprunner-instance-role
          port: 8080
          cpu: 1
          memory: 2
          wait-for-service-stability-seconds: 600
```

## 📁 项目结构

```
.
├── main.tf                    # Terraform 基础设施配置（必须使用）
├── terraform.tfvars.example   # Terraform 变量示例
├── app.py                     # FastAPI RAG 应用
├── Dockerfile                 # Docker 镜像配置
├── requirements.txt           # Python 依赖
├── ingest.py                  # FAISS 索引生成脚本
├── data.txt                   # RAG 数据源
├── .gitignore                 # Git 忽略文件
└── README.md                  # 本文件
```

## 🔧 重要说明

### 使用讲师提供的 main.tf

**学生必须使用讲师提供的 `main.tf` 脚本**，不要修改其中的配置。此文件包含了所有必需的 AWS 资源配置。

### Terraform 变量说明

- `github_org_or_user`: 您的 GitHub 用户名或组织名
- `github_repo_name`: 您的 GitHub 仓库名
- `openai_api_key`: OpenAI API Key（将存储在 AWS Secrets Manager）
- `manage_apprunner_via_terraform`: 
  - `false` (推荐): 由 GitHub Actions 创建和管理 App Runner 服务
  - `true`: 由 Terraform 创建 App Runner 服务（需要先推送镜像到 ECR）

### 验证部署

部署成功后，您可以通过以下方式验证：

1. 检查 AWS App Runner 控制台中的服务状态
2. 访问应用的健康检查端点：`https://your-app-runner-url/`
3. 测试聊天端点：`POST /chat` with `{"question": "your question"}`

## 🧹 清理资源

要删除所有创建的资源：

```bash
terraform destroy
```

**注意**: 这将删除所有 AWS 资源，包括 ECR 镜像、Secrets Manager 中的密钥等。

## 📚 参考资料

- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS App Runner](https://docs.aws.amazon.com/apprunner/)
- [GitHub Actions OIDC](https://docs.github.com/en/actions/deployment/security-hardening-your-deployments/about-security-hardening-with-openid-connect)

## ❓ 常见问题

### Q: Terraform apply 失败怎么办？

A: 检查以下几点：
- AWS 凭证是否正确配置
- 是否有足够的 IAM 权限
- 变量是否正确设置
- 资源名称是否冲突（某些资源名称在 AWS 账户中必须唯一）

### Q: GitHub Actions 部署失败？

A: 确保：
- GitHub Secrets 已正确配置
- IAM 角色 ARN 正确
- OIDC Provider 已创建
- 仓库名称和分支名称匹配 Terraform 配置

### Q: 如何更新 OpenAI API Key？

A: 更新 `terraform.tfvars` 中的 `openai_api_key`，然后运行 `terraform apply`。

---

**祝您部署顺利！** 🎉

