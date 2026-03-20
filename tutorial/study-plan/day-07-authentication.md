# Day 07 -- 认证系统：用户怎么登录的？

> 对应章节：[Chapter 7: 认证系统](../chapter-07-authentication.md)

---

## 一、今日学习目标

1. **理解 OAuth 认证的完整流程**：从用户点击"Sign in with GitHub"到成功登录，中间到底发生了什么
2. **看懂 NextAuth.js 的配置和回调机制**：知道 `lib/auth.ts` 里每一行在做什么
3. **理解中间件如何保护路由**：未登录用户为什么会被自动踢到登录页

---

## 二、前置知识速补

在看认证代码之前，有几个基础概念必须先搞清楚。别急，每个都会用最简单的话讲明白。

### 2.1 [Cookie](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies) 是什么

**类比：手环 / 通行证**

你去游乐园玩，买票后工作人员给你一个手环。之后你去坐过山车、摩天轮，不需要每次都重新买票，出示手环就行。

Cookie 就是浏览器版的"手环"：

```
1. 你登录网站（相当于买票）
2. 服务器给你一个 Cookie（相当于手环）
3. 之后每次你访问这个网站，浏览器都会自动带上 Cookie（出示手环）
4. 服务器看到 Cookie，就知道"哦，这个人已经登录了"
```

技术上来说，Cookie 是一小段文本，保存在你的浏览器里。服务器可以通过 HTTP 响应头 `Set-Cookie` 把它种到浏览器中。

### 2.2 HTTP Headers 基础

HTTP 请求和响应都有 **[headers](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers)**（头部信息），可以类比为信封上的信息：

```
信封正面（请求 headers）：
  收件人：www.example.com
  寄件人信息：我用的 Chrome 浏览器
  我带的通行证：Cookie: session=abc123

信封背面（响应 headers）：
  内容类型：这是一个 HTML 页面
  给你的通行证：Set-Cookie: session=abc123
```

