# Tauri 2.x 桌面应用开发指南

> 基于 WSL UI 项目的真实实践总结

## 一、项目架构

Tauri 2.x 采用 **Rust 后端 + Web 前端** 的双层架构：

```
项目根目录/
├── src/                  # 前端（React/Vue/Svelte 等任意 Web 框架）
│   ├── components/       # UI 组件
│   ├── store/            # 状态管理（Zustand）
│   ├── services/         # 服务层（封装 invoke 调用）
│   ├── hooks/            # 自定义 Hooks
│   ├── types/            # TypeScript 类型定义
│   ├── i18n/             # 国际化
│   ├── themes/           # 主题系统
│   └── utils/            # 工具函数
├── src-tauri/            # Rust 后端
│   ├── src/              # Rust 源码
│   │   ├── main.rs       # 入口：Builder、插件注册、命令注册、托盘
│   │   ├── commands/     # Tauri 命令（前端可调用的函数）
│   │   ├── wsl/          # 业务逻辑模块
│   │   ├── settings.rs   # 设置持久化
│   │   └── error.rs      # 错误类型
│   ├── capabilities/     # 权限声明（Tauri 2.x 新特性）
│   ├── icons/            # 应用图标
│   ├── resources/        # 打包资源
│   ├── Cargo.toml        # Rust 依赖
│   └── tauri.conf.json   # Tauri 配置
├── crates/               # 本地 Rust crate（共享库）
├── package.json          # 前端依赖
├── vite.config.ts        # Vite 构建配置
└── tsconfig.json         # TypeScript 配置
```

## 二、环境准备

### 2.1 系统要求

- **Node.js** v18 或更高版本
- **Rust** 最新稳定版（rustup）
- **Windows** 10/11（需启用 WebView2，Tauri 会自动处理）
- **构建必须在 Windows 主机上进行**（不可在 WSL 内构建）

### 2.2 安装依赖

```bash
# 安装 Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 安装 Tauri CLI
npm install -g @tauri-apps/cli

# 创建新项目
npm create tauri-app@latest
```

### 2.3 启动开发

```bash
npm run tauri dev    # 同时启动 Vite 开发服务器 + Rust 编译
```

## 三、Rust 后端开发

### 3.1 入口文件 main.rs

Tauri 应用的核心入口，负责：

1. **注册插件** — 日志、Shell、对话框、文件系统等
2. **管理状态** — 通过 `manage()` 注入全局状态
3. **初始化** — 托盘图标、菜单、启动逻辑
4. **窗口事件** — 关闭行为、最小化到托盘等
5. **注册命令** — 前端可调用的 Rust 函数

```rust
fn main() {
    tauri::Builder::default()
        // 1. 注册插件
        .plugin(tauri_plugin_single_instance::init(|app, _args, _cwd| {
            show_main_window(app);  // 单实例：重复启动时聚焦已有窗口
        }))
        .plugin(tauri_plugin_log::Builder::new()
            .clear_targets()
            .targets([
                tauri_plugin_log::Target::new(TargetKind::Folder { path: log_dir, file_name: Some("wsl-ui".into()) }),
                tauri_plugin_log::Target::new(TargetKind::Stdout),
            ])
            .level(if cfg!(debug_assertions) { log::LevelFilter::Debug } else { log::LevelFilter::Info })
            .build(),
        )
        .plugin(tauri_plugin_shell::init())
        .plugin(tauri_plugin_dialog::init())
        .plugin(tauri_plugin_fs::init())

        // 2. 管理全局状态
        .manage(TrayState { tray: Mutex::new(None) })

        // 3. 初始化设置
        .setup(|app| {
            // 创建托盘图标和菜单
            let tray = TrayIconBuilder::new()
                .icon(icon)
                .menu(&menu)
                .tooltip("WSL UI")
                .on_menu_event(|app, event| { /* 菜单事件 */ })
                .on_tray_icon_event(|tray, event| { /* 托盘事件 */ })
                .build(app)?;
            Ok(())
        })

        // 4. 窗口事件处理
        .on_window_event(|window, event| {
            if let tauri::WindowEvent::CloseRequested { api, .. } = event {
                // 根据设置决定关闭行为
                match settings.close_action {
                    CloseAction::Minimize => { window.hide(); api.prevent_close(); }
                    CloseAction::Quit => { /* 允许关闭 */ }
                    CloseAction::Ask => { window.emit("close-requested", ()); api.prevent_close(); }
                }
            }
        })

        // 5. 注册命令
        .invoke_handler(tauri::generate_handler![
            list_distributions,
            start_distribution,
            stop_distribution,
            get_settings,
            save_settings,
            // ... 更多命令
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

### 3.2 命令（Command）

命令是前端调用 Rust 后端的唯一方式：

```rust
#[tauri::command]
fn list_distributions() -> Result<Vec<Distribution>, WslError> {
    WslService::list_distributions()
}

