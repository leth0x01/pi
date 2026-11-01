# Paperclip + Pi：公司多 Agent 研发平台执行器接入方案

> 文档状态：架构纠偏与后续 PoC 计划（2026-11-01）。
>
> **首要前提：公司 Agent 平台已经采用 Paperclip 作为控制面。Pi 是要接入 Paperclip 的执行器，不是另建一套多 Agent 平台。**
>
> 本文为 `leth0x01/pi` 的执行器接入与扩展方案，不取代现有 Paperclip 总体架构文档。该 Fork 是公开仓库；不得写入实际凭据、内部访问令牌、业务数据、真实内部拓扑 IP 或未获授权公开的信息。

## 1. 已确定事实和不再讨论的方向

- 控制面：**Paperclip**，负责组织、项目、Agent 定义、任务分配、心跳/唤醒、审批、预算、执行记录和进度展示。
- 执行面：**Pi 为主要编码 Agent 运行时**；后续可以使用 OpenCode 等其他执行器，由 Paperclip 管理角色与任务归属。
- 已有验证记录：Paperclip `v2026.1023.0` + `pi_local` + Pi（`@earendil-works/pi-coding-agent` 1.4.0）+ DeepSeek 已跑通基本任务调用。此前记录还涉及已有多 Agent 角色与审批等 Paperclip 配置。**这些是历史验证记录，本文未再次登录服务器或复验当前健康状态。**
- GitHub Fork：`leth0x01/pi`，保留 `main` 与 `develop`。**尚未证明 Paperclip 目前使用的是 Fork 的二进制**：已安装的 Pi 和 GitHub 的 Fork 不自动等同。
- 优先沿用 Paperclip 的官方 `pi_local` Adapter；不重新实现任务控制面、状态机、组织角色、心跳、审批、成本计量和通用 Agent 编排。
- Paperclip Runner / ACP Pi 路线可以保留为未来评估项，**不要因其存在而擅自切换正在运行的 `pi_local`**。

## 2. 正确的系统关系

```text
                 员工 / 业务需求 / Web 或聊天入口
                                |
               +----------------v-----------------+
               |          Paperclip               |
               |  组织、项目、任务、Agent 管理      |
               |  分配、审批、预算、心跳、运行记录   |
               +----------------+-----------------+
                                |
                         adapter / execution target
                                |
                  +-------------+-------------+
                  |                           |
             pi_local                    opencode_local
                  |                           |
       +----------v----------+         +------v-------+
       |  Pi Fork Executor   |         | 其他执行器    |
       |  Agent Loop / Tools |         | 可选          |
       |  Skills / MCP       |         +--------------+
       |  Session / Context  |
       +----------+----------+
                  |
       独立工作区 / 受限沙箱 / 任务专属权限
                  |
     GitLab/Gitea/GitHub · Jenkins · MCP · 项目知识
                  |
              结构化执行结果回到 Paperclip
```

**职责分界：** Paperclip 负责“给谁做、何时做、批准什么、预算多少、是否完成”；Pi 负责“拿到当前任务后如何思考、读写代码、使用工具、记录会话及提供执行证据”。

任务是否通过验收应由 Paperclip 管理的流程及外部 CI/审核证据决定，而不能仅靠 Pi 自述。

### 2.1 组件职责和禁止重复开发的能力

| 组件 | 应承担的职责 | 不应重复开发 |
|---|---|---|
| Paperclip | 业务任务、Agent 角色/配置、调度、唤醒、审批、预算、运行状态 | 不在 Pi 里再建一个任务看板和角色编排服务 |
| `pi_local` Adapter | 启动 Pi、注入任务/工作区/模型配置、续接会话、采集输出与用量 | 不在 Fork 中实现一套新的 Paperclip Adapter，除非兼容性确实证明有缺口 |
| Pi Fork | Agent Loop、模型、工具、Skills、扩展、任务内上下文、项目研发工作流 | 不把 `pi-durable` 提升为 Paperclip 上层调度器 |
| Git 与 CI | 代码提交/MR/PR、构建与测试事实 | 不以 LLM 生成的“通过”替代真实 CI 结果 |
| 项目知识库 | `AGENTS.md`、ADR、规格、工具与检索索引 | 不用会话压缩摘要代替经批准的长期知识 |

## 3. Paperclip 与 Pi 的既有集成能力

