# Teable 前端代码规范与架构指南

## 一、技术栈概览

| 分类     | 技术                    | 用途               |
| -------- | ----------------------- | ------------------ |
| 框架     | Next.js 16              | SSR/SSG 全栈框架   |
| 视图     | React 18 + TypeScript   | 类型安全的 UI 开发 |
| 样式     | Tailwind CSS 3          | 原子化 CSS         |
| 状态管理 | Zustand 4               | 轻量级全局状态     |
| 数据请求 | TanStack React Query 5  | 服务端状态管理     |
| 国际化   | i18next + next-i18next  | 多语言支持         |
| UI 组件  | @teable/ui-lib (shadcn) | 基础组件库         |

---

## 二、布局与路由体系

### 2.1 技术选型

- **Next.js 约定式路由**：`src/pages/` 目录自动映射
- **布局模式**：通过 `Component.getLayout` 注入

### 2.2 编写习惯

```typescript
// 页面组件
const BasePage = () => <div>Content</div>;

BasePage.getLayout = (page: React.ReactElement) => {
  return <BaseLayout>{page}</BaseLayout>;
};
```

### 2.3 布局层级

```
AppProviders → AppLayout → MainLayout → BaseLayout/SpaceLayout
```

### 2.4 核心逻辑

- `_app.tsx` 中自动调用 `getLayout`
- 布局组件负责 Provider 注入和页面框架搭建

---

## 三、数据请求与状态管理

### 3.1 React Query（服务端状态）

**技术**：TanStack React Query 5
**习惯**：

- Query Key 使用常量定义（`ReactQueryKeys`）
- 分离 `queryKey`、`queryFn`、`enabled` 参数
- Mutation 成功后调用 `invalidateQueries`

```typescript
const { data } = useQuery({
  queryKey: ReactQueryKeys.getDashboard(dashboardId),
  queryFn: () => getDashboard(baseId, dashboardId).then((res) => res.data),
  enabled: Boolean(baseId),
});
```

### 3.2 Zustand（全局状态）

**技术**：Zustand 4 + persist middleware
**习惯**：

- 接口定义以 `I` 前缀（`ISidebarState`）
- Store 使用 `use` 前缀（`useSidebarStore`）
- 支持 localStorage 持久化

```typescript
const useSidebarStore = create<ISidebarState>()(
  persist(
    (set) => ({
      isVisible: true,
      setVisible: (v) => set((s) => ({ ...s, isVisible: v })),
    }),
    { name: LocalStorageKeys.Sidebar }
  )
);
```

### 3.3 状态分层

| 层级       | 工具        | 用途         |
| ---------- | ----------- | ------------ |
| 组件状态   | useState    | 局部 UI 状态 |
| 服务端状态 | React Query | API 数据缓存 |
| 全局状态   | Zustand     | 跨组件共享   |
| 上下文状态 | Context     | 局部树状传递 |

---

## 四、组件开发

### 4.1 文件命名

- **组件**：PascalCase（`DashboardGrid.tsx`）
- **Hook**：`use` 前缀 + camelCase（`useAI.ts`）
- **Store**：`use` 前缀 + `Store` 后缀（`useSelectionStore.ts`）

### 4.2 组件模式

**纯展示组件**：无状态，通过 props 接收数据

```typescript
interface PluginItemProps {
  name: string;
  dragging: boolean;
}

export const PluginItem = ({ name, dragging }: PluginItemProps) => (
  <div className={cn('p-4', { 'opacity-50': dragging })}>{name}</div>
);
```

**容器组件**：包含业务逻辑和状态

```typescript
export const DashboardGrid = ({ dashboardId }: { dashboardId: string }) => {
  const [isDragging, setIsDragging] = useState(false);
  const { data } = useQuery({ queryKey: [...], queryFn: ... });

  return <ResponsiveGridLayout onDrag={() => setIsDragging(true)}>...</ResponsiveGridLayout>;
};
```

### 4.3 自定义 Hook 模式

**数据获取 Hook**：

```typescript
export function useAI() {
  const baseId = useBaseId() as string;
  const { data } = useQuery({
    queryKey: ["ai-config", baseId],
    queryFn: () => getAIConfig(baseId).then(({ data }) => data),
  });
  return { enable: Boolean(data) };
}
```

**组合型 Hook**：封装多个子 Hook

```typescript
export const useBillingLevel = ({ spaceId, baseId }) => {
  const isCloud = useIsCloud();
  const { data: subscription } = useQuery({ ... });
  return subscription?.level;
};
```

---

## 五、国际化

### 5.1 技术选型

- **i18next**：国际化核心
- **next-i18next**：Next.js 集成

### 5.2 配置结构

```
src/features/i18n/
├── dashboard.config.ts
├── base.config.ts
└── auth.config.ts
```

### 5.3 使用方式

```typescript
import { useTranslation } from 'next-i18next';
import { dashboardConfig } from '@/features/i18n/dashboard.config';

const { t } = useTranslation(dashboardConfig.i18nNamespaces);
return <div>{t('pluginNotFound')}</div>;
```

---

## 六、工具函数

### 6.1 编写习惯

- 纯函数，无副作用
- 使用 Zod 进行类型验证
- 错误处理返回统一格式

### 6.2 示例

```typescript
export const extractHtmlHeader = (html?: string) => {
  if (!html) return { result: undefined };

  const validate = z.array(fieldVoSchema).safeParse(headers);
  if (!validate.success) {
    return { result: undefined, error: fromZodError(validate.error).message };
  }

  return { result: validate.data };
};
```

---

## 七、目录结构

```
src/
├── components/          # 全局通用组件（纯 UI）
├── features/            # 功能模块（按业务域划分）
│   ├── app/             # 主应用
│   │   ├── layouts/     # 布局组件
│   │   ├── hooks/       # 自定义 Hooks
│   │   ├── blocks/      # 业务区块
│   │   └── utils/       # 工具函数
│   ├── auth/            # 认证模块
│   └── system/          # 系统页面
├── lib/                 # 通用工具库
├── store/               # 全局状态
├── themes/              # 主题系统
└── pages/               # Next.js 路由
```

---

## 八、核心设计原则

1. **关注点分离**：UI 组件、业务逻辑、工具函数分离
2. **单一职责**：每个文件/组件只做一件事
3. **依赖倒置**：通过 SDK 封装，解耦 HTTP 层
4. **状态提升**：共享状态提升到公共父组件或使用全局 Store

---

## 九、最佳实践清单

- ✅ 组件使用 PascalCase，Hook 使用 `use` 前缀
- ✅ React Query 管理服务端状态，Zustand 管理全局 UI 状态
- ✅ API 调用通过 `@teable/openapi` SDK 封装
- ✅ 使用 Zod 进行运行时验证
- ✅ 布局通过 Next.js `getLayout` 模式注入
- ✅ 代码按功能模块组织（feature-based）
