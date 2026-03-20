# 第1章：项目全景 — 这个项目到底是什么？

## 学习目标

- 理解 Taxonomy 项目的定位（Next.js 13 全栈示范应用）
- 看懂 package.json 中的依赖关系
- 理解目录结构的组织逻辑
- 知道每个配置文件的作用

## 涉及文件

| 文件 | 说明 |
|------|------|
| `package.json` | 项目依赖和脚本命令 |
| `next.config.mjs` | Next.js 框架配置 |
| `tsconfig.json` | TypeScript 语言配置 |
| `tailwind.config.js` | CSS 框架配置 |
| `.env.example` | 环境变量模板 |
| `env.mjs` | 环境变量运行时验证 |

---

## 1. Taxonomy 是什么？

Taxonomy 是一个由 shadcn 开发的**开源全栈示范应用**，用来展示 Next.js 13 的最新特性。你可以把它理解为一个"教科书级别的项目"，它包含了现代 Web 应用的所有核心功能：

- **用户认证**（登录/注册）
- **数据库操作**（增删改查）
- **付费订阅**（Stripe 收款）
- **内容管理**（Markdown 博客/文档）
- **漂亮的 UI 组件**

> 想象一下：如果你要做一个 SaaS 产品（比如在线笔记应用），Taxonomy 就是一个现成的起步模板。

---

## 2. 目录结构

打开项目根目录，你会看到这样的文件夹结构：

```
taxonomy/
├── app/                  # 页面和路由（Next.js App Router）
│   ├── (marketing)/      # 营销页面组（首页、定价、博客）
│   ├── (dashboard)/      # 后台管理组（仪表盘、设置、账单）
│   ├── (auth)/           # 认证页面组（登录、注册）
│   ├── (editor)/         # 编辑器页面组
│   ├── (docs)/           # 文档页面组
│   └── api/              # 后端 API 接口
├── components/           # 可复用的 React 组件
│   └── ui/               # 基础 UI 组件（按钮、卡片等）
├── config/               # 配置文件（导航、订阅计划等）
├── content/              # MDX 内容文件（博客、文档）
├── lib/                  # 工具函数和核心逻辑
├── prisma/               # 数据库模型定义
├── public/               # 静态资源（图片、图标）
├── styles/               # 全局样式
└── types/                # TypeScript 类型定义
```

**关键概念：** 带括号的文件夹如 `(marketing)` 叫做 **Route Group（路由分组）**。括号本身不会出现在 URL 中，它只是用来组织代码的。比如 `app/(marketing)/pricing/page.tsx` 对应的 URL 是 `/pricing`，而不是 `/marketing/pricing`。

---

## 3. package.json — 项目的"购物清单"

`package.json` 就像一个购物清单，列出了项目需要的所有"材料"（依赖包）。让我们分类看看：

### 3.1 脚本命令（scripts）

```json
{
  "scripts": {
    "dev": "concurrently \"contentlayer dev\" \"next dev\"",
    "build": "contentlayer build && next build",
    "start": "next start",
    "lint": "next lint",
    "postinstall": "prisma generate"
  }
}
```

逐行解释：
- **`dev`**：启动开发服务器。`concurrently` 的意思是"同时运行"——它会同时启动 Contentlayer（处理 Markdown 内容）和 Next.js 开发服务器
- **`build`**：构建生产版本。先编译内容，再构建 Next.js
- **`start`**：启动生产服务器
- **`lint`**：检查代码质量
- **`postinstall`**：安装依赖后自动运行，生成 Prisma 数据库客户端代码

### 3.2 核心依赖分类

```
📦 框架层
├── next          → Web 框架（类似后端的 Express，但全栈）
├── react         → UI 库（构建界面的基础）
└── react-dom     → React 在浏览器中运行的桥梁

🎨 UI 层
├── tailwindcss             → CSS 工具类框架（用 class 名写样式）
├── class-variance-authority → 组件变体管理（按钮大小/颜色切换）
├── clsx                    → 条件拼接 CSS 类名
├── tailwind-merge          → 智能合并 Tailwind 类名
├── lucide-react            → 图标库
└── @radix-ui/*             → 无样式的可访问性 UI 组件

📝 内容层
├── contentlayer      → 将 Markdown 文件转成 TypeScript 数据
├── next-contentlayer → Contentlayer 的 Next.js 插件
├── rehype-*          → HTML 处理插件（语法高亮、自动链接等）
└── remark-gfm        → GitHub 风格 Markdown 支持

🗄️ 数据层
├── @prisma/client    → 数据库 ORM 客户端（用 JS 操作数据库）
└── zod               → 数据验证库（确保输入格式正确）

🔐 认证层
├── next-auth              → 认证框架（处理登录/登出）
└── @next-auth/prisma-adapter → NextAuth 的 Prisma 适配器

💰 支付层
└── stripe → Stripe 支付 SDK

📧 邮件层
├── nodemailer → 发送邮件
└── postmark   → Postmark 邮件服务 SDK
```

---

## 4. next.config.mjs — Next.js 的配置文件

```javascript
// next.config.mjs
import { withContentlayer } from "next-contentlayer"

// 导入环境变量验证（确保启动时环境变量齐全）
import "./env.mjs"

/** @type {import('next').NextConfig} */
const nextConfig = {
  // 开启严格模式，帮助发现潜在问题
  reactStrictMode: true,

  // 允许加载 GitHub 头像图片
  images: {
    domains: ["avatars.githubusercontent.com"],
  },

  // 实验性功能
  experimental: {
    appDir: true,  // 启用 App Router（Next.js 13 新路由系统）
    serverComponentsExternalPackages: ["@prisma/client"],  // Prisma 在服务端组件中使用
  },
}

// withContentlayer 是一个"包装函数"，让 Next.js 能识别 Contentlayer 生成的内容
export default withContentlayer(nextConfig)
```

