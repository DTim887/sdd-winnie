# T&D Frontend Development Specification

## 1. 项目目录规范

### 1.1 项目目录结构总览

```
app/                                     # Next.js App Router 根目录
  │
  ├── (auth)/                            # 路由分组：认证相关页面
  │   ├── login/
  │   │   └── page.tsx
  │   └── layout.tsx
  │
  ├── (main)/                            # 路由分组：主业务页面
  │   ├── [resource]/                    # 动态路由（如 products、orders）
  │   │   ├── page.tsx                   # 列表页
  │   │   ├── [id]/
  │   │   │   └── page.tsx              # 详情页
  │   │   └── layout.tsx
  │   └── layout.tsx
  │
  ├── api/                               # Next.js Route Handler（BFF 层）
  │   └── v1/
  │       └── [resource]/
  │           └── route.ts
  │
  ├── layout.tsx                         # 根布局
  ├── page.tsx                           # 根页面
  └── globals.css                        # 全局样式
  
components/                              # 组件目录
  ├── ui/                                # 通用 UI 基础组件（无业务语义）
  ├── features/                          # 业务功能组件（按模块划分）
  │   └── [feature]/
  └── layouts/                           # 布局组件

lib/                                     # 工具与基础设施
  ├── api/                               # HTTP 客户端封装
  │   ├── client.ts                      # 统一 fetch 封装
  │   └── [resource].ts                  # 各资源的 API 调用函数
  ├── hooks/                             # 自定义 React Hook
  ├── stores/                            # 全局状态（Zustand）
  ├── utils/                             # 纯工具函数（无副作用）
  ├── constants/                         # 常量
  └── types/                             # TypeScript 类型定义

public/                                  # 静态资源
```

### 1.2 各层职责说明

| 层 | 职责 | 禁止事项 |
|---|---|---|
| app/page.tsx | 页面入口，组合 feature 组件，处理路由参数 | 禁止含业务逻辑和直接 API 调用 |
| app/api/v1 | BFF Route Handler，转发/聚合后端 API | 禁止含前端渲染逻辑 |
| components/ui | 纯展示组件，通过 props 驱动 | 禁止直接访问全局状态或发起 API 请求 |
| components/features | 业务功能组合，可访问 store 和 API | 禁止跨模块直接引用其他 feature 内部组件 |
| lib/api | 封装所有 HTTP 调用，统一错误转换 | 禁止包含 UI 逻辑或 React Hook |
| lib/hooks | 封装异步状态与业务逻辑 | 禁止直接操作 DOM |
| lib/stores | 全局共享状态 | 禁止存储可从 URL/服务端派生的数据 |
| lib/utils | 纯函数工具 | 禁止引用 React、Next.js 或业务模块 |

### 1.3 命名约定

| 类型 | 格式 | 示例 |
|---|---|---|
| 页面组件（page.tsx） | 默认导出，PascalCase | `export default function ProductListPage()` |
| 业务组件 | PascalCase，文件与组件同名 | `ProductCard.tsx` |
| UI 基础组件 | PascalCase | `Button.tsx`、`Modal.tsx` |
| 自定义 Hook | `use` 前缀，camelCase | `useProductList.ts` |
| API 调用函数 | camelCase，动词+名词 | `fetchProducts`、`createOrder` |
| Store | camelCase + `Store` 后缀 | `useProductStore` |
| 类型/接口 | PascalCase，接口加 `I` 前缀（可选） | `ProductResponse`、`CreateOrderRequest` |
| 常量 | UPPER_SNAKE_CASE | `MAX_PAGE_SIZE` |
| 工具函数 | camelCase | `formatPrice`、`parseDate` |

---

## 2. API 对接规范

### 2.1 统一 HTTP 客户端

所有 API 请求必须通过 `lib/api/client.ts` 封装的统一客户端发起，禁止在组件或页面中直接调用 `fetch`。

```ts
// lib/api/client.ts
const BASE_URL = process.env.NEXT_PUBLIC_API_BASE_URL;

interface RequestOptions extends RequestInit {
  params?: Record<string, string | number>;
}

async function request<T>(path: string, options: RequestOptions = {}): Promise<T> {
  const { params, ...init } = options;
  const url = new URL(`${BASE_URL}${path}`);
  if (params) {
    Object.entries(params).forEach(([k, v]) => url.searchParams.set(k, String(v)));
  }

  const res = await fetch(url.toString(), {
    headers: { 'Content-Type': 'application/json', 'Accept': 'application/json', ...init.headers },
    ...init,
  });

  const body: ApiResponse<T> = await res.json();

  if (!res.ok || body.code !== 200) {
    throw new ApiError(body.code, body.msg);
  }

  return body.data;
}

export const apiClient = {
  get:    <T>(path: string, params?: Record<string, string | number>) =>
            request<T>(path, { method: 'GET', params }),
  post:   <T>(path: string, data: unknown) =>
            request<T>(path, { method: 'POST', body: JSON.stringify(data) }),
  put:    <T>(path: string, data: unknown) =>
            request<T>(path, { method: 'PUT', body: JSON.stringify(data) }),
  delete: <T>(path: string) =>
            request<T>(path, { method: 'DELETE' }),
};
```

