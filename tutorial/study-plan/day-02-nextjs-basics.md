# Day 02 — Next.js 基础：页面是怎么出现在浏览器的？

> 对应章节：[Chapter 2: Next.js 基础](../chapter-02-nextjs-basics.md)

---

## 1. 今日学习目标

完成今天的学习后，你应该能够：

1. **看到一个文件路径，立刻说出它对应的 URL**（例如 `app/(dashboard)/dashboard/settings/page.tsx` → `/dashboard/settings`）
2. **区分 Server Component 和 Client Component**，并解释各自能做什么、不能做什么
3. **独立创建一个新页面**，让它正确显示在浏览器中，并继承对应的布局

---

## 2. 前置知识速补

如果你之前没接触过 JavaScript/TypeScript/React，这一节帮你快速补齐今天需要用到的基础知识。

### 2.1 [export default](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/export#默认导出) vs `export`（默认导出 vs 命名导出）

```ts
// ---- 默认导出（一个文件只能有一个）----
export default function HomePage() {
  return <div>Hello</div>
}
// 导入时可以随便起名字：
// import MyPage from "./page"     ← 名字随意
// import Anything from "./page"   ← 这样也行

// ---- 命名导出（一个文件可以有多个）----
export function Button() { return <button>Click</button> }
export function Input() { return <input /> }
// 导入时必须用大括号，名字必须一致：
// import { Button, Input } from "./components"
```

**记忆口诀：** `default` = "这个文件的主角"，`export` = "这个文件的配角"。

### 2.2 [async/await](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Asynchronous/Promises#async_和_await) 的含义

有些操作需要等待（比如从网络获取数据），`async/await` 就是"等一等再继续"的意思：

```ts
// 普通函数：立刻返回结果
function add(a, b) { return a + b }

// 异步函数：需要等待才能拿到结果
async function fetchData() {
  const response = await fetch("https://api.example.com/data")
  //                     ^^^^^ 等网络请求完成
  const data = await response.json()
  //                 ^^^^^ 等 JSON 解析完成
  return data
}
```

**类比：** `await` 就像在餐厅点餐后说"我等着，做好了再给我"。

### 2.3 TypeScript 的 [interface](https://www.typescriptlang.org/docs/handbook/2/objects.html)

`interface` 用来描述"一个东西长什么样"：

```ts
// 定义：一个 User 必须有 name 和 age
interface User {
  name: string   // name 是字符串
  age: number    // age 是数字
  email?: string // email 可有可无（? 表示可选）
}

// 使用：TypeScript 会检查你传入的数据是否符合要求
function greet(user: User) {
  console.log(`Hello, ${user.name}`)
}
```

**类比：** `interface` 就像一份表格模板，规定了必须填哪些字段、每个字段填什么类型。

### 2.4 [React 函数组件](https://react.dev/learn/your-first-component)最小形式

React 组件就是一个返回 HTML（准确说是 JSX）的函数：

```tsx
// 最简单的组件
function Hello() {
  return <div>Hello World</div>
}
```

组件名**必须大写开头**（`Hello` 而不是 `hello`），否则 React 会把它当成普通 HTML 标签。

### 2.5 [JSX](https://react.dev/learn/writing-markup-with-jsx) 基础语法

JSX 看起来像 HTML，但其实是 JavaScript。几个关键区别：

```tsx
function Example() {
  const name = "小明"
  const isLoggedIn = true

  return (
    <div className="container">          {/* 用 className 代替 class */}
      <p>你好，{name}</p>                 {/* 大括号 {} 里放 JS 表达式 */}
      {isLoggedIn && <span>已登录</span>} {/* 条件渲染 */}
      {isLoggedIn ? (                     {/* 三元表达式 */}
        <button>退出</button>
      ) : (
        <button>登录</button>
      )}
    </div>
  )
}
```

---

## 3. 核心概念清单

