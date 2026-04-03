# 01. 项目概览与启动指南

## 一、项目简介

这是一个基于 Tauri v2 框架开发的桌面应用，用于下载禁漫天堂的漫画。

**技术架构:**
- 前端: Vue 3 + TypeScript + Vite + Naive UI + Pinia + UnoCSS
- 后端: Rust (Tauri)

## 二、程序入口

### 1. 前端入口

**文件路径:** `src/main.ts`

```typescript
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import 'virtual:uno.css'

const pinia = createPinia()
const app = createApp(App)

app.use(pinia).mount('#app')
```

**入口组件:** `src/App.vue` -> `src/AppContent.vue`

### 2. 后端入口

**文件路径:** `src-tauri/src/main.rs`

```rust
fn main() {
    jmcomic_downloader_lib::run()
}
```

**实际入口:** `src-tauri/src/lib.rs` 中的 `run()` 函数

```rust
pub fn run() {
    // 1. 使用 tauri_specta 收集所有命令和事件
    // 2. 在 debug 模式下自动生成 TypeScript 绑定
    // 3. 初始化 Tauri 应用
    // 4. 设置配置、JM客户端、下载管理器
    // 5. 启动应用
}
```

## 三、项目目录结构

```
jmcomic-downloader/
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
│   │   ├── ChapterPane.vue       # 章节详情页
│   │   ├── DownloadedPane/       # 已下载漫画
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
│       ├── commands.rs           # Tauri 命令(前端可调用的函数)
│       ├── config.rs             # 配置管理
│       ├── download_manager.rs   # 下载管理器
│       ├── errors.rs             # 错误处理
│       ├── events.rs             # 事件系统
│       ├── export.rs             # 导出功能
│       ├── jm_client.rs          # JM API 客户端
│       ├── lib.rs                # 库入口
│       ├── logger.rs             # 日志系统
│       ├── main.rs               # 应用入口
│       ├── utils.rs              # 工具函数
│       ├── responses/            # API 响应结构
│       └── types/                # Rust 类型定义
├── package.json
├── vite.config.ts
└── src-tauri/tauri.conf.json     # Tauri 配置
```

## 四、阅读顺序建议

### 第一阶段: 理解整体架构 (30分钟)
1. `src-tauri/tauri.conf.json` - 了解应用配置
2. `src/main.ts` - 前端入口
3. `src-tauri/src/lib.rs` - 后端入口和命令注册
4. `src/App.vue` 和 `src/AppContent.vue` - 主界面结构

### 第二阶段: 理解核心功能 (2小时)
1. `src-tauri/src/commands.rs` - 所有后端接口
2. `src-tauri/src/jm_client.rs` - JM API 封装
3. `src-tauri/src/download_manager.rs` - 下载逻辑
4. `src-tauri/src/config.rs` - 配置管理

### 第三阶段: 理解前端组件 (2小时)
1. `src/panes/SearchPane.vue` - 搜索功能
2. `src/panes/ChapterPane.vue` - 章节详情
3. `src/panes/DownloadedPane/DownloadedPane.vue` - 已下载
4. `src/components/ComicCard.vue` - 漫画卡片

### 第四阶段: 深入细节
1. `src-tauri/src/types/` - 类型定义
2. `src-tauri/src/responses/` - API 响应结构
3. `src-tauri/src/events.rs` - 事件系统
4. `src-tauri/src/export.rs` - 导出功能

## 五、主要启动和编译命令

### 前提条件
- Rust (最新稳定版)
- Node.js
- pnpm (版本 9.5.0)

### 常用命令

```bash
# 1. 安装依赖
pnpm install

# 2. 开发模式 (热重载)
pnpm tauri dev

# 3. 构建生产版本
pnpm tauri build

# 4. 仅构建前端
pnpm build

# 5. 仅启动前端开发服务器
pnpm dev
```

### 开发端口
- Vite 开发服务器: `localhost:5005`
- HMR (热模块替换): `localhost:1421`

## 六、主要配置信息

### 1. 前端配置

