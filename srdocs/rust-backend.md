# Rust 后端架构与实现详解

## 1. 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        main.rs (应用入口)                        │
│  Tauri Builder → 插件注册 → 托盘菜单 → 窗口事件 → 命令注册(80+) │
└──────────────────────────────┬──────────────────────────────────┘
                               │
┌──────────────────────────────▼──────────────────────────────────┐
│                    commands.rs (2481 行)                         │
│            所有 #[tauri::command] 函数入口                        │
│         参数验证 → 类型转换 → 调用业务模块 → 返回结果              │
└──┬──────┬──────┬──────┬──────┬──────┬──────┬──────┬────────────┘
   │      │      │      │      │      │      │      │
   ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐┌─────┐
│ WSL ││Actns││Settg││Meta ││Dwnld││Catlg││OCI  ││Valid│
│Svc  ││     ││s    ││data ││     ││     ││     ││ation│
└──┬──┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘└─────┘
   │
   ▼
┌──────────────────────────────────────────────────────────────┐
│              wsl/ (WSL 管理核心模块)                          │
│  ┌─────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │ core.rs │ │install.rs│ │ info.rs  │ │import_   │         │
│  │ 1082行  │ │  598行   │ │  517行   │ │export.rs │         │
│  └─────────┘ └──────────┘ └──────────┘ └──────────┘         │
│  ┌──────────┐ ┌─────────┐ ┌──────────┐ ┌──────────────┐     │
│  │terminal  │ │resources│ │distro_   │ │   executor/  │     │
│  │  .rs     │ │  .rs    │ │sources.rs│ │  (ACL层)     │     │
│  └──────────┘ └─────────┘ └──────────┘ └──────────────┘     │
└──────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│              wsl/executor/ (Anti-Corruption Layer)            │
│  ┌──────────────────┐ ┌──────────────┐ ┌──────────────────┐  │
│  │  wsl_command/    │ │  terminal/   │ │   resource/      │  │
│  │  (wsl.exe CLI)   │ │  (外部进程)   │ │  (系统资源监控)   │  │
│  │  Real + Mock     │ │  Real + Mock │ │  Real + Mock     │  │
│  └──────────────────┘ └──────────────┘ └──────────────────┘  │
└──────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────┐
│              crates/wsl-core/ (独立 crate)                    │
│              parser.rs — 解析 wsl --list 输出                 │
│              types.rs  — DistroState 枚举                     │
└──────────────────────────────────────────────────────────────┘
```

## 2. 源文件清单与职责

### 2.1 顶层模块 (`src-tauri/src/`)

| 文件 | 行数 | 职责 |
|------|------|------|
| `main.rs` | 582 | 应用入口：Tauri Builder 配置、插件注册、托盘菜单构建、窗口关闭行为、80+ 命令注册 |
| `commands.rs` | 2481 | 所有 `#[tauri::command]` 入口函数，参数验证、类型转换、调用业务模块 |
| `actions.rs` | 889 | 自定义动作系统：`CustomAction` 结构体、`DistroScope` 枚举、正则缓存、默认动作嵌入 |
| `settings.rs` | 964 | 应用设置：`AppSettings` 完整结构体、`.wslconfig`/`wsl.conf` INI 解析与生成、默认设置嵌入 |
| `metadata.rs` | 783 | 发行版元数据：`DistroMetadata` + `InstallSource` 枚举，持久化到 `distro-metadata.json` |
| `download.rs` | 1000 | HTTP 下载管理器：异步下载 + 进度事件 + SHA256 校验 + Mock 支持 |
| `distro_catalog.rs` | 405 | 发行版目录：MS Store / 下载 / 容器镜像三类，默认目录编译时嵌入 |
| `error.rs` | 207 | 统一错误类型 `AppError`，含 18 个变体 + From 转换 |
| `validation.rs` | 742 | 输入验证：发行版名称、路径、端口等 |
| `oci/mod.rs` | 11 | OCI 模块入口 |
| `oci/registry.rs` | 404 | OCI Registry 通信协议实现 |
| `oci/image.rs` | 307 | OCI 镜像拉取 + rootfs 创建 |
| `oci/types.rs` | 238 | OCI 类型定义（Manifest、Config、Layer 等） |
| `utils.rs` | 103 | 工具函数：配置目录、Mock 模式检测、隐藏命令执行 |
| `constants.rs` | 37 | 全局常量 |
| `temp_file_guard.rs` | 156 | 临时文件守卫：确保退出时清理临时文件 |

