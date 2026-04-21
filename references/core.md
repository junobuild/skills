# SDK Reference — Datastore, Storage, Authentication

Package: `@junobuild/core`

## Installation

```bash
npm i @junobuild/core
# or
yarn add @junobuild/core @junobuild/utils @icp-sdk/core @icp-sdk/auth
# or
pnpm add @junobuild/core @junobuild/utils @icp-sdk/core @icp-sdk/auth
```

## Initialization

```ts
import { initSatellite } from "@junobuild/core";

await initSatellite();
```

The plugin (`@junobuild/vite-plugin` or `@junobuild/nextjs-plugin`) injects the Satellite ID automatically. Call `initSatellite()` once at the top of your app.

---

## Authentication

Juno supports four sign-in providers. The `signIn()` call takes a typed provider object — not a bare call.

```ts
import { signIn, signUp, signOut, onAuthStateChange } from "@junobuild/core";

// Google (requires Client ID configured in juno.config + Console)
await signIn({ google: {} });

// GitHub (requires self-hosted Juno API proxy)
await signIn({
  github: {
    redirect: {
      clientId: "your-github-client-id",
      redirectUrl: "https://example.com/auth/callback/github",
      initUrl: "https://your-api.example.com/v1/auth/init/github",
      finalizeUrl: "https://your-api.example.com/v1/auth/finalize/github"
    }
  }
});

// Internet Identity
await signIn({ internet_identity: {} });

// Passkeys (WebAuthn / Face ID / Touch ID)
// New users must sign up first, returning users sign in
await signUp({ webauthn: {} });
await signIn({ webauthn: {} });

// Local dev only — recommended for local development, blocked in production
await signIn({ dev: {} });
// ⚡ For local development, this is the easiest option — no setup, no OAuth, instant sign-in.
```

After OAuth redirects (Google, GitHub), handle the callback:

```ts
import { handleRedirectCallback } from "@junobuild/core";
await handleRedirectCallback({ google: null }); // or { github: null }
```

Use `signOut` to end the user's session regardless of which provider they used:

```ts
await signOut();
```

Use `onAuthStateChange` to reactively track the user's authentication state — call it once at the top of your app:

```ts
const unsubscribe = onAuthStateChange((user) => {
  if (user) {
    console.log("Logged in as:", user.key); // user.key is the Principal as string
  } else {
    console.log("Logged out");
  }
});

// Stop listening when no longer needed
unsubscribe();
```

### Provider summary

