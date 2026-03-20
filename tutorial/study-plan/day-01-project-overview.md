# Day 01 — 项目全景：Taxonomy 到底是什么？

> 对应章节：[Chapter 1: 项目总览](../chapter-01-project-overview.md)

---

## 今日学习目标

完成今天的学习后，你应该能够：

1. **说出** Taxonomy 项目的技术栈组成（至少 6 个核心技术），并解释每个技术负责什么
2. **画出** 项目的顶层目录结构，解释每个目录的职责
3. **读懂** `env.mjs`、`config/site.ts`、`lib/utils.ts` 这三个文件的每一行代码

---

## 前置知识速补

这是今天最重要的部分。如果你之前从没写过 JavaScript 或 TypeScript，请认真过一遍。每个知识点我都会先解释概念，再给最小示例，最后对照 Taxonomy 项目里的真实用法。

### 1. [const](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/const) / [let](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/let) 与 `var` 的区别

在 JavaScript 里，声明变量有三种方式。现代项目几乎只用 `const` 和 `let`。

```javascript
// const — 声明后不能重新赋值（"常量"）
const name = "Taxonomy"
// name = "Other"  // ❌ 报错！const 不允许重新赋值

// let — 声明后可以重新赋值
let count = 0
count = 1  // ✅ 没问题

// var — 老写法，作用域规则奇怪，现代项目不推荐用
var old = "don't use me"
```

**Taxonomy 中的实际用法** — `config/site.ts`：

```typescript
export const siteConfig: SiteConfig = {  // 👈 用 const，因为站点配置一旦定义就不会变
  name: "Taxonomy",
  description: "An open source application...",
}
```

**经验法则：** 默认用 `const`，只有当你确实需要重新赋值时才用 `let`，永远不用 `var`。

### 2. [箭头函数](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/Arrow_functions) `() => {}`

箭头函数是 JavaScript 中定义函数的简洁写法。

```javascript
// 传统写法
function add(a, b) {
  return a + b
}

// 箭头函数写法（效果一样）
const add = (a, b) => {
  return a + b
}

// 如果函数体只有一行 return，可以更简洁
const add = (a, b) => a + b  // 👈 省略了 {} 和 return
```

**Taxonomy 中暂时还没出现复杂的箭头函数，但后面的章节会大量使用。** 现在先记住这个语法，遇到 `=>` 就知道它是函数。

### 3. [模板字符串](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Template_literals) `` `hello ${name}` ``

用反引号（键盘左上角，波浪线那个键）包裹的字符串，可以在里面嵌入变量。

```javascript
const name = "Taxonomy"
const version = "0.2.0"

// 传统拼接 — 可读性差
const msg1 = "Welcome to " + name + " v" + version

// 模板字符串 — 清晰直观
const msg2 = `Welcome to ${name} v${version}`  // 👈 ${} 里放变量或表达式
```

**Taxonomy 中的实际用法** — `lib/utils.ts`：

```typescript
export function absoluteUrl(path: string) {
  return `${env.NEXT_PUBLIC_APP_URL}${path}`  // 👈 用模板字符串拼接 URL
}
// 比如 env.NEXT_PUBLIC_APP_URL 是 "https://tx.shadcn.com"
// absoluteUrl("/blog") 的结果就是 "https://tx.shadcn.com/blog"
```

### 4. [import](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/import) / [export](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/export) 模块系统

现代 JavaScript 把代码拆成一个个"模块"（文件）。用 `export` 把东西暴露出去，用 `import` 把别人暴露的东西拿进来。

```javascript
// ---- 文件 math.js ----
export function add(a, b) {    // 👈 命名导出（named export）
  return a + b
}
export const PI = 3.14

// ---- 文件 main.js ----
import { add, PI } from "./math.js"   // 👈 用花括号导入命名导出
console.log(add(1, 2))  // 3

// ---- 另一种方式：默认导出 ----
// 文件 config.js
const config = { name: "app" }
export default config    // 👈 默认导出，一个文件只能有一个

// 文件 main.js
import config from "./config.js"  // 👈 导入默认导出时不用花括号
```

