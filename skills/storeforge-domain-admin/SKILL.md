---
name: storeforge-domain-admin
description: Vue 3 + Element Plus 硬约束和最佳实践
---

# Admin 硬约束 — Vue 3 + Element Plus

> 本 skill 约束管理后台（Admin Panel）的技术路径。违反规则 = 阻断当前工作。

## 定位

当任务涉及 **管理后台 / Admin Panel / 运营后台** 的前端开发时，必须调用 `storeforge-domain-admin`。本 skill 定义了管理后台的项目结构、技术栈、反模式清单和模式推荐，确保 Agent 不走错误路径。

与 `storeforge-domain-backend`、`storeforge-domain-website`、`storeforge-domain-flutter` 并列，分别覆盖后端、管理后台、官网、Flutter 四个技术域。

## 项目结构（固定，不可变）

```
src/
├── views/              # 页面级组件，每个路由对应一个 view
│   ├── dashboard/      # 数据看板
│   ├── product/        # 商品管理（列表、详情、SKU、批量操作）
│   ├── order/          # 订单管理（列表、详情、监控 iframe）
│   ├── user/           # 用户管理
│   ├── promotion/      # 营销管理（优惠券、活动）
│   └── settings/       # 系统设置
├── components/         # 可复用业务组件
│   ├── ProductTable/   # 商品表格（含 SKU 维度/批量操作/快速搜索）
│   ├── OrderTable/     # 订单表格（含监控 iframe 嵌入）
│   ├── Form/           # 通用表单组件（含动态校验/草稿保存）
│   ├── ExportTask/     # 异步导出任务进度组件
│   └── Layout/         # 布局组件（侧栏、顶栏、面包屑）
├── api/                # API 请求封装，按模块拆分
│   ├── product.ts
│   ├── order.ts
│   ├── user.ts
│   ├── promotion.ts
│   └── request.ts      # axios 实例 + interceptor 配置
├── router/             # 路由定义
│   ├── index.ts        # 路由注册 + 守卫（权限检查）
│   └── modules/        # 按模块拆分路由配置
├── store/              # Pinia 状态管理
│   ├── modules/        # 按业务域拆分 store
│   └── index.ts
├── types/              # TypeScript 类型定义（由 OpenAPI 生成）
└── utils/              # 工具函数
```

**目录职责**：
- `views/` 只负责页面布局和组件拼装，不包含业务逻辑
- `components/` 存放跨页面复用的业务组件，必须 props-driven
- `api/` 所有 HTTP 请求集中管理，禁止在组件内直接调用 axios
- `router/` 路由配置与权限守卫，懒加载视图
- `store/` 全局状态（用户信息、权限、缓存数据），禁止组件内维护跨页面状态

## 技术栈约束

| 类别 | 必须 | 禁止 |
|------|------|------|
| 框架 | Vue 3（Composition API + `<script setup>`） | Vue 2、Options API |
| UI 库 | Element Plus | Ant Design Vue、Naive UI、Vuetify 等其他 UI 库 |
| 路由 | Vue Router 4 | 手动 hash 路由、其他路由库 |
| 状态管理 | Pinia | Vuex、mitt 事件总线 |
| HTTP 客户端 | Axios（通过 interceptor 统一处理） | fetch、其他 HTTP 库直调 |
| 构建工具 | Vite | Webpack、其他构建工具 |
| 语言 | TypeScript（严格模式） | 无类型 JavaScript |

**版本要求**：
- Vue >= 3.4（使用 `defineModel`、`useTemplateRef` 等新 API）
- Element Plus >= 2.5
- Pinia >= 2.1
- Vite >= 5.0

## 代码规范

### 文件命名
- 组件文件：`PascalCase.vue`（如 `ProductTable.vue`）
- 工具/API 文件：`camelCase.ts`（如 `request.ts`、`product.ts`）
- 类型文件：`camelCase.ts`，放在 `types/` 下，与 api 模块对应
- Store 文件：`camelCase.ts`，放在 `store/modules/` 下

