# 第9章：付费系统 — Stripe 怎么收钱的？

## 学习目标

- 理解 SaaS 订阅模式的基本概念
- 理解 Stripe Checkout Session 的工作流程
- 理解 Webhook 回调的概念和安全验证
- 理解订阅状态在数据库中的存储
- 理解前端如何展示不同订阅计划

## 涉及文件

| 文件 | 说明 |
|------|------|
| `lib/stripe.ts` | Stripe 客户端初始化 |
| `lib/subscription.ts` | 获取用户订阅计划 |
| `config/subscriptions.ts` | 订阅计划定义 |
| `app/api/users/stripe/route.ts` | 创建 Stripe Session |
| `app/api/webhooks/stripe/route.ts` | Stripe Webhook 处理 |
| `components/billing-form.tsx` | 账单管理界面 |
| `lib/exceptions.ts` | RequiresProPlanError |

---

## 1. SaaS 订阅模式

**SaaS（Software as a Service）** 是一种软件商业模式：用户按月/年付费使用软件。

Taxonomy 项目实现了一个典型的 SaaS 订阅：

| 计划 | 价格 | 限制 |
|------|------|------|
| Free（免费） | $0/月 | 最多 3 篇帖子 |
| PRO（专业） | 付费/月 | 无限帖子 |

---

## 2. 订阅计划配置

```typescript
// config/subscriptions.ts
import { SubscriptionPlan } from "types"
import { env } from "@/env.mjs"

export const freePlan: SubscriptionPlan = {
  name: "Free",
  description: "The free plan is limited to 3 posts. Upgrade to the PRO plan for unlimited posts.",
  stripePriceId: "",       // 免费计划没有 Stripe Price ID
}

export const proPlan: SubscriptionPlan = {
  name: "PRO",
  description: "The PRO plan has unlimited posts.",
  stripePriceId: env.STRIPE_PRO_MONTHLY_PLAN_ID || "",  // 从环境变量读取
}
```

**`stripePriceId`** 是 Stripe 上创建的价格 ID（如 `price_1234abc`）。你在 Stripe Dashboard 中设置好价格后，把 ID 放到环境变量中。

---

## 3. Stripe 客户端

```typescript
// lib/stripe.ts
import Stripe from "stripe"
import { env } from "@/env.mjs"

export const stripe = new Stripe(env.STRIPE_API_KEY, {
  apiVersion: "2022-11-15",
  typescript: true,
})
```

一个简单的初始化——用 API 密钥创建 Stripe 客户端实例。后续所有与 Stripe 的通信都通过这个实例。

---

## 4. 获取用户订阅状态

```typescript
// lib/subscription.ts
import { freePlan, proPlan } from "@/config/subscriptions"
import { db } from "@/lib/db"

export async function getUserSubscriptionPlan(userId: string) {
  // 从数据库获取用户的 Stripe 信息
  const user = await db.user.findFirst({
    where: { id: userId },
    select: {
      stripeSubscriptionId: true,     // Stripe 订阅 ID
      stripeCurrentPeriodEnd: true,    // 当前计费周期结束时间
      stripeCustomerId: true,          // Stripe 客户 ID
      stripePriceId: true,             // 订阅的价格 ID
    },
  })

  if (!user) throw new Error("User not found")

  // 判断是否是 Pro 用户
  // 条件：有 stripePriceId 且订阅未过期（加 24 小时宽限期）
  const isPro =
    user.stripePriceId &&
    user.stripeCurrentPeriodEnd?.getTime() + 86_400_000 > Date.now()
  //                                         ↑ 86400000 毫秒 = 24 小时

  const plan = isPro ? proPlan : freePlan

  return {
    ...plan,
    ...user,
    stripeCurrentPeriodEnd: user.stripeCurrentPeriodEnd?.getTime(),
    isPro,
  }
}
```

**24 小时宽限期（`+ 86_400_000`）：** Stripe 续费有时会延迟几秒到几小时。加 24 小时宽限期可以避免在续费期间错误地将用户降级。

---

## 5. 付费流程 — Stripe Checkout Session

当用户点击"Upgrade to PRO"按钮时，会触发以下流程：

```
用户点击 "Upgrade to PRO"
    ↓
前端调用 GET /api/users/stripe
    ↓
后端创建 Stripe Checkout Session
    ↓
返回 Stripe 结账页面的 URL
    ↓
前端跳转到 Stripe 结账页面
    ↓
用户在 Stripe 页面输入信用卡信息
    ↓
Stripe 处理支付
    ↓
支付成功 → 用户被重定向回 /dashboard/billing
    ↓
同时，Stripe 发送 Webhook 到 /api/webhooks/stripe
    ↓
后端收到 Webhook，更新数据库中的订阅状态
```

### API 端代码

