# Bridge / Preload / IPC / 通信总线

## 这篇文档解决什么问题

这一篇专门回答：

> AionUi 的界面、主进程、WebUI、后台服务，到底怎么连起来？

如果你能把这一篇看懂，后面大部分“从按钮到业务逻辑”的跟踪都会顺很多。

---

## 先给最核心的结论

AionUi 的关键设计不是“把逻辑塞进组件里”，而是：

> 先在 `src/common/adapter/ipcBridge.ts` 定义统一的业务契约，再根据运行环境选择不同 transport（传输层）去承载它。

也就是说，系统在做两层分离：

1. **业务语义层**：例如 `conversation.create`、`team.create`、`webui.changePassword`
2. **传输层**：Electron IPC、WebSocket、standalone EventEmitter

这就是为什么桌面和 WebUI 能复用大量后台逻辑。

---

## 一、`ipcBridge` 是统一契约层

`src/common/adapter/ipcBridge.ts` 不是具体实现，而是**接口地图**。

这里定义了大量业务入口，例如：

- `conversation.create`
- `conversation.get`
- `conversation.sendMessage`
- `team.create`
- `application.openDevTools`
- `fs.readFile`
- `webui.start`
- `webui.changePassword`

它的意义很大：

- renderer 不需要知道后台模块具体怎么实现；
- main / standalone 只需要把这些 provider（提供者）注册起来；
- transport 换掉，业务名不变。

你可以把它理解成：

> AionUi 内部的“应用级 API 表”。

---

## 二、桌面模式下：Renderer 怎么把请求送到 Main

### 第一步：Renderer 调用 `ipcBridge.xxx.invoke(...)`

例如 Team 创建页面里，`TeamCreateModal.tsx` 调用：

```ts
ipcBridge.team.create.invoke({ ... })
```

这说明 renderer 侧并不直接 import `TeamSessionService`，而是先走 bridge 契约。

### 第二步：Preload 把 `electronAPI` 暴露到 `window`

`src/preload/main.ts` 里用：

```ts
contextBridge.exposeInMainWorld('electronAPI', { emit, on, ... })
```

这一步的意义是：

- renderer 不直接接触 `ipcRenderer`
- 所有桌面通信都经过 preload 白名单 API

### 第三步：`src/common/adapter/browser.ts` 绑定 Electron 适配器

在 Electron 环境里，`browser.ts` 会检测到 `window.electronAPI` 存在，于是使用：

- `emit(name, data)` → `window.electronAPI.emit(...)`
- `on(emitter)` → 监听 `window.electronAPI.on(...)`

也就是说，前端真正发出去的还是一个统一的 `{ name, data }` 事件。

### 第四步：`src/common/adapter/main.ts` 在主进程接收并转发

`main.ts` 里注册了：

- `ipcMain.handle(ADAPTER_BRIDGE_EVENT_KEY, ...)`
- `bridge.adapter({ emit, on })`

这层负责两件事：

1. 收到 renderer 请求后，把它交给 bridge emitter
2. 把后台事件广播回所有 BrowserWindow

而且它还做了一个很重要的额外动作：

- **同时广播给所有 WebSocket 客户端**

这意味着 Main adapter 不只是 Electron IPC 适配器，它还是桌面与 WebUI 的共同广播出口。

---

## 三、WebUI 模式下：为什么还能复用同一套 bridge

### Renderer 侧切换成 WebSocket 模式

`src/common/adapter/browser.ts` 里，如果发现 `window.electronAPI` 不存在，就不会走 Electron IPC，而是：

- 建立 WebSocket 连接
- 把 `{ name, data }` 序列化后发给服务端
- 收到服务端消息后，再 `emitter.emit(name, data)`

这就是 WebUI 模式的 transport 替换。

业务名没变，底层通道换了。

### 服务端把 WebSocket 接回 bridge emitter

`src/process/webserver/adapter.ts` 会：

- 初始化 `WebSocketManager`
- 注册 broadcaster（广播器）
- 把 WebSocket 收到的消息重新喂给 `bridgeEmitter`

这样一来，WebSocket 消息就重新回到了统一的 bridge 语义层。

---

## 四、纯服务端模式为什么还需要 `standalone.ts`

`src/common/adapter/standalone.ts` 的存在，是为了让 `src/server.ts` 在**没有 Electron** 的情况下，也能跑同一套 bridge 语义。

它做的事是：

