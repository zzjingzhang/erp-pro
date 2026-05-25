# PurchasePutServiceImpl.checkMaterialNorms 分析文档

## 一、checkMaterialNorms 主流程概述

`checkMaterialNorms` 方法位于 `PurchasePutServiceImpl.java:134-157`，是采购入库单校验和更新来源单据状态的核心方法。

### 核心参数准备：

```java
// 当前采购入库单的商品数量
Map<String, String> orderNormsNum = entity.getErpOrderItemList().stream()
    .collect(Collectors.toMap(ErpOrderItem::getNormsId, ErpOrderItem::getOperNumber));

// 获取已经下达采购入库单的商品信息（同 fromId 已生效的采购入库单）
Map<String, String> executeNum = calcMaterialNormsNumByFromId(entity.getFromId());
List<String> inSqlNormsId = new ArrayList<>(executeNum.keySet());
```

**注意**：`inSqlNormsId` 来源于**同 fromId 已生效采购入库单**的规格集合（`executeNum.keySet()`），不是当前采购入库单的规格集合。`calcMaterialNormsNumByFromId` 查询的是所有 `fromId = entity.getFromId()` 且 `idKey = PurchasePutServiceImpl.class.getName()` 且状态为 PASS/PARTIALLY_COMPLETED/COMPLETED 的采购入库单的商品数量汇总。

根据 `fromTypeId` 的不同，调用四个分支方法：

| fromTypeId | 分支方法 | 文件位置 |
|-----------|---------|---------|
| 采购订单 | `checkAndUpdatePurchaseOrderState` | 251-273 行 |
| 质检单 | `checkAndUpdateQualityInspectionPutState` | 221-249 行 |
| 到货单 | `checkAndUpdatePurchaseDeliveryPutState` | 192-219 行 |
| 整单委外单 | `checkAndUpdateWholeOrderOutState` | 159-190 行 |

---

## 二、四种 fromTypeId 的计算差异对比

### 2.1 计算公式汇总

| 来源单类型 | 计算逻辑 | 扣减项 | 调用方法 |
|-----------|---------|--------|---------|
| **采购订单** | `来源数量 - 当前入库数 - 已入库数 - 已退货数` | 当前单据数量、已入库、已退货 | `setOrCheckOperNumber` |
| **质检单** | `(合格数+让步接收数) - 当前入库数 - 已入库数` | 当前单据数量、已入库 | `ErpOrderUtil.checkOperNumber` |
| **到货单** | `来源数量 - 当前入库数 - 已入库数` | 当前单据数量、已入库 | `setOrCheckOperNumber` |
| **整单委外单** | `来源数量 - 当前入库数 - 已入库数 - 已退货数 - 已换货数` | 当前单据数量、已入库、已退货、已换货 | `setOrCheckOperNumber` |

### 2.2 各分支详细分析

#### 分支一：采购订单 (`checkAndUpdatePurchaseOrderState`)

**代码位置**：PurchasePutServiceImpl.java:251-273

**核心逻辑**：

```java
// 校验当前单据的规格在来源单中是否存在
super.checkFromOrderMaterialNorms(purchaseOrder.getErpOrderItemList(), inSqlNormsId);

// 获取已经下达采购退货单的商品信息
Map<String, String> returnExecuteNum = purchaseReturnsService.calcMaterialNormsNumByFromId(entity.getFromId());

// 来源单据的商品数量 - 当前单据的商品数量 - 已经入库的商品数量 - 已经退货的商品数量
super.setOrCheckOperNumber(purchaseOrder.getErpOrderItemList(), setData, orderNormsNum, executeNum, returnExecuteNum);
```

**特点**：
- 使用 `setOrCheckOperNumber` 方法（传入 3 个 Map：orderNormsNum, executeNum, returnExecuteNum）
- 需要扣减已退货数量
- 状态更新：`editStateById` 更改 `state` 字段
  - 全部完成 → `ErpOrderStateEnum.COMPLETED` ("completed")
  - 部分完成 → `ErpOrderStateEnum.PARTIALLY_COMPLETED` ("partiallyCompleted")

---

#### 分支二：质检单 (`checkAndUpdateQualityInspectionPutState`)

**代码位置**：PurchasePutServiceImpl.java:221-249

**核心逻辑**：

