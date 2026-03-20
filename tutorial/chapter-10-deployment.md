# 第10章：部署与工程化 — 怎么让全世界都能访问？

## 学习目标

- 理解环境变量的管理和验证（t3-env）
- 理解 Vercel 部署的流程
- 理解 PlanetScale（MySQL）的使用
- 理解代码质量工具（ESLint、Prettier、CommitLint）
- 理解 SEO 优化（metadata、OG 图片）
- 理解性能优化（Server Components、Loading 状态、骨架屏）

## 涉及文件

| 文件 | 说明 |
|------|------|
| `env.mjs` | T3 环境变量验证 |
| `.env.example` | 环境变量模板 |
| `next.config.mjs` | Next.js 生产配置 |
| `.eslintrc.json` | ESLint 配置 |
| `prettier.config.js` | Prettier 配置 |
| `.commitlintrc.json` | Commit 消息规范 |
| `app/api/og/route.tsx` | OG 图片动态生成 |
| `components/analytics.tsx` | Vercel Analytics |
| `components/tailwind-indicator.tsx` | 开发环境断点指示器 |

---

## 1. 环境变量管理

### 1.1 环境变量是什么？

环境变量是应用运行时需要的配置信息，通常包含敏感数据（密码、API 密钥等），不应该提交到代码仓库。

```bash
# .env（本地开发使用，不要提交到 Git）
DATABASE_URL="mysql://user:password@host:3306/dbname"
NEXTAUTH_SECRET="super-secret-key"
STRIPE_API_KEY="sk_test_xxx..."
NEXT_PUBLIC_APP_URL="http://localhost:3000"
```

### 1.2 `.env.example` — 环境变量模板

这个文件告诉其他开发者需要设置哪些环境变量，但不包含实际的值：

```bash
# .env.example（可以安全提交到 Git）
DATABASE_URL=
NEXTAUTH_SECRET=
STRIPE_API_KEY=
NEXT_PUBLIC_APP_URL=
```

新开发者克隆项目后，复制这个文件为 `.env` 并填入实际的值。

### 1.3 `env.mjs` — 运行时验证

第1章介绍过，`env.mjs` 使用 `@t3-oss/env-nextjs`（t3-env）在应用启动时验证所有环境变量：

```javascript
// env.mjs
import { createEnv } from "@t3-oss/env-nextjs"
import { z } from "zod"

export const env = createEnv({
  server: {
    DATABASE_URL: z.string().min(1),          // 必须设置且不为空
    NEXTAUTH_SECRET: z.string().min(1),
    STRIPE_API_KEY: z.string().min(1),
    // ...
  },
  client: {
    NEXT_PUBLIC_APP_URL: z.string().min(1),   // 客户端也能访问
  },
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    NEXTAUTH_SECRET: process.env.NEXTAUTH_SECRET,
    // ... 每个变量需要显式映射
  },
})
```

**在 `next.config.mjs` 中导入了 `env.mjs`：**

```javascript
// next.config.mjs
import "./env.mjs"  // 启动时就会执行验证
```

这意味着如果有环境变量缺失，`pnpm dev` 或 `pnpm build` 会**立即报错**，而不是等到用户触发某个功能时才崩溃。

---

## 2. 部署到 Vercel

### 2.1 什么是 Vercel？

**Vercel** 是 Next.js 的官方部署平台（Next.js 就是 Vercel 公司开发的）。它提供：
- 自动构建和部署
- 全球 CDN（让全世界用户快速访问）
- 无服务器函数（API Routes 自动部署为 Serverless Function）
- 预览部署（每个 PR 自动生成一个预览 URL）

### 2.2 部署流程

```
1. 将代码推送到 GitHub
       ↓
2. Vercel 检测到新的提交
       ↓
3. Vercel 运行 pnpm build（包括 contentlayer build && next build）
       ↓
4. 构建成功 → 自动部署到 https://your-app.vercel.app
       ↓
5. 配置环境变量（在 Vercel Dashboard 中设置）
       ↓
6. 配置自定义域名（可选）
```

