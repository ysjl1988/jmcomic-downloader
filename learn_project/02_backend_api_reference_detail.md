# 02. 后端接口列表 - 详细版

本文档是《02. 后端接口列表》的完整详细版，对所有 23 个 Tauri Commands 进行深入解释。

---

## 目录

1. [系统/配置接口详解](#1-系统配置接口详解)
2. [用户/认证接口详解](#2-用户认证接口详解)
3. [漫画搜索/获取接口详解](#3-漫画搜索获取接口详解)
4. [收藏夹接口详解](#4-收藏夹接口详解)
5. [每周推荐接口详解](#5-每周推荐接口详解)
6. [下载任务接口详解](#6-下载任务接口详解)
7. [导出接口详解](#7-导出接口详解)
8. [同步/工具接口详解](#8-同步工具接口详解)
9. [事件系统详解](#9-事件系统详解)

---

## 1. 系统/配置接口详解

这类接口主要用于应用的**基础功能**，如读取配置、保存配置、测试连接等。

### 1.1 greet - 测试问候函数

**用途：** 最简单的测试接口，确认前后端通信正常

**代码位置：** `src-tauri/src/commands.rs` 第 27-31 行

```rust
#[tauri::command]
#[specta::specta]
pub fn greet(name: &str) -> String {
    format!("Hello, {}! You've been greeted from Rust!", name)
}
```

**入参：**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| name | `&str` | 用户名 |

**出参：**
| 类型 | 说明 |
|------|------|
| `String` | 问候语 |

**前端调用示例：**
```typescript
const message = await invoke('greet', { name: '张三' })
console.log(message)  // "Hello, 张三! You've been greeted from Rust!"
```

**为什么需要这个接口？**
- 开发初期测试 Tauri 是否正常工作
- 确认 TypeScript 绑定生成正确
- 学习示例：最简单的 Command 写法

---

### 1.2 get_config - 获取配置

**用途：** 读取用户保存的所有配置

**代码位置：** `src-tauri/src/commands.rs` 第 33-38 行

```rust
#[tauri::command]
#[specta::specta]
pub fn get_config(app: AppHandle) -> Config {
    app.get_config().read().clone()
}
```

**特点：**
- **入参：** 无（通过 `AppHandle` 自动获取）
- **出参：** `Config` 结构体（包含所有配置字段）

**Config 包含的字段（19 个）：**

| 字段 | 类型 | 说明 |
|------|------|------|
| `username` | String | 登录用户名 |
| `password` | String | 登录密码（明文存储） |
| `download_dir` | PathBuf | 漫画下载目录 |
| `export_dir` | PathBuf | 导出目录（CBZ/PDF） |
| `download_format` | DownloadFormat | 图片保存格式（Jpg/Png/Webp/Original） |
| `dir_fmt` | String | 目录命名模板 |
| `proxy_mode` | ProxyMode | 代理模式（System/NoProxy/Custom） |
| `proxy_host` | String | 代理主机地址 |
| `proxy_port` | u16 | 代理端口 |
| `enable_file_logger` | bool | 是否启用文件日志 |
| `chapter_concurrency` | usize | 同时下载章节数 |
| `chapter_download_interval_sec` | u64 | 章节下载间隔（秒） |
| `img_concurrency` | usize | 同时下载图片数 |
| `img_download_interval_sec` | u64 | 图片下载间隔（秒） |
| `download_all_favorites_interval_sec` | u64 | 批量下载收藏夹间隔 |
| `update_downloaded_comics_interval_sec` | u64 | 更新库存间隔 |
| `api_domain_mode` | ApiDomainMode | API 域名选择 |
| `custom_api_domain` | String | 自定义 API 域名 |
| `should_download_cover` | bool | 是否下载封面 |

**执行流程：**
```
调用 get_config()
    │
    ▼
通过 AppHandle 获取 RwLock<Config>
    │
    ▼
加读锁获取配置引用
    │
    ▼
clone() 复制一份配置
    │
    ▼
返回给前端
```

**为什么使用 `RwLock`？**
- 允许多个读操作并发（多个命令同时读配置）
- 写操作会阻塞读操作（save_config 时）

**前端使用场景：**
```typescript
// 应用启动时读取配置初始化界面
const config = await invoke('get_config')
settingsStore.init(config)

// 设置对话框打开时显示当前配置
const currentConfig = await invoke('get_config')
form.setValues(currentConfig)
```

---

### 1.3 save_config - 保存配置

**用途：** 保存用户修改的配置

**代码位置：** `src-tauri/src/commands.rs` 第 40-81 行

```rust
#[tauri::command(async)]
#[specta::specta]
#[allow(clippy::needless_pass_by_value)]
pub fn save_config(app: AppHandle, config: Config) -> CommandResult<()> {
    let config_state = app.get_config();
    let jm_client = app.get_jm_client();

    // 检查代理是否变更
    let proxy_changed = {
        let config_state = config_state.read();
        config_state.proxy_mode != config.proxy_mode
            || config_state.proxy_host != config.proxy_host
            || config_state.proxy_port != config.proxy_port
    };

    // 检查日志配置是否变更
    let file_logger_changed = config_state.read().enable_file_logger != config.enable_file_logger;

    // 保存配置到内存和文件
    {
        let mut config_state = config_state.write();
        *config_state = config;
        config_state.save(&app)?;
    }

    // 如果代理变更，重新加载 HTTP 客户端
    if proxy_changed {
        jm_client.reload_client();
    }

    // 如果日志配置变更，重新加载日志系统
    if file_logger_changed {
        if config.enable_file_logger {
            logger::reload_file_logger()?;
        } else {
            logger::disable_file_logger()?;
        }
    }

    Ok(())
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `config` | Config | 新的配置对象（完整的 Config 结构体） |

**出参：**
- 成功：`Ok(())`（空元组，表示无返回值）
- 失败：`Err(CommandError)`

**副作用详解：**

**1. 代理配置变更 → 重新加载 HTTP 客户端**
```rust
if proxy_changed {
    jm_client.reload_client();
}
```
- 创建新的 `reqwest::Client`
- 使用新的代理设置
- 保持 Cookie 不变（登录状态不丢失）

**2. 日志配置变更 → 重新加载日志系统**
```rust
if file_logger_changed {
    if config.enable_file_logger {
        logger::reload_file_logger();   // 启用文件日志
    } else {
        logger::disable_file_logger();  // 禁用文件日志
    }
}
```

**执行流程：**
```
调用 save_config(new_config)
    │
    ▼
获取当前配置和 JM 客户端
    │
    ▼
检查代理和日志配置是否变更
    │
    ▼
加写锁更新配置
    │
    ▼
保存到 config.json 文件
    │
    ▼
如果代理变更 → reload_client()
    │
    ▼
如果日志变更 → reload/disable 日志
    │
    ▼
返回 Ok(())
```

**前端调用示例：**
```typescript
// 修改下载目录
async function updateDownloadDir(newDir: string) {
  const currentConfig = await invoke('get_config')
  const newConfig = { ...currentConfig, downloadDir: newDir }
  await invoke('save_config', { config: newConfig })
  message.success('保存成功')
}
```

---

### 1.4 get_logs_dir_size - 获取日志目录大小

**用途：** 获取日志文件夹的总大小（用于设置界面的"清理日志"功能）

**代码位置：** `src-tauri/src/commands.rs` 第 679-695 行

```rust
#[allow(clippy::needless_pass_by_value)]
#[tauri::command(async)]
#[specta::specta]
pub fn get_logs_dir_size(app: AppHandle) -> CommandResult<u64> {
    let logs_dir = logger::logs_dir(&app)
        .context("获取日志目录失败")?;
    
    let logs_dir_size = std::fs::read_dir(&logs_dir)?
        .filter_map(Result::ok)
        .filter_map(|entry| entry.metadata().ok())
        .map(|metadata| metadata.len())
        .sum::<u64>();
    
    tracing::debug!("获取日志目录大小成功");
    Ok(logs_dir_size)
}
```

**入参：** 无

**出参：**
| 类型 | 说明 |
|------|------|
| `CommandResult<u64>` | 日志目录大小（字节） |

**执行流程：**
```
调用 get_logs_dir_size()
    │
    ▼
获取日志目录路径
    │
    ▼
读取目录下所有文件
    │
    ▼
累加每个文件的大小
    │
    ▼
返回总字节数
```

**使用场景：**
```typescript
// 设置页面显示日志大小
async function loadLogsSize() {
  const bytes = await invoke('get_logs_dir_size')
  const mb = (bytes / 1024 / 1024).toFixed(2)
  setLogsSize(`${mb} MB`)
}

// 点击"清理日志"按钮后刷新
async function clearLogs() {
  await invoke('clear_logs')  // 假设有这个命令
  await loadLogsSize()  // 重新获取大小
}
```

---

## 2. 用户/认证接口详解

这类接口处理**用户登录和身份信息**。

### 2.1 login - 用户登录

**用途：** 使用用户名和密码登录，获取用户信息并保存登录状态（Cookie）

**代码位置：** `src-tauri/src/commands.rs` 第 83-98 行

```rust
#[tauri::command]
#[specta::specta]
pub async fn login(
    app: AppHandle,
    username: String,
    password: String,
) -> CommandResult<GetUserProfileRespData> {
    let jm_client = app.get_jm_client();

    let user_profile = jm_client
        .login(&username, &password)
        .await
        .map_err(|err| CommandError::from("登录失败", err))?;

    Ok(user_profile)
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `username` | String | 用户名（JM 账号） |
| `password` | String | 密码 |

**出参：** `GetUserProfileRespData`（用户信息）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | i64 | 用户 ID |
| `username` | String | 用户名 |
| `photo` | String | 头像 URL（完整路径） |
| `gender` | String | 性别 |
| `email` | String | 邮箱 |

**登录流程：**

```
前端调用 login(username, password)
    │
    ▼
JmClient 发送 POST 请求到 /login
    ├── 表单：{username, password}
    └── 返回：Set-Cookie 头（AVS=xxx）
    │
    ▼
reqwest 自动保存 Cookie 到 Cookie Jar
    │
    ▼
返回用户信息
    │
    ▼
后续请求自动带上 Cookie（已登录状态）
```

**Cookie 存储：**
- 使用 `reqwest` 的 `Cookie Jar` 功能
- Cookie 保存在内存中，应用关闭后失效
- 下次启动需要重新登录

**前端使用场景：**
```typescript
// 登录对话框点击"登录"
async function handleLogin() {
  try {
    const user = await invoke('login', {
      username: 'myname',
      password: 'mypassword'
    })
    // 登录成功，保存用户信息到状态
    userStore.setUser(user)
    message.success('登录成功')
  } catch (e) {
    message.error('登录失败：' + e.message)
  }
}
```

---

### 2.2 get_user_profile - 获取当前用户信息

**用途：** 使用已保存的 Cookie 获取用户信息（无需密码）

**代码位置：** `src-tauri/src/commands.rs` 第 100-111 行

```rust
#[tauri::command]
#[specta::specta]
pub async fn get_user_profile(app: AppHandle) -> CommandResult<GetUserProfileRespData> {
    let jm_client = app.get_jm_client();

    let user_profile = jm_client
        .get_user_profile()
        .await
        .map_err(|err| CommandError::from("获取用户信息失败", err))?;

    Ok(user_profile)
}
```

**特点：**
- **入参：** 无（自动使用 Cookie）
- **出参：** 与 `login` 相同，返回用户信息

**与 `login` 的区别：**

| 对比项 | `login` | `get_user_profile` |
|--------|---------|-------------------|
| 需要参数 | 用户名、密码 | 不需要 |
| 使用场景 | 首次登录 | 验证登录状态、刷新信息 |
| 失败原因 | 密码错误、网络问题 | Cookie 过期、未登录 |

**Cookie 过期处理：**
- 如果返回 401 错误，表示 Cookie 已过期
- 前端应该提示用户重新登录
- 清空本地保存的用户状态

**前端使用场景：**
```typescript
// 应用启动时检查登录状态
async function checkLoginStatus() {
  try {
    const user = await invoke('get_user_profile')
    // Cookie 有效，自动登录
    userStore.setUser(user)
  } catch (e) {
    // Cookie 过期或无效，显示登录按钮
    userStore.clearUser()
  }
}
```

**登录状态管理流程：**
```
应用启动
    │
    ▼
尝试调用 get_user_profile()
    │
    ├──► 成功 ──► 显示用户头像/名称 ──► 用户可使用收藏夹功能
    │
    └──► 失败(401) ──► 显示"登录"按钮 ──► 用户点击后调用 login()
                              │
                              ▼
                        输入用户名密码
                              │
                              ▼
                        调用 login() ──► 成功 ──► 保存用户信息
```

---

## 3. 漫画搜索/获取接口详解

这类接口是应用的**核心功能**。

### 3.1 search - 搜索漫画

**用途：** 根据关键词搜索漫画，支持分页和排序

**代码位置：** `src-tauri/src/commands.rs` 第 113-132 行

```rust
#[tauri::command]
#[specta::specta]
pub async fn search(
    app: AppHandle,
    keyword: String,
    page: i64,
    sort: SearchSort,
) -> CommandResult<SearchResultVariant> {
    let jm_client = app.get_jm_client();

    let search_resp = jm_client
        .search(&keyword, page, sort)
        .await
        .map_err(|err| CommandError::from("搜索失败", err))?;

    let search_result = SearchResultVariant::from_search_resp(&app, search_resp)
        .map_err(|err| CommandError::from("搜索失败", err))?;

    Ok(search_result)
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `keyword` | String | 搜索关键词 |
| `page` | i64 | 页码（从1开始） |
| `sort` | SearchSort | 排序方式 |

**SearchSort 枚举：**
```rust
pub enum SearchSort {
    Latest,  // 最新发布
    Views,   // 最多点击
    Likes,   // 最多喜欢
}
```

**出参：** `SearchResultVariant`（两种可能的结果）

```rust
pub enum SearchResultVariant {
    SearchRespData(SearchRespData),         // 正常的搜索列表
    ComicRespData(Box<GetComicRespData>),   // 精确匹配，直接跳转
}
```

**为什么会有两种结果？**
- 如果搜索关键词精确匹配某个漫画 ID，JM 会直接返回该漫画详情
- 否则返回搜索结果列表

**执行流程：**
```
前端调用 search(keyword, page, sort)
    │
    ▼
JmClient 构建查询参数
    │
    ▼
发送 HTTP GET 请求到 /search
    │
    ▼
解密响应数据
    │
    ▼
尝试解析为 RedirectRespData（跳转）
    │
    ├──► 成功 ──► 调用 get_comic() 获取详情 ──► 返回 ComicRespData
    │
    └──► 失败
            │
            ▼
    尝试解析为 SearchRespData（列表）
            │
            ├──► 成功 ──► 返回 SearchRespData
            │
            └──► 失败 ──► 返回错误
```

**前端处理：**
```typescript
const result = await invoke('search', { keyword: '火影', page: 1, sort: 'Latest' })

if (result.type === 'SearchRespData') {
  // 显示搜索结果列表
  displaySearchResults(result.data.content)
} else if (result.type === 'ComicRespData') {
  // 直接跳转到漫画详情页
  navigateToComic(result.data.id)
}
```

---

### 3.2 get_comic - 获取漫画详情

**用途：** 获取指定漫画的详细信息，包括所有章节

**代码位置：** `src-tauri/src/commands.rs` 第 134-142 行

```rust
#[tauri::command]
#[specta::specta]
pub async fn get_comic(app: AppHandle, aid: i64) -> CommandResult<Comic> {
    let comic = utils::get_comic(app.clone(), aid)
        .await
        .map_err(|err| CommandError::from(&format!("获取ID为`{aid}`的漫画信息失败"), err))?;

    Ok(comic)
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `aid` | i64 | 漫画 ID（Album ID） |

**出参：** `Comic` 结构体（完整漫画信息，23个字段）

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | i64 | 漫画 ID |
| `name` | String | 漫画标题 |
| `addtime` | String | 添加时间 |
| `description` | String | 简介/描述 |
| `total_views` | String | 总浏览量 |
| `likes` | String | 点赞数 |
| `chapter_infos` | Vec<ChapterInfo> | **章节列表**（最重要） |
| `series_id` | String | 系列 ID |
| `comment_total` | String | 评论数 |
| `author` | Vec<String> | 作者列表 |
| `tags` | Vec<String> | 标签列表 |
| `works` | Vec<String> | 作品类型 |
| `actors` | Vec<String> | 演员/角色 |
| `related_list` | Vec<...> | 相关推荐漫画 |
| `liked` | bool | 当前用户是否点赞 |
| `is_favorite` | bool | 当前用户是否收藏 |
| `is_aids` | bool | 是否是 AIDS 标记 |
| `is_downloaded` | Option<bool> | **是否已下载**（本地计算） |
| `comic_download_dir` | Option<PathBuf> | **本地下载目录**（本地计算） |

**ChapterInfo 字段：**
| 字段 | 类型 | 说明 |
|------|------|------|
| `chapter_id` | i64 | 章节 ID |
| `chapter_title` | String | 章节标题（如"第1话 开场"） |
| `order` | i64 | 章节顺序 |
| `is_downloaded` | Option<bool> | **是否已下载**（本地计算） |
| `chapter_download_dir` | Option<PathBuf> | **本地章节目录**（本地计算） |

**注意带 `Option<>` 的字段：**
- `is_downloaded` 和 `comic_download_dir` 不是 JM API 返回的
- 是后端根据本地文件系统**计算得出**的
- `Some(true)` 表示已下载，`None` 表示未知

**执行流程：**
```
调用 get_comic(aid)
    │
    ▼
调用 JM API 获取漫画详情
    │
    ▼
解析响应数据
    │
    ▼
扫描本地下载目录
    │
    ▼
更新 is_downloaded 等字段
    │
    ▼
返回 Comic 对象
```

**使用场景：**
```typescript
// 点击搜索结果或收藏夹中的漫画
async function openComic(aid: number) {
  const comic = await invoke('get_comic', { aid })
  // 显示漫画详情
  comicStore.setCurrentComic(comic)
  // 显示章节列表，已下载的章节标记为绿色
  navigateToChapterPane()
}
```

---

### 3.3 download_comic - 一键下载整本漫画

**用途：** 下载指定漫画的所有**未下载**章节

**代码位置：** `src-tauri/src/commands.rs` 第 264-295 行

```rust
#[tauri::command(async)]
#[specta::specta]
pub async fn download_comic(app: AppHandle, aid: i64) -> CommandResult<()> {
    let download_manager = app.get_download_manager();

    // 1. 获取漫画详情
    let comic = utils::get_comic(app.clone(), aid)
        .await
        .map_err(|err| CommandError::from(&format!("获取ID为`{aid}`的漫画信息失败"), err))?;

    let comic_title = &comic.name;

    // 2. 筛选出未下载的章节
    let chapter_ids: Vec<i64> = comic
        .chapter_infos
        .iter()
        .filter(|chapter_info| chapter_info.is_downloaded != Some(true))
        .map(|chapter_info| chapter_info.chapter_id)
        .collect();

    // 3. 如果没有需要下载的章节，报错
    if chapter_ids.is_empty() {
        let err = anyhow!("漫画`{comic_title}`的所有章节都已存在于下载目录，无需重复下载");
        return Err(CommandError::from("一键下载漫画失败", err));
    }

    // 4. 为每个未下载章节创建下载任务
    for chapter_id in chapter_ids {
        download_manager
            .create_download_task(comic.clone(), chapter_id)
            .map_err(|err| CommandError::from("一键下载漫画失败", err))?;
    }

    tracing::debug!("一键下载漫画成功，已为所有需要下载的章节创建下载任务");
    Ok(())
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `aid` | i64 | 漫画 ID |

**出参：** `CommandResult<()>`**（只关心成功/失败）

**行为流程：**
```
调用 download_comic(aid)
    │
    ▼
获取漫画详情（调用 utils::get_comic）
    │
    ▼
遍历所有章节，筛选 is_downloaded != true 的
    │
    ▼
如果全部已下载 ──► 返回错误"无需重复下载"
    │
    ▼
为每个未下载章节调用 download_manager.create_download_task()
    │
    ▼
返回 Ok(())
```

**与 `create_download_task` 的区别：**

| 接口 | 粒度 | 使用场景 |
|------|------|----------|
| `create_download_task` | 单章节 | 章节详情页勾选特定章节下载 |
| `download_comic` | 整本漫画 | 一键下载整本漫画的所有未下载章节 |

**使用场景：**
```typescript
// 漫画详情页的"一键下载"按钮
async function downloadWholeComic() {
  try {
    await invoke('download_comic', { aid: currentComic.id })
    message.success('已开始下载所有未下载章节')
  } catch (e) {
    if (e.message.includes('无需重复下载')) {
      message.info('所有章节已下载')
    } else {
      message.error('下载失败：' + e.message)
    }
  }
}
```

---

## 4. 收藏夹接口详解

这类接口处理**用户收藏夹**相关功能。需要**登录状态**。

### 4.1 get_favorite_folder - 获取收藏夹内容

**用途：** 获取登录用户的收藏夹漫画列表

**代码位置：** `src-tauri/src/commands.rs` 第 144-163 行

```rust
#[tauri::command(async)]
#[specta::specta]
pub async fn get_favorite_folder(
    app: AppHandle,
    folder_id: i64,
    page: i64,
    sort: FavoriteSort,
) -> CommandResult<GetFavoriteResult> {
    let jm_client = app.get_jm_client();

    let get_favorite_resp_data = jm_client
        .get_favorite_folder(folder_id, page, sort)
        .await
        .map_err(|err| CommandError::from("获取收藏夹失败", err))?;

    let get_favorite_result = GetFavoriteResult::from_resp_data(&app, get_favorite_resp_data)
        .map_err(|err| CommandError::from("获取收藏夹失败", err))?;

    Ok(get_favorite_result)
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `folder_id` | i64 | 文件夹 ID，`0` 表示默认收藏夹 |
| `page` | i64 | 页码 |
| `sort` | FavoriteSort | 排序方式 |

**FavoriteSort 枚举：**
```rust
pub enum FavoriteSort {
    FavoriteTime,  // 收藏时间（最新收藏的在前）
    UpdateTime,    // 更新时间（最近更新的在前）
    ViewTime,      // 查看时间（最近查看的在前）
}
```

**出参：** `GetFavoriteResult`

```rust
pub struct GetFavoriteResult {
    pub list: Vec<ComicInFavorite>,  // 收藏的漫画列表
    pub count: i64,                   // 每页数量
    pub total: String,                // 总数量（字符串形式）
}
```

**使用场景：**
```typescript
// 收藏夹页面加载
async function loadFavorites(page = 1) {
  const result = await invoke('get_favorite_folder', {
    folderId: 0,           // 默认收藏夹
    page: page,
    sort: 'FavoriteTime'   // 按收藏时间排序
  })
  
  favoriteStore.setFavorites(result.list)
  favoriteStore.setTotal(parseInt(result.total))
}
```

---

### 4.2 download_all_favorites - 下载整个收藏夹

**用途：** 一键下载收藏夹中**所有漫画的所有未下载章节**

**代码位置：** `src-tauri/src/commands.rs` 第 297-418 行（较长）

**这是一个重量级函数，逻辑复杂：**

```rust
#[allow(clippy::cast_possible_wrap)]
#[tauri::command(async)]
#[specta::specta]
pub async fn download_all_favorites(app: AppHandle) -> CommandResult<()> {
    let config = app.get_config();
    let jm_client = app.get_jm_client().inner().clone();
    let download_manager = app.get_download_manager();

    // 1. 发送开始获取收藏夹事件
    let _ = DownloadAllFavoritesEvent::GetFavoritesStart.emit(&app);

    // 2. 获取第一页收藏夹
    let first_page = jm_client
        .get_favorite_folder(0, 1, FavoriteSort::FavoriteTime)
        .await
        .map_err(|err| CommandError::from("获取收藏夹失败", err))?;
    
    let mut favorite_comics = first_page.list;
    let count = first_page.count;
    let total = first_page.total.parse::<i64>()?;
    let page_count = (total / count) + 1;  // 计算总页数

    // 3. 并发获取剩余页面（使用信号量限制并发数为5）
    let sem = Arc::new(Semaphore::new(5));
    let mut join_set = JoinSet::new();
    
    for page in 2..=page_count {
        let jm_client = jm_client.clone();
        let sem = sem.clone();
        join_set.spawn(async move {
            let _permit = sem.acquire().await?;  // 获取信号量许可
            jm_client.get_favorite_folder(0, page, FavoriteSort::FavoriteTime).await
        });
    }

    // 等待所有页面获取完成
    while let Some(Ok(get_favorite_result)) = join_set.join_next().await {
        let page = get_favorite_result
            .map_err(|err| CommandError::from("获取收藏夹失败", err))?;
        favorite_comics.extend(page.list);
    }

    // 4. 发送获取完成事件，开始处理每个漫画
    let total = favorite_comics.len() as i64;
    let interval_sec = config.read().download_all_favorites_interval_sec;

    for (i, favorite_comic) in favorite_comics.into_iter().enumerate() {
        let comic_title = &favorite_comic.name;
        let comic_id = favorite_comic.id.parse::<i64>()?;

        // 发送进度事件
        let _ = DownloadAllFavoritesEvent::GetComicsProgress { 
            current: (i + 1) as i64, 
            total 
        }.emit(&app);

        // 获取漫画详情
        let comic = match utils::get_comic(app.clone(), comic_id).await {
            Ok(comic) => comic,
            Err(err) => {
                tracing::error!("获取漫画失败: {}", err);
                sleep(Duration::from_secs(interval_sec)).await;
                continue;  // 跳过这个漫画
            }
        };

        // 筛选未下载章节
        let chapter_infos: Vec<&ChapterInfo> = comic
            .chapter_infos
            .iter()
            .filter(|ch| ch.is_downloaded != Some(true))
            .collect();

        // 创建下载任务
        for chapter_info in chapter_infos {
            let _ = download_manager.create_download_task(comic.clone(), chapter_info.chapter_id);
            sleep(Duration::from_millis(100)).await;
        }

        // 每个漫画处理后休息（防封）
        sleep(Duration::from_secs(interval_sec)).await;
    }

    // 5. 发送完成事件
    let _ = DownloadAllFavoritesEvent::GetComicsEnd.emit(&app);
    Ok(())
}
```

**特点：**
- **入参：** 无（使用当前登录用户的收藏夹）
- **出参：** `CommandResult<()>`**（只关心成功/失败）

**执行流程：**
```
开始
  │
  ▼
发送 GetFavoritesStart 事件
  │
  ▼
获取第1页收藏夹，计算总页数
  │
  ▼
并发获取所有页面（最多5个并发）
  │
  ▼
遍历每个收藏的漫画
  ├──► 发送进度事件
  ├──► 获取漫画详情（可能失败，跳过）
  ├──► 筛选未下载章节
  ├──► 为每个章节创建下载任务
  └──► 休息 interval_sec 秒（防封）
  │
  ▼
发送 GetComicsEnd 事件
```

**相关事件：**
```rust
pub enum DownloadAllFavoritesEvent {
    GetFavoritesStart,                    // 开始获取收藏夹
    GetComicsProgress { current, total }, // 获取进度
    StartCreateDownloadTasks { ... },     // 开始为某漫画创建任务
    CreatingDownloadTask { ... },         // 正在创建任务
    EndCreateDownloadTasks { ... },       // 某漫画任务创建完成
    GetComicsEnd,                         // 全部完成
}
```

**使用场景：**
```typescript
// 点击"下载整个收藏夹"按钮
async function downloadAllFavorites() {
  await invoke('download_all_favorites')
  message.success('已开始批量下载收藏夹')
}

// 监听进度事件
listen('download-all-favorites-event', (e) => {
  if (e.payload.event === 'GetComicsProgress') {
    const { current, total } = e.payload.data
    progressStore.setFavoriteProgress(current, total)
  }
})
```

---

### 4.3 sync_favorite_folder - 同步收藏夹

**用途：** 解决**多端收藏不同步**问题

**代码位置：** `src-tauri/src/commands.rs` 第 522-540 行

```rust
#[tauri::command(async)]
#[specta::specta]
pub async fn sync_favorite_folder(app: AppHandle) -> CommandResult<()> {
    let jm_client = app.get_jm_client();
    
    // 同步收藏夹的方式是随便收藏一个漫画
    // 调用两次toggle是因为要把新收藏的漫画取消收藏
    let task1 = jm_client.toggle_favorite_comic(468_984);
    let task2 = jm_client.toggle_favorite_comic(468_984);
    
    let (resp1, resp2) =
        tokio::try_join!(task1, task2).map_err(|err| CommandError::from("同步收藏夹失败", err))?;
    
    if resp1.toggle_type == resp2.toggle_type {
        let toggle_type = resp1.toggle_type;
        let err_title = "同步收藏夹失败";
        let err = anyhow!("两个请求都是`{toggle_type:?}`操作");
        return Err(CommandError::from(err_title, err));
    }

    Ok(())
}
```

**实现原理（Hack 技巧）：**

```
问题：用户在网页端收藏了漫画，但在 App 中看不到
原因：JM 服务器有缓存，多端不同步
解决方案：
  1. 收藏一个随机漫画（ID: 468984）
  2. 立即取消收藏该漫画
  3. 这两个操作会触发服务器刷新收藏列表缓存
  4. 之后获取收藏夹就是最新的了
```

**toggle_type 检查：**
- 第一次调用：`Favorite`（收藏）
- 第二次调用：`Unfavorite`（取消收藏）
- 如果两次结果相同，说明操作失败

**使用场景：**
```typescript
// 收藏夹页面点击"同步"按钮
async function syncFavorites() {
  await invoke('sync_favorite_folder')
  message.success('同步成功')
  await loadFavorites()  // 重新加载收藏夹
}
```

---

## 5. 每周推荐接口详解

这类接口获取 JM 官方的**每周推荐**内容，无需登录。

### 5.1 get_weekly_info - 获取每周推荐分类信息

**用途：** 获取推荐页面的分类结构

**代码位置：** `src-tauri/src/commands.rs` 第 165-176 行

```rust
#[tauri::command(async)]
#[specta::specta]
pub async fn get_weekly_info(app: AppHandle) -> CommandResult<GetWeeklyInfoRespData> {
    let jm_client = app.get_jm_client();

    let weekly_info = jm_client
        .get_weekly_info()
        .await
        .map_err(|err| CommandError::from("获取每周必看信息失败", err))?;

    Ok(weekly_info)
}
```

**入参：** 无

**出参：** `GetWeeklyInfoRespData`

```rust
pub struct GetWeeklyInfoRespData {
    pub categories: Vec<Category>,  // 分类列表
}

pub struct Category {
    pub id: Option<String>,         // 分类 ID
    pub title: Option<String>,      // 分类标题
    pub sub_categories: Vec<SubCategory>,
}

pub struct SubCategory {
    pub id: Option<String>,         // 子分类 ID
    pub title: Option<String>,      // 子分类标题
}
```

**示例数据结构：**
```
每周推荐
├── 本周必看
│   ├── 周一
│   ├── 周二
│   └── ...
├── 新作推荐
│   ├── 本周新作
│   └── 上月新作
└── 排行榜
    ├── 人气榜
    └── 收藏榜
```

**使用场景：**
```typescript
// 进入"每周推荐"页面时调用
async function loadWeeklyInfo() {
  const info = await invoke('get_weekly_info')
  for (const category of info.categories) {
    console.log(category.title)  // "本周必看"
    for (const sub of category.sub_categories) {
      console.log('  -', sub.title)  // "周一"...
    }
  }
}
```

---

### 5.2 get_weekly - 获取每周推荐列表

**用途：** 获取指定分类的具体漫画列表

**代码位置：** `src-tauri/src/commands.rs` 第 178-196 行

```rust
#[tauri::command(async)]
#[specta::specta]
pub async fn get_weekly(
    app: AppHandle,
    category_id: String,
    type_id: String,
) -> CommandResult<GetWeeklyResult> {
    let jm_client = app.get_jm_client();

    let get_weekly_resp_data = jm_client
        .get_weekly(&category_id, &type_id)
        .await
        .map_err(|err| CommandError::from("获取每周必看失败", err))?;

    let get_weekly_result = GetWeeklyResult::from_resp_data(&app, get_weekly_resp_data)
        .map_err(|err| CommandError::from("获取每周必看失败", err))?;

    Ok(get_weekly_result)
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `category_id` | String | 分类 ID（父级） |
| `type_id` | String | 类型 ID（子级） |

**这两个 ID 从哪来？**
- 从 `get_weekly_info` 返回的数据中获取

**出参：** `GetWeeklyResult`

```rust
pub struct GetWeeklyResult {
    pub list: Vec<ComicInWeekly>,  // 推荐漫画列表
}
```

**使用场景：**
```typescript
// 用户点击"本周必看 - 周一"后调用
async function loadWeeklyList(categoryId: string, typeId: string) {
  const result = await invoke('get_weekly', {
    categoryId: '1',
    typeId: '2'
  })
  displayComicList(result.list)
}
```

**数据流向：**
```
进入每周推荐页面
    │
    ▼
调用 get_weekly_info()
    │
    ▼
显示分类菜单
    │
    ▼
用户点击某个子分类
    │
    ▼
调用 get_weekly(category_id, type_id)
    │
    ▼
显示该分类的漫画列表
```

---

## 6. 下载任务接口详解

这类接口是应用的**核心下载控制**功能。

### 6.1 create_download_task - 创建下载任务

**用途：** 创建单个章节的下载任务

**代码位置：** `src-tauri/src/commands.rs` 第 198-214 行

```rust
#[allow(clippy::needless_pass_by_value)]
#[tauri::command(async)]
#[specta::specta]
pub fn create_download_task(app: AppHandle, comic: Comic, chapter_id: i64) -> CommandResult<()> {
    let download_manager = app.get_download_manager();
    let comic_title = comic.name.clone();

    download_manager
        .create_download_task(comic, chapter_id)
        .map_err(|err| {
            let err_title = format!("`{comic_title}`的章节ID为`{chapter_id}`的下载任务创建失败");
            CommandError::from(&err_title, err)
        })?;

    tracing::debug!("创建章节ID为`{chapter_id}`的下载任务成功");
    Ok(())
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `comic` | Comic | 完整的漫画对象 |
| `chapter_id` | i64 | 要下载的章节 ID |

**出参：** `CommandResult<()>`**

**执行流程：**
```
调用 create_download_task(comic, chapter_id)
    │
    ▼
获取 DownloadManager
    │
    ▼
检查该章节是否已在下载队列中
    │
    ▼
创建新的下载任务
    │
    ▼
将任务加入下载队列
    │
    ▼
触发下载（如果并发数未满）
    │
    ▼
发送 DownloadTaskEvent::Create 事件
```

**相关事件：**
```rust
DownloadTaskEvent::Create {
    state: DownloadTaskState,
    comic: Box<Comic>,
    chapter_info: Box<ChapterInfo>,
    downloaded_img_count: u32,
    total_img_count: u32,
}
```

**使用场景：**
```typescript
// 章节详情页勾选章节后点击"下载"
async function downloadChapters(selectedChapterIds: number[]) {
  for (const chapterId of selectedChapterIds) {
    await invoke('create_download_task', {
      comic: currentComic,
      chapterId: chapterId
    })
  }
  message.success(`已为 ${selectedChapterIds.length} 个章节创建下载任务`)
}
```

---

### 6.2 pause_download_task - 暂停下载任务

**用途：** 暂停正在下载的章节

**代码位置：** `src-tauri/src/commands.rs` 第 216-230 行

```rust
#[allow(clippy::needless_pass_by_value)]
#[tauri::command(async)]
#[specta::specta]
pub fn pause_download_task(app: AppHandle, chapter_id: i64) -> CommandResult<()> {
    let download_manager = app.get_download_manager();

    download_manager
        .pause_download_task(chapter_id)
        .map_err(|err| {
            CommandError::from(&format!("暂停章节ID为`{chapter_id}`的下载任务失败"), err)
        })?;

    tracing::debug!("暂停章节ID为`{chapter_id}`的下载任务成功");
    Ok(())
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `chapter_id` | i64 | 章节 ID |

**说明：**
- 通过 `chapter_id` 唯一标识下载任务
- 暂停后任务状态变为 `Paused`
- 已下载的图片会保留

**使用场景：**
```typescript
async function pauseDownload(chapterId: number) {
  await invoke('pause_download_task', { chapterId })
  message.success('已暂停')
}
```

---

### 6.3 resume_download_task - 恢复下载任务

**用途：** 恢复已暂停的下载任务

**代码位置：** `src-tauri/src/commands.rs` 第 232-246 行

```rust
#[allow(clippy::needless_pass_by_value)]
#[tauri::command(async)]
#[specta::specta]
pub fn resume_download_task(app: AppHandle, chapter_id: i64) -> CommandResult<()> {
    let download_manager = app.get_download_manager();

    download_manager
        .resume_download_task(chapter_id)
        .map_err(|err| {
            CommandError::from(&format!("恢复章节ID为`{chapter_id}`的下载任务失败"), err)
        })?;

    tracing::debug!("恢复章节ID为`{chapter_id}`的下载任务成功");
    Ok(())
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `chapter_id` | i64 | 章节 ID |

**说明：**
- 恢复后任务状态变为 `Downloading`
- 从上次暂停的位置继续下载

**使用场景：**
```typescript
async function resumeDownload(chapterId: number) {
  await invoke('resume_download_task', { chapterId })
  message.success('已恢复下载')
}
```

---

### 6.4 cancel_download_task - 取消下载任务

**用途：** 取消下载任务并删除已下载的文件

**代码位置：** `src-tauri/src/commands.rs` 第 248-262 行

```rust
#[allow(clippy::needless_pass_by_value)]
#[tauri::command(async)]
#[specta::specta]
pub fn cancel_download_task(app: AppHandle, chapter_id: i64) -> CommandResult<()> {
    let download_manager = app.get_download_manager();

    download_manager
        .cancel_download_task(chapter_id)
        .map_err(|err| {
            CommandError::from(&format!("取消章节ID为`{chapter_id}`的下载任务失败"), err)
        })?;

    tracing::debug!("取消章节ID为`{chapter_id}`的下载任务成功");
    Ok(())
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `chapter_id` | i64 | 章节 ID |

**说明：**
- 任务会被从队列中移除
- **已下载的部分文件会被删除**（与暂停不同）
- 状态变为 `Cancelled`

**使用场景：**
```typescript
async function cancelDownload(chapterId: number) {
  const confirmed = await showConfirm('确定要取消下载吗？')
  if (!confirmed) return
  
  await invoke('cancel_download_task', { chapterId })
  message.success('已取消下载')
}
```

---

### 6.5 get_downloaded_comics - 获取已下载漫画列表

**用途：** 扫描本地下载目录，获取所有已下载的漫画

**代码位置：** `src-tauri/src/commands.rs` 第 542-655 行

```rust
#[allow(clippy::needless_pass_by_value)]
#[allow(clippy::too_many_lines)]
#[tauri::command(async)]
#[specta::specta]
pub fn get_downloaded_comics(app: AppHandle) -> Vec<Comic> {
    let download_dir = app.get_config().read().download_dir.clone();

    // 1. 遍历下载目录，收集所有元数据文件
    let mut metadata_path_with_modify_time = Vec::new();
    for entry in WalkDir::new(&download_dir).into_iter().filter_map(Result::ok) {
        if !entry.is_comic_metadata() {
            continue;
        }
        let path = entry.path();
        let metadata = path.metadata().ok()?;
        let modify_time = metadata.modified().ok()?;
        metadata_path_with_modify_time.push((path.to_path_buf(), modify_time));
    }

    // 2. 按修改时间排序（最新的在前）
    metadata_path_with_modify_time.sort_by(|(_, a), (_, b)| b.cmp(a));

    // 3. 解析每个元数据文件
    let mut downloaded_comics = Vec::new();
    for (metadata_path, _) in metadata_path_with_modify_time {
        if let Ok(comic) = Comic::from_metadata(&metadata_path) {
            downloaded_comics.push(comic);
        }
    }

    // 4. 按漫画ID去重
    let mut unique_comics = Vec::new();
    let mut seen_ids = HashSet::new();
    for comic in downloaded_comics {
        if seen_ids.insert(comic.id) {
            unique_comics.push(comic);
        }
    }

    unique_comics
}
```

**入参：** 无

**出参：** `Vec<Comic>`（已下载漫画列表）

**行为：**
1. 遍历下载目录，查找所有 `元数据.json` 文件
2. 按文件修改时间排序（最新下载的在前）
3. 解析每个元数据文件为 `Comic` 对象
4. **按漫画 ID 去重**

**使用场景：**
```typescript
async function loadDownloadedComics() {
  const comics = await invoke('get_downloaded_comics')
  downloadedStore.setComics(comics)
}
```

---

### 6.6 update_downloaded_comics - 更新已下载漫画

**用途：** 检查已下载漫画是否有新章节，自动下载缺失的章节

**代码位置：** `src-tauri/src/commands.rs` 第 420-509 行

```rust
#[allow(clippy::cast_possible_wrap)]
#[tauri::command(async)]
#[specta::specta]
pub async fn update_downloaded_comics(app: AppHandle) -> CommandResult<()> {
    let config = app.get_config();
    let download_manager = app.get_download_manager();

    // 1. 获取所有已下载的漫画
    let downloaded_comics = get_downloaded_comics(app.clone());
    let total = downloaded_comics.len() as i64;

    // 2. 发送开始事件
    let _ = UpdateDownloadedComicsEvent::GetComicStart { total }.emit(&app);

    let interval_sec = config.read().update_downloaded_comics_interval_sec;

    // 3. 遍历每个已下载的漫画
    for (i, downloaded_comic) in downloaded_comics.into_iter().enumerate() {
        let comic_id = downloaded_comic.id;
        let current = (i + 1) as i64;
        
        // 发送进度事件
        let _ = UpdateDownloadedComicsEvent::GetComicProgress { current, total }.emit(&app);

        // 4. 获取最新漫画详情
        let comic = match utils::get_comic(app.clone(), comic_id).await {
            Ok(comic) => comic,
            Err(err) => {
                tracing::error!("更新时获取漫画失败: {}", err);
                sleep(Duration::from_secs(interval_sec)).await;
                continue;
            }
        };

        // 5. 筛选出新章节
        let new_chapters: Vec<&ChapterInfo> = comic
            .chapter_infos
            .iter()
            .filter(|ch| ch.is_downloaded != Some(true))
            .collect();

        // 6. 为新章节创建下载任务
        for chapter_info in new_chapters {
            let _ = download_manager.create_download_task(comic.clone(), chapter_info.chapter_id);
        }

        // 7. 休息（防封）
        sleep(Duration::from_secs(interval_sec)).await;
    }

    // 8. 发送完成事件
    let _ = UpdateDownloadedComicsEvent::GetComicEnd.emit(&app);
    Ok(())
}
```

**入参：** 无

**出参：** `CommandResult<()>`**

**相关事件：**
```rust
UpdateDownloadedComicsEvent::GetComicStart { total }
UpdateDownloadedComicsEvent::GetComicProgress { current, total }
UpdateDownloadedComicsEvent::CreateDownloadTasksStart { ... }
UpdateDownloadedComicsEvent::CreateDownloadTaskProgress { ... }
UpdateDownloadedComicsEvent::CreateDownloadTasksEnd { ... }
UpdateDownloadedComicsEvent::GetComicEnd
```

**使用场景：**
```typescript
// 点击"更新库存"按钮
async function updateDownloadedComics() {
  await invoke('update_downloaded_comics')
  message.success('已开始检查更新')
}
```

---

## 7. 导出接口详解

### 7.1 export_cbz - 导出为 CBZ 格式

**用途：** 将已下载的漫画章节打包为 CBZ 格式

**代码位置：** `src-tauri/src/commands.rs` 第 657-666 行

```rust
#[tauri::command(async)]
#[specta::specta]
#[allow(clippy::needless_pass_by_value)]
pub fn export_cbz(app: AppHandle, comic: Comic) -> CommandResult<()> {
    let comic_title = &comic.name;
    export::cbz(&app, &comic)
        .context(format!("漫画`{comic_title}`导出cbz失败"))
        .map_err(|err| CommandError::from("导出cbz失败", err))?;
    Ok(())
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `comic` | Comic | 漫画对象（必须已下载） |

**CBZ 是什么？**
- Comic Book ZIP 的缩写
- 本质就是 ZIP 压缩包，里面包含图片
- 扩展名改为 `.cbz` 以便漫画阅读器识别

**相关事件：**
```rust
ExportCbzEvent::Start { uuid, comic_title, total }
ExportCbzEvent::Progress { uuid, current }
ExportCbzEvent::Error { uuid }
ExportCbzEvent::End { uuid, chapter_export_dir }
```

---

### 7.2 export_pdf - 导出为 PDF 格式

**用途：** 将已下载的漫画章节合并为 PDF 文件

**代码位置：** `src-tauri/src/commands.rs` 第 668-677 行

```rust
#[tauri::command(async)]
#[specta::specta]
#[allow(clippy::needless_pass_by_value)]
pub fn export_pdf(app: AppHandle, comic: Comic) -> CommandResult<()> {
    let comic_title = &comic.name;
    export::pdf(&app, &comic)
        .context(format!("漫画`{comic_title}`导出pdf失败"))
        .map_err(|err| CommandError::from("导出pdf失败", err))?;
    Ok(())
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `comic` | Comic | 漫画对象（必须已下载） |

**相关事件：**
```rust
ExportPdfEvent::CreateStart { ... }
ExportPdfEvent::CreateProgress { ... }
ExportPdfEvent::CreateEnd { ... }
ExportPdfEvent::MergeStart { ... }
ExportPdfEvent::MergeEnd { ... }
```

**使用场景：**
```typescript
async function exportComic(format: 'cbz' | 'pdf') {
  const command = format === 'cbz' ? 'export_cbz' : 'export_pdf'
  await invoke(command, { comic: selectedComic })
  message.success(`已开始导出为 ${format.toUpperCase()}`)
}
```

---

## 8. 同步/工具接口详解

### 8.1 show_path_in_file_manager - 在文件管理器中显示

**用途：** 打开系统的文件管理器，并定位到指定文件或目录

**代码位置：** `src-tauri/src/commands.rs` 第 511-520 行

```rust
#[allow(clippy::needless_pass_by_value)]
#[tauri::command(async)]
#[specta::specta]
pub fn show_path_in_file_manager(app: AppHandle, path: &str) -> CommandResult<()> {
    app.opener()
        .reveal_item_in_dir(path)
        .context(format!("在文件管理器中打开`{path}`失败"))
        .map_err(|err| CommandError::from("在文件管理器中打开失败", err))?;
    Ok(())
}
```

**入参：**
| 参数 | 类型 | 说明 |
|------|------|------|
| `path` | &str | 文件或目录路径 |

**说明：**
- Windows：打开资源管理器，选中该文件
- macOS：打开 Finder
- Linux：打开默认文件管理器

**使用场景：**
```typescript
async function openDownloadDir() {
  await invoke('show_path_in_file_manager', {
    path: comic.comicDownloadDir
  })
}
```

---

### 8.2-8.5 同步漫画状态接口

这四个接口功能类似：

| 接口 | 输入类型 | 用途 |
|------|---------|------|
| `get_synced_comic` | `Comic` | 同步 Comic 对象 |
| `get_synced_comic_in_favorite` | `ComicInFavorite` | 同步收藏夹中的漫画 |
| `get_synced_comic_in_search` | `ComicInSearch` | 同步搜索结果中的漫画 |
| `get_synced_comic_in_weekly` | `ComicInWeekly` | 同步每周推荐中的漫画 |

**用途：** 根据本地文件系统，更新以下字段：
- `is_downloaded`：是否已下载
- `comic_download_dir`：下载目录
- `chapter_infos[i].is_downloaded`：章节是否已下载

**使用场景：**
```typescript
const syncedComic = await invoke('get_synced_comic_in_search', {
  comic: searchResult
})
if (syncedComic.isDownloaded) {
  console.log('这本漫画已经下载过了')
}
```

---

## 9. 事件系统详解

后端通过事件向前端推送实时更新：

| 事件名 | 用途 | 触发时机 |
|--------|------|----------|
| `DownloadSpeedEvent` | 下载速度 | 每秒更新 |
| `DownloadSleepingEvent` | 下载休眠倒计时 | 下载间隔等待时 |
| `DownloadTaskEvent` | 下载任务状态 | 任务创建/更新/完成 |
| `DownloadAllFavoritesEvent` | 批量下载收藏夹进度 | 下载整个收藏夹时 |
| `UpdateDownloadedComicsEvent` | 更新库存进度 | 检查已下载漫画更新 |
| `ExportCbzEvent` | CBZ 导出进度 | 导出 CBZ 时 |
| `ExportPdfEvent` | PDF 导出进度 | 导出 PDF 时 |
| `LogEvent` | 日志消息 | 有新的日志时 |

### 前端监听示例

```typescript
import { listen } from '@tauri-apps/api/event'

// 监听下载速度
listen('download-speed-event', (e) => {
  console.log('当前速度:', e.payload.speed)
})

// 监听下载任务状态
listen('download-task-event', (e) => {
  if (e.payload.event === 'Create') {
    console.log('新任务:', e.payload.data)
  } else if (e.payload.event === 'Update') {
    console.log('任务更新:', e.payload.data)
  }
})
```

---

## 附录：所有接口速查表

| 分类 | 数量 | 接口 |
|------|------|------|
| 系统/配置 | 4 | greet, get_config, save_config, get_logs_dir_size |
| 用户/认证 | 2 | login, get_user_profile |
| 漫画搜索/获取 | 3 | search, get_comic, download_comic |
| 收藏夹 | 3 | get_favorite_folder, download_all_favorites, sync_favorite_folder |
| 每周推荐 | 2 | get_weekly_info, get_weekly |
| 下载任务 | 6 | create/pause/resume/cancel_download_task, get_downloaded_comics, update_downloaded_comics |
| 导出 | 2 | export_cbz, export_pdf |
| 同步/工具 | 5 | show_path_in_file_manager, get_synced_comic*4 |

**总计：23 个 Commands**

---

**本文档结束。**
