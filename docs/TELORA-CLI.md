# Telora 验证流程

每个 Telora crate 的可复用模块位于 `src/`，应用入口位于 `src/bin/`，测试入口位于
`tests/`。`telora-deps.json` 固定该 crate 的依赖边界，不得修改。

命令必须从目标 crate 根目录执行。Telora 从当前目录向上查找最近的
`telora-deps.json`，命令参数使用稳定逻辑模块 ID，不使用物理文件名：

```text
cd ontology
../bin/telora run main
../bin/telora check @test/ontology.telora
../bin/telora show @bin/main.telora
../bin/telora show @src/ontology.telora -k type,let,def,import
../bin/telora show @src/ontology.telora --exports
../bin/telora show @src/ontology.telora --at 12:4
```

企业 crate 同理，在 `ent-1/` 下运行 `run main`，并以
`check @test/logistics.telora` 检查测试入口。

- `run name` 固定执行 `@bin/name.telora` 并打印其 export `output`。
- `show` 输出 JSONL 语义记录；默认列出选中模块的顶层 local definitions。
- `-p` 按名称的大小写敏感字面子串过滤，不是 glob 或正则。
- `-k` 按 `type,let,def,import` 过滤；`--exports` 改查公共接口。
- `--at line[:column]` 独立查询与位置相交的语义事实。

命令退出码为零表示请求成功；非零表示 CLI 或 Telora 拒绝。`show` 的空匹配成功且
没有输出。记录中的 `authority` 区分 `authoritative`、`recovery` 与 `debug` 事实。
