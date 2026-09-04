# 技术知识库

个人技术知识库，基于 [MkDocs](https://www.mkdocs.org/) + [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) 构建，部署于 GitHub Pages。

在线访问：<https://qinthe.github.io/knowledge_base/>

## 内容结构

| 分类 | 说明 |
| :--- | :--- |
| 基础设施 | CI/CD 自动化部署、局域网组网（ZeroTier） |
| 服务器 | Nginx 反向代理与负载均衡 |
| DevOps | DevOps 实践与工具链 |
| 开发框架 | 开发框架使用笔记 |

## 本地预览

```bash
pip install -r requirements.txt
mkdocs serve
```

浏览器打开 <http://127.0.0.1:8000> 即可预览。

## 部署

推送到 `main` 分支后，GitHub Actions 会自动构建并部署到 GitHub Pages（见 `.github/workflows/deploy-docs.yml`）。

## 目录约定

- 每个分类目录下用 `index.md` 作为该分类的首页
- 文档使用中文，代码块标注语言以获得语法高亮
