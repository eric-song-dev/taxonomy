# Day 11 — 综合实战（Capstone）

> 这一天没有对应的新章节。你将综合运用前 10 天学到的所有知识，完成跨模块的实战练习。

---

## 一、今日学习目标

1. **完整追踪一个用户流程**：从注册到创建文章到升级付费，跨越认证、数据库、API、编辑器、支付等多个模块
2. **尝试添加一个新功能**：综合修改数据库、API、UI 三层
3. **建立整体架构的心智模型**：能画出项目的完整架构图

---

## 二、练习 1：完整用户流程追踪（90 分钟）

### 目标

追踪一个用户从"第一次访问网站"到"创建文章并升级 Pro"的完整代码路径。不需要写代码，只需要阅读源码并记录每一步经过了哪些文件。

### 流程

```
访问首页 → 注册/登录 → 进入 Dashboard → 创建文章 → 编辑文章
→ 保存文章 → 创建第 4 篇文章（被限制） → 升级 Pro → 继续创建
```

### 步骤

#### Step 1：访问首页

1. 找到首页的路由文件：`app/(marketing)/page.tsx`
2. 找到首页使用的布局：`app/(marketing)/layout.tsx`
3. 记录首页渲染了哪些组件（Hero、Features 等）
4. 找到"Get Started"或"Sign In"按钮的链接目标

**记录：** 首页 → layout.tsx 加载了 `___` 组件 → 点击登录跳转到 `___`

#### Step 2：登录流程

1. 找到登录页面：`app/(auth)/login/page.tsx`
2. 找到登录表单组件：`components/user-auth-form.tsx`
3. 追踪 GitHub OAuth 的配置：`lib/auth.ts` 中的 `providers`
4. 追踪登录成功后的回调：`lib/auth.ts` 中的 `callbacks`
5. 找到 JWT 中存储了哪些用户信息

**记录：** 登录页 → `___` 组件 → NextAuth `___` provider → 回调写入 JWT `___` 字段

#### Step 3：进入 Dashboard

1. 找到 Dashboard 布局：`app/(dashboard)/dashboard/layout.tsx`
2. 确认中间件如何保护这个路由：`middleware.ts`
3. 找到 Dashboard 页面：`app/(dashboard)/dashboard/page.tsx`
4. 看看页面查询了什么数据（哪些 Prisma 查询）
5. 找到文章列表使用的组件

**记录：** 中间件检查 `___` → Dashboard 查询了 `___` → 使用 `___` 组件显示列表

#### Step 4：创建文章

1. 找到创建按钮：`components/post-create-button.tsx`
2. 追踪 API 调用：`POST /api/posts`
3. 找到 API 处理文件：`app/api/posts/route.ts`
4. 看看 API 做了什么检查（认证？文章数量限制？）
5. 找到创建成功后跳转到哪里

**记录：** 点击 New Post → `POST /api/posts` → API 检查 `___` → 创建后跳转到 `___`

#### Step 5：编辑和保存

1. 找到编辑器页面：`app/(editor)/editor/[postId]/page.tsx`
2. 追踪编辑器组件：`components/editor.tsx`
3. 找到保存 API 调用：`PATCH /api/posts/[postId]`
4. 确认内容以什么格式保存到数据库

**记录：** 编辑器加载 `___` → EditorJS 初始化 → 保存调用 `___` → 内容格式是 `___`

#### Step 6：达到免费限制

1. 在 `POST /api/posts` 中找到文章数量检查的代码
2. 确认免费用户的限制是多少篇
3. 找到返回 402 状态码的代码
4. 找到前端处理 402 的逻辑

**记录：** API 查询用户文章数 `___` → 超过 `___` 篇返回 402 → 前端显示 `___`

#### Step 7：升级 Pro

1. 找到 Billing 页面：`app/(dashboard)/dashboard/billing/page.tsx`
2. 追踪 Stripe Checkout 的创建：`app/api/users/stripe/route.ts`
3. 找到支付成功后的 Webhook 处理：`app/api/webhooks/stripe/route.ts`
4. 确认用户的 `stripePriceId` 字段如何被更新