### 2.3 PlanetScale — 云端 MySQL

Taxonomy 使用 **PlanetScale** 作为数据库。PlanetScale 是一个基于 MySQL 的云数据库服务，特点是：
- 基于 Vitess（YouTube 使用的数据库技术）
- 支持数据库分支（类似 Git 分支，可以在不影响生产的情况下修改数据库结构）
- 免费额度足够个人项目使用

数据库连接通过 `DATABASE_URL` 环境变量配置。

---

## 3. 代码质量工具

### 3.1 ESLint — 代码规范检查

ESLint 检查代码中的问题（如未使用的变量、不安全的写法）：

```json
// .eslintrc.json
{
  "$schema": "https://json.schemastore.org/eslintrc",
  "root": true,
  "extends": [
    "next/core-web-vitals",      // Next.js 推荐的规则
    "prettier"                    // 关闭与 Prettier 冲突的规则
  ],
  "plugins": [
    "tailwindcss"                 // Tailwind CSS 相关规则
  ],
  "rules": {
    "tailwindcss/classnames-order": "warn",        // 警告：Tailwind 类名顺序
    "tailwindcss/no-custom-classname": "off"        // 允许自定义类名
  }
}
```

运行方式：`pnpm lint`

### 3.2 Prettier — 代码格式化

Prettier 自动格式化代码（缩进、引号、分号等），确保团队代码风格一致：

```javascript
// prettier.config.js
module.exports = {
  semi: false,              // 不使用分号
  singleQuote: false,       // 使用双引号
  // 导入排序插件
  importOrder: [
    "^(react/(.*)$)|^(react$)",     // React 优先
    "^(next/(.*)$)|^(next$)",       // Next.js 其次
    "<THIRD_PARTY_MODULES>",         // 第三方模块
    "",
    "^@/(.*)$",                      // 项目内部模块
    "^[./]",                         // 相对路径
  ],
}
```

### 3.3 CommitLint — 提交消息规范

CommitLint 确保 Git 提交消息遵循规范格式：

```json
// .commitlintrc.json
{
  "extends": ["@commitlint/config-conventional"]
}
```

**Conventional Commits 格式：**

```
类型(范围): 描述

feat(auth): add GitHub OAuth login
fix(api): handle null response from Stripe
docs(readme): update installation instructions
refactor(db): simplify Prisma client initialization
```

常用类型：
- `feat` — 新功能
- `fix` — 修复 Bug
- `docs` — 文档变更
- `refactor` — 重构（不改功能）
- `style` — 代码格式（不影响运行）
- `test` — 添加测试

### 3.4 其他配置文件

```
.editorconfig    → 统一不同编辑器的设置（缩进、换行符等）
.nvmrc           → 锁定 Node.js 版本（确保团队用同一版本）
```

---

## 4. SEO 优化

### 4.1 Metadata（元数据）

每个页面都可以导出 `metadata` 对象来设置 SEO 信息：

```tsx
// app/layout.tsx — 全站默认元数据
export const metadata = {
  title: {
    default: "Taxonomy",                // 默认标题
    template: `%s | Taxonomy`,          // 子页面模板：%s 会被替换
  },
  description: "An open source application...",
  keywords: ["Next.js", "React", "Tailwind CSS"],
  openGraph: {                           // 社交媒体分享卡片
    type: "website",
    title: "Taxonomy",
    description: "...",
    images: ["/og.jpg"],
  },
  twitter: {                             // Twitter 卡片
    card: "summary_large_image",
    title: "Taxonomy",
  },
}
```

```tsx
// app/(dashboard)/dashboard/page.tsx — 页面级元数据
export const metadata = {
  title: "Dashboard",  // 实际标题会是 "Dashboard | Taxonomy"
}
```

### 4.2 `robots.ts` — 搜索引擎爬虫规则

```typescript
// app/robots.ts
import { MetadataRoute } from "next"

export default function robots(): MetadataRoute.Robots {
  return {
    rules: {
      userAgent: "*",    // 针对所有搜索引擎爬虫
      allow: "/",        // 允许抓取所有页面
    },
  }
}
```

