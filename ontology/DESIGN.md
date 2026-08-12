# 本体 eDSL 设计契约

本文档规定可复用本体 eDSL 的可观察行为。它不规定函数名称、模块布局、
图算法、内部状态表示或教学结构。

其目标是提供一个领域无关的 `ModellingFactory`：结构各异的企业向它提供私有的
领域知识，得到可接受查询意图的 `Model`；模型把合法意图自动编译为规范计划，
规范计划再被纯且确定的后端转换为 SQL。共享库统一负责能力编译、路径策略、
计划组装、覆盖检查、诊断和原子发布。

```text
Model := ModellingFactory(DomainKnowledge)
Plan | Rejection := Model(QueryIntent)
SqlQuery := transform(Plan)
SqlQuery := { sql: String, bindings: Array(Val) }
```

## 所有权边界

```text
可复用 eDSL
  语义角色 TypeMetadata family
  ModellingFactory 与 Model
  能力查找与独立 lowering
  完整性证据与需求收集
  路径选择与分类
  规范 Plan IR 与自动组装
  确定性 Plan -> 参数化 SQL transform
  诊断构造与发布策略
  原子发布编排

企业模型
  封闭的标识类型与领域值
  能力定义与公式
  关系目录与物理映射
  数据源、表达式和 SQL dialect 所需的物理事实
```

eDSL 不得包含企业实体、表、列、SQL 片段、公式、状态码或基于 String 的
标识。企业模型不得重复实现共享的编译规则和分类规则。

## 语义角色 family

eDSL 至少为下列角色暴露可执行的 TypeMetadata family：

- **MeasureDefinition**：标识、语义值、自然粒度实体、聚合分类，以及模型特定
  的 lowering 能力。
- **DimensionDefinition**：标识和模型特定的 lowering 能力；由维度派生的需求
  属于该维度的输出 payload。
- **RelationDefinition**：语义起点和终点、基数分类，以及企业所有的物理映射
  payload。

字段含义由库所有；所有具体的标识、输入、输出、实体、分类和映射类型均由模型
提供。规范 Plan 类型由 eDSL 所有，并通过类型参数容纳企业物理事实。各个 family
以及承诺提供的分类类型必须被导出，并且能够从其他模块使用。

## 能力编译

给定一组请求标识和一个有类型的目录，共享协议必须：

1. 独立检查每项能力的授权状态并定位该能力；
2. 使用原始请求标识调用 lowering；
3. 为每项请求保留对齐的 `Array(Option(Output))` 或等价证据；
4. 收集每个已完成的值，不得伪造替代值；
5. 通过有类型的 selector 派生关系需求并去重；
6. 一项预期的领域拒绝发生后，继续处理彼此独立的工作。

缺失、未授权、不匹配或未成功的能力必须作为带有创作主体的领域拒绝证据可被
观察。它们不得与意外的运行时错误无法区分。

## 路径输入

路径分类接收：

- 一个安全关系目录；
- 一个扇出关系目录；
- 一个基点；
- 一组有序的目标需求；
- 有类型的端点 selector；
- 所选 API 所需的任何有类型相等性能力。

每个关系值，包括其中由企业所有的物理映射，都必须原样通过选择过程。同一条
语义边同时出现在两个目录中属于无效目录事实，并产生带来源的诊断。

## 安全路径选择

对于从基点出发、存在不超过八条边的纯安全路径的每个目标，按照下列策略选择
一条路径：

1. 选择边数最少的路径；
2. 路径长度相同时，按字典序比较它们在安全目录中的索引序列，选择较小序列；
3. 在所选路径中保留从基点到目标的边顺序。

按目标顺序组合所选路径。如果一条边出现在多条所选路径中，只保留它第一次
出现的位置。因此：

- 没有目标时，不选择任何安全边；
- 与所有目标均无关的可达分支被排除；
- 多跳目标贡献其所选路径上的每条边；
- 不能把所有备选路径都传给计划构建器。

这些是可观察要求，并非要求使用某一种搜索算法。

## 扇出与缺失分类

完整可达性使用安全目录与扇出目录的并集，并允许两类边在一条路径中交替出现。

在八条边的界限内，每个有序目标必须恰好被分类为以下一种：

- **safe**：存在纯安全路径；
- **fan-out-only**：不存在纯安全路径，但在并集目录中存在路径；
- **missing**：在界限内，并集目录中不存在路径。

分类保留目标顺序。扇出和缺失诊断保留原始创作的需求主体，而不是只归责于
一个通用节点。

