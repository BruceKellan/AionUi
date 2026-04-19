# Team 多 Agent 协作架构

## 这篇文档解决什么问题

如果你已经理解单 conversation 链路，下一步最值得学的就是 Team（团队模式）。

因为 Team 不是简单地“多开几个会话”，而是在单会话模型之上，再加了一层：

- session（会话编排层）
- mailbox（代理间消息箱）
- task manager（团队任务管理）
- teammate manager（成员生命周期管理）
- MCP server（团队协调工具入口）

你要回答的问题是：

> 用户点击“Create Team”之后，后台到底创建了什么？
> 一个团队为什么能让多个 agent 协作？

---

## 先给总图

先把 Team 链路压成一张图：

```text
TeamCreateModal.tsx
  -> ipcBridge.team.create.invoke(...)
  -> teamBridge.ts
  -> TeamSessionService.createTeam(...)
     -> 为每个 agent 创建 / 复用 conversation
     -> 持久化 TTeam

真正开始协作时
  -> TeamSessionService.getOrStartSession(...)
  -> TeamSession
     -> startMcpServer()
     -> Mailbox
     -> TaskManager
     -> TeammateManager
  -> lead agent / teammate agent 通过 mailbox + MCP tools 协作
```

---

## 一、Team 的前端创建入口很薄

`src/renderer/pages/team/components/TeamCreateModal.tsx` 做的事情不多，但信息很关键。

它会收集：

- team name
- 选中的 leader agent
- workspace

然后调用：

```ts
ipcBridge.team.create.invoke({
  userId,
  name,
  workspace,
  workspaceMode: 'shared',
  agents,
})
```

这里要注意两点：

### 1. 前端提交的是“团队意图”，不是完整运行时

前端只告诉后台：

- 我想建一个 team
- leader 是谁
- workspace 是什么

至于：

- 每个 agent 对应什么 conversation
- 模型怎么解析
- workspace 如何回填
- session 什么时候启动

这些都在后台做。

### 2. TeamCreateModal 自己知道 team 有“lead”概念

它在 `agents` 里会显式放一个 `role: 'lead'` 的 agent。

这说明 Team 模型不是“平铺的多人列表”，而是**至少有 leader / teammate 的角色差异**。

---

## 二、`teamBridge.ts` 是 Team 的边界层

`src/process/bridge/teamBridge.ts` 负责把 renderer 的 team 操作接进后台。

它注册了很多 provider，包括：

- `team.create`
- `team.list`
- `team.get`
- `team.addAgent`
- `team.removeAgent`
- `team.renameTeam`
- `team.updateWorkspace`
- `team.sendMessage`
- `team.sendMessageToAgent`
- `team.ensureSession`
- `team.stop`

其中 `safeProvider(...)` 很值得注意。

它会把异常包装成：

```ts
{ __bridgeError: true, message }
```

这说明 Team 这层特别担心一个问题：

> provider 里未处理异常会让 renderer 侧 promise 卡死。

所以它选择在桥接层显式兜底，把错误可靠送回 UI。

---

## 三、`TeamSessionService.createTeam(...)` 才是真正的“建队”实现

`src/process/team/TeamSessionService.ts` 的 `createTeam(...)` 是 Team 架构的第一核心函数。

它会做几件关键的事。

### 1. 生成 teamId

Team 本身有自己的 id，不等于任何单个 conversation id。

### 2. 为每个 agent 绑定 conversation

这一步非常关键。

Team 并不是只存一份“团队成员表”，它会为每个 agent：

- 创建或复用一个 conversation
- 把 `teamId` 写进 conversation 的 `extra`
- 回填 workspace

这意味着：

> Team 的底座仍然是 conversation。

每个 agent 的“个人工作上下文”其实还是一个 conversation，只是被归属于某个 team。

### 3. lead agent 决定团队主工作空间

如果 team 创建时没有显式给 workspace，服务端会从 lead agent 实际得到的 workspace 回填到 team 上。

这说明 workspace 不是纯 UI 字段，而是团队运行时的重要共享上下文。

---

## 四、`getOrStartSession(...)` 才是 Team 真正“活起来”的时刻

创建 team 只是把结构落库，真正启动协作的是：

- `TeamSessionService.getOrStartSession(teamId)`

这个函数会：

1. 先看内存里是否已经有 session
2. 没有就读取 team
3. 创建 `TeamSession`
4. 调 `session.startMcpServer()`
5. 把每个 agent 的 `teamMcpStdioConfig` 写回 conversation extra
6. 强制重建相关 agent task，让它们吃到新的团队 MCP 配置
7. 最后才把 session 放进 `sessions` map 缓存

这一段非常能体现作者的谨慎：

> 只有当 MCP server 已经起来、配置已经注入 DB、agent task 已经重建后，session 才算真正可用。

也就是说，Team session 的缓存不是“先占位，后补齐”，而是“初始化完成后才注册”。

---

## 五、`TeamSession` 是团队运行时总协调器

`src/process/team/TeamSession.ts` 的注释写得很直白：

