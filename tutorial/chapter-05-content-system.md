# 第5章：Markdown 内容系统 — 写 Markdown 怎么就变成了网页？

## 学习目标

- 理解 MDX（Markdown + JSX）的概念
- 理解 Contentlayer 如何把 MDX 文件转成 TypeScript 可用的数据
- 理解内容模型定义（Doc、Post、Guide、Author、Page）
- 理解 rehype/remark 插件如何增强 Markdown
- 理解 MDX 自定义组件

## 涉及文件

| 文件 | 说明 |
|------|------|
| `contentlayer.config.js` | 内容类型定义和 MDX 插件配置 |
| `content/blog/*.mdx` | 博客文章 |
| `content/docs/*.mdx` | 文档 |
| `content/authors/*.mdx` | 作者信息 |
| `app/(marketing)/blog/[...slug]/page.tsx` | 博客渲染页 |
| `app/(docs)/docs/[[...slug]]/page.tsx` | 文档渲染页 |
| `components/mdx-components.tsx` | MDX 自定义组件映射 |
| `components/callout.tsx` | 提示框组件 |

---

## 1. MDX 是什么？

你可能知道 Markdown——用简单的符号写文档（`# 标题`、`**粗体**`、`- 列表`）。

**MDX = Markdown + JSX**。也就是说，你可以在 Markdown 中直接使用 React 组件：

```mdx
# 我的博客文章

这是普通的 Markdown 文本。

<Callout>
  这是一个自定义的提示框组件！
  在普通 Markdown 中是做不到的。
</Callout>

下面是一个卡片组件：
<Card href="/docs/getting-started">
  开始使用
</Card>
```

---

## 2. Contentlayer — 内容管理的"翻译官"

**Contentlayer** 是一个工具，它的工作是：
1. 读取 `content/` 文件夹下的 MDX 文件
2. 验证每个文件的元数据（frontmatter）
3. 将 MDX 编译成 JavaScript
4. 生成 TypeScript 类型定义
5. 输出一个可以 `import` 的数据源

**大白话：** Contentlayer 就是一个"翻译官"，把人类可读的 Markdown 翻译成代码可用的数据。

### 内容文件的结构

一个典型的 MDX 博客文章长这样：

```mdx
---
title: Preview Mode for Headless CMS
description: How to use preview mode with a headless CMS.
date: 2023-04-09
image: /images/blog/blog-post-4.jpg
authors:
  - shadcn
---

This is the blog post content written in Markdown.

## Features

- Feature 1
- Feature 2
```

上面的 `---` 包裹的部分叫 **frontmatter**（前置信息），类似于书的"封面信息"。下面是正文内容。

---

## 3. contentlayer.config.js — 定义内容模型

这个文件告诉 Contentlayer："我有哪些类型的内容？每种内容需要什么字段？"