### 2.2 WSL 核心模块 (`src-tauri/src/wsl/`)

| 文件 | 行数 | 职责 |
|------|------|------|
| `mod.rs` | 44 | 模块声明与 re-export |
| `service.rs` | 350 | `WslService` 门面：分类委托到各子模块 |
| `core.rs` | 1082 | WSL 核心操作：list/start/stop/delete/restart/move/rename/resize/compact/mount 等 |
| `install.rs` | 598 | 安装方式：Store 安装、容器镜像(Docker/Podman)、OCI 原生镜像 |
| `info.rs` | 517 | 信息查询：WSL 版本解析、VHDX 二进制头部读取、OS 信息、IP 地址 |
| `import_export.rs` | 350 | 导入/导出/克隆：export → import → clone，自动创建元数据 |
| `terminal.rs` | 40 | 终端集成：打开终端、文件浏览器、IDE |
| `resources.rs` | 89 | 资源监控：WSL2 VM 内存、单发行版内存/CPU、内存字符串解析 |
| `types.rs` | 443 | 类型定义：Distribution、WslError、MountedDisk、PhysicalDisk、CompactResult 等 |
| `distro_sources.rs` | 701 | 发行版源管理：HKLM 注册表操作、Manifest 解析、UAC 提权写入 |

### 2.3 执行器层 / Anti-Corruption Layer (`src-tauri/src/wsl/executor/`)

| 文件 | 行数 | 职责 |
|------|------|------|
| `mod.rs` | 259 | 全局执行器初始化（OnceLock）、Mock/Real 切换、E2E 测试 API、WSL 版本特性检测 |
| `wsl_command/mod.rs` | 147 | `WslCommandExecutor` trait 定义：27 个方法覆盖 wsl.exe 全部操作 |
| `wsl_command/real.rs` | 574 | 真实实现：调用 wsl.exe 进程 |
| `wsl_command/mock.rs` | 952 | Mock 实现：内存状态模拟、错误注入、顽固关机模拟 |
| `terminal/mod.rs` | 86 | `TerminalExecutor` trait 定义：终端检测、IDE 启动、容器运行时操作 |
| `terminal/real.rs` | 1189 | 真实实现：Windows Terminal 检测、进程启动、PowerShell 命令构建 |
| `terminal/mock.rs` | 111 | Mock 实现 |
| `resource/mod.rs` | 119 | `ResourceMonitor` trait 定义：内存/CPU 查询、注册表操作、VHDX 压缩、磁盘信息 |
| `resource/real.rs` | 855 | 真实实现：PowerShell 进程查询、注册表读写（winreg）、UAC 提权 |
| `resource/mock.rs` | 321 | Mock 实现：模拟资源数据 |

### 2.4 独立 Crate (`crates/wsl-core/`)

| 文件 | 行数 | 职责 |
|------|------|------|
| `lib.rs` | - | crate 入口 |
| `parser.rs` | - | 解析 `wsl --list -v` 文本输出为结构化数据 |
| `types.rs` | - | `DistroState` 枚举（Running/Stopped/Installing/...） |

---

## 3. 核心设计模式

### 3.1 Anti-Corruption Layer（防腐层）

这是整个后端最核心的架构决策。所有与外部系统的交互都通过三个 trait 隔离：

