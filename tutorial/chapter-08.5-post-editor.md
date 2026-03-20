# 第8.5章：文章编辑器 — 用户是怎样写文章的？

## 学习目标

- 理解 Editor 路由组的最小化布局设计
- 理解 EditorJS 富文本编辑器的集成方式
- 理解"use client" + 动态导入的初始化模式
- 理解文章的完整生命周期（创建 → 编辑 → 保存 → 删除）
- 理解 Dashboard 中的关键 UI 组件模式

## 涉及文件

| 文件 | 说明 |
|------|------|
| `app/(editor)/editor/layout.tsx` | 编辑器最小化布局 |
| `app/(editor)/editor/[postId]/page.tsx` | 编辑器页面（动态路由） |
| `app/(editor)/editor/[postId]/not-found.tsx` | 文章不存在时的 404 页面 |
| `app/(editor)/editor/[postId]/loading.tsx` | 编辑器加载状态 |
| `components/editor.tsx` | EditorJS 编辑器核心组件 |
| `styles/editor.css` | 编辑器暗色模式样式 |
| `components/post-item.tsx` | 文章列表项 |
| `components/post-operations.tsx` | 文章操作（编辑/删除） |
| `components/post-create-button.tsx` | 创建文章按钮 |
| `components/shell.tsx` | Dashboard Shell 布局 |
| `components/empty-placeholder.tsx` | 空状态占位符 |
| `components/card-skeleton.tsx` | 骨架屏组件 |

---

## 1. Editor 路由组 — 专属的编辑环境

### 1.1 为什么需要单独的布局？

当用户编辑文章时，应该专注于写作，不需要导航栏、侧边栏等干扰元素。Taxonomy 用 **路由组** `(editor)` 实现了这个设计：

```
app/
├── (marketing)/     ← 营销页面（有完整导航栏）
├── (dashboard)/     ← 仪表盘（有侧边栏）
├── (docs)/          ← 文档（有侧边导航）
└── (editor)/        ← 编辑器（极简布局）
    └── editor/
        └── [postId]/
            ├── page.tsx        ← 编辑器页面
            ├── not-found.tsx   ← 404 页面
            └── loading.tsx     ← 加载状态
```

### 1.2 编辑器布局

```tsx
// app/(editor)/editor/layout.tsx
interface EditorProps {
  children?: React.ReactNode
}

export default function EditorLayout({ children }: EditorProps) {
  return (
    <div className="container mx-auto grid items-start gap-10 py-8">
      {children}
    </div>
  )
}
```

注意这个布局**没有**导航栏（`<MainNav>`）、**没有**侧边栏——只有一个简单的容器。这和 `(dashboard)` 的布局形成鲜明对比，后者包含完整的导航和侧边栏。

### 1.3 编辑器页面（Server Component）

```tsx
// app/(editor)/editor/[postId]/page.tsx
import { notFound, redirect } from "next/navigation"
import { db } from "@/lib/db"
import { getCurrentUser } from "@/lib/session"
import { Editor } from "@/components/editor"

// 从数据库查找用户的文章
async function getPostForUser(postId: Post["id"], userId: User["id"]) {
  return await db.post.findFirst({
    where: {
      id: postId,
      authorId: userId,  // 只能编辑自己的文章
    },
  })
}

export default async function EditorPage({ params }: EditorPageProps) {
  // 1. 检查用户是否登录
  const user = await getCurrentUser()
  if (!user) {
    redirect(authOptions?.pages?.signIn || "/login")
  }

  // 2. 查找文章
  const post = await getPostForUser(params.postId, user.id)

  // 3. 文章不存在就显示 404
  if (!post) {
    notFound()
  }

  // 4. 渲染编辑器组件
  return (
    <Editor
      post={{
        id: post.id,
        title: post.title,
        content: post.content,
        published: post.published,
      }}
    />
  )
}
```

**关键模式：**
- 这是一个 **Server Component**，可以直接调用数据库
- 权限检查在服务器端完成（未登录就重定向）
- 只传必要字段给 `<Editor>`（`Pick<Post, "id" | "title" | "content" | "published">`）

### 1.4 加载状态