```java
List<String> fromNormsIds = qualityInspection.getQualityInspectionItemList().stream()
    .map(QualityInspectionItem::getNormsId).collect(Collectors.toList());
super.checkIdFromOrderMaterialNorms(fromNormsIds, inSqlNormsId);

qualityInspection.getQualityInspectionItemList().forEach(qualityInspectionItem -> {
    // 合格数量 + 让步接收数量 - 当前订单数量 - 采购入库单的数量
    String qualifiedAndConcession = CalculationUtil.add(ErpConstants.NUM_AFTER_DOT, 
        qualityInspectionItem.getQualifiedNumber(), qualityInspectionItem.getConcessionNumber());
    String surplusNum = ErpOrderUtil.checkOperNumber(qualifiedAndConcession, 
        qualityInspectionItem.getNormsId(), orderNormsNum, executeNum);
    if (setData) {
        qualityInspectionItem.setOperNumber(surplusNum);
    }
});
```

**特点**：
- 直接调用 `ErpOrderUtil.checkOperNumber`（而非父类的 `setOrCheckOperNumber`）
- 基数是 `合格数 + 让步接收数`（不是来源单原始数量）
- 不需要扣减退货/换货数量
- 使用 `checkIdFromOrderMaterialNorms`（传入规格ID列表）
- 状态更新：`editPutState` 更改 `putState` 字段
  - 全部入库 → `QualityInspectionPutState.COMPLATE_PUT` (4)
  - 部分入库 → `QualityInspectionPutState.PARTIAL_PUT` (3)

---

#### 分支三：到货单 (`checkAndUpdatePurchaseDeliveryPutState`)

**代码位置**：PurchasePutServiceImpl.java:192-219

**核心逻辑**：

```java
// 获取所有免检的商品进行采购入库
List<ErpOrderItem> erpOrderItemList = purchaseDelivery.getErpOrderItemList().stream()
    .filter(bean -> bean.getQualityInspection() == OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey())
    .collect(Collectors.toList());

super.checkFromOrderMaterialNorms(erpOrderItemList, inSqlNormsId);

// 来源单据的商品数量 - 当前单据的商品数量 - 已经入库的商品数量
super.setOrCheckOperNumber(erpOrderItemList, setData, orderNormsNum, executeNum);
```

**特点**：
- 先过滤出免检商品（`NOT_NEED_QUALITYINS_INS`），需质检的商品走质检流程
- 使用 `setOrCheckOperNumber`（传入 2 个 Map：orderNormsNum, executeNum）
- 不需要扣减退货/换货数量
- 状态更新：`editOtherState` 更改 `otherState` 字段
  - 全部入库 → `DeliveryPutState.COMPLATE_PUT` (4)
  - 部分入库 → `DeliveryPutState.PARTIAL_PUT` (3)

---

#### 分支四：整单委外单 (`checkAndUpdateWholeOrderOutState`)

**代码位置**：PurchasePutServiceImpl.java:159-190

**核心逻辑**：

```java
// 获取所有免检的商品进行采购入库
List<ErpOrderItem> erpOrderItemList = wholeOrderOut.getErpOrderItemList().stream()
    .filter(bean -> bean.getQualityInspection() == OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey())
    .collect(Collectors.toList());

super.checkFromOrderMaterialNorms(erpOrderItemList, inSqlNormsId);

// 获取已经下达采购退货单的商品信息
Map<String, String> returnExecuteNum = purchaseReturnsService.calcMaterialNormsNumByFromId(entity.getFromId());
// 获取已经下达采购换货单的商品信息
Map<String, String> exchangeExecuteNum = purchaseExchangesService.calcMaterialNormsNumByFromId(entity.getFromId());

// 来源单据的商品数量 - 当前单据的商品数量 - 已经入库的商品数量 - 已经退货的商品数量 - 已经换货的商品数量
super.setOrCheckOperNumber(erpOrderItemList, setData, orderNormsNum, executeNum, returnExecuteNum, exchangeExecuteNum);
```

**特点**：
- 先过滤出免检商品（`NOT_NEED_QUALITYINS_INS`），需质检的商品走质检流程
- 使用 `setOrCheckOperNumber`（传入 4 个 Map：orderNormsNum, executeNum, returnExecuteNum, exchangeExecuteNum）
- 需要扣减已退货数量 **和** 已换货数量（四种类型中唯一包含换货扣减的）
- 状态更新：`editStateById` 更改 `state` 字段
  - 全部完成 → `ErpOrderStateEnum.COMPLETED` ("completed")
  - 部分完成 → `ErpOrderStateEnum.PARTIALLY_COMPLETED` ("partiallyCompleted")

---

## 三、复合场景说明

### 3.1 场景设定

假设有一个规格为 `norms-001` 的商品，各数据如下：

