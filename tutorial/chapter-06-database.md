# 第6章：数据库层 — 数据是怎么存和取的？

## 学习目标

- 理解 ORM（对象关系映射）的概念
- 理解 Prisma Schema 如何定义数据模型
- 理解数据库关系（一对多、外键）
- 理解 Prisma Client 的 CRUD 操作
- 理解数据库迁移的概念

## 涉及文件

| 文件 | 说明 |
|------|------|
| `prisma/schema.prisma` | 数据模型定义 |
| `lib/db.ts` | Prisma Client 单例 |
| `app/api/posts/route.ts` | 帖子的创建查询 |
| `app/api/posts/[postId]/route.ts` | 帖子的更新删除 |
| `app/(dashboard)/dashboard/page.tsx` | 服务端直接查询数据库 |

---

## 1. 什么是 ORM？

**ORM（Object-Relational Mapping，对象关系映射）** 是一种让你用 JavaScript 对象操作数据库的工具，不需要写 SQL 语句。

```
传统方式（写 SQL）：
  SELECT * FROM posts WHERE author_id = 'abc123' ORDER BY created_at DESC;

ORM 方式（写 JavaScript）：
  db.post.findMany({
    where: { authorId: 'abc123' },
    orderBy: { createdAt: 'desc' },
  })
```

**Prisma** 就是 Taxonomy 项目使用的 ORM。它不仅可以操作数据库，还能帮你定义数据模型、做数据迁移、生成类型安全的客户端代码。

---

## 2. Prisma Schema — 数据模型定义

`prisma/schema.prisma` 是整个数据库的"蓝图"，定义了数据库中有哪些表、每个表有哪些字段、表之间的关系。

```prisma
// prisma/schema.prisma

// 生成器配置：告诉 Prisma 生成 JavaScript 客户端
generator client {
  provider = "prisma-client-js"
}

// 数据源配置：使用 MySQL 数据库
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")  // 从环境变量读取连接字符串
}

// ==================== 用户表 ====================
model User {
  id            String    @id @default(cuid())    // 主键，自动生成唯一 ID
  name          String?                            // 用户名（可选，? 表示可为空）
  email         String?   @unique                  // 邮箱（唯一）
  emailVerified DateTime?                          // 邮箱验证时间
  image         String?                            // 头像 URL
  createdAt     DateTime  @default(now()) @map(name: "created_at")  // 创建时间
  updatedAt     DateTime  @default(now()) @map(name: "updated_at")  // 更新时间

  // 关系：一个用户有多个账户、会话、帖子
  accounts Account[]
  sessions Session[]
  Post     Post[]

  // Stripe 相关字段
  stripeCustomerId       String?   @unique @map(name: "stripe_customer_id")
  stripeSubscriptionId   String?   @unique @map(name: "stripe_subscription_id")
  stripePriceId          String?   @map(name: "stripe_price_id")
  stripeCurrentPeriodEnd DateTime? @map(name: "stripe_current_period_end")

  @@map(name: "users")  // 数据库中的实际表名是 "users"
}

// ==================== 帖子表 ====================
model Post {
  id        String   @id @default(cuid())
  title     String                           // 帖子标题
  content   Json?                            // 帖子内容（JSON 格式，存 Editor.js 数据）
  published Boolean  @default(false)         // 是否发布
  createdAt DateTime @default(now()) @map(name: "created_at")
  updatedAt DateTime @default(now()) @map(name: "updated_at")
  authorId  String                           // 作者 ID（外键）

  // 关系：一个帖子属于一个用户
  author User @relation(fields: [authorId], references: [id])

  @@map(name: "posts")
}

// ==================== 账户表（OAuth 登录信息）====================
model Account {
  id                String   @id @default(cuid())
  userId            String                       // 关联的用户 ID
  type              String                       // 账户类型
  provider          String                       // 登录提供商（如 "github"）
  providerAccountId String                       // 提供商的用户 ID
  refresh_token     String?  @db.Text            // 刷新令牌
  access_token      String?  @db.Text            // 访问令牌
  expires_at        Int?                         // 过期时间
  token_type        String?
  scope             String?
  id_token          String?  @db.Text
  session_state     String?
  createdAt         DateTime @default(now()) @map(name: "created_at")
  updatedAt         DateTime @default(now()) @map(name: "updated_at")

  // 关系：一个账户属于一个用户，删除用户时级联删除账户
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([provider, providerAccountId])  // 联合唯一：同一提供商的同一用户只有一条记录
  @@map(name: "accounts")
}

// ==================== 会话表 ====================
model Session {
  id           String   @id @default(cuid())
  sessionToken String   @unique              // 会话令牌（唯一）
  userId       String
  expires      DateTime                      // 过期时间
  user         User     @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@map(name: "sessions")
}

// ==================== 验证令牌表 ====================
model VerificationToken {
  identifier String
  token      String   @unique
  expires    DateTime

  @@unique([identifier, token])
  @@map(name: "verification_tokens")
}
```