```tsx
// app/(editor)/editor/[postId]/loading.tsx
import { Skeleton } from "@/components/ui/skeleton"

export default function EditorLoading() {
  return (
    <div className="grid w-full gap-10">
      <div className="flex w-full items-center justify-between">
        <Skeleton className="h-[38px] w-[90px]" />   {/* 返回按钮 */}
        <Skeleton className="h-[38px] w-[80px]" />   {/* 保存按钮 */}
      </div>
      <div className="mx-auto w-[800px] space-y-6">
        <Skeleton className="h-[50px] w-[120px]" />  {/* 标题 */}
        <Skeleton className="h-[20px] w-2/3" />      {/* 内容行 */}
        <Skeleton className="h-[20px] w-full" />
        <Skeleton className="h-[20px] w-1/3" />
      </div>
    </div>
  )
}
```

当数据库查询进行中时，Next.js 自动显示这个骨架屏，用户不会看到空白页面。

### 1.5 404 页面

```tsx
// app/(editor)/editor/[postId]/not-found.tsx
import { EmptyPlaceholder } from "@/components/empty-placeholder"
import { buttonVariants } from "@/components/ui/button"

export default function NotFound() {
  return (
    <EmptyPlaceholder className="mx-auto max-w-[800px]">
      <EmptyPlaceholder.Icon name="warning" />
      <EmptyPlaceholder.Title>Uh oh! Not Found</EmptyPlaceholder.Title>
      <EmptyPlaceholder.Description>
        This post could not be found. Please try again.
      </EmptyPlaceholder.Description>
      <Link href="/dashboard" className={cn(buttonVariants({ variant: "ghost" }))}>
        Go to Dashboard
      </Link>
    </EmptyPlaceholder>
  )
}
```

当用户访问不存在的文章 ID 时，`notFound()` 会触发这个页面。

---

## 2. EditorJS 编辑器组件

### 2.1 为什么选择 EditorJS？

Taxonomy 没有用 Markdown 编辑器，而是选择了 **EditorJS** —— 一个基于"块"（Block）的富文本编辑器。

**Markdown 编辑器 vs Block 编辑器：**

| 特性 | Markdown 编辑器 | Block 编辑器（EditorJS） |
|------|-----------------|------------------------|
| 内容格式 | 纯文本（`.md`） | 结构化 JSON |
| 用户门槛 | 需要学 Markdown 语法 | 所见即所得 |
| 内容类型 | 主要是文本 | 支持多种 Block 类型 |
| 数据存储 | 字符串 | JSON 对象 |

### 2.2 EditorJS 的内容结构

EditorJS 的内容以 JSON 格式存储，每个"块"是一个对象：

```json
{
  "time": 1706000000000,
  "blocks": [
    {
      "type": "header",
      "data": { "text": "My Post Title", "level": 2 }
    },
    {
      "type": "paragraph",
      "data": { "text": "This is a paragraph with <b>bold</b> text." }
    },
    {
      "type": "list",
      "data": {
        "style": "unordered",
        "items": ["Item 1", "Item 2", "Item 3"]
      }
    },
    {
      "type": "code",
      "data": { "code": "console.log('hello')" }
    }
  ]
}
```

这个 JSON 被存储在数据库的 `post.content` 字段中。

### 2.3 编辑器组件源码解析

`components/editor.tsx` 是整个编辑器的核心。我们分段讲解：

**第一段：导入和类型定义**

```tsx
"use client"  // 编辑器需要浏览器 API（DOM 操作），必须是 Client Component

import * as React from "react"
import EditorJS from "@editorjs/editorjs"
import { useForm } from "react-hook-form"
import TextareaAutosize from "react-textarea-autosize"

import "@/styles/editor.css"  // 编辑器暗色模式样式

interface EditorProps {
  post: Pick<Post, "id" | "title" | "content" | "published">
}
```

**第二段：动态导入 EditorJS 及插件**

