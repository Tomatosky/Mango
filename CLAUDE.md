# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Mango 是一个自托管的漫画服务器和阅读器,使用 Crystal 语言编写后端,使用 JavaScript/jQuery + UIKit 编写前端。

**重要提示:** 该项目自 2025 年 3 月起不再维护。

## 技术栈

### 后端 (Crystal)
- **框架**: Kemal (轻量级 Web 框架,类似 Sinatra)
- **数据库**: SQLite3
- **会话管理**: kemal-session
- **模板**: ECR (Embedded Crystal)
- **CLI**: Clim
- **压缩包处理**: archive.cr
- **JavaScript 引擎**: duktape (用于插件系统)

### 前端
- **UI 框架**: UIKit 3.x
- **JavaScript 库**: jQuery, Alpine.js, Moment.js
- **构建工具**: Gulp + Babel + Less
- **图标**: FontAwesome

## 常用命令

### 开发环境设置
```bash
# 安装 Crystal 依赖
shards install --production

# 安装前端依赖并编译 LESS
make setup

# 或分步执行:
yarn install
yarn gulp dev
```

### 构建和运行
```bash
# 开发模式运行 (不编译,使用 public/ 目录的静态文件)
make run
# 或: crystal run src/mango.cr --error-trace

# 编译生产版本 (静态文件会嵌入二进制文件)
make build
# 或: crystal build src/mango.cr --release --progress --error-trace

# 完整构建 (包括前端资源压缩)
make all
```

### 测试和代码检查
```bash
# 运行测试
make test
# 或: crystal spec

# 代码格式和 lint 检查
make check
# 包含: crystal tool format --check 和 ./bin/ameba
```

### 前端资源构建
```bash
# 构建开发版前端资源
yarn gulp dev

# 构建生产版前端资源 (压缩到 dist/ 目录)
yarn gulp deploy
# 或: yarn uglify
```

### 用户管理 (CLI)
```bash
# 添加用户
./mango admin user add -u username -p password [-a]

# 删除用户
./mango admin user delete username

# 更新用户
./mango admin user update username [-u new_username] [-p new_password] [-a]

# 列出所有用户
./mango admin user list
```

## 项目架构

### 目录结构
```
src/
├── mango.cr              # 入口文件,CLI 定义
├── server.cr             # Kemal 服务器配置
├── config.cr             # 配置文件解析
├── storage.cr            # 数据库操作封装
├── library/              # 核心库管理
│   ├── library.cr        # 图书馆主类,扫描和缓存
│   ├── title.cr          # 漫画标题(系列)
│   ├── entry.cr          # 条目基类
│   ├── archive_entry.cr  # 压缩包条目
│   └── dir_entry.cr      # 目录条目
├── routes/               # HTTP 路由
│   ├── main.cr           # 主要页面路由
│   ├── api.cr            # RESTful API
│   ├── reader.cr         # 阅读器路由
│   ├── admin.cr          # 管理界面
│   └── opds.cr           # OPDS 协议支持
├── handlers/             # Kemal 中间件
│   ├── auth_handler.cr   # 认证处理
│   ├── static_handler.cr # 静态文件 (release 模式)
│   └── upload_handler.cr # 文件上传
├── plugin/               # 插件系统
│   ├── plugin.cr         # 插件加载和执行
│   ├── downloader.cr     # 下载管理
│   └── subscriptions.cr  # 订阅管理
└── util/                 # 工具函数
    ├── signature.cr      # 文件签名(用于变更检测)
    ├── validation.cr     # 输入验证
    └── chapter_sort.cr   # 章节排序算法
```

### 核心概念

#### Library (src/library/library.cr)
- 单例类,管理整个漫画库
- **扫描机制**: 定期扫描 `library_path` 目录,检测新增/删除的漫画
- **缓存系统**: 将库元数据序列化为 YAML.gz 文件以加速启动
- **缩略图生成**: 后台任务定期为漫画生成缩略图

#### Title (src/library/title.cr)
- 代表一个漫画系列(通常是一个文件夹)
- 包含多个 `Entry` (卷/章节)
- 存储阅读进度、标签等元数据

#### Entry (Archive/Dir Entry)
- **ArchiveEntry**: `.cbz`, `.zip`, `.cbr`, `.rar` 压缩包
- **DirEntry**: 包含图片文件的目录
- 负责提取图片页面并生成缩略图

#### Storage (src/storage.cr)
- SQLite 数据库封装层
- 使用 `mg` 库处理数据库迁移 (migration/ 目录)
- 存储用户、阅读进度、标签、插件账号等数据

#### Plugin System
- 基于 JavaScript (Duktape 引擎)
- 插件可以从第三方网站下载漫画
- 支持订阅和自动更新

### 构建系统

#### Crystal 编译
- **开发模式**: 不嵌入静态文件,使用 `public/` 目录
- **发布模式**: 使用 `baked_file_system` 将 `dist/` 目录嵌入二进制文件

#### 前端构建 (gulpfile.js)
1. **node-modules-copy**: 从 node_modules 复制必要资源
2. **less**: 编译 `.less` 文件为 `.css`
3. **babel**: 转译和压缩 JS 文件到 `dist/js/`
4. **minify-css**: 压缩 CSS 文件到 `dist/css/`
5. **copy-files**: 复制静态资源到 `dist/`

### 数据库迁移
- 迁移文件位于 `migration/` 目录,文件名格式: `描述.序号.cr`
- 使用 `mg` 库自动按序号执行迁移
- 迁移在 `Storage` 初始化时自动运行

## 开发注意事项

### Crystal 特性
- 使用 `MainFiber` 模式处理数据库和文件 I/O,避免阻塞主 Fiber
- 宏 `use_default` 实现单例模式
- 使用编译时标志 `{% if flag?(:release) %}` 区分开发/生产环境

### 前端开发
- 修改 `.less` 文件后需运行 `gulp less`
- 修改 JS 文件后开发模式直接刷新,生产环境需运行 `gulp babel`
- 页面使用 ECR 模板 (位于 `src/views/` 和各 Router 文件中)

### 测试
- 测试文件位于 `spec/` 目录
- 测试使用 Crystal 标准库的 `spec` 框架
- 运行 `crystal spec` 执行所有测试

### 代码风格
- 使用 `ameba` 作为 linter
- 遵循 Crystal 官方代码风格指南
- 运行 `crystal tool format` 自动格式化

## 配置文件
- 默认路径: `~/.config/mango/config.yml`
- 可通过 `-c` 参数指定自定义路径
- 主要配置项见 `src/config.cr` 和 README.md
