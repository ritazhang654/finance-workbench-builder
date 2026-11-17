# 财务域模块目录（已验证的 9 个模块模板）

每个模块给出：主表关键字段（系统字段 _created_at/_created_by/_updated_at/_updated_by 省略）→ 端点清单 → 页面要点。新需求先找同构模块，替换领域字段即可复用整套骨架。所有金额列 numeric、状态列 varchar 带 default。

## 1. 财务总览 Dashboard（路由 `/`）

不建独立主表，聚合其他模块数据。端点：GET `/api/dashboard/summary`（KPI 汇总）、GET `/api/dashboard/pending`（待办）、GET `/api/dashboard/trend`（图表数据）。

页面：KPI 卡片行（`font-mono text-3xl font-bold text-right`）+ 待办列表 + 趋势图表（ReactECharts，先调 charts-skill）。KPI 数值如「本月报销总额」「待审批笔数」「超期 AR 金额」。

## 2. 费用报销 Expenses（`/expenses`）

表 `expense`：applicant `user_profile`（报销人）、category varchar、amount numeric、occurredAt date、description text、status default `'pending'`（pending/approved/rejected）、rejectReason text（nullable）。

端点：GET 分页列表（status 筛选）、POST 创建（`@NeedLogin`，applicant 从 `req.userContext.userId` 取，禁止前端传）、PATCH `:id/approve`、PATCH `:id/reject`（body 必填 rejectReason）。

页面：审批工作台形态——状态筛选 tab、行内通过/驳回按钮、驳回弹窗填必填原因。列表需展示报销人（用 business-ui 的 UserDisplay，禁止裸 userId）。

## 3. 付款申请 Payment Requests（`/payments`）

表 `payment_request`：payee varchar（收款方）、amount、purpose text、expectedDate date、status default `'pending'`（pending/approved/paid/rejected）、rejectReason。

端点：GET 列表、POST 创建、PATCH `:id/approve`、PATCH `:id/reject`、PATCH `:id/pay`（标记已付款）。三次状态流转接口共享「先查现态→校验→更新」模式。

## 4. 发票管理 Invoices（`/invoices`）

表 `invoice`：invoiceNo varchar `uniqueIndex`（发票号唯一，重复录入靠 23505 唯一约束拦截）、category varchar、amount numeric、issueDate date。

端点：GET 列表、GET `:id`、POST 创建（invoiceNo 重复返回 409 Conflict，用 cause-aware SQLSTATE 提取判断 23505）。

页面：录入表单 + 列表，重号时 toast 明确提示「发票号已存在」。

## 5. 预算执行 Budgets（`/budgets`）

表 `budget`：name varchar、department varchar、amount numeric、usedAmount numeric default `'0'`、period varchar（如 2026-Q3）。

端点：GET 列表、POST 创建、PATCH `:id`（编辑基本字段）、GET `/api/dashboard` 复用其数据算执行率。

页面核心是**执行率预警**：`usedAmount / amount ≥ 90%` 的行 `bg-warning/15` 或整行琥珀色高亮；执行率列展示百分比进度条。金额与百分比右对齐。

## 6. S 账往来 S-Accounts（`/s-accounts`）

表 `s_account`：counterparty varchar（往来方）、amount numeric（正数=应收、负数=应付）、businessDate date、remark text、status default `'unsettled'`（unsettled/settled）。

端点：GET 列表（含账龄计算）、POST 创建、PATCH `:id/settle`。

页面核心是**双向超期规则**：负数挂账超 7 天 → 红色（`text-destructive`）；正数挂账超 30 天 → 琥珀（`text-warning`）；超期天数列 mono 右对齐。

## 7. AR 应收账龄 AR-Accounts（`/ar-accounts`）

表 `ar_account`：customer varchar、amount numeric、occurredAt date、status default `'uncollected'`（uncollected/collected）。

端点：GET 列表（含账龄分桶统计：30/60/90 天）、POST 创建、PATCH `:id/collect`（回款核销）。

页面核心是**账龄分桶**：90 天+ 红色整行高亮（左边框 3px）+ 顶部账龄分布 KPI；回款操作行内按钮。