### 2.2 资源 API 函数

每类后端资源对应 `lib/api/[resource].ts` 文件，统一封装该资源的所有接口调用。

```ts
// lib/api/products.ts
import { apiClient } from './client';
import type { ProductResponse, CreateProductRequest, PageResult } from '@/lib/types';

export const productsApi = {
  list:   (params: { startIndex: number; size: number }) =>
            apiClient.get<PageResult<ProductResponse>>('/api/v1/products', params),
  get:    (id: number) =>
            apiClient.get<ProductResponse>(`/api/v1/products/${id}`),
  create: (data: CreateProductRequest) =>
            apiClient.post<ProductResponse>('/api/v1/products', data),
  update: (id: number, data: Partial<CreateProductRequest>) =>
            apiClient.put<ProductResponse>(`/api/v1/products/${id}`, data),
  remove: (id: number) =>
            apiClient.delete<void>(`/api/v1/products/${id}`),
};
```

### 2.3 统一响应体类型

与后端响应结构保持严格对齐：

```ts
// lib/types/api.ts
export interface ApiResponse<T> {
  code: number | string;
  msg: string;
  data: T;
}

export interface PageResult<T> {
  rows: T[];
  pageSize: number;
  pageNum: number;
  total: number;
}

export interface ListResult<T> {
  list: T[];
}
```

### 2.4 请求头规范

- 所有请求默认携带 `Content-Type: application/json` 和 `Accept: application/json`
- 认证 Token 在 `client.ts` 中统一注入，禁止在业务代码中手动设置 `Authorization`
- 自定义私有请求头使用 `x-` 前缀（如 `x-request-id`），在客户端封装层统一添加

### 2.5 分页规范

与后端保持一致，使用 `startIndex` + `size` 参数：

```ts
// 分页请求示例
productsApi.list({ startIndex: 0, size: 20 });

// 分页状态 Hook
function useProductList() {
  const [startIndex, setStartIndex] = useState(0);
  const size = 20;
  // ...
}
```

---

## 3. 统一错误处理规范

### 3.1 错误类定义

```ts
// lib/types/error.ts
export class ApiError extends Error {
  constructor(
    public readonly code: number | string,
    public readonly msg: string,
  ) {
    super(msg);
    this.name = 'ApiError';
  }

  isClientError() { return String(this.code).includes('A'); }
  isSystemError() { return String(this.code).includes('B'); }
  isThirdPartyError() { return String(this.code).includes('C'); }
}
```

### 3.2 错误处理策略

| 错误类型 | HTTP 状态码 | 处理方式 |
|---|---|---|
| 客户端错误（A 类 / 4xx） | 400 / 404 等 | Toast 提示用户，不上报 |
| 系统错误（B 类 / 5xx） | 500 | Toast + 上报错误监控 |
| 第三方错误（C 类） | 502 / 504 | Toast 提示"服务暂时不可用"，上报 |
| 401 未认证 | 401 | 清除 Token，重定向到登录页 |
| 403 无权限 | 403 | 跳转到 403 页面 |
| 网络异常 | — | Toast 提示"网络异常，请检查网络" |

### 3.3 错误边界

- 每个页面级组件必须被 `ErrorBoundary` 包裹，防止单个组件错误导致整页崩溃
- 异步操作错误通过 `try/catch` 捕获后交由统一 `handleApiError` 函数处理，禁止在组件中各自实现错误提示逻辑

---

## 4. 测试与质量

### 4.1 目标与范围

- 测试类型：单元测试（Unit Test），不要求集成测试和 E2E 测试
- 测试层级：自定义 Hook（必须）+ 工具函数（必须）+ 关键业务组件（必须）
- UI 基础组件、页面布局：可选

### 4.2 测试框架

| 用途 | 工具 |
|---|---|
| 测试框架 | Vitest |
| 组件渲染 | React Testing Library |
| Mock | `vi.fn()` / `vi.mock()` |
| 断言库 | Vitest 内置 / Testing Library matchers |
| Mock HTTP | `msw`（Mock Service Worker） |

### 4.3 Hook 测试规范

