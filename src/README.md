# Hyprland 源码架构说明

## 概览

Hyprland 是一个基于 Wayland 的动态平铺窗口管理器（Compositor），其源码结构清晰地体现了现代 Wayland 合成器的架构设计。本文档详细介绍 `src/` 目录的组织结构和各模块功能。

## 核心入口文件

### `main.cpp` (283 行)

程序入口点，主要职责：
- 命令行参数解析
- 环境变量初始化（`XDG_RUNTIME_DIR`、`XDG_SESSION_TYPE`、`WAYLAND_DISPLAY` 等）
- 支持参数：
  - `--config FILE` / `-c FILE` - 指定配置文件
  - `--safe-mode` - 安全模式启动
  - `--verify-config` - 仅验证配置
  - `--wayland-fd FD` - Wayland socket 文件描述符
  - `--socket NAME` - Wayland socket 名称
  - `--watchdog-fd FD` - 看门狗文件描述符
  - `--systeminfo` - 打印系统信息
  - `--version` / `-v` - 显示版本
  - `--help` / `-h` - 显示帮助
- 创建 `CCompositor` 实例并启动主循环

### `Compositor.cpp/hpp` (3248 行 / 200 行)

Hyprland 的核心类，包含：
- **核心成员**：
  - `m_wlDisplay` - Wayland 显示服务器
  - `m_wlEventLoop` - 事件循环
  - `m_aqBackend` - Aquamarine 后端
  - `m_monitors` - 显示器列表
  - `m_windows` - 窗口列表
  - `m_layers` - 图层列表
  - `m_workspaces` - 工作区列表

- **主要方法**：
  - `initServer()` - 初始化 Wayland 服务器
  - `startCompositor()` - 启动合成器主循环
  - `getMonitorFromID()` / `getMonitorFromName()` - 显示器查询
  - `getWindowFromSurface()` - 从 surface 获取窗口
  - `getWorkspaceByID()` / `getWorkspaceByName()` - 工作区查询
  - `setWindowFullscreenInternal()` - 全屏管理
  - `moveWorkspaceToMonitor()` - 工作区迁移
  - `scheduleFrameForMonitor()` - 帧调度

- **全局实例**：`g_pCompositor`

---

## 主要目录结构

### 1. `desktop/` - 桌面对象管理

桌面环境的核心抽象层，管理所有可见对象。

#### `view/` - 视图系统
视图是所有可见元素的抽象基类：

| 文件 | 功能 |
|------|------|
| `View.cpp/hpp` | 视图基类，定义通用接口 |
| `Window.cpp/hpp` | 普通应用窗口（XDG Shell） |
| `LayerSurface.cpp/hpp` | 图层表面（状态栏、通知、壁纸） |
| `Popup.cpp/hpp` | 弹出窗口（菜单、工具提示） |
| `Subsurface.cpp/hpp` | 子表面（嵌套 surface） |
| `WLSurface.cpp/hpp` | Wayland 表面封装 |
| `Group.cpp/hpp` | 窗口组管理（标签页） |
| `SessionLock.cpp/hpp` | 会话锁屏界面 |
| `GlobalViewMethods.cpp/hpp` | 全局视图方法 |

#### `rule/` - 规则引擎
灵活的规则匹配和应用系统：

```
rule/
├── Engine.cpp/hpp              # 规则引擎核心
├── Rule.hpp                    # 规则基类
├── RuleWithEffects.hpp         # 带效果的规则
├── windowRule/                 # 窗口规则
│   ├── WindowRule.cpp/hpp      # 窗口规则定义
│   ├── WindowRuleApplicator    # 规则应用器
│   └── WindowRuleEffectContainer # 效果容器
├── layerRule/                  # 图层规则
│   ├── LayerRule.cpp/hpp
│   ├── LayerRuleApplicator
│   └── LayerRuleEffectContainer
├── matchEngine/                # 匹配引擎
│   ├── RegexMatchEngine        # 正则表达式匹配
│   ├── TagMatchEngine          # 标签匹配
│   ├── IntMatchEngine          # 整数匹配
│   └── WorkspaceMatchEngine    # 工作区匹配
└── utils/
    └── SetUtils.hpp            # 集合工具
```

规则示例：
- 窗口规则：位置、大小、浮动、全屏、不透明度、动画
- 图层规则：模糊、忽略、层级

#### `history/` - 历史记录
窗口和工作区的历史追踪：

