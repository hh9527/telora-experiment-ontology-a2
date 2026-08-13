# 面向本体 eDSL 作者的 Telora 教程

这是本体 eDSL 实验使用的有界语言输入，不是完整的语言规范。某项行为未在
本文档中说明时，不要发明 Host 能力，也不要绕过有类型的 Telora 模型。

Telora 是一门确定、纯、面向表达式的语言，用于把意图编译为不可变计划。
值不可变，模块显式声明，TypeMetadata 是普通数据，并由执行程序代码的同一个
VM 求值。

## 值与绑定

```telora
42                         # Int
3.5                        # Float
1e6                        # 带十进制指数的 Float
"text"                     # String
b"bytes"                   # Bytes
'Ready                     # Atom
'True                      # Bool 是封闭的 Atom 类型
'Some(1)                   # 带标签的值
(1, "one")                # Tuple 值
[1, 2, 3]                  # Array
{name: "Ada", active: 'True} # record/Dict 值
```

Bool 值是 `'True` 和 `'False`；Telora 不进行 truthiness 转换。Float 是有限的
IEEE 754 binary64。Float 字面量接受小数点形式（`3.5`）和指数形式（`1e6`、
`1.25e-3`）。NaN、正无穷和负无穷都不是 Telora 值。

```telora
let answer = 40 + 2;
def increment = fn(value) { value + 1 };
```

`let`、`def`、`type`、`for`、`fn`、`match`、`native`、`decl`、`import` 和
`export` 是语言保留字。`_` 用作模式或显式类型实参占位符；闭包参数使用具名
标识符。

## 运算符与控制流

比较运算符为：

```telora
left == right
left != right
left < right
left > right
left <= right
left >= right
```

对于普通复合值，相等和不等运算保持结构相等语义。有序比较只接受类型相同的
`Int`、`Float` 或 `String` 操作数；不存在混合数值强制转换。String 按其内部
UTF-8 字节序列精确地进行字典序比较，不做规范化，不使用 locale 规则、大小写
折叠或自然数排序。

`%` 接受类型相同的 Int 或 Float 操作数，与 `*` 和 `/` 具有相同优先级和左
结合性，并使用截断余数语义。非零结果与左操作数同号：`-7 % 3 == -1`，且
`7 % -3 == 1`。Int `% 0` 以 `DivisionByZero` 失败。

Float 的 `+`、`-`、`*`、`/` 和 `%` 必须产生有限 Float。产生 NaN 或无穷时，
抛出等价于 `fail!("NonFiniteFloat", left, right)` 的带来源 blame。这包括 Float
除以正零或负零、Float 对任一零求余，以及算术溢出。操作数按源码顺序各求值
一次。有限 Float 比较遵循普通数值语义，并且 `-0.0 == 0.0`。

前缀 `!` 对 Bool 返回相反的规范 Bool，对 Int 返回按位补。二元 `&`、`^` 和
`|` 只接受 Int。从紧到松的优先级依次为：一元运算、`*`/`/`/`%`、`+`/`-`、
`&`、`^`、`|`、比较、`&&`、`||`。

六种比较共享一个不可结合的优先级层。需要比较某个比较结果时必须写括号；
`a < b <= c` 不是链式比较。`&&` 和 `||` 接受 Bool，并执行短路求值。
`if` 表达式始终具有 `else` 分支。`ctrl_block` 可以是普通 block、`if`、
`if let`、`match` 或 `return expression;`。`if` 和 `if let` 的 `else` 接受
`ctrl_block`；非 block 形式会被规范化为包含该控制流表达式的 block。因此可以
连续写 `else if`：

```telora
if score >= 90 { 'Excellent }
else if score >= 60 { 'Pass }
else { 'Fail }
```

```telora
if ready { value }
else if let 'Some(cached) = candidate { cached }
else match fallback { 'Some(value) => value, 'None => default }
```

也可以直接把提前返回用作分支：

```telora
if ready { value } else return fallback;
```

## 函数与契约

```telora
fn(value) { value + 1 }

def identity: for(A) Fn(A) -> A = fn(value) { value };
def map_pair: Fn(Int, String) -> Tuple([String, Int]) =
    fn(number, text) { (text, number) };
```

函数类型写作 `Fn(P1, ..., Pn) -> R`。普通 TypeMetadata 构造器调用写作
`Func([P1, ..., Pn], R)`：

```telora
type Unary = Func([Int], String);
```

`Fn(Int) -> String` 与 `Func([Int], String)` 产生相同的规范函数元数据。