| Provider          | Setup needed                                                          | Domain-scoped identity |
| ----------------- | --------------------------------------------------------------------- | ---------------------- |
| Google            | Client ID in juno.config + Console                                    | ❌ No                  |
| GitHub            | Self-hosted proxy ([junobuild/api](https://github.com/junobuild/api)) | ❌ No                  |
| Internet Identity | None                                                                  | ✅ Yes                 |
| Passkeys          | None — requires `signUp` + `signIn`                                   | ✅ Yes (hostname)      |

> **Note:** Sessions cannot be extended or refreshed. Once a session expires, the user must sign in again.

> **Local development with Google auth:** The Google Client ID must be configured once in the local Console UI at `http://localhost:5866` — this triggers the emulator to start fetching the Google public JWKS required for token verification.

### Authentication config in juno.config

Google and GitHub require a `clientId` configured in `juno.config` under `satellite.authentication`:

```ts
satellite: {
  authentication: {
    google: { clientId: "1234567890-abc.apps.googleusercontent.com" },
    // OR
    github: { clientId: "your-github-client-id" }
  }
}
```

Each method can optionally be configured with a delegation object. For example:

```ts
google: {
    clientId: "1234567890-abc.apps.googleusercontent.com",
    delegation: {
        allowedTargets: ["<SATELLITE_ID>"],  // null = allow usage for calling any canister (use with caution)
        sessionDuration: BigInt(7 * 24 * 60 * 60 * 1_000_000_000)  // e.g. 7 days, max 30 days
    }
}
```

After updating `juno.config`, apply to the Satellite backend:

```bash
juno config apply
juno config apply --mode development
```

---

## Datastore vs Storage

| Feature                | Datastore                        | Storage                               |
| ---------------------- | -------------------------------- | ------------------------------------- |
| Use case               | App state, user profiles, config | Images, files, user-generated content |
| Data format            | JSON-like documents              | Binary files                          |
| Identifier             | `key` (string you define)        | `fullPath` (auto or custom)           |
| Accessible via web URL | No                               | Yes                                   |
| Size limit             | Max 2 MB per document            | No specific limit                     |

- **`key`**: unique string within a collection — commonly a UUID, nanoid, or meaningful slug like `user:42`
- **`fullPath`**: the full path of an asset that builds a valid web URL — e.g. `/images/logo.png` (collection: `images`). Auto-generated from filename unless overridden.

Assets in Storage are served directly on the web at:

```
https://<satellite-id>.icp0.io<fullPath>
# e.g. https://qsgjb-riaaa-aaaaa-aaaga-cai.icp0.io/images/logo.png
```

> ⚠️ Assets in Storage are **publicly accessible** unless you use the `token` option to make the URL hard to guess.

---

## Datastore

Use for **structured data** (documents). Not for binary files — use Storage for those.

Each document has:

- `key`: unique string within collection
- `data`: any JSON-serializable payload
- `description`: optional string (max 1024 chars) for filtering
- `version`: optimistic concurrency token (required on updates)
- `owner`: the user that created it (auto-set)
- `created_at` / `updated_at`: nanosecond timestamps

### setDoc — create or update

`setDoc` creates or updates a document. When updating, `version` must be provided to prevent conflicts.

```ts
import { setDoc } from "@junobuild/core";

// Create
await setDoc({
  collection: "posts",
  doc: {
    key: crypto.randomUUID(),
    data: { title: "Hello", body: "World" }
  }
});

// Update — spread the existing doc to carry over key and version
await setDoc({
  collection: "posts",
  doc: {
    ...existingDoc,
    data: { title: "Updated" }
  }
});
```

### getDoc

Retrieve a single document by key:

```ts
import { getDoc } from "@junobuild/core";

const doc = await getDoc<Post>({ collection: "posts", key: "my-key" });
// doc is null if not found
```

### getManyDocs

Fetch multiple documents at once (can span collections):

```ts
import { getManyDocs } from "@junobuild/core";

const docs = await getManyDocs({
  docs: [
    { collection: "posts", key: "key-1" },
    { collection: "comments", key: "key-2" }
  ]
});
```

### listDocs

List and filter documents in a collection:

```ts
import { listDocs } from "@junobuild/core";

const { items, items_length, matches_length } = await listDocs<Post>({
  collection: "posts",
  filter: {
    order: { field: "updated_at", desc: true }, // "keys" | "updated_at" | "created_at"
    paginate: { limit: 10, startAfter: lastKey },
    owner: userPrincipal, // filter to a specific user's docs
    matcher: {
      key: "^doc_", // regex on key
      description: "example", // regex on description
      createdAt: { matcher: "greaterThan", timestamp: 1627776000n },
      updatedAt: { matcher: "between", timestamps: { start: 1627770000n, end: 1627900000n } }
    }
  }
});
// items: document array
// items_length: count of returned items
// matches_length: total matching count (for pagination)
```

### countDocs

Count documents without fetching them — accepts same filter as `listDocs`:

```ts
import { countDocs } from "@junobuild/core";

const count = await countDocs({ collection: "posts" });
```

### setManyDocs

Set multiple documents atomically across collections — if any fails, all are reverted:

```ts
import { setManyDocs } from "@junobuild/core";

await setManyDocs({
  docs: [
    { collection: "posts", doc: { key: "key-1", data: { title: "Hello" } } },
    { collection: "comments", doc: { key: "key-2", data: { body: "Nice" } } }
  ]
});
```

### deleteDoc

Delete a single document — version is validated to prevent concurrent conflicts:

```ts
import { deleteDoc } from "@junobuild/core";

await deleteDoc({ collection: "posts", doc: existingDoc });
```

### deleteManyDocs

Delete multiple documents atomically:

```ts
import { deleteManyDocs } from "@junobuild/core";

await deleteManyDocs({ docs: [myDoc1, myDoc2, myDoc3] });
```

### deleteFilteredDocs

Delete all documents matching a filter (same options as `listDocs`):

```ts
import { deleteFilteredDocs } from "@junobuild/core";

await deleteFilteredDocs({
  collection: "posts",
  filter: { matcher: { key: "^draft_" } }
});
```

---

## Storage

Use for **binary files** (images, PDFs, JSON, etc.).

Assets are identified by `fullPath` — always starts with `/`, structured as `/collection/filename`. Uploading to an existing `fullPath` **overwrites** it.

### uploadFile

Upload a `File` or `Blob` selected from an `<input type="file">`:

```ts
import { uploadFile } from "@junobuild/core";

const result = await uploadFile({
  data: file, // File | Blob
  collection: "images",
  filename: "custom-name.jpg", // optional, overrides file.name
  headers: [["Cache-Control", "max-age=3600"]]
});
// result.fullPath → "/images/custom-name.jpg"
// result.downloadUrl → public URL to access the asset
```

### uploadBlob

Like `uploadFile` but for raw binary data — `filename` must be provided explicitly:

```ts
import { uploadBlob } from "@junobuild/core";

const result = await uploadBlob({
  data: new Blob([myBuffer]),
  filename: "generated.json",
  collection: "data"
});
```

### Protected assets

Make an asset URL hard to guess by adding an access token:

```ts
import { uploadFile } from "@junobuild/core";
import { nanoid } from "nanoid";

const result = await uploadFile({
  data: file,
  collection: "images",
  token: nanoid()
});
// Asset is only accessible at /images/file.jpg?token=<secret>
```

### downloadUrl

Generate a public URL for an asset on the fly:

```ts
import { downloadUrl } from "@junobuild/core";

const url = downloadUrl({
  assetKey: {
    fullPath: "/images/logo.png",
    token: "a-secret-token" // required if asset was uploaded with a token
  }
});
// Use in <img src={url} /> or <a href={url}>
```

### getAsset

Retrieve asset metadata:

```ts
import { getAsset } from "@junobuild/core";

const asset = await getAsset({
  collection: "images",
  fullPath: "/images/logo.png"
});
```

### listAssets

List and filter assets in a collection — same filter interface as `listDocs` (`matcher.key` filters on `fullPath`):

```ts
import { listAssets } from "@junobuild/core";

const { items, items_length, matches_length } = await listAssets({
  collection: "images",
  filter: {
    order: { field: "updated_at", desc: true },
    paginate: { limit: 20 },
    matcher: { key: ".*.png$" } // regex on fullPath
  }
});
```

### countAssets

Count assets without fetching them — accepts same filter as `listAssets`:

```ts
import { countAssets } from "@junobuild/core";

const count = await countAssets({ collection: "images" });
```

### deleteAsset

Delete an asset by providing the existing asset object:

```ts
import { deleteAsset } from "@junobuild/core";

await deleteAsset({ collection: "images", storageFile: myAsset });
// No version check needed for assets
```

### deleteManyAssets

Delete multiple assets atomically:

```ts
import { deleteManyAssets } from "@junobuild/core";

await deleteManyAssets({
  assets: [
    { collection: "images", fullPath: "/images/old.jpg" },
    { collection: "data", fullPath: "/data/temp.json" }
  ]
});
```

### deleteFilteredAssets

Delete all assets matching a filter (same options as `listAssets`):

```ts
import { deleteFilteredAssets } from "@junobuild/core";

await deleteFilteredAssets({
  collection: "images",
  filter: { matcher: { key: "^/images/thumb_" } }
});
```

---

## Collection permission model

Permissions are set per collection (in Console UI or via `juno.config`):

| Level        | Who can access                                 |
| ------------ | ---------------------------------------------- |
| `public`     | Anyone                                         |
| `private`    | Only the owner (creator) of the document/asset |
| `managed`    | Owner, Satellite admin, and editors            |
| `restricted` | Only Satellite admin and editors               |

---

## Memory types

| Memory   | Survives upgrade           | Good for               |
| -------- | -------------------------- | ---------------------- |
| `stable` | ✅ Yes                     | Most use cases         |
| `heap`   | ❌ No (cleared on upgrade) | Ephemeral / cache data |

Recommendation: use `stable` unless you have a specific reason not to.

---

## Collection limits

- Max document size: **2 MB**
- For binary files > 2 MB, use Storage
- Satellite has overall memory limits (check docs for current values)
- Optional per-collection limit: `max_changes_per_user`
