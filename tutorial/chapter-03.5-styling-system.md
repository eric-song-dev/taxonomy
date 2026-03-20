# 第3.5章：样式系统深入 — CSS 变量、字体和自定义 Hooks

## 学习目标

- 理解 CSS 变量主题系统的工作原理（亮色/暗色模式）
- 理解 Tailwind CSS 如何与 CSS 变量配合
- 理解 `next/font` 的两种字体加载方式
- 理解专用样式文件的作用（editor.css、mdx.css）
- 理解自定义 Hooks 的设计模式

## 涉及文件

| 文件 | 说明 |
|------|------|
| `styles/globals.css` | 全局样式 + CSS 变量定义 |
| `styles/editor.css` | EditorJS 编辑器暗色模式样式 |
| `styles/mdx.css` | MDX 代码高亮样式 |
| `tailwind.config.js` | Tailwind 颜色和字体映射 |
| `app/layout.tsx` | 字体加载和注入 |
| `hooks/use-lock-body.ts` | 锁定页面滚动的 Hook |
| `hooks/use-mounted.ts` | 组件挂载检测的 Hook |

---

## 1. CSS 变量主题系统

### 1.1 什么是 CSS 变量？

CSS 变量（也叫 CSS 自定义属性）允许你在一个地方定义颜色值，然后在整个项目中复用：

```css
/* 定义变量 */
:root {
  --my-color: blue;
}

/* 使用变量 */
.button {
  background-color: var(--my-color);  /* 结果是 blue */
}
```

**好处：** 如果你想把所有蓝色换成绿色，只要改一处 `--my-color` 的值就行，不用找遍所有文件。

### 1.2 `styles/globals.css` — 主题变量定义

Taxonomy 的主题系统核心就在这个文件里。它定义了两套 CSS 变量——亮色模式和暗色模式：

```css
/* styles/globals.css */
@tailwind base;
@tailwind components;
@tailwind utilities;

@layer base {
  /* 亮色模式（默认） */
  :root {
    --background: 0 0% 100%;           /* 白色 */
    --foreground: 222.2 47.4% 11.2%;   /* 深灰色（文字） */

    --muted: 210 40% 96.1%;            /* 柔和背景 */
    --muted-foreground: 215.4 16.3% 46.9%;

    --primary: 222.2 47.4% 11.2%;      /* 主色调 */
    --primary-foreground: 210 40% 98%;

    --secondary: 210 40% 96.1%;        /* 次色调 */
    --secondary-foreground: 222.2 47.4% 11.2%;

    --destructive: 0 100% 50%;         /* 危险色（红色） */
    --destructive-foreground: 210 40% 98%;

    --border: 214.3 31.8% 91.4%;       /* 边框颜色 */
    --input: 214.3 31.8% 91.4%;        /* 输入框边框 */
    --ring: 215 20.2% 65.1%;           /* 焦点环颜色 */

    --radius: 0.5rem;                   /* 圆角大小 */
  }

  /* 暗色模式 */
  .dark {
    --background: 224 71% 4%;           /* 深色背景 */
    --foreground: 213 31% 91%;          /* 浅色文字 */

    --primary: 210 40% 98%;             /* 注意：主色调反过来了 */
    --primary-foreground: 222.2 47.4% 1.2%;

    --border: 216 34% 17%;              /* 更暗的边框 */
    /* ... 其他变量也都换成暗色系 */
  }
}
```

### 为什么变量值只有数字，没有 `hsl(...)`？

你可能注意到变量值是 `222.2 47.4% 11.2%` 而不是 `hsl(222.2, 47.4%, 11.2%)`。这是为了配合 Tailwind CSS 使用。Tailwind 需要能添加透明度，所以用 HSL 的"裸值"，在实际使用时再包上 `hsl()`：

```css
/* Tailwind 内部生成的 CSS 类似这样 */
.bg-primary {
  background-color: hsl(var(--primary));   /* hsl(222.2 47.4% 11.2%) */
}
.bg-primary/50 {
  background-color: hsl(var(--primary) / 0.5);  /* 50% 透明度 */
}
```

### 1.3 全局基础样式

文件底部还有一段全局样式：

