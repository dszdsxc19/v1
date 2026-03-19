# Monorepo 内部包机制与应用内编译 (Transpilation-in-App)

本文总结了在 Monorepo 架构中，本地包（Internal Packages）是如何工作的，以及为什么我们不需要 `dist` 目录。

## 1. 为什么没有 `dist` 目录？

在传统的 npm 包开发中，我们习惯先编译 (`tsc`/`rollup`) 生成 `dist` 目录，然后再发布。但在我们的 Monorepo (通常基于 Turborepo/pnpm workspaces) 中，你会发现 `packages/ui` 等内部包直接通过 `package.json` 的 `exports` 字段指向了 **源码文件** (`.ts`, `.tsx`)。

```json
// packages/ui/package.json
{
  "name": "@v1/ui",
  "private": true,  // 标记为私有包，不会发布到 npm
  "exports": {
    ".": "./src/index.ts",
    "./cn": "./src/utils/cn.ts",        // 直接指向 .ts 源码
    "./button": "./src/components/button.tsx" // 直接指向 .tsx 源码
  }
}
```

这意味着这些包**不需要单独的构建步骤**。它们本身就是源码集合。

## 2. 什么是 Transpilation-in-App (应用内编译)？

既然包本身不编译，那么谁来编译它们？答案是：**使用这些包的应用（App）**。

例如，当 `apps/app` (Next.js) 引用 `@v1/ui/button` 时，Next.js 的构建系统（Webpack/TurboPack）会像处理它自己的源码一样，直接读取 `packages/ui/src/components/button.tsx` 并进行编译/转译。

这种模式被称为 **Transpilation-in-App**。

### 核心优势
*   **⚡️ 极速开发体验**：修改 UI 包代码后，应用端几乎瞬间热更新 (HMR)，无需等待中间包的构建流程。
*   **🛠 源码级调试**：在浏览器或编辑器中打断点时，直接定位到原始 TypeScript 源码，而不是混淆后的 `dist` 代码。
*   **📦 统一构建配置**：所有代码（App 和 Packages）共享同一套构建配置（如 Polyfills、目标环境），避免重复和不一致。
*   **🚫 消除发布包的冗余**：对于私有包 (`private: true`)，既然只有我们需要用，何必浪费时间生成 CommonJS/ESM 双重产物呢？

## 3. `exports` vs `paths`：谁在起作用？

这也是开发中常混淆的两个概念，它们分工明确但必须配合使用。

### 3.1 `package.json` 中的 `exports` (运行时/构建时)
这是 **Node.js 官方标准**，也是 Webpack/Vite 等现代构建工具遵循的规范。
*   **作用**：定义包的真实入口。
*   **场景**：当你 `import { Button } from "@v1/ui/button"` 时，构建工具查阅此字段找到物理文件 `./src/components/button.tsx`。
*   **关键点**：**必须配置**，否则运行时找不到文件，或者甚至无法 resolve 模块。

### 3.2 `tsconfig.json` 中的 `paths` (类型检查/IDE)
这是 **TypeScript 特有** 的配置。
*   **作用**：告诉 TS 编译器和 IDE 如何解析模块路径（路径映射）。
*   **场景**：
    *   让 VS Code 能正确跳转定义和补全（Ctrl+Click 能跳到源文件）。
    *   让 `tsc` 在检查类型时不报错。
*   **局限性**：它**只影响类型检查**，不会改变编译后的 JS 代码。如果你只配了 `paths` 没配 `exports`，TS 可能通过检查（因为它被骗过了），但代码运行起来会崩（Node/Webpack 找不到模块）。

### 总结对照表

| 特性 | `内部包 (Internal Package)` | `外部包 (External Package / node_modules)` |
| :--- | :--- | :--- |
| **引用目标** | **源码** (`.ts` / `.tsx`) | **产物** (`dist/index.js` + `.d.ts`) |
| **构建者** | **使用的主应用** (App's Bundler) | **包作者** (Package Author's Bundler) |
| **构建时机** | **运行时 (Next.js Build/Dev)** | **发布前 (Pre-publish)** |
| **调试体验** | 源码直出，无 SourceMap 问题 | 依赖 SourceMap，通常被压缩混淆 |
| **管理方式** | Workspace Symlink (软链) | `npm install` 下载 |
