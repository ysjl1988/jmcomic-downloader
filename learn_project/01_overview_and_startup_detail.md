# 01. 项目概览与启动指南 - 详细版

本文档是《01. 项目概览与启动指南》的完整详细版，包含从第一章到第六章的深入解释。

***

## 目录

1. [项目简介详解](#1-项目简介详解)
2. [程序入口详解](#2-程序入口详解)
3. [项目目录结构详解](#3-项目目录结构详解)
4. [阅读顺序建议](#4-阅读顺序建议)
5. [启动和编译命令详解](#5-启动和编译命令详解)
6. [主要配置信息详解](#6-主要配置信息详解)

***

## 1. 项目简介详解

### 1.1 什么是 Tauri？

Tauri 是一个用 Rust 编写的桌面应用开发框架，类似于 Electron，但有以下区别：

| 特性   | Electron | Tauri  |
| ---- | -------- | ------ |
| 后端语言 | Node.js  | Rust   |
| 包大小  | 100MB+   | 3-10MB |
| 内存占用 | 较高       | 较低     |
| 启动速度 | 较慢       | 快      |
| 前端技术 | Web 技术   | Web 技术 |

**Tauri 的工作原理：**

- 前端使用 Web 技术（HTML/CSS/JS），可以使用任何前端框架
- 后端使用 Rust 处理系统调用、文件操作、网络请求等
- 使用操作系统原生 WebView 渲染（Windows: WebView2, macOS: WebKit, Linux: WebKitGTK）

### 1.2 前端技术栈说明

| 技术                 | 版本     | 作用                                   |
| ------------------ | ------ | ------------------------------------ |
| **Vue 3**          | 3.5.13 | 前端 UI 框架，用于构建用户界面，使用 Composition API |
| **TypeScript**     | 5.2.2  | JavaScript 的超集，增加类型检查，减少运行时错误        |
| **Vite**           | 5.3.1  | 前端构建工具，开发时提供热更新（HMR），生产构建速度快         |
| **Naive UI**       | 2.40.1 | Vue 3 的组件库，提供按钮、对话框、输入框等现成组件         |
| **Pinia**          | 3.0.1  | Vue 的官方状态管理库，管理全局数据如下载列表、用户状态        |
| **UnoCSS**         | 0.63.0 | 原子化 CSS 框架，通过类名快速编写样式                |
| **Phosphor Icons** | 2.2.1  | 图标库，提供各种矢量图标                         |
| **viselect/vue**   | 3.9.0  | 选择组件，支持框选、多选等交互                      |

**Vue 3 Composition API 示例：**

```vue
<script setup lang="ts">
import { ref, computed, onMounted } from 'vue'

// 响应式状态
const count = ref(0)

// 计算属性
const doubleCount = computed(() => count.value * 2)

// 方法
function increment() {
  count.value++
}

// 生命周期钩子
onMounted(() => {
  console.log('组件挂载了')
})
</script>
```

### 1.3 后端技术栈说明

| 技术          | 版本           | 作用                      |
| ----------- | ------------ | ----------------------- |
| **Rust**    | 2021 edition | 系统编程语言，性能极高，内存安全        |
| **Tauri**   | 2.x          | 提供桌面应用壳层，处理窗口、系统调用等     |
| **reqwest** | 0.12         | HTTP 客户端，用于请求 JM API    |
| **tokio**   | 1.40         | 异步运行时，处理并发任务            |
| **serde**   | 1.x          | 序列化/反序列化库，处理 JSON       |
| **specta**  | 2.0-rc       | 类型生成，自动生成 TypeScript 绑定 |

**Rust 的优势：**

- 零成本抽象：高级语法不损失性能
- 内存安全：编译时检查，无内存泄漏、野指针
- 并发安全：所有权系统避免数据竞争
- 性能：与 C/C++ 相当，远高于 Node.js

### 1.4 前后端如何通信？

**两种方式：**

1. **Commands（命令）**：前端调用后端函数（请求/响应模式）
2. **Events（事件）**：后端向前端推送消息（发布/订阅模式）

```
前端 (Vue 3)                    后端 (Rust)
    |                               |
    |---- invoke('command') ------>|
    |<--- return result ----------|
    |                               |
    |<---- Event::emit() ---------|
    |    (download progress)        |
```

***

## 2. 程序入口详解

### 2.1 前端入口 main.ts

**文件路径：** `src/main.ts`

```typescript
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'
import 'virtual:uno.css'

const pinia = createPinia()
const app = createApp(App)

app.use(pinia).mount('#app')
```

**逐行解释：**

| 行 | 代码                                    | 作用                         |
| - | ------------------------------------- | -------------------------- |
| 1 | `import { createApp } from 'vue'`     | 从 Vue 导入创建应用的函数            |
| 2 | `import { createPinia } from 'pinia'` | 导入 Pinia 状态管理              |
| 3 | `import App from './App.vue'`         | 导入根组件                      |
| 4 | `import 'virtual:uno.css'`            | 导入 UnoCSS 的虚拟模块（原子化 CSS）   |
| 6 | `const pinia = createPinia()`         | 创建 Pinia 实例                |
| 7 | `const app = createApp(App)`          | 以 App.vue 为根组件创建 Vue 应用    |
| 9 | `app.use(pinia)`                      | 注册 Pinia 插件                |
| 9 | `.mount('#app')`                      | 将应用挂载到 HTML 中 id 为 app 的元素 |

**对应的 HTML：**

```html
<!-- index.html -->
<body>
  <div id="app"></div>  <!-- Vue 应用挂载到这里 -->
  <script type="module" src="/src/main.ts"></script>
</body>
```

**入口组件链：**

```
main.ts
  └── App.vue (根组件，整体布局)
       └── AppContent.vue (实际内容区)
            ├── SearchPane.vue (搜索面板)
            ├── FavoritePane.vue (收藏夹面板)
            ├── DownloadedPane.vue (已下载面板)
            ├── WeeklyPane.vue (每周推荐面板)
            └── ChapterPane.vue (章节详情面板)
```

### 2.2 后端入口详解

**main.rs - 程序入口：**

**文件路径：** `src-tauri/src/main.rs`

```rust
fn main() {
    jmcomic_downloader_lib::run()
}
```

**为什么这么简单？**

- 这是 Rust 的惯例：main.rs 只作为入口，实际逻辑放在 lib.rs
- 这样 lib.rs 可以被测试代码复用
- 也支持作为库被其他项目引用

**lib.rs - 实际入口 run() 函数：**

**文件路径：** `src-tauri/src/lib.rs`

```rust
pub fn run() {
    // 1. 创建 Builder，收集所有 Commands 和 Events
    let builder = tauri_specta::Builder::<Wry>::new()
        .commands(tauri_specta::collect_commands![
            greet, get_config, save_config, login, search, get_comic, ...
        ])
        .events(tauri_specta::collect_events![
            DownloadSpeedEvent, DownloadTaskEvent, ...
        ]);

    // 2. 开发模式下自动生成 TypeScript 绑定
    #[cfg(debug_assertions)]
    builder.export(..., "../src/bindings.ts");

    // 3. 构建 Tauri 应用
    tauri::Builder::default()
        .plugin(tauri_plugin_dialog::init())     // 文件对话框插件
        .plugin(tauri_plugin_opener::init())     // 打开文件插件
        .invoke_handler(builder.invoke_handler()) // 注册 Commands
        .setup(move |app| {                      // 初始化
            builder.mount_events(app);            // 注册 Events
            
            // 创建应用数据目录
            let app_data_dir = app.path().app_data_dir()?;
            std::fs::create_dir_all(&app_data_dir)?;
            
            // 初始化全局状态
            app.manage(RwLock::new(Config::new(app.handle())?));
            app.manage(JmClient::new(app.handle().clone()));
            app.manage(DownloadManager::new(app.handle().clone()));
            
            logger::init(app.handle())?;
            Ok(())
        })
        .run(generate_context())
        .expect("error while running tauri application");
}
```

**初始化流程详解：**

| 步骤 | 代码                             | 作用                        |
| -- | ------------------------------ | ------------------------- |
| 1  | `Builder::new()`               | 创建 tauri\_specta Builder  |
| 2  | `collect_commands!`            | 收集 27 个前端可调用的 Command     |
| 3  | `collect_events!`              | 收集 8 个可发送的事件类型            |
| 4  | `export(...)`                  | Debug 模式下生成 TypeScript 绑定 |
| 5  | `Builder::default()`           | 创建 Tauri 应用构建器            |
| 6  | `plugin(dialog)`               | 注册文件选择对话框插件               |
| 7  | `plugin(opener)`               | 注册打开文件管理器插件               |
| 8  | `invoke_handler(...)`          | 注册 Commands 处理器           |
| 9  | `setup(...)`                   | 应用初始化闭包                   |
| 10 | `mount_events(...)`            | 注册 Events 系统              |
| 11 | `create_dir_all(...)`          | 创建应用数据目录                  |
| 12 | `app.manage(config)`           | 注册配置状态                    |
| 13 | `app.manage(jm_client)`        | 注册 JM 客户端                 |
| 14 | `app.manage(download_manager)` | 注册下载管理器                   |
| 15 | `logger::init(...)`            | 初始化日志系统                   |
| 16 | `run(...)`                     | 启动应用，进入事件循环               |

***

## 3. 项目目录结构详解

### 3.1 前端目录 src/

```
src/
├── components/          # 通用组件（可复用，不依赖业务）
│   ├── ComicCard.vue       # 漫画卡片：显示封面、标题、作者
│   ├── DownloadAllFavoriteButton.vue  # 下载收藏夹按钮
│   ├── FloatLabelInput.vue  # 浮动标签输入框
│   └── IconButton.vue       # 图标按钮
│
├── dialogs/            # 对话框组件（弹窗）
│   ├── AboutDialog.vue      # 关于对话框
│   ├── LogDialog.vue        # 日志查看对话框
│   ├── LoginDialog.vue      # 登录对话框
│   └── SettingsDialog.vue   # 设置对话框
│
├── panes/              # 主面板组件（应用的主要功能区）
│   ├── ChapterPane.vue      # 章节详情页：显示漫画章节列表
│   ├── DownloadedPane/      # 已下载漫画管理
│   │   ├── DownloadedPane.vue
│   │   └── components/
│   ├── FavoritePane.vue     # 收藏夹：登录用户的收藏
│   ├── ProgressesPane/      # 下载进度面板
│   ├── SearchPane.vue       # 搜索页：关键词搜索
│   └── WeeklyPane.vue       # 每周推荐：编辑推荐
│
├── bindings.ts         # 自动生成的 Tauri 绑定（类型定义）
├── main.ts            # 前端入口
├── store.ts           # Pinia 状态管理
└── types.ts           # 前端自定义类型
```

**组件分类说明：**

| 目录            | 说明            | 示例                      |
| ------------- | ------------- | ----------------------- |
| `components/` | 纯展示组件，无业务逻辑   | ComicCard 只接收 props 显示  |
| `dialogs/`    | 模态对话框，覆盖在主界面上 | SettingsDialog 弹出设置窗口   |
| `panes/`      | 主功能面板，通过标签切换  | SearchPane、FavoritePane |

### 3.2 后端目录 src-tauri/src/

```
src-tauri/src/
├── commands.rs         # Tauri Commands：前端可调用的 27 个函数
├── config.rs          # 配置管理：Config 结构体，读写 config.json
├── download_manager.rs # 下载管理器：并发控制、任务调度
├── errors.rs          # 错误处理：CommandError 等
├── events.rs          # 事件定义：DownloadTaskEvent 等 8 个事件
├── export.rs          # 导出功能：生成 CBZ 和 PDF
├── extensions.rs      # 扩展方法：给现有类型添加方法
├── jm_client.rs       # JM API 客户端：登录、搜索、下载图片
├── lib.rs             # 库入口：run() 函数
├── logger.rs          # 日志系统
├── main.rs            # 应用入口
├── utils.rs           # 工具函数：MD5、解密等
│
├── responses/         # JM API 响应数据结构
│   ├── get_comic_resp_data.rs
│   ├── search_resp.rs
│   └── ...
│
└── types/             # Rust 类型定义
    ├── comic.rs
    ├── chapter_info.rs
    └── ...
```

**核心文件说明：**

| 文件                    | 职责      | 关键内容                       |
| --------------------- | ------- | -------------------------- |
| `commands.rs`         | 前端接口    | 27 个 #\[tauri::command] 函数 |
| `jm_client.rs`        | API 客户端 | 封装 JM 的所有 HTTP 请求          |
| `download_manager.rs` | 下载控制    | 并发下载、暂停/恢复/取消              |
| `config.rs`           | 配置管理    | Config 结构体、读写配置            |
| `export.rs`           | 文件导出    | CBZ、PDF 生成                 |
| `events.rs`           | 事件定义    | 8 个事件类型                    |

***

## 4. 阅读顺序建议

### 4.1 第一阶段：理解整体架构（30分钟）

**目标：** 了解项目是怎么跑起来的

| 顺序 | 文件                          | 看什么           |
| -- | --------------------------- | ------------- |
| 1  | `src-tauri/tauri.conf.json` | 应用基本信息、窗口配置   |
| 2  | `src/main.ts`               | Vue 应用如何创建和挂载 |
| 3  | `src-tauri/src/lib.rs`      | 初始化流程、模块注册    |
| 4  | `src/App.vue`               | 整体布局结构        |
| 5  | `src/AppContent.vue`        | 标签页切换逻辑       |

### 4.2 第二阶段：理解核心功能（2小时）

**目标：** 理解主要业务逻辑

| 顺序 | 文件                                  | 看什么           |
| -- | ----------------------------------- | ------------- |
| 1  | `src-tauri/src/commands.rs`         | 所有后端接口，按功能分组看 |
| 2  | `src-tauri/src/jm_client.rs`        | API 封装、加密解密   |
| 3  | `src-tauri/src/download_manager.rs` | 下载控制逻辑        |
| 4  | `src-tauri/src/config.rs`           | 配置管理          |

**Commands 分组：**

- 配置：`get_config`, `save_config`
- 用户：`login`, `get_user_profile`
- 漫画：`search`, `get_comic`, `download_comic`
- 下载任务：`create/pause/resume/cancel_download_task`

### 4.3 第三阶段：理解前端组件（2小时）

**目标：** 理解用户界面实现

| 顺序 | 文件                                            | 看什么     |
| -- | --------------------------------------------- | ------- |
| 1  | `src/panes/SearchPane.vue`                    | 搜索功能    |
| 2  | `src/panes/ChapterPane.vue`                   | 章节选择、下载 |
| 3  | `src/panes/DownloadedPane/DownloadedPane.vue` | 已下载管理   |
| 4  | `src/components/ComicCard.vue`                | 漫画卡片展示  |

### 4.4 第四阶段：深入细节（按需）

| 主题     | 文件                         |
| ------ | -------------------------- |
| 类型定义   | `src-tauri/src/types/`     |
| API 响应 | `src-tauri/src/responses/` |
| 事件系统   | `src-tauri/src/events.rs`  |
| 导出功能   | `src-tauri/src/export.rs`  |

***

## 5. 启动和编译命令详解

### 5.1 前提条件

| 工具      | 最低版本  | 检查命令              |
| ------- | ----- | ----------------- |
| Rust    | 1.70+ | `rustc --version` |
| Node.js | 18+   | `node --version`  |
| pnpm    | 9.x   | `pnpm --version`  |

**安装命令：**

```bash
# 安装 Rust
# 下载 https://rustup.rs/ 的运行器，或执行：
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 安装 pnpm
npm install -g pnpm
```

### 5.2 命令详解

#### pnpm install - 安装依赖

```bash
pnpm install
```

**作用：**

- 下载前端依赖到 `node_modules/`
- 根据 `package.json` 和 `pnpm-lock.yaml` 安装精确版本
- 只在第一次克隆或依赖变更后需要执行

\*\*耗时：\*\*1-3 分钟

***

#### pnpm tauri dev - 开发模式

```bash
pnpm tauri dev
```

**作用：**

- 启动 Vite 开发服务器（前端热重载）
- 编译并启动 Rust 后端
- 打开应用窗口

**启动过程：**

```
1. 启动 Vite 服务器 @ localhost:5005
2. 编译 Rust 代码（首次较慢，3-10分钟）
3. 启动 Tauri 应用
4. 加载前端页面
5. 应用窗口出现
```

**开发特性：**

- 前端代码修改 → 自动热更新（秒级）
- Rust 代码修改 → 需要重新编译（较慢）
- 自动生成 TypeScript 绑定

**耗时：**

- 首次：3-10 分钟（编译 Rust）
- 后续：10-30 秒（利用缓存）

***

#### pnpm tauri build - 构建生产版本

```bash
pnpm tauri build
```

**作用：**

- 构建生产环境可分发包

**构建流程：**

```
1. pnpm build              # 构建前端到 dist/
2. 编译 Rust (Release)     # 优化编译，生成二进制
3. 打包资源                # 将前端资源嵌入二进制
4. 生成安装包              # .exe, .msi 等
```

**输出位置：**

```
src-tauri/target/release/bundle/
├── msi/           # Windows 安装包 (.msi)
├── nsis/          # Windows 安装包 (.exe)
└── ...
```

\*\*耗时：\*\*5-15 分钟

***

#### pnpm build - 仅构建前端

```bash
pnpm build
```

**作用：**

- 只构建前端代码到 `dist/` 目录
- 不编译 Rust 后端

**用途：**

- 测试前端生产构建
- CI/CD 流程

***

#### pnpm dev - 仅启动前端

```bash
pnpm dev
```

**作用：**

- 只启动 Vite 开发服务器
- 不启动 Rust 后端

**用途：**

- 纯前端开发（mock 数据）
- 快速预览前端修改

***

### 5.3 端口说明

| 端口   | 用途            | 配置位置             |
| ---- | ------------- | ---------------- |
| 5005 | Vite 开发服务器    | `vite.config.ts` |
| 1421 | HMR WebSocket | `vite.config.ts` |

**端口冲突解决：**

- Vite 配置设置了 `strictPort: true`
- 如果 5005 被占用会报错
- 关闭占用端口的程序，或修改两个配置文件保持一致

***

## 6. 主要配置信息详解

### 6.1 前端配置

#### vite.config.ts

**关键配置：**

```typescript
export default defineConfig({
  plugins: [
    vue(),              // Vue 3 支持
    UnoCSS(),           // 原子化 CSS
    vueJsx(),           // JSX 支持
    vueDevTools(),      // 开发者工具
    AutoImport({        // 自动导入 Vue API
      imports: ['vue', {'naive-ui': ['useDialog', ...]}]
    }),
    Components({        // 自动导入组件
      resolvers: [NaiveUiResolver()]
    }),
  ],
  server: {
    port: 5005,
    strictPort: true,   // 端口被占用时报错
    watch: {
      ignored: ['**/src-tauri/**']  // 忽略 Rust 目录
    }
  }
})
```

#### uno.config.ts

UnoCSS 原子化 CSS 配置，定义实用类规则。

**示例：**

```html
<div class="flex items-center justify-center p-4">
  <!-- flex 布局，水平垂直居中，内边距 1rem -->
</div>
```

#### tsconfig.json

TypeScript 编译配置，包含路径别名：

```json
{
  "compilerOptions": {
    "paths": {
      "@/*": ["./src/*"]
    }
  }
}
```

使用：`import X from '@/components/X'`

***

### 6.2 后端配置

#### tauri.conf.json

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

**关键字段：**

| 字段                 | 说明             |
| ------------------ | -------------- |
| `identifier`       | 唯一标识符，决定数据目录名称 |
| `devUrl`           | 开发时加载的前端地址     |
| `beforeDevCommand` | 开发前执行的命令       |
| `windows`          | 窗口配置（大小、标题等）   |

#### Cargo.toml

**Release 优化配置：**

```toml
[profile.release]
strip = true           # 移除符号表，减小体积
lto = true             # 链接时优化
codegen-units = 1      # 单编译单元，更激进优化
panic = "abort"        # panic 时直接中止
```

***

### 6.3 应用运行时配置

#### Config 结构体

**文件：** `src-tauri/src/config.rs`

```rust
pub struct Config {
    // 账号
    pub username: String,
    pub password: String,
    
    // 路径
    pub download_dir: PathBuf,
    pub export_dir: PathBuf,
    pub dir_fmt: String,
    
    // 下载设置
    pub download_format: DownloadFormat,
    pub should_download_cover: bool,
    
    // 并发控制
    pub chapter_concurrency: usize,      // 默认: 3
    pub img_concurrency: usize,          // 默认: 20
    
    // 防封间隔（秒）
    pub chapter_download_interval_sec: u64,
    pub img_download_interval_sec: u64,
    pub download_all_favorites_interval_sec: u64,
    pub update_downloaded_comics_interval_sec: u64,
    
    // 代理
    pub proxy_mode: ProxyMode,           // System/NoProxy/Custom
    pub proxy_host: String,
    pub proxy_port: u16,
    
    // 其他
    pub enable_file_logger: bool,
    pub api_domain_mode: ApiDomainMode,
    pub custom_api_domain: String,
}
```

#### 配置项说明

| 配置项                   | 默认值         | 说明           |
| --------------------- | ----------- | ------------ |
| `download_dir`        | `应用数据/漫画下载` | 下载保存位置       |
| `export_dir`          | `应用数据/漫画导出` | CBZ/PDF 导出位置 |
| `chapter_concurrency` | 3           | 同时下载章节数      |
| `img_concurrency`     | 20          | 同时下载图片数      |
| `proxy_mode`          | `System`    | 代理模式         |
| `api_domain_mode`     | `Domain2`   | API 域名选择     |

#### 配置文件位置

| 平台      | 路径                                                      |
| ------- | ------------------------------------------------------- |
| Windows | `%APPDATA%\com.lanyeeee.jmcomic-downloader\config.json` |
| Linux   | `~/.local/share/.../config.json`                        |
| macOS   | `~/Library/Application Support/.../config.json`         |

***

### 6.4 API 域名与加密

#### API 域名

```rust
const API_DOMAIN_1: &str = "www.cdnzack.cc";
const API_DOMAIN_2: &str = "www.cdnhth.cc";
const API_DOMAIN_3: &str = "www.cdnhth.net";
const API_DOMAIN_4: &str = "www.cdnbea.net";
const API_DOMAIN_5: &str = "www.cdn-mspjmapiproxy.xyz";
```

#### 加密密钥

```rust
const APP_TOKEN_SECRET: &str = "18comicAPP";
const APP_TOKEN_SECRET_2: &str = "18comicAPPContent";
const APP_DATA_SECRET: &str = "185Hcomic3PAPP7R";
const APP_VERSION: &str = "2.0.13";
```

**加密流程：**

1. 生成时间戳 `ts`
2. 计算 `token = md5(ts + SECRET)`
3. 请求头带上 `token` 和 `tokenparam`
4. 服务器返回 AES-256-ECB 加密数据
5. 用 `md5(ts + APP_DATA_SECRET)` 解密

***

### 6.5 前后端通信机制

#### Commands（命令）

```typescript
// 前端调用
const result = await invoke('get_comic', { aid: 12345 })
```

```rust
// 后端实现
#[tauri::command]
pub async fn get_comic(app: AppHandle, aid: i64) -> CommandResult<Comic> {
    // ... 获取漫画
    Ok(comic)
}
```

#### Events（事件）

```rust
// 后端发送
DownloadSpeedEvent { speed: "1.5MB/s" }.emit(&app)
```

```typescript
// 前端监听
listen('download-speed-event', (e) => {
    console.log(e.payload.speed)
})
```

#### 类型绑定

通过 `tauri-specta` 自动生成 `src/bindings.ts`，保证前后端类型一致。

***

## 附录：pnpm vs npm

| 特性       | npm      | pnpm            |
| -------- | -------- | --------------- |
| 存储方式     | 每个项目复制一份 | 全局存储 + 硬链接      |
| 磁盘占用     | 大        | 小（多个项目共享）       |
| 安装速度     | 慢        | 快（二次安装几乎瞬间）     |
| 依赖严格性    | 宽松（幽灵依赖） | 严格（只能访问显式声明的依赖） |
| Monorepo | 需要额外工具   | 原生支持            |

**为什么本项目选择 pnpm？**

- 节省磁盘空间
- 安装速度快
- 依赖更严格，避免隐性 bug
- 现代前端项目的趋势

***

**本文档结束。建议接下来阅读《02\_backend\_api\_reference.md》了解所有后端接口的详细信息。**