- `WindowHistoryTracker` - 窗口焦点历史
- `WorkspaceHistoryTracker` - 工作区切换历史

#### 其他子目录

- `state/` - 状态管理
- `reserved/` - 保留区域管理（面板、Dock 等占用的空间）
- `types/` - 类型定义
  - `MultiAnimatedVariable.hpp` - 多值动画变量
  - `OverridableVar.hpp` - 可覆盖变量

#### 根文件

- `Workspace.cpp/hpp` - 工作区实现
- `DesktopTypes.hpp` - 桌面类型定义

---

### 2. `managers/` - 管理器集合

Hyprland 采用管理器模式，每个功能模块由专门的管理器负责：

#### 核心管理器

| 管理器 | 功能 |
|--------|------|
| `KeybindManager` | 键盘绑定和快捷键处理 |
| `PointerManager` | 指针设备管理 |
| `CursorManager` | 光标状态和渲染 |
| `XCursorManager` | X11 光标主题 |
| `SeatManager` | Wayland seat（输入源集合） |
| `EventManager` | 事件分发和回调 |
| `ProtocolManager` | Wayland 协议注册 |
| `SessionLockManager` | 会话锁定（锁屏） |
| `XWaylandManager` | XWayland 兼容层 |
| `TokenManager` | XDG 激活令牌 |
| `ANRManager` | 应用无响应检测 |
| `VersionKeeperManager` | 版本管理和更新检查 |
| `DonationNagManager` | 捐赠提示 |
| `WelcomeManager` | 欢迎界面和首次启动 |

#### 子目录

- `animation/` - 动画系统（缓动函数、时间线）
- `cursor/` - 光标处理（点击、拖拽、悬停）
- `eventLoop/` - 事件循环管理
- `input/` - 输入设备（键盘、鼠标、触摸板、数位板）
- `permissions/` - 权限管理
- `screenshare/` - 屏幕共享

---

### 3. `render/` - 渲染系统

基于 OpenGL 的高性能渲染引擎。

#### 核心渲染类

| 文件 | 功能 |
|------|------|
| `Renderer.cpp/hpp` | 渲染器抽象基类 |
| `GLRenderer.cpp/hpp` | OpenGL 渲染器实现 |
| `OpenGL.cpp/hpp` | OpenGL 上下文和扩展管理 |
| `ElementRenderer.cpp/hpp` | UI 元素渲染 |
| `Texture.cpp/hpp` | 纹理管理（加载、缓存、绑定） |
| `Framebuffer.cpp/hpp` | 帧缓冲对象（FBO） |
| `Renderbuffer.cpp/hpp` | 渲染缓冲对象（RBO） |
| `Shader.cpp/hpp` | 着色器程序 |
| `ShaderLoader.cpp/hpp` | 着色器编译和加载 |
| `Transformer.cpp/hpp` | 坐标变换和矩阵运算 |
| `SyncFDManager.cpp/hpp` | 同步文件描述符 |
| `AsyncResourceGatherer.hpp` | 异步资源收集 |

#### 子目录

- **`gl/`** - OpenGL 特定实现
- **`shaders/`** - GLSL 着色器代码
  - 顶点着色器、片段着色器
  - 特效着色器（模糊、阴影、圆角）
- **`pass/`** - 多通道渲染
  - 背景层、窗口层、覆盖层
  - 后处理效果
- **`decorations/`** - 窗口装饰
  - 边框渲染
  - 阴影渲染
  - 圆角处理

#### 渲染流程

1. 收集脏区域（damage regions）
2. 按层级排序渲染对象
3. 设置 FBO 和渲染目标
4. 应用变换矩阵
5. 绘制纹理和装饰
6. 应用后处理效果
7. 提交到显示器

---

### 4. `protocols/` - Wayland 协议实现

Hyprland 支持 **100+ 个 Wayland 协议**，这是其强大兼容性的基础。

#### 核心协议

##### 窗口管理
- `XDGShell` - 标准应用窗口协议
- `LayerShell` - wlroots 图层协议（用于状态栏、Dock）
- `ForeignToplevel` / `ForeignToplevelWlr` - 外部窗口列表
- `XDGDecoration` - 服务器端窗口装饰（SSD）
- `ServerDecorationKDE` - KDE 装饰协议
- `XDGDialog` - 对话框提示
- `XWaylandShell` - XWayland 集成

