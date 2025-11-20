# GitHub Actions 工作流设置指南

本指南将帮助您完成 GitHub Actions 工作流的配置，使其能够自动部署到 AWS App Runner。

## 📋 前置检查清单

在开始之前，请确保：

- ✅ Terraform 已成功应用（`terraform apply`）
- ✅ AWS 资源已创建（ECR、IAM 角色、OIDC Provider 等）
- ✅ GitHub 仓库已创建并可以访问
- ✅ 您有权限配置 GitHub Secrets

## 🚀 完整设置步骤

### 步骤 1: 获取 Terraform 输出值

在本地运行以下命令获取所需的配置值：

```bash
cd /Users/jg/Documents/github_cicd
terraform output
```

您应该看到类似以下的输出：

```
aws_region = "us-east-1"
ecr_repository_name = "bee-edu-rag-app"
github_actions_role_arn = "arn:aws:iam::922187738328:role/github-actions-deploy-role"
apprunner_service_arn = null  # 如果为 null，这是正常的
```

**记录这些值，您将在下一步中使用它们。**

### 步骤 2: 配置 GitHub Secrets

1. **打开您的 GitHub 仓库**
   - 访问：`https://github.com/gjunjie/GitHub_cicd`（根据您的实际仓库地址）

2. **进入 Secrets 设置页面**
   - 点击仓库顶部的 **Settings** 标签
   - 在左侧菜单中，点击 **Secrets and variables** → **Actions**

3. **添加以下 4 个 Secrets**

   点击 **New repository secret** 按钮，依次添加：

   #### Secret 1: `AWS_REGION`
   - **Name**: `AWS_REGION`
   - **Value**: `us-east-1`（或您使用的 AWS 区域）
   - **来源**: Terraform output `aws_region`

   #### Secret 2: `ECR_REPOSITORY`
   - **Name**: `ECR_REPOSITORY`
   - **Value**: `bee-edu-rag-app`（或您的 ECR 仓库名称）
   - **来源**: Terraform output `ecr_repository_name`

   #### Secret 3: `AWS_IAM_ROLE_TO_ASSUME`
   - **Name**: `AWS_IAM_ROLE_TO_ASSUME`
   - **Value**: `arn:aws:iam::922187738328:role/github-actions-deploy-role`（您的实际 ARN）
   - **来源**: Terraform output `github_actions_role_arn`
   - **重要**: 这是 OIDC 认证的关键，必须完全匹配

   #### Secret 4: `APP_RUNNER_ARN`（可选，首次部署可留空）
   - **Name**: `APP_RUNNER_ARN`
   - **Value**: 
     - 如果 Terraform output `apprunner_service_arn` 不为 `null`，使用该值
     - 如果为 `null`，可以暂时不设置，工作流会自动创建服务
     - 首次部署后，从工作流日志中获取创建的 ARN，然后添加此 Secret

### 步骤 3: 验证工作流文件

确保工作流文件已提交到仓库：

```bash
# 检查文件是否存在
ls -la .github/workflows/deploy.yml

# 如果文件存在，提交到 Git
git add .github/workflows/deploy.yml
git commit -m "Add GitHub Actions deployment workflow"
git push origin main
```

### 步骤 4: 准备应用文件

确保以下文件已准备好并提交到仓库：

- ✅ `app.py` - FastAPI 应用
- ✅ `Dockerfile` - Docker 镜像配置
- ✅ `requirements.txt` - Python 依赖
- ✅ `data.txt` - 数据文件
- ✅ `ingest.py` - 索引生成脚本（如果需要）

**注意**: 如果使用 FAISS 索引，确保 `faiss_index/` 目录已创建并包含索引文件。

### 步骤 5: 触发工作流

工作流会在以下情况自动触发：

1. **推送到 main 分支**
   ```bash
   git push origin main
   ```

2. **手动触发**（如果已配置）
   - 在 GitHub 仓库页面，点击 **Actions** 标签
   - 选择 **Deploy to AWS App Runner** 工作流
   - 点击 **Run workflow** 按钮

