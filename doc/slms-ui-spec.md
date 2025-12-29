# File: slms-ui-spec.md
# Version: slms-suite-v2025.12.29
# Date: 2025-12-29

本文件定义页面结构、按钮出现条件、关键交互与规则。按模块可整段替换维护。

---

## 1. 页面：集装箱管理（container-management）

### 1.1 目标
- 统一查看箱号（含有效/作废/掏箱等历史）
- 提供受控操作：更正箱号、合并业务结果、掏箱重装登记、查看日志/照片
- 为“未匹配计划箱”提供快速入口（跳转到匹配工作台）

### 1.2 列表默认视图
- 默认仅显示：`container_effective_status = FINAL`
- 开关：显示历史（VOID / REPLACED / REPACKED_OUT）
- 默认过滤：`assignment_status` 可选（未匹配/已匹配）

### 1.3 列表字段建议
- 箱号、箱型、自重(final)、客户（若有）、Plan/Booking（若有）
- `assignment_status`
- `current_location_status + location`
- 管理标签（可隐藏给一线）：`created_source`、`unassigned_reason`

### 1.4 按钮与权限（简表）
- 【更正箱号】：DRAFT/CHECKED/CONFIRMED 可用；PACKING/PACKED 仅受控更正；CLOSED 仅管理员
- 【合并业务结果】：仅主管/管理员，且要求明确“目标箱号”
- 【掏箱重装】：PACKING/PACKED 可用（主管/管理员），写审计
- 【去匹配出运计划】：对 UNASSIGNED 一键跳转

---

## 2. 页面：出运计划匹配工作台（plan-container-match-workbench）

> 用途：同时看到“出运计划”和“未匹配箱（含先装箱/异常无计划箱）”，实现人工高效匹配。

### 2.1 页面布局
- 左：出运计划列表（Shipping Plan）
- 右：未匹配箱池（Containers: assignment_status = UNASSIGNED）
- 中：匹配/改绑/解绑（受控）

### 2.2 出运计划区（左）
**查询条件**：客户、ETD范围、船公司、型号（混装命中任一型号仍显示）、Booking No  
**列表字段**：booking_no、客户、ETD、船公司、计划箱量、已匹配箱量、缺口

### 2.3 未匹配箱池（右）
**查询条件**：箱型、位置、装箱完成时间范围；（管理员可见）未匹配原因  
**列表字段**：箱号、箱型、自重、当前位置、装箱完成时间、来源（created_source）

### 2.4 操作
- 【匹配】：
  - 选一个 plan + 选一个或多个箱 → 匹配
  - 系统写：container.shipping_plan_id、assignment_status=ASSIGNED
  - 写日志：ASSIGN_PLAN
  - 需要 CRP 时：生成/更新 CRP 并绑定箱（crp_container_link）
- 【改绑】：
  - 仅 UNASSIGNED/ASSIGNED 且未 LOCKED 的箱可改绑
  - assignment_status → REASSIGNED
  - 写日志：REASSIGN_PLAN（必填原因）
- 【解绑】：
  - 仅管理员且未出运前允许
  - assignment_status → UNASSIGNED
  - 写日志（必填原因）

---

## 3. 页面：CRP 录入与配箱（crp-entry-and-assign）

### 3.1 目标
- 录入/修改 CRP（booking_no、航次等）
- 配箱号：绑定箱号到 CRP（单箱或多箱/混装）
- booking 不一致时给提示，并提供跳转到“匹配工作台”做修复

### 3.2 核心规则（延续你已有设计）
- 保存前必须通过箱号检查（ISO 6346）并完成“是否已存在箱号记录”的确认
- 列表可滚动、选中项明显
- 录入提交后：按你既定规则清空/保留字段（以便连续录入）

---

## 4. 页面：直装匹配工作台（direct-loading-workbench）

### 4.1 布局
- 左：生产计划（含生产线、型号、时间、数量）
- 右：出运计划（按箱，含 booking、箱号、混装型号、计划装箱数量）
- 选中生产计划 → 右侧自动高亮可能匹配项
- 点击【生成】→ 左侧生产计划状态变化

### 4.2 查询条件
- 左：生产线、产品、型号、客户、生产日期
- 右：客户、ETD范围、船公司、型号（混装命中任一型号显示整箱）、Booking No

### 4.3 生成按钮行为（最小可用）
- 生产计划 status: NEW → GENERATED
- 记录关联：linked_booking_no / linked_container_id（或生成 packing_task）
- 更新右侧“已匹配数量”
- 写日志（建议 task_status_event）

---

## 5. 入口拆分（不让用户选装箱类型）

### 5.1 本作业入口（PLAN_ENTRY）
- 必须先选出运计划/订单
- 创建的箱默认 ASSIGNED（如后续变 UNASSIGNED → 标记异常）

### 5.2 先装箱入口（PREPACK_ENTRY）
- 允许无计划
- 创建的箱默认 UNASSIGNED（正常）
- 后续通过“匹配工作台”关联到计划，生成 CRP（无需提箱环节）

---

## 6. 版本与同步维护要求

- 每次修改必须同步更新三件套：schema / data-dict / ui-spec
- 以最新三件套为唯一真源
