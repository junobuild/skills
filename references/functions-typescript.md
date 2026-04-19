# Serverless Functions — TypeScript (Sputnik)

## Setup

```bash
juno functions init --lang ts
```

Scaffolds an `index.ts` entry file in your project.

```bash
juno functions build   # compile
```

When the emulator is running and a Satellite exists with its ID configured in `juno.config`, `juno functions build` automatically upgrades the local Satellite — no manual redeploy needed.

---

## Upgrade

> ⚠️ **Never upgrade a Satellite with custom functions through the Console UI** — it replaces your Satellite with the stock version, overwriting your custom code. When the Console notifies you of a new version, pull the latest Docker image and redeploy via CLI instead. Always upgrade iteratively — do not skip versions.

The entire toolchain ships inside the emulator Docker image. To stay up to date, pull the latest image. If a release also includes JS library updates, bump the relevant packages in your project.

### Versioning

The version embedded in your compiled Wasm is read from `package.json`. You can optionally version functions independently:

```json
{
  "name": "my-app",
  "version": "0.0.10",
  "juno": {
    "functions": {
      "version": "0.0.4"
    }
  }
}
```

---

## Hooks

Hooks run **asynchronously** after an operation completes. They don't block the client response.

```ts
import { defineHook, type OnSetDoc } from "@junobuild/functions";
import { decodeDocData, encodeDocData, setDocStore } from "@junobuild/functions/sdk";

interface Person {
  hello: string;
}

export const onSetDoc = defineHook<OnSetDoc>({
  collections: ["posts"],
  run: async (context) => {
    const data = decodeDocData<Person>(context.data.data.after.data);
    const updated = { hello: `${data.hello} checked` };

    await setDocStore({
      caller: context.caller,
      collection: context.data.collection,
      key: context.data.key,
      doc: {
        data: encodeDocData(updated),
        description: context.data.data.after.description,
        version: context.data.data.after.version
      }
    });
  }
});
```

### Available hooks

| Hook type            | Triggers when                             |
| -------------------- | ----------------------------------------- |
| `OnSetDoc`           | A document is created or updated          |
| `OnSetManyDocs`      | Multiple documents are created or updated |
| `OnDeleteDoc`        | A document is deleted                     |
| `OnDeleteManyDocs`   | Multiple docs are deleted                 |
| `OnUploadAsset`      | An asset is uploaded                      |
| `OnDeleteAsset`      | An asset is deleted                       |
| `OnDeleteManyAssets` | Multiple assets are deleted               |

All accept `{ collections: string[], run: async (context) => void }`.

---

## Assertions

Run **synchronously** before an operation. Throw to reject.

```ts
import { defineAssert, type AssertSetDoc } from "@junobuild/functions";
import { decodeDocData } from "@junobuild/functions/sdk";

interface NoteData {
  text: string;
}

export const assertSetDoc = defineAssert<AssertSetDoc>({
  collections: ["posts"],
  assert: (context) => {
    const data = decodeDocData<NoteData>(context.data.data.proposed.data);
    if (!data.title) {
      throw new Error("Title is required");
    }
  }
});
```

Zod is bundled with `@junobuild/functions` — you can use it directly in assertions:

```ts
import { z } from "zod";

const schema = z.object({ title: z.string().min(1) });

export const assertSetDoc = defineAssert<AssertSetDoc>({
  collections: ["posts"],
  assert: (context) => {
    const data = decodeDocData<NoteData>(context.data.data.proposed.data);
    schema.parse(data);
  }
});
```

| Assert type         | Guards                   |
| ------------------- | ------------------------ |
| `AssertSetDoc`      | Before doc create/update |
| `AssertDeleteDoc`   | Before doc delete        |
| `AssertUploadAsset` | Before asset upload      |
| `AssertDeleteAsset` | Before asset delete      |

---

## Custom endpoints: defineQuery / defineUpdate

Callable functions explicitly invoked from the frontend (not event-triggered):

