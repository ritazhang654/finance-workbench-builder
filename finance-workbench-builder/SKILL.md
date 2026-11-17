---
name: finance-workbench-builder
description: 构建企业级财务/业务管理工作台的完整方法论——模块划分、数据库表设计、共享类型、NestJS 后端模块、React 前端页面、财务风控设计规范、测试验收的端到端流程；另含运营域进阶模式：门店档案管理、能耗抄表与多维表格同步、飞书消息提醒自动化、Excel 报表导入建档、单房成本公式引擎、角色成员管理与可配置页面权限矩阵。当用户要搭建财务工作台、资金管理、对账系统、报销审批、付款申请、发票管理、预算控制、账龄管理、异常操作监控、经营数据看板、门店档案、能耗管理、抄表、单房成本、成本模型、GOP 报表导入、角色权限管理、页面权限配置，或任何「多模块 + 数据表格 + 状态流转 + 风险预警」形态的企业业务系统时，务必使用本 skill——即使用户只说「做个管理系统」「搭个工作台」「把这几张表管理起来」也要触发。已在一个酒店集团运营工作台（16 个导航模块、16 张数据库表）上完整验证。
---

# 财务/业务工作台构建方法论

本 skill 沉淀自一个真实交付的酒店集团运营工作台：16 个导航模块、16 张数据库表，覆盖财务资金域（审批流转、对账、账龄预警）与运营域（门店档案、能耗抄表、单房成本、集团看板）及权限治理（角色成员管理、可配置页面权限矩阵）。核心价值是把「多模块业务系统」的搭建变成可复制的流水线：同样的模块骨架、同样的代码模式、同样的设计语言，只换领域字段。

模块分三个域：

- **财务资金域（9 模块）**：财务总览、费用报销、付款申请、发票管理、预算管理、S 账、AR 账、异常操作监控、OTA 对账——审批流、唯一单证、阈值预警、账龄分桶、主从核对五类通用形态
- **运营域（4 组）**：门店档案（80+ 店批量导入与上下线）、能耗管理（逐表抄表 + 多维表格同步 + 飞书每日提醒）、单房成本（9 科目公式引擎 + GOP 报表导入 + 集团看板）、通用报表看板
- **权限治理（2 模块）**：角色成员管理（平台 AuthorizationSDK）、页面权限配置（「角色 × 页面」矩阵持久化）

## 总体架构与数据流

数据流向单向：DDL 建表 → schema.ts 自动生成 → shared 类型 → 后端模块 → 前端页面。严格按此顺序推进，任何一步跳过都会在下一步还债：

```
需求 → 模块划分
     → 数据库表（DDL，系统字段自动生成）
     → shared/api.interface.ts（类型契约，前后端唯一事实）
     → server/modules/<name>/ 三文件
     → 聚合文件预填（app.module.ts / app.tsx / api/index.ts）
     → client/src/pages/<Name>Page/ 四文件
     → 测试验收（API 自测 + E2E）
```

## 八步搭建流程

### Step 1 模块划分

把用户需求转成领域模块清单。判据：一个模块 = 一张主表 + 一组围绕它的操作 + 一个侧边栏入口。经验阈值：8 个左右模块为一个工作台的合理规模，超过时考虑合并相近领域（如报销与付款可分开但共享审批模式）。

典型模块见 `references/module-catalog.md`——13 个模块条目（财务域 9 + 运营域 4）的表结构、端点、页面要点全在里面，新需求先在里面找同构模板再定制。

### Step 2 数据库表设计