```css
@layer base {
  * {
    @apply border-border;       /* 所有元素的默认边框颜色 */
  }
  body {
    @apply bg-background text-foreground;  /* body 的默认背景和文字颜色 */
    font-feature-settings: "rlig" 1, "calt" 1;  /* 字体连字特性 */
  }
}
```

**`@apply`** 是 Tailwind 的语法，在 CSS 文件中使用 Tailwind 的工具类。`bg-background` 就会解析为 `background-color: hsl(var(--background))`。

### 1.4 暗色模式切换原理

当用户切换暗色模式时：
1. `next-themes` 库在 `<html>` 标签上添加 `class="dark"`
2. CSS 选择器 `.dark` 生效，所有 CSS 变量被覆盖为暗色值
3. 所有使用这些变量的元素自动更新颜色

```
切换暗色模式
    ↓
<html class="dark">
    ↓
.dark { --background: 224 71% 4%; }  ← 覆盖 :root 中的值
    ↓
bg-background → hsl(224 71% 4%)     ← 自动变暗
```

---

## 2. Tailwind CSS 与 CSS 变量的桥梁

### 2.1 `tailwind.config.js` 颜色配置

`tailwind.config.js` 是连接 CSS 变量和 Tailwind 类名的"桥梁"：

```javascript
// tailwind.config.js（颜色部分）
module.exports = {
  darkMode: ["class"],  // 暗色模式通过 class 控制（不是 media query）
  theme: {
    extend: {
      colors: {
        // 每个颜色都映射到 CSS 变量
        border: "hsl(var(--border))",
        input: "hsl(var(--input))",
        ring: "hsl(var(--ring))",
        background: "hsl(var(--background))",
        foreground: "hsl(var(--foreground))",

        // 带子颜色的（DEFAULT + foreground）
        primary: {
          DEFAULT: "hsl(var(--primary))",
          foreground: "hsl(var(--primary-foreground))",
        },
        secondary: {
          DEFAULT: "hsl(var(--secondary))",
          foreground: "hsl(var(--secondary-foreground))",
        },
        destructive: {
          DEFAULT: "hsl(var(--destructive))",
          foreground: "hsl(var(--destructive-foreground))",
        },
        muted: {
          DEFAULT: "hsl(var(--muted))",
          foreground: "hsl(var(--muted-foreground))",
        },
        // ... accent, popover, card 同理
      },
      borderRadius: {
        lg: "var(--radius)",
        md: "calc(var(--radius) - 2px)",
        sm: "calc(var(--radius) - 4px)",
      },
    },
  },
}
```

**效果：** 在代码中写 `className="bg-primary text-primary-foreground"` 就能自动适配亮色/暗色模式，**不需要写 `dark:` 前缀**。

### 2.2 为什么不直接用 `dark:` 前缀？

传统做法：

```tsx
// 需要写两遍颜色
<div className="bg-white text-gray-900 dark:bg-gray-900 dark:text-white">
```

CSS 变量做法：

```tsx
// 只写一遍，自动适配
<div className="bg-background text-foreground">
```

好处：
- 代码更简洁（不用每个元素都写 `dark:` 前缀）
- 颜色集中管理（改一个变量，整站都变）
- 容易添加新主题（只要新增一套 CSS 变量）

---

## 3. 字体配置

### 3.1 两种字体加载方式

Taxonomy 使用了两种不同的字体加载方式，都来自 `next/font`：

```typescript
// app/layout.tsx

// 方式一：Google Fonts（网络字体）
import { Inter as FontSans } from "next/font/google"

const fontSans = FontSans({
  subsets: ["latin"],           // 只加载拉丁字符子集（减小体积）
  variable: "--font-sans",      // 注入 CSS 变量名
})

// 方式二：本地字体文件
import localFont from "next/font/local"

const fontHeading = localFont({
  src: "../assets/fonts/CalSans-SemiBold.woff2",  // 字体文件路径
  variable: "--font-heading",                       // 注入 CSS 变量名
})
```

**为什么用 `next/font`？**
- 自动优化：字体在构建时下载，不会阻塞页面加载
- 无闪烁：避免 FOUT（Flash of Unstyled Text）
- 自托管：Google Fonts 被下载到你的服务器，不依赖 Google CDN

### 3.2 CSS 变量注入

字体加载后，通过 `variable` 属性注入为 CSS 变量：