```tsx
const initializeEditor = React.useCallback(async () => {
  // 动态导入——只在客户端加载，不在服务器端
  const EditorJS = (await import("@editorjs/editorjs")).default
  const Header = (await import("@editorjs/header")).default
  const Embed = (await import("@editorjs/embed")).default
  const Table = (await import("@editorjs/table")).default
  const List = (await import("@editorjs/list")).default
  const Code = (await import("@editorjs/code")).default
  const LinkTool = (await import("@editorjs/link")).default
  const InlineCode = (await import("@editorjs/inline-code")).default

  // 创建编辑器实例
  if (!ref.current) {
    const editor = new EditorJS({
      holder: "editor",            // 挂载到 id="editor" 的 DOM 元素
      onReady() {
        ref.current = editor       // 保存实例引用
      },
      placeholder: "Type here to write your post...",
      inlineToolbar: true,         // 选中文字时显示行内工具栏
      data: body.content,          // 从数据库加载的初始内容
      tools: {                     // 注册 Block 插件
        header: Header,            // 标题
        linkTool: LinkTool,        // 链接
        list: List,                // 列表
        code: Code,                // 代码块
        inlineCode: InlineCode,    // 行内代码
        table: Table,              // 表格
        embed: Embed,              // 嵌入（YouTube 等）
      },
    })
  }
}, [post])
```

**为什么用动态 `import()` 而不是顶部 `import`？**

EditorJS 需要访问 `document` 和 `window` 等浏览器 API。如果用顶部 `import`，Next.js 在服务端渲染时会报错（服务端没有 `document`）。动态 `import()` 确保代码只在客户端执行。

**第三段：生命周期管理**

```tsx
const [isMounted, setIsMounted] = React.useState<boolean>(false)

// 检测是否在客户端
React.useEffect(() => {
  if (typeof window !== "undefined") {
    setIsMounted(true)
  }
}, [])

// isMounted 变为 true 后初始化编辑器
React.useEffect(() => {
  if (isMounted) {
    initializeEditor()

    // 清理函数：组件卸载时销毁编辑器
    return () => {
      ref.current?.destroy()
      ref.current = undefined
    }
  }
}, [isMounted, initializeEditor])

// 在编辑器初始化前不渲染任何内容
if (!isMounted) {
  return null
}
```

**生命周期流程：**

```
组件挂载
    ↓
useEffect → setIsMounted(true)
    ↓
useEffect（依赖 isMounted）→ initializeEditor()
    ↓
动态 import EditorJS + 插件
    ↓
new EditorJS({ holder: "editor", ... })
    ↓
编辑器就绪，用户可以编辑
    ↓
用户离开页面
    ↓
清理函数 → ref.current.destroy()
```

**第四段：保存功能**

```tsx
async function onSubmit(data: FormData) {
  setIsSaving(true)

  // 从编辑器获取当前内容（JSON 格式）
  const blocks = await ref.current?.save()

  // 调用 API 保存到数据库
  const response = await fetch(`/api/posts/${post.id}`, {
    method: "PATCH",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      title: data.title,
      content: blocks,   // EditorJS 的 JSON 数据
    }),
  })

  setIsSaving(false)

  if (!response?.ok) {
    return toast({
      title: "Something went wrong.",
      description: "Your post was not saved. Please try again.",
      variant: "destructive",
    })
  }

  router.refresh()  // 刷新服务器数据缓存
  return toast({ description: "Your post has been saved." })
}
```

**第五段：UI 渲染**

```tsx
return (
  <form onSubmit={handleSubmit(onSubmit)}>
    <div className="grid w-full gap-10">
      {/* 顶部工具栏 */}
      <div className="flex w-full items-center justify-between">
        {/* 左侧：返回按钮 + 发布状态 */}
        <div className="flex items-center space-x-10">
          <Link href="/dashboard" className={cn(buttonVariants({ variant: "ghost" }))}>
            <Icons.chevronLeft className="mr-2 h-4 w-4" />
            Back
          </Link>
          <p className="text-sm text-muted-foreground">
            {post.published ? "Published" : "Draft"}
          </p>
        </div>
        {/* 右侧：保存按钮 */}
        <button type="submit" className={cn(buttonVariants())}>
          {isSaving && <Icons.spinner className="mr-2 h-4 w-4 animate-spin" />}
          <span>Save</span>
        </button>
      </div>

      {/* 编辑区域 */}
      <div className="prose prose-stone mx-auto w-[800px] dark:prose-invert">
        <TextareaAutosize
          autoFocus
          id="title"
          defaultValue={post.title}
          placeholder="Post title"
          className="w-full resize-none appearance-none overflow-hidden bg-transparent text-5xl font-bold focus:outline-none"
          {...register("title")}
        />
        <div id="editor" className="min-h-[500px]" />
        <p className="text-sm text-gray-500">
          Use <kbd>Tab</kbd> to open the command menu.
        </p>
      </div>
    </div>
  </form>
)
```