| 概念 | 一句话说明 | 类比 |
|------|-----------|------|
| [App Router](https://nextjs.org/docs/app/building-your-application/routing) | Next.js 基于 `app/` 目录的路由系统，文件路径就是 URL | 图书馆的书架编号对应书的位置 |
| [Server Component](https://nextjs.org/docs/app/building-your-application/rendering/server-components) | 默认组件类型，只在服务器上运行，可直接查数据库/调 API | 厨房里做好端出来的菜 |
| [Client Component](https://nextjs.org/docs/app/building-your-application/rendering/client-components) | 文件顶部加 `"use client"`，在浏览器中运行，可响应用户交互 | 在用户桌上现场烹饪的菜 |
| [layout.tsx](https://nextjs.org/docs/app/building-your-application/routing/layouts-and-templates) | 共享布局（导航栏、侧边栏等），切换子页面时布局不会重新渲染 | 相框（照片换来换去，相框不变） |
| [page.tsx](https://nextjs.org/docs/app/building-your-application/routing/pages) | 定义一个路由的页面内容，文件位置决定 URL | 相框里的照片 |
| [loading.tsx](https://nextjs.org/docs/app/building-your-application/routing/loading-ui-and-streaming) | 页面加载时的占位 UI，基于 [React Suspense](https://react.dev/reference/react/Suspense) | 餐厅上菜前的"请稍候"牌子 |
| [error.tsx](https://nextjs.org/docs/app/building-your-application/routing/error-handling) | 错误边界，捕获子组件的运行时错误（必须是 Client Component） | 安全气囊，出事时弹出保护页面 |
| [Route Groups](https://nextjs.org/docs/app/building-your-application/routing/route-groups) | 用圆括号命名的文件夹 `(marketing)`，不影响 URL，只用于组织代码和共享布局 | 文件柜的分类标签，不印在文件上 |
| [Metadata](https://nextjs.org/docs/app/building-your-application/optimizing/metadata) | 通过 `export const metadata` 设置页面标题、描述等 SEO 信息 | 书的封面信息（书名、作者、简介） |

---

## 4. 源码精读

### 4.1 根布局 `app/layout.tsx` — 所有页面的外壳

这是整个应用最顶层的布局，每个页面都会被它包裹。

```tsx
import { Inter as FontSans } from "next/font/google"   // 👈 从 Google Fonts 加载 Inter 字体
import localFont from "next/font/local"                 // 👈 加载本地字体文件

import "@/styles/globals.css"                           // 👈 全局 CSS 样式
import { siteConfig } from "@/config/site"              // 👈 站点配置（名称、描述等）
import { absoluteUrl, cn } from "@/lib/utils"           // 👈 工具函数，cn 用于拼接 CSS 类名
import { Toaster } from "@/components/ui/toaster"       // 👈 Toast 通知组件
import { Analytics } from "@/components/analytics"      // 👈 数据统计
import { TailwindIndicator } from "@/components/tailwind-indicator"  // 👈 开发时显示屏幕断点
import { ThemeProvider } from "@/components/theme-provider"          // 👈 暗色/亮色主题切换

const fontSans = FontSans({
  subsets: ["latin"],
  variable: "--font-sans",   // 👈 定义 CSS 变量，方便在样式中引用
})

const fontHeading = localFont({
  src: "../assets/fonts/CalSans-SemiBold.woff2",  // 👈 标题专用字体，从本地文件加载
  variable: "--font-heading",
})

interface RootLayoutProps {       // 👈 TypeScript interface，描述这个组件接收什么参数
  children: React.ReactNode       // 👈 children 就是"被这个布局包裹的子页面内容"
}

export const metadata = {         // 👈 注意这里：命名导出 metadata，Next.js 会自动读取
  title: {
    default: siteConfig.name,                     // 默认标题："Taxonomy"
    template: `%s | ${siteConfig.name}`,          // 子页面标题格式："Dashboard | Taxonomy"
  },
  description: siteConfig.description,
  // ... 省略 openGraph, twitter, icons 等字段
}

export default function RootLayout({ children }: RootLayoutProps) {  // 👈 默认导出，Next.js 识别为布局
  return (
    <html lang="en" suppressHydrationWarning>
      <head />
      <body
        className={cn(                              // 👈 cn() 把多个类名合并成一个字符串
          "min-h-screen bg-background font-sans antialiased",
          fontSans.variable,                        // 👈 注入字体 CSS 变量
          fontHeading.variable
        )}
      >
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
          {children}              {/* 👈 注意这里：子页面内容会渲染在这个位置 */}
          <Analytics />           {/* 每个页面都有数据统计 */}
          <Toaster />             {/* 每个页面都能弹 Toast 通知 */}
          <TailwindIndicator />   {/* 仅开发时可见，显示 sm/md/lg 等断点 */}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

**逐行要点：**

- `import "@/styles/globals.css"` — `@/` 是项目根目录的别名，在 `tsconfig.json` 中配置。这行导入了全局样式。
- `export const metadata` — 这不是随便起的变量名。Next.js **规定**布局和页面可以导出一个叫 `metadata` 的对象来设置 SEO 信息。
- `export default function RootLayout` — 同样，Next.js **规定** `layout.tsx` 必须默认导出一个组件。
- `{children}` — 这是 React 的核心概念。布局组件通过 `children` 接收子页面的内容，就像一个"插槽"。

### 4.2 营销布局 `app/(marketing)/layout.tsx` — 首页等页面的专属布局

```tsx
import Link from "next/link"                          // 👈 Next.js 的链接组件
import { marketingConfig } from "@/config/marketing"
import { cn } from "@/lib/utils"
import { buttonVariants } from "@/components/ui/button"
import { MainNav } from "@/components/main-nav"       // 👈 顶部导航栏组件
import { SiteFooter } from "@/components/site-footer" // 👈 页脚组件

interface MarketingLayoutProps {
  children: React.ReactNode
}

export default async function MarketingLayout({       // 👈 注意这里：async 函数！
  children,
}: MarketingLayoutProps) {
  return (
    <div className="flex min-h-screen flex-col">
      <header className="container z-40 bg-background">
        <div className="flex h-20 items-center justify-between py-6">
          <MainNav items={marketingConfig.mainNav} /> {/* 👈 导航栏 */}
          <nav>
            <Link
              href="/login"
              className={cn(
                buttonVariants({ variant: "secondary", size: "sm" }),
                "px-4"
              )}
            >
              Login
            </Link>
          </nav>
        </div>
      </header>
      <main className="flex-1">{children}</main>      {/* 👈 页面内容插入这里 */}
      <SiteFooter />                                   {/* 👈 页脚 */}
    </div>
  )
}
```

**要点：** 营销页面（首页、定价页、博客）都共享这个布局——顶部有导航栏，底部有页脚，中间放页面内容。

### 4.3 首页 `app/(marketing)/page.tsx` — Server Component 实战

```tsx
import Link from "next/link"
import { env } from "@/env.mjs"                       // 👈 环境变量（如 API Token）
import { siteConfig } from "@/config/site"
import { cn } from "@/lib/utils"
import { buttonVariants } from "@/components/ui/button"

// 👈 注意这里：这个函数在服务器上运行，浏览器永远看不到这段代码
async function getGitHubStars(): Promise<string | null> {
  //                              ^^^^^^^^^^^^^^^^^^^^^^ TypeScript 类型标注：返回 string 或 null
  try {
    const response = await fetch(                      // 👈 在服务器端调用 GitHub API
      "https://api.github.com/repos/shadcn/taxonomy",
      {
        headers: {
          Accept: "application/vnd.github+json",
          Authorization: `Bearer ${env.GITHUB_ACCESS_TOKEN}`,  // 👈 API 密钥，只存在于服务器
        },
        next: {
          revalidate: 60,  // 👈 缓存 60 秒，60 秒内重复访问不会重新请求 API
        },
      }
    )
    if (!response?.ok) { return null }
    const json = await response.json()
    return parseInt(json["stargazers_count"]).toLocaleString()
  } catch (error) {
    return null           // 👈 出错时返回 null，页面不会崩溃
  }
}

export default async function IndexPage() {           // 👈 注意：async！Server Component 才能这样写
  const stars = await getGitHubStars()                 // 👈 直接在组件里 await，不需要 useEffect

  return (
    <>                                                  {/* 👈 空标签 <>，叫 Fragment，不渲染额外 DOM */}
      <section className="space-y-6 pb-8 pt-6 md:pb-12 md:pt-10 lg:py-32">
        <div className="container flex max-w-[64rem] flex-col items-center gap-4 text-center">
          <h1 className="font-heading text-3xl sm:text-5xl md:text-6xl lg:text-7xl">
            An example app built using Next.js 13 server components.
          </h1>
          <div className="space-x-4">
            <Link href="/login" className={cn(buttonVariants({ size: "lg" }))}>
              Get Started
            </Link>
            <Link
              href={siteConfig.links.github}
              target="_blank"
              rel="noreferrer"
              className={cn(buttonVariants({ variant: "outline", size: "lg" }))}
            >
              GitHub
            </Link>
          </div>
        </div>
      </section>

      {/* Features 区域和 Open Source 区域省略... */}

      {stars && (                                       // 👈 条件渲染：stars 有值时才显示
        <Link href={siteConfig.links.github} className="flex">
          <div className="flex h-10 items-center rounded-md border border-muted bg-muted px-4 font-medium">
            {stars} stars on GitHub                     {/* 👈 大括号里放 JS 变量 */}
          </div>
        </Link>
      )}
    </>
  )
}
```

**三个关键发现：**

1. **`async function IndexPage()`** — 组件本身是 `async` 函数，可以直接 `await`。这在传统 React 中是不可能的，只有 Server Component 才行。
2. **`env.GITHUB_ACCESS_TOKEN`** — API 密钥写在服务器端代码里，不会泄露给浏览器。这是 Server Component 的安全优势。
3. **`next: { revalidate: 60 }`** — Next.js 专属的缓存策略，"60 秒内有人再访问就用缓存"。

### 4.4 仪表盘布局 `app/(dashboard)/dashboard/layout.tsx`

```tsx
import { notFound } from "next/navigation"             // 👈 显示 404 页面的函数
import { dashboardConfig } from "@/config/dashboard"
import { getCurrentUser } from "@/lib/session"          // 👈 获取当前登录用户
import { MainNav } from "@/components/main-nav"
import { DashboardNav } from "@/components/nav"         // 👈 侧边栏导航
import { SiteFooter } from "@/components/site-footer"
import { UserAccountNav } from "@/components/user-account-nav"  // 👈 用户头像下拉菜单

interface DashboardLayoutProps {
  children?: React.ReactNode     // 👈 注意 ? 表示可选
}

export default async function DashboardLayout({
  children,
}: DashboardLayoutProps) {
  const user = await getCurrentUser()                   // 👈 在服务器上获取用户信息

  if (!user) {
    return notFound()                                   // 👈 未登录则显示 404
  }

  return (
    <div className="flex min-h-screen flex-col space-y-6">
      <header className="sticky top-0 z-40 border-b bg-background">
        <div className="container flex h-16 items-center justify-between py-4">
          <MainNav items={dashboardConfig.mainNav} />
          <UserAccountNav user={{                        // 👈 传入用户信息显示头像
            name: user.name,
            image: user.image,
            email: user.email,
          }} />
        </div>
      </header>
      <div className="container grid flex-1 gap-12 md:grid-cols-[200px_1fr]">
        <aside className="hidden w-[200px] flex-col md:flex">  {/* 👈 侧边栏 */}
          <DashboardNav items={dashboardConfig.sidebarNav} />
        </aside>
        <main className="flex w-full flex-1 flex-col overflow-hidden">
          {children}
        </main>
      </div>
      <SiteFooter className="border-t" />
    </div>
  )
}
```

**对比营销布局和仪表盘布局：** 营销布局有导航栏 + 页脚，仪表盘布局多了侧边栏和用户头像。这就是 Route Groups 的作用——不同分组用不同布局。

### 4.5 仪表盘页面 `app/(dashboard)/dashboard/page.tsx`

```tsx
import { redirect } from "next/navigation"
import { authOptions } from "@/lib/auth"
import { db } from "@/lib/db"                           // 👈 数据库客户端（Prisma）
import { getCurrentUser } from "@/lib/session"
import { EmptyPlaceholder } from "@/components/empty-placeholder"
import { DashboardHeader } from "@/components/header"
import { PostCreateButton } from "@/components/post-create-button"
import { PostItem } from "@/components/post-item"
import { DashboardShell } from "@/components/shell"

export const metadata = {
  title: "Dashboard",      // 👈 浏览器标签显示 "Dashboard | Taxonomy"（配合根布局的 template）
}

export default async function DashboardPage() {
  const user = await getCurrentUser()                   // 👈 第1步：获取当前用户

  if (!user) {
    redirect(authOptions?.pages?.signIn || "/login")    // 👈 第2步：未登录就跳转
  }

  const posts = await db.post.findMany({                // 👈 第3步：直接查数据库！
    where: { authorId: user.id },
    select: { id: true, title: true, published: true, createdAt: true },
    orderBy: { updatedAt: "desc" },
  })

  return (                                              // 👈 第4步：把数据渲染成页面
    <DashboardShell>
      <DashboardHeader heading="Posts" text="Create and manage posts.">
        <PostCreateButton />
      </DashboardHeader>
      <div>
        {posts?.length ? (                              // 👈 有数据：显示列表
          <div className="divide-y divide-border rounded-md border">
            {posts.map((post) => (
              <PostItem key={post.id} post={post} />    // 👈 .map() 循环渲染每篇文章
            ))}
          </div>
        ) : (                                           // 👈 没数据：显示空状态
          <EmptyPlaceholder>
            <EmptyPlaceholder.Title>No posts created</EmptyPlaceholder.Title>
            <PostCreateButton variant="outline" />
          </EmptyPlaceholder>
        )}
      </div>
    </DashboardShell>
  )
}
```

**重要观察：** 这个页面直接用 `db.post.findMany()` 查询了数据库，没有写任何 API 接口。这是 Server Component 最强大的地方——组件就是 API。

### 4.6 加载状态 `app/(dashboard)/dashboard/loading.tsx`

```tsx
import { DashboardHeader } from "@/components/header"
import { PostCreateButton } from "@/components/post-create-button"
import { PostItem } from "@/components/post-item"
import { DashboardShell } from "@/components/shell"

export default function DashboardLoading() {
  return (
    <DashboardShell>
      <DashboardHeader heading="Posts" text="Create and manage posts.">
        <PostCreateButton />
      </DashboardHeader>
      <div className="divide-border-200 divide-y rounded-md border">
        <PostItem.Skeleton />     {/* 👈 骨架屏：灰色占位矩形，模拟真实内容的形状 */}
        <PostItem.Skeleton />
        <PostItem.Skeleton />
        <PostItem.Skeleton />
        <PostItem.Skeleton />
      </div>
    </DashboardShell>
  )
}
```

**工作原理：** 当 `page.tsx` 中的 `await db.post.findMany()` 还在等待数据库返回时，Next.js 会先显示 `loading.tsx` 的内容。数据加载完成后，自动替换为真实页面。用户看到的是"从骨架屏平滑过渡到真实内容"。

---

## 5. 动手练习

### 练习 1：创建一个 About 页面（初级，约 20 分钟）

**目标：** 在营销分组下新建一个页面。

1. 创建文件 `app/(marketing)/about/page.tsx`
2. 写入以下代码：

```tsx
export default function AboutPage() {
  return (
    <section className="container flex flex-col gap-6 py-8 md:max-w-[64rem] md:py-12">
      <h1 className="text-3xl font-bold">关于我们</h1>
      <p className="text-muted-foreground">这是一个学习 Next.js 的练习页面。</p>
    </section>
  )
}
```

3. 启动开发服务器（`pnpm dev`），访问 `http://localhost:3000/about`
4. 观察：页面是否自动有了导航栏和页脚？

**排错提示：**
- 如果页面显示 404：检查文件路径是否正确，文件名必须是 `page.tsx`（不是 `Page.tsx`，不是 `about.tsx`）
- 如果样式不对：确认 `className` 拼写正确（不是 `classname`）

### 练习 2：给 About 页面添加 Metadata（中级，约 20 分钟）

**目标：** 练习 Next.js 的 Metadata API。

在 `page.tsx` 中添加 metadata，使浏览器标签显示"About | Taxonomy"：

```tsx
// 在 export default function 之前添加：
export const metadata = {
  title: "About",
  description: "了解更多关于 Taxonomy 项目的信息",
}
```

刷新页面后，检查浏览器标签页标题是否变成了 "About | Taxonomy"。

**思考：** 为什么你只写了 `title: "About"`，浏览器却显示 "About | Taxonomy"？回头看看根布局中 `metadata.title.template` 的定义。

### 练习 3：创建一个带数据的 Server Component（进阶，约 30 分钟）

**目标：** 体会 Server Component 直接获取数据的能力。

1. 创建文件 `app/(marketing)/stars/page.tsx`
2. 写入以下代码：

```tsx
async function getRepoInfo() {
  const response = await fetch(
    "https://api.github.com/repos/vercel/next.js",
    { next: { revalidate: 3600 } }  // 缓存 1 小时
  )
  if (!response.ok) return null
  return response.json()
}

export default async function StarsPage() {
  const repo = await getRepoInfo()

  return (
    <section className="container py-8">
      <h1 className="text-3xl font-bold">Next.js GitHub 信息</h1>
      {repo ? (
        <ul className="mt-4 space-y-2">
          <li>Stars: {repo.stargazers_count?.toLocaleString()}</li>
          <li>Forks: {repo.forks_count?.toLocaleString()}</li>
          <li>Open Issues: {repo.open_issues_count?.toLocaleString()}</li>
        </ul>
      ) : (
        <p className="mt-4 text-muted-foreground">无法获取数据</p>
      )}
    </section>
  )
}
```

3. 访问 `http://localhost:3000/stars`，验证数据是否正确显示

**排错提示：**
- 如果数据全部为 null：GitHub API 对匿名请求有速率限制，稍等一会儿再试
- 如果组件报错"不能在 Client Component 中使用 async"：确认你**没有**在文件顶部写 `"use client"`

---

## 6. 概念图解

### 6.1 路由匹配流程

```
用户在浏览器输入 URL: /dashboard/settings
          │
          ▼
   Next.js App Router 开始匹配
          │
          ▼
   app/
    ├── (marketing)/  ── /dashboard 不匹配这里的 page.tsx (/)
    ├── (dashboard)/  ── 继续往下找
    │    └── dashboard/  ── 匹配！/dashboard
    │         └── settings/
    │              └── page.tsx  ── 匹配！/dashboard/settings
    │
    ▼
   找到了！渲染这个 page.tsx
```

### 6.2 布局嵌套渲染流程

```
┌─────────────────────────────────────────────┐
│  app/layout.tsx （根布局）                     │
│  ┌─────────────────────────────────────────┐ │
│  │  <html>  <body>  <ThemeProvider>        │ │
│  │                                         │ │
│  │  ┌───────────────────────────────────┐  │ │
│  │  │  app/(dashboard)/dashboard/       │  │ │
│  │  │  layout.tsx （仪表盘布局）           │  │ │
│  │  │                                   │  │ │
│  │  │  ┌──────────┐  ┌──────────────┐  │  │ │
│  │  │  │ 侧边栏    │  │  {children}  │  │  │ │
│  │  │  │ Dashboard │  │              │  │  │ │
│  │  │  │ Nav       │  │  page.tsx    │  │  │ │
│  │  │  │           │  │  的内容       │  │  │ │
│  │  │  └──────────┘  └──────────────┘  │  │ │
│  │  └───────────────────────────────────┘  │ │
│  │                                         │ │
│  │  </ThemeProvider>  </body>  </html>     │ │
│  └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘
```

### 6.3 Server Component vs Client Component 数据流

```
                    服务器端                              浏览器端
          ┌──────────────────────┐            ┌──────────────────────┐
          │                      │            │                      │
          │  Server Component    │   HTML     │   用户看到的页面       │
          │  ┌────────────────┐  │  ------→   │                      │
          │  │ await db.query │  │            │  静态内容，无需 JS     │
          │  │ await fetch()  │  │            │                      │
          │  └────────────────┘  │            └──────────────────────┘
          │                      │
          │  Client Component    │   HTML+JS  ┌──────────────────────┐
          │  ┌────────────────┐  │  ------→   │                      │
          │  │ "use client"   │  │            │  可交互内容            │
          │  │ useState       │  │            │  onClick, onChange... │
          │  │ onClick        │  │            │                      │
          │  └────────────────┘  │            └──────────────────────┘
          └──────────────────────┘
```

---

## 7. 自检问题

### Q1（基础）
`app/(marketing)/pricing/page.tsx` 对应的 URL 是什么？

<details>
<summary>答案提示</summary>

URL 是 `/pricing`。圆括号 `(marketing)` 是 Route Group，不会出现在 URL 中。

</details>

### Q2（理解）
在 `app/(marketing)/page.tsx` 中，为什么 `IndexPage` 可以是 `async` 函数？如果在组件里加上 `"use client"` 还能用 `async` 吗？

<details>
<summary>答案提示</summary>

因为 `IndexPage` 是 Server Component（默认），它在服务器上运行，服务器端支持 `async/await`。如果加上 `"use client"` 就变成了 Client Component，不能用 `async` 作为组件函数——Client Component 需要用 `useEffect` + `useState` 来处理异步数据。

</details>

### Q3（理解）
访问 `/dashboard/settings` 时，哪些 `layout.tsx` 会参与渲染？它们的嵌套顺序是什么？

<details>
<summary>答案提示</summary>

会有两层布局嵌套：
1. `app/layout.tsx`（根布局）— 提供 `<html>`, `<body>`, 主题
2. `app/(dashboard)/dashboard/layout.tsx`（仪表盘布局）— 提供导航栏、侧边栏、用户头像

嵌套顺序：根布局包裹仪表盘布局，仪表盘布局包裹 settings 页面。

</details>

### Q4（应用）
如果你想创建一个 URL 为 `/blog/2024/my-first-post` 的页面，需要创建什么文件？`params` 的值是什么？

<details>
<summary>答案提示</summary>

文件路径：`app/(marketing)/blog/[...slug]/page.tsx`（项目中已经存在这个文件）。`[...slug]` 是 Catch-all 路由。`params.slug` 的值是 `["2024", "my-first-post"]`（一个字符串数组）。

</details>

### Q5（进阶）
仪表盘布局中 `getCurrentUser()` 在服务器上运行。如果有 100 个用户同时访问 `/dashboard`，这个函数会被调用多少次？会不会"互相串数据"？

<details>
<summary>答案提示</summary>

会调用 100 次，每个请求独立运行。不会串数据——每次服务器端渲染都是独立的请求上下文，用户 A 的请求不会看到用户 B 的数据。这和传统 API 服务器处理请求的方式一样。

</details>

---

## 8. 预计时间分配

| 环节 | 时间 | 说明 |
|------|------|------|
| 前置知识速补 | 30 分钟 | 如果已有 JS 基础可跳过 |
| 核心概念清单 | 15 分钟 | 先整体浏览，建立全局印象 |
| 源码精读 | 60 分钟 | 逐文件阅读，对照注释理解 |
| 练习 1：创建页面 | 20 分钟 | 必做 |
| 练习 2：Metadata | 20 分钟 | 必做 |
| 练习 3：Server Component | 30 分钟 | 建议做，可加深理解 |
| 自检问题 | 15 分钟 | 先自己想，再看答案 |
| **合计** | **约 3 - 3.5 小时** | |

---

## 9. 常见踩坑点

### 踩坑 1：在 Server Component 中使用 `useState` / `useEffect`

**错误信息：**
```
Error: useState only works in Client Components. Add the "use client" directive at the
top of the file to use it.
```

**原因：** `useState`、`useEffect`、`onClick` 等都是浏览器端的功能，Server Component 不支持。

**解决：** 在文件最顶部（第一行）加上 `"use client"`。或者把需要交互的部分拆成单独的 Client Component。

### 踩坑 2：`layout.tsx` 放错位置，影响了不该影响的页面

**现象：** 你在 `app/(marketing)/` 下新建了一个 `layout.tsx`，结果发现首页、定价页、博客页全部受到了影响。

**原因：** `layout.tsx` 会包裹同目录及所有子目录的页面。

**解决：** 仔细规划 layout 的放置位置。如果只想影响特定页面，把 layout 放在更深层的目录中。

### 踩坑 3：文件名拼写错误

**现象：** 创建了 `Page.tsx`（大写 P）或 `pages.tsx`（多了 s），页面显示 404。

**原因：** Next.js App Router 只识别**精确的**文件名：`page.tsx`、`layout.tsx`、`loading.tsx`、`error.tsx`。大小写敏感，不能有拼写偏差。

**解决：** 严格使用小写 `page.tsx`。

### 踩坑 4：动态路由参数类型搞混

**错误信息：**
```
TypeError: params.slug.split is not a function
```

**原因：** `[slug]` 的参数类型是 `string`，而 `[...slug]` 的参数类型是 `string[]`（数组）。在 `[...slug]` 上调用字符串方法会报错。

**解决：** `[slug]` → `params.slug` 是字符串；`[...slug]` → `params.slug` 是数组，用 `params.slug.join("/")` 来拼接。

### 踩坑 5：在 Server Component 中访问 `window` 或 `document`

**错误信息：**
```
ReferenceError: window is not defined
```

**原因：** Server Component 在服务器（Node.js）上运行，服务器没有浏览器的 `window`、`document`、`localStorage` 等对象。

**解决：** 把需要访问浏览器 API 的代码放到 Client Component 中（加 `"use client"`）。

---

## 10. 延伸阅读

1. **Next.js 官方文档 — Routing Fundamentals**：搜索 `Next.js App Router documentation`，官方文档是最权威的参考，有交互式的路由示意图。
2. **Next.js 官方文档 — Server and Client Components**：搜索 `Next.js Server Components vs Client Components`，官方提供了详细的决策流程图，帮你判断何时该用哪种组件。
3. **关键词搜索建议**：`React Server Components explained`、`Next.js App Router tutorial for beginners` — 可以找到很多图文并茂的入门教程。