- DDL 通过数据库操作 skill（lark-apps-db 的 `+db-execute`）执行，禁止裸连数据库；执行后系统自动生成 `server/database/schema.ts`，**必须重新读取**再写业务代码
- 金额列一律 `numeric`（精确十进制），禁止 float；计数列 `integer`；状态列 `varchar` 带 `not null default '<初始状态>'`（如 `'pending'`、`'diff_pending'`）
- 系统字段 `_created_at/_created_by/_updated_at/_updated_by` 由平台自动注入，表定义里声明即可，业务代码不写、不更新（更新契约见后端模式）
- 唯一业务键加 `uniqueIndex`（如发票号 `idx_invoice_invoice_no`），高频外键查询加普通 index（如对账明细的 batch_id）
- 用户引用列用 `user_profile` 类型（如报销的 applicant），它的读写有专门写法，见平台 coding-guide

### Step 3 共享类型先行

`shared/api.interface.ts` 是前后端唯一契约，**先写类型再写任何实现**：

- 顶部定义通用 `PageParams`（page/pageSize）与 `PageResult<T>`（items/total/page/pageSize），所有列表接口复用
- 按模块用 `/* ========== 模块名 ========== */` 注释分段，段内放 CreateXxxPayload、XxxSummary（列表行）、XxxDetail（详情含子表）
- 属性一律 camelCase，数据库 snake_case 不泄漏到接口；时间字段统一 `string`（服务端出口 `.toISOString()`，date 列原样透传）

### Step 4 后端模块

每个模块固定三文件（复杂模块加 helpers 放纯函数），模板与代码骨架见 `references/backend-patterns.md`。核心约束：

- Controller 前缀 `api/<kebab-name>`，端点覆盖：GET 分页列表（筛选参数走 Query）、GET 详情、POST 创建、PATCH `:id/<action>` 状态流转
- 写操作加 `@NeedLogin()`；DTO 用 interface + 手写校验抛 `BadRequestException`（本项目不引入 class-validator，保持轻量）
- 状态流转必须先校验当前状态是否允许转换，非法转换抛 400；资源不存在抛 `NotFoundException`
- 分页用 Drizzle `limit/offset` + 二次 `select({ count: count() })` 拿 total；条件用 `and(...conditions)` 数组动态拼装，禁止 `let query` 重赋值

### Step 5 聚合文件预填（关键调度动作）

三个聚合文件——`server/app.module.ts`、`client/src/app.tsx`、`client/src/api/index.ts`——必须由主 agent 在派发业务模块任务**之前一次性串行预填**，业务模块任务不得编辑它们（并发编辑同一聚合文件是冲突与启动失败的头号根因）。ViewModule 必须保持在 app.module.ts imports 数组最后一项。

### Step 6 前端页面

每个模块一个页面目录，固定四文件拆分，模板见 `references/frontend-patterns.md`：

| 文件 | 职责 | 规模 |
|---|---|---|
| `XxxPage.tsx` | 容器：state + 取数 + 布局 + 弹窗开关 | 70–200 行 |
| `XxxTable.tsx` | antd-table 列定义与渲染 | 100–180 行 |
| `XxxFormDialog.tsx` | 新建/编辑弹窗（react-hook-form + zod） | 100–200 行 |
| `xxx-helpers.ts` | 金额格式化、状态颜色映射等纯函数 | <100 行 |

侧边栏导航在 `Layout.tsx` 的 NAV_ITEMS 配置，与 app.tsx 路由一一对应；首页路由必须是 `/` 的 index。

### Step 7 设计规范落地

财务工作台的设计语言是「冷峻商务蓝 + Grid 网格 + 双警示色体系」——数据排版优先于装饰，风险色克制使用。完整 token 与 tailwind 类名对照见 `references/design-tokens.md`。识别签名（保持一致性的关键）：

- 金额列：`font-mono tabular-nums text-right`，KPI 数字 `font-mono text-3xl font-bold`
- 高风险整行：`bg-destructive/5` + 左侧 3px `border-l-destructive`
- 状态胶囊：语义色 10% 背景 + 40% 边框 + 同色文字
- 卡片 `rounded-sm shadow-none`，用 1px border 替代阴影

### Step 8 测试验收

