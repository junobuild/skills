# Concepts

Juno abstracts most of the underlying complexity, but knowing these concepts helps debug and design better.

---

## Internet Computer (ICP)

A compute platform that runs containers — WebAssembly modules with persistent state. Juno's Satellites, Mission Controls, and Console all run as containers on ICP.

Key properties:

- **No servers**: code runs across a network of nodes
- **Persistent memory**: container state survives as long as the container has cycles — it's pay-in-advance; run out of cycles and it eventually gets deleted
- **Deterministic execution**: all nodes run the same code and reach the same state
- **Built-in HTTPS**: containers can serve HTTP directly — no CDN needed for Satellites

---

## Containers

A container = WebAssembly module + state + cycles balance. It's the unit of deployment on ICP. On ICP, a container is called a **canister**.

A Juno Satellite is a container that bundles:

- The Juno runtime (data, auth, storage, hosting)
- Your custom serverless functions (compiled into the same WASM)

Containers have a **principal** — a unique identifier (e.g. `aaaaa-bbbbb-ccccc-ddddd-cai`).

---

## Principals & Identities

- A **principal** is the universal ID for any entity on ICP: users, containers, apps
- When a user signs in, they get a principal
- CLI and GitHub Actions use **access key principals** to authenticate

---

## Cycles

ICP's compute/resource currency. Analogous to AWS compute credits.

- Containers burn cycles to execute code, store data, make HTTP calls
- Satellites need to be **topped up** periodically (or they freeze/get deleted)
- Top up in Console → Satellite → Cycles/Wallet
- Enable monitoring in the Console to auto-top up Satellites automatically

> ⚠️ If a Satellite runs out of cycles it freezes. If it stays frozen too long it gets deleted. Set up monitoring.

> **Note:** Cycle costs observed in the local emulator cannot be compared to production — the emulator runs a simplified environment and does not reflect real-world usage.

---

## Container upgrades

Upgrading a Satellite = replacing its WASM while preserving memory.

- `stable` memory persists across upgrades ✅
- `heap` memory is deserialized and re-serialized (limited ⚠️)

This is why collection memory type matters. Prefer `stable`.

---

## Query vs Update calls

ICP has two types of calls — think of them as two different kinds of requests:

- **Query**: read-only, fast (~200ms), free. The response is not cryptographically verified.
- **Update**: can read and write state, slower (~2s), costs cycles. The response is verified and guaranteed.

| Type       | Speed  | Cycles cost  |
| ---------- | ------ | ------------ |
| **Query**  | ~200ms | Free         |
| **Update** | ~2s    | Costs cycles |

---

## Glossary

See the full terminology reference at [juno.build/docs/terminology](https://juno.build/docs/terminology).
