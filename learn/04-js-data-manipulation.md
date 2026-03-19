# JavaScript / TypeScript 数据操作核心方法

> 基于项目实际使用场景整理，结合 `posts.server.tsx` 等代码示例。

---

## 一、Array 数组方法

### 增删元素（⚠️ 以下方法均原地修改数组）

#### `.push()` / `.pop()` — 尾部增/删

```ts
const arr = [1, 2, 3];
arr.push(4)    // arr → [1, 2, 3, 4]，返回新长度 4
arr.pop()      // arr → [1, 2, 3]，返回被删除的元素 4
```

#### `.unshift()` / `.shift()` — 头部增/删

```ts
const arr = [1, 2, 3];
arr.unshift(0) // arr → [0, 1, 2, 3]，返回新长度 4
arr.shift()    // arr → [1, 2, 3]，返回被删除的元素 0
```

#### `.splice()` — 任意位置增删改（最灵活，⚠️ 原地修改）

```ts
// splice(起始index, 删除数量, ...插入的元素)
const arr = [1, 2, 3, 4, 5];

arr.splice(1, 2)        // 从index 1 删除 2 个 → arr: [1, 4, 5]，返回 [2, 3]
arr.splice(1, 0, 99)    // 从index 1 插入 99，不删除 → arr: [1, 99, 4, 5]
arr.splice(1, 1, 88)    // 从index 1 删1个并替换为 88 → arr: [1, 88, 4, 5]
```

---

### 截取与合并（不修改原数组）

#### `.slice()` — 截取子数组（不改原数组）

```ts
const arr = [0, 1, 2, 3, 4];

arr.slice(1, 3)   // [1, 2]     — [start, end)，不含 end
arr.slice(2)      // [2, 3, 4]  — 从 index 2 到末尾
arr.slice(-2)     // [3, 4]     — 倒数 2 个
arr.slice()       // [0,1,2,3,4] — 浅拷贝整个数组（常用！）
```

> **`slice` vs `splice` 记忆**：slice = 切片不动原数组；splice = 外科手术改原数组。

#### `.concat()` — 合并数组（不改原数组）

```ts
[1, 2].concat([3, 4])        // [1, 2, 3, 4]
[1, 2].concat([3], [4, 5])   // [1, 2, 3, 4, 5]
// 现代写法更常用展开运算符：
[...a, ...b]
```

---

### 查找与判断

#### `.indexOf()` / `.lastIndexOf()` — 查找值的位置

```ts
[1, 2, 3, 2].indexOf(2)      // 1（第一次出现的 index）
[1, 2, 3, 2].lastIndexOf(2)  // 3（最后一次出现的 index）
[1, 2, 3].indexOf(99)        // -1（找不到）
```

> ⚠️ 用严格相等 `===` 比较，不适合查找对象（用 `.find()` 代替）。

#### `.includes()` — 是否包含某个值

```ts
[1, 2, 3].includes(2)   // true
[1, 2, 3].includes(99)  // false
```

---

### 转换与格式化

#### `.join()` — 数组 → 字符串

```ts
['a', 'b', 'c'].join('-')   // 'a-b-c'
['a', 'b', 'c'].join('')    // 'abc'
['a', 'b', 'c'].join()      // 'a,b,c'（默认逗号）
```

#### `String.split()` — 字符串 → 数组（与 join 互为逆操作）

```ts
'a-b-c'.split('-')   // ['a', 'b', 'c']
'hello'.split('')    // ['h', 'e', 'l', 'l', 'o']
```

---

### 填充与创建

#### `.fill()` — 填充指定值（⚠️ 原地修改）

```ts
new Array(3).fill(0)         // [0, 0, 0]
[1, 2, 3, 4].fill(0, 1, 3)  // [1, 0, 0, 4] — fill(值, start, end)
```

#### `Array.from()` — 从类数组/可迭代对象创建数组

```ts
Array.from('hello')                // ['h', 'e', 'l', 'l', 'o']
Array.from({ length: 3 }, (_, i) => i)  // [0, 1, 2]
Array.from(new Set([1, 2, 2, 3])) // [1, 2, 3]（Set 去重后转数组）
```

#### `Array.isArray()` — 判断是否是数组

