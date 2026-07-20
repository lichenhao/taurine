# Tauri 测试注入扩展（Test Extension）设计调研

> 调研日期：2026-07-19
> 方向：在 Tauri 基础上提供 Chrome 扩展式的非侵入测试注入能力，覆盖引擎核心生命周期事件监听与捕获，用于日志、问题定位与自动化测试。

---

## 1. 背景与动机

### 1.1 Tauri 现状

Tauri 是「系统 WebView 渲染 Web 前端 + 编译型语言原生后端 + 异步消息桥」的桌面/移动应用框架，不打包 Chromium，体积小。其分层：

```
前端 UI (HTML/CSS/JS)
        │  invoke() / listen() / emit()
        ▼
系统 WebView (Windows: WebView2 / macOS: WKWebView / Linux: WebKitGTK)
        │  IPC 桥
        ▼
Tauri Core (Rust)
  ├─ 命令处理器 (command handlers)
  ├─ 事件系统
  ├─ 权限/安全策略
  └─ 插件 (窗口、托盘、文件系统、updater…)
```

关键上游分层：
- **TAO** — 跨平台窗口/事件循环
- **WRY** — 跨平台 WebView 抽象
- **tauri crate** — 在 TAO+WRY 之上组合配置、命令宏、资源打包、OS API
- **`@tauri-apps/api`** — 前端 JS/TS API

IPC 模型：
- **Command**：请求/响应，`invoke("greet", {name})` → Rust 函数 → 序列化返回值 resolve Promise
- **Event**：单向通知，`emit/listen`

### 1.2 痛点

Tauri 最痛的一块是**可观测性与可测性**：
- WebView/窗口/平台 API 耦合较深，难 mock
- 引擎核心生命周期事件（WebView 创建/导航/崩溃、事件循环 tick、权限决策点）外部无法感知
- 测试只能靠端到端真实启动，慢且难定位

### 1.3 思路

不重新发明轮子，而是补上可观测性与可测性。参考 Chrome 扩展模型，提供：
1. 统一 CDP 网关（三平台调试协议归一）
2. 引擎生命周期事件总线（TAO/WRY/Core 埋点）
3. Extension Host（manifest 驱动的非侵入注入）

---

## 2. 三大 WebView 自带的远程调试协议

| 平台 | 原生调试协议 |
|---|---|
| WebView2 | CDP（`CallDevToolsProtocolMethodAsync`） |
| WKWebView | Safari Web Inspector Protocol（私有 `_remoteInspector`） |
| WebKitGTK | `WebKitWebInspector` / remote inspector |

**Chrome DevTools Protocol 本身就是非侵入的**——测试进程通过 WebSocket 附着，不改 App 代码。Tauri 缺的是一个**统一的 CDP 网关** + **引擎生命周期事件总线**。这两件事补上，就是「Chrome 扩展式测试注入」。

---

## 3. 目标架构

```
┌──────────────────────────────────────────────┐
│  Test Harness / DevTools Panel (外部进程)      │
│  - 订阅事件、断言、录制回放、截图               │
└──────────────┬───────────────────────────────┘
               │  CDP-over-WS (统一协议)
               ▼
┌──────────────────────────────────────────────┐
│  Tauri Debug Gateway (新增核心模块)            │
│  ├─ CDP 路由器: 统一三平台协议差异              │
│  ├─ Engine Lifecycle Bus (新增)               │
│  │   ├─ TAO 事件: 窗口创建/事件循环/托盘/...   │
│  │   ├─ WRY 事件: WebView 挂载/导航/崩溃/...   │
│  │   └─ Core 事件: 命令派发/插件加载/权限决策  │
│  ├─ Extension Host (新增, 类 Chrome 扩展)      │
│  │   ├─ content-script 注入 (document-start)  │
│  │   ├─ background observer (OOB 订阅)        │
│  │   └─ chrome.debugger 等价 API              │
│  └─ Manifest 解析 + 权限沙箱                   │
└──────────────┬───────────────────────────────┘
               │
               ▼
         现有 Tauri Core (改动极小)
```

---

## 4. Chrome 扩展概念对照

