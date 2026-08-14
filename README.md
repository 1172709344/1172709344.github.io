# 1172709344.github.io

这是一个同时发布到 GitHub Pages 和 Cloudflare 的 Jekyll 站点。

## 本地构建

```powershell
bundle install
bundle exec jekyll build
```

可部署的静态站点会生成到 `_site/`。不要把仓库根目录直接作为静态资源发布，
因为根目录包含尚未处理的 Jekyll Front Matter、Liquid 模板和 Markdown 文章。

## Cloudflare Workers

`1172709344pages` Worker 由 `wrangler.jsonc` 配置。自定义构建会先生成 `_site/`，
部署时只把该目录作为静态资源上传。

```powershell
wrangler deploy
```

生产地址：
<https://1172709344pages.1172709344.workers.dev/>

## Cloudflare Pages

在 `1172709344-github-io` 项目的 **Settings > Builds & deployments** 中使用
以下精确配置：

| 配置项                 | 值                         |
| ---------------------- | -------------------------- |
| Production branch      | `main`                     |
| Framework preset       | `Jekyll`                   |
| Build command          | `bundle exec jekyll build` |
| Build output directory | `_site`                    |
| Root directory         | 留空（仓库根目录）         |

Pages Git 集成会安装 `Gemfile` 中声明的依赖，然后发布 `_site/`。完成本地构建后，
也可以使用下面的命令手动部署同一份产物：

```powershell
wrangler pages deploy _site --project-name 1172709344-github-io --branch main
```

生产地址：
<https://1172709344-github-io.pages.dev/>

## 冒烟测试

两个 Cloudflare 生产端点都必须返回文章正文，不能回退到首页或返回 404：

```text
/articles/is-it-gitops-if-it-is-stored-in-git/
```