**Taxonomy 中的实际用法** — `lib/utils.ts`：

```typescript
import { ClassValue, clsx } from "clsx"       // 👈 从 clsx 库导入两个东西
import { twMerge } from "tailwind-merge"       // 👈 从 tailwind-merge 库导入一个函数
import { env } from "@/env.mjs"                // 👈 从项目内部导入（@/ 是路径别名）

export function cn(...inputs: ClassValue[]) {  // 👈 命名导出 cn 函数
  return twMerge(clsx(inputs))
}
```

### 5. [async](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/async_function) / [await](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/await) 基础概念

有些操作需要等待（比如读数据库、调 API）。`async`/`await` 让你用"看起来同步"的方式写异步代码。

```javascript
// 假设 fetchUser 是一个需要等待的操作（比如查数据库）
async function getUser(id) {    // 👈 函数前加 async
  const user = await fetchUser(id)  // 👈 await 会"暂停"在这里，等到结果返回
  console.log(user.name)
  return user
}
```

**今天的文件里暂时没有 `async`/`await`，但从明天开始会频繁出现。** 如果你用过 Python 的 `asyncio`，概念是一样的。

### 6. [解构赋值](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Destructuring_assignment)

从对象或数组里"拆"出你需要的值。

```javascript
// 对象解构
const config = { name: "Taxonomy", version: "0.2.0", author: "shadcn" }
const { name, version } = config  // 👈 一行代码拿到 name 和 version
console.log(name)     // "Taxonomy"
console.log(version)  // "0.2.0"

// 数组解构
const colors = ["red", "green", "blue"]
const [first, second] = colors  // 👈 按位置取值
console.log(first)   // "red"
console.log(second)  // "green"
```

**Taxonomy 中的实际用法** — `config/site.ts`（导入时就是解构）：

```typescript
import { SiteConfig } from "types"  // 👈 从 "types" 模块中解构出 SiteConfig 这个类型
```

### 7. [展开运算符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Spread_syntax) `...`

三个点 `...` 可以把数组或对象"展开"。

```javascript
// 展开数组
const arr1 = [1, 2, 3]
const arr2 = [...arr1, 4, 5]  // [1, 2, 3, 4, 5]

// 展开对象
const base = { color: "red", size: "large" }
const extended = { ...base, size: "small" }  // 👈 size 被覆盖
// 结果：{ color: "red", size: "small" }
```

**Taxonomy 中的实际用法** — `lib/utils.ts`：

```typescript
export function cn(...inputs: ClassValue[]) {  // 👈 ...inputs 表示"收集所有参数到一个数组"
  return twMerge(clsx(inputs))
}
// 调用示例：cn("px-4", "py-2", isActive && "bg-blue-500")
// inputs 会是 ["px-4", "py-2", "bg-blue-500"] 或 ["px-4", "py-2", false]
```

这里 `...` 出现在函数参数中，叫做"[剩余参数](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Functions/rest_parameters)"（rest parameter），意思是"把传进来的所有参数收集成一个数组"。

### 8. [TypeScript 基础类型注解](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html)

TypeScript 就是 JavaScript 加上"类型标注"。类型标注告诉编辑器和编译器：这个变量应该是什么类型。

```typescript
// 基础类型
const name: string = "Taxonomy"     // 字符串
const count: number = 42            // 数字
const isReady: boolean = true       // 布尔值

// 函数参数和返回值的类型
function greet(name: string): string {  // 👈 参数是 string，返回值也是 string
  return `Hello, ${name}`
}

// interface — 定义对象的"形状"
interface User {
  id: number
  name: string
  email: string
}
const user: User = { id: 1, name: "Alice", email: "alice@example.com" }
```

**Taxonomy 中的实际用法** — `lib/utils.ts`：

