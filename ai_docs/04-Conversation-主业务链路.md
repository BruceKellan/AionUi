# Conversation 主业务链路

## 这篇文档解决什么问题

前面你已经知道：

- 项目是多运行时系统；
- bridge 负责通信；
- renderer 只是 UI 外壳。

这一篇开始进入**最核心的一条真实业务链路**：conversation（会话）。

你要回答的问题是：

> 用户发起一个新会话时，项目到底做了什么？
> 用户真正发送消息时，后台又是怎么把它交给 agent runtime 的？

---

## 先给总图

把 conversation 链路先压成一句话：

```text
Guid / 页面动作
  -> ipcBridge.conversation.create.invoke(...)
  -> conversationBridge.create.provider(...)
  -> ConversationServiceImpl.createConversation(...)
  -> createGeminiAgent / createAcpAgent / createAionrsAgent ...
  -> 写入 repository
  -> 返回 conversation
  -> 页面导航到 /conversation/:id

用户发送消息
  -> ipcBridge.conversation.sendMessage.invoke(...)
  -> conversationBridge.sendMessage.provider(...)
  -> workerTaskManager.getOrBuildTask(...)
  -> 对应 AgentManager.sendMessage(...)
```

只要你抓住这两段，conversation 主链路就不会乱。

---

## 一、会话的“创建入口”其实不在 Conversation 页面里

一个很容易踩的坑是：

> 初学者会以为 Conversation 页面自己创建会话。

实际上，从当前代码看，很多新会话是从 **Guid 页** 或其他入口先创建，再跳转进 `/conversation/:id`。

例如 `src/renderer/pages/guid/hooks/useGuidSend.ts`：

- 根据用户选中的 agent 类型，构造不同的 conversation 参数；
- 调用 `ipcBridge.conversation.create.invoke(...)`；
- 拿到返回的 conversation 后，把初始消息暂存在 `sessionStorage`；
- 最后导航到 `/conversation/${conversation.id}`。

这说明：

- “创建会话”和“展示会话页面”是两件事；
- 页面跳转之前，后台 conversation 已经存在了；
- conversation 页面往往接手的是一个**已存在会话**。

---

## 二、`conversationBridge.create.provider(...)` 是创建入口的后台落点

在 `src/process/bridge/conversationBridge.ts` 里，`ipcBridge.conversation.create.provider(...)` 是主入口。

它做了几件关键事情：

### 1. 校验 conversation type

只允许这些类型：

- `gemini`
- `acp`
- `codex`（会被重映射）
- `openclaw-gateway`
- `nanobot`
- `remote`
- `aionrs`

这说明 bridge 层不是单纯透传，它会先做一层边界约束。

### 2. 处理兼容与映射

`codex` 不直接走独立类型，而是会重映射为 `acp`，并把 backend 写进 `extra.backend`。

这反映出一个设计特点：

> Conversation 的“显示类型”和底层 runtime 类型不一定一一对应。

### 3. 交给 `ConversationServiceImpl.createConversation(...)`

bridge 层不负责真正创建 conversation 对象，它把请求交给 service 层。

这也是整个项目一贯的分层方式：

- bridge：边界与调度
- service：业务创建
- repository：落库存储

---

## 三、`ConversationServiceImpl` 负责把“创建意图”变成真实 conversation

`src/process/services/ConversationServiceImpl.ts` 的 `createConversation(...)` 非常关键。

它根据 `params.type` 分流到不同的 factory（工厂函数）：

- `createGeminiAgent(...)`
- `createAcpAgent(...)`
- `createOpenClawAgent(...)`
- `createNanobotAgent(...)`
- `createRemoteAgent(...)`
- `createAionrsAgent(...)`

这说明 conversation 的创建本质上不是“建一条数据库记录”这么简单，而是：

> 先根据 backend 规则构造一个完整 conversation 对象，再持久化。

### 这里有两个重要观察

#### 1. 会话对象是运行时配置的载体

conversation 不只是：

- id
- name
- type

它还带很多运行时信息，尤其在 `extra` 里，比如：

- `workspace`
- `customWorkspace`
- `backend`
- `enabledSkills`
- `excludeBuiltinSkills`
- `presetAssistantId`
- `sessionMode`

#### 2. 工厂函数会顺手准备 workspace

`src/process/utils/initAgent.ts` 里的 `createGeminiAgent(...)`、`createAcpAgent(...)` 等不会只返回类型信息，它们还会：

- 计算工作目录
- 为临时 workspace 建目录
- 为原生 skill 发现创建 symlink（符号链接）
- 组织 `extra` 字段

所以 conversation 的创建同时也是一次**运行环境准备**。

---

## 四、会话创建后，前端为什么没有立刻发送消息

从 `useGuidSend.ts` 可以看到，创建完成后，很多路径会把“初始消息”先写进 `sessionStorage`。

这说明项目把流程拆成了两步：

1. 先确保 conversation 创建成功；
2. 再由具体平台的 SendBox 读取初始消息并真正发送。

