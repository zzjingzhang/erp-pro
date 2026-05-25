# SalesOutLetServiceImpl 完整链路分析

## 一、整体流程概览

### 1.1 完整链路图

```
销售订单/销售换货单 审批通过
           ↓
    生成销售出库单
           ↓
    validatorEntity()  — 校验（setData=false，只校验不写回）
           ↓
    createPrepose()    — 创建前置处理
           ↓
    流程审批 (Flowable)
           ↓
    approvalEndIsSuccess()  — 审批成功处理
           ↓
    querySalesOutLetTransById()  — 查询可转仓库出库单数量
           ↓
    insertSalesOutLetToTurnDepot()  — 转仓库出库单（仅允许 PASS）
           ↓
    仓库出库单 审批成功 (DepotOutServiceImpl)
           ↓
    扣减 ORDER_STOCK（实际库存）
```

### 1.2 核心类与枚举

| 类/枚举 | 位置 | 作用 |
|--------|------|------|
| `SalesOutLetServiceImpl` | `com.skyeye.seal.service.impl` | 销售出库单服务实现 |
| `SalesOrderServiceImpl` | `com.skyeye.seal.service.impl` | 销售订单服务实现 |
| `SalesExchangesServiceImpl` | `com.skyeye.seal.service.impl` | 销售换货单服务实现 |
| `DepotOutServiceImpl` | `com.skyeye.depot.service.impl` | 仓库出库单服务实现 |
| `SkyeyeErpOrderServiceImpl` | `com.skyeye.business.service.impl` | ERP 单据通用服务实现 |
| `SealOutLetFromType` | `com.skyeye.seal.classenum` | 销售出库单来源类型枚举 |
| `ErpOrderStateEnum` | `com.skyeye.classenum` | ERP 单据状态枚举 |
| `MaterialNormsStockType` | `com.skyeye.material.classenum` | 库存类型枚举 |
| `FlowableStateEnum` | `com.skyeye.common.enumeration` | 流程状态枚举 |

### 1.3 核心枚举值定义

**SealOutLetFromType（销售出库单来源类型）**：

| Key | Value | 说明 |
|-----|-------|------|
| 1 | SEAL_ORDER | 销售订单 |
| 2 | SALE_EXCHANGES | 销售换货单 |

**ErpOrderStateEnum（ERP 单据状态）**：

| Key | Value | 说明 |
|-----|-------|------|
| `partiallyCompleted` | 部分完成 | PARTIALLY_COMPLETED |
| `completed` | 已完成 | COMPLETED |
| `waitOutDepot` | 待出库 | NEED_Out |
| `partOutDepot` | 部分出库 | PARTIAL_Out |
| `allOutDepot` | 全部出库 | All_Out |

**MaterialNormsStockType（库存类型）**：

| Key | Value | 说明 |
|-----|-------|------|
| 2 | ORDER_STOCK | 现有库存（实际库存） |
| 4 | ALLOCATED_STOCK | 已分配量（"已经分配给销售订单的物料"） |

**FlowableStateEnum（流程状态）**：

| Key | Value | 说明 |
|-----|-------|------|
| `draft` | 草稿 | DRAFT |
| `inExamine` | 审核中 | IN_EXAMINE |
| `pass` | 审核通过 | PASS |
| `reject` | 驳回 | REJECT |
| `revoke` | 撤销 | REVOKE |

---

## 二、创建阶段详解

### 2.1 销售出库单的创建入口

销售出库单可以从两个来源创建：

#### 入口一：从销售订单创建

**调用方法**：`SalesOrderServiceImpl.insertSalesOrderToTurnPut()`

