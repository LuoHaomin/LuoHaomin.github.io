# Hexo使用速查表
<!--more-->

### 1. 初始化和安装

- **`npm install -g hexo-cli`**
    - 全局安装 Hexo 命令行工具。
- **`hexo init`**
    - 初始化一个新的 Hexo 项目。
- **`npm install`**
    - 安装项目依赖。

> 配置：在`_config.yml`中可以修改博客的配置。

### 2. 生成和预览

- **`hexo g`** 或 **`hexo generate`**
    - 生成静态文件。
- **`hexo s`** 或 **`hexo server`**
    - 启动本地开发服务器预览博客。

### 3. 部署

- **`npm install hexo-deployer-git --save`**
    - 安装 Git 部署插件。
- **`hexo d`** 或 **`hexo deploy`**
    - 部署博客到远程仓库。

### 4. 主题管理

- **`git clone <主题仓库URL> themes/<主题名>`**
    - 克隆主题到项目主题目录。
- **`cd themes/<主题名> && npm install`**
    - 进入主题目录并安装主题依赖。

### 5. 项目维护

- **`npm ls --depth 0`**
    - 查看项目依赖树。
- **`npm audit fix`**
    - 修复项目依赖的安全漏洞。
- **`hexo clean`**
    - 清理生成的文件和缓存。如果修改了配置，需要先清理缓存才生效。

### 6. 页面和文章管理

- **`hexo new page <页面名>`**
    - 创建新页面。
- **`hexo new post <文章名>`**
    - 创建新文章。

### 7. 其他实用命令

- **`npm cache clean --force`**
    - 清理 npm 缓存。
- **`rm -rf node_modules && rm package-lock.json`**
    - 删除 `node_modules` 目录和 `package-lock.json` 文件，用于重新安装依赖。