**vite.config.ts** - Vite 配置
```typescript
export default defineConfig({
  plugins: [
    vue(),
    UnoCSS(),
    vueJsx(),
    vueDevTools(),
    AutoImport({...}),      // 自动导入 Vue API
    Components({...}),      // 自动导入组件
  ],
  server: {
    port: 5005,             // 固定端口
    strictPort: true,
  },
})
```

**uno.config.ts** - UnoCSS 配置 (原子化 CSS)

**tsconfig.json** - TypeScript 配置

### 2. 后端配置

**src-tauri/tauri.conf.json** - Tauri 应用配置
```json
{
  "productName": "jmcomic-downloader",
  "version": "0.17.0",
  "identifier": "com.lanyeeee.jmcomic-downloader",
  "build": {
    "beforeDevCommand": "pnpm dev",
    "devUrl": "http://localhost:5005",
    "beforeBuildCommand": "pnpm build",
    "frontendDist": "../dist"
  },
  "app": {
    "windows": [{
      "title": "禁漫天堂下载器",
      "width": 800,
      "height": 600
    }]
  }
}
```

**src-tauri/Cargo.toml** - Rust 依赖配置

### 3. 应用配置 (运行时)

**Config 结构** (`src-tauri/src/config.rs`):
```rust
pub struct Config {
    pub username: String,                           // 用户名
    pub password: String,                           // 密码
    pub download_dir: PathBuf,                      // 下载目录
    pub export_dir: PathBuf,                        // 导出目录
    pub download_format: DownloadFormat,            // 下载格式
    pub dir_fmt: String,                            // 目录格式
    pub proxy_mode: ProxyMode,                      // 代理模式
    pub proxy_host: String,                         // 代理主机
    pub proxy_port: u16,                            // 代理端口
    pub enable_file_logger: bool,                   // 启用文件日志
    pub chapter_concurrency: usize,                 // 章节并发数
    pub chapter_download_interval_sec: u64,         // 章节下载间隔
    pub img_concurrency: usize,                     // 图片并发数
    pub img_download_interval_sec: u64,             // 图片下载间隔
    pub download_all_favorites_interval_sec: u64,   // 下载收藏夹间隔
    pub update_downloaded_comics_interval_sec: u64, // 更新库存间隔
    pub api_domain_mode: ApiDomainMode,             // API 域名模式
    pub custom_api_domain: String,                  // 自定义 API 域名
    pub should_download_cover: bool,                // 是否下载封面
}
```

**配置文件位置:** 
- Windows: `%APPDATA%/com.lanyeeee.jmcomic-downloader/config.json`
- 存储在应用数据目录中

### 4. API 域名配置

```rust
const API_DOMAIN_1: &str = "www.cdnzack.cc";
const API_DOMAIN_2: &str = "www.cdnhth.cc";
const API_DOMAIN_3: &str = "www.cdnhth.net";
const API_DOMAIN_4: &str = "www.cdnbea.net";
const API_DOMAIN_5: &str = "www.cdn-mspjmapiproxy.xyz";
```

### 5. 加密密钥 (JM API)

```rust
const APP_TOKEN_SECRET: &str = "18comicAPP";
const APP_TOKEN_SECRET_2: &str = "18comicAPPContent";
const APP_DATA_SECRET: &str = "185Hcomic3PAPP7R";
const APP_VERSION: &str = "2.0.13";
```

## 七、前后端通信方式

### 1. Commands (前端调用后端)

前端使用 `invoke` 调用后端命令:
```typescript
import { invoke } from '@tauri-apps/api/core'

const result = await invoke('command_name', { arg1: value1, arg2: value2 })
```

### 2. Events (后端推送前端)

后端使用 `Event::emit` 发送事件:
```rust
let _ = DownloadSpeedEvent { speed }.emit(&app);
```

前端使用 `listen` 监听事件:
```typescript
import { listen } from '@tauri-apps/api/event'

listen('event_name', (event) => {
  console.log(event.payload)
})
```

### 3. 类型绑定

通过 `tauri-specta` 自动生成 TypeScript 类型:
- 开发模式下自动导出到 `src/bindings.ts`
- 保证前后端类型一致
