# WebUI 与桌面双运行时

## 这篇文档解决什么问题

到这里，你已经见过：

- 桌面启动链；
- bridge 通信总线；
- conversation 与 Team 的主链路。

这一篇要解决的是另一个非常关键的问题：

> 为什么 AionUi 同时能作为桌面应用运行，又能作为 WebUI 运行？
> 两者复用了什么？又在哪些地方明确分叉？

---

## 先给结论

AionUi 的双运行时设计，不是维护两套独立业务系统，而是：

- **共用**：大量 bridge 语义、service、task、team、extension、channel 能力
- **分叉**：transport、认证方式、窗口壳、服务端生命周期

你可以先记下面这张图：

```text
桌面模式
Renderer -> Preload -> Main(adapter/main.ts) -> bridge/service/task

WebUI 模式
Browser -> browser.ts(WebSocket) -> webserver/adapter.ts -> bridge/service/task

Standalone server
server.ts -> standalone.ts -> initBridgeStandalone() -> bridge/service/task
```

真正复用的是中后段。

---

## 一、桌面模式与 WebUI 模式在入口上就分流了

### 桌面模式

`src/index.ts` 的正常路径会：

- `initializeProcess()`
- `createWindow()`
- 加载 renderer URL / HTML
- 初始化桌面专属能力（托盘、菜单、DevTools、窗口控制等）

### WebUI 模式

如果传 `--webui`，`src/index.ts` 会改走另一条路径：

- 读取用户 WebUI 配置
- 解析端口与 remote access
- `startWebServer(...)`
- 不创建主窗口
- 还会在 WebUI 模式下补做 ACP 检测

这说明：

> WebUI 不是桌面窗口里的一个页面，而是启动路径上的一级分支。

---

## 二、`src/server.ts` 说明 WebUI 甚至可以脱离 Electron 壳独立运行

`src/server.ts` 是纯服务端入口。

它会：

1. 注册 Node 平台服务
2. 注册 standalone adapter
3. `initStorage()`
4. 初始化 `ExtensionRegistry`
5. 初始化 `ChannelManager`
6. 调 `initBridgeStandalone()`
7. 启动 `startWebServerWithInstance(...)`

也就是说，很多后台能力本来就不是 Electron 专属，而是**可独立运行的 Node 服务能力**。

这是 WebUI 能成立的根基。

---

## 三、真正被复用的是什么

### 1. 复用 bridge 语义

`ipcBridge.ts` 里定义的很多业务动作，在桌面和 WebUI 中都保持一致。

例如：

- `conversation.create`
- `conversation.sendMessage`
- `team.create`
- `team.sendMessage`
- `webui.changePassword`

这意味着前端或浏览器并不需要知道后台是 Electron 还是 Node，只需要知道“我要调用哪个业务名”。

### 2. 复用 service / task / team

无论桌面还是 WebUI，后面接的很多能力都是同一套：

- `ConversationServiceImpl`
- `WorkerTaskManager`
- `TeamSessionService`
- `ExtensionRegistry`
- `ChannelManager`

这就是为什么前面你看到的 conversation / team 文档，对 WebUI 依然有效。

---

## 四、真正分叉的是 transport（传输层）

### 桌面：Electron IPC

- preload 暴露 `electronAPI`
- `browser.ts` 检测到 `window.electronAPI`
- 走 IPC 发送 `{ name, data }`
- `adapter/main.ts` 用 `ipcMain.handle(...)` 接住

### WebUI：WebSocket

- `browser.ts` 检测不到 `electronAPI`
- 建 WebSocket
- 把同样的 `{ name, data }` 发给服务端
- `webserver/adapter.ts` 再把它喂回 bridge emitter

所以你可以把桌面和 WebUI 的差异先粗暴理解成：

> 业务名没变，只是“送消息的管道”变了。

---

## 五、AuthContext 暴露了前端层最关键的分叉

`src/renderer/hooks/context/AuthContext.tsx` 直接体现了桌面与 WebUI 的差异。

### 桌面模式

- `isDesktopRuntime === true`
- 默认直接进入 `authenticated`
- 不依赖 `/api/auth/user`
- 不走 Cookie / CSRF

### WebUI 模式

- 通过 `/api/auth/user` 取当前用户
- 登录请求走 `/login`
- 使用 `credentials: 'include'`
- 请求体里带 `withCsrfToken(...)`
- 登陆成功后调用 `__websocketReconnect()`

这说明桌面和 WebUI 在前端最大的分叉之一是：

> 桌面是本地应用信任模型，WebUI 是典型 Web 鉴权模型。

---

## 六、WebUI 配置与生命周期在 `webuiBridge.ts`

