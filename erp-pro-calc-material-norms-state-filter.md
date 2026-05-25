# SkyeyeErpOrderServiceImpl 与 PurchaseOrderServiceImpl 中 calcMaterialNormsNumByFromId 方法比较分析

## 一、方法概述

`calcMaterialNormsNumByFromId` 方法的核心作用是：**根据来源单据ID（fromId）统计已下达到目标单据的商品规格数量**。该方法常用于计算"合同/计划中已下达多少，还剩余多少可下达"的业务场景。

---

## 二、两个实现的对比分析

### 2.1 SkyeyeErpOrderServiceImpl.calcMaterialNormsNumByFromId（父类实现）

**源码位置**：[SkyeyeErpOrderServiceImpl.java:477-506](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L477-L506)

```java
@Override
public Map<String, String> calcMaterialNormsNumByFromId(String fromId) {
    QueryWrapper<T> queryWrapper = new QueryWrapper<>();
    queryWrapper.select(CommonConstants.ID);
    queryWrapper.eq(MybatisPlusUtil.toColumns(ErpOrderCommon::getFromId), fromId);
    queryWrapper.eq(MybatisPlusUtil.toColumns(ErpOrderCommon::getIdKey), getServiceClassName());
    // 只查询审批通过，部分出入库，已完成的单据
    List<String> stateList = Arrays.asList(new String[]{FlowableStateEnum.PASS.getKey(), ErpOrderStateEnum.PARTIALLY_COMPLETED.getKey(),
        ErpOrderStateEnum.COMPLETED.getKey()});
    queryWrapper.in(MybatisPlusUtil.toColumns(ErpOrderCommon::getState), stateList);
    List<T> entityList = list(queryWrapper);
    List<String> ids = entityList.stream().map(ErpOrderCommon::getId).collect(Collectors.toList());
    if (CollectionUtil.isEmpty(ids)) {
        return new HashMap<>();
    }
    List<ErpOrderItem> erpOrderItemList = skyeyeErpOrderItemService.queryErpOrderItemByPIds(ids);
    Map<String, String> collect = erpOrderItemList.stream()
        .collect(Collectors.groupingBy(
            ErpOrderItem::getNormsId,
            Collectors.reducing(
                CommonNumConstants.NUM_ZERO.toString(),
                ErpOrderItem::getOperNumber,
                (sum, operNumber) -> {
                    String operNum = StrUtil.isEmpty(operNumber) ? CommonNumConstants.NUM_ZERO.toString() : operNumber;
                    String sumValue = StrUtil.isEmpty(sum) ? CommonNumConstants.NUM_ZERO.toString() : sum;
                    return CalculationUtil.add(ErpConstants.NUM_AFTER_DOT, sumValue, operNum);
                }
            )
        ));
    return collect;
}
```

### 2.2 PurchaseOrderServiceImpl.calcMaterialNormsNumByFromId（子类实现）

**源码位置**：[PurchaseOrderServiceImpl.java:239-271](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L239-L271)

```java
@Override
public Map<String, String> calcMaterialNormsNumByFromId(String fromId) {
    QueryWrapper<PurchaseOrder> queryWrapper = new QueryWrapper<>();
    queryWrapper.eq(MybatisPlusUtil.toColumns(PurchaseOrder::getFromId), fromId);
    queryWrapper.eq(MybatisPlusUtil.toColumns(PurchaseOrder::getIdKey), getServiceClassName());
    // 只查询审批通过，部分入库，已完成的采购订单
    List<String> stateList = Arrays.asList(new String[]{FlowableStateEnum.PASS.getKey(), ErpOrderStateEnum.PARTIALLY_COMPLETED.getKey(),
        ErpOrderStateEnum.COMPLETED.getKey()});
    queryWrapper.in(MybatisPlusUtil.toColumns(PurchaseOrder::getState), stateList);
    List<PurchaseOrder> purchaseOrderList = list(queryWrapper);
    List<String> ids = purchaseOrderList.stream().map(PurchaseOrder::getId).collect(Collectors.toList());
    if (CollectionUtil.isEmpty(ids)) {
        return new HashMap<>();
    }
    // 获取所有的商品信息
    List<ErpOrderItem> erpOrderItemList = skyeyeErpOrderItemService.queryErpOrderItemByPIds(ids);
    if (CollectionUtil.isNotEmpty(erpOrderItemList)) {
        // 分组计算已经下达订单的数量
        return erpOrderItemList.stream()
            .collect(Collectors.groupingBy(
                ErpOrderItem::getNormsId,
                Collectors.reducing(
                    CommonNumConstants.NUM_ZERO.toString(),
                    ErpOrderItem::getOperNumber,
                    (sum, operNumber) -> CalculationUtil.add(
                        ErpConstants.NUM_AFTER_DOT,
                        StrUtil.isEmpty(sum) ? CommonNumConstants.NUM_ZERO.toString() : sum,
                        StrUtil.isEmpty(operNumber) ? CommonNumConstants.NUM_ZERO.toString() : operNumber
                    )
                )
            ));
    }
    return MapUtil.newHashMap();
}
```

