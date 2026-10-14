# AI 辅助开发分享

**形式：**10 分钟截图讲解，会后阅读约 10—15 分钟。  
**对象：**有基础编程能力、缺少 AI 开发经验的同事。  
**工具：**Windows 上的 Codex 桌面端，结合项目现有 OpenSpec。  
**案例：**Agent 运行期间继续输入，消息排队发送。主讲后端，前端作为同一需求的一部分。

这次只教三件事：把需求写成可验证行为；让 Codex 复用现有代码完成改动；用失败场景和证据判断结果。工具安装、AgentScope 架构、语音、费用统计和生产部署不在本次范围。

配图为 Codex 桌面风格的教学示意，对话是重建文本，不是历史聊天截图；项目页面来自仓库的历史隔离预览。本次复跑、历史证据和未验证内容在第四章分开说明。

## 10 分钟讲解顺序

| 时间 | 展示内容 | 讲解重点 |
| --- | --- | --- |
| 0:00—0:45 | 一句话需求 | AI 能写代码，但“什么算对”由人确定。 |
| 0:45—2:00 | 图 01、02 | 问清范围，把答案固定为验收场景。 |
| 2:00—4:00 | 图 03 | 主讲后端：最小改动、归属检查和执行互斥。 |
| 4:00—5:15 | 图 04 | 不会 Vue，也能提供状态规则和接口契约。 |
| 5:15—7:30 | 图 05 | 用停止后抢跑的反例检查实现。 |
| 7:30—9:00 | 图 06 | 区分类型检查、离线回归、真实联调和截图。 |
| 9:00—10:00 | 第五章问题 1 | 请同事先判断，再解释答案，给出会后资料。 |

讲师提前打开六张图，按顺序切换，不等待现场生成代码。若时间不足，缩短工具介绍，保留失败场景和验收证据。可复制文本集中在[提示词速查](提示词速查.md)。

**会后操作顺序：**在 Codex 桌面端打开已有项目；把后端与前端目录作为本次任务上下文；新建本地聊天，依次使用速查的需求澄清、规格、实现和验收提示词。先查看相关规格与已有改动，避免在已完成功能上机械重复实现。需要重演旧功能时，应另用教学分支或工作副本，不覆盖现有工作。

## 一、需求与验收

### 从一句话开始

> Agent 运行时，我还想继续输入下一条消息。

不要马上让 AI “加个队列”。先判断这里的“队列”是页面里的待发消息，还是后台持久化任务系统。后者涉及刷新恢复、跨设备等要求，这次没有提出。

![图01：需求澄清](images/01.png)

**讲师说：**“先让 Codex 阅读项目，再问会改变方案的问题。我们不是考它能列出多少问题，而是尽快确定边界。比如，排队要不要刷新恢复？工具等待确认时还发不发？停止后怎样判断上一轮已结束？”

这次明确：队列仅存在于当前页面；默认按顺序发送；取消消息不发送；待确认或执行失败时暂停；切换会话、身份后清理旧队列。插队属于后续扩展，不能默默增加后台调度系统。

### 用 OpenSpec 保存正确预期

![图02：规格与范围](images/02.png)

| 场景 | 可观察的验收结果 |
| --- | --- |
| A 运行时提交 B、C | A 不被普通排队打断；B、C 等待并按顺序执行。 |
| B 尚未发送时取消 B | 不产生 B 的 `/runs` 请求，也不进入已执行历史。 |
| 客户端流结束，后端仍运行 | B 继续等待，不提前启动下一轮。 |
| 遇到工具待确认 | 暂停自动发送，不自动批准或拒绝。 |
| 切换账号或会话 | 旧队列和旧异步回调不得向新会话发送。 |

`proposal.md` 写为什么改、范围是什么；`specs` 写行为与场景；`design.md` 写复用方案；`tasks.md` 写实施与验证。实现和审查读取同一份规则，减少凭聊天记忆判断。[OpenSpec 结构说明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md)

**学员检查点：**能否说出至少一个“页面看起来正常，但实际违反需求”的场景？例如，取消后消息仍被发出去。说不出来时，先补场景，而不是继续堆提示词。

