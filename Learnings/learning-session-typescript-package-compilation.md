# Learning Session: TypeScript Package Compilation, JS Module Systems & Monorepo Architecture

## Why This Session Exists

This is a deep-dive into a real-world problem pattern I've encountered: a shared React component library that ships raw TypeScript source instead of compiled JavaScript. The topic touches almost every one of my high-priority learning goals at once — monorepo structure, TypeScript configuration, build tooling, module systems, micro-frontends, and bundle optimization.

**The scenario**: A shared React component library called `@acme/ui` currently ships raw TypeScript (`.ts`) source files instead of compiled JavaScript. This causes friction for every external consumer. The goal: compile the library once inside its own repo, ship compiled `.js` files, and set up a proper `exports` field in `package.json`.

---

## Learning Opportunities This Session Covers

These map directly to my HIGH PRIORITY goals from [Opportunities_To_Learn.md](./Opportunities_To_Learn.md):

| My Goal | What This Session Teaches |
| --- | --- |
| **Monorepo structure & workspace management** | How sub-packages live in a monorepo, how `workspace:*` references work, and what it means to "collapse" sub-packages |
| **Package managers (Yarn/npm)** | The `exports` field, conditional exports (`import`/`require`/`types`), how Node.js resolves package imports |
| **TypeScript configuration (multi-project)** | How `tsconfig paths` work, why they're needed for monorepo resolution, and why they differ from `package.json exports` |
| **Build tooling** | What `tsc` actually does, why you'd run it twice (ESM + CJS), what `declaration: true` generates |
| **Modern bundler configuration** | CJS vs. ESM, why ESM enables tree-shaking, why bundlers need the `exports` field |
| **Micro-frontends** | How MFEs consume shared libraries, why aliasing is needed today, what the post-compilation world looks like |
| **Bundle size analysis** | Tree-shaking depends on ESM; understanding this is a prerequisite for proper bundle optimization |
| **CI/CD pipelines** | How compilation correctness can be verified automatically in CI |

---

## The Problem We're Exploring

### What is a "package"?

In the JavaScript world, a **package** is a folder of code shared and reused across projects. When you run `npm install`, npm downloads packages from the internet into `node_modules/`. Your code imports from them.

```
Your project
├── src/
│   └── MyComponent.tsx    ← your code
└── node_modules/
    ├── react/             ← package you installed
    ├── lodash/            ← package you installed
    └── @acme/ui/          ← the package in this scenario
```

### What is `@acme/ui`?

`@acme/ui` is a fictional shared component library. It contains things like `UserProfile`, `Button`, `Modal` — components used by many different apps across a project.

It lives in a monorepo under `packages/ui/` and is published as an npm package. Today it ships its raw TypeScript source — which is unusual and causes problems.

**Inside the monorepo** (how developers write code in the repo):
```tsx
import { UserProfile } from '@acme/components';
import { ColorScheme } from '@acme/types';
```

**Outside the monorepo** (how external micro-frontends consume it today):
```tsx
import { Components, Types } from '@acme/ui-libraries';
const { UserProfile } = Components;
const { ColorScheme } = Types;
```

Notice how different these are. The external pattern — stuffing everything under `Components` and `Types` namespaces in a single barrel package — makes **tree-shaking impossible**. Even if you only need `UserProfile`, you pull in the entire library.

**What the design proposes:**
```tsx
// After: clean, tree-shakeable subpath imports — same path inside and outside
import { UserProfile } from '@acme/ui/components';
import { ColorScheme } from '@acme/ui/types';
```

```
TODAY:
  Inside monorepo   →  import from '@acme/components'
  Outside monorepo  →  import from '@acme/ui-libraries' (barrel namespace)

PROPOSED:
  Inside monorepo   →  import from '@acme/ui/components'
  Outside monorepo  →  import from '@acme/ui/components'  ← same path!
```

### TypeScript vs. JavaScript — why the difference matters

**TypeScript** adds type annotations to JavaScript:
```typescript
// TypeScript (.ts) — what developers write
function add(a: number, b: number): number {
  return a + b;
}
```

But **browsers and Node.js cannot run TypeScript directly**. They only understand plain JavaScript. TypeScript must be "compiled" (translated) first:
```javascript
// JavaScript (.js) — what browsers and Node.js can actually run
function add(a, b) {
  return a + b;
}
```

This compilation is done by `tsc` (the TypeScript Compiler):
```
You write:          tsc compiles it:       Browser/Node runs:
─────────────       ────────────────       ──────────────────
MyComponent.ts  →   MyComponent.js    →    ✅ works
```

### The problem: `@acme/ui` skips compilation

Most packages ship compiled JavaScript. When you import `react`, you get `.js` files.

`@acme/ui` ships the raw TypeScript source (`.ts`) directly. This forces every consumer to configure TypeScript compilation inside `node_modules` — which build tools skip by default.

```
Most packages (e.g. react):          @acme/ui (today — the problem):
────────────────────────────         ──────────────────────────────────
node_modules/react/                  node_modules/@acme/ui/
  ├── index.js         ✅             ├── components/
  └── package.json                   │   └── index.ts   ← raw TypeScript!
                                     └── package.json
```

Think of it like an IKEA instruction PDF in a format your reader can't open. You have to install a special plugin. Then tell every new flatmate to install it too. That's what using `@acme/ui` is like today.

