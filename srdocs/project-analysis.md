# WSL UI 项目分析文档

## 项目概述

**WSL UI** 是一个基于 [Tauri 2.x](https://tauri.app/) 框架的 Windows 桌面应用程序，为 Windows Subsystem for Linux (WSL) 提供图形化管理界面。项目版本号为 `0.19.0`，采用 GPL-3.0-only 开源许可证（前端）与 BUSL-1.1（后端）。

- **项目名称**: wsl-ui
- **开发组织**: octasoft-ltd
- **仓库地址**: https://github.com/octasoft-ltd/wsl-ui
- **目标平台**: Windows（NSIS / MSI 安装包）
- **主窗口尺寸**: 1280x720（最小 800x720）

---

## 技术栈

### 前端

| 技术 | 版本 | 用途 |
|------|------|------|
| React | 19.2.3 | UI 框架 |
| TypeScript | 5.9.3 | 类型安全 |
| Vite | 7.2.7 | 构建工具 |
| Tailwind CSS | 4.1.18 | 样式系统 |
| Zustand | 5.0.0 | 状态管理 |
| i18next | 25.8.13 | 国际化 |
| @tauri-apps/api | 2.9.1 | Tauri 前端 API |
| Vitest | 3.2.4 | 单元测试 |
| WebdriverIO | 9.21.1 | E2E 测试 |

### 后端（Rust / Tauri）

| 技术 | 版本 | 用途 |
|------|------|------|
| Tauri | 2.5 | 桌面框架 |
| serde / serde_json | 1.x | JSON 序列化 |
| reqwest | 0.12 | HTTP 请求（下载发行版） |
| tokio | 1.x | 异步运行时 |
| regex | 1.x | 正则解析 |
| configparser | 3.x | INI 文件解析（.wslconfig / wsl.conf） |
| winreg | 0.55 | Windows 注册表操作 |
| sha2 / flate2 / tar | - | OCI 镜像解压与校验 |
| thiserror | 2.x | 错误类型定义 |

---

## 项目结构

```
wsl-ui/
├── src/                          # 前端源码（React + TypeScript）
│   ├── components/               # UI 组件
│   │   ├── icons/                # 图标组件
│   │   ├── settings/             # 设置页面子组件（16 个）
│   │   ├── ui/                   # 基础 UI 组件（Button, Dialog 等 9 个）
│   │   └── [Dialog/List/Card].tsx # 业务组件（~30 个）
│   ├── store/                    # Zustand 状态管理
│   ├── services/                 # 服务层（调用 Tauri 后端）
│   ├── hooks/                    # 自定义 Hooks
│   ├── i18n/                     # 国际化配置与翻译文件
│   ├── themes/                   # 主题系统
│   ├── types/                    # TypeScript 类型定义
│   ├── utils/                    # 工具函数
│   ├── constants/                # 常量定义
│   └── test/                     # 测试配置与 E2E 测试
├── src-tauri/                    # 后端源码（Rust）
│   ├── src/
│   │   ├── main.rs               # 应用入口（托盘、窗口、命令注册）
│   │   ├── commands.rs           # Tauri invoke 命令（~2481 行）
│   │   ├── wsl/                  # WSL 核心逻辑模块
│   │   │   ├── core.rs           # WSL 核心操作（~1082 行）
│   │   │   ├── executor/         # 命令执行器（真实 + Mock）
│   │   │   ├── install.rs        # 发行版安装逻辑
│   │   │   ├── info.rs           # 系统信息查询
│   │   │   ├── service.rs        # WSL 服务封装
│   │   │   └── types.rs          # WSL 类型定义
│   │   ├── oci/                  # OCI 镜像拉取（内置实现）
│   │   ├── actions.rs            # 自定义动作系统
│   │   ├── settings.rs           # 设置持久化
│   │   ├── metadata.rs           # 发行版元数据管理
│   │   ├── distro_catalog.rs     # 发行版目录（可下载/容器/商店）
│   │   ├── download.rs           # 下载管理器（带进度事件）
│   │   ├── validation.rs         # 输入验证
│   │   └── error.rs              # 错误类型定义
│   └── resources/                # 打包资源
├── crates/wsl-core/              # 独立 Rust crate（解析与类型）
├── scripts/                      # 构建/脚本工具
├── public/                       # 静态资源（Logo 等）
└── docs/                         # 文档与截图
```

---

## 架构设计

### 前后端通信

前端通过 `@tauri-apps/api` 的 `invoke()` 函数调用 Rust 后端暴露的命令。所有命令在 `main.rs` 中通过 `tauri::generate_handler![]` 注册，共约 **80+ 个命令**。

```
React 组件
    ↓ 调用
wslService / actionsService（服务层）
    ↓ invoke()
Tauri Commands（commands.rs）
    ↓ 调用
WslService / Settings / Metadata 等模块
    ↓ 执行
系统命令（wsl.exe / powershell.exe / diskpart 等）
```

### 状态管理（Zustand Store）

| Store | 文件 | 职责 |
|-------|------|------|
| `useDistroStore` | distroStore.ts | 发行版列表、启动/停止/删除/克隆等操作 |
| `useSettingsStore` | settingsStore.ts | 应用设置加载与保存 |
| `useActionsStore` | actionsStore.ts | 自定义动作管理 |
| `useMountStore` | mountStore.ts | 磁盘挂载对话框状态 |
| `useNotificationStore` | notificationStore.ts | 通知消息管理 |
| `usePreflightStore` | preflightStore.ts | WSL 预检测状态 |
| `usePollingStore` | pollingStore.ts | 轮询配置 |
| `useResourceStore` | resourceStore.ts | 资源监控数据 |
| `useHealthStore` | healthStore.ts | WSL 健康状态 |
| `useConfigPendingStore` | configPendingStore.ts | .wslconfig 待重启提示 |

### 数据流

- **轮询机制**: `usePolling` hook 集中管理定时刷新（发行版列表、资源、健康状态）
- **事件驱动**: 后端通过 Tauri Event 发送 `distro-state-changed`、`close-requested`、`download-progress` 等事件
- **合并策略**: `fetchDistros` 使用 fetchId 防止过期请求覆盖新数据，并合并已有数据避免 UI 闪烁

---

## 核心功能模块

### 1. 发行版管理

- **列表展示**: 显示所有 WSL 发行版（名称、状态、版本、磁盘大小、OS 信息）
- **生命周期**: 启动 / 停止 / 强制停止 / 重启 / 删除
- **克隆**: 通过 export + import 实现发行版克隆
- **导入/导出**: tar 格式导入导出
- **移动**: 更改发行版安装位置
- **重命名**: 修改发行版名称（含注册表、终端配置、快捷方式）
- **调整大小**: 调整 VHDX 虚拟磁盘大小
- **压缩**: 回收 VHDX 未使用空间（需管理员权限）
- **版本切换**: WSL 1 ↔ WSL 2 转换
- **默认用户**: 设置发行版默认登录用户
- **稀疏磁盘**: 启用/禁用自动空间回收

### 2. 新建发行版

- **MS Store 安装**: 通过 `wsl --install` 快速安装
- **下载安装**: 从 URL 下载 rootfs 并导入（带进度条）
- **OCI 容器镜像**: 内置 OCI 实现，支持 Docker/Podman 作为备选
- **LXC 目录浏览**: 从 LXC 镜像目录安装发行版
- **自定义镜像**: 从本地文件创建

### 3. 终端与远程桌面

- **打开终端**: 支持 Windows Terminal / 自定义终端命令
- **远程桌面（RDP）**: 自动检测 xrdp 服务，启动 mstsc.exe 连接
- **文件管理器**: 通过 `\\wsl$` UNC 路径打开
- **IDE 集成**: 支持 VS Code 等 IDE 快速打开

### 4. 自定义动作系统

- 用户可定义在发行版内执行的自定义 Shell 命令
- 支持 `runOnStartup`：发行版启动时自动执行
- 支持 `showOutput`：显示命令输出
- 支持导入/导出动作配置（JSON 格式）

### 5. WSL 全局配置

- **.wslconfig 编辑**: 内存、处理器、交换空间、网络模式等
- **wsl.conf 编辑**: 每个发行版的自动挂载、网络、interop、systemd 配置
- **GPU 状态检测**: 检测 DirectX / NVIDIA GPU 可用性
- **NVIDIA Container Toolkit**: 检测 CDI 规范状态

### 6. 磁盘挂载

- 挂载物理磁盘到 WSL
- 支持 VHD 挂载
- 列出已挂载磁盘与可用物理磁盘

### 7. 发行版目录管理

- 内置可下载发行版列表（Ubuntu、Debian、Kali 等）
- 容器镜像支持（docker.io 等）
- MS Store 发行版元数据
- 用户可添加自定义发行版源

### 8. 设置系统

| 设置项 | 说明 |
|--------|------|
| IDE 命令 | 默认 `code`（VS Code） |
| 终端命令 | `auto` 自动检测或自定义 |
| 语言 | 15 种语言（含 RTL 阿拉伯语） |
| 关闭行为 | 询问 / 最小化到托盘 / 退出 |
| 遥测 | Aptabase 匿名使用统计 |
| 轮询 | 可配置刷新间隔 |
| WSL 超时 | 快速/默认/长时间操作超时 |
| 可执行路径 | WSL/PowerShell/CMD/Explorer 路径 |
| 容器运行时 | 内置 / Docker / Podman / 自定义 |
| 安装路径 | 默认安装基础路径 |
| 调试日志 | 运行时切换日志级别 |

---

## 主题系统

内置 **17 个主题**，分为三类：

### 暗色主题（8 个）
- Mission Control（青色/深色，原始主题）
- Obsidian（暖石/琥珀色，默认）
- Cobalt（深蓝/金色）
- Dracula（紫/粉）
- Nord（北极蓝）
- Solarized Dark
- Monokai
- GitHub Dark

### 中间色调主题（4 个）
- Slate Dusk（蓝灰/紫罗兰）
- Forest Mist（鼠尾草/琥珀）
- Rose Quartz（玫瑰/紫红）
- Ocean Fog（蓝灰/水鸭色）

### 亮色主题（3 个）
- Daylight（明亮/天蓝）
- Mission Control Light
- Obsidian Light

### 无障碍主题（2 个）
- High Contrast（纯黑/纯白，最大对比度）
- High Contrast Light

支持**自定义主题**，用户可修改所有颜色值。

---

## 国际化（i18n）

支持 **15 种语言**，采用懒加载策略（英文内联，其他语言按需加载）：

| 语言 | 代码 | 备注 |
|------|------|------|
| English | en | 默认，内联打包 |
| 简体中文 | zh-CN | |
| 繁體中文 | zh-TW | |
| 日本語 | ja | |
| 한국어 | ko | |
| Español | es | |
| हिन्दी | hi | |
| Français | fr | |
| Deutsch | de | |
| Português (Brasil) | pt-BR | |
| العربية | ar | RTL 支持 |
| Русский | ru | |
| Polski | pl | |
| Türkçe | tr | |
| Italiano | it | |

翻译命名空间：`common`, `header`, `dashboard`, `dialogs`, `settings`, `actions`, `install`, `errors`, `help`, `statusbar`

---

## Rust 后端模块详解

### 命令执行器架构（Executor Pattern）

后端使用策略模式隔离 WSL 命令执行，支持真实环境与 Mock 测试：

```
wsl/executor/
├── mod.rs              # Executor trait 定义
├── wsl_command/
│   ├── mod.rs          # WslCommandExecutor trait
│   ├── real.rs         # 真实 wsl.exe 调用
│   └── mock.rs         # Mock 模式（E2E 测试用）
├── terminal/
│   ├── mod.rs          # TerminalExecutor trait
│   ├── real.rs         # 真实终端启动（wt.exe / 自定义）
│   └── mock.rs         # Mock 终端
└── resource/
    ├── mod.rs          # ResourceExecutor trait
    ├── real.rs         # 真实资源查询（tasklist / wsl.exe）
    └── mock.rs         # Mock 资源数据
```

### 关键 Rust 模块

| 模块 | 行数 | 职责 |
|------|------|------|
| commands.rs | ~2481 | 所有 Tauri 命令入口 |
| wsl/core.rs | ~1082 | 发行版列表/启动/停止/删除等核心逻辑 |
| download.rs | ~1000 | HTTP 下载 + 进度事件发射 |
| settings.rs | ~964 | 设置读写（JSON 文件持久化） |
| actions.rs | ~889 | 自定义动作 CRUD + 执行 |
| metadata.rs | ~783 | 发行版元数据（GUID → 安装来源） |
| validation.rs | ~742 | 名称/路径验证 |
| wsl/distro_sources.rs | ~701 | 发行版注册源管理 |
| wsl/install.rs | ~598 | 安装流程（Store/下载/OCI） |
| wsl/info.rs | ~517 | 系统信息（版本/IP/GPU） |
| oci/registry.rs | ~404 | OCI Registry 通信 |
| distro_catalog.rs | ~405 | 发行版目录管理 |
| wsl/import_export.rs | ~350 | tar 导入导出 |
| wsl/service.rs | ~350 | WSL 服务封装层 |
| oci/image.rs | ~307 | OCI 镜像解析 |

---

## 测试策略

### 单元测试（Vitest）

- 位于 `src/` 目录内，与源码并列（`*.test.ts` / `*.test.tsx`）
- 使用 `@testing-library/react` 测试 React 组件
- 使用 `jsdom` 环境
- 覆盖：组件、Store、Hook、工具函数

### E2E 测试（WebdriverIO）

- 位于 `src/test/e2e/`
- 使用 Mock 模式（`WSL_MOCK=1`）运行
- 支持截图生成与演示视频录制
- 配置：`wdio.conf.ts`

### Rust 测试

- `wsl-core` crate 内含解析器测试
- 后端使用 `wiremock` 进行 HTTP Mock 测试

---

## 构建与发布

### 开发

```bash
npm run dev          # 启动 Vite 开发服务器（端口 1420）
npm run tauri:dev    # 启动 Tauri 开发模式（Rust + 前端热重载）
```

### 构建

```bash
npm run tauri:build          # 生产构建（NSIS + MSI）
npm run tauri:build:debug    # 调试构建（不打包）
npm run tauri:build:quick    # 快速打包构建
```

### 测试

```bash
npm run test             # Vitest 监视模式
npm run test:run         # 单次运行单元测试
npm run test:coverage    # 覆盖率报告
npm run test:e2e         # E2E 测试
```

### 发布流程

- 使用 **Release Please** 自动化版本管理与 CHANGELOG 生成
- 配置文件：`release-please-config.json`、`.release-please-manifest.json`
- GitHub Actions 工作流位于 `.github/workflows/`

---

## 安全与隐私

- **CSP**: 未设置（`csp: null`），依赖 Tauri 默认安全策略
- **单实例**: 使用 `tauri-plugin-single-instance` 防止多开
- **权限**: 部分操作（磁盘压缩、注册表修改）需 UAC 提权
- **遥测**: 使用 Aptabase，默认关闭，用户主动开启
- **隐私政策**: 见 `docs/PRIVACY.md`

---

## 关键设计决策

1. **Tauri 而非 Electron**: 更小的包体积与内存占用，利用系统 WebView2
2. **Zustand 而非 Redux**: 轻量级状态管理，减少样板代码
3. **Executor 模式**: 隔离系统命令执行，便于 Mock 测试
4. **懒加载 i18n**: 英文内联，其他语言按需加载，减小初始包大小
5. **fetchId 防竞态**: 使用递增 ID 丢弃过期的异步请求结果
6. **合并策略**: 轮询时保留已有的 diskSize/osInfo 避免 UI 闪烁
7. **内置 OCI**: 不依赖 Docker 即可拉取容器镜像
8. **托盘常驻**: 支持最小化到系统托盘，后台保持运行
9. **异步托盘菜单**: 鼠标悬停时异步加载发行版列表，避免阻塞启动

---

## 文件统计

| 类别 | 数量 |
|------|------|
| 前端组件 | ~50 个 |
| Zustand Store | 10 个 |
| 服务层 | 4 个 |
| Rust 源文件 | ~36 个 |
| i18n 语言 | 15 种 |
| i18n 命名空间 | 10 个 |
| 翻译文件 | ~164 个 JSON |
| 内置主题 | 17 个 |
| Tauri 命令 | ~80+ 个 |
| 测试文件 | ~20+ 个 |
