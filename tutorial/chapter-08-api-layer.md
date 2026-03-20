# 第8章：API 层 — 前后端怎么对话的？

## 学习目标

- 理解 RESTful API 的概念（GET、POST、PATCH、DELETE）
- 理解 Next.js Route Handlers 的写法
- 理解请求验证（Zod schema）
- 理解错误处理和状态码
- 理解前端如何调用 API（fetch）
- 理解权限控制（API 中检查登录状态）

## 涉及文件

| 文件 | 说明 |
|------|------|
| `app/api/posts/route.ts` | 帖子列表 & 创建 |
| `app/api/posts/[postId]/route.ts` | 帖子详情 & 更新 & 删除 |
| `app/api/users/[userId]/route.ts` | 用户更新 |
| `lib/validations/post.ts` | 帖子数据验证 |
| `lib/exceptions.ts` | 自定义异常类 |
| `components/post-create-button.tsx` | 前端创建帖子 |
| `components/post-operations.tsx` | 前端删除帖子 |
| `components/user-name-form.tsx` | 表单提交 API |

---

## 1. RESTful API 基础

**API（Application Programming Interface）** 是前端和后端通信的"接口"。就像餐厅的菜单——前端（顾客）看菜单（API 文档）点菜（发请求），后端（厨房）做菜（处理请求）并上菜（返回响应）。

**RESTful** 是一种 API 设计规范，用 HTTP 方法表示不同的操作：

| HTTP 方法 | 用途 | 例子 |
|-----------|------|------|
| `GET` | 读取数据 | 获取帖子列表 |
| `POST` | 创建数据 | 创建新帖子 |
| `PATCH` | 部分更新 | 修改帖子标题 |
| `PUT` | 完整替换 | 替换整个帖子 |
| `DELETE` | 删除数据 | 删除帖子 |

### HTTP 状态码

| 状态码 | 含义 | 什么时候用 |
|--------|------|-----------|
| `200` | 成功 | 请求处理成功 |
| `204` | 成功但无内容 | 删除成功 |
| `400` | 请求错误 | 参数格式不对 |
| `402` | 需要付费 | 免费用户超过限制 |
| `403` | 禁止访问 | 没有权限 |
| `404` | 未找到 | 资源不存在 |
| `422` | 验证失败 | 数据格式不符合要求 |
| `500` | 服务器错误 | 代码出 bug 了 |

---

## 2. Next.js Route Handlers

在 Next.js 中，API 路由放在 `app/api/` 目录下。文件名和路径决定了 API 的 URL：

```
app/api/posts/route.ts           → GET/POST  /api/posts
app/api/posts/[postId]/route.ts  → PATCH/DELETE /api/posts/:postId
app/api/users/[userId]/route.ts  → PATCH /api/users/:userId
app/api/users/stripe/route.ts    → GET /api/users/stripe
```

每个 `route.ts` 文件导出与 HTTP 方法同名的函数：

```typescript
// 处理 GET /api/posts
export async function GET(req: Request) { ... }

// 处理 POST /api/posts
export async function POST(req: Request) { ... }
```

---

## 3. 帖子 API 完整解读

### 3.1 GET — 获取帖子列表

```typescript
// app/api/posts/route.ts
export async function GET() {
  try {
    // 第1步：检查用户是否已登录
    const session = await getServerSession(authOptions)

    if (!session) {
      return new Response("Unauthorized", { status: 403 })
    }

    // 第2步：查询当前用户的所有帖子
    const { user } = session
    const posts = await db.post.findMany({
      select: {
        id: true,
        title: true,
        published: true,
        createdAt: true,
      },
      where: {
        authorId: user.id,
      },
    })

    // 第3步：返回 JSON 响应
    return new Response(JSON.stringify(posts))
  } catch (error) {
    return new Response(null, { status: 500 })
  }
}
```

### 3.2 POST — 创建新帖子

