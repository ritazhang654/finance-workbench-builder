# 批量导入与自动化任务模式

两类「数据从外部进入系统」的场景：A 报表导入（人触发、预览确认）、B 自动化任务（机器触发、无人值守）。两者都必须把「外部数据不可信」作为前提设计。

## 模式 A：Excel 报表批量导入（GOP 报表 / 门店档案 / 异常台账）

### 架构：客户端解析，后端落库

服务端不做文件解析（FaaS 不支持文件上传），Excel 在浏览器解析后按结构化 JSON 提交：

1. 前端用 xlsx 类库解析多 sheet 工作簿，逐 sheet 提取表头与数据行
2. **sheet 名 ↔ 档案名精确匹配**：以主数据表（如 `store_profile.storeName`）的唯一档案名为锚点
3. 匹配失败的处理是导入设计的核心（见下「防重复建档」）
4. 预览界面逐行展示：匹配结果、待写入字段、错误行（标红可跳过）
5. 确认后 POST `batch-import`，后端按月/行唯一索引 upsert，返回 `{ imported, skipped, errors }` 汇总

### 防重复建档（最大坑，必须做）

未匹配 sheet 强制建档 = 同名店复发（别名、旧名、错别字都会造出新店，数据静默挂错）：

```typescript
// 相似名提示：编辑距离/包含关系找疑似已有档案
function findSimilarStoreName(input: string, known: string[]): string | null {
  const hit = known.find(
    (name) => name.includes(input) || input.includes(name.replace(/酒店$/, ''))
  );
  return hit ?? null;
}
```

- 匹配到的 sheet：直接映射到已有档案
- 未匹配的 sheet：生成「新店草稿」，预览界面**默认不勾选**，勾选才建档——用户保留跳过手段
- 草稿建档时补全必填维度（城市档位、房量），并写外部编号到 remark 列作为后续匹配锚点

### 数据校验要点

- 金额从 Excel 读出后立即转 `Number` 校验，非法（NaN/负数/空）入错误行；落库走 numeric 列（后端字符串透传）
- 期间列统一 `YYYY-MM`；同一档案同一期间重复导入走 upsert（唯一索引兜底），不做追加
- 导入摘要逐字段回显（如「非客房成本 = 营业成本 ÷ 可售间夜」），让用户能核对口径

## 模式 B：自动化任务（同步 / 定时提醒 / 通知推送）

### 触发器启用顺序（顺序错了任务静默不跑）

cron 触发请求打到**线上常驻实例**，dev 沙箱不接收定时触发：

1. 代码提交并**发布**（确认发布 commit 包含自动化代码）
2. **ENABLE** 触发器
3. GET 回读 `Active=true`
4. 手动调一次任务接口验证线上 handler 存在

### 多维表格只读同步（外部数据 → 应用库）

以插件实例（只读权限）拉取外部多维表格记录，写入应用库：

```typescript
// 关键约束：时间窗口 + 同步日志去重 + 唯一键兜底
const WINDOW_DAYS = 30;
const records = await capability.load(BITABLE_INSTANCE_ID).call('list_records', {
  app_token, table_id,
  filter: { date_from: todayMinus(WINDOW_DAYS) }, // 窗口外不同步，防全量拉取
});

for (const rec of validRecords) {
  // data_sync_log 按 (module, record_id) 唯一，先查日志跳过已同步
  const synced = await db.select({ id: dataSyncLog.id })
    .from(dataSyncLog)
    .where(and(eq(dataSyncLog.module, 'energy_sync'), eq(dataSyncLog.recordId, rec.id)));
  if (synced.length > 0) continue;
  // 业务唯一键 upsert（onConflictDoNothing），重复推送静默幂等
  await db.insert(targetTable).values(mapFields(rec)).onConflictDoNothing();
  await db.insert(dataSyncLog).values({ module: 'energy_sync', recordId: rec.id, status: 'success' });
}
```

- 字段映射显式写映射表（外部字段名逐字对齐，单选字段读回是纯字符串选项名）
- 选项名到内部主数据的映射用显式常量表（键 = 外部编号），逐字一致才生效，同步时对缺映射记录打 warn 不中断
- 同步可手动触发（`POST .../sync/run`），供补数与排障

### 飞书卡片提醒（按人逐条发送）

每日定时向各责任人发消息，**按人逐条调发送插件**，一人失败不影响他人：

```typescript
const result = await capability.load(MESSAGE_INSTANCE_ID).call('send_feishu_message', {
  receiverIds: [miaodaUserId],   // 必须是妙搭 userId（纯数字），open_id 必报 10003
  content: buildCard(receiver),  // 每人一条，内容可个性化
});
```

- 接收人来自主数据列（如 `store_profile.meter_reader` 的 `user_profile` 类型，读取后取 userId）
- 消息里的表单/页面链接要**按人预填参数**：`<formUrl>?prefill_字段名=<encodeURIComponent(选项值)>`——预填参数名与目标表单问题名逐字一致，不匹配会静默不生效
- 卡片文案用业务语言（今日日期、门店名、操作入口按钮），来源署名指向应用首页
- 提供测试发送接口（`POST .../reminder/test`），支持可选 body 指定单人 + 指定业务对象，便于上线前验证与给用户演示，不必真等 cron

### 清库类接口的红线

凡命名含 reset/clear 的「测试数据清理」接口，在 dev 与线上同库部署形态下**全量真实生效**：只允许在用户明确授权后调用；接口文案与日志明示「全量清除，不可恢复」；有真实数据后优先禁用或加确认参数。
