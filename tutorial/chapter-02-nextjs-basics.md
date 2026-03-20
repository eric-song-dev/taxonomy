# 第2章：Next.js 基础 — 页面是怎么出现在浏览器的？

## 学习目标

- 理解 App Router 路由系统（文件即路由）
- 理解 Route Groups `(marketing)`, `(dashboard)` 等分组概念
- 理解 Server Components vs Client Components
- 理解 `page.tsx`, `layout.tsx`, `loading.tsx` 的作用

## 涉及文件

| 文件 | 说明 |
|------|------|
| `app/layout.tsx` | 根布局（所有页面共享） |
| `app/(marketing)/page.tsx` | 首页 |
| `app/(marketing)/pricing/page.tsx` | 定价页 |
| `app/(dashboard)/dashboard/page.tsx` | 仪表盘页 |
| `app/(dashboard)/dashboard/loading.tsx` | 加载状态 |
| `app/(auth)/login/page.tsx` | 登录页 |
| `app/(editor)/editor/[postId]/page.tsx` | 编辑器页（动态路由） |

---

## 1. 文件即路由 — App Router 的核心思想

在 Next.js 中，**文件的位置就决定了 URL**。这叫做"文件系统路由"。

```
app/
├── page.tsx                    → URL: /
├── (marketing)/
│   ├── page.tsx                → URL: /（首页）
│   ├── pricing/
│   │   └── page.tsx            → URL: /pricing
│   └── blog/
│       └── [...slug]/
│           └── page.tsx        → URL: /blog/任意路径
├── (dashboard)/
│   └── dashboard/
│       ├── page.tsx            → URL: /dashboard
│       ├── billing/
│       │   └── page.tsx        → URL: /dashboard/billing
│       └── settings/
│           └── page.tsx        → URL: /dashboard/settings
├── (auth)/
│   ├── login/
│   │   └── page.tsx            → URL: /login
│   └── register/
│       └── page.tsx            → URL: /register
└── (editor)/
    └── editor/
        └── [postId]/
            └── page.tsx        → URL: /editor/abc123（动态）
```

**规则很简单：** `app/xxx/page.tsx` 文件对应 URL `/xxx`。

---

## 2. 特殊文件名的含义

Next.js 的 App Router 中有几个特殊的文件名，每个都有固定的作用：

| 文件名 | 作用 | 比喻 |
|--------|------|------|
| `page.tsx` | 页面内容 | 书的正文 |
| `layout.tsx` | 页面的外壳（导航栏、侧边栏等） | 书的封面和装帧 |
| `loading.tsx` | 加载中的占位内容 | "请稍候"的提示牌 |
| `error.tsx` | 出错时显示的内容 | "出错了"的提示页 |
| `not-found.tsx` | 404 页面 | "找不到"的提示页 |

---

## 3. 根布局 — 所有页面的"外壳"

`app/layout.tsx` 是整个应用的根布局，**每个页面都会被它包裹**：