**UI 结构：**

```
┌──────────────────────────────────────────────┐
│  ← Back    Draft                      [Save] │  ← 顶部工具栏
├──────────────────────────────────────────────┤
│                                              │
│  Post title                                  │  ← TextareaAutosize
│  ────────────────────────                    │
│                                              │
│  Type here to write your post...             │  ← EditorJS 挂载区域
│                                              │  （id="editor"）
│                                              │
│  Use [Tab] to open the command menu.         │
│                                              │
└──────────────────────────────────────────────┘
```

---

## 3. 完整用户流程

让我们追踪用户从"文章列表"到"编辑保存"的完整流程。

### 3.1 创建文章 — `PostCreateButton`

```tsx
// components/post-create-button.tsx（简化版）
"use client"

export function PostCreateButton({ className, variant, ...props }) {
  const router = useRouter()
  const [isLoading, setIsLoading] = React.useState(false)

  async function onClick() {
    setIsLoading(true)

    // 调用 API 创建一篇空文章
    const response = await fetch("/api/posts", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ title: "Untitled Post" }),
    })

    setIsLoading(false)

    // 402 表示免费用户已达文章数量上限
    if (response.status === 402) {
      return toast({
        title: "Limit of 3 posts reached.",
        description: "Please upgrade to the PRO plan.",
        variant: "destructive",
      })
    }

    const post = await response.json()

    router.refresh()                    // 刷新文章列表
    router.push(`/editor/${post.id}`)   // 跳转到编辑器
  }

  return (
    <button onClick={onClick} disabled={isLoading} {...props}>
      {isLoading ? <Icons.spinner /> : <Icons.add />}
      New post
    </button>
  )
}
```

**亮点：** 免费用户最多只能创建 3 篇文章。如果 API 返回 402（Payment Required），提示用户升级 PRO 计划。这就是第 9 章支付系统的实际应用。

### 3.2 文章列表 — `PostItem`

```tsx
// components/post-item.tsx
export function PostItem({ post }) {
  return (
    <div className="flex items-center justify-between p-4">
      <div className="grid gap-1">
        {/* 标题链接到编辑器 */}
        <Link href={`/editor/${post.id}`} className="font-semibold hover:underline">
          {post.title}
        </Link>
        <p className="text-sm text-muted-foreground">
          {formatDate(post.createdAt?.toDateString())}
        </p>
      </div>
      {/* 右侧操作菜单 */}
      <PostOperations post={{ id: post.id, title: post.title }} />
    </div>
  )
}

// 骨架屏（加载中显示）
PostItem.Skeleton = function PostItemSkeleton() {
  return (
    <div className="p-4">
      <div className="space-y-3">
        <Skeleton className="h-5 w-2/5" />
        <Skeleton className="h-4 w-4/5" />
      </div>
    </div>
  )
}
```

**设计模式：** `PostItem.Skeleton` 是把加载骨架屏作为组件的静态属性。这样使用时非常自然：

```tsx
{isLoading ? <PostItem.Skeleton /> : <PostItem post={post} />}
```

### 3.3 文章操作 — `PostOperations`

```tsx
// components/post-operations.tsx（简化版）
"use client"

export function PostOperations({ post }) {
  const [showDeleteAlert, setShowDeleteAlert] = React.useState(false)
  const [isDeleteLoading, setIsDeleteLoading] = React.useState(false)

  return (
    <>
      {/* 下拉菜单 */}
      <DropdownMenu>
        <DropdownMenuTrigger>
          <Icons.ellipsis className="h-4 w-4" />
        </DropdownMenuTrigger>
        <DropdownMenuContent align="end">
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

      {/* 删除确认弹窗 */}
      <AlertDialog open={showDeleteAlert} onOpenChange={setShowDeleteAlert}>
        <AlertDialogContent>
          <AlertDialogTitle>
            Are you sure you want to delete this post?
          </AlertDialogTitle>
          <AlertDialogDescription>
            This action cannot be undone.
          </AlertDialogDescription>
          <AlertDialogFooter>
            <AlertDialogCancel>Cancel</AlertDialogCancel>
            <AlertDialogAction
              onClick={async (event) => {
                event.preventDefault()
                setIsDeleteLoading(true)
                await deletePost(post.id)  // DELETE /api/posts/[postId]
                setIsDeleteLoading(false)
                setShowDeleteAlert(false)
                router.refresh()
              }}
            >
              {isDeleteLoading ? <Icons.spinner /> : <Icons.trash />}
              Delete
            </AlertDialogAction>
          </AlertDialogFooter>
        </AlertDialogContent>
      </AlertDialog>
    </>
  )
}
```