##### 输入协议
- `InputMethodV2` - 输入法框架
- `TextInputV1` / `TextInputV3` - 文本输入
- `VirtualKeyboard` - 虚拟键盘（用于远程桌面）
- `VirtualPointer` - 虚拟指针
- `PointerConstraints` - 指针锁定和限制（游戏）
- `PointerGestures` - 手势识别（捏合、滑动）
- `RelativePointer` - 相对指针移动（FPS 游戏）
- `CursorShape` - 光标形状协议
- `Tablet` - 数位板支持
- `PointerWarp` - 指针传送

##### 显示和输出
- `OutputManagement` - 显示器配置（分辨率、刷新率、位置）
- `OutputPower` - 显示器电源管理（DPMS）
- `XDGOutput` - 输出元数据（名称、描述、逻辑尺寸）
- `FractionalScale` - 分数缩放（1.25x、1.5x）
- `Viewporter` - 视口裁剪和缩放
- `PresentationTime` - 精确的帧呈现时间
- `TearingControl` - 画面撕裂控制（VRR/G-Sync）
- `GammaControl` - 伽马曲线调整
- `ColorManagement` / `CTMControl` - 色彩管理

##### 屏幕捕获
- `Screencopy` - wlroots 屏幕截图协议
- `ToplevelExport` - 窗口截图
- `ImageCaptureSource` - 捕获源管理
- `ImageCopyCapture` - 图像复制捕获

##### 数据传输
- `DataDeviceWlr` - wlroots 数据设备
- `ExtDataDevice` - 扩展数据设备
- `PrimarySelection` - 主选择（中键粘贴）
- `LinuxDMABUF` - DMA-BUF 零拷贝内存共享

##### 安全和会话
- `SessionLock` - 会话锁定协议
- `LockNotify` - 锁定通知
- `SecurityContext` - 安全上下文（沙箱应用）
- `GlobalShortcuts` - 全局快捷键注册
- `ShortcutsInhibit` - 快捷键抑制
- `FocusGrab` - 焦点抓取

##### 空闲和电源
- `IdleInhibit` - 空闲抑制（播放视频时）
- `IdleNotify` - 空闲通知

##### 其他功能协议
- `XDGActivation` - 窗口激活请求
- `XDGForeignV2` - 跨进程 surface 引用
- `XDGTag` - 窗口标签
- `XDGBell` - 铃声通知
- `AlphaModifier` - Alpha 通道修改
- `ContentType` - 内容类型提示（游戏、视频、照片）
- `CommitTiming` - 提交时机控制
- `HyprlandSurface` - Hyprland 特有扩展
- `ExtWorkspace` - 工作区协议
- `ToplevelMapping` - 顶层窗口映射
- `SinglePixel` - 单像素缓冲
- `Fifo` - FIFO 同步
- `DRMLease` - DRM 租赁（VR 直通）
- `DRMSyncobj` - DRM 同步对象
- `MesaDRM` - Mesa DRM 扩展

#### 子目录

- **`core/`** - 核心 Wayland 协议（compositor、surface、region）
- **`types/`** - 协议类型定义
- `WaylandProtocol.cpp/hpp` - 协议基类

---

### 5. `config/` - 配置系统

灵活的配置管理系统，支持多种配置格式。

#### 核心类

- `ConfigManager.cpp/hpp` - 配置管理器
  - 配置文件解析
  - 配置值存储和查询
  - 配置热重载
  - 配置验证
- `ConfigValue.cpp/hpp` - 配置值抽象类
  - 类型安全的配置值
  - 默认值管理
  - 值变更通知

#### 子目录

- **`legacy/`** - 传统配置格式
  - Hyprland 配置语言（HCL）
  - 兼容旧版本配置
  
- **`lua/`** - Lua 配置支持
  - Lua 脚本配置
  - 动态配置逻辑
  
- **`shared/`** - 共享配置对象
  - `workspace/WorkspaceRule.hpp` - 工作区规则
  
- **`supplementary/`** - 补充配置
  - 主题、颜色、字体
  
- **`values/`** - 配置值类型
  - 整数、浮点、字符串、布尔
  - 颜色、渐变
  - 列表、映射

#### 配置示例类型

- 通用设置（gaps、borders、rounding）
- 动画配置（速度、曲线、启用状态）
- 输入设备配置（灵敏度、加速度）
- 窗口规则（windowrule、windowrulev2）
- 工作区规则（workspace）
- 绑定（bind、bindm、binde、bindr）
- 环境变量（env）
- 执行命令（exec、exec-once）
- 显示器配置（monitor）

