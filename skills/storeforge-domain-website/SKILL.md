---
name: storeforge-domain-website
description: Next.js + shadcn/ui 硬约束和最佳实践
---

# Website 硬约束 — Next.js + shadcn/ui

> 本 skill 约束电商官网（Website）的技术路径。违反规则 = 阻断当前工作。

## 定位

当任务涉及 **电商官网 / 面向消费者的前端站点** 开发时，必须调用 `storeforge-domain-website`。本 skill 定义了官网的项目结构、技术栈、反模式清单和模式推荐，确保 Agent 不走错误路径。

与 `storeforge-domain-backend`、`storeforge-domain-admin`、`storeforge-domain-flutter` 并列，分别覆盖后端、管理后台、官网、Flutter 四个技术域。

## 项目结构（固定，不可变）

```
src/
├── app/                # App Router（Next.js 13+）
│   ├── (marketing)/    # 营销路由组（首页、关于我们、帮助）
│   │   ├── page.tsx
│   │   └── layout.tsx
│   ├── (shop)/         # 商城路由组（商品列表、详情、分类）
│   │   ├── products/
│   │   │   ├── page.tsx          # 商品列表（SSR + ISR）
│   │   │   └── [slug]/
│   │   │       └── page.tsx      # 商品详情（SSR + ISR）
│   │   ├── cart/
│   │   │   └── page.tsx
│   │   └── checkout/
│   │       └── page.tsx
│   ├── api/              # Route Handlers（后端代理）
│   │   ├── products/
│   │   ├── cart/
│   │   ├── orders/
│   │   └── webhooks/     # 支付回调等
│   ├── [locale]/         # i18n 路由（next-intl）
│   │   ├── layout.tsx    # next-intl 中间件配置
│   │   └── ...
│   ├── layout.tsx        # 根布局（metadata、Provider）
│   ├── page.tsx          # 首页
│   ├── not-found.tsx     # 404 页面
│   └── robots.ts         # robots.txt
├── components/           # 可复用组件
│   ├── ui/               # shadcn/ui 组件（由 CLI 生成，不手写）
│   ├── layout/           # 布局组件（Header、Footer、Nav）
│   ├── product/          # 商品组件（Card、Grid、SKU 选择器）
│   ├── cart/             # 购物车组件
│   └── checkout/         # 结账流程组件
├── lib/                  # 工具函数和业务逻辑
│   ├── api.ts            # 后端 API 封装（经 route handler 代理）
│   ├── auth.ts           # 认证工具
│   ├── constants.ts      # 常量定义
│   ├── utils.ts          # 通用工具函数
│   └── ssrf-guard.ts     # SSRF 防护工具
├── i18n/                 # 国际化配置
│   ├── messages/         # 多语言 JSON 文件
│   │   ├── en.json
│   │   ├── zh.json
│   │   └── ja.json
│   ├── request.ts        # next-intl 请求配置
│   └── routing.ts        # 路由本地化配置
├── middleware.ts         # Next.js 中间件（i18n + 认证守卫）
└── types/                # TypeScript 类型定义（由 OpenAPI 生成）
public/                   # 静态资源
├── images/
├── icons/
└── fonts/
```

**目录职责**：
- `app/` 中每个目录对应一个路由，`page.tsx` 是路由入口，`layout.tsx` 是布局
- `(marketing)` 和 `(shop)` 是路由组，不影响 URL 路径，仅用于逻辑分组和布局隔离
- `api/` 下的 Route Handler 是客户端与后端的唯一通道，禁止客户端直接请求后端
- `components/ui/` 由 shadcn/ui CLI 生成，禁止手动修改
- `i18n/` 集中管理多语言资源，禁止在组件中硬编码文案

## 技术栈约束

| 类别 | 必须 | 禁止 |
|------|------|------|
| 框架 | Next.js（App Router） | Pages Router、其他 SSR 框架 |
| UI 库 | shadcn/ui + Tailwind CSS | Ant Design、MUI、Chakra UI 等其他 UI 库 |
| 国际化 | next-intl | i18next、react-intl、手动替换文案 |
| 样式 | Tailwind CSS + CSS Modules（特殊场景） | Styled Components、Emotion |
| 语言 | TypeScript（严格模式） | JavaScript |
| HTTP | fetch（经 Route Handler 代理） | axios 直调后端、客户端直连后端 API |
| 图片优化 | `next/image` | 原生 `<img>` 标签（除非有充分理由） |

**版本要求**：
- Next.js >= 15（App Router 稳定版）
- React >= 19
- TypeScript >= 5.4（严格模式）
- Tailwind CSS >= 3.4

## 代码规范

### 文件命名
- 组件文件：`PascalCase.tsx`（如 `ProductCard.tsx`）
- Route Handler：按 RESTful 路径组织（`app/api/products/route.ts`）
- 工具文件：`camelCase.ts`（如 `utils.ts`、`api.ts`）
- 类型文件：`camelCase.ts`，放在 `types/` 下，与后端 OpenAPI 生成类型对应

