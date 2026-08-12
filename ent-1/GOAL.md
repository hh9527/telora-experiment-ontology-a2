# A3 目标：使用本体 eDSL 建立企业模型

作为只接触公共交付的企业建模者，使用 `ontology` package 表达
`ent-1/DOMAIN.md` 中的物流履约分析模型。不得查看或依赖 eDSL 的私有设计、源码
和验证入口，也不得在企业模型中重新实现 eDSL 负责的共享规则。

## 稳定输入

首次工作开始前，完整阅读：

- `docs/LANG-TUTORIAL.md`；
- `docs/TELORA-CLI.md`；
- `ent-1/GOAL.md`；
- `ent-1/DOMAIN.md`；
- `ontology/DSL-TUTORIAL.md`；
- `ontology/AI3-CONTRACT.md`。

不得读取 `ontology/GOAL.md`、`ontology/DESIGN.md`、`ontology/NOTES.md`、
`ontology/src/` 或 `ontology/bin-src/`。

## 交付物

完成并保持下列交付：

- `ent-1/src/`：企业拥有的封闭 vocabulary、能力、关系、物理映射和构建策略；
- `ent-1/bin-src/main.telora`：执行合法场景并得到完整发布计划；
- `ent-1/bin-src/test.telora`：执行非法场景并同时保留规定的拒绝证据；
- `ent-1/FEEDBACK.md`：只记录实际使用公共 eDSL 时观察到的具体摩擦与缺口；
- `ent-1/NOTES.md`：记录企业模型选择、验证结果和剩余风险。

不得修改 `ent-1/telora-deps.json`。不得通过复制共享算法、String 标识、`Any`、
`Dyn` 或绕过公共 API 来完成题面。

## 验证

实际运行并根据结果修正企业模型：

```text
./bin/telora run ent-1/bin-src/main.telora
./bin/telora run ent-1/bin-src/test.telora
./bin/telora types ent-1/bin-src/main.telora
./bin/telora show ent-1/bin-src/main.telora
```

完成时报告实际交付物、真实验证结果和具体反馈，不要求 Git commit。

## 上游更新复验

被恢复且上游公开交付已更新时，保持同一个企业题面与模型，重新运行验证并更新
`ent-1/FEEDBACK.md` 和 `ent-1/NOTES.md`。反馈必须来自复验观察，不得推测上游
私有实现。
