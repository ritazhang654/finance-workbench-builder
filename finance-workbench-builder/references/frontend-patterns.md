# 前端页面模式（React 19 + antd-table + react-hook-form）

## 页面目录四文件拆分

每个模块一个 `client/src/pages/<Name>Page/` 目录。拆分不是为了好看，是为了让每类关注点（数据、渲染、表单、纯逻辑）独立可测、单文件不超行数上限：

| 文件 | 职责 | 规模 |
|---|---|---|
| `XxxPage.tsx` | 容器：state、数据获取、布局编排、弹窗开关 | 70–200 行 |
| `XxxTable.tsx` | antd-table 列定义、单元格渲染、行操作按钮 | 100–180 行 |
| `XxxFormDialog.tsx` | 新建/编辑弹窗（react-hook-form + zodResolver） | 100–200 行 |
| `xxx-helpers.ts` | 金额格式化、状态→颜色映射、错误信息提取等纯函数 | <100 行 |

容器组件不写列定义，表格组件不持有数据源——数据由容器 `useState` + `useCallback` 拉取后经 props 下传。状态管理用原生 hooks 即可，不引入额外状态库。

## 容器模式骨架

```typescript
const XxxPage: React.FC = () => {
  const [items, setItems] = useState<XxxSummary[]>([]);
  const [loading, setLoading] = useState(false);
  const [dialogOpen, setDialogOpen] = useState(false);

  const load = useCallback(async () => {
    setLoading(true);
    try {
      const res = await xxxApi.listXxx();
      setItems(res.items);
    } finally {
      setLoading(false);
    }
  }, []);

  useEffect(() => { void load(); }, [load]);

  const handleAction = async (id: string) => {
    await xxxApi.approveXxx(id);   // 出错由 axios 拦截器统一 toast
    toast.success('操作成功');
    void load();                    // 刷新列表，保持数据新鲜
  };

  return (
    <div className="space-y-4">
      <div className="flex justify-between items-center">
        <h1>模块标题</h1>
        <Button onClick={() => setDialogOpen(true)}>新建</Button>
      </div>
      <XxxTable items={items} loading={loading} onAction={handleAction} />
      <XxxFormDialog open={dialogOpen} onClose={() => setDialogOpen(false)}
        onSuccess={() => { setDialogOpen(false); void load(); }} />
    </div>
  );
};
```

编写顺序：Hook 声明 → useEffect → 事件处理 → JSX。导入顺序：React → 三方库 → 内置工具 → 相对路径。

## API 层

`client/src/api/<name>/index.ts` 每模块一个文件，`client/src/api/index.ts` 用 namespace 聚合（`export * as xxx from './xxx'`）。请求一律走 `axiosForBackend`：

```typescript
import { axiosForBackend } from '@lark-apaas/client-toolkit/utils/getAxiosForBackend';
import type { XxxSummary, PageResult } from '@shared/api.interface';

export async function listXxx(): Promise<PageResult<XxxSummary>> {
  const res = await axiosForBackend.get('/api/xxx');
  return res.data;
}
```

禁止 fetch、禁止 mock。前后端 method/path 必须与 Controller 装饰器完全一致。

## 表格模式（antd-table）

用 `@lark-apaas/client-toolkit/antd-table`（先调 table-skill），列类型 `TableColumnsType<T>`：

```typescript
export const columns: TableColumnsType<XxxSummary> = [
  {
    title: '金额',
    dataIndex: 'amount',
    align: 'right',
    render: (v: string) => (
      <span className="font-mono tabular-nums text-right">{formatAmount(v)}</span>
    ),
  },
  {
    title: '状态',
    dataIndex: 'status',
    render: (v: XxxStatus) => <StatusPill status={v} />,
  },
  {
    title: '操作',
    render: (_, record) => (
      <div className="flex gap-2">
        <Button size="sm" onClick={() => onApprove(record.id)}>通过</Button>
        <Button size="sm" variant="destructive"
          onClick={() => setRejectTarget(record)}>驳回</Button>
      </div>
    ),
  },
];
```

- 金额列 `align: 'right'` + mono 渲染；日期统一 `formatDate`（YYYY-MM-DD）
- 高风险行：rowClassName 或行级 td 类 `[&>td]:bg-destructive/5 [&>td:first-child]:border-l-[3px] [&>td:first-child]:border-l-destructive`
- 驳回必须走弹窗填原因（必填），禁止 confirm/alert/prompt

## 表单弹窗模式

react-hook-form + zodResolver + shadcn Form/Dialog/Input（先调 forms-skill）。骨架：

```typescript
const formSchema = z.object({
  name: z.string().min(1, '名称必填'),
  amount: z.coerce.number().positive('金额必须为正数'),
  expectedDate: z.string().min(1, '请选择日期'),
});
type FormValues = z.infer<typeof formSchema>;

const form = useForm<FormValues>({ resolver: zodResolver(formSchema) });

const onSubmit = async (values: FormValues) => {
  await xxxApi.createXxx(values);
  onSuccess();
};
```

- 输入组件用 shadcn（Input/Select/Calendar+Popover），禁止原生 `<input>`/`<select>`/`input[type=date]`
- SelectItem 的 value 禁止空值，占位用 `<SelectValue placeholder="..." />`
- 金额输入 z.coerce.number，展示层格式化与提交值分离

## 交互闭环

所有写操作遵循统一节拍：**行内按钮 → 弹窗确认（必要时填原因）→ API 调用 → toast 反馈 → 刷新列表**。

- 驳回/删除按钮 `variant="destructive"` 且原因必填
- toast 用 sonner（`toast.success` / `toast.error`）
- 错误不静默：接口报错显示明确信息，由 helpers 里的 `extractErrorMessage` 统一提取后端 `ApiErrorResponse` 的 message

## 路由与导航

- 路由在 `client/src/app.tsx` 的 `<Routes>` 内集中声明，每个页面唯一路由，首页有 `/` 的 index 路由
- 侧边栏在 `Layout.tsx` 的 NAV_ITEMS 配置（路径、标题、lucide 图标），与路由表一一对应，NavLink 激活态高亮
- 页面跳转只用 NavLink/Link/useNavigate，禁止 window.location.href
- 导航显隐、路由守卫、页面只读态由可配置权限矩阵驱动（usePagePermission），设计见 `rbac-matrix.md`；新增页面登记三处（导航/种子矩阵/默认矩阵）

## 数据安全渲染

渲染前检查数据存在；loading/空态/错误各有 fallback UI：

```tsx
if (loading) return <TableSkeleton rows={6} />;
if (!items.length) return <EmptyState text="暂无数据" />;
```
