# Repository Guidelines

## Project Structure & Module Organization
This is a Turborepo monorepo using Bun workspaces. Product code lives in `apps/app` (main Next.js app) and `apps/web` (marketing site). Backend services and database assets are in `apps/api` (Supabase config, migrations, edge functions, and `seed.sql`). Shared packages are in `packages/*` (for example `packages/ui`, `packages/supabase`, `packages/analytics`), and shared TypeScript configs live in `tooling/typescript`.

## Build, Test, and Development Commands
- `bun dev`: run all dev tasks in parallel via Turbo.
- `bun dev:app` / `bun dev:web`: run a single Next.js app.
- `bun dev:jobs`: run Trigger.dev jobs locally.
- `bun build`: build all apps and packages.
- `bun lint`: run Biome lint plus repo checks.
- `bun format`: auto-format with Biome.
- `bun typecheck`: run TypeScript checks.
- `bun test`: run workspace test tasks (if configured).
- From `apps/api`: `bun run dev` (Supabase local), `bun run migrate`, `bun run seed`, `bun run generate` (types).

## Coding Style & Naming Conventions
Use Biome for formatting and linting; run `bun format` before pushing. Indentation is spaces (Biome-managed). Prefer `kebab-case` for file names (for example `copy-text.tsx`), `PascalCase` for React components, and `camelCase` for functions and variables. Environment variables should be `SCREAMING_SNAKE_CASE`.

## Testing Guidelines
There is no dedicated test framework wired in yet. If you add tests, use `*.test.ts(x)` colocated with the source and ensure `bun test`/`turbo test` runs them. Keep tests deterministic and avoid relying on real external services.

## Commit & Pull Request Guidelines
Commits in this repo are short and imperative, often with a conventional prefix (for example `fix:`, `refactor:`, `update:`). Pull requests should include: a clear summary, testing notes or commands run, linked issues, and screenshots for UI changes. Call out any new environment variables or Supabase migrations in the description.

## Configuration & Data
Runtime configuration is per app; check each app’s `.env` (and `.env.example` when present). Database schema changes belong in `apps/api/supabase/migrations`, and data samples belong in `apps/api/supabase/seed.sql`.
