# Turborepo Advanced Patterns: The "topo" Task and Caching

This guide explains two common but often misunderstood patterns in Turborepo configurations: the `topo` task and the `cache: false` setting.

## 1. The `topo` Task (Topological Preparation)

### What is it?
`topo` is a community convention (often seen in the T3 Stack and other modern monorepos) used to act as a **lightweight preparation phase** before other tasks run. It typically handles code generation or type definition creation without performing a full build.

### The Problem it Solves
In a TypeScript monorepo, an application (e.g., `apps/web`) often needs type definitions from its dependencies (e.g., `packages/db` or `packages/ui`) before it can run linting or type checking.

*   **Without `topo`**: You might depend on `^build`. This forces a full compilation of dependencies (transpiling to JS, minifying, etc.) just to get type definitions. This is slow and wasteful.
*   **With `topo`**: You define a specific task that *only* generates the necessary artifacts (like `.d.ts` files or Prisma clients) and returns immediately.

### Configuration Example
In `turbo.json`:

```json
{
  "tasks": {
    "topo": {
      "dependsOn": ["^topo"] // Runs strictly in dependency order
    },
    "typecheck": {
      "dependsOn": ["^topo"] // Waits for dependencies to be "ready", but not "built"
    }
  }
}
```

In `packages/db/package.json`:
```json
"scripts": {
  "topo": "prisma generate" // Do existing work
}
```

In `packages/ui/package.json`:
```json
"scripts": {
  "topo": "echo 'Nothing to do'" // No-op if not needed
}
```

### Key Syntax: `^topo` vs `topo`
*   **`topo`**: The task defined in the *current* package.
*   **`^topo`**: The `topo` task of all *upstream dependencies*.
*   **`dependsOn: ["^topo"]`**: "Before running my task, run the `topo` task of all packages I depend on."

---

## 2. Why `cache: false` for Dev/Start?

You will often see this configuration:

```json
"dev": {
  "cache": false,
  "persistent": true
},
"start": {
  "cache": false
}
```

### Why no cache?
1.  **Persistent Processes**: Tasks like `dev` start a long-running server (e.g., `next dev`). You cannot "restore" a running server process from a file cache.
2.  **Source Dependencies**: In modern monorepos (Next.js + Turborepo), local packages are consumer via **source code** (using `paths` in `tsconfig.json` or workspace symlinks), not built artifacts.
    *   When you change `packages/ui/Button.tsx`, the consuming app (via Webpack/Turbopack) detects the file change and Hot Module Reloads (HMR) instantly.
    *   There is no intermediate "build" step to cache.

### Summary
*   **Use Cache**: For finite, determinstic tasks checking or producing files (`build`, `lint`, `test`, `typecheck`).
*   **Disable Cache**: For verify long-running processes or tasks with side-effects that need to run every time (`dev`, `start`, `db:push`).