| 数据项 | 数值 | 说明 |
|-------|------|------|
| 来源单原始数量 | 100 | 采购订单/到货单/整单委外单的下单数量 |
| 质检合格数 | 85 | 仅质检单有 |
| 质检让步接收数 | 10 | 仅质检单有 |
| 当前采购入库单数量 | 20 | 本次要入库的数量 |
| 已入库数量 | 50 | 之前已经审核通过的采购入库单数量 |
| 已退货数量 | 5 | 从来源单发起的退货数量 |
| 已换货数量 | 3 | 从来源单发起的换货数量 |

### 3.2 各分支在该场景下的计算

#### 采购订单分支

**调用链**：
```
checkMaterialNorms (setData=false, 校验阶段)
  → checkAndUpdatePurchaseOrderState
    → checkFromOrderMaterialNorms (校验规格是否存在)
    → setOrCheckOperNumber
      → ErpOrderUtil.checkOperNumber (仅校验，不设置)
```

**计算过程**：
```
剩余数量 = 100 (来源数量) - 20 (当前入库) - 50 (已入库) - 5 (已退货)
         = 25
25 ≥ 0，校验通过
```

**审批通过后 (setData=true)**：
- 采购订单的该商品 `operNumber` 被设置为 25
- 如果所有商品剩余数量均为 0 → 采购订单状态改为 `COMPLETED`
- 否则改为 `PARTIALLY_COMPLETED`

---

#### 质检单分支

**调用链**：
```
checkMaterialNorms (setData=false, 校验阶段)
  → checkAndUpdateQualityInspectionPutState
    → checkIdFromOrderMaterialNorms (校验规格是否存在)
    → 直接循环调用 ErpOrderUtil.checkOperNumber (仅校验，不设置)
```

**计算过程**：
```
可用数 = 85 (合格) + 10 (让步接收) = 95
剩余数量 = 95 - 20 (当前入库) - 50 (已入库)
         = 25
25 ≥ 0，校验通过
```

**注意**：质检单不计入已退货和已换货数量，因为退货/换货是针对采购订单发起的，而非质检单。

**审批通过后 (setData=true)**：
- 质检单商品的 `operNumber` 被设置为 25
- 如果所有商品剩余数量均为 0 → 质检单 `putState` 改为 `COMPLATE_PUT` (4)
- 否则改为 `PARTIAL_PUT` (3)

---

#### 到货单分支

**调用链**：
```
checkMaterialNorms (setData=false, 校验阶段)
  → checkAndUpdatePurchaseDeliveryPutState
    → 先过滤出免检商品 (NOT_NEED_QUALITYINS_INS)
    → checkFromOrderMaterialNorms (校验规格是否存在)
    → setOrCheckOperNumber
      → ErpOrderUtil.checkOperNumber (仅校验，不设置)
```

**计算过程**：
```
剩余数量 = 100 (来源数量) - 20 (当前入库) - 50 (已入库)
         = 30
30 ≥ 0，校验通过
```

**审批通过后 (setData=true)**：
- 到货单的该商品 `operNumber` 被设置为 30
- 如果所有免检商品剩余数量均为 0 → 到货单 `otherState` 改为 `COMPLATE_PUT` (4)
- 否则改为 `PARTIAL_PUT` (3)

**关键**：到货单的状态只针对**免检商品**的入库情况，需质检的商品状态由质检单管理。

---

#### 整单委外单分支

**调用链**：
```
checkMaterialNorms (setData=false, 校验阶段)
  → checkAndUpdateWholeOrderOutState
    → 先过滤出免检商品 (NOT_NEED_QUALITYINS_INS)
    → checkFromOrderMaterialNorms (校验规格是否存在)
    → setOrCheckOperNumber (传入 4 个 Map)
      → ErpOrderUtil.checkOperNumber (仅校验，不设置)
```

**计算过程**：
```
剩余数量 = 100 (来源数量) - 20 (当前入库) - 50 (已入库) - 5 (已退货) - 3 (已换货)
         = 22
22 ≥ 0，校验通过
```

**审批通过后 (setData=true)**：
- 整单委外单的该商品 `operNumber` 被设置为 22
- 如果所有免检商品剩余数量均为 0 → 整单委外单状态改为 `COMPLETED`
- 否则改为 `PARTIALLY_COMPLETED`

**关键**：整单委外单是唯一需要扣减换货数量的来源单类型。

---

## 四、setOrCheckOperNumber vs ErpOrderUtil.checkOperNumber

### 4.1 setOrCheckOperNumber (父类方法)

**位置**：SkyeyeErpOrderServiceImpl.java:439-446

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

**逻辑**：
- 遍历来源单的商品列表
- 以 `erpOrderItem.getOperNumber()` 作为初始基数（即来源单商品数量）
- 调用 `ErpOrderUtil.checkOperNumber` 连续扣减所有传入的 Map
- 根据 `setData` 决定是否将剩余数量写回来源单商品