### 2.3 功能相同点

| 维度 | 过滤/统计逻辑 |
|------|--------------|
| **来源过滤** | 都按 `fromId` 过滤，只统计来自同一来源单据的数据 |
| **单据类型过滤** | 都按 `idKey` 过滤，确保只统计当前服务类对应的单据类型 |
| **状态过滤** | 都只统计以下三种状态：<br>1. `FlowableStateEnum.PASS`（审批通过）<br>2. `ErpOrderStateEnum.PARTIALLY_COMPLETED`（部分完成）<br>3. `ErpOrderStateEnum.COMPLETED`（已完成） |
| **统计逻辑** | 都按 `normsId` 分组，对 `operNumber` 进行累加求和 |

### 2.4 实现差异点

| 差异点 | SkyeyeErpOrderServiceImpl（父类，第477-506行） | PurchaseOrderServiceImpl（子类，第239-271行） |
|--------|---------------------------------------------|---------------------------------------------|
| **泛型使用** | 使用泛型 `T extends ErpOrderCommon`，更加通用（第477行） | 直接使用具体类 `PurchaseOrder`（第240行） |
| **ID 选择** | 显式 `queryWrapper.select(CommonConstants.ID)`，只查 ID 字段（第479行） | 未指定 select，查询全部字段（第240行） |
| **空值处理** | **只有一步判断**：<br>第488-490行 `if (CollectionUtil.isEmpty(ids))` 直接 return；<br>第491行之后不判断 erpOrderItemList 是否为空，直接进行 stream 分组 | **两步判断**：<br>第249-251行 `if (CollectionUtil.isEmpty(ids))` 先判断 ids；<br>第254行 `if (CollectionUtil.isNotEmpty(erpOrderItemList))` 再判断 erpOrderItemList 是否非空才分组，否则第270行 `return MapUtil.newHashMap()` |
| **空 Map 实现** | `new HashMap<>()`（第489行） | `MapUtil.newHashMap()`（第270行） |

> **注意**：PurchaseOrderServiceImpl 的空值处理比父类多了一步 `erpOrderItemList` 非空判断。父类在第491行直接对 erpOrderItemList 做 stream 操作，当 erpOrderItemList 为空时，`Collectors.groupingBy` 返回空 Map，功能上等价但父类少了一次显式判断。

---

## 三、为什么草稿、审批中、驳回单据不计入"已下达数量"

### 3.1 设计意图分析

从业务逻辑角度看，"已下达数量"代表的是**已经生效并产生实际业务影响**的数量。以下状态的单据不计入：

#### （1）草稿状态（DRAFT）
- 草稿状态说明单据还在创建/编辑中，未提交审批
- 此时商品数量还可能被修改，甚至单据可能被删除
- **如果计入**：会导致来源单据的"剩余数量"提前被占用，但实际上这些采购订单可能永远不会被提交

#### （2）审批中状态（SUBMIT/IN_PROGRESS）
- 审批中的单据还没有获得最终确认
- 审批结果可能通过，也可能驳回
- **如果计入**：在审批过程中就占用来源单据的额度，一旦审批驳回，需要做复杂的回滚处理

#### （3）驳回状态（REJECT）
- 驳回的单据说明审批不通过，业务上不认可该单据
- 驳回的单据需要修改后重新提交，或者直接废弃
- **如果计入**：会导致来源单据的"剩余数量"被错误扣减，影响后续正常业务