**代码位置**：[SalesOrderServiceImpl.java:L249-L268](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOrderServiceImpl.java#L249-L268)

```java
public void insertSalesOrderToTurnPut(InputObject inputObject, OutputObject outputObject) {
    SalesOutLet salesOutLet = inputObject.getParams(SalesOutLet.class);
    SalesOrder order = selectById(salesOutLet.getId());
    if (ObjectUtil.isEmpty(order)) {
        throw new CustomException("该数据不存在.");
    }
    // 审核通过/部分完成的可以进行出库
    if (FlowableStateEnum.PASS.getKey().equals(order.getState()) 
        || ErpOrderStateEnum.PARTIALLY_COMPLETED.getKey().equals(order.getState())) {
        String userId = inputObject.getLogParams().get("id").toString();
        salesOutLet.setFromId(salesOutLet.getId());
        salesOutLet.setFromTypeId(SealOutLetFromType.SEAL_ORDER.getKey());  // fromTypeId = 1
        salesOutLet.setId(StrUtil.EMPTY);
        salesOutLetService.createEntity(salesOutLet, userId);
    } else {
        outputObject.setreturnMessage("状态错误，无法出库.");
    }
}
```

**状态校验**：`order.getState()` 必须等于 `FlowableStateEnum.PASS` 或 `ErpOrderStateEnum.PARTIALLY_COMPLETED`（L259）。

#### 入口二：从销售换货单创建

**调用方法**：`SalesExchangesServiceImpl.insertSalesExchangesToSalesOutLet()`

**代码位置**：[SalesExchangesServiceImpl.java:L240-L260](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesExchangesServiceImpl.java#L240-L260)

```java
public void insertSalesExchangesToSalesOutLet(InputObject inputObject, OutputObject outputObject) {
    DepotPut salesOutLet = inputObject.getParams(DepotPut.class);
    SalesExchanges salesExchanges = selectById(salesOutLet.getId());
    if (ObjectUtil.isEmpty(salesExchanges)) {
        throw new CustomException("该数据不存在.");
    }
    // 当状态为待出库、部分出库、全部出库时，表示已经审核通过   此时可以进行出库操作
    if (Arrays.asList(ErpOrderStateEnum.NEED_Out.getKey(), ErpOrderStateEnum.PARTIAL_Out.getKey(), ErpOrderStateEnum.All_Out.getKey())
            .contains(salesExchanges.getState())) {
        String userId = inputObject.getLogParams().get("id").toString();
        salesOutLet.setFromId(salesOutLet.getId());
        salesOutLet.setFromTypeId(SealOutLetFromType.SALE_EXCHANGES.getKey());  // fromTypeId = 2
        salesOutLet.setId(StrUtil.EMPTY);
        // 使用 JSON 序列化和反序列化进行转换
        SalesOutLet salesOutLetConverted = JSONUtil.toBean(JSONUtil.toJsonStr(salesOutLet), SalesOutLet.class);
        salesOutLetService.createEntity(salesOutLetConverted, userId);
    } else {
        outputObject.setreturnMessage("状态错误，无法下达仓库入库单.");
    }
}
```

**状态校验**：`salesExchanges.getState()` 必须是 `NEED_Out`、`PARTIAL_Out`、`All_Out` 三者之一（L248-L249）。

**关键差异**：

| 维度 | 销售订单 (SEAL_ORDER) | 销售换货单 (SALE_EXCHANGES) |
|------|----------------------|---------------------------|
| `fromTypeId` | `1` (SEAL_ORDER) | `2` (SALE_EXCHANGES) |
| 前置状态 | `FlowableStateEnum.PASS` 或 `ErpOrderStateEnum.PARTIALLY_COMPLETED` | `ErpOrderStateEnum.NEED_Out`、`PARTIAL_Out`、`All_Out` |
| 对象转换 | 直接使用 `SalesOutLet` | 需要从 `DepotPut` JSON 转换为 `SalesOutLet` |
| `needDepot` 检查 | 无 | **无**（与转入库单不同） |

> **重要修正**：`insertSalesExchangesToSalesOutLet`（L240-L260）**不检查** `needDepot` 字段。只有 `insertSalesExchangesToTurnDepot`（L204-L205）才要求 `needDepot == ENABLE_USING`。这意味着：换货单生成销售出库单和生成仓库入库单是**两个独立的操作**，不要求先入库再出库。

### 2.2 换货单转仓库入库单 vs 转销售出库单

**转入库单**：`insertSalesExchangesToTurnDepot()`

**代码位置**：[SalesExchangesServiceImpl.java:L196-L214](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesExchangesServiceImpl.java#L196-L214)

```java
public void insertSalesExchangesToTurnDepot(InputObject inputObject, OutputObject outputObject) {
    DepotPut depotPut = inputObject.getParams(DepotPut.class);
    SalesExchanges salesExchanges = selectById(depotPut.getId());
    // 当状态为待出库、部分出库、全部出库时，表示已经审核通过     并且该单据需要入库时，则可以进行转入库
    if (Arrays.asList(ErpOrderStateEnum.NEED_Out.getKey(), ErpOrderStateEnum.PARTIAL_Out.getKey(), ErpOrderStateEnum.All_Out.getKey())
            .contains(salesExchanges.getState()) 
            && Objects.equals(salesExchanges.getNeedDepot(), WhetherEnum.ENABLE_USING.getKey())) {
        // ...
    } else {
        outputObject.setreturnMessage("状态错误，无法下达仓库入库单.");
    }
}
```

**校验条件**：`state ∈ {NEED_Out, PARTIAL_Out, All_Out}` **且** `needDepot == ENABLE_USING`（L204-L205）。

**转出库单**：`insertSalesExchangesToSalesOutLet()`

**代码位置**：[SalesExchangesServiceImpl.java:L240-L260](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesExchangesServiceImpl.java#L240-L260)

**校验条件**：仅 `state ∈ {NEED_Out, PARTIAL_Out, All_Out}`（L248-L249），**不检查** `needDepot`。

**可达条件总结**：

| 操作 | 状态条件 | needDepot 条件 | 代码行号 |
|------|----------|---------------|---------|
| 转入库单 | `NEED_Out/PARTIAL_Out/All_Out` | `== ENABLE_USING` | L204-L205 |
| 转销售出库单 | `NEED_Out/PARTIAL_Out/All_Out` | **不检查** | L248-L249 |

> **结论**：从源码可推断，换货单审批通过后，转入库单和转销售出库单是**并行可达**的两个分支，不存在先后依赖。

### 2.3 查询可转数量

**查询可转仓库入库单**：`querySalesExchangesToDepotPutById()`

**代码位置**：[SalesExchangesServiceImpl.java:L173-L193](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesExchangesServiceImpl.java#L173-L193)

```java
public void querySalesExchangesToDepotPutById(InputObject inputObject, OutputObject outputObject) {
    String id = inputObject.getParams().get("id").toString();
    SalesExchanges salesExchanges = selectById(id);
    if (salesExchanges.getNeedDepot() == WhetherEnum.DISABLE_USING.getKey()) {
        throw new CustomException("该销售退货单无需进行转入库操作");
    }
    // ...
}
```

**查询可转销售出库单**：`querySalesExchangesToSalesOutLetById()`

**代码位置**：[SalesExchangesServiceImpl.java:L217-L237](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesExchangesServiceImpl.java#L217-L237)

```java
public void querySalesExchangesToSalesOutLetById(InputObject inputObject, OutputObject outputObject) {
    String id = inputObject.getParams().get("id").toString();
    SalesExchanges salesExchanges = selectById(id);
    if (salesExchanges.getNeedDepot() == WhetherEnum.DISABLE_USING.getKey()) {
        throw new CustomException("该销售退货单无需进行转入库操作");
    }
    // ...
}
```

> **注意**：`querySalesExchangesToSalesOutLetById`（L220-L222）**会检查** `needDepot == DISABLE_USING`，如果禁用则抛出异常。但 `insertSalesExchangesToSalesOutLet`（L240-L260）**不检查** `needDepot`。如果前端调用了 `query...` 接口再调用 `insert...` 接口，则存在隐式的 `needDepot` 检查；如果直接调用 `insert...` 接口，则绕过了该检查。

### 2.4 创建阶段的校验与前置处理

#### validatorEntity() — 校验阶段

**代码位置**：[SalesOutLetServiceImpl.java:L79-L83](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOutLetServiceImpl.java#L79-L83)

```java
public void validatorEntity(SalesOutLet entity) {
    entity.setOtherState(DepotOutState.NEED_OUT.getKey());  // 设置为"待出库"
    checkMaterialNorms(entity, false);  // 校验，setData=false（只校验不写回 operNumber）
}
```

**checkMaterialNorms() 核心逻辑**：

**代码位置**：[SalesOutLetServiceImpl.java:L104-L122](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOutLetServiceImpl.java#L104-L122)

```java
private void checkMaterialNorms(SalesOutLet entity, boolean setData) {
    if (StrUtil.isEmpty(entity.getFromId())) {
        return;
    }
    // 当前销售出库单的商品数量
    Map<String, String> orderNormsNum = entity.getErpOrderItemList().stream()
        .collect(Collectors.toMap(ErpOrderItem::getNormsId,
            item -> StrUtil.isEmpty(item.getOperNumber()) ? CommonNumConstants.NUM_ZERO.toString() : item.getOperNumber()));
    // 获取已经下达销售出库单的商品信息
    Map<String, String> executeNum = calcMaterialNormsNumByFromId(entity.getFromId());
    List<String> inSqlNormsId = new ArrayList<>(executeNum.keySet());
    if (entity.getFromTypeId() == SealOutLetFromType.SEAL_ORDER.getKey()) {
        checkAndUpdateSalesOrderState(entity, setData, orderNormsNum, executeNum, inSqlNormsId);
    } else if (entity.getFromTypeId() == SealOutLetFromType.SALE_EXCHANGES.getKey()) {
        checkAndUpdateSalesExchangesState(entity, setData, orderNormsNum, executeNum, inSqlNormsId);
    }
}
```

#### createPrepose() — 创建前置处理

**代码位置**：[SalesOutLetServiceImpl.java:L85-L92](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOutLetServiceImpl.java#L85-L92)

```java
public void createPrepose(SalesOutLet entity) {
    super.createPrepose(entity);
    entity.setType(DepotPutOutType.OUT.getKey());  // 出库类型
    entity.getErpOrderItemList().forEach(erpOrderItem -> {
        erpOrderItem.setMType(MaterialInOrderType.GENERAL.getKey());  // 普通商品类型
    });
}
```

---

## 三、SEAL_ORDER 与 SALE_EXCHANGES 的核心差异

### 3.1 校验逻辑差异

#### SEAL_ORDER 分支 — checkAndUpdateSalesOrderState()

**代码位置**：[SalesOutLetServiceImpl.java:L124-L150](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOutLetServiceImpl.java#L124-L150)

```java
private void checkAndUpdateSalesOrderState(SalesOutLet entity, boolean setData, 
        Map<String, String> orderNormsNum, 
        Map<String, String> executeNum, 
        List<String> inSqlNormsId) {
    SalesOrder salesOrder = salesOrderService.selectById(entity.getFromId());
    if (CollectionUtil.isEmpty(salesOrder.getErpOrderItemList())) {
        throw new CustomException("该销售订单下未包含商品.");
    }
    super.checkFromOrderMaterialNorms(salesOrder.getErpOrderItemList(), inSqlNormsId);
    // 获取已经下达销售退货单的商品信息
    Map<String, String> returnExecuteNum = salesReturnsService.calcMaterialNormsNumByFromId(entity.getFromId());
    // 来源单据的商品数量 - 当前单据的商品数量 - 已经出库的商品数量 - 已经退货的商品数量
    super.setOrCheckOperNumber(salesOrder.getErpOrderItemList(), setData, orderNormsNum, executeNum, returnExecuteNum);
    if (setData) {
        // 过滤掉剩余数量为0的商品
        List<ErpOrderItem> erpOrderItemList = salesOrder.getErpOrderItemList().stream()
            .filter(erpOrderItem -> {
                String operNumber = StrUtil.isEmpty(erpOrderItem.getOperNumber())
                    ? CommonNumConstants.NUM_ZERO.toString()
                    : erpOrderItem.getOperNumber();
                return CalculationUtil.compareTo(operNumber, CommonNumConstants.NUM_ZERO.toString(), ErpConstants.NUM_AFTER_DOT, RoundingMode.UP) > 0;
            }).collect(Collectors.toList());
        // 如果该销售订单的商品已经全部生成了销售出库单/销售退货单，那说明已经完成了销售订单的内容
        if (CollectionUtil.isEmpty(erpOrderItemList)) {
            salesOrderService.editStateById(salesOrder.getId(), ErpOrderStateEnum.COMPLETED.getKey());
        } else {
            salesOrderService.editStateById(salesOrder.getId(), ErpOrderStateEnum.PARTIALLY_COMPLETED.getKey());
        }
    }
}
```

**关键点**：
- L131：获取已退货数量 `returnExecuteNum`
- L133：传入 **3 个** Map 给 `setOrCheckOperNumber`：`orderNormsNum`（当前出库）、`executeNum`（已出库）、`returnExecuteNum`（已退货）
- L145/L147：更新 `SalesOrder` 的 `state` 为 `COMPLETED` 或 `PARTIALLY_COMPLETED`

#### SALE_EXCHANGES 分支 — checkAndUpdateSalesExchangesState()

**代码位置**：[SalesOutLetServiceImpl.java:L152-L177](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOutLetServiceImpl.java#L152-L177)

```java
private void checkAndUpdateSalesExchangesState(SalesOutLet entity, boolean setData, 
        Map<String, String> orderNormsNum, 
        Map<String, String> executeNum, 
        List<String> inSqlNormsId) {
    SalesExchanges salesExchanges = salesExchangesService.selectById(entity.getFromId());
    if (CollectionUtil.isEmpty(salesExchanges.getErpOrderItemList())) {
        throw new CustomException("该销售订单下未包含商品.");
    }
    super.checkFromOrderMaterialNorms(salesExchanges.getErpOrderItemList(), inSqlNormsId);
    // 来源单据的商品数量 - 当前单据的商品数量 - 已经出库的商品数量
    super.setOrCheckOperNumber(salesExchanges.getErpOrderItemList(), setData, orderNormsNum, executeNum);
    if (setData) {
        // 过滤掉剩余数量为0的商品
        List<ErpOrderItem> erpOrderItemList = salesExchanges.getErpOrderItemList().stream()
            .filter(erpOrderItem -> {
                String operNumber = StrUtil.isEmpty(erpOrderItem.getOperNumber())
                    ? CommonNumConstants.NUM_ZERO.toString()
                    : erpOrderItem.getOperNumber();
                return CalculationUtil.compareTo(operNumber, CommonNumConstants.NUM_ZERO.toString(), ErpConstants.NUM_AFTER_DOT, RoundingMode.UP) > 0;
            }).collect(Collectors.toList());
        // 如果该销售换货订单的商品已经全部生成了销售出库单，那说明已经完成销售换货订单的内容
        if (CollectionUtil.isEmpty(erpOrderItemList)) {
            // 和退货不一样，换货同时需要出库和入库，state-> 出库状态， otherState-> 入库状态
            salesExchangesService.editStateById(salesExchanges.getId(), ErpOrderStateEnum.All_Out.getKey());
        } else {
            salesExchangesService.editStateById(salesExchanges.getId(), ErpOrderStateEnum.PARTIAL_Out.getKey());
        }
    }
}
```

**关键点**：
- L159：传入 **2 个** Map 给 `setOrCheckOperNumber`：`orderNormsNum`（当前出库）、`executeNum`（已出库）
- **不获取**退货数量
- L172/L174：更新 `SalesExchanges` 的 `state` 为 `All_Out` 或 `PARTIAL_Out`

### 3.2 setOrCheckOperNumber 的实际行为

**代码位置**：[SkyeyeErpOrderServiceImpl.java:L439-L446](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L439-L446)

```java
protected void setOrCheckOperNumber(List<ErpOrderItem> erpOrderItemList, boolean setData, Map<String, String>... nums) {
    erpOrderItemList.forEach(erpOrderItem -> {
        String surplusNum = ErpOrderUtil.checkOperNumber(erpOrderItem.getOperNumber(), erpOrderItem.getNormsId(), nums);
        if (setData) {
            erpOrderItem.setOperNumber(surplusNum);
        }
    });
}
```

**底层计算逻辑**（`ErpOrderUtil.checkOperNumber`）：

**代码位置**：[ErpOrderUtil.java:L25-L33](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-common/src/main/java/com/skyeye/util/ErpOrderUtil.java#L25-L33)

```java
public static String checkOperNumber(String surplusNum, String normsId, Map<String, String>... nums) {
    for (Map<String, String> num : nums) {
        surplusNum = CalculationUtil.subtract(surplusNum, 
            num.containsKey(normsId) ? num.get(normsId) : CommonNumConstants.NUM_ZERO.toString(), 
            ErpConstants.NUM_AFTER_DOT);
    }
    if (CalculationUtil.compareTo(surplusNum, CommonNumConstants.NUM_ZERO.toString(), 0, RoundingMode.UP) < 0) {
        throw new CustomException("超出来源单据的商品数量.");
    }
    return surplusNum;
}
```

**实际行为总结**：

| setData | 行为 | 说明 |
|---------|------|------|
| `false`（校验阶段） | 计算 `surplusNum = operNumber - Σnums`，若结果 < 0 抛异常；不写回 | 仅校验数量是否超限 |
| `true`（审批成功阶段） | 计算 `surplusNum`，然后调用 `erpOrderItem.setOperNumber(surplusNum)` 写回 | 将来源单据的 `operNumber` **覆盖为剩余数量** |

> **修正**：`setOrCheckOperNumber` 的实际行为是 **覆盖来源单据 erpOrderItem 的 operNumber 为剩余数量**，而不是"更新剩余数量"。它是对已存在的 operNumber 字段进行减法覆盖写入。

### 3.3 差异对比总结表

| 维度 | SEAL_ORDER (销售订单) | SALE_EXCHANGES (销售换货单) |
|------|----------------------|---------------------------|
| **状态更新对象** | `SalesOrder`（L145/L147） | `SalesExchanges`（L172/L174） |
| **setOrCheckOperNumber 传入 Map 数** | 3 个（当前出库+已出库+已退货）（L133） | 2 个（当前出库+已出库）（L159） |
| **是否扣减退货** | ✅ 是（L131 获取 returnExecuteNum） | ❌ 否 |
| **枚举值** | `COMPLETED` / `PARTIALLY_COMPLETED`（L145/L147） | `All_Out` / `PARTIAL_Out`（L172/L174） |
| **状态含义** | 订单"完成/部分完成" | 换货单"全部出库/部分出库" |
| **计算公式** | `剩余 = 订单数 - 当前出库 - 已出库 - 已退货` | `剩余 = 换单数 - 当前出库 - 已出库` |

### 3.4 为什么销售订单扣减"已出库+已退货"，换货单只扣减"已出库"？

#### 销售订单的三条分支

从 `SalesOrderServiceImpl.querySealsOrderTransById()`（L225-L247）可以看到，销售订单有三条子单据分支：

```
销售订单 (SalesOrder)
   ├── 销售出库单 (SalesOutLet)  — 出库给客户  [L230: normsNum]
   ├── 销售退货单 (SalesReturns) — 客户退货回来 [L232: normsReturnsNum]
   └── 销售换货单 (SalesExchanges) — 换货操作 [L234: normsExchangeNum]
```

销售订单可转单时的计算公式（L236）：
```
剩余 = 订单数 - 已出库 - 已退货 - 已换货
```

但在 `checkAndUpdateSalesOrderState`（L133）中只扣减 **2 个**（当前出库+已出库+已退货），缺少"已换货"。

> **从代码可推断**：换货单是独立的单据类型，在 `SalesExchangesServiceImpl.approvalEndIsSuccess`（L166-L170）中通过 `checkMaterialNorms(entity, true)` 自行扣减销售订单的剩余数量。因此从销售订单直接生成销售出库单时，不需要再考虑换货数量。

#### 销售换货单的生命周期

换货单审批通过后的状态设置（`approvalEnd` L152-L163）：

```java
public void approvalEnd(String processInstanceId, String result) {
    SalesExchanges entity = selectByProcessInstanceId(processInstanceId);
    if (FlowableConstants.APPROVAL_PASS.equalsIgnoreCase(result)) {
        approvalEndIsSuccess(entity);
        // 换货单审批通过后修改状态为待出库
        editStateById(entity.getId(), ErpOrderStateEnum.NEED_Out.getKey());
    }
}
```

换货单只追踪"已出库"，不追踪"已退货"：

| 原因 | 源码依据 |
|------|---------|
| 业务语义不同 | 换货单本身就是"换货"操作单据，不是退货单 |
| 退货走独立流程 | 客户退货走 `SalesReturnsServiceImpl`，不影响换货单 |
| 换货单扣减逻辑对称 | `insertSalesExchangesToSalesOutLet`（L240-L260）只检查出库状态 |

---

## 四、审批成功阶段详解

### 4.1 approvalEndIsSuccess() 核心逻辑

**代码位置**：[SalesOutLetServiceImpl.java:L179-L189](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOutLetServiceImpl.java#L179-L189)

```java
public void approvalEndIsSuccess(SalesOutLet entity) {
    entity = selectById(entity.getId());
    // 修改来源单据信息
    checkMaterialNorms(entity, true);
    // 减少已分配量库存
    entity.getErpOrderItemList().forEach(erpOrderItem -> {
        erpCommonService.editMaterialNormsDepotStock(
            MaterialNormsStockType.ALLOCATED_STOCK.getDefaultDepotId(),
            erpOrderItem.getMaterialId(),
            erpOrderItem.getNormsId(),
            erpOrderItem.getOperNumber(),
            DepotPutOutType.OUT.getKey(),
            MaterialNormsStockType.ALLOCATED_STOCK.getKey());
    });
}
```

**执行步骤**：
1. **L183**：`checkMaterialNorms(entity, true)` — setData=true，将来源单据 erpOrderItem.operNumber 覆盖为剩余数量，并更新来源单据 state
2. **L185-L188**：遍历 erpOrderItemList，无条件减少 `ALLOCATED_STOCK`

### 4.2 SALE_EXCHANGES 来源下 ALLOCATED_STOCK 的不对称

#### SalesExchangesServiceImpl.approvalEndIsSuccess — 不增加 ALLOCATED_STOCK

**代码位置**：[SalesExchangesServiceImpl.java:L166-L170](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesExchangesServiceImpl.java#L166-L170)

```java
public void approvalEndIsSuccess(SalesExchanges entity) {
    entity = selectById(entity.getId());
    // 修改来源单据信息
    checkMaterialNorms(entity, true);
}
```

**仅调用** `checkMaterialNorms(entity, true)`，**不增加** ALLOCATED_STOCK。

#### SalesOrderServiceImpl.approvalEndIsSuccess — 增加 ALLOCATED_STOCK

**代码位置**：[SalesOrderServiceImpl.java:L178-L187](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOrderServiceImpl.java#L178-L187)

```java
public void approvalEndIsSuccess(SalesOrder entity) {
    entity = selectById(entity.getId());
    checkMaterialNorms(entity, true);
    // 增加已分配量库存
    entity.getErpOrderItemList().forEach(erpOrderItem -> {
        erpCommonService.editMaterialNormsDepotStock(
            MaterialNormsStockType.ALLOCATED_STOCK.getDefaultDepotId(),
            erpOrderItem.getMaterialId(),
            erpOrderItem.getNormsId(),
            erpOrderItem.getOperNumber(),
            DepotPutOutType.PUT.getKey(),                    // 入库操作（增加）
            MaterialNormsStockType.ALLOCATED_STOCK.getKey());
    });
}
```

**增加** ALLOCATED_STOCK（L183-L186）。

#### SalesOutLetServiceImpl.approvalEndIsSuccess — 无条件减少 ALLOCATED_STOCK

**代码位置**：[SalesOutLetServiceImpl.java:L179-L189](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOutLetServiceImpl.java#L179-L189)

```java
public void approvalEndIsSuccess(SalesOutLet entity) {
    entity = selectById(entity.getId());
    checkMaterialNorms(entity, true);
    // 减少已分配量库存（无分支，不区分 fromTypeId）
    entity.getErpOrderItemList().forEach(erpOrderItem -> {
        erpCommonService.editMaterialNormsDepotStock(
            MaterialNormsStockType.ALLOCATED_STOCK.getDefaultDepotId(),
            erpOrderItem.getMaterialId(),
            erpOrderItem.getNormsId(),
            erpOrderItem.getOperNumber(),
            DepotPutOutType.OUT.getKey(),                    // 出库操作（减少）
            MaterialNormsStockType.ALLOCATED_STOCK.getKey());
    });
}
```

**L185-L188 无分支**，不检查 `entity.getFromTypeId()`，对 SEAL_ORDER 和 SALE_EXCHANGES 来源一律减少 ALLOCATED_STOCK。

#### 不对称性总结

| 操作 | SEAL_ORDER 来源 | SALE_EXCHANGES 来源 |
|------|----------------|-------------------|
| 审批通过增加 ALLOCATED_STOCK | ✅ SalesOrder L183-L186 | ❌ SalesExchanges L166-L170 |
| 销售出库单审批减少 ALLOCATED_STOCK | ✅ SalesOutLet L185-L188 | ✅ SalesOutLet L185-L188（无条件） |

> **从代码可推断**：对于换货单来源的销售出库单，ALLOCATED_STOCK 会被减少但从未被增加，这可能是代码设计上的遗漏。因为换货单审批通过时没有增加 ALLOCATED_STOCK，而销售出库单审批通过时却无条件减少了它。

### 4.3 库存流转的完整链路

```
阶段1：销售订单审批通过
   SalesOrderServiceImpl.approvalEndIsSuccess() [L178-L187]
       ├── checkMaterialNorms(entity, true)
       └── 增加 ALLOCATED_STOCK [L183-L186]
   库存状态：ORDER_STOCK 不变，ALLOCATED_STOCK 增加

阶段2：销售出库单审批通过
   SalesOutLetServiceImpl.approvalEndIsSuccess() [L179-L189]
       ├── checkMaterialNorms(entity, true)  — 覆盖来源单据 operNumber 为剩余数量
       └── 减少 ALLOCATED_STOCK [L185-L188]  — 无条件，不区分 fromTypeId
   库存状态：ORDER_STOCK 不变，ALLOCATED_STOCK 减少

阶段3：仓库出库单审批通过
   DepotOutServiceImpl.approvalEndIsSuccess()
       └── super.depotOutOrPutSuccess()  — 减少 ORDER_STOCK
   库存状态：ORDER_STOCK 减少
```

### 4.4 为什么减少的是 ALLOCATED_STOCK 而不是 ORDER_STOCK？

| 维度 | 源码依据 | 说明 |
|------|---------|------|
| **业务分层** | SalesOrder L183-L186 增加，SalesOutLet L185-L188 减少 | 销售层只操作分配量，仓库层操作实际库存 |
| **ALLOCATED_STOCK 是过渡库存** | MaterialNormsStockType 定义 | "已经分配给销售订单的物料" |
| **ORDER_STOCK 是最终库存** | SkyeyeErpOrderServiceImpl.depotOutOrPutSuccess | 只有仓库实际出库才扣减 |
| **形成闭环** | SalesOrder 增加，SalesOutLet 减少 | 分配→解除分配，形成完整闭环 |
| **防止重复扣减** | 三层单据各自扣减不同类型 | 避免同一库存被多次扣减 |

---

## 五、转仓库出库单阶段详解

### 5.1 查询可转单数量

**方法**：`querySalesOutLetTransById()`

**代码位置**：[SalesOutLetServiceImpl.java:L191-L209](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOutLetServiceImpl.java#L191-L209)

```java
public void querySalesOutLetTransById(InputObject inputObject, OutputObject outputObject) {
    String id = inputObject.getParams().get("id").toString();
    SalesOutLet salesOutLet = selectById(id);
    // 该销售出库单下的已经下达仓库出库单(审核通过)的数量
    Map<String, String> depotNumMap = depotOutService.calcMaterialNormsNumByFromId(salesOutLet.getId());
    // 设置未下达商品数量 ----- 销售出库单数量 - 已出库数量
    super.setOrCheckOperNumber(salesOutLet.getErpOrderItemList(), true, depotNumMap);
    // 过滤掉数量为0的商品信息
    salesOutLet.setErpOrderItemList(salesOutLet.getErpOrderItemList().stream()
        .filter(erpOrderItem -> {
            String operNumber = StrUtil.isEmpty(erpOrderItem.getOperNumber())
                ? CommonNumConstants.NUM_ZERO.toString()
                : erpOrderItem.getOperNumber();
            return CalculationUtil.compareTo(operNumber, CommonNumConstants.NUM_ZERO.toString(), ErpConstants.NUM_AFTER_DOT, RoundingMode.UP) > 0;
        }).collect(Collectors.toList()));
    outputObject.setBean(salesOutLet);
    outputObject.settotal(CommonNumConstants.NUM_ONE);
}
```

**关键点**（L198）：计算"还能生成多少仓库出库单" = 销售出库单 operNumber - 已生成的仓库出库单数量。

> **修正**：分批方向是 **一个 SalesOutLet → 多个 DepotOut**，即一个销售出库单可以分多次生成仓库出库单。因为 `querySalesOutLetTransById` 计算的是 SalesOutLet 扣减已生成的 DepotOut 后的剩余数量。

### 5.2 真正仓库出库单的生成

**方法**：`insertSalesOutLetToTurnDepot()`

**代码位置**：[SalesOutLetServiceImpl.java:L211-L230](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/seal/service/impl/SalesOutLetServiceImpl.java#L211-L230)

```java
@Transactional(value = TRANSACTION_MANAGER_VALUE, rollbackFor = Exception.class)
public void insertSalesOutLetToTurnDepot(InputObject inputObject, OutputObject outputObject) {
    DepotOut depotOut = inputObject.getParams(DepotOut.class);
    SalesOutLet salesOutLet = selectById(depotOut.getId());
    if (ObjectUtil.isEmpty(salesOutLet)) {
        throw new CustomException("该数据不存在.");
    }
    // 审核通过的可以转到仓库出库单
    if (FlowableStateEnum.PASS.getKey().equals(salesOutLet.getState())) {
        String userId = inputObject.getLogParams().get("id").toString();
        depotOut.setFromId(depotOut.getId());
        depotOut.setFromTypeId(DepotOutFromType.SEAL_OUTLET.getKey());
        depotOut.setId(StrUtil.EMPTY);
        depotOutService.createEntity(depotOut, userId);
    } else {
        outputObject.setreturnMessage("状态错误，无法下达仓库出库单.");
    }
}
```

**状态校验**（L221）：`salesOutLet.getState()` 必须等于 `FlowableStateEnum.PASS`。

### 5.3 为什么只允许 FlowableStateEnum.PASS？

**对比各转单方法的状态校验**：

| 转单方法 | 允许的状态 | 代码行号 |
|---------|-----------|---------|
| `insertSalesOrderToTurnPut` | `FlowableStateEnum.PASS` 或 `ErpOrderStateEnum.PARTIALLY_COMPLETED` | SalesOrder L259 |
| `insertSalesExchangesToSalesOutLet` | `ErpOrderStateEnum.NEED_Out`、`PARTIAL_Out`、`All_Out` | SalesExchanges L248-L249 |
| `insertSalesOutLetToTurnDepot` | **仅** `FlowableStateEnum.PASS` | SalesOutLet L221 |
| `insertDepotOutToTurnPut` | `FlowableStateEnum.PASS` | DepotOut L599-L611 |

**原因分析**：

| 原因 | 说明 |
|------|------|
| **仓库是最终执行层** | 仓库出库是实物出库，必须确保销售出库单已完全审批通过 |
| **流程状态的区别** | `PASS` 是 Flowable 流程的最终稳定状态；而 `PARTIALLY_COMPLETED` 是销售订单的**业务状态**，表示"部分商品已生成出库单"，不是流程状态 |
| **与仓库出库单的转单保持一致** | `DepotOutServiceImpl` 中转物料接收单同样只允许 `FlowableStateEnum.PASS` |

> **从代码可推断**：SalesOutLet 本身没有 `PARTIALLY_COMPLETED` 这样的业务状态，它的 state 字段由 Flowable 流程控制，只有 PASS/REJECT/IN_EXAMINE 等流程状态。因此只能用 `FlowableStateEnum.PASS` 作为转单条件。

---

## 六、真正库存扣减的时机

### 6.1 仓库出库单审批通过时的库存操作

**方法**：`DepotOutServiceImpl.approvalEndIsSuccess()`

**代码位置**：[DepotOutServiceImpl.java:L313-L337](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/depot/service/impl/DepotOutServiceImpl.java#L313-L337)

```java
public void approvalEndIsSuccess(DepotOut entity) {
    entity = selectById(entity.getId());
    if (StrUtil.isEmpty(entity.getFromId())) {
        return;
    }
    String fromTypeIdKey = DepotOutFromType.getItemIdKey(entity.getFromTypeId());
    // 修改来源单据信息（更新销售出库单的 otherState）
    boolean result = checkMaterialNorms(entity, fromTypeIdKey, true);
    // 校验并修改条形码信息
    List<String> normsCodeList = checkNormsCodeAndOutbound(entity, false);
    // ... 其他业务逻辑
    // 修改库存信息以及记录客户/供应商/会员关联的商品
    super.depotOutOrPutSuccess(
        entity.getHolderId(), 
        entity.getHolderKey(), 
        entity.getErpOrderItemList(), 
        DepotPutOutType.OUT.getKey(),
        entity.getFromId(), 
        fromTypeIdKey);
}
```

### 6.2 depotOutOrPutSuccess() — 真正扣减 ORDER_STOCK

**代码位置**：[SkyeyeErpOrderServiceImpl.java:L379-L406](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L379-L406)

```java
protected void depotOutOrPutSuccess(String holderId, String holderKey, 
        List<ErpOrderItem> erpOrderItemList, int type,
        String orderId, String orderIdKey) {
    // ...
    for (ErpOrderItem bean : erpOrderItemList) {
        // 修改库存 —— 扣减 ORDER_STOCK（实际库存）
        erpCommonService.editMaterialNormsDepotStock(
            bean.getDepotId(),                // 具体仓库ID
            bean.getMaterialId(),
            bean.getNormsId(),
            bean.getOperNumber(),
            type,
            MaterialNormsStockType.ORDER_STOCK.getKey());  // 库存类型 = 实际库存
        // ... 记录关联信息
    }
}
```

### 6.3 库存流转完整时序图

```
时间线 ──────────────────────────────────────────────────────────────────────>

阶段1：销售订单审批通过 [SalesOrderServiceImpl L178-L187]
  │
  │   SalesOrderServiceImpl.approvalEndIsSuccess()
  │       ├── checkMaterialNorms(entity, true)
  │       │   └── setOrCheckOperNumber: 覆盖合同子表 operNumber 为剩余
  │       └── 增加 ALLOCATED_STOCK [L183-L186]
  │
  │   库存状态：
  │     ORDER_STOCK     = 100（不变）
  │     ALLOCATED_STOCK = 10（新增）
  │

阶段2：销售出库单审批通过 [SalesOutLetServiceImpl L179-L189]
  │
  │   SalesOutLetServiceImpl.approvalEndIsSuccess()
  │       ├── checkMaterialNorms(entity, true)
  │       │   ├── SEAL_ORDER 分支: setOrCheckOperNumber(3 Map) [L133]
  │       │   │   └── 覆盖 SalesOrder.operNumber = 剩余
  │       │   │   └── 更新 SalesOrder.state → COMPLETED/PARTIALLY_COMPLETED
  │       │   └── SALE_EXCHANGES 分支: setOrCheckOperNumber(2 Map) [L159]
  │       │       └── 覆盖 SalesExchanges.operNumber = 剩余
  │       │       └── 更新 SalesExchanges.state → All_Out/PARTIAL_Out
  │       └── 减少 ALLOCATED_STOCK [L185-L188]（无条件）
  │
  │   库存状态：
  │     ORDER_STOCK     = 100（不变）
  │     ALLOCATED_STOCK = 0（减少）
  │

阶段3：仓库出库单审批通过 [DepotOutServiceImpl L313-L337]
  │
  │   DepotOutServiceImpl.approvalEndIsSuccess()
  │       ├── checkMaterialNorms()
  │       │   └── 更新 SalesOutLet.otherState
  │       └── super.depotOutOrPutSuccess()
  │           └── 减少 ORDER_STOCK
  │
  │   库存状态：
  │     ORDER_STOCK     = 90（减少）← 真正的库存扣减在这里
  │     ALLOCATED_STOCK = 0
  │
```

---

## 七、核心要点总结

### 7.1 fromTypeId 差异总结

| 维度 | SEAL_ORDER (1) | SALE_EXCHANGES (2) | 源码行号 |
|------|---------------|-------------------|---------|
| **状态更新对象** | `SalesOrder` | `SalesExchanges` | SalesOutLet L145/L147 vs L172/L174 |
| **枚举值** | `COMPLETED` / `PARTIALLY_COMPLETED` | `All_Out` / `PARTIAL_Out` | 同上 |
| **setOrCheckOperNumber 传入 Map 数** | 3 个（当前+已出库+已退货） | 2 个（当前+已出库） | L133 vs L159 |

### 7.2 库存类型差异

| 阶段 | 操作的库存类型 | 源码位置 |
|------|-------------|---------|
| 销售订单审批通过 | **增加** `ALLOCATED_STOCK` | SalesOrder L183-L186 |
| 销售出库单审批通过 | **减少** `ALLOCATED_STOCK`（无条件） | SalesOutLet L185-L188 |
| 仓库出库单审批通过 | **减少** `ORDER_STOCK` | SkyeyeErpOrder L379-L406 |

### 7.3 ALLOCATED_STOCK 不对称性

| 来源 | 审批增加 ALLOCATED_STOCK | 出库减少 ALLOCATED_STOCK |
|------|------------------------|------------------------|
| SEAL_ORDER | ✅ SalesOrder L183-L186 | ✅ SalesOutLet L185-L188 |
| SALE_EXCHANGES | ❌ SalesExchanges L166-L170 | ✅ SalesOutLet L185-L188（无条件） |

> **从代码可推断**：换货单来源下 ALLOCATED_STOCK 存在"减少但不增加"的不对称性，可能为设计遗漏。

### 7.4 换货单转入库单 vs 转销售出库单

| 操作 | 状态条件 | needDepot 条件 | 源码行号 |
|------|----------|---------------|---------|
| 转入库单 | `NEED_Out/PARTIAL_Out/All_Out` | `== ENABLE_USING` | SalesExchanges L204-L205 |
| 转销售出库单 | `NEED_Out/PARTIAL_Out/All_Out` | **不检查** | SalesExchanges L248-L249 |

> **从代码可推断**：换货单转入库单和转销售出库单是并行可达的两个分支，不存在先后依赖。原文档中"必须先生成仓库入库单后才能生成销售出库单"的说法不准确。

### 7.5 只允许 FlowableStateEnum.PASS 转仓库出库单

| 原因 | 说明 |
|------|------|
| 仓库是最终执行层 | 实物出库必须确保前置单据完全审批通过 |
| 流程状态的区别 | `PASS` 是 Flowable 最终稳定状态，`PARTIALLY_COMPLETED` 是销售订单的业务状态 |
| 与其他转单保持一致 | 仓库出库单转物料接收单同样只允许 `PASS` |

---

## 八、关键代码位置索引

| 功能 | 文件 | 方法 | 行号 |
|------|------|------|------|
| 销售出库单校验 | SalesOutLetServiceImpl | `validatorEntity()` | L79-L83 |
| 销售出库单创建前置 | SalesOutLetServiceImpl | `createPrepose()` | L85-L92 |
| 销售订单分支校验 | SalesOutLetServiceImpl | `checkAndUpdateSalesOrderState()` | L124-L150 |
| 销售换货单分支校验 | SalesOutLetServiceImpl | `checkAndUpdateSalesExchangesState()` | L152-L177 |
| 销售出库单审批成功 | SalesOutLetServiceImpl | `approvalEndIsSuccess()` | L179-L189 |
| 查询可转仓库出库单 | SalesOutLetServiceImpl | `querySalesOutLetTransById()` | L191-L209 |
| 转仓库出库单 | SalesOutLetServiceImpl | `insertSalesOutLetToTurnDepot()` | L211-L230 |
| 销售订单审批成功(增加分配量) | SalesOrderServiceImpl | `approvalEndIsSuccess()` | L178-L187 |
| 销售订单转出库单 | SalesOrderServiceImpl | `insertSalesOrderToTurnPut()` | L249-L268 |
| 换货单审批成功(不增分配量) | SalesExchangesServiceImpl | `approvalEndIsSuccess()` | L166-L170 |
| 换货单审批通过(设置出库状态) | SalesExchangesServiceImpl | `approvalEnd()` | L152-L163 |
| 换货单转仓库入库单 | SalesExchangesServiceImpl | `insertSalesExchangesToTurnDepot()` | L196-L214 |
| 换货单查可转入库单 | SalesExchangesServiceImpl | `querySalesExchangesToDepotPutById()` | L173-L193 |
| 换货单转销售出库单 | SalesExchangesServiceImpl | `insertSalesExchangesToSalesOutLet()` | L240-L260 |
| 换货单查可转销售出库单 | SalesExchangesServiceImpl | `querySalesExchangesToSalesOutLetById()` | L217-L237 |
| setOrCheckOperNumber 实现 | SkyeyeErpOrderServiceImpl | `setOrCheckOperNumber()` | L439-L446 |
| checkOperNumber 底层计算 | ErpOrderUtil | `checkOperNumber()` | L25-L33 |
| 仓库出库单审批成功(扣减实际库存) | DepotOutServiceImpl | `approvalEndIsSuccess()` | L313-L337 |
| 库存扣减实现 | SkyeyeErpOrderServiceImpl | `depotOutOrPutSuccess()` | L379-L406 |
| 计算已下达数量 | SkyeyeErpOrderServiceImpl | `calcMaterialNormsNumByFromId()` | L477-L506 |
