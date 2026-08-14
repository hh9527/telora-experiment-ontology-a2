# 本体 eDSL 原生多 Agent 实验

本仓库是一个可直接 clone 的 opencode 实验计划，在 Telora 主仓库中以 submodule
固定 revision。它用 opencode 原生 coordinator/subagent/session-resume 能力运行
一次确定的 A2-A3 交付，并由 Host 显式驱动反馈修订。整个实验只有一个共享 Git workspace、一个 coordinator session、
一个 TUI 和一个 daemon；A2、A3 是 coordinator 创建并恢复的原生子会话。

实验检验的核心链条是：A2 创建领域无关的 `ModellingFactory`、规范 Plan IR 与
确定性 SQL transform；A3 只声明企业领域知识并实例化 Model；查询意图经 Model
自动得到 Plan，再由 Plan 得到 SQL。A3 不手工组装 Plan，也不绕过 Plan 拼 SQL。

## 工作区

```text
docs/
  LANG-TUTORIAL.md       # A2/A3 公共语言教程
  TELORA-CLI.md          # A2/A3 公共 CLI 教程
bin/
  telora                 # Host 注入的固定二进制
ontology/
  GOAL.md                # A2 的稳定任务定义
  DESIGN.md              # 仅 A2 可见的设计契约
  DSL-TUTORIAL.md        # A2 交付给企业作者的公开教程
  AI3-CONTRACT.md        # A2 交付给 A3 的公开契约
  NOTES.md               # A2 私有过程记录
  telora-deps.json
  src/
    bin/
  tests/
ent-1/
  GOAL.md                # A3 的稳定任务定义
  DOMAIN.md              # 仅 A3 可见的企业事实
  FEEDBACK.md            # A3 给 A2 的公开反馈
  NOTES.md               # A3 私有过程记录
  telora-deps.json
  src/
    bin/
  tests/
.opencode/agents/
  coordinator.md         # 协调角色元数据与状态转换协议
  a2.md                  # A2 角色元数据与提示词
  a3.md                  # A3 角色元数据与提示词
```

共享目录不代表共享权限。A3 看不到 `ontology/GOAL.md`、`ontology/DESIGN.md`、源码、验证入口和
`NOTES.md`；A2 看不到 `ent-1/DOMAIN.md`、源码和 notes，只能在修订轮读取
`ent-1/FEEDBACK.md`。coordinator 不写文件、不运行 Telora，只能调用 A2/A3 并
读取少量公开交付和进度材料。coordinator 不是任务解释层：它的角色定义是一组
“可观察状态 -> 唯一动作”的转换规则；它不知道领域任务的含义，向子代理发送的
固定提示只要求其遵循自身 agent prompt 与可见文件，不得转述、补充或改写任务
定义。

A2/A3 的角色协议都包含 Telora 自学规则。首次启动时，A2 实现 eDSL，A3 同时只
学习公共语言/CLI 教程并分析企业题面；A2 公开交付就绪后，A3 才读取 eDSL 教程
和契约并开始建模。正式探索可在各自 `src/bin/` 下创建入口，通过
`telora run/show` 验证；探索权限不改变角色间的文件可见性边界。

所有角色协议均明确列出：收到的白名单指令或发生的状态、判断依据、唯一动作和
完成标准。Host 只向 coordinator 发送 `请开始实验。` 或 `恢复执行。`；
coordinator 只向 A2/A3 发送各自角色文件列出的固定指令。白名单之外的文本不触发
工作流动作。

`.opencode/agents/*.md` 是 OpenCode 原生的角色配置：frontmatter 保存角色元数据
和权限，正文保存角色提示词。`opencode.json` 只选择默认 coordinator。
`experiment.json` 是 Host 使用的
artifact、固定启动提示词、OpenCode daemon 环境、validation 与归档范围的 SSOT。
本计划把 `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX` 固定为 `128000`，避免 A2 的
单次深度推理被 OpenCode 默认的 32000 token 上限截断。该值由 `oc-run` 在启动
握手 daemon 和 TUI daemon 时注入，不需要在外部终端手工设置。每个 agent 都通过
read/glob/grep/edit 规则显式拒绝 `experiment.json`，因此文件保留在 clone 中但不
对 agent 可见。这些权限在 plan 中确定后即封闭；控制器不解析或安全审查角色
配置，只驱动会话、观察和解读状态并归档证据。

`oc-run` 只接受干净且已提交的 plan worktree，记录其 commit 和 origin，随后直接
clone 到临时 workspace。clone 天然具有正确的 Git worktree 根，opencode 可按
workspace-relative 路径执行权限判断。运行中的 agent 改动不会污染 submodule。
构建完成的 Telora binary 也会复制进 workspace 并记录摘要；实验启动后主仓库的
重新构建不会改变该次实验使用的语言版本。

## 运行

在外部终端准备并进入 TUI：

```bash
./oc-run ontology-edsl t001 --port 4196
```

TUI 准备好后，在控制会话启动 coordinator：

```bash
./oc-ctl start t001
```

coordinator 按固定首次流程执行：并行启动 A2 首次实现和 A3 语言/领域准备，等待
A2 公开交付后恢复原 A3 完成建模与原始反馈，然后无条件挂起。它不会自动把原始
反馈交给 A2，也不判断任务是否收敛。
观察命令不会中断实验：

```bash
./oc-ctl status t001
./oc-ctl recent t001 3
./oc-ctl children t001
./oc-ctl child-recent t001 <session-id> 3
./oc-ctl files t001
```

挂起后，由 Host 审计 `ent-1/FEEDBACK.md`：筛除应由 Telora issue 跟踪、A2 无法
修复的语言/机制问题，也可以在设计固化前增补待验证观点或临时缓解要求。Host
把文件更新为一个明确批准的反馈批次后，执行 `./oc-ctl continue t001`。一次恢复
只允许原 A2 处理这一批反馈、原 A3 复验一次，随后无条件再次挂起；不会自动开始
下一轮。反馈未变化时，恢复不会调用子代理。不要把一次 HTTP connection refusal
判断为 daemon 死亡；客户端会对这种繁忙期瞬态失败重试。

实验完成后：

```bash
./oc-ctl validate t001
./oc-ctl finish t001
```

自动 validation 使用普通严格 `run` 执行 A2/A3 提供的 `main` 与 `verify` binary。
`check` 只用于开发期分析，不作为最终行为验收。当前 validation runner 只接受成功
命令，因此预期失败的 `invalid` binary 不进入自动 required validation；A2/A3 和 Host
使用 `run --best-effort` 单独执行它，并按非零退出、完整诊断、无 output 和无 Entry
effect 四项观察。

`finish` 归档 coordinator 和所有直接子会话、子会话 messages、工作区以及 Host
验证结果。冻结后退出 TUI，临时目录由 `oc-run` 精确清理。
