# 第3章：UI 组件层 — 按钮、卡片这些界面元素是怎么搭出来的？

## 学习目标

- 理解 React 组件的基本概念（props、JSX）
- 理解 Tailwind CSS 工具类的写法
- 理解 shadcn/ui 组件库（基于 Radix UI）的设计思路
- 理解 `cn()` 工具函数和 `cva`（class-variance-authority）

## 涉及文件

| 文件 | 说明 |
|------|------|
| `lib/utils.ts` | cn() 工具函数 |
| `components/ui/button.tsx` | 按钮组件（cva 变体示例） |
| `components/ui/card.tsx` | 卡片组件 |
| `components/ui/input.tsx` | 输入框 |
| `components/ui/toast.tsx` | Toast 通知 |
| `components/ui/use-toast.ts` | Toast 状态管理 |
| `components/ui/dialog.tsx` | 对话框 |
| `components/ui/dropdown-menu.tsx` | 下拉菜单 |
| `components/icons.tsx` | 图标组件 |

---

## 1. React 组件基础回顾

如果你完全没接触过 React，先理解这几个核心概念：

### 1.1 组件 = 函数

React 组件就是一个返回 JSX（长得像 HTML 的 JavaScript）的函数：

```tsx
// 最简单的组件
function Hello() {
  return <h1>Hello World</h1>
}
```

### 1.2 Props = 组件的参数

```tsx
// 接收参数的组件
function Greeting({ name }: { name: string }) {
  return <h1>Hello, {name}!</h1>
}

// 使用时传入参数
<Greeting name="Alice" />   // 渲染：Hello, Alice!
<Greeting name="Bob" />     // 渲染：Hello, Bob!
```

### 1.3 JSX 中的 JavaScript

在 JSX 中用花括号 `{}` 插入 JavaScript 表达式：

```tsx
function UserCard({ name, age }: { name: string; age: number }) {
  return (
    <div>
      <h2>{name}</h2>                          {/* 变量 */}
      <p>{age >= 18 ? "成人" : "未成年"}</p>    {/* 三元表达式 */}
      <p>{`${name} is ${age} years old`}</p>    {/* 模板字符串 */}
    </div>
  )
}
```

---

## 2. cn() — 最常用的工具函数

项目中几乎每个组件都用到了 `cn()` 函数。它定义在 `lib/utils.ts` 中：

```typescript
// lib/utils.ts
import { ClassValue, clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}
```

这只有一行代码，但理解它很重要。它做了两件事：

### 第1步：`clsx()` — 条件拼接类名

```typescript
import { clsx } from "clsx"

clsx("bg-red-500", "text-white")
// → "bg-red-500 text-white"

clsx("base-class", isActive && "active-class")
// isActive 为 true  → "base-class active-class"
// isActive 为 false → "base-class"

clsx("text-sm", undefined, null, "font-bold")
// → "text-sm font-bold"（自动忽略 undefined 和 null）
```

### 第2步：`twMerge()` — 智能合并 Tailwind 类名

```typescript
import { twMerge } from "tailwind-merge"

twMerge("bg-red-500 bg-blue-500")
// → "bg-blue-500"（后面的覆盖前面的，避免冲突）

twMerge("px-4 px-6")
// → "px-6"（后面的覆盖前面的）
```

### 为什么需要 `cn()`？

```tsx
// 一个可以自定义样式的组件
function MyButton({ className }: { className?: string }) {
  return (
    <button className={cn(
      "bg-blue-500 px-4 py-2 rounded",  // 默认样式
      className                           // 用户传入的自定义样式
    )}>
      Click me
    </button>
  )
}

// 使用时覆盖背景色
<MyButton className="bg-red-500" />
// cn() 会智能处理：移除 bg-blue-500，保留 bg-red-500
// 最终：className="px-4 py-2 rounded bg-red-500"
```

如果不用 `cn()`，而是简单拼接字符串，结果会是 `"bg-blue-500 px-4 py-2 rounded bg-red-500"`，浏览器会不知道该用哪个背景色。

---

## 3. Button 组件 — 理解 cva（class-variance-authority）

`components/ui/button.tsx` 是项目中最典型的组件，展示了如何用 `cva` 管理组件变体：

```tsx
// components/ui/button.tsx
import * as React from "react"
import { VariantProps, cva } from "class-variance-authority"
import { cn } from "@/lib/utils"

// cva 定义了按钮的所有样式变体
const buttonVariants = cva(
  // 第1个参数：所有按钮共享的基础样式
  "inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:opacity-50 disabled:pointer-events-none ring-offset-background",
  {
    // 第2个参数：变体配置
    variants: {
      // 变体1：外观风格
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive: "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline: "border border-input hover:bg-accent hover:text-accent-foreground",
        secondary: "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "underline-offset-4 hover:underline text-primary",
      },
      // 变体2：尺寸
      size: {
        default: "h-10 py-2 px-4",
        sm: "h-9 px-3 rounded-md",
        lg: "h-11 px-8 rounded-md",
      },
    },
    // 默认值
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)

// 按钮组件的 Props 类型
// ButtonHTMLAttributes = 原生 button 的所有属性（onClick, disabled 等）
// VariantProps = cva 生成的 variant 和 size 属性
export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {}

// 使用 forwardRef 让父组件可以获取到 button DOM 元素的引用
const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, ...props }, ref) => {
    return (
      <button
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}  // 展开传入的其他属性（onClick, disabled 等）
      />
    )
  }
)
Button.displayName = "Button"

export { Button, buttonVariants }
```

