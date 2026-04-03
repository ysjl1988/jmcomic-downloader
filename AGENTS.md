# JMComic Downloader - Agent Guide

## 项目概述

这是一个基于 **Tauri v2** 的桌面应用，用于下载禁漫天堂（18comic.vip / JMComic）的漫画。应用采用前后端分离架构：

- **前端**: Vue 3 + TypeScript + Vite
- **后端**: Rust (Tauri)

## 技术栈

### 前端
- **框架**: Vue 3.5.13 (Composition API)
- **语言**: TypeScript 5.2.2
- **构建工具**: Vite 5.3.1
- **UI 组件库**: Naive UI 2.40.1
- **状态管理**: Pinia 3.0.1
- **CSS 框架**: UnoCSS 0.63.0 (原子化 CSS)
- **图标**: Phosphor Icons Vue 2.2.1
- **选择组件**: viselect/vue 3.9.0

### 后端 (Rust)
- **框架**: Tauri v2
- **HTTP 客户端**: reqwest 0.12 + reqwest-retry + reqwest-middleware
- **图像处理**: image 0.25 (JPEG/PNG/WebP)
- **PDF 生成**: lopdf (fork 版本)
- **压缩**: zip 2.2.3
- **加密**: aes 0.8.4 + md5 + base64
- **异步运行时**: tokio 1.40
- **并行计算**: rayon 1.10
- **序列化**: serde + serde_json + yaserde
- **类型生成**: specta 2.0 + tauri-specta (自动生成 TS 绑定)

## 项目结构

```
├── src/                          # 前端代码
│   ├── components/               # 通用组件
│   │   ├── ComicCard.vue
│   │   ├── DownloadAllFavoriteButton.vue
│   │   ├── FloatLabelInput.vue
│   │   └── IconButton.vue
│   ├── dialogs/                  # 对话框组件
│   │   ├── AboutDialog.vue
│   │   ├── LogDialog.vue
│   │   ├── LoginDialog.vue
│   │   └── SettingsDialog.vue
│   ├── panes/                    # 主面板组件
│   │   ├── ChapterPane.vue       # 章节详情
│   │   ├── DownloadedPane/       # 已下载
│   │   ├── FavoritePane.vue      # 收藏夹
│   │   ├── ProgressesPane/       # 下载进度
│   │   ├── SearchPane.vue        # 搜索
│   │   └── WeeklyPane.vue        # 每周推荐
│   ├── bindings.ts               # 自动生成的 Tauri 绑定
│   ├── main.ts                   # 前端入口
│   ├── store.ts                  # Pinia store
│   └── types.ts                  # 前端类型定义
├── src-tauri/                    # Rust 后端代码
│   └── src/
│       ├── commands.rs           # Tauri 命令定义
│       ├── config.rs             # 配置管理
│       ├── download_manager.rs   # 下载管理器
│       ├── errors.rs             # 错误处理
│       ├── events.rs             # 事件系统
│       ├── export.rs             # 导出功能 (PDF/CBZ)
│       ├── extensions.rs         # 扩展方法
│       ├── jm_client.rs          # JM API 客户端
│       ├── lib.rs                # 库入口
│       ├── logger.rs             # 日志系统
│       ├── main.rs               # 应用入口
│       ├── utils.rs              # 工具函数
│       ├── responses/            # API 响应结构
│       └── types/                # Rust 类型定义
├── package.json
├── vite.config.ts
├── tsconfig.json
├── uno.config.ts                 # UnoCSS 配置
└── components.d.ts               # 组件自动导入声明
```

## 开发命令

```bash
# 安装依赖
pnpm install

# 开发模式 (带热重载)
pnpm tauri dev

# 构建生产版本
pnpm tauri build

# 仅构建前端
pnpm build
```

## 架构说明

### 前后端通信

使用 **Tauri Commands** 和 **Events** 进行通信：

- **Commands**: 前端调用 Rust 函数（如搜索、登录、下载）
- **Events**: Rust 向前端推送实时更新（如下载进度、日志）

通过 `tauri-specta` 自动生成 TypeScript 绑定，确保类型安全。

### 核心模块

1. **JmClient**: 处理与 JM API 的所有交互（搜索、获取章节、登录等）
2. **DownloadManager**: 管理并发下载任务，支持暂停/恢复/取消
3. **Export**: 支持导出为 PDF 和 CBZ 格式
4. **Config**: 管理应用配置（下载路径、代理设置等）

### 状态管理

- 使用 **Pinia** 管理前端全局状态
- Rust 端使用 `parking_lot::RwLock` 管理共享状态

## 代码规范

### 前端
- 使用 Vue 3 Composition API (`<script setup>` 语法)
- 使用 TypeScript 严格模式
- 组件名使用 PascalCase
- 使用自动导入（`unplugin-auto-import` 和 `unplugin-vue-components`）

### Rust
- 使用 `anyhow` 进行错误处理
- 异步函数使用 `tokio`
- 重度使用 `serde` 进行序列化

## 特殊注意事项

1. **类型绑定**: 修改 Rust 命令或事件后，运行 `pnpm tauri dev` 会自动更新 `src/bindings.ts`

2. **开发端口**: Vite 开发服务器使用固定端口 5005

3. **图像处理**: 使用 `rayon` 进行并行图像解码/编码，可能导致高 CPU 占用

4. **PDF 生成**: 使用 fork 版本的 `lopdf` 以支持 WebP 图像嵌入

5. **日志**: 日志存储在应用数据目录，可通过 `LogDialog` 查看

## 构建配置

Release 配置启用以下优化：
- `strip = true`: 移除符号
- `lto = true`: 链接时优化
- `codegen-units = 1`: 单代码生成单元
- `panic = "abort"`:  panic 时中止

## 依赖管理

- 使用 **pnpm** 作为包管理器（版本 9.5.0）
- 前端依赖在 `package.json`
- Rust 依赖在 `src-tauri/Cargo.toml`
