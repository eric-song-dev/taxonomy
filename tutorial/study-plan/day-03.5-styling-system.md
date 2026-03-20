# Day 03.5 — 样式系统深入（Styling System）

> 对应章节：[Chapter 3.5: 样式系统深入](../chapter-03.5-styling-system.md)

---

## 一、今日学习目标

1. **理解 CSS 变量主题系统**：知道亮色/暗色模式是怎样通过一套 CSS 变量切换的
2. **能追踪一个 Tailwind 类名的完整链路**：从 `bg-primary` 到 CSS 变量再到实际颜色值
3. **理解自定义 Hook 的设计模式**：知道 `useLockBody` 和 `useMounted` 解决了什么问题

---

## 二、前置知识速补

> 今天的内容涉及 CSS 和 React Hooks 的基础概念。如果你已经掌握，可以跳过。

### 2.1 [CSS 选择器](https://developer.mozilla.org/zh-CN/docs/Web/CSS/CSS_selectors)基础

```css
/* 标签选择器 — 匹配所有同名标签 */
h1 { color: red; }

/* 类选择器 — 匹配 class 属性 */
.dark { background: black; }

/* 伪根选择器 — 匹配文档根元素（<html>） */
:root { --my-color: blue; }

/* 属性选择器 — 匹配有特定属性的元素 */
[data-theme="dark"] { background: black; }
```

**为什么要学这个？** `styles/globals.css` 里用 `:root` 和 `.dark` 来定义两套主题变量。

### 2.2 [CSS 变量（Custom Properties）](https://developer.mozilla.org/zh-CN/docs/Web/CSS/Using_CSS_custom_properties)

```css
/* 定义变量（以 -- 开头） */
:root {
  --primary-color: #3490dc;
}

/* 使用变量 */
.button {
  color: var(--primary-color);
}

/* 变量可以被覆盖 */
.dark {
  --primary-color: #90cdf4;  /* 暗色模式用不同的蓝色 */
}
```

**为什么要学这个？** Taxonomy 的整个主题系统都基于 CSS 变量。

### 2.3 [HSL 颜色模式](https://developer.mozilla.org/zh-CN/docs/Web/CSS/color_value/hsl)

```
HSL = Hue（色相）Saturation（饱和度）Lightness（亮度）

hsl(222.2, 47.4%, 11.2%)
     │       │       │
     │       │       └── 亮度 11.2%（接近黑色）
     │       └────────── 饱和度 47.4%（中等鲜艳）
     └──────────────── 色相 222.2°（蓝色系）
```

**为什么要学这个？** `globals.css` 中的 CSS 变量值都是 HSL 格式的数字。

### 2.4 React [useEffect](https://react.dev/reference/react/useEffect) 和 [useLayoutEffect](https://react.dev/reference/react/useLayoutEffect)

```javascript
// useEffect — 在浏览器绘制 UI 之后执行
React.useEffect(() => {
  console.log("页面已渲染")
  return () => console.log("组件将卸载，执行清理")
}, [])

// useLayoutEffect — 在浏览器绘制 UI 之前执行
// 用于需要立即修改 DOM 样式的场景
React.useLayoutEffect(() => {
  document.body.style.overflow = "hidden"
  return () => { document.body.style.overflow = "auto" }
}, [])
```

**为什么要学这个？** 自定义 Hook `useLockBody` 用 `useLayoutEffect` 来锁定滚动，避免闪烁。

### 2.5 React [useState](https://react.dev/reference/react/useState)

```javascript
// useState — 在组件中保存可变状态
const [mounted, setMounted] = React.useState(false)
// mounted 是当前值（初始为 false）
// setMounted 是更新函数
// 调用 setMounted(true) 后组件重新渲染，mounted 变为 true
```

**为什么要学这个？** `useMounted` Hook 就是用 `useState` + `useEffect` 来检测组件是否已挂载。

---

## 三、核心概念清单

