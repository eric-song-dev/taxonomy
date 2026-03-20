# Day 10 — 部署与工程化：让全世界都能访问你的应用

> 对应章节：[Chapter 10: 部署与工程化](../chapter-10-deployment.md)

---

## 今日学习目标

1. 理解环境变量的作用和 `t3-env` 验证机制——知道为什么不能把密码写在代码里
2. 掌握 ESLint / Prettier 等代码质量工具的配置和使用
3. 了解 SEO 优化（metadata、OG Image、robots.ts）和 Vercel 部署流程

---

## 一、前置知识速补

> 今天的内容会涉及一些新概念，我们先花几分钟快速过一遍。

### 1.1 环境变量（`process.env`）— 为什么不把密码写在代码里

想象你写了一段代码：

```javascript
// 千万不要这样做！
const password = "abc123"
```

如果你把代码放到 GitHub 上，全世界都能看到你的密码。这就是为什么我们需要**环境变量**。

**环境变量**是存储在你电脑（或服务器）上的"隐藏配置"，代码通过 `process.env.变量名` 来读取它们：

```javascript
// 正确做法：从环境变量读取
const password = process.env.DATABASE_PASSWORD
// process 是 Node.js 提供的全局对象
// env 是它的一个属性，里面存着所有环境变量
// DATABASE_PASSWORD 是我们自己起的变量名
```

环境变量通常写在一个叫 `.env` 的文件里，这个文件**不会**被上传到 GitHub（因为 `.gitignore` 会忽略它）。

### 1.2 构建流程（source code -> build -> production）

你写的代码（源码）不能直接放到网上让用户访问。需要经过一个"构建"过程：

```
源码（你写的 .tsx / .ts 文件）
    ↓  pnpm build（构建命令）
打包后的文件（优化过的 HTML / CSS / JS）
    ↓  部署到 Vercel
用户可以访问的网站
```

就像你写了一份手稿（源码），需要排版、印刷（构建），然后送到书店（部署），读者才能买到。

### 1.3 模块系统（CommonJS vs ESM）

JavaScript 有两种"导入其他文件"的写法，你在项目里会同时看到：

```javascript
// 写法一：CommonJS（老写法，Node.js 最早支持的）
// 文件后缀通常是 .js
const fs = require("fs")        // require 是"引入"的意思
module.exports = { ... }        // module.exports 是"导出"的意思

// 写法二：ESM（新写法，现在推荐用的）
// 文件后缀可以是 .mjs 或 .ts
import fs from "fs"             // import 是"引入"的意思
export default { ... }          // export 是"导出"的意思
```

为什么项目里有些文件用 `.mjs` 后缀？因为 `.mjs` 告诉 Node.js "这个文件用 ESM 写法"。比如 `env.mjs`、`next.config.mjs` 都是这样。

### 1.4 JSON.stringify / JSON.parse

这两个方法在配置文件处理中经常出现：

```javascript
// JSON.stringify：把 JavaScript 对象变成字符串
const obj = { name: "小明", age: 18 }
const str = JSON.stringify(obj)
// str 的值是 '{"name":"小明","age":18}'

// JSON.parse：把字符串变回 JavaScript 对象
const obj2 = JSON.parse(str)
// obj2 的值是 { name: "小明", age: 18 }
```

ESLint、CommitLint 等工具的配置文件（`.eslintrc.json`、`.commitlintrc.json`）都是 JSON 格式，Node.js 读取它们时内部就用到了 `JSON.parse`。

### 1.5 正则表达式在配置中的使用

在 `prettier.config.js` 中你会看到类似这样的写法：

```javascript
importOrder: [
  "^(react/(.*)$)|^(react$)",   // 以 react 开头的导入
  "^(next/(.*)$)|^(next$)",     // 以 next 开头的导入
  "^@/(.*)$",                   // 以 @/ 开头的（项目内部文件）
  "^[./]",                      // 以 . 或 / 开头的（相对路径）
]
```

这些 `^`、`$`、`(.*)` 就是**正则表达式**（regex），用来描述"匹配什么样的文本"：
- `^` 表示"以...开头"
- `$` 表示"以...结尾"
- `(.*)` 表示"任意内容"
- `|` 表示"或者"

