# 第7章：认证系统 — 用户怎么登录的？

## 学习目标

- 理解 OAuth 认证流程（以 GitHub 登录为例）
- 理解 JWT 和 Session 的区别
- 理解 NextAuth.js 的配置和工作原理
- 理解中间件如何保护路由
- 理解 NextAuth 的 Prisma Adapter

## 涉及文件

| 文件 | 说明 |
|------|------|
| `lib/auth.ts` | NextAuth 配置 |
| `lib/session.ts` | 获取当前用户 |
| `middleware.ts` | 路由保护中间件 |
| `app/(auth)/login/page.tsx` | 登录页面 |
| `components/user-auth-form.tsx` | 登录表单组件 |
| `components/user-account-nav.tsx` | 用户账户菜单 |
| `types/next-auth.d.ts` | NextAuth 类型扩展 |

---

## 1. 认证基础概念

### 什么是认证？

**认证（Authentication）** = 确认"你是谁"。就像进入公司大楼需要刷工牌——工牌证明你是这家公司的员工。

### OAuth 是什么？

**OAuth** 是一种让第三方（如 GitHub）帮你验证身份的协议。流程如下：

```
用户点击 "Sign in with GitHub"
    ↓
跳转到 GitHub 登录页
    ↓
用户在 GitHub 输入账号密码
    ↓
GitHub 验证成功，问用户："Taxonomy 想访问你的信息，允许吗？"
    ↓
用户点击"允许"
    ↓
GitHub 把用户信息（名字、邮箱、头像）发给 Taxonomy
    ↓
Taxonomy 创建或更新用户记录，登录成功
```

**好处：** 你的应用不需要存用户的密码（GitHub 帮你管了）。

### JWT vs Session

| | JWT | Session |
|---|-----|---------|
| 存储位置 | 客户端（Cookie/localStorage） | 服务端（数据库/内存） |
| 内容 | 加密的用户信息 | 一个 Session ID |
| 验证方式 | 解密令牌读取信息 | 用 Session ID 查数据库 |
| 优点 | 不需要查数据库 | 可以随时撤销 |

Taxonomy 使用 **JWT 策略**（在 `lib/auth.ts` 中配置 `strategy: "jwt"`）。

---

## 2. NextAuth 配置

`lib/auth.ts` 是整个认证系统的核心配置：

```typescript
// lib/auth.ts
import { PrismaAdapter } from "@next-auth/prisma-adapter"
import { NextAuthOptions } from "next-auth"
import EmailProvider from "next-auth/providers/email"
import GitHubProvider from "next-auth/providers/github"
import { Client } from "postmark"

import { env } from "@/env.mjs"
import { db } from "@/lib/db"

// Postmark 邮件客户端（用于发送登录验证邮件）
const postmarkClient = new Client(env.POSTMARK_API_TOKEN)

export const authOptions: NextAuthOptions = {
  // ========== 适配器 ==========
  // PrismaAdapter 让 NextAuth 把用户数据存到 Prisma 数据库中
  // 也就是用 schema.prisma 中定义的 User、Account、Session 表
  adapter: PrismaAdapter(db as any),

  // ========== 会话策略 ==========
  session: {
    strategy: "jwt",  // 使用 JWT（而不是数据库 Session）
  },

  // ========== 自定义页面 ==========
  pages: {
    signIn: "/login",  // 登录页不用 NextAuth 默认的，用我们自己的
  },

  // ========== 登录方式 ==========
  providers: [
    // 方式1：GitHub OAuth 登录
    GitHubProvider({
      clientId: env.GITHUB_CLIENT_ID,
      clientSecret: env.GITHUB_CLIENT_SECRET,
    }),

    // 方式2：邮箱魔法链接登录
    EmailProvider({
      from: env.SMTP_FROM,
      sendVerificationRequest: async ({ identifier, url, provider }) => {
        // 查看用户是否已注册
        const user = await db.user.findUnique({
          where: { email: identifier },
          select: { emailVerified: true },
        })

        // 根据是否首次登录选择不同的邮件模板
        const templateId = user?.emailVerified
          ? env.POSTMARK_SIGN_IN_TEMPLATE      // 已注册：发登录链接
          : env.POSTMARK_ACTIVATION_TEMPLATE   // 未注册：发激活链接

        // 通过 Postmark 发送邮件
        await postmarkClient.sendEmailWithTemplate({
          TemplateId: parseInt(templateId),
          To: identifier,
          From: provider.from as string,
          TemplateModel: {
            action_url: url,            // 登录链接
            product_name: siteConfig.name,  // "Taxonomy"
          },
        })
      },
    }),
  ],

  // ========== 回调函数 ==========
  callbacks: {
    // 当创建 session 时调用 — 决定 session 中包含哪些信息
    async session({ token, session }) {
      if (token) {
        session.user.id = token.id        // 把用户 ID 放进 session
        session.user.name = token.name
        session.user.email = token.email
        session.user.image = token.picture
      }
      return session
    },

    // 当创建/更新 JWT 时调用 — 决定 JWT 中包含哪些信息
    async jwt({ token, user }) {
      // 用 token 中的邮箱查找数据库用户
      const dbUser = await db.user.findFirst({
        where: { email: token.email },
      })

      if (!dbUser) {
        // 首次登录，用户还没在数据库中
        if (user) {
          token.id = user?.id
        }
        return token
      }

      // 返回包含用户信息的 token
      return {
        id: dbUser.id,
        name: dbUser.name,
        email: dbUser.email,
        picture: dbUser.image,
      }
    },
  },
}
```