## 二、用 Codex 推进前后端实现

### 后端：优先复用现有能力

![图03：后端最小改动](images/03.png)

**讲师说：**“后端不必接管页面待发列表。现有 `/runs` 执行消息，`/interrupt` 请求停止，我们补充或核对 `/execution`，让前端知道上一轮是否还在运行。历史需求曾被扩大成后台持久化队列，最后通过人工纠偏收回到页面交互。”

| 接口 | 职责 | 必须检查的边界 |
| --- | --- | --- |
| `POST /conversation/agentscope/conversations/{id}/runs` | 发起一轮运行，返回事件流。 | 后端原子互斥仍保留，占用时拒绝新运行。 |
| `POST .../{id}/interrupt` | 请求协作式停止。 | 请求成功不等于执行已经释放。 |
| `GET .../{id}/execution` | 返回 `data.running`。 | 先核对当前登录用户能否访问会话，再读共享租约。 |

核心路径：`ConversationController.execution → sessions.resolve → AgentScopeConnection.isRunning → DistributedSessionTurnGate`。入口代码很短，但审查重点是读取谁的状态、状态来自哪里，以及运行入口是否仍有原子保护。[后端入口源码](https://github.com/onthebed/phoenix-service/blob/e03537acf670f3017f50bc15f774351073c1e56c/phoenix-modules/phoenix-agent-web/src/main/java/org/phoenix/agent/web/controller/ConversationController.java#L119)

**不能只靠先查询再执行。**查询得到 `running=false` 后，另一请求仍可能先获取执行权。因此 `/execution` 是调度观察，`/runs` 的互斥是执行保护，不能让 AI 为了“方便排队”删掉后者。

每轮只要求一个可检查结果：先把执行状态和归属测试做好，再推进页面。代码已经存在时，让 Codex 核对并补缺口，避免机械重写。

### 前端：不会 Vue，也要说清交互状态

![图04：用接口与状态指导前端](images/04.png)

后端开发者不需要先写 Vue，但需要提供三类输入：

- **页面状态：**工作中也能输入，发送后进入待发列表；待发项可取消；失败或确认等待时暂停。
- **接口契约：**使用既有 API 封装，`running=true` 时不派发，状态不明时保留消息并提示；未发内容不进入模型历史。
- **实现约束：**沿用当前 Vue、组件和样式；不另建框架或持久化队列；切换身份与会话清理旧异步结果。

**讲师说：**“这里需要我决定交互规则，Vue 代码让 Codex 写。生成后我不只问好不好看，还问：取消有没有真的阻止请求？停止后立即发送有没有抢跑？错误时消息有没有丢？”

图中的页面取自历史真实 Vue 组件隔离预览，网络与 Agent 回复使用模拟数据。它证明当时的组件呈现，不证明今天真实后端可用。可[单独查看页面原图](images/项目队列原图.png)。项目使用 Vue 3、TypeScript、Vite、Element Plus、`@ag-ui/client`，应沿用现有实现。

实现可以由 OpenSpec 的 apply 流程推进；规格、设计发生变化时再更新对应文件。Codex 的具体技能名以项目生成的文件为准，不能把终端里的 `openspec ...` 命令与聊天中的技能调用混用。[OpenSpec 命令说明](https://github.com/Fission-AI/OpenSpec/blob/main/docs/commands.md)

## 三、验证与排障

### 用一个反例推翻错误假设

![图05：停止后抢跑](images/05.png)

**先让学员判断：**“`isStreaming=false`，是不是就能立即发送下一条？”

答案是不能。客户端流结束、停止请求返回、后端释放执行权是不同事件。仓库记录中，历史隔离预览通过模拟服务端收尾延迟，暴露了停止后空队列立即发送的竞态。

| 事件 | 页面观察 | 正确行为 |
| --- | --- | --- |
| 用户请求停止 | 停止接口可能已经返回。 | 标记后端可能仍在收尾。 |
| 浏览器流结束 | `isStreaming=false`。 | 新消息保留在队列。 |
| 查询执行状态 | `running=true`。 | 等待，不启动下一轮。 |
| 后端释放 | `running=false`，且没有待确认。 | 尝试派发，由后端互斥兜底。 |

修复要围绕这个反例：用 `executionMayBeFinishing` 保留收尾边界，让停止后的新消息也进入待发逻辑；通过 `/execution` 核对释放；查询失败或超时则暂停并保留消息。不要靠固定等待两秒、删互斥或无限重试掩盖问题。

### 回归测试要断言行为

现有测试 `first new message after stop waits for server release even with an empty queue` 检查：

1. 第一轮结束客户端流，但模拟后端仍忙。
2. 立即提交新消息，新消息留在待发列表。
3. 后端释放前，实际运行次数仍为 1。
4. 释放后运行次数才变成 2，内容就是那条新消息。

断言“按钮恢复发送”不够；断言请求次数、顺序和消息内容，才回答是否抢跑。[队列回归源码](https://github.com/onthebed/phoenix-web/blob/aa1fb0fc5b87cbd8e5f74e20ce044d068aae0dd2/tests/conversation-queue.test.mjs#L363)

**讲师说：**“对抗性审查不等于要求 AI 多挑毛病。让它给出触发条件，再把成立的反例固化成回归。测试失败后，先核对测试预期和接口字段，也可能是探针写错，不要直接改业务代码。”

## 四、代码审查与验收

![图06：限定证据范围](images/06.png)

### 三个必问问题

1. **边界有没有被突破？**会话归属是否校验？确认是否必须由人处理？旧身份回调会不会串发？
2. **是否引入不必要的机制？**页面队列是否被升级成数据库队列？是否为了降低报错率删除互斥？
3. **验证是否回答需求？**是否覆盖取消、失败、停止后发送与会话切换？哪些只是模拟，哪些接了真实服务？

| 证据 | 能说明什么 | 本案例状态 |
| --- | --- | --- |
| 格式、类型与 OpenSpec 校验 | 文件与接口约束符合相应检查。 | 不能单独证明交互正确。 |
| 本次离线队列回归 | 当前快照的 Vue 队列逻辑满足模拟场景断言。 | **17/17 通过**，实际执行，网络与服务端模拟。 |
| 历史真实 HTTP 检查 | 2026-10-13 单实例服务的指定接口行为。 | 19/19；没有在本次重跑。 |
| 历史实际队列接真实后端 | 当时 Vue/API/官方客户端到真实服务的流程。 | 17/17；没有浏览器渲染，记录 8 次真实运行。 |
| 历史隔离浏览器检查 | 当时页面组件及模拟交互。 | 34/34；模拟 Agent/ASR，不代表生产联调。 |
| 截图和任务勾选 | 帮助理解状态或记录进度。 | 不作为业务正确性的充分证据。 |

本次没有重跑 Java 测试、完整项目构建、真实后端、模型、数据库或完整浏览器验收；真实多实例与 EHR 提交也没有验证。历史记录的脏工作区完整起点和服务二进制版本未完全保存，不能承诺按某个提交即可重现全部历史结果。

验收完成后再同步、归档 OpenSpec 变更，主规格用于后续需求。规格通过校验、tasks 打勾或归档完成，都不等于业务已经验收。

## 五、综合判断与复盘

### 会中问答：确认等待时怎么办？

**问题：**用户提交一条待发消息，此时上一轮出现“提交申请前请确认”。消息是否应该继续自动发送？

**答案：**暂停待发队列，保留消息，等待用户处理确认。不能把排队消息当成同意，也不能为了让队列继续而自动批准或拒绝。检查 `pendingInterrupts` 对派发的保护，并验证确认期间没有新 `/runs` 请求。

### 会后自测：比较两个方案

**问题：**A 方案停止后固定等两秒；B 方案核对 `/execution` 并保留后端互斥。选哪个？还缺什么？

**答案：**B 更符合执行状态语义，但仍需处理查询失败、等待超时、身份切换和查询后的竞争。让等待期限受控，失败时暂停并保留消息，运行入口继续拒绝并发占用。必须用断言验证这些行为。

### 复盘要留下什么

记录四项即可：错误假设是什么；哪个场景推翻了它；修复如何改变行为；哪些证据支持修复、哪些范围仍未验证。能独立解释这四项，才说明学会了方法。只重复操作、复制 Prompt 或运行命令，不够。

本次分享的结束语：**“先定义正确行为，让 AI 做最小实现，再用反例和证据判断。OpenSpec 保存规则，Codex 推进任务，判断责任仍在人。”**

## 附录：复现与资料

### 案例版本

- 后端：`onthebed/phoenix-service`，`master` 快照 `e03537acf670f3017f50bc15f774351073c1e56c`。
- 前端：`onthebed/phoenix-web`，`master` 快照 `aa1fb0fc5b87cbd8e5f74e20ce044d068aae0dd2`。
- 分支快照固定的是本次读取版本；历史记录另有自己的起点、证据范围与限制。两者不能混称。

### 会后可运行的命令

在**后端项目终端**运行，输出需自行核对，本文未声称本次重跑：

```powershell
openspec validate add-conversation-message-queue --strict --no-interactive
mvn -pl phoenix-modules/phoenix-agent-web -am -Dtest=ConversationControllerRequestBodyTest,DistributedSessionTurnGateTest -Dsurefire.failIfNoSpecifiedTests=false test
```

在**前端项目终端**运行：

```powershell
node --test tests/conversation-queue.test.mjs
pnpm exec vue-tsc -b --pretty false
```

本次在隔离资料目录使用相同队列测试及其源码依赖实际执行了第一条前端回归。没有运行项目类型检查或把测试模拟器当真实联调。[本次核对摘要](evidence/本次核对摘要.json)

### 源码与历史材料

- [后端队列规格](https://github.com/onthebed/phoenix-service/blob/e03537acf670f3017f50bc15f774351073c1e56c/openspec/changes/add-conversation-message-queue/specs/conversation-message-queue/spec.md)
- [前端输入区规格](https://github.com/onthebed/phoenix-web/blob/aa1fb0fc5b87cbd8e5f74e20ce044d068aae0dd2/openspec/changes/align-conversation-composer/specs/conversation-composer/spec.md)
- [后端执行状态测试](https://github.com/onthebed/phoenix-service/blob/e03537acf670f3017f50bc15f774351073c1e56c/phoenix-modules/phoenix-agent-web/src/test/java/org/phoenix/agent/web/controller/ConversationControllerRequestBodyTest.java#L97)
- [前端状态与队列逻辑](https://github.com/onthebed/phoenix-web/blob/aa1fb0fc5b87cbd8e5f74e20ce044d068aae0dd2/src/pages/conversation/useConversation.ts#L307)
- [实际后端验证记录](https://github.com/onthebed/phoenix-service/blob/e03537acf670f3017f50bc15f774351073c1e56c/vibecodingdoc/2026-10-13/20261013-101220-codex-composer-backend-validation.md)
- [历史前端实施记录](https://github.com/onthebed/phoenix-web/blob/aa1fb0fc5b87cbd8e5f74e20ce044d068aae0dd2/vibecodingdoc/2026-10-13/20261013-101220-codex-composer.md)

私有仓库链接需要已有访问权限，教材不包含登录凭证。

### 教学方法参考

- [The Carpentries：Live Coding is a Skill](https://carpentries.github.io/instructor-training/instructor/17-live.html)：参考讲解操作与输出、展示排障过程、控制认知负担的做法。本分享采用截图而非完整现场编程，不照搬长课形式。
- [Anthropic：Effective harnesses for long-running agents](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)：参考增量任务、清晰状态记录和用户路径验证；讲师与学员仍只使用 Codex 桌面端。
- [OpenAI Docs：Windows 桌面端](https://learn.chatgpt.com/docs/windows/windows-app)：核对 Windows 项目、终端和审查能力。教学图不承诺像素级复现任何版本界面。
