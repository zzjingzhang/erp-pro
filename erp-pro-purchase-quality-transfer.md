# ERP-Pro 采购订单质检与转单逻辑深度分析（修正版）

## 1. 问题背景

本分析聚焦于 `PurchaseOrderServiceImpl` 中采购订单的质检状态管理和转单逻辑，特别关注以下场景：

- 采购订单包含多条明细
- 部分明细需要抽检或全检
- 部分明细免检

核心问题：在这种"部分质检"场景下，系统会如何处理？

---

## 2. 关键枚举定义

### 2.1 订单级质检类型 (OrderQualityInspectionType)

文件位置：[OrderQualityInspectionType.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/classenum/OrderQualityInspectionType.java#L23-L28)

| Key | Value | 字段名 |
|-----|-------|--------|
| 1 | 免检 | NOT_NEED_QUALITYINS_INS |
| 2 | 需要质检 | NEED_QUALITYINS_INS |
| 3 | 部分质检完成 | PARTIAL_QUALITY_INSPECTION |
| 4 | 全部质检完成 | COMPLATE_QUALITY_INSPECTION |

### 2.2 明细级质检类型 (OrderItemQualityInspectionType)

文件位置：[OrderItemQualityInspectionType.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/classenum/OrderItemQualityInspectionType.java#L27-L31)

| Key | Value | 字段名 |
|-----|-------|--------|
| 1 | 免检 | NOT_NEED_QUALITYINS_INS |
| 2 | 抽检 | SAMPLING_INS |
| 3 | 全检 | FULL_INSPECTION |

### 2.3 采购订单到货状态 (OrderArrivalState)

文件位置：[OrderArrivalState.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/classenum/OrderArrivalState.java#L23-L28)

| Key | Value | 字段名 |
|-----|-------|--------|
| 1 | 无需到货 | NOT_NEED_ARRIVAL |
| 2 | 待到货 | NEED_ARRIVAL |
| 3 | 部分到货 | PARTIAL_ARRIVAL |
| 4 | 全部到货 | COMPLATE_ARRIVAL |

### 2.4 到货单免检商品入库状态 (DeliveryPutState)

文件位置：[DeliveryPutState.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/classenum/DeliveryPutState.java#L23-L28)

| Key | Value | 字段名 |
|-----|-------|--------|
| 1 | 无需入库 | NOT_NEED_PUT |
| 2 | 待入库 | NEED_PUT |
| 3 | 部分入库 | PARTIAL_PUT |
| 4 | 全部入库 | COMPLATE_PUT |

### 2.5 质检单入库状态 (QualityInspectionPutState)

文件位置：[QualityInspectionPutState.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/classenum/QualityInspectionPutState.java#L23-L28)

| Key | Value | 字段名 |
|-----|-------|--------|
| 1 | 无需入库 | NOT_NEED_PUT |
| 2 | 待入库 | NEED_PUT |
| 3 | 部分入库 | PARTIAL_PUT |
| 4 | 全部入库 | COMPLATE_PUT |

### 2.6 采购入库单来源类型 (PurchasePutFromType)

文件位置：[PurchasePutFromType.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/classenum/PurchasePutFromType.java#L23-L28)

| Key | Value | 字段名 |
|-----|-------|--------|
| 1 | 采购订单 | PURCHASE_ORDER |
| 2 | 质检单 | QUALITY_INSPECTION |
| 3 | 到货单 | PURCHASE_DELIVERY |
| 4 | 整单委外单 | WHOLE_ORDER_OUT |

### 2.7 到货单来源类型 (PurchaseDeliveryFromType)

文件位置：[PurchaseDeliveryFromType.java](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/classenum/PurchaseDeliveryFromType.java#L23-L27)

| Key | Value | 字段名 |
|-----|-------|--------|
| 1 | 采购订单 | PURCHASE_ORDER |
| 2 | 整单委外单 | WHOLE_ORDER_OUT |
| 3 | 换货单 | PURCHASE_EXCHANGES |

---

## 3. createPrepose/updatePrepose 阶段状态设置

### 3.1 调用链

```
createPrepose(PurchaseOrder entity)
    ↓
super.createPrepose(entity)  // SkyeyeErpOrderServiceImpl
    ↓
setOtherMation(entity)      // PurchaseOrderServiceImpl
    ↓
遍历所有明细调用 setQualityInspection()
    ↓
设置 entity.qualityInspection
    ↓
设置 entity.otherState (到货状态)
```

### 3.2 PurchaseOrderServiceImpl.setOtherMation 核心逻辑

文件位置：[PurchaseOrderServiceImpl.java#L136-L151](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L136-L151)

```java
private static void setOtherMation(PurchaseOrder entity) {
    // 设置质检类型，初始值为 1（免检）
    Integer qualityInspection = OrderQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey();
    for (ErpOrderItem erpOrderItem : entity.getErpOrderItemList()) {
        qualityInspection = setQualityInspection(erpOrderItem, qualityInspection);
        erpOrderItem.setMType(MaterialInOrderType.GENERAL.getKey());
    }
    entity.setQualityInspection(qualityInspection);
    // 设置到货状态
    if (qualityInspection == OrderQualityInspectionType.NEED_QUALITYINS_INS.getKey()) {
        // 2（需要质检）→ 需要先下【到货单】
        entity.setOtherState(OrderArrivalState.NEED_ARRIVAL.getKey());
    } else {
        // 1（免检）→ 无需到货
        entity.setOtherState(OrderArrivalState.NOT_NEED_ARRIVAL.getKey());
    }
}
```

### 3.3 SkyeyeErpOrderServiceImpl.setQualityInspection 核心逻辑

文件位置：[SkyeyeErpOrderServiceImpl.java#L223-L245](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L223-L245)

```java
protected static Integer setQualityInspection(ErpOrderItem erpOrderItem, Integer qualityInspection) {
    // 关键点：只要当前明细是抽检(2)或全检(3)，就将订单级状态设为 2（需要质检）
    if (erpOrderItem.getQualityInspection() == OrderItemQualityInspectionType.SAMPLING_INS.getKey()
        || erpOrderItem.getQualityInspection() == OrderItemQualityInspectionType.FULL_INSPECTION.getKey()) {
        qualityInspection = OrderQualityInspectionType.NEED_QUALITYINS_INS.getKey();
    }
    
    // 抽检校验逻辑
    if (erpOrderItem.getQualityInspection() == OrderItemQualityInspectionType.SAMPLING_INS.getKey()) {
        String qualityInspectionRatio = erpOrderItem.getQualityInspectionRatio();
        if (StrUtil.isEmpty(qualityInspectionRatio)) {
            throw new CustomException("抽检比例不能为空.");
        }
        if (CommonNumConstants.NUM_ZERO.equals(Integer.parseInt(qualityInspectionRatio))) {
            throw new CustomException("抽检比例不能为0.");
        }
        if (Integer.parseInt(qualityInspectionRatio) > 100) {
            throw new CustomException("抽检比例不能大于100.");
        }
    } else if (erpOrderItem.getQualityInspection() == OrderItemQualityInspectionType.FULL_INSPECTION.getKey()) {
        // 全检时清空抽检比例字段
        erpOrderItem.setQualityInspectionRatio(StrUtil.EMPTY);
    }
    
    return qualityInspection;
}
```

### 3.4 场景分析：部分明细需要质检

假设采购订单有 3 条明细：
- 明细 A：免检 (1)
- 明细 B：抽检 (2)，抽检比例 20%
- 明细 C：免检 (1)

**执行流程：**

1. 初始值：`qualityInspection = 1` (免检)
2. 遍历明细 A：免检，不修改，`qualityInspection 仍为 1`
3. 遍历明细 B：抽检，执行：
   - `qualityInspection = 2` (需要质检)
   - 校验抽检比例 20% 在有效范围 (0-100) 内
4. 遍历明细 C：免检，不修改，`qualityInspection 仍为 2`

**最终状态：**
- `entity.qualityInspection = 2` (需要质检)
- `entity.otherState = 2` (待到货)

### 3.5 关键发现

**结论 1：订单级质检状态是"或"逻辑，而非"与"逻辑**

只要**任意一条**明细是抽检(2)或全检(3)，整个订单的 `qualityInspection` 就会被设置为 **2（需要质检）**。

这意味着：
- 全免检 → 订单级 = 1（免检）
- 部分质检 + 部分免检 → 订单级 = 2（需要质检）
- 全质检 → 订单级 = 2（需要质检）

**结论 2：到货状态完全由订单级质检状态决定**

- 订单级 = 1（免检）→ 无需到货（可直接转入库）
- 订单级 = 2（需要质检）→ 待到货（必须先转到货单）

---

## 4. setQualityInspection 对各字段的影响

### 4.1 对抽检比例字段 (qualityInspectionRatio) 的影响

| 明细级质检类型 | 抽检比例字段行为 | 代码位置 |
|---------------|-----------------|----------|
| 免检 (1) | 不校验、不修改（保持原值） | [SkyeyeErpOrderServiceImpl.java#L224-L228](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L224-L228) |
| 抽检 (2) | **必须校验**：不能为空、不能为 0、不能大于 100 | [SkyeyeErpOrderServiceImpl.java#L228-L239](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L228-L239) |
| 全检 (3) | **强制清空**：设置为空字符串 | [SkyeyeErpOrderServiceImpl.java#L240-L243](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L240-L243) |

### 4.2 对全检比例字段的影响

代码中**不存在**"全检比例"字段。全检类型的明细，系统会：
- 将 `qualityInspectionRatio` 设置为空字符串
- 隐含逻辑：全检 = 100% 检查（但不通过比例字段体现）

### 4.3 qualityInspectionRatio 边界情况完整分析

| 边界情况 | 代码行为 | 异常类型 | 代码位置 |
|---------|---------|---------|----------|
| 空值/null/空字符串 | 抛出 CustomException | CustomException "抽检比例不能为空" | [SkyeyeErpOrderServiceImpl.java#L231-L232](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L231-L232) |
| 0 | 抛出 CustomException | CustomException "抽检比例不能为0" | [SkyeyeErpOrderServiceImpl.java#L234-L235](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L234-L235) |
| > 100 | 抛出 CustomException | CustomException "抽检比例不能大于100" | [SkyeyeErpOrderServiceImpl.java#L237-L238](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L237-L238) |
| 负数 | 代码未显式检查，会通过 `> 100` 检查（负数 < 100），实际可通过验证 | 无异常（设计遗漏） | [SkyeyeErpOrderServiceImpl.java#L237](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L237) |
| 非整数/非数字 | Integer.parseInt 抛出 NumberFormatException | NumberFormatException（运行时异常，未被捕获） | [SkyeyeErpOrderServiceImpl.java#L234](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L234) |
| 全检 | 强制清空 qualityInspectionRatio 为空字符串 | 无异常 | [SkyeyeErpOrderServiceImpl.java#L242](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L242) |

### 4.4 对订单级质检状态的影响

订单级质检状态的计算逻辑是**向上覆盖**：
- 初始值 = 1（免检）
- 一旦遇到抽检(2)或全检(3)的明细 → 升级为 2（需要质检）
- 升级后**不再降级**（即使后续遇到免检明细也不会变回 1）

**示例：**

| 遍历顺序 | 明细质检类型 | 订单级状态变化 |
|---------|-------------|---------------|
| 第1条 | 免检(1) | 保持 1 |
| 第2条 | 抽检(2) | 升级为 2 |
| 第3条 | 免检(1) | 保持 2（不降级） |
| 第4条 | 全检(3) | 保持 2 |

---

## 5. 转单逻辑分析

### 5.1 PurchaseOrderServiceImpl.insertPurchaseOrderToTurnPut（转采购入库单）

文件位置：[PurchaseOrderServiceImpl.java#L292-L312](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L292-L312)

```java
public void insertPurchaseOrderToTurnPut(InputObject inputObject, OutputObject outputObject) {
    PurchasePut purchasePut = inputObject.getParams(PurchasePut.class);
    PurchaseOrder order = selectById(purchasePut.getId());
    
    if (ObjectUtil.isEmpty(order)) {
        throw new CustomException("该数据不存在.");
    }
    
    // 条件1：订单状态必须是审批通过 或 部分完成
    if (FlowableStateEnum.PASS.getKey().equals(order.getState()) 
        || ErpOrderStateEnum.PARTIALLY_COMPLETED.getKey().equals(order.getState())) {
        
        // 条件2：订单不能需要质检
        if (order.getQualityInspection() == OrderQualityInspectionType.NEED_QUALITYINS_INS.getKey()) {
            // 异常路径：订单需要质检时，禁止直接转入库
            throw new CustomException("该订单需要进行质检，无法直接转采购入库，请先转【到货单】.");
        }
        
        // 正常路径：创建采购入库单
        String userId = inputObject.getLogParams().get("id").toString();
        purchasePut.setFromId(purchasePut.getId());
        purchasePut.setFromTypeId(PurchasePutFromType.PURCHASE_ORDER.getKey());
        purchasePut.setId(StrUtil.EMPTY);
        purchasePutService.createEntity(purchasePut, userId);
    } else {
        outputObject.setreturnMessage("状态错误，无法下达采购入库单.");
    }
}
```

### 5.2 PurchaseOrderServiceImpl.insertPurchaseOrderToTurnDelivery（转到货单）

文件位置：[PurchaseOrderServiceImpl.java#L316-L336](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L316-L336)

```java
public void insertPurchaseOrderToTurnDelivery(InputObject inputObject, OutputObject outputObject) {
    PurchaseDelivery purchaseDelivery = inputObject.getParams(PurchaseDelivery.class);
    PurchaseOrder order = selectById(purchaseDelivery.getId());
    
    if (ObjectUtil.isEmpty(order)) {
        throw new CustomException("该数据不存在.");
    }
    
    // 条件1：订单状态必须是审批通过 或 部分完成
    if (FlowableStateEnum.PASS.getKey().equals(order.getState()) 
        || ErpOrderStateEnum.PARTIALLY_COMPLETED.getKey().equals(order.getState())) {
        
        // 条件2：订单必须需要质检
        if (order.getQualityInspection() == OrderQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey()) {
            // 异常路径：订单免检时，禁止转到货单
            throw new CustomException("该订单无需进行质检，请直接转采购入库.");
        }
        
        // 正常路径：创建到货单
        String userId = inputObject.getLogParams().get("id").toString();
        purchaseDelivery.setFromId(purchaseDelivery.getId());
        purchaseDelivery.setFromTypeId(PurchaseDeliveryFromType.PURCHASE_ORDER.getKey());
        purchaseDelivery.setId(StrUtil.EMPTY);
        purchaseDeliveryService.createEntity(purchaseDelivery, userId);
    } else {
        outputObject.setreturnMessage("状态错误，无法下达到货单.");
    }
}
```

### 5.3 采购订单两个转单入口的真实代码可达性（精确版本）

**insertPurchaseOrderToTurnPut（转采购入库）：**

代码判断条件：`order.getQualityInspection() == OrderQualityInspectionType.NEED_QUALITYINS_INS.getKey()` 即 `== 2`

| qualityInspection | 代码判断 | 结果 |
|-------------------|---------|------|
| 1 (免检) | `1 == 2` → false | ✅ 允许 |
| 2 (需要质检) | `2 == 2` → true | ❌ 禁止（抛出 CustomException） |
| 3 (部分质检完成) | `3 == 2` → false | ✅ 允许（代码可达） |
| 4 (全部质检完成) | `4 == 2` → false | ✅ 允许（代码可达） |

**insertPurchaseOrderToTurnDelivery（转到货单）：**

代码判断条件：`order.getQualityInspection() == OrderQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey()` 即 `== 1`

| qualityInspection | 代码判断 | 结果 |
|-------------------|---------|------|
| 1 (免检) | `1 == 1` → true | ❌ 禁止（抛出 CustomException） |
| 2 (需要质检) | `2 == 1` → false | ✅ 允许 |
| 3 (部分质检完成) | `3 == 1` → false | ✅ 允许（代码可达） |
| 4 (全部质检完成) | `4 == 1` → false | ✅ 允许（代码可达） |

### 5.4 PurchaseOrderServiceImpl.queryPurchaseOrderTransById（查询转单信息）

文件位置：[PurchaseOrderServiceImpl.java#L423-L447](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L423-L447)

```java
public void queryPurchaseOrderTransById(InputObject inputObject, OutputObject outputObject) {
    String id = inputObject.getParams().get("id").toString();
    PurchaseOrder purchaseOrder = selectById(id);
    // 该采购订单下的已经下达采购退货单(审核通过)的数量
    Map<String, String> normsReturnMap = purchaseReturnsService.calcMaterialNormsNumByFromId(purchaseOrder.getId());
    // 该采购订单下的已经下达采购换货单(审核通过)的数量
    Map<String, String> normsExchangeMap = purchaseExchangesService.calcMaterialNormsNumByFromId(purchaseOrder.getId());
    if (purchaseOrder.getQualityInspection() == OrderQualityInspectionType.NEED_QUALITYINS_INS.getKey()) {
        // 需要质检，计算未到货数量
        Map<String, String> normsNum = purchaseDeliveryService.calcMaterialNormsNumByFromId(id);
        // 设置未下达到货单的商品数量-----采购订单数量 - 已到货数量 - 已退货数量 - 已换货数量
        super.setOrCheckOperNumber(purchaseOrder.getErpOrderItemList(), true, normsNum, normsReturnMap, normsExchangeMap);
    } else {
        // 免检，计算未入库的数量
        Map<String, String> normsNum = purchasePutService.calcMaterialNormsNumByFromId(id);
        // 设置未下达采购入库单的商品数量-----采购订单数量 - 已入库数量 - 已退货数量 - 已换货数量
        super.setOrCheckOperNumber(purchaseOrder.getErpOrderItemList(), true, normsNum, normsReturnMap, normsExchangeMap);
    }
    // 过滤掉数量为0的进行生成采购入库单/到货单/退货单
    purchaseOrder.setErpOrderItemList(purchaseOrder.getErpOrderItemList().stream()
        .filter(erpOrderItem -> CalculationUtil.compareTo(erpOrderItem.getOperNumber(), CommonNumConstants.NUM_ZERO.toString(), ErpConstants.NUM_AFTER_DOT, RoundingMode.UP) > 0)
        .collect(Collectors.toList()));
    outputObject.setBean(purchaseOrder);
    outputObject.settotal(CommonNumConstants.NUM_ONE);
}
```

**关键逻辑：**
- 如果 `qualityInspection == 2`（需要质检）：计算未到货数量，用于下达到货单
- 如果 `qualityInspection != 2`（免检、部分质检完成、全部质检完成）：计算未入库数量，用于下达采购入库单

### 5.5 场景分析：部分明细需要质检的订单

假设订单包含：
- 明细 A：免检（可直接入库）
- 明细 B：抽检（需要先到货再质检）

**转单行为：**

由于订单级 `qualityInspection = 2`（需要质检）：
1. **调用 insertPurchaseOrderToTurnPut** → 抛出 CustomException："该订单需要进行质检，无法直接转采购入库，请先转【到货单】"
2. **调用 insertPurchaseOrderToTurnDelivery** → 成功，创建到货单

**后续流程：**
- 到货单包含所有明细（包括免检和质检）
- 质检单会过滤掉免检明细，只处理需要质检的明细（见 5.7）
- 免检明细在到货单审批后可直接转到采购入库单（来源类型=到货单，见 5.8）
- 质检明细在质检完成后转到采购入库单（来源类型=质检单）

### 5.6 PurchaseDeliveryServiceImpl 关键方法

#### 5.6.1 createPrepose/updatePrepose/setOtherMation

文件位置：[PurchaseDeliveryServiceImpl.java#L112-L164](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L112-L164)

```java
public void createPrepose(PurchaseDelivery entity) {
    super.createPrepose(entity);
    entity.setType(DepotPutOutType.PUT.getKey());
    entity.getErpOrderItemList().forEach(erpOrderItem -> {
        erpOrderItem.setMType(MaterialInOrderType.GENERAL.getKey());
    });
    setOtherMation(entity);
}

private static void setOtherMation(PurchaseDelivery entity) {
    // 设置质检类型
    Integer qualityInspection = OrderQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey();
    for (ErpOrderItem erpOrderItem : entity.getErpOrderItemList()) {
        qualityInspection = setQualityInspection(erpOrderItem, qualityInspection);
        erpOrderItem.setMType(MaterialInOrderType.GENERAL.getKey());
    }
    entity.setQualityInspection(qualityInspection);
    // 获取所有免检的商品
    List<ErpOrderItem> erpOrderItemList = entity.getErpOrderItemList().stream()
        .filter(bean -> bean.getQualityInspection() == OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey())
        .collect(Collectors.toList());
    if (CollectionUtil.isEmpty(erpOrderItemList)) {
        entity.setOtherState(DeliveryPutState.NOT_NEED_PUT.getKey());
    } else {
        entity.setOtherState(DeliveryPutState.NEED_PUT.getKey());
    }
}
```

**到货单的 otherState 含义：**
- 有免检明细 → `otherState = 2`（待入库）
- 无免检明细 → `otherState = 1`（无需入库）

#### 5.6.2 approvalEndIsSuccess（到货单审批成功）

文件位置：[PurchaseDeliveryServiceImpl.java#L259-L262](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L259-L262)

```java
public void approvalEndIsSuccess(PurchaseDelivery entity) {
    entity = selectById(entity.getId());
    checkMaterialNorms(entity, true);
}
```

调用 `checkMaterialNorms` → 如果来源是采购订单，调用 `checkAndUpdatePurchaseOrderState`：

文件位置：[PurchaseDeliveryServiceImpl.java#L234-L256](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L234-L256)

```java
private void checkAndUpdatePurchaseOrderState(PurchaseDelivery entity, boolean setData, ...) {
    PurchaseOrder purchaseOrder = purchaseOrderService.selectById(entity.getFromId());
    // ...
    if (setData) {
        // 过滤掉剩余数量为0的商品
        List<ErpOrderItem> erpOrderItemList = purchaseOrder.getErpOrderItemList().stream()
            .filter(erpOrderItem -> CalculationUtil.compareTo(...) > 0)
            .collect(Collectors.toList());
        // 如果该订单的商品已经全部下达了到货单，那说明已经完成了订单的【到货内容】
        if (CollectionUtil.isEmpty(erpOrderItemList)) {
            purchaseOrderService.editOtherState(purchaseOrder.getId(), OrderArrivalState.COMPLATE_ARRIVAL.getKey());
        } else {
            purchaseOrderService.editOtherState(purchaseOrder.getId(), OrderArrivalState.PARTIAL_ARRIVAL.getKey());
        }
    }
}
```

**到货单审批成功后变化：**
- 更新采购订单的 `otherState`（到货状态）：部分到货(3) 或 全部到货(4)
- **不修改** 采购订单的 `qualityInspection`

#### 5.6.3 editQualityInspection（更新到货单质检状态，并同步到采购订单）

文件位置：[PurchaseDeliveryServiceImpl.java#L265-L327](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L265-L327)

```java
public void editQualityInspection(String id, Integer qualityInspection) {
    UpdateWrapper<PurchaseDelivery> updateWrapper = new UpdateWrapper<>();
    updateWrapper.eq(CommonConstants.ID, id);
    updateWrapper.set(MybatisPlusUtil.toColumns(PurchaseDelivery::getQualityInspection), qualityInspection);
    update(updateWrapper);
    refreshCache(id);
    // 设置父节点质检状态
    PurchaseDelivery purchaseDelivery = selectById(id);
    if (StrUtil.isEmpty(purchaseDelivery.getFromId())) {
        return;
    }
    // 获取同一个单据下的所有已经质检的商品数量
    Map<String, String> qualityInspectionNumMap = queryDeliveryQualityNumByParentId(purchaseDelivery.getFromId());
    if (purchaseDelivery.getFromTypeId() == PurchaseDeliveryFromType.PURCHASE_ORDER.getKey()) {
        // 来源-采购订单
        PurchaseOrder purchaseOrder = purchaseOrderService.selectById(purchaseDelivery.getFromId());
        // 过滤掉【采购订单】中免检的商品
        List<ErpOrderItem> erpOrderItemList = purchaseOrder.getErpOrderItemList().stream()
            .filter(bean -> bean.getQualityInspection() != OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey())
            .collect(Collectors.toList());
        // ... 计算剩余数量 ...
        // 该采购订单的商品已经全部进行了质检
        if (CollectionUtil.isEmpty(erpOrderItemList)) {
            purchaseOrderService.editQualityInspection(purchaseDelivery.getFromId(), OrderQualityInspectionType.COMPLATE_QUALITY_INSPECTION.getKey());
        } else {
            purchaseOrderService.editQualityInspection(purchaseDelivery.getFromId(), OrderQualityInspectionType.PARTIAL_QUALITY_INSPECTION.getKey());
        }
    }
}
```

**关键发现：editQualityInspection 会同步更新采购订单的 qualityInspection！**

- 所有需要质检的商品都质检完 → 采购订单 `qualityInspection = 4`（全部质检完成）
- 还有未质检的 → 采购订单 `qualityInspection = 3`（部分质检完成）

这个方法在质检单审批成功时被调用。

#### 5.6.4 queryPurchaseDeliveryTransById（查询转质检单）

文件位置：[PurchaseDeliveryServiceImpl.java#L352-L369](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L352-L369)

```java
public void queryPurchaseDeliveryTransById(InputObject inputObject, OutputObject outputObject) {
    String id = inputObject.getParams().get("id").toString();
    PurchaseDelivery purchaseDelivery = selectById(id);
    Map<String, String> normsNum = qualityInspectionService.calcMaterialNormsNumByFromId(id);
    // 查询被转到换货单的商品
    Map<String, String> normsNumMap = purchaseExchangesService.calcMaterialNormsNumByFromId(id);
    // 过滤掉【采购到货单】中免检的商品
    List<ErpOrderItem> erpOrderItemList = purchaseDelivery.getErpOrderItemList().stream()
        .filter(bean -> bean.getQualityInspection() != OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey())
        .collect(Collectors.toList());
    super.setOrCheckOperNumber(erpOrderItemList, true, normsNum, normsNumMap);
    // 过滤掉数量为0的进行生成质检单
    purchaseDelivery.setErpOrderItemList(erpOrderItemList.stream()
        .filter(erpOrderItem -> CalculationUtil.compareTo(...) > 0)
        .collect(Collectors.toList()));
    outputObject.setBean(purchaseDelivery);
}
```

**关键逻辑：只保留非免检明细用于转质检单**

- 过滤条件：`bean.getQualityInspection() != OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey()`
- 即：只保留抽检(2)和全检(3)的明细
- 免检(1)的明细被过滤掉，不会出现在质检单中

#### 5.6.5 queryPurchaseDeliveryTransPurchasePutById（查询转到货单入库）

文件位置：[PurchaseDeliveryServiceImpl.java#L394-L411](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L394-L411)

```java
public void queryPurchaseDeliveryTransPurchasePutById(InputObject inputObject, OutputObject outputObject) {
    String id = inputObject.getParams().get("id").toString();
    PurchaseDelivery purchaseDelivery = selectById(id);
    Map<String, String> normsNum = purchasePutService.calcMaterialNormsNumByFromId(id);
    // 查询被转到换货单的商品
    Map<String, String> normsNumMap = purchaseExchangesService.calcMaterialNormsNumByFromId(id);
    // 获取【采购到货单】中免检的商品
    List<ErpOrderItem> erpOrderItemList = purchaseDelivery.getErpOrderItemList().stream()
        .filter(bean -> bean.getQualityInspection() == OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey())
        .collect(Collectors.toList());
    super.setOrCheckOperNumber(erpOrderItemList, true, normsNum, normsNumMap);
    // 过滤掉数量为0的进行生成采购入库单
    purchaseDelivery.setErpOrderItemList(erpOrderItemList.stream()
        .filter(erpOrderItem -> CalculationUtil.compareTo(...) > 0)
        .collect(Collectors.toList()));
    outputObject.setBean(purchaseDelivery);
}
```

**关键逻辑：只保留免检明细用于转到货单入库**

- 过滤条件：`bean.getQualityInspection() == OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey()`
- 即：只保留免检(1)的明细
- 抽检(2)和全检(3)的明细被过滤掉，不会出现在这个入库单中

#### 5.6.6 deliveryToPurchasePut（到货单转采购入库单）

文件位置：[PurchaseDeliveryServiceImpl.java#L414-L433](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L414-L433)

```java
public void deliveryToPurchasePut(InputObject inputObject, OutputObject outputObject) {
    PurchasePut purchasePut = inputObject.getParams(PurchasePut.class);
    PurchaseDelivery purchaseDelivery = selectById(purchasePut.getId());
    if (ObjectUtil.isEmpty(purchaseDelivery)) {
        throw new CustomException("该数据不存在.");
    }
    // 【审核通过】的并且【免检商品入库状态为待入库，部分入库】的可以进行采购入库
    if (FlowableStateEnum.PASS.getKey().equals(purchaseDelivery.getState()) &&
        (purchaseDelivery.getOtherState() == DeliveryPutState.NEED_PUT.getKey()
            || purchaseDelivery.getOtherState() == DeliveryPutState.PARTIAL_PUT.getKey())) {
        String userId = inputObject.getLogParams().get("id").toString();
        purchasePut.setFromId(purchasePut.getId());
        purchasePut.setFromTypeId(PurchasePutFromType.PURCHASE_DELIVERY.getKey());
        purchasePut.setId(StrUtil.EMPTY);
        purchasePutService.createEntity(purchasePut, userId);
    } else {
        outputObject.setreturnMessage("状态错误，无法转采购入库单.");
    }
}
```

**创建的采购入库单来源类型：**
- `fromTypeId = PurchasePutFromType.PURCHASE_DELIVERY.getKey()` → **3（到货单）**

### 5.7 QualityInspectionServiceImpl 关键方法

#### 5.7.1 approvalEndIsSuccess（质检单审批成功）

文件位置：[QualityInspectionServiceImpl.java#L301-L305](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L301-L305)

```java
public void approvalEndIsSuccess(QualityInspection entity) {
    entity = selectById(entity.getId());
    // 修改来源单据的质检信息
    checkMaterialNorms(entity, true);
}
```

调用 `checkMaterialNorms` → 如果来源是到货单：

文件位置：[QualityInspectionServiceImpl.java#L208-L257](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L208-L257)

```java
if (entity.getFromTypeId() == QualityInspectionFromType.PURCHASE_DELIVERY.getKey()) {
    // 到货单
    PurchaseDelivery purchaseDelivery = purchaseDeliveryService.selectById(entity.getFromId());
    // 过滤掉到货单中免检的商品
    List<ErpOrderItem> erpOrderItemList = purchaseDelivery.getErpOrderItemList().stream()
        .filter(bean -> bean.getQualityInspection() != OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey())
        .collect(Collectors.toList());
    // ...
    if (setData) {
        // 过滤掉剩余数量为0的商品
        erpOrderItemList = erpOrderItemList.stream()
            .filter(erpOrderItem -> CalculationUtil.compareTo(...) > 0)
            .collect(Collectors.toList());
        // 该到货单的商品已经全部进行了质检
        if (CollectionUtil.isEmpty(erpOrderItemList)) {
            purchaseDeliveryService.editQualityInspection(purchaseDelivery.getId(), OrderQualityInspectionType.COMPLATE_QUALITY_INSPECTION.getKey());
        } else {
            purchaseDeliveryService.editQualityInspection(purchaseDelivery.getId(), OrderQualityInspectionType.PARTIAL_QUALITY_INSPECTION.getKey());
        }
    }
}
```

**质检单审批成功后变化：**
- 更新到货单的 `qualityInspection`：部分质检完成(3) 或 全部质检完成(4)
- 通过 `editQualityInspection` 同步更新采购订单的 `qualityInspection`（见 5.6.3）

#### 5.7.2 queryQualityInspectionTransById（查询转采购入库）

文件位置：[QualityInspectionServiceImpl.java#L398-L419](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L398-L419)

```java
public void queryQualityInspectionTransById(InputObject inputObject, OutputObject outputObject) {
    String id = inputObject.getParams().get("id").toString();
    QualityInspection qualityInspection = selectById(id);
    Map<String, String> normsNum = purchasePutService.calcMaterialNormsNumByFromId(id);
    qualityInspection.getQualityInspectionItemList().forEach(qualityInspectionItem -> {
        String qualifiedAndConcession = CalculationUtil.add(ErpConstants.NUM_AFTER_DOT, qualityInspectionItem.getQualifiedNumber(), qualityInspectionItem.getConcessionNumber());
        String normsNumValue = normsNum.containsKey(qualityInspectionItem.getNormsId())
            ? normsNum.get(qualityInspectionItem.getNormsId())
            : CommonNumConstants.NUM_ZERO.toString();
        String surplusNum = CalculationUtil.subtract(qualifiedAndConcession, normsNumValue, ErpConstants.NUM_AFTER_DOT);
        // 设置未下达采购入库单的商品数量
        qualityInspectionItem.setOperNumber(surplusNum);
    });
    // 过滤掉数量为0的进行生成采购入库单
    qualityInspection.setQualityInspectionItemList(qualityInspection.getQualityInspectionItemList().stream()
        .filter(qualityInspectionItem -> CalculationUtil.compareTo(...) > 0)
        .collect(Collectors.toList()));
    outputObject.setBean(qualityInspection);
}
```

#### 5.7.3 qualityInspectionToPurchasePut（质检单转采购入库单）

文件位置：[QualityInspectionServiceImpl.java#L422-L439](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L422-L439)

```java
public void qualityInspectionToPurchasePut(InputObject inputObject, OutputObject outputObject) {
    PurchasePut purchasePut = inputObject.getParams(PurchasePut.class);
    QualityInspection qualityInspection = selectById(purchasePut.getId());
    if (ObjectUtil.isEmpty(qualityInspection)) {
        throw new CustomException("该数据不存在.");
    }
    // 审核通过的可以进行采购入库单
    if (FlowableStateEnum.PASS.getKey().equals(qualityInspection.getState())) {
        String userId = inputObject.getLogParams().get("id").toString();
        purchasePut.setFromId(purchasePut.getId());
        purchasePut.setFromTypeId(PurchasePutFromType.QUALITY_INSPECTION.getKey());
        purchasePut.setId(StrUtil.EMPTY);
        purchasePutService.createEntity(purchasePut, userId);
    } else {
        outputObject.setreturnMessage("状态错误，无法下达质检单.");
    }
}
```

**创建的采购入库单来源类型：**
- `fromTypeId = PurchasePutFromType.QUALITY_INSPECTION.getKey()` → **2（质检单）**

### 5.8 PurchasePutServiceImpl 关键方法

#### 5.8.1 checkAndUpdatePurchaseDeliveryPutState（到货单来源的采购入库单）

文件位置：[PurchasePutServiceImpl.java#L192-L219](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L192-L219)

```java
private void checkAndUpdatePurchaseDeliveryPutState(PurchasePut entity, boolean setData, ...) {
    PurchaseDelivery purchaseDelivery = purchaseDeliveryService.selectById(entity.getFromId());
    // 获取所有免检的商品进行采购入库
    List<ErpOrderItem> erpOrderItemList = purchaseDelivery.getErpOrderItemList().stream()
        .filter(bean -> bean.getQualityInspection() == OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey())
        .collect(Collectors.toList());
    if (CollectionUtil.isEmpty(erpOrderItemList)) {
        throw new CustomException("该到货单下未包含需要免检的商品，请走质检流程");
    }
    // ...
    if (setData) {
        // 如果该到货单的商品(免检)已经全部生成了采购入库单，那说明已经完成了到货单的入库内容
        if (CollectionUtil.isEmpty(erpOrderItemList)) {
            purchaseDeliveryService.editOtherState(purchaseDelivery.getId(), DeliveryPutState.COMPLATE_PUT.getKey());
        } else {
            purchaseDeliveryService.editOtherState(purchaseDelivery.getId(), DeliveryPutState.PARTIAL_PUT.getKey());
        }
    }
}
```

**关键逻辑：**
- 只处理免检明细：`bean.getQualityInspection() == OrderItemQualityInspectionType.NOT_NEED_QUALITYINS_INS.getKey()`
- 如果到货单没有免检明细，抛出 CustomException："该到货单下未包含需要免检的商品，请走质检流程"
- 更新到货单的 `otherState`（DeliveryPutState）：部分入库(3) 或 全部入库(4)

#### 5.8.2 checkAndUpdateQualityInspectionPutState（质检单来源的采购入库单）

文件位置：[PurchasePutServiceImpl.java#L221-L249](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L221-L249)

```java
private void checkAndUpdateQualityInspectionPutState(PurchasePut entity, boolean setData, ...) {
    QualityInspection qualityInspection = qualityInspectionService.selectById(entity.getFromId());
    // ...
    if (setData) {
        // 如果该质检单的商品已经全部生成了采购入库单，那说明已经完成了质检单的内容
        if (CollectionUtil.isEmpty(qualityInspectionItemList)) {
            qualityInspectionService.editPutState(qualityInspection.getId(), QualityInspectionPutState.COMPLATE_PUT.getKey());
        } else {
            qualityInspectionService.editPutState(qualityInspection.getId(), QualityInspectionPutState.PARTIAL_PUT.getKey());
        }
    }
}
```

**关键逻辑：**
- 更新质检单的 `putState`：部分入库(3) 或 全部入库(4)

#### 5.8.3 approvalEndIsSuccess（采购入库单审批成功）

文件位置：[PurchasePutServiceImpl.java#L276-L285](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L276-L285)

```java
public void approvalEndIsSuccess(PurchasePut entity) {
    entity = selectById(entity.getId());
    // 修改来源单据信息
    checkMaterialNorms(entity, true);
    // 减少在途库存
    entity.getErpOrderItemList().forEach(erpOrderItem -> {
        erpCommonService.editMaterialNormsDepotStock(MaterialNormsStockType.IN_TRANSIT_STOCK.getDefaultDepotId(), erpOrderItem.getMaterialId(),
            erpOrderItem.getNormsId(), erpOrderItem.getOperNumber(), DepotPutOutType.OUT.getKey(), MaterialNormsStockType.IN_TRANSIT_STOCK.getKey());
    });
}
```

**采购入库单审批成功后变化：**
- 根据来源类型更新对应来源单据的状态
- 减少在途库存

---

## 6. 免检明细与质检明细的采购入库单来源类型

### 6.1 明细流转路径总结

假设采购订单包含 2 条明细：
- 明细 A：免检(1)
- 明细 B：抽检(2)

**流转路径：**

```
采购订单 (qualityInspection=2, otherState=2)
    ↓
    ├─→ 转到货单 (PurchaseOrderServiceImpl.insertPurchaseOrderToTurnDelivery)
    ↓
    到货单 (包含明细A和B)
    ↓
    ├─→ 免检明细A → queryPurchaseDeliveryTransPurchasePutById → deliveryToPurchasePut
    │       ↓
    │   采购入库单 (fromTypeId=3, 来源=到货单)
    │       ↓
    │   采购入库单审批 → checkAndUpdatePurchaseDeliveryPutState
    │       ↓
    │   更新到货单 otherState (DeliveryPutState: 部分入库/全部入库)
    ↓
    └─→ 质检明细B → queryPurchaseDeliveryTransById → deliveryToQualityInspection
            ↓
        质检单 (只包含明细B)
            ↓
        质检单审批 → checkMaterialNorms → editQualityInspection
            ↓
        更新到货单 qualityInspection + 同步更新采购订单 qualityInspection
            ↓
        queryQualityInspectionTransById → qualityInspectionToPurchasePut
            ↓
        采购入库单 (fromTypeId=2, 来源=质检单)
            ↓
        采购入库单审批 → checkAndUpdateQualityInspectionPutState
            ↓
        更新质检单 putState (部分入库/全部入库)
```

### 6.2 采购入库单来源类型对照表

| 明细类型 | 转单入口 | 采购入库单 fromTypeId | 来源类型 | 代码位置 |
|---------|---------|----------------------|---------|----------|
| 免检明细（从到货单） | `deliveryToPurchasePut` | 3 (PURCHASE_DELIVERY) | 到货单 | [PurchaseDeliveryServiceImpl.java#L427](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L427) |
| 质检明细（从质检单） | `qualityInspectionToPurchasePut` | 2 (QUALITY_INSPECTION) | 质检单 | [QualityInspectionServiceImpl.java#L433](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L433) |
| 整单免检（从采购订单） | `insertPurchaseOrderToTurnPut` | 1 (PURCHASE_ORDER) | 采购订单 | [PurchaseOrderServiceImpl.java#L307](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L307) |

---

## 7. 为什么"只看部分明细免检"会得出错误结论

### 7.1 错误推理路径

错误模型可能会这样推理：
```
1. 订单有 3 条明细：A(免检)、B(抽检)、C(免检)
2. 部分明细是免检的（A 和 C）
3. 因此：免检的明细应该可以直接转入库
4. 结论：应该允许直接转采购入库
```

### 7.2 实际系统逻辑

系统的实际逻辑是：
```
1. 订单有 3 条明细：A(免检)、B(抽检)、C(免检)
2. 任意一条明细需要质检（B 是抽检）
3. 因此：整个订单被标记为"需要质检"
4. 因此：到货状态 = "待到货"
5. 结论：必须先转到货单，不能直接转入库
```

### 7.3 核心差异

| 维度 | 错误模型 | 实际系统 |
|-----|---------|---------|
| 判断粒度 | 明细级（看部分） | 订单级（看整体） |
| 逻辑关系 | "部分免检" → 推断可以直接入库 | "任意需要质检" → 整体需要质检流程 |
| 状态设置 | 可能认为有中间状态 | 只有 1 或 2（二选一） |
| 流程控制 | 可能支持部分入库 | 强制统一走质检流程 |

### 7.4 设计意图

系统这样设计的原因：
1. **流程简化**：避免部分入库、部分到货的复杂状态管理
2. **库存统一**：所有商品统一走到货流程，便于库存追踪
3. **合规要求**：只要有需要质检的商品，整个批次都必须走质检流程
4. **财务统一**：采购订单的财务处理基于整单状态

### 7.5 免检明细的实际处理

免检明细并不会被"忽略"，而是：
1. 跟随订单走"采购订单 → 到货单"流程
2. 在质检阶段被过滤（质检单只包含需要质检的明细，见 [PurchaseDeliveryServiceImpl.java#L359-L361](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L359-L361)）
3. 免检明细可独立从到货单转到采购入库单（来源类型=到货单），见 [PurchaseDeliveryServiceImpl.java#L401-L403](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L401-L403)

---

## 8. 状态变化阶段划分

### 8.1 Create/Update 阶段确定的状态

在创建/更新订单时（审批前）就确定的状态：

| 单据 | 字段 | 初始值 | 确定时机 | 代码位置 |
|-----|------|--------|---------|----------|
| 采购订单 | `qualityInspection` | 1(免检) 或 2(需要质检) | create/update 时 | [PurchaseOrderServiceImpl.java#L138-L143](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L138-L143) |
| 采购订单 | `otherState` | 1(无需到货) 或 2(待到货) | create/update 时 | [PurchaseOrderServiceImpl.java#L145-L150](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L145-L150) |
| 到货单 | `qualityInspection` | 1(免检) 或 2(需要质检) | create/update 时 | [PurchaseDeliveryServiceImpl.java#L148-L153](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L148-L153) |
| 到货单 | `otherState` | 1(无需入库) 或 2(待入库) | create/update 时 | [PurchaseDeliveryServiceImpl.java#L155-L162](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L155-L162) |

### 8.2 各审批阶段变化的状态

#### 8.2.1 采购订单审批成功后

文件位置：[PurchaseOrderServiceImpl.java#L412-L420](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L412-L420)

```java
public void approvalEndIsSuccess(PurchaseOrder entity) {
    entity = selectById(entity.getId());
    checkMaterialNorms(entity, true);  // 检查并更新来源单据状态
    // 增加在途库存
    entity.getErpOrderItemList().forEach(erpOrderItem -> {
        erpCommonService.editMaterialNormsDepotStock(MaterialNormsStockType.IN_TRANSIT_STOCK.getDefaultDepotId(), erpOrderItem.getMaterialId(),
            erpOrderItem.getNormsId(), erpOrderItem.getOperNumber(), DepotPutOutType.PUT.getKey(), MaterialNormsStockType.IN_TRANSIT_STOCK.getKey());
    });
}
```

**变化内容：**

| 变化内容 | 说明 | 代码位置 |
|---------|------|----------|
| 来源单据状态 | 如果订单来自合同/到货计划，更新来源单据的下达状态 | [PurchaseOrderServiceImpl.java#L172-L236](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L172-L236) |
| 在途库存 | 增加在途库存数量 | [PurchaseOrderServiceImpl.java#L416-L418](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L416-L418) |
| 订单主状态 | 从审批中变为 PASS（审批通过） | 框架层处理 |

**不变化的内容：**
- 采购订单的 `qualityInspection`
- 采购订单的 `otherState`

#### 8.2.2 到货单审批成功后

文件位置：[PurchaseDeliveryServiceImpl.java#L259-L262](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L259-L262)

**变化内容：**

| 变化内容 | 说明 | 代码位置 |
|---------|------|----------|
| 采购订单 `otherState` | 更新为：部分到货(3) 或 全部到货(4) | [PurchaseDeliveryServiceImpl.java#L250-L254](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L250-L254) |

**不变化的内容：**
- 采购订单的 `qualityInspection`

#### 8.2.3 质检单审批成功后

文件位置：[QualityInspectionServiceImpl.java#L301-L305](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L301-L305)

**变化内容：**

| 变化内容 | 说明 | 代码位置 |
|---------|------|----------|
| 到货单 `qualityInspection` | 更新为：部分质检完成(3) 或 全部质检完成(4) | [QualityInspectionServiceImpl.java#L251-L254](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L251-L254) |
| 采购订单 `qualityInspection` | 通过 `editQualityInspection` 同步更新为：部分质检完成(3) 或 全部质检完成(4) | [PurchaseDeliveryServiceImpl.java#L297-L300](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L297-L300) |

**关键发现：采购订单的 qualityInspection 在质检单审批后才会从 2 变为 3 或 4！**

#### 8.2.4 采购入库单审批成功后

文件位置：[PurchasePutServiceImpl.java#L276-L285](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L276-L285)

**变化内容（根据来源类型）：**

| 来源类型 | 变化内容 | 说明 | 代码位置 |
|---------|---------|------|----------|
| 1 (采购订单) | 采购订单状态 | 更新为：部分完成 或 全部完成 | [PurchasePutServiceImpl.java#L267-L271](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L267-L271) |
| 2 (质检单) | 质检单 `putState` | 更新为：部分入库 或 全部入库 | [PurchasePutServiceImpl.java#L243-L247](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L243-L247) |
| 3 (到货单) | 到货单 `otherState` | 更新为：部分入库 或 全部入库 | [PurchasePutServiceImpl.java#L213-L217](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L213-L217) |
| 所有类型 | 在途库存 | 减少在途库存 | [PurchasePutServiceImpl.java#L280-L283](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L280-L283) |

### 8.3 状态变化时间线

```
阶段1：创建/更新采购订单
    ├─ qualityInspection = 1 或 2
    ├─ otherState = 1 或 2
    └─ 【这两个状态在审批前已确定】

阶段2：采购订单审批成功
    ├─ 增加在途库存
    ├─ 更新来源单据状态
    └─ qualityInspection 和 otherState 不变

阶段3：转到货单 + 到货单审批成功
    ├─ 更新采购订单 otherState → 3(部分到货) 或 4(全部到货)
    └─ qualityInspection 仍为 1 或 2（不变）

阶段4：质检单审批成功
    ├─ 更新到货单 qualityInspection → 3(部分质检) 或 4(全部质检)
    ├─ 同步更新采购订单 qualityInspection → 3 或 4
    └─ 【采购订单 qualityInspection 首次变化！】

阶段5：采购入库单审批成功
    ├─ 根据来源类型更新对应单据状态
    ├─ 减少在途库存
    └─ 采购订单可能变为：部分完成 或 全部完成
```

---

## 9. CustomException 与 outputObject.setreturnMessage 的区别

### 9.1 CustomException

**特点：**
- 抛出运行时异常
- 会中断当前方法的执行
- 需要上层捕获处理或导致事务回滚
- 通常用于严重错误或业务规则违反

**使用场景（本代码中）：**

| 场景 | 方法 | 异常信息 | 代码位置 |
|-----|------|---------|----------|
| 订单需要质检时尝试直接转入库 | insertPurchaseOrderToTurnPut | "该订单需要进行质检，无法直接转采购入库，请先转【到货单】" | [PurchaseOrderServiceImpl.java#L302](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L302) |
| 订单免检时尝试转到货单 | insertPurchaseOrderToTurnDelivery | "该订单无需进行质检，请直接转采购入库" | [PurchaseOrderServiceImpl.java#L326](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L326) |
| 订单不存在 | 两个方法 | "该数据不存在" | [PurchaseOrderServiceImpl.java#L297](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L297) |
| 抽检比例为空 | setQualityInspection | "抽检比例不能为空" | [SkyeyeErpOrderServiceImpl.java#L232](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L232) |
| 抽检比例为0 | setQualityInspection | "抽检比例不能为0" | [SkyeyeErpOrderServiceImpl.java#L235](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L235) |
| 抽检比例>100 | setQualityInspection | "抽检比例不能大于100" | [SkyeyeErpOrderServiceImpl.java#L238](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L238) |
| 到货单无免检明细却转到入库 | checkAndUpdatePurchaseDeliveryPutState | "该到货单下未包含需要免检的商品，请走质检流程" | [PurchasePutServiceImpl.java#L202](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L202) |

### 9.2 outputObject.setreturnMessage

**特点：**
- 设置返回消息
- 不会中断方法执行
- 不会导致事务回滚
- 通常用于状态检查失败时的提示

**使用场景（本代码中）：**

| 场景 | 方法 | 返回消息 | 代码位置 |
|-----|------|---------|----------|
| 订单状态不是PASS或PARTIALLY_COMPLETED | insertPurchaseOrderToTurnPut | "状态错误，无法下达采购入库单" | [PurchaseOrderServiceImpl.java#L310](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L310) |
| 订单状态不是PASS或PARTIALLY_COMPLETED | insertPurchaseOrderToTurnDelivery | "状态错误，无法下达到货单" | [PurchaseOrderServiceImpl.java#L334](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L334) |
| 到货单状态不满足入库条件 | deliveryToPurchasePut | "状态错误，无法转采购入库单" | [PurchaseDeliveryServiceImpl.java#L431](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L431) |
| 到货单状态不满足质检条件 | deliveryToQualityInspection | "状态错误，无法下达质检单" | [PurchaseDeliveryServiceImpl.java#L389](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L389) |
| 质检单状态不满足入库条件 | qualityInspectionToPurchasePut | "状态错误，无法下达质检单" | [QualityInspectionServiceImpl.java#L437](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L437) |

---

## 10. 结论

### 10.1 核心结论

1. **订单级质检状态是"或"逻辑**：只要有一条明细需要质检，整个订单就被标记为"需要质检"

2. **到货状态与订单级质检状态强绑定**：
   - 订单需要质检 → 必须先转到货单
   - 订单免检 → 直接转入库

3. **部分质检的订单不能部分入库**：即使有免检明细，整个订单也必须走质检流程

4. **状态确定时机**：
   - `qualityInspection` 和 `otherState` 在 create/update 时确定
   - 采购订单审批不改变这两个状态
   - 到货单审批更新采购订单的 `otherState`（到货状态）
   - 质检单审批更新采购订单的 `qualityInspection`（从 2 变为 3 或 4）

5. **免检明细与质检明细分开入库**：
   - 免检明细 → 从到货单转采购入库单（fromTypeId=3）
   - 质检明细 → 从质检单转采购入库单（fromTypeId=2）

### 10.2 模型容易出错的地方

1. **粒度错误**：看明细级"部分免检"，而不是订单级"是否需要质检"
2. **逻辑错误**：认为"有免检明细就可以直接入库"，实际是"有需要质检的明细就必须走质检流程"
3. **状态理解错误**：可能认为有中间状态（如"部分免检"），实际只有二选一（1或2）
4. **流程理解错误**：可能认为免检明细可以独立流转，实际是整单统一流程后再分流

### 10.3 代码引用汇总

| 关键方法 | 文件位置 | 行号 |
|---------|---------|------|
| createPrepose | PurchaseOrderServiceImpl.java | [L115-L119](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L115-L119) |
| updatePrepose | PurchaseOrderServiceImpl.java | [L122-L125](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L122-L125) |
| setOtherMation | PurchaseOrderServiceImpl.java | [L136-L151](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L136-L151) |
| setQualityInspection | SkyeyeErpOrderServiceImpl.java | [L223-L245](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/business/service/impl/SkyeyeErpOrderServiceImpl.java#L223-L245) |
| insertPurchaseOrderToTurnPut | PurchaseOrderServiceImpl.java | [L292-L312](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L292-L312) |
| insertPurchaseOrderToTurnDelivery | PurchaseOrderServiceImpl.java | [L316-L336](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L316-L336) |
| queryPurchaseOrderTransById | PurchaseOrderServiceImpl.java | [L423-L447](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L423-L447) |
| approvalEndIsSuccess | PurchaseOrderServiceImpl.java | [L412-L420](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseOrderServiceImpl.java#L412-L420) |
| approvalEndIsSuccess | PurchaseDeliveryServiceImpl.java | [L259-L262](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L259-L262) |
| editQualityInspection | PurchaseDeliveryServiceImpl.java | [L265-L327](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L265-L327) |
| queryPurchaseDeliveryTransById | PurchaseDeliveryServiceImpl.java | [L352-L369](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L352-L369) |
| queryPurchaseDeliveryTransPurchasePutById | PurchaseDeliveryServiceImpl.java | [L394-L411](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L394-L411) |
| deliveryToPurchasePut | PurchaseDeliveryServiceImpl.java | [L414-L433](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchaseDeliveryServiceImpl.java#L414-L433) |
| approvalEndIsSuccess | QualityInspectionServiceImpl.java | [L301-L305](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L301-L305) |
| queryQualityInspectionTransById | QualityInspectionServiceImpl.java | [L398-L419](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L398-L419) |
| qualityInspectionToPurchasePut | QualityInspectionServiceImpl.java | [L422-L439](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/inspection/service/impl/QualityInspectionServiceImpl.java#L422-L439) |
| checkAndUpdatePurchaseDeliveryPutState | PurchasePutServiceImpl.java | [L192-L219](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L192-L219) |
| checkAndUpdateQualityInspectionPutState | PurchasePutServiceImpl.java | [L221-L249](file:///Users/zhangjing/Desktop/so-coders/so-coder-projects/0508/under/erp-pro/skyeye-erp/erp-pro/src/main/java/com/skyeye/purchase/service/impl/PurchasePutServiceImpl.java#L221-L249) |