## 8. 异常操作监控 Anomalies（`/anomalies`）

表 `anomaly_record`：opType varchar（操作类型）、operator varchar（操作人）、content text、occurredAt timestamptz、riskLevel varchar（high/medium/low）、status default `'unprocessed'`、handleNote text。

端点：GET 列表（riskLevel/status 筛选）、GET 统计（各风险等级计数）、PATCH `:id/handle`（body 必填 handleNote，标记已处理）。

页面：风险等级色标签（high 红/medium 琥珀/low 中性）、未处理置顶、处理动作弹窗填备注。

## 9. OTA 对账 Ota-Reconciliation（`/ota-reconciliation`）⭐ 最复杂，主从双表

表 `ota_reconciliation`（批次）：otaPlatform varchar、periodStart/periodEnd date、status default `'diff_pending'`（diff_pending/settled）、hotelCount/hotelAmount（酒店侧笔数金额）、otaCount/otaAmount（OTA 侧）、matchedCount/diffCount、roomPayable/commissionPayable numeric。

表 `ota_reconciliation_item`（明细，`index(batchId)`）：batchId uuid、orderNo varchar、checkInDate date、hotelAmount/otaRoomAmount/otaCommission numeric、roomDiff numeric（差异额）、matchStatus varchar（matched/diff）、resolutionNote text（差异处理备注）。

端点：GET 批次分页列表、GET `:id`（批次+明细聚合，inArray 防 N+1）、POST 创建批次（带明细数组，事务写入）、PATCH `:id/settle`（结算，前置校验 status=diff_pending）、PATCH `items/:itemId`（明细备注维护）。

页面：批次列表（状态/金额汇总/差异计数）→ 详情页（明细表：匹配行中性、差异行红色高亮 + roomDiff mono 显示差额）→ 差异处理（逐单填 resolutionNote）→ 全部差异处理完结算。这是「主从表 + 差异核对 + 状态闭环」的完整范本。

## 10. 门店档案 Stores（`/stores`）——运营域的基础主数据

表 `store_profile`：storeName varchar `uniqueIndex`（档案名，导入匹配的锚点）、cityTier varchar（tier_1/2/3，影响成本模型档位）、rooms integer（房量，成本分母）、remark text（存门店编号 UC**** 等外部编号）、manager `user_profile`（店长，门店隔离依据）、meterReader `user_profile`（抄表负责人，提醒发送对象）、status varchar default `'offline'`（online/offline 上下线）。

端点：GET 分页列表（含 `?status=` 筛选）、GET `list`（全量轻量列表供下拉）、POST `batch-import`（Excel 批量建档）、PUT `:id`（编辑/上下线，复用同一 PATCH 契约）。

页面：全量档案表（含搜索）、编辑弹窗（含上线/下线快捷按钮）、批量导入弹窗（预览 + 新店草稿勾选，默认不勾跳过，见 import-automation 模式 A）。**其他模块按业务域拉 online 子集**（看板/筛选），档案页显示全量供管理。

## 11. 能耗管理 Energy（`/energy` + `/group-dashboard`）——抄表闭环 + 多维表格同步 + 飞书提醒

表 `energy_meter_reading`：recordDate date、energyType varchar（electric/water/gas）、reading numeric（表底读数，用量由相邻读数差算）、meterName varchar（`''`=合计，非空=逐表）、source varchar（manual/bitable）、storeId uuid，唯一索引 `(store_id, record_date, energy_type, meter_name)` 强制同店同日同类型「合计与逐表互斥」。
表 `energy_price`：energyType varchar `uniqueIndex`、price numeric、unit varchar；可加 sewage 行存排污系数（默认 0.81）。

端点：GET/PUT `prices`（单价维护）、GET 分页 `readings`（store/date/type 筛选）、POST/PUT/DELETE `readings`（写入时触发排污费自动计算：当月水用量×系数×水单价，写入月度成本表）、GET `analysis`（按月/按日聚合）、POST `automation/sync/run`（手动同步）、POST `automation/reminder/test`（测试提醒，可选 body 指定单人）。

