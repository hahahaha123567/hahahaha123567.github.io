# hahahaha123567.github.io

这是基于 Hexo 的个人博客源码仓库。

## 环境

- Node.js 24.20.0
- Hexo 8.1.2
- 主题：[`themes/cactus`](https://github.com/hahahaha123567/hexo-theme-cactus.git)

## 初始化

```bash
git clone <repository-url>
cd hahahaha123567.github.io
git submodule update --init --recursive
npm ci
```

## 本地开发

启动本地预览服务器：

```bash
npm run server
```

然后访问 <http://localhost:4000>。

本地构建：

```bash
npm run build
```

构建结果位于 `public/`，该目录由 `.gitignore` 忽略，不提交到源码仓库。

## 发布

发布由 [`.github/workflows/pages.yml`](.github/workflows/pages.yml) 自动完成：

```text
push main → GitHub Actions → npm ci → npm run build → GitHub Pages
```

日常发布只需要提交源码并推送：

```bash
git add -A
git commit -m "更新博客内容"
git push
```

不要使用本地 `hexo deploy` 或 `deploy.sh` 发布。GitHub 仓库的 Pages 来源应设置为 `GitHub Actions`。

## 收藏文章

收藏索引位于 [`source/favorite/index.md`](source/favorite/index.md)。未来新增文章链接时，按照根目录 [`AGENTS.md`](AGENTS.md) 中的规则进行去重、归类和重要程度评估。

## 主题子模块

主题是独立 Git 子模块。修改主题时：

1. 在 `themes/cactus` 内提交并推送主题仓库；
2. 回到根仓库提交更新后的子模块指针；
3. 推送根仓库，让 GitHub Actions 使用该主题版本构建。

根仓库只记录主题提交指针，不记录主题仓库的未提交修改。

## Git 边界

源码、配置、依赖锁文件、Workflow 和主题子模块指针进入 Git。`public/`、`node_modules/`、`db.json`、`.deploy_git/` 等构建产物或本地状态由 `.gitignore` 排除。