```rust
// wsl.exe CLI 操作
pub trait WslCommandExecutor: Send + Sync {
    fn list_verbose(&self) -> Result<CommandOutput, WslError>;
    fn start(&self, distro: &str, id: Option<&str>) -> Result<CommandOutput, WslError>;
    fn terminate(&self, distro: &str) -> Result<CommandOutput, WslError>;
    fn exec(&self, distro: &str, id: Option<&str>, command: &str) -> Result<CommandOutput, WslError>;
    // ... 共 27 个方法
}

// 外部 Windows 进程启动
pub trait TerminalExecutor: Send + Sync {
    fn open_terminal(&self, distro: &str, id: Option<&str>, terminal_command: &str) -> Result<(), WslError>;
    fn detect_container_runtime(&self) -> ContainerRuntime;
    fn container_pull(&self, runtime: &str, image: &str) -> Result<(), WslError>;
    // ... 共 12 个方法
}

// 系统资源监控
pub trait ResourceMonitor: Send + Sync {
    fn get_wsl_health(&self) -> WslHealth;
    fn get_wsl_memory_usage(&self) -> Result<u64, WslError>;
    fn compact_vhdx(&self, vhdx_path: &str) -> Result<(), WslError>;
    fn rename_distribution_registry(&self, id: &str, new_name: &str) -> Result<RenameRegistryResult, WslError>;
    // ... 共 11 个方法
}
```

**全局单例管理**：使用 `OnceLock<Arc<dyn Trait>>` 实现线程安全的全局执行器：

```rust
static WSL_EXECUTOR: OnceLock<Arc<dyn WslCommandExecutor>> = OnceLock::new();
static TERMINAL_EXECUTOR: OnceLock<Arc<dyn TerminalExecutor>> = OnceLock::new();
static RESOURCE_MONITOR: OnceLock<Arc<dyn ResourceMonitor>> = OnceLock::new();
```

启动时根据 `WSL_MOCK` 环境变量自动选择 Real 或 Mock 实现。

### 3.2 门面模式（Facade）

`WslService` 将所有 WSL 操作组织为分类方法，内部委托到子模块：

```rust
pub struct WslService;

impl WslService {
    // Core Operations
    pub fn list_distributions() -> Result<Vec<Distribution>, WslError>
    pub fn start_distribution(name: &str, id: Option<&str>) -> Result<(), WslError>
    pub fn stop_distribution(name: &str) -> Result<(), WslError>

    // Terminal & IDE
    pub fn open_terminal(name: &str, id: Option<&str>, cmd: &str) -> Result<(), WslError>

    // Import-Export
    pub fn clone_distribution(source: &str, new_name: &str, loc: Option<&str>) -> Result<(), WslError>

    // Installation
    pub fn quick_install_distribution(distro: &str, name: Option<&str>, loc: Option<&str>) -> Result<(), WslError>

    // Information
    pub fn get_wsl_version() -> Result<WslVersionInfo, WslError>

    // Resource Monitoring
    pub fn get_wsl_health() -> WslHealth

    // Disk Mount Operations
    pub fn mount_disk(disk: &str, options: MountDiskOptions) -> Result<(), WslError>
}
```

### 3.3 WSL 版本特性检测

使用 `AtomicU8` 缓存检测结果，避免重复查询：

```rust
static DISTRIBUTION_ID_SUPPORT: AtomicU8 = AtomicU8::new(0);
// 0 = 未检测, 1 = 支持, 2 = 不支持

pub fn supports_distribution_id() -> bool {
    // 检查 WSL 版本 >= 2.4.4，首次检测后缓存
}
```

---

## 4. 各模块详细实现

### 4.1 WSL 核心操作 (`wsl/core.rs` — 1082 行)

这是最大的业务逻辑模块，实现了所有 WSL 发行版管理操作：

#### 发行版列表与注册表合并

```
list_distributions()
  ├── wsl_executor().list_verbose()    // 调用 wsl --list -v
  ├── wsl_core::parser 解析输出        // 提取 name, state, version
  ├── resource_monitor().get_all_distro_registry_info()  // 查询注册表获取 GUID
  └── 合并：为每个发行版附加 id(GUID), is_default, location(BasePath)
```

#### 生命周期管理

| 操作 | 实现方式 | 特殊处理 |
|------|----------|----------|
| `start_distribution` | `wsl --distribution-id <id>` 或 `wsl -d <name>` | 支持精确 GUID 定位 |
| `stop_distribution` | `wsl --terminate` + 30 秒轮询验证 | 循环检查是否真正停止 |
| `force_stop_distribution` | `wsl --shutdown` | 关闭所有 WSL 实例 |
| `force_kill_wsl` | `wsl --shutdown --force` | 强制关闭 |
| `delete_distribution` | `wsl --unregister` | 自动清理关联元数据 |
| `restart_distribution` | stop → sleep(1s) → start | 确保完全停止后重启 |