```tsx
// app/layout.tsx
import { Inter as FontSans } from "next/font/google"  // Google 字体
import localFont from "next/font/local"                // 本地字体

import "@/styles/globals.css"                          // 全局样式
import { siteConfig } from "@/config/site"             // 站点配置
import { cn } from "@/lib/utils"                       // CSS 类名工具
import { Toaster } from "@/components/ui/toaster"      // Toast 通知组件
import { Analytics } from "@/components/analytics"     // 数据统计
import { TailwindIndicator } from "@/components/tailwind-indicator"  // 开发工具
import { ThemeProvider } from "@/components/theme-provider"          // 主题（暗色/亮色）

// 加载 Inter 字体
const fontSans = FontSans({
  subsets: ["latin"],
  variable: "--font-sans",  // 定义 CSS 变量，方便在样式中使用
})

// 加载本地标题字体
const fontHeading = localFont({
  src: "../assets/fonts/CalSans-SemiBold.woff2",
  variable: "--font-heading",
})

// 页面的 SEO 元数据（在浏览器标签页和搜索引擎中显示）
export const metadata = {
  title: {
    default: siteConfig.name,                    // 默认标题："Taxonomy"
    template: `%s | ${siteConfig.name}`,         // 子页面标题格式："页面名 | Taxonomy"
  },
  description: siteConfig.description,            // 站点描述
  // ... 还有 OpenGraph、Twitter 卡片等社交媒体元数据
}

// 根布局组件
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en" suppressHydrationWarning>
      <head />
      <body
        className={cn(
          "min-h-screen bg-background font-sans antialiased",  // 基础样式
          fontSans.variable,     // 注入字体 CSS 变量
          fontHeading.variable   // 注入标题字体 CSS 变量
        )}
      >
        {/* ThemeProvider 提供暗色/亮色模式切换能力 */}
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
          {children}             {/* 这里会渲染具体的页面内容 */}
          <Analytics />          {/* Vercel 数据统计（在每个页面都生效） */}
          <Toaster />            {/* Toast 通知容器（在每个页面都能弹通知） */}
          <TailwindIndicator />  {/* 开发时显示当前断点（xs/sm/md/lg/xl） */}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

**大白话理解布局嵌套：** 根布局就像一个相框，所有页面都是放在这个相框里的照片。照片换来换去，但相框不变。

---

## 4. Route Groups — 圆括号的魔法

注意到 `app/(marketing)/`、`app/(dashboard)/` 这些带括号的文件夹了吗？这就是 **Route Groups（路由分组）**。

```
app/
├── (marketing)/       ← 营销页面分组
│   ├── layout.tsx     ← 营销页面的专属布局（有导航栏 + 页脚）
│   └── page.tsx       ← 首页 → URL: /
├── (dashboard)/       ← 后台管理分组
│   └── dashboard/
│       ├── layout.tsx ← 仪表盘的专属布局（有侧边栏）
│       └── page.tsx   ← 仪表盘 → URL: /dashboard
└── (auth)/            ← 认证页面分组
    ├── layout.tsx     ← 认证页的专属布局（居中简洁样式）
    └── login/
        └── page.tsx   ← 登录页 → URL: /login
```

**为什么需要分组？** 因为不同类型的页面需要不同的布局：
- 营销页（首页、定价页）需要导航栏 + 页脚
- 仪表盘需要侧边栏 + 用户头像
- 登录页只需要一个简洁的居中表单

**重要：** 括号不会出现在 URL 中！`(marketing)` 只是代码组织工具。

---

## 5. 首页的实际代码

让我们看看首页 `app/(marketing)/page.tsx` 是怎么写的：

```tsx
// app/(marketing)/page.tsx
import Link from "next/link"  // Next.js 提供的链接组件（比 <a> 标签更好）

import { siteConfig } from "@/config/site"         // 站点配置
import { cn } from "@/lib/utils"                    // CSS 类名工具
import { buttonVariants } from "@/components/ui/button"  // 按钮样式变体

// 在服务器端获取 GitHub 星标数
// 注意：这个函数在服务器上运行，浏览器看不到这段代码
async function getGitHubStars(): Promise<string | null> {
  try {
    const response = await fetch(
      "https://api.github.com/repos/shadcn/taxonomy",
      {
        headers: {
          Accept: "application/vnd.github+json",
          Authorization: `Bearer ${env.GITHUB_ACCESS_TOKEN}`,
        },
        next: {
          revalidate: 60,  // 每 60 秒重新获取一次（缓存策略）
        },
      }
    )
    if (!response?.ok) return null
    const json = await response.json()
    return parseInt(json["stargazers_count"]).toLocaleString()
  } catch (error) {
    return null
  }
}

