# 02. 后端接口列表

本文档列出所有后端提供的 Tauri Commands，包括功能描述、入参、出参和对应代码文件。

---

## 目录

1. [系统/配置接口](#1-系统配置接口)
2. [用户/认证接口](#2-用户认证接口)
3. [漫画搜索/获取接口](#3-漫画搜索获取接口)
4. [收藏夹接口](#4-收藏夹接口)
5. [每周推荐接口](#5-每周推荐接口)
6. [下载任务接口](#6-下载任务接口)
7. [导出接口](#7-导出接口)
8. [同步/工具接口](#8-同步工具接口)

---

## 1. 系统/配置接口

### 1.1 greet
- **功能**: 测试用的问候函数
- **代码文件**: `src-tauri/src/commands.rs` (第 27-31 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| name | `&str` | 用户名 |

**出参:**
| 类型 | 说明 |
|------|------|
| `String` | 问候语 |

---

### 1.2 get_config
- **功能**: 获取当前配置
- **代码文件**: `src-tauri/src/commands.rs` (第 33-38 行)

**入参:** 无 (通过 AppHandle 获取)

**出参:**
| 类型 | 说明 |
|------|------|
| `Config` | 配置对象 |

**Config 字段:**
```rust
pub struct Config {
    pub username: String,
    pub password: String,
    pub download_dir: PathBuf,
    pub export_dir: PathBuf,
    pub download_format: DownloadFormat,
    pub dir_fmt: String,
    pub proxy_mode: ProxyMode,
    pub proxy_host: String,
    pub proxy_port: u16,
    pub enable_file_logger: bool,
    pub chapter_concurrency: usize,
    pub chapter_download_interval_sec: u64,
    pub img_concurrency: usize,
    pub img_download_interval_sec: u64,
    pub download_all_favorites_interval_sec: u64,
    pub update_downloaded_comics_interval_sec: u64,
    pub api_domain_mode: ApiDomainMode,
    pub custom_api_domain: String,
    pub should_download_cover: bool,
}
```

---

### 1.3 save_config
- **功能**: 保存配置
- **代码文件**: `src-tauri/src/commands.rs` (第 40-81 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| config | `Config` | 新的配置对象 |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(())，失败返回错误 |

**副作用:**
- 如果代理配置变更，会重新加载 HTTP 客户端
- 如果日志配置变更，会重新加载/禁用文件日志

---

### 1.4 get_logs_dir_size
- **功能**: 获取日志目录大小
- **代码文件**: `src-tauri/src/commands.rs` (第 679-695 行)

**入参:** 无

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<u64>` | 日志目录大小(字节) |

---

## 2. 用户/认证接口

### 2.1 login
- **功能**: 用户登录
- **代码文件**: `src-tauri/src/commands.rs` (第 83-98 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| username | `String` | 用户名 |
| password | `String` | 密码 |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<GetUserProfileRespData>` | 用户信息 |

**GetUserProfileRespData 字段:**
```rust
pub struct GetUserProfileRespData {
    pub id: i64,
    pub username: String,
    pub photo: String,       // 头像 URL
    pub gender: String,      // 性别
    pub email: String,
}
```

---

### 2.2 get_user_profile
- **功能**: 获取当前登录用户信息 (使用 Cookie)
- **代码文件**: `src-tauri/src/commands.rs` (第 100-111 行)

**入参:** 无

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<GetUserProfileRespData>` | 用户信息 |

**注意:** 如果 Cookie 无效或过期，返回 401 错误

---

## 3. 漫画搜索/获取接口

### 3.1 search
- **功能**: 搜索漫画
- **代码文件**: `src-tauri/src/commands.rs` (第 113-132 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| keyword | `String` | 搜索关键词 |
| page | `i64` | 页码 |
| sort | `SearchSort` | 排序方式 |

**SearchSort 枚举:**
```rust
pub enum SearchSort {
    Latest,       // 最新发布
    Views,        // 最多点击
    Likes,        // 最多喜欢
}
```

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<SearchResultVariant>` | 搜索结果 |

**SearchResultVariant 枚举:**
```rust
pub enum SearchResultVariant {
    SearchRespData(SearchRespData),         // 搜索列表
    ComicRespData(Box<GetComicRespData>),   // 直接跳转到漫画详情
}
```

---

### 3.2 get_comic
- **功能**: 获取漫画详情
- **代码文件**: `src-tauri/src/commands.rs` (第 134-142 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| aid | `i64` | 漫画 ID |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<Comic>` | 漫画详情 |

**Comic 字段:**
```rust
pub struct Comic {
    pub id: i64,
    pub name: String,
    pub addtime: String,
    pub description: String,
    pub total_views: String,
    pub likes: String,
    pub chapter_infos: Vec<ChapterInfo>,
    pub series_id: String,
    pub comment_total: String,
    pub author: Vec<String>,
    pub tags: Vec<String>,
    pub works: Vec<String>,
    pub actors: Vec<String>,
    pub related_list: Vec<RelatedListRespData>,
    pub liked: bool,
    pub is_favorite: bool,
    pub is_aids: bool,
    pub is_downloaded: Option<bool>,      // 是否已下载
    pub comic_download_dir: Option<PathBuf>, // 下载目录
}
```

**ChapterInfo 字段:**
```rust
pub struct ChapterInfo {
    pub chapter_id: i64,
    pub chapter_title: String,
    pub order: i64,
    pub is_downloaded: Option<bool>,
    pub chapter_download_dir: Option<PathBuf>,
}
```

---

### 3.3 download_comic
- **功能**: 一键下载整本漫画(所有未下载章节)
- **代码文件**: `src-tauri/src/commands.rs` (第 264-295 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| aid | `i64` | 漫画 ID |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

**行为:**
- 获取漫画详情
- 筛选出未下载的章节
- 为每个未下载章节创建下载任务

---

## 4. 收藏夹接口

### 4.1 get_favorite_folder
- **功能**: 获取收藏夹内容
- **代码文件**: `src-tauri/src/commands.rs` (第 144-163 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| folder_id | `i64` | 文件夹 ID (0 为默认收藏夹) |
| page | `i64` | 页码 |
| sort | `FavoriteSort` | 排序方式 |

**FavoriteSort 枚举:**
```rust
pub enum FavoriteSort {
    FavoriteTime,  // 收藏时间
    UpdateTime,    // 更新时间
    ViewTime,      // 查看时间
}
```

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<GetFavoriteResult>` | 收藏夹结果 |

**GetFavoriteResult 字段:**
```rust
pub struct GetFavoriteResult {
    pub list: Vec<ComicInFavorite>,
    pub count: i64,
    pub total: String,
}
```

---

### 4.2 download_all_favorites
- **功能**: 下载整个收藏夹
- **代码文件**: `src-tauri/src/commands.rs` (第 297-418 行)

**入参:** 无

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

**行为:**
- 获取所有收藏夹页面
- 逐个获取漫画详情
- 为每个漫画的未下载章节创建任务
- 可配置间隔时间防封

**相关事件:** `DownloadAllFavoritesEvent`

---

### 4.3 sync_favorite_folder
- **功能**: 同步收藏夹(解决多端收藏不同步问题)
- **代码文件**: `src-tauri/src/commands.rs` (第 522-540 行)

**入参:** 无

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

**实现原理:**
- 随机收藏一个漫画
- 再取消收藏该漫画
- 触发服务器刷新收藏列表

---

## 5. 每周推荐接口

### 5.1 get_weekly_info
- **功能**: 获取每周推荐分类信息
- **代码文件**: `src-tauri/src/commands.rs` (第 165-176 行)

**入参:** 无

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<GetWeeklyInfoRespData>` | 分类信息 |

---

### 5.2 get_weekly
- **功能**: 获取每周推荐列表
- **代码文件**: `src-tauri/src/commands.rs` (第 178-196 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| category_id | `String` | 分类 ID |
| type_id | `String` | 类型 ID |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<GetWeeklyResult>` | 推荐列表 |

---

## 6. 下载任务接口

### 6.1 create_download_task
- **功能**: 创建下载任务
- **代码文件**: `src-tauri/src/commands.rs` (第 198-214 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| comic | `Comic` | 漫画对象 |
| chapter_id | `i64` | 章节 ID |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

**相关事件:** `DownloadTaskEvent::Create`

---

### 6.2 pause_download_task
- **功能**: 暂停下载任务
- **代码文件**: `src-tauri/src/commands.rs` (第 216-230 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| chapter_id | `i64` | 章节 ID |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

---

### 6.3 resume_download_task
- **功能**: 恢复下载任务
- **代码文件**: `src-tauri/src/commands.rs` (第 232-246 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| chapter_id | `i64` | 章节 ID |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

---

### 6.4 cancel_download_task
- **功能**: 取消下载任务
- **代码文件**: `src-tauri/src/commands.rs` (第 248-262 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| chapter_id | `i64` | 章节 ID |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

---

### 6.5 get_downloaded_comics
- **功能**: 获取已下载的漫画列表
- **代码文件**: `src-tauri/src/commands.rs` (第 542-655 行)

**入参:** 无

**出参:**
| 类型 | 说明 |
|------|------|
| `Vec<Comic>` | 已下载漫画列表 |

**行为:**
- 扫描下载目录中的元数据文件
- 按修改时间排序(最新的在前)
- 自动去重

---

### 6.6 update_downloaded_comics
- **功能**: 更新已下载漫画(检查新章节并下载)
- **代码文件**: `src-tauri/src/commands.rs` (第 420-509 行)

**入参:** 无

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

**行为:**
- 遍历所有已下载漫画
- 获取最新漫画详情
- 为新章节创建下载任务

**相关事件:** `UpdateDownloadedComicsEvent`

---

## 7. 导出接口

### 7.1 export_cbz
- **功能**: 导出为 CBZ 格式
- **代码文件**: `src-tauri/src/commands.rs` (第 657-666 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| comic | `Comic` | 漫画对象 |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

**相关事件:** `ExportCbzEvent`

---

### 7.2 export_pdf
- **功能**: 导出为 PDF 格式
- **代码文件**: `src-tauri/src/commands.rs` (第 668-677 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| comic | `Comic` | 漫画对象 |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

**相关事件:** `ExportPdfEvent`

---

## 8. 同步/工具接口

### 8.1 show_path_in_file_manager
- **功能**: 在文件管理器中显示路径
- **代码文件**: `src-tauri/src/commands.rs` (第 511-520 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| path | `&str` | 文件或目录路径 |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<()>` | 成功返回 Ok(()) |

---

### 8.2 get_synced_comic
- **功能**: 同步 Comic 对象的本地状态
- **代码文件**: `src-tauri/src/commands.rs` (第 697-712 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| comic | `Comic` | 漫画对象 |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<Comic>` | 更新后的漫画对象 |

**作用:** 更新 is_downloaded 和 comic_download_dir 字段

---

### 8.3 get_synced_comic_in_favorite
- **功能**: 同步 ComicInFavorite 对象的本地状态
- **代码文件**: `src-tauri/src/commands.rs` (第 714-731 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| comic | `ComicInFavorite` | 收藏夹中的漫画 |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<ComicInFavorite>` | 更新后的漫画对象 |

---

### 8.4 get_synced_comic_in_search
- **功能**: 同步 ComicInSearch 对象的本地状态
- **代码文件**: `src-tauri/src/commands.rs` (第 733-750 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| comic | `ComicInSearch` | 搜索结果中的漫画 |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<ComicInSearch>` | 更新后的漫画对象 |

---

### 8.5 get_synced_comic_in_weekly
- **功能**: 同步 ComicInWeekly 对象的本地状态
- **代码文件**: `src-tauri/src/commands.rs` (第 752-769 行)

**入参:**
| 参数名 | 类型 | 说明 |
|--------|------|------|
| comic | `ComicInWeekly` | 每周推荐中的漫画 |

**出参:**
| 类型 | 说明 |
|------|------|
| `CommandResult<ComicInWeekly>` | 更新后的漫画对象 |

---

## 事件列表

后端通过事件向前端推送实时更新:

| 事件名 | 用途 | 定义文件 |
|--------|------|----------|
| DownloadSpeedEvent | 下载速度 | `src-tauri/src/events.rs` |
| DownloadSleepingEvent | 下载休眠倒计时 | `src-tauri/src/events.rs` |
| DownloadTaskEvent | 下载任务状态 | `src-tauri/src/events.rs` |
| DownloadAllFavoritesEvent | 批量下载收藏夹进度 | `src-tauri/src/events.rs` |
| UpdateDownloadedComicsEvent | 更新已下载漫画进度 | `src-tauri/src/events.rs` |
| ExportCbzEvent | CBZ 导出进度 | `src-tauri/src/events.rs` |
| ExportPdfEvent | PDF 导出进度 | `src-tauri/src/events.rs` |
| LogEvent | 日志消息 | `src-tauri/src/events.rs` |

---

## 前端调用示例

```typescript
import { invoke } from '@tauri-apps/api/core'
import { listen } from '@tauri-apps/api/event'

// 调用命令
const comic = await invoke('get_comic', { aid: 12345 })

// 监听事件
listen('download-task-event', (event) => {
  console.log(event.payload)
})
```