### 配置结构解读

```
authOptions
├── adapter: PrismaAdapter      → 用户数据存到哪（数据库）
├── session: { strategy: "jwt" } → 认证策略（JWT 令牌）
├── pages: { signIn: "/login" }  → 自定义登录页
├── providers: [                 → 支持的登录方式
│   ├── GitHubProvider           → GitHub OAuth
│   └── EmailProvider            → 邮箱链接
│]
└── callbacks: {                 → 回调函数
    ├── session()                → 自定义 session 内容
    └── jwt()                    → 自定义 JWT 内容
}
```

---

## 3. 获取当前用户

```typescript
// lib/session.ts
import { getServerSession } from "next-auth/next"
import { authOptions } from "@/lib/auth"

export async function getCurrentUser() {
  const session = await getServerSession(authOptions)
  return session?.user
}
```

只有 3 行代码！它封装了 NextAuth 的 `getServerSession` 函数，方便在任何 Server Component 或 API Route 中使用：

```tsx
// 在页面中使用
const user = await getCurrentUser()
if (!user) redirect("/login")

// user 包含：
// {
//   id: "clk...",
//   name: "John",
//   email: "john@example.com",
//   image: "https://avatars.githubusercontent.com/..."
// }
```

---

## 4. 登录表单组件

```tsx
// components/user-auth-form.tsx
"use client"

import { signIn } from "next-auth/react"  // NextAuth 客户端函数
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { userAuthSchema } from "@/lib/validations/auth"

export function UserAuthForm({ className, ...props }) {
  const {
    register,        // 注册表单字段
    handleSubmit,    // 处理表单提交
    formState: { errors },  // 验证错误
  } = useForm<FormData>({
    resolver: zodResolver(userAuthSchema),  // 用 Zod Schema 验证
  })

  const [isLoading, setIsLoading] = React.useState(false)
  const [isGitHubLoading, setIsGitHubLoading] = React.useState(false)

  // 邮箱登录
  async function onSubmit(data: FormData) {
    setIsLoading(true)

    const signInResult = await signIn("email", {
      email: data.email.toLowerCase(),
      redirect: false,                              // 不自动跳转
      callbackUrl: searchParams?.get("from") || "/dashboard",  // 登录后跳转
    })

    setIsLoading(false)

    if (!signInResult?.ok) {
      return toast({ title: "Something went wrong.", variant: "destructive" })
    }

    return toast({
      title: "Check your email",
      description: "We sent you a login link.",
    })
  }

  return (
    <div className={cn("grid gap-6", className)} {...props}>
      {/* 邮箱登录表单 */}
      <form onSubmit={handleSubmit(onSubmit)}>
        <div className="grid gap-2">
          <Input
            id="email"
            placeholder="name@example.com"
            type="email"
            disabled={isLoading || isGitHubLoading}
            {...register("email")}  // 注册到 react-hook-form
          />
          {errors?.email && (
            <p className="text-xs text-red-600">{errors.email.message}</p>
          )}
          <button className={cn(buttonVariants())} disabled={isLoading}>
            {isLoading && <Icons.spinner className="mr-2 h-4 w-4 animate-spin" />}
            Sign In with Email
          </button>
        </div>
      </form>

      {/* 分隔线 */}
      <div className="relative">
        <span className="bg-background px-2 text-muted-foreground">
          Or continue with
        </span>
      </div>

      {/* GitHub 登录按钮 */}
      <button
        className={cn(buttonVariants({ variant: "outline" }))}
        onClick={() => {
          setIsGitHubLoading(true)
          signIn("github")  // 一行代码搞定 GitHub 登录！
        }}
        disabled={isLoading || isGitHubLoading}
      >
        {isGitHubLoading ? (
          <Icons.spinner className="mr-2 h-4 w-4 animate-spin" />
        ) : (
          <Icons.gitHub className="mr-2 h-4 w-4" />
        )}
        Github
      </button>
    </div>
  )
}
```

**注意 `signIn("github")` 这行——NextAuth 把复杂的 OAuth 流程封装成了一个函数调用。**

---

## 5. 中间件 — 路由保护

`middleware.ts` 像一个"保安"，在用户访问受保护的页面之前检查身份：