### 使用效果

```tsx
// 默认按钮
<Button>Click me</Button>

// 红色危险按钮
<Button variant="destructive">Delete</Button>

// 轮廓按钮 + 大号
<Button variant="outline" size="lg">Learn More</Button>

// 幽灵按钮（无背景，hover 时有背景）
<Button variant="ghost" size="sm">Cancel</Button>
```

### 为什么导出 `buttonVariants`？

有时候你想要按钮的样式，但不想用 `<button>` 标签（比如用 `<Link>` 标签）：

```tsx
// 首页中的例子 — 链接看起来像按钮
import Link from "next/link"
import { buttonVariants } from "@/components/ui/button"

<Link href="/login" className={cn(buttonVariants({ size: "lg" }))}>
  Get Started
</Link>
```

---

## 4. Card 组件 — 复合组件模式

`components/ui/card.tsx` 展示了 **复合组件（Compound Component）** 模式：

```tsx
// components/ui/card.tsx
const Card = React.forwardRef<HTMLDivElement, React.HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div
      ref={ref}
      className={cn(
        "rounded-lg border bg-card text-card-foreground shadow-sm",
        className
      )}
      {...props}
    />
  )
)

const CardHeader = React.forwardRef<HTMLDivElement, React.HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div ref={ref} className={cn("flex flex-col space-y-1.5 p-6", className)} {...props} />
  )
)

const CardTitle = React.forwardRef<HTMLParagraphElement, React.HTMLAttributes<HTMLHeadingElement>>(
  ({ className, ...props }, ref) => (
    <h3 ref={ref} className={cn("text-lg font-semibold leading-none tracking-tight", className)} {...props} />
  )
)

const CardDescription = React.forwardRef<HTMLParagraphElement, React.HTMLAttributes<HTMLParagraphElement>>(
  ({ className, ...props }, ref) => (
    <p ref={ref} className={cn("text-sm text-muted-foreground", className)} {...props} />
  )
)

const CardContent = React.forwardRef<HTMLDivElement, React.HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div ref={ref} className={cn("p-6 pt-0", className)} {...props} />
  )
)

const CardFooter = React.forwardRef<HTMLDivElement, React.HTMLAttributes<HTMLDivElement>>(
  ({ className, ...props }, ref) => (
    <div ref={ref} className={cn("flex items-center p-6 pt-0", className)} {...props} />
  )
)
```

### 使用方式

```tsx
<Card>
  <CardHeader>
    <CardTitle>Your Name</CardTitle>
    <CardDescription>
      Please enter your full name or a display name.
    </CardDescription>
  </CardHeader>
  <CardContent>
    <Input placeholder="Enter your name" />
  </CardContent>
  <CardFooter>
    <Button>Save</Button>
  </CardFooter>
</Card>
```

**大白话：** Card 就像一个容器盒子，里面有固定的"格子"：头部、内容区、底部。每个格子都是一个独立的子组件。你可以自由组合——不需要底部就不写 `CardFooter`。

---

## 5. forwardRef 是什么？

你会发现每个 UI 组件都用了 `React.forwardRef`。简单解释：

```tsx
// 没有 forwardRef 的组件
function MyInput(props) {
  return <input {...props} />
}

// 有 forwardRef 的组件
const MyInput = React.forwardRef((props, ref) => {
  return <input ref={ref} {...props} />
})
```

`ref` 让父组件可以直接操作 DOM 元素。比如表单库（react-hook-form）需要 ref 来跟踪输入框的值。如果不用 forwardRef，表单库就无法正常工作。

**作为初学者，你只需要知道：** UI 组件一般都要用 forwardRef，这是一个标准做法。

---

## 6. Radix UI — 无障碍访问的 UI 基础

项目中的 Dialog（对话框）、DropdownMenu（下拉菜单）、Toast（通知）等组件都基于 **Radix UI**。

**Radix UI 是什么？** 它是一个提供"行为"但不提供"样式"的组件库：
- 它处理复杂的交互逻辑（键盘导航、焦点管理、屏幕阅读器支持）
- 它不带任何样式（你用 Tailwind 自己加）

