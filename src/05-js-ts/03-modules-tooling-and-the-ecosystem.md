# Modules, Tooling and the Ecosystem

Coming from .NET, the JavaScript ecosystem can feel chaotic. In .NET, one SDK gives you
the compiler, build system, package manager, test runner and formatter, all from one
vendor. In JavaScript, each of those is a separate tool, usually from a different project,
and the popular choices change every few years. A new frontend project can involve a
package manager, a bundler, a compiler, a linter, a formatter, a test runner and a dozen
configuration files.

This chapter maps that landscape onto concepts you already know, explains the module
system everything is built on, and sets up Beacon's frontend toolchain.

---

## 1. The problem: from source files to something a browser can run

Your source code is hundreds of TypeScript files importing each other and third-party
packages. A browser needs a small number of optimized JavaScript, CSS and asset files it
can download quickly. Between the two:

1. **Dependencies** must be installed and versioned.
2. **TypeScript** (and JSX) must be compiled to JavaScript.
3. **Modules** must be resolved and combined (bundled) for efficient loading.
4. Output must be **minified**, **split** into chunks, and given **content hashes** for
   caching.
5. During development, all of this should happen **instantly** on save.
6. Code must be **linted**, **formatted** and **tested**.

---

## 2. The mental model: the .NET mapping

| Job | .NET | JavaScript/TypeScript |
|---|---|---|
| Runtime | CLR | Browser engine (V8, etc.) / Node.js |
| Package manager | NuGet | **npm**, pnpm, Yarn, Bun |
| Package manifest | `.csproj` `<PackageReference>` | `package.json` |
| Lock file | `packages.lock.json` (optional) | `package-lock.json` / `pnpm-lock.yaml` (always commit) |
| Compiler | Roslyn | **TypeScript (`tsc`)** for type checking; esbuild/SWC/Oxc for fast transpiling |
| Build system | MSBuild | **Vite** (dev server + bundler), webpack, Rolldown, Rspack |
| Task runner | `dotnet` CLI, MSBuild targets | `package.json` **scripts** (`npm run build`) |
| Linter / analyzers | Roslyn analyzers | **ESLint** (with typescript-eslint), Biome, Oxlint |
| Formatter | `dotnet format` | **Prettier**, Biome |
| Test runner | xUnit + `dotnet test` | **Vitest**, Jest; Playwright for E2E |
| Assembly | `.dll` | **Module** (a file) / **package** (a folder with `package.json`) |

---

## 3. Modules

### ES modules

Every modern JavaScript file can be a **module** with its own scope. It exports what it
wants to share and imports what it needs:

```js
// tickets/api.js
export async function getTicket(id) { /* ... */ }
export const PAGE_SIZE = 20;
export default class TicketClient { /* ... */ }

// app.js
import TicketClient, { getTicket, PAGE_SIZE } from './tickets/api.js';
import * as api from './tickets/api.js';
const { formatDistance } = await import('date-fns');   // dynamic import: loads on demand
```

Properties of **ES modules (ESM)**:

- **Static structure**: `import`/`export` are declared at the top level, so tools can
  analyze dependencies without running code. That enables **tree shaking**: removing exports
  that are never imported.
- **Live bindings**: imports are references to the exporter's variables, not copies.
- **Evaluated once**: a module is executed the first time it's imported; later imports get
  the same instance (like a static class initializer).
- **Strict mode** by default.
- **Dynamic `import()`** returns a promise and is the basis of **code splitting**: loading
  parts of the app only when needed.

### CommonJS: the legacy format

Before ESM, Node.js used **CommonJS**: `const x = require('x')` and `module.exports = ...`.
Much of npm still ships CommonJS, and interop between the two formats is a frequent source
of confusing errors ("ERR_REQUIRE_ESM", "default is not a function"). Modern projects use ESM
(`"type": "module"` in `package.json`); bundlers and Node.js handle the interop.

### Named vs default exports

Prefer **named exports**: they're explicit, consistent across imports, refactor-friendly
(rename updates all imports) and tree-shake reliably. Default exports let every importer
choose a different name for the same thing.

### Module resolution

`import x from './utils'` is a *relative* import. `import { z } from 'zod'` is a *bare
specifier* resolved to `node_modules/zod`, using the package's `package.json` `exports`
field to pick the right file for the environment (ESM vs CommonJS, browser vs Node).
TypeScript and bundlers must agree on resolution rules; `"moduleResolution": "bundler"` in
`tsconfig.json` matches how Vite resolves.

---

## 4. Packages and npm

### `package.json`

```json
{
  "name": "beacon-web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "tsc --noEmit && vite build",
    "test": "vitest",
    "lint": "eslint .",
    "format": "prettier --write ."
  },
  "dependencies": {
    "react": "^19.2.0",
    "react-dom": "^19.2.0"
  },
  "devDependencies": {
    "typescript": "^7.0.0",
    "vite": "...",
    "vitest": "..."
  }
}
```

