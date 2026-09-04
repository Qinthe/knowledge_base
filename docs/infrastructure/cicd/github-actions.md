#  项目 CI/CD 自动化部署完整指南

## 项目概述

通过 GitHub Actions，实现 `git push` 后自动构建 Docker 镜像并部署到腾讯云服务器。


## 📋 目录

1. 准备工作
2. 第一步：腾讯云 TCR 镜像仓库配置
3. 第二步：服务器 SSH 密钥配置
4. 第三步：项目文件配置
5. 第四步：GitHub Actions 配置
6. 第五步：GitHub Secrets 配置
7. 第六步：服务器部署目录准备
8. 第七步：提交代码测试
9. 常见问题排查


## 1. 准备工作

### 需要准备的东西

| 项目         | 说明                   |
| :----------- | :--------------------- |
| GitHub 仓库  | 代码已 push 到 GitHub  |
| 腾讯云服务器 | 已有 Docker 环境       |
| 腾讯云 TCR   | 容器镜像服务（个人版） |
| 域名/公网 IP | 服务器可访问           |


## 2. 第一步：腾讯云 TCR 镜像仓库配置

### 2.1 开通容器镜像服务

1. 登录腾讯云控制台
2. 搜索"**容器镜像服务**"并进入
3. 选择"**个人版**"（免费），不要选企业版
4. 设置**访问密码**（⚠️ **这个密码要记住！**）

> ⚠️ **坑点 1**：TCR 密码是**单独设置的**，不是腾讯云登录密码！忘记后需到控制台重置。

### 2.2 创建命名空间

1. 左侧菜单点击"**命名空间**"
2. 点击"**新建**"
3. 输入命名空间名称，如 `glue`
4. 点击确认

> ⚠️ **坑点 2**：命名空间是**全局唯一**的，如果提示被占用，换个名字如 `glue2024`。

### 2.3 创建镜像仓库

1. 左侧菜单点击"**镜像仓库**"
2. 点击"**新建**"
3. 选择命名空间（刚创建的）
4. 仓库名称填写：`fullstack-app`
5. 类型选"**私有**"
6. 点击确认

### 2.4 获取镜像地址

创建完成后，镜像地址格式为：

```text
ccr.ccs.tencentyun.com/你的命名空间/你的镜像名:latest
```



例如：

```text
ccr.ccs.tencentyun.com/glue/fullstack-app:latest
```



> 📝 记下这个地址，后面多处要用到。


## 3. 第二步：服务器 SSH 密钥配置

### 3.1 在服务器上生成 SSH 密钥

登录服务器，执行：

```bash
# 生成 SSH 密钥对（一路回车，不要设置密码）
ssh-keygen -t rsa -b 4096
```



### 3.2 添加公钥到授权列表

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```



### 3.3 获取私钥内容

```bash
cat ~/.ssh/id_rsa
```



复制全部输出内容（包括开头和结尾的标记行）：

```text
-----BEGIN OPENSSH PRIVATE KEY-----
b3BlbnNzaC1rZXktdjEAAAAABG5vbmUAAAAEbm9uZQAAAAAAAAABAAACFwAAAAdzc2gtcn
...（中间内容）...
mW0+g7xP6F5VrUZqvVNjNpD+wTqwVg8tAE8zBRdMioPZRcCxKihc=
-----END OPENSSH PRIVATE KEY-----
```



> ⚠️ **坑点 3**：私钥内容**绝不能泄露**！只存在 GitHub Secrets 和你的本地。
>
> ⚠️ **坑点 4**：复制私钥时要**包含开头和结尾的标记行**，否则无效。


## 4. 第三步：项目文件配置

### 4.1 确认项目结构

你的项目结构：

```text
项目根目录/
├── GlueWeb/                  # Vue 项目
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
├── GlueBackend/
│   └── Glue.API/             # .NET API 项目
│       ├── Glue.API.csproj
│       ├── Program.cs
│       └── ...
├── Dockerfile                # 需要创建（根目录）
└── docker-compose.yml        # 需要创建（根目录）
```



### 4.2 创建 Dockerfile（项目根目录）

文件路径：`E:\Repository\GitHub\Glue\Dockerfile`

```dockerfile
# ===== 阶段 1: 构建 Vue 前端 =====
FROM node:18-alpine AS vue-builder
WORKDIR /app/vue
COPY GlueWeb/package*.json ./
RUN npm ci
COPY GlueWeb/ .
RUN npm run build

