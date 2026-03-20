# Day 08.5 — 文章编辑器（Post Editor）

> 对应章节：[Chapter 8.5: 文章编辑器](../chapter-08.5-post-editor.md)

---

## 一、今日学习目标

1. **理解 EditorJS 的集成方式**：知道为什么要动态导入、怎样初始化和销毁编辑器
2. **能追踪文章的完整生命周期**：从创建到编辑到保存到删除
3. **理解 Dashboard 中的关键 UI 组件模式**：Shell、EmptyPlaceholder、Skeleton

---

## 二、前置知识速补

> 今天的内容综合了前面多天的知识。如果你已经掌握，可以跳过。

### 2.1 动态 [import()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/import) — 按需加载

```javascript
// 普通 import — 在文件顶部，所有代码一起加载
import EditorJS from "@editorjs/editorjs"

// 动态 import() — 在运行时按需加载
const EditorJS = (await import("@editorjs/editorjs")).default
```

**区别：**
- 普通 `import`：代码在模块加载时就执行（如果在 SSR 中会报错）
- 动态 `import()`：代码在调用时才加载（可以控制只在客户端执行）

**为什么要学这个？** EditorJS 依赖浏览器 DOM API，不能在服务端加载。

### 2.2 [useCallback](https://react.dev/reference/react/useCallback) — 缓存函数引用

```javascript
// 每次组件渲染都创建新函数（可能导致 useEffect 反复触发）
const initEditor = () => { /* ... */ }

// useCallback — 只在依赖变化时创建新函数
const initEditor = React.useCallback(() => {
  /* ... */
}, [依赖项])
```

**为什么要学这个？** `components/editor.tsx` 中的 `initializeEditor` 使用了 `useCallback`。

### 2.3 [useRef](https://react.dev/reference/react/useRef) — 保存可变引用

```javascript
const ref = React.useRef(null)
// ref.current 可以被赋值和读取，但变化不会触发重新渲染

ref.current = someValue   // 赋值
console.log(ref.current)  // 读取
```

**为什么要学这个？** 编辑器实例保存在 `ref.current` 中，这样保存按钮可以调用 `ref.current.save()` 获取内容。

### 2.4 [react-hook-form](https://react-hook-form.com/) 基础

```tsx
const { register, handleSubmit } = useForm()

// register — 把输入框注册到表单系统
<input {...register("title")} />

// handleSubmit — 提交时自动收集和验证数据
<form onSubmit={handleSubmit((data) => {
  console.log(data.title)  // 用户输入的标题
})}>
```

**为什么要学这个？** 编辑器的标题输入框使用了 `react-hook-form`。

### 2.5 [router.refresh()` vs `router.push()](https://nextjs.org/docs/app/api-reference/functions/use-router)

```javascript
const router = useRouter()

router.push("/dashboard")    // 导航到新页面
router.refresh()             // 重新获取当前页面的服务器数据（不刷新页面）
```

**为什么要学这个？** 保存/删除文章后，用 `router.refresh()` 让页面显示最新数据。

---

## 三、核心概念清单