#[tauri::command]
fn start_distribution(distro_id: String) -> Result<(), WslError> {
    WslService::start_distribution(&distro_id)
}

#[tauri::command]
fn get_settings() -> AppSettings {
    settings::get_settings()
}
```

**要点：**
- 参数自动从 JSON 反序列化（需实现 `Deserialize`）
- 返回值自动序列化为 JSON（需实现 `Serialize`）
- 错误类型需实现 `Into<tauri::Error>` 或使用 `Result<T, E>`

### 3.3 系统托盘

```rust
let tray = TrayIconBuilder::new()
    .icon(icon)
    .menu(&menu)
    .tooltip("WSL UI")
    .on_menu_event(|app, event| {
        match event.id.as_ref() {
            "show" => show_main_window(app),
            "quit" => app.exit(0),
            "shutdown_all" => { let _ = WslService::shutdown_all(); }
            id if id.starts_with("terminal_") => {
                let distro_name = id.strip_prefix("terminal_").unwrap_or("");
                let _ = WslService::open_terminal(distro_name, ...);
                let _ = app.emit("distro-state-changed", ());
            }
            _ => {}
        }
    })
    .on_tray_icon_event(|tray, event| {
        match event {
            TrayIconEvent::Click { button: MouseButton::Left, .. } => {
                show_main_window(tray.app_handle());
            }
            TrayIconEvent::Enter { .. } => {
                // 鼠标悬停时异步刷新菜单
                tauri::async_runtime::spawn(async move {
                    let distros = WslService::list_distributions();
                    let menu = build_tray_menu_with_distros(&app, distros);
                    tray_icon.set_menu(Some(menu));
                });
            }
            _ => {}
        }
    })
    .build(app)?;
```

### 3.4 事件系统

Rust → 前端通信：

```rust
// 发送事件到前端
app.emit("distro-state-changed", ());
window.emit("close-requested", ());
app.emit("download-progress", progress_data);
```

### 3.5 权限系统（Capabilities）

Tauri 2.x 使用 `capabilities` 替代 1.x 的 `allowlist`：

```json
// src-tauri/capabilities/default.json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "Default capabilities for WSL UI",
  "windows": ["main"],
  "permissions": [
    "core:default",
    "shell:default",
    "dialog:default",
    "fs:default",
    "log:default"
  ]
}
```

### 3.6 常用插件

| 插件 | 用途 | 安装 |
|------|------|------|
| `tauri-plugin-shell` | 执行系统命令 | `cargo add tauri-plugin-shell` |
| `tauri-plugin-dialog` | 文件对话框、消息框 | `cargo add tauri-plugin-dialog` |
| `tauri-plugin-fs` | 文件系统操作 | `cargo add tauri-plugin-fs` |
| `tauri-plugin-log` | 日志记录 | `cargo add tauri-plugin-log` |
| `tauri-plugin-single-instance` | 单实例限制 | `cargo add tauri-plugin-single-instance` |

## 四、前端开发

### 4.1 技术栈

当前项目使用：
- **React 19** — UI 框架
- **TypeScript 5** — 类型安全
- **Vite 7** — 构建工具
- **Tailwind CSS 4** — 样式
- **Zustand 5** — 状态管理
- **i18next** — 国际化

### 4.2 调用 Rust 后端

```typescript
import { invoke } from "@tauri-apps/api/core";