- **`dependencies`** are needed at run time (shipped in the bundle); **`devDependencies`**
  are build and test tools.
- **Scripts** are your build targets: `npm run dev`, `npm run build`, `npm test`.

### Semantic versioning and ranges

Versions are `MAJOR.MINOR.PATCH`. Ranges:

- `^19.2.0`: any `19.x.y` ≥ 19.2.0 (minor and patch updates).
- `~19.2.0`: any `19.2.y` (patch updates only).
- `19.2.0`: exactly that version.

The **lock file** records the exact resolved version of every package (including transitive
ones). **Always commit it**, and install from it in CI with `npm ci` (which fails if
`package.json` and the lock file disagree), so every build uses identical dependencies.

### `node_modules` and dependency depth

A typical frontend app has a few dozen direct dependencies and **hundreds to over a thousand
transitive ones**. This is the npm ecosystem's culture of small packages, and it has costs:
install size, build times and, most importantly, a large supply-chain attack surface.

> **⚠️ What can go wrong:** npm packages run code at install time (`postinstall` scripts)
> and at build time with your developer and CI credentials. Malicious or compromised
> packages, typosquats (`reacct`) and hijacked maintainer accounts are real and recurring
> attacks. Defenses: lock files and `npm ci`; `npm audit` and Dependabot; be skeptical of
> new, obscure dependencies; prefer well-maintained packages with many users; pnpm's or
> npm's options to restrict install scripts; and never install packages an AI assistant
> suggests without checking that they exist and are the real thing (Book XI, Chapter 11).

### npm, pnpm, Yarn, Bun

All install from the same registry. **pnpm** uses a content-addressed store with links,
making installs fast and disk-efficient, and its strict `node_modules` layout prevents
importing packages you didn't declare. **npm** ships with Node and is perfectly adequate.
Pick one per repository and stick with it (the lock file format differs). This book uses
**pnpm**; the commands map directly to npm.

---

## 5. Build tools

### What a bundler does

```text
 src/main.tsx ──imports──► App.tsx ──► TicketList.tsx ──► api.ts ──► node_modules/zod
         │
         ▼  Bundler: resolve graph → transform (TS/JSX → JS) → tree-shake → split chunks → minify → hash
 dist/
   index.html
   assets/index-3f9a1c.js        (app code)
   assets/vendor-91bd02.js       (third-party code, changes rarely → caches well)
   assets/TicketDetail-7c2e.js   (lazy-loaded route chunk)
   assets/index-a71f.css
```

**Content hashes** in filenames enable aggressive caching: the browser can cache
`index-3f9a1c.js` forever (`Cache-Control: immutable`), because any change produces a new
name. `index.html` must not be cached long, since it points at the current hashes (Book VIII revisits this when hosting the app).

### Vite

**Vite** is the standard tool for new React projects:

- **Dev server**: serves source files as native ES modules to the browser, transforming
  each file on request. Startup is near-instant regardless of app size, and **Hot Module
  Replacement (HMR)** updates the changed component in place, preserving state.
- **Production build**: a full optimized bundle.
- **Plugins** for React, TypeScript paths, PWA and more.

> **🔄 Current (as of October 2026):** Vite's internals are moving to **Rolldown**
> (a Rust-based bundler) for faster builds, part of a broader trend of JavaScript tooling
> being rewritten in Rust and Go (esbuild, SWC, Oxc, Biome, Rspack, and TypeScript 7's
> native compiler). The concepts in this chapter are unchanged; the tools get faster.

### Transpiling vs type checking

A key detail: in a Vite project, **the bundler does not type-check**. It strips types (very
fast) and moves on. Type errors are found by running `tsc --noEmit` separately: in your
editor, in a pre-commit hook, and in CI. That's why the `build` script above runs `tsc`
first.

---

## 6. Code quality tools

### ESLint

Finds bugs and bad patterns: unused variables, missing `await`, React hooks used
incorrectly (Book VI), accessibility problems in JSX, and with **typescript-eslint**'s
type-aware rules, things like floating promises and unsafe `any` usage.

```js
// eslint.config.js (flat config)
import js from '@eslint/js';
import tseslint from 'typescript-eslint';
import reactHooks from 'eslint-plugin-react-hooks';

export default tseslint.config(
  js.configs.recommended,
  ...tseslint.configs.recommendedTypeChecked,
  reactHooks.configs['recommended-latest'],
  { languageOptions: { parserOptions: { projectService: true } } },
  { rules: { '@typescript-eslint/no-floating-promises': 'error' } },
);
```

### Prettier

An opinionated formatter: it ends formatting debates (Book II, Chapter 3: automate the
objective). Run on save and check in CI. **Biome** combines linting and formatting in one
fast tool and is a popular alternative.

### Editor integration

VS Code (and Rider/WebStorm) run the TypeScript language service, ESLint and Prettier as
you type. Configure format-on-save and the experience is as integrated as Visual Studio for
C#.