```ts
Array.isArray([1, 2, 3])  // true
Array.isArray('hello')    // false
Array.isArray(null)       // false
```

---

### 高阶函数（变换 / 筛选 / 聚合）

#### `.map()` — 变换，返回等长新数组

```ts
// 项目用法
data?.map((post) => <div key={post.id}>{post.title}</div>)

// 基础用法
[1, 2, 3].map(n => n * 2)                    // [2, 4, 6]
[1, 2, 3].map((n, index) => `${index}:${n}`) // ['0:1', '1:2', '2:3']
```

#### `.filter()` — 筛选，返回满足条件的子数组

```ts
posts.filter(p => p.published)   // 只保留已发布的
posts.filter(p => p.id > 10)     // id 大于 10 的
```

#### `.find()` / `.findIndex()` — 找第一个匹配项

```ts
posts.find(p => p.id === 1)        // 返回对象 或 undefined
posts.findIndex(p => p.id === 1)   // 返回 index 或 -1
```

#### `.reduce()` — 聚合，归并为单个值

```ts
// 求和
[1, 2, 3].reduce((acc, cur) => acc + cur, 0)  // 6

// 数组 → 对象（高频用法）
posts.reduce((acc, post) => {
  acc[post.id] = post;
  return acc;
}, {} as Record<number, Post>)
```

#### `.forEach()` — 遍历副作用，无返回值

```ts
posts.forEach(post => console.log(post.title))
// ⚠️ 不能 break，不能 return 值，只做副作用
```

#### `.some()` / `.every()` — 存在/全部判断

```ts
posts.some(p => p.published)   // 至少一个已发布 → boolean
posts.every(p => p.published)  // 全部已发布 → boolean
```

#### `.sort()` — 排序（⚠️ 原地修改原数组）

```ts
[...posts].sort((a, b) => a.id - b.id)  // 升序（先展开避免改原数组）
[...posts].sort((a, b) => b.id - a.id)  // 降序
```

#### `.flat()` / `.flatMap()` — 展平嵌套

```ts
[[1, 2], [3, 4]].flat()  // [1, 2, 3, 4]

['hello world', 'foo bar']
  .flatMap(s => s.split(' '))  // ['hello', 'world', 'foo', 'bar']
```

---

## 二、Object 对象操作

### `Object.keys/values/entries()`

```ts
const user = { id: 1, name: 'Alice' };

Object.keys(user)    // ['id', 'name']
Object.values(user)  // [1, 'Alice']
Object.entries(user) // [['id', 1], ['name', 'Alice']]

// 遍历对象
Object.entries(user).forEach(([key, value]) => console.log(key, value))
```

### 展开运算符 `...` — 合并/克隆/更新

```ts
const a = { x: 1, y: 2 };
const b = { y: 99, z: 3 };

{ ...a, ...b }    // { x: 1, y: 99, z: 3 } — 后者覆盖前者
{ ...a, x: 999 } // { x: 999, y: 2 }       — immutable 更新（React 常用）
```

---

## 三、TS 特有：可选链 `?.` 与 空值合并 `??`

```ts
data?.map(post => post.title)  // data 为 null/undefined 时不报错，直接返回 undefined
post.title ?? '无标题'          // 左边是 null/undefined 才取右边的值
```

---

## 四、链式操作（实战最重要）

```ts
// 筛选已发布 → 按 views 降序 → 提取标题
const titles = posts
  .filter(p => p.published)
  .sort((a, b) => b.views - a.views)
  .map(p => p.title);
```

---

## 五、快速对比

