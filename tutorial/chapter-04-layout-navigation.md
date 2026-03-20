# 第4章：布局与导航 — 页面的骨架是怎么搭的？

## 学习目标

- 理解嵌套布局（Nested Layouts）的概念
- 理解导航栏、侧边栏、页脚的组件组织
- 理解配置驱动的导航（config 对象控制菜单项）
- 理解响应式设计（桌面导航 vs 移动端导航）
- 理解暗色模式切换

## 涉及文件

| 文件 | 说明 |
|------|------|
| `app/layout.tsx` | 根布局 |
| `app/(marketing)/layout.tsx` | 营销页布局 |
| `app/(dashboard)/dashboard/layout.tsx` | 仪表盘布局 |
| `components/main-nav.tsx` | 主导航栏 |
| `components/mobile-nav.tsx` | 移动端导航 |
| `components/sidebar-nav.tsx` | 侧边栏导航 |
| `components/site-footer.tsx` | 页脚 |
| `components/mode-toggle.tsx` | 暗色模式切换 |
| `components/theme-provider.tsx` | 主题提供者 |
| `config/site.ts` | 站点配置 |
| `config/marketing.ts` | 营销导航配置 |
| `config/dashboard.ts` | 仪表盘导航配置 |

---

## 1. 嵌套布局 — 俄罗斯套娃

Next.js 的布局是**嵌套的**，就像俄罗斯套娃。当你访问 `/dashboard` 时，实际的渲染结构是：

```
根布局 (app/layout.tsx)
└── 仪表盘布局 (app/(dashboard)/dashboard/layout.tsx)
    └── 仪表盘页面 (app/(dashboard)/dashboard/page.tsx)
```

当你访问 `/pricing` 时：

```
根布局 (app/layout.tsx)
└── 营销布局 (app/(marketing)/layout.tsx)
    └── 定价页面 (app/(marketing)/pricing/page.tsx)
```

**关键特性：** 切换页面时，如果共享同一个布局，布局部分不会重新渲染。比如在仪表盘内切换 `/dashboard` → `/dashboard/billing`，侧边栏和导航栏保持不动，只有内容区域变化。

---

## 2. 营销页布局 — 顶部导航 + 页脚

```tsx
// app/(marketing)/layout.tsx
import Link from "next/link"

import { marketingConfig } from "@/config/marketing"
import { cn } from "@/lib/utils"
import { buttonVariants } from "@/components/ui/button"
import { MainNav } from "@/components/main-nav"
import { SiteFooter } from "@/components/site-footer"

export default async function MarketingLayout({
  children,
}: {
  children: React.ReactNode
}) {
  return (
    // flex + min-h-screen + flex-col：页面至少占满屏幕高度
    <div className="flex min-h-screen flex-col">

      {/* 顶部导航栏 */}
      <header className="container z-40 bg-background">
        <div className="flex h-20 items-center justify-between py-6">
          {/* 左侧：Logo + 菜单项 */}
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

      {/* 主内容区（flex-1 让它占据所有剩余空间） */}
      <main className="flex-1">{children}</main>

      {/* 页脚 */}
      <SiteFooter />
    </div>
  )
}
```

布局结构示意：

```
┌─────────────────────────────┐
│  Logo   Features Pricing  Login  │  ← header
├─────────────────────────────┤
│                                 │
│        {children}               │  ← main (flex-1)
│    （页面内容在这里渲染）         │
│                                 │
├─────────────────────────────┤
│     Built by shadcn  🌙/☀️     │  ← footer
└─────────────────────────────┘
```

---

## 3. 仪表盘布局 — 侧边栏 + 用户头像