**使用场景**：采购订单、到货单、整单委外单

---

### 4.2 ErpOrderUtil.checkOperNumber (工具方法)

**位置**：ErpOrderUtil.java:25-33

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

**逻辑**：
- 接收初始 `surplusNum` 和可变数量的 Map
- 循环扣减每个 Map 中对应规格的数量（不存在则扣减 0）
- 如果结果小于 0，抛出异常
- 返回剩余数量

**质检单的特殊用法**：
- 质检单不使用父类的 `setOrCheckOperNumber`
- 而是手动计算 `合格数+让步接收数` 作为初始基数
- 手动调用 `ErpOrderUtil.checkOperNumber`
- 手动判断 `setData` 并写回

---

## 五、状态更新汇总

| 来源单类型 | 更新方法 | 更新字段 | 全部完成值 | 部分完成值 |
|-----------|---------|---------|-----------|-----------|
| 采购订单 | `editStateById` | `state` | `"completed"` | `"partiallyCompleted"` |
| 质检单 | `editPutState` | `putState` | `4 (COMPLATE_PUT)` | `3 (PARTIAL_PUT)` |
| 到货单 | `editOtherState` | `otherState` | `4 (COMPLATE_PUT)` | `3 (PARTIAL_PUT)` |
| 整单委外单 | `editStateById` | `state` | `"completed"` | `"partiallyCompleted"` |

---

## 六、inSqlNormsId 的来源及容易误读的原因

### 6.1 inSqlNormsId 的来源

**代码位置**：PurchasePutServiceImpl.java:142-143

```java
// 获取已经下达采购入库单的商品信息
Map<String, String> executeNum = calcMaterialNormsNumByFromId(entity.getFromId());
List<String> inSqlNormsId = new ArrayList<>(executeNum.keySet());
```

**`calcMaterialNormsNumByFromId` 的实现**（SkyeyeErpOrderServiceImpl.java:477-506）：

```java
public Map<String, String> calcMaterialNormsNumByFromId(String fromId) {
    QueryWrapper<T> queryWrapper = new QueryWrapper<>();
    queryWrapper.select(CommonConstants.ID);
    queryWrapper.eq(MybatisPlusUtil.toColumns(ErpOrderCommon::getFromId), fromId);
    queryWrapper.eq(MybatisPlusUtil.toColumns(ErpOrderCommon::getIdKey), getServiceClassName());
    
    // 只查询审批通过，部分出入库，已完成的单据
    List<String> stateList = Arrays.asList(new String[]{
        FlowableStateEnum.PASS.getKey(), 
        ErpOrderStateEnum.PARTIALLY_COMPLETED.getKey(),
        ErpOrderStateEnum.COMPLETED.getKey()
    });
    queryWrapper.in(MybatisPlusUtil.toColumns(ErpOrderCommon::getState), stateList);
    
    // ... 汇总这些单据的商品数量
}
```

### 6.2 容易误读的原因

**变量名 `inSqlNormsId` 的误导性**：

| 容易误解的含义 | 实际含义 |
|-------------|---------|
| 从 SQL 查询出来的规格 ID（来源单的规格） | 已经下达采购入库单的规格 ID |
| "inSql" = 在数据库中的规格 | "inSql" = 在已生效的入库单中的规格 |
| 应该包含来源单的所有规格 | 只包含**已有入库记录**的规格 |

**关键问题**：

如果来源单包含 5 个规格，但只有其中 3 个曾经被入库过，那么：
- `inSqlNormsId` 只包含这 3 个规格的 ID
- 当前采购入库单如果引用了那 2 个"从未入库过"的规格
- 在校验时会出现问题

---

## 七、checkFromOrderMaterialNorms / checkIdFromOrderMaterialNorms 实际在校验什么

### 7.1 checkIdFromOrderMaterialNorms (底层方法)

**位置**：SkyeyeErpOrderServiceImpl.java:420-430

```java
protected void checkIdFromOrderMaterialNorms(List<String> fromNormsIds, List<String> inSqlNormsId) {
    // 求差集(当前单据在来源单据中不包含的商品)
    List<String> diffList = inSqlNormsId.stream()
        .filter(num -> !fromNormsIds.contains(num)).collect(Collectors.toList());
    
    if (CollectionUtil.isNotEmpty(diffList)) {
        List<MaterialNorms> materialNormsList = materialNormsService.selectByIds(diffList.toArray(new String[]{}));
        List<String> normsNames = materialNormsList.stream().map(MaterialNorms::getName).collect(Collectors.toList());
        throw new CustomException(String.format(Locale.ROOT, "该来源单据下未包含如下商品规格：【%s】.",
            Joiner.on(CommonCharConstants.COMMA_MARK).join(normsNames)));
    }
}
```