**记录：** Billing 页 → Stripe Checkout → Webhook 更新 `___` 字段 → 用户成为 Pro

### 完成后

用你自己的话画一张完整的流程图（可以用纸笔或任何工具），标注每一步经过的文件。这就是你对整个项目的心智模型。

---

## 三、练习 2：功能设计练习（60 分钟）

### 目标

设计一个"文章分类/标签"功能的实现方案。不需要实际编码——只需要写出你会修改哪些文件、添加什么代码。

### 需求

- 每篇文章可以有一个分类（如 "Tech", "Life", "Tutorial"）
- 在 Dashboard 文章列表中显示分类
- 在编辑器中可以选择分类

### 步骤

#### Step 1：数据库层（参考 Chapter 6）

1. 打开 `prisma/schema.prisma`
2. 设计 schema 变更：给 `Post` 模型添加分类字段
3. 思考：用 `String` 字段还是创建一个新的 `Category` 模型？各有什么优缺点？

**写下你的方案：**

```prisma
// 方案 A：简单字符串字段
model Post {
  // ... 已有字段
  category  String?  @default("uncategorized")
}

// 方案 B：关联模型
model Category {
  id    String @id @default(cuid())
  name  String @unique
  posts Post[]
}
```

#### Step 2：API 层（参考 Chapter 8）

1. 找到 `app/api/posts/route.ts`（创建文章的 API）
2. 找到 `app/api/posts/[postId]/route.ts`（更新文章的 API）
3. 思考：需要修改哪些验证规则（Zod schema）？
4. 写出修改后的验证规则

**写下你的方案：**

```typescript
// lib/validations/post.ts
export const postPatchSchema = z.object({
  title: z.string().min(3).max(128).optional(),
  content: z.any().optional(),
  // 你要添加什么？
})
```

#### Step 3：UI 层（参考 Chapter 3 和 Chapter 8.5）

1. 编辑器中需要一个分类选择器——用什么组件？（Select? DropdownMenu? Input?）
2. 文章列表中需要显示分类标签——修改哪个组件？
3. 是否需要新增组件？

**写下你的方案：** 列出需要修改的文件和修改内容

### 完成后

对比你的方案和以下参考思路：

<details>
<summary>参考思路</summary>

**数据库：** 对于 MVP，方案 A（简单字符串字段）就够了。方案 B 更灵活但增加了复杂度。

**API：**
- 修改 `lib/validations/post.ts`，在 `postPatchSchema` 中添加 `category: z.string().optional()`
- 修改 `POST /api/posts` 和 `PATCH /api/posts/[postId]`，在数据库操作中包含 `category` 字段

**UI：**
- 修改 `components/editor.tsx`，在标题区域下方添加一个 `<Select>` 组件让用户选择分类
- 修改 `components/post-item.tsx`，在日期旁边显示分类标签（用 `<Badge>` 组件）
- 保存时在 API 请求体中包含 `category` 字段

</details>

---

## 四、练习 3：架构理解（30 分钟）

### 目标

画出 Taxonomy 项目的完整架构图，验证你对整个系统的理解。

### 步骤

1. 在纸上或工具中画出以下层级：
   - **前端层**：列出所有路由组（marketing, dashboard, editor, docs, auth）
   - **组件层**：列出关键组件（Editor, PostItem, EmptyPlaceholder 等）
   - **API 层**：列出所有 API 端点（/api/posts, /api/users/stripe, /api/webhooks/stripe, /api/og）
   - **数据层**：画出 Prisma 模型关系（User ↔ Post, User ↔ Account）
   - **外部服务**：标注 GitHub OAuth, Stripe, PlanetScale

2. 用箭头连接各层之间的数据流动

3. 标注哪些是 Server Components，哪些是 Client Components

### 参考架构图

<details>
<summary>查看参考</summary>