这个文件会自动生成 `/robots.txt`。搜索引擎爬虫（如 Google Bot）访问网站时，会先读取 `robots.txt` 确认哪些页面可以抓取。这里配置为允许抓取所有页面。

如果想禁止抓取某些页面（如后台管理页），可以添加 `disallow` 规则：

```typescript
rules: [
  { userAgent: "*", allow: "/" },
  { userAgent: "*", disallow: "/dashboard" },
]
```

### 4.3 OG 图片动态生成

`app/api/og/route.tsx` 使用 `@vercel/og` 动态生成社交分享图片。

当你分享一个链接到 Twitter/Slack 时，会显示一个预览图片——这就是 **OG（Open Graph）图片**。

```tsx
// app/api/og/route.tsx（简化版）
import { ImageResponse } from "@vercel/og"
import { ogImageSchema } from "@/lib/validations/og"

export const runtime = "edge"  // 在 Edge Runtime 运行（全球分布，更快）

// 预加载字体文件
const interRegular = fetch(
  new URL("../../../assets/fonts/Inter-Regular.ttf", import.meta.url)
).then((res) => res.arrayBuffer())

const interBold = fetch(
  new URL("../../../assets/fonts/CalSans-SemiBold.ttf", import.meta.url)
).then((res) => res.arrayBuffer())

export async function GET(req: Request) {
  const fontRegular = await interRegular
  const fontBold = await interBold

  // 从 URL 参数获取标题和模式
  const url = new URL(req.url)
  const values = ogImageSchema.parse(Object.fromEntries(url.searchParams))
  const heading = values.heading.length > 140
    ? `${values.heading.substring(0, 140)}...`
    : values.heading

  const { mode } = values  // "dark" 或 "light"
  const paint = mode === "dark" ? "#fff" : "#000"

  // 用 JSX 生成图片！
  return new ImageResponse(
    <div
      tw="flex relative flex-col p-12 w-full h-full items-start"
      style={{ color: paint, background: mode === "dark" ? "#000" : "white" }}
    >
      {/* Logo SVG */}
      <svg>...</svg>
      <div tw="flex leading-[1.1] text-[80px] font-bold">
        {heading}
      </div>
    </div>,
    {
      width: 1200, height: 630,  // 标准 OG 图片尺寸
      fonts: [
        { name: "Inter", data: fontRegular, weight: 400 },
        { name: "Cal Sans", data: fontBold, weight: 700 },
      ],
    }
  )
}
// 访问 /api/og?heading=Hello&type=Blog&mode=dark → 生成一张暗色预览图
```

**Zod 验证参数：**

```typescript
// lib/validations/og.ts
export const ogImageSchema = z.object({
  heading: z.string(),
  type: z.string(),
  mode: z.enum(["light", "dark"]).default("dark"),
})
```

**关键点：**
- **Edge Runtime**：`export const runtime = "edge"` 让这个 API 在 Vercel 的 Edge 网络运行，全球分布，响应速度快
- **`@vercel/og`** 使用 Satori 引擎渲染，用 `tw` 属性写 Tailwind 类名（不是 `className`）
- 字体文件需要预加载为 `ArrayBuffer` 传入
- 支持亮色/暗色两种模式

---

## 5. 性能优化

### 5.1 Server Components（默认）

Next.js 13 的最大性能优化就是 **Server Components**。回顾前面学过的：

```
传统 React 应用：
  服务器发送空 HTML → 浏览器下载 JS → JS 运行 → 页面出现
  （用户等待时间长）

Server Components：
  服务器渲染 HTML → 直接发送给浏览器 → 页面立即出现
  （用户几乎不等待）
```

Taxonomy 中大部分页面都是 Server Components，只有需要交互的部分才用 Client Components。

### 5.2 Loading 状态和骨架屏