结果还要暴露两次有界遍历中是否有任何一次在八条边后仍有未展开的前沿。
当且仅当配置的界限可能隐藏了更远的可达性时，该 `truncated` 证据为 `True`。
截断的结果不得授权发布。fuel 耗尽是运行时失败，不是截断证据。

## 查询意图

`QueryIntent` 是 Model 面向查询方的有类型输入。API 可以采用不同名称、family
布局或柯里化形式，但其最小语义应接近以下参考形状：

```telora
@struct type Request(Id, Subject, Input) = {
  id: Id,
  subject: Subject,
  input: Input,
};

@struct type QueryRequest(
  MeasureId,
  DimensionId,
  Subject,
  MeasureInput,
  DimensionInput,
) = {
  measures: Array(Request(MeasureId, Subject, MeasureInput)),
  dimensions: Array(Request(DimensionId, Subject, DimensionInput)),
};
```

- `measures` 与 `dimensions` 都是有序请求；顺序必须贯穿能力证据、规范 Plan 的
  对应投影，以及 SQL 的稳定投影/grouping 顺序。
- `id` 是封闭的有类型能力身份，不是 String label；`subject` 用于保留查询意图的
  创作或归责来源；`input` 是该能力 lowering 所需的有类型输入。
- 度量与维度的 `Id`、`Input` 分别参数化，不要求两个 family 为了进入同一个查询
  而共享不必要的具体类型；若某个模型本来使用相同输入，可以实例化为相同类型。
- 查询方只表达“要什么”，不得在 QueryIntent 中提供领域表达式、数据源、物理
  mapping、Plan 节点或 SQL；这些只能来自已经实例化的 Model。
- 实现可以为 filter、order、limit 等增加有类型字段，但只有当它们被 Model 验证、
  完整进入规范 Plan 并由 transform 确定降低时才属于有效扩展。不得接受字段后
  静默忽略，也不得把这些扩展做成任意 SQL String。

Model 必须把 QueryIntent 视为一个原子请求：任一必要能力或路径拒绝时，不发布
部分 Plan 或部分 SqlQuery。若实现对重复请求进行去重，其规范化策略必须公开、确定，
并在证据与 Plan 覆盖检查中可观察；否则保持原请求的逐项对应关系。

## 建模工厂与模型

公共 API 必须让企业用结构化 `DomainKnowledge` 实例化一个模型。领域知识至少
包含能力目录、基础数据源、度量与维度表达式、关系目录与物理映射，以及公共 API
要求的有类型 selector 或相等性能力。

模型是一个查询编译器，不只是领域目录。给定 `QueryIntent`，模型必须执行本文的
能力、路径、诊断和发布协议，返回显式拒绝或已发布的规范 Plan。企业不得通过一个
任意的最终 builder 回调手工构造 Plan，也不得在模型外重复共享编译规则。

API 名称与柯里化形式属于实现选择，但必须能够表达以下关系：

```text
make_model: DomainKnowledge -> Model
make_plan_creator: Model -> Fn(QueryIntent) -> CompileResult(Plan)
make_query_creator: Model -> Fn(QueryIntent) -> CompileResult(SqlQuery)
```

`make_query_creator(model)` 必须在语义上等价于同一模型的 plan creator 与参数化
SQL transform 的组合，不能存在一条绕过 Plan 的独立查询实现。

## 规范 Plan 与自动组装

eDSL 必须拥有并公开一种领域无关、SQL 可降低的规范 Plan IR，以及构成它的封闭
关系表达式表示。Plan 的具体 family、记录和表达式代数由实现选择，但表达式不能
只是预先渲染的 `SELECT`、`JOIN`、`GROUP BY` 或任意 SQL 片段 String。一个已发布
Plan 至少完整保留：

- 基础数据源；
- 按查询意图顺序排列的全部度量投影及其物理表达式；
- 按查询意图顺序排列的全部维度投影及其物理表达式；
- 与维度投影一致的 grouping；
- 按确定性路径顺序排列的所选安全关系及其物理映射；
- 生成确定性只读 SQL 所需的其他规范信息。

Plan 必须由 eDSL 根据已验证的能力输出和所选关系自动组装。只传递目标节点、
丢弃所选边、根据 String label 反查领域目录，或者要求企业回调自行拼出整个 Plan，
都违反本契约。每个所选关系携带的企业物理映射必须原样进入 Plan。

计划组装必须提供覆盖保证：每个成功请求的度量或维度在 Plan 中恰有对应项；每个
被 Plan 使用的非基点需求均有对应的所选关系路径；不得静默漏项、增加未请求项、
用占位值补齐不同输出形状，或把所有备选路径写入 Plan。