```javascript
// contentlayer.config.js
import { defineDocumentType, makeSource } from "contentlayer/source-files"
import rehypeAutolinkHeadings from "rehype-autolink-headings"
import rehypePrettyCode from "rehype-pretty-code"
import rehypeSlug from "rehype-slug"
import remarkGfm from "remark-gfm"

// 计算字段 —— 自动从文件路径生成 slug
const computedFields = {
  slug: {
    type: "string",
    // doc._raw.flattenedPath = "blog/my-post"
    // resolve 后 slug = "/blog/my-post"
    resolve: (doc) => `/${doc._raw.flattenedPath}`,
  },
  slugAsParams: {
    type: "string",
    // "blog/my-post" → "my-post"（去掉第一段路径）
    resolve: (doc) => doc._raw.flattenedPath.split("/").slice(1).join("/"),
  },
}

// 📄 文档类型
export const Doc = defineDocumentType(() => ({
  name: "Doc",                            // 类型名称
  filePathPattern: `docs/**/*.mdx`,       // 匹配 content/docs/ 下的所有 .mdx 文件
  contentType: "mdx",                     // 内容格式
  fields: {
    title:       { type: "string", required: true },  // 标题（必填）
    description: { type: "string" },                  // 描述（可选）
    published:   { type: "boolean", default: true },  // 是否发布
  },
  computedFields,
}))

// 📝 博客文章类型
export const Post = defineDocumentType(() => ({
  name: "Post",
  filePathPattern: `blog/**/*.mdx`,
  contentType: "mdx",
  fields: {
    title:       { type: "string", required: true },
    description: { type: "string" },
    date:        { type: "date", required: true },     // 发布日期
    published:   { type: "boolean", default: true },
    image:       { type: "string", required: true },   // 封面图
    authors: {                                          // 作者列表
      type: "list",
      of: { type: "string" },
      required: true,
    },
  },
  computedFields,
}))

// 👤 作者类型
export const Author = defineDocumentType(() => ({
  name: "Author",
  filePathPattern: `authors/**/*.mdx`,
  contentType: "mdx",
  fields: {
    title:   { type: "string", required: true },   // 作者名
    avatar:  { type: "string", required: true },   // 头像
    twitter: { type: "string", required: true },   // Twitter 链接
  },
  computedFields,
}))

// 📑 静态页面类型（如隐私政策、使用条款）
export const Page = defineDocumentType(() => ({
  name: "Page",
  filePathPattern: `pages/**/*.mdx`,
  contentType: "mdx",
  fields: {
    title:       { type: "string", required: true },
    description: { type: "string" },
  },
  computedFields,
}))

// 导出数据源配置
export default makeSource({
  contentDirPath: "./content",                             // 内容文件夹
  documentTypes: [Page, Doc, Guide, Post, Author],        // 所有内容类型
  mdx: {
    remarkPlugins: [remarkGfm],                            // GitHub 风格 Markdown
    rehypePlugins: [
      rehypeSlug,                                          // 给标题加 id
      [rehypePrettyCode, { theme: "github-dark" }],       // 代码语法高亮
      [rehypeAutolinkHeadings, {                           // 标题旁加锚点链接
        properties: { className: ["subheading-anchor"] },
      }],
    ],
  },
})
```

### 插件解释

| 插件 | 作用 | 效果 |
|------|------|------|
| `remarkGfm` | GitHub 风格 Markdown | 支持表格、任务列表、删除线等 |
| `rehypeSlug` | 给标题添加 id 属性 | `## Features` → `<h2 id="features">` |
| `rehypePrettyCode` | 代码语法高亮 | 代码块有颜色 |
| `rehypeAutolinkHeadings` | 标题旁添加锚点链接 | 点击标题旁的 # 可以跳转 |

**Remark vs Rehype：**
- **Remark** 处理 Markdown 语法（文本层面）
- **Rehype** 处理 HTML（标签层面）
- 处理流程：Markdown → Remark 插件处理 → 转成 HTML → Rehype 插件处理 → 最终输出

---

## 4. 使用 Contentlayer 生成的数据

Contentlayer 处理完 MDX 文件后，会在 `.contentlayer/generated` 目录下生成 TypeScript 数据。你可以像普通 JS 对象一样导入使用：

```typescript
// 导入所有博客文章
import { allPosts } from "contentlayer/generated"

// allPosts 是一个数组，每个元素是一篇文章
console.log(allPosts[0])
// {
//   title: "Preview Mode for Headless CMS",
//   description: "How to use preview mode...",
//   date: "2023-04-09",
//   image: "/images/blog/blog-post-4.jpg",
//   authors: ["shadcn"],
//   slug: "/blog/preview-mode-headless-cms",
//   slugAsParams: "preview-mode-headless-cms",
//   body: { code: "..." },  ← 编译后的 MDX 代码
// }
```

---

## 5. 博客文章渲染页

`app/(marketing)/blog/[...slug]/page.tsx` 展示了如何渲染一篇博客文章：

```tsx
// app/(marketing)/blog/[...slug]/page.tsx（简化版）
import { allPosts, allAuthors } from "contentlayer/generated"
import { notFound } from "next/navigation"
import { Mdx } from "@/components/mdx-components"

interface PostPageProps {
  params: {
    slug: string[]  // [...slug] 捕获的路径段数组
  }
}

// 生成静态路径（构建时预生成所有博客文章页面）
export async function generateStaticParams() {
  return allPosts.map((post) => ({
    slug: post.slugAsParams.split("/"),
  }))
}

export default async function PostPage({ params }: PostPageProps) {
  // 根据 URL 中的 slug 找到对应的文章
  const slug = params.slug.join("/")
  const post = allPosts.find((post) => post.slugAsParams === slug)

  // 没找到就显示 404
  if (!post) {
    notFound()
  }

  // 查找文章的作者信息
  const authors = post.authors.map((author) =>
    allAuthors.find(({ slug }) => slug === `/authors/${author}`)
  )

  return (
    <article className="container max-w-3xl py-6 lg:py-10">
      {/* 文章标题 */}
      <h1 className="font-heading text-4xl">{post.title}</h1>

      {/* 发布日期 */}
      <time>{formatDate(post.date)}</time>

      {/* 作者信息 */}
      {authors.map((author) => (
        <div key={author._id}>
          <img src={author.avatar} alt={author.title} />
          <span>{author.title}</span>
        </div>
      ))}

      {/* 文章正文 —— 使用 Mdx 组件渲染 */}
      <Mdx code={post.body.code} />
    </article>
  )
}
```

