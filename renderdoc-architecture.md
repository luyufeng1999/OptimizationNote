# RenderDoc 架构文档

## 概述

RenderDoc 是一个基于帧捕获的图形调试器，支持 Vulkan、D3D11、D3D12、OpenGL 和 OpenGL ES，可在 Windows、Linux、Android 和 Nintendo Switch 平台上运行。

## 项目结构

```
renderdoc/
├── qrenderdoc/              # Qt UI 层
│   ├── 3rdparty/            # 第三方依赖 (Qt, SWIG, Scintilla 等)
│   ├── Code/
│   │   ├── Interface/       # 外部接口头文件
│   │   ├── pyrenderdoc/     # Python 绑定 (SWIG 生成)
│   │   ├── Widgets/         # Qt 控件
│   │   ├── Windows/         # Qt 窗口
│   │   └── Styles/          # 样式定义
│   └── Resources/           # 资源文件
│
├── renderdoc/               # 核心库
│   ├── api/
│   │   ├── app/             # 应用层 API (renderdoc_app.h)
│   │   └── replay/          # 重放/分析 API
│   ├── common/              # 通用工具类
│   ├── core/                # 核心运行时 (RenderDoc 中枢)
│   ├── driver/              # 图形 API 后端实现
│   ├── hooks/               # 函数拦截/钩子机制
│   ├── os/                  # 操作系统抽象层
│   ├── replay/              # 重放控制器和入口点
│   └── serialise/           # 序列化系统 (RDC 文件格式)
│
├── renderdoccmd/            # 命令行工具
├── renderdocshim/           # 全局钩子 DLL (Windows)
└── util/                    # 构建脚本、测试、CI 配置
```

---

## 核心架构

### 1. 分层架构

RenderDoc 采用分层架构设计：

```
┌─────────────────────────────────────────────────────────┐
│                    qrenderdoc (UI 层)                     │
│         Qt 图形界面，Python 脚本绑定，资源查看器             │
├─────────────────────────────────────────────────────────┤
│                  renderdoc (核心库层)                      │
│  ┌───────────┬───────────┬───────────┬─────────────┐    │
│  │   core    │  replay   │ serialise │    hooks    │    │
│  │  状态管理  │  重放控制  │  RDC 文件  │  函数拦截   │    │
│  └───────────┴───────────┴───────────┴─────────────┘    │
├─────────────────────────────────────────────────────────┤
│                    driver (驱动层)                        │
│  ┌──────┬──────┬──────┬──────┬──────┬────────────────┐  │
│  │ D3D11│ D3D12│Vulkan│  GL  │ GLES│    Metal       │  │
│  └──────┴──────┴──────┴──────┴──────┴────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                      os (系统抽象层)                       │
│  ┌──────────────┬──────────────┬──────────────────────┐ │
│  │   win32      │    posix     │      android         │ │
│  │  (Windows)   │  (Linux/macOS)│     (Android)       │ │
│  └──────────────┴──────────────┴──────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### 2. 核心组件

#### 2.1 RenderDoc 单例类 (`core/core.h`)

`RenderDoc` 类是所有操作的中枢，负责：
- 管理全局状态和捕获选项
- 注册驱动提供商（重放/远程）
- 管理帧捕获器 (`IFrameCapturer`)
- 处理捕获生命周期（开始/结束帧捕获）
- 管理子进程和远程连接
- 配置设置和供应商扩展

关键状态枚举 `CaptureState`：
```cpp
enum class CaptureState {
  LoadingReplay,      // 加载捕获文件，初始化资源
  ActiveReplaying,    // 重放进行中
  StructuredExport,   // 结构化数据导出
  BackgroundCapturing,// 后台捕获模式（已注入但未捕获）
  ActiveCapturing,    // 活动捕获模式（正在捕获帧）
};
```

#### 2.2 钩子系统 (`hooks/hooks.h`)

RenderDoc 通过钩住图形 API 函数调用来实现捕获：

```cpp
// 钩子注册流程
LibraryHook 在构造函数中注册 → LibraryHooks::RegisterHooks() 
→ BeginHookRegistration() → 各驱动 RegisterHooks() → EndHookRegistration()
```

**平台特定的钩子实现：**

| 平台 | 钩子方法 |
|------|----------|
| **Windows** | 遍历加载的库，钩住 IAT (导入地址表) 表；钩住 LoadLibrary 动态加载 |
| **Linux** | 依赖 `LD_PRELOAD`，导出所有钩子符号，钩住 `dlopen` 重定向库句柄 |
| **Android** | 无 interceptor-lib: 类似 Windows，补丁导入表，钩住 `dlsym`<br>有 interceptor-lib: 使用 LLVM 重写目标库汇编代码 |

#### 2.3 序列化系统 (`serialise/serialiser.h`)

RDC 捕获文件格式基于 chunk 的序列化系统：

- **Chunk**: 数据流的基本单位，包含 chunk ID 和元数据（时间戳、调用栈、线程 ID 等）
- **Serialise 模板**: 使用模板特化 `DoSerialise` 实现任意类型的序列化
- **结构化导出**: 可选择将二进制数据导出为结构化数据 (SDFile/SDObject)

```cpp
// 序列化示例
template <class SerialiserType>
void DoSerialise(SerialiserType &ser, MyStruct &el)
{
  SERIALISE_MEMBER(obj);
  SERIALISE_MEMBER_ARRAY(data, count);
  SERIALISE_MEMBER_OPT(optionalPtr);
}
```

---

## 工作原理

### 捕获流程

```
┌─────────────────────────────────────────────────────────────┐
│                      应用启动                                │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  RenderDoc 注入 ( ExecuteAndInject / InjectIntoProcess )    │
│  - 设置环境变量 (RENDERDOC_HOOK_APIS)                        │
│  - 加载 renderdoc.dll/.so                                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  库初始化 (LibraryHooks::RegisterHooks)                      │
│  - 各图形 API 驱动注册钩子函数                                │
│  - 钩子应用于已加载模块                                       │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  BackgroundCapturing 状态                                   │
│  - 最小化开销，准备捕获                                       │
│  - 监听触发信号 (按键/API 调用)                                │
└─────────────────────────────────────────────────────────────┘
                            │
              ┌─────────────┴─────────────┐
              │  StartFrameCapture()      │
              ▼                           │
