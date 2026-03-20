# Day 08 — API 层：前端和后端怎么对话？

> 对应章节：[Chapter 8: API 层](../chapter-08-api-layer.md)

---

## 今日学习目标

1. 理解 HTTP 请求的基本概念，能看懂 Next.js Route Handler 的写法
2. 掌握 Zod 数据校验的用法，理解"为什么后端一定要验证数据"
3. 能从头到尾读懂一个完整的 CRUD API（创建、读取、更新、删除）

---

## 一、前置知识速补（约 50 分钟）

> 如果你对 HTTP 完全陌生，这一节非常重要，请务必认真读完。

### 1.1 [HTTP 方法](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Methods) — 类比餐厅点单

想象你去一家餐厅：

| 你想做的事 | 餐厅类比 | HTTP 方法 | 代码示例 |
|-----------|---------|----------|---------|
| 看菜单 | "服务员，菜单给我看看" | `GET` | `fetch("/api/posts")` |
| 点菜 | "我要一份宫保鸡丁" | `POST` | `fetch("/api/posts", { method: "POST" })` |
| 改菜 | "鸡丁少放辣" | `PATCH` | `fetch("/api/posts/123", { method: "PATCH" })` |
| 退菜 | "这道菜不要了" | `DELETE` | `fetch("/api/posts/123", { method: "DELETE" })` |

重点记住：**GET 读、POST 创建、PATCH 更新、DELETE 删除**。

### 1.2 [JSON](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Objects/JSON) 格式 — 数据的"通用语言"

JSON（JavaScript Object Notation）就是一种写数据的格式。前端和后端用它来"说话"。

```json
{
  "title": "我的第一篇帖子",
  "content": "Hello World",
  "published": false
}
```

几条规则：
- 键（key）必须用**双引号**包裹：`"title"` 而不是 `title`
- 值（value）可以是字符串、数字、布尔值、数组、对象、`null`
- 最后一项后面**不能**加逗号（和 JS 对象不同）

```javascript
// JavaScript 对象 → JSON 字符串
const obj = { title: "Hello" }
const jsonString = JSON.stringify(obj)  // '{"title":"Hello"}'

// JSON 字符串 → JavaScript 对象
const parsed = JSON.parse('{"title":"Hello"}')  // { title: "Hello" }
```

### 1.3 [fetch API](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API) — 浏览器内置的"发请求"方法

`fetch` 是浏览器内置的函数，用来向服务器发送请求。

```javascript
// 最简单的 GET 请求
const response = await fetch("/api/posts")
//    ^^^^^^^^                ^^^^^^^^^^^
//    服务器的回复              请求的地址（URL）

// 带请求体的 POST 请求
const response = await fetch("/api/posts", {
  method: "POST",                                // 用 POST 方法
  headers: { "Content-Type": "application/json" }, // 告诉服务器：我发的是 JSON
  body: JSON.stringify({ title: "新帖子" }),       // 请求体：要发送的数据
})
```