| 方法 | 返回值 | 改变原数组 | 用途 |
|------|--------|:---------:|------|
| `push` | 新长度 | ⚠️ **是** | 尾部追加元素 |
| `pop` | 被删元素 | ⚠️ **是** | 删除尾部元素 |
| `unshift` | 新长度 | ⚠️ **是** | 头部追加元素 |
| `shift` | 被删元素 | ⚠️ **是** | 删除头部元素 |
| `splice` | 被删元素数组 | ⚠️ **是** | 任意位置增删改 |
| `fill` | 原数组 | ⚠️ **是** | 填充指定值 |
| `sort` | 原数组 | ⚠️ **是** | 排序 |
| `slice` | 新数组 | ❌ | 截取子数组 |
| `concat` | 新数组 | ❌ | 合并数组 |
| `indexOf` | number / `-1` | ❌ | 查找值的位置 |
| `includes` | `boolean` | ❌ | 是否包含某值 |
| `join` | 字符串 | ❌ | 数组转字符串 |
| `map` | 新数组（等长） | ❌ | 转换每个元素 |
| `filter` | 新数组（可短） | ❌ | 筛选元素 |
| `find` | 单个元素 / `undefined` | ❌ | 查找第一个匹配 |
| `findIndex` | number / `-1` | ❌ | 查找第一个匹配的索引 |
| `reduce` | 任意类型 | ❌ | 聚合/归并 |
| `forEach` | `undefined` | ❌ | 副作用遍历 |
| `some` | `boolean` | ❌ | 存在判断 |
| `every` | `boolean` | ❌ | 全部判断 |
| `flat` | 新数组 | ❌ | 展平嵌套 |
| `flatMap` | 新数组 | ❌ | map + flat(1层) |
| `Array.from` | 新数组 | ❌ | 从类数组/迭代器创建 |
| `Array.isArray` | `boolean` | ❌ | 判断是否为数组 |

---

## 练习题

假设有以下数据：

```ts
type Post = {
  id: number;
  title: string;
  published: boolean;
  views: number;
  tags: string[];
};

const posts: Post[] = [
  { id: 1, title: 'Hello World', published: true,  views: 120, tags: ['react', 'ts'] },
  { id: 2, title: 'Draft Post',  published: false, views: 0,   tags: ['draft'] },
  { id: 3, title: 'Deep Dive',   published: true,  views: 350, tags: ['ts', 'node'] },
  { id: 4, title: 'Quick Tips',  published: true,  views: 80,  tags: ['react'] },
  { id: 5, title: 'Unpublished', published: false, views: 10,  tags: ['misc'] },
];
```

### 🟢 基础

**Q1.** 提取所有 post 的 `title`，得到字符串数组。

**Q2.** 找出所有 `published` 为 `true` 的 post。

**Q3.** 找到 `id` 为 `3` 的 post 对象。

---

### 🟡 进阶

**Q4.** 计算所有已发布 post 的总 `views`。

**Q5.** 将 posts 数组转为以 `id` 为 key 的对象 `{ [id]: Post }`。

**Q6.** 获取所有 post 的 `tags`，合并去重后得到一个标签列表。

---

### 🔴 综合

**Q7.** 获取已发布 post 的标题列表，按 `views` 从高到低排序。

**Q8.** 将每个 post 转换为 `{ id, title, summary }` 格式，`summary` = `已发布` 或 `草稿`。

**Q9.** 判断是否存在 `views` 超过 300 的 post；以及是否所有 post 都有 tags。

---

### 答案

<details><summary>Q1</summary>

```ts
posts.map(p => p.title)
// ['Hello World', 'Draft Post', 'Deep Dive', 'Quick Tips', 'Unpublished']
```
</details>

<details><summary>Q2</summary>

```ts
posts.filter(p => p.published)
```
</details>

<details><summary>Q3</summary>

```ts
posts.find(p => p.id === 3)
```
</details>

<details><summary>Q4</summary>

```ts
posts
  .filter(p => p.published)
  .reduce((acc, p) => acc + p.views, 0)
// 550
```
</details>

<details><summary>Q5</summary>

```ts
posts.reduce((acc, post) => {
  acc[post.id] = post;
  return acc;
}, {} as Record<number, Post>)
```
</details>

<details><summary>Q6</summary>

```ts
const allTags = posts.flatMap(p => p.tags);
const unique = [...new Set(allTags)];
// ['react', 'ts', 'draft', 'node', 'misc']
```
</details>

<details><summary>Q7</summary>

```ts
posts
  .filter(p => p.published)
  .sort((a, b) => b.views - a.views)
  .map(p => p.title)
// ['Deep Dive', 'Hello World', 'Quick Tips']
```
</details>

<details><summary>Q8</summary>

```ts
posts.map(p => ({
  id: p.id,
  title: p.title,
  summary: p.published ? '已发布' : '草稿',
}))
```
</details>

<details><summary>Q9</summary>

```ts
posts.some(p => p.views > 300)       // true
posts.every(p => p.tags.length > 0)  // true
```
</details>