```typescript
// middleware.ts
import { getToken } from "next-auth/jwt"
import { withAuth } from "next-auth/middleware"
import { NextResponse } from "next/server"

export default withAuth(
  async function middleware(req) {
    // 检查用户是否有有效的 JWT 令牌
    const token = await getToken({ req })
    const isAuth = !!token

    // 判断当前是否在认证页面（登录/注册）
    const isAuthPage =
      req.nextUrl.pathname.startsWith("/login") ||
      req.nextUrl.pathname.startsWith("/register")

    if (isAuthPage) {
      if (isAuth) {
        // 已登录的用户访问登录页 → 跳转到仪表盘
        return NextResponse.redirect(new URL("/dashboard", req.url))
      }
      // 未登录的用户访问登录页 → 正常显示
      return null
    }

    if (!isAuth) {
      // 未登录的用户访问受保护页面 → 跳转到登录页
      let from = req.nextUrl.pathname
      if (req.nextUrl.search) {
        from += req.nextUrl.search
      }
      return NextResponse.redirect(
        new URL(`/login?from=${encodeURIComponent(from)}`, req.url)
      )
    }
  },
  {
    callbacks: {
      async authorized() {
        // 始终返回 true，让上面的 middleware 函数来处理
        return true
      },
    },
  }
)

// 配置：哪些路径需要经过中间件
export const config = {
  matcher: ["/dashboard/:path*", "/editor/:path*", "/login", "/register"],
}
```

### 中间件的工作流程

```
用户访问 /dashboard
    ↓
中间件检查 JWT
    ↓
├── 有 JWT → 允许访问（显示仪表盘）
└── 无 JWT → 跳转到 /login?from=/dashboard
                ↓
            用户登录成功
                ↓
            跳回 /dashboard（使用 from 参数）
```

### `matcher` 配置

```typescript
export const config = {
  matcher: ["/dashboard/:path*", "/editor/:path*", "/login", "/register"],
}
```

这告诉 Next.js：只有这些路径需要经过中间件。其他路径（如首页 `/`、定价页 `/pricing`）不需要检查登录状态。

- `/dashboard/:path*` — 匹配 `/dashboard` 及其所有子路径
- `/editor/:path*` — 匹配 `/editor` 及其所有子路径

---

## 6. 认证流程全景图

```
┌─────────────────────────────────────────┐
│                用户访问                    │
└─────────┬───────────────────────────────┘
          ↓
┌─────────────────────────────────────────┐
│        middleware.ts（路由保安）            │
│  检查 JWT → 有则放行，无则跳转登录          │
└─────────┬───────────────────────────────┘
          ↓
┌─────────────────────────────────────────┐
│    components/user-auth-form.tsx          │
│  显示登录表单（邮箱 or GitHub）            │
└──┬──────────────────────────┬───────────┘
   │                          │
   ↓                          ↓
 signIn("email")        signIn("github")
   │                          │
   ↓                          ↓
 发送魔法链接邮件        跳转 GitHub 授权
   │                          │
   ↓                          ↓
┌─────────────────────────────────────────┐
│         lib/auth.ts（NextAuth 配置）       │
│  callbacks.jwt → 生成/更新 JWT            │
│  callbacks.session → 构建 session 对象     │
│  adapter → 存储用户信息到数据库             │
└─────────┬───────────────────────────────┘
          ↓
┌─────────────────────────────────────────┐
│        lib/session.ts                     │
│  getCurrentUser() → 在页面中获取用户信息    │
└─────────────────────────────────────────┘
```

---

## 7. PrismaAdapter 的作用

`PrismaAdapter(db)` 让 NextAuth 自动使用 Prisma 来管理用户数据。当用户首次通过 GitHub 登录时：

1. NextAuth 收到 GitHub 返回的用户信息
2. PrismaAdapter 在 `users` 表中创建一条新记录
3. 在 `accounts` 表中创建一条记录，关联 GitHub 账号和用户
4. 下次用同一个 GitHub 账号登录时，PrismaAdapter 查找已有记录，直接登录

**你不需要自己写任何用户创建/查找的代码。**

---

## 小练习

### 练习 1：追踪登录流程
1. 在 `lib/auth.ts` 的 `jwt` 回调中添加 `console.log("JWT callback:", token)`
2. 在 `session` 回调中添加 `console.log("Session callback:", session)`
3. 尝试登录，在终端观察输出的顺序

### 练习 2：理解中间件
1. 打开浏览器（确保未登录）
2. 直接访问 `/dashboard`
3. 观察 URL 变化——你被重定向到了哪里？URL 中的 `from` 参数是什么？

### 练习 3：阅读 schema
对照 `prisma/schema.prisma` 中的 Account 模型，理解 OAuth 登录需要存储哪些信息。

### 练习 4：思考安全性
为什么中间件把已登录用户从 `/login` 重定向到 `/dashboard`？如果不这么做，可能会有什么问题？

---

## 本章小结

- OAuth 让用户通过第三方（如 GitHub）登录，你的应用不需要存密码
- NextAuth.js 把复杂的认证逻辑封装成简单的配置和 API
- `PrismaAdapter` 自动将用户数据存入数据库
- JWT 策略在令牌中存储用户信息，避免每次查数据库
- 中间件保护受限路由，未登录用户会被重定向
- `getCurrentUser()` 是获取当前用户的便捷函数

下一章我们将学习 API 层——前后端是怎么对话的。