### 3.2 只计入三种状态的业务合理性

| 状态 | 业务含义 | 为什么应该计入 |
|------|----------|----------------|
| `FlowableStateEnum.PASS` | 审批通过 | 单据已正式生效，可以开始执行后续业务（如入库、出库） |
| `ErpOrderStateEnum.PARTIALLY_COMPLETED` | 部分完成 | 单据已经部分执行（如部分入库），剩余部分仍然有效 |
| `ErpOrderStateEnum.COMPLETED` | 已完成 | 单据已经全部执行完毕，是已经发生的业务事实 |

**结论**：只有这三种状态的单据才代表"已经确认的业务承诺"，因此应该计入"已下达数量"。

---

## 四、getServiceClassName() 与 idKey 的写入/查询一致性分析

### 4.1 idKey 的写入与查询对应关系

**写入端**：在 [SkyeyeErpOrderServiceImpl.createPrepose](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L182-L191) 中，第184行将当前 Service 的全限定名写入数据库：

```java
@Override
public void createPrepose(T entity) {
    chectErpOrderItem(entity.getErpOrderItemList());
    entity.setIdKey(getServiceClassName());  // 第184行：写入 Service 类全限定名
    getTotalPrice(entity);
    // ...
    super.createPrepose(entity);
}
```

**查询端**：在 `calcMaterialNormsNumByFromId` 中用同样的 `getServiceClassName()` 作为过滤条件：

```java
// 父类第481行
queryWrapper.eq(MybatisPlusUtil.toColumns(ErpOrderCommon::getIdKey), getServiceClassName());

// 子类第242行
queryWrapper.eq(MybatisPlusUtil.toColumns(PurchaseOrder::getIdKey), getServiceClassName());
```

`getServiceClassName()` 继承自 `SkyeyeBusinessServiceImpl`，返回的是**当前运行时 Service 类的全限定名**。例如：
- `PurchaseOrderServiceImpl` 调用时返回 `com.skyeye.purchase.service.impl.PurchaseOrderServiceImpl`
- `SalesOrderServiceImpl` 调用时返回 `com.skyeye.seal.service.impl.SalesOrderServiceImpl`

### 4.2 复制到其他 Service 时的风险分析

**风险核心：写入时写入数据库的 idKey 值，与查询时当前 Service 的 getServiceClassName() 返回值是否一致。**

由于写入和查询都在**同一个 Service 类内部**使用同一个 `getServiceClassName()` 方法，所以直接复制方法体到另一个 Service 本身不会出问题——新 Service 的 `getServiceClassName()` 会返回新类名，查询时匹配的也是该类名写入的数据。

**真正的风险出现在以下场景：**

#### 场景1：数据由 Service A 写入，却由 Service B 查询

假设历史上某张单据在创建时由 `PurchaseOrderServiceImpl` 处理（`createPrepose` 第184行将 idKey 写为 `com.skyeye.purchase.service.impl.PurchaseOrderServiceImpl`），后来因为代码重构或业务变更，统计逻辑被移到另一个 Service（例如 `OtherService`）中调用。此时 `OtherService.getServiceClassName()` 返回不同的值，查询时的 eq 条件就匹配不到任何数据，导致统计结果为 0。

#### 场景2：复制后人为替换为硬编码字符串

如果开发者复制方法后做了如下修改：

```java
// 写入端（createPrepose 第184行）仍然使用 getServiceClassName()
// 查询端复制后被改成了硬编码：
queryWrapper.eq(MybatisPlusUtil.toColumns(PurchaseOrder::getIdKey),
    "com.skyeye.purchase.service.impl.PurchaseOrderServiceImpl");
```

这种情况看似没问题，但如果类被重命名（例如通过 IDE Refactor Rename），`getServiceClassName()` 会自动返回新类名，而硬编码字符串不会被自动更新，导致写入和查询不一致——写入端写的是新类名，查询端搜的是旧类名。

**结论**：使用 `getServiceClassName()` 动态获取类名是正确的做法，它保证了写入端和查询端的天然一致性。不应简单替换为硬编码字符串，除非写入端也做了同样的修改并确保两边严格同步。

