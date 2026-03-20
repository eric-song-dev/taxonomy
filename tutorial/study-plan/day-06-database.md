# Day 06 — 数据库层：数据是怎么存和取的？

> 对应章节：[Chapter 6: 数据库](../chapter-06-database.md)

---

## 今日学习目标

1. **读懂 Prisma Schema**：理解 `schema.prisma` 如何定义表结构、字段类型和表之间的关系
2. **掌握 CRUD 四大操作**：能看懂并仿写 [create](https://www.prisma.io/docs/orm/reference/prisma-client-reference#create)、[findMany](https://www.prisma.io/docs/orm/reference/prisma-client-reference#findmany)、[update](https://www.prisma.io/docs/orm/reference/prisma-client-reference#update)、[delete](https://www.prisma.io/docs/orm/reference/prisma-client-reference#delete) 数据库操作代码
3. **理解 ORM 的价值**：明白为什么不直接写 SQL，而是用 Prisma 这样的工具来操作数据库

---

## 前置知识速补

数据库操作涉及很多异步编程和 JS 高级语法。如果你是零基础，这一节**务必先读完再往下走**，否则后面的代码会看不懂。

### 1. Promise 概念和 `.then()` 链

数据库操作需要时间（毕竟要去硬盘或远程服务器读数据），JavaScript 不会傻等，而是用 **[Promise](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Promise)**（承诺）来处理这种"要等一会儿才有结果"的操作。

```javascript
// 想象你去奶茶店点单，店员给你一张取餐号（Promise）
// 你不用站在柜台傻等，可以先去找座位

// Promise 的基本用法：
const 取餐号 = fetch("/api/posts")  // 发起请求，拿到一个 Promise

取餐号.then((response) => {
  // .then() 意思是"等好了以后，执行这个函数"
  console.log("拿到数据了！", response)
})

// 可以链式调用：
fetch("/api/posts")
  .then((response) => response.json())   // 第一步：把响应转成 JSON
  .then((data) => console.log(data))      // 第二步：打印数据
  .catch((error) => console.log("出错了", error))  // 出错时执行
```

**关键理解**：`.then()` 里的函数不会立刻执行，而是"等前面的操作完成后"才执行。就像你把电话号码留给奶茶店，等做好了店员打电话通知你。

### 2. async/await 深入 — 为什么数据库操作是异步的

`.then()` 链写多了会很难读，所以 JavaScript 发明了 [async/await](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/async_function) 语法，让异步代码看起来像同步代码：

```javascript
// .then() 写法（嵌套多了很难读）：
db.post.findMany({ where: { authorId: "abc" } })
  .then((posts) => {
    console.log(posts)
  })

// async/await 写法（清爽！）：
async function getPosts() {
  const posts = await db.post.findMany({ where: { authorId: "abc" } })
  //            ^^^^^ await 意思是"等这个操作完成，把结果赋给 posts"
  console.log(posts)
}
```

**为什么数据库操作必须是异步的？** 因为数据库可能在远程服务器上，网络传输需要时间。如果用同步方式，整个程序就会卡住等待，其他用户就没法用了。用 `await` 等待时，程序可以去处理其他请求。

**重要规则**：
- `await` 只能写在 `async` 函数里面
- 忘记写 `await` 是非常常见的 bug，你拿到的会是 Promise 对象而不是实际数据

```javascript
// 错误示范 —— 忘记写 await
async function getPosts() {
  const posts = db.post.findMany()  // 少了 await！
  console.log(posts)  // 输出: Promise { <pending> }  不是你想要的数据！
}

// 正确写法
async function getPosts() {
  const posts = await db.post.findMany()  // 加上 await
  console.log(posts)  // 输出: [{ id: "xxx", title: "Hello" }, ...]  这才是数据
}
```

### 3. try/catch 错误处理

数据库操作可能出错（网络断了、数据格式不对等），需要用 [try/catch](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/try...catch) 捕获错误：

```javascript
async function createPost() {
  try {
    // try 块里放"可能出错的代码"
    const post = await db.post.create({
      data: { title: "Hello", authorId: "abc" }
    })
    console.log("创建成功！", post)
  } catch (error) {
    // 如果 try 块里的代码出错，就跳到这里
    console.log("创建失败！", error.message)
    // error.message 是错误的描述文字
  }
}
```

**类比**：`try/catch` 就像"小心翼翼地做一件事，如果搞砸了就执行备用方案"。没有 `try/catch`，程序出错就会直接崩溃。

### 4. 对象[展开运算符](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Spread_syntax) `{ ...obj, newProp: value }`

三个点 `...` 可以把一个对象"展开"，然后覆盖或追加属性：

```javascript
const user = { name: "小明", age: 20 }

// 展开 user 的所有属性，然后覆盖 age、添加 email
const updatedUser = { ...user, age: 21, email: "xm@test.com" }
// 结果: { name: "小明", age: 21, email: "xm@test.com" }

// 在数据库操作中经常这样用：
const data = { ...body, authorId: session.user.id }
// 把请求体的所有字段展开，再加上 authorId
```

### 5. [可选链](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Operators/Optional_chaining) `?.` 和 null vs undefined

```javascript
// 可选链 ?.  —— 安全地访问可能不存在的属性
const user = null
console.log(user.name)    // 报错！Cannot read property 'name' of null
console.log(user?.name)   // 输出: undefined （不报错，安全！）

// 在项目中经常见到：
session?.user.id   // 如果 session 是 null，直接返回 undefined，不报错

// null vs undefined 的区别：
let a = null       // null = 明确表示"没有值"（人为设置的）
let b = undefined  // undefined = "还没赋值"（系统默认的）
let c              // c 也是 undefined

// 在 Prisma Schema 中，字段名后面加 ? 表示可为 null：
// name String?   --> name 的值可以是 null
```

---

## 核心概念清单

在开始读源码之前，先快速了解这些概念。不需要死记硬背，遇到的时候回来查就行。

| 概念 | 英文 | 一句话解释 | 类比 |
|------|------|-----------|------|
| [ORM](https://www.prisma.io/docs/orm/overview/introduction/what-is-prisma) | Object-Relational Mapping | 用 JS 对象操作数据库，不用写 SQL | 翻译官，帮你把 JS 翻译成数据库语言 |
| Prisma | Prisma | 本项目使用的 ORM 工具 | 一位具体的翻译官 |
| [Schema](https://www.prisma.io/docs/orm/prisma-schema/overview) | Schema | 数据库的"蓝图"，定义有哪些表和字段 | 房屋设计图纸 |
| Migration | Migration（迁移） | 把 Schema 的变更应用到真实数据库 | 按照新图纸改造房屋 |
| CRUD | Create/Read/Update/Delete | 数据库四大基本操作：增、查、改、删 | 就像记事本的"新建、查看、编辑、删除" |
| [Relations](https://www.prisma.io/docs/orm/prisma-schema/data-model/relations) | Relations（关系） | 表和表之间的关联（如用户有多个帖子） | 家庭关系：一个爸爸可以有多个孩子 |
| 外键 | Foreign Key | 一个表中存另一个表的 ID，用来建立关联 | 快递单上写的收件人手机号（指向另一个人） |
| 主键 | Primary Key | 每条记录的唯一标识（通常是 `id` 字段） | 身份证号码，每个人不同 |

---

## 源码精读

### 第一站：prisma/schema.prisma — 数据库的蓝图

这个文件定义了整个数据库的结构。我们逐段来读。

**文件位置**：`prisma/schema.prisma`

#### 配置部分

```prisma
// 告诉 Prisma：请帮我生成 JavaScript/TypeScript 客户端代码
generator client {
  provider = "prisma-client-js"
}

// 告诉 Prisma：我用的是 MySQL 数据库，连接地址从环境变量读取
datasource db {
  provider = "mysql"
  url      = env("DATABASE_URL")
  // env("DATABASE_URL") 会去 .env 文件里找 DATABASE_URL 这个变量
  // 例如: DATABASE_URL="mysql://user:password@localhost:3306/mydb"
}
```

#### User 模型 — 用户表

```prisma
model User {
  // -------- 基本字段 --------
  id            String    @id @default(cuid())
  //            ^^^^^^    ^^^  ^^^^^^^^^^^^^^^
  //            类型:字符串  主键  默认值:自动生成唯一ID
  //
  // cuid() 生成的 ID 长这样: "clh1234abcd5678efgh"

  name          String?
  //                  ^ 问号表示"可选"，这个字段可以是 null（没有值）

  email         String?   @unique
  //                      ^^^^^^^ 唯一约束：不能有两个用户用同一个邮箱

  emailVerified DateTime?            // 邮箱是否验证过，验证过就存验证时间
  image         String?              // 头像 URL

  createdAt     DateTime  @default(now()) @map(name: "created_at")
  //                      ^^^^^^^^^^^^^^  ^^^^^^^^^^^^^^^^^^^^^^^^
  //                      默认值:当前时间   数据库里的列名叫 created_at
  //                                       （JS 用驼峰 createdAt，数据库用下划线 created_at）

  updatedAt     DateTime  @default(now()) @map(name: "updated_at")

  // -------- 关系字段 --------
  accounts Account[]     // 一个用户有多个 Account（登录账户）
  sessions Session[]     // 一个用户有多个 Session（会话）
  Post     Post[]        // 一个用户有多个 Post（帖子）
  //       ^^^^^^
  //       Post[] 中的 [] 表示"数组"，即多个

  // -------- Stripe 支付相关 --------
  stripeCustomerId       String?   @unique @map(name: "stripe_customer_id")
  stripeSubscriptionId   String?   @unique @map(name: "stripe_subscription_id")
  stripePriceId          String?   @map(name: "stripe_price_id")
  stripeCurrentPeriodEnd DateTime? @map(name: "stripe_current_period_end")

  @@map(name: "users")
  // ^^ 两个 @：这是表级别的设置
  // 意思是：这个 model 叫 User，但数据库里的表名是 "users"
}
```

#### Post 模型 — 帖子表

```prisma
model Post {
  id        String   @id @default(cuid())
  title     String                           // 标题（必填，没有 ? 号）
  content   Json?                            // 内容，JSON 格式，可以为空
  published Boolean  @default(false)         // 是否已发布，默认为 false
  createdAt DateTime @default(now()) @map(name: "created_at")
  updatedAt DateTime @default(now()) @map(name: "updated_at")
  authorId  String                           // 作者的用户 ID —— 这就是"外键"！

  author User @relation(fields: [authorId], references: [id])
  //     ^^^^ ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
  //     类型  @relation 定义关系：
  //           fields: [authorId]   → Post 表里的 authorId 字段
  //           references: [id]     → 对应 User 表里的 id 字段
  //           简单说：authorId 存的是某个 User 的 id

  @@map(name: "posts")
}
```

**外键 authorId 怎么理解？** 想象每篇帖子上都贴了一张标签，标签上写着"作者是 xxx"。这个 `authorId` 就是那张标签，它存的是 User 表中某个用户的 `id`。通过这个 `id`，你就能找到帖子的作者是谁。

#### Account 模型 — 登录账户表

```prisma
model Account {
  id                String   @id @default(cuid())
  userId            String                        // 外键，关联到 User
  type              String                        // 账户类型
  provider          String                        // 登录提供商，如 "github"
  providerAccountId String                        // 在 GitHub 那边的用户 ID
  refresh_token     String?  @db.Text             // @db.Text 表示长文本类型
  access_token      String?  @db.Text
  expires_at        Int?
  token_type        String?
  scope             String?
  id_token          String?  @db.Text
  session_state     String?
  createdAt         DateTime @default(now()) @map(name: "created_at")
  updatedAt         DateTime @default(now()) @map(name: "updated_at")

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  //                                                      ^^^^^^^^^^^^^^^^
  //   onDelete: Cascade 的意思是：如果删除了这个 User，
  //   那么它所有的 Account 也自动删除（级联删除）
  //   类比：注销手机号后，绑定的所有服务也自动解绑

  @@unique([provider, providerAccountId])
  // 联合唯一约束：同一个 provider（如 github）+ 同一个 providerAccountId
  // 只能有一条记录。防止同一个 GitHub 账号被绑定两次。

  @@map(name: "accounts")
}
```

### 第二站：lib/db.ts — 数据库连接的"管家"

**文件位置**：`lib/db.ts`

这个文件很短，但非常重要。我们逐行解读：

```typescript
import { PrismaClient } from "@prisma/client"
// 从 Prisma 库导入 PrismaClient 类
// PrismaClient 是你和数据库沟通的工具

declare global {
  // eslint-disable-next-line no-var
  var cachedPrisma: PrismaClient
}
// [declare global](https://www.typescriptlang.org/docs/handbook/declaration-merging.html#global-augmentation) 是告诉 TypeScript：
// "我在全局作用域上加了一个 cachedPrisma 变量，你别报错"
// 这只是类型声明，不会真的创建变量

let prisma: PrismaClient

if (process.env.NODE_ENV === "production") {
  // 生产环境（部署上线后）：直接创建新实例
  prisma = new PrismaClient()
} else {
  // 开发环境：使用全局缓存
  if (!global.cachedPrisma) {
    // 如果全局还没有缓存的实例，就创建一个
    global.cachedPrisma = new PrismaClient()
  }
  // 无论如何，都用缓存的那个实例
  prisma = global.cachedPrisma
}

export const db = prisma
// 导出为 db，其他文件 import { db } from "@/lib/db" 就能用了
```

**为什么开发环境需要缓存？**

Next.js 开发时有一个叫 **Hot Module Replacement（HMR，热模块替换）** 的功能。你每次保存代码文件，Next.js 就会重新加载那个模块。如果每次重新加载都 `new PrismaClient()`，每个 PrismaClient 实例会建立好几条数据库连接。保存几十次后，连接数就爆了，你会看到一条报错：

```
Error: Too many database connections
```

解决方案就是把 PrismaClient 存到 `global`（全局对象）上。`global` 不受 HMR 影响，模块重新加载后还能找到之前那个实例。

### 第三站：app/api/posts/route.ts — CRUD 之 Create 和 Read

**文件位置**：`app/api/posts/route.ts`

#### GET — 查询帖子列表（Read）

```typescript
export async function GET() {
  try {
    const session = await getServerSession(authOptions)
    // 获取当前登录用户的会话信息

    if (!session) {
      return new Response("Unauthorized", { status: 403 })
      // 没登录？返回 403（禁止访问）
    }

    const { user } = session
    // 解构赋值：从 session 对象中取出 user
    // 等同于 const user = session.user

    const posts = await db.post.findMany({
    // db.post        → 操作 Post 表
    // .findMany()    → 查询多条记录
      select: {
        id: true,           // 只返回这4个字段
        title: true,        // select 的好处：减少数据传输量
        published: true,    // 不需要的字段（如 content）不查
        createdAt: true,
      },
      where: {
        authorId: user.id,  // 条件：只查"当前用户"的帖子
      },
    })

    return new Response(JSON.stringify(posts))
    // 把查询结果转成 JSON 字符串返回
  } catch (error) {
    return new Response(null, { status: 500 })
    // 出错了返回 500（服务器内部错误）
  }
}
```

#### POST — 创建新帖子（Create）

```typescript
export async function POST(req: Request) {
  try {
    const session = await getServerSession(authOptions)

    if (!session) {
      return new Response("Unauthorized", { status: 403 })
    }

    const { user } = session
    const subscriptionPlan = await getUserSubscriptionPlan(user.id)

    // 免费用户最多只能创建 3 篇帖子
    if (!subscriptionPlan?.isPro) {
      //                 ^^ 可选链！如果 subscriptionPlan 是 null/undefined，
      //                    不会报错，?.isPro 直接返回 undefined
      const count = await db.post.count({
        where: { authorId: user.id },
      })
      // db.post.count() 返回匹配条件的记录数量

      if (count >= 3) {
        throw new RequiresProPlanError()
        // throw 抛出错误，代码跳到下面的 catch 块
      }
    }

    const json = await req.json()         // 读取请求体中的 JSON 数据
    const body = postCreateSchema.parse(json) // 用 Zod 验证数据格式

    const post = await db.post.create({
    // db.post.create() → 在 Post 表中创建一条新记录
      data: {
        title: body.title,            // 标题来自请求
        content: body.content,        // 内容来自请求
        authorId: session.user.id,    // 作者 ID 来自当前登录用户
      },
      select: {
        id: true,   // 创建后只返回新帖子的 id
      },
    })

    return new Response(JSON.stringify(post))
  } catch (error) {
    if (error instanceof z.ZodError) {
      // 如果是数据验证错误（格式不对）
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }

    if (error instanceof RequiresProPlanError) {
      // 如果是免费用户超过限制
      return new Response("Requires Pro Plan", { status: 402 })
    }

    return new Response(null, { status: 500 })
  }
}
```

### 第四站：app/api/posts/[postId]/route.ts — CRUD 之 Update 和 Delete

**文件位置**：`app/api/posts/[postId]/route.ts`

#### PATCH — 更新帖子（Update）

```typescript
export async function PATCH(
  req: Request,
  context: z.infer<typeof routeContextSchema>
  // context 里面有 params.postId，就是 URL 中的帖子 ID
) {
  try {
    const { params } = routeContextSchema.parse(context)

    // 权限检查：这篇帖子是当前用户的吗？
    if (!(await verifyCurrentUserHasAccessToPost(params.postId))) {
      return new Response(null, { status: 403 })
    }

    const json = await req.json()
    const body = postPatchSchema.parse(json)

    await db.post.update({
    // db.post.update() → 更新一条记录
      where: {
        id: params.postId,      // 条件：帖子 ID
      },
      data: {
        title: body.title,      // 要更新的字段
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

#### DELETE — 删除帖子（Delete）

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
    // db.post.delete() → 删除一条记录
      where: {
        id: params.postId as string,
      },
    })

    return new Response(null, { status: 204 })
    // 204 = No Content，删除成功但不返回内容
  } catch (error) {
    if (error instanceof z.ZodError) {
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }
    return new Response(null, { status: 500 })
  }
}
```

#### 权限验证函数

```typescript
async function verifyCurrentUserHasAccessToPost(postId: string) {
  const session = await getServerSession(authOptions)

  const count = await db.post.count({
    where: {
      id: postId,                  // 帖子 ID 匹配
      authorId: session?.user.id,  // 并且作者是当前用户
      //        ^^^^^^^^ 可选链：session 可能为 null
    },
  })

  return count > 0
  // 如果 count > 0，说明存在一篇"ID 匹配且是当前用户写的"帖子 → 有权限
  // 如果 count === 0，说明帖子不存在或者不是当前用户的 → 没权限
}
```

### 第五站：app/(dashboard)/dashboard/page.tsx — Server Component 直接查数据库

**文件位置**：`app/(dashboard)/dashboard/page.tsx`

这是 Next.js 最酷的特性之一 —— [Server Component](https://react.dev/reference/rsc/server-components) 可以直接在组件里查数据库！

```tsx
export default async function DashboardPage() {
  //            ^^^^^ 注意这个 async！Server Component 可以是异步函数
  const user = await getCurrentUser()

  if (!user) {
    redirect(authOptions?.pages?.signIn || "/login")
    // 没登录就跳转到登录页
  }

  // 直接在组件里查数据库！不需要写 API！
  const posts = await db.post.findMany({
    where: {
      authorId: user.id,
    },
    select: {
      id: true,
      title: true,
      published: true,
      createdAt: true,
    },
    orderBy: {
      updatedAt: "desc",   // 按更新时间倒序（最新的在前面）
    },
  })

  return (
    <DashboardShell>
      {/* 把查到的帖子渲染成列表 */}
      {posts?.length ? (
        <div>
          {posts.map((post) => (
            <PostItem key={post.id} post={post} />
          ))}
        </div>
      ) : (
        <EmptyPlaceholder>
          {/* 没有帖子时显示空状态 */}
        </EmptyPlaceholder>
      )}
    </DashboardShell>
  )
}
```

**为什么 Server Component 可以直接查数据库？安全吗？**

安全！因为 Server Component 的代码**只在服务器上运行**，永远不会发送到用户的浏览器。用户看到的是渲染后的 HTML，看不到你的数据库查询代码。

```
传统方式（绕一大圈）：
  浏览器 → 发请求到 /api/posts → API 查数据库 → 返回 JSON → 浏览器拿到 JSON → 渲染页面

Server Component 方式（直达）：
  服务器上的组件 → 直接查数据库 → 直接渲染 HTML → 发给浏览器
```

---

## 动手练习

### 练习 1：在纸上画出数据库关系图（30 分钟）

**目标**：通过手绘加深对数据表关系的理解。

**步骤**：

1. 打开 `prisma/schema.prisma`，通读一遍
2. 准备一张纸（或打开任何画图工具），画出以下内容：
   - 画 4 个方框，分别标注 `User`、`Post`、`Account`、`Session`
   - 在每个方框里列出关键字段（不需要全部，挑 3-5 个重要的）
   - 用箭头连接有关系的表，标注 `1:N`（一对多）
   - 在箭头旁标注外键字段名（如 `authorId`）
3. 回答以下问题（写在纸上）：
   - 如果删除一个 User，哪些表的数据会被级联删除？哪些不会？
   - 为什么 Post 表没有 `onDelete: Cascade`？这意味着什么？

**排错提示**：如果你分不清哪边是"一"哪边是"多"，看看字段类型。`Post[]` 带 `[]` 的是"多"的那边，`User`（不带 `[]`）是"一"的那边。

### 练习 2：模拟 CRUD 操作（45 分钟）

**目标**：不运行代码，只用纸笔模拟数据库操作的输入和输出。

**步骤**：

1. 假设数据库里有这些数据：

   ```
   User 表:
   | id      | name  | email          |
   |---------|-------|----------------|
   | "u001"  | 小明  | xm@test.com    |
   | "u002"  | 小红  | xh@test.com    |

   Post 表:
   | id      | title      | published | authorId |
   |---------|------------|-----------|----------|
   | "p001"  | 我的日记    | true      | "u001"   |
   | "p002"  | 学习笔记    | false     | "u001"   |
   | "p003"  | 旅行见闻    | true      | "u002"   |
   ```

2. 写出以下操作的结果：

   ```javascript
   // 操作 A：查询小明的所有帖子
   await db.post.findMany({
     where: { authorId: "u001" },
     select: { id: true, title: true },
   })
   // 结果是？____________________

   // 操作 B：统计小红的帖子数量
   await db.post.count({
     where: { authorId: "u002" },
   })
   // 结果是？____________________

   // 操作 C：创建一篇新帖子
   await db.post.create({
     data: { title: "新文章", authorId: "u001", published: false },
   })
   // 执行后 Post 表变成什么样？____________________

   // 操作 D：更新帖子标题
   await db.post.update({
     where: { id: "p002" },
     data: { title: "TypeScript 学习笔记" },
   })
   // 执行后 p002 这条记录变成什么样？____________________

   // 操作 E：删除帖子
   await db.post.delete({
     where: { id: "p003" },
   })
   // 执行后 Post 表还剩哪些记录？____________________
   ```

**排错提示**：`select` 只影响返回的字段，不影响查询条件。`where` 是过滤条件。

### 练习 3：阅读真实源码，回答问题（30 分钟）

**目标**：锻炼读真实代码的能力。

打开 `app/api/posts/route.ts`，回答以下问题：

1. `POST` 函数中，免费用户的帖子数量限制是多少？是怎么检查的？
2. `POST` 函数中有几个 `catch` 分支？分别处理什么错误？
3. 如果一个没有登录的用户发 POST 请求，会得到什么响应？
4. `postCreateSchema` 是做什么的？为什么需要它？

打开 `app/api/posts/[postId]/route.ts`，回答：

5. `verifyCurrentUserHasAccessToPost` 函数是怎么判断权限的？
6. 如果用户 A 尝试删除用户 B 的帖子，会发生什么？

**排错提示**：如果你看不懂 `z.object({ ... })` 的写法，那是 Zod 库的语法，我们在后面的章节会详细学。现在只要知道"它用来检查数据格式是否正确"就行了。

---

## 概念图解

### 数据库关系图

```
┌─────────────────────────┐
│         User            │
│─────────────────────────│
│  id (PK)                │
│  name                   │
│  email (UNIQUE)         │
│  createdAt              │
│  stripeCustomerId       │
│  ...                    │
└────┬──────┬──────┬──────┘
     │      │      │
     │ 1:N  │ 1:N  │ 1:N
     │      │      │
     ▼      ▼      ▼
┌─────────┐ ┌─────────┐ ┌─────────────┐
│  Post   │ │ Session │ │   Account   │
│─────────│ │─────────│ │─────────────│
│ id (PK) │ │ id (PK) │ │ id (PK)     │
│ title   │ │ token   │ │ provider    │
│ content │ │ userId  │ │ userId (FK) │
│authorId │ │ (FK)    │ │ access_token│
│ (FK)    │ │ expires │ │ onDelete:   │
│published│ │onDelete:│ │  Cascade    │
└─────────┘ │ Cascade │ └─────────────┘
            └─────────┘

PK = Primary Key（主键）
FK = Foreign Key（外键）
1:N = 一对多关系
```

### ORM 工作流程

```
你写的 JavaScript 代码          Prisma ORM（翻译官）           数据库（MySQL）
─────────────────────          ──────────────────           ───────────────

db.post.findMany({      →     SELECT id, title             →    在硬盘/内存中
  where: {authorId:"u1"},      FROM posts                       查找数据
  select: {id:true,            WHERE author_id = 'u1'
    title:true},               ORDER BY updated_at DESC;
  orderBy: {updatedAt:"desc"}
})

        ↑                              ↑                          ↑
  JS 对象语法，好写好读         自动翻译成 SQL 语句          执行 SQL，返回结果
                               还帮你做类型检查！
```

### Prisma Client 单例模式

```
开发环境（不用缓存的问题）：

  保存文件 → HMR 重新加载 → new PrismaClient() → 5 个连接
  保存文件 → HMR 重新加载 → new PrismaClient() → 又 5 个连接
  保存文件 → HMR 重新加载 → new PrismaClient() → 又 5 个连接
  ...
  保存 20 次后 → 100 个连接 → 数据库：连接数爆了！

开发环境（用缓存）：

  第一次加载 → new PrismaClient() → 存到 global → 5 个连接
  保存文件 → HMR 重新加载 → global 已经有了，复用 → 还是 5 个连接
  保存文件 → HMR 重新加载 → global 已经有了，复用 → 还是 5 个连接
  ...
  保存 100 次 → 始终只有 5 个连接 → 数据库：谢谢你！
```

---

## 自检问题

完成今天的学习后，试着不看笔记回答以下问题。答不出来也没关系，翻回去看看对应的章节。

### 问题 1（基础）
`prisma/schema.prisma` 中，`String?` 末尾的 `?` 表示什么意思？

<details>
<summary>点击查看答案</summary>

`?` 表示这个字段是**可选的（optional）**，它的值可以是 `null`（没有值）。比如 `name String?` 意味着用户可以没有名字。而 `title String`（没有 `?`）表示标题是必填的，不能为 `null`。
</details>

### 问题 2（理解）
`lib/db.ts` 中为什么开发环境要用 `global.cachedPrisma` 缓存？不缓存会怎样？

<details>
<summary>点击查看答案</summary>

因为 Next.js 开发时的 **Hot Module Replacement（HMR）** 会在每次保存文件时重新加载模块。如果不缓存，每次重新加载都会 `new PrismaClient()`，创建新的数据库连接。保存多次后，连接数会耗尽，报错 `Too many database connections`。把实例缓存到 `global` 上，HMR 重新加载时可以复用已有的实例。
</details>

### 问题 3（应用）
写出一条 Prisma 查询：查找所有 `published` 为 `true` 的帖子，只返回 `id` 和 `title`，按 `createdAt` 倒序排列。

<details>
<summary>点击查看答案</summary>

```typescript
const posts = await db.post.findMany({
  where: {
    published: true,
  },
  select: {
    id: true,
    title: true,
  },
  orderBy: {
    createdAt: "desc",
  },
})
```
</details>

### 问题 4（分析）
在 `app/(dashboard)/dashboard/page.tsx` 中，为什么可以直接写 `await db.post.findMany(...)` 而不需要通过 API？这样做安全吗？

<details>
<summary>点击查看答案</summary>

因为这个组件是 **Server Component**，它的代码只在服务器上运行，不会被发送到用户的浏览器。用户看到的只是渲染后的 HTML，看不到数据库查询代码，也无法直接调用 `db.post.findMany()`。所以这样做是安全的。这也是 Next.js App Router 的核心优势之一：省去了"组件 → API → 数据库"的中间环节，直接"组件 → 数据库"。
</details>

### 问题 5（进阶）
如果要给 Post 模型新增一个 `category` 字段（字符串，可选），需要做哪些步骤？

<details>
<summary>点击查看答案</summary>

1. 修改 `prisma/schema.prisma`，在 Post 模型中添加 `category String?`
2. 运行 `npx prisma migrate dev --name add_post_category` 创建数据库迁移
3. 运行 `npx prisma generate` 重新生成 Prisma Client（`migrate dev` 通常会自动执行这一步）
4. 现在就可以在代码中使用 `db.post.create({ data: { ..., category: "tech" } })` 了

如果只是本地开发测试，也可以用 `npx prisma db push` 代替 `migrate dev`，它直接把 Schema 变更推送到数据库，但不会生成迁移文件。
</details>

---

## 预计时间分配

| 环节 | 时间 | 建议 |
|------|------|------|
| 前置知识速补 | 40 分钟 | 如果 async/await 已经熟悉，可以跳过或快速浏览 |
| 源码精读（schema + db.ts） | 45 分钟 | 逐行对照注释阅读，边读边在纸上记笔记 |
| 源码精读（CRUD 操作） | 40 分钟 | 重点理解 where、select、data 三个参数 |
| 练习 1（画关系图） | 30 分钟 | 不要用工具画，手绘更有助于记忆 |
| 练习 2（模拟 CRUD） | 45 分钟 | 用纸笔模拟，不要急着看答案 |
| 练习 3（读源码答题） | 30 分钟 | 试着独立回答，答不上再去看代码 |
| 自检与笔记整理 | 20 分钟 | 把今天学到的关键概念用自己的话写一遍 |
| **合计** | **约 4 小时** | 如果超时不要焦虑，数据库内容量本身就大 |

---

## 常见踩坑点

### 1. 忘记写 await

这是新手最最最常见的错误。数据库操作是异步的，必须加 `await`。

```javascript
// 错误 —— 忘记 await
const posts = db.post.findMany()
console.log(posts)  // Promise { <pending> } 而不是实际数据！

// 正确
const posts = await db.post.findMany()
console.log(posts)  // [{ id: "...", title: "..." }, ...]
```

**排查方法**：如果你拿到的数据显示 `[object Promise]` 或 `Promise { <pending> }`，99% 是忘记写 `await` 了。

### 2. DATABASE_URL 环境变量没配置

Prisma 需要知道数据库的地址。如果 `.env` 文件里没有 `DATABASE_URL`，启动项目时会报错。

```bash
# .env 文件中需要有这一行（格式取决于你用的数据库）：
DATABASE_URL="mysql://user:password@localhost:3306/taxonomy"
```

### 3. 修改 Schema 后忘记同步数据库

改了 `schema.prisma` 之后，数据库不会自动更新。你需要手动运行命令：

```bash
# 开发环境：推送 Schema 变更到数据库
npx prisma db push

# 或者：创建迁移文件（正式项目推荐）
npx prisma migrate dev --name 描述你改了什么
```

忘记同步的话，代码里用了新字段但数据库里没有，就会报错。

### 4. 关系查询没有用 include

`findMany()` 默认不返回关联表的数据：

```javascript
// 这样查只会返回 User 本身的字段，不包含帖子
const user = await db.user.findUnique({ where: { id: "u001" } })
console.log(user.Post)  // undefined！

// 想一起查出帖子，需要用 include：
const user = await db.user.findUnique({
  where: { id: "u001" },
  include: { Post: true },  // 把关联的帖子也查出来
})
console.log(user.Post)  // [{ id: "p001", title: "..." }, ...]
```

### 5. 搞混 select 和 where

- `where`：过滤条件（查哪些记录）—— 相当于"在哪个抽屉找"
- `select`：返回字段（返回哪些列）—— 相当于"从抽屉里拿哪些东西"

```javascript
// where 决定"查谁的帖子"，select 决定"返回帖子的哪些信息"
await db.post.findMany({
  where: { authorId: "u001" },        // 查 u001 的帖子
  select: { id: true, title: true },  // 只返回 id 和 title
})
```

---

## 延伸阅读

以下资料适合在完成今天的学习后，有余力时继续探索：

- **Prisma 官方文档 — 快速入门**：[prisma.io/docs/getting-started](https://www.prisma.io/docs/getting-started) — 从零开始建数据库，适合动手练习
- **Prisma Schema 参考**：[prisma.io/docs/reference/api-reference/prisma-schema-reference](https://www.prisma.io/docs/reference/api-reference/prisma-schema-reference) — 所有 Schema 语法的详细说明
- **Prisma Client API**：[prisma.io/docs/reference/api-reference/prisma-client-reference](https://www.prisma.io/docs/reference/api-reference/prisma-client-reference) — `findMany`、`create` 等方法的完整参数
- **MDN — async/await**：[developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Asynchronous/Promises](https://developer.mozilla.org/zh-CN/docs/Learn/JavaScript/Asynchronous/Promises) — 想深入理解异步编程的同学必读
- **Prisma Studio**：运行 `npx prisma studio` 可以在浏览器里可视化查看和编辑数据库，非常适合学习和调试