### 关键点

1. **`generateStaticParams()`** — 告诉 Next.js 构建时应该预生成哪些页面。这样博客文章在部署后就是静态 HTML，访问速度极快。

2. **`[...slug]`** — Catch-all 路由，支持多层级的路径。比如：
   - `/blog/my-post` → `slug = ["my-post"]`
   - `/blog/2024/my-post` → `slug = ["2024", "my-post"]`

3. **`<Mdx code={post.body.code} />`** — 将编译后的 MDX 代码渲染成 React 组件。

---

## 6. MDX 自定义组件 — 让 Markdown 更强大

`components/mdx-components.tsx` 定义了 MDX 中每个 HTML 元素的自定义渲染：

```tsx
// components/mdx-components.tsx
import Image from "next/image"
import { useMDXComponent } from "next-contentlayer/hooks"
import { cn } from "@/lib/utils"
import { Callout } from "@/components/callout"
import { MdxCard } from "@/components/mdx-card"

const components = {
  // 覆盖默认的 HTML 元素样式
  h1: ({ className, ...props }) => (
    <h1 className={cn("mt-2 scroll-m-20 text-4xl font-bold tracking-tight", className)} {...props} />
  ),
  h2: ({ className, ...props }) => (
    <h2 className={cn("mt-10 scroll-m-20 border-b pb-1 text-3xl font-semibold tracking-tight first:mt-0", className)} {...props} />
  ),
  p: ({ className, ...props }) => (
    <p className={cn("leading-7 [&:not(:first-child)]:mt-6", className)} {...props} />
  ),
  a: ({ className, ...props }) => (
    <a className={cn("font-medium underline underline-offset-4", className)} {...props} />
  ),
  pre: ({ className, ...props }) => (
    <pre className={cn("mb-4 mt-6 overflow-x-auto rounded-lg border bg-black py-4", className)} {...props} />
  ),
  code: ({ className, ...props }) => (
    <code className={cn("relative rounded border px-[0.3rem] py-[0.2rem] font-mono text-sm", className)} {...props} />
  ),

  // 注入自定义 React 组件
  Image,                    // Next.js 优化的图片组件
  Callout,                  // 自定义提示框
  Card: MdxCard,            // 自定义卡片链接
}

// Mdx 渲染组件
export function Mdx({ code }: { code: string }) {
  const Component = useMDXComponent(code)  // 将编译后的代码转成 React 组件

  return (
    <div className="mdx">
      <Component components={components} />  {/* 传入自定义组件映射 */}
    </div>
  )
}
```

### 工作原理

当 MDX 中写了 `## Features` 时：
1. Markdown 编译器将它转成 `<h2>Features</h2>`
2. `components` 映射表中定义了 `h2` 的自定义渲染
3. 最终渲染成带有 Tailwind 样式的 `<h2 className="mt-10 scroll-m-20 border-b ...">`

当 MDX 中写了 `<Callout>提示</Callout>` 时：
1. Callout 在 `components` 映射表中
2. 直接渲染 `Callout` React 组件

### Callout 组件

```tsx
// components/callout.tsx
interface CalloutProps {
  icon?: string
  children?: React.ReactNode
  type?: "default" | "warning" | "danger"
}

export function Callout({ children, icon, type = "default", ...props }: CalloutProps) {
  return (
    <div
      className={cn("my-6 flex items-start rounded-md border border-l-4 p-4", {
        "border-red-900 bg-red-50": type === "danger",
        "border-yellow-900 bg-yellow-50": type === "warning",
      })}
      {...props}
    >
      {icon && <span className="mr-4 text-2xl">{icon}</span>}
      <div>{children}</div>
    </div>
  )
}
```

