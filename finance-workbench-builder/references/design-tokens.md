# 财务工作台设计规范

设计原型：App 设计（Admin/SaaS）。情绪基调是「冷静专业、避免焦虑与信息过载」——财务人员高频处理多账期、高风险数据，界面的首要任务是让异常一眼可见、让数字精确可读，装饰一律让位。

## 色彩系统

基底：深靛蓝主色 + 冷灰白底 + 琥珀/绯红双警示体系。比例 70% 中性底 / 20% 主色 / 10% 警示色。警示色只允许出现在四类场景：S账超期、AR 90天+、高风险异常、预算执行率 ≥90%。

| Token | HSL | 用途 |
|---|---|---|
| background | hsl(220 20% 97%) | 冷灰白底，长时间注视不疲劳 |
| card | hsl(0 0% 100%) | 纯白卡片容器 |
| foreground | hsl(220 25% 14%) | 深蓝墨文字 |
| muted-foreground | hsl(220 10% 50%) | 次级说明 |
| primary | hsl(217 80% 56%) | 商务蓝，核心操作/激活态/图表主线 |
| accent | hsl(217 40% 96%) | 极浅蓝交互反馈背景 |
| border | hsl(220 15% 88%) | 低对比边框，不干扰数据阅读 |

语义色（shadcn 命名）：

| 用途 | token | 说明 |
|---|---|---|
| 成功/已结清/已匹配 | `destructive` 的对偶 `success`（hsl(152 60% 42%)） | 背景用 hsl(152 50% 95%) |
| 警告/预算≥90%/待处理 | `warning`（hsl(38 92% 50%)） | 小字号用 hsl(38 90% 35%) 保证对比度 |
| 错误/超期/高危/驳回 | `destructive`（hsl(4 75% 55%)） | 高风险整行背景 hsl(4 70% 96%) |

## 字体排版

- Heading/Body：Inter, "PingFang SC", system-ui, sans-serif
- 数字（KPI 金额/列表金额/天数）：JetBrains Mono, "SF Mono", monospace——等宽保证金额列纵向对齐，是财务表格可读性的核心

## 布局

- 左侧深色侧边栏（hsl(220 25% 12%)，非透明，右侧 1px border 分隔）+ 主内容区流式布局
- 主内容 max-w-[1400px] 居中约束，防止超宽屏表格过度拉伸
- 激活导航项 primary 填充 + 白字；hover accent 背景；对比度 ≥7:1
- 响应式：移动端侧边栏折叠为底部 Tab Bar，表格横向滚动

## 视觉语言

- 圆角一律 `rounded-sm`（2px），卡片用 1px border 替代阴影（`shadow-none`）——大圆角和柔和阴影会削弱财务报表的精确感与权威感
- 间距 compact：`gap-3` / `p-4` 两级为主
- 无装饰图形、无渐变、无毛玻璃；层级只靠网格对齐、字重对比、警示色块传递
- 动效只保留功能性反馈：状态切换 150ms ease-out，Toast 200ms

## Tailwind 实际类名对照（识别签名）

金额与 KPI：

```
列表金额列:   font-mono tabular-nums text-right
KPI 卡数字:   font-mono text-3xl font-bold text-right
```

状态胶囊（统一 pill 形态，语义色 10% 背景 + 40% 边框 + 同色文字）：

```
高风险/驳回:   bg-destructive/10 text-destructive border-destructive/40
待处理/预警:   bg-warning/15 text-warning border-warning/40 rounded-full px-2 py-0.5
已结清/成功:   bg-success/10 text-success border-success/40
中性/草稿:     bg-muted text-muted-foreground border-border
```

高风险整行高亮（叠加左边框，色弱场景也能识别）：

```
行 td 级:     [&>td]:bg-destructive/5 [&>td:first-child]:border-l-[3px] [&>td:first-child]:border-l-destructive
```

表格与表单：

```
表头:          bg-muted font-semibold text-xs uppercase tracking-wider，行高 h-12
斑马纹:        偶数行 bg-muted/30
展开详情区:    bg-accent/30 缩进 p-4
日期:          统一 YYYY-MM-DD
```

## 组件原则

- 层级：Primary 按钮填充色；Secondary/Ghost 透明 + border；危险操作（驳回/删除）用 destructive outline 变体
- Focus ring：`primary/50 offset-2`，Button/Input/Table Row 的 Default/Hover/Focus/Disabled/Error 状态齐全
- 一致性：所有金额右对齐 + mono；所有日期同格式；状态标签同形态
- 空状态用极简线条图标，不用插画；禁止商务人物照片、通用科技渐变、3D 图标

## 反模式

- ❌ 大面积渐变/毛玻璃：分散对密集数据的注意力
- ❌ 圆角 >4px 或柔和阴影：削弱精确感
- ❌ 警示色滥用：红色出现在日常状态里会让真预警失去信号价值
- ❌ 金额左对齐或非等宽字体：列内数字无法纵向对齐，扫读效率崩塌