```tsx
// components/ui/dropdown-menu.tsx（简化版）
import * as DropdownMenuPrimitive from "@radix-ui/react-dropdown-menu"

// 用 Radix 的行为 + Tailwind 的样式
const DropdownMenuContent = React.forwardRef(({ className, ...props }, ref) => (
  <DropdownMenuPrimitive.Portal>
    <DropdownMenuPrimitive.Content
      ref={ref}
      className={cn(
        // 这些样式是我们自己加的
        "z-50 min-w-[8rem] overflow-hidden rounded-md border bg-popover p-1 text-popover-foreground shadow-md",
        // 动画
        "animate-in data-[side=bottom]:slide-in-from-top-2",
        className
      )}
      {...props}
    />
  </DropdownMenuPrimitive.Portal>
))
```

**大白话：** Radix UI 负责"行为"（点击打开、按 Esc 关闭、键盘导航），Tailwind 负责"外观"（圆角、阴影、颜色）。两者配合 = shadcn/ui 组件。

---

## 7. Icons 组件 — 图标的统一管理

```tsx
// components/icons.tsx
import {
  AlertTriangle, ArrowRight, Check, ChevronLeft, Command,
  CreditCard, File, FileText, HelpCircle, Image, Laptop,
  Loader2, Moon, MoreVertical, Pizza, Plus, Settings,
  SunMedium, Trash, Twitter, User, X,
  type Icon as LucideIcon,
} from "lucide-react"

export const Icons = {
  logo: Command,         // 项目 logo 用的是 Command 图标
  close: X,              // 关闭
  spinner: Loader2,      // 加载动画
  trash: Trash,          // 删除
  post: FileText,        // 帖子
  settings: Settings,    // 设置
  billing: CreditCard,   // 账单
  add: Plus,             // 添加
  sun: SunMedium,        // 太阳（亮色模式）
  moon: Moon,            // 月亮（暗色模式）

  // 自定义 SVG 图标
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

<Icons.spinner className="h-4 w-4 animate-spin" />  // 旋转的加载图标
<Icons.add className="mr-2 h-4 w-4" />               // 添加图标
<Icons.gitHub className="h-5 w-5" />                  // GitHub 图标
```

**为什么这样做？** 集中管理图标有两个好处：
1. 换图标库时只改一个文件（比如从 lucide 换成 heroicons）
2. 使用时语义化（`Icons.billing` 比 `CreditCard` 更清晰）

---

## 8. Toast 通知 — 状态管理实战

Toast 是那种从屏幕角落弹出的小通知。它的实现涉及到一个小型的**状态管理系统**：

```typescript
// components/ui/use-toast.ts（简化版）

// 用一个数组存储所有 toast
const TOAST_LIMIT = 1  // 同时最多显示 1 个

// 动作类型
type Action =
  | { type: "ADD_TOAST"; toast: Toast }
  | { type: "UPDATE_TOAST"; toast: Partial<Toast> }
  | { type: "DISMISS_TOAST"; toastId?: string }
  | { type: "REMOVE_TOAST"; toastId?: string }

// 提供一个 toast() 函数给外部调用
function toast({ title, description, variant }) {
  const id = genId()

  dispatch({
    type: "ADD_TOAST",
    toast: { id, title, description, variant, open: true },
  })

  return { id, dismiss: () => dispatch({ type: "DISMISS_TOAST", toastId: id }) }
}
```

**使用方式：**

```tsx
import { toast } from "@/components/ui/use-toast"

// 成功通知
toast({
  title: "保存成功",
  description: "你的修改已保存。",
})

// 错误通知
toast({
  title: "出错了",
  description: "请稍后重试。",
  variant: "destructive",  // 红色样式
})
```

---

## 小练习

### 练习 1：理解 cn() 函数
在浏览器控制台或 Node.js 中试试这些：
```javascript
// 安装依赖后
const { clsx } = require("clsx")

clsx("a", "b")              // → ?
clsx("a", false && "b")     // → ?
clsx("a", undefined, "c")   // → ?
```

### 练习 2：创建一个 Badge 组件
参考 Button 组件的写法，创建一个 Badge（徽章）组件：
```tsx
// components/ui/badge.tsx
// 提示：用 cva 定义 default 和 secondary 两个变体
```

### 练习 3：使用 Card 组件
在某个页面中使用 Card 组件创建一个简单的用户信息卡片，包含头像、名字和邮箱。

### 练习 4：添加一个新图标
1. 在 `components/icons.tsx` 中添加一个新的图标映射
2. 从 `lucide-react` 导入一个你喜欢的图标
3. 在某个组件中使用它

---

## 本章小结

- `cn()` 是项目中最常用的工具函数，用于智能合并 CSS 类名
- `cva` 让组件支持多种样式变体（如按钮的 variant 和 size）
- 组件使用 `forwardRef` 来支持 ref 转发
- Radix UI 提供行为，Tailwind 提供样式，两者配合形成 shadcn/ui
- Icons 集中管理，方便替换和使用
- Toast 使用 reducer 模式管理通知状态

下一章我们将学习页面的布局和导航系统。