### Server vs Client Components
- **默认使用 Server Components**：数据获取、SEO 渲染、API 调用必须在 Server Component 中完成
- **仅以下场景使用 Client Components**（`"use client"`）：交互状态（useState/useEffect）、浏览器 API（localStorage/window）、事件监听
- Server Component 不能作为子组件传给 Client Component，必须通过 children prop 或 slot 模式传递

### 错误处理
- Route Handler 统一使用 `storeforge-domain-backend` 定义的 `BaseResponse<T>` 格式
- 错误码使用 `{MODULE}_{SEQ}` 格式（如 `PRODUCT_001`）
- 客户端通过 `use server` 或 Route Handler 获取错误信息，前端展示对应 i18n 文案

### 与后端 API 契约
- 类型定义由 `storeforge-domain-backend` 的 OpenAPI spec 通过 `openapi-typescript-codegen` 生成
- 禁止手写 API 类型，必须从生成的类型导入
- 金额字段：后端 int64（分）→ 前端统一 `formatAmount(amount: string | bigint): string` 转换为元（JS `number` 无法完整表示 int64，OpenAPI 生成类型需使用 `string`）

## 反模式清单（BLOCK，违反即阻断）

### 1. 禁止 Pages Router，必须 App Router

**原因**：Pages Router 已被 Next.js 官方标记为 legacy 模式，不支持 React Server Components、Streaming SSR、Partial Prerendering 等新特性。继续使用 Pages Router 会导致技术债累积，无法利用 Next.js 最新性能优化能力。

### 2. 禁止手动替换文案，必须 next-intl

**原因**：手动替换文案（if-else 语言判断、硬编码字典）无法维护、容易遗漏、不支持 RTL 布局、无法动态切换语言。next-intl 提供类型安全的多语言支持、路由本地化、消息格式化、日期/数字本地化，是电商多语言站点的唯一可行方案。

### 3. 禁止引入其他 UI 库，必须 shadcn/ui

**原因**：shadcn/ui 提供无运行时依赖的组件源码（非 npm 包），可以完全自定义、与 Tailwind CSS 深度集成、包体积最小化。引入其他 UI 库会导致样式冲突、包体积膨胀、维护成本增加。

### 4. 禁止页面不配置 metadata 和 OG 标签

**原因**：电商官网依赖搜索引擎和社交媒体引流。不配置 metadata（title、description、canonical）会导致 SEO 排名下降。不配置 OG 标签（og:title、og:description、og:image、og:url）会导致社交媒体分享时显示不完整或默认信息。每个 `page.tsx` 必须导出 `generateMetadata` 函数。

### 5. 禁止商品列表 CSR-only，必须 SSR + ISR 缓存（revalidate < 30s，热点 < 5s）

**原因**：CSR-only 的商品列表会导致首屏白屏、SEO 不收录、Lighthouse 评分极低。电商商品列表必须 SSR 首屏渲染 + ISR 增量静态再生成。常规商品 revalidate 间隔 < 30s，热点商品（秒杀、爆款）revalidate 间隔 < 5s。使用 `fetch` 的 `next: { revalidate }` 选项实现。

### 6. 禁止客户端直连后端 API，必须 route handler 代理（含 SSRF 防护）

**原因**：客户端直连后端 API 会暴露后端地址、绕过安全策略、无法统一日志和监控、容易被恶意利用。所有客户端请求必须通过 Next.js Route Handler 代理。Route Handler 必须实现 SSRF 防护：URL 白名单校验、禁止内网 IP（127.0.0.1、10.x、192.168.x、172.16.x）、禁止协议切换（只允许 https）。

### 7. 禁止 Lighthouse 性能评分不达标

**原因**：电商官网的性能直接影响转化率和 SEO 排名。发布前必须通过 Lighthouse CI 验证。代码层面必须遵守以下可执行约束：
- 禁止不使用 `next/image` 渲染商品图片（自动 WebP/AVIF、响应式、懒加载）
- 禁止页面不导出 `generateMetadata`（SEO 元信息基础要求）
- 禁止首屏加载超过 3 个阻塞资源
- 目标指标：LCP < 2.5s, CLS < 0.1, TTFB < 800ms, FCP < 1.2s

### 8. 禁止使用 `any` 类型，必须 TypeScript 严格模式

**原因**：`any` 类型破坏了 TypeScript 的类型安全，无法在编译期发现类型错误，导致运行时崩溃。`tsconfig.json` 必须启用 `strict: true`、`noImplicitAny: true`、`strictNullChecks: true`。所有 API 响应必须有显式类型定义，由 OpenAPI 生成。

### 9. 禁止安全头缺失（CSP, X-Frame-Options, X-Content-Type-Options, HSTS）