#### 磁盘与存储管理

| 操作 | 实现方式 |
|------|----------|
| `move_distribution` | 验证已停止 → `wsl --manage --move` |
| `set_sparse` | `wsl --manage --set-sparse` 设置稀疏模式 |
| `resize_distribution` | 使用 `diskpart` 扩展 VHDX 虚拟磁盘 |
| `compact_distribution` | 发行版内 `fstrim` → `diskpart compact` → 需要 UAC 提权 |
| `set_default_user` | 通过 Windows 注册表修改默认用户 |

#### 重命名操作 (`rename_distribution`)

这是最复杂的操作之一，涉及多个子系统：
1. 注册表重命名（通过 `resource_monitor().rename_distribution_registry()`）
2. Windows Terminal 配置文件更新
3. 开始菜单快捷方式更新

#### 磁盘挂载操作

| 操作 | 实现方式 |
|------|----------|
| `mount_disk` | `wsl --mount` 支持 VHD/bare/文件系统类型/分区等选项 |
| `unmount_disk` | `wsl --unmount [disk]` |
| `list_mounted_disks` | 解析已挂载磁盘信息 |
| `list_physical_disks` | 枚举系统物理磁盘 |

#### WSL 更新

```
update_wsl()
  ├── 记录当前版本
  ├── wsl --update [--pre-release]
  ├── 获取新版本
  └── 返回版本对比信息
```

### 4.2 安装模块 (`wsl/install.rs` — 598 行)

支持三种安装方式：

#### 方式一：Microsoft Store 安装

```rust
quick_install_distribution(distro, name, location)
  ├── wsl --install <distro> --name <name> --location <location> --no-launch
  ├── 轮询验证安装完成（检查发行版是否出现在列表中）
  └── 自动创建 DistroMetadata（InstallSource::Store）
```

#### 方式二：容器镜像安装（Docker/Podman）

```rust
create_from_image(image_ref, name, location, runtime)
  ├── 检测容器运行时（podman 优先于 docker）
  ├── container_pull(image)          // 拉取镜像
  ├── container_create(image)        // 创建容器，获取 container_id
  ├── container_export(container_id) // 导出为 tar
  ├── import_distribution(tar)       // 导入为 WSL 发行版
  ├── container_rm(container_id)     // 清理容器
  └── 自动创建 DistroMetadata（InstallSource::Container）
```

#### 方式三：OCI 原生镜像安装

```rust
create_from_oci_image(image_ref, name, location)
  ├── OCI Registry 通信（直接 HTTP，无外部依赖）
  ├── 拉取镜像层 → 创建 rootfs
  ├── wsl --import(rootfs tar)
  └── 自动创建 DistroMetadata（InstallSource::Download）
```

### 4.3 信息查询模块 (`wsl/info.rs` — 517 行)

#### WSL 版本解析

```rust
get_wsl_version() -> WslVersionInfo {
    // 执行 wsl --version
    // 使用位置解析而非键名匹配（支持多语言）
    // 提取：WSL 版本、内核版本、映像版本
}
```

#### VHDX 虚拟磁盘大小

```rust
get_distribution_vhd_size(name) -> VhdSizeInfo {
    // 通过注册表 BasePath 找到 ext4.vhdx 文件
    // 直接读取 VHDX 二进制头部（前 512 字节）
    // 解析虚拟磁盘大小（非文件大小）
}
```

#### 发行版 OS 信息

```rust
get_distribution_os_info(name) -> String {
    // 在发行版内执行 cat /etc/os-release
    // 解析 PRETTY_NAME 字段
}
```

### 4.4 导入/导出/克隆 (`wsl/import_export.rs` — 350 行)

```rust
clone_distribution(source, new_name, install_location) {
    // 1. 获取源发行版 GUID（用于元数据血统追踪）
    // 2. 导出到临时文件 (wsl-clone-{pid}.tar)
    // 3. 导入为新发行版（自动创建安装目录）
    // 4. 清理临时文件
    // 5. 创建 DistroMetadata::new_clone() 记录血统
}
```

