# Day 05 — 内容系统（Content System）

> 对应章节：[Chapter 5: Markdown 内容系统](../chapter-05-content-system.md)

---

## 一、今日学习目标

1. **理解内容流水线**：一个 `.mdx` 文件是怎样经过 Contentlayer、remark/rehype 插件，最终变成浏览器里的网页的
2. **能读懂 `contentlayer.config.js`**：知道每个字段（field）是什么意思，`computedFields` 是怎么自动算出来的
3. **能自己创建一篇 MDX 博客文章并在本地看到效果**

---

## 二、前置知识速补

> 这一节帮你快速补齐 Day 05 需要的 JavaScript 基础。如果你已经掌握，可以跳过。

### 2.1 [对象字面量](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Working_with_objects) `{ key: value }`

在 JavaScript 里，用花括号 `{}` 可以创建一个"对象"（object）。对象就是"一组有名字的数据"。

```javascript
// 创建一个对象，描述一篇博客文章
const post = {
  title: "My First Post",   // key 是 title，value 是字符串
  date: "2024-01-01",       // key 是 date，value 也是字符串
  published: true,           // key 是 published，value 是布尔值
}

// 读取对象的值：用 "点" 语法
console.log(post.title)      // 输出：My First Post
console.log(post.published)  // 输出：true
```

**为什么要学这个？** Contentlayer 的配置文件里到处都是对象字面量。每个文档类型的 `fields` 就是一个嵌套对象。

### 2.2 数组方法：[map](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/map)、[filter](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/filter)、[find](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/find)、[sort](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/sort)

数组（Array）是"一组数据的列表"。JavaScript 提供了很多方法来操作数组：

```javascript
const posts = [
  { title: "Post A", date: "2024-01-01", published: true },
  { title: "Post B", date: "2024-03-15", published: false },
  { title: "Post C", date: "2024-02-10", published: true },
]

// map — 把每个元素"变"成另一个东西，返回新数组
const titles = posts.map((post) => post.title)
// 结果：["Post A", "Post B", "Post C"]
// 解读：对数组里的每一项执行箭头函数，把结果收集成新数组

// filter — 筛选出符合条件的元素
const published = posts.filter((post) => post.published)
// 结果：只保留 published 为 true 的，即 Post A 和 Post C

// find — 找到第一个符合条件的元素（不是数组，是单个元素）
const target = posts.find((post) => post.title === "Post B")
// 结果：{ title: "Post B", date: "2024-03-15", published: false }

// sort — 排序（会修改原数组！）
posts.sort((a, b) => new Date(b.date) - new Date(a.date))
// 按日期从新到旧排序：Post B → Post C → Post A
```

**为什么要学这个？** 博客列表页用 `filter` 筛选已发布的文章，用 `sort` 按日期排序；博客详情页用 `find` 根据 slug 找到对应的文章。这些都是项目源码里的真实用法。

### 2.3 [正则表达式](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Guide/Regular_expressions)基础

正则表达式（Regular Expression, 简称 regex）是一种"模式匹配"工具，用来在文本中查找/替换符合特定规则的内容。

```javascript
// 最简单的正则：用 / / 包裹
const pattern = /hello/

// test() — 检测字符串里是否包含匹配
pattern.test("say hello world")  // true
pattern.test("say hi world")    // false

// 常见的特殊符号
// .   匹配任意一个字符
// *   前面的字符可以出现 0 次或多次
// +   前面的字符至少出现 1 次
// \s  匹配空格、制表符等空白字符
// \w  匹配字母、数字、下划线

// 在 Taxonomy 的 lib/toc.ts 中，用到了类似的文本处理逻辑
```

**为什么要学这个？** `lib/toc.ts`（目录生成工具）会遍历 Markdown 的语法树来提取标题。虽然没有直接用正则，但理解"从文本中按规则提取信息"的思路很重要。

### 2.4 [Date](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Date) 对象基础

```javascript
// 创建一个日期对象
const d = new Date("2024-01-15")

console.log(d.getFullYear())  // 2024
console.log(d.getMonth())     // 0（注意！0 = 一月，11 = 十二月）
console.log(d.getDate())      // 15

// 比较两个日期：可以直接相减，得到毫秒差
const d1 = new Date("2024-01-01")
const d2 = new Date("2024-03-15")
console.log(d2 - d1)  // 正数，说明 d2 比 d1 晚
```