### 模块划分
- 按业务域垂直切分（product、order、user、promotion），而非按技术层切分
- 每个域包含自己的 view、component、api、store、type
- 跨域共享的组件放入 `components/` 根目录

### 错误处理
- 所有 API 错误通过 axios response interceptor 统一处理
- 错误码使用 `storeforge-domain-backend` 定义的 `{MODULE}_{SEQ}` 格式
- 金额显示：后端 int64（分）→ 前端统一 `formatAmount(amount: string | bigint): string` 转换为元（JS `number` 无法完整表示 int64，OpenAPI 生成类型需使用 `string`）

### 与后端 API 契约
- 类型定义由 `storeforge-domain-backend` 的 OpenAPI spec 通过 `openapi-typescript-codegen` 生成
- 禁止手写 API 类型，必须从生成的类型导入
- 分页参数统一 `page` / `pageSize`，响应包含 `total`

## 反模式清单（BLOCK，违反即阻断）

### 1. 禁止引入其他 UI 组件库，必须 Element Plus

**原因**：保证 UI 一致性、减少包体积、降低维护成本。多个 UI 库会导致样式冲突、主题不统一、组件行为不一致。Element Plus 已覆盖管理后台所需全部组件（表格、表单、弹窗、树、上传等）。

### 2. 禁止表格不支持排序/筛选/分页/列宽拖拽

**原因**：管理后台的核心交互是数据表格。不支持排序/筛选/分页/列宽拖拽的表格会严重降低运营效率。必须使用 `el-table` 配合 `sortable`、`filter`、`pagination` 组件，列宽拖拽使用 `el-table` 的 `resizable` 属性或 `vue-draggable-resizable`。

### 3. 禁止表单不支持动态校验/草稿保存/重置

**原因**：后台表单通常字段多、校验规则复杂。不支持动态校验会导致错误提交；不支持草稿保存会导致数据丢失（运营人员填到一半被打断是常见场景）；不支持重置会导致操作混乱。必须使用 `el-form` + `el-form-item` 的 `rules` 属性，草稿通过 Pinia 或 localStorage 实现。

### 4. 禁止订单管理页不嵌入监控 iframe

**原因**：订单管理页需要实时展示订单履约状态和物流轨迹。不嵌入监控 iframe 会导致运营人员需要切换到其他系统查看信息，降低效率。必须在 `order/views/` 的详情页中嵌入 iframe 指向监控面板。

### 5. 禁止 API 调用不经过 axios interceptor 统一处理

**原因**：不经过 interceptor 的请求无法统一处理 token 刷新、错误码映射、请求重试、loading 状态。分散的错误处理会导致不一致的用户体验和遗漏的安全问题。必须在 `api/request.ts` 中配置统一的 request/response interceptor，所有 API 调用通过该实例发起。

### 6. 禁止商品列表不支持 SKU 维度展示/批量操作/快速搜索

**原因**：商品管理是管理后台最高频的操作。不支持 SKU 维度展示会导致运营人员无法查看和管理变体；不支持批量操作（批量上架/下架/调价）会极大降低效率；不支持快速搜索会导致查找困难。必须使用 `ProductTable` 组件，支持 SKU 展开、批量选择、搜索框。

### 7. 禁止同步导出大文件，必须异步任务

**原因**：同步导出大文件（如万级订单导出）会阻塞浏览器、导致请求超时、消耗大量服务器内存。必须采用异步任务模式：前端发起导出请求 → 后端创建导出任务 → 前端通过 `ExportTask` 组件轮询进度 → 完成后下载。

### 8. 禁止直接操作 DOM，必须 Vue 响应式