关键设计：安装目录自动创建（修复 OCT-799）：
```rust
fn ensure_install_location_exists(path) {
    std::fs::create_dir_all(path)  // 递归创建父目录
}
```

### 4.5 发行版源管理 (`wsl/distro_sources.rs` — 701 行)

管理 HKLM 注册表中的自定义发行版清单 URL：

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Lxss
  ├── DistributionListUrl      (Replace 模式 — 替换默认列表)
  └── DistributionListUrlAppend (Append 模式 — 追加到默认列表)
```

核心功能：
- **读取**：直接读 HKLM（不需要管理员权限）
- **写入**：通过 `Start-Process -Verb RunAs` PowerShell 提权（需要 UAC）
- **Manifest 解析**：解析 `ModernDistributions` JSON 格式
- **URL 验证**：仅允许 `http://`、`https://`、`file://`
- **安全防护**：Manifest 大小限制 10MB

### 4.6 执行器真实实现详解

#### WslCommandExecutor::Real (`wsl_command/real.rs` — 574 行)

所有 wsl.exe 调用通过统一的命令构建模式：
```rust
fn execute_wsl_command(&self, args: &[&str]) -> Result<CommandOutput, WslError> {
    // 使用 hidden_command() 创建隐藏窗口进程
    // 设置 stdout/stderr 管道
    // 执行并捕获输出
    // 超时处理
}
```

#### TerminalExecutor::Real (`terminal/real.rs` — 1189 行)

最复杂的执行器实现，处理：
- **Windows Terminal 检测**：查询 Store 安装信息（通过 `Get-AppxPackage`）
- **终端启动**：构建 `wt.exe` 命令参数（profile、title、command）
- **IDE 集成**：VS Code / Cursor 的 `--remote wsl+<distro>` 启动
- **容器运行时**：Podman/Docker 的 pull/create/export/rm 操作
- **文件浏览器**：`explorer.exe \\<distro>\rootfs`

#### ResourceMonitor::Real (`resource/real.rs` — 855 行)

系统级操作实现：
- **内存监控**：通过 `Get-Process wsl*` 查询 WSL2 VM 内存
- **CPU 监控**：通过 PowerShell 查询发行版进程 CPU 使用率
- **注册表操作**：使用 `winreg` crate 读写 `HKCU\Software\Microsoft\Windows\CurrentVersion\Lxss`
- **VHDX 压缩**：优先使用 `Optimize-VHD`（Hyper-V），回退到 `diskpart compact`
- **重命名**：注册表 `PackageName` 值修改 + Terminal profile 碎片文件更新 + 快捷方式重命名
- **UAC 提权**：通过 `Start-Process -Verb RunAs` + 临时文件捕获输出

### 4.7 自定义动作系统 (`actions.rs` — 889 行)

```rust
pub struct CustomAction {
    pub id: String,
    pub name: String,
    pub icon: String,
    pub command: String,           // 在发行版内执行的命令
    pub scope: DistroScope,        // 作用域
    pub confirm_before_run: bool,
    pub show_output: bool,
    pub requires_sudo: bool,
    pub requires_stopped: bool,    // 是否需要发行版停止
    pub run_in_terminal: bool,
    pub run_on_startup: bool,      // 启动时自动执行
    pub order: i32,
}

pub enum DistroScope {
    All,                           // 所有发行版
    Specific { distros: Vec<String> },  // 指定发行版
    Pattern { regex: String },     // 正则匹配
}
```

特性：
- **正则缓存**：`thread_local!` 缓存编译后的正则表达式，避免重复编译
- **默认动作**：使用 `include_str!` 在编译时嵌入默认动作 JSON
- **导入/导出**：支持 JSON 文件导入导出自定义动作配置

### 4.8 设置持久化 (`settings.rs` — 964 行)