---

### 6. `layout/` - 布局算法

窗口布局和空间管理系统。

#### 子目录

- **`algorithm/`** - 布局算法实现
  - Dwindle（递归分割）
  - Master（主从布局）
  - Spiral（螺旋布局）
  
- **`space/`** - 空间管理
  - 可用空间计算
  - 保留区域处理
  
- **`supplementary/`** - 补充布局功能
  - 全屏管理
  - 浮动窗口
  
- **`target/`** - 布局目标
  - 窗口定位
  - 尺寸计算

#### 布局特性

- 动态平铺（dynamic tiling）
- 伪平铺（pseudo-tiling）
- 浮动窗口
- 全屏模式（真全屏、最大化全屏）
- 窗口组（标签页式）

---

### 7. `helpers/` - 辅助工具

通用辅助类和工具函数集合。

#### 显示器相关

- `Monitor.cpp/hpp` - 显示器辅助类
  - 显示器属性管理
  - 工作区关联
- `MonitorFrameScheduler.cpp/hpp` - 帧调度器
  - VRR（可变刷新率）
  - 帧时间预测
- `MonitorZoomController.cpp/hpp` - 缩放控制器
  - 无障碍缩放
- `MonitorResources.cpp/hpp` - 显示器资源

#### 图形和颜色

- `Color.cpp/hpp` - 颜色处理
  - RGB/HSV 转换
  - 颜色插值
- `Format.cpp/hpp` - 像素格式转换
  - RGBA、BGRA、XRGB 等
- `DamageRing.cpp/hpp` - 损伤环
  - 脏区域追踪
  - 部分重绘优化
- `TransferFunction.cpp/hpp` - 传输函数
  - sRGB、线性、PQ、HLG

#### 动画和数学

- `AnimatedVariable.hpp` - 动画变量
  - 平滑插值
  - 缓动函数
- `math/` - 数学工具
  - 向量运算
  - 方向枚举（Direction.hpp）
  - 几何计算

#### 系统和环境

- `MiscFunctions.cpp/hpp` - 杂项函数
  - 字符串处理
  - 路径操作
- `SystemInfo.cpp/hpp` - 系统信息
  - CPU、GPU、内存
  - 发行版检测
- `Drm.cpp/hpp` - DRM 相关
  - DRM 设备查询
- `SdDaemon.cpp/hpp` - systemd 守护进程
  - sd_notify 支持

#### 内存和资源

- `memory/` - 内存管理
  - 智能指针扩展
  - 内存池
- `signal/` - 信号处理
  - 观察者模式
  - 信号连接
- `sync/` - 同步原语
  - 互斥锁、条件变量
- `defer/` - 延迟执行
  - RAII 风格的作用域退出

#### 其他工具

- `env/` - 环境变量
  - 环境变量读写
  - 环境设置
- `fs/` - 文件系统
  - 路径操作
  - 文件读写
- `time/` - 时间工具
  - 时间戳、计时器
- `varlist/` - 变量列表
  - 命令行参数解析
- `cm/` - 色彩管理
  - ICC profile
  - 色彩空间转换
- `WLClasses.cpp/hpp` - Wayland 类包装
- `TagKeeper.cpp/hpp` - 标签管理
- `AsyncDialogBox.cpp/hpp` - 异步对话框
- `ByteOperations.hpp` - 字节操作
- `CursorShapes.hpp` - 光标形状定义
- `Splashes.hpp` - 启动欢迎语

---

### 8. `devices/` - 设备管理

输入设备的抽象和管理：

- 键盘设备
- 指针设备（鼠标、触摸板）
- 触摸屏
- 数位板
- 游戏手柄

每种设备都有独立的配置和事件处理。

---

### 9. `xwayland/` - XWayland 集成

X11 兼容层，使 Hyprland 能够运行传统 X11 应用程序。

功能：
- X11 窗口到 Wayland surface 的映射
- X11 原子（atoms）处理
- X11 事件转换
- ICCCM/EWMH 协议支持
- X11 选择（剪贴板）集成
- XWayland 进程管理

---

### 10. `debug/` - 调试系统

完善的调试和诊断工具。

#### 子目录

- **`log/`** - 日志系统
  - 多级别日志（DEBUG、INFO、WARN、ERROR、CRITICAL）
  - 日志输出到文件和控制台
  - 彩色日志格式化
  