| Chrome 扩展概念 | Tauri 等价 | 用途 |
|---|---|---|
| `chrome.debugger` + CDP | 统一 CDP Gateway | 程序化断点、网络拦截、DOM 查询 |
| content scripts | `content-script` 注入声明 | 在 WebView document-start/end 注入断言 JS |
| background service worker | `background` observer 进程 | 订阅引擎事件、录制、跨窗口关联 |
| `chrome.events` 命名空间 | Engine Lifecycle Bus | 捕获核心生命周期做日志/定位 |
| manifest.json | `test-ext.json`（capability 声明） | 声明订阅哪些事件、能调哪些 CDP 域 |
| DevTools panel | Tauri Inspector 面板 | 可视化时间线、命令流水、权限决策 |

---

## 5. 引擎生命周期事件总线（核心创新）

这是 Tauri 目前**完全没有**的一层，也是「连引擎核心生命周期都支持监听」的关键。设计为**纯加法的可选 trait**：

```rust
// wry/src/lib.rs 新增（伪代码）
pub trait WebViewLifecycleSink {
    fn on_webview_created(&self, id: WindowId, info: &WebViewInfo);
    fn on_navigation(&self, id: WindowId, url: &Url, kind: NavKind);
    fn on_ipc_message(&self, id: WindowId, payload: &Value);  // 拦截点
    fn on_renderer_crash(&self, id: WindowId, reason: CrashReason);
    fn on_eval_script(&self, id: WindowId, script: &str);     // 注入审计
}

// tao/src/lib.rs 同理
pub trait WindowLifecycleSink {
    fn on_event_loop_tick(&self);
    fn on_window_event(&self, id: WindowId, event: &WindowEvent);
    fn on_tray_event(&self, ...);
}
```

App 代码**完全不用改**。测试时 Gateway 注册一个 Sink 把事件转发到 CDP `Tauri.lifecycle` 域；生产时可挂一个日志 Sink 写 `tracing`。同一个埋点同时服务测试和线上诊断——这是它比 Chrome 扩展模型更值的地方。

---

## 6. 测试扩展目标形态

### 6.1 扩展清单

```jsonc
// test-ext.json —— 测试扩展清单
{
  "subscribe": ["Tauri.lifecycle.*", "Tauri.command.*", "Tauri.permission.decision"],
  "inject": {
    "document-start": ["assert-helpers.js"],
    "document-end": ["ui-probes.js"]
  },
  "cdp": ["Page", "Network", "Runtime", "Tauri"]
}
```

### 6.2 测试侧 API

```js
// 测试侧 (Playwright/独立 runner)
const ext = await tauri.attachExtension('./test-ext.json');
ext.on('Tauri.lifecycle.onRendererCrash', e => assert.fail(e.reason));
ext.on('Tauri.command.onDispatch', e => trace.push(e));
await ext.CDP.Page.navigate('tauri://localhost/index.html');
const res = await ext.eval('window.__tauri__.invoke("greet","Ada")');
assert.equal(res, 'Hello, Ada!');
```

App 源码零改动，测试像 Chrome DevTools 一样附着——这就是「非侵入」的真正含义。

---

## 7. 源码改动边界（关键决策）

### 7.1 结论

**部分必须改源码，但可以做到极小且纯加法。** 事件源头在 TAO/WRY 内部，外部进程无法感知，所以「引擎生命周期事件总线」这一层必然要动 TAO/WRY 源码；但可以做成**纯加法的可选 trait**，不破坏现有任何路径，对普通 App 零开销零行为变化。

### 7.2 三层改动分类

#### A. 必须改 Tauri 源码（无法绕开）

| 改动点 | 位置 | 原因 | 改动量 |
|---|---|---|---|
| `WebViewLifecycleSink` trait | `wry/src/lib.rs` | WebView 创建/导航/崩溃/IPC 拦截发生在 WRY 内部，外部拿不到时机 | ~150 行 |
| `WindowLifecycleSink` trait | `tao/src/lib.rs` | 事件循环 tick、窗口事件、托盘事件在 TAO 内部 | ~120 行 |
| `CommandDispatchSink` trait | `tauri/src/app.rs` | 命令路由派发点在 Core 内部，要拦截需挂 hook | ~80 行 |
| `PermissionDecisionSink` | `tauri/src/permission/` | 权限决策是 Core 内部同步调用，外部只能事后看日志 | ~60 行 |
| Sink 注册入口 | `tauri::Builder` | App 启动时注册 sink，类似现有 `plugin` 注册 | ~40 行 |

