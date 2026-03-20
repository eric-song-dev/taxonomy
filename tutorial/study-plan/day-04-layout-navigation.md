# Day 04 — 布局与导航：给页面搭骨架

> 对应章节：[Chapter 4: 布局与导航](../chapter-04-layout-navigation.md)

---

## 今日学习目标

1. **理解嵌套布局（Nested Layouts）**：知道 Next.js 是如何像套娃一样一层层包裹页面的，以及这样做的性能好处。
2. **掌握配置驱动的导航菜单**：能读懂 `config/` 文件 + 导航组件的协作模式，并自己添加菜单项。
3. **理解暗色模式的完整实现**：从 ThemeProvider 到 ModeToggle 按钮，搞清楚"点一下就变暗"背后发生了什么。

---

## 前置知识速补

今天的代码会涉及一些新的 React 概念。如果你是第一次见到它们，不要慌——我们先用最简单的方式过一遍。

### 1. [useState](https://react.dev/reference/react/useState) — 让组件"记住"一个值

在前面几天，我们写的组件都是"无状态"的——给什么 props 就渲染什么，自己不记任何东西。但有时候组件需要自己维护一些数据，比如"移动端菜单是打开还是关闭"。这就是 `useState` 的用处。

```tsx
import { useState } from "react"

function Counter() {
  // useState 返回一个数组，包含两个东西：
  //   count     —— 当前值（初始值是 0）
  //   setCount  —— 用来更新这个值的函数
  const [count, setCount] = useState(0)

  return (
    <div>
      <p>你点了 {count} 次</p>
      {/* 每次点击，setCount 把 count 更新为 count + 1 */}
      <button onClick={() => setCount(count + 1)}>
        点我
      </button>
    </div>
  )
}
```

**要点：**
- `useState(初始值)` —— 括号里放初始值
- 返回 `[当前值, 更新函数]`，名字可以随便起，但约定是 `[xxx, setXxx]`
- 调用 `setXxx(新值)` 后，React 会自动重新渲染这个组件

### 2. [useEffect](https://react.dev/reference/react/useEffect) — 组件挂载后做点事

有时候你想在组件"出现在页面上"之后做一些事情，比如锁定页面滚动（移动端菜单打开时）。这就是 `useEffect` 的用途。

```tsx
import { useEffect } from "react"

function MyComponent() {
  useEffect(() => {
    // 这段代码在组件"挂载到页面上"之后执行
    console.log("组件出现了！")

    // 返回一个函数 = 组件"从页面移除"时执行（清理工作）
    return () => {
      console.log("组件消失了，做清理工作")
    }
  }, [])  // 空数组 [] 表示"只在第一次挂载时执行"

  return <div>Hello</div>
}
```

**要点：**
- `useEffect(回调函数, 依赖数组)` —— 依赖数组里放"我关心的变量"
- `[]` 空数组 = 只执行一次；`[count]` = 每次 count 变了就重新执行
- 今天你会在 `MobileNav` 的 `useLockBody` hook 里见到它

### 3. [`children` prop](https://react.dev/learn/passing-props-to-a-component#passing-jsx-as-children) — 组件嵌套的秘密

这是今天最重要的概念之一。还记得 HTML 里标签可以嵌套吗？

```html
<div>
  <p>我是被包裹的内容</p>
</div>
```

React 组件也可以这样嵌套，被包裹的内容会通过一个特殊的 prop 叫 `children` 传进来：

```tsx
// 定义一个"盒子"组件
function Box({ children }: { children: React.ReactNode }) {
  return (
    <div style={{ border: "1px solid black", padding: 16 }}>
      {children}  {/* 这里会渲染被包裹的内容 */}
    </div>
  )
}

// 使用：<Box> 和 </Box> 之间的东西就是 children
<Box>
  <p>我是被包裹的内容</p>
</Box>
```

**这和布局有什么关系？** Next.js 的 `layout.tsx` 就是这个模式——布局组件接收 `children`，然后把它放在导航栏和页脚之间渲染。

### 4. [React Context](https://react.dev/reference/react/createContext) — 全局共享数据

假设你有一个数据（比如"当前主题是亮色还是暗色"），需要在很多组件里用到。一个一个通过 props 传太麻烦了。Context 就是让你"广播"一个数据，任何子组件都能直接拿到。

```tsx
// 第 1 步：创建一个 Context + Provider 组件
const ThemeContext = React.createContext("light")

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light")
  return (
    <ThemeContext.Provider value={theme}>
      {children}
    </ThemeContext.Provider>
  )
}

// 第 2 步：在最外层包裹
<ThemeProvider>
  <整个App />
</ThemeProvider>

// 第 3 步：在任意子组件里直接拿
function SomeDeepComponent() {
  const theme = useContext(ThemeContext)  // 拿到 "light" 或 "dark"
  return <div>当前主题：{theme}</div>
}
```

