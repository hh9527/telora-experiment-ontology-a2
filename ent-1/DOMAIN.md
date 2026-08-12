# 私有企业题面：物流履约分析

Customer 创建 Order，Order 从 Warehouse 发货，Warehouse 位于 Region。每个
Order 选择 Carrier 和 ServiceLevel，并产生一个或多个 Package。Package 包含
PackageItem，PackageItem 指向 Product，Product 属于 Category。

## 关系与物理映射

下列 many-to-one 方向在当前 grain 下安全：

```text
Order -> Customer
Order -> Warehouse
Warehouse -> Region
Order -> Carrier
Order -> ServiceLevel
Package -> Order
PackageItem -> Package
PackageItem -> Product
Product -> Category
```

下列方向会扩张 grain：

```text
Order -> Package
Package -> PackageItem
```

每个关系携带由企业所有的物理 mapping，至少表达源表、目标表和 join predicate。

## 指标与维度

- `OrdersCreated`：Order grain，`COUNT(orders.id)`，获准。
- `DeliveredPackages`：Package grain，获准，并要求 Order。
- `UnitsShipped`：PackageItem grain，获准，并要求 Package 和 Order。
- `OrderMonth`：要求 Order，获准。
- `CustomerTier`：要求 Customer，获准。
- `OriginRegion`：要求 Region，获准。
- `CarrierName`：要求 Carrier，获准。
- `ServiceName`：要求 ServiceLevel，获准。
- `ProductCategory`：要求 Category，能力获准，但从 Order grain 只能经 fan-out
  到达。
- `DeliveryException`：属于封闭 Dimension vocabulary，但没有获准 capability。

不同 natural grain 的指标不能自动组合。当前没有预聚合或 allocation policy。

## 可见验证场景

合法场景：`OrdersCreated`，按 `OrderMonth`、`CustomerTier`、`OriginRegion`、
`CarrierName` 和 `ServiceName` 分组。它应发布 Order grain 的只读计划，并保留到
这些实体的安全关系和物理 mapping。

非法场景：`OrdersCreated`，按 `ProductCategory` 和 `DeliveryException` 分组。
一次结果必须同时保留：

- `DeliveryException` 缺少获准 capability；
- `ProductCategory` 从 Order grain 只能经 fan-out 到达；
- 不发布任何部分计划。

最终计划 revision 固定为 `logistics-ontology-v1`。