在 MDX 中使用：

```mdx
<Callout icon="⚠️" type="warning">
  请确保在部署前设置所有环境变量。
</Callout>
```

---

## 7. 内容文件夹结构

```
content/
├── blog/                          # 博客文章
│   ├── preview-mode-headless-cms.mdx
│   ├── dynamic-routing.mdx
│   └── ...
├── docs/                          # 文档
│   ├── index.mdx                  # 文档首页
│   ├── getting-started.mdx
│   └── ...
├── authors/                       # 作者信息
│   └── shadcn.mdx
├── pages/                         # 静态页面
│   ├── privacy.mdx
│   └── terms.mdx
└── guides/                        # 指南
    └── ...
```

**每种内容类型对应一个文件夹**，Contentlayer 通过 `filePathPattern` 来匹配：
- `docs/**/*.mdx` → 匹配 `content/docs/` 下所有 MDX 文件
- `blog/**/*.mdx` → 匹配 `content/blog/` 下所有 MDX 文件

---

## 8. 构建流程总览

```
写 MDX 文件
    ↓
Contentlayer 处理
    ↓ remarkGfm（处理表格、任务列表）
    ↓ rehypeSlug（给标题加 id）
    ↓ rehypePrettyCode（代码高亮）
    ↓ rehypeAutolinkHeadings（标题锚点）
    ↓
生成 .contentlayer/generated/
    ├── allPosts.json        ← 所有博客文章数据
    ├── allDocs.json         ← 所有文档数据
    └── index.d.ts           ← TypeScript 类型定义
    ↓
页面组件 import 数据
    ↓
Mdx 组件渲染 + 自定义组件映射
    ↓
最终 HTML 页面
```

---

## 9. 目录生成 — `lib/toc.ts`

Taxonomy 的文档页面有一个"On This Page"侧边栏，自动列出当前页面的所有标题。这个功能由两个文件实现。

### 9.1 `lib/toc.ts` — 从 Markdown 提取标题

```typescript
// lib/toc.ts（简化版）
import { toc } from "mdast-util-toc"
import { remark } from "remark"
import { visit } from "unist-util-visit"

interface Item {
  title: string      // 标题文本
  url: string        // 锚点链接，如 #features
  items?: Item[]     // 子标题（嵌套结构）
}

export type TableOfContents = { items?: Item[] }

export async function getTableOfContents(content: string): Promise<TableOfContents> {
  // 用 remark 解析 Markdown，提取目录结构
  const result = await remark().use(getToc).process(content)
  return result.data
}
```

**工作原理：**
1. `remark` 将 Markdown 文本解析成语法树（AST）
2. `mdast-util-toc` 从语法树中提取所有标题，生成嵌套的目录结构
3. `flattenNode` 函数将语法树节点转为纯文本标题
4. `getItems` 函数递归遍历目录树，生成 `{ title, url, items }` 结构

**输出示例：**

```javascript
// 输入 Markdown：
// ## Features
// ### Performance
// ### Security
// ## Installation

// 输出结构：
{
  items: [
    { title: "Features", url: "#features", items: [
      { title: "Performance", url: "#performance" },
      { title: "Security", url: "#security" },
    ]},
    { title: "Installation", url: "#installation" },
  ]
}
```

### 9.2 `components/toc.tsx` — 渲染目录 + 高亮当前位置