```tsx
// app/(dashboard)/dashboard/layout.tsx
import { dashboardConfig } from "@/config/dashboard"
import { getCurrentUser } from "@/lib/session"
import { MainNav } from "@/components/main-nav"
import { DashboardNav } from "@/components/nav"
import { SiteFooter } from "@/components/site-footer"
import { UserAccountNav } from "@/components/user-account-nav"

export default async function DashboardLayout({
  children,
}: {
  children?: React.ReactNode
}) {
  // 在布局中获取用户信息（Server Component 可以做到）
  const user = await getCurrentUser()

  if (!user) {
    return notFound()  // 未登录：显示 404
  }

  return (
    <div className="flex min-h-screen flex-col space-y-6">

      {/* 顶部导航栏（粘性定位，滚动时固定在顶部） */}
      <header className="sticky top-0 z-40 border-b bg-background">
        <div className="container flex h-16 items-center justify-between py-4">
          {/* 左侧：Logo + 文档/支持链接 */}
          <MainNav items={dashboardConfig.mainNav} />
          {/* 右侧：用户头像和下拉菜单 */}
          <UserAccountNav
            user={{
              name: user.name,
              image: user.image,
              email: user.email,
            }}
          />
        </div>
      </header>

      {/* 内容区：左侧边栏 + 右内容 */}
      <div className="container grid flex-1 gap-12 md:grid-cols-[200px_1fr]">
        {/* 侧边栏（md 以上屏幕才显示） */}
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

布局结构示意：

```
┌──────────────────────────────┐
│  Logo   Docs Support    👤 User  │  ← sticky header
├──────────┬───────────────────┤
│ Posts    │                       │
│ Billing  │     {children}        │
│ Settings │  （页面内容在这里）     │
│          │                       │
├──────────┴───────────────────┤
│         Footer                   │
└──────────────────────────────┘
```

**关键技巧：** `md:grid-cols-[200px_1fr]`
- 这是 Tailwind 的响应式写法
- `md:` 前缀意味着"在中等及以上屏幕（≥768px）才生效"
- `grid-cols-[200px_1fr]` 意思是两列：左列 200px 宽，右列占满剩余空间
- 手机上侧边栏隐藏（`hidden md:flex`），内容全屏显示

---

## 4. 配置驱动的导航

导航菜单项不是写死在组件里的，而是由配置文件定义：

```typescript
// config/marketing.ts — 营销页的导航项
export const marketingConfig: MarketingConfig = {
  mainNav: [
    { title: "Features", href: "/#features" },
    { title: "Pricing",  href: "/pricing" },
    { title: "Blog",     href: "/blog" },
    { title: "Documentation", href: "/docs" },
  ],
}
```

```typescript
// config/dashboard.ts — 仪表盘的导航项
export const dashboardConfig: DashboardConfig = {
  mainNav: [
    { title: "Documentation", href: "/docs" },
    { title: "Support", href: "/support", disabled: true },  // 禁用状态
  ],
  sidebarNav: [
    { title: "Posts",    href: "/dashboard",          icon: "post" },
    { title: "Billing",  href: "/dashboard/billing",  icon: "billing" },
    { title: "Settings", href: "/dashboard/settings", icon: "settings" },
  ],
}
```

**为什么用配置驱动？**
- 添加/删除菜单项只需修改配置文件，不用改组件代码
- 配置文件可以根据环境不同（开发/生产）动态调整
- 所有导航项集中管理，一目了然

---

## 5. MainNav 组件 — 响应式导航实战

```tsx
// components/main-nav.tsx
"use client"

import * as React from "react"
import Link from "next/link"
import { useSelectedLayoutSegment } from "next/navigation"

import { MainNavItem } from "types"
import { siteConfig } from "@/config/site"
import { cn } from "@/lib/utils"
import { Icons } from "@/components/icons"
import { MobileNav } from "@/components/mobile-nav"

