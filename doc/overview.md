# CardputerZero Files — 工程概述

> **CardputerZero Files** 是一个为 M5Stack CardputerZero 设备开发的嵌入式文件浏览器和预览应用，使用 C++17 + LVGL 构建，采用 MVVM 架构。

---

## 目录

- [功能特性](#功能特性)
- [系统架构](#系统架构)
- [目录结构](#目录结构)
- [MVVM 分层说明](#mvvm-分层说明)
- [预览引擎](#预览引擎)
- [构建方式](#构建方式)
- [依赖项](#依赖项)
- [硬件适配](#硬件适配)
- [关键设计点](#关键设计点)

---

## 功能特性

| 功能 | 说明 |
|------|------|
| **文件浏览** | 带动画效果的文件菜单，支持长文件名自动滚动 |
| **文件操作** | 复制、剪切、粘贴、重命名、删除、查看元数据 |
| **文本预览** | 使用 Noto Sans CJK 字体渲染，支持中日韩字符，带滚动进度条 |
| **图片预览** | 支持平移、缩放、全屏、重置和 GIF 动画 |
| **音频播放** | 基于 miniaudio 引擎，支持跳转和 1x / 2x / 5x 倍速控制 |
| **视频播放** | 通过 `ffmpeg` 直接渲染到设备 framebuffer |
| **信息回退** | 不支持的格式自动降级为文件元信息展示 |

---

## 系统架构

```
main.cpp  ←  初始化 LVGL / HAL / 输入设备，启动事件循环
    │
    ▼
┌────────────────────────────────────────────┐
│               FilesApp                     │  ──  应用主控
│  ┌──────────┐  ┌──────────┐               │
│  │ FilesRouter│  │ FilesModel│             │  ──  路由 & 数据
│  └────┬─────┘  └────┬─────┘               │
│       │              │                     │
│  ┌────▼──────────────▼────┐                │
│  │   ViewModel (×2)      │  ──  视图模型   │
│  │   observable 绑定      │                │
│  └────┬──────────────┬────┘                │
│       │              │                     │
│  ┌────▼────┐   ┌────▼────┐                │
│  │BrowserView│  │PreviewView│  ──  LVGL 视图│
│  │  +Magic  │  │  +子预览  │                │
│  └─────────┘  └──────────┘                │
└────────────────────────────────────────────┘
```

**数据流**: 用户输入 → `FilesApp.onKey()` → ViewModel 方法 → Model 更新 → Observable 通知 → View 重渲染

---

## 目录结构

```
src/
├── main.cpp                        # 程序入口
│
├── core/                           # 核心层
│   ├── files_app.hpp/cpp           # 应用主控（组合所有模块）
│   ├── files_router.hpp/cpp        # 页面路由（Browser / Preview）
│   ├── files_config.hpp/cpp        # 应用配置
│   └── files_types.hpp/cpp         # 公共类型定义（PageId, FileEntry, FileKind 等）
│
├── models/                         # 数据模型层
│   ├── files_model.hpp/cpp         # 全局文件系统状态
│   ├── file_browser_model.hpp/cpp  # 浏览页面数据（目录列表、选中项）
│   ├── file_preview_model.hpp/cpp  # 预览页面数据
│   └── file_type_registry.hpp/cpp  # 文件类型 ↔ 扩展名映射注册表
│
├── view_models/                    # 视图模型层
│   ├── browser_view_model.hpp/cpp  # 浏览器 VM（文件操作、动作菜单状态）
│   ├── preview_view_model.hpp/cpp  # 预览 VM（预览类型切换、生命周期）
│   └── view_model.hpp              # ViewModel 基类
│
├── views/                          # 视图层（纯 LVGL 渲染）
│   ├── view.hpp                    # View 基类
│   ├── browser_view.hpp/cpp        # 文件浏览器主界面
│   ├── preview_view.hpp/cpp        # 预览容器界面
│   └── magic_view.hpp/cpp          # 魔法粒子动画（Box2D 物理引擎驱动）
│
├── preview/                        # 预览引擎
│   ├── preview_support.hpp/cpp     # 预览公共工具
│   ├── text/text_preview.*         # 文本预览
│   ├── image/image_preview.*       # 图片预览
│   ├── audio/                      # 音频预览（含完整 MVVM 子分层）
│   │   ├── audio_preview.*         #   音频预览主控
│   │   ├── audio_preview_model.*   #   音频数据模型
│   │   ├── audio_preview_view.*    #   音频 UI 视图
│   │   └── audio_preview_view_model.*  # 音频 VM
│   ├── video/video_preview.*       # 视频预览（ffmpeg 管道）
│   ├── info/info_preview.*         # 信息预览（降级回退）
│   └── common/                     # 预览通用组件
│       ├── bottom_key_bar.*        #   底部快捷键提示栏
│       └── miniaudio_impl.c        #   miniaudio 引擎 C 实现
│
├── hal/                            # 硬件抽象层
│   ├── files_lvgl_hal.hpp/cpp      # LVGL 显示/输入 HAL 初始化
│
├── input/                          # 输入处理
│   ├── files_keypad.hpp/cpp        # CardputerZero 物理键盘驱动
│
└── assets/                         # 资源
    ├── assets.h                    # 资源声明
    ├── convert_images.py           # 图片 -> LVGL C 数组 转换脚本
    ├── convert_fonts.py            # 字体 -> LVGL 位图 转换脚本
    ├── font_assets.hpp/cpp         # 字体资源加载
    ├── fonts/                      # 原始字体文件 (Noto Sans, Chivo Mono)
    ├── images/                     # 原始 PNG 图标
    └── res_c/                      # 编译后的 C 数组资源
```

---

## MVVM 分层说明

### View — 纯渲染，无逻辑

- 实现 `View` 虚基类（`onEnter` / `onExit` / `tick`）
- 通过 LVGL API 构建 UI 控件树
- 订阅 ViewModel 的 `Observable`，收到变化通知后更新界面
- `ponytail: 纯渲染 = 可测试、可替换`

### ViewModel — 状态管理

- 持有 Model 引用，不下放 LVGL 或输入事件给 View
- 通过 `smooth_ui_toolkit::Observable<T>` 暴露状态供 View 订阅
- 处理 View 发来的用户操作意图（打开文件 → 切换路由 / 启动预览）

### Model — 数据与业务逻辑

- `FilesModel`：文件系统实际读写（POSIX API）
- `FileBrowserModel`：当前目录列表、选中索引、文件操作（复制/删除/重命名）
- `FilePreviewModel`：预览文件路径、预览类型切换
- `FileTypeRegistry`：扩展名 → `FileKind` 映射

---

## 预览引擎

预览通过 `FileKind` 多路分发：

```
FileEntry.kind
    ├── Image  → ImagePreview  (LVGL image 控件 + pan/zoom/fs)
    ├── Audio  → AudioPreview  (miniaudio 播放 + 倍速/跳转)
    ├── Video  → VideoPreview  (fork ffmpeg → 管道 → framebuffer)
    ├── Text   → TextPreview   (文件读入 → LVGL textarea 显示)
    └── *      → InfoPreview   (文件大小/时间/类型摘要)
```

音频预览内部复用 MVVM 分层，拥有独立的 `AudioPreviewModel` / `AudioPreviewViewModel` / `AudioPreviewView`。

---

## 构建方式

### SDL 桌面测试（Linux）

```bash
cmake -S . -B build/sdl -DFILES_USE_SDL=ON
cmake --build build/sdl -j8
```

### CardputerZero 设备构建

```bash
cmake -S . -B build/cp0 -DFILES_USE_SDL=OFF
cmake --build build/cp0 -j8
```

### aarch64 交叉编译

```bash
cmake -S . -B build/cp0 \
  -DFILES_USE_SDL=OFF \
  -DCMAKE_TOOLCHAIN_FILE=cmake/aarch64-linux-gnu.cmake
cmake --build build/cp0 -j8
```

### Debian 打包

```bash
./packaging/deb/package_deb.sh
# → dist/m5cardputerzero-files_0.1.1_m5stack1_arm64.deb
```

---

## 依赖项

| 依赖 | 版本 | 用途 |
|------|------|------|
| [LVGL](https://github.com/lvgl/lvgl) | v9.5.0 | 嵌入式 GUI 框架 |
| [spdlog](https://github.com/gabime/spdlog) | v1.17.0 | 日志 |
| [smooth_ui_toolkit](https://github.com/Forairaaaaa/smooth_ui_toolkit) | v2.13.0 | 动画 / Observable 响应式绑定 |
| [miniaudio](https://github.com/mackron/miniaudio) | 0.11.25 | 音频解码与播放 |
| [Box2D](https://github.com/erincatto/box2d) | v3.1.1 | 物理引擎（MagicView 粒子动画） |
| ffmpeg | 运行时依赖 | 视频播放 |

---

## 硬件适配

### 显示

- 分辨率：`320 × 170`
- 桌面：SDL2 模拟
- 设备：Linux framebuffer（`CONFIG_LV_USE_SDL == 0`）

### 输入

- 桌面：PC 键盘映射
- 设备：CardputerZero 物理键盘
  - `F` / `X` / `Z` / `C` → 方向键
  - `Enter` 打开，`Esc` 返回，`Tab` 动作菜单

### 运行环境变量

| 变量 | 用途 |
|------|------|
| `FILES_START_DIR` | 设置起始目录 |
| `FILES_KEYBOARD_DEBUG` | 启用键盘输入调试日志 |
| `FILES_VIDEO_PULSE_SINK` | 视频音频 PulseAudio 输出 sink 名称 |

---

## 关键设计点

1. **MVVM 模式**：适用于嵌入式资源受限场景，View 和 Model 解耦清晰，ViewModel 通过 `Observable` 驱动 UI 更新
2. **统一路由**：`FilesRouter` 基于 `PageId` 枚举管理页面栈，支持单层 Browser ↔ Preview 切换
3. **文件类型注册表**：`FileTypeRegistry` 以扩展名映射驱动图标和预览引擎选择，扩展新类型只需添加映射
4. **物理动画**：`MagicView` 使用 Box2D 物理引擎驱动粒子动画——既是视觉趣味也是技术演示（"魔法"）
5. **双平台复用**：SDL 与 framebuffer 后端通过 LVGL 的编译时选项切换，最大程度共享代码