泛型调用默认推断类型实参，也可以使用显式的 `@[...]` 应用：

```telora
identity@[Int](1)
pair@[Int, _](1, "text")
```

`_` 表示由完整调用上下文推断该类型实参。没有标记的 `value[index]` 只表示
Array 索引。

推断会综合完整泛型调用中的证据。当另一个实参能够确定外围 enum 时，单独一个
封闭 Atom 实参不会过早地把共享参数固定为其 singleton 类型。例如，`'Base`
实参和 `Array(NodeId)` 实参可以共同推断出 `NodeId`。只有完整调用仍确实存在
歧义或约束不足时，才使用显式 `@[...]`。

匿名 Struct 实参同样参与完整调用上下文。泛型回调结果可以拓宽较早 seed 中的
singleton Atom 字段和空集合字段。因此，下列 fold 会直接推断出 `flag: Bool`
和 `items: Array(Int)`：

```telora
array.fold([1, 2, 3], {flag: 'False, items: []}, fn(state, item) {
    {flag: item > 1 || state.flag, items: array.push(state.items, item)}
})
```

当回调分支返回多个结构对应的 Struct variant 时，它们的 union 会保留字段间
关系。seed 会根据唯一兼容的 variant 完成推断；无关 Atom 或存在歧义的完成方式
仍然报错。

由闭包初始化且没有标注的局部绑定，可以从后续泛型调用中获得预期函数类型。
闭包分支产生的 variant union 在每个 variant 和 payload 均兼容时，会细化为
预期的封闭 enum。例如，`'None | 'Some(String)` 会细化为 `Option(String)`。
未知 variant 和不兼容 payload 仍然报错。

泛型函数体内的局部标注不能引用外围模块级 `for` 契约引入的类型参数。应当把
完整泛型契约放在模块级辅助函数上，不得使用 `Any` 擦除类型来绕过这一边界。

Telora 在拓宽结果之前合并分支证据：泛型代码中的 `if` 若为同一个预期 enum
结果贡献不同的窄 variant，会先 join 它们，再拓宽为该 enum。例如，有类型的
`Array(Option(Output))` fold 可以在一个分支 push `'None`，在另一个分支 push
`'Some(output)`。当回调仍然约束不足时，带有完整契约的具名辅助函数依然有用。

## Struct、enum 与模式

```telora
@enum type Entity = {
    Ticket: 'None,
    Agent: 'None,
};

@struct type Requirement = {
    target: Entity,
    reason: String,
};
```

Struct 和 enum 都是封闭的。字段使用 `.field`；enum 值使用 Atom 或 Tagged
语法。

```telora
match result {
    'Some(value) => value,
    'None => fallback,
}

match pair {
    (left, right) => left,
}
```

对已知封闭 variant 的 match 必须穷尽，或者包含 catch-all。`_` 是通配模式。

## Array

显式导入标准 Array 模块：

```telora
import "std/array" as array;
```

本实验主要使用以下操作：

```telora
array.length(values)                 # Int
values[index]                        # A；以 OutOfRange blame 失败
array.get(values, index)             # Option(A)
array.enumerate(values)              # Array(Tuple([Int, A]))
array.find(values, predicate)        # Option(A)
array.any(values, predicate)         # Bool
array.all(values, predicate)         # Bool
array.map(values, mapper)            # Array(B)
array.flat_map(values, mapper)       # Array(B)
array.filter(values, predicate)      # Array(A)
array.fold(values, initial, folder)  # State
array.concat([left, right])          # Array(A)
array.push(values, item)             # 新的 Array(A)
```

Array 保留顺序。`find` 返回第一个匹配项，`filter` 保留输入顺序，fold 从左到右
处理各项。这些顺序属性可以成为确定性 eDSL 契约的一部分。

在结构对应的 `if`、`if let` 或 `match` 结果中，含有具体 Array 或 Dict 元素的
分支会向空分支提供元素类型证据，该行为与分支顺序无关。如果每个可达分支都为
空，且不存在预期元素类型，应添加显式标注，例如
`let none: Array(Item) = [];`。

两种索引形式都使用从零开始的 Int 索引。`values[index]` 直接返回元素；索引为
负数或越界时，以 `fail!("OutOfRange", values, index)` 失败。对于同样的缺失
位置，`array.get` 返回 `'None`。`enumerate` 在保留来源顺序和重复项的同时，
把每一项与其从零开始的 Int 索引配对：

```telora
array.get(["a", "b"], 1)       # 'Some("b")
["a", "b"][1]                 # "b"
array.enumerate(["a", "b"])   # [(0, "a"), (1, "b")]
```