```ts
import { defineUpdate, defineQuery } from "@junobuild/functions";
import { j } from "@junobuild/schema";

const PostSchema = j.strictObject({
  title: j.string(),
  authorId: j.principal()
});

export const createPost = defineUpdate({
  args: PostSchema,
  returns: PostSchema,
  handler: async ({ args }) => {
    // write logic
    return args;
  }
});

export const getPost = defineQuery({
  args: j.strictObject({ id: j.string() }),
  returns: j.nullable(PostSchema),
  handler: async ({ args }) => {
    // read-only logic
    return null;
  }
});
```

### `j` type system (from `@junobuild/schema`)

`j` extends Zod. Validated at runtime and type-safe at build time. Candid bindings and a frontend API are auto-generated.

| Type                        | Description            |
| --------------------------- | ---------------------- |
| `j.string()`                | String                 |
| `j.number()`                | Number                 |
| `j.boolean()`               | Boolean                |
| `j.principal()`             | ICP Principal          |
| `j.uint8array()`            | Uint8Array             |
| `j.strictObject({})`        | Object (no extra keys) |
| `j.array(schema)`           | Array                  |
| `j.nullable(schema)`        | Optional / null        |
| `j.discriminatedUnion(...)` | Discriminated union    |

> ⚠️ `j.union()` is not supported — use `j.discriminatedUnion()` instead.

### Calling from frontend

```ts
import { functions } from "../declarations/satellite/satellite.api.ts";

await functions.createPost({ title: "Hello", authorId: Principal.anonymous() });
```

---

## Datastore access inside functions

Import from `@junobuild/functions/sdk`:

```ts
import { getDocStore, setDocStore, listDocsStore, deleteDocStore } from "@junobuild/functions/sdk";

// Get
const doc = getDocStore({ caller, collection: "posts", key: "my-key" });

// Set
setDocStore({
  caller,
  collection: "posts",
  key: "my-key",
  doc: {
    data: encodeDocData({ title: "Hello" }),
    description: undefined,
    version: doc?.version
  }
});

// List
const { items } = listDocsStore({ caller, collection: "posts", params: {} });

// Delete
deleteDocStore({ caller, collection: "posts", key: "my-key", doc: { version: undefined } });
```

Also available: `countDocsStore`, `countCollectionDocsStore`, `deleteDocsStore`, `deleteFilteredDocsStore`.