┌─────────────────────────────────────────┐│
│  ActiveCapturing 状态                   ││
│  - 拦截所有图形 API 调用                  ││
│  - 记录命令和资源状态                    ││
│  - 序列化到 RDC 文件                       ││
└─────────────────────────────────────────┘│
              │                            │
              │  EndFrameCapture()         │
              ▼                            │
┌─────────────────────────────────────────┐│
│  返回 BackgroundCapturing               ││
│  - 完成 RDC 文件写入                       ││
│  - 通知 UI 捕获完成                       │◄───────────────┘
└─────────────────────────────────────────┘
```

### 重放流程

```
┌─────────────────────────────────────────────────────────────┐
│  加载 RDC 文件                                                │
│  - 读取文件头，验证格式                                       │
│  - 解析结构化数据 (SDFile)                                   │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  LoadingReplaying 状态                                      │
│  - 创建重放设备 (IReplayDriver)                              │
│  - 初始化资源 ( textures, buffers, shaders )                │
│  - 构建动作列表 (ActionDescription)                          │
└─────────────────────────────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────┐
│  ActiveReplaying 状态                                       │
│  - 设置帧事件 (SetFrameEvent)                               │
│  - 获取管道状态 (GetPipelineState)                           │
│  - 渲染输出 (ReplayOutput)                                  │
│  - 支持：像素历史、Shader 调试、计数器查询等                   │
└─────────────────────────────────────────────────────────────┘
```

---

## 图形 API 驱动实现

### 驱动目录结构

```
renderdoc/driver/
├── d3d11/                 # D3D11 实现
│   ├── d3d11_context*.cpp # 设备上下文包装
│   ├── d3d11_device*.cpp  # 设备包装
│   ├── d3d11_hooks.cpp    # 钩子注册
│   ├── d3d11_manager.cpp  # 资源管理
│   ├── d3d11_replay.cpp   # 重放实现
│   ├── d3d11_serialise.cpp# 序列化
│   └── wrappers/          # COM 接口包装
│
├── d3d12/                 # D3D12 实现
│   ├── d3d12_command_list*.cpp  # 命令列表包装
│   ├── d3d12_device*.cpp        # 设备包装
│   ├── d3d12_hooks.cpp          # 钩子注册
│   ├── d3d12_rootsig.cpp        # 根签名处理
│   └── d3d12_dxil_debug.cpp     # DXIL 调试
│
├── vulkan/                # Vulkan 实现
│   ├── vk_core.cpp        # 核心 Vulkan 封装
│   ├── vk_layer.cpp       # Vulkan Layer 实现
│   ├── vk_dispatch*.cpp   # 分发表处理
│   ├── wrappers/vk_*.cpp  # 函数包装器
│   └── official/          # Vulkan 官方头文件
│
├── gl/                    # OpenGL/OpenGL ES 实现
│   ├── gl_common.cpp      # GL 通用代码
│   ├── gl_hooks.cpp       # 钩子注册
│   ├── gl_replay.cpp      # 重放实现
│   ├── cgl_hooks.cpp      # macOS CGL 钩子
│   ├── egl_hooks.cpp      # EGL 钩子 (Linux/Android)
│   └── wgl_hooks.cpp      # WGL 钩子 (Windows)
│
├── metal/                 # Metal 实现 (macOS/iOS)
│   ├── metal_common.cpp
│   ├── metal_hooks.cpp
│   └── ...
│
└── ihv/                   # 硬件厂商扩展
    ├── amd/               # AMD RGA/RGP 支持
    ├── arm/               # ARM 性能查询
    ├── intel/             # Intel GL 性能查询
    └── nv/                # NVIDIA PerfSDK/NvAPI