```typescript
// app/api/users/stripe/route.ts
import { stripe } from "@/lib/stripe"
import { absoluteUrl } from "@/lib/utils"

const billingUrl = absoluteUrl("/dashboard/billing")

export async function GET(req: Request) {
  try {
    const session = await getServerSession(authOptions)

    if (!session?.user || !session?.user.email) {
      return new Response(null, { status: 403 })
    }

    const subscriptionPlan = await getUserSubscriptionPlan(session.user.id)

    // ========== 情况1：已是 Pro 用户 ==========
    // 创建 Billing Portal Session（管理现有订阅）
    if (subscriptionPlan.isPro && subscriptionPlan.stripeCustomerId) {
      const stripeSession = await stripe.billingPortal.sessions.create({
        customer: subscriptionPlan.stripeCustomerId,
        return_url: billingUrl,  // 管理完后返回的页面
      })

      return new Response(JSON.stringify({ url: stripeSession.url }))
    }

    // ========== 情况2：免费用户 ==========
    // 创建 Checkout Session（新订阅结账页）
    const stripeSession = await stripe.checkout.sessions.create({
      success_url: billingUrl,              // 支付成功后跳转
      cancel_url: billingUrl,               // 取消支付后跳转
      payment_method_types: ["card"],        // 支付方式：信用卡
      mode: "subscription",                  // 模式：订阅（而非一次性付款）
      billing_address_collection: "auto",    // 自动收集账单地址
      customer_email: session.user.email,    // 预填用户邮箱
      line_items: [
        {
          price: proPlan.stripePriceId,      // 订阅的价格 ID
          quantity: 1,
        },
      ],
      metadata: {
        userId: session.user.id,             // 自定义数据：用户 ID（Webhook 会用到）
      },
    })

    return new Response(JSON.stringify({ url: stripeSession.url }))
  } catch (error) {
    return new Response(null, { status: 500 })
  }
}
```

**两种情况：**
1. **免费用户** → 创建 Checkout Session（结账页面，让用户输入信用卡）
2. **Pro 用户** → 创建 Billing Portal Session（管理页面，可以取消/更改订阅）

---

## 6. Webhook — Stripe 的"回调通知"

**Webhook** 是一种"反向 API"——不是你去调 Stripe，而是 Stripe 主动通知你。

**为什么需要 Webhook？**

用户在 Stripe 结账页面完成支付后，你的后端需要知道"支付成功了"以便更新数据库。但用户可能在支付成功后关闭浏览器不回来，所以不能依赖前端告诉后端。Stripe 通过 Webhook 直接通知你的服务器。

```typescript
// app/api/webhooks/stripe/route.ts
import { headers } from "next/headers"
import Stripe from "stripe"

import { env } from "@/env.mjs"
import { db } from "@/lib/db"
import { stripe } from "@/lib/stripe"

export async function POST(req: Request) {
  // 第1步：读取请求体和签名
  const body = await req.text()
  const signature = headers().get("Stripe-Signature") as string

  // 第2步：验证签名（确保请求确实来自 Stripe，不是伪造的）
  let event: Stripe.Event
  try {
    event = stripe.webhooks.constructEvent(
      body,
      signature,
      env.STRIPE_WEBHOOK_SECRET  // 用于验证的密钥
    )
  } catch (error) {
    return new Response(`Webhook Error: ${error.message}`, { status: 400 })
  }

  // 第3步：处理不同的事件类型
  const session = event.data.object as Stripe.Checkout.Session

  // 事件类型1：首次订阅完成
  if (event.type === "checkout.session.completed") {
    // 从 Stripe 获取订阅详情
    const subscription = await stripe.subscriptions.retrieve(
      session.subscription as string
    )

    // 更新数据库：记录用户的 Stripe 信息
    await db.user.update({
      where: {
        id: session?.metadata?.userId,  // 之前在 metadata 中传入的用户 ID
      },
      data: {
        stripeSubscriptionId: subscription.id,
        stripeCustomerId: subscription.customer as string,
        stripePriceId: subscription.items.data[0].price.id,
        stripeCurrentPeriodEnd: new Date(
          subscription.current_period_end * 1000  // Unix 时间戳转 Date
        ),
      },
    })
  }

  // 事件类型2：续费成功
  if (event.type === "invoice.payment_succeeded") {
    const subscription = await stripe.subscriptions.retrieve(
      session.subscription as string
    )

    // 更新订阅周期结束时间
    await db.user.update({
      where: {
        stripeSubscriptionId: subscription.id,
      },
      data: {
        stripePriceId: subscription.items.data[0].price.id,
        stripeCurrentPeriodEnd: new Date(
          subscription.current_period_end * 1000
        ),
      },
    })
  }

  return new Response(null, { status: 200 })
}
```

### Webhook 安全验证

```typescript
event = stripe.webhooks.constructEvent(body, signature, STRIPE_WEBHOOK_SECRET)
```

这行代码做了**签名验证**。Stripe 发送 Webhook 时会用一个密钥生成签名。你的服务器用同样的密钥验证签名，确保请求确实来自 Stripe。

如果有人伪造一个 POST 请求到 `/api/webhooks/stripe`，签名验证会失败，你的代码不会执行任何数据库操作。

---

## 7. 前端账单管理界面