`fetch` 返回一个 [Response](https://developer.mozilla.org/zh-CN/docs/Web/API/Response) 对象（服务器的回复），常用属性：

```javascript
response.ok        // true 表示成功（状态码 200-299）
response.status    // 状态码数字，比如 200、403、500
await response.json()  // 把回复的 JSON 解析成 JS 对象
```

### 1.4 [HTTP 状态码](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status) — 服务器的"表情包"

状态码是服务器用一个数字告诉你"结果怎么样"。

| 状态码 | 含义 | 大白话 |
|--------|------|--------|
| `200` | OK，成功 | "搞定了！" |
| `204` | 成功，但没内容 | "删掉了，没啥好回你的" |
| `400` | 请求有误 | "你说的啥？没听懂" |
| `401` | 未认证 | "你谁啊？先登录" |
| `402` | 需要付费 | "这是付费功能" |
| `403` | 禁止访问 | "你没权限干这事" |
| `404` | 未找到 | "没这个东西" |
| `422` | 数据校验失败 | "格式不对，重填" |
| `500` | 服务器内部错误 | "我（服务器）出 bug 了" |

记忆技巧：**2xx 是好事，4xx 是你（客户端）的问题，5xx 是服务器的问题**。

### 1.5 [Request](https://developer.mozilla.org/zh-CN/docs/Web/API/Request) 和 Response 对象

在 Next.js [Route Handler](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) 里，你会频繁接触两个东西：

```typescript
// Request — 客户端发来的请求
export async function POST(req: Request) {
  const data = await req.json()  // 获取请求体里的 JSON 数据
  const url = req.url             // 获取请求的完整 URL
}

// Response — 你要返回给客户端的回复
return new Response(
  JSON.stringify({ id: "123" }),  // 第1个参数：回复的内容（字符串）
  { status: 200 }                 // 第2个参数：配置项（状态码等）
)

// 返回空内容的回复
return new Response(null, { status: 204 })
```

---

## 二、核心概念清单

| 概念 | 一句话解释 | 在项目中的位置 |
|------|-----------|---------------|
| Route Handler | Next.js 里写后端 API 的方式，放在 `app/api/` 下 | `app/api/posts/route.ts` |
| [Zod](https://zod.dev/) | 数据校验库，检查输入数据是否合格 | `lib/validations/post.ts` |
| CRUD API | 增删改查四种操作的统称 | `app/api/posts/` 下的两个文件 |
| 错误处理 | [try...catch](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/try...catch) 捕获异常，返回合适的状态码 | 每个 route handler 都有 |
| 权限校验 | 检查用户是否登录、是否有权操作 | `verifyCurrentUserHasAccessToPost()` |
| 自定义异常 | 继承 `Error` 类，用于区分不同错误类型 | `lib/exceptions.ts` |

---

## 三、源码精读（约 60 分钟）

### 3.1 文件结构一览

```
app/api/
  posts/
    route.ts                → GET /api/posts     获取帖子列表
                            → POST /api/posts    创建新帖子
    [postId]/
      route.ts              → PATCH /api/posts/帖子ID  更新帖子
                            → DELETE /api/posts/帖子ID  删除帖子
  users/
    [userId]/
      route.ts              → PATCH /api/users/用户ID  更新用户名

lib/
  validations/
    post.ts                 → 帖子数据的 Zod 校验规则
    user.ts                 → 用户数据的 Zod 校验规则
  exceptions.ts             → 自定义异常类
```

文件路径中的 `[postId]` 是[**动态参数**](https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes)。访问 `/api/posts/abc123` 时，`postId` 的值就是 `"abc123"`。

### 3.2 精读：GET — 获取帖子列表

打开 `app/api/posts/route.ts`，我们逐行来看 `GET` 函数：

```typescript
// === 文件：app/api/posts/route.ts ===

import { getServerSession } from "next-auth/next"  // 获取当前登录用户
import * as z from "zod"                            // 数据校验库
import { authOptions } from "@/lib/auth"            // 认证配置
import { db } from "@/lib/db"                       // 数据库连接

export async function GET() {
  // ↑ 导出一个叫 GET 的函数，Next.js 会自动把它绑到 GET /api/posts

  try {
    // 第1步：检查用户是否已登录
    const session = await getServerSession(authOptions)
    // session 长这样：{ user: { id: "xxx", name: "张三", email: "..." } }
    // 如果没登录，session 是 null

    if (!session) {
      // 没登录？返回 403（禁止访问）
      return new Response("Unauthorized", { status: 403 })
    }

    // 第2步：从数据库查询当前用户的所有帖子
    const { user } = session   // 从 session 里取出 user 对象
    const posts = await db.post.findMany({
      select: {                // select：只查这几个字段（性能优化）
        id: true,
        title: true,
        published: true,
        createdAt: true,
      },
      where: {
        authorId: user.id,     // 只查"我的"帖子
      },
    })

    // 第3步：把查到的数据转成 JSON 字符串，返回给前端
    return new Response(JSON.stringify(posts))
    // 默认状态码是 200（成功）
  } catch (error) {
    // 出了任何意外错误，返回 500
    return new Response(null, { status: 500 })
  }
}
```

> 注意 `select` 的作用：不写 `select` 会查出帖子的**所有字段**（包括很长的 `content`）。写了 `select` 只返回列表页需要的字段，减少传输量。

### 3.3 精读：POST — 创建新帖子

同一个文件中的 `POST` 函数更复杂一些：

```typescript
// === 文件：app/api/posts/route.ts（续） ===

// 用 Zod 定义"创建帖子"的数据格式要求
const postCreateSchema = z.object({
  title: z.string(),              // title 必须是字符串
  content: z.string().optional(), // content 可选，是字符串
})
// 如果传入 { title: 123 }，z.string() 会报错，因为 123 不是字符串

export async function POST(req: Request) {
  //                        ^^^^^^^^^^^
  //                        req 就是前端发来的请求
  try {
    // 第1步：检查登录
    const session = await getServerSession(authOptions)
    if (!session) {
      return new Response("Unauthorized", { status: 403 })
    }

    // 第2步：检查订阅计划（免费用户最多 3 篇帖子）
    const { user } = session
    const subscriptionPlan = await getUserSubscriptionPlan(user.id)

    if (!subscriptionPlan?.isPro) {        // 不是 Pro 用户
      const count = await db.post.count({  // 数一下已有几篇帖子
        where: { authorId: user.id },
      })
      if (count >= 3) {
        throw new RequiresProPlanError()   // 超过 3 篇，抛出自定义异常
      }
    }

    // 第3步：从请求体中取出 JSON 数据，并用 Zod 校验
    const json = await req.json()           // 把请求体解析成 JS 对象
    const body = postCreateSchema.parse(json) // Zod 校验，不合格会抛 ZodError

    // 第4步：往数据库里创建一条帖子记录
    const post = await db.post.create({
      data: {
        title: body.title,
        content: body.content,
        authorId: session.user.id,  // 作者就是当前登录用户
      },
      select: { id: true },         // 只返回新帖子的 id
    })

    // 第5步：返回新帖子的 id
    return new Response(JSON.stringify(post))  // { "id": "xxx" }

  } catch (error) {
    // 根据异常类型返回不同状态码
    if (error instanceof z.ZodError) {
      // 数据校验失败 → 422
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }
    if (error instanceof RequiresProPlanError) {
      // 免费用户超限 → 402
      return new Response("Requires Pro Plan", { status: 402 })
    }
    // 其他未知错误 → 500
    return new Response(null, { status: 500 })
  }
}
```

### 3.4 精读：PATCH 和 DELETE — 更新与删除帖子

打开 `app/api/posts/[postId]/route.ts`：

```typescript
// === 文件：app/api/posts/[postId]/route.ts ===

// 校验路由参数（确保 postId 是字符串）
const routeContextSchema = z.object({
  params: z.object({
    postId: z.string(),
  }),
})

export async function PATCH(
  req: Request,
  context: z.infer<typeof routeContextSchema>
  //       ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  //       context 的类型是 { params: { postId: string } }
  //       Next.js 会自动把 URL 里的动态参数传进来
) {
  try {
    // 第1步：校验路由参数
    const { params } = routeContextSchema.parse(context)

    // 第2步：权限检查 — 这篇帖子是不是"我的"？
    if (!(await verifyCurrentUserHasAccessToPost(params.postId))) {
      return new Response(null, { status: 403 })  // 不是你的，没权限
    }

    // 第3步：校验请求体
    const json = await req.json()
    const body = postPatchSchema.parse(json)  // 来自 lib/validations/post.ts

    // 第4步：更新数据库
    await db.post.update({
      where: { id: params.postId },
      data: {
        title: body.title,
        content: body.content,
      },
    })

    return new Response(null, { status: 200 })  // 更新成功
  } catch (error) {
    if (error instanceof z.ZodError) {
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }
    return new Response(null, { status: 500 })
  }
}
```

DELETE 的逻辑类似，只是第3步直接删除，不需要请求体：

```typescript
export async function DELETE(
  req: Request,
  context: z.infer<typeof routeContextSchema>
) {
  try {
    const { params } = routeContextSchema.parse(context)

    if (!(await verifyCurrentUserHasAccessToPost(params.postId))) {
      return new Response(null, { status: 403 })
    }

    await db.post.delete({
      where: { id: params.postId as string },
    })

    return new Response(null, { status: 204 })  // 204 = 删除成功，无内容返回
  } catch (error) {
    if (error instanceof z.ZodError) {
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }
    return new Response(null, { status: 500 })
  }
}
```

### 3.5 精读：权限校验函数

在同一个文件底部有一个辅助函数：

```typescript
// === 文件：app/api/posts/[postId]/route.ts（底部） ===

async function verifyCurrentUserHasAccessToPost(postId: string) {
  const session = await getServerSession(authOptions)  // 获取当前用户
  const count = await db.post.count({
    where: {
      id: postId,              // 帖子 ID 匹配
      authorId: session?.user.id,  // 且作者是当前用户
    },
  })
  return count > 0  // 如果查到了，说明这篇帖子属于当前用户
}
```

这个函数的思路很巧妙：不是先查帖子再比较作者，而是**直接在一次查询里同时匹配帖子 ID 和作者 ID**。如果能查到（`count > 0`），就说明有权限。

### 3.6 精读：Zod 校验规则

```typescript
// === 文件：lib/validations/post.ts ===
import * as z from "zod"

export const postPatchSchema = z.object({
  title: z.string().min(3).max(128).optional(),
  //     ^^^^^^^^^ ^^^^^^ ^^^^^^^^ ^^^^^^^^^^
  //     必须是字符串  至少3字  最多128字  这个字段可以不传
  content: z.any().optional(),
  //       ^^^^^^
  //       任意类型都行（因为编辑器内容格式复杂）
})
```

```typescript
// === 文件：lib/validations/user.ts ===
export const userNameSchema = z.object({
  name: z.string().min(3).max(32),
  //    必须是字符串，3-32 个字符，且是必填的（没有 .optional()）
})
```

Zod 校验的逻辑是：调用 [.parse()](https://zod.dev/?id=parse) 时，**数据合格就放行，不合格就抛出 `ZodError`**。

```typescript
// 合格
postPatchSchema.parse({ title: "Hello World" })  // 正常返回 { title: "Hello World" }

// 不合格：标题太短
postPatchSchema.parse({ title: "Hi" })
// 抛出 ZodError: String must contain at least 3 character(s)

// 不合格：类型错误
postPatchSchema.parse({ title: 123 })
// 抛出 ZodError: Expected string, received number
```

### 3.7 精读：自定义异常类

```typescript
// === 文件：lib/exceptions.ts ===
export class RequiresProPlanError extends Error {
  constructor(message = "This action requires a pro plan") {
    super(message)
  }
}
```

为什么要自定义异常？因为在 `catch` 块里，你需要区分"到底出了什么错"：

```typescript
catch (error) {
  if (error instanceof z.ZodError) { ... }           // 数据校验错误
  if (error instanceof RequiresProPlanError) { ... }  // 免费用户超限
  // 其他：未知错误
}
```

[instanceof](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/instanceof) 就像在问"这个错误是不是属于某一类？"。不同类型的错误，返回不同的状态码。

---

## 四、动手练习（约 50 分钟）

### 练习 1：写一个 Hello World API（20 分钟）

**目标**：从零创建一个 API 端点，体验 Route Handler 的写法。

**步骤**：

1. 创建文件 `app/api/hello/route.ts`
2. 写入以下代码：

```typescript
export async function GET() {
  return new Response(
    JSON.stringify({ message: "Hello World", time: new Date().toISOString() })
  )
}
```

3. 启动项目（`npm run dev`），在浏览器中访问 `http://localhost:3000/api/hello`
4. 你应该能看到 JSON 格式的响应

**进阶**：给它加一个 POST 方法，接收 `{ name: "你的名字" }` 并返回 `{ message: "Hello, 你的名字!" }`：

```typescript
export async function POST(req: Request) {
  const body = await req.json()
  return new Response(
    JSON.stringify({ message: `Hello, ${body.name}!` })
  )
}
```

测试方法 — 在浏览器控制台（按 F12 打开）里运行：

```javascript
const res = await fetch("/api/hello", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "小明" })
})
const data = await res.json()
console.log(data)  // { message: "Hello, 小明!" }
```

> 排错提示：如果看到 404，检查文件路径是否正确（必须是 `app/api/hello/route.ts`，不是 `page.ts`）。如果 POST 请求报错，检查有没有写 `headers: { "Content-Type": "application/json" }`。

### 练习 2：用浏览器 Network 面板观察真实 API 调用（15 分钟）

**目标**：亲眼看看前端和后端是怎么"对话"的。

**步骤**：

1. 用浏览器打开项目的 Dashboard 页面
2. 按 `F12` 打开开发者工具，切到 **Network**（网络）面板
3. 点击页面上的 **New post** 按钮
4. 在 Network 面板中找到 `/api/posts` 这条请求，点击它
5. 依次查看：
   - **Headers** 标签：看 Request Method 是不是 `POST`
   - **Payload** 标签：看发送的 JSON 数据（应该有 `title: "Untitled Post"`）
   - **Response** 标签：看服务器返回了什么（应该有新帖子的 `id`）
   - **状态码**：左边应该显示 `200`

> 排错提示：如果 Network 面板是空的，先点面板左上角的红色圆点确保"录制"是开启的，然后再操作。

### 练习 3：给 Hello API 加上 Zod 校验（15 分钟）

**目标**：体验 Zod 校验，理解校验失败时会发生什么。

**步骤**：

1. 修改你在练习 1 创建的 `app/api/hello/route.ts`：

```typescript
import { z } from "zod"

const helloSchema = z.object({
  name: z.string().min(2).max(50),
  //    必须是字符串，2-50 个字符
})

export async function POST(req: Request) {
  try {
    const json = await req.json()
    const body = helloSchema.parse(json)  // 用 Zod 校验

    return new Response(
      JSON.stringify({ message: `Hello, ${body.name}!` })
    )
  } catch (error) {
    if (error instanceof z.ZodError) {
      return new Response(
        JSON.stringify(error.issues),  // 返回具体的校验错误信息
        { status: 422 }
      )
    }
    return new Response(null, { status: 500 })
  }
}
```

2. 在浏览器控制台中测试各种情况：

```javascript
// 正常情况
await fetch("/api/hello", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "小明" })
}).then(r => r.json()).then(console.log)
// 预期：{ message: "Hello, 小明!" }

// 名字太短
await fetch("/api/hello", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: "A" })
}).then(r => r.json()).then(console.log)
// 预期：[{ code: "too_small", ... }]，状态码 422

// 类型错误
await fetch("/api/hello", {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ name: 123 })
}).then(r => r.json()).then(console.log)
// 预期：[{ code: "invalid_type", ... }]，状态码 422
```

> 排错提示：如果 `z` 导入报错，确保项目已经安装了 zod 依赖（`npm list zod` 查看）。

---

## 五、概念图解

### 5.1 API 请求的完整生命周期

```
前端（浏览器）                              后端（Next.js Route Handler）
===========                              ==========================

1. 用户点击按钮
       |
2. fetch("/api/posts", {
     method: "POST",
     body: JSON.stringify(...)
   })
       |
       | ------ HTTP 请求 ------>
       |                          3. 检查登录状态（session）
       |                                |
       |                          4. 检查权限（是否有权操作）
       |                                |
       |                          5. Zod 校验请求数据
       |                                |
       |                          6. 操作数据库（Prisma）
       |                                |
       |                          7. 构建 Response
       |                                |
       | <---- HTTP 响应 ---------
       |
8. 检查 response.ok
       |
9. 解析 response.json()
       |
10. 更新页面 / 显示提示
```

### 5.2 项目中的 API 统一模式

每个 Route Handler 都遵循相同的"五步走"：

```
Step 1  验证登录  →  没登录？返回 403
           ↓
Step 2  验证权限  →  没权限？返回 403
           ↓
Step 3  Zod 校验  →  数据不合格？返回 422
           ↓
Step 4  数据库操作  →  Prisma 增删改查
           ↓
Step 5  返回响应  →  JSON.stringify(result)
```

### 5.3 REST 架构 — 资源和操作的对应关系

```
资源：帖子（Post）

URL                          方法      操作        对应函数
/api/posts                   GET       查列表      GET()  in posts/route.ts
/api/posts                   POST      创建        POST() in posts/route.ts
/api/posts/[postId]          PATCH     更新        PATCH() in posts/[postId]/route.ts
/api/posts/[postId]          DELETE    删除        DELETE() in posts/[postId]/route.ts

资源：用户（User）

URL                          方法      操作        对应函数
/api/users/[userId]          PATCH     改名        PATCH() in users/[userId]/route.ts
```

同一个 URL 可以对应不同的操作，**靠 HTTP 方法来区分**。

---

## 六、自检问题（附答案）

### 问题 1：Route Handler 文件必须叫什么名字？放在哪个目录下？

<details>
<summary>点击查看答案</summary>

文件必须叫 `route.ts`（不是 `page.ts` 或其他名字）。放在 `app/api/` 目录下。文件的路径决定了 API 的 URL，比如 `app/api/posts/route.ts` 对应 `GET/POST /api/posts`。

注意：同一个目录下**不能同时有** `route.ts` 和 `page.tsx`。
</details>

### 问题 2：为什么 API 要先检查登录、再校验数据？能不能反过来？

<details>
<summary>点击查看答案</summary>

技术上可以反过来，但先查登录更合理，原因是：
- 没登录的用户根本不应该走到"校验数据"这一步，直接拒绝更高效
- 这是一个安全最佳实践：**先验身份，再处理数据**
- 类比：餐厅先检查你有没有预约（身份），再接受你的点单（数据）
</details>

### 问题 3：`postPatchSchema` 中 `title` 加了 `.optional()`，这意味着什么？

<details>
<summary>点击查看答案</summary>

[.optional()](https://zod.dev/?id=optionals) 表示这个字段**可以不传**。在 PATCH（部分更新）场景下，用户可能只改内容不改标题，所以标题是可选的。但如果传了 `title`，它必须满足 `.min(3).max(128)` 的规则（3-128 个字符）。

对比 `postCreateSchema` 中的 `title: z.string()` 没有 `.optional()`，说明创建时标题是必填的。
</details>

### 问题 4：前端调用 API 成功后为什么要执行 `router.refresh()`？

<details>
<summary>点击查看答案</summary>

`router.refresh()` 会**重新运行当前页面的 Server Component**，从数据库获取最新数据。

比如你创建了一篇新帖子，API 返回成功后，Dashboard 页面的帖子列表还是旧的。调用 `router.refresh()` 后，Next.js 会重新在服务端查询数据库，帖子列表就会自动包含新帖子。

如果不调用，页面数据不会自动更新，用户需要手动刷新浏览器才能看到变化。
</details>

### 问题 5：`verifyCurrentUserHasAccessToPost` 是怎么判断权限的？为什么用 `count` 而不是先查帖子再比较作者？

<details>
<summary>点击查看答案</summary>

这个函数在数据库中执行了一次 `count` 查询，同时匹配帖子 ID 和作者 ID：

```typescript
const count = await db.post.count({
  where: {
    id: postId,
    authorId: session?.user.id,
  },
})
return count > 0
```

如果查到了（`count > 0`），说明这篇帖子存在且属于当前用户。

这比"先查帖子，再比较作者"更好，因为：
1. 只需要**一次数据库查询**（而不是两次）
2. `count` 只返回一个数字，比返回整个帖子对象**更轻量**
3. 代码更简洁，不需要 `if (post.authorId !== user.id)` 的判断
</details>

---

## 七、预计时间分配（共约 3.5 小时）

| 环节 | 时间 | 说明 |
|------|------|------|
| 前置知识速补 | 50 分钟 | HTTP、JSON、fetch、状态码 |
| 源码精读 | 60 分钟 | 逐行读懂每个 Route Handler |
| 练习 1：Hello API | 20 分钟 | 动手写一个最简单的 API |
| 练习 2：Network 面板 | 15 分钟 | 用浏览器观察真实的 API 调用 |
| 练习 3：Zod 校验 | 15 分钟 | 体验数据校验的效果 |
| 自检问题 | 10 分钟 | 检查理解程度 |
| **合计** | **约 2 小时 50 分钟** | 如果基础较弱可留 3.5 小时 |

---

## 八、常见踩坑点

### 1. 文件名写错

API 文件**必须**叫 `route.ts`，不是 `page.ts`、`api.ts` 或其他名字。写错了 Next.js 不会报错，但访问时会 404。

### 2. 忘了 `await`

`req.json()`、`db.post.findMany()`、`getServerSession()` 都是**异步函数**，必须加 [await](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/await)。忘了 `await` 不会报语法错误，但你拿到的是一个 Promise 对象而不是实际数据：

```typescript
// 错误写法
const data = req.json()           // data 是 Promise，不是你要的数据
console.log(data)                 // 输出：Promise { <pending> }

// 正确写法
const data = await req.json()     // data 是解析后的 JS 对象
console.log(data)                 // 输出：{ title: "Hello" }
```

### 3. POST/PATCH 请求忘了设 Content-Type

前端用 `fetch` 发送 JSON 数据时，必须加 `headers: { "Content-Type": "application/json" }`。不加的话，后端 `req.json()` 可能无法正确解析请求体。

### 4. Zod `.parse()` vs `.safeParse()`

`.parse()` 在校验失败时会**抛出异常**（需要 `try...catch`）。如果你想避免异常，可以用 [.safeParse()](https://zod.dev/?id=safeparse)，它返回一个结果对象：

```typescript
const result = postPatchSchema.safeParse({ title: "Hi" })
if (!result.success) {
  console.log(result.error.issues)  // 校验错误详情
} else {
  console.log(result.data)          // 校验通过的数据
}
```

项目中统一使用 `.parse()` + `try...catch`，这也是常见做法。

### 5. 动态路由参数类型

`[postId]` 从 URL 中拿到的参数永远是**字符串**。即使 URL 看起来像数字（`/api/posts/123`），`postId` 的值也是 `"123"`（字符串），不是 `123`（数字）。

---

## 九、延伸阅读

- [Next.js 官方文档：Route Handlers](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) — 英文，但有完整的 API 说明
- [Zod 官方文档](https://zod.dev/) — 想深入了解各种校验规则可以看这里
- [MDN Web Docs: HTTP 状态码](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status) — 中文，所有状态码的详细解释
- [MDN Web Docs: Fetch API](https://developer.mozilla.org/zh-CN/docs/Web/API/Fetch_API/Using_Fetch) — 中文，`fetch` 的完整用法
- [RESTful API 设计指南（阮一峰）](https://www.ruanyifeng.com/blog/2014/05/restful_api.html) — 中文，REST 架构的通俗讲解
