# 可配置权限矩阵模式（角色 × 页面）

需求场景：管理员要能随时调整「哪个角色能看/能写哪个页面」，而不是改代码发版。核心是**三层模型**：前端矩阵控制导航与路由、后端 @CanRole 兜底写接口、矩阵持久化到表里可配置。

## 三层模型

```
page_permission_config 表（事实源）
   ↓ GET（登录后拉取，模块级缓存）
前端 usePagePermission —— 控制导航项过滤、路由守卫、页面内只读态
   ↓ 写请求仍会被后端拦截
后端 @CanRole([roles]) —— 写接口硬校验，防前端被绕过
```

- **后端是底座**：@CanRole 只加写接口（POST/PUT/PATCH/DELETE），GET 一律不加（游客浏览只读）
- **前端是体验层**：矩阵决定导航显隐与路由跳转，被拦截的写操作在 UI 上呈现为只读态而非报错
- **表是可配置层**：管理员改矩阵即时生效（前端保存后同步缓存），接口失败回退内置默认矩阵保证可用

## 矩阵表设计

```sql
CREATE TABLE page_permission_config (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  page_key varchar(100) NOT NULL,        -- 页面标识（与路由一致，如 'energy'）
  role_biz_id varchar(50) NOT NULL,      -- 角色 bizID（平台角色标识）
  can_view boolean NOT NULL DEFAULT false,
  can_write boolean NOT NULL DEFAULT false,
  -- 系统字段省略
);
CREATE UNIQUE INDEX idx_page_perm_page_role ON page_permission_config (page_key, role_biz_id);
```

端点：`GET /api/<module>/permission_config`（登录即可读）、`PUT` 同路径（仅管理员，全量替换保存）。

## 前端守卫要点（client/src/hooks/page-permissions.ts）

```typescript
// 模块级缓存：登录后拉一次，保存矩阵后同步更新
let cachedEntries: PagePermEntry[] | null = null;

export function usePagePermission(pageKey: string): { canView: boolean; canWrite: boolean; readOnly: boolean } {
  // 1. 管理员角色硬编码恒全量（防自锁：把唯一管理员锁出系统是灾难）
  // 2. 未登录（游客）放行 canView=true + readOnly=true —— 预览/分享场景只读浏览
  // 3. 已登录查矩阵；矩阵为空或接口失败 → 回退内置 DEFAULT_ENTRIES
}
```

- **游客放行只读是硬规则**：预览环境与分享链接无登录态，`useCurrentUserProfile` 无 user_id；按角色矩阵判定会把游客全拦死（应用「全挂」的头号误修）
- 角色集合用**正向判断**：仅显式声明门店维度的角色才做数据过滤，不要「非白名单即受限」的反向推断——请求上下文 roles 可能为 null
- 路由守卫只拦 canView=false 且已登录的用户；游客不跳无权限页

## 新增页面必须登记三处（缺一即错）

1. 导航项配置（NAV_ITEMS）：path / label / icon
2. 种子矩阵：新页面为每个默认角色插一行（INSERT ... ON CONFLICT DO NOTHING，避免重复启动报错）
3. 内置默认矩阵（DEFAULT_ENTRIES）：接口失败时的回退依据

## 角色成员管理：不自建角色表

- 角色与成员的增删改查走**平台 AuthorizationSDK**（roles.list/create/update/delete + members.add/remove/clear + search），后端薄封装一层即可；自建角色表会与平台鉴权脱节
- 平台对 bizID 有字符白名单（仅字母/数字/下划线），中文/横线一律拒绝——前端提交前先校验，角色名称可中文
- 管理角色成员页本身仅管理员可见可写

## 平台陷阱：内置 admin 角色含「企业全员」

预置 admin 角色通常绑定了「企业全员」（只读属性，不可移除、角色不可删），用它做管理鉴权等于**全员都是管理员**：

- 判定方法：用一个非成员账号实测角色命中（平台角色匹配接口）
- 修复：新建显式成员的管理角色（如 super_admin），把全部管理绑定（后端 @CanRole、前端矩阵、矩阵表数据）迁移过去；admin 保留展示但不再绑定任何权限
- 从第一天就用独立管理角色，不要事后迁移