### 7.2 校验逻辑解析

**参数**：
- `fromNormsIds`：来源单据中包含的规格 ID 列表
- `inSqlNormsId`：当前采购入库单中**已经入库过**的规格 ID 列表（从 `executeNum.keySet()` 来）

**实际校验**：

```
diffList = inSqlNormsId - fromNormsIds
         = {已有入库记录的规格} ∩ {不在来源单中的规格}
```

**真正的校验目的**：

> 检查：当前采购入库单引用的规格中，**曾经被入库过**的那些规格，是否都存在于来源单据中？

**换句话说**：

1. 如果一个规格之前从未被入库过（不在 `inSqlNormsId` 中） → **不校验**
2. 如果一个规格之前被入库过（在 `inSqlNormsId` 中） → **校验它是否在来源单中**

### 7.3 潜在问题

**场景**：
- 来源单 P 包含规格 A、B
- 之前用来源单 P 创建过入库单，入库了规格 A
- 现在来源单 P 被修改了，删除了规格 A（只剩 B）
- 再创建新的入库单引用规格 A

**校验流程**：
1. `inSqlNormsId = [A]`（因为 A 有历史入库记录）
2. `fromNormsIds = [B]`（来源单现在只有 B）
3. `diffList = [A] - [B] = [A]`
4. 抛出异常："该来源单据下未包含如下商品规格：【规格A】"

**但如果是另一个场景**：
- 来源单 P 包含规格 A、B
- 从未创建过入库单
- 现在创建入库单，引用了不存在的规格 C

**校验流程**：
1. `inSqlNormsId = []`（没有历史入库记录）
2. `fromNormsIds = [A, B]`
3. `diffList = [] - [A, B] = []`
4. **校验通过！**（这是一个问题）

### 7.4 两个方法的区别

| 方法 | 参数 | 使用场景 |
|-----|------|---------|
| `checkFromOrderMaterialNorms` | `List<ErpOrderItem>`, `inSqlNormsId` | 采购订单、到货单、整单委外单（从 ErpOrderItem 提取 normsId） |
| `checkIdFromOrderMaterialNorms` | `List<String>`, `inSqlNormsId` | 质检单（直接从 QualityInspectionItem 提取 normsId 列表） |

**本质是一样的**：`checkFromOrderMaterialNorms` 内部只是先把 `ErpOrderItem` 列表转换成 `normsId` 列表，然后调用 `checkIdFromOrderMaterialNorms`。

---

## 八、总结

### 四种来源单的核心差异

| 维度 | 采购订单 | 质检单 | 到货单 | 整单委外单 |
|-----|---------|--------|--------|-----------|
| 基数 | 来源数量 | 合格+让步接收 | 来源数量 | 来源数量 |
| 扣减项 | 当前+已入库+已退货 | 当前+已入库 | 当前+已入库 | 当前+已入库+已退货+已换货 |
| 免检过滤 | 否 | 否 | 是 | 是 |
| 调用方法 | `setOrCheckOperNumber` (3 Map) | 直接 `checkOperNumber` | `setOrCheckOperNumber` (2 Map) | `setOrCheckOperNumber` (4 Map) |
| 状态字段 | `state` | `putState` | `otherState` | `state` |
| 状态枚举 | `ErpOrderStateEnum` | `QualityInspectionPutState` | `DeliveryPutState` | `ErpOrderStateEnum` |

### 关键注意点

1. **质检单入库分支的特殊性**：以 `合格数+让步接收数` 为基数，在 `PurchasePutServiceImpl` 的入库分支中不扣减退货/换货数量，直接调用 `ErpOrderUtil.checkOperNumber`
2. **质检单退货/换货**：质检单可以独立发起退货（`PurchaseReturnsFromType.QUALITY_INSPECTION`，以 `returnNumber` 为基数）和换货（`PurchaseExchangesFromType.QUALITY_INSPECTION`，以 `exchangesNumber` 为基数）
3. **整单委外单的特殊性**：唯一扣减换货数量的类型
4. **到货单和整单委外单**：都只处理免检商品，需质检的走质检流程
5. **inSqlNormsId 的命名误导**：实际是"同 fromId 已生效采购入库单的规格"，不是"来源单的规格"或"当前采购入库单的规格"
6. **校验逻辑的缺陷**：`checkIdFromOrderMaterialNorms` 只校验有历史入库记录的规格，对于首次入库的规格无法校验其是否存在于来源单中
