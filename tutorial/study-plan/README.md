# 十三天学习计划：Taxonomy 全栈项目精读

> 基于 [Taxonomy](https://github.com/shadcn/taxonomy) 开源项目的系统性学习路线，从零到部署，每天 3-4 小时。

## 本计划适合谁

- **JavaScript / TypeScript 零基础**的学员 — 每天的学习内容都包含"前置知识速补"，从 `const/let`、箭头函数讲起，不会跳过任何基础概念
- 可能有其他语言（Python、Java、C++ 等）经验，但**第一次接触前端开发**
- 想通过一个**真实的全栈项目**来学习，而不是看孤立的语法教程
- 希望理解"为什么这样写"，而不仅仅是"这是什么"

> 如果你已经熟悉 JavaScript 和 React，可以跳过每天的"前置知识速补"部分，直接进入源码精读和动手练习。

## 路线总览


| 天数                                        | 主题         | 对应章节         | 难度   | 关键词                                    |
| ----------------------------------------- | ---------- | ------------ | ---- | -------------------------------------- |
| [Day 01](day-01-project-overview.md)      | 项目全景       | Chapter 1    | ⭐    | 项目结构、依赖、配置                             |
| [Day 02](day-02-nextjs-basics.md)         | Next.js 基础 | Chapter 2    | ⭐    | App Router、路由、Server/Client Components |
| [Day 03](day-03-ui-components.md)         | UI 组件层     | Chapter 3    | ⭐⭐   | Tailwind、shadcn/ui、组件模式                |
| [Day 03.5](day-03.5-styling-system.md)    | 样式系统深入     | Chapter 3.5  | ⭐⭐   | CSS 变量主题、字体配置、自定义 Hooks               |
| [Day 04](day-04-layout-navigation.md)     | 布局与导航      | Chapter 4    | ⭐⭐   | 嵌套布局、响应式、暗色模式                          |
| [Day 05](day-05-content-system.md)        | 内容系统       | Chapter 5    | ⭐⭐   | MDX、Contentlayer、自定义组件                 |
| [Day 06](day-06-database.md)              | 数据库        | Chapter 6    | ⭐⭐⭐  | Prisma、数据建模、CRUD                       |
| [Day 07](day-07-authentication.md)        | 认证系统       | Chapter 7    | ⭐⭐⭐  | NextAuth、OAuth、JWT、中间件                 |
| [Day 08](day-08-api-layer.md)             | API 层      | Chapter 8    | ⭐⭐⭐  | Route Handlers、Zod 校验、错误处理             |
| [Day 08.5](day-08.5-post-editor.md)      | 文章编辑器      | Chapter 8.5  | ⭐⭐⭐  | EditorJS、动态导入、文章生命周期                   |
| [Day 09](day-09-payment-system.md)        | 支付系统       | Chapter 9    | ⭐⭐⭐⭐ | Stripe、Webhook、订阅管理                    |
| [Day 10](day-10-deployment.md)            | 部署与工程化     | Chapter 10   | ⭐⭐⭐  | 环境变量、Vercel、代码质量、SEO                   |
| [Day 11](day-11-capstone.md)              | 综合实战       | 跨章节综合        | ⭐⭐⭐⭐ | 全流程追踪、功能设计、架构理解                        |


## 环境准备

开始学习前，请确保完成以下准备：

- 安装 [Node.js](https://nodejs.org/) >= 18
- 安装 [pnpm](https://pnpm.io/)（项目使用的包管理器）
- 安装 [VS Code](https://code.visualstudio.com/) 及推荐插件：
  - Tailwind CSS IntelliSense
  - Prisma
  - ESLint
  - Prettier
- 克隆 Taxonomy 仓库并安装依赖：
  ```bash
  git clone https://github.com/shadcn/taxonomy.git
  cd taxonomy
  pnpm install
  ```
- 复制环境变量文件：
  ```bash
  cp .env.example .env
  ```
- 注册相关服务账号（按学习进度逐步注册即可）：
  - GitHub OAuth App（Day 07 需要）
  - PlanetScale 数据库（Day 06 需要）
  - Stripe 测试账号（Day 09 需要）

## 学习建议

1. **先补基础，再看源码**：每天先完成"前置知识速补"部分，确保基础概念清楚后再进入源码精读。
2. **动手优先**：每天的练习务必亲手完成，不要只看不练。练习之间有连续性，串成一条主线。
3. **写学习笔记**：用自己的话复述核心概念，加深理解。
4. **不要跳章**：章节之间有依赖关系，按顺序学习效果最好。
5. **遇到问题先思考**：自检问题先独立思考，再对照答案。每天末尾的"常见踩坑点"可以在遇到问题时查阅。
6. **善用概念图解**：每天都有 ASCII 架构图/流程图，帮助你建立全局视角。

## 进度 Checklist

- Day 01 — 项目全景
- Day 02 — Next.js 基础
- Day 03 — UI 组件层
- Day 03.5 — 样式系统深入
- Day 04 — 布局与导航
- Day 05 — 内容系统
- Day 06 — 数据库
- Day 07 — 认证系统
- Day 08 — API 层
- Day 08.5 — 文章编辑器
- Day 09 — 支付系统
- Day 10 — 部署与工程化
- Day 11 — 综合实战