```tsx
// components/card-skeleton.tsx（假设实现）
export function CardSkeleton() {
  return (
    <div className="p-4">
      <div className="space-y-3">
        <div className="h-5 w-2/5 rounded-lg bg-muted" />   {/* 灰色矩形模拟标题 */}
        <div className="h-4 w-4/5 rounded-lg bg-muted" />   {/* 灰色矩形模拟内容 */}
      </div>
    </div>
  )
}
```

使用 `loading.tsx`，在数据加载期间自动显示骨架屏，而不是空白页面。

### 5.3 Vercel Analytics

```tsx
// components/analytics.tsx
"use client"

import { Analytics as VercelAnalytics } from "@vercel/analytics/react"

export function Analytics() {
  return <VercelAnalytics />
}
```

这个组件在 `app/layout.tsx` 中被引入，自动收集所有页面的性能数据：

```tsx
// app/layout.tsx（相关部分）
import { Analytics } from "@/components/analytics"

export default function RootLayout({ children }) {
  return (
    <html>
      <body>
        {children}
        <Analytics />   {/* 放在 body 末尾，不影响页面渲染 */}
      </body>
    </html>
  )
}
```

**注意事项：**
- 组件是 `"use client"` 因为 Vercel Analytics SDK 需要浏览器 API
- 只在部署到 Vercel 后才有数据，本地开发不会收集
- 在 Vercel Dashboard → Analytics 页面查看数据（页面浏览量、Web Vitals 等）

### 5.4 TailwindIndicator — 开发专用

```tsx
// components/tailwind-indicator.tsx
export function TailwindIndicator() {
  if (process.env.NODE_ENV === "production") return null  // 生产环境不显示

  return (
    <div className="fixed bottom-1 left-1 z-50 flex h-6 w-6 items-center justify-center rounded-full bg-gray-800 p-3 font-mono text-xs text-white">
      <div className="block sm:hidden">xs</div>
      <div className="hidden sm:block md:hidden">sm</div>
      <div className="hidden md:block lg:hidden">md</div>
      <div className="hidden lg:block xl:hidden">lg</div>
      <div className="hidden xl:block 2xl:hidden">xl</div>
      <div className="hidden 2xl:block">2xl</div>
    </div>
  )
}
```

`process.env.NODE_ENV === "production"` 时返回 null——**这段代码在生产构建时会被完全移除**，不会增加任何包体积。

---

## 6. 构建和部署检查清单

部署前确保：

```
☐ 所有环境变量已设置（对照 .env.example）
☐ 数据库已迁移（npx prisma migrate deploy）
☐ 代码通过 lint 检查（pnpm lint）
☐ 构建成功（pnpm build）
☐ Stripe Webhook URL 已配置为生产域名
☐ NextAuth 回调 URL 已更新
☐ NEXT_PUBLIC_APP_URL 设置为生产域名
```

---

## 7. 项目技术栈全景回顾

经过 10 章的学习，让我们回顾整个项目的技术栈：

```
┌──────────────────────────────────────────────┐
│                    前端                       │
│  React 18 + Next.js 13 (App Router)          │
│  Tailwind CSS + Radix UI (shadcn/ui)         │
│  Editor.js (富文本编辑器)                      │
│  react-hook-form + Zod (表单验证)              │
├──────────────────────────────────────────────┤
│                   内容层                      │
│  Contentlayer (MDX → TypeScript)             │
│  rehype/remark 插件 (语法高亮、自动链接)        │
├──────────────────────────────────────────────┤
│                   API 层                      │
│  Next.js Route Handlers                      │
│  Zod (请求验证)                               │
├──────────────────────────────────────────────┤
│                   认证层                      │
│  NextAuth.js (OAuth + Email)                 │
│  JWT 策略                                     │
│  中间件路由保护                                │
├──────────────────────────────────────────────┤
│                   数据层                      │
│  Prisma ORM                                  │
│  PlanetScale (MySQL)                         │
├──────────────────────────────────────────────┤
│                   支付层                      │
│  Stripe (Checkout + Webhook)                 │
├──────────────────────────────────────────────┤
│                  部署/工具                     │
│  Vercel (部署 + Analytics)                    │
│  ESLint + Prettier (代码质量)                 │
│  CommitLint + Husky (提交规范)                │
│  t3-env (环境变量验证)                         │
└──────────────────────────────────────────────┘
```