缺失属于预期控制流时使用 `array.get`；缺失违反不变量时使用直接索引。算法需要
保留位置时使用 `array.enumerate(values)`。

图或集合等任何必要的有界结构，都使用有类型的不可变值和普通库函数构建。

## Tuple 元数据

Tuple 值和 Tuple TypeMetadata 是普通值的两种不同用途：

```telora
let pair: Tuple([Int, String]) = (1, "one");
let number: Int = pair.0;
type Pair = Tuple([Int, String]);
```

`Tuple` 恰好接收一个实参，即 TypeMetadata 的 Array：`Tuple([A, B])`。Tuple
值使用非负整数字面量投影，例如 `pair.0`。投影是可组合的后缀操作：
`value.1.0` 表示 `(value.1).0`，并且可以与字段选择、索引和调用组合。已知的
越界位置属于分析错误。`Fn(A) -> Array(Tuple([B, C]))` 等嵌套形式合法。

## TypeMetadata family

类型是一等元数据值。`Type` 是任意有效 TypeMetadata 的类型；`TypeOf(A)` 是
描述 `A` 的元数据的精确证据。

参数化声明定义可复用的 TypeMetadata family：

```telora
@struct
type Capability(Id, Input, Output) = {
    id: Id,
    lower: Fn(Id, Input) -> Option(Output),
};

type TicketCapability = Capability(TicketId, Request, TicketPlan);
```

family 在值位置也是普通的有类型元数据能力。family 必须接收全部参数，其阶数
为 rank-1，并且不能是 higher-kinded。无环的 family 可以引用同一模块中的具体
类型或另一个 family，且不受声明顺序影响：

```telora
@struct type Result(Value) = {outcome: Outcome, value: Option(Value)};
@enum type Outcome = {Published: 'None, Rejected: 'None};
```

包含 family 的环、递归的具体依赖，以及对普通局部辅助项的依赖仍然非法。
标识、payload、映射和计划使用模型提供的具体类型。不得用 `Any`、`Dyn` 或
String 标识替代未知关系。

## JSON、codec 与 schema

JSON 值是 Telora 的普通不可变值。`std/json` 负责 JSON 文本和 schema，
`std/codec` 根据 TypeMetadata 在 JSON 值与有类型值之间转换：

```telora
import "std/codec" as codec;
import "std/json" as json;
import "std/result" as result;

@struct
type Query = {
    subject: String,
    limit: Int,
};

let raw = json.parse("{\"subject\":\"orders\",\"limit\":20}")
    |> result.unwrap;
let query: Query = codec.decode(Query, raw) |> result.unwrap;
let encoded = codec.encode(Query, query) |> result.unwrap;
let compact: String = json.stringify(encoded);
let pretty: String = encoded |> json.stringify_pretty(2);
let query_schema = json.schema(Query);
```

也可以用 `json.decode(Query, text)` 直接把 JSON 文本解码成 `Query`。两条路径的
区别是边界位置：`json.parse` 只解析文本并返回 JSON 值；`codec.decode` 对已经
存在的 JSON 值施加类型契约。`codec.encode` 返回 JSON 值，只有需要文本边界时
才调用 `json.stringify` 或 `json.stringify_pretty`。

上述 parse、decode 和 encode 都返回 `Result(..., BlameError)`。只有输入无效确实
违反当前程序不变量时才使用 `result.unwrap`；可预期的外部输入错误应通过 `match`
保留为显式拒绝或诊断。Codec 失败不会发布部分解码值。

Struct 和 enum 默认从同一份 TypeMetadata 派生 codec 与 JSON schema。常用的
`std/json` decorator 可以显式调整其 JSON 表示：

```telora
@json.rename_all('CamelCase)
@struct
type Details = {
    order_id: String,
    @json.default('None)
    note: Option(String),
};

@struct
type Envelope = {
    kind: String,
    @json.flatten details: Details,
};

@json.untagged
@enum
type Scalar = {
    Text: String,
    Count: Int,
};
```

此外可以用 `@json.rename("name")` 改写单个字段名，用
`@json.skip_serializing_if('None)`、`@json.skip_serializing_if('Empty)` 或一个返回
Bool 的函数省略满足条件的字段。Decorator 是产生 attribute 的普通元数据函数；
codec 和 schema 读取相同 attribute，因此二者不会形成两套独立模型。