export function MainNav({ items, children }: { items?: MainNavItem[]; children?: React.ReactNode }) {
  // 获取当前 URL 的第一段路径（如 /pricing → "pricing"）
  const segment = useSelectedLayoutSegment()

  // 移动端菜单的开关状态
  const [showMobileMenu, setShowMobileMenu] = React.useState<boolean>(false)

  return (
    <div className="flex gap-6 md:gap-10">
      {/* Logo — 只在桌面端显示（md:flex） */}
      <Link href="/" className="hidden items-center space-x-2 md:flex">
        <Icons.logo />
        <span className="hidden font-bold sm:inline-block">
          {siteConfig.name}
        </span>
      </Link>

      {/* 桌面端导航菜单 — 只在桌面端显示 */}
      {items?.length ? (
        <nav className="hidden gap-6 md:flex">
          {items?.map((item, index) => (
            <Link
              key={index}
              href={item.disabled ? "#" : item.href}
              className={cn(
                "flex items-center text-lg font-medium transition-colors hover:text-foreground/80 sm:text-sm",
                // 当前页面的链接高亮显示
                item.href.startsWith(`/${segment}`)
                  ? "text-foreground"        // 当前页：全黑/全白
                  : "text-foreground/60",    // 其他页：60% 透明度
                item.disabled && "cursor-not-allowed opacity-80"
              )}
            >
              {item.title}
            </Link>
          ))}
        </nav>
      ) : null}

      {/* 移动端汉堡菜单按钮 — 只在移动端显示（md:hidden） */}
      <button
        className="flex items-center space-x-2 md:hidden"
        onClick={() => setShowMobileMenu(!showMobileMenu)}
      >
        {showMobileMenu ? <Icons.close /> : <Icons.logo />}
        <span className="font-bold">Menu</span>
      </button>

      {/* 移动端展开的菜单 */}
      {showMobileMenu && items && (
        <MobileNav items={items}>{children}</MobileNav>
      )}
    </div>
  )
}
```

### 响应式设计的核心技巧

```tsx
// 桌面端显示，移动端隐藏
<div className="hidden md:flex">桌面内容</div>

// 移动端显示，桌面端隐藏
<div className="flex md:hidden">移动端内容</div>
```

Tailwind 的断点前缀：
- `sm:` → ≥640px
- `md:` → ≥768px
- `lg:` → ≥1024px
- `xl:` → ≥1280px

没有前缀 = 默认（移动端优先）。

### `useSelectedLayoutSegment()` — 高亮当前菜单

这个 Hook 返回当前 URL 的路径段，用于高亮当前页面对应的菜单项：

```
URL: /pricing      → segment = "pricing"
URL: /blog/hello   → segment = "blog"
URL: /dashboard     → segment = "dashboard"
```

---

## 6. 暗色模式 — ThemeProvider + ModeToggle

### 第1步：ThemeProvider 包裹整个应用

```tsx
// components/theme-provider.tsx
"use client"

import { ThemeProvider as NextThemesProvider } from "next-themes"

export function ThemeProvider({ children, ...props }) {
  return <NextThemesProvider {...props}>{children}</NextThemesProvider>
}
```

在根布局中使用：

```tsx
// app/layout.tsx
<ThemeProvider attribute="class" defaultTheme="system" enableSystem>
  {children}
</ThemeProvider>
```

`attribute="class"` 意味着暗色模式通过在 `<html>` 标签上添加 `class="dark"` 来实现。Tailwind 会根据这个类名切换样式：

```css
/* 亮色模式 */
.bg-background { background: white; }