```

### 驱动接口

每个驱动实现以下核心接口：

```cpp
// 捕获接口
struct IFrameCapturer {
  virtual RDCDriver GetFrameCaptureDriver() = 0;
  virtual void StartFrameCapture(DeviceOwnedWindow devWnd) = 0;
  virtual bool EndFrameCapture(DeviceOwnedWindow devWnd) = 0;
  virtual bool DiscardFrameCapture(DeviceOwnedWindow devWnd) = 0;
};

// 重放接口
class IReplayDriver {
  // 资源创建、命令执行、状态查询等
};
```

### Vulkan Layer 机制

Vulkan 驱动使用 Vulkan Layer 机制进行拦截：

```json
// renderdoc.json (Vulkan Layer 配置文件)
{
  "file_format_version": "1.1.2",
  "layer": {
    "name": "VK_LAYER_RENDERDOC_Capture",
    "type": "GLOBAL",
    "library_path": "librenderdoc.so",
    "api_version": "1.3.242",
    "implementation_version": "1",
    "functions": {
      "vkGetInstanceProcAddr": "renderdoc_GetInstanceProcAddr",
      "vkGetDeviceProcAddr": "renderdoc_GetDeviceProcAddr"
    }
  }
}
```

---

## 平台特定实现

### 操作系统抽象层 (`os/os_specific.h`)

RenderDoc 定义了统一的 OS 接口，各平台实现：

| 接口类别 | Windows | Linux | Android | macOS |
|----------|---------|-------|---------|-------|
| **进程** | win32_process.cpp | posix_process.cpp | android_process.cpp | apple_process.cpp |
| **线程** | win32_threading.cpp | posix_threading.cpp | android_threading.cpp | apple_threading.cpp |
| **钩子** | win32_hook.cpp | linux_hook.cpp | android_hook.cpp | apple_hook.cpp |
| **网络** | win32_network.cpp | posix_network.cpp | android_network.cpp | apple_network.cpp |
| **调用栈** | win32_callstack.cpp | linux_callstack.cpp | android_callstack.cpp | apple_callstack.cpp |

### 关键平台差异

#### Windows (win32/)
- 使用 IAT 钩子技术
- `renderdocshim.dll` 用于全局钩子
- COM 接口包装 (D3D11/D3D12)
- WGL/DXGI 窗口系统集成

#### Linux (posix/linux/)
- 依赖 `LD_PRELOAD` 进行库注入
- 使用 `plthook` 库进行 PLT 钩子
- X11/XCB 窗口系统集成
- Wayland 支持（实验性）

#### Android (posix/android/)
- 可选使用 `interceptor-lib` (LLVM 基于的可靠钩子)
- 无 interceptor-lib 时使用 PLT 钩子
- EGL 窗口系统集成
- JDWP 调试器支持

#### macOS (posix/apple/)
- CGL/Metal 窗口系统集成
- 使用 `dyld` 库注入
- QuartzCore 支持

---

## 数据流

### 捕获数据流

```
应用图形 API 调用
       │
       ▼
┌──────────────────┐
│ 钩子函数 (Hook)   │ ← 拦截 API 调用
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ 包装对象 (Wrapped)│ ← 转发 + 记录状态
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ Serialiser       │ ← 序列化到 Chunk
└──────────────────┘
       │
       ▼
┌──────────────────┐
│ RDCFile          │ ← 写入 .rdc 文件
└──────────────────┘
```

### 资源管理

每个驱动维护资源映射：

```cpp
// 资源描述
struct TextureDescription {
  ResourceId resourceId;      // 唯一标识符
  uint32_t width, height, depth;
  ResourceFormat format;
  // ... 其他元数据
};

