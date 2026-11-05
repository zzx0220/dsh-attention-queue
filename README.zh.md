# dsh-attention-queue

[English](./README.md) | 简体中文

本插件由 AI 生成(DeepSeek Harness,`deepseek-flash` 模型)。

固定在 DeepSeek Harness Web 会话列表上方的持久化待处理队列,用于回答一个跨工作区的问题:
**哪些会话需要我处理?**

队列中的每一条记录处于两种状态之一:

- **待查看**:会话在你看别的地方时跑完了。
- **待回复**:你已经打开过它,但它仍停留在审批、计划评审或提问上。

界面文案跟随宿主应用的语言设置:英文环境显示英文,中文环境显示中文。

本插件是纯前端实现:没有构建步骤,也没有运行时依赖。`client.js` 是手写的
`ModuleLoader` bundle,`react` 直接取自宿主。

## 安装

依赖:DSH CLI(`dsh`)、`pnpm`,以及 `git`(安装 git 依赖时使用)。

macOS、Linux 与 Windows 使用同一条命令:

```sh
dsh plugin --profile desktop add github:zzx0220/dsh-attention-queue
```

安装后重启 DeepSeek Harness,客户端 bundle 在启动时组装。插件生效后,队列卡片出现在
侧边栏顶部。

### 确认登记

`dsh plugin` 只是把参数转发给 pnpm,因此"下载包"与"登记到 profile 的 bundle 列表"是两步。
安装完成后确认一次:

```sh
dsh --profile desktop --dump-config | grep attention-queue
```

有输出即表示已登记。若无输出,在 `~/.dsh/profiles/desktop/package.json` 的
`dsh.profile.bundles` 数组中补上 `"dsh-attention-queue"`,然后重启。

### 升级

git 依赖锁定在安装时的 commit,不会跟随仓库更新。需要新版本时显式指定:

```sh
dsh plugin --profile desktop add github:zzx0220/dsh-attention-queue#v0.1.1
```

### 卸载

```sh
dsh plugin --profile desktop remove dsh-attention-queue
```

随后从 `dsh.profile.bundles` 中删除对应条目并重启。

## 与已有指示器的区别

宿主 UI 已经掌握本队列所依赖的三项事实,但没有对它们做累积:

| 事实 | 来源(DSH Web 内部 store) |
| --- | --- |
| 会话正在运行或已停止 | `useSessionStatus` → `running` |
| 会话正在等待你 | `pendingInteraction.kind` = `approval` / `plan-review` / `question` |
| 运行结束但未被查看 | `completionUnread` |

`completionUnread` 是页面内存中的 `Set`,刷新后即丢失;同时这些状态只以小组点的形式分散在
各工作区分组内。本插件维护独立的**持久化**队列,并将其渲染为一张固定卡片。

## 状态迁移

- 会话由 `running` 变为停止,且你正在查看的是另一个会话 → 进入**待查看**。
- 出现 `pendingInteraction` → 直接进入**待回复**,优先级最高。它压过 `running`:审批是卡在
  回合*内部*等待的,期间会话一直上报 `running`,若先判 `running` 就永远看不到待回复。
  正在查看的那个会话同样会入列——审批、提问、计划评审不回答就不会继续。
- 打开一条**待查看**记录时,该行保留在列表中,副标题变为*查看中*:正在阅读的条目不应突然消失。
- 离开一条已查看的**待查看**记录即移除;若此时它已在等你回答,则已经是**待回复**。
- 会话重新开始运行(你已回复)→ **待回复**行立即移除。
- 推断出的**待回复**在会话不再等待时也会消失(含刷新后已无待办的情形);只有你手动
  *留待回复* 的那条会一直留着,直到点 *已处理*。对仍挂着的审批点 *已处理*,在那段活动里
  不会被重新加回。
- 悬停某一行可执行*留待回复*与*标为已处理*,用于覆盖自动判定无法处理的情况,例如助手在
  正文中提问。
- 重启后,`updatedAt` 晚于上次离开客户端时间、且从未被处理过的会话,会恢复为**待查看**。

## 实现说明

- **挂载点**:`shell.overlay`,这是一个 list 类型、root 作用域的插槽,专供 badge、toast 与状态条
  使用。它默认穿透点击,由条目自行选择接收指针事件。
  `sidebar.workspaces` 是 **single** 占用槽,占用它会遮蔽宿主的会话列表。
- **定位**:测量 `[class*="_regionArea"]` 的包围盒,将卡片放在会话区域正上方;无法获取该元素时
  退回到框架角落。
- **导航**:`ctx.uiWorkspace.openSession(id)`,与宿主 fork-and-jump 走同一条路径。
- **通知**:`new Notification`,同一批次在 1.2 秒窗口内合并为一条通知;点击通知可直接打开该会话。

## 已知限制

- "待回复"依赖 `pendingInteraction` 与手动标记。助手在正文中提出的问题不会被自动识别。
- 队列按 origin 存放在 `localStorage`。多窗口共享同一份数据,后写入的内容覆盖先写入的内容。
- 没有端到端测试。验证方式是安装插件并完成一个会话,观察它是否进入队列。
- 插件依赖宿主内部实现:`shell.overlay` 挂载点、`useSessionStatus` / `useSessions`,以及
  `_regionArea` 类名都不是稳定 API,DSH 的小版本升级可能使其失效。在新机器上安装后请执行验收:
  卡片出现、跑完的会话进入队列、点击条目可跳转、通知可弹出。

## 开发

```sh
node scripts/smoke.mjs
```

脚本以 stub 的 `window` / `localStorage` / `Notification` 加载 `client.js`,驱动状态机完成关键迁移,
是任何改动后的第一道回归检查。`client.js` 没有构建步骤,重启宿主即可加载改动。

本地迭代时也可以把工作目录以 `link:` 依赖的形式接入 profile,做法即早期版本中描述的软链方式。

## 许可证

MIT