JSON/TOML/YAML 文件也可以作为静态数据模块 import。它们在封闭模块图建立时由
Host 加载，不是运行时文件 IO；JSON 解析严格拒绝重复 key，并保留字段来源。
不要为了打印中间值而手写 `*_desc` 函数：公开结果需要稳定 JSON 形状时使用
codec，需要临时观察任意局部值时使用 `dbg!`：

```telora
let plan = dbg!(make_plan(model, request));
let checked = plan.dbg!("before lowering");
```

`dbg!` 返回原值并保留其精确类型，因此可以直接插入表达式或管道。后置写法是通用
contextual intrinsic 糖：`value.dbg!("message")` 等价于
`dbg!(value, "message")`。message 必须是 String literal。

`telora run` 把观察写到 stderr，每行是一个 JSON object：

```json
{"name":"plan","repr":"{...}","module":"@src/query.telora","line":42,"message":"before lowering"}
```

`name` 是被观察表达式的源码文本，`repr` 是有界、确定且能处理 cycle 的临时表示。
它不是 JSON 编码契约。稳定结构化边界使用 codec，长期面向人的领域摘要应显式建模。
`dbg!` 不捕获其他局部变量，也不应观察敏感值或作为生产日志接口。

## 当前实现限制与缓解方法

本节描述当前实现中已经通过实验确认的边界。遇到这些边界时，应使用给出的
有类型写法，不要用 `Any`、`Dyn` 或 String 标识绕过。

### 多元素能力目录的类型推断

多个能力记录直接写进同一个 Array 时，各元素的 singleton Atom 标识、闭包类型
以及 `'Some`/`'None` 窄 variant 可能无法自动收敛到同一个 family 实例。典型表现
是错误中出现很大的 variant union，或者数组元素类型彼此不一致。

为每个构建函数声明完整的具体返回契约，再把构建结果放入数组：

```telora
type DimDef = DimensionDefinition(DimId, DimOutput, Entity, DimInput, Expr);

def order_month: Fn() -> DimDef = fn() {
    {
        id: 'OrderMonth,
        capability: { /* ... */ },
        formula: { /* ... */ },
    }
};

let dimensions: Array(DimDef) = [order_month(), customer_tier()];
```

同样的原则适用于 `QueryIntent` 等高阶 family 的记录字面量：当外围 family 无法
从局部 singleton 值唯一确定时，给完整记录或具名构建函数添加 concrete family
契约。巨大 union 错误应首先检查是否缺少这个公共期望类型。

### enum payload 不能是匿名 Struct 类型

`@enum` 声明中的 variant payload 必须引用具名类型。匿名 Struct payload 会报
`variants.X.kind must be an Atom`：

```telora
# 不支持：Column 的 payload 是匿名 Struct 类型
# @enum type Expr = { Column: {alias: String, column: String} };

@struct type ColumnRef = {alias: String, column: String};
@enum type Expr = {Column: ColumnRef};

# 值位置的匿名记录仍然合法
let expr: Expr = 'Column({alias: "o", column: "id"});
```

### Family 与递归具体类型

递归 enum/struct 在函数契约、参数化 family 契约和模块接口中保持精确类型，不会把
递归位置擦除为 `Any`。Family 可以引用已经封闭的非参数化递归具体类型：

```telora
@enum type Expr = {Literal: Value, Call: CallExpr};
@struct type CallExpr = {name: String, args: Array(Expr)};

@struct type Dialect(Context) = {
    render: Fn(Context, Expr) -> String,
};
```

递归类型可以经完整、选择性、alias 或 open import 进入其他模块的函数与 family
契约。Family 自身不能参数化递归、形成循环 family component，也不能调用同模块
普通 helper。

### 泛型函数和外围类型参数

多态函数不能作为“尚未实例化的普通值”依赖后续任意使用来决定全部类型参数。
优先在调用点推断，必要时用 `@[...]` 显式应用，或给具名辅助函数声明完整契约。
另外，泛型函数体内的局部类型标注不能引用外围模块级 `for` 引入的类型参数；把
需要该参数的完整契约提升到模块级辅助函数上。

## String 与诊断

普通字符串不进行插值。插值使用反引号：

```telora
let message = `missing capability \{name}`;
let progress = `ratio=\{3.0}, offset=\{-0.0}`; # "ratio=3, offset=-0"
```

插值支持 String、Int、Float 和 Atom。Float 与 `fmt.display(Float, value)` 都使用
有限 binary64 的 Display 表示：最短、可往返、不受 locale 影响；`3.0` 显示为 `3`，
`-0.0` 显示为 `-0`，原始小数或指数拼写不会保留。`dbg!` 的 Float `repr` 使用
Debug 表示，会将二者分别写成
`3.0` 和 `-0.0`。插值不渲染任意 Tagged、
Struct、Array、Dict、Tuple、Dyn 或用户值。主体是结构化值时，消息应保持静态。