`src/process/bridge/webuiBridge.ts` 是最值得读的后台入口之一。

它负责：

- `webui.getStatus`
- `webui.start`
- `webui.stop`
- `webui.changePassword`
- `webui.changeUsername`
- `webui.resetPassword`
- `webui.generateQRToken`
- `webui.verifyQRToken`

而且它还维护：

- 当前 `webServerInstance`
- 启停时的 WebSocket 清理
- 状态变更事件广播
- direct IPC handlers（直连 IPC）

这里有一个非常重要的设计点：

> 同一个 WebUI 功能，同时支持 bridge provider 和 direct IPC 两种入口。

这就是为什么 `WebuiModalContent.tsx` 会优先走：

```ts
window.electronAPI?.webuiChangePassword
```

如果没有桌面环境，再退回：

```ts
webui.changePassword.invoke(...)
```

这是一种很实际的“双入口兼容”设计。

---

## 七、Web server 本身也有独立职责

`src/process/webserver/index.ts` 说明 WebUI 不只是一个 WebSocket 转发器，它本身也是完整 Web 服务：

- 基于 Express（Web 框架）
- 有 HTTP server + WebSocket server
- 有默认管理员初始化
- 有认证与用户仓储
- 有静态资源路由
- 有 API 路由
- 有二维码登录 URL 生成

这说明 WebUI 模式并不是“借 Electron 后台顺便跑个网页”，而是一条完整的服务端路线。

---

## 八、一个真实链路：修改 WebUI 密码

这是理解双运行时最好的例子之一。

### 前端入口

`WebuiModalContent.tsx`：

- 用户输入新密码
- 如果是桌面环境，优先走 `window.electronAPI.webuiChangePassword(...)`
- 否则走 `webui.changePassword.invoke(...)`

### 后台落点

`webuiBridge.ts`：

- bridge provider：`webui.changePassword.provider(...)`
- direct IPC：`ipcMain.handle('webui-direct-change-password', ...)`

### 业务实现

再交给 `WebuiService.changePassword(...)`

这条链路清楚地展示出：

- UI 语义相同
- 传输方式不同
- 最终业务落地相同

---

## 九、默认端口与多实例也体现了运行时意识

`src/common/config/constants.ts` 定义了：

- 生产默认端口：`25808`
- 开发默认端口：`25809`
- 多实例开发默认端口：`25810`

这说明 WebUI 设计从一开始就考虑了：

- 生产 / 开发差异
- 本机多实例运行
- 浏览器连接默认端口推导

配合 `browser.ts` 里的 `WEBUI_DEFAULT_PORT` 使用，前后端都对默认连接目标有一致预期。

---

## 十、这套双运行时设计的价值是什么

### 1. Electron 不再是唯一宿主

AionUi 的后台能力可以脱离桌面壳存在。

### 2. 同一套业务能力能被更多入口复用

桌面 UI、浏览器、甚至未来其他入口都能沿用 bridge 语义。

### 3. 桌面体验与 Web 体验都能保留各自优势

- 桌面：本地信任、窗口控制、托盘、直连 IPC
- WebUI：标准 Web 鉴权、远程访问、HTTP / WebSocket

### 4. 后台平台化更自然

因为业务层不是绑定在 BrowserWindow 上的。

---

## 你读完这一篇后，下一步读哪里

下一篇建议读：

- `ai_docs/07-测试-CI-质量保障与工程约束.md`

因为现在你已经理解了两种运行时，接下来最值得看的是：

> 这个项目如何验证这些复杂路径没有坏掉。

---

## 证据文件

- `src/index.ts:467-586`
- `src/server.ts:1-123`
- `src/common/adapter/browser.ts:23-254`
- `src/common/adapter/main.ts:39-108`
- `src/common/adapter/standalone.ts:1-34`
- `src/process/webserver/adapter.ts:1-64`
- `src/process/bridge/webuiBridge.ts:46-260`
- `src/process/webserver/index.ts:246-260`
- `src/renderer/hooks/context/AuthContext.tsx:45-239`
- `src/renderer/components/settings/SettingsModal/contents/WebuiModalContent.tsx:460-498`
- `src/common/config/constants.ts:54-61`
- `tests/e2e/specs/webui.e2e.ts:14-128`

---

## 容易误解的点

- **误解：WebUI 只是桌面里的一个设置页。**
  - 不是。它是一条独立运行时路线。
- **误解：桌面和 WebUI 共用同一套认证。**
  - 不是。桌面默认本地可信，WebUI 走 Web 鉴权与 CSRF。
- **误解：WebUI 功能只能通过 HTTP 做。**
  - 不是。它大量依赖 WebSocket 承载 bridge 语义。