**原因**：直接操作 DOM（`document.querySelector`、`element.style`）违反 Vue 响应式原则，导致状态不同步、组件行为不可预测、难以测试。所有 DOM 操作必须通过 Vue 响应式 API（ref、reactive）或组件方法完成。特殊场景（如第三方图表库初始化）必须使用 `onMounted` + `ref` 封装。

## 模式推荐

| 场景 | 推荐模式 | 说明 |
|------|----------|------|
| 数据表格 | `el-table` + 排序 + 筛选 + 分页 + 列宽拖拽 | 表格是后台核心组件，必须功能完整 |
| 复杂表单 | `el-form` + 动态 rules + 草稿保存 + 重置 | 校验规则根据状态动态变化 |
| 大文件导出 | 后端异步任务 + 前端进度轮询（`ExportTask` 组件） | 使用 `setInterval` 轮询任务状态 |
| 权限控制 | Vue Router 全局守卫 + Pinia 权限状态 | 路由级 + 按钮级双重控制 |
| 数据看板 | ECharts（通过 `echarts-for-vue` 封装） | 图表组件统一封装 |
| 文件上传 | `el-upload` + 后端异步处理 + CDN URL 返回 | 参考 `storeforge-domain-backend` 文件上传规范 |

### 决策树

```
遇到数据表格？
  └─> el-table + 排序 + 筛选 + 分页 + 列宽拖拽
遇到复杂表单？
  └─> el-form + 动态 rules + 草稿保存 + 重置
遇到大文件导出？
  └─> 后端异步任务 + 前端进度轮询
遇到权限控制？
  └─> Vue Router 全局守卫 + Pinia 权限状态
遇到金额显示？
  └─> formatAmount(string | bigint) → 元，禁止直接用 number 计算
```

## 常见坑

### Element Plus 表格大数据渲染

**问题**：`el-table` 渲染超过 500 行数据时明显卡顿。

**解决方案**：
- 使用 Element Plus 的虚拟滚动（`el-table-v2`），适合万级数据
- 对于万级以上数据，必须服务端分页，前端只渲染当前页
- 避免在 `el-table` 的 slot 中使用复杂计算，用 `computed` 或模板预处理

### 表单校验性能

**问题**：复杂表单（50+ 字段）每次输入都触发全量校验，导致输入卡顿。

**解决方案**：
- 使用 `trigger: 'blur'` 代替 `trigger: 'change'` 减少校验频率
- 懒校验模式：提交时全量校验，输入时只校验当前字段
- 使用 `el-form` 的 `validateField` 替代 `validate` 做单字段校验

### Token 过期与无感刷新

**问题**：Token 过期后多个请求同时触发刷新，导致多次刷新。

**解决方案**：
- axios response interceptor 中用 Promise 队列机制：第一个 401 触发刷新，后续 401 等待刷新完成后重试
- 参考 `storeforge-domain-backend` 的 JWT 规范（RS256/ES256、refresh token 存 HttpOnly cookie）

### 路由懒加载与首屏性能

**问题**：所有路由组件打包到一个 chunk，首屏加载慢。

**解决方案**：
- 使用 `defineAsyncComponent` 或 `() => import()` 懒加载视图组件
- `router/index.ts` 中每个路由使用动态 import
- 公共组件（Layout、头部导航）不懒加载

## 与其他 Skill 的关联

- **storeforge-domain-backend**：API 类型由后端 OpenAPI spec 生成，金额字段需处理 int64→元转换
- **storeforge-testing**：Admin 端使用 `vitest` 做组件测试，覆盖率 ≥ 70%
- **storeforge-review**：api-contract-checker 子代理会校验前后端字段是否对齐
- **storeforge-harness**：本 skill 的 8 条反模式自动加载到 harness 检查清单，每个子任务完成后自动检查
- **storeforge-executing**：子任务开发完成后，必须通过 harness 约束检查才能进入 L1 测试
- **storeforge-verification**：harness 检查结果写入验证清单，BLOCK 违规项必须修复