> **大白话理解 `withContentlayer`：** 想象你有一个普通的蛋糕（nextConfig），`withContentlayer` 就是给它加了一层奶油——让蛋糕拥有了处理 Markdown 内容的能力。

---

## 5. tsconfig.json — TypeScript 配置

```json
{
  "compilerOptions": {
    "target": "es5",           // 编译成 ES5 版本的 JS（兼容旧浏览器）
    "lib": ["dom", "dom.iterable", "esnext"],  // 可以使用浏览器 API 和最新 JS 特性
    "strict": false,           // 没有开启严格类型检查（对新手友好）
    "strictNullChecks": true,  // 但开启了空值检查（防止 null/undefined 错误）
    "baseUrl": ".",
    "paths": {
      "@/*": ["./*"],          // 路径别名：@/ 代表项目根目录
      "contentlayer/generated": ["./.contentlayer/generated"]
    }
  }
}
```

**路径别名的好处：**

```typescript
// ❌ 没有路径别名 — 相对路径很混乱
import { db } from "../../../lib/db"

// ✅ 有路径别名 — 清晰直观
import { db } from "@/lib/db"
```

`@/` 就是项目根目录的快捷方式，不管你在哪个文件夹下，`@/lib/db` 都指向根目录下的 `lib/db.ts`。

---

## 6. env.mjs — 环境变量的"保安"

环境变量是一些敏感信息（比如数据库密码、API 密钥），不应该写在代码里。`env.mjs` 使用 `@t3-oss/env-nextjs` 库来确保所有必需的环境变量都已设置：

```javascript
import { createEnv } from "@t3-oss/env-nextjs"
import { z } from "zod"

export const env = createEnv({
  // 服务端环境变量（只在服务器上可用，浏览器看不到）
  server: {
    NEXTAUTH_URL: z.string().url().optional(),  // 认证回调 URL
    NEXTAUTH_SECRET: z.string().min(1),          // 认证密钥（必填）
    GITHUB_CLIENT_ID: z.string().min(1),         // GitHub OAuth 客户端 ID
    GITHUB_CLIENT_SECRET: z.string().min(1),     // GitHub OAuth 客户端密钥
    DATABASE_URL: z.string().min(1),             // 数据库连接字符串
    STRIPE_API_KEY: z.string().min(1),           // Stripe 支付密钥
    STRIPE_WEBHOOK_SECRET: z.string().min(1),    // Stripe Webhook 密钥
    // ... 更多
  },

  // 客户端环境变量（浏览器也能看到，所以不能放敏感信息）
  client: {
    NEXT_PUBLIC_APP_URL: z.string().min(1),      // 应用的公开 URL
  },

  // 运行时映射
  runtimeEnv: {
    NEXTAUTH_URL: process.env.NEXTAUTH_URL,
    // ... 每个变量都需要映射
  },
})
```

**为什么要验证环境变量？**

想象你忘了设置数据库密码就启动了项目。如果没有验证，项目会启动成功，但在用户尝试登录时突然崩溃——这很糟糕。有了 `env.mjs`，项目在启动时就会告诉你："嘿，你忘了设置 `DATABASE_URL`！"

**`server` vs `client` 的区别：**
- `server` 环境变量只在服务器端代码中可用（变量名不以 `NEXT_PUBLIC_` 开头）
- `client` 环境变量在浏览器中也可用（变量名必须以 `NEXT_PUBLIC_` 开头）
- 永远不要把密钥放在 `client` 中！

---

## 7. tailwind.config.js — 样式配置

Tailwind CSS 是一个"工具类优先"的 CSS 框架。传统 CSS 你需要写选择器和样式：

```css
/* 传统 CSS */
.my-button {
  background-color: blue;
  color: white;
  padding: 8px 16px;
  border-radius: 4px;
}
```

Tailwind 让你直接在 HTML 上用类名写样式：

```html
<!-- Tailwind CSS -->
<button class="bg-blue-500 text-white py-2 px-4 rounded">
  Click me
</button>
```

`tailwind.config.js` 定义了项目的设计系统（颜色、字体、间距等）。

---

## 小练习

### 练习 1：找到项目入口
打开 `package.json`，找到 `dev` 脚本。运行 `pnpm dev`（或 `npm run dev`），观察终端输出。你应该能看到 Contentlayer 和 Next.js 同时启动。

### 练习 2：修改站点名称
1. 打开 `config/site.ts`
2. 把 `name: "Taxonomy"` 改成你喜欢的名字
3. 保存后刷新浏览器，看看哪些地方发生了变化

### 练习 3：理解路径别名
在项目中搜索 `@/lib/utils`，数一数有多少个文件导入了这个模块。想想看，如果没有 `@/` 别名，这些导入语句会变成什么样？

### 练习 4：环境变量
1. 打开 `.env.example` 文件
2. 对照 `env.mjs` 中的定义，理解每个变量的用途
3. 思考：为什么 `NEXT_PUBLIC_APP_URL` 在 `client` 里而不是 `server` 里？

---

## 本章小结

你已经了解了 Taxonomy 项目的全貌：
- 它是一个基于 Next.js 13 的全栈示范应用
- `package.json` 管理着所有依赖
- 目录结构按功能模块清晰组织
- 配置文件各司其职：Next.js 配置、TypeScript 配置、样式配置、环境变量验证
- 路径别名 `@/` 让导入更清晰

下一章我们将深入 Next.js 的路由系统，看看页面是怎么出现在浏览器中的。