```
┌─────────────────────────────────────────────────────────┐
│                      路由层（Pages）                      │
│                                                          │
│  (marketing)   (dashboard)    (editor)    (docs)  (auth) │
│  ┌────────┐   ┌──────────┐  ┌────────┐  ┌─────┐ ┌────┐│
│  │首页    │   │文章列表  │  │EditorJS│  │文档 │ │登录││
│  │博客    │   │Billing   │  │编辑器  │  │指南 │ │注册││
│  │定价    │   │设置      │  │        │  │     │ │    ││
│  └────────┘   └──────────┘  └────────┘  └─────┘ └────┘│
│       SC            SC          SC→CC       SC      CC  │
├─────────────────────────────────────────────────────────┤
│                      API 层                              │
│                                                          │
│  /api/posts      /api/posts/[postId]                     │
│  POST(创建)      PATCH(更新) DELETE(删除)                 │
│                                                          │
│  /api/users/stripe         /api/webhooks/stripe          │
│  POST(创建Checkout)        POST(处理支付回调)              │
│                                                          │
│  /api/og                                                 │
│  GET(生成OG图片)                                          │
├─────────────────────────────────────────────────────────┤
│                    业务逻辑层                              │
│                                                          │
│  lib/auth.ts    lib/session.ts    lib/stripe.ts          │
│  (NextAuth)     (获取当前用户)     (Stripe客户端)          │
│                                                          │
│  lib/validations/    middleware.ts                        │
│  (Zod schemas)       (路由保护)                           │
├─────────────────────────────────────────────────────────┤
│                      数据层                               │
│                                                          │
│  Prisma ORM → PlanetScale (MySQL)                        │
│                                                          │
│  User ──┬── Post (1:N)                                   │
│         ├── Account (1:N, OAuth)                         │
│         └── Session (1:N)                                │
│                                                          │
│  Contentlayer → content/*.mdx (静态内容)                  │
├─────────────────────────────────────────────────────────┤
│                    外部服务                               │
│                                                          │
│  GitHub OAuth    Stripe    Vercel     Postmark           │
│  (登录)          (支付)    (部署)     (邮件)              │
└─────────────────────────────────────────────────────────┘

SC = Server Component    CC = Client Component
```

</details>

---

## 五、回顾与总结

### 技术栈清单

完成 11 天的学习后，你已经接触了以下技术：

| 类别 | 技术 | 学习日 |
|------|------|--------|
| 框架 | Next.js 13 (App Router) | Day 2 |
| 语言 | TypeScript | 贯穿全部 |
| 样式 | Tailwind CSS + CSS 变量 | Day 3, 3.5 |
| UI 库 | Radix UI (shadcn/ui) | Day 3 |
| 内容 | MDX + Contentlayer | Day 5 |
| 数据库 | Prisma + PlanetScale | Day 6 |
| 认证 | NextAuth.js | Day 7 |
| API | Route Handlers + Zod | Day 8 |
| 编辑器 | EditorJS | Day 8.5 |
| 支付 | Stripe | Day 9 |
| 部署 | Vercel | Day 10 |
| 工具 | ESLint, Prettier, CommitLint | Day 10 |

### 下一步

1. **Fork Taxonomy**，尝试添加练习 2 中设计的分类功能
2. **用 Taxonomy 作为模板**，构建你自己的项目
3. **深入学习**每个技术的官方文档
4. **关注社区**：shadcn/ui、Next.js 的更新日志

---

## 六、预计时间分配

| 环节 | 预计时间 | 说明 |
|------|---------|------|
| 练习 1：用户流程追踪 | 90 分钟 | 最重要的综合练习 |
| 练习 2：功能设计 | 60 分钟 | 跨层设计能力 |
| 练习 3：架构理解 | 30 分钟 | 画出心智模型 |
| 回顾总结 + 笔记 | 30 分钟 | 回顾整个学习历程 |
| **合计** | **约 3.5 小时** | |

建议在练习 1 和练习 2 之间安排一次较长的休息（15-20 分钟）。

---

恭喜你完成了全部学习计划！
