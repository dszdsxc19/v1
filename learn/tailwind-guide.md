# Tailwind CSS 使用指南

## 1. tailwind.config.ts 的作用

直接引入 Tailwind 已经提供了几百个内置工具类（`flex`、`text-lg`、`bg-blue-500` 等），但当需要**项目专属的自定义样式**时，就需要在 `tailwind.config.ts` 里扩展主题。

### `theme.extend` vs `theme`

| 写法 | 效果 |
|---|---|
| `theme.extend.xxx` | **追加**到 Tailwind 默认主题，保留所有内置工具类 |
| `theme.xxx`（不加 extend）| **替换**掉 Tailwind 对应的所有内置样式 |

通常应该用 `extend`，避免丢失内置功能。

---

## 2. 自定义 animation 和 keyframes

`animation` 和 `keyframes` 配合使用，用来定义自定义 CSS 动画，注册后即可作为 Tailwind class 使用。

```ts
// tailwind.config.ts
theme: {
  extend: {
    animation: {
      // key 名 → 对应 class: animate-marquee
      marquee: "marquee 25s linear infinite",
      marquee2: "marquee2 25s linear infinite",
    },
    keyframes: {
      marquee: {
        "0%":   { transform: "translateX(0%)" },
        "100%": { transform: "translateX(-100%)" },
      },
      marquee2: {
        "0%":   { transform: "translateX(100%)" },
        "100%": { transform: "translateX(0%)" },
      },
    },
  },
},
```

配置后在组件中直接使用 `animate-marquee` / `animate-marquee2` class 即可。

### 跑马灯原理

跑马灯需要**两段内容**首尾衔接，才能实现无缝循环：

- **marquee（第一段）**：从原位 `translateX(0%)` → `translateX(-100%)`，向左滚出屏幕
- **marquee2（第二段）**：从右侧 `translateX(100%)` → `translateX(0%)`，从右补进屏幕

```tsx
<div className="relative flex overflow-x-hidden">
  {/* 第一段：正常文档流，向左滚出 */}
  <div className="animate-marquee whitespace-nowrap lg:animate-none">
    {/* Logo 内容 */}
  </div>

  {/* 第二段：absolute 定位，从右侧补进来；大屏幕隐藏 */}
  <div className="animate-marquee2 absolute top-0 lg:hidden">
    {/* 同样的 Logo 内容 */}
  </div>
</div>
```

两段动画时长相同（`25s`），一段滚出的同时另一段正好补进，形成视觉上无缝的循环。

> 大屏幕（`lg` 以上）时所有 logo 能一行展示，无需跑马灯，所以第一段设 `lg:animate-none` 停止动画，第二段 `lg:hidden` 直接隐藏。

---

## 3. Tailwind 编译原理

Tailwind **不是运行时替换 class**，而是在**构建时生成 CSS 文件**。

### 编译流程

```
1. 扫描源码
   tailwind.config.ts 的 content 字段指定扫描范围：
   content: ["./src/**/*.{ts,tsx}", "../../packages/ui/src/**/*.{ts,tsx}"]
   → 用正则找出所有完整的 class 字面量

2. 生成 CSS（Tree-shaking）
   只为用到的 class 生成对应 CSS，未使用的全部丢弃
   .flex { display: flex; }
   .animate-marquee { animation: marquee 25s linear infinite; }
   @keyframes marquee { ... }

3. 注入页面
   globals.css 中的三行指令被替换为实际 CSS：
   @tailwind base;       /* 重置样式 */
   @tailwind components; /* 组件样式 */
   @tailwind utilities;  /* 所有用到的工具类 */
```

### ⚠️ 不能动态拼接 class 名

Tailwind 扫描是**纯文本匹配**，不执行 JS，因此完整 class 名必须出现在源码中：

```tsx
// ❌ 错误：Tailwind 扫描到的是 "text-${color}-500"，无法识别，不生成 CSS
const color = "blue"
<div className={`text-${color}-500`}>

// ✅ 正确：完整 class 名出现在源码中
<div className="text-blue-500">

// ✅ 正确：条件选择完整 class 名
<div className={isBlue ? "text-blue-500" : "text-red-500"}>
```

---

## 4. `cn` 工具函数

`cn` 是**运行时**的 class 合并工具，与 Tailwind 编译无关。

```ts
// packages/ui/src/utils/cn.ts
import { clsx } from "clsx"
import { twMerge } from "tailwind-merge"

export function cn(...inputs) {
  return twMerge(clsx(inputs))
}
```

- **`clsx`**：根据条件合并多个 class 字符串
- **`tailwind-merge`**：解决 Tailwind class 冲突（如 `p-2` 和 `p-4` 同时存在，后者覆盖前者）

```tsx
// 典型用法：条件 class + 冲突解决
<div className={cn(
  "flex items-center p-4",   // 基础 class（写死在源码，Tailwind 能扫描到）
  isActive && "bg-blue-500", // 条件 class（完整 class 名，Tailwind 能扫描到）
  className                  // 外部传入，tailwind-merge 自动解决冲突
)}>
```

### `cn` 为何不影响 Tailwind 扫描

`cn` 内传入的字符串仍然是**完整的 class 字面量**，Tailwind 扫描时能找到它们。动态拼接之所以失败，是因为完整 class 名根本不存在于源码中。