```typescript
// app/api/posts/route.ts

// 用 Zod 定义请求体的验证规则
const postCreateSchema = z.object({
  title: z.string(),            // 标题必须是字符串
  content: z.string().optional(), // 内容可选
})

export async function POST(req: Request) {
  try {
    // 第1步：验证登录状态
    const session = await getServerSession(authOptions)
    if (!session) {
      return new Response("Unauthorized", { status: 403 })
    }

    // 第2步：检查订阅计划（免费用户限制 3 篇帖子）
    const { user } = session
    const subscriptionPlan = await getUserSubscriptionPlan(user.id)

    if (!subscriptionPlan?.isPro) {
      const count = await db.post.count({
        where: { authorId: user.id },
      })

      if (count >= 3) {
        throw new RequiresProPlanError()  // 抛出自定义异常
      }
    }

    // 第3步：验证请求数据
    const json = await req.json()          // 解析请求体的 JSON
    const body = postCreateSchema.parse(json)  // 用 Zod 验证

    // 第4步：创建帖子
    const post = await db.post.create({
      data: {
        title: body.title,
        content: body.content,
        authorId: session.user.id,
      },
      select: { id: true },
    })

    // 第5步：返回新帖子的 ID
    return new Response(JSON.stringify(post))
  } catch (error) {
    // 错误处理：根据错误类型返回不同的状态码
    if (error instanceof z.ZodError) {
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }

    if (error instanceof RequiresProPlanError) {
      return new Response("Requires Pro Plan", { status: 402 })
    }

    return new Response(null, { status: 500 })
  }
}
```

### 3.3 PATCH — 更新帖子

```typescript
// app/api/posts/[postId]/route.ts

// 验证路由参数
const routeContextSchema = z.object({
  params: z.object({
    postId: z.string(),
  }),
})

export async function PATCH(
  req: Request,
  context: z.infer<typeof routeContextSchema>  // { params: { postId: string } }
) {
  try {
    const { params } = routeContextSchema.parse(context)

    // 权限检查
    if (!(await verifyCurrentUserHasAccessToPost(params.postId))) {
      return new Response(null, { status: 403 })
    }

    // 验证请求数据
    const json = await req.json()
    const body = postPatchSchema.parse(json)

    // 更新帖子
    await db.post.update({
      where: { id: params.postId },
      data: {
        title: body.title,
        content: body.content,
      },
    })

    return new Response(null, { status: 200 })
  } catch (error) {
    if (error instanceof z.ZodError) {
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }
    return new Response(null, { status: 500 })
  }
}
```

### 3.4 DELETE — 删除帖子

```typescript
// app/api/posts/[postId]/route.ts
export async function DELETE(req: Request, context) {
  try {
    const { params } = routeContextSchema.parse(context)

    // 权限检查：确保是自己的帖子
    if (!(await verifyCurrentUserHasAccessToPost(params.postId))) {
      return new Response(null, { status: 403 })
    }

    await db.post.delete({
      where: { id: params.postId },
    })

    return new Response(null, { status: 204 })  // 204 = 删除成功，无内容
  } catch (error) {
    if (error instanceof z.ZodError) {
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }
    return new Response(null, { status: 500 })
  }
}
```

---

## 4. Zod 数据验证

**Zod** 是一个数据验证库。它确保输入的数据格式正确，防止无效数据进入数据库。

```typescript
// lib/validations/post.ts
import * as z from "zod"

export const postPatchSchema = z.object({
  title: z.string().min(3).max(128).optional(),  // 标题：3-128字符，可选
  content: z.any().optional(),                    // 内容：任意类型，可选
})
```

### 验证流程

```typescript
// 正常数据 ✅
postPatchSchema.parse({ title: "My Post" })  // → { title: "My Post" }

// 标题太短 ❌
postPatchSchema.parse({ title: "Hi" })
// 抛出 ZodError: String must contain at least 3 character(s)

// 标题类型错误 ❌
postPatchSchema.parse({ title: 123 })
// 抛出 ZodError: Expected string, received number
```

**为什么需要验证？** 因为你永远不能信任来自客户端的数据。即使前端做了验证，恶意用户可以绕过前端直接调用 API。

---

## 5. 自定义异常类

```typescript
// lib/exceptions.ts
export class RequiresProPlanError extends Error {
  constructor(message = "This action requires a pro plan") {
    super(message)
  }
}
```

**为什么要自定义异常？** 这样在 `catch` 块中可以区分不同类型的错误，返回不同的状态码：