- API 自测：写接口用 `[API_TEST]` 标记的最小合法数据完成「写入 → 读回 → 状态变更 → 清理」闭环；只删自建数据
- 状态流转接口至少验证：正常流转一次、重复流转/非法状态被拒（非 500）
- 前端验收用 E2E agent 走一遍核心链路（创建 → 核对/审批 → 终态）
- 残留测试数据无法清理时，在交付说明里声明标识与原因

## 常见坑（本项目踩过或规避过的）

- **先写代码后建表**：schema.ts 是自动生成的，跳过「DDL → 重读 schema」直接编码是 TS2339 批量报错的头号根因
- **聚合文件并发编辑**：多模块并行开发时必须先预填聚合文件，否则 old_string mismatch 导致 LLM 兜底重写
- **PATCH 覆盖未提供字段**：状态更新只写 DTO 中明确提供的字段，`dto.field ?? 默认值` 会把用户没传的字段冲掉
- **状态机不设防**：没有前置状态校验的流转接口，前端重复点击就产生脏数据
- **金额用 float**：对账场景 0.1+0.2 误差直接导致差异误报，必须 numeric + 后端字符串透传、前端统一格式化
- **警示色滥用**：红色只在「超期/高危/驳回」类场景出现，日常状态中性色，否则整个工作台的预警会失去意义
- **报表导入同名店复发**：Excel sheet 名与档案名不一致（别名、旧名、错别字）会被当新店建档，数据挂错店且难察觉——必须做精确匹配 + 相似名提示 + 未匹配默认跳过
- **「重置测试数据」接口清掉真实数据**：清库类接口在 dev 与线上同库时全量生效，只允许在用户明确授权后调用，且接口命名与文案要明示「全量清除」
- **定时任务只提交不发布 = 静默不跑**：cron 触发器打到线上常驻实例，部署必须走「发布 → ENABLE → 回读 Active=true」顺序，dev 沙箱不接收定时触发
- **内置 admin 角色含企业全员**：平台预置 admin 角色不可移除「企业全员」，用它鉴权等于全员管理员——管理权限必须挂到显式成员的独立角色（如 super_admin）

## 运营域进阶模式（references 详解）

基础八步覆盖财务资金域即可交付；需求涉及以下三类场景时，在对应步骤套用进阶模式：

- **批量报表导入**（GOP 报表、门店档案、异常台账）：客户端解析 Excel → sheet 与档案名精确匹配 + 相似名提示 → 未匹配店默认跳过（防重复建档）→ 预览确认 → 后端 batch-import。详见 `references/import-automation.md` 模式 A
- **自动化任务**（多维表格同步、定时提醒、通知推送）：cron 触发器启用顺序（发布 → ENABLE → 回读）、只读插件同步（30 天窗口 + 唯一键 upsert + 同步日志去重）、飞书卡片按人逐条发送（接收人传妙搭 userId、表单链接 encodeURIComponent 预填）。详见 `references/import-automation.md` 模式 B
- **可配置权限矩阵**（页面可见/可写随管理员配置变化）：`page_permission_config` 表 + 前端 usePagePermission 守卫 + 后端 @CanRole 兜底的三层模型。详见 `references/rbac-matrix.md`

## 交付前检查单

- [ ] 每个模块：表 → shared 类型 → 后端三文件 → 前端四文件，链路完整
- [ ] 聚合文件预填完成，ViewModule 在 imports 最后
- [ ] 所有金额列右对齐等宽字体，日期 YYYY-MM-DD
- [ ] 状态标签统一胶囊形，语义色符合 design-tokens
- [ ] API 自测通过，无 500；测试数据已清理或已声明
- [ ] 侧边栏导航与路由一一对应，含 `/` index 路由
- [ ] 导入类接口有重复/冲突防护（唯一约束 + 相似名提示），自动化任务已发布并回读 Active
- [ ] 新增页面已登记权限矩阵三处（导航、种子数据、默认矩阵），后端写接口有 @CanRole 兜底