```
Consumer tries to use @acme/ui:

import { UserProfile } from '@acme/ui/components'
                                    │
                                    ▼
                     Finds: components/index.ts
                                    │
                                    ▼
               ❌ "I can't read .ts files from node_modules.
                   I need you to set this up for me."
```

### What the consumer has to do today (manual setup)

Every project consuming `@acme/ui` must do all three of these:

**Step 1 — Tell the bundler to compile TypeScript from node_modules:**
```javascript
// webpack.config.js — extra setup every consumer must add
module: {
  rules: [{
    test: /\.ts$/,
    exclude: /node_modules\/(?!@acme\/ui)/,   // "compile everything EXCEPT node_modules,
    use: 'ts-loader'                           //  but DO compile @acme/ui"
  }]
}
```

**Step 2 — Add path aliases for internal sub-packages:**

Inside `@acme/ui`, files import from each other using workspace package names like `@acme/components`. Outside the monorepo, these don't exist:
```javascript
// webpack.config.js — more extra setup
resolve: {
  alias: {
    '@acme/components': '/path/to/node_modules/@acme/ui/components',
    '@acme/types': '/path/to/node_modules/@acme/ui/types',
    // ... one entry per internal sub-package
  }
}
```

**Step 3 — Do it all again in Jest:**

Jest is a completely separate tool. It has its own module resolution. Everything above must be repeated in a different config format:
```javascript
// jest.config.js — yet more extra setup
moduleNameMapper: {
  '@acme/components': '<rootDir>/node_modules/@acme/ui/components',
  '@acme/types': '<rootDir>/node_modules/@acme/ui/types',
},
transformIgnorePatterns: ['node_modules/(?!@acme/ui)']
```

### What the proposal solves

Compile `@acme/ui` **once**, **inside the library repo**, and ship the resulting `.js` files. Then consumers get simple compiled JavaScript like every other normal package:

```
Consumer project (after compilation):

import { UserProfile } from '@acme/ui/components'
                                    │
                                    ▼
                     Finds: dist/esm/components/index.js
                                    │
                                    ▼
                     ✅ Plain JavaScript. Just works.
                        No extra config needed.
```

---

## Key Concepts

---

### 📦 Concept 1: Source-only vs. Compiled packages

A **source-only** package ships its original TypeScript (`.ts`) files directly, instead of compiled JavaScript. This is unusual — most published packages ship `.js`.

When you `import { UserProfile } from '@acme/ui/components'`, your build tool needs to know what to do with what it finds. If it finds a `.ts` file, it needs a TypeScript compiler. If it finds a `.js` file, it just uses it.

**Today (painful — source-only):**
```
┌─────────────────────────────────────────────────────────────────┐
│  Consumer (MFE or app)                                          │
│                                                                 │
│  import { UserProfile } from '@acme/ui/components'                │
│           │                                                     │
│           ▼                                                     │
│  @acme/ui ships .ts files ◄── unusual! most packages ship .js  │
│           │                                                     │
│           ▼                                                     │
│  ❌ ERROR: can't parse TypeScript                               │
│                                                                 │
│  Consumer must fix this manually:                               │
│    1. Add bundler aliases for internal sub-packages             │
│    2. Configure webpack to compile TS from node_modules         │
│    3. Configure Jest separately too                             │
└─────────────────────────────────────────────────────────────────┘
```

**After compilation (simple):**
```
┌─────────────────────────────────────────────────────────────────┐
│  Consumer (MFE or app)                                          │
│                                                                 │
│  import { UserProfile } from '@acme/ui/components'                │
│           │                                                     │
│           ▼                                                     │
│  @acme/ui ships .js files ✅  ◄── compiled once, works everywhere
│           │                                                     │
│           ▼                                                     │
│  ✅ Works. No extra config needed.                              │
└─────────────────────────────────────────────────────────────────┘
```

#### 🖐️ Try it — See a source-only vs. compiled package

In a temp folder, create a test project and install a couple of packages:

```bash
mkdir /tmp/package-inspect
cd /tmp/package-inspect
npm init -y
npm install decimal.js date-fns
```

Now compare:
```bash
# date-fns — a compiled package. Look at what it ships:
ls node_modules/date-fns/
# You'll see .js files, .d.ts files, and a package.json

# Check its package.json exports field:
cat node_modules/date-fns/package.json | python3 -m json.tool | grep -A5 '"exports"'
```

This is what a properly compiled package looks like. `@acme/ui` (before the fix) would instead show raw `.ts` files.