```typescript
catch (error) {
  if (error instanceof z.ZodError) {
    return new Response(..., { status: 422 })   // 验证错误
  }
  if (error instanceof RequiresProPlanError) {
    return new Response(..., { status: 402 })   // 需要付费
  }
  return new Response(null, { status: 500 })     // 未知错误
}
```

---

## 6. 前端调用 API

### 6.1 创建帖子（PostCreateButton）

```tsx
// components/post-create-button.tsx
"use client"

export function PostCreateButton({ className, variant, ...props }) {
  const router = useRouter()
  const [isLoading, setIsLoading] = React.useState(false)

  async function onClick() {
    setIsLoading(true)

    // 调用 API 创建帖子
    const response = await fetch("/api/posts", {
      method: "POST",                              // HTTP 方法
      headers: { "Content-Type": "application/json" },  // 告诉服务器发的是 JSON
      body: JSON.stringify({ title: "Untitled Post" }), // 请求体
    })

    setIsLoading(false)

    // 处理不同的响应状态
    if (!response?.ok) {
      if (response.status === 402) {
        return toast({
          title: "Limit of 3 posts reached.",
          description: "Please upgrade to the PRO plan.",
          variant: "destructive",
        })
      }
      return toast({
        title: "Something went wrong.",
        variant: "destructive",
      })
    }

    const post = await response.json()

    router.refresh()               // 刷新页面数据（重新运行 Server Component）
    router.push(`/editor/${post.id}`)  // 跳转到编辑器
  }

  return (
    <button onClick={onClick} disabled={isLoading} {...props}>
      {isLoading ? <Icons.spinner className="animate-spin" /> : <Icons.add />}
      New post
    </button>
  )
}
```

### 6.2 删除帖子（PostOperations）

```tsx
// components/post-operations.tsx
"use client"

// 独立的 API 调用函数
async function deletePost(postId: string) {
  const response = await fetch(`/api/posts/${postId}`, {
    method: "DELETE",
  })

  if (!response?.ok) {
    toast({
      title: "Something went wrong.",
      description: "Your post was not deleted. Please try again.",
      variant: "destructive",
    })
  }

  return true
}

export function PostOperations({ post }) {
  const router = useRouter()
  const [showDeleteAlert, setShowDeleteAlert] = React.useState(false)
  const [isDeleteLoading, setIsDeleteLoading] = React.useState(false)

  return (
    <>
      {/* 下拉菜单 */}
      <DropdownMenu>
        <DropdownMenuTrigger>
          <Icons.ellipsis className="h-4 w-4" />
        </DropdownMenuTrigger>
        <DropdownMenuContent>
          <DropdownMenuItem>
            <Link href={`/editor/${post.id}`}>Edit</Link>
          </DropdownMenuItem>
          <DropdownMenuSeparator />
          <DropdownMenuItem
            className="text-destructive"
            onSelect={() => setShowDeleteAlert(true)}
          >
            Delete
          </DropdownMenuItem>
        </DropdownMenuContent>
      </DropdownMenu>

      {/* 删除确认对话框 */}
      <AlertDialog open={showDeleteAlert} onOpenChange={setShowDeleteAlert}>
        <AlertDialogContent>
          <AlertDialogHeader>
            <AlertDialogTitle>
              Are you sure you want to delete this post?
            </AlertDialogTitle>
            <AlertDialogDescription>
              This action cannot be undone.
            </AlertDialogDescription>
          </AlertDialogHeader>
          <AlertDialogFooter>
            <AlertDialogCancel>Cancel</AlertDialogCancel>
            <AlertDialogAction
              onClick={async (event) => {
                event.preventDefault()
                setIsDeleteLoading(true)

                const deleted = await deletePost(post.id)

                if (deleted) {
                  setIsDeleteLoading(false)
                  setShowDeleteAlert(false)
                  router.refresh()  // 刷新页面，帖子列表会自动更新
                }
              }}
            >
              {isDeleteLoading ? <Icons.spinner className="animate-spin" /> : <Icons.trash />}
              Delete
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </>
  )
}
```

### 6.3 表单提交（UserNameForm）

