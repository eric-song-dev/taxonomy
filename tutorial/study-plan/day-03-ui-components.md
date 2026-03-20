# Day 03 — UI 组件层：按钮、卡片这些界面元素是怎么搭出来的？

> 对应章节：[Chapter 3: UI 组件](../chapter-03-ui-components.md)
> 预计学习时间：3.5 - 4 小时

---

## 今日学习目标

1. **理解 JSX 和 React 组件基础** — 能看懂组件函数、props 传参、条件渲染等常见写法
2. **掌握 `cn()` 和 `cva` 的工作原理** — 能解释为什么项目中每个组件都用 `cn()`，能用 `cva` 定义组件变体
3. **能照着 Button 组件的模式，自己写一个简单的 UI 组件**

---

## 一、前置知识速补

这一节补充 Day 03 必须的前置知识。如果你之前完全没接触过 React，请务必花时间读完。

### 1.1 JSX 语法详解

[JSX](https://react.dev/learn/writing-markup-with-jsx) 是 React 发明的一种语法，让你可以在 JavaScript 里直接写"类似 HTML"的代码。

```tsx
// 这就是 JSX —— 看着像 HTML，但其实是 JavaScript
const element = <h1>Hello World</h1>
```

**JSX 和 HTML 的关键区别：**

| HTML 写法 | JSX 写法 | 为什么不一样 |
|-----------|---------|-------------|
| `class="btn"` | `className="btn"` | `class` 是 JavaScript 保留关键字，所以 JSX 用 `className` |
| `for="email"` | `htmlFor="email"` | `for` 也是 JavaScript 保留关键字 |
| `style="color: red"` | `style={{ color: "red" }}` | JSX 中 style 必须是对象，不是字符串 |
| `<img src="a.png">` | `<img src="a.png" />` | JSX 要求所有标签必须闭合，用 `/>` |
| `onclick="fn()"` | `onClick={fn}` | 事件名用驼峰命名法（camelCase） |

```tsx
// HTML 版本
// <div class="card" onclick="handleClick()">
//   <img src="photo.png">
// </div>

// JSX 版本
<div className="card" onClick={handleClick}>
  <img src="photo.png" />
</div>
```

**JSX 中插入 JavaScript 表达式 —— 用花括号 `{}`：**

```tsx
const name = "Alice"
const age = 25

return (
  <div>
    <h2>{name}</h2>                          {/* 插入变量 */}
    <p>{age >= 18 ? "成人" : "未成年"}</p>    {/* 三元表达式 */}
    <p>{`${name} is ${age} years old`}</p>    {/* 模板字符串 */}
    <p>{1 + 2}</p>                            {/* 任何 JS 表达式 */}
  </div>
)
```

> 小提醒：`{/* ... */}` 是 JSX 中的注释写法，和 HTML 的 `<!-- -->` 不同。

### 1.2 React 组件函数

React 组件就是一个**返回 JSX 的函数**，函数名必须大写字母开头：

```tsx
// 最简单的组件 —— 一个函数，返回 JSX
function Hello() {
  return <h1>Hello World</h1>
}

// 使用时就像一个自定义的 HTML 标签
<Hello />
```

你也可以用箭头函数写组件（在项目源码中经常见到）：

```tsx
// 箭头函数写法，效果完全一样
const Hello = () => {
  return <h1>Hello World</h1>
}
```

### 1.3 [Props](https://react.dev/learn/passing-props-to-a-component) 是什么、怎么传递

Props（properties 的缩写）就是**组件的参数**。就像函数有参数一样，组件通过 props 接收外部传入的数据。

```tsx
// 定义组件：接收一个 name 参数
//            ↓ 这里用 { name } 从 props 对象中取出 name
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}!</h1>
}

// 使用组件时传入参数（就像 HTML 属性一样）
<Greeting name="Alice" />   // 页面显示：Hello, Alice!
<Greeting name="Bob" />     // 页面显示：Hello, Bob!
```

**类比理解：**
- 组件 = 一个蛋糕模具
- Props = 你往模具里倒的原料（巧克力味、草莓味……）
- 渲染结果 = 不同口味的蛋糕

**TypeScript 中的 props 类型：**

```tsx
// 方式 1：内联类型（简单组件常用）
function Greeting({ name }: { name: string }) { ... }

// 方式 2：用 interface 定义（复杂组件常用）
interface GreetingProps {
  name: string
  age?: number    // ? 表示可选参数，传不传都行
}
function Greeting({ name, age }: GreetingProps) { ... }
```

### 1.4 [条件渲染](https://react.dev/learn/conditional-rendering)

在 JSX 中，你经常需要"如果 A 成立就显示 X，否则显示 Y"。有两种常见写法：

```tsx
function UserStatus({ isLoggedIn }: { isLoggedIn: boolean }) {
  return (
    <div>
      {/* 方式 1：三元运算符 —— 条件 ? 真 : 假 */}
      {isLoggedIn ? <p>欢迎回来！</p> : <p>请先登录</p>}

      {/* 方式 2：&& 短路运算 —— 条件为 true 时才渲染 */}
      {isLoggedIn && <button>退出登录</button>}
      {/*
        解释：如果 isLoggedIn 是 true，就渲染 <button>
              如果 isLoggedIn 是 false，什么都不渲染
      */}
    </div>
  )
}
```

### 1.5 [列表渲染](https://react.dev/learn/rendering-lists)（array.map）

当你有一组数据要渲染成列表时，用 [array.map()](https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Array/map)：

```tsx
function FruitList() {
  const fruits = ["苹果", "香蕉", "橙子"]

  return (
    <ul>
      {fruits.map((fruit) => (
        // key 是 React 要求的，用来追踪列表项
        <li key={fruit}>{fruit}</li>
      ))}
    </ul>
  )
  // 渲染结果：
  // - 苹果
  // - 香蕉
  // - 橙子
}
```

### 1.6 [TypeScript](https://www.typescriptlang.org/docs/handbook/intro.html) 类型速览（看源码时会遇到的）

在项目源码中你会频繁看到这些类型，先混个眼熟：

```tsx
// React.HTMLAttributes<HTMLButtonElement>
// 意思是："原生 <button> 标签上能写的所有属性"
// 包括 onClick、disabled、className、id、style 等等

// VariantProps<typeof buttonVariants>
// 意思是："cva 定义的变体属性"
// 对于 Button 来说，就是 { variant?: "default" | "destructive" | ..., size?: "default" | "sm" | "lg" }

// extends 关键字 —— 类型的"继承"
export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,  // 包含原生 button 属性
    VariantProps<typeof buttonVariants> {}                 // 再加上 variant、size 属性
// ButtonProps = 原生 button 的所有属性 + variant + size
```

---

## 二、核心概念清单

| 概念 | 说明 | 生活类比 |
|------|------|---------|
| **[Tailwind CSS](https://tailwindcss.com/docs/utility-first)** | 用预定义的"工具类"来写样式，比如 `bg-red-500` = 红色背景、`px-4` = 左右内边距 | 像乐高积木，每块积木是一个样式，拼在一起组成完整外观 |
| **`cn()` 函数** | `clsx` + `tailwind-merge` 的组合，智能拼接 CSS 类名，自动处理冲突 | 像一个聪明的助理：你给了两张背景色卡片，它只留最后一张 |
| **[`cva()` 函数](https://cva.style/docs)** | Class Variance Authority，用 JS 对象定义组件的多种样式"变体" | 像一个自动售货机：你按"大杯/中杯"按钮，它自动出对应的饮料 |
| **[Radix UI](https://www.radix-ui.com/primitives/docs/overview/introduction)** | 无样式的底层 UI 库，只提供行为（键盘导航、无障碍访问等） | 像毛坯房，结构和水电都搞好了，但没有装修 |
| **shadcn/ui** | Radix UI（行为）+ Tailwind（样式）的组合，代码直接复制到项目中 | 像精装修的毛坯房，可以随意改装 |
| **Compound Components** | 复合组件模式，一组组件搭配使用，如 `Card` + `CardHeader` + `CardContent` | 像一套餐具：盘子、碗、筷子各自独立，但组合使用 |
| **[`forwardRef`](https://react.dev/reference/react/forwardRef)** | React 的 ref 转发机制，让父组件能直接操作子组件里的 DOM 元素 | 像给房间装了一个窗口，外面的人可以直接操作里面的东西 |

---

## 三、源码精读

### 3.1 `cn()` 函数 — 项目中最常用的工具函数

文件路径：`lib/utils.ts`

```typescript
// -------- lib/utils.ts（节选） --------

import { ClassValue, clsx } from "clsx"       // clsx：条件拼接类名的小工具
import { twMerge } from "tailwind-merge"       // twMerge：智能合并 Tailwind 类名

export function cn(...inputs: ClassValue[]) {  // ...inputs 表示接收任意数量的参数
  return twMerge(clsx(inputs))                 // 先用 clsx 拼接，再用 twMerge 去重
}
```

**逐步拆解这个函数做了什么：**

**第 1 步 `clsx(inputs)` — 条件拼接类名：**

```typescript
clsx("bg-red-500", "text-white")
// 结果 → "bg-red-500 text-white"
// 就是把多个字符串拼在一起，用空格隔开

clsx("base", true && "active", false && "hidden")
// 结果 → "base active"
// false 的那个被自动忽略了！

clsx("text-sm", undefined, null, "font-bold")
// 结果 → "text-sm font-bold"
// undefined 和 null 也会被自动忽略
```

**第 2 步 `twMerge(...)` — 智能合并 Tailwind 类名：**

```typescript
twMerge("bg-red-500 bg-blue-500")
// 结果 → "bg-blue-500"
// 两个背景色冲突了！twMerge 只保留后面的

twMerge("px-4 px-6")
// 结果 → "px-6"
// 两个水平内边距冲突了！twMerge 只保留后面的

twMerge("bg-red-500 text-white px-4")
// 结果 → "bg-red-500 text-white px-4"
// 没有冲突，全部保留
```

**为什么需要 `cn()`？—— 让组件可以被"覆盖样式"：**

```tsx
function MyButton({ className }: { className?: string }) {
  return (
    <button className={cn(
      "bg-blue-500 px-4 py-2 rounded",  // 组件内部的默认样式
      className                           // 使用者传入的自定义样式
    )}>
      Click me
    </button>
  )
}

// 使用时传入 bg-red-500 来覆盖背景色
<MyButton className="bg-red-500" />

// cn() 的处理过程：
// 1. clsx("bg-blue-500 px-4 py-2 rounded", "bg-red-500")
//    → "bg-blue-500 px-4 py-2 rounded bg-red-500"
// 2. twMerge("bg-blue-500 px-4 py-2 rounded bg-red-500")
//    → "px-4 py-2 rounded bg-red-500"  ← bg-blue-500 被移除了！

// 如果不用 cn() 而是直接拼字符串：
// "bg-blue-500 px-4 py-2 rounded bg-red-500"
// 浏览器看到两个背景色，行为不可预测！
```

### 3.2 Button 组件 — 理解 `cva` 变体模式

文件路径：`components/ui/button.tsx`

这是项目中最典型的 UI 组件，我们逐行解读：

```tsx
// -------- components/ui/button.tsx --------

import * as React from "react"
// ↑ 导入 React 库，组件必须用到

import { VariantProps, cva } from "class-variance-authority"
// ↑ cva：用来定义组件的多种"变体"样式
// ↑ VariantProps：从 cva 定义中自动提取 TypeScript 类型

import { cn } from "@/lib/utils"
// ↑ 导入我们刚才讲的 cn() 函数
// ↑ @/ 是路径别名，代表项目根目录

// ========== 第 1 部分：用 cva 定义所有按钮样式 ==========

const buttonVariants = cva(
  // 第 1 个参数：所有按钮都有的「基础样式」
  // 不管你选什么变体，这些样式都会应用
  "inline-flex items-center justify-center rounded-md text-sm font-medium " +
  "transition-colors focus-visible:outline-none focus-visible:ring-2 " +
  "focus-visible:ring-ring focus-visible:ring-offset-2 " +
  "disabled:opacity-50 disabled:pointer-events-none ring-offset-background",
  // ↑ inline-flex items-center justify-center → 内容水平垂直居中
  // ↑ rounded-md → 中等圆角
  // ↑ text-sm font-medium → 小号字、中等粗细
  // ↑ transition-colors → 颜色变化时有过渡动画
  // ↑ disabled:opacity-50 → 禁用时变半透明
  {
    // 第 2 个参数：变体配置对象
    variants: {
      // ---------- 变体维度 1：外观风格 ----------
      variant: {
        default:
          "bg-primary text-primary-foreground hover:bg-primary/90",
          // ↑ 默认：主色背景 + 白色文字 + hover 时稍暗
        destructive:
          "bg-destructive text-destructive-foreground hover:bg-destructive/90",
          // ↑ 危险：红色背景（用于删除等危险操作）
        outline:
          "border border-input hover:bg-accent hover:text-accent-foreground",
          // ↑ 轮廓：只有边框，hover 时出现背景色
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
          // ↑ 次要：灰色背景
        ghost:
          "hover:bg-accent hover:text-accent-foreground",
          // ↑ 幽灵：平时透明，hover 时才出现背景
        link:
          "underline-offset-4 hover:underline text-primary",
          // ↑ 链接：看起来像文本链接
      },
      // ---------- 变体维度 2：尺寸 ----------
      size: {
        default: "h-10 py-2 px-4",    // 标准大小
        sm: "h-9 px-3 rounded-md",     // 小号
        lg: "h-11 px-8 rounded-md",    // 大号
      },
    },
    // ---------- 默认值 ----------
    defaultVariants: {
      variant: "default",  // 如果不传 variant，就用 default
      size: "default",     // 如果不传 size，就用 default
    },
  }
)

// ========== 第 2 部分：定义 Props 类型 ==========

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
  // ↑ 继承原生 <button> 的所有属性（onClick, disabled, type 等）
    VariantProps<typeof buttonVariants> {}
  // ↑ 再加上 cva 生成的 variant 和 size 属性
  // 最终 ButtonProps = { onClick?, disabled?, variant?, size?, className?, ... }

// ========== 第 3 部分：组件本体 ==========

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  // ↑ forwardRef：让外部可以通过 ref 直接访问内部的 <button> DOM 元素
  // ↑ <HTMLButtonElement, ButtonProps>：告诉 TS "ref 指向 button 元素，props 是 ButtonProps"
  ({ className, variant, size, ...props }, ref) => {
    // ↑ 从 props 中取出 className、variant、size
    // ↑ ...props 收集剩余的所有属性（onClick、disabled、children 等）
    // ↑ ref 是父组件传来的引用
    return (
      <button
        className={cn(buttonVariants({ variant, size, className }))}
        // ↑ buttonVariants({ variant, size }) 根据传入的变体生成对应的 class 字符串
        // ↑ className 参数让使用者可以追加/覆盖样式
        // ↑ cn() 智能合并，处理冲突
        ref={ref}
        // ↑ 把 ref 绑定到真实的 <button> 元素上
        {...props}
        // ↑ 把剩余属性（onClick、disabled 等）全部传给 <button>
      />
    )
  }
)
Button.displayName = "Button"
// ↑ 在 React DevTools 中显示组件名为 "Button"（forwardRef 组件需要手动设置）

export { Button, buttonVariants }
// ↑ 导出组件本身和变体函数
// ↑ 导出 buttonVariants 是为了让 <Link> 等非按钮元素也能复用按钮样式
```

**使用效果：**

```tsx
// 默认按钮（variant="default", size="default"）
<Button>Click me</Button>

// 红色危险按钮
<Button variant="destructive">Delete</Button>

// 轮廓按钮 + 大号
<Button variant="outline" size="lg">Learn More</Button>

// 幽灵按钮 + 小号
<Button variant="ghost" size="sm">Cancel</Button>

// 为什么要导出 buttonVariants？—— 让链接看起来像按钮：
import Link from "next/link"
import { buttonVariants } from "@/components/ui/button"

<Link href="/login" className={cn(buttonVariants({ size: "lg" }))}>
  Get Started
</Link>
// ↑ 这是一个 <a> 标签，但外观和按钮一模一样
```

### 3.3 Card 组件 — 复合组件模式

文件路径：`components/ui/card.tsx`

Card 组件展示了**复合组件（Compound Component）** 模式。一个"卡片"被拆成 6 个小组件：

```tsx
// -------- components/ui/card.tsx（关键部分） --------

// Card —— 最外层容器
const Card = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>    // 接收原生 <div> 的所有属性
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn(
      "rounded-lg border bg-card text-card-foreground shadow-sm",
      // ↑ 圆角、边框、背景色、文字色、阴影
      className   // 允许使用者覆盖样式
    )}
    {...props}
  />
))
Card.displayName = "Card"

// CardHeader —— 卡片头部
const CardHeader = React.forwardRef<...>(({ className, ...props }, ref) => (
  <div ref={ref} className={cn("flex flex-col space-y-1.5 p-6", className)} {...props} />
  //                               ↑ 纵向排列、子元素间距 1.5、内边距 6
))

// CardTitle —— 卡片标题
// CardDescription —— 卡片描述文字
// CardContent —— 卡片内容区域
// CardFooter —— 卡片底部

export { Card, CardHeader, CardFooter, CardTitle, CardDescription, CardContent }
```

**使用方式 —— 像搭积木一样组合：**

```tsx
<Card>
  <CardHeader>
    <CardTitle>你的名字</CardTitle>
    <CardDescription>请输入你的全名或昵称</CardDescription>
  </CardHeader>
  <CardContent>
    <input placeholder="输入名字" />
  </CardContent>
  <CardFooter>
    <Button>保存</Button>
  </CardFooter>
</Card>

{/* 不需要底部？不写 CardFooter 就行，非常灵活 */}
<Card>
  <CardHeader>
    <CardTitle>通知</CardTitle>
  </CardHeader>
  <CardContent>
    <p>你有 3 条未读消息</p>
  </CardContent>
</Card>
```

### 3.4 Icons 组件 — 图标的统一管理

文件路径：`components/icons.tsx`

```tsx
// -------- components/icons.tsx（节选） --------

// 从 lucide-react 图标库导入需要的图标
import { Command, X, Loader2, Plus, Moon, SunMedium, ... } from "lucide-react"

// 用一个对象集中管理所有图标，起语义化的名字
export const Icons = {
  logo: Command,         // 项目 logo
  close: X,              // 关闭按钮
  spinner: Loader2,      // 加载动画
  add: Plus,             // 添加
  sun: SunMedium,        // 亮色模式图标
  moon: Moon,            // 暗色模式图标
  // ... 更多图标

  // 对于没有现成图标的（如 GitHub logo），用自定义 SVG
  gitHub: ({ ...props }: LucideProps) => (
    <svg viewBox="0 0 496 512" {...props}>
      <path fill="currentColor" d="M165.9 397.4c0 2-2.3..." />
    </svg>
  ),
}
```

**使用方式：**

```tsx
import { Icons } from "@/components/icons"

<Icons.spinner className="h-4 w-4 animate-spin" />  // 旋转加载图标
<Icons.add className="mr-2 h-4 w-4" />               // 添加图标
<Icons.gitHub className="h-5 w-5" />                  // GitHub 图标
```

**为什么这样管理图标？**
- 换图标库时只改 `icons.tsx` 这一个文件
- 语义化命名：`Icons.billing` 比 `CreditCard` 更清晰

---

## 四、概念图解 — 组件的组合关系

```
shadcn/ui 组件的"技术栈层级"
================================

  你写的页面代码
       │
       ▼
  ┌──────────────┐
  │  shadcn/ui   │  ← 最终组件（Button、Card、Dialog……）
  │  组件层      │    代码在你的 components/ui/ 目录中
  └──────┬───────┘
         │ 组合了以下三样东西：
         ▼
  ┌──────────────────────────────────────────────┐
  │                                              │
  │  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
  │  │ Radix UI │  │ Tailwind │  │ cva      │   │
  │  │ (行为)   │  │ CSS      │  │ (变体)   │   │
  │  │          │  │ (样式)   │  │          │   │
  │  └──────────┘  └──────────┘  └──────────┘   │
  │                      │                       │
  │                      ▼                       │
  │               ┌──────────┐                   │
  │               │  cn()    │  ← 智能合并类名   │
  │               │ =clsx +  │                   │
  │               │ twMerge  │                   │
  │               └──────────┘                   │
  └──────────────────────────────────────────────┘

Button 组件的数据流
================================

  <Button variant="destructive" size="lg" onClick={fn}>
       │
       ▼
  buttonVariants({ variant: "destructive", size: "lg" })
       │
       ▼
  生成类名字符串：
  "inline-flex items-center ... bg-destructive ... h-11 px-8 ..."
       │
       ▼
  cn(上面的字符串, 用户传入的 className)
       │
       ▼
  最终 className 写到 <button> 标签上
```

---

## 五、动手练习

### 练习 1：在终端里体验 `cn()` 的效果（20 分钟）

**目标：** 亲手验证 `clsx` 和 `tailwind-merge` 的行为。

**步骤：**

1. 打开终端，进入项目目录
2. 启动 Node.js 交互式环境：
   ```bash
   node
   ```
3. 逐行输入以下代码，观察输出：
   ```javascript
   // 导入（注意路径，在项目根目录下执行）
   const clsx = require("clsx").clsx

   // 测试 clsx：基础拼接
   clsx("a", "b", "c")
   // 你觉得结果是什么？

   // 测试 clsx：条件拼接
   clsx("base", true && "active", false && "hidden")
   // 你觉得结果是什么？

   // 测试 clsx：忽略空值
   clsx("text-sm", undefined, null, "", "font-bold")
   // 你觉得结果是什么？
   ```

4. 再测试 tailwind-merge：
   ```javascript
   const { twMerge } = require("tailwind-merge")

   // 冲突类名会怎样？
   twMerge("bg-red-500 bg-blue-500")
   // 你觉得结果是什么？

   twMerge("px-4 px-6")
   // 你觉得结果是什么？

   // 不冲突的呢？
   twMerge("bg-red-500 text-white px-4")
   // 你觉得结果是什么？
   ```

**排错提示：**
- 如果报 `Cannot find module 'clsx'`，先在项目根目录运行 `npm install`
- 如果用的是 ES Module 项目，可能需要写个小脚本文件而不是在 REPL 里直接 `require`

### 练习 2：创建一个 Alert 组件（40 分钟）

**目标：** 仿照 Button 组件的模式，用 `cva` 创建一个 Alert（提示框）组件。

**步骤：**

1. 创建文件 `components/ui/my-alert.tsx`（用 `my-` 前缀避免和项目已有的 `alert.tsx` 冲突）

2. 写入以下代码框架，把 `???` 部分补充完整：

```tsx
import * as React from "react"
import { cva, type VariantProps } from "class-variance-authority"
import { cn } from "@/lib/utils"

// 第 1 步：定义变体
const alertVariants = cva(
  // 基础样式：圆角、内边距、边框
  "rounded-lg border p-4 text-sm",
  {
    variants: {
      variant: {
        default: "bg-background text-foreground",
        // ??? 添加一个 warning 变体：黄色背景、深色文字
        // ??? 添加一个 error 变体：红色背景、白色文字
      },
    },
    defaultVariants: {
      variant: "default",
    },
  }
)

// 第 2 步：定义 Props 类型
export interface MyAlertProps
  extends React.HTMLAttributes<HTMLDivElement>,
    VariantProps<typeof alertVariants> {
  title?: string  // 可选的标题
}

// 第 3 步：写组件
function MyAlert({ className, variant, title, children, ...props }: MyAlertProps) {
  return (
    <div className={cn(alertVariants({ variant }), className)} {...props}>
      {/* ??? 如果有 title，就渲染一个加粗的标题 */}
      {children}
    </div>
  )
}

export { MyAlert, alertVariants }
```

3. 在任意页面中测试：

```tsx
<MyAlert>这是一条普通提示</MyAlert>
<MyAlert variant="warning" title="注意">请检查你的输入</MyAlert>
<MyAlert variant="error" title="错误">操作失败，请重试</MyAlert>
```

**排错提示：**
- Tailwind 的黄色类名：`bg-yellow-50`（浅黄背景）、`text-yellow-800`（深黄文字）、`border-yellow-200`（黄色边框）
- Tailwind 的红色类名：`bg-red-50`（浅红背景）、`text-red-800`（深红文字）、`border-red-200`（红色边框）
- 如果样式不生效，检查 `tailwind.config.js` 中的 `content` 配置是否包含了你的文件路径

### 练习 3：用 Card 组件搭建一个用户信息卡片（30 分钟）

**目标：** 练习复合组件的使用方式。

**步骤：**

1. 在任意页面文件中导入 Card 系列组件：
   ```tsx
   import { Card, CardHeader, CardTitle, CardDescription, CardContent, CardFooter } from "@/components/ui/card"
   import { Button } from "@/components/ui/button"
   ```

2. 搭建卡片：
   ```tsx
   <Card className="w-[350px]">
     <CardHeader>
       <CardTitle>用户信息</CardTitle>
       <CardDescription>这是你的个人资料</CardDescription>
     </CardHeader>
     <CardContent>
       {/* 在这里添加：头像（可以用一个圆形 div 代替）、名字、邮箱 */}
     </CardContent>
     <CardFooter>
       <Button>编辑资料</Button>
     </CardFooter>
   </Card>
   ```

3. 尝试做以下改动，观察效果：
   - 给 Card 加上 `className="w-[350px] mx-auto"`（水平居中）
   - 给 Button 换成 `variant="outline"`
   - 去掉 CardFooter，看看卡片是否仍然正常显示

**排错提示：**
- 如果卡片看不见，可能是背景色和卡片颜色一样。试试给外层 div 加个深色背景
- `w-[350px]` 是 Tailwind 的"任意值"语法，方括号里写具体的 CSS 值

---

## 六、自检问题（附答案）

**Q1：`cn("p-4", "p-6")` 的结果是什么？为什么不是 `"p-4 p-6"`？**

> **答案：** 结果是 `"p-6"`。因为 `cn()` 内部的 `tailwind-merge` 检测到 `p-4` 和 `p-6` 都是 padding 属性，产生了冲突，所以只保留后面的 `p-6`。如果两个都保留，浏览器的行为是不确定的（取决于 CSS 文件中两个类的定义顺序）。

**Q2：`cva` 的 `defaultVariants` 有什么作用？如果不设置会怎样？**

> **答案：** `defaultVariants` 定义了不传参数时使用的默认变体。比如 `<Button>` 不传 variant，就会自动使用 `variant: "default"` 的样式。如果不设置 `defaultVariants`，那么不传参数时就不会应用任何变体样式，按钮只会有基础样式（没有背景色、没有指定尺寸）。

**Q3：Card 组件为什么要拆成 `Card`、`CardHeader`、`CardContent` 等多个子组件？**

> **答案：** 这是"复合组件"模式，好处是灵活。使用者可以自由组合——不需要底部就不写 `CardFooter`，需要多个内容区就写多个 `CardContent`。如果用 props 传入所有内容（如 `<Card title="..." content="..." footer="...">`），一旦需求变化（比如 footer 里要放两个按钮），props 就变得非常复杂。

**Q4：`forwardRef` 在什么场景下是必要的？**

> **答案：** 当外部需要直接操作组件内部的 DOM 元素时。最常见的场景是表单库（如 react-hook-form）需要通过 `ref` 来注册和读取输入框的值。如果组件不用 `forwardRef`，表单库拿不到 DOM 引用，就无法正常管理表单状态。

**Q5：为什么 Button 组件要单独导出 `buttonVariants` 这个函数？**

> **答案：** 因为有时候你想要按钮的"外观"，但不想用 `<button>` 标签。比如用 Next.js 的 `<Link>` 做一个看起来像按钮的链接：`<Link className={buttonVariants({ size: "lg" })}>Get Started</Link>`。导出变体函数让样式可以脱离组件使用。

---

## 七、预计时间分配

| 环节 | 时间 | 说明 |
|------|------|------|
| 前置知识速补 | 40 分钟 | JSX、props、条件渲染等基础 |
| 核心概念 + 源码精读 | 60 分钟 | cn()、cva、Button、Card 源码逐行读 |
| 练习 1（cn 体验） | 20 分钟 | 在终端里动手验证 |
| 练习 2（Alert 组件） | 40 分钟 | 仿写一个新组件 |
| 练习 3（Card 卡片） | 30 分钟 | 练习复合组件用法 |
| 自检与笔记整理 | 20 分钟 | 回答自检问题、记笔记 |
| **合计** | **约 3.5 小时** | |

---

## 八、常见踩坑点

1. **`className` 不是 `class`**
   JSX 中必须写 `className`，写 `class` 会报错或不生效。这是初学者最容易犯的错误。

2. **忘记用 `cn()` 合并 className**
   ```tsx
   // 错误写法 —— 直接用 className，丢失了组件内部的默认样式
   <div className={className}>

   // 正确写法 —— 用 cn() 合并
   <div className={cn("默认样式", className)}>
   ```

3. **Tailwind 类名冲突不报错**
   如果你写了 `className="p-4 p-6"`，不用 `cn()` 的话浏览器不会报错，但表现不可预测。一定要用 `cn()` 处理冲突。

4. **`cva` 的变体名拼写错误**
   如果你写了 `<Button variant="destrutive">`（少了一个 c），TypeScript 会报错。但如果你忽略了报错，按钮只会显示基础样式而没有变体样式，看起来"样式丢了"。

5. **自定义组件忘记传递 `...props`**
   ```tsx
   // 错误 —— onClick、disabled 等属性会丢失
   const MyBtn = ({ className }: ButtonProps) => (
     <button className={cn("...", className)}>Click</button>
   )

   // 正确 —— 用 ...props 收集并传递剩余属性
   const MyBtn = ({ className, ...props }: ButtonProps) => (
     <button className={cn("...", className)} {...props}>Click</button>
   )
   ```

6. **Radix UI 组件的受控/非受控混用**
   有些 Radix 组件（如 Dialog）同时支持受控模式（你管理 open 状态）和非受控模式（组件自己管理）。如果你一会儿传 `open` 一会儿不传，会导致奇怪的行为。选定一种模式，保持一致。

---

## 九、延伸阅读

| 资源 | 链接 | 说明 |
|------|------|------|
| React 官方教程（中文） | https://zh-hans.react.dev/learn | 零基础入门 React 的最佳起点 |
| Tailwind CSS 文档 | https://tailwindcss.com/docs | 查看所有工具类的作用 |
| cva 官方文档 | https://cva.style/docs | class-variance-authority 的详细用法 |
| shadcn/ui 官网 | https://ui.shadcn.com | 查看所有可用组件及其代码 |
| Radix UI 文档 | https://www.radix-ui.com/docs/primitives | 底层组件的 API 文档 |
| clsx 文档 | https://github.com/lukeed/clsx | 理解条件类名拼接的所有用法 |
| tailwind-merge 文档 | https://github.com/dcastil/tailwind-merge | 理解类名冲突合并的规则 |
