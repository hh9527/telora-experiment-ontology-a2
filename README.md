# 本体 eDSL 原生多 Agent 实验

本仓库是一个可直接 clone 的 opencode 实验计划，在 Telora 主仓库中以 submodule
固定 revision。它用 opencode 原生 coordinator/subagent/session-resume 能力运行
A2-A3 反馈闭环。整个实验只有一个共享 Git workspace、一个 coordinator session、
一个 TUI 和一个 daemon；A2、A3 是 coordinator 创建并恢复的原生子会话。

## 工作区

```text
docs/
  LANG-TUTORIAL.md       # A2/A3 公共语言教程
  TELORA-CLI.md          # A2/A3 公共 CLI 教程
bin/
  telora                 # Host 注入的固定二进制
ontology/
  DESIGN.md              # 仅 A2 可见的设计契约
  DSL-TUTORIAL.md        # A2 交付给企业作者的公开教程
  AI3-CONTRACT.md        # A2 交付给 A3 的公开契约
  NOTES.md               # A2 私有过程记录
  telora-deps.json
  src/
  bin-src/
ent-1/
  DOMAIN.md              # 仅 A3 可见的企业事实
  FEEDBACK.md            # A3 给 A2 的公开反馈
  NOTES.md               # A3 私有过程记录
  telora-deps.json
  src/
  bin-src/
```

共享目录不代表共享权限。A3 看不到 `ontology/DESIGN.md`、源码、验证入口和
`NOTES.md`；A2 看不到 `ent-1/DOMAIN.md`、源码和 notes，只能在修订轮读取
`ent-1/FEEDBACK.md`。coordinator 不写文件、不运行 Telora，只能调用 A2/A3 并
读取少量公开交付和进度材料。coordinator 不是任务解释层：它不知道领域任务的
含义，向子代理发送的固定提示只要求其遵循自身 agent prompt 与可见文件，不得
转述、补充或改写任务定义。

`opencode.json` 是 agent prompt 和权限的 SSOT；`experiment.json` 是 Host 使用的
artifact、固定启动提示词、OpenCode daemon 环境、validation 与归档范围的 SSOT。
本计划把 `OPENCODE_EXPERIMENTAL_OUTPUT_TOKEN_MAX` 固定为 `128000`，避免 A2 的
单次深度推理被 OpenCode 默认的 32000 token 上限截断。该值由 `oc-run` 在启动
握手 daemon 和 TUI daemon 时注入，不需要在外部终端手工设置。每个 agent 都通过
read/glob/grep/edit 规则显式拒绝 `experiment.json`，因此文件保留在 clone 中但不
对 agent 可见。

`oc-run` 只接受干净且已提交的 plan worktree，记录其 commit 和 origin，随后直接
clone 到临时 workspace。clone 天然具有正确的 Git worktree 根，opencode 可按
workspace-relative 路径执行权限判断。运行中的 agent 改动不会污染 submodule。

## 运行

在外部终端准备并进入 TUI：

```bash
./oc-run ontology-edsl t001 --port 4196
```

TUI 准备好后，在控制会话启动 coordinator：

```bash
./oc-ctl start t001
```

coordinator 按固定流程执行：A2 创建、A3 建模并反馈、恢复原 A2 子会话修订、恢复
原 A3 子会话复验。观察命令不会中断实验：

```bash
./oc-ctl status t001
./oc-ctl recent t001 3
./oc-ctl children t001
./oc-ctl child-recent t001 <session-id> 3
./oc-ctl files t001
```

如果 coordinator 因上下文长度结束，使用 `./oc-ctl continue t001`。该命令要求它
恢复需要继续的既有子会话。不要把一次 HTTP connection refusal 判断为 daemon
死亡；客户端会对这种繁忙期瞬态失败重试。

实验完成后：

```bash
./oc-ctl validate t001
./oc-ctl finish t001
```

`finish` 归档 coordinator 和所有直接子会话、子会话 messages、工作区以及 Host
验证结果。冻结后退出 TUI，临时目录由 `oc-run` 精确清理。
