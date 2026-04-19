# CLI & Configuration Reference

## Installing the CLI

```bash
npm i -g @junobuild/cli
# or yarn global add @junobuild/cli
# or pnpm add -g @junobuild/cli
```

## Authentication

Required before any CLI command that interacts with a Satellite.

```bash
juno login                              # production
juno login --mode development           # local emulator
juno login --mode development --emulator  # shorthand: skips all steps, sets up local dev automatically
```

## Core CLI commands

### Emulator (local dev)

```bash
juno emulator start           # Start local environment (Skylab)
juno emulator stop
juno emulator clear           # Wipe local state
```

Requires Docker (default). For Podman, add to `juno.config`:

```ts
import { defineConfig } from "@junobuild/config";

export default defineConfig({
  satellite: {
    ids: {
      development: "<DEV_SATELLITE_ID>",
      production: "<PROD_SATELLITE_ID>"
    }
  },
  emulator: {
    runner: {
      type: "podman"
    },
    skylab: {}
  }
});
```

### Functions (serverless)

```bash
juno functions init            # Scaffold function files
juno functions init --lang ts   # TypeScript
juno functions init --lang rs   # Rust

juno functions build          # Compile functions (emulator reloads automatically)
```

**Deploying functions:**

```bash
# Direct deploy (requires admin access key)
juno functions upgrade

# CI/CD with editor access key: publish to CDN, then upgrade from CDN
juno functions publish                  # in CI
juno functions upgrade --cdn            # locally or in Console UI

# CI/CD with submit-only access key: publish, approve, then upgrade
juno functions publish --no-apply       # in CI — creates a pending change
juno changes list                       # review pending changes
juno changes apply --id <change_id>     # approve
juno functions upgrade --cdn-path <path>
```

Shortcut: `fn` works as alias for `functions` (e.g. `juno fn build`)

### Modes & profiles

```bash
--mode development    # Target local emulator
--mode staging
--mode production     # Default

--profile personal    # Switch between different console accounts/identities
--profile team
```

---

## juno.config structure

The config file lives at project root. Accepted names:

- `juno.config.ts` / `juno.config.js` / `juno.config.mjs` / `juno.config.json`

### Full example (TypeScript)

```ts
import { defineConfig } from "@junobuild/config";

export default defineConfig({
  satellite: {
    ids: {
      development: "<DEV_SATELLITE_ID>",
      production: "<PROD_SATELLITE_ID>"
    },
    hosting: {
      source: "dist",
      predeploy: ["npm run build"]
    }
  }
});
```

### Notes on `ids`

- `ids` is an object with environment keys. `production` is the default.
- The development Satellite is to be created by the developer in the local Console UI at `http://localhost:5866` and then its ID should be set as the `development` value.

### Environment-conditional config (function form)

`defineConfig` also accepts a function receiving `{ mode }`:

```ts
export default defineConfig(({ mode }) => ({
  satellite: {
    ids: { development: "<DEV_ID>", production: "<PROD_ID>" },
    ...(mode === "production" && {
      authentication: { google: { clientId: "your-client-id" } }
    })
  }
}));
```

### Applying configuration

Authentication, Datastore, Storage, and other settings can be configured either manually in the Console UI or via `juno.config`. Once updated in the file, apply to the Satellite:

```bash
juno config apply
juno config apply --mode development
```

### Orbiter config (Analytics)

```ts
export default defineConfig({
  satellite: {
    /* ... */
  },
  orbiter: {
    ids: {
      production: "aaaa-bbbbb-ccccc-ddddd-cai",
      development: "ffff-eeee-ddddd-ccccc-cai"
    }
  }
});
```

### Collection permissions

| Value        | Who can read/write                             |
| ------------ | ---------------------------------------------- |
| `public`     | Anyone                                         |
| `private`    | Only the owner (creator) of the document/asset |
| `managed`    | Owner, Satellite admin, and editors            |
| `restricted` | Only Satellite admin and editors               |