Paperclip 官方 [Pi Adapter 文档](https://docs.paperclip.ing/reference/adapters/pi/) 对 `pi_local` 说明：
- 执行 Pi CLI，并以 `provider/model` 选择模型；`model` 是必填项。
- 使用 `--session` 在 Paperclip 心跳之间恢复同一个 Agent 的 Pi 会话。
- 支持 `cwd`、`command`（默认 `pi`）、`thinking`、`promptTemplate`、`instructionsFilePath`、`env`、`timeoutSec`、`graceSec` 等配置。
- 支持 `PAPERCLIP_PI_PROVIDERS` 创建托管的 `models.json`，并通过 `PI_CODING_AGENT_DIR` 指向托管目录。
- 对远程 execution target，模型配置可同步到远程沙箱；远程运行前需要单独验证权限和工作区映射。

之前提供的 Paperclip 本地 `@paperclipai/adapter-pi-local` 代码片段还确认：
- 可注入 `PAPERCLIP_RUN_ID`、`PAPERCLIP_TASK_ID`、唤醒、审批与关联 Issue 等执行上下文；
- 适配器有 Paperclip Skills 注入及 PATH 处理；
- 存在 execution target、工作区映射、远端 managed home / bridge 的实现路径。

**验证范围说明：** 看到这些代码路径不代表远程执行已经部署成功，也不代表本 Fork 已与该 Adapter 兼容。执行模式以当前 Paperclip 实例与实测为准。

Paperclip Runner 同样有 Pi 支持，但运行生命周期、ACP/Runner 设置与直接 `pi_local` 不同。参考 [Paperclip Runner 文档](https://docs.paperclip.ing/reference/adapters/paperclip-runner/)；本项目暂不迁移。

## 4. Pi Fork 需要做什么

### 4.1 首要任务：验证 Fork 能被现有 Paperclip 调起

**目前风险不是缺少 SDK 编排，而是 Paperclip 调用的 Pi CLI 可能仍是系统安装的官方版本。**

推荐过程：

1. 在实验/开发环境从 `leth0x01/pi` 的 `develop` 分支取得源码，固定测试 commit，核对 `AGENTS.md`、`README.md`；使用 Node.js 24（源码要求 Node >=22.19）。
2. 使用仓库已有脚本安装、构建并运行 Pi：`npm ci --ignore-scripts`，依 README 选择 `npm run build` 或模型数据已就绪时的 `npm run build:offline`，运行 `./pi-test.sh` 验证命令行。
3. 检查 Paperclip **实际运行用户**的 `PATH`、`command -v pi`、`pi --version`，不能根据管理账号的环境推断服务进程使用哪个 Pi。
4. 使用 Paperclip Agent 的 `pi_local` `command` 字段（绝对路径的可执行入口或固定版本启动包装脚本）指向 Fork 构建，先用实验 Agent，不直接替换现有生产 Agent 的 `pi`。
5. 验证 `model` 配置、托管 `PI_CODING_AGENT_DIR`、`PAPERCLIP_PI_PROVIDERS`、Skills 注入、会话路径、JSONL 输出、取消、错误与超时处理。
6. 提交一个低风险测试任务，在 Paperclip 中核对 run/task/agent 标识、Pi 实际构建版本、日志、用量、运行结果和后续 heartbeat resume。

**回退方案：** 保留原 `pi_local` 的命令路径及配置，Fork 不兼容时恢复原运行时；不修改原 Paperclip 安装目录中的 `dist` 文件。

### 4.2 其次：围绕 Pi 执行器做轻量定制

优先级按从高到低：
1. **Skills/工程规范**：需求阅读、代码实现、代码审查、测试取证、PR/MR 说明等，按 Paperclip 角色注入，不将角色调度写死到 Pi。
2. **工具接入**：Pi MCP / 扩展对接受控 Git、CI Bridge、测试环境；默认限制破坏性操作。
3. **执行证据规范**：每项工作关联 Paperclip task/run ID、仓库 commit、PR/MR、Jenkins job 和测试记录。
4. **上下文与知识**：Pi session 用于任务内上下文；长期项目知识放在版本化的工程文档/知识服务，定期沉淀带来源、可审核的经验。
5. **Fork 核心改动**：只有在 Skills、Extension、CLI 参数、Adapter 配置均无法满足，并且有复现测试时，才在 `develop` 修改 Pi 核心。

### 4.3 不应在 Pi 仓库里新建的东西

- 不创建独立 `apps/agent-platform/` 来重复实现 Task API / Coordinator / Agent 状态机。
- 不重做 Paperclip 的角色、计划、任务依赖、预算、审批、提醒、心跳/唤醒和 Agent 运行面板。
- 不让 Pi 自行决定业务任务的最终审核通过或生产发布。
- 不为了“实现多 Agent”先引入 `pi-durable` 的另一层独立调度。它作为 Pi 内部高级执行实验可以单独验证，但不能替代 Paperclip 控制面。

## 5. 角色协作的推荐方式

在 **Paperclip 中**管理产品/规划、架构、开发、Reviewer、QA 等角色，并按任务状态和审批规则触发不同 Agent。Pi 只作为这些角色的一个执行 Provider。

示例（逻辑流程，不是新建状态机）：

```text
Paperclip 中创建需求和验收标准
   -> Planner Agent（Pi）输出需求拆解/方案
   -> Paperclip 审批与分配
   -> Developer Agent（Pi）操作独立 worktree，提交 MR/PR
   -> 现有 Git/CI 系统构建、测试
   -> Reviewer / QA Agent 检查代码与实际测试证据
   -> 结果与阻塞情况回写 Paperclip
   -> 人工验收/批准合并或发布
```

一项跨端业务需求可以由 Paperclip 分配 Android/iOS/服务端等 Agent；不同写入者须隔离工作区，避免代码覆盖。不要把多个 Pi 会话共用同一目录当作协同机制。

## 6. 安全和网络边界

Pi 官方 [Security](../packages/coding-agent/docs/security.md) 文档明确：Pi 默认以宿主用户权限执行，Project Trust 不是进程级安全隔离。必须在 Paperclip 远程执行目标、容器/沙箱和工作区层实施边界：

- Paperclip 控制节点与 Agent 执行节点按规划分离。现阶段可先在非敏感实验环境验证，正式部署再拆分。
- 每项写代码任务有独立的仓库工作区，按角色提供最低权限；Reviewer 不写入 Developer 的工作区。
- Agent 执行节点禁止直达生产环境、编译隔离网和管理网。CI 通过既有批准的桥接接口调用。
- 不把 Paperclip API Key、模型 Key、Git 密钥或生产凭据放进公开 Fork、AGENTS.md、日志和长期记忆。
- 工具调用需审计，Jenkins 只开放白名单任务；发布与受限操作保留审批门禁。
- 启用 Paperclip 官方远程执行隔离时，需要验证会话、工作区、技能和凭据是否正确传输以及能否在重启后恢复。

## 7. 修正后的验证清单

| 阶段 | 具体目标 | 当前状态 |
|---|---|---|
| P0（已有基础） | Paperclip -> `pi_local` -> Pi -> DeepSeek 单任务链路 | 历史记录：已跑通；当前实例未复验 |
| P0.5（下一步） | Paperclip 通过 `command` 使用 **本 Fork 的 Pi CLI**，验证版本、模型、Skills、日志与 session | 待执行 |
| P1 | Paperclip 多角色分工 -> Pi 开发 -> Git MR/PR -> CI -> Reviewer/QA -> Paperclip 状态回写 | 待端到端验收 |
| P2 | Pi 任务隔离、权限、恢复、超时/中断、成本与证据 | 待故障演练 |
| P3 | 多项目并行、受控 Jenkins/内网服务、知识体系、可观测性 | 分步接入 |

### P0.5 必须验收

- [ ] Paperclip 执行日志能证实实际调用 Fork 对应的 Pi CLI/commit；
- [ ] 模型与 `PAPERCLIP_PI_PROVIDERS` 配置正确；
- [ ] Paperclip task/run 标识传入 Pi，执行日志、usage 回传；
- [ ] Skills 能正确发现并运行，`cwd` 为隔离测试工作区；
- [ ] 任务跨心跳可恢复，终止和超时能正确回传状态；
- [ ] 未触及其他 Agent 的工作区与不应访问的资源；
- [ ] 回退为原 `pi_local` 后原执行链路仍正常。

### P1 必须验收

- [ ] 一次真实需求生成可追踪的 commit + MR/PR；
- [ ] Reviewer 与 Developer 分离，不由开发者 Agent 自我批准；
- [ ] 测试失败阻断验收；返工后可再次提交结果；
- [ ] CI 产物、日志与 Paperclip 任务可关联；
- [ ] Paperclip 展示执行、审批和最终结论，Git/CI 保有权威工件记录。

## 8. 本 Fork 的仓库策略

- `main`：跟踪上游，尽量避免加入公司专属代码。
- `develop`：经过验证的 Pi 执行器定制的集成分支。
- `docs/agent-platform-feasibility`：当前调研文档 PR，完成评审后可合并到 `develop`。
- 后续需求新开 `feature/paperclip-...`，从 `develop` 分出，提交 PR 后再合并。
- 工程定制优先以独立 `extensions/`、`skills/`、`docs/` 或项目本身的 `.pi/` 配置实现；确有必要再修改上游包。

## 9. 推荐的下一实际动作

**不是开发新的 Coordinator 或 Agent 平台 API，而是在 Paperclip 的现有 `pi_local` 上验证 Fork 可运行。**

最低交付：
1. Fork Pi 固定版本的可执行入口；
2. 一个 Paperclip 实验 Agent 指向该入口（不影响现有 Agent）；
3. 一份端到端任务的运行证据（run/task、版本、日志、模型、Skills、用量及会话恢复）；
4. 若出现兼容问题，定位是 Pi CLI 接口、Paperclip Adapter 解析、配置还是执行环境问题，再有针对性地修 Pi Fork 或提交 Paperclip upstream Issue/PR。

## 10. 参考资料

- Paperclip [Pi Adapter](https://docs.paperclip.ing/reference/adapters/pi/)
- Paperclip [Adapters Overview](https://docs.paperclip.ing/reference/adapters/overview/)
- Paperclip [Paperclip Runner](https://docs.paperclip.ing/reference/adapters/paperclip-runner/)
- Pi [README](../README.md)、[AGENTS.md](../AGENTS.md)
- Pi [SDK](../packages/coding-agent/docs/sdk.md)、[RPC](../packages/coding-agent/docs/rpc.md)
- Pi [Extensions](../packages/coding-agent/docs/extensions.md)、[MCP](../packages/coding-agent/docs/mcp.md)
- Pi [Sessions](../packages/coding-agent/docs/sessions.md)、[Security](../packages/coding-agent/docs/security.md)

> 注意：本文仅修正文档和下一步计划；没有对正在运行的 Paperclip 实例做部署更改，也没有声称 Fork Pi 已接入成功。