### 4.3 idKey 过滤的业务必要性

`idKey` 过滤的目的是：**在共享的数据表或关联查询中，精确区分不同业务类型的单据**。

例如，同一个 `fromId`（来源单据ID）可能被多个业务单据引用：
- 采购合同的 `fromId` 可能关联了多个采购订单（`PurchaseOrder`）
- 如果存在其他类型的单据也引用了同一个 `fromId`，没有 `idKey` 过滤就会把其他业务类型的单据数量也统计进来，导致"已下达数量"被错误放大

因此，`idKey` 过滤是确保数据隔离的关键条件，必须保留。

---

## 五、ErpOrderStateEnum 状态枚举说明

**源码位置**：[ErpOrderStateEnum.java:27-33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/classenum/ErpOrderStateEnum.java#L27-L33)

```java
public enum ErpOrderStateEnum implements SkyeyeEnumClass {
    PARTIALLY_COMPLETED("partiallyCompleted", "部分完成", "orange", true, false),
    COMPLETED("completed", "已完成", "green", true, false),
    NEED_Out("waitOutDepot", "待出库", "blue", true, false),
    PARTIAL_Out("partOutDepot", "部分出库", "orange", true, false),
    All_Out("allOutDepot", "全部出库", "green", true, false);
}
```

状态过滤固定使用以下三种：
1. **`FlowableStateEnum.PASS`** — 审批通过
2. **`ErpOrderStateEnum.PARTIALLY_COMPLETED`** — 部分完成
3. **`ErpOrderStateEnum.COMPLETED`** — 已完成

该列表是业务规则的统一约束，不应随意扩展或修改。

---

## 六、代码改进建议

### 6.1 建议1：子类无特殊需求时不重写 calcMaterialNormsNumByFromId

父类 `SkyeyeErpOrderServiceImpl` 通过泛型 `T` 已实现通用逻辑（第477-506行）。当前 `PurchaseOrderServiceImpl` 的重写（第239-271行）与父类逻辑完全一致，属于代码冗余。建议子类仅在状态过滤条件或聚合逻辑与父类不同时才重写。

### 6.2 建议2：保持 getServiceClassName() 动态机制

不应将 `getServiceClassName()` 替换为硬编码字符串常量。动态机制保证了写入端（`createPrepose` 第184行）和查询端（`calcMaterialNormsNumByFromId` 第481/242行）使用同一个值，天然避免不一致风险。

如果确实需要引入常量，必须确保写入端和查询端同时修改并使用同一常量：

```java
// 示例：在 Service 中定义常量
public static final String ID_KEY = "com.skyeye.purchase.service.impl.PurchaseOrderServiceImpl";

// 写入端
entity.setIdKey(ID_KEY);

// 查询端
queryWrapper.eq(MybatisPlusUtil.toColumns(PurchaseOrder::getIdKey), ID_KEY);
```

### 6.3 建议3：统一空值处理风格

PurchaseOrderServiceImpl 比父类多了一步 `erpOrderItemList` 非空判断（第254行）。虽然功能上等价，但风格不一致。建议要么在父类中统一加上该判断，要么子类移除，保持代码风格一致。

---

## 七、总结

| 要点 | 结论 |
|------|------|
| **两个实现的关系** | 功能完全相同，PurchaseOrderServiceImpl 是 SkyeyeErpOrderServiceImpl 的冗余重写 |
| **状态过滤设计** | 只统计 PASS / PARTIALLY_COMPLETED / COMPLETED 三种状态，确保只统计"已经生效的业务承诺" |
| **草稿/审批中/驳回不计入** | 避免提前占用额度、避免复杂的审批回滚、确保数据一致性 |
| **getServiceClassName() 机制** | 写入端和查询端使用同一动态值，天然保证一致性。复制到其他 Service 时需确认该 Service 的 `getServiceClassName()` 与数据写入时的 idKey 一致 |
| **idKey 过滤的必要性** | 是不同业务类型单据之间数据隔离的关键条件，不可省略 |
| **ErpOrderStateEnum 状态** | 仅 PARTIALLY_COMPLETED 和 COMPLETED 两个状态参与过滤，与 FlowableStateEnum.PASS 共同组成三状态过滤规则 |
