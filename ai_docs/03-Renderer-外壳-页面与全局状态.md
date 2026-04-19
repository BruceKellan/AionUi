# Renderer 外壳、页面与全局状态

## 这篇文档解决什么问题

前两篇你已经知道：

- AionUi 是多运行时系统；
- bridge 是通信中枢。

这一篇开始回到前端，但不是去看零散组件，而是先看 **renderer 外壳**。

你要回答的问题是：

> React 这层到底怎么组织？哪些状态是全局的？哪些页面是入口级页面？桌面模式和 WebUI 模式在前端层有什么差异？

---

## 一、Renderer 入口并不复杂，但非常“克制”

`src/renderer/main.tsx` 做的事情很集中：

1. 注册 Electron 环境下的 Sentry
2. 导入 `browser` adapter
3. 挂载 React Provider 树
4. 设置 Arco Design（组件库）和 i18n
5. 注册 PWA
6. 渲染 Router + Layout

它没有在入口里塞业务逻辑。这是一个好信号。

说明前端入口主要负责：

- **壳**
- **上下文**
- **主题与国际化**
- **路由挂载**

而不是“顺手把业务做了”。

---

## 二、Provider 树是前端的第一层骨架

在 `main.tsx` 里，最重要的不是 JSX 长什么样，而是 Provider 的顺序。

当前主入口可以概括成：

```text
AuthProvider
  -> ThemeProvider
    -> PreviewProvider
      -> ConversationTabsProvider
        -> ConfigProvider(Arco)
          -> Router
            -> Layout
              -> ConversationHistoryProvider
```

这条链路说明了前端的几类全局状态：

### 1. 认证状态

由 `AuthProvider` 管。

这是最靠外的一层，因为几乎所有页面都依赖“当前是否可访问”。

### 2. 主题状态

由 `ThemeProvider` 管。

说明主题并不是某个页面的局部状态，而是全局 UI 能力。

### 3. 预览与会话页上下文

`PreviewProvider` 和 `ConversationTabsProvider` 把 conversation（会话）相关的 UI 能力提前挂在外层。

### 4. 历史会话上下文

`ConversationHistoryProvider` 是放在 `Layout` 内部的，说明它与主面板布局强相关。

---

## 三、路由层是“受保护的控制台”，不是公开站点

`src/renderer/components/layout/Router.tsx` 显示得非常清楚：

- 路由器使用 `HashRouter`
- 有 `ProtectedLayout`
- `/login` 是登录页
- 其他主要页面都在受保护区域里

当前主要路由包括：

- `/guid`
- `/conversation/:id`
- `/team/:id`
- 各种 `/settings/...`
- `/scheduled`
- `/test/components`

这说明 AionUi 的前端不是“营销站点”式结构，而更像一个**带后台控制面板特征的工作台应用**。

---

## 四、AuthContext 是桌面模式和 WebUI 模式分叉的关键点

`src/renderer/hooks/context/AuthContext.tsx` 很值得初学者仔细看。

它做了一个关键判断：

```ts
const isDesktopRuntime = typeof window !== 'undefined' && Boolean(window.electronAPI)
```

然后分成两套行为：

### 桌面模式

- 直接把状态设为 `authenticated`
- 不走 `/api/auth/user`
- 不依赖 Cookie / CSRF

### WebUI 模式

- 会请求 `/api/auth/user`
- 登录请求走 `/login`
- 依赖 CSRF token（跨站请求伪造防护）
- 登陆成功后会重连 WebSocket

这能帮你理解一个重要事实：

> 桌面模式和 WebUI 模式的“前端壳”看起来接近，但认证模型并不相同。

---

## 五、Layout 是 UI 壳，不是业务页面

`src/renderer/components/layout/Layout.tsx` 很长，但你第一次读时不要陷进去。

先只把它理解成 5 类职责：

1. **侧边栏收起 / 展开与移动端适配**
2. **标题栏与窗口控制**
3. **自定义 CSS / 主题同步**
4. **深链、通知点击、快捷键等系统级交互**
5. **为 `Outlet` 提供主工作区壳**

从 `workspaceAvailable` 的判断还能看出：

- `/conversation/...` 和 `/team/...` 被视为主工作区页面
- Team 页面在布局层已经被视为 conversation 同级工作区

这再次说明：

> Team 不是“附加功能页”，而是主工作流的一部分。

---

## 六、Renderer 页面组织的阅读顺序

如果你现在是初学者，不建议随机点开组件目录。建议按下面的顺序读：

### 第一步：入口壳

- `src/renderer/main.tsx`
- `src/renderer/components/layout/Router.tsx`
- `src/renderer/components/layout/Layout.tsx`

目标：先看页面怎么被挂起来。

### 第二步：全局上下文

- `src/renderer/hooks/context/AuthContext.tsx`
- `src/renderer/hooks/context/ThemeContext.tsx`
- `src/renderer/hooks/context/ConversationHistoryContext.tsx`

目标：搞清楚哪些状态是全局态。

### 第三步：具体页面

推荐先看：

- `src/renderer/pages/team/...`
- `src/renderer/pages/settings/...`
- `src/renderer/pages/conversation/...`

因为它们分别代表：

- Team 协作
- 配置与系统控制
- AI 主业务界面

---

## 七、为什么前端入口要这样设计

从代码看，Renderer 层有三个明显设计取向。

### 1. 用 Provider 隔离横切关注点

认证、主题、预览、历史会话都不是页面私有逻辑，所以放在上下文层。

### 2. 用 Layout 隔离工作台壳

这样每个页面不用重复管理标题栏、侧边栏、主内容区域。

### 3. 让业务动作尽快离开组件层

组件只收集输入、更新局部状态、调用 `ipcBridge`。真正行为尽量继续往后台桥接层走。

这也是为什么你在页面里经常会看到：

```ts
ipcBridge.xxx.invoke(...)
```

而不是直接 import 主进程服务。

---

## 八、一个最值得初学者建立的心智模型

读 Renderer 时，你可以一直拿下面这张图套代码：

```text
main.tsx        -> 挂壳
Router.tsx      -> 分页面
Layout.tsx      -> 提供工作台外框
Context         -> 提供全局状态
Page            -> 收集用户意图
ipcBridge       -> 把意图送到后台
```

只要你坚持这个顺序，你就不会把“页面代码”和“真实业务实现”混为一谈。

---

## 你读完这一篇后，下一步读哪里

接下来建议先读：

- `ai_docs/04-Conversation-主业务链路.md`

因为现在你已经知道前端壳怎么组织了，下一步就应该顺着一条最核心的业务链路，去看一次请求如何真正落到后台。

---

## 证据文件

- `src/renderer/main.tsx:15-132`
- `src/renderer/components/layout/Router.tsx:1-89`
- `src/renderer/hooks/context/AuthContext.tsx:1-260`
- `src/renderer/components/layout/Layout.tsx:85-260`
- `src/common/config/constants.ts:55-61`

---

## 容易误解的点

- **误解：前端入口很轻，所以前端不重要。**
  - 不是。它轻，恰恰说明边界划得清楚。
- **误解：AuthContext 只是登录页逻辑。**
  - 不是。它决定整个 renderer 的访问态。
- **误解：Layout 只是样式容器。**
  - 不是。它承载了不少工作台级状态和系统交互。
