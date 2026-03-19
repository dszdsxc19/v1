# 设计模式：提供者 / 消费者 及其衍生模式

> 这些模式背后都有同一个核心思想：**需要数据/能力的人，不直接生产它；生产它的人，不关心谁用。**  
> 这个思想的正式名称叫「**控制反转（IoC, Inversion of Control）**」。

---

## 一、三种维度的体现

| 维度 | 模式名称 | 典型场景 |
|---|---|---|
| 模块（构建时） | Provider / Consumer | monorepo `packages/` vs `apps/` |
| UI 运行时 | React Context | `<ThemeProvider>` / `useContext` |
| 事件 / 消息（运行时） | Observer / Pub-Sub | EventEmitter、Redis、Kafka |

---

## 二、模块层面（本项目的 packages / apps 拆分）

### 判断标准：**"会被多个 app 用到吗？"**

```
packages/（提供者）       apps/（消费者）
  ├── ui                   ├── app  → 依赖 @v1/ui, @v1/analytics
  ├── analytics            ├── web  → 依赖 @v1/ui
  ├── supabase             └── api  → 依赖 @v1/supabase, @v1/analytics
  ├── kv
  ├── logger
  └── ...
```

- `packages/` 通过 `exports` 字段暴露对外接口，内部实现对消费者不透明。
- 包之间也有层次依赖（如 `analytics` 依赖 `logger`），**最底层的包不依赖任何内部包**。
- 只在单个 app 内使用的代码留在 `apps/xx/src/`，**不要过度提取**。

```
决策口诀：
  只有一个 app 用  → apps/xx/src/
  多个 app 都用    → packages/
  是构建/lint 配置 → tooling/
```

---

## 三、on/emit（EventEmitter）—— 事件驱动

### 是什么

Node.js **原生**提供，在 `node:events` 模块中，**无需安装任何依赖**。

```ts
import { EventEmitter } from 'node:events'

const bus = new EventEmitter()

// 消费者：监听事件
bus.on('user:login', (data) => {
  console.log('用户登录了', data.userId)
})

// 提供者：发出事件
bus.emit('user:login', { userId: 123 })
```

### 特点

- 发布者和订阅者**共享同一个 EventEmitter 实例**（同一进程内）
- 关注点是「**事件**」：某个动作发生了 → 通知关心的人（动词）
- 无法跨进程、跨服务器

---

## 四、publish/subscribe（Pub/Sub）—— 消息驱动

### 是什么

在发布者和订阅者之间引入一个**中间人（Broker）**，如 Redis、Kafka、RabbitMQ。

```
Publisher → [Broker / Channel] → Subscriber A
                    ↓              Subscriber B
                               Subscriber C（甚至在另一台服务器）
```

```ts
// 发布者（不知道谁在订阅）
await redis.publish('orders:created', JSON.stringify(order))

// 订阅者（不知道谁发布的，甚至可以是另一个服务）
await redis.subscribe('orders:created', (message) => {
  const order = JSON.parse(message)
  // 处理订单...
})
```

### 特点

- 发布者和订阅者**完全不认识对方**，通过 Broker 解耦
- 关注点是「**消息/数据**」：传递一条内容（名词）
- 可以**跨进程、跨服务、跨语言**
- 需要额外的基础设施（Broker）

---

## 五、核心对比

| | `on/emit` (EventEmitter) | `publish/subscribe` (Pub/Sub) |
|---|---|---|
| 中间人 | ❌ 无，共享对象引用 | ✅ 有 Broker |
| 耦合程度 | 弱耦合（同进程） | 极度解耦（可跨网络） |
| 关注点 | 事件（动作，动词） | 消息（数据内容，名词） |
| 能否跨进程 | ❌ 不能 | ✅ 能 |
| 典型实现 | Node.js `EventEmitter`、DOM 事件 | Redis、Kafka、MQTT、RabbitMQ |

---

## 六、一句话记忆

| 模式 | 比喻 |
|---|---|
| `on/emit` | 同一个办公室里喊一声「会议开始了！」 |
| `publish/subscribe` | 发邮件给邮件组，发件人不知道谁订阅了这个组 |

---

## 七、总结：它们都是 IoC 的不同维度

```
控制反转（IoC）
  ├── 空间维度（模块）  → packages / apps 拆分
  ├── 树形维度（UI）    → React Context Provider / Consumer
  ├── 同进程时间维度    → on / emit (EventEmitter)
  └── 跨进程时间维度    → publish / subscribe (Kafka / Redis)
```

> **解耦程度从上到下递增，引入的基础设施复杂度也从上到下递增。**  
> 按需选择，不要过度设计。