# ===== 阶段 2: 构建 .NET Core API =====
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS api-build
ARG BUILD_CONFIGURATION=Release
WORKDIR /src
COPY GlueBackend/Glue.API/Glue.API.csproj .
RUN dotnet restore "./Glue.API.csproj"
COPY GlueBackend/Glue.API/ .
RUN dotnet build "./Glue.API.csproj" -c $BUILD_CONFIGURATION -o /app/build
RUN dotnet publish "./Glue.API.csproj" -c $BUILD_CONFIGURATION -o /app/publish /p:UseAppHost=false

# ===== 阶段 3: 组装最终镜像 =====
FROM mcr.microsoft.com/dotnet/aspnet:10.0 AS final
WORKDIR /app

COPY --from=api-build /app/publish .
COPY --from=vue-builder /app/vue/dist ./wwwroot

ENV ASPNETCORE_URLS=http://+:8080
EXPOSE 8080

ENTRYPOINT ["dotnet", "Glue.API.dll"]
```



> ⚠️ **坑点 5**：路径必须和你的实际项目结构一致！Vue 在 `GlueWeb/`，API 在 `GlueBackend/Glue.API/`。

### 4.3 修改 Program.cs（托管前端页面）

文件路径：`GlueBackend/Glue.API/Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

// ... 原有服务注册代码 ...

var app = builder.Build();

// ===== 添加这两行（在 MapControllers 之前）=====
app.UseDefaultFiles();  // 默认找 index.html
app.UseStaticFiles();   // 托管 wwwroot 目录
// ==============================================

// ... 原有路由配置（app.MapControllers() 等）...

app.Run();
```



> ⚠️ **坑点 6**：`UseDefaultFiles()` 和 `UseStaticFiles()` 要放在 `MapControllers()` 之前。

### 4.4 删除子项目的 Dockerfile

如果有以下文件，删除它们（避免混淆）：

```bash
rm GlueWeb/Dockerfile
rm GlueBackend/Glue.API/Dockerfile
```




## 5. 第四步：GitHub Actions 配置

### 5.1 创建 workflows 目录

在项目根目录创建：

```bash
mkdir -p .github/workflows
```



### 5.2 创建 deploy.yml

文件路径：`.github/workflows/deploy.yml`

```yaml
name: Build and Deploy

on:
  push:
    branches: [ main ]

env:
  REGISTRY: ccr.ccs.tencentyun.com
  NAMESPACE: glue                    # 改成你的命名空间
  IMAGE_NAME: fullstack-app          # 改成你的镜像名

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Tencent Cloud TCR
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ secrets.TCR_USERNAME }}
          password: ${{ secrets.TCR_PASSWORD }}

      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.NAMESPACE }}/${{ env.IMAGE_NAME }}:latest
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to server via SSH
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.SERVER_IP }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /opt/app
            docker-compose pull
            docker-compose up -d
            docker image prune -f
```



> ⚠️ **坑点 7**：`NAMESPACE` 和 `IMAGE_NAME` 必须和 TCR 上创建的一致！


## 6. 第五步：GitHub Secrets 配置

### 6.1 进入 Secrets 页面

1. 打开 GitHub 仓库
2. 点击 `Settings` → 左侧菜单 `Secrets and variables` → `Actions`
3. 点击 `New repository secret`

### 6.2 添加以下 5 个 Secrets

| Secret 名称      | 值                 | 说明                      |
| :--------------- | :----------------- | :------------------------ |
| `TCR_USERNAME`   | `100047838092`     | 腾讯云账号 ID（OwnerUin） |
| `TCR_PASSWORD`   | 你设置的 TCR 密码  | ⚠️ 不是腾讯云登录密码！    |
| `SERVER_IP`      | 服务器公网 IP      | 如 `123.456.789.0`        |
| `SERVER_USER`    | `root` 或 `ubuntu` | SSH 登录用户名            |
| `SERVER_SSH_KEY` | SSH 私钥完整内容   | ⚠️ 包含开头结尾标记行      |

> ⚠️ **坑点 8**：`TCR_USERNAME` 填的是 `OwnerUin`（账号 ID），不是邮箱！
>
> ⚠️ **坑点 9**：如果忘记 TCR 密码，去腾讯云 TCR 控制台 → 个人版 → 重置密码。

### 6.3 检查 Secrets 是否添加成功

添加完成后，Secrets 列表应该显示 5 个已配置的密钥（值被隐藏）。


## 7. 第六步：服务器部署目录准备

### 7.1 登录服务器

```bash
ssh root@你的服务器IP
```



### 7.2 创建目录

```bash
mkdir -p /opt/app
cd /opt/app
```



### 7.3 创建 docker-compose.yml

```bash
cat > docker-compose.yml << 'EOF'
version: '3.8'