/* 暗色模式 */
.dark .bg-background { background: #1a1a1a; }
```

### 第2步：ModeToggle 切换按钮

```tsx
// components/mode-toggle.tsx
"use client"

import { useTheme } from "next-themes"
import { Button } from "@/components/ui/button"
import { DropdownMenu, DropdownMenuContent, DropdownMenuItem, DropdownMenuTrigger } from "@/components/ui/dropdown-menu"
import { Icons } from "@/components/icons"

export function ModeToggle() {
  const { setTheme } = useTheme()  // 获取设置主题的函数

  return (
    <DropdownMenu>
      <DropdownMenuTrigger asChild>
        <Button variant="ghost" size="sm" className="h-8 w-8 px-0">
          {/* 太阳图标：亮色模式时显示，暗色模式时旋转消失 */}
          <Icons.sun className="rotate-0 scale-100 transition-all dark:-rotate-90 dark:scale-0" />
          {/* 月亮图标：亮色模式时隐藏，暗色模式时显示 */}
          <Icons.moon className="absolute rotate-90 scale-0 transition-all dark:rotate-0 dark:scale-100" />
          <span className="sr-only">Toggle theme</span>
        </Button>
      </DropdownMenuTrigger>
      <DropdownMenuContent align="end">
        <DropdownMenuItem onClick={() => setTheme("light")}>Light</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("dark")}>Dark</DropdownMenuItem>
        <DropdownMenuItem onClick={() => setTheme("system")}>System</DropdownMenuItem>
      </DropdownMenuContent>
    </DropdownMenu>
  )
}
```

**动画效果解析：**
```
亮色模式时：
  太阳 → rotate-0 scale-100（正常显示）
  月亮 → rotate-90 scale-0（旋转 90° 并缩小到 0）

暗色模式时（dark: 前缀生效）：
  太阳 → dark:-rotate-90 dark:scale-0（反转消失）
  月亮 → dark:rotate-0 dark:scale-100（正常显示）
```

两个图标叠加在同一位置（`absolute`），通过 CSS 动画平滑切换。

---

## 7. 站点配置 — 集中管理站点信息

```typescript
// config/site.ts
export const siteConfig: SiteConfig = {
  name: "Taxonomy",
  description:
    "An open source application built using the new router, server components and everything new in Next.js 13.",
  url: "https://tx.shadcn.com",
  ogImage: "https://tx.shadcn.com/og.jpg",
  links: {
    twitter: "https://twitter.com/shadcn",
    github: "https://github.com/shadcn/taxonomy",
  },
}
```

这个配置在整个项目中被多处引用：
- 根布局中的 `metadata`（SEO 标题、描述）
- 导航栏中的 Logo 文字
- 页脚中的链接
- 社交媒体分享卡片

**好处：** 修改站点名称只需改一个地方，所有地方自动更新。

---

## 8. TailwindIndicator — 开发者的好帮手

```tsx
// components/tailwind-indicator.tsx
export function TailwindIndicator() {
  // 生产环境不显示
  if (process.env.NODE_ENV === "production") return null

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

这个小组件在开发环境下会在屏幕左下角显示当前的断点名称（xs/sm/md/lg/xl/2xl），帮你调试响应式布局。

---

## 小练习

### 练习 1：添加一个导航项
1. 在 `config/marketing.ts` 的 `mainNav` 数组中添加一个新项：`{ title: "About", href: "/about" }`
2. 刷新页面，看看导航栏是否出现了新的链接
3. 点击它——会显示 404 因为还没创建对应的页面

### 练习 2：修改仪表盘侧边栏
1. 在 `config/dashboard.ts` 的 `sidebarNav` 中添加一项
2. 设置 `icon` 为 `"help"`（这个图标在 icons.tsx 中已定义）
3. 观察侧边栏的变化

### 练习 3：理解布局嵌套
1. 在 `app/(marketing)/layout.tsx` 中添加一行 `console.log("marketing layout rendered")`
2. 在 `app/layout.tsx` 中添加 `console.log("root layout rendered")`
3. 访问不同页面，在终端观察哪些布局被渲染了

### 练习 4：测试暗色模式
1. 在浏览器中切换暗色/亮色模式
2. 打开开发者工具，观察 `<html>` 标签上 class 属性的变化
3. 理解 Tailwind 的 `dark:` 前缀是如何工作的

---

## 本章小结

- 布局嵌套让不同页面组共享不同的"外壳"，切换页面时布局不重新渲染
- 营销页用顶部导航 + 页脚，仪表盘用顶部导航 + 侧边栏
- 导航项由配置文件驱动，修改菜单只需改配置
- 响应式设计靠 Tailwind 的断点前缀（`md:`, `lg:` 等）
- 暗色模式靠 `next-themes` 库 + Tailwind 的 `dark:` 前缀

下一章我们将探索 Markdown 内容系统。