**今天的应用：** `ThemeProvider` 就是用这个模式把主题数据共享给全部组件的。子组件通过 [`useContext`](https://react.dev/reference/react/useContext) 直接读取共享数据。

### 5. 事件处理（`onClick`）

在 React 里，给元素绑定点击事件用 `onClick`（注意大写的 C）：

```tsx
<button onClick={() => console.log("被点了！")}>
  点我
</button>
```

箭头函数 `() => ...` 的意思是"定义一个匿名函数"。你也可以先定义再传：

```tsx
function handleClick() {
  console.log("被点了！")
}

<button onClick={handleClick}>点我</button>
```

### 6. TypeScript [泛型](https://www.typescriptlang.org/docs/handbook/2/generics.html)基础 `<T>`

你会在代码里见到类似 `useState<boolean>(false)` 这样的写法。尖括号里的 `boolean` 是"泛型参数"，意思是"告诉 TypeScript 这个 state 的类型是布尔值"。

```tsx
const [showMenu, setShowMenu] = useState<boolean>(false)
//                                       ^^^^^^^^^ 告诉 TS：showMenu 是 boolean 类型
```

大多数时候 TypeScript 能自己推断类型，所以泛型是可选的。但写上去可以让代码更清晰。

---

## 核心概念清单

| 概念 | 一句话说明 | 对应文件 |
|------|-----------|---------|
| [嵌套布局（Nested Layouts）](https://nextjs.org/docs/app/building-your-application/routing/layouts-and-templates) | 布局一层套一层，像俄罗斯套娃；切换同组页面时外层布局不会重新渲染 | `app/layout.tsx`、各 `layout.tsx` |
| 配置驱动导航 | 菜单项定义在 `config/` 文件里，组件从配置读取并渲染，增删菜单不用改组件代码 | `config/marketing.ts`、`config/dashboard.ts` |
| [响应式设计](https://tailwindcss.com/docs/responsive-design) | 用 Tailwind 的 `md:`、`lg:` 等前缀，让同一套代码在手机和电脑上显示不同的布局 | `components/main-nav.tsx` |
| [暗色模式](https://tailwindcss.com/docs/dark-mode) | 通过 [next-themes](https://github.com/pacocoursey/next-themes) 库 + Tailwind 的 `dark:` 前缀，给 `<html>` 加 `class="dark"` 实现切换 | `components/theme-provider.tsx`、`components/mode-toggle.tsx` |

---

## 源码精读

### 精读 1：根布局 `app/layout.tsx` — 最外层的"套娃"

这是整个网站最外层的壳。无论你访问哪个页面，都会经过这个布局。

```tsx
// app/layout.tsx（简化版，只看关键部分）

// ① 导入字体
import { Inter as FontSans } from "next/font/google"
import localFont from "next/font/local"

// ② 导入全局样式
import "@/styles/globals.css"

// ③ 导入配置和组件
import { siteConfig } from "@/config/site"
import { cn } from "@/lib/utils"
import { ThemeProvider } from "@/components/theme-provider"
import { TailwindIndicator } from "@/components/tailwind-indicator"

// ④ 定义 props 类型：这个布局接收 children
interface RootLayoutProps {
  children: React.ReactNode  // React.ReactNode = "任何能渲染的东西"
}

// ⑤ metadata 对象 —— 用于 SEO（搜索引擎优化）
export const metadata = {
  title: {
    default: siteConfig.name,              // 默认标题："Taxonomy"
    template: `%s | ${siteConfig.name}`,   // 子页面标题格式："页面名 | Taxonomy"
  },
  description: siteConfig.description,
  // ...还有 openGraph、twitter 等社交分享配置
}

// ⑥ 布局组件本体
export default function RootLayout({ children }: RootLayoutProps) {
  return (
    <html lang="en" suppressHydrationWarning>
      <head />
      <body
        className={cn(
          "min-h-screen bg-background font-sans antialiased",
          fontSans.variable,    // 把字体变量注入到 body 上
          fontHeading.variable
        )}
      >
        {/* ⑦ ThemeProvider 包裹一切 —— 让所有子组件都能访问主题数据 */}
        <ThemeProvider attribute="class" defaultTheme="system" enableSystem>
          {children}           {/* ← 子布局和页面会渲染在这里 */}
          <TailwindIndicator /> {/* 开发时显示当前断点的小工具 */}
        </ThemeProvider>
      </body>
    </html>
  )
}
```

**逐行重点：**

- **`children: React.ReactNode`**：这就是前面说的 `children` prop。根布局接收子内容，放在 `<body>` 里渲染。
- **[next/font](https://nextjs.org/docs/app/building-your-application/optimizing/fonts)**：Next.js 内置的字体优化方案，自动下载和托管 Google Fonts，避免布局偏移。
- **[metadata](https://nextjs.org/docs/app/building-your-application/optimizing/metadata)** 对象：Next.js App Router 提供的 SEO 配置方式，定义页面标题、描述等元信息。
- **`ThemeProvider`**：用 Context 模式包裹整个应用。`attribute="class"` 表示通过给 `<html>` 加 `class="dark"` 来切换暗色模式。`defaultTheme="system"` 表示默认跟随操作系统设置。
- **`suppressHydrationWarning`**：因为 `next-themes` 会在客户端修改 `<html>` 的 class，服务端和客户端可能不一致，加这个属性告诉 React "这里不一致没关系，别报警告"。
- **`cn(...)`**：一个工具函数，把多个 CSS 类名合并成一个字符串，并处理冲突。

### 精读 2：营销页布局 `app/(marketing)/layout.tsx` — 第二层套娃

`(marketing)` 是 Next.js 的[路由组（Route Groups）](https://nextjs.org/docs/app/building-your-application/routing/route-groups)语法——括号表示这个文件夹只用于组织代码，不会出现在 URL 中。当你访问 `/pricing` 或 `/blog` 这些公开页面时，会经过这个布局。

```tsx
// app/(marketing)/layout.tsx

import Link from "next/link"
import { marketingConfig } from "@/config/marketing"   // ← 从配置文件读取导航项
import { cn } from "@/lib/utils"
import { buttonVariants } from "@/components/ui/button"
import { MainNav } from "@/components/main-nav"
import { SiteFooter } from "@/components/site-footer"

// children 的类型定义
interface MarketingLayoutProps {
  children: React.ReactNode
}

export default async function MarketingLayout({
  children,       // ← 这里的 children 就是具体的页面内容
}: MarketingLayoutProps) {
  return (
    // flex + min-h-screen + flex-col：让整个页面至少占满一屏高度
    <div className="flex min-h-screen flex-col">

      {/* === 顶部导航栏 === */}
      <header className="container z-40 bg-background">
        <div className="flex h-20 items-center justify-between py-6">
          {/* 左侧：Logo + 导航菜单（从配置文件读取） */}
          <MainNav items={marketingConfig.mainNav} />
          {/* 右侧：登录按钮 */}
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

      {/* === 主内容区 === */}
      {/* flex-1 的意思：占据所有剩余空间，把 footer 推到底部 */}
      <main className="flex-1">{children}</main>

      {/* === 页脚 === */}
      <SiteFooter />
    </div>
  )
}
```

**画个图帮你理解布局结构：**

```
┌─────────────────────────────────────┐
│  Logo   Features  Pricing  Blog  Login  │  <-- header（导航栏）
├─────────────────────────────────────┤
│                                         │
│            {children}                   │  <-- main（flex-1，撑满剩余空间）
│        （页面内容在这里渲染）              │
│                                         │
├─────────────────────────────────────┤
│         Built by shadcn                 │  <-- footer（页脚）
└─────────────────────────────────────┘
```

**重点理解 `flex-1`：** 想象一个弹簧——header 和 footer 各占固定高度，中间的 `main` 像弹簧一样把剩余空间全部撑开，这样 footer 永远在底部。注意代码中使用了 Next.js 的 [Link](https://nextjs.org/docs/app/api-reference/components/link) 组件代替 `<a>` 标签，实现客户端导航。

### 精读 3：仪表盘布局 `app/(dashboard)/dashboard/layout.tsx` — 另一种套法

管理后台用的是不同的布局：顶部导航 + 左侧边栏。

```tsx
// app/(dashboard)/dashboard/layout.tsx

import { notFound } from "next/navigation"
import { dashboardConfig } from "@/config/dashboard"
import { getCurrentUser } from "@/lib/session"
import { MainNav } from "@/components/main-nav"
import { DashboardNav } from "@/components/nav"
import { SiteFooter } from "@/components/site-footer"
import { UserAccountNav } from "@/components/user-account-nav"

interface DashboardLayoutProps {
  children?: React.ReactNode   // 注意这里用了 ? 表示可选
}

export default async function DashboardLayout({
  children,
}: DashboardLayoutProps) {
  // 在服务端获取当前用户（Server Component 才能用 await）
  const user = await getCurrentUser()

  // 没有登录？显示 404 页面
  if (!user) {
    return notFound()
  }

  return (
    <div className="flex min-h-screen flex-col space-y-6">

      {/* 顶部导航栏（sticky = 滚动时固定在顶部） */}
      <header className="sticky top-0 z-40 border-b bg-background">
        <div className="container flex h-16 items-center justify-between py-4">
          <MainNav items={dashboardConfig.mainNav} />
          {/* 右上角：用户头像 + 下拉菜单 */}
          <UserAccountNav
            user={{
              name: user.name,
              image: user.image,
              email: user.email,
            }}
          />
        </div>
      </header>

      {/* 内容区：左侧边栏 + 右主内容 */}
      <div className="container grid flex-1 gap-12 md:grid-cols-[200px_1fr]">
        {/* 侧边栏：手机上隐藏，md 以上屏幕才显示 */}
        <aside className="hidden w-[200px] flex-col md:flex">
          <DashboardNav items={dashboardConfig.sidebarNav} />
        </aside>
        {/* 主内容 */}
        <main className="flex w-full flex-1 flex-col overflow-hidden">
          {children}
        </main>
      </div>

      <SiteFooter className="border-t" />
    </div>
  )
}
```

**布局结构图：**

```
┌──────────────────────────────────────┐
│  Logo   Docs  Support        👤 User  │  <-- sticky header（固定顶栏）
├──────────┬───────────────────────────┤
│ Posts    │                           │
│ Billing  │       {children}          │
│ Settings │    （页面内容在这里）        │
│          │                           │
├──────────┴───────────────────────────┤
│              Footer                   │
└──────────────────────────────────────┘
```

**关键 CSS 解读：**
- `md:grid-cols-[200px_1fr]` —— 中等屏幕以上分两列：左边 200px，右边占满
- `hidden md:flex` —— 手机上隐藏侧边栏，电脑上显示
- `sticky top-0` —— 滚动时导航栏固定在顶部不动

### 精读 4：嵌套是怎么发生的？

当你访问 `/pricing` 时，Next.js 实际渲染的结构是这样的：

```
app/layout.tsx（根布局）
  └── app/(marketing)/layout.tsx（营销布局）
        └── app/(marketing)/pricing/page.tsx（定价页面）
```

翻译成组件嵌套就是：

```tsx
<RootLayout>                    {/* 提供字体、主题、基础样式 */}
  <MarketingLayout>              {/* 提供导航栏、页脚 */}
    <PricingPage />              {/* 真正的页面内容 */}
  </MarketingLayout>
</RootLayout>
```

**性能好处：** 当你从 `/pricing` 跳到 `/blog`，两个页面共享同一个 `MarketingLayout`。Next.js 知道这一点，所以不会重新渲染导航栏和页脚——只替换 `{children}` 部分的内容。这就是嵌套布局的核心价值。

### 精读 5：导航组件 `components/main-nav.tsx`

这是今天最复杂的组件，我们一段段来看。

```tsx
// components/main-nav.tsx
"use client"   // ← 因为用了 useState 和 useSelectedLayoutSegment，必须声明为客户端组件

import * as React from "react"
import Link from "next/link"
import { useSelectedLayoutSegment } from "next/navigation"

import { MainNavItem } from "types"
import { siteConfig } from "@/config/site"
import { cn } from "@/lib/utils"
import { Icons } from "@/components/icons"
import { MobileNav } from "@/components/mobile-nav"

// 定义这个组件接收的 props 类型
interface MainNavProps {
  items?: MainNavItem[]      // 导航项数组（可选）
  children?: React.ReactNode // 子内容（可选）
}

export function MainNav({ items, children }: MainNavProps) {
  // 获取当前 URL 的第一段路径
  // 比如 /pricing → "pricing"，/blog/hello → "blog"
  const segment = useSelectedLayoutSegment()

  // 移动端菜单的开关状态：false = 关闭，true = 打开
  const [showMobileMenu, setShowMobileMenu] = React.useState<boolean>(false)

  return (
    <div className="flex gap-6 md:gap-10">

      {/* ---- Logo（桌面端才显示）---- */}
      <Link href="/" className="hidden items-center space-x-2 md:flex">
        <Icons.logo />
        <span className="hidden font-bold sm:inline-block">
          {siteConfig.name}     {/* 从配置文件读取站点名 */}
        </span>
      </Link>

      {/* ---- 桌面端导航菜单（桌面端才显示）---- */}
      {items?.length ? (           // items?.length 的意思：如果 items 存在且长度 > 0
        <nav className="hidden gap-6 md:flex">
          {items?.map((item, index) => (  // 遍历每个导航项
            <Link
              key={index}
              href={item.disabled ? "#" : item.href}  // 禁用的链接指向 #
              className={cn(
                "flex items-center text-lg font-medium transition-colors hover:text-foreground/80 sm:text-sm",
                // 高亮逻辑：当前页面对应的菜单项用实色，其他用半透明
                item.href.startsWith(`/${segment}`)
                  ? "text-foreground"       // 当前页：实色
                  : "text-foreground/60",   // 其他页：60% 透明度
                item.disabled && "cursor-not-allowed opacity-80"
              )}
            >
              {item.title}
            </Link>
          ))}
        </nav>
      ) : null}

      {/* ---- 移动端汉堡菜单按钮（手机端才显示）---- */}
      <button
        className="flex items-center space-x-2 md:hidden"
        onClick={() => setShowMobileMenu(!showMobileMenu)}  // 点击切换开关
      >
        {showMobileMenu ? <Icons.close /> : <Icons.logo />}
        <span className="font-bold">Menu</span>
      </button>

      {/* ---- 移动端展开的菜单 ---- */}
      {showMobileMenu && items && (
        <MobileNav items={items}>{children}</MobileNav>
      )}
    </div>
  )
}
```

**响应式设计的核心技巧：**

```
桌面端（宽度 >= 768px）            手机端（宽度 < 768px）
──────────────────               ──────────────────
Logo  Features  Pricing  Blog     [Logo] Menu
      ↑ hidden md:flex                ↑ flex md:hidden
      只有 md 以上才显示               只有 md 以下才显示
```

Tailwind 是 **mobile-first**（移动优先）的：
- 不带前缀 = 默认样式（手机端生效）
- `md:` 前缀 = 768px 以上才生效
- `lg:` 前缀 = 1024px 以上才生效

所以 `hidden md:flex` 的意思是："默认隐藏，768px 以上才显示为 flex 布局"。

### 精读 6：配置驱动导航 — `config/` 文件

导航菜单项不是写死在组件里的，而是集中定义在配置文件中。

```typescript
// config/marketing.ts
import { MarketingConfig } from "types"

export const marketingConfig: MarketingConfig = {
  mainNav: [
    { title: "Features",      href: "/#features" },
    { title: "Pricing",       href: "/pricing" },
    { title: "Blog",          href: "/blog" },
    { title: "Documentation", href: "/docs" },
  ],
}
```

```typescript
// config/dashboard.ts
import { DashboardConfig } from "types"

export const dashboardConfig: DashboardConfig = {
  mainNav: [
    { title: "Documentation", href: "/docs" },
    { title: "Support",       href: "/support", disabled: true },  // 禁用状态
  ],
  sidebarNav: [
    { title: "Posts",    href: "/dashboard",          icon: "post" },
    { title: "Billing",  href: "/dashboard/billing",  icon: "billing" },
    { title: "Settings", href: "/dashboard/settings", icon: "settings" },
  ],
}
```

**数据流向：**

```
config/marketing.ts          布局文件                        组件
─────────────────           ──────────                     ──────
marketingConfig.mainNav  →  <MainNav items={...} />  →  items.map() 渲染每个链接
```

**好处：** 想加一个菜单项？只需要在配置文件里加一行 `{ title: "About", href: "/about" }`，不用碰任何组件代码。

### 精读 7：暗色模式实现

暗色模式的实现分两步：Provider（提供数据）和 Toggle（切换按钮）。

**第 1 步：ThemeProvider 包裹整个应用**

```tsx
// components/theme-provider.tsx
"use client"

import * as React from "react"
import { ThemeProvider as NextThemesProvider } from "next-themes"
import { ThemeProviderProps } from "next-themes/dist/types"

// 这就是一个简单的包装组件
// 把 next-themes 的 Provider 重新导出，方便项目内使用
export function ThemeProvider({ children, ...props }: ThemeProviderProps) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>
}
```

在根布局中这样使用（回顾之前看到的 `app/layout.tsx`）：

```tsx
<ThemeProvider attribute="class" defaultTheme="system" enableSystem>
  {children}
</ThemeProvider>
```

- `attribute="class"` —— 通过给 `<html>` 添加 `class="dark"` 来切换模式
- `defaultTheme="system"` —— 默认跟随操作系统
- `enableSystem` —— 允许系统偏好模式

**第 2 步：ModeToggle 切换按钮**

```tsx
// components/mode-toggle.tsx
"use client"

import * as React from "react"
import { useTheme } from "next-themes"    // ← 从 Context 中拿到主题相关的函数
// ...省略 import

export function ModeToggle() {
  const { setTheme } = useTheme()  // 拿到设置主题的函数

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="sm" className="h-8 w-8 px-0">
          {/* 太阳图标：亮色时正常显示，暗色时旋转缩小消失 */}
          <Icons.sun className="rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
          {/* 月亮图标：亮色时隐藏，暗色时正常显示 */}
          <Icons.moon className="absolute rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
          <span className="sr-only">Toggle theme</span>
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        {/* 点击后调用 setTheme 切换主题 */}
        <DropdownMenuItem onClick={() => setTheme("light")}>Light</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("dark")}>Dark</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("system")}>System</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

**点击"Dark"后发生了什么？完整链路：**

```
你点击 "Dark"
  → setTheme("dark") 被调用
  → next-themes 把 <html> 的 class 改成 "dark"
  → Tailwind 的 dark: 前缀全部生效
  → 所有 dark:bg-xxx、dark:text-xxx 样式自动切换
  → 页面变暗 ✓
```

两个图标的动画技巧：太阳和月亮重叠在同一位置（`absolute`），通过 `scale-0/scale-100` 控制谁显示，`rotate` 添加旋转过渡动画。

---

## 动手练习

### 练习 1：添加一个导航项（20 分钟）

**目标：** 在营销页导航栏添加一个 "About" 链接。

**步骤：**

1. 打开 `config/marketing.ts`
2. 在 `mainNav` 数组里添加一项：
   ```ts
   {
     title: "About",
     href: "/about",
   },
   ```
3. 保存文件，刷新浏览器
4. 你应该能在导航栏看到新的 "About" 链接
5. 点击它——会显示 404，因为还没创建 `/about` 页面（这是正常的）

**思考题：** 为什么不需要修改 `MainNav` 组件的代码就能新增菜单项？

> 提示：因为 `MainNav` 用了 `items.map()` 遍历数组渲染链接——数组多了一项，自动多渲染一个链接。

**排错提示：**
- 如果看不到变化，检查是否保存了文件
- 如果报 TypeScript 错误，检查新加的对象是否有 `title` 和 `href` 两个字段
- 注意数组项之间用逗号分隔

### 练习 2：观察布局嵌套行为（25 分钟）

**目标：** 验证"切换同组页面时布局不重新渲染"这个特性。

**步骤：**

1. 打开 `app/(marketing)/layout.tsx`，在 `return` 语句前添加一行：
   ```tsx
   console.log("=== marketing layout rendered ===")
   ```
2. 打开 `app/(marketing)/pricing/page.tsx`（或任意一个 marketing 页面），也加一行 `console.log`
3. 启动开发服务器（`npm run dev`）
4. 打开浏览器访问首页，然后导航到 `/pricing`
5. 观察终端输出：
   - 布局的 log 是否每次都出现？
   - 页面的 log 呢？

**注意：** 因为这些是 Server Component，`console.log` 输出在**终端**而非浏览器控制台。

**完成后记得删除添加的 `console.log`。**

### 练习 3：测试暗色模式（15 分钟）

**目标：** 亲手体验暗色模式切换，并理解背后的 CSS 原理。

**步骤：**

1. 在浏览器中找到主题切换按钮（通常在页脚或导航栏附近）
2. 切换到 Dark 模式
3. 打开浏览器开发者工具（F12），查看 `<html>` 标签：
   - 亮色模式下：`<html lang="en" class="light" ...>`
   - 暗色模式下：`<html lang="en" class="dark" ...>`
4. 在 Elements 面板搜索 `bg-background`，观察在两种模式下的计算值有何不同

**排错提示：**
- 如果切换后页面没变化，检查 `app/layout.tsx` 中 `ThemeProvider` 是否有 `attribute="class"` 参数
- 如果刷新后模式重置，可能是 `ThemeProvider` 没正确配置 `enableSystem`

---

## 概念图解

### 图 1：布局嵌套关系

```
访问 /pricing:

┌── app/layout.tsx ──────────────────────────┐
│  <html>                                     │
│    <body>                                   │
│      <ThemeProvider>                         │
│        ┌── (marketing)/layout.tsx ────────┐  │
│        │  <header> 导航栏 </header>        │  │
│        │  <main>                           │  │
│        │    ┌── pricing/page.tsx ────────┐ │  │
│        │    │  定价页面的实际内容         │ │  │
│        │    └───────────────────────────┘ │  │
│        │  </main>                          │  │
│        │  <footer> 页脚 </footer>          │  │
│        └──────────────────────────────────┘  │
│      </ThemeProvider>                        │
│    </body>                                   │
│  </html>                                     │
└─────────────────────────────────────────────┘

访问 /dashboard:

┌── app/layout.tsx ──────────────────────────┐
│  <html>                                     │
│    <body>                                   │
│      <ThemeProvider>                         │
│        ┌── (dashboard)/dashboard/layout ──┐  │
│        │  <header> 导航栏 + 用户头像 </header>│
│        │  ┌─────────┬─────────────────┐   │  │
│        │  │ 侧边栏   │  dashboard/     │   │  │
│        │  │ Posts    │  page.tsx       │   │  │
│        │  │ Billing  │  的实际内容     │   │  │
│        │  │ Settings │                │   │  │
│        │  └─────────┴─────────────────┘   │  │
│        │  <footer />                       │  │
│        └──────────────────────────────────┘  │
│      </ThemeProvider>                        │
│    </body>                                   │
│  </html>                                     │
└─────────────────────────────────────────────┘
```

### 图 2：组件树（以营销页为例）

```
RootLayout
  └─ ThemeProvider            ← Context Provider（提供主题数据）
       ├─ MarketingLayout     ← 营销页布局
       │    ├─ <header>
       │    │    ├─ MainNav   ← Client Component（用了 useState）
       │    │    │    ├─ Logo
       │    │    │    ├─ 桌面导航链接 (hidden md:flex)
       │    │    │    └─ 手机汉堡按钮 (flex md:hidden)
       │    │    │         └─ MobileNav（条件渲染）
       │    │    └─ Login 按钮
       │    ├─ <main>
       │    │    └─ {children} → Page 组件
       │    └─ SiteFooter
       ├─ Analytics
       ├─ Toaster
       └─ TailwindIndicator
```

### 图 3：数据流向（导航配置 → 页面渲染）

```
config/marketing.ts                     组件渲染
─────────────────                     ──────────
mainNav: [                            ┌──────────────────────┐
  { title: "Features", href: "..." }  │  Features  Pricing   │
  { title: "Pricing",  href: "..." }  →  Blog  Documentation │
  { title: "Blog",     href: "..." }  │                      │
  { title: "Documentation", ... }     └──────────────────────┘
]
         ↓
    MarketingLayout
    <MainNav items={marketingConfig.mainNav} />
         ↓
    MainNav 组件
    items.map(item => <Link>{item.title}</Link>)
```

---

## 自检问题

完成今天的学习后，试试不看代码回答以下问题。

### 问题 1：嵌套布局的性能优势是什么？

<details>
<summary>点击查看答案</summary>

当用户在同一组页面之间导航时（比如从 `/pricing` 跳到 `/blog`），它们共享的布局（`MarketingLayout`）不会重新渲染。只有 `{children}` 部分（即具体页面内容）会被替换。这意味着导航栏、页脚等不用重新加载，减少了不必要的渲染和网络请求，页面切换更快。

</details>

### 问题 2：`useSelectedLayoutSegment()` 返回什么？它是怎么帮助实现导航高亮的？

<details>
<summary>点击查看答案</summary>

它返回当前 URL 的第一段路径。比如访问 `/pricing` 时返回 `"pricing"`，访问 `/blog/hello` 时返回 `"blog"`。在 `MainNav` 组件中，通过判断 `item.href.startsWith(\`/\${segment}\`)` 来决定当前菜单项是否应该高亮——如果匹配，用实色 `text-foreground`；不匹配，用半透明 `text-foreground/60`。

</details>

### 问题 3：`"use client"` 是什么意思？为什么 `MainNav` 需要它而 `MarketingLayout` 不需要？

<details>
<summary>点击查看答案</summary>

`"use client"` 声明这个文件是客户端组件（Client Component），意味着它的代码会在浏览器中运行。`MainNav` 需要它是因为用了 `useState`（管理移动菜单开关状态）和 `useSelectedLayoutSegment()`（获取当前路由段），这些都是 React hooks，只能在客户端运行。而 `MarketingLayout` 只做简单的组件组合和渲染，不需要 hooks，所以可以保持为 Server Component（默认行为）。

</details>

### 问题 4：如果要给 Dashboard 侧边栏添加一个 "Analytics" 菜单项，需要修改哪个文件？怎么改？

<details>
<summary>点击查看答案</summary>

只需要修改 `config/dashboard.ts`，在 `sidebarNav` 数组中添加一项：

```ts
{
  title: "Analytics",
  href: "/dashboard/analytics",
  icon: "analytics",  // 需要确保 icons.tsx 中有这个图标
}
```

不需要修改任何组件代码，因为 `DashboardNav` 会自动遍历 `sidebarNav` 数组并渲染所有项。

</details>

### 问题 5：暗色模式切换的完整数据流是怎样的？从用户点击到页面变暗，经历了哪些步骤？

<details>
<summary>点击查看答案</summary>

1. 用户点击 ModeToggle 下拉菜单中的 "Dark"
2. `onClick` 触发 `setTheme("dark")`
3. `next-themes` 库接收到新主题值，把 `<html>` 标签的 class 改为 `"dark"`
4. 同时把主题偏好存入 `localStorage`，下次刷新还能记住
5. Tailwind CSS 检测到 `<html>` 上的 `dark` class，所有带 `dark:` 前缀的样式生效
6. 比如 `dark:bg-background` 的值从白色变成深色，页面整体变暗
7. ThemeProvider 的 Context 值更新，所有使用 `useTheme()` 的组件拿到最新主题

</details>

---

## 预计时间分配

| 环节 | 时间 | 说明 |
|------|------|------|
| 前置知识速补 | 30 分钟 | useState、useEffect、children、Context、事件处理 |
| 源码精读：根布局 + 营销布局 | 40 分钟 | 重点理解 children 和嵌套机制 |
| 源码精读：仪表盘布局 | 25 分钟 | 对比营销布局的不同之处 |
| 源码精读：导航组件 + 配置 | 30 分钟 | 理解响应式设计和配置驱动模式 |
| 源码精读：暗色模式 | 20 分钟 | ThemeProvider + ModeToggle |
| 动手练习（3 个） | 60 分钟 | 边做边理解 |
| 自检问题 + 笔记 | 15 分钟 | 回顾巩固 |
| **合计** | **约 3.5 小时** | |

建议中间休息 1-2 次，每次 10 分钟。如果前置知识部分已经熟悉，可以加速通过。

---

## 常见踩坑点

### 踩坑 1：在 Server Component 里用了 `useState`

```
错误信息：You're importing a component that needs useState. It only works in a Client Component...
```

**原因：** Next.js 里组件默认是 Server Component，不能用 hooks。
**解决：** 在文件最顶部加上 `"use client"` 声明。
**最佳实践：** 不要把整个布局都变成 Client Component。把需要 hooks 的部分提取成单独的小组件（就像 `MainNav` 被单独拎出来一样），布局本身保持为 Server Component。

### 踩坑 2：响应式断点理解反了

```
// 错误理解：md:hidden = "中等屏幕上隐藏"
// 正确理解：md:hidden = "中等屏幕及以上隐藏"
```

Tailwind 是 **mobile-first** 的，断点前缀表示的是"从这个宽度开始生效"。所以：
- `hidden` = 默认隐藏（所有屏幕）
- `md:flex` = 768px 以上变成 flex 显示
- 合起来 `hidden md:flex` = "手机上隐藏，电脑上显示"

### 踩坑 3：暗色模式闪烁

**现象：** 刷新页面时，先看到亮色再闪成暗色。
**原因：** 服务端渲染时不知道用户的主题偏好，默认用亮色渲染。客户端加载后才读取 `localStorage` 切换成暗色。
**解决：** `next-themes` 通过在 `<html>` 上注入一段 inline script（在 React 渲染之前执行），提前设置好 class，从而避免闪烁。这就是为什么根布局里需要 `suppressHydrationWarning`。

### 踩坑 4：配置文件的 TypeScript 类型报错

```
Type '{ title: string; href: string; foo: string; }' is not assignable to type 'MainNavItem'.
```

**原因：** 配置对象被 TypeScript 类型（如 `MarketingConfig`、`DashboardConfig`）约束了。如果你添加了不存在的字段或者缺少必填字段，TS 会报错。
**解决：** 查看 `types/index.d.ts` 中的类型定义，确保你的数据结构符合要求。这其实是好事——类型系统帮你发现了错误。

### 踩坑 5：`children` 是 `undefined` 导致空白页

**现象：** 布局渲染了导航栏和页脚，但中间是空白的。
**原因：** 布局的 `children` prop 没有被正确传递，或者对应的 `page.tsx` 文件不存在。
**解决：** 确认 `{children}` 被正确放在了 JSX 中，并且对应路径下有 `page.tsx` 文件。

---

## 延伸阅读

- [Next.js 官方文档 — Layouts and Templates](https://nextjs.org/docs/app/building-your-application/routing/layouts-and-templates) — 布局系统的权威指南
- [React 官方文档 — useState](https://react.dev/reference/react/useState) — 状态管理的基础
- [React 官方文档 — useEffect](https://react.dev/reference/react/useEffect) — 副作用处理
- [React 官方文档 — useContext](https://react.dev/reference/react/useContext) — Context 的使用方法
- [Tailwind CSS — Responsive Design](https://tailwindcss.com/docs/responsive-design) — 断点前缀的完整参考
- [Tailwind CSS — Dark Mode](https://tailwindcss.com/docs/dark-mode) — 暗色模式的配置
- [next-themes GitHub](https://github.com/pacocoursey/next-themes) — 主题切换库的文档和配置选项
- [Taxonomy 项目源码](https://github.com/shadcn/taxonomy) — 本教程基于的开源项目