services:
  fullstack-app:
    image: ccr.ccs.tencentyun.com/glue/fullstack-app:latest
    container_name: fullstack-app
    restart: always
    ports:
      - "80:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
EOF
```



> ⚠️ **坑点 10**：镜像地址要**和 TCR 上完全一致**！

### 7.4 测试登录 TCR

```bash
docker login ccr.ccs.tencentyun.com -u 100047838092
```



输入 TCR 密码，看到 `Login Succeeded` 表示成功。

### 7.5 测试拉取镜像

```bash
docker pull ccr.ccs.tencentyun.com/glue/fullstack-app:latest
```



> 首次拉取会失败（因为还没构建过），这是正常的，但要确保**没有报权限错误**。


## 8. 第七步：提交代码测试

### 8.1 提交所有文件

```bash
git add .
git commit -m "添加 CI/CD 配置"
git push origin main
```



### 8.2 查看构建进度

1. 打开 GitHub 仓库
2. 点击 `Actions` 标签
3. 可以看到正在运行的 workflow
4. 点击进入查看详细日志

### 8.3 部署成功后访问

```text
http://你的服务器IP
```



应该能看到你的 Vue 前端页面。


## 9. 常见问题排查

| 问题                    | 可能原因                             | 解决方法                                  |
| :---------------------- | :----------------------------------- | :---------------------------------------- |
| GitHub Actions 构建失败 | Dockerfile 路径不对                  | 检查 `COPY` 路径是否和项目结构一致        |
| 推送镜像失败            | TCR 用户名/密码错误                  | 检查 `TCR_USERNAME` 和 `TCR_PASSWORD`     |
| 服务器拉取镜像失败      | 未登录 TCR                           | 在服务器上执行 `docker login`             |
| 容器启动失败            | 端口被占用                           | 修改 `docker-compose.yml` 中的端口映射    |
| 访问页面空白            | Program.cs 未添加 `UseStaticFiles()` | 检查并添加代码                            |
| Vue 页面 404            | Vue 构建输出目录不是 `dist`          | 检查 `vite.config.ts` 中的 `build.outDir` |
| SSH 连接失败            | 私钥不正确或公钥未添加               | 重新生成 SSH 密钥并添加公钥               |


## 📝 关键信息汇总表

| 项目            | 值                                                 |
| :-------------- | :------------------------------------------------- |
| 腾讯云 TCR 地址 | `ccr.ccs.tencentyun.com`                           |
| 腾讯云账号 ID   | `100047838092`                                     |
| 命名空间        | `glue`（改成你的）                                 |
| 镜像名          | `fullstack-app`（改成你的）                        |
| 完整镜像地址    | `ccr.ccs.tencentyun.com/glue/fullstack-app:latest` |
| 服务器部署目录  | `/opt/app`                                         |
| 服务器端口映射  | `80:8080`                                          |


## ✅ 最终检查清单

- [ ] 腾讯云 TCR 已开通，命名空间和镜像仓库已创建

- [ ] TCR 密码已设置并记录

- [ ] 服务器 SSH 密钥已生成，公钥已添加到 `authorized_keys`

- [ ] 项目根目录有 `Dockerfile`，路径正确

- [ ] `Program.cs` 已添加 `UseDefaultFiles()` 和 `UseStaticFiles()`

- [ ] `.github/workflows/deploy.yml` 已创建

- [ ] 5 个 GitHub Secrets 已添加

- [ ] 服务器 `/opt/app/docker-compose.yml` 已创建

- [ ] 服务器已 `docker login` 成功


**全部完成后，每次 `git push origin main` 就会自动构建并部署了！** 🚀