```rust
pub struct AppSettings {
    pub polling: PollingIntervals,
    pub wsl_timeouts: WslTimeoutConfig,
    pub executables: ExecutablePaths,
    pub distro_sources: DistributionSourceSettings,
    pub terminal_command: String,
    pub ide_command: String,
    pub close_action: CloseAction,
    pub theme: String,
    pub language: String,
    pub debug_logging: bool,
    // ...
}
```

持久化机制：
- **JSON 文件**：`app-settings.json` 存储在配置目录
- **默认设置**：`include_str!("../resources/default-settings.json")` 编译时嵌入
- **INI 文件**：`.wslconfig`（全局 WSL 设置）和 `wsl.conf`（每发行版设置）的解析与生成

### 4.9 发行版元数据 (`metadata.rs` — 783 行)

```rust
pub struct DistroMetadata {
    pub distro_id: String,         // GUID
    pub distro_name: String,
    pub install_source: InstallSource,
    pub installed_at: String,      // ISO 8601 时间戳
    pub image_reference: Option<String>,
    pub download_url: Option<String>,
    pub catalog_entry: Option<String>,
    pub cloned_from: Option<String>,  // 克隆来源 GUID
    pub original_tar_path: Option<String>,
}

pub enum InstallSource {
    Store,      // Microsoft Store 安装
    Container,  // Docker/Podman 容器镜像
    Download,   // 下载的 rootfs
    Lxc,        // LXC 容器
    Import,     // 从 tar 导入
    Clone,      // 克隆自发行人
    Unknown,
}
```

持久化到 `distro-metadata.json`，版本标记 `"2.0"`。

### 4.10 下载管理器 (`download.rs` — 1000 行)

```
下载流程：
  1. 创建 reqwest Client
  2. 发送 GET 请求
  3. 使用 futures_util::StreamExt 逐块读取
  4. 每块读取后发射 Tauri Event "download-progress"
  5. 写入临时文件
  6. 完成后计算 SHA256 校验和
  7. 与预期校验和比对
  8. 重命名临时文件为最终文件名
```

Mock 模式：模拟下载过程，用于 E2E 测试。

### 4.11 OCI 原生镜像 (`oci/` — 960 行)

无需 Docker/Podman 即可拉取 OCI 容器镜像：

```
oci/registry.rs — OCI Distribution Spec 实现
  ├── GET /v2/                      → 版本检查
  ├── GET /v2/<name>/manifests/<ref> → 获取 Manifest
  └── GET /v2/<name>/blobs/<digest>  → 下载 Blob（层）

oci/image.rs — 镜像处理
  ├── 拉取所有层
  ├── 解压并合并为 rootfs
  └── 创建可导入的 tar 包

oci/types.rs — 类型定义
  ├── OciManifest
  ├── OciImageConfig
  └── OciLayer
```

### 4.12 错误处理体系 (`error.rs` — 207 行)

```rust
pub enum AppError {
    // WSL 相关
    WslCommand(String),        // 命令执行失败
    DistroNotFound(String),    // 发行版未找到
    WslParseError(String),     // 输出解析失败
    Timeout(String),           // 操作超时

    // 容器运行时
    PodmanNotInstalled,        // Podman 未安装
    PodmanCommand(String),     // Podman 命令失败

    // 配置
    ConfigRead(String),        // 读取配置失败
    ConfigWrite(String),       // 写入配置失败
    InvalidConfig(String),     // 配置值无效

    // 动作
    ActionNotFound(String),
    ActionNotApplicable { action, distro },

    // 文件/路径
    FileOperation(String),
    InvalidPath(String),

    // 验证
    Validation(String),

    // 网络/下载
    DownloadFailed(String),
    NetworkError(String),

    // 通用
    Io(io::Error),
    Json(serde_json::Error),
    Http(String),
    Other(String),
}
```

自动转换：`From<WslError>`, `From<ValidationError>`, `From<reqwest::Error>`, `From<io::Error>`

### 4.13 应用入口 (`main.rs` — 582 行)