你不需要完全看懂，只要知道它们的作用是"按类别排列 import 语句的顺序"就够了。

---

## 二、核心概念清单

| 概念 | 一句话说明 | 对应文件 |
|------|-----------|---------|
| [Vercel](https://vercel.com/docs) | Next.js 官方部署平台，推送代码自动上线 | — |
| 环境变量 | 不写在代码里的敏感配置（密码、密钥等） | `.env` / `.env.example` |
| [t3-env](https://env.t3.gg/) | 启动时验证环境变量是否齐全，缺了就报错 | `env.mjs` |
| [ESLint](https://eslint.org/docs/latest/) | 代码"语法警察"，检查潜在问题 | `.eslintrc.json` |
| [Prettier](https://prettier.io/docs/en/) | 代码"美容师"，统一格式风格 | `prettier.config.js` |
| CommitLint | Git 提交信息规范检查（[Conventional Commits](https://www.conventionalcommits.org/)） | `.commitlintrc.json` |
| [SEO metadata](https://nextjs.org/docs/app/building-your-application/optimizing/metadata) | 告诉搜索引擎这个页面是关于什么的 | `app/layout.tsx` |
| OG Image | 分享链接到社交媒体时显示的预览图 | `app/api/og/route.tsx` |
| [robots.ts](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/robots) | 告诉搜索引擎爬虫哪些页面可以抓取 | `app/robots.ts` |
| [sitemap](https://nextjs.org/docs/app/api-reference/file-conventions/metadata/sitemap) | 网站地图，帮助搜索引擎发现所有页面 | 本项目未配置 |

---

## 三、源码精读

### 3.1 部署配置

#### `.env.example` — 环境变量模板

打开 `.env.example`，你会看到项目需要的所有环境变量：

```bash
# .env.example（可以安全提交到 GitHub，因为没有真实密码）

# ---- App ----
NEXT_PUBLIC_APP_URL=http://localhost:3000
# NEXT_PUBLIC_ 开头的变量，浏览器端也能访问
# 没有这个前缀的变量，只有服务器端才能访问

# ---- 认证 ----
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
GITHUB_ACCESS_TOKEN=

# ---- 数据库 ----
DATABASE_URL="mysql://root:root@localhost:3306/taxonomy?schema=public"

# ---- 邮件 ----
SMTP_FROM=
POSTMARK_API_TOKEN=
POSTMARK_SIGN_IN_TEMPLATE=
POSTMARK_ACTIVATION_TEMPLATE=

# ---- 支付 ----
STRIPE_API_KEY=
STRIPE_WEBHOOK_SECRET=
STRIPE_PRO_MONTHLY_PLAN_ID=
```

**要点：**
- 这个文件是"模板"——告诉新成员"你需要配置这些变量"
- 真实的值写在 `.env` 文件里（`.env` 被 `.gitignore` 忽略，不会上传到 GitHub）
- 新成员拿到代码后，复制 `.env.example` 为 `.env`，然后填入自己的值

#### `env.mjs` — 运行时验证（t3-env）

这是整个环境变量管理的核心。我们逐行读：

```javascript
// env.mjs

import { createEnv } from "@t3-oss/env-nextjs"
// 从 t3-env 库导入 createEnv 函数

import { z } from "zod"
// 从 zod 库导入 z，用来定义"验证规则"

export const env = createEnv({
  // 服务器端环境变量（只有后端代码能用）
  server: {
    NEXTAUTH_URL: z.string().url().optional(),
    // z.string()  → 必须是字符串
    // .url()      → 必须是合法的网址格式
    // .optional() → 可以不填（开发环境可选）

    NEXTAUTH_SECRET: z.string().min(1),
    // .min(1)     → 至少 1 个字符（即不能为空）

    GITHUB_CLIENT_ID: z.string().min(1),
    GITHUB_CLIENT_SECRET: z.string().min(1),
    GITHUB_ACCESS_TOKEN: z.string().min(1),
    DATABASE_URL: z.string().min(1),
    SMTP_FROM: z.string().min(1),
    POSTMARK_API_TOKEN: z.string().min(1),
    POSTMARK_SIGN_IN_TEMPLATE: z.string().min(1),
    POSTMARK_ACTIVATION_TEMPLATE: z.string().min(1),
    STRIPE_API_KEY: z.string().min(1),
    STRIPE_WEBHOOK_SECRET: z.string().min(1),
    STRIPE_PRO_MONTHLY_PLAN_ID: z.string().min(1),
  },

  // 客户端环境变量（浏览器也能用）
  client: {
    NEXT_PUBLIC_APP_URL: z.string().min(1),
    // 注意：客户端变量名必须以 NEXT_PUBLIC_ 开头
    // 这是 Next.js 的安全规则，防止你不小心把密码暴露给浏览器
  },

  // 把 process.env 的值映射过来
  runtimeEnv: {
    NEXTAUTH_URL: process.env.NEXTAUTH_URL,
    NEXTAUTH_SECRET: process.env.NEXTAUTH_SECRET,
    GITHUB_CLIENT_ID: process.env.GITHUB_CLIENT_ID,
    GITHUB_CLIENT_SECRET: process.env.GITHUB_CLIENT_SECRET,
    GITHUB_ACCESS_TOKEN: process.env.GITHUB_ACCESS_TOKEN,
    DATABASE_URL: process.env.DATABASE_URL,
    SMTP_FROM: process.env.SMTP_FROM,
    POSTMARK_API_TOKEN: process.env.POSTMARK_API_TOKEN,
    POSTMARK_SIGN_IN_TEMPLATE: process.env.POSTMARK_SIGN_IN_TEMPLATE,
    POSTMARK_ACTIVATION_TEMPLATE: process.env.POSTMARK_ACTIVATION_TEMPLATE,
    STRIPE_API_KEY: process.env.STRIPE_API_KEY,
    STRIPE_WEBHOOK_SECRET: process.env.STRIPE_WEBHOOK_SECRET,
    STRIPE_PRO_MONTHLY_PLAN_ID: process.env.STRIPE_PRO_MONTHLY_PLAN_ID,
    NEXT_PUBLIC_APP_URL: process.env.NEXT_PUBLIC_APP_URL,
  },
})
```

**为什么需要 `runtimeEnv`？** 因为 Next.js 在构建时会把 `process.env.XXX` 替换成实际的值。`t3-env` 需要你显式列出每个变量，确保不会遗漏。

#### `next.config.mjs` — Next.js 生产配置

```javascript
// next.config.mjs

import { withContentlayer } from "next-contentlayer"
// 导入 Contentlayer 的 Next.js 插件（第 5 章学过的内容系统）

import "./env.mjs"
// 关键！导入 env.mjs，这样每次启动/构建时都会执行环境变量验证
// 如果有变量缺失，这里就会报错，不会等到用户触发时才崩溃

/** @type {import('next').NextConfig} */
const nextConfig = {
  reactStrictMode: true,
  // 开启 React 严格模式，帮助发现潜在问题

  images: {
    domains: ["avatars.githubusercontent.com"],
    // 允许从 GitHub 加载用户头像
    // Next.js 默认不允许加载外部图片，需要在这里白名单
  },

  experimental: {
    appDir: true,
    // 启用 App Router（Next.js 13 的新路由系统）

    serverComponentsExternalPackages: ["@prisma/client"],
    // 告诉 Next.js 不要把 Prisma 打包进浏览器端代码
    // Prisma 是数据库工具，只在服务器端运行
  },
}

export default withContentlayer(nextConfig)
// withContentlayer 会在构建时处理 MDX 内容
```

### 3.2 代码质量工具

#### `.eslintrc.json` — 代码规范检查

ESLint 就像一个"代码审查员"，自动帮你检查代码中的问题：

```json
{
  "$schema": "https://json.schemastore.org/eslintrc",
  // 这行告诉编辑器这个文件的格式，提供自动补全

  "root": true,
  // "我是项目根目录的配置，不要再往上级目录找了"

  "extends": [
    "next/core-web-vitals",
    // 继承 Next.js 官方推荐的规则（检查性能、可访问性等）

    "prettier",
    // 继承 prettier 配置，关闭与 Prettier 冲突的格式规则
    // 思路：格式化交给 Prettier，ESLint 只管代码质量

    "plugin:tailwindcss/recommended"
    // 继承 Tailwind CSS 推荐的规则
  ],

  "plugins": ["tailwindcss"],
  // 加载 tailwindcss 插件，提供 Tailwind 相关的检查能力

  "rules": {
    "@next/next/no-html-link-for-pages": "off",
    // 关闭"不要用 <a> 标签"的规则

    "react/jsx-key": "off",
    // 关闭"列表元素必须有 key"的规则

    "tailwindcss/no-custom-classname": "off",
    // 允许自定义 CSS 类名（不限制只用 Tailwind 的类）

    "tailwindcss/classnames-order": "error"
    // Tailwind 类名必须按规定顺序排列，否则报错
    // "error" 表示报错（构建会失败）
    // "warn" 表示警告（构建不会失败）
    // "off" 表示关闭这条规则
  },

  "settings": {
    "tailwindcss": {
      "callees": ["cn"],
      // 告诉插件：cn() 函数（项目封装的工具函数）也包含 Tailwind 类名

      "config": "tailwind.config.js"
      // 指定 Tailwind 配置文件的位置
    },
    "next": {
      "rootDir": true
    }
  },

  "overrides": [
    {
      "files": ["*.ts", "*.tsx"],
      "parser": "@typescript-eslint/parser"
      // TypeScript 文件用专门的解析器
    }
  ]
}
```

运行 `pnpm lint` 就可以检查整个项目。

#### `prettier.config.js` — 代码格式化

Prettier 是"代码美容师"，保证团队写出来的代码长得一样：

```javascript
// prettier.config.js

/** @type {import('prettier').Config} */
module.exports = {
  // 注意：这个文件用 CommonJS（module.exports），不是 ESM（export default）
  // 因为 Prettier 目前还主要用 CommonJS 加载配置

  endOfLine: "lf",
  // 换行符用 LF（Linux/Mac 风格），不用 CRLF（Windows 风格）

  semi: false,
  // 不加分号。比如写 const x = 1 而不是 const x = 1;

  singleQuote: false,
  // 用双引号 "hello"，不用单引号 'hello'

  tabWidth: 2,
  // 缩进用 2 个空格

  trailingComma: "es5",
  // 在对象/数组最后一项后面加逗号：{ a: 1, b: 2, }

  importOrder: [
    "^(react/(.*)$)|^(react$)",      // 第一组：React 相关
    "^(next/(.*)$)|^(next$)",        // 第二组：Next.js 相关
    "<THIRD_PARTY_MODULES>",          // 第三组：第三方库
    "",                               // 空行分隔
    "^types$",                        // 第四组：类型定义
    "^@/env(.*)$",                   // 第五组：环境变量
    "^@/types/(.*)$",               // 第六组：类型文件
    "^@/config/(.*)$",              // 第七组：配置文件
    "^@/lib/(.*)$",                 // 第八组：工具函数
    "^@/hooks/(.*)$",               // 第九组：React hooks
    "^@/components/ui/(.*)$",       // 第十组：UI 基础组件
    "^@/components/(.*)$",          // 第十一组：业务组件
    "^@/styles/(.*)$",             // 第十二组：样式文件
    "^@/app/(.*)$",                // 第十三组：页面文件
    "",                              // 空行分隔
    "^[./]",                         // 最后：相对路径的导入
  ],
  // 这套规则自动按类别排列 import 语句，你不用手动整理

  importOrderSeparation: false,
  importOrderSortSpecifiers: true,
  importOrderBuiltinModulesToTop: true,
  importOrderParserPlugins: ["typescript", "jsx", "decorators-legacy"],
  importOrderMergeDuplicateImports: true,
  importOrderCombineTypeAndValueImports: true,

  plugins: ["@ianvs/prettier-plugin-sort-imports"],
  // 加载 import 排序插件
}
```

**ESLint vs Prettier 分工：**
- ESLint 管"代码写得对不对"（未使用的变量、危险写法等）
- Prettier 管"代码长得好不好看"（缩进、引号、分号等）
- 两者可能冲突，所以用 `eslint-config-prettier` 让 Prettier 优先管格式

#### `.commitlintrc.json` — 提交信息规范

```json
{
  "extends": ["@commitlint/config-conventional"]
}
```

只有一行配置，但它要求每次 `git commit` 的信息必须符合格式：

```
类型(范围): 描述

例如：
feat(auth): add GitHub OAuth login     ← 新功能
fix(api): handle null response         ← 修复 Bug
docs(readme): update install guide     ← 文档变更
refactor(db): simplify Prisma client   ← 重构
```

如果你写 `git commit -m "update"`，CommitLint 会拦截并报错。这样做的好处：
- 看 git 记录就知道每次改了什么
- 可以自动生成 changelog（更新日志）

### 3.3 SEO 实现

#### `app/layout.tsx` — 全站 metadata

```tsx
// app/layout.tsx（精简展示关键部分）

export const metadata = {
  title: {
    default: siteConfig.name,              // 默认标题，比如 "Taxonomy"
    template: `%s | ${siteConfig.name}`,   // 子页面模板
    // 比如 Dashboard 页面的标题会变成 "Dashboard | Taxonomy"
    // %s 是占位符，会被子页面的 title 替换
  },
  description: siteConfig.description,
  // 搜索引擎结果里显示的描述文字

  keywords: ["Next.js", "React", "Tailwind CSS", "Server Components", "Radix UI"],
  // 帮助搜索引擎理解页面内容的关键词

  authors: [{ name: "shadcn", url: "https://shadcn.com" }],

  openGraph: {
    type: "website",
    locale: "en_US",
    url: siteConfig.url,
    title: siteConfig.name,
    description: siteConfig.description,
    siteName: siteConfig.name,
  },
  // openGraph：当你把链接分享到 Twitter / Slack / 微信时
  // 这些信息决定了预览卡片上显示什么标题、描述、图片

  themeColor: [
    { media: "(prefers-color-scheme: light)", color: "white" },
    { media: "(prefers-color-scheme: dark)", color: "black" },
  ],
  // 浏览器地址栏的颜色（手机浏览器比较明显）
}
```

#### `app/robots.ts` — 搜索引擎爬虫规则

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
// 这个文件会自动生成 /robots.txt
// 搜索引擎爬虫访问网站时，会先看 robots.txt 确认哪些页面可以抓取
```

#### `app/api/og/route.tsx` — OG 图片动态生成

当你把链接分享到社交媒体时，会显示一张预览图。这个文件就是动态生成那张图片的：

```tsx
// app/api/og/route.tsx（精简讲解）

import { ImageResponse } from "@vercel/og"
// @vercel/og 能用 JSX（类似写网页的语法）来画图片

import { ogImageSchema } from "@/lib/validations/og"
// 用 Zod schema 验证传入的参数

export const runtime = "edge"
// 在 Vercel 的 Edge 网络运行（全球分布，速度更快）

export async function GET(req: Request) {
  // 1. 加载字体文件
  const fontRegular = await interRegular
  const fontBold = await interBold

  // 2. 从 URL 参数获取标题
  const url = new URL(req.url)
  const values = ogImageSchema.parse(Object.fromEntries(url.searchParams))

  // 3. 用 JSX 画出图片
  return new ImageResponse(
    <div tw="flex relative flex-col p-12 w-full h-full items-start"
         style={{ color: paint, background: "white" }}>
      <div tw="flex leading-[1.1] text-[80px] font-bold">
        {heading}
      </div>
    </div>,
    { width: 1200, height: 630 }  // 标准 OG 图片尺寸
  )
}
// 访问 /api/og?heading=Hello&type=Blog → 生成一张预览图
```

**[Zod](https://zod.dev) 验证参数：** `lib/validations/og.ts` 定义了 OG 图片的参数验证：

```typescript
// lib/validations/og.ts
export const ogImageSchema = z.object({
  heading: z.string(),
  type: z.string(),
  mode: z.enum(["light", "dark"]).default("dark"),
})
```

**关键点：**
- `export const runtime = "edge"` 表示运行在 Vercel Edge 网络（全球分布，毫秒级响应）
- 字体文件通过 `fetch` + `arrayBuffer()` 预加载，传入 `ImageResponse` 的 `fonts` 选项
- 支持亮色/暗色两种模式，由 URL 参数 `mode` 控制
- `@vercel/og` 使用 Satori 引擎渲染，只支持有限的 CSS（不能用 `className`，要用 `tw` 属性或内联 `style`）

---

## 四、概念图解

### 4.1 部署流水线

```
你的电脑                     GitHub                    Vercel
┌──────────┐    git push    ┌──────────┐   自动触发   ┌──────────────────┐
│ 写代码    │ ──────────→  │ 代码仓库  │ ──────────→ │ 1. 安装依赖       │
│ pnpm dev  │               │          │              │    pnpm install   │
│ 本地测试  │               │          │              │ 2. 验证环境变量    │
└──────────┘               └──────────┘              │    (env.mjs)      │
                                                      │ 3. 构建           │
                                                      │    pnpm build     │
                                                      │ 4. 部署到全球 CDN │
                                                      └────────┬─────────┘
                                                               ↓
                                                      ┌──────────────────┐
                                                      │ https://你的域名  │
                                                      │ 全世界用户可访问  │
                                                      └──────────────────┘
```

### 4.2 构建过程详解

```
pnpm build 执行时发生了什么？

Step 1: env.mjs 验证环境变量
        ↓ 缺变量？→ 立即报错，构建终止
Step 2: contentlayer build（把 MDX 文件转成 TypeScript 数据）
        ↓
Step 3: next build（Next.js 主构建）
        ├── 分析每个页面的类型（静态 / 动态）
        ├── 在服务端渲染 Server Components
        ├── 打包 Client Components 的 JS
        ├── 优化图片、字体
        └── 生成 robots.txt、sitemap 等
        ↓
Step 4: 输出构建结果
        ├── Static  (SSG) → 完全静态的 HTML 文件
        ├── Dynamic (SSR) → 每次请求时在服务器渲染
        └── 每个页面的大小（First Load JS）
```

### 4.3 环境变量安全模型

```
┌─────────────────────────────────────────┐
│           服务器端（Node.js）              │
│                                          │
│  可以访问所有环境变量：                     │
│  - DATABASE_URL（数据库密码）              │
│  - STRIPE_API_KEY（支付密钥）             │
│  - NEXTAUTH_SECRET（认证密钥）            │
│  - NEXT_PUBLIC_APP_URL（公开 URL）        │
│                                          │
├─────────────────────────────────────────┤
│           浏览器端（用户电脑）              │
│                                          │
│  只能访问 NEXT_PUBLIC_ 开头的变量：        │
│  - NEXT_PUBLIC_APP_URL ✅               │
│  - DATABASE_URL ❌（看不到）             │
│  - STRIPE_API_KEY ❌（看不到）           │
│                                          │
└─────────────────────────────────────────┘
```

---

## 五、动手练习

### 练习 1：运行构建并分析输出（40 分钟）

**目标：** 亲手执行一次生产构建，理解构建输出的含义。

**步骤：**

1. 确保 `.env` 文件已配置（复制 `.env.example` 并填入值）
2. 运行构建命令：

```bash
pnpm build
```

3. 观察输出，回答以下问题：
   - 一共有多少个路由？
   - 哪些页面标记为 Static（静态），哪些是 Dynamic（动态）？
   - 哪个页面的 "First Load JS" 最大？想想为什么？
   - 构建总共花了多少秒？

**排错提示：**
- 如果报 `Invalid environment variables`，说明 `.env` 文件缺少必填变量，对照 `.env.example` 补全
- 如果报 `Module not found`，运行 `pnpm install` 重新安装依赖
- 如果 Contentlayer 报错，可能是 MDX 文件格式有问题

### 练习 2：体验代码质量工具（30 分钟）

**目标：** 亲手触发 ESLint 检查，感受工具如何帮你发现问题。

**步骤：**

1. 先运行一次 lint 检查，看看项目当前状态：

```bash
pnpm lint
```

2. 故意制造一个 lint 错误。打开任意一个组件文件，把 Tailwind 类名顺序打乱：

```tsx
// 原来的（正确顺序）
<div className="flex items-center justify-center">

// 改成（错误顺序）
<div className="justify-center items-center flex">
```

3. 再次运行 `pnpm lint`，观察报错信息
4. 撤销修改，恢复正确代码

**排错提示：**
- 如果 `pnpm lint` 没有报错，说明项目代码质量很好
- 如果出现大量错误，不要慌——先看第一个错误的描述，通常会告诉你怎么修

### 练习 3：理解环境变量验证（20 分钟）

**目标：** 亲眼看到 `t3-env` 在缺少变量时如何报错。

**步骤：**

1. 备份你的 `.env` 文件：

```bash
cp .env .env.backup
```

2. 编辑 `.env`，注释掉 `DATABASE_URL`（在行首加 `#`）：

```bash
# DATABASE_URL="mysql://..."
```

3. 运行 `pnpm dev` 或 `pnpm build`，观察报错信息
4. 还原文件：

```bash
cp .env.backup .env
```

**排错提示：**
- 报错信息会明确告诉你缺少了哪个变量
- 这就是 `t3-env` 的价值：宁可启动时报错，也不要等用户操作时才崩溃

---

## 六、自检问题

完成今天的学习后，试试能否回答以下问题。

**问题 1：** 为什么不能把密码直接写在代码里，而要用环境变量？

<details>
<summary>查看答案</summary>

因为代码通常会上传到 GitHub 等平台，如果密码写在代码里，任何人都能看到。环境变量存储在服务器的配置中（`.env` 文件或 Vercel Dashboard），不会被上传到代码仓库。即使代码开源，密码也是安全的。
</details>

**问题 2：** `t3-env` 里 `server` 和 `client` 两个配置有什么区别？为什么客户端变量必须以 `NEXT_PUBLIC_` 开头？

<details>
<summary>查看答案</summary>

`server` 里的变量只能在服务器端代码中访问（如 API Routes、Server Components），浏览器看不到。`client` 里的变量在浏览器和服务器都能访问。Next.js 强制要求客户端变量以 `NEXT_PUBLIC_` 开头，这是一个安全措施——防止开发者不小心把 `DATABASE_URL` 这样的敏感变量暴露给浏览器。
</details>

**问题 3：** ESLint 和 Prettier 有什么区别？为什么项目里 `.eslintrc.json` 的 `extends` 数组包含 `"prettier"`？

<details>
<summary>查看答案</summary>

ESLint 检查代码逻辑问题（如未使用的变量、不安全的写法），Prettier 统一代码格式（如缩进、引号、分号）。两者可能在格式上发生冲突（比如 ESLint 要求加分号，Prettier 要求不加），所以 `extends` 里加了 `"prettier"` 来关闭 ESLint 中与 Prettier 冲突的格式规则，让 Prettier 全权负责格式化。
</details>

**问题 4：** `next.config.mjs` 里的 `import "./env.mjs"` 这行代码有什么作用？

<details>
<summary>查看答案</summary>

这行代码在 Next.js 启动或构建时就导入并执行 `env.mjs`。`env.mjs` 里面有环境变量验证逻辑（通过 Zod），如果任何必填的环境变量缺失或格式不对，会立即抛出错误，终止启动/构建。这样可以在最早的时刻发现配置问题，而不是等到用户触发某个功能时才报错。
</details>

**问题 5：** `robots.ts` 文件的作用是什么？如果不写这个文件会怎样？

<details>
<summary>查看答案</summary>

`robots.ts` 会自动生成 `/robots.txt` 文件，告诉搜索引擎爬虫（如 Google Bot）哪些页面可以抓取、哪些不可以。项目中配置的是 `allow: "/"` 即允许抓取所有页面。如果不写这个文件，搜索引擎默认也会抓取所有页面，但有 `robots.txt` 是 SEO 的最佳实践，可以精细控制爬虫行为（比如阻止抓取后台管理页面）。
</details>

---

## 七、预计时间分配

| 环节 | 时间 | 说明 |
|------|------|------|
| 前置知识速补 | 20 分钟 | 环境变量、构建流程、模块系统 |
| 核心概念 + 源码精读 | 80 分钟 | 重点读 `env.mjs` 和 `.eslintrc.json` |
| 练习 1：运行构建 | 40 分钟 | 含排错时间 |
| 练习 2：代码质量工具 | 30 分钟 | ESLint 体验 |
| 练习 3：环境变量验证 | 20 分钟 | 感受 t3-env 报错 |
| 自检问题 + 笔记 | 20 分钟 | 回顾总结 |
| **合计** | **约 3.5 小时** | |

---

## 八、常见踩坑点

### 踩坑 1：构建时报 "Invalid environment variables"

**原因：** `.env` 文件缺少必填的环境变量，或者变量值格式不对。

**解决：** 对照 `.env.example` 检查每个变量是否已填写。特别注意 `NEXTAUTH_URL` 需要是合法的 URL 格式（`http://` 或 `https://` 开头）。

### 踩坑 2：ESLint 和 Prettier 冲突

**现象：** ESLint 说要加分号，Prettier 又把分号删掉，来回反复。

**原因：** 没有正确配置 `eslint-config-prettier`。

**解决：** 确保 `.eslintrc.json` 的 `extends` 数组里包含 `"prettier"`，并且放在最后（后面的规则会覆盖前面的）。

### 踩坑 3：Vercel 部署成功但功能异常

**原因：** 忘记在 Vercel Dashboard 中设置环境变量。`.env` 文件是本地文件，不会被上传到 Vercel。

**解决：** 在 Vercel 项目设置 -> Environment Variables 中逐一添加所有必需的变量。

### 踩坑 4：OG 图片样式不生效

**原因：** `@vercel/og` 的 Satori 引擎只支持有限的 CSS 属性，不能用完整的 Tailwind CSS。

**解决：** 用 `tw` 属性写简单的 Tailwind 类，或者用 `style` 对象写内联样式。不支持的属性包括 `grid`、`animation`、复杂选择器等。

### 踩坑 5：`NEXT_PUBLIC_` 变量修改后不生效

**原因：** `NEXT_PUBLIC_` 开头的变量是在**构建时**嵌入代码的，不是运行时读取的。

**解决：** 修改 `NEXT_PUBLIC_` 变量后，必须重新构建（`pnpm build`），光重启 `pnpm dev` 不够。

---

## 九、延伸阅读

学完今天的内容，如果你想深入了解，可以看看这些资料：

- **Vercel 官方文档**：https://vercel.com/docs — 部署、环境变量、域名配置
- **t3-env 文档**：https://env.t3.gg — 环境变量验证的更多用法
- **ESLint 官方文档**：https://eslint.org/docs/latest/ — 所有规则的详细说明
- **Prettier 官方文档**：https://prettier.io/docs/en/ — 配置选项一览
- **Next.js Metadata API**：https://nextjs.org/docs/app/api-reference/functions/generate-metadata — SEO 相关配置
- **Open Graph 协议**：https://ogp.me — 理解社交媒体预览图的标准
- **Conventional Commits**：https://www.conventionalcommits.org — 提交信息规范的完整说明

---

## 恭喜完成 Day 10！

回顾一下你的学习路径：

```
Day 01 项目全景   →  Day 02 路由基础    →  Day 03 UI 组件
→  Day 03.5 样式系统  →  Day 04 布局导航  →  Day 05 内容系统
→  Day 06 数据库      →  Day 07 认证系统  →  Day 08 API 层
→  Day 08.5 文章编辑器 →  Day 09 支付系统  →  Day 10 部署工程化（今天！）
```

你已经从整体到细节、从前端到后端、从开发到部署，全面了解了一个现代全栈 Next.js 项目。

接下来还有 **Day 11 — 综合实战**，你将把前面所学融会贯通，追踪完整的用户流程并尝试设计新功能。继续加油！