**设计模式：** 危险操作（删除）使用 `AlertDialog` 要求用户确认。按钮颜色用 `destructive`（红色）来提醒用户。

### 3.4 完整流程图

```
Dashboard 文章列表
┌──────────────────────────────────────┐
│  [+ New post]                        │  ← PostCreateButton
│                                      │
│  ┌─────────────────────────────────┐│
│  │ My First Post        ⋯ │  ← PostItem + PostOperations
│  │ Jan 15, 2024            ├──→ Edit │
│  │                         ├──→ Delete │
│  └─────────────────────────────────┘│
│  ┌─────────────────────────────────┐│
│  │ Untitled Post           ⋯ ││
│  │ Jan 16, 2024                    ││
│  └─────────────────────────────────┘│
└──────────────────────────────────────┘
        │
        │ 点击标题 or 点击 Edit
        ↓
Editor 页面
┌──────────────────────────────────────┐
│  ← Back    Draft             [Save]  │
│                                      │
│  Post Title                          │
│  ─────────────                       │
│  Content blocks...                   │
│                                      │
└──────────────────────────────────────┘
        │
        │ 点击 Save
        ↓
PATCH /api/posts/[postId]
        │
        │ 200 OK
        ↓
Toast: "Your post has been saved."
```

---

## 4. 关键 UI 组件模式

### 4.1 `DashboardShell` — 布局容器

```tsx
// components/shell.tsx
export function DashboardShell({ children, className, ...props }) {
  return (
    <div className={cn("grid items-start gap-8", className)} {...props}>
      {children}
    </div>
  )
}
```

看起来很简单，但它的设计理念重要：**布局关注点分离**。Dashboard 的每个页面都用 `DashboardShell` 包裹，确保一致的间距和对齐：

```tsx
// 使用示例
<DashboardShell>
  <DashboardHeader heading="Posts" text="Create and manage posts.">
    <PostCreateButton />
  </DashboardHeader>
  <div>
    {posts.map((post) => <PostItem key={post.id} post={post} />)}
  </div>
</DashboardShell>
```

### 4.2 `EmptyPlaceholder` — 空状态组件

当列表为空时，不应该显示空白——应该引导用户操作：

```tsx
// components/empty-placeholder.tsx — 复合组件模式
export function EmptyPlaceholder({ className, children, ...props }) {
  return (
    <div className={cn(
      "flex min-h-[400px] flex-col items-center justify-center",
      "rounded-md border border-dashed p-8 text-center",
      className
    )} {...props}>
      <div className="mx-auto flex max-w-[420px] flex-col items-center justify-center text-center">
        {children}
      </div>
    </div>
  )
}

// 子组件挂在父组件上
EmptyPlaceholder.Icon = function ({ name, className }) {
  const Icon = Icons[name]
  return (
    <div className="flex h-20 w-20 items-center justify-center rounded-full bg-muted">
      <Icon className={cn("h-10 w-10", className)} />
    </div>
  )
}

EmptyPlaceholder.Title = function ({ className, ...props }) {
  return <h2 className={cn("mt-6 text-xl font-semibold", className)} {...props} />
}

EmptyPlaceholder.Description = function ({ className, ...props }) {
  return <p className={cn("mb-8 mt-2 text-center text-sm text-muted-foreground", className)} {...props} />
}
```

**使用示例：**

```tsx
<EmptyPlaceholder>
  <EmptyPlaceholder.Icon name="post" />
  <EmptyPlaceholder.Title>No posts created</EmptyPlaceholder.Title>
  <EmptyPlaceholder.Description>
    You don&apos;t have any posts yet. Start creating content.
  </EmptyPlaceholder.Description>
  <PostCreateButton variant="outline" />
</EmptyPlaceholder>
```