**为什么要学这个？** 博客列表页用 `compareDesc(new Date(a.date), new Date(b.date))` 来按发布时间排序。你需要理解 `new Date(...)` 是在把字符串变成日期对象。

### 2.5 TypeScript 的 [type](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#type-aliases) 和 [interface](https://www.typescriptlang.org/docs/handbook/2/objects.html)

TypeScript 是 JavaScript 的超集，增加了"类型检查"。`type` 和 `interface` 都是用来描述"数据长什么样"的：

```typescript
// interface — 描述一个对象的形状
interface Item {
  title: string       // title 必须是字符串
  url: string         // url 必须是字符串
  items?: Item[]      // items 是可选的（? 表示可选），类型是 Item 数组
}

// type — 也可以描述对象形状，还能做更多事
type PostStatus = "draft" | "published"   // 联合类型：只能是这两个值之一

type Post = {
  title: string
  status: PostStatus
}
```

**简单区分：** `interface` 主要用于描述对象的形状；`type` 更灵活，能定义联合类型、交叉类型等。在本项目中两种都有使用，看到时知道"它在描述数据结构"就行。

---

## 三、核心概念清单

| 概念 | 英文 | 一句话解释 | 在项目中对应的文件 |
|------|------|-----------|-------------------|
| [MDX](https://mdxjs.com/) | MDX (Markdown + JSX) | 在 Markdown 里直接写 React 组件 | `content/blog/*.mdx` |
| [Contentlayer](https://contentlayer.dev/) | Contentlayer | 把 MDX 文件编译成 TypeScript 可用数据的"翻译官" | `contentlayer.config.js` |
| 前置信息 | frontmatter | MDX 文件顶部 `---` 包裹的元数据（标题、日期等） | 每个 `.mdx` 文件的头部 |
| 文档类型 | Document Type | Contentlayer 中定义的内容模型（Post、Doc、Author 等） | `contentlayer.config.js` |
| 计算字段 | computedFields | 从文件路径自动算出 slug 等字段 | `contentlayer.config.js` 第 8-17 行 |
| [remark](https://github.com/remarkjs/remark) 插件 | remark plugin | 在 Markdown 层面增强功能（如支持表格） | `remarkGfm` |
| [rehype](https://github.com/rehypejs/rehype) 插件 | rehype plugin | 在 HTML 层面增强功能（如代码高亮、标题锚点） | `rehypeSlug`, `rehypePrettyCode`, `rehypeAutolinkHeadings` |
| 组件映射表 | MDX Components Map | 告诉 MDX "遇到 `<h1>` 用什么组件渲染" | `components/mdx-components.tsx` |

---

## 四、源码精读

### 4.1 `contentlayer.config.js` — 内容模型配置

> 文件路径：`/contentlayer.config.js`（项目根目录）

这个文件的作用是告诉 Contentlayer："我有哪些类型的内容？每种内容需要什么字段？用什么插件处理？"

**第一段：导入依赖**

```javascript
import { defineDocumentType, makeSource } from "contentlayer/source-files"
import rehypeAutolinkHeadings from "rehype-autolink-headings"
import rehypePrettyCode from "rehype-pretty-code"
import rehypeSlug from "rehype-slug"
import remarkGfm from "remark-gfm"
```

逐行解读：
- `defineDocumentType` — 用来定义一种文档类型（比如"博客文章"、"作者信息"）
- `makeSource` — 用来创建最终的 Contentlayer 配置
- 后面四行分别导入了四个插件（下面会详细讲）

**第二段：计算字段（computedFields）**

```javascript
const computedFields = {
  slug: {
    type: "string",
    resolve: (doc) => `/${doc._raw.flattenedPath}`,
  },
  slugAsParams: {
    type: "string",
    resolve: (doc) => doc._raw.flattenedPath.split("/").slice(1).join("/"),
  },
}
```

这是什么意思？举个例子：

假设你有一个文件 `content/blog/my-first-post.mdx`，Contentlayer 会自动把它的路径变成 `blog/my-first-post`（去掉 `content/` 前缀和 `.mdx` 后缀），存到 `doc._raw.flattenedPath` 里。

- `slug` 的 `resolve` 函数：在前面加一个 `/`，得到 `/blog/my-first-post`（完整 URL 路径）
- `slugAsParams` 的 `resolve` 函数：
  - `"blog/my-first-post".split("/")` → `["blog", "my-first-post"]`
  - `.slice(1)` → `["my-first-post"]`（去掉第一个元素）
  - `.join("/")` → `"my-first-post"`（只留文章名）

**为什么需要两个？** `slug` 用于生成完整链接（如 `<Link href={post.slug}>`），`slugAsParams` 用于在路由中匹配参数。

**第三段：定义文档类型**

项目一共定义了 5 种文档类型。我们重点看 `Post`（博客文章）：

```javascript
export const Post = defineDocumentType(() => ({
  name: "Post",                        // 类型名称，Contentlayer 会生成 allPosts 数组
  filePathPattern: `blog/**/*.mdx`,    // 匹配 content/blog/ 下所有 .mdx 文件
  contentType: "mdx",                  // 告诉 Contentlayer 这是 MDX 格式
  fields: {                            // 定义 frontmatter 里必须/可以有哪些字段
    title: {
      type: "string",                  // 字段类型是字符串
      required: true,                  // 必填
    },
    description: {
      type: "string",                  // 可选（没写 required: true）
    },
    date: {
      type: "date",                    // 日期类型，Contentlayer 会自动解析
      required: true,
    },
    published: {
      type: "boolean",                 // 布尔值：true 或 false
      default: true,                   // 默认值：如果不写就是 true
    },
    image: {
      type: "string",                  // 封面图路径
      required: true,
    },
    authors: {
      type: "list",                    // 列表类型（数组）
      of: { type: "string" },         // 列表里的每一项是字符串
      required: true,
    },
  },
  computedFields,                      // 使用上面定义的计算字段
}))
```

**其他 4 种文档类型一览：**

| 类型 | name | 匹配路径 | 特殊字段 |
|------|------|----------|---------|
| 文档 | `Doc` | `docs/**/*.mdx` | title, description, published |
| 指南 | `Guide` | `guides/**/*.mdx` | title, description, date, published, featured |
| 作者 | `Author` | `authors/**/*.mdx` | title, avatar, twitter |
| 页面 | `Page` | `pages/**/*.mdx` | title, description |

**第四段：`makeSource` 最终配置**

```javascript
export default makeSource({
  contentDirPath: "./content",                          // 内容文件夹在项目根目录的 content/
  documentTypes: [Page, Doc, Guide, Post, Author],     // 注册所有文档类型
  mdx: {
    remarkPlugins: [remarkGfm],                         // Markdown 层插件
    rehypePlugins: [                                    // HTML 层插件
      rehypeSlug,
      [rehypePrettyCode, { theme: "github-dark", /* ...回调配置... */ }],
      [rehypeAutolinkHeadings, { properties: { className: ["subheading-anchor"] } }],
    ],
  },
})
```

### 4.2 MDX 文件结构 — 以真实博客文章为例

> 文件路径：`/content/blog/preview-mode-headless-cms.mdx`

一个 MDX 文件分两部分：

**Part 1：frontmatter（前置信息）**

```yaml
---
title: Preview Mode for Headless CMS
description: How to implement preview mode in your headless CMS.
date: "2023-04-09"
image: /images/blog/blog-post-1.jpg
authors:
  - shadcn
---
```

这些字段 **必须和 `contentlayer.config.js` 中 `Post` 类型定义的 `fields` 一一对应**。比如 `title` 是 `required: true`，如果你不写就会报错。

**Part 2：正文内容**

正文就是普通 Markdown，但可以混入 React 组件：

```mdx
<Callout>
  这是一个提示框，在普通 Markdown 里做不到。
</Callout>

## What to expect from here on out

普通的 Markdown 文本……

<Image src="/images/blog/blog-post-4.jpg" width="718" height="404" alt="Image" />
```

注意：`<Callout>` 和 `<Image>` 都是 React 组件，不是 HTML 标签。这就是 MDX 的"X"——JSX。

**作者文件的结构更简单：**

> 文件路径：`/content/authors/shadcn.mdx`

```yaml
---
title: shadcn
avatar: /images/avatars/shadcn.png
twitter: shadcn
---
```

这个文件只有 frontmatter 没有正文，因为 `Author` 类型只需要元数据。

### 4.3 内容渲染组件 — `components/mdx-components.tsx`

> 文件路径：`/components/mdx-components.tsx`

这个文件做了两件事：

**第一件事：定义组件映射表（components）**

```tsx
const components = {
  // 覆盖 HTML 默认标签的样式
  h1: ({ className, ...props }) => (
    <h1 className={cn("mt-2 scroll-m-20 text-4xl font-bold tracking-tight", className)}
      {...props} />
  ),
  h2: ({ className, ...props }) => (
    <h2 className={cn("mt-10 scroll-m-20 border-b pb-1 text-3xl font-semibold ...", className)}
      {...props} />
  ),
  p: ({ className, ...props }) => (
    <p className={cn("leading-7 [&:not(:first-child)]:mt-6", className)}
      {...props} />
  ),
  // ... 还有 a, ul, ol, li, blockquote, pre, code, table 等等

  // 注入自定义 React 组件（MDX 正文中可以直接使用）
  Image,              // Next.js 的图片优化组件
  Callout,            // 自定义提示框
  Card: MdxCard,      // 自定义卡片链接
}
```

**工作原理：** 当 MDX 中写了 `## 标题` 时，Markdown 编译器把它变成 `<h2>`，然后映射表说"遇到 `h2` 就用我自定义的带 Tailwind 样式的组件"。当 MDX 中写了 `<Callout>` 时，映射表直接把它替换成 `Callout` React 组件。

**第二件事：导出 `Mdx` 渲染组件**

```tsx
export function Mdx({ code }: MdxProps) {
  const Component = useMDXComponent(code)  // 把编译后的 MDX 代码变成 React 组件

  return (
    <div className="mdx">
      <Component components={components} />  {/* 传入映射表 */}
    </div>
  )
}
```

`useMDXComponent` 来自 `next-contentlayer/hooks`，它接收 Contentlayer 编译好的 MDX 代码字符串，返回一个 React 组件。

### 4.4 博客文章页面 — `app/(marketing)/blog/[...slug]/page.tsx`

> 文件路径：`/app/(marketing)/blog/[...slug]/page.tsx`

这个页面的核心逻辑（简化版）：

```tsx
import { allAuthors, allPosts } from "contentlayer/generated"
import { Mdx } from "@/components/mdx-components"

// 1. 根据 URL 参数找文章
async function getPostFromParams(params) {
  const slug = params?.slug?.join("/")                      // ["my-post"] → "my-post"
  const post = allPosts.find((post) => post.slugAsParams === slug)  // 用 find 查找
  return post
}

// 2. 告诉 Next.js 构建时需要预生成哪些页面
export async function generateStaticParams() {
  return allPosts.map((post) => ({
    slug: post.slugAsParams.split("/"),                     // "my-post" → ["my-post"]
  }))
}

// 3. 渲染页面
export default async function PostPage({ params }) {
  const post = await getPostFromParams(params)
  if (!post) notFound()                                     // 没找到就 404

  // 根据文章的 authors 字段找到对应的作者信息
  const authors = post.authors.map((author) =>
    allAuthors.find(({ slug }) => slug === `/authors/${author}`)
  )

  return (
    <article>
      <time>{formatDate(post.date)}</time>                  {/* 发布日期 */}
      <h1>{post.title}</h1>                                 {/* 标题 */}
      {authors.map((author) => (                            /* 作者头像和名字 */
        <Image src={author.avatar} alt={author.title} />
      ))}
      <Mdx code={post.body.code} />                        {/* 渲染 MDX 正文 */}
    </article>
  )
}
```

### 4.5 博客列表页 — `app/(marketing)/blog/page.tsx`

> 文件路径：`/app/(marketing)/blog/page.tsx`

```tsx
import { allPosts } from "contentlayer/generated"
import { compareDesc } from "date-fns"

export default async function BlogPage() {
  const posts = allPosts
    .filter((post) => post.published)   // 只要 published 为 true 的文章
    .sort((a, b) => {
      return compareDesc(new Date(a.date), new Date(b.date))  // 按日期从新到旧排序
    })

  return (
    <div>
      {posts.map((post) => (
        <article key={post._id}>
          <Image src={post.image} alt={post.title} />
          <h2>{post.title}</h2>
          <p>{post.description}</p>
          <p>{formatDate(post.date)}</p>
          <Link href={post.slug}>View Article</Link>
        </article>
      ))}
    </div>
  )
}
```

注意这里同时用到了 `filter`、`sort`、`map` 三个数组方法，就是前置知识里讲的内容。

### 4.6 目录生成工具 — `lib/toc.ts`

> 文件路径：`/lib/toc.ts`

这个文件用 `remark` 解析 Markdown 内容，自动提取所有标题生成目录（Table of Contents）：

```typescript
import { toc } from "mdast-util-toc"
import { remark } from "remark"
import { visit } from "unist-util-visit"

// 递归提取文本节点的内容
function flattenNode(node) {
  const p = []
  visit(node, (node) => {
    if (!["text", "emphasis", "strong", "inlineCode"].includes(node.type)) return
    p.push(node.value)
  })
  return p.join("")
}

interface Item {
  title: string      // 标题文本
  url: string        // 锚点链接，如 #features
  items?: Item[]     // 子标题（嵌套结构）
}

// 递归遍历 toc 节点，构建 Item 结构
function getItems(node, current) {
  if (node.type === "paragraph") {
    // 从 paragraph 中提取链接和标题文本
    visit(node, (item) => {
      if (item.type === "link") {
        current.url = item.url
        current.title = flattenNode(node)
      }
    })
    return current
  }

  if (node.type === "list") {
    // 列表 → 递归处理每个子项
    current.items = node.children.map((i) => getItems(i, {}))
    return current
  }
  // ...
}

export async function getTableOfContents(content: string): Promise<TableOfContents> {
  const result = await remark().use(getToc).process(content)
  return result.data
}
```

**工作流程：**
1. `remark` 将 Markdown 文本解析成语法树（AST）
2. `mdast-util-toc` 从语法树中提取标题，生成目录树
3. `getItems` 递归遍历目录树，转换为 `{ title, url, items }` 格式

**输出示例：** 对于包含 `## Features`、`### Performance`、`## Installation` 的 Markdown，输出：

```json
{
  "items": [
    { "title": "Features", "url": "#features", "items": [
      { "title": "Performance", "url": "#performance" }
    ]},
    { "title": "Installation", "url": "#installation" }
  ]
}
```

### 4.7 目录渲染组件 — `components/toc.tsx`

> 文件路径：`/components/toc.tsx`

```tsx
"use client"

export function DashboardTableOfContents({ toc }) {
  const itemIds = toc.items?.flatMap((item) =>
    [item.url, item?.items?.map((item) => item.url)]
  ).flat().filter(Boolean).map((id) => id?.split("#")[1])

  const activeHeading = useActiveItem(itemIds)  // IntersectionObserver 追踪
  const mounted = useMounted()  // 避免 SSR hydration 问题

  return mounted ? (
    <div>
      <p className="font-medium">On This Page</p>
      <Tree tree={toc} activeItem={activeHeading} />
    </div>
  ) : null
}
```

`useActiveItem` 使用 `IntersectionObserver` 检测哪个标题在视口中，实现"滚动时目录自动高亮"效果。`Tree` 组件递归渲染嵌套目录。

### 4.8 文档导航配置 — `config/docs.ts`

> 文件路径：`/config/docs.ts`

文档侧边栏的结构定义在配置文件中：

```typescript
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
      ],
    },
  ],
}
```

**注意：** `disabled: true` 的项目会显示为灰色，点击无效——用于标记"正在开发中"的页面。

`components/pager.tsx` 中的 `DocsPager` 组件利用这个配置来生成"上一篇/下一篇"导航，核心逻辑是把嵌套导航展平为一维数组，然后用 `index - 1` 和 `index + 1` 找到前后项。

这说明 `remark` 不仅在 Contentlayer 的插件中使用，也可以单独用来处理 Markdown 文本。

---

## 五、动手练习

### 练习 1：创建一篇博客文章（40 分钟）

**目标：** 从零创建一篇 MDX 博客文章，理解整个流程。

**步骤：**

1. 在 `content/blog/` 目录下新建文件 `my-learning-note.mdx`

2. 写入以下内容（注意 frontmatter 的格式要严格）：

```mdx
---
title: My Day 05 Learning Note
description: What I learned about the content system in Taxonomy.
date: "2024-01-15"
image: /images/blog/blog-post-1.jpg
authors:
  - shadcn
published: true
---

## What I Learned Today

I learned how **Contentlayer** turns MDX files into TypeScript data.

### The Content Pipeline

1. Write a `.mdx` file in the `content/` folder
2. Contentlayer reads and compiles it
3. The page component imports and renders it

### A Code Example

```tsx
const posts = allPosts.filter((post) => post.published)
```

This line filters out unpublished posts.
```

3. 确保开发服务器正在运行（`pnpm dev`），等待 Contentlayer 自动编译

4. 在浏览器中访问 `http://localhost:3000/blog/my-learning-note`

5. 然后访问 `http://localhost:3000/blog`，确认列表页也出现了你的文章

**排错提示：**

- 如果页面显示 404：检查 frontmatter 中的 `date` 和 `image` 是否填了（它们是 `required: true`）
- 如果报编译错误：检查 `---` 之间的 YAML 格式，冒号后面必须有空格
- 如果文章不在列表页出现：检查 `published` 是否为 `true`（默认是 true，但如果你写了 `published: false` 就会被过滤掉）

### 练习 2：在 MDX 中使用自定义组件（30 分钟）

**目标：** 体验 MDX 的"超能力"——在 Markdown 中使用 React 组件。

**步骤：**

1. 打开练习 1 中创建的 `my-learning-note.mdx`

2. 在正文中添加一个 `Callout` 组件：

```mdx
<Callout>
  This is a callout component rendered inside MDX.
  You cannot do this in regular Markdown!
</Callout>

<Callout icon="⚠️" type="warning">
  Make sure you set all required fields in the frontmatter.
</Callout>
```

3. 保存文件，刷新页面，观察两种 Callout 的不同样式

4. 再添加一个 `Image` 组件：

```mdx
<Image
  src="/images/blog/blog-post-1.jpg"
  width="718"
  height="404"
  alt="A sample image in MDX"
/>
```

5. 保存并刷新，观察图片的渲染效果

**排错提示：**

- 如果 `<Callout>` 没有显示：确保组件名拼写正确（大小写敏感！`<callout>` 不行，必须是 `<Callout>`）
- 如果 `<Image>` 报错：`width` 和 `height` 必须是字符串（用引号包裹）
- 组件标签必须闭合：`<Callout>内容</Callout>`，不能缺少结束标签

### 练习 3：阅读并修改 computedFields（20 分钟）

**目标：** 深入理解计算字段的工作原理。

**步骤：**

1. 打开 `contentlayer.config.js`，找到 `computedFields` 部分

2. 手动推演：假设文件路径是 `content/blog/2024/my-post.mdx`
   - `doc._raw.flattenedPath` 的值是什么？→ `blog/2024/my-post`
   - `slug` 的值是什么？→ `"/blog/2024/my-post"`（前面加 `/`）
   - `slugAsParams` 的值是什么？→ `"2024/my-post"`（`split("/")` 后 `slice(1)` 去掉 `"blog"`）

3. 思考题：如果想给每篇文章增加一个 `readingTime`（阅读时间）的计算字段，`resolve` 函数应该怎么写？提示：`doc.body.raw` 里存着原始文本内容，可以根据字数估算阅读时间。

**排错提示：**

- `split("/")` 返回的是数组，`slice(1)` 是去掉第一个元素，`join("/")` 是把数组合并回字符串
- 如果你不确定某个值，可以在 `resolve` 函数里加 `console.log(doc._raw)` 看看完整的原始数据

---

## 六、概念图解：内容处理流水线

```
你写的 MDX 文件
  content/blog/my-post.mdx
         │
         ▼
┌─────────────────────────────────┐
│   Contentlayer 读取文件          │
│                                 │
│  1. 解析 frontmatter (YAML)     │  ← 提取 title, date, authors 等
│  2. 验证字段是否符合 config 定义  │  ← 缺了 required 字段就报错
│  3. 计算 computedFields          │  ← 从文件路径算出 slug
│                                 │
│  4. 处理 MDX 正文：              │
│     Markdown 原文                │
│         │                       │
│         ▼                       │
│     remarkGfm 插件              │  ← 支持表格、任务列表、删除线
│     (Markdown 语法树层面处理)     │
│         │                       │
│         ▼                       │
│     转换成 HTML                  │
│         │                       │
│         ▼                       │
│     rehypeSlug 插件              │  ← 给 <h2> 加上 id="features"
│         │                       │
│         ▼                       │
│     rehypePrettyCode 插件        │  ← 代码块语法高亮 + 行号
│         │                       │
│         ▼                       │
│     rehypeAutolinkHeadings 插件  │  ← 标题旁加 # 锚点链接
│                                 │
│  5. 编译成 JavaScript 代码       │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  .contentlayer/generated/       │
│                                 │
│  allPosts — 所有博客文章的数组    │  ← import { allPosts } from "contentlayer/generated"
│  allDocs  — 所有文档的数组       │
│  allAuthors — 所有作者的数组     │
│  index.d.ts — TypeScript 类型   │
└─────────────────────────────────┘
         │
         ▼
┌─────────────────────────────────┐
│  页面组件渲染                    │
│                                 │
│  blog/page.tsx                  │
│    allPosts.filter → sort → map │  ← 筛选、排序、循环渲染列表
│                                 │
│  blog/[...slug]/page.tsx        │
│    allPosts.find(slug匹配)       │  ← 根据 URL 找到具体文章
│    <Mdx code={post.body.code}/> │  ← 渲染 MDX 正文
│         │                       │
│         ▼                       │
│  mdx-components.tsx             │
│    h1 → 自定义样式组件           │
│    h2 → 自定义样式组件           │
│    Callout → 提示框组件          │
│    Image → Next.js 图片组件     │
└─────────────────────────────────┘
         │
         ▼
   浏览器看到的最终网页
```

**一句话总结：** MDX 文件 → Contentlayer 编译（经过 remark/rehype 插件处理）→ 生成 TypeScript 数据 → 页面组件 import 数据 → `Mdx` 组件渲染（使用组件映射表）→ 最终 HTML。

---

## 七、自检问题

完成学习后，试着不看资料回答以下问题。答案附在每题下方。

### 问题 1：MDX 和普通 Markdown 的区别是什么？

<details>
<summary>查看答案</summary>

MDX = Markdown + JSX。普通 Markdown 只能写文本格式（标题、列表、代码块等），而 MDX 可以在 Markdown 中直接使用 React 组件（如 `<Callout>`、`<Image>`）。这让内容页面变得更灵活、更交互。

</details>

### 问题 2：`contentlayer.config.js` 中 `computedFields` 的 `slug` 和 `slugAsParams` 分别是什么？文件 `content/docs/getting-started.mdx` 的两个值分别是什么？

<details>
<summary>查看答案</summary>

- `slug`：在 `flattenedPath` 前加 `/`，结果是 `/docs/getting-started`——用于页面链接
- `slugAsParams`：去掉第一段路径，结果是 `getting-started`——用于动态路由参数匹配

推导过程：`flattenedPath` = `"docs/getting-started"` → `slug` = `"/docs/getting-started"` → `slugAsParams` = `"getting-started"`

</details>

### 问题 3：remark 插件和 rehype 插件的区别是什么？本项目用了哪些？

<details>
<summary>查看答案</summary>

- **remark 插件**在 Markdown 语法树（AST）层面工作，处理的是 Markdown 文本。本项目用了 `remarkGfm`（GitHub 风格 Markdown，支持表格、任务列表等）。
- **rehype 插件**在 HTML 层面工作，处理的是 HTML 标签。本项目用了三个：
  - `rehypeSlug`：给标题元素加 `id` 属性
  - `rehypePrettyCode`：代码块语法高亮
  - `rehypeAutolinkHeadings`：标题旁添加锚点链接

处理顺序：Markdown → remark 插件 → 转 HTML → rehype 插件 → 最终输出。

</details>

### 问题 4：博客列表页是怎样筛选和排序文章的？用到了哪些数组方法？

<details>
<summary>查看答案</summary>

在 `app/(marketing)/blog/page.tsx` 中：

```tsx
const posts = allPosts
  .filter((post) => post.published)   // filter：只保留 published 为 true 的
  .sort((a, b) => compareDesc(new Date(a.date), new Date(b.date)))  // sort：按日期从新到旧
```

然后用 `.map()` 循环渲染每篇文章的卡片。一共用了 `filter`、`sort`、`map` 三个数组方法。

</details>

### 问题 5：`components/mdx-components.tsx` 中的 `components` 对象映射表是怎么工作的？如果我想让所有 MDX 中的 `<a>` 链接在新标签页打开，应该怎么改？

<details>
<summary>查看答案</summary>

`components` 对象告诉 MDX 渲染器：遇到某个 HTML 标签或组件时，用我定义的 React 组件代替。比如 `h2: (...) => <h2 className="...">` 就是把所有 `<h2>` 替换成带 Tailwind 样式的版本。

要让 `<a>` 在新标签页打开，修改映射表中的 `a`：

```tsx
a: ({ className, ...props }) => (
  <a
    className={cn("font-medium underline underline-offset-4", className)}
    target="_blank"          // 加上这一行
    rel="noopener noreferrer" // 安全考虑，加上这一行
    {...props}
  />
),
```

</details>

---

## 八、预计时间分配

| 环节 | 预计时间 | 说明 |
|------|---------|------|
| 前置知识速补 | 30 分钟 | 对象、数组方法、正则、Date、type/interface |
| 核心概念 + 源码精读 | 60 分钟 | 配合章节教程一起读，边读边对照源码 |
| 练习 1：创建博客文章 | 40 分钟 | 包含排错时间 |
| 练习 2：使用自定义组件 | 30 分钟 | 体验 MDX 的组件嵌入能力 |
| 练习 3：理解 computedFields | 20 分钟 | 手动推演，加深理解 |
| 自检问题 + 笔记整理 | 20 分钟 | 回顾全天所学 |
| **合计** | **约 3.5 小时** | |

建议休息安排：每 50 分钟休息 10 分钟。可以在源码精读和练习之间安排一次休息。

---

## 九、常见踩坑点

### 踩坑 1：Contentlayer 编译缓存不更新

**症状：** 修改了 MDX 文件后，页面没有变化或者报错。

**解决：** 停掉 dev server，删除 `.contentlayer/` 目录，再重新运行 `pnpm dev`：

```bash
rm -rf .contentlayer
pnpm dev
```

### 踩坑 2：frontmatter 的 YAML 格式错误

**症状：** Contentlayer 报编译错误，类似 "Invalid frontmatter"。

**常见原因：**
- 冒号后面没有空格：`title:Hello`（错）→ `title: Hello`（对）
- 包含特殊字符没有加引号：`title: What's new`（错）→ `title: "What's new"`（对）
- `date` 字段格式不对：`date: 2024/01/15`（错）→ `date: "2024-01-15"`（对）
- 缩进用了 Tab 而不是空格（YAML 只允许空格缩进）

### 踩坑 3：MDX 中的 JSX 语法陷阱

**注意事项：**
- 用 `className` 不要用 `class`（JSX 的规则）
- 组件名必须大写开头：`<Callout>` 对，`<callout>` 错
- 组件标签必须闭合：`<Image ... />` 或 `<Callout>...</Callout>`
- MDX 中不能写 JavaScript 注释 `// ...`，要用 `{/* ... */}`

### 踩坑 4：图片路径问题

**规则：** MDX 中引用的图片必须放在 `public/` 目录下，路径以 `/` 开头。

```mdx
<!-- 正确 — 图片文件在 public/images/blog/my-pic.jpg -->
<Image src="/images/blog/my-pic.jpg" width="718" height="404" alt="My pic" />

<!-- 错误 — 不要用相对路径 -->
<Image src="../public/images/blog/my-pic.jpg" ... />
```

### 踩坑 5：`[...slug]` 参数是数组不是字符串

在博客详情页中，`params.slug` 是一个字符串数组，不是单个字符串：

```typescript
// URL: /blog/my-first-post
// params.slug 的值是 ["my-first-post"]，不是 "my-first-post"

// 所以需要 join("/") 转回字符串
const slug = params.slug.join("/")  // "my-first-post"
```

如果你直接拿 `params.slug` 去和 `slugAsParams` 比较，会找不到文章，因为数组和字符串不相等。

---

## 十、延伸阅读

1. **MDX 官方文档**：https://mdxjs.com/ — 了解 MDX 的完整语法和高级用法
2. **Contentlayer 官方文档**：https://contentlayer.dev/ — 深入了解配置项、类型定义、插件系统
3. **unified 生态系统**（remark/rehype 的上层项目）：https://unifiedjs.com/ — 理解 Markdown/HTML 处理的统一架构
4. **rehype-pretty-code**：https://rehype-pretty-code.netlify.app/ — 代码高亮插件的完整配置选项
5. **remark-gfm**：https://github.com/remarkjs/remark-gfm — GitHub 风格 Markdown 的扩展语法说明
6. **Next.js 静态生成**：https://nextjs.org/docs/app/building-your-application/rendering/server-components — 理解 `generateStaticParams()` 的工作原理
