# Day 09 — 支付系统（Stripe 订阅收款）

> 对应章节：[Chapter 9: 支付系统](../chapter-09-payment-system.md)

---

## 今日学习目标

1. 理解 Stripe 支付流程：从用户点击"升级"到数据库更新的完整链路
2. 掌握 Webhook 的概念——为什么需要它、怎么验证它是安全的
3. 能逐行读懂项目中 Stripe 相关的所有源码文件

---

## 前置知识速补

今天的内容会涉及一些新的编程概念。如果你之前没接触过，先花 30 分钟把下面几个知识点弄清楚，后面读源码就会顺畅很多。

### 1. 事件驱动编程——[Webhook](https://developer.mozilla.org/zh-CN/docs/Glossary/Webhook) 就像"订阅通知"

你在网上买东西，商家说"发货了会给你发短信通知"。你不用一直刷物流页面，短信来了你就知道了。

Webhook 就是这个道理：

- **你（服务器）** 提前告诉 **Stripe**："支付成功后，请往这个地址发个通知。"
- **Stripe** 在支付完成后，主动往你的服务器发一个 HTTP POST 请求。
- 你的服务器收到通知后，更新数据库。

这就是**事件驱动**——你不用反复去问"支付好了没"，而是等通知来了再行动。

```
传统做法（轮询）：
你 → 每隔 5 秒问 Stripe："支付好了吗？" ← 浪费资源

事件驱动（Webhook）：
Stripe → 支付好了，主动通知你 ← 高效
```

### 2. 签名验证——确保消息来自可信来源

想象一下：如果任何人都能往你的 Webhook 地址发一个"支付成功"的假消息，那岂不是可以白嫖 Pro 功能？

所以 Stripe 在发通知时会带一个**签名**（signature），就像一封信上的蜡封印章。你的服务器用一个只有你和 Stripe 知道的**密钥**（secret）来验证这个签名——如果签名对得上，说明消息确实是 Stripe 发的；如果对不上，就直接拒绝。

```
Stripe 发送请求时：
  请求体 + 密钥 → 生成签名 → 放在请求头里

你的服务器收到时：
  请求体 + 同样的密钥 → 重新算签名 → 和请求头里的签名比对
  ├─ 一致 → 可信，继续处理
  └─ 不一致 → 不可信，返回 400 错误
```

### 3. `switch` 语句

[switch](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Statements/switch) 是一种根据变量值执行不同逻辑的写法，适合变量有多种可能值的情况。虽然本项目的 Webhook 用的是 `if` 判断，但 Webhook 处理中 `switch` 非常常见，值得了解：

```javascript
// 用 if 写（项目中的写法）
if (event.type === "checkout.session.completed") {
  // 处理首次订阅
}
if (event.type === "invoice.payment_succeeded") {
  // 处理续费成功
}

// 用 switch 写（等价写法，事件类型多时更清晰）
switch (event.type) {
  case "checkout.session.completed":
    // 处理首次订阅
    break    // break 很重要！不写的话会"穿透"执行下一个 case
  case "invoice.payment_succeeded":
    // 处理续费成功
    break
  default:
    // 其他事件类型，不处理
    break
}
```

### 4. `Buffer` 和二进制数据基础

在 Webhook 签名验证中，数据是以"原始文本"（raw body）的形式参与计算的，而不是解析后的 JSON 对象。这就是为什么代码用 `req.text()` 而不是 `req.json()` 来读取请求体：

```javascript
// req.text() 返回原始字符串，保持数据的"原样"
const body = await req.text()

// 如果用 req.json()，数据会被解析成对象，再转回字符串时格式可能变化
// 格式一变，签名验证就会失败！
```