**全部是 `trait + default empty impl + 几处 emit 调用`，不动现有逻辑、不改签名、不破坏 ABI。** 改动总量约 400–500 行，可以拆成 4 个独立 PR 上游 review。

#### B. 可以做成独立 crate（零源码改动）

| 模块 | 形态 | 依赖 |
|---|---|---|
| CDP Gateway（协议路由+三平台归一） | `tauri-debug-gateway` crate | 只用 A 的 trait |
| Extension Host（manifest 解析+注入调度） | `tauri-test-ext` crate | 用 Gateway |
| Engine Lifecycle Bus 实现（把 sink 事件转 CDP 域 / tracing） | `tauri-lifecycle-bus` crate | 用 A 的 trait |
| 测试 Harness API（attach/eval/assert） | 独立 crate + CLI | 用 Gateway 的 WS |
| Inspector 面板 | 独立 Web App | 用 Gateway 的 WS |
| 录制/回放 | 独立工具 | 读 bus 事件流 |

B 全部是 Tauri 生态里的「消费者」，不进 Tauri 主仓库，可以独立发版、独立演进。

#### C. 纯插件能力（现有 `tauri-plugin` 就能做）

不用动源码、也不用新 trait，现有插件 hook 够用：
- WebView 创建后注入测试 JS（`on_webview_created` 钩子已有）
- 监听 `WindowEvent`（已有 `on_window_event`）
- 拦截导航（已有 `on_navigation` 钩子）
- 注册测试用 command（普通插件机制）
- WebView2 的 CDP 调用（通过现有 `WebviewWindow::eval` + `CallDevToolsProtocolMethodAsync` 经 eval 桥）

**局限**：拿不到事件循环 tick、拿不到 document-start 注入时机（现有钩子是 created 之后）、拿不到 renderer crash 的早期信号、拿不到权限决策点、拿不到命令派发前的拦截点。

### 7.3 为什么不能纯插件化

四个硬限制：

1. **document-start 注入**：现有 `on_webview_created` 是 WebView ready 之后才回调，已经错过 `document_start`。Chrome 扩展能注入是因为它挂在浏览器渲染管线里。要在 Tauri 做到，必须在 WRY 的 WebView 初始化路径里加一个 hook——这是源码改动。
2. **renderer crash 早期信号**：WebView2 的 `ProcessFailed` 事件、WKWebView 的 `webViewWebContentProcessDidTerminate` 都在 WRY 内部处理，目前只走 Tauri 自己的事件，没有外部 sink。要导出就得加 trait。
3. **事件循环 tick**：TAO 的 `EventLoop::run` 内部循环外部完全不可见。做性能 profiling / 死锁检测必须埋点。
4. **权限决策拦截**：`tauri::permission::resolve` 是同步函数，外部只能事后看日志，不能在决策点注入测试策略（比如「测试模式下强制 deny 某个 scope」）。要支持就得加 sink 并允许返回 override。

### 7.4 最小改动 PR 拆分（便于上游合入）

```
PR1: tao  — 加 WindowLifecycleSink trait + 埋点（不影响现有行为）
PR2: wry  — 加 WebViewLifecycleSink trait + 埋点
PR3: tauri— 加 CommandDispatchSink + PermissionDecisionSink + Builder 注册入口
PR4: tauri— 文档 + 一个示例 sink 写 tracing 日志
```

每个 PR 都满足：**default impl 空方法 → 现有代码路径字节级不变 → 普通用户无感知**。这种纯加法 trait 上游接受度很高（Tauri 团队自己也想要可观测性，issue 里反复提过）。

### 7.5 替代方案对比

| 方案 | 源码改动 | 能力 | 上游阻力 | 适合 |
|---|---|---|---|---|
| 纯插件（C 层） | 0 | 受限：无 document-start、无 crash 早期、无权限拦截、无事件循环 | 0 | 只做 UI 断言 |
| **最小 trait 加法（推荐）** | ~500 行，4 PR | 全功能 | 低，纯加法 | 生产级 |
| Fork 重度改造 | 大 | 全功能 + 自定义协议 | 高，维护分叉 | 不推荐 |
| 外部进程纯 CDP | 0 | 只能测 WebView 侧，测不到 Tauri Core | 0 | 只测前端 |