```tsx
// app/layout.tsx
export default function RootLayout({ children }) {
  return (
    <html lang="en" className={cn(fontSans.variable, fontHeading.variable)}>
      <body>{children}</body>
    </html>
  )
}
```

这会在 `<html>` 标签上生成类似这样的 CSS：

```css
.--font-sans-xxxxx {
  --font-sans: 'Inter', sans-serif;
}
.--font-heading-xxxxx {
  --font-heading: 'Cal Sans', sans-serif;
}
```

### 3.3 Tailwind 字体映射

最后在 `tailwind.config.js` 中把 CSS 变量映射为 Tailwind 类名：

```javascript
// tailwind.config.js（字体部分）
const { fontFamily } = require("tailwindcss/defaultTheme")

module.exports = {
  theme: {
    extend: {
      fontFamily: {
        sans: ["var(--font-sans)", ...fontFamily.sans],      // font-sans
        heading: ["var(--font-heading)", ...fontFamily.sans], // font-heading
      },
    },
  },
}
```

使用效果：

```tsx
<h1 className="font-heading">标题用 Cal Sans 字体</h1>
<p className="font-sans">正文用 Inter 字体</p>
```

### 3.4 完整流程

```
Inter (Google Fonts)          CalSans (本地文件)
    ↓                               ↓
next/font/google              next/font/local
    ↓                               ↓
variable: "--font-sans"       variable: "--font-heading"
    ↓                               ↓
<html class="...">  ←  CSS 变量注入到页面根元素
    ↓                               ↓
tailwind.config.js 映射
    ↓                               ↓
font-sans 类名                font-heading 类名
```

---

## 4. 专用样式文件

### 4.1 `styles/editor.css` — EditorJS 暗色模式

EditorJS 是第三方富文本编辑器，它自带的样式不支持暗色模式。所以项目写了专门的 CSS 来覆盖：

```css
/* styles/editor.css — 给 EditorJS 的暗色模式样式 */

/* 编辑器工具栏和弹出层的背景色 */
.dark .ce-inline-toolbar,
.dark .ce-toolbar__settings-btn,
.dark .ce-popover,
.dark .ce-settings {
  background: theme('colors.popover.DEFAULT');  /* 使用 Tailwind 主题色 */
  color: inherit;
  border-color: theme('colors.border');
}

/* 工具栏按钮的 hover 效果 */
.dark .ce-toolbox__button:hover,
.dark .ce-inline-tool:hover,
.dark .ce-popover__item:hover {
  background-color: theme('colors.accent.DEFAULT');
  color: theme('colors.accent.foreground');
}
```

**关键技巧：** 使用 `theme('...')` 函数引用 `tailwind.config.js` 中定义的颜色值。这样 EditorJS 的暗色模式颜色就和项目主题完全一致。

### 4.2 `styles/mdx.css` — 代码高亮样式

MDX 内容中的代码块使用 `rehype-pretty-code` 插件进行语法高亮。这个文件为高亮后的代码块定义样式：

```css
/* styles/mdx.css */

/* 代码块的基础样式 */
[data-rehype-pretty-code-fragment] code {
  @apply grid min-w-full break-words rounded-none border-0 bg-transparent p-0 text-sm text-black;
  counter-reset: line;    /* 用于行号计数 */
}

/* 每行代码的内边距 */
[data-rehype-pretty-code-fragment] .line {
  @apply px-4 py-1;
}

/* 行号显示 */
[data-rehype-pretty-code-fragment] [data-line-numbers] > .line::before {
  counter-increment: line;
  content: counter(line);           /* CSS 计数器自动生成行号 */
  display: inline-block;
  width: 1rem;
  margin-right: 1rem;
  text-align: right;
  color: gray;
}

/* 高亮行的背景色 */
[data-rehype-pretty-code-fragment] .line--highlighted {
  @apply bg-slate-300 bg-opacity-10;
}
```

**`[data-rehype-pretty-code-fragment]`** 是属性选择器，只匹配由 `rehype-pretty-code` 生成的代码块，不会影响其他元素。

---

## 5. 自定义 Hooks

Taxonomy 项目有两个自定义 React Hook，它们体现了常见的设计模式。

### 5.1 `hooks/use-lock-body.ts` — 锁定页面滚动

当手机端的导航菜单打开时，需要阻止背景页面滚动：

