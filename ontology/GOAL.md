# A2 目标：实现可复用的本体 eDSL

使用 Telora 实现 `ontology/DESIGN.md` 规定的可复用本体 eDSL，供不知道其内部
实现的企业建模者作为 package 使用。实现完整覆盖设计契约中的语义角色、建模
工厂、能力编译、路径选择与分类、规范 Plan 自动组装、覆盖保证、确定性 SQL
转换、诊断和原子发布协议。

## 稳定输入

首次工作开始前，完整阅读：

- `docs/LANG-TUTORIAL.md`；
- `docs/TELORA-CLI.md`；
- `ontology/GOAL.md`；
- `ontology/DESIGN.md`。

不得从 `ent-1/DOMAIN.md`、`ent-1/src/` 或 `ent-1/tests/` 获取企业领域信息。

## 交付物

完成并保持下列交付：

- `ontology/src/`：领域无关的可复用 Telora package 源码；
- `ontology/src/bin/main.telora`：展示完整成功路径的可执行入口；
- `ontology/src/bin/verify.telora`：实际执行请求覆盖、关系选择和转换确定性检查，
  成功时正常退出的行为验证入口；
- `ontology/src/bin/invalid.telora`：展示无法满足契约时使用 `fail!`，必须以非零状态
  退出并产生正确的 Host 诊断，不得产生 output；
- `ontology/tests/ontology.telora`：固定关键类型与模块契约的开发期检查入口；
- `ontology/DSL-TUTORIAL.md`：企业作者仅凭本文即可使用公共 API 的教程；
- `ontology/AI3-CONTRACT.md`：列出企业模型必须提供的有类型输入、公共 API、
  QueryIntent 参考形状、eDSL 保证以及验证入口；
- `ontology/NOTES.md`：记录 API 选择、权衡、验证结果和已知限制。

不得修改 `ontology/telora-deps.json`。不得把企业实体、标识、表、列、具体 SQL、公式
或物理映射实例写入可复用 package。

成功入口必须实际展示：结构化领域知识经 factory 得到 model；同一个查询意图经
model 得到规范 Plan；该 Plan 经公共 transform 得到确定 SQL；组合门面
`make_query_creator(model)` 得到相同 SQL。`verify` 必须实际执行 Plan 的请求覆盖、
关系选择覆盖和转换确定性检查。`invalid` 必须证明失败时没有部分 Plan/SQL。公共 API
不得为了验证而返回 `Rejection`、诊断数组或发布状态；不要求 eDSL 为单次收集多个
诊断实现 Evidence、恢复状态机或诊断调度器。

## 验证

实际运行并根据结果修正实现：

```text
./bin/telora run main -C ontology
./bin/telora run verify -C ontology
# 预期非零；遇到问题时用诊断模式检查 Host 诊断且不得出现 output
./bin/telora run invalid -C ontology --best-effort
# 只作开发期分析，不代替上述 run
./bin/telora check @test/ontology.telora -C ontology
./bin/telora show @bin/main.telora -C ontology
```

完成时报告实际交付物、真实验证结果和仍存在的限制，不要求 Git commit。

## 反馈修订

被恢复且 `ent-1/FEEDBACK.md` 已更新时，阅读具体建模反馈，自行判断是否接受，
修订公共实现与文档并重新运行验证。在 `ontology/NOTES.md` 中记录接受或拒绝的
反馈及理由。不得读取 A3 的私有题面、源码或 notes。