- 用 Node.js `EventEmitter` 做内部消息总线
- 设置 `bridge.adapter({ emit, on })`
- 把服务端内部消息路由进 bridge handler

所以现在你可以看到 3 种 transport：

| 运行环境 | transport |
| --- | --- |
| Electron 桌面 | IPC |
| WebUI 浏览器 | WebSocket |
| 纯服务端内部 | EventEmitter |

但它们背后都尽量对齐到同一层 bridge 语义。

---

## 五、真正的业务逻辑在哪里挂进去

定义了契约，并不等于实现了业务。

真正的业务 provider 注册发生在两个地方：

### 桌面模式

- `src/process/utils/initBridge.ts`
- `src/process/bridge/index.ts`

这里会注入：

- `conversationService`
- `conversationRepo`
- `workerTaskManager`
- `channelRepo`
- `teamSessionService`

然后统一调用 `initAllBridges(...)`，把几十个 bridge 模块都注册起来。

### standalone 模式

- `src/process/utils/initBridgeStandalone.ts`

它会跳过 Electron 专属 bridge，但保留大量共享业务桥，例如：

- conversation
- auth
- mcp
- model
- task
- channel
- database
- systemSettings

这正好说明：

> 业务层和运行环境层是故意拆开的。

---

## 六、两个真实例子

### 例子 1：Team 创建

链路是：

```text
TeamCreateModal.tsx
  -> ipcBridge.team.create.invoke(...)
  -> teamBridge.ts 的 provider
  -> TeamSessionService.createTeam(...)
```

这里前端不需要知道：

- team 怎么存数据库
- session 什么时候启动
- agent conversation 怎么创建

它只需要调用统一契约。

### 例子 2：WebUI 改密码

链路是：

```text
WebuiModalContent.tsx
  -> Electron 环境: window.electronAPI.webuiChangePassword(...)
  -> Web 环境: webui.changePassword.invoke(...)
  -> webuiBridge.ts
  -> WebuiService.changePassword(...)
```

这条链路非常适合初学者理解：

- 同一个功能
- 两种 transport 入口
- 最终落到同一组后台语义

---

## 七、这套设计的价值是什么

### 1. UI 不绑死后台实现

组件层更轻，改 transport 不用重写业务。

### 2. 桌面与 WebUI 复用语义

不用为两种入口维护两套独立业务接口。

### 3. 后台事件可以统一广播

`main.ts` 同时给 BrowserWindow 和 WebSocket 客户端广播，就是这个设计的体现。

### 4. 更适合平台化扩展

只要新能力能接到 bridge 契约层，它就能被 UI、WebUI、甚至外部渠道复用。

---

## 读代码时你该怎么用这一篇

以后你只要看到 renderer 里这几种代码：

- `ipcBridge.xxx.invoke(...)`
- `ipcBridge.xxx.emit(...)`
- `window.electronAPI.xxx(...)`

你就要立刻问自己两个问题：

1. 这个名字在 `src/common/adapter/ipcBridge.ts` 里是怎么定义的？
2. 它在 `src/process/bridge/*.ts` 里是谁注册的 provider？

这两个问题会帮你把“界面动作”和“后台实现”连起来。

---

## 你读完这一篇后，下一步读哪里

下一篇请读：

- `ai_docs/03-Renderer-外壳-页面与全局状态.md`

因为现在你已经知道“怎么通信”，接下来应该理解：

> 前端这层壳，到底是如何组织页面、Provider、路由和鉴权状态的。

---

## 证据文件

- `src/common/adapter/ipcBridge.ts:26-320`
- `src/preload/main.ts:13-66`
- `src/common/adapter/browser.ts:23-254`
- `src/common/adapter/main.ts:39-108`
- `src/common/adapter/registry.ts:1-46`
- `src/common/adapter/standalone.ts:1-34`
- `src/process/webserver/adapter.ts:1-64`
- `src/process/utils/initBridge.ts:19-40`
- `src/process/bridge/index.ts:48-95`
- `src/process/bridge/teamBridge.ts:33-120`
- `src/process/bridge/webuiBridge.ts:46-255`

---

## 容易误解的点

- **误解：`ipcBridge` 就是 IPC 实现。**
  - 不是。它首先是业务契约层。
- **误解：WebUI 不能复用桌面业务。**
  - 不是。它主要替换的是 transport。
- **误解：Preload 才是通信中枢。**
  - 不是。Preload 是桌面 transport 的一部分；真正的中枢是 bridge 契约 + provider 注册。