在认证系统中，[JWT](https://jwt.io/introduction)（后面会讲）就是通过 Cookie header 在浏览器和服务器之间传递的。

### 2.3 回调函数

回调函数就是"你先帮我做某件事，做完了调用我给你的这个函数"。

```javascript
// 这就是一个回调函数的使用
// "做完煎蛋这件事之后，执行 function 里面的代码"
煎蛋(function (err, result) {
  if (err) {
    console.log("煎糊了！")
  } else {
    console.log("煎好了！", result)
  }
})
```

在 NextAuth 中，`callbacks` 就是这种思路：

```typescript
callbacks: {
  // NextAuth 在创建 session 时，会"回调"这个函数
  // 意思是："session 准备好了，你想往里面加什么？"
  async session({ token, session }) {
    // 我们往 session 里加入用户 ID
    session.user.id = token.id
    return session
  },
}
```

### 2.4 [闭包](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Closures)（函数记住外部变量）

闭包是 JavaScript 中一个重要概念：**函数可以"记住"它被创建时所在作用域的变量**。

```javascript
function createGreeter(name) {
  // 内部函数可以使用外部的 name 变量
  return function () {
    console.log("Hello, " + name)
  }
}

const greetAlice = createGreeter("Alice")
greetAlice() // "Hello, Alice" — 虽然 createGreeter 已经执行完了，但 name 被记住了
```

在 Taxonomy 的认证代码中，闭包出现在 `sendVerificationRequest` 里：

```typescript
// env、db、siteConfig 都是外部变量
// 但 sendVerificationRequest 这个函数可以"记住"并使用它们
sendVerificationRequest: async ({ identifier, url, provider }) => {
  const user = await db.user.findUnique({ ... })        // db 来自外部
  const templateId = user?.emailVerified
    ? env.POSTMARK_SIGN_IN_TEMPLATE                      // env 来自外部
    : env.POSTMARK_ACTIVATION_TEMPLATE
  // ...
}
```

### 2.5 TypeScript 的[泛型](https://www.typescriptlang.org/docs/handbook/2/generics.html) `<T>` 和[类型断言](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-assertions) `as`

**泛型 `<T>`** -- 可以理解为"类型占位符"：

```typescript
// useState<boolean> 意思是：这个 state 的类型是 boolean
const [isLoading, setIsLoading] = React.useState<boolean>(false)

// useForm<FormData> 意思是：这个表单的数据类型是 FormData
const { register } = useForm<FormData>({ ... })
```

就像函数参数是"值的占位符"，泛型是"类型的占位符"。

**类型断言 `as`** -- 告诉 TypeScript"相信我，这个值就是这个类型"：

```typescript
// 告诉 TypeScript：provider.from 一定是 string，别报错
From: provider.from as string,

// 告诉 TypeScript：db 可以当 any 用（临时解决类型兼容问题）
adapter: PrismaAdapter(db as any),
```

---

## 三、核心概念清单

| 概念 | 英文 | 一句话解释 | 在哪里用到 |
|------|------|-----------|-----------|
| 认证 | Authentication | 确认"你是谁"，就像刷工牌进公司 | 整个认证系统的目标 |
| [OAuth](https://oauth.net/2/) | Open Authorization | 让第三方（如 GitHub）帮你验证身份的协议 | `GitHubProvider` |
| [NextAuth.js](https://next-auth.js.org/getting-started/introduction) | -- | Next.js 生态的认证框架，帮你处理登录的各种复杂逻辑 | `lib/auth.ts` |
| JWT | JSON Web Token | 加密的"通行证"，里面包含用户信息，存在 Cookie 里 | `strategy: "jwt"` |
| Session | 会话 | 一次登录周期，包含当前用户的信息 | `callbacks.session` |
| [Provider](https://next-auth.js.org/providers/) | 登录方式提供者 | 配置用什么方式登录（GitHub、邮箱等） | `providers: [...]` |
| [Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware) | 中间件 | 请求到达页面之前的"保安"，检查你有没有登录 | `middleware.ts` |
| [Adapter](https://authjs.dev/concepts/database-adapters) | 适配器 | 告诉 NextAuth 用什么方式存储用户数据（这里用 Prisma） | `PrismaAdapter(db)` |

---

## 四、源码精读

### 4.1 认证核心配置 -- `lib/auth.ts`

这是整个认证系统最重要的文件。我们逐块拆解。

**文件路径：** `lib/auth.ts`

```typescript
// === 第 1 部分：导入依赖 ===
import { PrismaAdapter } from "@next-auth/prisma-adapter"  // 数据库适配器
import { NextAuthOptions } from "next-auth"                 // NextAuth 的配置类型
import EmailProvider from "next-auth/providers/email"       // 邮箱登录方式
import GitHubProvider from "next-auth/providers/github"     // GitHub 登录方式
import { Client } from "postmark"                           // 发邮件的工具

import { env } from "@/env.mjs"           // 环境变量（API 密钥等）
import { siteConfig } from "@/config/site" // 网站名称等配置
import { db } from "@/lib/db"              // Prisma 数据库客户端
```

> 每一行 `import` 就是"引入一个工具"。就像做菜前先把食材和厨具摆出来。

```typescript
// === 第 2 部分：创建邮件客户端 ===
const postmarkClient = new Client(env.POSTMARK_API_TOKEN)
```

> Postmark 是一个发邮件的服务。`new Client(...)` 创建一个客户端，之后可以用它发邮件。

```typescript
// === 第 3 部分：NextAuth 配置对象 ===
export const authOptions: NextAuthOptions = {
  // 适配器：告诉 NextAuth 把用户数据存到 Prisma 数据库
  adapter: PrismaAdapter(db as any),

  // 会话策略：用 JWT（加密令牌），而不是数据库 Session
  session: {
    strategy: "jwt",
  },

  // 自定义页面：登录页用我们自己做的 /login，不用 NextAuth 默认的
  pages: {
    signIn: "/login",
  },
```

> 这三个配置回答了三个问题：用户数据存哪？用什么认证策略？登录页面长什么样？

```typescript
  // === 第 4 部分：登录方式 ===
  providers: [
    // 方式一：GitHub OAuth 登录
    GitHubProvider({
      clientId: env.GITHUB_CLIENT_ID,         // 从环境变量读取
      clientSecret: env.GITHUB_CLIENT_SECRET,  // 从环境变量读取
    }),

    // 方式二：邮箱魔法链接登录
    EmailProvider({
      from: env.SMTP_FROM,  // 发件人地址
      sendVerificationRequest: async ({ identifier, url, provider }) => {
        // identifier = 用户输入的邮箱地址
        // url = NextAuth 生成的登录链接
        // provider = 当前 Provider 的配置信息

        // 查数据库，看这个邮箱是否已经注册过
        const user = await db.user.findUnique({
          where: { email: identifier },
          select: { emailVerified: true },
        })

        // 如果已注册 → 发"登录"邮件模板
        // 如果未注册 → 发"激活"邮件模板
        const templateId = user?.emailVerified
          ? env.POSTMARK_SIGN_IN_TEMPLATE
          : env.POSTMARK_ACTIVATION_TEMPLATE

        // 通过 Postmark 发送邮件
        const result = await postmarkClient.sendEmailWithTemplate({
          TemplateId: parseInt(templateId),
          To: identifier,
          From: provider.from as string,
          TemplateModel: {
            action_url: url,               // 邮件中的登录链接
            product_name: siteConfig.name,  // "Taxonomy"
          },
          Headers: [
            {
              Name: "X-Entity-Ref-ID",
              Value: new Date().getTime() + "",  // 防止 Gmail 把邮件合并到同一个对话
            },
          ],
        })

        if (result.ErrorCode) {
          throw new Error(result.Message)  // 发送失败就抛错
        }
      },
    }),
  ],
```

> `providers` 是一个数组，里面放了所有支持的登录方式。你可以类比为"这家公司支持工牌和指纹两种方式进门"。

```typescript
  // === 第 5 部分：回调函数 ===
  callbacks: {
    // 回调一：构建 session 对象
    // 每次需要获取用户信息时都会调用
    async session({ token, session }) {
      if (token) {
        session.user.id = token.id         // 从 JWT token 拿到用户 ID
        session.user.name = token.name     // 用户名
        session.user.email = token.email   // 邮箱
        session.user.image = token.picture // 头像
      }
      return session
    },

    // 回调二：构建/更新 JWT token
    // 登录时和每次验证 session 时都会调用
    async jwt({ token, user }) {
      // 用 token 中的邮箱去数据库查找用户
      const dbUser = await db.user.findFirst({
        where: { email: token.email },
      })

      if (!dbUser) {
        // 数据库中找不到 → 可能是首次登录
        // user 参数只在首次登录时存在
        if (user) {
          token.id = user?.id
        }
        return token
      }

      // 找到了 → 返回数据库中的最新用户信息
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

> 两个回调的执行顺序：先 `jwt`（生成令牌），再 `session`（用令牌构建会话）。就像先做蛋糕（jwt），再装盘呈上（session）。

### 4.2 路由保护中间件 -- `middleware.ts`

中间件就像大楼门口的保安，每个人进门都要检查一下。

**文件路径：** `middleware.ts`

```typescript
import { getToken } from "next-auth/jwt"      // 从请求中提取 JWT
import { withAuth } from "next-auth/middleware" // NextAuth 的中间件包装器
import { NextResponse } from "next/server"      // Next.js 的响应工具

export default withAuth(
  async function middleware(req) {
    // 第一步：检查这个请求有没有有效的 JWT
    const token = await getToken({ req })
    const isAuth = !!token  // !! 把值转为布尔值：有 token → true，没有 → false

    // 第二步：判断用户当前访问的是不是登录/注册页
    const isAuthPage =
      req.nextUrl.pathname.startsWith("/login") ||
      req.nextUrl.pathname.startsWith("/register")

    // 第三步：根据情况做不同处理
    if (isAuthPage) {
      if (isAuth) {
        // 已登录的人访问登录页 → 不用再登录了，跳转到仪表盘
        return NextResponse.redirect(new URL("/dashboard", req.url))
      }
      return null  // 未登录的人访问登录页 → 正常显示登录页
    }

    if (!isAuth) {
      // 未登录的人访问受保护的页面 → 踢到登录页
      let from = req.nextUrl.pathname
      if (req.nextUrl.search) {
        from += req.nextUrl.search
      }
      // 把原始路径存到 from 参数里，登录后可以跳回来
      return NextResponse.redirect(
        new URL(`/login?from=${encodeURIComponent(from)}`, req.url)
      )
    }
  },
  {
    callbacks: {
      async authorized() {
        // 总是返回 true，让上面的 middleware 函数来做具体判断
        // 这是一个 workaround（临时解决方案）
        return true
      },
    },
  }
)

// 配置：只有这些路径需要经过中间件
export const config = {
  matcher: ["/dashboard/:path*", "/editor/:path*", "/login", "/register"],
}
```

> `matcher` 指定了"保安管哪些门"。首页 `/` 和定价页 `/pricing` 不在列表里，所以谁都能访问。

### 4.3 获取当前用户 -- `lib/session.ts`

这个文件很短，但非常实用：

```typescript
import { getServerSession } from "next-auth/next"
import { authOptions } from "@/lib/auth"

export async function getCurrentUser() {
  const session = await getServerSession(authOptions)
  return session?.user
}
```

> 只有 3 行有效代码！它把 NextAuth 的 `getServerSession` 封装了一下，让你在页面里用起来更方便：

```typescript
// 在任何 Server Component 或 API Route 中
const user = await getCurrentUser()
if (!user) redirect("/login")

// user 的结构：
// { id: "clk...", name: "John", email: "john@example.com", image: "https://..." }
```

### 4.4 NextAuth 路由处理器 -- `app/api/auth/[...nextauth]/_route.ts`

```typescript
import NextAuth from "next-auth"
import { authOptions } from "@/lib/auth"

const handler = NextAuth(authOptions)
export { handler as GET, handler as POST }
```

> 这个文件是 NextAuth 的"入口"。`[...nextauth]` 是 Next.js 的 catch-all 路由语法，它会匹配 `/api/auth/signin`、`/api/auth/callback/github` 等所有 NextAuth 需要的路径。

### 4.5 类型扩展 -- `types/next-auth.d.ts`

```typescript
import { User } from "next-auth"
import { JWT } from "next-auth/jwt"

type UserId = string

declare module "next-auth/jwt" {
  interface JWT {
    id: UserId   // 给 JWT 类型增加 id 字段
  }
}

declare module "next-auth" {
  interface Session {
    user: User & {
      id: UserId  // 给 Session.user 类型增加 id 字段
    }
  }
}
```

> NextAuth 默认的 Session 和 JWT 类型没有 `id` 字段。我们在回调里加了 `session.user.id = token.id`，如果不扩展类型，TypeScript 会报错说"id 不存在"。这个文件就是告诉 TypeScript："放心，id 是有的"。

---

## 五、动手练习

### 练习 1：追踪登录流程（约 40 分钟）

**目标：** 亲眼看到 NextAuth 回调的执行顺序。

**步骤：**

1. 打开 `lib/auth.ts`，在 `callbacks` 中加入 `console.log`：

```typescript
callbacks: {
  async session({ token, session }) {
    console.log("=== SESSION 回调被调用 ===")
    console.log("token:", token)
    console.log("session:", session)
    if (token) {
      session.user.id = token.id
      // ... 原有代码不动
    }
    return session
  },
  async jwt({ token, user }) {
    console.log("=== JWT 回调被调用 ===")
    console.log("token:", token)
    console.log("user:", user)  // 注意：只有登录时 user 才有值！
    // ... 原有代码不动
  },
}
```

2. 启动开发服务器 `pnpm dev`
3. 打开浏览器访问 `http://localhost:3000/login`
4. 点击 "Github" 按钮登录
5. 回到终端，观察 `console.log` 的输出顺序

**你应该看到：**
- 先打印 `=== JWT 回调被调用 ===`（此时 `user` 有值）
- 然后打印 `=== SESSION 回调被调用 ===`
- 之后每次刷新页面，JWT 回调会再次被调用（但 `user` 是 `undefined`）

**排错提示：**
- 如果 GitHub 登录报错，检查 `.env` 中 `GITHUB_CLIENT_ID` 和 `GITHUB_CLIENT_SECRET` 是否正确
- GitHub OAuth App 的 Callback URL 必须是 `http://localhost:3000/api/auth/callback/github`（一个字符都不能错）
- 练习完成后记得**删除所有 console.log**

### 练习 2：测试中间件保护（约 25 分钟）

**目标：** 亲身体验中间件的路由保护效果。

**步骤：**

1. 确保你已退出登录（如果已登录，点击头像菜单里的 "Sign out"）
2. 在浏览器地址栏直接输入 `http://localhost:3000/dashboard`，按回车
3. 观察发生了什么：
   - URL 变成了什么？（应该是 `/login?from=%2Fdashboard`）
   - `%2F` 是 `/` 的 URL 编码形式
4. 现在登录
5. 登录成功后，你是否被自动跳回了 `/dashboard`？

**进一步探索：**
- 试试直接访问 `http://localhost:3000/editor/123`，看看被重定向时 `from` 参数是什么
- 试试访问 `http://localhost:3000/pricing`，会被重定向吗？为什么不会？（因为不在 `matcher` 列表中）

**排错提示：**
- 如果没有被重定向，检查 `middleware.ts` 文件是否在项目根目录（不是 `src/` 下面）
- 中间件只对 `matcher` 中配置的路径生效

### 练习 3：阅读登录表单组件（约 20 分钟）

**目标：** 理解前端登录表单如何触发认证流程。

**步骤：**

1. 打开 `components/user-auth-form.tsx`
2. 找到 `signIn("email", { ... })` 这行，回答以下问题：
   - `"email"` 这个参数对应的是 `lib/auth.ts` 中哪个 Provider？
   - `redirect: false` 是什么意思？（提示：不让页面自动跳转，而是用代码控制后续行为）
   - `callbackUrl` 的值从哪里来的？（提示：看看 `searchParams`）
3. 找到 `signIn("github")` 这行：
   - 为什么 GitHub 登录不需要传 `redirect: false`？（提示：GitHub 登录需要跳转到 GitHub 页面）
4. 注意 `useForm<FormData>` 中的 `<FormData>` -- 这就是泛型，告诉 `useForm` 表单数据的类型

---

## 六、概念图解

### 6.1 OAuth 登录流程（以 GitHub 为例）

```
┌──────────┐     1. 点击"Sign in with GitHub"     ┌───────────────┐
│          │ ──────────────────────────────────────→│               │
│  用户的   │     2. 跳转到 GitHub 登录页            │   GitHub      │
│  浏览器   │ ←──────────────────────────────────── │   服务器      │
│          │     3. 用户输入 GitHub 账号密码          │               │
│          │ ──────────────────────────────────────→│               │
│          │     4. GitHub 验证通过                  │               │
│          │         返回授权码 (code)               │               │
│          │ ←──────────────────────────────────── │               │
└────┬─────┘                                       └───────┬───────┘
     │                                                     │
     │  5. 浏览器带着 code 访问                              │
     │     /api/auth/callback/github                        │
     ↓                                                     │
┌──────────────┐   6. NextAuth 用 code 换取用户信息    │
│              │ ──────────────────────────────────────→│
│  Taxonomy    │   7. GitHub 返回用户信息（名字/邮箱/头像）   │
│  服务器      │ ←──────────────────────────────────── │
│              │                                       │
│  8. 执行 jwt 回调 → 生成 JWT                          │
│  9. 执行 session 回调 → 构建 session                   │
│  10. PrismaAdapter 存储用户信息到数据库                  │
│  11. 设置 Cookie（包含 JWT）返回给浏览器                 │
└──────────────┘
```

### 6.2 认证生命周期（登录后每次请求）

```
用户访问 /dashboard
       │
       ↓
┌─────────────────────────────────┐
│  middleware.ts                   │
│  "我是保安，让我看看你的通行证"     │
│                                 │
│  1. getToken({ req })           │
│     从 Cookie 中提取 JWT         │
│                                 │
│  2. 有 JWT？                     │
│     ├── 有 → 放行，进入页面       │
│     └── 没有 → 重定向到 /login    │
└────────────┬────────────────────┘
             │ （有 JWT 的情况）
             ↓
┌─────────────────────────────────┐
│  页面 Server Component          │
│                                 │
│  const user = await             │
│    getCurrentUser()              │
│                                 │
│  内部流程：                       │
│  1. getServerSession(authOptions)│
│  2. 触发 jwt 回调 → 验证/刷新 JWT │
│  3. 触发 session 回调 → 构建会话  │
│  4. 返回 session.user            │
└─────────────────────────────────┘
```

### 6.3 项目中认证相关文件一览

```
taxonomy/
├── lib/
│   ├── auth.ts              ← 认证核心配置（Provider、回调、适配器）
│   ├── session.ts           ← getCurrentUser() 工具函数
│   └── validations/
│       └── auth.ts          ← 邮箱表单的验证规则（Zod schema）
├── middleware.ts             ← 路由保护中间件
├── app/
│   ├── api/auth/
│   │   └── [...nextauth]/
│   │       └── _route.ts    ← NextAuth 路由处理器（入口）
│   └── (auth)/
│       └── login/
│           └── page.tsx     ← 登录页面
├── components/
│   ├── user-auth-form.tsx   ← 登录表单（邮箱 + GitHub）
│   └── user-account-nav.tsx ← 已登录用户的头像下拉菜单
└── types/
    └── next-auth.d.ts       ← TypeScript 类型扩展
```

---

## 七、自检问题

完成学习后，试着不看代码回答以下问题。答不上来的，回去重新看对应的源码。

### 问题 1：OAuth 中谁负责验证密码？

**答案：** GitHub 负责验证密码。Taxonomy 的服务器永远看不到用户的 GitHub 密码。这正是 OAuth 的核心好处 -- 你的应用不需要管密码。

### 问题 2：`jwt` 回调和 `session` 回调的执行顺序是什么？

**答案：** 先执行 `jwt` 回调（生成/更新 JWT token），再执行 `session` 回调（用 token 中的数据构建 session 对象）。所以我们要在 `jwt` 回调里先把 `user.id` 存到 token 中，`session` 回调才能从 token 中取到 id。

### 问题 3：`middleware.ts` 中 `authorized()` 为什么总是返回 `true`？

**答案：** 这是一个 workaround。`withAuth` 的默认行为是：`authorized()` 返回 `false` 时直接重定向到登录页。但我们需要更细粒度的控制（比如已登录用户访问 `/login` 要跳转到 `/dashboard`），所以让 `authorized()` 总是返回 `true`，把真正的逻辑放在上面的 `middleware` 函数中。

### 问题 4：如果要新增 Google 登录，需要改哪些地方？

**答案：** 主要改两个地方：
1. `lib/auth.ts` -- 在 `providers` 数组中添加 `GoogleProvider({ clientId: ..., clientSecret: ... })`
2. `components/user-auth-form.tsx` -- 添加一个 "Sign in with Google" 按钮，点击时调用 `signIn("google")`
3. `.env` -- 添加 Google OAuth 的 Client ID 和 Secret

### 问题 5：JWT 存在 Cookie 里有什么限制？

**答案：** Cookie 有 4KB 的大小限制。JWT 本质上是把用户信息加密后存到 Cookie 中，所以不能往 token 里塞太多数据。Taxonomy 只存了 `id`、`name`、`email`、`picture` 四个字段，数据量很小，不会超限。

---

## 八、预计时间分配

| 环节 | 时间 | 说明 |
|------|------|------|
| 前置知识速补 | 30 分钟 | Cookie、HTTP headers、回调、闭包、泛型 |
| 源码精读 | 60 分钟 | 逐行读懂 auth.ts、middleware.ts、session.ts 等 |
| 练习 1：追踪登录流程 | 40 分钟 | 加 console.log、观察回调顺序 |
| 练习 2：测试中间件 | 25 分钟 | 手动测试路由保护效果 |
| 练习 3：阅读登录表单 | 20 分钟 | 理解前端如何触发认证 |
| 自检与笔记 | 15 分钟 | 回答自检问题、整理今日收获 |
| **合计** | **约 3 小时 10 分钟** | |

> 如果前置知识部分已经熟悉，可以快速跳过，总时间约 2.5 小时。

---

## 九、常见踩坑点

1. **GitHub OAuth 回调地址不匹配**
   Callback URL 必须严格设置为 `http://localhost:3000/api/auth/callback/github`。多一个 `/`、少一个字母都会报错。在 GitHub Settings -> Developer settings -> OAuth Apps 中检查。

2. **`NEXTAUTH_SECRET` 未设置**
   生产环境必须在 `.env` 中设置 `NEXTAUTH_SECRET`，否则 JWT 无法加密，认证系统完全失效。开发环境 NextAuth 会用默认值，但千万别在生产环境依赖默认值。

3. **JWT 回调中 `user` 参数有时是 `undefined`**
   `jwt` 回调在两种情况下被调用：(a) 用户登录时 -- `user` 有值；(b) 之后每次验证 session 时 -- `user` 是 `undefined`。所以代码中有 `if (user)` 的判断，不加这个判断会导致 `token.id` 被覆盖为 `undefined`。

4. **中间件运行在 Edge Runtime**
   `middleware.ts` 运行在边缘环境（Edge Runtime），不能使用 Node.js 专有的 API（比如 `fs`、`path`）。如果你在中间件里 import 了不兼容的库，会报错。

5. **Cookie 大小限制导致认证失败**
   JWT 存在 Cookie 里，Cookie 有 4KB 大小限制。如果你往 token 里塞了太多数据（比如用户的所有文章列表），Cookie 会超限，导致认证莫名其妙地失效。只存必要的字段（id、name、email、image）。

6. **`db as any` 的类型 hack**
   你可能注意到 `PrismaAdapter(db as any)` 里的 `as any`。这是因为 Prisma 客户端和 NextAuth 适配器之间有类型兼容性问题（是一个已知的 Prisma bug）。不用担心，功能上是完全正常的。

---

## 十、延伸阅读

- [NextAuth.js 官方文档](https://next-auth.js.org/) -- 最权威的参考，特别是 Configuration 和 Callbacks 部分
- [OAuth 2.0 简化解释](https://aaronparecki.com/oauth-2-simplified/) -- 用简单的语言解释 OAuth 协议（英文）
- [JWT.io](https://jwt.io/) -- 可以在线解码 JWT，看看里面到底存了什么
- [Next.js Middleware 文档](https://nextjs.org/docs/app/building-your-application/routing/middleware) -- 理解中间件的运行时机和限制
- [Prisma Adapter for NextAuth](https://authjs.dev/reference/adapter/prisma) -- 了解 PrismaAdapter 自动创建了哪些数据库表
- [HTTP Cookie 详解 (MDN)](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Cookies) -- Cookie 的完整技术文档（有中文）