---

## 8. 下一步学习建议

恭喜你完成了 10 章的学习！以下是一些进阶方向：

### 动手实践
1. **Fork Taxonomy**，尝试添加新功能（如帖子评论系统）
2. **修改订阅计划**，添加第三个计划（如 Team 计划）
3. **添加新的内容类型**，在 `contentlayer.config.js` 中定义
4. **创建新的 UI 组件**，参考 shadcn/ui 的模式

### 深入学习
- **Next.js 官方文档**：https://nextjs.org/docs
- **Prisma 官方文档**：https://www.prisma.io/docs
- **Tailwind CSS 文档**：https://tailwindcss.com/docs
- **shadcn/ui 组件库**：https://ui.shadcn.com
- **Stripe 文档**：https://stripe.com/docs

### 构建自己的项目
用 Taxonomy 作为起点，构建你自己的 SaaS 应用！建议的项目想法：
- 在线笔记应用
- 任务管理工具
- 博客平台
- 简历生成器

---

## 小练习

### 练习 1：检查构建
运行 `pnpm build`，观察输出。注意哪些页面是静态生成的（SSG），哪些是服务端渲染的（SSR）。

### 练习 2：添加新的 lint 规则
在 `.eslintrc.json` 中添加一条规则，比如禁止使用 `console.log`：
```json
"rules": {
  "no-console": "warn"
}
```
然后运行 `pnpm lint`，看看有多少个警告。

### 练习 3：创建一个环境变量
1. 在 `env.mjs` 中添加一个新的环境变量定义
2. 在 `.env.example` 中添加对应的模板
3. 在 `.env` 中设置实际值
4. 在某个组件中使用 `env.YOUR_NEW_VAR`

### 练习 4：性能分析
1. 运行 `pnpm build`
2. 查看构建输出中的页面大小
3. 想想哪些页面最大，为什么？
4. 如何减小页面大小？

---

## 本章小结

- 环境变量用 `.env` 文件管理，`t3-env` 在启动时验证
- Vercel 提供零配置的 Next.js 部署
- ESLint、Prettier、CommitLint 保证代码质量和团队协作
- SEO 通过 metadata 和 OG 图片优化
- Server Components 是最大的性能优化
- Loading 状态和骨架屏改善用户体验

---

## 全教程总结

通过全部章节的学习，你已经掌握了一个现代全栈 Web 应用的所有核心概念：

| 章节 | 主题 | 关键技术 |
|------|------|---------|
| 第1章 | 项目全景 | package.json, 目录结构, 配置文件 |
| 第2章 | Next.js 基础 | App Router, Server/Client Components |
| 第3章 | UI 组件 | shadcn/ui, Tailwind CSS, cva |
| 第3.5章 | 样式系统 | CSS 变量主题, next/font, 自定义 Hooks |
| 第4章 | 布局导航 | 嵌套布局, 响应式设计, 暗色模式 |
| 第5章 | 内容系统 | MDX, Contentlayer, rehype/remark, TOC |
| 第6章 | 数据库 | Prisma ORM, Schema, CRUD |
| 第7章 | 认证 | NextAuth.js, OAuth, JWT, 中间件 |
| 第8章 | API 层 | Route Handlers, Zod, fetch |
| 第8.5章 | 文章编辑器 | EditorJS, 动态导入, 文章生命周期 |
| 第9章 | 付费系统 | Stripe, Checkout, Webhook |
| 第10章 | 部署工程化 | Vercel, ESLint, SEO, 性能 |

**最重要的一点：** 不要只是阅读——打开编辑器，跟着代码一行一行地看，尝试修改，运行 `pnpm dev` 观察效果。编程是一门实践技能，只有动手才能真正掌握。

祝你学习愉快！