```tsx
// components/toc.tsx（简化版）
"use client"

export function DashboardTableOfContents({ toc }) {
  // 提取所有标题的 id
  const itemIds = toc.items?.flatMap((item) =>
    [item.url, item?.items?.map((item) => item.url)]
  ).flat().filter(Boolean).map((id) => id?.split("#")[1])

  // 用 IntersectionObserver 追踪当前可见的标题
  const activeHeading = useActiveItem(itemIds)
  const mounted = useMounted()  // 避免 SSR hydration 问题

  return mounted ? (
    <div className="space-y-2">
      <p className="font-medium">On This Page</p>
      <Tree tree={toc} activeItem={activeHeading} />
    </div>
  ) : null
}

// 递归渲染目录树
function Tree({ tree, level = 1, activeItem }) {
  return tree?.items?.length && level < 3 ? (
    <ul className={cn("m-0 list-none", { "pl-4": level !== 1 })}>
      {tree.items.map((item, index) => (
        <li key={index}>
          <a
            href={item.url}
            className={cn(
              item.url === `#${activeItem}`
                ? "font-medium text-primary"       // 当前位置高亮
                : "text-sm text-muted-foreground"  // 其他项目灰色
            )}
          >
            {item.title}
          </a>
          {item.items?.length ? (
            <Tree tree={item} level={level + 1} activeItem={activeItem} />
          ) : null}
        </li>
      ))}
    </ul>
  ) : null
}
```

**`useActiveItem`** 使用浏览器的 `IntersectionObserver` API 来检测哪个标题当前在视口中，实现"滚动时自动高亮目录项"的效果。

---

## 10. 文档导航配置 — `config/docs.ts`

文档页面的侧边栏导航结构定义在配置文件中：

```typescript
// config/docs.ts
export const docsConfig = {
  mainNav: [
    { title: "Documentation", href: "/docs" },
    { title: "Guides", href: "/guides" },
  ],
  sidebarNav: [
    {
      title: "Getting Started",
      items: [
        { title: "Introduction", href: "/docs" },
      ],
    },
    {
      title: "Documentation",
      items: [
        { title: "Introduction", href: "/docs/documentation" },
        { title: "Contentlayer", href: "/docs/in-progress", disabled: true },
        { title: "Components", href: "/docs/documentation/components" },
        { title: "Code Blocks", href: "/docs/documentation/code-blocks" },
      ],
    },
    // ... 更多分组
  ],
}
```

### Pager 组件 — 上一篇 / 下一篇导航

```tsx
// components/pager.tsx
export function DocsPager({ doc }) {
  const pager = getPagerForDoc(doc)

  return (
    <div className="flex flex-row items-center justify-between">
      {pager?.prev && (
        <Link href={pager.prev.href}>
          <Icons.chevronLeft /> {pager.prev.title}
        </Link>
      )}
      {pager?.next && (
        <Link href={pager.next.href}>
          {pager.next.title} <Icons.chevronRight />
        </Link>
      )}
    </div>
  )
}

// 核心逻辑：把嵌套导航展平，找到当前文档的前后项
export function getPagerForDoc(doc) {
  const flattenedLinks = [null, ...flatten(docsConfig.sidebarNav), null]
  const activeIndex = flattenedLinks.findIndex((link) => doc.slug === link?.href)
  return {
    prev: flattenedLinks[activeIndex - 1],
    next: flattenedLinks[activeIndex + 1],
  }
}
```

**`flatten` 函数**将嵌套的导航结构展平为一维数组，这样就能用简单的 `index - 1` 和 `index + 1` 找到前后项。

---

## 小练习

### 练习 1：创建一篇博客文章
1. 在 `content/blog/` 下创建 `my-first-post.mdx`：
```mdx
---
title: My First Post
description: This is my first blog post.
date: 2024-01-01
image: /images/blog/blog-post-1.jpg
authors:
  - shadcn
---

## Hello World

This is my first blog post written in MDX.

**Bold text** and *italic text*.

- Item 1
- Item 2
```
2. 运行 `pnpm dev`，访问 `/blog/my-first-post`
3. 观察标题的样式——它会自动使用 `mdx-components.tsx` 中定义的样式

### 练习 2：在 MDX 中使用自定义组件
在你的博客文章中添加：
```mdx
<Callout>
  This is a callout component inside MDX!
</Callout>
```
刷新页面看看效果。

### 练习 3：理解 computedFields
在 `contentlayer.config.js` 中，`computedFields` 的 `slug` 和 `slugAsParams` 有什么区别？对于文件路径 `content/blog/2024/my-post.mdx`，两个字段的值分别是什么？

---

## 本章小结

- MDX = Markdown + JSX，让你在文档中使用 React 组件
- Contentlayer 是"翻译官"，将 MDX 文件编译成 TypeScript 可用的数据
- `contentlayer.config.js` 定义了内容模型（字段类型和验证规则）
- rehype/remark 插件增强了 Markdown 的功能（语法高亮、自动链接等）
- `mdx-components.tsx` 自定义了 MDX 中每个元素的渲染方式
- `generateStaticParams()` 让博客文章在构建时预生成，访问速度快

下一章我们将学习数据库层——数据是怎么存和取的。