---

## 8. 工作量估算

### 8.1 全功能对标 Tauri v2（参考）

约 24–35 人月（一个熟练工程师全职）。包含：WebView 抽象层、窗口/事件循环、IPC 核心、命令宏、Runtime/App Builder、权限/安全、资源打包、构建工具链、官方插件集（~30 个）、移动端、前端 JS API、文档/脚手架。

### 8.2 本测试注入扩展 v1

| 模块 | 复杂度 | 人月 |
|---|---|---|
| CDP Gateway + 三平台协议统一 | 高 | 1.5–2 |
| Engine Lifecycle Bus（TAO/WRY 埋点） | 中 | 1–1.2 |
| Extension Host（manifest + 注入 + background） | 中 | 1 |
| 测试 Harness API + 录制回放 | 中 | 0.8–1 |
| Inspector 面板 UI | 中 | 1–1.5 |
| 文档/SDK/示例 | 低 | 0.5 |
| **合计 v1** | | **6–7 人月** |

AI 驱动 + 人工 review：v1 可用约 3–4 周，完整带 UI 面板 2 个月左右。

---

## 9. 落地顺序

1. **第 1 周**：在 Tauri fork 里加 `LifecycleSink` trait，TAO/WRY 埋点，先跑通「事件能出来」
2. **第 2 周**：CDP Gateway 单平台（WebView2 先，CDP 最标准），`Tauri.lifecycle` 自定义域
3. **第 3 周**：Extension Host + `test-ext.json` + content-script 注入
4. **第 4 周**：一个端到端测试 demo（用真实 Tauri 示例 App 跑非侵入断言）+ 提 PR 给 Tauri 官方讨论

### 9.1 第一步动作

- Fork Tauri 仓库
- 创建 dev 分支 `tauri-test-ext`
- PR1：`tao` 加 `WindowLifecycleSink` trait + 埋点
- PR2：`wry` 加 `WebViewLifecycleSink` trait + 埋点
- PR3：`tauri` 加 `CommandDispatchSink` + `PermissionDecisionSink` + `Builder` 注册入口
- PR4：文档 + 一个 `tracing` sink 示例

---

## 10. 为什么这条路比 Zig 重写 Tauri 更值

- **杠杆**：6 人月撬动整个 Tauri 生态的可测性，而不是 30 人月做一个二等公民
- **复用**：三平台调试协议都是现成的，Gateway 做协议归一即可
- **双用**：同一套生命周期事件总线，测试用 + 生产可观测用（`tracing`/OpenTelemetry 导出），ROI 翻倍
- **生态**：作为 Tauri 官方插件进主线，所有 App 自动受益，比独立框架有分发
- **Zig 版本可以后做**：有了这套测试注入规范，Zig 版只要实现相同协议就能直接复用全部测试工具

---

## 11. 待办（等仓库 fork 后展开）

- [ ] Fork Tauri 到本地 `D:\works\github\zigit\tauri`
- [ ] 创建 dev 分支 `tauri-test-ext`
- [ ] 调研 TAO/WRY/tauri 当前源码结构，确认埋点位置
- [ ] PR1：`tao` `WindowLifecycleSink` trait + 埋点 + 单测
- [ ] PR2：`wry` `WebViewLifecycleSink` trait + 埋点 + 单测
- [ ] PR3：`tauri` `CommandDispatchSink` + `PermissionDecisionSink` + Builder 注册入口
- [ ] PR4：`tracing` sink 示例 + 文档

---

## 参考资料

- Tauri 架构：https://github.com/tauri-apps/tauri/blob/dev/ARCHITECTURE.md
- Tauri IPC 概念：https://v2.tauri.app/concept/inter-process-communication/
- Tauri 权限/安全：https://deepwiki.com/tauri-apps/tauri/2.6-permission-and-security-system
- zero-native（Zig 版 Tauri 类框架）：https://github.com/vercel-labs/zero-native
- webview/webview（跨平台 C wrapper）：https://github.com/webview/webview_csharp
- WebView2 文档：https://learn.microsoft.com/en-us/microsoft-edge/webview2/
- WebKitGTK：https://www.webkitgtk.org/
- WKWebView：https://developer.apple.com/documentation/WebKit/WKWebView