// 直接调用
const distros = await invoke("list_distributions");
await invoke("start_distribution", { distroId: "Ubuntu" });

// 推荐：封装为 Service 层
class WslService {
  static async listDistros(): Promise<Distribution[]> {
    return invoke("list_distributions");
  }

  static async startDistro(distroId: string): Promise<void> {
    return invoke("start_distribution", { distroId });
  }
}
```

### 4.3 状态管理（Zustand）

```typescript
import { create } from "zustand";

interface DistroStore {
  distros: Distribution[];
  loading: boolean;
  fetchDistros: () => Promise<void>;
  startDistro: (id: string) => Promise<void>;
}

const useDistroStore = create<DistroStore>((set, get) => ({
  distros: [],
  loading: false,

  fetchDistros: async () => {
    set({ loading: true });
    const distros = await WslService.listDistros();
    set({ distros, loading: false });
  },

  startDistro: async (id) => {
    await WslService.startDistro(id);
    await get().fetchDistros();  // 刷新列表
  },
}));
```

### 4.4 事件监听

```typescript
import { listen } from "@tauri-apps/api/event";

// 监听 Rust 后端事件
useEffect(() => {
  const unlisten = listen("distro-state-changed", () => {
    fetchDistros();  // 刷新数据
  });

  return () => { unlisten.then(fn => fn()); };
}, []);
```

### 4.5 轮询机制

```typescript
// usePolling hook — 封装轮询逻辑
function usePolling() {
  const { start, stop, pause, resume } = usePollingStore();

  // 窗口焦点变化时暂停/恢复轮询
  useEffect(() => {
    const unlisten = appWindow.onFocusChanged(({ payload }) => {
      if (payload.focused) resume();
      else pause();
    });
    return () => { unlisten.then(fn => fn()); };
  }, []);

  // 启动轮询
  useEffect(() => {
    start();
    return () => stop();
  }, []);
}
```

### 4.6 主题系统

```typescript
// ThemeProvider — CSS 变量注入
function applyThemeToDocument(colors: ThemeColors) {
  const root = document.documentElement;
  root.style.setProperty("--bg-primary", colors.bgPrimary);
  root.style.setProperty("--text-primary", colors.textPrimary);
  // ... 29 个 CSS 变量 + 8 个 RGB 变量
}

// 使用主题
function MyComponent() {
  const { currentTheme, setTheme } = useTheme();
  return (
    <select onChange={(e) => setTheme(e.target.value)}>
      {availableThemes.map(t => (
        <option key={t.id} value={t.id}>{t.name}</option>
      ))}
    </select>
  );
}
```

### 4.7 国际化

```typescript
// i18n 配置
i18n.use(LanguageDetector).use(initReactI18next).init({
  resources: { en: { common: commonEn, ... } },
  fallbackLng: "en",
  detection: {
    order: ["localStorage", "navigator"],
    lookupLocalStorage: "wsl-ui-language",
  },
});

// 懒加载其他语言
export async function loadLanguage(lng: string) {
  const resources = await import(`./locales/${lng}/index`);
  for (const [ns, translations] of Object.entries(resources)) {
    i18n.addResourceBundle(lng, ns, translations);
  }
}