// 注意这是一个 async 函数！这是 Server Component 才能做到的
export default async function IndexPage() {
  const stars = await getGitHubStars()  // 直接在组件中 await

  return (
    <>
      {/* Hero 区域 */}
      <section className="space-y-6 pb-8 pt-6 md:pb-12 md:pt-10 lg:py-32">
        <div className="container flex max-w-[64rem] flex-col items-center gap-4 text-center">
          <h1 className="font-heading text-3xl sm:text-5xl md:text-6xl lg:text-7xl">
            An example app built using Next.js 13 server components.
          </h1>
          <div className="space-x-4">
            {/* 使用 buttonVariants 获取按钮样式，应用到 Link 上 */}
            <Link href="/login" className={cn(buttonVariants({ size: "lg" }))}>
              Get Started
            </Link>
            <Link
              href={siteConfig.links.github}
              className={cn(buttonVariants({ variant: "outline", size: "lg" }))}
            >
              GitHub
            </Link>
          </div>
        </div>
      </section>

      {/* Features 区域 */}
      <section id="features" className="container space-y-6 bg-slate-50 py-8 dark:bg-transparent">
        {/* ... 展示 6 个特性卡片 */}
      </section>
    </>
  )
}
```

### 重点解析

1. **`async function IndexPage()`** — 组件是一个 async 函数！这在传统 React 中是不可能的。这就是 **Server Component** 的威力——可以直接在组件中做异步操作（如调用 API）。

2. **`next: { revalidate: 60 }`** — 这是 Next.js 的数据缓存策略。意思是"60 秒内如果有人再访问这个页面，直接用缓存，不要重新调 GitHub API"。

3. **`Link` 组件** — 来自 `next/link`，比原生 `<a>` 标签更好。它会预加载目标页面，让页面切换更快。

---

## 6. Server Components vs Client Components

这是 Next.js 13 最重要的概念之一。

| 特性 | Server Component | Client Component |
|------|-----------------|-----------------|
| 标记 | 默认就是（不用做任何事） | 文件顶部加 `"use client"` |
| 运行位置 | 只在服务器上运行 | 在浏览器中运行 |
| 能做什么 | 直接查数据库、调 API、读文件 | 使用 useState、useEffect、事件处理 |
| 不能做什么 | 不能用 useState、onClick 等 | 不能直接查数据库 |
| 发给浏览器 | 只发送渲染后的 HTML | 发送 JS 代码 |

**大白话：**
- Server Component = "厨房里做好了端出来的菜"（用户只看到结果）
- Client Component = "在用户桌上现场制作的菜"（需要在浏览器里运行 JS）

### 实际例子

```tsx
// ✅ Server Component（默认）— 可以直接查数据库
// app/(dashboard)/dashboard/page.tsx
export default async function DashboardPage() {
  const user = await getCurrentUser()        // 在服务器上获取用户
  const posts = await db.post.findMany({     // 在服务器上查数据库
    where: { authorId: user.id },
  })
  return <div>{posts.map(post => <PostItem key={post.id} post={post} />)}</div>
}
```

```tsx
// ✅ Client Component — 可以用交互功能
// components/main-nav.tsx
"use client"  // 👈 这行声明告诉 Next.js：这个组件需要在浏览器中运行

import * as React from "react"

export function MainNav({ items }) {
  // useState 是浏览器端的功能，所以需要 "use client"
  const [showMobileMenu, setShowMobileMenu] = React.useState(false)

  return (
    <div>
      <button onClick={() => setShowMobileMenu(!showMobileMenu)}>
        Menu
      </button>
      {showMobileMenu && <MobileNav items={items} />}
    </div>
  )
}
```

---

## 7. 仪表盘页面 — 服务端数据获取的实战

`app/(dashboard)/dashboard/page.tsx` 是一个很好的 Server Component 示例：

```tsx
// app/(dashboard)/dashboard/page.tsx
import { redirect } from "next/navigation"

import { authOptions } from "@/lib/auth"
import { db } from "@/lib/db"
import { getCurrentUser } from "@/lib/session"

// 设置页面标题
export const metadata = {
  title: "Dashboard",  // 浏览器标签会显示 "Dashboard | Taxonomy"
}