这样拆的好处是：

- 页面跳转可以稳定落到已存在会话；
- 不会把“创建失败”和“发送失败”混成一个错误；
- 不同平台（Gemini / Aionrs / Remote）可以有自己的发送箱逻辑。

---

## 五、真正发送消息时，`workerTaskManager` 才开始接管

发送消息的后台入口在 `conversationBridge.ts` 的：

- `ipcBridge.conversation.sendMessage.provider(...)`

这段逻辑的核心是：

1. 先根据 `conversation_id` 获取或构建 task
2. 如果 task 不存在，则由 `workerTaskManager.getOrBuildTask(...)` 懒创建
3. 再调用具体 `IAgentManager` 的发送行为

这就是 conversation 链路里最重要的分界点：

> 创建 conversation 并不等于 runtime 已经完全运行；
> runtime 往往是在发送消息时，或者 warmup 时，被真正拉起来的。

---

## 六、`WorkerTaskManager` 是 conversation 和 runtime 之间的连接器

`src/process/task/WorkerTaskManager.ts` 做了 4 件重要的事：

### 1. 用 `conversationId` 找 task

它把 conversation 和运行中的 agent task 关联起来。

### 2. 懒创建 task

如果缓存里没有，就从 repository 读取 conversation，再通过 factory 构建。

### 3. 负责替换与 kill

如果同一个 conversation 重新构建 task，会先 kill 旧进程，避免 orphan process（孤儿进程）。

### 4. 负责空闲清理

CLI 型 agent（如 `acp`、`aionrs`）空闲超时后会被自动清理。

这意味着：

- conversation 是持久状态；
- task 是运行时状态；
- `WorkerTaskManager` 是两者之间的桥。

---

## 七、一个具体发送路径例子：Aionrs

`src/renderer/pages/conversation/platforms/aionrs/AionrsSendBox.tsx` 很适合当例子。

它在用户发送消息时会：

- 先把 user message 写入 UI / message cache；
- 如果当前 conversation 属于 team，就改走 `ipcBridge.team.sendMessage` 或 `team.sendMessageToAgent`；
- 否则调用 `ipcBridge.conversation.sendMessage.invoke(...)`；
- 失败时回滚刚刚加到界面上的消息。

这说明前端 SendBox 做的是：

- 输入组织
- 乐观更新 / UI 提前显示
- 错误回滚
- 选择正确后端入口

而真正 AI 执行还是在后台 task manager 一侧。

---

## 八、Conversation 链路里还有哪些重要辅助动作

### 1. warmup

`conversationBridge` 里有 `conversation.warmup`。

它会：

- 预先 `getOrBuildTask(...)`
- 对 ACP conversation 额外触发 `initAgent()`

它的目的不是发送消息，而是**提前热身 runtime**，减少第一次发送时的等待。

### 2. get / update / remove

`conversationBridge` 还统一处理：

- `get`
- `update`
- `remove`
- `reset`

例如删除时，它不只是删库，还会：

- kill 对应 task
- 如果来源不是 `aionui`，还会尝试清理 channel 相关资源

这说明 bridge 层承担了一部分生命周期编排责任。

---

## 九、这一条链路体现了什么设计理念

### 1. conversation 是“长期状态”

它持久化，能恢复，能迁移，能跨页面。

### 2. task 是“短期运行态”

它可以懒创建、被回收、被重建。

### 3. 创建与发送分离

先创建 conversation，再决定何时真正启动 runtime。

### 4. renderer 只负责意图表达

创建和发送的真正落地都不在页面组件里。

---

## 你读完这一篇后，下一步读哪里

建议接着读：

- `ai_docs/05-Team-多Agent协作架构.md`

因为 Team 模式本质上是在单 conversation 模式之上，再叠加一层 session、mailbox、任务协调机制。

---

## 证据文件

- `src/renderer/pages/guid/hooks/useGuidSend.ts:150-206`
- `src/renderer/pages/guid/hooks/useGuidSend.ts:315-360`
- `src/process/bridge/conversationBridge.ts:127-173`
- `src/process/bridge/conversationBridge.ts:371-518`
- `src/process/services/ConversationServiceImpl.ts:124-197`
- `src/process/utils/initAgent.ts:34-129`
- `src/process/utils/initAgent.ts:161-260`
- `src/process/task/WorkerTaskManager.ts:21-122`
- `src/renderer/pages/conversation/platforms/aionrs/AionrsSendBox.tsx:184-217`
- `src/renderer/pages/conversation/platforms/aionrs/AionrsSendBox.tsx:259-284`

---

## 容易误解的点

- **误解：会话页面负责创建会话。**
  - 不一定。很多会话在进入页面前就创建好了。
- **误解：创建 conversation 就等于 AI 已经启动。**
  - 不等于。runtime 往往要等发送消息或 warmup。
- **误解：task 就是 conversation。**
  - 不是。conversation 是持久状态，task 是运行时实例。