```telora
let error = blame!("missing capability", authored_subject);
let reported = report('Error, error);
let warning = emit_warn!("fallback policy used", authored_subject);
let ignored = emit_error!("missing capability", authored_subject);
raise!(error)
```

Contextual intrinsic 支持 `receiver.ident!(arguments...)` 后置糖，严格等价于把 receiver
放到前置调用的第一个参数。例如 `error.raise!()` 等价于 `raise!(error)`，
`"missing capability".blame!(authored_subject)` 等价于上面的 `blame!` 调用。它不是
method lookup，也不允许调用未由语言定义的 intrinsic。

- `blame!` 构造带来源的 `BlameError` 值，但不报告它。
- `report` 发布诊断，并返回同一个错误。
- `emit_info!` 和 `emit_warn!` 报告非阻塞诊断。
- `emit_error!` 是报告便利形式；Error 诊断会使严格模块执行在其发布边界失败，
  即使局部求值会继续到足以收集彼此独立诊断的位置。
- `raise!` 立即以 `Never` 退出最近的函数。

预期的领域拒绝必须能够与意外的运行时失败区分。如果 eDSL 契约要求值级拒绝
结果，应返回 `Option`、`Result`、显式 enum 或诊断值；不得用 `emit_error!`
替代该结果通道。

## 模块

```telora
import "std/array" as array;
import "@src/local.telora" { compile };
import "ontology-lib/types.telora" as types;

export def compile: Fn(Input) -> Output = fn(input) { ... };
export { Entity, Requirement, compile };
```

import 是静态的。模块只暴露显式 export。eDSL 必须导出向企业作者承诺的每个
类型和函数。

`ontology/src/bin/` 或 `ent-1/src/bin/` 下的入口由 Host 以 `@bin/...` 选择；
`tests/` 下的入口以 `@test/...` 选择。从这些入口导入可复用代码时，应使用显式源码根路径：

```telora
# ontology/src/bin/main.telora
import "@src/ontology.telora" { compile };
```

稳定逻辑模块 ID 与 crate 布局一一对应：

```text
@src/model.telora       -> <crate>/src/model.telora
@bin/main.telora        -> <crate>/src/bin/main.telora
@test/ontology.telora   -> <crate>/tests/ontology.telora
ontology-lib/types.telora -> <ontology-lib>/src/types.telora
```

Host 从当前目录向上查找最近的 `telora-deps.json`，因此可以在 crate 根目录或其任意
子目录运行命令。`run main` 选择 `@bin/main.telora`；binary name 是不含路径分隔符和
`.telora` 后缀的单个 stem。`@main` 不是模块 ID。完整示例：

```text
cd ontology
../bin/telora run main
../bin/telora check @test/ontology.telora
```

在 binary/test 入口中，`./ontology.telora` 以及其他 `./` 或 `../` import 非法。
在 `src/` 下的模块中，相对 import 仍然合法，并从导入模块的逻辑目录解析。
`@src/` 始终从导入模块所属 crate 的源码根解析。`ontology-lib/types.telora`
等 package 路径选择由 crate manifest 固定的依赖；`std/...` 选择由 Host 注册的
内置模块。依赖只公开自身的 `src/`，不公开 `src/bin/` 或 `tests/`。

单文件探索使用 `telora run -S path/to/file.telora`。该模式不查找
`telora-deps.json`，即使文件祖先目录中存在 manifest；它只接受根文件中声明的
`crate.dependency` 和 `crate.format` resolver options，并相对根文件所在目录解析。
普通 crate 模块不能声明这些 resolver options，被 standalone 根导入的模块也不能
再次声明它们。

## 递归与有界工作

Telora 支持带显式契约的递归函数。调用和 back-edge 消耗 fuel；分配也受配额
限制。领域算法必须暴露其契约要求的任何语义深度界限，而不能依赖 Host fuel
最终耗尽来终止。

## 工作规则

- 通过 selector 和回调保留精确的泛型类型。
- 嵌套回调约束不足时，优先使用带显式契约的小型具名辅助函数。
- 企业事实和物理映射留在可复用 eDSL 之外。
- 不得参考仓库中已有的本体实现。
- 不得添加 Host 函数、native 声明、`Any` 或 `Dyn` 来绕过困难的泛型关系。

具体任务会指出还需要阅读的设计或领域材料。