| 概念 | 英文 | 一句话解释 | 在项目中对应的文件 |
|------|------|-----------|-------------------|
| 路由组 | Route Group | `(editor)` 括号包裹，不影响 URL 但有独立布局 | `app/(editor)/` |
| 动态导入 | Dynamic Import | `await import()` 按需加载模块 | `components/editor.tsx` |
| Block 编辑器 | Block Editor | 以"块"为单位的富文本编辑，内容是 JSON | [EditorJS](https://editorjs.io/) |
| 复合组件 | Compound Component | 子组件挂在父组件上（如 `EmptyPlaceholder.Title`） | `components/empty-placeholder.tsx` |
| 骨架屏 | Skeleton | 加载中用灰色矩形模拟内容布局 | `components/card-skeleton.tsx` |
| 空状态 | Empty State | 列表为空时引导用户操作的占位页 | `components/empty-placeholder.tsx` |

---

## 四、源码精读

### 4.1 Editor 路由组布局

> 文件路径：`/app/(editor)/editor/layout.tsx`

```tsx
export default function EditorLayout({ children }) {
  return (
    <div className="container mx-auto grid items-start gap-10 py-8">
      {children}
    </div>
  )
}
```

**对比其他布局：**
- `(marketing)/layout.tsx`：有导航栏 `<MainNav>` + 底部 footer
- `(dashboard)/layout.tsx`：有导航栏 + 侧边栏
- `(editor)/layout.tsx`：什么都没有，只有一个容器——让用户专注写作

### 4.2 编辑器页面 — Server Component

> 文件路径：`/app/(editor)/editor/[postId]/page.tsx`

```tsx
export default async function EditorPage({ params }) {
  // 1. 验证用户身份（Server Component 可以直接查数据库）
  const user = await getCurrentUser()
  if (!user) redirect("/login")

  // 2. 查询文章（只能编辑自己的）
  const post = await db.post.findFirst({
    where: { id: params.postId, authorId: user.id },
  })

  // 3. 不存在就 404
  if (!post) notFound()

  // 4. 把数据传给 Client Component
  return <Editor post={{ id: post.id, title: post.title, content: post.content, published: post.published }} />
}
```

**注意：** 页面本身是 Server Component（异步，可以查数据库），但它渲染的 `<Editor>` 是 Client Component（有 `"use client"`，可以用浏览器 API）。

### 4.3 EditorJS 编辑器组件

> 文件路径：`/components/editor.tsx`

这是本章最核心的文件。我们分四步理解：

**Step 1：为什么需要 `isMounted` 检查**

```tsx
const [isMounted, setIsMounted] = React.useState(false)

React.useEffect(() => {
  if (typeof window !== "undefined") {
    setIsMounted(true)
  }
}, [])

if (!isMounted) return null  // 服务端不渲染任何内容
```

EditorJS 需要 `document.getElementById("editor")` 来挂载。服务端没有 `document`，所以必须等到客户端才开始初始化。

**Step 2：动态导入所有 EditorJS 模块**

```tsx
const initializeEditor = React.useCallback(async () => {
  const EditorJS = (await import("@editorjs/editorjs")).default
  const Header = (await import("@editorjs/header")).default
  const Table = (await import("@editorjs/table")).default
  const List = (await import("@editorjs/list")).default
  const Code = (await import("@editorjs/code")).default
  // ... 更多插件
```

每个 `await import()` 都是异步的——浏览器会按需下载这些 JS 文件。

**Step 3：创建 EditorJS 实例**

```tsx
  if (!ref.current) {  // 防止重复创建
    const editor = new EditorJS({
      holder: "editor",        // 挂载到 DOM
      placeholder: "Type here to write your post...",
      data: body.content,      // 初始内容（从数据库加载）
      tools: {                 // 注册插件
        header: Header,
        list: List,
        code: Code,
        table: Table,
        // ...
      },
    })
  }
}, [post])
```

**Step 4：清理（组件卸载时销毁编辑器）**

```tsx
React.useEffect(() => {
  if (isMounted) {
    initializeEditor()
    return () => {
      ref.current?.destroy()    // 销毁 EditorJS 实例
      ref.current = undefined   // 清空引用
    }
  }
}, [isMounted, initializeEditor])
```

**完整生命周期图：**

```
组件挂载
    │
    ↓
useEffect: setIsMounted(true)
    │
    ↓
useEffect: initializeEditor()
    │
    ├── await import("@editorjs/editorjs")
    ├── await import("@editorjs/header")
    ├── await import("@editorjs/list")
    ├── ... 更多插件
    │
    ↓
new EditorJS({ holder: "editor", data: 初始内容, tools: {...} })
    │
    ↓
onReady() → ref.current = editor 实例
    │
    ↓
用户编辑内容...
    │
    ↓
点击 Save → ref.current.save() → 获取 JSON → PATCH /api/posts/[id]
    │
    ↓
用户离开页面
    │
    ↓
清理函数: ref.current.destroy()
```

### 4.4 文章操作组件

> 文件路径：`/components/post-operations.tsx`

```tsx
// 下拉菜单 + 删除确认弹窗的组合

<DropdownMenu>                        {/* Radix UI 下拉菜单 */}
  <DropdownMenuTrigger>               {/* 触发按钮（三个点图标） */}
  <DropdownMenuContent>
    <DropdownMenuItem>Edit</DropdownMenuItem>
    <DropdownMenuItem>Delete</DropdownMenuItem>  {/* 点击打开确认弹窗 */}
  </DropdownMenuContent>
</DropdownMenu>

<AlertDialog>                         {/* Radix UI 确认弹窗 */}
  <AlertDialogContent>
    "Are you sure you want to delete this post?"
    <AlertDialogCancel>Cancel</AlertDialogCancel>
    <AlertDialogAction onClick={deletePost}>Delete</AlertDialogAction>
  </AlertDialogContent>
</AlertDialog>
```

**删除流程：** 点击 "⋯" → 选择 Delete → 弹出确认 → 确认 → DELETE /api/posts/[id] → router.refresh()

### 4.5 EmptyPlaceholder 复合组件模式

> 文件路径：`/components/empty-placeholder.tsx`

```tsx
// 父组件
function EmptyPlaceholder({ children }) { ... }

// 子组件挂在父组件上
EmptyPlaceholder.Icon = function ({ name }) { ... }
EmptyPlaceholder.Title = function ({ ...props }) { ... }
EmptyPlaceholder.Description = function ({ ...props }) { ... }

// 使用时像"积木"一样组合
<EmptyPlaceholder>
  <EmptyPlaceholder.Icon name="post" />
  <EmptyPlaceholder.Title>No posts</EmptyPlaceholder.Title>
  <EmptyPlaceholder.Description>Create your first post.</EmptyPlaceholder.Description>
  <PostCreateButton />
</EmptyPlaceholder>
```

**为什么用这种模式而不是 props？**

```tsx
// Props 方式 — 不灵活，不能插入自定义按钮
<EmptyPlaceholder icon="post" title="No posts" description="Create..." />

// 复合组件方式 — 灵活，可以自由组合子内容
<EmptyPlaceholder>
  <EmptyPlaceholder.Icon name="post" />
  <EmptyPlaceholder.Title>No posts</EmptyPlaceholder.Title>
  <PostCreateButton />   {/* 可以插入任何内容 */}
</EmptyPlaceholder>
```

---

## 五、概念图解：文章完整生命周期

```
┌──────────────────────────────────────────────────┐
│                 Dashboard 页面                    │
│                                                   │
│  [+ New post]  ──→  POST /api/posts              │
│                     body: { title: "Untitled" }   │
│                         │                         │
│                         ↓                         │
│                     创建空文章                     │
│                         │                         │
│                         ↓                         │
│                 router.push(/editor/abc123)        │
│                                                   │
├──────────────────────────────────────────────────┤
│                 Editor 页面                       │
│                                                   │
│  ← Back    Draft               [Save]            │
│                                                   │
│  Post Title                                       │
│  ───────────                                      │
│  EditorJS 内容块...                               │
│                                                   │
│  Save ──→  PATCH /api/posts/abc123               │
│            body: { title: "...", content: {...} } │
│                                                   │
├──────────────────────────────────────────────────┤
│                 删除文章                           │
│                                                   │
│  PostOperations ⋯ → Delete                        │
│       ↓                                           │
│  "Are you sure?" → 确认                           │
│       ↓                                           │
│  DELETE /api/posts/abc123                         │
│       ↓                                           │
│  router.refresh() → 列表更新                      │
│                                                   │
└──────────────────────────────────────────────────┘
```

---

## 六、动手练习

### 练习 1：追踪编辑器初始化流程（30 分钟）

**目标：** 理解 EditorJS 从加载到可用的完整流程。

**步骤：**

1. 打开 `components/editor.tsx`
2. 找到以下代码，标注它们的执行顺序（1-6）：
   - `setIsMounted(true)`
   - `const EditorJS = (await import(...)).default`
   - `new EditorJS({ holder: "editor", ... })`
   - `ref.current = editor`（在 `onReady` 回调中）
   - `if (!isMounted) return null`
   - `ref.current?.destroy()`

3. 思考：如果去掉 `isMounted` 检查，直接在 `useEffect` 中调用 `initializeEditor()`，会发生什么？

**排错提示：**
- 注意两个 `useEffect` 的依赖数组不同
- `useCallback` 的依赖是 `[post]`，意味着 `post` 变化时函数会重新创建

### 练习 2：理解文章创建流程（30 分钟）

**目标：** 追踪从点击 "New post" 到进入编辑器的完整代码路径。

**步骤：**

1. 打开 `components/post-create-button.tsx`
2. 找到 `onClick` 函数，逐行阅读：
   - API 请求发到哪里？
   - 请求体是什么？
   - 如果返回 402 会怎样？
   - 创建成功后做了哪两件事？
3. 思考：`router.refresh()` 和 `router.push()` 的顺序能不能反过来？为什么？

**排错提示：**
- 402 状态码表示 "Payment Required"——免费用户的文章数量限制
- `router.refresh()` 在 `push()` 前执行，是为了确保从编辑器返回时列表是最新的

### 练习 3：分析 UI 组件模式（30 分钟）

**目标：** 理解 Dashboard 中不同 UI 模式的使用场景。

**步骤：**

1. 阅读 `components/shell.tsx`——它做了什么？为什么不直接写 `<div className="grid gap-8">`？
2. 阅读 `components/empty-placeholder.tsx`——复合组件模式的好处是什么？
3. 阅读 `components/card-skeleton.tsx`——骨架屏和 loading spinner 相比有什么优势？
4. 阅读 `components/post-item.tsx`——`PostItem.Skeleton` 这种写法有什么好处？

**排错提示：**
- Shell 组件虽然简单，但它的价值在于一致性——所有 Dashboard 页面用同一个容器
- 复合组件让调用者可以自由组合子内容

---

## 七、自检问题

### 问题 1：为什么 EditorJS 不能用普通 `import` 导入，而要用 `await import()`？

<details>
<summary>查看答案</summary>

EditorJS 依赖浏览器的 DOM API（如 `document`、`window`）。普通 `import` 在模块加载时就执行，而 Next.js 的 Server Components 在服务端运行，没有 DOM。使用 `await import()` 可以确保代码只在客户端执行时才加载。

</details>

### 问题 2：Editor 组件中的 `ref.current?.destroy()` 是在什么时候调用的？为什么需要这一步？

<details>
<summary>查看答案</summary>

在 `useEffect` 的清理函数中调用，即组件卸载时（用户离开编辑器页面时）。需要这一步是因为 EditorJS 创建了 DOM 元素和事件监听器，如果不销毁会造成内存泄漏。这是 React 中使用第三方 DOM 库的标准模式：在 `useEffect` 中初始化，在清理函数中销毁。

</details>

### 问题 3：`PostCreateButton` 中为什么要处理 HTTP 402 状态码？

<details>
<summary>查看答案</summary>

402 是 "Payment Required" 状态码。当免费用户已经创建了 3 篇文章（达到免费额度上限），API 会返回 402。前端收到 402 后弹出 Toast 提示用户升级到 PRO 计划。这是 Chapter 9（支付系统）和 Chapter 8（API 层）的实际应用。

</details>

### 问题 4：`EmptyPlaceholder.Icon`、`.Title`、`.Description` 这种复合组件模式和直接传 props 相比，有什么优势？

<details>
<summary>查看答案</summary>

复合组件模式更灵活：(1) 可以自由控制子组件的顺序；(2) 可以插入任意额外内容（如按钮）；(3) 每个子组件可以接收自己的 props。Props 方式则固定了渲染结构，不容易添加自定义内容。比如，复合组件模式允许在 Title 和 Description 之间插入一个 `<PostCreateButton>`，这在 props 方式中很难实现。

</details>

### 问题 5：EditorJS 的内容以什么格式存储？和 MDX（Chapter 5）的内容存储有什么区别？

<details>
<summary>查看答案</summary>

EditorJS 的内容以 JSON 格式存储在数据库中，每个"块"是一个 `{ type, data }` 对象。MDX 内容以 Markdown 文本文件（`.mdx`）形式存储在 `content/` 目录中。区别在于：EditorJS 的 JSON 是结构化数据，适合程序处理和可视化编辑；MDX 是文本格式，适合版本控制和开发者编写。Taxonomy 中这两种方式服务不同场景——MDX 用于营销页面和文档，EditorJS 用于用户创建的文章。

</details>

---

## 八、预计时间分配

| 环节 | 预计时间 | 说明 |
|------|---------|------|
| 前置知识速补 | 25 分钟 | 动态 import、useCallback、useRef、react-hook-form |
| 核心概念 + 源码精读 | 75 分钟 | 重点读 editor.tsx 和 post-operations.tsx |
| 练习 1：追踪编辑器初始化 | 30 分钟 | 理解异步初始化流程 |
| 练习 2：理解文章创建流程 | 30 分钟 | 追踪 API 调用链 |
| 练习 3：分析 UI 组件模式 | 30 分钟 | 比较不同组件模式 |
| 自检问题 + 笔记整理 | 20 分钟 | 回顾全天所学 |
| **合计** | **约 3.5 小时** | |

建议休息安排：源码精读后休息一次（这部分信息密度较大），练习之间可以适当休息。

---

## 九、常见踩坑点

### 踩坑 1：EditorJS 在开发模式下报 "holder is undefined"

**原因：** 编辑器尝试在 DOM 元素存在之前初始化。

**解决：** 确保 `isMounted` 为 `true` 之后再调用 `initializeEditor()`，并且 `id="editor"` 的 div 已经渲染。

### 踩坑 2：保存后内容没有更新

**原因：** 忘记调用 `router.refresh()` 或 API 返回了错误但没有提示。

**解决：** 检查 API 响应状态码，确保 `response.ok` 为 `true`。`router.refresh()` 是必须的，它告诉 Next.js 重新获取服务器数据。

### 踩坑 3：React 严格模式下 EditorJS 初始化两次

**症状：** 开发模式下编辑器出现两个，或者控制台报错。

**原因：** React 严格模式在开发环境会把 useEffect 执行两次来检测问题。

**解决：** `if (!ref.current)` 的检查就是为了防止这个问题——如果已经创建了实例就不再创建。

### 踩坑 4：删除文章后列表没刷新

**原因：** `router.refresh()` 没有在删除成功后调用。

**解决：** 确保在 `deletePost` 成功后调用 `router.refresh()`。

---

## 十、延伸阅读

1. **EditorJS 官方文档**：https://editorjs.io/ — 了解所有可用的 Block 类型和配置
2. **Next.js 动态导入**：https://nextjs.org/docs/app/building-your-application/optimizing/lazy-loading
3. **React useRef**：https://react.dev/reference/react/useRef
4. **react-hook-form**：https://react-hook-form.com/ — 表单管理库
5. **Radix UI AlertDialog**：https://www.radix-ui.com/docs/primitives/components/alert-dialog — 确认弹窗
6. **复合组件模式**：搜索 "React Compound Component Pattern" 了解更多