Full reference: [juno.build/docs/reference/functions/typescript/sdk](https://juno.build/docs/reference/functions/typescript/sdk)

---

## Storage access inside functions

```ts
import { setAssetHandler, getAssetStore, deleteAssetStore } from "@junobuild/functions/sdk";

setAssetHandler({
  key: {
    name: "output.json",
    full_path: "/data/output.json",
    collection: "data",
    owner: canisterSelf()
  },
  content: new TextEncoder().encode(JSON.stringify(data)),
  headers: [["Content-Type", "application/json"]]
});
```

Also available: `listAssetsStore`, `countAssetsStore`, `deleteAssetsStore`, `deleteFilteredAssetsStore`, `setAssetTokenStore`.

Full reference: [juno.build/docs/reference/functions/typescript/sdk](https://juno.build/docs/reference/functions/typescript/sdk)

---

## Encoding / decoding document data

Import from `@junobuild/functions/sdk`:

```ts
import { decodeDocData, encodeDocData } from "@junobuild/functions/sdk";

const decoded = decodeDocData<MyType>(doc.data);
const encoded: Uint8Array = encodeDocData({ title: "Hello" });
```

---

## IC-CDK utilities

Import from `@junobuild/functions/ic-cdk`:

```ts
import { canisterSelf, msgCaller, time } from "@junobuild/functions/ic-cdk";

// Principal ID of the current Satellite — useful as caller for admin operations in hooks
const satelliteId = canisterSelf();

// Principal ID of the caller of the current function
const caller = msgCaller();
// Note: in hooks, the caller is the Satellite itself — get the user's Principal from context.caller

// Current IC timestamp in nanoseconds
const now = time();
```

### call — inter-canister calls

```ts
import { call } from "@junobuild/functions/ic-cdk";
import { IDL } from "@icp-sdk/core/candid";
import { Principal } from "@icp-sdk/core/principal";

// Define Candid types for encoding/decoding
const Account = IDL.Record({
  owner: IDL.Principal,
  subaccount: IDL.Opt(IDL.Vec(IDL.Nat8))
});
const Icrc1Tokens = IDL.Nat;

const balance = await call<bigint>({
  canisterId: Principal.fromText("ryjl3-tyaaa-aaaaa-aaaba-cai"),
  method: "icrc1_balance_of",
  args: [[Account, { owner: Principal.from(context.caller), subaccount: [] }]],
  result: Icrc1Tokens
});
```

> **Note:** For well-known IC canisters, use the pre-built classes in `@junobuild/functions/canisters/*` — no IDL needed. `call()` with raw `@dfinity/candid` IDL is the escape hatch for canisters that don't have a pre-built class.

### httpRequest — HTTPS outcalls

```ts
import { httpRequest } from "@junobuild/functions/ic-cdk";

const response = await httpRequest({
  url: "https://api.example.com/data",
  method: "GET",
  isReplicated: false // single-node, cheaper; omit for verified consensus (all nodes must agree)
});

const body = new TextDecoder().decode(response.body);
const json = JSON.parse(body);
```

`httpRequest` options:

| Option             | Description                                                                          |
| ------------------ | ------------------------------------------------------------------------------------ |
| `url`              | The target URL                                                                       |
| `method`           | `"GET"`, `"POST"`, or `"HEAD"`                                                       |
| `headers`          | Optional request headers                                                             |
| `body`             | Optional `Uint8Array` request body                                                   |
| `maxResponseBytes` | Optional max response size in bytes                                                  |
| `isReplicated`     | `false` = single-node (cheaper); omit = all nodes must agree on response             |
| `transform`        | Optional name of a `defineQuery` function to transform the response before consensus |

---

## Canister integrations

Pre-built classes for well-known IC canisters — no Candid or IDL needed:

```ts
// ICP Ledger
import { IcpLedgerCanister } from "@junobuild/functions/canisters/ledger/icp";

const ledger = new IcpLedgerCanister();
const result = await ledger.transfer({
    args: {
        to: destinationAccountIdentifier, // Uint8Array
        amount: { e8s: 100_000_000n },    // 1 ICP
        fee: { e8s: 10_000n },
        memo: 0n
    }
});

// ICRC Ledger (ckBTC, ckETH, etc.)
import { IcrcLedgerCanister } from "@junobuild/functions/canisters/ledger/icrc";

const icrc = new IcrcLedgerCanister({ canisterId: "your-icrc-ledger-id" });
const balance = await icrc.icrc1BalanceOf({ account: { owner: Principal.fromText("...") } });
await icrc.icrc1Transfer({ args: { to: { owner: ... }, amount: 1_000_000n } });
await icrc.icrc2TransferFrom({ args: { from: ..., to: ..., amount: ... } });
await icrc.icrc2Approve({ args: { spender: ..., amount: ... } });

// Cycle Minting Canister
import { CMCCanister } from "@junobuild/functions/canisters/cmc";

const cmc = new CMCCanister();
await cmc.notifyTopUp({ args: { block_index: blockIndex, canister_id: Principal.fromText("...") } });
```

For canisters without a pre-built class, declarations-only exports are available for use with `call()`:

```ts
// Declarations only — use with call()
import {
  IcManagementIdl,
  type IcManagementDid
} from "@junobuild/functions/canisters/ic-management";
import { NnsGovernanceIdl } from "@junobuild/functions/canisters/nns";
import { CkBTCMinterIdl } from "@junobuild/functions/canisters/ckbtc";
// also: cketh, sns
```

Full reference: [juno.build/docs/reference/functions/typescript/canisters](https://juno.build/docs/reference/functions/typescript/canisters)

---

## Node.js compatibility

Not all Node.js APIs are available. What works:

- `console.log` / `console.info` — fully supported, output visible in Satellite logs and Console UI
- `Math.random()` — supported but seeded once after upgrade, not suitable for lotteries or unpredictable randomness

No `fs`, `http`, `crypto` from Node, and many npm packages that rely on those will not work. Missing polyfills are added iteratively — open an issue if you're blocked.

---

## Limitations

- Not all Node.js polyfills — many npm packages won't work
- TypeScript executes slower than Rust (interpreted via rquickjs)
- Async is supported but beware of ICP instruction limits per call