```typescript
export function formatDate(input: string | number): string {
//                         ^^^^^ ^^^^^^^^^^^^^^   ^^^^^^
//                         参数名  参数类型          返回值类型
//                                string | number 表示"可以是字符串，也可以是数字"
  const date = new Date(input)
  return date.toLocaleDateString("en-US", {
    month: "long",
    day: "numeric",
    year: "numeric",
  })
}
```

**Taxonomy 中的实际用法** — `config/site.ts`：

```typescript
import { SiteConfig } from "types"         // 👈 导入 SiteConfig 这个 interface

export const siteConfig: SiteConfig = {    // 👈 告诉 TS：siteConfig 必须符合 SiteConfig 的形状
  name: "Taxonomy",
  description: "An open source application...",
  url: "https://tx.shadcn.com",
  ogImage: "https://tx.shadcn.com/og.jpg",
  links: {
    twitter: "https://twitter.com/shadcn",
    github: "https://github.com/shadcn/taxonomy",
  },
}
```

如果你少写了一个必需的字段（比如漏掉了 `url`），TypeScript 会立刻报错，不用等到运行时才发现。

---

## 核心概念清单

| 概念 | 一句话说明 | 类比 |
|------|-----------|------|
| [Next.js 13 App Router](https://nextjs.org/docs/app/building-your-application/routing) | 基于文件系统的全栈 Web 框架 | 类似 Python 的 Django，但前后端一体 |
| [Route Groups](https://nextjs.org/docs/app/building-your-application/routing/route-groups) `(marketing)` | 用括号命名的目录，组织代码但不影响 URL | 文件柜的抽屉标签，方便归类但客户看不到 |
| [Tailwind CSS](https://tailwindcss.com/docs/utility-first) | 用 class 名写样式的 CSS 框架 | 乐高积木——用小块拼出界面 |
| [Prisma](https://www.prisma.io/docs/concepts/overview/what-is-prisma) | 用 TypeScript 操作数据库的 ORM | 类似 Python 的 SQLAlchemy |
| [NextAuth.js](https://next-auth.js.org/getting-started/introduction) | Next.js 专用的登录/认证方案 | 门卫系统，管理谁能进、谁不能进 |
| [Contentlayer](https://contentlayer.dev/docs/getting-started) | 把 Markdown 文件变成类型安全的数据 | 自动把 Word 文档转成结构化的 JSON |
| t3-env (`env.mjs`) | 启动时验证环境变量是否齐全 | 出门前检查钥匙、手机、钱包的清单 |
| 路径别名 `@/` | 用 `@/lib/utils` 代替 `../../../lib/utils` | 快捷方式/书签 |
| [zod](https://zod.dev/?id=introduction) | 数据校验库，确保数据格式正确 | 快递收件时核对包裹尺寸和重量 |

---

## 源码精读

下面我们逐文件、逐函数地读代码。每段代码我都会加中文注释，重点行用 `// 👈` 标记。

### 文件 1：`config/site.ts` — 站点配置

这是项目里最简单的文件之一，适合用来热身。

```typescript
import { SiteConfig } from "types"  // 👈 导入类型定义，SiteConfig 描述了配置对象应该长什么样

export const siteConfig: SiteConfig = {  // 👈 导出一个常量，类型是 SiteConfig
  name: "Taxonomy",                      // 站点名称，会出现在浏览器标签页和页面标题
  description:
    "An open source application built using the new router, server components and everything new in Next.js 13.",
  url: "https://tx.shadcn.com",          // 站点的正式 URL
  ogImage: "https://tx.shadcn.com/og.jpg", // 社交媒体分享时显示的预览图
  links: {                               // 👈 嵌套对象——作者的社交链接
    twitter: "https://twitter.com/shadcn",
    github: "https://github.com/shadcn/taxonomy",
  },
}
```

**为什么这样写？**

把站点信息集中在一个配置文件里，好处是"改一处，全站生效"。比如你想把站点名从 "Taxonomy" 改成 "My App"，只需要改这一行，所有引用了 `siteConfig.name` 的地方都会自动更新。这就是所谓的"单一数据源"（Single Source of Truth）。

### 文件 2：`lib/utils.ts` — 工具函数集

这个文件只有 21 行，却包含了三个非常典型的工具函数。

```typescript
import { ClassValue, clsx } from "clsx"      // 👈 clsx：根据条件拼接 CSS class 名
import { twMerge } from "tailwind-merge"      // 👈 twMerge：智能合并 Tailwind class（避免冲突）

import { env } from "@/env.mjs"               // 👈 注意 @/ 路径别名，指向项目根目录的 env.mjs

// --- 函数 1：cn（className 的缩写）---
export function cn(...inputs: ClassValue[]) { // 👈 ...inputs 收集所有参数成数组
  return twMerge(clsx(inputs))               // 👈 先用 clsx 拼接，再用 twMerge 去重
}
// 使用场景：cn("px-4 py-2", isActive && "bg-blue-500", "text-white")
// 如果 isActive 为 true  → "px-4 py-2 bg-blue-500 text-white"
// 如果 isActive 为 false → "px-4 py-2 text-white"（false 会被 clsx 自动忽略）

// --- 函数 2：formatDate ---
export function formatDate(input: string | number): string {  // 👈 接受字符串或数字
  const date = new Date(input)                                // 👈 转成 Date 对象
  return date.toLocaleDateString("en-US", {                   // 👈 格式化成美式英语日期
    month: "long",    // "January", "February" 这样的完整月份
    day: "numeric",   // "1", "2", "15"
    year: "numeric",  // "2024"
  })
}
// formatDate("2024-01-15") → "January 15, 2024"

// --- 函数 3：absoluteUrl ---
export function absoluteUrl(path: string) {
  return `${env.NEXT_PUBLIC_APP_URL}${path}`  // 👈 把相对路径变成完整 URL
}
// absoluteUrl("/blog") → "https://tx.shadcn.com/blog"
```

**为什么要把 `cn` 函数单独封装？**

在 Tailwind CSS 项目里，你经常需要根据条件动态切换 class。直接用字符串拼接很容易出问题（比如 `"px-4 px-8"` 这样的冲突）。`cn` 函数把 `clsx`（条件拼接）和 `twMerge`（冲突去重）组合在一起，是 Tailwind 项目的标准实践。整个项目里几乎所有组件都会用到它。

### 文件 3：`env.mjs` — 环境变量验证

```javascript
import { createEnv } from "@t3-oss/env-nextjs"  // 👈 从 t3-env 库导入创建函数
import { z } from "zod"                          // 👈 zod 是数据验证库

export const env = createEnv({
  // === 服务端环境变量 === 只在服务器上可用，浏览器看不到
  server: {
    NEXTAUTH_URL: z.string().url().optional(),      // 👈 .optional() 表示可以不填
    NEXTAUTH_SECRET: z.string().min(1),             // 👈 .min(1) 表示至少 1 个字符（不能为空）
    GITHUB_CLIENT_ID: z.string().min(1),            // GitHub 登录需要的 OAuth 凭证
    GITHUB_CLIENT_SECRET: z.string().min(1),        // 👈 注意：SECRET 类的变量都在 server 里
    GITHUB_ACCESS_TOKEN: z.string().min(1),
    DATABASE_URL: z.string().min(1),                // 数据库连接字符串
    SMTP_FROM: z.string().min(1),                   // 发件人邮箱地址
    POSTMARK_API_TOKEN: z.string().min(1),          // Postmark 邮件服务的密钥
    POSTMARK_SIGN_IN_TEMPLATE: z.string().min(1),
    POSTMARK_ACTIVATION_TEMPLATE: z.string().min(1),
    STRIPE_API_KEY: z.string().min(1),              // Stripe 支付密钥
    STRIPE_WEBHOOK_SECRET: z.string().min(1),
    STRIPE_PRO_MONTHLY_PLAN_ID: z.string().min(1),
  },

  // === 客户端环境变量 === 浏览器也能看到，所以绝对不能放密钥！
  client: {
    NEXT_PUBLIC_APP_URL: z.string().min(1),  // 👈 注意命名必须以 NEXT_PUBLIC_ 开头
  },

  // === 运行时映射 === 把 process.env 里的值"传"给上面的 schema
  runtimeEnv: {                              // 👈 每个变量都要在这里写一遍
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

**为什么要这么麻烦地验证环境变量？**

想象你忘了设置 `DATABASE_URL` 就启动了项目。如果没有验证，项目能正常启动，但用户尝试登录时突然崩溃——这种 bug 很难排查。有了 `env.mjs`，项目启动时就会报错："缺少 DATABASE_URL！"，帮你第一时间发现问题。

**`server` vs `client` 的核心区别：**

- `server` 里的变量只存在于 Node.js 服务端，浏览器端的代码拿不到——适合放密码、密钥
- `client` 里的变量会被打包到前端 JavaScript 中，任何人打开浏览器开发者工具都能看到——只能放公开信息
- Next.js 的约定：客户端变量名必须以 `NEXT_PUBLIC_` 开头

### 文件 4：`tsconfig.json` — TypeScript 配置

```json
{
  "compilerOptions": {
    "target": "es5",              // 👈 编译产物兼容到 ES5（支持旧浏览器）
    "lib": ["dom", "dom.iterable", "esnext"],  // 可以使用浏览器 DOM API + 最新 JS 特性
    "allowJs": true,              // 允许项目中混用 .js 和 .ts 文件
    "skipLibCheck": true,         // 跳过第三方库的类型检查（加速编译）
    "strict": false,              // 👈 没有开启最严格的类型检查（对新手友好）
    "forceConsistentCasingInFileNames": true,  // 文件名大小写必须一致
    "noEmit": true,               // 不输出编译结果（Next.js 自己负责编译）
    "esModuleInterop": true,      // 让 CommonJS 和 ES Module 可以互相导入
    "module": "esnext",           // 使用最新的模块系统
    "moduleResolution": "node",   // 按 Node.js 的方式查找模块
    "resolveJsonModule": true,    // 允许 import JSON 文件
    "isolatedModules": true,      // 每个文件独立编译
    "jsx": "preserve",            // 保留 JSX 语法（交给 Next.js 处理）
    "incremental": true,          // 增量编译（只编译改动的文件，加速开发）
    "baseUrl": ".",               // 👈 路径解析的基准目录是项目根目录
    "paths": {
      "@/*": ["./*"],             // 👈 重点！@/ 映射到项目根目录
      "contentlayer/generated": ["./.contentlayer/generated"]
    },
    "plugins": [{ "name": "next" }],
    "strictNullChecks": true      // 👈 虽然 strict 是 false，但单独开了空值检查
  },
  "include": ["next-env.d.ts", "**/*.ts", "**/*.tsx", ".next/types/**/*.ts", ".contentlayer/generated"],
  "exclude": ["node_modules"]
}
```

**路径别名 `@/` 到底解决了什么问题？**

```typescript
// ❌ 没有路径别名时——你需要数有几层 ../
import { db } from "../../../lib/db"
import { cn } from "../../lib/utils"

// ✅ 有路径别名后——永远从根目录出发，清晰明了
import { db } from "@/lib/db"
import { cn } from "@/lib/utils"
```

不管你的文件在 `app/(marketing)/pricing/page.tsx` 还是在 `components/ui/button.tsx`，`@/lib/utils` 永远指向同一个地方。

### 文件 5：`package.json` — 项目依赖全景（节选）

```json
{
  "name": "taxonomy",
  "version": "0.2.0",
  "private": true,       // 👈 私有项目，不会被发布到 npm
  "scripts": {
    "dev": "concurrently \"contentlayer dev\" \"next dev\"",  // 👈 同时启动两个进程
    "build": "contentlayer build && next build",              // 👈 && 表示"前一个成功后再执行下一个"
    "start": "next start",
    "lint": "next lint",                                      // 代码质量检查
    "postinstall": "prisma generate"                          // 👈 装完依赖后自动生成 Prisma 客户端
  }
}
```

**`scripts` 中每条命令的作用：**

| 命令 | 触发方式 | 作用 |
|------|---------|------|
| `dev` | `pnpm dev` | 启动开发服务器，支持热更新 |
| `build` | `pnpm build` | 构建生产版本，输出优化过的静态文件 |
| `start` | `pnpm start` | 启动生产服务器（先要 `build`） |
| `lint` | `pnpm lint` | 检查代码规范 |
| `postinstall` | 自动触发 | `pnpm install` 执行完后自动运行 |

**依赖分类一览：**

```
框架层（核心骨架）
├── next@13.3.2         → 全栈 Web 框架
├── react@18.2          → UI 渲染库
└── react-dom@18.2      → React 在浏览器中的运行桥梁

样式层（界面外观）
├── tailwindcss         → 原子化 CSS 框架
├── class-variance-authority → 管理组件变体（如按钮的大/中/小）
├── clsx + tailwind-merge    → CSS class 条件拼接与去重
├── lucide-react        → 图标库
└── @radix-ui/*         → 无障碍 UI 组件基础库（20+ 个组件）

内容层（博客/文档）
├── contentlayer        → 把 Markdown 变成 TypeScript 数据
└── rehype-* / remark-* → Markdown 处理插件

数据层
├── @prisma/client      → 数据库 ORM（用 TS 写 SQL）
└── zod                 → 运行时数据校验

认证层
├── next-auth           → 登录/注册/会话管理
└── @next-auth/prisma-adapter → 把用户数据存到 Prisma 管理的数据库

支付层
└── stripe              → Stripe 支付集成

邮件层
├── nodemailer          → 通用邮件发送
└── postmark            → Postmark 邮件服务
```

---

## 动手练习

### 练习 1：启动项目并观察输出（难度：入门）

**目标：** 成功启动开发服务器，理解 `dev` 脚本做了什么。

**步骤：**

1. 复制环境变量模板：
   ```bash
   cp .env.example .env
   ```
2. 安装依赖：
   ```bash
   pnpm install
   ```
3. 启动开发服务器：
   ```bash
   pnpm dev
   ```
4. 观察终端输出——你应该能看到两个进程同时启动：
   - `[0]` Contentlayer：处理 Markdown 内容
   - `[1]` Next.js：Web 开发服务器
5. 打开浏览器访问 `http://localhost:3000`

**预期结果：** 浏览器显示 Taxonomy 首页。如果因为环境变量缺失报错，那也是正常的——说明 `env.mjs` 的验证在起作用。

**排错提示：**
- 如果报 `command not found: pnpm`，需要先安装 pnpm：`npm install -g pnpm`
- 如果报环境变量错误，检查 `.env` 文件是否存在，暂时可以先填假值

### 练习 2：修改站点配置并观察变化（难度：入门）

**目标：** 理解"单一数据源"模式——改一个配置文件就能影响全站。

**步骤：**

1. 打开 `config/site.ts`
2. 把 `name: "Taxonomy"` 改成 `name: "My Awesome App"`
3. 保存文件
4. 回到浏览器，页面会自动刷新（这就是 Hot Reload）
5. 观察浏览器标签页标题、页面上的站点名称等是否发生了变化
6. 改完后记得改回来：`name: "Taxonomy"`

**预期结果：** 所有引用了 `siteConfig.name` 的地方都显示新名称。

**参考代码：**
```typescript
// config/site.ts — 修改这一行
export const siteConfig: SiteConfig = {
  name: "My Awesome App",  // 👈 改这里
  // ... 其他不动
}
```

### 练习 3：给 `lib/utils.ts` 添加一个工具函数（难度：进阶）

**目标：** 练习 TypeScript 函数定义、模板字符串、类型注解。

**步骤：**

1. 打开 `lib/utils.ts`
2. 在文件末尾添加一个 `truncate` 函数——它的作用是截断过长的字符串
3. 保存文件，确保没有 TypeScript 报错

**参考代码：**
```typescript
// 在 lib/utils.ts 末尾添加
export function truncate(str: string, maxLength: number): string {
  if (str.length <= maxLength) {
    return str
  }
  return `${str.slice(0, maxLength)}...`
}
// truncate("Hello, World!", 5)  → "Hello..."
// truncate("Hi", 5)            → "Hi"
```

**预期结果：** 文件保存后无报错。你可以在浏览器控制台或 Node.js REPL 里测试这个函数。

**排错提示：**
- 如果报类型错误，检查参数的类型注解是否正确
- 确保函数前有 `export` 关键字，否则其他文件无法导入

---

## 概念图解

### Taxonomy 项目技术栈全景

```
┌─────────────────────────────────────────────────────────┐
│                    用户浏览器                              │
│  ┌─────────────────────────────────────────────────────┐ │
│  │              React 组件 + Tailwind CSS               │ │
│  │         （页面渲染、交互、样式）                        │ │
│  └───────────────────────┬─────────────────────────────┘ │
└──────────────────────────┼──────────────────────────────┘
                           │ HTTP 请求
┌──────────────────────────┼──────────────────────────────┐
│                    Next.js 服务端                         │
│                          │                               │
│  ┌───────────┐  ┌───────┴───────┐  ┌─────────────────┐ │
│  │ NextAuth  │  │  API Routes   │  │  Contentlayer   │ │
│  │ (认证)    │  │  (后端接口)    │  │  (Markdown→数据) │ │
│  └─────┬─────┘  └───────┬───────┘  └────────┬────────┘ │
│        │                │                    │          │
│  ┌─────┴────────────────┴──────┐     ┌──────┴───────┐  │
│  │         Prisma ORM          │     │  MDX 文件     │  │
│  │      (数据库读写)            │     │  (博客/文档)  │  │
│  └─────────────┬───────────────┘     └──────────────┘  │
└────────────────┼────────────────────────────────────────┘
                 │
          ┌──────┴──────┐
          │  数据库       │
          │ (PostgreSQL) │
          └──────┬──────┘
                 │
          ┌──────┴──────┐
          │   Stripe     │
          │  (支付处理)   │
          └─────────────┘
```

### 配置文件之间的关系

```
项目启动时的加载顺序：

  package.json          tsconfig.json          tailwind.config.js
  (依赖 + 脚本)         (TS 编译选项)           (样式配置)
       │                     │                       │
       ▼                     ▼                       ▼
  next.config.mjs ──导入──▶ env.mjs ◀──被导入── lib/utils.ts
  (Next.js 配置)          (环境变量验证)        (工具函数)
       │                     │
       ▼                     ▼
  启动 Next.js 开发服务器     验证所有环境变量
  如果变量缺失 → 立即报错，不启动
```

---

## 自检问题

**Q1（基础）：** `app/` 目录下的 `(marketing)` 和 `(dashboard)` 为什么要用括号？访问 `app/(marketing)/pricing/page.tsx` 的 URL 是什么？

> 提示：括号目录叫 Route Group，它的作用是组织代码结构。URL 是 `/pricing`，括号部分不会出现在路径中。

**Q2（基础）：** `const` 和 `let` 有什么区别？在 Taxonomy 项目里为什么大多数变量都用 `const`？

> 提示：`const` 声明后不能重新赋值。项目中的配置、函数定义、导入的模块都不需要重新赋值，所以用 `const` 更安全。

**Q3（理解）：** `env.mjs` 中 `server` 和 `client` 两组环境变量有什么区别？为什么 `STRIPE_API_KEY` 放在 `server` 而不是 `client` 里？

> 提示：`server` 变量只存在于服务端，浏览器拿不到。`STRIPE_API_KEY` 是支付密钥，如果放在 `client` 里，任何人打开浏览器开发者工具都能看到。

**Q4（应用）：** 如果你要新增一个服务端环境变量 `ANALYTICS_SECRET_KEY`，需要修改 `env.mjs` 中的哪些地方？

> 提示：需要改三处——`server` 对象里加 schema 定义，`runtimeEnv` 对象里加映射，以及在 `.env` 文件里写上实际的值。

**Q5（分析）：** `lib/utils.ts` 中的 `cn` 函数为什么要先调用 `clsx` 再调用 `twMerge`，而不是直接用其中一个？

> 提示：`clsx` 负责处理条件逻辑（比如 `isActive && "bg-blue-500"` 中 `false` 值的过滤），`twMerge` 负责处理 Tailwind class 冲突（比如同时有 `px-4` 和 `px-8` 时保留后者）。两者各司其职，组合起来才完整。

---

## 预计时间分配

| 环节 | 时间 | 说明 |
|------|------|------|
| 前置知识速补 | 50 分钟 | 如果你已有 JS 基础可以快速跳过 |
| 核心概念清单 + 概念图解 | 15 分钟 | 对照图解理解技术栈全貌 |
| 源码精读（5 个文件） | 50 分钟 | 逐行读注释，不懂就回查前置知识 |
| 练习 1：启动项目 | 25 分钟 | 包含安装依赖的时间 |
| 练习 2：修改配置 | 15 分钟 | 体验 Hot Reload |
| 练习 3：写工具函数 | 20 分钟 | 第一次写 TypeScript |
| 自检问题 + 笔记 | 15 分钟 | 回顾今天学到的内容 |
| **合计** | **约 3 - 3.5 小时** | |

---

## 常见踩坑点

### 1. 环境变量缺失导致启动失败

**错误信息：**
```
❌ Invalid environment variables: { NEXTAUTH_SECRET: [ 'Required' ], DATABASE_URL: [ 'Required' ] }
```

**原因：** `env.mjs` 中的 `zod` 验证发现必填的环境变量没有设置。

**解决：**
1. 确保项目根目录有 `.env` 文件：`cp .env.example .env`
2. 打开 `.env` 文件，逐项填入值（开发环境可以先填假值）
3. 重新启动：`pnpm dev`

### 2. Node.js 版本过低

**错误信息：**
```
SyntaxError: Unexpected token 'export'
```
或者各种奇怪的语法错误。

**原因：** 项目需要 Node.js 18+，老版本不支持新语法。

**解决：**
```bash
node -v  # 查看当前版本
# 如果低于 18，用 nvm 切换：
nvm install 18
nvm use 18
```

### 3. 路径别名 `@/` 在编辑器中报红

**错误信息：** IDE 显示 `Cannot find module '@/lib/utils'`，但项目能正常运行。

**原因：** 编辑器没有正确加载 `tsconfig.json`。

**解决：**
- VS Code：按 `Cmd+Shift+P`（Mac）或 `Ctrl+Shift+P`（Windows），输入 `TypeScript: Restart TS Server`
- 确保 `tsconfig.json` 中的 `paths` 配置正确（参考源码精读部分）

### 4. `pnpm` 未安装

**错误信息：**
```
command not found: pnpm
```

**解决：**
```bash
npm install -g pnpm
```

> 为什么用 `pnpm` 而不是 `npm`？`pnpm` 的依赖管理更严格、安装速度更快、磁盘占用更少。Taxonomy 项目推荐使用 `pnpm`，用 `npm` 可能会遇到依赖解析差异。

---

## 延伸阅读

1. **JavaScript 基础速查** — 搜索关键词：`MDN JavaScript Guide`。MDN 是 Mozilla 维护的官方文档，是学习 JS 最权威的参考。重点看 "Grammar and types"、"Functions"、"Expressions and operators" 三个章节。

2. **TypeScript 入门** — 搜索关键词：`TypeScript Handbook Basics`。官方手册的前三章（Basics、Everyday Types、Narrowing）覆盖了 Taxonomy 项目中 90% 的 TS 用法。

3. **Next.js 13 App Router** — 搜索关键词：`Next.js App Router Documentation`。官方文档的 "Getting Started" 部分，帮你理解为什么目录结构要这样组织。