// 使用
import { useTranslation } from "react-i18next";
function MyComponent() {
  const { t } = useTranslation("common");
  return <h1>{t("welcome")}</h1>;
}
```

## 五、配置文件详解

### 5.1 tauri.conf.json

```json
{
  "$schema": "https://schema.tauri.app/config/2",
  "productName": "WSL UI",
  "identifier": "wsl-ui",
  "build": {
    "frontendDist": "../dist",
    "devUrl": "http://localhost:1420",
    "beforeDevCommand": "npm run dev",
    "beforeBuildCommand": "npm run build"
  },
  "app": {
    "withGlobalTauri": true,
    "windows": [{
      "label": "main",
      "title": "WSL UI",
      "width": 1280,
      "height": 720,
      "minWidth": 800,
      "minHeight": 720,
      "resizable": true,
      "fullscreen": false
    }],
    "security": {
      "csp": null
    }
  },
  "bundle": {
    "active": true,
    "targets": ["nsis", "msi"],
    "resources": ["resources/*"],
    "icon": [
      "icons/32x32.png",
      "icons/128x128.png",
      "icons/icon.ico",
      "icons/icon.png"
    ],
    "windows": {
      "webviewInstallMode": { "type": "downloadBootstrapper" },
      "nsis": {
        "installMode": "currentUser",
        "sidebarImage": "icons/sidebar.bmp",
        "headerImage": "icons/header.bmp"
      }
    }
  }
}
```

### 5.2 vite.config.ts

```typescript
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    port: 1420,        // 固定端口（Tauri 要求）
    strictPort: true,  // 端口占用时直接报错
    watch: {
      ignored: ["**/src-tauri/**"],  // 不监听 Rust 目录
    },
  },
});
```

### 5.3 Cargo.toml

```toml
[package]
name = "wsl-ui"
version = "0.19.0"
edition = "2021"

[build-dependencies]
tauri-build = { version = "2.1", features = [] }

[dependencies]
tauri = { version = "2.5", features = ["tray-icon"] }
tauri-plugin-shell = "2.2"
tauri-plugin-dialog = "2.2"
tauri-plugin-fs = "2"
tauri-plugin-log = { version = "2", features = ["colored"] }
tauri-plugin-single-instance = "2"
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tokio = { version = "1", features = ["fs"] }
reqwest = { version = "0.12", features = ["stream", "json"] }

# Windows 特定依赖
[target.'cfg(windows)'.dependencies]
winreg = "0.55"

[profile.release]
panic = "abort"
codegen-units = 1
lto = true
opt-level = "s"
strip = true
```

## 六、开发流程

### 6.1 日常开发

```bash
# 启动开发模式
npm run tauri dev

# 这会同时：
# 1. 启动 Vite 开发服务器（http://localhost:1420）
# 2. 编译 Rust 后端
# 3. 启动桌面窗口
```

### 6.2 构建发布

```bash
# 构建生产安装包
npm run tauri build

# 构建调试版（不打包）
npm run tauri build --debug --no-bundle

# 构建调试版（含安装包）
npm run tauri build --debug
```

### 6.3 测试

```bash
# 单元测试（Vitest）
npm run test
npm run test:coverage

# E2E 测试（WebdriverIO）
npm run test:e2e
```

## 七、关键设计模式

### 7.1 Service 层

封装所有 `invoke` 调用，提供类型安全的 API：

```typescript
// services/wslService.ts
export class WslService {
  static async listDistros(): Promise<Distribution[]> {
    return invoke("list_distributions");
  }

  static async startDistro(id: string): Promise<void> {
    return invoke("start_distribution", { distroId: id });
  }

  static async getSettings(): Promise<AppSettings> {
    return invoke("get_settings");
  }
}
```

### 7.2 Store 模式

使用 Zustand 管理全局状态，每个功能模块一个 store：

```typescript
// store/distroStore.ts — 发行版状态
// store/settingsStore.ts — 设置状态
// store/pollingStore.ts — 轮询状态
// store/actionsStore.ts — 自定义动作状态
// store/healthStore.ts — WSL 健康状态
// store/mountStore.ts — 磁盘挂载状态
```

### 7.3 防竞态（fetchId）

```typescript
let fetchId = 0;

async function fetchDistros() {
  const currentFetch = ++fetchId;
  const distros = await WslService.listDistros();

  // 丢弃过期请求
  if (currentFetch !== fetchId) return;

  set({ distros });
}
```

### 7.4 轮询策略

```typescript
// 交错调度：防止多个轮询同时触发
const STAGGER_OFFSETS = [0, 200, 400]; // distros, resources, health

// 退避策略：连续超时时增加间隔
if (consecutiveTimeouts > 0) {
  interval = Math.min(interval * backoffMultiplier, maxInterval);
}