- **`crash/`** - 崩溃处理
  - 崩溃信号捕获
  - 栈回溯（backtrace）
  - 崩溃报告生成

#### HyprCtl

运行时控制接口，允许通过 socket 控制 Hyprland：

```bash
# 示例命令
hyprctl monitors          # 查看显示器信息
hyprctl clients           # 查看窗口列表
hyprctl workspaces        # 查看工作区
hyprctl dispatch          # 执行调度命令
hyprctl keyword           # 修改配置
hyprctl version           # 查看版本
hyprctl systeminfo        # 系统信息
```

支持多种输出格式：
- 普通文本（human-readable）
- JSON（机器可读）
- 用于脚本集成

---

### 11. `event/` - 事件处理

事件系统和回调管理：

- 窗口事件（打开、关闭、移动、调整大小）
- 工作区事件（创建、销毁、切换）
- 显示器事件（连接、断开）
- 输入事件（键盘、鼠标）
- 配置事件（重载）

事件可以被插件订阅，实现自定义功能。

---

### 12. `init/` - 初始化

启动和初始化相关代码：

- `initHelpers.hpp` - 初始化辅助函数
  - 实时优先级设置（`gainRealTime()`）
  - Sudo 检查（`isSudo()`）
  - 环境检查

初始化流程：
1. 参数解析
2. 环境变量设置
3. 创建 Compositor
4. 初始化管理器（分阶段：PRIORITY、BASICINIT、LATE）
5. 加载配置
6. 启动 Wayland 服务器
7. 启动 XWayland
8. 进入主循环

---

### 13. `plugins/` - 插件系统

Hyprland 的插件 API，允许动态扩展功能。

插件能力：
- 注册自定义调度命令
- 订阅事件回调
- 添加窗口装饰
- 实现自定义布局
- 注册配置选项
- 访问 Hyprland 内部 API

插件加载机制：
- 共享库（.so）动态加载
- 版本兼容性检查
- 符号解析和初始化

---

### 14. `notification/` - 通知系统

桌面通知支持，用于：
- 错误提示
- 警告信息
- 配置重载通知
- 插件消息
- 自定义通知

与 layer-shell 协议集成，显示通知窗口。

---

### 15. `i18n/` - 国际化

多语言支持：

- 翻译字符串管理
- 区域设置（locale）处理
- UTF-8 文本处理
- 输入法集成

---

### 16. `errorOverlay/` - 错误覆盖层

错误信息显示系统：

- 配置错误提示
- 崩溃信息显示
- 警告覆盖层
- 用户友好的错误消息

在屏幕上直接显示错误，而不是仅在日志中。

---

### 17. `pch/` - 预编译头

预编译头文件目录，用于加速编译：

- 标准库头文件
- 常用第三方库头文件
- Wayland/wlroots 头文件

通过预编译这些很少改变的头文件，显著减少编译时间。

---

## 其他重要文件

### 全局头文件

- **`defines.hpp`** - 全局定义
  - 类型别名（`PHLWINDOW`、`PHLMONITOR` 等）
  - 枚举定义
  - 常量定义

- **`includes.hpp`** - 通用头文件包含
  - 标准库
  - 第三方库
  - 平台特定头文件

- **`macros.hpp`** (10KB+) - 宏定义
  - 调试宏
  - 日志宏
  - 断言宏
  - 性能测量宏

- **`SharedDefs.hpp`** - 共享定义
  - 跨模块共享的常量
  - 接口定义

- **`version.h.in`** - 版本信息模板
  - 版本号（major、minor、patch）
  - 提交哈希
  - 构建日期
  - 分支信息

---

## 架构特点

### 1. 模块化设计

每个功能模块都有独立的管理器，职责明确：
- 单一职责原则
- 松耦合
- 易于测试和维护

### 2. 协议丰富

支持 **100+ 个 Wayland 协议**：
- 核心协议完整实现
- wlroots 扩展协议
- KDE/GNOME 特定协议
- 游戏和 VR 专用协议
- 极佳的应用兼容性

### 3. 规则引擎

灵活的窗口规则匹配系统：
- 正则表达式匹配
- 标签匹配
- 条件组合
- 动态规则应用

示例：
```
windowrulev2 = float, class:^(pavucontrol)$
windowrulev2 = size 800 600, title:^(Volume Control)$
windowrulev2 = move 100 100, floating:1
```

### 4. 动画系统

内置动画框架：
- 插值动画变量
- 多种缓动函数（ease-in、ease-out、bezier）
- 每属性独立配置
- 高性能实现

