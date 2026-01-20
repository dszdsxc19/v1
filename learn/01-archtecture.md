# DeepWiki Q&A with Code Context for Repository: midday-ai/v1
## Q1
V1 uses a monorepo structure that organizes code into apps, packages, and tooling directories. 这样拆分包的原理是什么？Saas, 以及 baas? 类似的思路还是？
### Answer
V1的monorepo结构基于模块化和代码复用的现代架构原则，与SaaS和BaaS的设计思路有相似之处。

## 核心原理

### 1. 关注点分离 (Separation of Concerns)
V1将代码按功能职责组织到不同层级：
- **apps/** - 独立的应用程序，每个有明确的业务职责 v1:46-51 
- **packages/** - 可复用的共享功能模块 v1:52-59 
- **tooling/** - 共享的开发配置和工具

### 2. 代码复用和依赖管理
通过workspace机制实现包间依赖：
```json
// apps/app/package.json中的依赖示例
"dependencies": {
  "@v1/analytics": "workspace:*",
  "@v1/supabase": "workspace:*",
  "@v1/ui": "workspace:*"
}
``` v1:14-18 

### 3. 微服务架构思维
虽然代码在同一个仓库中，但运行时体现为独立服务：
- **web** (3001端口) - 营销网站
- **app** (3000端口) - 主产品应用  
- **api** - Supabase后端服务

## 与SaaS/BaaS的相似性

### SaaS模式特征
- **多租户架构**: 通过Supabase实现数据隔离和认证 v1:5-25 
- **服务分层**: 前端应用、API网关、数据存储分层设计
- **可扩展性**: 模块化包结构支持功能扩展

### BaaS模式特征
- **后端即服务**: Supabase提供完整的后端功能（认证、数据库、存储） v1:29-29 
- **API抽象**: 通过packages/supabase封装数据库操作 v1:100-123 
- **服务集成**: 集成多种第三方服务（Upstash、Resend、Trigger.dev等）

## 构建系统支持
使用Turborepo管理复杂的依赖关系和构建流程 v1:9-26 ，确保：
- 增量构建
- 依赖缓存
- 并行执行

这种架构既保持了代码的统一管理，又实现了运行时的服务解耦，是现代SaaS产品开发的最佳实践。

## Notes
V1的monorepo结构体现了现代前端架构的演进趋势，通过工具链（Turborepo、Biome）和设计模式的结合，实现了开发效率和代码质量的平衡。这种模式特别适合快速迭代的SaaS产品开发。

Wiki pages you might want to explore:
- [Architecture Overview (midday-ai/v1)](/wiki/midday-ai/v1#1.2)
- [Applications (midday-ai/v1)](/wiki/midday-ai/v1#2)
### Citations
**File:** README.md (L29-29)
```markdown
[Supabase](https://supabase.com/) - Authentication, database, storage<br>
```
**File:** README.md (L46-51)
```markdown
.
├── apps                         # App workspace
│    ├── api                     # Supabase (API, Auth, Storage, Realtime, Edge Functions)
│    ├── app                     # App - your product
│    ├── web                     # Marketing site
│    └── ...
```
**File:** README.md (L52-59)
```markdown
├── packages                     # Shared packages between apps
│    ├── analytics               # OpenPanel analytics
│    ├── email                   # React email library
│    ├── jobs                    # Trigger.dev background jobs
│    ├── kv                      # Upstash rate-limited key-value storage
│    ├── logger                  # Logger library
│    ├── supabase                # Supabase - Queries, Mutations, Clients
│    └── ui                      # Shared UI components (Shadcn)
```
**File:** apps/app/package.json (L14-18)
```json
  "dependencies": {
    "@v1/analytics": "workspace:*",
    "@v1/kv": "workspace:*",
    "@v1/supabase": "workspace:*",
    "@v1/ui": "workspace:*",
```
**File:** packages/supabase/src/clients/server.ts (L5-25)
```typescript
export const createClient = () => {
  const cookieStore = cookies();

  return createServerClient<Database>(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return cookieStore.getAll();
        },
        setAll(cookiesToSet) {
          try {
            for (const { name, value, options } of cookiesToSet) {
              cookieStore.set(name, value, options);
            }
          } catch (error) {}
        },
      },
    },
  );
```
**File:** packages/supabase/src/types/db.ts (L100-123)
```typescript
export type Tables<
  PublicTableNameOrOptions extends
    | keyof (PublicSchema["Tables"] & PublicSchema["Views"])
    | { schema: keyof Database },
  TableName extends PublicTableNameOrOptions extends { schema: keyof Database }
    ? keyof (Database[PublicTableNameOrOptions["schema"]]["Tables"] &
        Database[PublicTableNameOrOptions["schema"]]["Views"])
    : never = never,
> = PublicTableNameOrOptions extends { schema: keyof Database }
  ? (Database[PublicTableNameOrOptions["schema"]]["Tables"] &
      Database[PublicTableNameOrOptions["schema"]]["Views"])[TableName] extends {
      Row: infer R;
    }
    ? R
    : never
  : PublicTableNameOrOptions extends keyof (PublicSchema["Tables"] &
        PublicSchema["Views"])
    ? (PublicSchema["Tables"] &
        PublicSchema["Views"])[PublicTableNameOrOptions] extends {
        Row: infer R;
      }
      ? R
      : never
    : never;
```
**File:** turbo.json (L9-26)
```json
    "build": {
      "env": [
        "SUPABASE_SERVICE_KEY",
        "SUPABASE_API_KEY",
        "RESEND_API_KEY",
        "LOOPS_ENDPOINT",
        "LOOPS_API_KEY",
        "UPSTASH_REDIS_REST_URL",
        "UPSTASH_REDIS_REST_TOKEN",
        "OPENPANEL_SECRET_KEY",
        "SENTRY_AUTH_TOKEN",
        "SENTRY_ORG",
        "SENTRY_PROJECT"
      ],
      "inputs": ["$TURBO_DEFAULT$", ".env"],
      "dependsOn": ["^build"],
      "outputs": [".next/**", "!.next/cache/**", "next-env.d.ts", ".expo/**"]
    },
```