Collections can be configured in the Console UI or in `juno.config`:

```ts
satellite: {
  collections: {
    datastore: [
      {
        collection: "posts",
        memory: "stable",   // "stable" | "heap"
        read: "managed",
        write: "managed",
        mutablePermissions: true
      }
    ],
    storage: [
      {
        collection: "images",
        memory: "stable",
        read: "public",
        write: "managed",
        mutablePermissions: true
      }
    ]
  }
}
```

---

## Access keys (formerly "controllers")

Managed in the Console UI under **Satellite → Setup → Access keys**.

| Role        | Can do                                                       |
| ----------- | ------------------------------------------------------------ |
| `Admin`     | Everything, including stop/delete                            |
| `Editor`    | Deploy assets, publish functions. Good default for CI.       |
| `Submitter` | Propose changes only; a human must approve in Console or CLI |

GitHub Actions OIDC (recommended): no long-lived token stored in secrets. Configure allowed repos in Console → Satellite → Setup → Deployments.

---

## Deploying to production

Juno does not support SSR — frontend must be built as a static site (SSG). Use the official plugins to handle build output and environment variable injection automatically.

Configure the `source` in `juno.config` to match your framework's output directory:

| Framework | `source`                      |
| --------- | ----------------------------- |
| Next.js   | `out`                         |
| Angular   | `dist/<project-name>/browser` |
| Astro     | `dist`                        |
| React     | `dist`                        |
| SvelteKit | `build`                       |
| Vue       | `dist`                        |

```ts
satellite: {
  hosting: {
    source: "dist",
    predeploy: ["npm run build"]
  }
}
```

### GitHub Actions (recommended)

Juno uses GitHub OIDC — no secret tokens needed. Each workflow run gets short-lived credentials automatically.

**Prerequisites:**

1. Connect your repository in Console → Satellite → Deployments → Connect repository
2. Add `permissions: id-token: write` to your workflow

#### Deploy frontend

Uses `junobuild/juno-action@main` (lightweight):

```yaml
# .github/workflows/deploy.yml
name: Deploy to Juno
on:
  workflow_dispatch:
  push:
    branches: [main]
jobs:
  deploy:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: Check out the repo
        uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          registry-url: "https://registry.npmjs.org"
      - name: Install Dependencies
        run: npm ci
      - name: Deploy to Juno
        uses: junobuild/juno-action@main
        with:
          args: hosting deploy
```

#### Build and publish serverless functions

Uses `junobuild/juno-action@full` (includes Rust + TS build toolchain):

```yaml
# .github/workflows/publish.yml
name: Publish Serverless Functions
on:
  workflow_dispatch:
  push:
    branches: [main]
jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      id-token: write
    steps:
      - name: Check out the repo
        uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          registry-url: "https://registry.npmjs.org"
      - name: Install Dependencies
        run: npm ci
      - name: Build
        uses: junobuild/juno-action@full
        with:
          args: functions build
      - name: Publish
        uses: junobuild/juno-action@full
        with:
          args: functions publish
```

If the access key is an **editor**, changes deploy automatically. If it's a **submitter**, use `functions publish --no-apply` to create a pending change for manual approval.

### Plugins

#### Next.js plugin

```js
// next.config.js
import { withJuno } from "@junobuild/nextjs-plugin";
export default withJuno();
```

The plugin always ensures `output: "export"` is set. Injects `NEXT_PUBLIC_SATELLITE_ID`, `NEXT_PUBLIC_ORBITER_ID`, etc.

#### Vite plugin

```bash
npm i @junobuild/vite-plugin -D
```

```js
// vite.config.js
import juno from "@junobuild/vite-plugin";
export default defineConfig({
  plugins: [juno()]
});
```

Injects `VITE_SATELLITE_ID`, `VITE_ORBITER_ID`, etc.

### CLI

```bash
juno hosting deploy     # deploy frontend assets
juno functions upgrade  # deploy serverless functions
```