export default async function DashboardPage() {
  // 第1步：获取当前登录用户
  const user = await getCurrentUser()

  // 第2步：如果没登录，跳转到登录页
  if (!user) {
    redirect(authOptions?.pages?.signIn || "/login")
  }

  // 第3步：从数据库查询该用户的所有帖子
  const posts = await db.post.findMany({
    where: {
      authorId: user.id,  // 只查当前用户的帖子
    },
    select: {
      id: true,           // 只取需要的字段（性能优化）
      title: true,
      published: true,
      createdAt: true,
    },
    orderBy: {
      updatedAt: "desc",  // 按更新时间倒序排列
    },
  })

  // 第4步：渲染页面
  return (
    <DashboardShell>
      <DashboardHeader heading="Posts" text="Create and manage posts.">
        <PostCreateButton />
      </DashboardHeader>
      <div>
        {posts?.length ? (
          // 有帖子：显示帖子列表
          <div className="divide-y divide-border rounded-md border">
            {posts.map((post) => (
              <PostItem key={post.id} post={post} />
            ))}
          </div>
        ) : (
          // 没帖子：显示空状态占位符
          <EmptyPlaceholder>
            <EmptyPlaceholder.Icon name="post" />
            <EmptyPlaceholder.Title>No posts created</EmptyPlaceholder.Title>
            <EmptyPlaceholder.Description>
              You don&apos;t have any posts yet. Start creating content.
            </EmptyPlaceholder.Description>
            <PostCreateButton variant="outline" />
          </EmptyPlaceholder>
        )}
      </div>
    </DashboardShell>
  )
}
```

注意这里**没有** `"use client"`，也**没有** `useEffect`。数据是在服务器上获取的，直接传给了 React 组件来渲染。

---

## 8. 动态路由 — 方括号的魔法

`app/(editor)/editor/[postId]/page.tsx` 中的 `[postId]` 是**动态路由参数**：

```
URL: /editor/abc123
       └── [postId] = "abc123"

URL: /editor/xyz789
       └── [postId] = "xyz789"
```

方括号告诉 Next.js："这个位置可以是任意值，把它当作参数传给页面组件。"

还有一种特殊的动态路由——**Catch-all Routes（全捕获路由）**：

```
[...slug]     → 匹配一个或多个路径段
               /blog/hello        → slug = ["hello"]
               /blog/2024/hello   → slug = ["2024", "hello"]

[[...slug]]   → 匹配零个或多个路径段（可选）
               /docs              → slug = undefined
               /docs/getting-started → slug = ["getting-started"]
```

---

## 9. Loading 状态

`app/(dashboard)/dashboard/loading.tsx` 定义了页面加载时的占位内容。当仪表盘页面在服务器上获取数据时，用户会先看到这个 loading 状态：

```tsx
// app/(dashboard)/dashboard/loading.tsx
import { DashboardHeader } from "@/components/header"
import { PostCreateButton } from "@/components/post-create-button"
import { PostItem } from "@/components/post-item"
import { DashboardShell } from "@/components/shell"
import { CardSkeleton } from "@/components/card-skeleton"

export default function DashboardLoading() {
  return (
    <DashboardShell>
      <DashboardHeader heading="Posts" text="Create and manage posts.">
        <PostCreateButton />
      </DashboardHeader>
      <div className="divide-y divide-border rounded-md border">
        <CardSkeleton />   {/* 骨架屏 —— 灰色的占位矩形 */}
        <CardSkeleton />
        <CardSkeleton />
      </div>
    </DashboardShell>
  )
}
```

**骨架屏（Skeleton）** 是一种加载占位符，用灰色的矩形模拟最终内容的形状，比空白页面或转圈圈的用户体验更好。

---

## 小练习

### 练习 1：创建一个新页面
1. 在 `app/(marketing)/` 下创建一个 `about/page.tsx` 文件
2. 写一个简单的组件：
```tsx
export default function AboutPage() {
  return <div className="container py-10"><h1>About Us</h1></div>
}
```
3. 访问 `http://localhost:3000/about`，看看是否显示了你的页面
4. 注意：这个页面自动继承了营销布局的导航栏和页脚！

### 练习 2：理解布局嵌套
打开浏览器开发者工具，分别访问 `/`（首页）和 `/dashboard`（仪表盘）。对比两个页面的导航栏有什么不同。想想为什么会不同——提示：看看各自的 `layout.tsx`。

### 练习 3：Server vs Client
搜索项目中所有包含 `"use client"` 的文件。数一数有多少个。然后思考：为什么大部分文件没有这行代码？

---

## 本章小结

- Next.js 的 App Router 让文件路径直接对应 URL
- `page.tsx` = 页面内容，`layout.tsx` = 页面外壳，`loading.tsx` = 加载占位
- Route Groups（圆括号文件夹）用于组织代码和共享布局，不影响 URL
- Server Components 默认在服务器上运行，可以直接查数据库
- Client Components 需要标记 `"use client"`，可以使用交互功能
- 动态路由用方括号 `[param]` 表示

下一章我们将学习 UI 组件是怎么搭建的。