```tsx
// components/user-name-form.tsx
"use client"

export function UserNameForm({ user, className, ...props }) {
  const router = useRouter()
  const {
    handleSubmit,
    register,
    formState: { errors },
  } = useForm<FormData>({
    resolver: zodResolver(userNameSchema),  // 前端也用 Zod 验证
    defaultValues: { name: user?.name || "" },
  })
  const [isSaving, setIsSaving] = React.useState(false)

  async function onSubmit(data: FormData) {
    setIsSaving(true)

    const response = await fetch(`/api/users/${user.id}`, {
      method: "PATCH",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ name: data.name }),
    })

    setIsSaving(false)

    if (!response?.ok) {
      return toast({ title: "Something went wrong.", variant: "destructive" })
    }

    toast({ description: "Your name has been updated." })
    router.refresh()
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Card>
        <CardHeader>
          <CardTitle>Your Name</CardTitle>
        </CardHeader>
        <CardContent>
          <Input id="name" {...register("name")} />
          {errors?.name && <p className="text-red-600">{errors.name.message}</p>}
        </CardContent>
        <CardFooter>
          <button type="submit" disabled={isSaving}>
            {isSaving && <Icons.spinner className="animate-spin" />}
            Save
          </button>
        </CardFooter>
      </Card>
    </form>
  )
}
```

---

## 7. API 设计模式总结

项目中每个 API 都遵循相同的模式：

```typescript
export async function METHOD(req: Request) {
  try {
    // 1. 验证登录状态
    const session = await getServerSession(authOptions)
    if (!session) return new Response("Unauthorized", { status: 403 })

    // 2. 验证权限（如果需要）
    if (!(await verifyAccess(...))) return new Response(null, { status: 403 })

    // 3. 验证请求数据（使用 Zod）
    const json = await req.json()
    const body = schema.parse(json)

    // 4. 执行数据库操作
    const result = await db.model.operation(...)

    // 5. 返回响应
    return new Response(JSON.stringify(result))
  } catch (error) {
    // 6. 统一错误处理
    if (error instanceof z.ZodError) {
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }
    return new Response(null, { status: 500 })
  }
}
```

---

## 8. `router.refresh()` 的作用

你会在前端代码中频繁看到 `router.refresh()`。它的作用是**重新运行当前页面的 Server Component**，获取最新数据。

```
用户创建帖子
    ↓
调用 POST /api/posts
    ↓
API 返回成功
    ↓
router.refresh()  ← 告诉 Next.js："数据变了，重新跑一遍 Server Component"
    ↓
仪表盘页面重新查数据库，帖子列表自动包含新帖子
```

这比传统的"手动更新前端状态"更简单——你不需要维护复杂的前端状态，直接让服务器重新查一遍。

---

## 小练习

### 练习 1：用浏览器开发者工具观察 API 调用
1. 打开仪表盘页面
2. 打开浏览器的"网络"面板
3. 点击"New post"按钮
4. 观察网络请求：URL 是什么？方法是什么？请求体是什么？响应是什么？

### 练习 2：理解错误处理
如果在创建帖子时传入 `{ title: 123 }`（数字而不是字符串），API 会返回什么状态码？为什么？

### 练习 3：写一个简单的 API
在 `app/api/hello/route.ts` 中创建一个简单的 API：
```typescript
export async function GET() {
  return new Response(JSON.stringify({ message: "Hello World" }))
}
```
然后在浏览器中访问 `/api/hello`。

### 练习 4：前端验证 vs 后端验证
`user-name-form.tsx` 中前端用了 `zodResolver(userNameSchema)` 验证。API 也做了验证。为什么两边都要验证？能不能只在前端验证？

---

## 本章小结

- RESTful API 用 HTTP 方法（GET/POST/PATCH/DELETE）区分操作
- Next.js Route Handlers 通过导出同名函数处理对应的 HTTP 方法
- Zod 用于验证请求数据，确保格式正确
- API 遵循统一模式：验证登录 → 验证权限 → 验证数据 → 操作数据库 → 返回响应
- 前端用 `fetch` 调用 API，用 `router.refresh()` 刷新数据
- 前后端都应该做数据验证（前端为体验，后端为安全）

下一章我们将学习 Stripe 付费系统。