```
main() 执行流程：
  1. 配置日志（文件 + stdout，Debug/Info 级别）
  2. 注册插件：
     - tauri-plugin-single-instance  → 单实例保证
     - tauri-plugin-log             → 日志系统
     - tauri-plugin-shell           → Shell 操作
     - tauri-plugin-dialog          → 对话框
     - tauri-plugin-fs              → 文件系统
  3. 构建托盘菜单：
     - 初始跳过 WSL 查询（避免阻塞启动）
     - 鼠标悬停时异步刷新发行版列表
     - 左键点击显示主窗口
  4. 窗口关闭行为：
     - Minimize → 隐藏到托盘
     - Quit     → 退出
     - Ask      → 发射事件让前端弹对话框
  5. 注册 80+ 个 Tauri 命令
```

---

## 5. 关键数据结构

### Distribution（发行版）

```rust
pub struct Distribution {
    pub id: String,          // Windows GUID（注册表键名）
    pub name: String,        // 显示名称
    pub state: DistroState,  // Running / Stopped / Installing / ...
    pub version: u8,         // WSL 1 或 2
    pub is_default: bool,    // 是否默认发行版
    pub location: Option<String>,  // 安装路径（来自注册表 BasePath）
}
```

### WslPreflightStatus（预检状态）

```rust
pub enum WslPreflightStatus {
    Ready,
    NotInstalled,
    FeatureDisabled,
    KernelUpdateRequired,
    VirtualizationDisabled,
    Unknown,
}
```

### 磁盘挂载类型

```rust
pub struct MountedDisk {
    pub disk_path: String,
    pub mount_point: String,
    pub fs_type: Option<String>,
}

pub struct PhysicalDisk {
    pub disk_number: u32,
    pub friendly_name: String,
    pub size_bytes: u64,
    pub partitions: Vec<DiskPartition>,
}

pub struct MountDiskOptions {
    pub vhd: bool,
    pub bare: bool,
    pub name: Option<String>,
    pub fs_type: Option<String>,
    pub options: Option<String>,
    pub partition: Option<u32>,
}
```

---

## 6. 测试与 Mock 策略

### Mock 模式架构

```
环境变量 WSL_MOCK=1
  └── init_executors()
      ├── WSL_EXECUTOR    → MockWslExecutor (952 行)
      ├── TERMINAL_EXECUTOR → MockTerminalExecutor (111 行)
      └── RESOURCE_MONITOR  → MockResourceMonitor (321 行)
```

### E2E 测试 API

```rust
// 通过 Tauri 命令暴露给 E2E 测试
set_mock_error(operation, error_type, delay_ms)  // 注入错误
clear_mock_errors()                                // 清除错误
set_stubborn_shutdown(enabled)                     // 模拟顽固发行版
was_force_shutdown_used()                          // 验证强制关机
reset_mock_state()                                 // 重置所有状态
set_mock_update_result(result)                     // 模拟更新结果
set_mock_download_cmd(url, behavior)               // 模拟下载
```

### MockWslExecutor 特性

- 内存维护发行版状态列表
- 支持错误注入（按操作名 + 延迟）
- 顽固关机模拟（graceful shutdown 失败，需要 force）
- 注册表信息模拟

---

## 7. Windows 注册表交互

后端大量使用 Windows 注册表获取/修改 WSL 信息：

| 注册表路径 | 用途 | 权限 |
|-----------|------|------|
| `HKCU\...\Lxss\{GUID}` | 发行版信息（PackageName、BasePath、DefaultUid） | 用户级 |
| `HKLM\...\Lxss\DistributionListUrl` | 替换默认发行版列表 | 管理员（UAC） |
| `HKLM\...\Lxss\DistributionListUrlAppend` | 追加发行版列表 | 管理员（UAC） |

GUID 是发行版的唯一标识符，通过注册表查询获取，用于精确的发行版定位（`--distribution-id`）。

---

## 8. 代码统计

| 类别 | 文件数 | 总行数（约） |
|------|--------|-------------|
| 顶层业务模块 | 16 | ~10,400 |
| WSL 核心模块 | 10 | ~4,200 |
| 执行器层（ACL） | 10 | ~5,300 |
| **总计** | **36** | **~20,000** |

最大文件 Top 5：
1. `commands.rs` — 2481 行
2. `wsl/core.rs` — 1082 行
3. `download.rs` — 1000 行
4. `settings.rs` — 964 行
5. `actions.rs` — 889 行