> Thin coordinator that owns Mailbox, TaskManager, TeammateManager, and MCP server.

你可以把它理解成 Team 运行时的大脑外壳。它自己不做所有细节，而是组装四个子系统：

### 1. `Mailbox`

负责代理之间的消息收发。

### 2. `TaskManager`

负责团队任务的创建、状态更新、依赖解除。

### 3. `TeammateManager`

负责成员唤醒、超时、崩溃、失败、移除等生命周期管理。

### 4. `TeamMcpServer`

给 agent 暴露团队协作工具。

这说明 Team 不是前端 UI 拼出来的，而是**一个真实的运行时子系统**。

---

## 六、Team 为什么需要 Mailbox

`src/process/team/Mailbox.ts` 很薄，但意义很大。

它把代理间消息抽象成独立结构：

- `teamId`
- `toAgentId`
- `fromAgentId`
- `content`
- `type`
- `summary`
- `files`
- `read`
- `createdAt`

这说明 Team 里的“谁给谁发了什么”不是临时内存事件，而是**有持久语义的消息记录**。

所以当 `TeamSession.sendMessage(...)` 被调用时，它不是直接把用户输入硬塞给某个运行进程，而是：

1. 先确保 MCP server 已启动
2. 再把消息写入 lead 的 mailbox
3. 再把用户消息同步成 conversation 里的 user bubble
4. 最后唤醒 lead

这个顺序说明设计重点是：

> 先保证团队内消息被可靠接受，再推进 agent 执行。

---

## 七、Team 为什么需要 TaskManager

`src/process/team/TaskManager.ts` 不是 AI task manager，而是**团队任务流管理器**。

它管理的是：

- task subject
- owner
- blockedBy
- blocks
- status

并且维护依赖图。

也就是说，Team 不只是“聊天协作”，它还有明确的任务分配和阻塞关系建模。

这让 Team 更像一个小型协作操作系统，而不是一组松散 agent。

---

## 八、TeammateManager 体现了真正的多 agent 生命周期管理

`src/process/team/TeammateManager.ts` 有很多细节，尤其值得注意的是：

- inactivity timeout（长时间无响应）
- crash handling（崩溃处理）
- wake / active / idle / failed 状态管理
- 给 lead 发 idle_notification / crash testament

这说明 Team 的设计目标不是“假设每个 agent 都很可靠”，而是：

> 默认多 agent 协作会出问题，所以系统必须显式建模超时、失败、通知与恢复。

这也是 Team 层比单 conversation 层复杂得多的根本原因。

---

## 九、一个真实链路：用户给团队发消息

`TeamSession.sendMessage(...)` 的链路大致是：

```text
renderer 调 team.sendMessage
  -> TeamSessionService.getOrStartSession(teamId)
  -> TeamSession.startMcpServer()
  -> mailbox.write(to lead)
  -> addMessage(lead conversation)
  -> ipcBridge.conversation.responseStream.emit(user bubble)
  -> wake lead
```

这里最值得你学的是：

- 团队消息首先进入 mailbox
- UI 展示和 agent 执行是并行但分层的
- 唤醒 lead 是一个单独步骤

所以“团队对话”不是直接把输入发给 leader runtime，而是先走团队内部的消息总线。

---

## 十、这一层体现了什么设计理念

### 1. Team 复用 conversation，而不是另起炉灶

这样可以沿用已有存储、消息、task runtime 体系。

### 2. Team session 是运行时态，team 记录是持久态

`TTeam` 落库，`TeamSession` 缓存于内存。

### 3. 多 agent 协作必须显式建模失败

Mailbox、timeout、failed、idle_notification 都是这个理念的产物。

### 4. 协作能力通过 MCP server 提供给 agent

而不是把所有团队逻辑硬编码在某个单点函数里。

---

## 你读完这一篇后，下一步读哪里

建议接着读：

- `ai_docs/06-WebUI-与桌面双运行时.md`

因为你现在已经看了一条桌面主链路和一条 Team 协作链路，接下来最值得理解的是：

> 为什么同一套业务还能在 WebUI 运行。

---

## 证据文件

- `src/renderer/pages/team/components/TeamCreateModal.tsx:84-136`
- `src/process/bridge/teamBridge.ts:33-120`
- `src/process/team/TeamSessionService.ts:475-559`
- `src/process/team/TeamSessionService.ts:720-832`
- `src/process/team/TeamSession.ts:21-79`
- `src/process/team/TeamSession.ts:103-189`
- `src/process/team/Mailbox.ts:1-52`
- `src/process/team/TaskManager.ts:21-106`
- `src/process/team/TeammateManager.ts:335-430`
- `src/process/team/TeammateManager.ts:439-537`

---

## 容易误解的点

- **误解：Team 就是多个 conversation 标签页。**
  - 不是。它有独立 session、mailbox、task、MCP 协作层。
- **误解：创建 team 就已经启动协作。**
  - 不完全。真正启动在 `getOrStartSession(...)`。
- **误解：leader 只是 UI 上的名字。**
  - 不是。很多路由、唤醒和消息流都围绕 lead 展开。