| 概念 | 英文 | 一句话解释 | 在项目中对应的文件 |
|------|------|-----------|-------------------|
| CSS 变量 | CSS Custom Properties | 在 CSS 中定义可复用的值 | `styles/globals.css` |
| 主题系统 | Theming System | 通过切换 CSS 变量实现亮/暗模式 | `styles/globals.css` + `tailwind.config.js` |
| [@layer base](https://developer.mozilla.org/zh-CN/docs/Web/CSS/@layer) | CSS Layer | Tailwind 的样式层级，`base` 是最底层 | `styles/globals.css` |
| [@apply](https://tailwindcss.com/docs/functions-and-directives#apply) | Tailwind directive | 在 CSS 文件中使用 Tailwind 类名 | `styles/globals.css` |
| [next/font](https://nextjs.org/docs/app/building-your-application/optimizing/fonts) | Next.js Font Optimization | 自动优化字体加载，避免闪烁 | `app/layout.tsx` |
| [theme()](https://tailwindcss.com/docs/functions-and-directives#theme) | Tailwind theme function | 在 CSS 中引用 Tailwind 配置值 | `styles/editor.css` |
| [自定义 Hook](https://react.dev/learn/reusing-logic-with-custom-hooks) | Custom Hook | 以 `use` 开头的可复用逻辑函数 | `hooks/use-lock-body.ts` |
| Hydration | SSR Hydration | 服务端 HTML 与客户端 JS "对接"的过程 | `hooks/use-mounted.ts` |

---

## 四、源码精读

### 4.1 `styles/globals.css` — 主题变量系统

> 文件路径：`/styles/globals.css`

这是整个样式系统的根基。我们逐段读：

**第一段：Tailwind 指令**

```css
@tailwind base;        /* Tailwind 的基础样式（reset 等） */
@tailwind components;  /* Tailwind 的组件类 */
@tailwind utilities;   /* Tailwind 的工具类（如 flex, p-4 等） */
```

这三行让 Tailwind CSS 注入它的所有类。

**第二段：亮色模式变量**

```css
@layer base {
  :root {
    --background: 0 0% 100%;         /* HSL 裸值：白色 */
    --foreground: 222.2 47.4% 11.2%; /* 深蓝灰色 */
    --primary: 222.2 47.4% 11.2%;    /* 主色调 = 前景色 */
    --primary-foreground: 210 40% 98%; /* 主色调上的文字 = 接近白色 */
    --destructive: 0 100% 50%;       /* 红色 */
    --border: 214.3 31.8% 91.4%;     /* 浅灰色边框 */
    --radius: 0.5rem;                 /* 圆角大小（不是颜色！） */
  }
}
```

注意变量值只有数字，没有 `hsl()`。这是为了配合 Tailwind 的透明度功能。

**第三段：暗色模式变量**

```css
  .dark {
    --background: 224 71% 4%;         /* 深色背景 */
    --foreground: 213 31% 91%;        /* 浅色文字 */
    --primary: 210 40% 98%;           /* 主色调反转为浅色 */
    --primary-foreground: 222.2 47.4% 1.2%; /* 深色文字 */
    --border: 216 34% 17%;            /* 暗色边框 */
  }
```

**关键观察：** 亮色模式下 `--primary` 是深色，暗色模式下 `--primary` 是浅色。这意味着 `bg-primary` 在亮色模式显示深色背景，暗色模式显示浅色背景——自动适配！

**第四段：全局基础样式**

```css
@layer base {
  * {
    @apply border-border;  /* 所有元素默认使用 --border 颜色 */
  }
  body {
    @apply bg-background text-foreground;  /* body 使用主题背景和前景色 */
  }
}
```

### 4.2 `tailwind.config.js` — 颜色和字体映射

> 文件路径：`/tailwind.config.js`

这个文件把 CSS 变量"翻译"成 Tailwind 可用的类名：

Tailwind 的 [darkMode](https://tailwindcss.com/docs/dark-mode) 配置决定了暗色模式的切换方式：

```javascript
module.exports = {
  darkMode: ["class"],  // 暗色模式通过 CSS class 控制

  theme: {
    extend: {
      colors: {
        // 简单映射
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",
        border: "hsl(var(--border))",

        // 带子颜色的映射
        primary: {
          DEFAULT: "hsl(var(--primary))",           // bg-primary
          foreground: "hsl(var(--primary-foreground))", // text-primary-foreground
        },
      },

      fontFamily: {
        sans: ["var(--font-sans)", ...fontFamily.sans],
        heading: ["var(--font-heading)", ...fontFamily.sans],
      },
    },
  },
}
```

**翻译后的效果：**

| 写的类名 | 实际 CSS | 亮色模式 | 暗色模式 |
|---------|---------|---------|---------|
| `bg-background` | `background: hsl(var(--background))` | 白色 | 深色 |
| `text-foreground` | `color: hsl(var(--foreground))` | 深灰 | 浅灰 |
| `bg-primary` | `background: hsl(var(--primary))` | 深蓝灰 | 浅色 |
| `border-border` | `border-color: hsl(var(--border))` | 浅灰 | 暗灰 |

### 4.3 `app/layout.tsx` — 字体注入

> 文件路径：`/app/layout.tsx`（字体相关部分）

```typescript
// Google 字体
import { Inter as FontSans } from "next/font/google"
const fontSans = FontSans({
  subsets: ["latin"],
  variable: "--font-sans",  // CSS 变量名
})

// 本地字体
import localFont from "next/font/local"
const fontHeading = localFont({
  src: "../assets/fonts/CalSans-SemiBold.woff2",
  variable: "--font-heading",
})

// 在 <html> 标签上注入字体 CSS 变量
<html className={cn(fontSans.variable, fontHeading.variable)}>
```

### 4.4 `styles/editor.css` — 第三方库暗色适配

> 文件路径：`/styles/editor.css`

EditorJS 不自带暗色模式，所以项目手写了暗色样式：

```css
/* 所有选择器都以 .dark 开头，只在暗色模式生效 */
.dark .ce-inline-toolbar,
.dark .ce-popover {
  background: theme('colors.popover.DEFAULT');  /* 引用 Tailwind 配置 */
  color: inherit;
  border-color: theme('colors.border');
}
```

**`theme()` 函数** 是 Tailwind 提供的，让你在 CSS 文件中引用 `tailwind.config.js` 里定义的值。

### 4.5 自定义 Hooks

> 文件路径：`/hooks/use-lock-body.ts` 和 `/hooks/use-mounted.ts`

```typescript
// use-lock-body.ts — 锁定页面滚动
export function useLockBody() {
  React.useLayoutEffect((): (() => void) => {
    const originalStyle = window.getComputedStyle(document.body).overflow
    document.body.style.overflow = "hidden"    // 锁定
    return () => (document.body.style.overflow = originalStyle)  // 恢复
  }, [])
}

// use-mounted.ts — 检测组件是否挂载
export function useMounted() {
  const [mounted, setMounted] = React.useState(false)
  React.useEffect(() => { setMounted(true) }, [])
  return mounted
}
```

**使用场景：**
- `useLockBody`：在 `components/mobile-nav.tsx` 中使用，打开手机导航时阻止背景滚动
- `useMounted`：在 `components/toc.tsx` 中使用，避免 IntersectionObserver 在服务端报错

---

## 五、概念图解：颜色变量的完整链路

```
styles/globals.css                tailwind.config.js              组件代码
┌─────────────────────┐          ┌──────────────────────┐       ┌──────────────┐
│ :root {             │          │ colors: {            │       │ className=   │
│   --primary:        │          │   primary: {         │       │   "bg-primary│
│     222.2 47.4%     │───────→  │     DEFAULT:         │──→    │    text-     │
│     11.2%           │          │       "hsl(var(      │       │    primary-  │
│ }                   │          │       --primary))"   │       │    foreground│
│                     │          │   }                  │       │   "          │
│ .dark {             │          │ }                    │       └──────────────┘
│   --primary:        │          └──────────────────────┘
│     210 40% 98%     │
│ }                   │
└─────────────────────┘
         │
    切换暗色模式
         ↓
  <html class="dark">
         ↓
  --primary 值变化
         ↓
  所有 bg-primary 元素自动更新颜色
```

---

## 六、动手练习

### 练习 1：追踪颜色链路（30 分钟）

**目标：** 完整追踪一个 Tailwind 颜色类从类名到实际像素颜色的全过程。

**步骤：**

1. 打开 `tailwind.config.js`，找到 `destructive` 颜色的定义
2. 记下它引用的 CSS 变量名
3. 打开 `styles/globals.css`，找到这个变量在 `:root`（亮色）和 `.dark`（暗色）中的 HSL 值
4. 使用在线 HSL 颜色工具（搜索 "HSL color picker"），输入这两个值，看看分别是什么颜色
5. 在项目中搜索 `destructive` 类名的使用，找到 2-3 个使用它的组件

**排错提示：**
- HSL 值在 `globals.css` 中没有 `hsl()` 包裹，是"裸值"
- 在 `tailwind.config.js` 中才用 `hsl(var(...))` 包上

### 练习 2：修改主题色（40 分钟）

**目标：** 亲手修改主题色，体验 CSS 变量系统的威力。

**步骤：**

1. 打开 `styles/globals.css`
2. 把 `:root` 中的 `--primary` 改成蓝色：`210 100% 50%`
3. 保存，观察页面中所有使用 `primary` 颜色的元素是否都变了
4. 改回原来的值
5. 尝试改 `--background` 的值，观察整个页面背景的变化

**排错提示：**
- 如果改了之后页面没变化，可能是浏览器缓存。尝试强制刷新（Ctrl+Shift+R）
- 只改 `:root` 部分的值，暗色模式的值（`.dark`）先不改

### 练习 3：阅读自定义 Hook（20 分钟）

**目标：** 理解自定义 Hook 的代码和使用场景。

**步骤：**

1. 阅读 `hooks/use-lock-body.ts`，回答：
   - 这个 Hook 在组件挂载时做了什么？
   - 在组件卸载时做了什么？
   - 为什么用 `useLayoutEffect` 而不是 `useEffect`？

2. 阅读 `hooks/use-mounted.ts`，回答：
   - 初始状态 `mounted` 是什么值？
   - 什么时候变成 `true`？
   - 在 `components/toc.tsx` 中，为什么要等 `mounted` 为 `true` 才渲染内容？

---

## 七、自检问题

### 问题 1：为什么 `globals.css` 中的 CSS 变量值只有数字（如 `222.2 47.4% 11.2%`），没有 `hsl()` 包裹？

<details>
<summary>查看答案</summary>

这是为了配合 Tailwind CSS 的透明度功能。Tailwind 需要能在颜色后面追加透明度值（如 `bg-primary/50` 表示 50% 透明），所以在 `tailwind.config.js` 中用 `hsl(var(--primary))` 包裹。如果变量本身就包含 `hsl()`，就无法在外面再加透明度了。

</details>

### 问题 2：用 CSS 变量做主题切换和用 Tailwind 的 `dark:` 前缀相比，有什么优势？

<details>
<summary>查看答案</summary>

CSS 变量方式只需要写一个类名（`bg-primary`），系统自动适配亮暗模式。`dark:` 前缀方式需要每个元素都写两遍颜色（`bg-white dark:bg-gray-900`），代码更冗长。此外，CSS 变量方式更容易添加新主题（只需新增一套变量定义），也更容易全局修改配色方案（只改变量值）。

</details>

### 问题 3：`useLockBody` 为什么用 `useLayoutEffect` 而不是 `useEffect`？

<details>
<summary>查看答案</summary>

`useLayoutEffect` 在浏览器绘制前执行，`useEffect` 在浏览器绘制后执行。如果用 `useEffect`，用户可能会先看到页面可以滚动，然后突然锁定——出现闪烁。`useLayoutEffect` 确保在用户看到画面之前就锁定滚动，避免视觉跳动。

</details>

### 问题 4：`useMounted` 解决了什么问题？

<details>
<summary>查看答案</summary>

解决了 SSR Hydration 不匹配的问题。Next.js 先在服务器端渲染 HTML，然后在客户端"注水"（hydration）。如果某些内容（如使用 IntersectionObserver 的 TOC 组件）在服务器端和客户端渲染结果不同，React 会报错。`useMounted` 确保这些内容只在客户端渲染，避免不匹配。

</details>

### 问题 5：`styles/editor.css` 中的 `theme('colors.popover.DEFAULT')` 是什么意思？

<details>
<summary>查看答案</summary>

`theme()` 是 Tailwind 提供的 CSS 函数，用于在 CSS 文件中引用 `tailwind.config.js` 中定义的值。`theme('colors.popover.DEFAULT')` 等价于 `tailwind.config.js` 中 `theme.extend.colors.popover.DEFAULT` 的值，即 `hsl(var(--popover))`。这样 EditorJS 的暗色样式就能和项目主题保持一致。

</details>

---

## 八、预计时间分配

| 环节 | 预计时间 | 说明 |
|------|---------|------|
| 前置知识速补 | 25 分钟 | CSS 选择器、CSS 变量、HSL、Hooks |
| 核心概念 + 源码精读 | 55 分钟 | globals.css + tailwind.config.js + hooks |
| 练习 1：追踪颜色链路 | 30 分钟 | 理解变量到颜色的完整路径 |
| 练习 2：修改主题色 | 40 分钟 | 亲手体验 CSS 变量的威力 |
| 练习 3：阅读自定义 Hook | 20 分钟 | 理解 Hook 的设计模式 |
| 自检问题 + 笔记整理 | 20 分钟 | 回顾全天所学 |
| **合计** | **约 3.5 小时** | |

建议休息安排：源码精读后休息一次，练习 2 后再休息一次。

---

## 九、常见踩坑点

### 踩坑 1：修改 CSS 变量后页面没变化

**原因：** 浏览器缓存了旧的 CSS。

**解决：** 强制刷新（Ctrl+Shift+R 或 Cmd+Shift+R）。如果还不行，停掉 `pnpm dev` 重新启动。

### 踩坑 2：不理解 HSL 裸值

**症状：** 看到 `222.2 47.4% 11.2%` 不知道这是什么颜色。

**解决：** 用在线 HSL 颜色工具。记住 HSL 三个值：色相（0-360°）、饱和度（0-100%）、亮度（0-100%）。亮度接近 0% 是黑色，接近 100% 是白色。

### 踩坑 3：useLayoutEffect 在服务器端报警告

**症状：** 控制台出现 "useLayoutEffect does nothing on the server" 警告。

**原因：** `useLayoutEffect` 需要 DOM，服务器端没有 DOM。

**解决：** 这个警告通常无害。如果想消除它，可以在使用 `useLockBody` 的组件上确保它是 Client Component（`"use client"`）。

### 踩坑 4：`theme()` 函数在普通 CSS 中无效

**注意：** `theme('...')` 只能在 Tailwind CSS 处理的文件中使用（通常是 PostCSS 处理的 `.css` 文件）。如果你在行内样式或 JS 中写 `theme()`，不会生效。

---

## 十、延伸阅读

1. **CSS 变量详解**：https://developer.mozilla.org/zh-CN/docs/Web/CSS/Using_CSS_custom_properties
2. **HSL 颜色模型**：https://developer.mozilla.org/zh-CN/docs/Web/CSS/color_value/hsl
3. **Tailwind CSS 主题配置**：https://tailwindcss.com/docs/theme
4. **next/font 文档**：https://nextjs.org/docs/app/building-your-application/optimizing/fonts
5. **React useLayoutEffect**：https://react.dev/reference/react/useLayoutEffect
6. **shadcn/ui 主题系统**：https://ui.shadcn.com/docs/theming — 本项目的主题方案就来自 shadcn/ui