### 关键语法解释

| 语法 | 含义 | 例子 |
|------|------|------|
| `@id` | 主键 | `id String @id` |
| `@default(cuid())` | 默认值：自动生成唯一 ID | ID 自动生成 |
| `@default(now())` | 默认值：当前时间 | 创建时间自动填充 |
| `@unique` | 唯一约束 | 邮箱不能重复 |
| `?` | 可选字段（可为 null） | `name String?` |
| `@map(name: "xxx")` | 映射到数据库中的实际列名 | `createdAt` → `created_at` |
| `@@map(name: "xxx")` | 映射到数据库中的实际表名 | `User` → `users` |
| `@relation` | 定义外键关系 | 帖子关联到用户 |
| `@db.Text` | 使用数据库的 TEXT 类型（长文本） | 令牌可能很长 |
| `onDelete: Cascade` | 删除用户时，自动删除关联的数据 | 删除用户 → 删除其账户 |

### 数据关系图

```
User (用户)
 ├── 1:N → Account (账户)    一个用户可以有多个登录方式
 ├── 1:N → Session (会话)    一个用户可以有多个会话
 └── 1:N → Post (帖子)       一个用户可以有多个帖子
```

**1:N（一对多）** 的意思是：一个用户可以有多个帖子，但一个帖子只属于一个用户。在 Prisma 中：
- 在"一"的那边写 `Post[]`（数组）
- 在"多"的那边写 `author User @relation(...)`

---

## 3. Prisma Client 单例 — 为什么不直接 new？

```typescript
// lib/db.ts
import { PrismaClient } from "@prisma/client"

// 声明全局变量（避免 TypeScript 报错）
declare global {
  var cachedPrisma: PrismaClient
}

let prisma: PrismaClient

if (process.env.NODE_ENV === "production") {
  // 生产环境：直接创建新实例
  prisma = new PrismaClient()
} else {
  // 开发环境：使用全局缓存
  if (!global.cachedPrisma) {
    global.cachedPrisma = new PrismaClient()
  }
  prisma = global.cachedPrisma
}

export const db = prisma
```

**为什么开发环境要缓存？**

Next.js 开发服务器有一个特性叫 **Hot Module Replacement（热模块替换）**——每次你修改代码，它会重新加载模块。如果每次都创建新的 `PrismaClient`，很快就会耗尽数据库连接数（每个 PrismaClient 会创建多个数据库连接）。

通过把 PrismaClient 存在全局变量 `global.cachedPrisma` 上，即使模块重新加载，也会复用同一个实例。

---

## 4. CRUD 操作实战

CRUD 是数据库操作的四个基本动作：**C**reate（创建）、**R**ead（读取）、**U**pdate（更新）、**D**elete（删除）。

### 4.1 Create（创建）

```typescript
// app/api/posts/route.ts 中的 POST 函数
const post = await db.post.create({
  data: {
    title: "Untitled Post",     // 帖子标题
    content: body.content,       // 帖子内容
    authorId: session.user.id,   // 作者 ID（关联到当前用户）
  },
  select: {
    id: true,   // 只返回新帖子的 ID
  },
})
```

### 4.2 Read（读取）

```typescript
// app/(dashboard)/dashboard/page.tsx 中
const posts = await db.post.findMany({
  where: {
    authorId: user.id,       // 条件：只查当前用户的帖子
  },
  select: {
    id: true,                // 只查这些字段（性能优化）
    title: true,
    published: true,
    createdAt: true,
  },
  orderBy: {
    updatedAt: "desc",       // 按更新时间倒序
  },
})
```

**`select` 的作用：** 不选择所有字段，只选需要的。这就像去超市只买需要的东西，而不是把整个货架搬回家。

### 4.3 Update（更新）

```typescript
// app/api/posts/[postId]/route.ts 中的 PATCH 函数
await db.post.update({
  where: {
    id: params.postId,       // 条件：指定帖子 ID
  },
  data: {
    title: body.title,       // 新标题
    content: body.content,   // 新内容
  },
})
```

### 4.4 Delete（删除）