// 条件跳过
if (!preflightReady) return;      // WSL 未就绪
if (actionInProgress) return;     // 有操作进行中
```

### 7.5 错误处理

```typescript
// 三层错误处理
enum ErrorCode {
  NETWORK_ERROR = "NETWORK_ERROR",
  TIMEOUT = "TIMEOUT",
  WSL_NOT_INSTALLED = "WSL_NOT_INSTALLED",
  DISTRO_NOT_FOUND = "DISTRO_NOT_FOUND",
}

// 1. 解析错误
function parseError(error: unknown): AppError { ... }

// 2. 记录日志
function logError(error: AppError): void { ... }

// 3. 格式化显示
function formatError(error: AppError): string { ... }
```

### 7.6 合并策略

轮询更新时保留已有数据，避免 UI 闪烁：

```typescript
function mergeDistros(oldList: Distro[], newList: Distro[]): Distro[] {
  return newList.map(newDistro => {
    const oldDistro = oldList.find(d => d.id === newDistro.id);
    if (!oldDistro) return newDistro;

    return {
      ...newDistro,
      // 保留已有的详细数据
      diskSize: oldDistro.diskSize ?? newDistro.diskSize,
      osInfo: oldDistro.osInfo ?? newDistro.osInfo,
    };
  });
}
```

## 八、Tauri 2.x vs 1.x 对比

| 特性 | Tauri 1.x | Tauri 2.x |
|------|-----------|-----------|
| 插件系统 | 内置功能 | 独立插件（`tauri-plugin-*`） |
| 权限模型 | `allowlist` 配置 | `capabilities` + `permissions` |
| 移动端支持 | 无 | 支持 iOS/Android |
| IPC 通信 | `invoke` | `invoke` + Event |
| 包体积 | ~3MB | ~2MB |
| 系统托盘 | `systemTray` | `tray-icon` feature |
| 窗口配置 | `tauri.conf.json > tauri.windows` | `tauri.conf.json > app.windows` |

## 九、最佳实践

### 9.1 性能优化

1. **异步命令** — Rust 命令中使用 `async` 避免阻塞
2. **懒加载** — 前端路由/语言包按需加载
3. **轮询优化** — 交错调度、焦点暂停、退避策略
4. **数据合并** — 保留已有数据避免 UI 闪烁

### 9.2 用户体验

1. **单实例** — 防止重复启动
2. **托盘** — 关闭窗口时最小化到托盘
3. **事件驱动** — 后端状态变化主动通知前端
4. **错误提示** — 友好的错误信息和解决建议

### 9.3 代码组织

1. **Service 层** — 封装所有 `invoke` 调用
2. **Store 分离** — 每个功能模块独立 store
3. **类型安全** — TypeScript 类型与 Rust 结构体对应
4. **错误分类** — 统一的错误码和错误处理

## 十、常见问题

### Q: 前端如何调用 Rust 函数？

```typescript
import { invoke } from "@tauri-apps/api/core";
const result = await invoke("command_name", { param1: "value" });
```

### Q: Rust 如何发送事件到前端？

```rust
app.emit("event-name", payload);
```

### Q: 如何调试？

```bash
# 开发模式自动打开 DevTools
npm run tauri dev

# Rust 日志
RUST_LOG=debug npm run tauri dev
```

### Q: 如何打包发布？

```bash
npm run tauri build
# 输出：src-tauri/target/release/bundle/nsis/*.exe
```

## 十一、参考资源

- [Tauri 官方文档](https://tauri.app/)
- [Tauri API 文档](https://docs.rs/tauri/)
- [Tauri 插件列表](https://github.com/tauri-apps/tauri-plugin-* )
- [WSL UI 源码](https://github.com/octasoft-ltd/wsl-ui)

---

**总结**：Tauri 2.x 开发的核心模式是：
1. **Rust 后端** 通过 `#[tauri::command]` 暴露能力
2. **前端** 通过 `invoke` 调用 Rust 命令
3. **双向通信** 通过 Event 系统实现
4. **插件系统** 提供系统级功能
5. **权限控制** 通过 capabilities 声明

当前 WSL UI 项目是一个完整的参考实现，涵盖了托盘、插件、权限、状态管理、轮询、主题、国际化等所有核心功能。