**原因**：缺少安全头的电商站点容易遭受 XSS 攻击、点击劫持、MIME 类型嗅探、中间人攻击。必须在 `middleware.ts` 或 `next.config.ts` 中配置完整的安全头：
- `Content-Security-Policy`：限制脚本、样式、图片等资源来源
- `X-Frame-Options: DENY` 或 `SAMEORIGIN`：防止点击劫持
- `X-Content-Type-Options: nosniff`：防止 MIME 嗅探
- `Strict-Transport-Security`：强制 HTTPS（max-age >= 31536000）

## 模式推荐

| 场景 | 推荐模式 | 说明 |
|------|----------|------|
| SSR 数据获取 | Server Components + `fetch` | 在服务端直接获取数据，SEO 友好 |
| ISR 缓存 | `fetch` 的 `next: { revalidate }` 选项 | 常规商品 < 30s，热点 < 5s |
| i18n 路由 | next-intl 中间件 + `[locale]` 路由组 | 自动语言检测、路由本地化 |
| 图片优化 | `next/image` + CDN | 自动 WebP/AVIF 转换、响应式、懒加载 |
| 表单处理 | React Hook Form + shadcn/ui Form 组件 | 客户端表单校验 |
| 支付流程 | Route Handler 代理 + 后端签名 | 客户端 → Route Handler → 后端 |
| SEO | `generateMetadata` + 结构化数据（JSON-LD） | 商品详情页必须包含 Product Schema |

## 常见坑

### ISR 缓存击穿

**问题**：热点商品在缓存过期瞬间大量请求同时到达，导致缓存击穿、数据库压力骤增。

**解决方案**：
- revalidate 时间带 jitter（随机抖动），避免所有商品同时过期
- 使用 `stale-while-revalidate` 策略：返回过期缓存的同时后台刷新
- 热点商品（秒杀、活动商品）单独设置更短的 revalidate 间隔（< 5s）
- 缓存层级：Next.js ISR → Redis → 数据库，多级缓存防护

### SSRF 防护

**问题**：Route Handler 代理用户传入的 URL 时，可能请求到内网服务。

**解决方案**：
- 客户端请求必须经 Route Handler 代理，禁止直连后端
- Route Handler 校验目标 URL：白名单域名校验 + 内网 IP 拦截 + 协议限制（仅 https）
- 后端也需要二次校验：URL 白名单 + DNS 解析后 IP 校验（防止 DNS 重绑定攻击）
- 参考 `knowledge/ecommerce-patterns/image-cdn-pattern.md` 中的图片 URL 白名单策略

### 商品列表 CSR-only 导致 SEO 失效

**问题**：在 Client Component 中使用 `useEffect` 获取商品数据并渲染，搜索引擎爬虫无法获取内容。

**解决方案**：
- 商品列表和详情页必须使用 Server Component，在 `page.tsx` 中直接 `await fetch(...)`
- 数据获取逻辑写在 Server Component 中，传递给 Client Component 展示
- 使用 `generateMetadata` 动态生成 SEO 元信息

### Lighthouse 评分优化

**问题**：商品详情页图片过多导致 LCP 超标。

**解决方案**：
- 首屏图片使用 `priority` 属性的 `next/image`，预加载关键资源
- 非首屏图片使用 `loading="lazy"` 懒加载
- 图片使用 CDN 并按设备 DPR 输出合适尺寸（`sizes` 属性）
- 参考 `knowledge/ecommerce-patterns/image-cdn-pattern.md`

### 大文本搜索性能

**问题**：前端搜索商品时，大量数据导致主线程阻塞。

**解决方案**：
- 搜索走后端 API（由 Route Handler 代理），利用 `knowledge/ecommerce-patterns/search-patterns.md` 中的搜索策略
- 商品 < 5000 时使用 PG full-text search，>= 5000 时使用 Elasticsearch
- 前端搜索框使用 debounce（300ms）减少请求频率
- 热门搜索走 Redis 缓存（5min TTL）

## 与其他 Skill 的关联

- **storeforge-domain-backend**：API 类型由后端 OpenAPI spec 生成，Route Handler 调用后端 gRPC/HTTP 服务
- **storeforge-testing**：Website 端使用 `vitest` 做核心逻辑测试，覆盖率 ≥ 80%；L3 E2E 测试覆盖官网黄金路径
- **storeforge-review**：security-auditor 子代理会检查 SSRF 防护和安全头配置；api-contract-checker 会校验前后端字段对齐
- **storeforge-harness**：本 skill 的 9 条反模式自动加载到 harness 检查清单，每个子任务完成后自动检查
- **storeforge-executing**：子任务开发完成后，必须通过 harness 约束检查才能进入 L1 测试
- **storeforge-verification**：harness 检查结果写入验证清单，BLOCK 违规项必须修复
- **knowledge/ecommerce-patterns/image-cdn-pattern.md**：CDN 策略、WebP/AVIF 转换、防 SSRF 图片 URL 白名单
- **knowledge/ecommerce-patterns/search-patterns.md**：搜索策略、PG full-text vs Elasticsearch、Redis 缓存热门搜索