struct BufferDescription {
  ResourceId resourceId;
  uint64_t size;
  BufferDesc desc;
  // ... 其他元数据
};
```

---

## API 支持矩阵

| API | Windows | Linux | Android | macOS |
|-----|---------|-------|---------|-------|
| **Vulkan** | ✅ | ✅ | ✅ | ❌ |
| **D3D11** | ✅ | ❌ | ❌ | ❌ |
| **D3D12** | ✅ | ❌ | ❌ | ❌ |
| **OpenGL** | ✅ | ✅ | N/A | ✅ |
| **OpenGL ES** | ✅ | ✅ | ✅ | ✅ |
| **Metal** | ❌ | ❌ | ❌ | ✅ |

---

## 关键设计模式

### 1. 包装器模式 (Wrapper Pattern)

所有图形 API 对象都被包装：

```cpp
// D3D11 示例
class WrappedID3D11Device : public ID3D11Device1 {
  ID3D11Device1 *m_pReal;  // 真实对象
  // 包装方法...
};
```

### 2. 反射/序列化

使用模板特化实现类型反射：

```cpp
template <class SerialiserType>
void DoSerialise(SerialiserType &ser, MyEnum &el) {
  ser.SerialiseValue(SDBasic::Enum, sizeof(MyEnum), (uint32_t &)el);
  ser.SerialiseStringify(el);  // 导出字符串表示
}
```

### 3. 状态追踪

每个驱动维护完整的状态机：

```cpp
// D3D11 管道状态
struct D3D11Pipe {
  struct State {
    ID3D11Device* device;
    ID3D11DeviceContext* context;
    // 输入装配器
    IAState ia;
    // 光栅化器
    RSState rs;
    // 着色器阶段
    ShaderStage vs, hs, ds, gs, ps, cs;
    // 输出合并器
    OMState om;
  };
};
```

---

## 扩展机制

### 供应商扩展 (`VendorExtensions`)

```cpp
enum class VendorExtensions {
  NvAPI = 0,       // NVIDIA
  OpenGL_Ext,      // OpenGL 扩展
  Vulkan_Ext,      // Vulkan 扩展
  Count,
};
```

### IHV 集成

```
driver/ihv/
├── amd/           # AMD RGA (Shader Analyzer), RGP (GPU Profiler)
├── arm/           # ARM Frame Advisor, Mali GPUTimer
├── intel/         # Intel GPA, GL 性能查询
└── nv/            # NVIDIA Nsight, PerfSDK, NvAPI
```

---

## 构建系统

### CMake 配置选项

```cmake
# 主要选项
option(ENABLE_VULKAN     "启用 Vulkan 捕获" ON)
option(ENABLE_GL         "启用 OpenGL 捕获" ON)
option(ENABLE_GLES       "启用 OpenGL ES 捕获" ON)
option(ENABLE_METAL      "启用 Metal 捕获" ON)
option(ENABLE_QRENDERDOC "构建 Qt UI" ON)
option(ENABLE_PYRENDERDOC "构建 Python 绑定" ON)

# 平台特定
option(ENABLE_XLIB       "X11 支持" ON)
option(ENABLE_XCB        "XCB 支持" ON)
option(ENABLE_UNSUPPORTED_EXPERIMENTAL_POSSIBLY_BROKEN_WAYLAND "Wayland 支持" OFF)

# Android
option(USE_INTERCEPTOR_LIB "使用 interceptor-lib" OFF)
```

### 目录依赖

```
qrenderdoc
    └── renderdoc (核心库)
        ├── driver/vulkan
        ├── driver/gl
        ├── driver/d3d11 (Windows)
        ├── driver/d3d12 (Windows)
        └── driver/metal (macOS/iOS)
```

---

## 调试功能

### Shader 调试器

支持所有 API 的逐指令调试：

```cpp
class ShaderDebugger {
  virtual ShaderDebugTrace *DebugVertex(...) = 0;
  virtual ShaderDebugTrace *DebugPixel(...) = 0;
  virtual ShaderDebugTrace *DebugThread(...) = 0;
  virtual rdcarray<ShaderDebugState> ContinueDebug() = 0;
};
```

### 像素历史

追踪像素的完整渲染历史：

```cpp
struct PixelModification {
  uint32_t eventId;           // 触发修改的事件
  ResourceId outputTarget;    // 输出目标
  PixelValue newColor;        // 新颜色值
  PixelValue oldColor;        // 旧颜色值
  // ... 深度/模板值
};
```

---

## 总结

RenderDoc 的架构设计核心原则：

1. **模块化驱动**: 每个图形 API 独立实现，可单独启用/禁用
2. **平台抽象**: OS 层统一接口，隔离平台差异
3. **钩子隔离**: 钩子机制集中在 `hooks/` 目录，按平台实现
4. **序列化统一**: 所有驱动使用相同的 RDC 序列化格式
5. **状态追踪**: 完整记录 API 状态，支持精确重放
6. **扩展性**: 供应商扩展和自定义驱动支持