**设计模式：复合组件（Compound Component）**

把 `Icon`、`Title`、`Description` 挂在 `EmptyPlaceholder` 上作为子组件。这样它们在语义上属于 `EmptyPlaceholder`，但又可以灵活组合。

### 4.3 `CardSkeleton` — 骨架屏

```tsx
// components/card-skeleton.tsx
export function CardSkeleton() {
  return (
    <Card>
      <CardHeader className="gap-2">
        <Skeleton className="h-5 w-1/5" />    {/* 标题位置 */}
        <Skeleton className="h-4 w-4/5" />    {/* 描述位置 */}
      </CardHeader>
      <CardContent className="h-10" />          {/* 内容区 */}
      <CardFooter>
        <Skeleton className="h-8 w-[120px]" /> {/* 按钮位置 */}
      </CardFooter>
    </Card>
  )
}
```

骨架屏用灰色矩形"占位"，模拟内容的布局结构。这比显示一个旋转的 loading 图标要好得多，因为用户能提前知道页面的大致结构。

---

## 5. 技术架构总结

```
┌─────────────────────────────────────────────────────┐
│                    用户操作流程                       │
│                                                      │
│  Dashboard             Editor              API       │
│  ┌─────────┐         ┌──────────┐       ┌────────┐ │
│  │ Post    │──点击──→│ Editor   │──保存→│ PATCH  │ │
│  │ Item    │         │ Component│       │/api/   │ │
│  │         │         │          │       │posts/  │ │
│  │ Post    │         │ EditorJS │       │[id]   │ │
│  │ Create  │──创建──→│          │       │        │ │
│  │ Button  │  POST   │ (Client) │       │(Server)│ │
│  │         │ /api/   │          │       │        │ │
│  │ Post    │ posts   └──────────┘       │ DELETE │ │
│  │ Ops     │──删除──────────────────────→│/api/   │ │
│  │         │  DELETE                     │posts/  │ │
│  └─────────┘                            │[id]   │ │
│                                          └────────┘ │
│                                              ↕      │
│                                          ┌────────┐ │
│                                          │ Prisma │ │
│                                          │  DB    │ │
│                                          └────────┘ │
└─────────────────────────────────────────────────────┘
```

---

## 小练习

### 练习 1：追踪编辑器初始化

1. 打开 `components/editor.tsx`
2. 标记出以下步骤发生的顺序：
   - `isMounted` 变为 `true`
   - `initializeEditor()` 被调用
   - 动态 `import()` 加载 EditorJS
   - `new EditorJS(...)` 创建实例
   - 编辑器渲染到页面上
3. 思考：为什么不能把 `initializeEditor()` 直接放在组件顶层调用？

### 练习 2：理解完整用户流程

1. 打开 `components/post-create-button.tsx`
2. 找到 API 调用的代码，确认：
   - 请求方法是什么？（POST）
   - 请求体包含什么？（`{ title: "Untitled Post" }`）
   - 创建成功后跳转到哪里？（`/editor/${post.id}`）
   - 如果返回 402 会怎样？（提示升级 PRO）
3. 打开 `components/post-operations.tsx`，找到删除文章的 API 调用

### 练习 3：分析组件模式

1. 比较 `DashboardShell`、`EmptyPlaceholder`、`CardSkeleton` 三个组件
2. 它们分别解决什么问题？
3. `EmptyPlaceholder` 的复合组件模式（`EmptyPlaceholder.Icon`、`.Title`、`.Description`）和普通 props 传值相比，有什么优势？

---

## 本章小结

- Editor 路由组 `(editor)` 提供最小化布局，让用户专注写作
- EditorJS 通过动态 `import()` 加载，避免 SSR 问题
- 编辑器内容以 JSON 格式存储，通过 PATCH API 保存
- 文章的完整生命周期：创建（POST）→ 编辑（PATCH）→ 删除（DELETE）
- Dashboard 使用多个 UI 模式：Shell 布局、空状态占位、骨架屏、复合组件

下一章我们将学习支付系统——怎样让用户付费升级。
