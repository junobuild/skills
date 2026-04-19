# Serverless Functions — Rust

## Setup

```bash
juno functions init --lang rs
```

Scaffolds `src/satellite/src/lib.rs` and `src/satellite/Cargo.toml`.

```bash
juno functions build   # compile
```

When the emulator is running and a Satellite exists with its ID configured in `juno.config`, `juno functions build` automatically upgrades the local Satellite — no manual redeploy needed.

---

## Cargo.toml dependencies

> ⚠️ Always check [juno.build/docs/reference/functions/rust/crate-versions](https://juno.build/docs/reference/functions/rust/crate-versions) or [github.com/junobuild/juno/releases](https://github.com/junobuild/juno/releases) for the latest versions.

Current versions (Juno v0.0.72):

```toml
[dependencies]
candid = "0.10.20"
ic-cdk = "0.19.0"
ic-cdk-macros = "0.19.0"
serde = "1.0.225"
serde_cbor = "0.11.2"
junobuild-satellite = "0.6.0"
junobuild-macros = "0.4.0"
junobuild-utils = "0.4.0"

[lib]
crate-type = ["cdylib"]
```

### Feature selection (optional)

By default, `juno functions init` scaffolds all hooks and assertions, and all must be implemented. If you only need specific ones, disable default features to opt out of the rest:

```toml
junobuild-satellite = { version = "0.6.0", default-features = false, features = ["on_set_doc", "assert_set_doc"] }
```

With this, only `on_set_doc` and `assert_set_doc` need to be implemented — other hooks are not included.

---

## include_satellite!()

Always call `include_satellite!()` at the end of `lib.rs` — it wires in the default Satellite runtime and exposes the necessary endpoints for the Console and CLI.

> ⚠️ **Never upgrade a Satellite with custom functions through the Console UI** — it replaces your Satellite with the stock version, overwriting your custom code. When the Console notifies you of a new version, bump your Cargo.toml dependencies and redeploy via CLI instead. Always upgrade iteratively — do not skip versions.

---

## Hooks

Hooks run **asynchronously** after an operation. Use proc macros from `junobuild-macros`.

```rust
use junobuild_macros::on_set_doc;
use junobuild_satellite::{include_satellite, OnSetDocContext};

#[on_set_doc(collections = ["posts"])]
async fn on_set_doc(context: OnSetDocContext) -> Result<(), String> {
    // context.data.data.after is the new document
    // context.data.data.before is Option<Doc>
    ic_cdk::println!("Doc set: {}", context.data.key);
    Ok(())
}

include_satellite!();
```

### Available hook macros

| Macro                      | Event                   |
| -------------------------- | ----------------------- |
| `#[on_set_doc]`            | Doc created or updated  |
| `#[on_set_many_docs]`      | Batch doc create/update |
| `#[on_delete_doc]`         | Doc deleted             |
| `#[on_delete_many_docs]`   | Batch doc delete        |
| `#[on_upload_asset]`       | Asset uploaded          |
| `#[on_delete_asset]`       | Asset deleted           |
| `#[on_delete_many_assets]` | Batch asset delete      |

All accept an optional `collections = ["name1", "name2"]` filter. Omit to catch all collections.

---

## Assertions

Run **synchronously** before an operation. Return `Err(String)` to reject.

```rust
use junobuild_macros::assert_set_doc;
use junobuild_satellite::AssertSetDocContext;
use junobuild_utils::decode_doc_data;

#[assert_set_doc(collections = ["posts"])]
fn assert_set_doc(context: AssertSetDocContext) -> Result<(), String> {
    let data: MyStruct = decode_doc_data(&context.data.data.proposed.data)?;

    if data.title.is_empty() {
        return Err("Title is required".to_string());
    }

    Ok(())
}
```

| Macro                    | Guards                   |
| ------------------------ | ------------------------ |
| `#[assert_set_doc]`      | Before doc create/update |
| `#[assert_delete_doc]`   | Before doc delete        |
| `#[assert_upload_asset]` | Before asset upload      |
| `#[assert_delete_asset]` | Before asset delete      |

---

## Custom functions

Define callable endpoints using standard `ic_cdk` macros. Unlike hooks, these are explicitly invoked from your frontend.

```rust
use junobuild_satellite::{caller_is_admin, include_satellite};

fn my_guard() -> Result<(), String> {
    caller_is_admin()
}

#[ic_cdk::query(guard = "my_guard")]
fn hello_world() -> String {
    "Hello, admin!".to_string()
}

include_satellite!();
```

- `#[ic_cdk::query]` — read-only, fast, no state changes
- `#[ic_cdk::update]` — can read and write state, goes through consensus

Juno automatically generates a frontend client API from your function definitions at build time.

### Built-in guards

| Guard                           | Allows                     |
| ------------------------------- | -------------------------- |
| `caller_is_admin()`             | Admin access keys only     |
| `caller_has_write_permission()` | Admin + editor access keys |
| `caller_is_access_key()`        | Any recognized access key  |

---

## Datastore access

```rust
use junobuild_satellite::{get_doc_store, set_doc_store, list_docs_store, delete_doc_store};
use junobuild_utils::{decode_doc_data, encode_doc_data};

// Get
let doc = get_doc_store(context.caller, "posts".to_string(), "my-key".to_string())?;

// Decode
let post: Post = decode_doc_data(&doc.unwrap().data.data)?;

// Set
set_doc_store(
    context.caller,
    context.data.collection.clone(),
    context.data.key.clone(),
    SetDoc {
        data: encode_doc_data(&post)?,
        description: context.data.data.after.description.clone(),
        version: context.data.data.after.version,
    }
)?;

// List
let result = list_docs_store(context.caller, "posts".to_string(), &ListParams::default())?;

// Delete
delete_doc_store(context.caller, "posts".to_string(), "my-key".to_string(), DelDoc { version: None })?;
```

Also available: `count_docs_store`, `count_collection_docs_store`, `delete_docs_store`, `delete_filtered_docs_store`.

Full reference: [juno.build/docs/reference/functions/rust/sdk](https://juno.build/docs/reference/functions/rust/sdk)

---

## Storage access

```rust
use junobuild_satellite::{get_asset_store, set_asset_handler, delete_asset_store};

// Store an asset (e.g. a generated JSON file)
set_asset_handler(
    &AssetKey {
        full_path: "/data/output.json".to_string(),
        collection: "data".to_string(),
        name: "output.json".to_string(),
        ..Default::default()
    },
    &json_bytes,
    &[],
)?;

// Get
let asset = get_asset_store(context.caller, &"data".to_string(), "/data/output.json".to_string())?;
```

Also available: `list_assets_store`, `count_assets_store`, `delete_assets_store`, `delete_filtered_assets_store`, `set_asset_token_store`.

Full reference: [juno.build/docs/reference/functions/rust/sdk](https://juno.build/docs/reference/functions/rust/sdk)

---

## Serialization

Use `junobuild_utils` helpers to encode/decode your Rust structs to/from document data:

```rust
use junobuild_utils::{decode_doc_data, encode_doc_data, encode_doc_data_to_string};
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct Post {
    title: String,
    body: String,
}

let post: Post = decode_doc_data(&raw_bytes)?;
let encoded: Vec<u8> = encode_doc_data(&post)?;
let json: String = encode_doc_data_to_string(&post)?;  // useful for storing as JSON in Storage
```

Full reference: [juno.build/docs/reference/functions/rust/utils](https://juno.build/docs/reference/functions/rust/utils)

---

## HTTPS outcalls

```rust
use ic_cdk::management_canister::{http_request as http_request_outcall, HttpMethod, HttpRequestArgs};

let request = HttpRequestArgs {
    url: "https://api.example.com/data".to_string(),
    method: HttpMethod::GET,
    body: None,
    max_response_bytes: None,
    transform: None,
    headers: vec![],
    is_replicated: Some(false),  // single-node, cheaper; Some(true) for verified consensus
};

match http_request_outcall(&request).await {
    Ok(response) => {
        let body = String::from_utf8(response.body).unwrap();
    }
    Err(e) => return Err(format!("HTTP request failed: {e:?}"))
}
```

HTTPS outcalls cost cycles. With `is_replicated: Some(true)` (default), all nodes execute the request and must agree — the API must return identical responses. `Some(false)` skips consensus, cheaper but unverified.

---

## Calling other canisters

```rust
use candid::Principal;
use ic_cdk::api::call;

let canister_id = Principal::from_text("ryjl3-tyaaa-aaaaa-aaaba-cai").unwrap();
let (result,): (SomeType,) = call(canister_id, "method_name", (arg,))
    .await
    .map_err(|e| format!("Call failed: {e:?}"))?;
```

Note the tuple unpacking `(result,)` — `ic_cdk::call` always returns a tuple.

Full ic-cdk reference: [juno.build/docs/reference/functions/rust/ic-cdk](https://juno.build/docs/reference/functions/rust/ic-cdk)