### 5. 插件架构

支持动态加载插件扩展功能：
- 运行时加载/卸载
- 版本兼容性检查
- 丰富的 API
- 社区插件生态

### 6. 调试友好

完善的调试工具：
- 详细的日志系统
- HyprCtl 运行时控制
- 崩溃报告
- 性能分析支持

### 7. 现代 C++

使用 C++23 特性：
- 智能指针（`SP<T>`、`UP<T>`、`WP<T>`）
- 范围库（`std::ranges`、`std::views`）
- 概念和约束
- 模块化（部分）
- 协程（事件处理）

### 8. 性能优化

多项性能优化：
- 脏区域追踪（只重绘改变的部分）
- 硬件加速渲染（OpenGL）
- DMA-BUF 零拷贝
- VRR/FreeSync 支持
- 预编译头加速编译

### 9. 安全性

安全特性：
- 安全上下文协议（沙箱应用）
- 会话锁定
- 权限管理
- 输入验证
- 内存安全（智能指针）

---

## 依赖关系

### 核心依赖

- **Aquamarine** - 后端抽象层（DRM、X11）
- **Wayland** - Wayland 协议
- **wlroots** - Wayland 合成器库（部分协议）
- **OpenGL/EGL** - 渲染
- **libinput** - 输入设备
- **libudev** - 设备检测
- **pixman** - 像素操作
- **cairo** - 2D 图形
- **pango** - 文本渲染

### 可选依赖

- **XWayland** - X11 兼容
- **libdrm** - 直接渲染管理
- **gbm** - 通用缓冲区管理
- **systemd** - 服务集成
- **Lua** - 脚本配置

---

## 编译系统

Hyprland 使用 **Meson** 构建系统：

```bash
meson setup build
meson compile -C build
meson install -C build
```

特点：
- 快速并行编译
- 自动依赖检测
- 预编译头支持
- 交叉编译支持

---

## 代码统计

```
核心文件：
- main.cpp: 283 行
- Compositor.cpp: 3248 行
- 协议实现: 100+ 个文件
- 总代码量: 约 10 万行 C++
```

---

## 开发流程

### 代码组织原则

1. **头文件和源文件分离**
   - `.hpp` 声明接口
   - `.cpp` 实现细节

2. **命名约定**
   - 类名：`CClassName`（C 前缀）
   - 成员变量：`m_variableName`（m_ 前缀）
   - 全局变量：`g_pGlobalName`（g_p 前缀）
   - 指针类型：`PHLWINDOW`、`PHLMONITOR`（智能指针别名）

3. **智能指针别名**
   ```cpp
   using PHLWINDOW = SP<CWindow>;      // 共享指针
   using PHLWINDOWREF = WP<CWindow>;   // 弱指针
   using PHLMONITOR = SP<CMonitor>;
   // ...
   ```

4. **信号和回调**
   - 使用观察者模式
   - 信号连接和断开
   - 自动生命周期管理

### 添加新功能流程

1. 确定功能属于哪个模块
2. 创建或修改相应的管理器
3. 实现 Wayland 协议（如需要）
4. 添加配置选项
5. 实现核心逻辑
6. 添加事件回调
7. 更新文档
8. 编写测试

---

## 学习路径建议

### 初学者

1. 从 `main.cpp` 开始，理解启动流程
2. 阅读 `Compositor.hpp`，了解核心架构
3. 查看 `desktop/view/Window.cpp`，理解窗口管理
4. 探索 `managers/KeybindManager`，学习输入处理

### 中级开发者

1. 深入 `render/` 目录，理解渲染管道
2. 学习 `protocols/` 中的协议实现
3. 研究 `layout/` 布局算法
4. 了解 `config/` 配置系统

### 高级开发者

1. 优化渲染性能（`render/`、`helpers/DamageRing`）
2. 实现新的 Wayland 协议
3. 开发复杂插件
4. 贡献核心功能

---

## 相关资源

- **官方网站**: https://hyprland.org
- **文档**: https://wiki.hyprland.org
- **GitHub**: https://github.com/hyprwm/Hyprland
- **Discord**: https://discord.gg/hQ9XvMUjjr
- **Wiki 中文**: https://wiki.hyprland.org/zh/

---

## 许可证

Hyprland 采用 BSD-3-Clause 许可证。

---

**最后更新**: 2026-05-13
**Hyprland 版本**: 基于当前 main 分支