#### 📚 Resources
- **Article**: [The difference between source and compiled npm packages](https://blog.logrocket.com/publishing-node-modules-typescript-es-modules/) — LogRocket, practical walkthrough
- **YouTube search**: `"npm package typescript compile tsc tutorial"` — Matt Pocock and Theo (t3.gg) both explain this clearly

---

### 📂 Concept 2: CJS vs. ESM (CommonJS vs. ES Modules)

These are the two JavaScript module formats. You've written both without necessarily knowing their names:

```
╔══════════════════════════════════╗   ╔══════════════════════════════════╗
║  CJS (CommonJS)                  ║   ║  ESM (ES Modules)                ║
║  Born: ~2009 with Node.js        ║   ║  Born: ~2015 in the JS standard  ║
╠══════════════════════════════════╣   ╠══════════════════════════════════╣
║                                  ║   ║                                  ║
║  const { X } = require('./file') ║   ║  import { X } from './file'      ║
║  module.exports = { X }          ║   ║  export { X }                    ║
║                                  ║   ║                                  ║
║  ✅ Works everywhere in Node.js  ║   ║  ✅ Modern standard               ║
║  ✅ Jest uses this by default    ║   ║  ✅ Enables tree-shaking          ║
║  ❌ Can't tree-shake             ║   ║  ✅ Vite/webpack/Rspack prefer    ║
║  ❌ Harder to statically analyze ║   ║  ❌ Some older tools don't support║
╚══════════════════════════════════╝   ╚══════════════════════════════════╝
```

A compiled library ships **both** so every tool can pick what it understands:

```
┌─────────────────────────────────────────────────────┐
│                     @acme/ui                        │
│                                                     │
│  dist/esm/  ◄──── webpack, Rspack (production)      │
│  dist/cjs/  ◄──── Jest, Node.js scripts             │
│  dist/types/ ◄─── TypeScript (type checking only)   │
└─────────────────────────────────────────────────────┘
```

#### 🖐️ Try it — Run both formats yourself

```bash
mkdir /tmp/modules-demo
cd /tmp/modules-demo
```

Create a CJS file:
```bash
cat > math.cjs << 'EOF'
function add(a, b) { return a + b; }
function subtract(a, b) { return a - b; }
module.exports = { add, subtract };
EOF
```

Create an ESM file:
```bash
cat > math.mjs << 'EOF'
export function add(a, b) { return a + b; }
export function subtract(a, b) { return a - b; }
EOF
```

Try using them:
```bash
node -e "const { add } = require('./math.cjs'); console.log(add(2, 3));"
node --input-type=module --eval "import { add } from '/tmp/modules-demo/math.mjs'; console.log(add(2, 3));"
```

CJS uses `require()` and `module.exports`. ESM uses `import` and `export`. Node.js uses the file extension (`.cjs` vs `.mjs`) or `"type": "module"` in `package.json` to know which to expect.

#### 📚 Resources
- **Video**: [JavaScript Modules in 100 Seconds](https://www.youtube.com/watch?v=qgRUr-YUk1Q) — Fireship (2 minutes, perfect starting point)
- **Video**: [ES Modules and CommonJS](https://www.youtube.com/watch?v=mK54Cn4ceac) — Theo (t3.gg), goes deeper on the why
- **Article (MDN)**: [JavaScript modules](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Modules) — the authoritative reference
- **Article**: [CommonJS vs ES Modules](https://www.syncfusion.com/blogs/post/js-commonjs-vs-es-modules) — side-by-side comparison

---

### 📄 Concept 3: Declaration files (`.d.ts`) — how types survive compilation

When you compile TypeScript to JavaScript, type annotations are stripped out of the `.js` output. But the type information isn't *lost*. TypeScript's compiler (`tsc`) can generate a separate **declaration file** (`.d.ts`) alongside each `.js` file that contains *only* the types, with all runtime logic removed.

```
You write one file:      tsc produces two files:
────────────────────     ──────────────────────────────────────────────────────
                         UserProfile.js   ← the real code that runs in browsers
UserProfile.ts        →
                         UserProfile.d.ts ← type information only, never "runs"
```

Here's what that looks like concretely:

```typescript
// UserProfile.ts — what you write (source)
export function UserProfile(props: {
  user: User;
  fields: FieldDef[];
}): JSX.Element {
  // ...500 lines of real logic...
  return <div>...</div>;
}
```

```typescript
// UserProfile.d.ts — what tsc generates (declaration file)
export declare function UserProfile(props: {
  user: User;
  fields: FieldDef[];
}): JSX.Element;
// No logic. No function body. Just the shape.
```

Think of the `.d.ts` file as the **packaging label** on a product. The `.js` file is the product itself — it does the work. The `.d.ts` file just tells TypeScript "here's what's inside and what types to expect." Your editor reads the `.d.ts` when giving you autocomplete and type errors. The browser never sees it.

**How each tool uses the files:**

```
Consumer writes: import { UserProfile } from '@acme/ui/components'
                                                   │
                                    ┌──────────────┼──────────────┐
                                    │              │              │
                                webpack          Jest         TypeScript
                               (bundling)      (testing)    (type checking)
                                    │              │              │
                                    ▼              ▼              ▼
                           .js ESM file     .js CJS file    .d.ts file
                           (executes it)   (executes it)  (reads types,
                                                           never executes)
```

This is why a properly structured `exports` field in `package.json` has three conditions — one for each tool:

```json
"./components": {
  "import":  "./dist/esm/components/index.js",     ← webpack uses this
  "require": "./dist/cjs/components/index.js",     ← Jest uses this
  "types":   "./dist/types/components/index.d.ts"  ← TypeScript uses this
}
```

This is completely standard. Every TypeScript package you've used works this way — React, lodash, React Query, all of them. When you get autocomplete for `useState` in VS Code, that's coming from `node_modules/react/index.d.ts`. The actual runtime logic is in the `.js` file next to it.

#### 🖐️ Try it — See a `.d.ts` file in the wild

Set up a temp project with `decimal.js`:

```bash
mkdir /tmp/dts-explore
cd /tmp/dts-explore
npm init -y
npm install decimal.js
```

Now run these two commands and compare the output:

```bash
# The .d.ts file — all types, zero logic
head -60 node_modules/decimal.js/decimal.d.ts
```

```bash
# The .js file — all logic, no types
head -20 node_modules/decimal.js/decimal.js
```

The `.d.ts` output will show things like:
```typescript
export declare class Decimal {
  constructor(n: Decimal.Value);
  abs(): Decimal;
  add(n: Decimal.Value): Decimal;
  // ... just signatures, no function bodies
}
```

The `.js` output will start with `(function (globalScope) { 'use strict'; ...` — thousands of lines of actual math logic, with no mention of types anywhere.

Same package. Two files. Two completely different purposes: one runs your code, one tells TypeScript what the types are.

#### 🖐️ Try it — Generate a `.d.ts` file yourself

```bash
mkdir /tmp/dts-demo
cd /tmp/dts-demo

cat > math.ts << 'EOF'
export function add(a: number, b: number): number {
  return a + b;
}
EOF

# Compile with declaration output
npx tsc math.ts --declaration --emitDeclarationOnly --outDir ./out
cat out/math.d.ts
```

You'll see:
```typescript
export declare function add(a: number, b: number): number;
```

The function body is gone — just the signature remains. That's your `.d.ts` file.

#### 📚 Resources
- **Article (TypeScript docs)**: [Declaration Files — Introduction](https://www.typescriptlang.org/docs/handbook/declaration-files/introduction.html)
- **Video**: search YouTube for `"TypeScript declaration files d.ts explained"` — Matt Pocock has a clear walkthrough

---

### 🌳 Concept 4: Tree-shaking

Tree-shaking is how bundlers remove code you don't import. If you only use `UserProfile` from `@acme/ui/components`, a good bundler removes everything else from your final bundle.

**Imagine `@acme/ui/components` is a tree:**

```
@acme/ui/components
├── UserProfile      ✅  ← you imported this
├── Modal          🍂  ← never imported → removed from bundle
├── Button         🍂  ← never imported → removed from bundle
├── Tooltip        🍂  ← never imported → removed from bundle
└── NavMenu      🍂  ← never imported → removed from bundle

Before tree-shaking:  Bundle = 800KB  (everything)
After tree-shaking:   Bundle =  80KB  (just UserProfile)
```

**Why ESM enables this but CJS doesn't:**

```
CJS — dynamic, can be called anywhere:             ESM — static, always at the top:
────────────────────────────────────               ──────────────────────────────────────
// This is valid CJS — could require               // This is ALL you can write in ESM
// anything at runtime!                            import { UserProfile } from '...'
if (condition) {
  require('./UserProfile')                           // Bundler sees it at build time:
} else {                                           // "Only UserProfile is needed → remove rest"
  require('./Tooltip')                              ✅ Safe to tree-shake
}
❌ Bundler can't predict what's needed
```

In plain English: because CJS supports **dynamic `require()` calls** — you can call `require()` inside an `if`, inside a loop, inside a function — the bundler can never be certain at build time whether something will be needed. So it has to keep *everything*, just in case.

ESM flips this around. `import` statements must be at the top of the file, unconditionally, always. The bundler reads the file once at build time, sees exactly what's imported, and can safely throw away the rest. There's no "maybe I'll need this later" — everything is declared upfront.

> **Terminology note**: "dynamic `require()` calls" (what's described above) is different from "conditional exports" (the `"import"`/`"require"`/`"types"` conditions in the `exports` field). They sound similar but are separate concepts. Dynamic requires are a CJS runtime feature that defeats tree-shaking. Conditional exports are a `package.json` routing feature.

#### 🖐️ Try it — Feel the difference in bundle size

In any empty folder, create a Vite project and measure bundle size with targeted vs. wildcard imports:

```bash
mkdir /tmp/tree-shake-demo
cd /tmp/tree-shake-demo
npm create vite@latest app -- --template vanilla
cd app
npm install
npm install lodash-es
```

Edit `src/main.js`:
```javascript
// Test 1: import just one function
import { add } from 'lodash-es';
console.log(add(1, 2));
```

Build and check the bundle size:
```bash
npm run build
ls -la dist/assets/*.js
```

Now change the import to:
```javascript
// Test 2: import everything
import * as _ from 'lodash-es';
console.log(_.add(1, 2));
```

Rebuild and compare. The difference is tree-shaking in action — `lodash-es` ships ESM so the bundler can eliminate unused functions.

#### 📚 Resources
- **Video**: search YouTube for `"tree shaking webpack explained"` or `"tree shaking vite explained"`
- **Article**: [Tree shaking — web.dev](https://web.dev/articles/reduce-javascript-payloads-with-tree-shaking) — Google's guide, practical and well-illustrated
- **Article**: [Webpack tree shaking docs](https://webpack.js.org/guides/tree-shaking/)

---

### 📋 Concept 5: The `exports` field in `package.json`

The `exports` field is a routing table inside `package.json`. It tells Node.js and bundlers: "when someone imports from me, send them to the right file depending on *who is asking*."

**How the routing works:**

```
Someone writes:  import { UserProfile } from '@acme/ui/components'
                                                     │
                                      package.json  │
                                     ┌──────────────▼──────────────────┐
                                     │  "exports": {                   │
                                     │    "./components": {            │
                                     │      "import":  → dist/esm/...  │ ← webpack picks this
                                     │      "require": → dist/cjs/...  │ ← Jest picks this
                                     │      "types":   → dist/types/.. │ ← TypeScript picks this
                                     │    }                            │
                                     │  }                              │
                                     └─────────────────────────────────┘
```

**Before `exports` (old approach):**
```json
{
  "main":   "./dist/cjs/index.js",    ← CJS only, limited
  "module": "./dist/esm/index.js"     ← non-standard, bundlers invented this
}
```

**After `exports` (modern approach):**
```json
{
  "exports": {
    "./components": {
      "import":  "./dist/esm/components/index.js",    ← standard ESM
      "require": "./dist/cjs/components/index.js",    ← standard CJS
      "types":   "./dist/types/components/index.d.ts" ← TypeScript
    },
    "./types": {
      "import":  "./dist/esm/types/index.js",
      "require": "./dist/cjs/types/index.js",
      "types":   "./dist/types/types/index.d.ts"
    }
  }
}
```

`exports` is also a security boundary: if a path isn't listed, it **can't be imported**. This prevents consumers from importing internal files like `@acme/ui/internal/helpers`.

#### 🖐️ Try it — Read real `exports` fields

In the `/tmp/package-inspect` folder from Concept 1 (or a fresh one):

```bash
mkdir /tmp/exports-explore
cd /tmp/exports-explore
npm init -y
npm install date-fns react react-dom
```

Inspect their `exports` fields:
```bash
# date-fns — dual-format package with conditional exports
grep -A15 '"exports"' node_modules/date-fns/package.json

# react — simpler, but still uses exports
grep -A10 '"exports"' node_modules/react/package.json
```

Notice how `date-fns` routes different tools to different files depending on whether they're asking for ESM (`"import"`) or CJS (`"require"`).

#### 🖐️ Try it — Build a package with an `exports` field

Create a minimal dual-format package by hand:

```bash
mkdir /tmp/my-package
cd /tmp/my-package
```

Create `src/math.ts`:
```typescript
export function add(a: number, b: number): number {
  return a + b;
}
export function multiply(a: number, b: number): number {
  return a * b;
}
```

Create `package.json`:
```json
{
  "name": "my-package",
  "version": "1.0.0",
  "exports": {
    ".": {
      "import":  "./dist/esm/math.js",
      "require": "./dist/cjs/math.js",
      "types":   "./dist/types/math.d.ts"
    }
  }
}
```

Create `tsconfig.esm.json`:
```json
{
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "target": "ES2020",
    "declaration": true,
    "declarationDir": "./dist/types",
    "outDir": "./dist/esm"
  },
  "include": ["src"]
}
```

Create `tsconfig.cjs.json`:
```json
{
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "target": "ES2020",
    "outDir": "./dist/cjs"
  },
  "include": ["src"]
}
```

Compile both:
```bash
npx tsc --project tsconfig.esm.json
npx tsc --project tsconfig.cjs.json
ls dist/esm/ dist/cjs/ dist/types/
```

You now have a properly structured package with separate ESM, CJS, and type outputs — exactly what `@acme/ui` is being designed to produce.

#### 📚 Resources
- **Video**: search YouTube for `"package.json exports field explained"` — Matt Pocock has a clear short video
- **Article (Node.js docs)**: [Package entry points](https://nodejs.org/api/packages.html#package-entry-points)
- **Article**: [Conditional exports](https://nodejs.org/api/packages.html#conditional-exports)

---

### 🗺️ Concept 6: TypeScript path aliases

TypeScript path aliases let you write `import { X } from '@acme/components'` instead of `import { X } from '../../../packages/ui/components'`. But the catch is that **the same problem must be solved three separate times** — once per tool:

```
                    import { X } from '@acme/components'
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              ▼               ▼               ▼
         TypeScript        webpack           Jest
              │               │               │
    tsconfig.base.json   webpack.config.ts  jest.config.js
         "paths": {        alias: {          moduleNameMapper: {
          "@acme/           {                  '@acme/components':
           components":      '@acme/            '<rootDir>/../../
           ["./packages/     components':        packages/ui/
            ui/components"]   './packages/       components'
         }                    ui/components'  }
                           }
              │               │               │
              ▼               ▼               ▼
       ./packages/      ./packages/      ./packages/
         ui/components/   ui/components/   ui/components/
```

All three arrows point to the same physical folder — but you have to configure it three times in three different files.

**The key insight**: TypeScript `paths` only helps TypeScript know where files *are*. It doesn't help webpack find them at bundle time, and it doesn't help Jest find them at test time. Each tool has its own resolution system.

#### 🖐️ Try it — Experience path aliases in a real project

Create a monorepo structure from scratch:

```bash
mkdir /tmp/monorepo-demo
cd /tmp/monorepo-demo
mkdir -p packages/ui/src apps/web/src
```

Create `packages/ui/src/index.ts`:
```typescript
export const greeting = (name: string) => `Hello, ${name}!`;
```

Create `apps/web/src/main.ts`:
```typescript
import { greeting } from '@my-monorepo/ui'; // path alias, not relative path
console.log(greeting('world'));
```

Create `tsconfig.base.json` in `/tmp/monorepo-demo`:
```json
{
  "compilerOptions": {
    "baseUrl": ".",
    "paths": {
      "@my-monorepo/ui": ["./packages/ui/src"]
    }
  }
}
```

Create `apps/web/tsconfig.json`:
```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "outDir": "./dist"
  },
  "include": ["src"]
}
```

Try compiling:
```bash
cd apps/web
npx tsc --project tsconfig.json --noEmit
```

TypeScript resolves the alias. Now try running the file without TypeScript:
```bash
node src/main.ts  # fails — Node.js doesn't know about the alias
```

Node.js can't use TypeScript `paths`. This is why each tool needs its own configuration.

#### 📚 Resources
- **Video**: search YouTube for `"TypeScript path aliases tsconfig paths tutorial"`
- **Article (TypeScript docs)**: [Module resolution — paths](https://www.typescriptlang.org/tsconfig#paths)

---

### 🧪 Concept 7: `transformIgnorePatterns` and `moduleNameMapper` in Jest

**`transformIgnorePatterns`** controls what Jest compiles. By default, Jest skips `node_modules` entirely (for speed). If a package ships `.ts`, you have to whitelist it:

```
Jest encounters:  import { UserProfile } from '@acme/ui/components'
                                                    │
                                                    ▼
                              Is this path in node_modules?
                                          │
                              ┌───────────┴────────────┐
                              │ YES                    │ NO
                              ▼                        ▼
            Is it in transformIgnorePatterns?    Compile it normally ✅
                      │
          ┌───────────┴────────────┐
          │ excluded (default)     │ whitelisted
          ▼                        ▼
    ❌ Skip compilation       Compile it ✅
       → fails on .ts files
```

**`moduleNameMapper`** is Jest's equivalent of TypeScript `paths`. It uses regex to intercept imports and redirect them:

```
Jest sees:  import { X } from '@acme/components'
                                       │
                                moduleNameMapper
                                       │
              '^@acme/components$'  →  '<rootDir>/../../packages/ui/components'
                                       │
                                       ▼
                           ./packages/ui/components/  ✅
```

A typical Jest config for a project consuming a TypeScript library:

```javascript
// jest.config.js
module.exports = {
  // Tell Jest to compile TypeScript from this specific package in node_modules
  transformIgnorePatterns: [
    'node_modules/(?!@acme/ui)'  // "skip all node_modules EXCEPT @acme/ui"
  ],

  // Map internal package names to their actual paths
  moduleNameMapper: {
    '^@acme/components$': '<rootDir>/node_modules/@acme/ui/components',
    '^@acme/types$': '<rootDir>/node_modules/@acme/ui/types',
    // one entry per internal sub-package
  }
};
```

After compilation (the fix), **none of this is needed** — Jest gets `.js` files from `dist/cjs/` via the `exports` field's `"require"` condition, just like any other package.

#### 🖐️ Try it — Set up a Jest project with moduleNameMapper

```bash
mkdir /tmp/jest-mapper-demo
cd /tmp/jest-mapper-demo
npm init -y
npm install --save-dev jest @types/jest ts-jest typescript
```

Create `src/greeting.ts`:
```typescript
export const greet = (name: string) => `Hello, ${name}!`;
```

Create `src/app.ts`:
```typescript
import { greet } from '@my-lib/greeting'; // will need mapping
export const run = () => greet('world');
```

Create `jest.config.js`:
```javascript
module.exports = {
  preset: 'ts-jest',
  moduleNameMapper: {
    '^@my-lib/greeting$': '<rootDir>/src/greeting.ts'
  }
};
```

Create `src/app.test.ts`:
```typescript
import { run } from './app';
test('runs', () => {
  expect(run()).toBe('Hello, world!');
});
```

```bash
npx jest
```

The test passes because `moduleNameMapper` intercepts `@my-lib/greeting` and redirects it to the actual file.

#### 📚 Resources
- **Article (Jest docs)**: [moduleNameMapper](https://jestjs.io/docs/configuration#modulenamemapper-objectstring-string--arraystring)
- **Article (Jest docs)**: [transformIgnorePatterns](https://jestjs.io/docs/configuration#transformignorepatterns-arraystring)

---

### 🔄 Concept 8: Self-referencing a package

"Self-referencing" means a package imports from itself using its own published name. Here's the before and after:

```
BEFORE:
─────────────────────────────────────────────────────────
Inside the library repo:               External consumer:

import { ColorScheme }                   import { UserProfile }
  from '@acme/types'                     from '@acme/ui/components'
       ▲                                           ▲
       │                                           │
  internal workspace name               published package name

→ Two different path schemes for inside vs outside.
  Documentation has to explain both. Confusing.


AFTER (self-referencing):
─────────────────────────────────────────────────────────
Inside the library repo:               External consumer:

import { ColorScheme }                   import { UserProfile }
  from '@acme/ui/types'                  from '@acme/ui/components'
       ▲                                           ▲
       │                                           │
  same path scheme ✅                    same path scheme ✅

→ One path scheme everywhere. Docs are simpler.
```

**The catch**: For self-referencing to work *inside the repo without building first*, TypeScript must know that `@acme/ui/types` → `./packages/ui/types/index.ts`. This requires adding new entries to `tsconfig.base.json`:

```
tsconfig.base.json today:              tsconfig.base.json needs:
─────────────────────────────          ──────────────────────────────────
"paths": {                             "paths": {
  "@acme/components": [...],  ✅         "@acme/ui/components": [...],  ← add this
  "@acme/types":      [...],  ✅         "@acme/ui/types":      [...],  ← add this
  "@acme/framework":  [...],  ✅         "@acme/ui/framework":  [...],  ← add this
}                                      }
```

Without these new entries, TypeScript compilation breaks the moment any file inside the library starts using the new self-referencing import paths.

#### 🖐️ Try it — Build a self-referencing package

Extend the package from Concept 5's exercise:

```bash
cd /tmp/my-package
mkdir -p src/utils
```

Create `src/utils/format.ts`:
```typescript
export const formatNumber = (n: number) => n.toFixed(2);
```

Now make `src/math.ts` import from itself using the package name (self-reference):
```typescript
import { formatNumber } from 'my-package/utils'; // self-referencing!
export function add(a: number, b: number): string {
  return formatNumber(a + b);
}
```

Update `package.json`:
```json
{
  "name": "my-package",
  "exports": {
    ".":       { "import": "./dist/esm/math.js",        "require": "./dist/cjs/math.js",        "types": "./dist/types/math.d.ts" },
    "./utils": { "import": "./dist/esm/utils/format.js", "require": "./dist/cjs/utils/format.js", "types": "./dist/types/utils/format.d.ts" }
  }
}
```

Add to `tsconfig.esm.json` (so TypeScript can find the self-reference during development):
```json
{
  "compilerOptions": {
    "paths": {
      "my-package/utils": ["./src/utils/format.ts"]
    }
  }
}
```

Compile:
```bash
npx tsc --project tsconfig.esm.json
npx tsc --project tsconfig.cjs.json
```

Without the `paths` entry, TypeScript would fail on the self-referencing import.

#### 📚 Resources
- **Article (Node.js docs)**: [Self-referencing a package](https://nodejs.org/api/packages.html#self-referencing-a-package-using-its-name)

---

### 🏗️ Concept 9: Monorepo workspace packages

In a monorepo, each sub-library gets its own `package.json` with a name like `@acme/types`. Yarn/npm workspaces stitches them together with symlinks:

```
my-monorepo/  (root)
├── package.json                ← root, declares workspaces
│     "workspaces": ["packages/*", "apps/*"]
│
├── packages/
│   ├── ui/
│   │   └── package.json        ← name: "@acme/ui"
│   ├── components/
│   │   └── package.json        ← name: "@acme/components"
│   └── types/
│       └── package.json        ← name: "@acme/types"
│
└── node_modules/
    └── @acme/
        ├── components  ────────symlink──────→ ../../packages/components/
        ├── types  ─────────────symlink──────→ ../../packages/types/
        └── ...
              ▲
              │
    Yarn created these symlinks
    automatically during `yarn install`
```

When code does `import X from '@acme/components'`, Node.js finds the symlink at `node_modules/@acme/components` and follows it back to `./packages/components/`. That's how monorepo packages resolve without being published to npm.

**The collapsing option**: Instead of 18 packages with 18 `package.json` files, consolidate into one root package with sub-folders:

```
BEFORE (many workspace packages):     AFTER (one package, sub-folders):
────────────────────────────────       ─────────────────────────────
packages/                              packages/
├── ui/package.json @acme/ui           ├── ui/package.json @acme/ui
├── components/                        ├── components/     (just a folder)
│   └── package.json @acme/components  │   └── index.ts
├── types/                             ├── types/          (just a folder)
│   └── package.json @acme/types       │   └── index.ts
└── ...                                └── ...
```

The sub-folders become sub-paths of `@acme/ui` (`@acme/ui/components`, `@acme/ui/types`) instead of separate packages.

#### 🖐️ Try it — Set up a Yarn workspaces monorepo from scratch

```bash
mkdir /tmp/yarn-workspace-demo
cd /tmp/yarn-workspace-demo
```

Create root `package.json`:
```json
{
  "name": "my-monorepo",
  "private": true,
  "workspaces": ["packages/*", "apps/*"]
}
```

Create two packages:
```bash
mkdir -p packages/math-utils packages/string-utils apps/web
```

`packages/math-utils/package.json`:
```json
{ "name": "@my-monorepo/math-utils", "version": "1.0.0", "main": "index.js" }
```

`packages/math-utils/index.js`:
```javascript
module.exports = { add: (a, b) => a + b };
```

`packages/string-utils/package.json`:
```json
{ "name": "@my-monorepo/string-utils", "version": "1.0.0", "main": "index.js",
  "dependencies": { "@my-monorepo/math-utils": "*" } }
```

`packages/string-utils/index.js`:
```javascript
const { add } = require('@my-monorepo/math-utils');
module.exports = { repeat: (s, n) => s.repeat(add(0, n)) };
```

`apps/web/package.json`:
```json
{ "name": "web-app", "version": "1.0.0",
  "dependencies": { "@my-monorepo/string-utils": "*" } }
```

Now install:
```bash
yarn install
```

Check the symlinks Yarn created:
```bash
ls -la node_modules/@my-monorepo/
# Shows symlinks to ../../packages/math-utils and ../../packages/string-utils

# Run something that uses the cross-package dependency:
node -e "const { repeat } = require('@my-monorepo/string-utils'); console.log(repeat('ha', 3));"
```

The packages resolve through symlinks. That's Yarn workspaces in action.

#### 📚 Resources
- **Video**: search YouTube for `"Yarn workspaces monorepo tutorial"` or `"npm workspaces explained"`
- **Article (Yarn docs)**: [Workspaces](https://yarnpkg.com/features/workspaces)
- **Article**: [monorepo.tools](https://monorepo.tools/) — independent guide covering all approaches

---

## Practice Project: Build a Dual-Format TypeScript Package

This project ties everything together. By the end, you'll have built exactly what `@acme/ui` is being redesigned to be.

### Goal

Build a TypeScript component utility library that:
- Ships both ESM and CJS
- Includes `.d.ts` type declarations
- Uses a proper `exports` field
- Can be consumed by a Vite app (ESM) and by Jest tests (CJS)
- Demonstrates tree-shaking

### Step 1 — Set up the library

```bash
mkdir /tmp/my-component-lib
cd /tmp/my-component-lib
npm init -y
npm install --save-dev typescript
```

Create `src/button.ts`:
```typescript
export type ButtonVariant = 'primary' | 'secondary' | 'danger';

export interface ButtonProps {
  label: string;
  variant?: ButtonVariant;
  disabled?: boolean;
}

export function createButton(props: ButtonProps): string {
  const { label, variant = 'primary', disabled = false } = props;
  return `<button class="btn btn-${variant}"${disabled ? ' disabled' : ''}>${label}</button>`;
}
```

Create `src/form.ts`:
```typescript
export interface FormField {
  name: string;
  type: 'text' | 'email' | 'number';
  required?: boolean;
}

export function createInput(field: FormField): string {
  return `<input type="${field.type}" name="${field.name}"${field.required ? ' required' : ''}>`;
}
```

Create `src/index.ts`:
```typescript
export * from './button';
export * from './form';
```

### Step 2 — Configure TypeScript for dual output

Create `tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2019",
    "strict": true,
    "declaration": true,
    "declarationDir": "./dist/types",
    "sourceMap": false
  },
  "include": ["src"]
}
```

Create `tsconfig.esm.json`:
```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "module": "ESNext",
    "moduleResolution": "bundler",
    "outDir": "./dist/esm"
  }
}
```

Create `tsconfig.cjs.json`:
```json
{
  "extends": "./tsconfig.base.json",
  "compilerOptions": {
    "module": "CommonJS",
    "moduleResolution": "node",
    "outDir": "./dist/cjs",
    "declaration": false
  }
}
```

### Step 3 — Wire up `package.json`

Update `package.json`:
```json
{
  "name": "my-component-lib",
  "version": "1.0.0",
  "exports": {
    ".": {
      "import":  "./dist/esm/index.js",
      "require": "./dist/cjs/index.js",
      "types":   "./dist/types/index.d.ts"
    },
    "./button": {
      "import":  "./dist/esm/button.js",
      "require": "./dist/cjs/button.js",
      "types":   "./dist/types/button.d.ts"
    },
    "./form": {
      "import":  "./dist/esm/form.js",
      "require": "./dist/cjs/form.js",
      "types":   "./dist/types/form.d.ts"
    }
  },
  "scripts": {
    "build:esm": "tsc --project tsconfig.esm.json",
    "build:cjs": "tsc --project tsconfig.cjs.json",
    "build": "npm run build:esm && npm run build:cjs"
  },
  "devDependencies": {
    "typescript": "^5.0.0"
  }
}
```

### Step 4 — Build

```bash
npm run build
ls dist/
# esm/  cjs/  types/
```

### Step 5 — Create a consumer that uses Jest (CJS path)

```bash
mkdir /tmp/my-app
cd /tmp/my-app
npm init -y
npm install --save-dev jest
```

In `package.json`, add the local package:
```json
"dependencies": {
  "my-component-lib": "file:/tmp/my-component-lib"
}
```

```bash
npm install
```

Create `test/button.test.js`:
```javascript
const { createButton } = require('my-component-lib');

test('creates a primary button', () => {
  const html = createButton({ label: 'Click me' });
  expect(html).toContain('btn-primary');
  expect(html).toContain('Click me');
});

test('creates a disabled button', () => {
  const html = createButton({ label: 'Save', disabled: true });
  expect(html).toContain('disabled');
});
```

```bash
npx jest
```

Jest uses the `"require"` condition from the `exports` field → picks up `dist/cjs/index.js`. Tests pass.

### Step 6 — Create a consumer that uses Vite (ESM path)

```bash
cd /tmp
npm create vite@latest vite-consumer -- --template vanilla
cd vite-consumer
npm install
npm install my-component-lib@file:/tmp/my-component-lib
```

Edit `src/main.js`:
```javascript
// Only importing from the button subpath — tests tree-shaking
import { createButton } from 'my-component-lib/button';

document.querySelector('#app').innerHTML = `
  <div>
    ${createButton({ label: 'Hello!', variant: 'primary' })}
    ${createButton({ label: 'Danger!', variant: 'danger' })}
  </div>
`;
```

```bash
npm run build
```

Vite uses the `"import"` condition → picks up `dist/esm/button.js`. Only the button module is bundled — the `form` module is tree-shaken away.

---

## Concepts Map

| Concept | What you built / observed |
| --- | --- |
| Source-only vs. compiled | Inspected `decimal.js` and `date-fns` to see compiled packages |
| CJS vs. ESM | Created `.cjs` and `.mjs` files, ran both with Node.js |
| Declaration files `.d.ts` | Generated one from scratch with `tsc --declaration` |
| Tree-shaking | Measured bundle size with targeted vs. wildcard imports in Vite |
| `exports` field | Built a full conditional `exports` field, consumed by Vite and Jest |
| Path aliases | Set up `tsconfig paths` and `moduleNameMapper` in Jest |
| `transformIgnorePatterns` | Understood how Jest decides what to compile |
| Self-referencing | Made a package import from its own published name |
| Monorepo workspaces | Created a Yarn workspace with symlinked cross-package dependencies |
