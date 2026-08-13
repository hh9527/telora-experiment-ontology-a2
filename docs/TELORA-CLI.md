# Telora 验证流程

每个 Telora crate 的可复用模块位于 `src/`，应用入口位于 `src/bin/`，测试入口位于
`tests/`。`telora-deps.json` 固定该 crate 的依赖边界，不得修改。

Telora 从当前目录向上查找最近的 `telora-deps.json`，因此命令可以从目标 crate 根
目录或其任意子目录执行。`-C` 可以显式改变查找的起始目录，该目录不必是 crate 根。
命令参数使用稳定逻辑模块 ID，不使用物理文件名：

```text
./bin/telora run main -C ontology
./bin/telora check @test/ontology.telora -C ontology
./bin/telora show @bin/main.telora -C ontology
./bin/telora show @src/ontology.telora -C ontology -k type,let,def,import
./bin/telora show @src/ontology.telora -C ontology --exports
./bin/telora show @src/ontology.telora -C ontology --at 12:4
./bin/telora run -S path/to/file.telora
```

企业 crate 同理，在 `ent-1/` 下运行 `run main`，并以
`check @test/logistics.telora` 检查测试入口。

- `run name` 固定执行 `@bin/name.telora` 并打印其 export `output`；`name` 是不含路径
  分隔符和 `.telora` 后缀的单个 stem。
- `run name -C context` 从 `context` 开始向上发现 manifest。
- `run -S file` 进入 standalone 模式，不发现 manifest，只使用根文件内的
  `crate.dependency` / `crate.format` options；这些 options 相对文件父目录解析。
  `-S` 与 binary name、`-C` 互斥。
- `run`、`check` 和 `show` 的 `-C context` 都从 `context` 开始向上发现 manifest；
  `check` 和 `show` 接受完整稳定模块 ID，`check @test/...` 检查测试入口，`run` 只
  接受 binary name。
- `show` 输出 `telora.show/v1` JSONL 语义记录；默认列出选中模块的顶层 local
  definitions。
- `-p` 按名称的大小写敏感字面子串过滤，不是 glob 或正则。
- `-k` 接受逗号分隔的 `type,let,def,import`；`--exports` 改查公共接口并与 `-k`
  互斥。
- `--at line[:column]` 使用从一开始计数的位置：行号选择与该行相交的事实，行列选择
  覆盖该点的事实；它与 `-p`、`-k`、`--exports` 互斥。

程序中的 `dbg!(expr)` 和 `expr.dbg!()` 把旁路观察写入 stderr，不改变 stdout 的
`output`。每个事件是一行紧凑 JSON：

```json
{"name":"var","repr":"3","module":"@bin/main.telora","line":12}
{"name":"plan","repr":"{...}","module":"@bin/main.telora","line":13,"message":"generated"}
```

固定字段为 `name`、`repr`、`module`、`line`；只有显式 message 时才有 `message`。
`repr` 是有界 debug 表示，不是可反序列化的 JSON 值。Host 是否输出或丢弃事件对
Telora 程序不可感知。Float 的 `repr` 使用 Debug 表示，例如 `3.0` 和 `-0.0`；它与
字符串插值和 `fmt.display` 使用的 Display 表示不同。

命令退出码为零表示请求成功；非零表示 CLI 或 Telora 拒绝。`show` 的空匹配成功且
没有输出。记录中的 `authority` 区分 `authoritative`、`recovery` 与 `debug` 事实。
表达式级记录属于 `debug`；错误恢复记录的 authority 服从其事实和模块状态。
