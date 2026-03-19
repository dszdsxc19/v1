# Next.js 中 Analytics 组件的接入原理

## 背景：为什么用 `<AnalyticsProvider />` 而不是纯函数调用？

在 Next.js App Router 中，`layout.tsx` 默认是 **Server Component**，不能使用 `useEffect` 或 `document.createElement`。

因此，需要客户端副作用（如初始化第三方 SDK）时，标准做法是封装成一个 **Client Component**，插入到 Server Component 的渲染树中：

```
RootLayout (Server Component)
  └── <AnalyticsProvider />   ← 'use client' 组件，负责注入脚本
```

---

## `OpenPanelComponent` 实际渲染了什么？

它渲染了**两个真实的 `<script>` 标签**（通过 Next.js `<Script>`）：

```tsx
return (
  <>
    {/* 1. 加载远程 SDK */}
    <Script src="https://openpanel.dev/op1.js" async defer />

    {/* 2. 内联初始化脚本 */}
    <Script dangerouslySetInnerHTML={{ __html: `
      window.op = window.op || function(...args) {
        (window.op.q = window.op.q || []).push(args)
      };
      window.op('init', { clientId: "...", trackScreenViews: true });
    `}} />
  </>
);
```

---

## 关键设计：`window.op` 队列模式

```js
window.op = window.op || function(...args) {
  (window.op.q = window.op.q || []).push(args)  // SDK 未就绪时，先入队
};
window.op('init', { ... });
```

**原理**：SDK 脚本异步加载，加载完成前所有 `window.op(...)` 调用会缓存进队列；SDK ready 后自动回放队列。这是 Google Analytics、Segment 等主流 SDK 的通用模式。

---

## 为什么用 Next.js `<Script>` 而不直接写 `<script>`？

| 能力 | 原生 `<script>` | Next.js `<Script>` |
|---|---|---|
| 加载时机控制 | ❌ 浏览器自行决定 | ✅ `strategy` 参数精细控制 |
| 脚本去重 | ❌ 需手动管理 | ✅ 同 `src` 自动只注入一次 |
| SSR / Hydration 兼容 | ❌ 难以控制时序 | ✅ 参与 React 渲染流程 |
| App Router 支持 | ❌ SC 中无法使用 DOM API | ✅ 官方推荐方式 |

### `strategy` 选项

```tsx
<Script strategy="afterInteractive" />  // DOMContentLoaded 后（默认，适合分析）
<Script strategy="lazyOnload" />        // 页面完全空闲后（适合非关键脚本）
<Script strategy="beforeInteractive" /> // 页面渲染前，阻塞（适合关键脚本）
```

---

## 纯 React（Vite/CRA）中有区别吗？

在纯 React 中，这套组件封装**基本没必要**，原因：

1. **React JSX 里的 `<script>` 不会执行**（安全机制，浏览器不触发动态插入的脚本）
2. 可以直接在 `public/index.html` 里写原生 `<script>`，最简单干净
3. 动态加载用 `useEffect` + `document.createElement('script')` 即可

```html
<!-- public/index.html：纯 React 中最推荐的方式 -->
<script src="https://openpanel.dev/op1.js" async defer></script>
<script>
  window.op = window.op || function(...args) { (window.op.q = []).push(args) };
  window.op('init', { clientId: '...' });
</script>
```

| | 纯 React（Vite/CRA） | Next.js App Router |
|---|---|---|
| 推荐方式 | `index.html` 直接写 | `<Script>` 组件封装 |
| 需要组件形式？ | ❌ 通常不需要 | ✅ 是唯一合规方式 |
| 原因 | 有 HTML 模板可控制 | SC 中不能用 DOM API / `useEffect` |

> **结论**：`<OpenPanelComponent />` 这套封装本质上是在解决 **Next.js App Router** 的约束，在纯 React 项目中直接写 `index.html` 反而更干净。

---

## 什么时候"纯逻辑"不该用组件形式？

如果副作用**不产生任何 DOM/Context 输出**，用组件包装是反模式：

```tsx
// ❌ 反模式：组件只有副作用，return null
const TrackingSetup = () => {
  useEffect(() => { setupMyTracker(); }, []);
  return null;
};

// ✅ 正确：直接在模块顶层初始化，或用 hook
setupMyTracker();
```

> **判断标准**：组件是否输出 React element（包括不可见的 `<Script>`、`<Context.Provider>` 等）？如果是 → 组件形式合理；如果完全没有任何 React element 产出 → 不要用组件。