## 确定性参数化 SQL 转换

公共 eDSL 必须提供 `Plan -> SqlQuery` 的转换能力，其中 SQL 执行计划至少具有
以下概念形状（实现可以将该类型命名为 `SqlPlan`、`SqlQuery` 或等价名称）：

```telora
@struct type SqlQuery = {
  sql: String,
  bindings: Array(Val),
};
```

`Val` 表示封闭的、可由数据库驱动绑定的值。实现可以使用等价的具名 sum type
承载 String、Bytes、Int、Float、Bool 等值，但不得使用 `Any` 或预渲染 SQL
片段替代它。转换可以由 dialect 参数化，但对固定 Plan 与固定 dialect 必须满足：

- 纯且确定：相同输入产生结构相等的 SqlQuery，包括逐字节相同的 `sql` 和顺序
  相同的 `bindings`；
- 对任何已发布 Plan 均可完成，不再产生领域拒绝；
- 不读取 Model 或领域目录，不重新查找能力、选择路径或判断授权；
- 不根据展示 label 猜测表、列、表达式或 join；
- 不静默丢弃 Plan 中的投影、grouping 或关系；
- 只生成查询语句，不产生写入或执行授权。

企业可以提供 dialect 所需的结构化物理事实或有类型的叶节点渲染能力，但不能在
transform 阶段重新作领域决策。String 只可承载表名、列名、alias 等受约束的
结构名称；运行时值不是 SQL 文本。完整表达式、join predicate 和 SQL clause
必须保留结构，不能以预渲染 String 替代。

值叶节点不得作为 SQL 文本拼接，也不得由 eDSL 执行 SQL literal escaping。
每次值出现只在 `sql` 中产生一个 dialect 所规定的占位符，并把原始值追加到
`bindings`。占位符数量、编号和 bindings 顺序必须由同一次确定性遍历产生并严格
对应。同一表达式分别出现在 SELECT 与 GROUP BY 时，每次出现都贡献自己的
占位符和 binding；不得依靠隐式共享或重排。

表名、列名、alias、函数名和运算符属于 SQL 结构位置，不能成为 binding；它们
必须来自 Plan 中受约束的结构化事实并由 dialect 确定性渲染。最终 SqlQuery 不是
能力标识、路径选择输入或规范 Plan 的替代品。

## 诊断与决策通道

预期的模型结果使用显式表示。编译结果必须至少让调用方区分：

- 已发布的计划；
- 因能力证据不完整而拒绝；
- 因路径仅能扇出、缺失或截断而拒绝；
- 因规范计划无法完整组装而拒绝。

结果还保留逐请求的完成证据和结构化 `BlameError` 值，或者同等精确的有类型
表示。eDSL 不得把已报告的致命诊断用作预期拒绝的唯一表示。

适用时，诊断保留下列三个来源：

1. 意图：请求的标识；
2. 模型事实：涉及的能力或关系；
3. 共享规则：作出拒绝的 eDSL 检查。

彼此独立的诊断先于最终发布决策运行。过早的 gate 不得掩盖无关的失败。

## 原子发布

只有下列条件全部成立时，才能发布计划：

- 每个请求的能力都产生了值；
- 路径分类未被截断；
- 每个必要目标都具有符合上述策略且可接受的安全路径；
- 彼此独立的下游诊断均已运行；
- 规范 Plan 已由 eDSL 完整组装并通过覆盖检查。

否则，结果被显式拒绝，并且不包含部分计划、SQL 文本或 bindings。

## 企业扩展点

企业提供：

1. 封闭的标识、实体、需求和领域输出类型；
2. 具体的度量与维度能力目录；
3. 带有物理映射 payload 的安全关系事实和扇出关系事实；
4. 有类型的标识、lowering、端点、需求和主体 selector；
5. 基础数据源、度量与维度物理表达式，以及 dialect 所需的结构化叶节点事实；
6. 授权和模型特定的语义组合策略；
7. 由其他语义阶段产生的任何附加需求。

企业不重新实现能力遍历、完整性、需求收集、路径选择、分类、诊断顺序或发布
gate，也不手工组装规范 Plan 或绕过 Plan 直接生成 SQL。

## 实现自由与约束

公共 API、文件布局、辅助函数、TypeMetadata family 形状、内部状态和算法均为
实现选择。实现必须保持纯、确定、有类型且领域无关。

不得使用 `Any`、`Dyn`、String 标识、Host 原生图操作、隐藏的可变状态或复制
仓库中的实现。必须使用普通 Telora 完成有界实现。