### 步骤 6: 监控部署过程

1. **查看工作流运行状态**
   - 在 GitHub 仓库页面，点击 **Actions** 标签
   - 查看最新的工作流运行

2. **检查每个步骤**
   - ✅ Checkout code
   - ✅ Configure AWS credentials using OIDC
   - ✅ Login to Amazon ECR
   - ✅ Build, tag, and push Docker image
   - ✅ Create or Update App Runner Service
   - ✅ Wait for deployment to complete

3. **查看日志**
   - 点击失败的步骤查看详细错误信息
   - 检查 AWS 权限是否正确配置

### 步骤 7: 获取 App Runner 服务 ARN（首次部署）

如果这是首次部署且 `APP_RUNNER_ARN` Secret 未设置：

1. **从工作流日志中获取 ARN**
   - 在工作流运行的日志中，查找 "New App Runner service created:"
   - 复制显示的 ARN

2. **添加到 GitHub Secrets**
   - 进入 **Settings** → **Secrets and variables** → **Actions**
   - 添加或更新 `APP_RUNNER_ARN` Secret
   - 使用从日志中复制的 ARN

3. **后续部署**
   - 下次推送代码时，工作流将自动更新现有服务

## 🔍 故障排除

### 问题 1: OIDC 认证失败

**错误信息**: `Error: Could not assume role with OIDC`

**解决方案**:
- 检查 `AWS_IAM_ROLE_TO_ASSUME` Secret 是否正确
- 验证 Terraform 中的 GitHub 仓库名称和分支名称是否匹配
- 确认 OIDC Provider 已正确创建：
  ```bash
  aws iam list-open-id-connect-providers
  ```

### 问题 2: ECR 推送失败

**错误信息**: `Error: AccessDenied`

**解决方案**:
- 检查 IAM 角色是否有 ECR 推送权限
- 验证 ECR 仓库名称是否正确
- 确认 `ECR_REPOSITORY` Secret 的值与 Terraform 输出一致

### 问题 3: App Runner 服务创建失败

**错误信息**: `Error: InvalidParameterException`

**解决方案**:
- 检查 Docker 镜像是否成功推送到 ECR
- 验证镜像标识符格式是否正确
- 确认 IAM 角色 ARN 是否正确

### 问题 4: Secrets Manager 访问失败

**错误信息**: `Error: ResourceNotFoundException`

**解决方案**:
- 确认 Secrets Manager 中的 secret 已创建
- 检查 secret 名称是否为 `bee-edu-openai-key-secret`
- 如果不需要 OpenAI API Key，工作流会自动跳过

## ✅ 验证部署成功

部署成功后，您可以通过以下方式验证：

1. **检查 App Runner 服务状态**
   ```bash
   aws apprunner describe-service \
     --service-arn <YOUR_SERVICE_ARN> \
     --region us-east-1
   ```

2. **获取服务 URL**
   ```bash
   aws apprunner describe-service \
     --service-arn <YOUR_SERVICE_ARN> \
     --region us-east-1 \
     --query 'Service.ServiceUrl' \
     --output text
   ```

3. **测试应用端点**
   ```bash
   curl https://<YOUR_SERVICE_URL>/
   ```

## 📝 重要提示

1. **OIDC 认证**: 工作流使用 OIDC 进行无密钥认证，这是最安全的方式
2. **分支限制**: 工作流只在 `main` 分支触发，这是由 Terraform 配置的
3. **首次部署**: 如果 App Runner 服务不存在，工作流会自动创建
4. **后续更新**: 设置 `APP_RUNNER_ARN` Secret 后，工作流会自动更新现有服务

## 🎉 完成！

如果所有步骤都成功完成，您的应用现在应该已经部署到 AWS App Runner 了！

每次推送到 `main` 分支时，工作流都会自动：
- 构建新的 Docker 镜像
- 推送到 ECR
- 更新 App Runner 服务

---

**需要帮助？** 检查工作流日志或参考 [README.md](README.md) 中的常见问题部分。