```tsx
// components/billing-form.tsx
"use client"

export function BillingForm({ subscriptionPlan, className, ...props }) {
  const [isLoading, setIsLoading] = React.useState(false)

  async function onSubmit(event) {
    event.preventDefault()
    setIsLoading(true)

    // 调用 API 获取 Stripe Session URL
    const response = await fetch("/api/users/stripe")

    if (!response?.ok) {
      return toast({ title: "Something went wrong.", variant: "destructive" })
    }

    // 跳转到 Stripe 页面
    const session = await response.json()
    if (session) {
      window.location.href = session.url
    }
  }

  return (
    <form onSubmit={onSubmit}>
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
            {isLoading && <Icons.spinner className="animate-spin" />}
            {/* 根据订阅状态显示不同的按钮文字 */}
            {subscriptionPlan.isPro ? "Manage Subscription" : "Upgrade to PRO"}
          </button>

          {/* Pro 用户显示续费/取消时间 */}
          {subscriptionPlan.isPro && (
            <p className="text-xs">
              {subscriptionPlan.isCanceled
                ? "Your plan will be canceled on "
                : "Your plan renews on "}
              {formatDate(subscriptionPlan.stripeCurrentPeriodEnd)}.
            </p>
          )}
        </CardFooter>
      </Card>
    </form>
  )
}
```

---

## 8. 免费计划限制的实现

还记得第8章中创建帖子的 API 吗？限制逻辑就在那里：

```typescript
// app/api/posts/route.ts POST 函数中

// 获取用户订阅计划
const subscriptionPlan = await getUserSubscriptionPlan(user.id)

// 免费用户限制
if (!subscriptionPlan?.isPro) {
  const count = await db.post.count({
    where: { authorId: user.id },
  })

  if (count >= 3) {
    throw new RequiresProPlanError()  // 返回 402
  }
}
```

前端处理 402 状态码：

```typescript
// components/post-create-button.tsx
if (response.status === 402) {
  return toast({
    title: "Limit of 3 posts reached.",
    description: "Please upgrade to the PRO plan.",
    variant: "destructive",
  })
}
```

---

## 9. 完整付费流程图

```
┌──────────────────────────────────────────────────┐
│                  用户视角                          │
├──────────────────────────────────────────────────┤
│                                                    │
│  1. 创建第 4 篇帖子 → 提示"请升级到 PRO"           │
│                     ↓                              │
│  2. 进入 /dashboard/billing                        │
│                     ↓                              │
│  3. 点击 "Upgrade to PRO"                          │
│                     ↓                              │
│  4. 跳转到 Stripe 结账页面                          │
│                     ↓                              │
│  5. 输入信用卡信息并支付                            │
│                     ↓                              │
│  6. 跳转回 /dashboard/billing，显示 "PRO" 计划     │
│                     ↓                              │
│  7. 现在可以创建无限帖子了！                        │
│                                                    │
└──────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────┐
│                  服务器视角                        │
├──────────────────────────────────────────────────┤
│                                                    │
│  1. POST /api/posts → 402（免费限制）              │
│                                                    │
│  2. GET /api/users/stripe                          │
│     → 创建 Stripe Checkout Session                 │
│     → 返回 session.url                             │
│                                                    │
│  3. Stripe 处理支付                                │
│                                                    │
│  4. POST /api/webhooks/stripe                      │
│     → checkout.session.completed 事件              │
│     → 更新数据库（stripeSubscriptionId 等）         │
│                                                    │
│  5. POST /api/posts → 200（Pro 用户，不限制）      │
│                                                    │
└──────────────────────────────────────────────────┘
```

---

## 小练习

### 练习 1：理解订阅状态
查看 `prisma/schema.prisma` 中 User 模型的 Stripe 相关字段。思考：
1. `stripeCustomerId` 和 `stripeSubscriptionId` 的区别是什么？
2. `stripeCurrentPeriodEnd` 是什么意思？
3. 为什么 `isPro` 判断中要加 24 小时？

### 练习 2：追踪 Webhook
阅读 `app/api/webhooks/stripe/route.ts`，回答：
1. `checkout.session.completed` 和 `invoice.payment_succeeded` 分别在什么时候触发？
2. 如果 Webhook 验证失败，会发生什么？
3. `session.metadata.userId` 是在哪里设置的？

### 练习 3：理解两种 Stripe Session
1. Checkout Session 和 Billing Portal Session 有什么区别？
2. 什么情况下创建 Checkout Session？
3. 什么情况下创建 Billing Portal Session？

### 练习 4：思考安全性
如果没有 Webhook 签名验证，攻击者可以做什么？

---

## 本章小结

- SaaS 订阅模式：免费用户有限制，付费用户无限制
- Stripe Checkout Session 提供预构建的结账页面
- Webhook 是 Stripe 主动通知你的"反向 API"
- 签名验证确保 Webhook 请求来自 Stripe
- 用户订阅状态存储在数据库中（User 表的 stripe* 字段）
- 前端根据订阅状态显示不同的 UI（升级按钮 vs 管理按钮）

下一章（最后一章）我们将学习部署和工程化。