```typescript
// hooks/use-lock-body.ts
import * as React from "react"

export function useLockBody() {
  React.useLayoutEffect((): (() => void) => {
    // 保存当前的 overflow 样式
    const originalStyle: string = window.getComputedStyle(
      document.body
    ).overflow

    // 锁定滚动
    document.body.style.overflow = "hidden"

    // 组件卸载时恢复原始样式
    return () => (document.body.style.overflow = originalStyle)
  }, [])
}
```

**使用方式：**

```tsx
function MobileNav() {
  useLockBody()  // 组件挂载时锁定，卸载时自动恢复

  return <nav>...</nav>
}
```

**关键点：**
- 使用 `useLayoutEffect` 而不是 `useEffect`，因为样式变更需要在浏览器绘制前生效，避免闪烁
- 返回清理函数恢复原始样式——这是 React Hook 的最佳实践

### 5.2 `hooks/use-mounted.ts` — 解决 SSR Hydration 问题

```typescript
// hooks/use-mounted.ts
import * as React from "react"

export function useMounted() {
  const [mounted, setMounted] = React.useState(false)

  React.useEffect(() => {
    setMounted(true)
  }, [])

  return mounted
}
```

**为什么需要这个？**

Next.js 使用 Server-Side Rendering (SSR)。有些内容在服务器和客户端渲染时可能不一致（比如主题颜色、浏览器 API 等）。如果不一致，React 会报 hydration 错误。

**解决方案：** 在组件挂载前不渲染可能不一致的内容。

```tsx
// 示例：components/toc.tsx 中使用了 useMounted
function DashboardTableOfContents({ toc }) {
  const mounted = useMounted()

  // 服务器渲染时 mounted 是 false，返回 null
  // 客户端 hydration 后 mounted 变成 true，正常渲染
  return mounted ? (
    <div>
      <p>On This Page</p>
      <Tree tree={toc} />
    </div>
  ) : null
}
```

### 5.3 自定义 Hook 的设计模式

从这两个 Hook 中可以总结出自定义 Hook 的设计原则：

| 原则 | 说明 | 示例 |
|------|------|------|
| 以 `use` 开头 | React 的规则，不能违反 | `useLockBody`, `useMounted` |
| 做一件事 | 每个 Hook 只解决一个问题 | 锁滚动 / 检测挂载 |
| 返回状态或执行副作用 | Hook 要么返回有用的值，要么做有意义的事 | `useMounted` 返回布尔值 |
| 包含清理逻辑 | 使用 `useEffect` 的返回值做清理 | `useLockBody` 恢复样式 |

---

## 小练习

### 练习 1：理解主题切换

1. 打开浏览器开发者工具（F12）
2. 在 Elements 面板找到 `<html>` 标签
3. 手动给 `<html>` 添加 `class="dark"`，观察页面颜色变化
4. 删除 `dark` 类名，观察页面恢复亮色
5. 打开 `styles/globals.css`，找到 `--primary` 在亮色和暗色模式下的不同值

### 练习 2：追踪一个颜色

追踪 `bg-primary` 这个类名的完整链路：
1. 在 `tailwind.config.js` 中找到 `primary` 颜色的定义
2. 找到它引用的 CSS 变量名
3. 在 `styles/globals.css` 中找到这个变量在 `:root` 和 `.dark` 中的值
4. 用 HSL 颜色工具确认这两个值分别是什么颜色

### 练习 3：理解自定义 Hook

1. 阅读 `hooks/use-lock-body.ts`
2. 思考：如果用 `useEffect` 替代 `useLayoutEffect`，会有什么视觉差异？
3. 阅读 `hooks/use-mounted.ts`
4. 在 `components/toc.tsx` 中找到 `useMounted` 的使用，理解为什么在 SSR 场景下需要它

---

## 本章小结

- CSS 变量定义了两套主题色（亮色/暗色），通过 `.dark` 类名切换
- `tailwind.config.js` 将 CSS 变量映射为 Tailwind 类名，实现自动主题适配
- `next/font` 提供 Google Fonts 和本地字体两种加载方式，通过 CSS 变量注入
- 专用样式文件为 EditorJS 和 MDX 代码高亮定制暗色模式样式
- 自定义 Hooks 封装了可复用的逻辑：`useLockBody` 锁滚动，`useMounted` 解决 hydration

下一章我们将学习布局与导航——页面是怎么拼起来的。