```ts
// lib/hooks/__tests__/useProductList.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { useProductList } from '../useProductList';
import { server } from '@/mocks/server';
import { http, HttpResponse } from 'msw';

describe('useProductList', () => {
  it('初始加载时返回产品列表', async () => {
    const { result } = renderHook(() => useProductList());

    await waitFor(() => expect(result.current.isLoading).toBe(false));

    expect(result.current.products).toHaveLength(2);
  });

  it('请求失败时 error 不为空', async () => {
    server.use(http.get('/api/v1/products', () => HttpResponse.error()));

    const { result } = renderHook(() => useProductList());

    await waitFor(() => expect(result.current.error).toBeTruthy());
  });
});
```

### 4.4 组件测试规范

组件层不测业务逻辑，只测：渲染是否正确、用户交互是否触发正确回调、Loading / Error 状态是否展示。

```ts
// components/features/products/__tests__/ProductCard.test.tsx
import { render, screen, fireEvent } from '@testing-library/react';
import { ProductCard } from '../ProductCard';

describe('ProductCard', () => {
  it('正确渲染产品名称和价格', () => {
    render(<ProductCard name="测试商品" price={99.9} onSelect={vi.fn()} />);

    expect(screen.getByText('测试商品')).toBeInTheDocument();
    expect(screen.getByText('¥99.90')).toBeInTheDocument();
  });

  it('点击时触发 onSelect 回调', () => {
    const onSelect = vi.fn();
    render(<ProductCard name="测试商品" price={99.9} onSelect={onSelect} />);

    fireEvent.click(screen.getByRole('button'));

    expect(onSelect).toHaveBeenCalledTimes(1);
  });
});
```

### 4.5 必测场景清单

| 场景类型 | 说明 | 是否必须 |
|---|---|---|
| Happy Path | 正常数据，组件/Hook 渲染/返回预期结果 | 必须 |
| Loading 状态 | 异步请求进行中时的展示逻辑 | 必须 |
| Error 状态 | 请求失败时的错误展示或回调 | 必须 |
| 空数据 | 列表为空时的空状态展示 | 必须 |
| 用户交互 | 点击、输入等事件触发正确回调 | 必须（有交互时） |
| 参数边界 | 极长字符串、0 值、null props 的处理 | 建议 |

### 4.6 覆盖率要求

| 指标 | 目标 | 说明 |
|---|---|---|
| 整体行覆盖率 | ≥ 70% | 重点保障 |
| 自定义 Hook 行覆盖率 | ≥ 80% | 重点保障 |
| 工具函数行覆盖率 | ≥ 80% | 重点保障 |

- 覆盖率为自检项，不卡 CI 合并，但 PR 描述中须注明当前覆盖率数值

查看覆盖率报告：

```bash
npm run test:coverage
# 报告路径：coverage/index.html
```

### 4.7 测试命名规范

统一格式：`描述_场景_预期结果`（中文或英文均可，项目内保持一致）

```ts
// 好的命名
'初始加载时返回产品列表'
'请求失败时 error 不为空'
'点击删除按钮时调用 onDelete 回调'

// 避免
'test1'
'works correctly'
'should work'
```

### 4.8 PR 提交前自检清单

提交 PR 前，逐项确认：

- [ ] 自定义 Hook 的 Happy Path、Loading、Error 场景均已覆盖
- [ ] 关键业务组件的渲染和用户交互已测试
- [ ] 工具函数的边界输入已覆盖
- [ ] 所有测试本地执行通过（`npm test`）
- [ ] 无 TypeScript 类型错误（`npm run type-check`）
- [ ] ESLint 无报错（`npm run lint`）

### 4.9 禁止事项

- 禁止在测试中使用真实网络请求，所有 HTTP 调用必须通过 `msw` Mock
- 禁止测试共享可变状态，每个测试用例必须独立可重复执行
- 禁止只测 UI 快照（Snapshot Test）来堆砌覆盖率
- 禁止注释掉失败的测试来让构建通过，失败测试必须修复或删除并说明原因
- 禁止在测试中直接操作 `localStorage` / `sessionStorage`，通过 Mock 隔离

---

## 5. 研发流程

### 5.1 分支策略

与后端保持一致：

- 功能分支命名：`[Jira Ticket ID|Feature]/<工单号>-简短描述`
- 示例：`SHDRP-433820|product-list-page`

---

## 6. 前端技术栈

该规范仅描述 Next.js 前端应用，不包含后端服务。可以被覆写。

| 约束项 | 标准 | 覆盖策略 |
|:---:|:---|:---|
| 框架 | Next.js 15+（App Router） | 不允许覆盖 |
| 语言 | TypeScript 5+，严格模式（`strict: true`） | 不允许覆盖 |
| 包管理 | npm | 不允许覆盖 |
| 样式方案 | Tailwind CSS | 不允许覆盖 |
| 全局状态 | Zustand | 不允许覆盖 |
| 测试框架 | Vitest + React Testing Library | 不允许覆盖 |
| HTTP Mock | msw | 不允许覆盖 |
| 代码规范 | ESLint（Next.js 官方配置）+ Prettier | 不允许覆盖 |