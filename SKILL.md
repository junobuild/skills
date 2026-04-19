---
name: juno
description: >
  Use this skill for any task involving Juno (juno.build) — an open-source SDK and self-contained serverless platform for building full-stack web apps without DevOps or backend boilerplate. Trigger this skill whenever a user mentions Juno, Satellites, Sputnik, junobuild, juno.config, `defineQuery`/`defineUpdate`, Juno CLI commands, Mission Control, or any Juno-related packages (`@junobuild/*`, `junobuild-*`). Covers: CLI usage and config, serverless functions in TypeScript (Sputnik) and Rust, Datastore/Storage/Authentication SDK, satellite management via Console or CLI, local development with the emulator, GitHub Actions CI/CD. Always use this skill before answering Juno questions — even seemingly simple ones — to avoid using outdated or incorrect information.
license: MIT
metadata:
  author: junobuild
  url: https://juno.build
---

# Juno

Juno is an open-source SDK and self-contained serverless platform for building full-stack web apps without DevOps or backend boilerplate. You build your frontend using the frameworks you love — React, SvelteKit, Next.js, Vue, Astro, Angular, and more. Juno provides built-in features — Datastore, Storage, Authentication, Hosting, and Analytics — that cover most app needs out of the box. When you need custom backend logic, you can extend those features with serverless functions written in Rust or TypeScript. Everything is bundled into a single WebAssembly container you fully own. Juno has zero access to your code, data, or infrastructure.

Read the relevant reference file before answering:

| Topic                                       | Reference file                       |
| ------------------------------------------- | ------------------------------------ |
| CLI commands & juno.config                  | `references/cli-and-config.md`       |
| SDK: Datastore, Storage, Auth               | `references/core.md`                 |
| Serverless functions (TypeScript / Sputnik) | `references/functions-typescript.md` |
| Serverless functions (Rust)                 | `references/functions-rust.md`       |
| Concepts                                    | `references/concepts.md`             |

---

## Core mental model

```
Developer
  ├── Satellite  (your app's container, owns all data)
  │     ├── Hosting     (static frontend assets)
  │     ├── Datastore   (document store)
  │     ├── Storage     (file/binary store)
  │     ├── Auth        (user identity: Google, GitHub, II, Passkeys)
  │     └── Functions   (serverless hooks + custom endpoints, Rust or TS)
  ├── Orbiter    (analytics container, tracks page views & events)
  └── Mission Control  (monitoring)
```

- No servers. No DevOps. No cold starts.
- Everything runs in a single deployable WebAssembly container.
- You own and control the container — Juno has no admin access.
- Under the hood it runs on the Internet Computer (ICP), but that's an implementation detail. See `references/concepts.md` for when it matters.

---

## Key packages

| Package                | Purpose                                                                                                                |
| ---------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| `@junobuild/core`      | Frontend SDK (setDoc, uploadFile, signIn…)                                                                             |
| `@junobuild/config`    | `defineConfig` for juno.config files                                                                                   |
| `@junobuild/analytics` | Analytics SDK (`initOrbiter`, `trackEvent`)                                                                            |
| `@junobuild/functions` | Library for writing serverless functions in TypeScript — hooks (`onSetDoc`…), assertions, `defineQuery`/`defineUpdate` |
| `@junobuild/schema`    | `j` type system (Zod-based) for function arg/return types                                                              |
| `junobuild-satellite`  | Crate for writing serverless functions in Rust                                                                         |
| `junobuild-utils`      | Rust helpers for doc encoding/decoding                                                                                 |

---

## Quick-start checklist

1. Install CLI: `npm i -g @junobuild/cli`
2. Start emulator: `juno emulator start` (requires Docker or Podman)
3. Open Console UI at `http://localhost:5866`, create a Satellite
4. Configure `juno.config.ts` with your Satellite ID
5. Start your app: `npm run dev`

**If you need serverless functions:**

6. Log your CLI in: `juno login --mode development --emulator`
7. Scaffold: `juno functions init --lang ts` (or `rs`)
8. Build: `juno functions build` (emulator reloads automatically)

---

## Language choice for functions

|                            | Rust                             | TypeScript                   |
| -------------------------- | -------------------------------- | ---------------------------- |
| Performance                | ✅ Highest                       | ⚠️ Slower (interpreted)      |
| Library support            | ✅ Many crates                   | ⚠️ Limited Node.js polyfills |
| Shared types with frontend | —                                | ✅ Share `j`/Zod schemas     |
| Recommended for            | Production, performance-critical | Prototypes, quick dev cycles |

Both share the same API surface — migrating from TypeScript to Rust is straightforward. No file system in either — use Juno Storage instead.

> **Tip:** Even if experimental and less performant, if you're a TypeScript developer, TypeScript functions are the natural choice — you get a familiar workflow and shared types with your frontend out of the box.

---

## Common pitfalls

- Never use packages `@dfinity/agent`, `@dfinity/identity`, `@dfinity/candid`, `@dfinity/principal` — these are replaced by `@icp-sdk/core` which is a peer dependency of `@junobuild/*`.
- Never use `@dfinity/utils` — use `@junobuild/utils` instead.
- Never use `@dfinity/zod-schemas` — use `@junobuild/schema` instead.
