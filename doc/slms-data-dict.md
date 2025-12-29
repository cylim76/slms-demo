# File: slms-data-dict.md
# Version: slms-suite-v2025.12.29
# Date: 2025-12-29

本文件解释 **字段口径/业务含义/统计规则**。

---

## 0. 总体原则（必须）

1. **箱号身份**、**当前位置**、**任务进度**是三种不同维度：  
   - 箱号身份：`container.container_status / container.container_effective_status`  
   - 当前位置：`container.current_location_*`  
   - 作业进度：`packing_task.task_status`（以及未来摆位/回收类任务）

2. **错误录入允许纠错，但必须留痕**：任何更正/合并/作废/掏箱/改绑计划必须写 `container_change_log`。

3. **统计只认最终有效箱号**：仅 `container_effective_status = FINAL` 进入统计/导出/结算。

4. **发生过真实装箱行为的箱，不允许用“直接改箱号文本”覆盖历史**。需要走更正/合并/掏箱流程。

---

## 1. 集装箱身份状态（container_status）

- **DRAFT**：草稿/临时，未完成箱号校验，不能进入装箱/出运。
- **CHECKED**：箱号校验通过（算法 + 查重确认）。
- **CONFIRMED**：司机已确认（关键节点）。
- **PACKING_IN_PROGRESS**：装箱中（真实绑定已发生）。
- **PACKED**：装箱完成（历史事实必须保留）。
- **REPACKED_OUT**：已掏箱/转移（原箱发生过装箱但已不作为最终出货箱）。
- **CLOSED**：出运/对账后锁定（只读）。
- **VOID**：作废（错误箱号/被替代）。不可删除、不可统计，但可追溯。

### 1.1 统计口径
- 出货统计：只统计 `container_effective_status = FINAL` 且 `container_status` 不为 VOID/REPACKED_OUT。

---

## 2. 最终有效性（container_effective_status）

- **FINAL**：最终有效箱号（全局唯一进入统计）。
- **REPLACED**：被其他箱号替代（历史保留）。
- **INVALID**：确认错误（历史保留）。

---

## 3. 未匹配计划统一管理（assignment_status）

你希望**不让用户选择“装箱类型”**，因此用“是否匹配出运计划/订单”统一管理：

- `assignment_status`：UNASSIGNED / ASSIGNED / REASSIGNED / LOCKED  
- `shipping_plan_id`、`order_id`：可为空（先装箱/异常时）

### 3.1 业务含义
- **UNASSIGNED**：未匹配计划/订单（统一管理“先装箱”与“booking录错造成无计划”）。
- **ASSIGNED**：已匹配。
- **REASSIGNED**：改绑过（必须留痕）。
- **LOCKED**：出运/对账后锁定。

### 3.2 未匹配原因（unassigned_reason，系统写入）
- **PREPACK_LIKELY**：先装箱入口创建、或装箱完成时没有计划关联。
- **BOOKING_MISMATCH**：本应有计划但 booking/关联不一致导致“看起来无计划”。
- **PLAN_NOT_IMPORTED**：计划未导入/延迟导入。
- **DATA_ERROR / UNKNOWN**：其他。

> 说明：该字段可对一线隐藏，仅供主管/管理员分析与治理。

### 3.3 入口自动区分（created_source）
- **PLAN_ENTRY**：本作业入口（必须绑定出运计划/订单）
- **PREPACK_ENTRY**：先装箱入口（允许无计划）
- **DIRECT_ENTRY**：直装工作台生成
- **IMPORT**：导入/接口

---

## 4. 当前位置（current_location_*）

- `current_location_id`：当前位置/工位。
- `current_location_status`：OUTSIDE_FACTORY / IN_FACTORY / AT_WORKSITE / IN_TRANSIT_INSIDE / LEFT_FACTORY  
- 轨迹：`container_location_event`（用于等待时间/搬运次数/停留时长等统计）

> 位置字段仅回答“现在在哪”，不等同于任务进度。

---

## 5. 装箱任务进度（packing_task.task_status）

- **PRE_CHECK**：装箱前检查
- **READY**：待装箱
- **PACKING**：装箱中
- **PAUSED**：暂停装箱（可恢复）
- **PACKED**：装箱完成
- **CANCELLED**：取消（不进入统计）

---

## 6. 关键异常处理口径

### 6.1 箱号更正（Correct）
适用：箱号录错但希望保留已完成业务结果（检查/自重/照片等）。  
要求：
- 写 `container_change_log(change_type=CORRECT_CONTAINER_NO)`
- 更正后必须重新执行“箱号校验+查重确认”
- 已锁定（CLOSED/LOCKED）需管理员/审批

### 6.2 业务结果合并（Merge Business Results）
适用：同一真实箱产生两条记录：一条有检查信息，另一条有自重/司机确认等。  
要求：
- 以“最终箱号”为目标，将业务结果 rebind
- 被合并记录置为 VOID/REPLACED
- 写 `container_change_log(change_type=MERGE_BUSINESS_RESULTS)`

### 6.3 掏箱重装（Repacking）
适用：货物从箱 A 全部掏出并转移至箱 B。  
要求：
- A：`container_status = REPACKED_OUT`（历史保留）
- B：继续作为最终箱匹配出运计划
- 写 `container_change_log(change_type=REPACKING)`，建议附照片证据

---

## 7. CRP 与 booking 错误的处理口径

- CRP 是执行侧单据，`booking_no` 必须存在。
- 当出现“CRP booking 录错，但提箱/实际正确”的情况，本质是 **关联不一致**：
  - 在“出运计划匹配工作台”修复 `shipping_plan_id`（或改绑），并写日志（ASSIGN_PLAN/REASSIGN_PLAN）
  - 必要时对 CRP 做更正（保留审计）

---

## 8. 直装（Direct Loading）口径

- 直装通过“生产计划 ↔ 出运计划（按箱）”匹配生成：`created_source = DIRECT_ENTRY`。
- 型号搜索支持混装：命中混装型号任一项 → 显示整箱明细。
- 点击“生成直装”后：生产计划 `status = GENERATED`，并记录关联（booking/箱/任务）。
