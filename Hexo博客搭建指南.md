# nju-blog Hexo 博客项目指南

这份指南是根据本项目（`nju-blog`）的实际环境和配置整理的最新操作手册，涵盖了从环境修复到日常写作发布的完整流程。

## 一、项目概况

*   **项目名称**：nju-blog
*   **仓库地址**：[https://github.com/xuyafei1996/nju-blog](https://github.com/xuyafei1996/nju-blog)
*   **博客网址**：[https://xuyafei1996.github.io/nju-blog/](https://xuyafei1996.github.io/nju-blog/)
*   **当前分支**：`master`

---

## 二、首次运行与环境恢复

如果你是刚接手这个项目，或者换了电脑，请按照以下步骤初始化环境。

### 1. 安装基础工具
确保电脑已安装：
*   **Node.js** (建议 v18 或 v20 LTS)
*   **Git**

### 2. 安装依赖 (关键步骤)
本项目由于部分插件依赖较旧，**必须**使用 `--legacy-peer-deps` 参数安装，否则会报错。

在项目根目录 (`c:\Develop\my-blog`) 打开终端执行：

```powershell
# 推荐：同时使用淘宝镜像源加速
npm install --legacy-peer-deps --registry=https://registry.npmmirror.com
```

> **常见报错处理**：
> 如果遇到 `Cannot read properties of null` 或其他奇怪的 npm 报错，请执行以下命令彻底清理后重试：
> ```powershell
> Remove-Item -Recurse -Force node_modules
> Remove-Item -Force package-lock.json
> npm cache clean --force
> npm install --legacy-peer-deps
> ```

---

## 三、本地写作与预览

### 1. 启动本地服务器
```powershell
npx hexo s
```
或者
```powershell
npx hexo server
```

### 2. 预览文章
启动成功后，访问以下地址（注意后缀）：
👉 **http://localhost:4000/nju-blog/**

> **注意**：由于我们在 `_config.yml` 中配置了 `root: /nju-blog/`，所以本地访问必须带上这个后缀，否则样式和图片会加载失败。

### 3. 新建文章
```powershell
npx hexo new "文章标题"
```
文件会生成在 `source/_posts/文章标题.md`。

---

## 四、发布上线 (GitHub Actions)

本项目已配置自动化部署（CI/CD），你**不需要**手动运行 `hexo d`。

### 1. 发布步骤
只需将源码推送到 GitHub 的 `master` 分支：

```powershell
# 1. 添加修改
git add .

# 2. 提交修改
git commit -m "写了一篇新文章: 标题"

# 3. 推送到远程
git push
```

### 2. 等待部署
1.  推送后，GitHub Actions 会自动触发构建。
2.  等待约 1-2 分钟。
3.  访问 [https://xuyafei1996.github.io/nju-blog/](https://xuyafei1996.github.io/nju-blog/) 查看更新。

---

## 五、关键配置说明 (已修复)

以下是我们在搭建过程中修复的关键配置，了解这些有助于你排查未来可能遇到的问题。

### 1. 站点配置文件 `_config.yml`
为了适配 GitHub Pages 的子目录项目（`nju-blog`），必须配置：

```yaml
# 这里的 URL 是 GitHub Pages 的访问地址
url: https://xuyafei1996.github.io/nju-blog
# 这里的 root 是项目在域名下的子路径
root: /nju-blog/
```
*如果不配置 `root`，会导致 CSS 样式丢失，页面一片空白。*

### 2. 自动化部署脚本 `.github/workflows/deploy.yml`
*   **目录修正**：脚本必须放在 `.github/workflows/` (复数) 目录下，之前因为写成 `workflow` 导致无法触发。
*   **分支匹配**：脚本中配置了监听 `master` 分支的 push 事件。
*   **权限设置**：配置了 `permissions: contents: read, pages: write, id-token: write` 以允许 Actions 推送静态文件。

---

## 六、常用命令速查表

| 场景 | 命令 | 说明 |
| :--- | :--- | :--- |
| **安装依赖** | `npm install --legacy-peer-deps` | **必用**，解决依赖冲突 |
| **清理缓存** | `hexo clean` | 样式错乱时优先执行 |
| **本地预览** | `npx hexo s` | 访问 localhost:4000/nju-blog/ |
| **新建文章** | `npx hexo new "标题"` | 生成 md 文件 |
| **生成静态文件** | `npx hexo g` | (通常不需要手动跑，Actions 会跑) |