```typescript
// app/api/posts/[postId]/route.ts 中的 DELETE 函数
await db.post.delete({
  where: {
    id: params.postId,       // 条件：指定帖子 ID
  },
})
```

### 4.5 Count（计数）

```typescript
// 用于检查免费用户是否超过帖子限制
const count = await db.post.count({
  where: {
    authorId: user.id,
  },
})

if (count >= 3) {
  throw new RequiresProPlanError()  // 免费用户最多 3 篇帖子
}
```

---

## 5. 在 Server Component 中直接查数据库

这是 Next.js 13 最强大的特性之一——Server Component 可以直接查数据库，不需要写 API：

```tsx
// app/(dashboard)/dashboard/page.tsx
export default async function DashboardPage() {
  const user = await getCurrentUser()

  if (!user) {
    redirect("/login")
  }

  // 直接在组件中查数据库！
  const posts = await db.post.findMany({
    where: { authorId: user.id },
    select: { id: true, title: true, published: true, createdAt: true },
    orderBy: { updatedAt: "desc" },
  })

  return (
    <div>
      {posts.map((post) => (
        <PostItem key={post.id} post={post} />
      ))}
    </div>
  )
}
```

**为什么这样做是安全的？**

因为 Server Component 的代码**只在服务器上运行**，不会发送到浏览器。用户在浏览器的开发者工具中看不到数据库查询代码，也无法直接调用 `db.post.findMany()`。

**对比传统方式：**

```
传统方式（需要 API 中转）：
  浏览器 → 调用 /api/posts → API 查数据库 → 返回 JSON → 浏览器渲染

Server Component 方式（直接查）：
  服务器上的组件直接查数据库 → 渲染 HTML → 发送给浏览器
```

---

## 6. 数据库权限验证

看看项目如何确保用户只能操作自己的数据：

```typescript
// app/api/posts/[postId]/route.ts

// 验证当前用户是否有权限操作这篇帖子
async function verifyCurrentUserHasAccessToPost(postId: string) {
  const session = await getServerSession(authOptions)

  // 查询：是否存在一篇帖子，同时满足：
  // 1. 帖子 ID 匹配
  // 2. 作者 ID 是当前用户
  const count = await db.post.count({
    where: {
      id: postId,
      authorId: session?.user.id,
    },
  })

  return count > 0  // count > 0 说明有权限
}

// 在 DELETE 操作中使用
export async function DELETE(req, context) {
  const { params } = routeContextSchema.parse(context)

  // 先检查权限
  if (!(await verifyCurrentUserHasAccessToPost(params.postId))) {
    return new Response(null, { status: 403 })  // 403 = 禁止访问
  }

  // 有权限才删除
  await db.post.delete({
    where: { id: params.postId },
  })

  return new Response(null, { status: 204 })  // 204 = 删除成功，无内容
}
```

---

## 7. 数据库迁移

当你修改了 `schema.prisma`（比如添加新字段），需要"迁移"数据库以应用变更：

```bash
# 创建迁移（生成 SQL 文件）
npx prisma migrate dev --name add_new_field

# 生成新的 Prisma Client（更新类型定义）
npx prisma generate

# 查看数据库内容（可视化工具）
npx prisma studio
```

`prisma/migrations/` 文件夹保存了所有迁移记录，就像数据库的"版本历史"。

---

## 小练习

### 练习 1：理解数据模型
画出 User、Post、Account、Session 四个表的关系图。标注出外键和关系类型（1:N）。

### 练习 2：阅读 Prisma 查询
看 `app/(dashboard)/dashboard/page.tsx` 中的 `db.post.findMany` 查询，回答：
1. 它查的是哪张表？
2. 查询条件是什么？
3. 返回了哪些字段？
4. 排序方式是什么？

### 练习 3：使用 Prisma Studio
运行 `npx prisma studio`，在浏览器中查看数据库内容。尝试在 Studio 中手动添加一条数据。

### 练习 4：思考
为什么 `lib/db.ts` 中开发环境要用全局缓存？如果不缓存会发生什么？

---

## 本章小结

- Prisma 是一个 ORM，让你用 JavaScript 操作数据库
- `schema.prisma` 定义数据模型和表关系
- Prisma Client 单例模式避免开发环境连接泄漏
- CRUD 操作对应 `create`、`findMany`、`update`、`delete` 方法
- Server Component 可以直接查数据库，代码不会暴露给用户
- 权限验证确保用户只能操作自己的数据

下一章我们将学习用户认证系统。
