# A2 目标：实现可复用的本体 eDSL

使用 Telora 实现 `ontology/DESIGN.md` 规定的可复用本体 eDSL，供不知道其内部
实现的企业建模者作为 package 使用。实现完整覆盖设计契约中的语义角色、能力
编译、路径选择与分类、诊断、构建器传输和原子发布协议。

## 稳定输入

首次工作开始前，完整阅读：

- `docs/LANG-TUTORIAL.md`；
- `docs/TELORA-CLI.md`；
- `ontology/GOAL.md`；
- `ontology/DESIGN.md`。

不得从 `ent-1/DOMAIN.md`、`ent-1/src/` 或 `ent-1/bin-src/` 获取企业领域信息。

## 交付物

完成并保持下列交付：

- `ontology/src/`：领域无关的可复用 Telora package 源码；
- `ontology/bin-src/main.telora`：展示完整成功路径的可执行入口；
- `ontology/bin-src/test.telora`：覆盖关键拒绝路径和确定性规则的验证入口；
- `ontology/DSL-TUTORIAL.md`：企业作者仅凭本文即可使用公共 API 的教程；
- `ontology/AI3-CONTRACT.md`：列出企业模型必须提供的有类型输入、公共 API、
  eDSL 保证以及验证入口；
- `ontology/NOTES.md`：记录 API 选择、权衡、验证结果和已知限制。

不得修改 `ontology/telora-deps.json`。不得把企业实体、标识、表、列、SQL、公式
或物理映射实例写入可复用 package。

## 验证

实际运行并根据结果修正实现：

```text
./bin/telora run ontology/bin-src/main.telora
./bin/telora run ontology/bin-src/test.telora
./bin/telora types ontology/bin-src/main.telora
./bin/telora show ontology/bin-src/main.telora
```

完成时报告实际交付物、真实验证结果和仍存在的限制，不要求 Git commit。

## 反馈修订

被恢复且 `ent-1/FEEDBACK.md` 已更新时，阅读具体建模反馈，自行判断是否接受，
修订公共实现与文档并重新运行验证。在 `ontology/NOTES.md` 中记录接受或拒绝的
反馈及理由。不得读取 A3 的私有题面、源码或 notes。