自动化（见 import-automation 模式 B）：cron 每日从多维表格只读同步读数（30 天窗口 + data_sync_log 去重）→ 每日 17:00 向各店抄表负责人发飞书卡片（预填门店选项与当天日期的表单链接）。

页面：抄表记录表（录入弹窗可选表名列，导入冲突提示）+ 单价设置卡 + 用量/费用分析图表；集团看板页展示能耗排名、提交监控。

## 12. 单房成本 Room Cost（`/room-cost` + `/room-cost-analysis`）——公式引擎 + GOP 导入

表 `room_cost_model`：version/cityTier/roomScale `uniqueIndex`（三键定位模型档位），9 科目成本列 numeric + occupancyRate/revpar 假设列，totalOperation/netRoomCost 为派生列。
表 `store_cost_monthly`：storeId+month `uniqueIndex`，9 科目实际成本列 + 剔除金额列（secondaryServiceCost 等 5 列，净额口径存原值）+ soldRoomNights/availableRoomNights/营收/GOP/预算列。

**公式唯一出处**：抽成纯函数文件（如 `room-cost-formula.ts`，输入 Amounts 接口兼容 number|string，输出单房成本），store-cost / room-cost / group-dashboard 三处共用——禁止在 service 或前端重复实现公式。核心口径：客房成本与销售费用 ÷ 已售间夜，其余科目 ÷ 可售间夜（缺省按房量×当月天数）；运营成本 = 9 科目之和，净客房 = 运营 − 非客房 − 其他（净额）。

端点：GET/POST/PATCH `api/room-cost`（模型维护）、GET `api/room-cost-analysis`（某店逐月分析）、`api/store-cost-monthly` 的 GET/POST/`batch-import`/`heating-season`（供暖费均摊）/PATCH/DELETE、`api/group-dashboard` 的 latest-month/energy-ranking/cost-ranking/submission-monitor。

GOP 报表导入走客户端解析（见 import-automation 模式 A）：多 sheet Excel → sheet 名匹配门店档案 → 逐月写入 store_cost_monthly。

## 13. 权限治理（`/role-management` + `/permission-config`）——角色管理 + 可配置页面矩阵

角色成员管理：**不自建角色表**，走平台 AuthorizationSDK（roles.list/create/update/delete + members.add/remove/clear + search），后端 `api/role_manager` 只做薄封装；bizID 仅字母/数字/下划线（平台白名单），名称可中文。

页面权限矩阵：表 `page_permission_config`（pageKey + roleBizId `uniqueIndex`，canView/canWrite boolean）。端点：GET/PUT `api/role_manager/permission_config`（写仅管理员）。页面：矩阵编辑表（角色 × 页面开关）。

前端守卫与三层模型、admin 企业全员陷阱、新增页面登记三处，详见 `references/rbac-matrix.md`。

## 通用形态速查

| 需求形态 | 复用模块 | 核心机制 |
|---|---|---|
| 审批流（通过/驳回+原因） | Expenses/Payments | PATCH `:id/approve`/`:id/reject`，rejectReason 必填 |
| 唯一单证录入 | Invoices | uniqueIndex + 23505 → 409 |
| 阈值预警 | Budgets | 比率计算 → 行级 warning 高亮 |
| 账龄/时效分桶 | S-Accounts/AR | 日期差分桶 → 红/琥珀分级高亮 |
| 主从核对结算 | OTA 对账 | 双表事务创建 + 明细聚合 + 状态机结算 |
| 风险事件台账 | Anomalies | 等级标签 + 处理闭环 |
| 主数据 + 批量建档 | Stores | 唯一档案名 + 导入草稿勾选 + 上下线状态 |
| 外部数据源同步 | Energy | 只读插件 + 时间窗口 + 同步日志去重 + cron 提醒 |
| 公式引擎 + 报表导入 | Room Cost | 纯函数唯一出处 + sheet 匹配 + 月度唯一索引 |
| 权限随配置变化 | Role Mgmt/Perm Config | AuthorizationSDK + 矩阵表 + 三层守卫 |