---

## 7. Monorepos and where the frontend lives

Beacon keeps the frontend in the same repository as the backend, in `web/`. Benefits: one
pull request can change an API endpoint and the UI that uses it; contracts stay in sync
(Book VII, Chapter 2); one CI pipeline. Larger organizations use workspace tools (pnpm
workspaces, Nx, Turborepo) to manage many packages in one repository.

---

## 8. In practice: scaffolding Beacon's frontend

```bash
# from the repository root
pnpm create vite web --template react-ts
cd web
pnpm install
pnpm dev                    # http://localhost:5173
```

Proxy API calls to Beacon.Api during development, so the browser sees one origin (no CORS
in development, and cookies behave as in production behind the BFF; Book VI, Chapter 5):

```ts
// web/vite.config.ts
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      '/api': { target: 'https://localhost:7001', changeOrigin: true, secure: false },
      '/hubs': { target: 'https://localhost:7001', ws: true, secure: false },   // SignalR
    },
  },
  build: { sourcemap: true },
});
```

Strict TypeScript configuration (Chapter 4 explains each flag):

```jsonc
// web/tsconfig.app.json (excerpt)
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true,
    "noImplicitOverride": true,
    "verbatimModuleSyntax": true,
    "noEmit": true
  },
  "include": ["src"]
}
```

Add the quality tools and wire them into CI next to the .NET build (Book II, Chapter 3):

```bash
pnpm add -D eslint @eslint/js typescript-eslint eslint-plugin-react-hooks prettier vitest
```

```yaml
# .github/workflows/ci.yml (additional job)
  web:
    runs-on: ubuntu-latest
    defaults: { run: { working-directory: web } }
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 24, cache: pnpm, cache-dependency-path: web/pnpm-lock.yaml }
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec tsc -b --noEmit
      - run: pnpm lint
      - run: pnpm exec prettier --check .
      - run: pnpm test -- --run
      - run: pnpm build
```

Move the API client from Chapter 2 into `web/src/api/`, rename it to `.ts`, and the next two
chapters add its types.

---

## 9. What can go wrong

- **Not committing the lock file**, or installing with `npm install` in CI: builds that
  differ from your machine.
- **Assuming the bundler type-checks.** Type errors ship unless `tsc` runs in CI.
- **Dependency bloat**: adding a 500 KB library for one function. Check bundle size
  (`vite build` reports chunk sizes; tools like `rollup-plugin-visualizer` show what's inside).
- **Supply-chain attacks** via malicious or compromised packages.
- **ESM/CommonJS interop errors** from older packages.
- **Caching `index.html`** long-term, so users keep loading old asset hashes after a deploy.
- **Tooling churn**: chasing the newest tool without a reason.

---

## 10. How an experienced engineer thinks about this

- **Map new tools to known concepts.** Package manager, compiler, bundler, linter, test
  runner: the jobs are the same as in .NET.
- **Reproducible builds**: lock files, `npm ci`/`--frozen-lockfile`, pinned Node versions.
- **Every dependency is a liability as well as an asset.** Fewer, well-maintained packages.
- **Fast feedback loops** (instant dev server, editor diagnostics) matter for productivity;
  **strict checks in CI** matter for correctness.
- **Choose boring, mainstream tools** unless there's a concrete reason not to.

---

## 11. Check yourself

**Questions**

1. What's the difference between ES modules and CommonJS?
2. What is tree shaking, and why do named exports help it?
3. What does `^19.2.0` allow? Why commit the lock file?
4. What does a bundler do? Why are content hashes in filenames useful?
5. Why doesn't Vite's dev server need to bundle?
6. Why must `tsc` run separately from the Vite build?
7. What are the main supply-chain risks in the npm ecosystem?

**Exercises**

1. Scaffold `web/` as in section 8, run `pnpm build`, and inspect the `dist/` output and
   chunk sizes.
2. Add a dynamic `import()` for a rarely used page and see the separate chunk appear.
3. Run `pnpm audit` (or `npm audit`) and research one finding.
4. Count the packages in `node_modules` (`pnpm list --depth Infinity | wc -l`) for the
   scaffolded app.

**Interview-style questions**

- "What does a bundler like Vite or webpack do?"
- "How do you keep frontend dependencies secure?"
- "What's the difference between dependencies and devDependencies?"

---

## 12. Going deeper

- [MDN: JavaScript modules](https://developer.mozilla.org/docs/Web/JavaScript/Guide/Modules)
- [Vite documentation](https://vite.dev/guide/)
- [npm docs: package.json](https://docs.npmjs.com/cli/configuring-npm/package-json) and
  [pnpm docs](https://pnpm.io/)
- [typescript-eslint](https://typescript-eslint.io/)

**Next:** [Chapter 4 — TypeScript Fundamentals](04-typescript-fundamentals.md) adds the
type system on top of everything in this book so far.