[Buffer](https://developer.mozilla.org/zh-CN/docs/Glossary/Buffer) 是 Node.js 中处理二进制数据的工具。签名验证底层用到了它，但 Stripe SDK 帮我们封装好了，我们只需要传入原始字符串即可。

### 5. `headers()` 函数

[headers()](https://nextjs.org/docs/app/api-reference/functions/headers) 是 Next.js 提供的函数，用来在服务端获取 HTTP 请求头：

```javascript
import { headers } from "next/headers"

// 获取 Stripe 放在请求头里的签名
const signature = headers().get("Stripe-Signature")
// 返回类似 "t=1234567890,v1=abc123..." 的字符串
```

请求头（headers）就像信封上的附加信息——收件人地址、邮戳、挂号编号等。请求体（body）才是信的内容。Stripe 把签名放在请求头里，把事件数据放在请求体里。

---

## 核心概念清单

| 概念 | 英文 | 一句话解释 |
|------|------|-----------|
| [Stripe](https://stripe.com/docs) | Stripe | 全球最流行的在线支付平台，帮你处理信用卡收款 |
| Webhook | Webhook | Stripe 主动通知你的服务器"有事发生了"的机制 |
| 订阅 | [Subscription](https://stripe.com/docs/billing/subscriptions/overview) | 用户按月/年定期付费的商业模式 |
| [Checkout Session](https://stripe.com/docs/payments/checkout) | Checkout Session | Stripe 托管的结账页面，用户在上面输入信用卡信息 |
| Customer Portal | [Billing Portal](https://stripe.com/docs/billing/subscriptions/integrating-customer-portal) | Stripe 托管的订阅管理页面，用户可以取消或更改订阅 |
| 签名验证 | Signature Verification | 用密钥确认 Webhook 请求确实来自 Stripe |
| 宽限期 | Grace Period | 订阅到期后额外给的 24 小时缓冲时间 |
| Price ID | Price ID | Stripe 上某个价格方案的唯一标识，如 `price_1234abc` |
| [402 状态码](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/402) | 402 Payment Required | 表示"你需要付费才能使用这个功能" |

---

## 源码精读

下面我们逐个文件来读。每个文件都不长，但每一行都有它的用意。

### 文件 1：Stripe 客户端初始化 — `lib/stripe.ts`

```typescript
import Stripe from "stripe"          // 引入 Stripe 官方的 Node.js SDK

import { env } from "@/env.mjs"      // 引入环境变量（API 密钥存在这里）

export const stripe = new Stripe(env.STRIPE_API_KEY, {
  apiVersion: "2022-11-15",           // 指定使用的 Stripe API 版本
  typescript: true,                   // 启用 TypeScript 类型支持
})
```

**逐行解读：**

- `import Stripe from "stripe"` — 从 npm 包 `stripe` 中引入 Stripe 类。这个包是 Stripe 官方提供的，封装了所有与 Stripe 通信的方法。
- `new Stripe(env.STRIPE_API_KEY, {...})` — 创建一个 Stripe 实例。`STRIPE_API_KEY` 是你在 Stripe 控制台获取的密钥，类似一把钥匙，证明"我是这个商户"。
- `apiVersion: "2022-11-15"` — 锁定 API 版本。Stripe 的 API 会不断更新，锁定版本可以避免升级时代码突然出问题。
- `export const stripe` — 导出这个实例，让其他文件都能用它与 Stripe 通信。

整个文件就做了一件事：**创建并导出一个 Stripe 客户端实例**。

### 文件 2：订阅计划配置 — `config/subscriptions.ts`

```typescript
import { SubscriptionPlan } from "types"
import { env } from "@/env.mjs"

export const freePlan: SubscriptionPlan = {
  name: "Free",
  description:
    "The free plan is limited to 3 posts. Upgrade to the PRO plan for unlimited posts.",
  stripePriceId: "",                              // 免费计划没有价格 ID，空字符串
}

export const proPlan: SubscriptionPlan = {
  name: "PRO",
  description: "The PRO plan has unlimited posts.",
  stripePriceId: env.STRIPE_PRO_MONTHLY_PLAN_ID || "",  // 从环境变量读取
}
```

**逐行解读：**

- `freePlan` 和 `proPlan` 是两个"计划对象"，定义了计划名、描述和对应的 Stripe Price ID。
- `stripePriceId: ""` — 免费计划不需要在 Stripe 上收费，所以 Price ID 留空。
- `env.STRIPE_PRO_MONTHLY_PLAN_ID || ""` — 从环境变量中读取 PRO 计划的价格 ID。`|| ""` 的意思是：如果环境变量没设置，就用空字符串代替（避免报错）。
- 这个 Price ID 是你在 Stripe 控制台创建产品和价格后得到的，格式类似 `price_1N4xYz...`。

### 文件 3：获取用户订阅状态 — `lib/subscription.ts`

```typescript
import { UserSubscriptionPlan } from "types"
import { freePlan, proPlan } from "@/config/subscriptions"
import { db } from "@/lib/db"

export async function getUserSubscriptionPlan(
  userId: string
): Promise<UserSubscriptionPlan> {
  // 从数据库查询用户的 Stripe 相关字段
  const user = await db.user.findFirst({
    where: {
      id: userId,                         // 找到这个用户
    },
    select: {
      stripeSubscriptionId: true,         // Stripe 订阅 ID
      stripeCurrentPeriodEnd: true,       // 当前计费周期的结束时间
      stripeCustomerId: true,             // Stripe 客户 ID
      stripePriceId: true,               // 用户订阅的价格 ID
    },
  })

  if (!user) {
    throw new Error("User not found")     // 用户不存在就报错
  }

  // 判断是否是 Pro 用户——两个条件必须同时满足：
  const isPro =
    user.stripePriceId &&                 // 条件1：有 stripePriceId（说明订阅过）
    user.stripeCurrentPeriodEnd?.getTime() + 86_400_000 > Date.now()
  //                                        ↑ 86400000 毫秒 = 24 小时（宽限期）
  //  条件2：订阅周期结束时间 + 24小时 > 当前时间（说明还没过期）

  const plan = isPro ? proPlan : freePlan // 根据判断结果选择计划

  return {
    ...plan,                              // 展开计划信息（name, description 等）
    ...user,                              // 展开用户的 Stripe 字段
    stripeCurrentPeriodEnd: user.stripeCurrentPeriodEnd?.getTime(),
    isPro,                                // 附上 isPro 判断结果
  }
}
```

**重点理解——为什么要加 24 小时宽限期？**

Stripe 续费不是精确到秒的。比如用户的订阅在 3 月 1 日到期，Stripe 可能在 3 月 1 日当天尝试扣款，但信用卡处理可能需要几个小时。如果你在 3 月 1 日 00:01 就判定用户"已过期"，用户体验很差。加 24 小时宽限期可以避免这种误判。

`86_400_000` 是 JS 中的数字分隔符写法，等价于 `86400000`，就是 24 x 60 x 60 x 1000 毫秒。

### 文件 4：创建 Stripe Session 的 API — `app/api/users/stripe/route.ts`

```typescript
import { getServerSession } from "next-auth/next"
import { z } from "zod"

import { proPlan } from "@/config/subscriptions"
import { authOptions } from "@/lib/auth"
import { stripe } from "@/lib/stripe"
import { getUserSubscriptionPlan } from "@/lib/subscription"
import { absoluteUrl } from "@/lib/utils"

const billingUrl = absoluteUrl("/dashboard/billing")
// absoluteUrl 把相对路径变成完整 URL，如 "http://localhost:3000/dashboard/billing"

export async function GET(req: Request) {
  try {
    // 第 1 步：检查用户是否已登录
    const session = await getServerSession(authOptions)

    if (!session?.user || !session?.user.email) {
      return new Response(null, { status: 403 })   // 未登录，返回 403 禁止
    }

    // 第 2 步：获取用户当前的订阅状态
    const subscriptionPlan = await getUserSubscriptionPlan(session.user.id)

    // ========== 情况 A：已经是 Pro 用户 ==========
    // 打开"订阅管理"页面（Billing Portal），让用户管理、取消订阅
    if (subscriptionPlan.isPro && subscriptionPlan.stripeCustomerId) {
      const stripeSession = await stripe.billingPortal.sessions.create({
        customer: subscriptionPlan.stripeCustomerId,  // 告诉 Stripe 是哪个客户
        return_url: billingUrl,                       // 管理完后跳回这个页面
      })

      return new Response(JSON.stringify({ url: stripeSession.url }))
    }

    // ========== 情况 B：免费用户 ==========
    // 打开"结账"页面（Checkout Session），让用户输入信用卡升级
    const stripeSession = await stripe.checkout.sessions.create({
      success_url: billingUrl,              // 支付成功后跳转的页面
      cancel_url: billingUrl,               // 用户取消支付后跳转的页面
      payment_method_types: ["card"],        // 接受信用卡支付
      mode: "subscription",                  // 订阅模式（按月自动扣款）
      billing_address_collection: "auto",    // 自动收集账单地址
      customer_email: session.user.email,    // 预填用户的邮箱
      line_items: [
        {
          price: proPlan.stripePriceId,      // 要订阅的价格方案
          quantity: 1,                       // 数量 1
        },
      ],
      metadata: {
        userId: session.user.id,             // 自定义数据：带上用户 ID
        // 这个 userId 在后面 Webhook 里会用到！
      },
    })

    return new Response(JSON.stringify({ url: stripeSession.url }))
  } catch (error) {
    if (error instanceof z.ZodError) {
      return new Response(JSON.stringify(error.issues), { status: 422 })
    }

    return new Response(null, { status: 500 })
  }
}
```

**重点理解——`metadata` 是什么？**

`metadata` 是 Stripe 允许你附加的自定义数据。Stripe 不关心里面放什么，但会在 Webhook 回调时原样带回来。这里放了 `userId`，这样 Webhook 处理时就知道是哪个用户完成了支付。

### 文件 5：Webhook 处理 — `app/api/webhooks/stripe/route.ts`

这是今天最核心的文件，请仔细阅读每一行注释。

```typescript
import { headers } from "next/headers"
import Stripe from "stripe"

import { env } from "@/env.mjs"
import { db } from "@/lib/db"
import { stripe } from "@/lib/stripe"

export async function POST(req: Request) {
  // ====== 第 1 步：读取请求数据 ======
  const body = await req.text()
  // 用 req.text() 而不是 req.json()！
  // 因为签名验证需要"原始文本"，JSON 解析后再转回来格式可能变

  const signature = headers().get("Stripe-Signature") as string
  // 从请求头中取出 Stripe 的签名

  // ====== 第 2 步：验证签名 ======
  let event: Stripe.Event
  // Stripe.Event 是 Stripe 定义的事件类型

  try {
    event = stripe.webhooks.constructEvent(
      body,                        // 原始请求体
      signature,                   // 请求头中的签名
      env.STRIPE_WEBHOOK_SECRET    // 你的 Webhook 密钥（只有你和 Stripe 知道）
    )
  } catch (error) {
    // 签名验证失败 → 这个请求不是 Stripe 发的，可能是伪造的
    return new Response(`Webhook Error: ${error.message}`, { status: 400 })
  }

  // ====== 第 3 步：取出事件中的 session 对象 ======
  const session = event.data.object as Stripe.Checkout.Session
  // event.data.object 是事件的具体数据，这里断言为 Checkout.Session 类型

  // ====== 第 4 步：根据事件类型执行不同逻辑 ======

  // 事件类型 A：首次订阅完成（用户刚刚在 Checkout 页面付完款）
  if (event.type === "checkout.session.completed") {
    // 从 Stripe 获取这笔订阅的详细信息
    const subscription = await stripe.subscriptions.retrieve(
      session.subscription as string
    )

    // 把 Stripe 的信息写入我们自己的数据库
    await db.user.update({
      where: {
        id: session?.metadata?.userId,
        // ↑ 还记得创建 Checkout Session 时传入的 metadata.userId 吗？这里用上了！
      },
      data: {
        stripeSubscriptionId: subscription.id,              // 订阅 ID
        stripeCustomerId: subscription.customer as string,  // 客户 ID
        stripePriceId: subscription.items.data[0].price.id, // 价格 ID
        stripeCurrentPeriodEnd: new Date(
          subscription.current_period_end * 1000
          // ↑ Stripe 返回的是 Unix 时间戳（秒），JS 的 Date 需要毫秒，所以 ×1000
        ),
      },
    })
  }

  // 事件类型 B：续费发票支付成功（订阅每月自动续费时触发）
  if (event.type === "invoice.payment_succeeded") {
    const subscription = await stripe.subscriptions.retrieve(
      session.subscription as string
    )

    // 更新订阅周期结束时间（续了一个月）
    await db.user.update({
      where: {
        stripeSubscriptionId: subscription.id,
        // ↑ 续费时不是用 metadata.userId，而是用 subscriptionId 找用户
        // 因为续费事件不一定有 metadata
      },
      data: {
        stripePriceId: subscription.items.data[0].price.id,
        stripeCurrentPeriodEnd: new Date(
          subscription.current_period_end * 1000
        ),
      },
    })
  }

  // 无论什么事件，都返回 200 告诉 Stripe "收到了"
  return new Response(null, { status: 200 })
}
```

**为什么两个事件的 `where` 条件不同？**

- `checkout.session.completed`（首次订阅）：用户还没有 `stripeSubscriptionId`，只能通过 `metadata.userId` 找到用户。
- `invoice.payment_succeeded`（续费）：用户已经有 `stripeSubscriptionId`（首次订阅时存入的），可以直接用它找到用户。

### 文件 6：前端账单界面 — `components/billing-form.tsx`

```tsx
"use client"

import * as React from "react"

import { UserSubscriptionPlan } from "types"
import { cn, formatDate } from "@/lib/utils"
import { buttonVariants } from "@/components/ui/button"
import {
  Card, CardContent, CardDescription,
  CardFooter, CardHeader, CardTitle,
} from "@/components/ui/card"
import { toast } from "@/components/ui/use-toast"
import { Icons } from "@/components/icons"

interface BillingFormProps extends React.HTMLAttributes<HTMLFormElement> {
  subscriptionPlan: UserSubscriptionPlan & {
    isCanceled: boolean
  }
}

export function BillingForm({
  subscriptionPlan,
  className,
  ...props
}: BillingFormProps) {
  const [isLoading, setIsLoading] = React.useState<boolean>(false)

  async function onSubmit(event) {
    event.preventDefault()           // 阻止表单默认提交行为
    setIsLoading(!isLoading)         // 显示加载状态

    // 调用我们的 API，获取 Stripe Session URL
    const response = await fetch("/api/users/stripe")

    if (!response?.ok) {
      return toast({
        title: "Something went wrong.",
        description: "Please refresh the page and try again.",
        variant: "destructive",      // 红色错误提示
      })
    }

    // 拿到 URL 后，跳转到 Stripe 页面
    const session = await response.json()
    if (session) {
      window.location.href = session.url   // 浏览器跳转
    }
  }

  return (
    <form className={cn(className)} onSubmit={onSubmit} {...props}>
      <Card>
        <CardHeader>
          <CardTitle>Subscription Plan</CardTitle>
          <CardDescription>
            You are currently on the <strong>{subscriptionPlan.name}</strong> plan.
          </CardDescription>
        </CardHeader>
        <CardContent>{subscriptionPlan.description}</CardContent>
        <CardFooter>
          <button type="submit" disabled={isLoading}>
            {isLoading && (
              <Icons.spinner className="mr-2 h-4 w-4 animate-spin" />
            )}
            {/* 根据是否是 Pro 用户显示不同按钮文字 */}
            {subscriptionPlan.isPro ? "Manage Subscription" : "Upgrade to PRO"}
          </button>
          {/* Pro 用户才显示续费/取消时间 */}
          {subscriptionPlan.isPro ? (
            <p className="rounded-full text-xs font-medium">
              {subscriptionPlan.isCanceled
                ? "Your plan will be canceled on "     // 已取消，显示到期时间
                : "Your plan renews on "}              // 正常订阅，显示续费时间
              {formatDate(subscriptionPlan.stripeCurrentPeriodEnd)}.
            </p>
          ) : null}
        </CardFooter>
      </Card>
    </form>
  )
}
```

**这个组件的"聪明"之处：** 同一个按钮、同一个 API 调用，但后端根据用户状态返回不同的 Stripe URL。免费用户被带到结账页，Pro 用户被带到管理页。前端完全不用关心这个区别。

---

## 动手练习

### 练习 1：追踪 metadata 的旅程（25 分钟）

**目标：** 理解 `userId` 如何从你的服务器传到 Stripe，又从 Stripe 回到你的服务器。

**步骤：**

1. 打开 `app/api/users/stripe/route.ts`，找到 `metadata: { userId: session.user.id }`。这是 `userId` 出发的起点。
2. 打开 `app/api/webhooks/stripe/route.ts`，找到 `session?.metadata?.userId`。这是 `userId` 回来的终点。
3. 用自己的话画出 `userId` 的完整旅程：

```
你的服务器 → metadata.userId → Stripe 保存
                                  ↓
                              用户完成支付
                                  ↓
你的服务器 ← metadata.userId ← Stripe Webhook 回传
```

4. 思考：如果创建 Checkout Session 时忘了传 `metadata`，Webhook 处理时会发生什么？

**排错提示：** 如果你在 `session?.metadata?.userId` 这里看到 `undefined`，99% 的情况是创建 Checkout Session 时 `metadata` 没传对。去检查创建 Session 的代码。

### 练习 2：对比两种 Stripe Session（20 分钟）

**目标：** 搞清楚 Checkout Session 和 Billing Portal Session 的区别。

**步骤：**

1. 在 `app/api/users/stripe/route.ts` 中找到两个 `stripe.xxx.sessions.create` 调用
2. 填写下面的对比表：

```
| 比较项             | Checkout Session            | Billing Portal Session       |
|--------------------|-----------------------------|------------------------------|
| 什么时候用？       | （你来填）                    | （你来填）                     |
| 创建方法           | stripe.checkout.sessions     | stripe.billingPortal.sessions |
| 需要什么参数？     | （你来填）                    | （你来填）                     |
| 用户看到什么页面？ | （你来填）                    | （你来填）                     |
```

3. 问自己：如果一个免费用户点了按钮，走的是哪条路径？Pro 用户呢？代码中用什么条件判断？

**排错提示：** 注意 `if (subscriptionPlan.isPro && subscriptionPlan.stripeCustomerId)` 这个判断——两个条件缺一不可。想想为什么要同时检查这两个。

### 练习 3：手动模拟 Webhook 签名验证失败（15 分钟）

**目标：** 通过"故意出错"来理解签名验证的重要性。

**步骤：**

1. 阅读 Webhook 处理文件中 `try...catch` 那段代码
2. 假设你用 curl 直接发一个 POST 请求到 Webhook 地址：

```bash
curl -X POST http://localhost:3000/api/webhooks/stripe \
  -H "Content-Type: application/json" \
  -d '{"type": "checkout.session.completed", "data": {"object": {}}}'
```

3. 预测会发生什么？（提示：请求头里没有 `Stripe-Signature`）
4. 为什么这样设计是安全的？如果没有签名验证，上面这个请求会造成什么后果？

---

## 概念图解

### 支付流程全景图

```
┌─────────────────────── 用户视角 ───────────────────────┐
│                                                         │
│  1. 用户写了第 4 篇文章 → 弹出提示"请升级到 PRO"       │
│                    ↓                                    │
│  2. 用户进入 /dashboard/billing 页面                    │
│                    ↓                                    │
│  3. 点击 "Upgrade to PRO" 按钮                          │
│                    ↓                                    │
│  4. 页面跳转到 Stripe 结账页面                          │
│                    ↓                                    │
│  5. 输入信用卡号 4242 4242 4242 4242（测试卡号）        │
│                    ↓                                    │
│  6. 跳转回 /dashboard/billing，显示 "PRO" 计划          │
│                    ↓                                    │
│  7. 开始愉快地创建无限文章                              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### Webhook 事件处理流程

```
                    Stripe 服务器
                         │
          ┌──────────────┼──────────────┐
          │              │              │
    checkout.session   invoice.payment  其他事件
      .completed        _succeeded      （忽略）
          │              │
          ↓              ↓
  ┌───────────────────────────────────┐
  │    POST /api/webhooks/stripe      │
  │                                   │
  │  1. req.text() 读取原始请求体     │
  │  2. headers() 获取签名            │
  │  3. constructEvent() 验证签名     │
  │     ├─ 失败 → 返回 400           │
  │     └─ 成功 → 继续处理           │
  │  4. 判断 event.type               │
  │     ├─ checkout.session.completed │
  │     │  → 用 metadata.userId       │
  │     │  → 首次写入 4 个字段        │
  │     └─ invoice.payment_succeeded  │
  │        → 用 stripeSubscriptionId  │
  │        → 更新 2 个字段            │
  │  5. 返回 200                      │
  └───────────────────────────────────┘
```

### 数据库中的 Stripe 字段

```
User 表
├── id                       ← 我们系统的用户 ID
├── name, email, ...         ← 基本信息
├── stripeCustomerId         ← Stripe 客户 ID（cus_xxx）
├── stripeSubscriptionId     ← Stripe 订阅 ID（sub_xxx）
├── stripePriceId            ← 订阅的价格 ID（price_xxx）
└── stripeCurrentPeriodEnd   ← 当前计费周期结束时间

判断用户是否是 Pro：
  stripePriceId 不为空 AND stripeCurrentPeriodEnd + 24h > 现在
```

---

## 自检问题

### 问题 1：为什么用户的信用卡信息不经过我们的服务器？

<details>
<summary>查看答案</summary>

因为用户是在 Stripe 托管的结账页面（Checkout Session）上输入信用卡号的，信用卡信息直接由 Stripe 处理。我们的服务器只负责创建 Session 和接收 Webhook 通知，从头到尾不接触任何支付敏感信息。

这样做的好处：(1) 不用担心信用卡信息泄露的安全风险；(2) 不需要通过严格的 PCI DSS 合规认证（这个认证成本很高）。
</details>

### 问题 2：`req.text()` 和 `req.json()` 有什么区别？Webhook 处理为什么必须用 `req.text()`？

<details>
<summary>查看答案</summary>

- `req.text()` 返回请求体的原始字符串。
- `req.json()` 会把请求体解析成 JavaScript 对象。

Webhook 必须用 `req.text()` 是因为签名验证需要原始字符串。如果用 `req.json()` 解析后再 `JSON.stringify()` 转回来，空格、键的顺序等可能发生变化，导致签名验证失败。
</details>

### 问题 3：如果 Webhook 处理过程中数据库暂时不可用（比如网络故障），会发生什么？

<details>
<summary>查看答案</summary>

`db.user.update()` 会抛出异常，导致 API 返回 500 错误（而不是 200）。Stripe 看到非 200 响应后，会自动重试发送 Webhook——通常会在几分钟后重试，最多重试若干次。这就是 Stripe 保证"最终一致性"的机制。

这也意味着你的 Webhook 处理逻辑最好是幂等的（多次处理同一事件，结果一样），因为同一事件可能被发送多次。
</details>

### 问题 4：`checkout.session.completed` 和 `invoice.payment_succeeded` 分别在什么时候触发？

<details>
<summary>查看答案</summary>

- `checkout.session.completed`：用户首次完成订阅支付时触发（只触发一次）。
- `invoice.payment_succeeded`：每次续费成功时触发（每月自动扣款成功后）。

所以首次订阅走的是第一个分支（需要写入 4 个字段），之后每月续费走的是第二个分支（只需要更新 2 个字段）。
</details>

### 问题 5：如果要新增一个 "Team" 计划（每月 $49，支持 10 个用户），需要修改哪些文件？

<details>
<summary>查看答案</summary>

至少需要修改以下文件：

1. **`config/subscriptions.ts`** — 新增 `teamPlan` 对象，设置 name、description 和 stripePriceId。
2. **`.env`** — 新增 `STRIPE_TEAM_MONTHLY_PLAN_ID` 环境变量。
3. **`env.mjs`** — 在环境变量校验中新增这个变量。
4. **`lib/subscription.ts`** — 修改 `isPro` 判断逻辑，增加对 Team 计划的识别。
5. **`app/api/users/stripe/route.ts`** — 增加 Team 计划对应的 Checkout Session 创建逻辑。
6. **`components/billing-form.tsx`** — 修改 UI 以展示多个计划选项。
7. **Stripe 控制台** — 创建新的 Product 和 Price。
</details>

---

## 预计时间分配

| 环节 | 时间 | 说明 |
|------|------|------|
| 前置知识速补 | 30 分钟 | 事件驱动、签名验证、switch、Buffer、headers |
| 教程阅读 | 50 分钟 | 完整阅读 chapter-09-payment-system.md |
| 源码精读 | 60 分钟 | 逐行阅读上述 6 个文件，对照本文注释 |
| 练习 1 | 25 分钟 | 追踪 metadata 的旅程 |
| 练习 2 | 20 分钟 | 对比两种 Stripe Session |
| 练习 3 | 15 分钟 | 模拟签名验证失败 |
| 自检与笔记 | 20 分钟 | 回答自检问题，整理今日笔记 |
| **合计** | **约 3.5 小时** | |

---

## 常见踩坑点

### 1. 测试模式 vs 生产模式
开发时务必使用 Stripe 测试模式的 API Key（以 `sk_test_` 开头）。测试信用卡号是 `4242 4242 4242 4242`，有效期填未来任意日期，CVC 填任意三位数。千万不要用真实信用卡测试。

### 2. Webhook Secret 不匹配
本地开发时，Stripe 的 Webhook 无法直接发到 `localhost`。你需要用 Stripe CLI 做转发：

```bash
stripe listen --forward-to localhost:3000/api/webhooks/stripe
```

CLI 会输出一个 Webhook Secret（以 `whsec_` 开头），把它填到 `.env` 的 `STRIPE_WEBHOOK_SECRET` 中。注意：这个和 Stripe Dashboard 上的 Webhook Secret 是不同的！

### 3. Webhook 幂等性
同一个 Webhook 事件可能被 Stripe 重复发送（比如你的服务器第一次返回了 500）。所以处理逻辑必须是"幂等"的——执行一次和执行多次效果一样。本项目用 `db.user.update()` 天然满足这个要求（多次更新同样的值，结果不变）。

### 4. Unix 时间戳单位
Stripe 返回的时间戳单位是**秒**，JavaScript 的 `Date` 构造函数和 `Date.now()` 用的是**毫秒**。转换时记得乘以 1000：

```javascript
// Stripe 返回: 1700000000（秒）
// JS 需要:    1700000000000（毫秒）
new Date(subscription.current_period_end * 1000)
```

忘了乘 1000 的话，日期会变成 1970 年。

### 5. 数据库缺少 Stripe 字段
如果你从零开始搭建项目，记得在 Prisma schema 的 User 模型中添加这四个字段：`stripeCustomerId`、`stripeSubscriptionId`、`stripePriceId`、`stripeCurrentPeriodEnd`。否则 Webhook 处理时 `db.user.update()` 会报错。

### 6. 时区问题
`stripeCurrentPeriodEnd` 存储的是 UTC 时间。在比较时不要用本地时间（比如北京时间 UTC+8），用 `Date.now()` 获取的是 UTC 毫秒数，可以直接比较。

---

## 延伸阅读

- [Stripe 官方文档：Checkout 快速入门](https://stripe.com/docs/checkout/quickstart) — 从零创建一个 Checkout Session
- [Stripe 官方文档：Webhook 入门](https://stripe.com/docs/webhooks) — 理解 Webhook 的完整生命周期
- [Stripe 官方文档：测试模式](https://stripe.com/docs/testing) — 所有测试卡号和测试场景
- [Stripe CLI 文档](https://stripe.com/docs/stripe-cli) — 本地开发调试 Webhook 的必备工具
- [MDN: HTTP 状态码 402](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Status/402) — 了解 402 Payment Required 的含义
- [Node.js Buffer 文档](https://nodejs.org/api/buffer.html) — 如果你对二进制数据处理感兴趣
