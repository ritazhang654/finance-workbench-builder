# 后端模块模式（NestJS + Drizzle）

## 模块组织

每个业务模块固定三文件，放在 `server/modules/<kebab-name>/`：

```
server/modules/ota-reconciliation/
├── ota-reconciliation.module.ts    # @Module 声明，注册 controller 与 service
├── ota-reconciliation.controller.ts
├── ota-reconciliation.service.ts
└── ota-reconciliation.helpers.ts   # 可选：纯函数（金额汇总、状态映射）
```

命名约定：目录名/文件名 kebab-case，Module/Controller/Service 类名 PascalCase。模块在 `app.module.ts` 注册时必须排在 ViewModule 之前（ViewModule 是 fallback 路由，永远最后一项）。

## Controller 模板

```typescript
@Controller('api/ota-reconciliation')
export class OtaReconciliationController {
  constructor(private readonly svc: OtaReconciliationService) {}

  @Get()
  async list(@Query('status') status?: string) { /* 分页+筛选，返回 PageResult */ }

  @Get(':id')
  async detail(@Param('id') id: string) { /* 主记录 + 子表明细 */ }

  @NeedLogin()
  @Post()
  async create(@Req() req: Request, @Body() dto: CreateOtaBatchPayload) {}

  @NeedLogin()
  @Patch(':id/settle')
  async settle(@Param('id') id: string) { /* 状态流转：校验→更新→返回 */ }
}
```

要点：

- 前缀 `api/<kebab-name>`（对外接口才用 `openapi/`）
- 读接口不加鉴权装饰器；POST/PUT/PATCH/DELETE 按需加 `@NeedLogin()`（个人数据写入必加）
- 静态路由在动态 `/:id` 之前注册，避免路径冲突
- `@Query/@Param` 拿到的是 string，数字参数在 controller 层 `Number()` 转换后再进 service
- 状态流转子路径用动词：`:id/approve`、`:id/reject`、`:id/settle`、`:id/confirm`；子资源用 `:id/items`、`:id/items/:itemId`

## Service 核心模式

### 分页列表（条件动态拼装）

```typescript
const conditions: SQL[] = [];
if (status) conditions.push(eq(table.status, status));
const rows = await this.db.select().from(table)
  .where(conditions.length > 0 ? and(...conditions) : undefined)
  .orderBy(desc(table.createdAt))
  .limit(pageSize).offset((page - 1) * pageSize);
const [{ count: total }] = await this.db
  .select({ count: count() }).from(table)
  .where(conditions.length > 0 ? and(...conditions) : undefined);
```

`count()` 返回 string（PostgreSQL bigint），出口转 `Number(total)`。

### 状态流转（先校验后更新，写生效行数）

```typescript
async settle(id: string): Promise<void> {
  const [batch] = await this.db.select().from(otaReconciliation)
    .where(eq(otaReconciliation.id, id));
  if (!batch) throw new NotFoundException('对账批次不存在');
  if (batch.status !== 'diff_pending') {
    throw new BadRequestException('仅待核对批次可结算');
  }
  const updated = await this.db.update(otaReconciliation)
    .set({ status: 'settled', updatedAt: new Date() })
    .where(eq(otaReconciliation.id, id))
    .returning({ id: otaReconciliation.id });
  if (updated.length === 0) throw new NotFoundException('对账批次不存在');
}
```

### PATCH 更新契约

只写 DTO 中明确提供的字段，禁止 `dto.field ?? 默认值` 兜底——那会把用户没传的字段冲掉。用户要清空某字段时由 DTO 显式传空值。

```typescript
const patch: Partial<typeof table.$inferInsert> = {};
if (dto.title !== undefined) patch.title = dto.title;
if (Object.keys(patch).length === 0) throw new BadRequestException('未提供可更新字段');
patch.updatedAt = new Date();
```

注意：`Partial<$inferInsert>` 只用于 update 的 `set()`；create 直接构造完整对象。

### 详情聚合（禁止 N+1）

主记录 + 子表先收 id 再 `inArray` 一次查回，内存 Map 分组：

```typescript
const items = await this.db.select().from(itemTable)
  .where(inArray(itemTable.batchId, batches.map(b => b.id)));
const byBatch = new Map<string, Item[]>();
for (const it of items) {
  byBatch.set(it.batchId, [...(byBatch.get(it.batchId) ?? []), it]);
}
```

## 错误处理

- service 层抛 NestJS 内置 HttpException 子类，禁止裸 `throw new Error`（会被全局过滤器兜成 500，前端拿不到语义状态码）
- 场景映射：资源不存在/更新删除未命中 → `NotFoundException`；参数或状态非法 → `BadRequestException`；并发冲突/余额不足 → `ConflictException`；越权 → `ForbiddenException`
- controller 不 try/catch，异常交给全局 ExceptionFilter 统一包装成 `ApiErrorResponse`
- DTO 不引入 class-validator 时，在 service 入口手写校验（必填、长度、枚举值），非法抛 `BadRequestException`

## Drizzle 速查（易错点）

- `db` 从 `@Inject(DRIZZLE_DATABASE)` 注入，禁止自建连接
- 金额列 `numeric` 在 TS 侧是 string，直接透传，不要 `parseFloat`
- 时间列：`timestamp/timestamptz` 是 Date，service 出口 `.toISOString()`；`date` 列本身就是 string，禁止 `new Date()` 包装
- `user_profile` 列读写有专门三侧写法（读 `(col).user_id`、写 `ROW(${id})::user_profile`、过滤 `(col).user_id = ${id}`），raw SQL 里禁止裸列比较
- 多条写操作需要整体一致时用 `db.transaction` 包裹；单条语句不套事务
- 数值增减用原子 UPDATE（`set({ stock: sql\`col - ${qty}\` })` + WHERE 条件校验），禁止 SELECT→计算→UPDATE

## 日志

后端禁止 console，统一 `@nestjs/common` 的 Logger（无 info 方法，用 `logger.log`）；参数必须 string，对象 `JSON.stringify`。

## 相关参考

- 报表导入（batch-import）与自动化任务（同步/提醒/触发器）：`import-automation.md`
- 后端 @CanRole 兜底写接口、AuthorizationSDK 薄封装：`rbac-matrix.md`
- 复杂业务口径抽纯函数层（多模块共用唯一出处），参考模块目录的单房成本条目：`module-catalog.md`
