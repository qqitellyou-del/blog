# 我的博客

基于 [Hexo](https://hexo.io/) 搭建，推送到 `main` 分支后由 GitHub Actions 自动构建并发布到 GitHub Pages。

## 本地开发

```bash
npm install
npx hexo server   # http://localhost:4000
```

## 写文章

```bash
npx hexo new "文章标题"
```

## 部署

在仓库 Settings → Pages 中将 Source 设为 **GitHub Actions**，之后每次推送到 `main` 都会自动部署。
