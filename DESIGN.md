# lsp-client-rs — Design

A Rust-native LSP **client** library. This document is the design specification.
No implementation code exists yet; this file defines the architecture, the
public API shape, and the rationale behind every major decision.

The design is informed by an existing Python LSP client library whose core
*functionality* we want to keep, but the API is redesigned from scratch around
Rust's type system rather than translated from Python's dynamic features.

---

## 1. Goals

Functional parity with the reference design — nothing more, nothing less:

1. A set of **composable LSP capabilities** (hover, definition, references, …)
   that can both (a) define a concrete language client and (b) constrain generic
   code to "any client supporting these capabilities".
2. Each capability carries three things:
   - the LSP method name + request params + result type,
   - a **client capability declaration** (what we advertise in `initialize`),
   - a **server capability check** (validate the server actually supports it).
3. Client **lifecycle**: connect → `initialize` negotiation → use → shutdown.
4. Multiple **transport backends** with prioritized fallback
   (user-provided → local subprocess → container → local-with-auto-install).
5. **Document synchronization** (open/change/close with reference counting) and
   handling of **server-to-client messages** (notifications + reverse requests).

## 2. Non-Goals

- Building a language *server* framework (this is a client only).
- Re-deriving LSP types or the JSON-RPC wire format — we reuse `lsp-types`.
- Faithful 1:1 mirroring of the Python API. We keep the *capabilities*, not the
  shape.

---

## 3. Design Philosophy (Rust-native, first principles)

The Python reference fuses capability *declaration* and capability
*implementation* into a single mixin and aggregates them at runtime via
reflection (`issubclass`, MRO `super()` chains). Rust's worldview is the
opposite. Three principles drive this design:

1. **Types as capability proofs.** Whether you *can* call `hover` is proven by
   the type system, not checked at runtime. Errors surface at compile time
   wherever possible.
2. **Data/behavior separation, zero-cost composition.** A capability's
   *metadata* (method name, `ClientCapabilities` fragment) is compile-time
   constant data; its *behavior* (the `hover()` call) is derived through generic
   dispatch with no runtime indirection.
3. **Transport/protocol decoupling.** The LSP semantic layer is agnostic to
   whether the bytes flow over a subprocess, a container, or a socket. Dynamic
   dispatch is paid for in exactly one place: choosing a backend at runtime.

### The key Rust leverage

`lsp-types` already implements a `Request` trait for every LSP request, binding
**`METHOD` (the method-name string), `Params`, and `Result`** together at the
type level:

```rust
// from lsp-types — we do NOT redefine this
pub enum HoverRequest {}
impl Request for HoverRequest {
    const METHOD: &'static str = "textDocument/hover";
    type Params = HoverParams;
    type Result = Option<Hover>;
}
```

This single fact removes most of the hand-written metadata the Python version
needs. A capability *is* a `Request` type; we only add the two things
`lsp-types` does not have (declaration + server check).

### Async in traits

This design targets Rust with native `async fn` in traits (AFIT, stable since
1.75). **We do not use the `#[async_trait]` macro** — it forces
`Box<dyn Future>` heap allocation and type erasure on every call. Native AFIT
monomorphizes and is zero-cost.

The one consequence to design around: **AFIT traits are not object-safe by
default** (`fn f(&self) -> impl Future` cannot go behind `dyn`). This shapes the
runtime-dispatch story in §6.3.

---

## 4. Layered Architecture

```
┌──────────────────────────────────────────────────────────┐
│  capability layer                                          │  ← primary user surface
│  capability = a type; "has capability" = a type-level proof│
├──────────────────────────────────────────────────────────┤
│  client core                                               │  ← protocol semantics
│  LspConn: call_raw / notify_raw / document sync / lifecycle│
├──────────────────────────────────────────────────────────┤
│  transport                                                 │  ← byte stream + JSON-RPC framing
│  Transport trait: stdio / container / socket               │
└──────────────────────────────────────────────────────────┘
```

### Crate layout

```
lsp-client-rs/                  (this workspace)
├── lsp-client-core/      # Transport, Channel, JSON-RPC framing, LspConn
├── lsp-client/           # Negotiate, Supports, Lsp facade, lifecycle, bundles
├── lsp-client-macros/    # the `client!` proc-macro
└── lsp-client-langs/     # prebuilt clients: Pyright, RustAnalyzer, Deno, …
```

---

## 5. Capability Layer (the core redesign)

### 5.1 A capability is an `lsp_types::Request` + a `Negotiate` impl

We reuse the `Request` type as the capability. The two missing pieces —
client-side declaration and server-side validation — are supplied by a
companion trait:

```rust
/// Lift an LSP request/notification type into a negotiable capability.
pub trait Negotiate {
    /// Write this capability's fragment into the client capabilities we
    /// advertise during `initialize` (≈ Python `register_*_capability`).
    fn declare(caps: &mut ClientCapabilities);

    /// Validate the server actually supports it (≈ Python
    /// `check_server_capability`). Returns the offending method on failure.
    fn check(server: &ServerCapabilities) -> Result<(), Unsupported>;
}

impl Negotiate for HoverRequest {
    fn declare(c: &mut ClientCapabilities) {
        c.text_document.get_or_insert_default().hover = Some(HoverClientCapabilities {
            content_format: Some(vec![MarkupKind::Markdown, MarkupKind::PlainText]),
            ..Default::default()
        });
    }
    fn check(s: &ServerCapabilities) -> Result<(), Unsupported> {
        s.hover_provider
            .as_ref()
            .map(|_| ())
            .ok_or(Unsupported("textDocument/hover"))
    }
}
```

> A capability = `lsp-types`'s `Request` type (gives `METHOD`/`Params`/`Result`)
> + our `Negotiate` impl (gives declare + check). Method name, params, and
> result type are all reused from `lsp-types`.

### 5.2 "Client `C` supports capability `R`" = a type-level proof

```rust
/// Marker: client `C` declares support for capability `R`.
/// Zero-sized, generated by the `client!` macro, purely compile-time.
pub trait Supports<R: Request + Negotiate> {}
```

### 5.3 The unified call facade (native AFIT, zero-cost)

There is exactly **one** generic entry point. Every capability call goes through
it; there is no per-capability boilerplate trait.

```rust
/// Object-safe-friendly core implemented by every connected client.
pub trait LspConn: Send + Sync {
    fn call_raw(&self, method: &str, params: Value)
        -> impl Future<Output = Result<Value, LspError>> + Send;
    fn notify_raw(&self, method: &str, params: Value)
        -> impl Future<Output = Result<(), LspError>> + Send;
    fn workspace(&self) -> &Workspace;
}

/// Generic facade: `request::<R>` exists only when `Self: Supports<R>`.
pub trait Lsp: LspConn {
    fn request<R>(&self, params: R::Params)
        -> impl Future<Output = Result<R::Result, LspError>> + Send
    where
        R: Request + Negotiate,
        Self: Supports<R>,                 // ← the capability proof is the bound
    {
        async move {
            let raw = self.call_raw(R::METHOD, serde_json::to_value(params)?).await?;
            Ok(serde_json::from_value(raw)?)
        }
    }

    fn notify<N>(&self, params: N::Params)
        -> impl Future<Output = Result<(), LspError>> + Send
    where
        N: NotificationTrait + Negotiate,
        Self: Supports<N>,
    {
        async move {
            self.notify_raw(N::METHOD, serde_json::to_value(params)?).await
        }
    }
}
impl<T: LspConn> Lsp for T {}
```

Usage:

```rust
let hover  = client.request::<HoverRequest>(params).await?;   // ✅ if Supports<HoverRequest>
let rename = client.request::<RenameRequest>(params).await?;  // ❌ compile error: capability not declared
```

### 5.5 Capability declaration rules (`Negotiate::declare`)

`declare` writes directly into `&mut ClientCapabilities`. Two explicit rules
prevent conflicts and keep per-capability impls independent:

**Rule 1 — one impl owns one leaf field.**
Each `Negotiate` impl writes exactly the sub-field of `ClientCapabilities` that
corresponds to its LSP method. No two impls touch the same leaf. `HoverRequest`
owns `text_document.hover`; `GotoDefinition` owns `text_document.definition`.

```rust
// Inside ClientDescriptor::build_capabilities:

HoverRequest::declare(caps);
GotoDefinition::declare(caps);
DidOpenTextDocument::declare(caps);
```

**Rule 2 — global settings are not capability fields.**
Fields like `dynamic_registration` or any future `insert_text_mode` default are
client-wide policy, not per-capability concerns. They are set directly in
`ClientDescriptor::build_capabilities`, never inside individual `Negotiate` impls.

#### Why not a fragment-and-merge pattern for everything?

For Rule 1 fields, returning a fragment value and then merging provides no
benefit over direct writes — the fields are disjoint, so "merging" is just
assignment. The extra type machinery (a `ClientCapabilitiesFragment` sum type
plus a `merge` function) would be pure complexity for zero safety gain.

### 5.4 Ergonomic methods are an optional sugar layer

`client.hover(path, pos)` style short methods are *not* the capability
mechanism. They are thin wrappers over `request::<HoverRequest>` plus result
normalization, living in an optional extension trait:

```rust
pub trait HoverExt: Lsp + Supports<HoverRequest> {
    fn hover(&self, path: &Url, pos: Position)
        -> impl Future<Output = Result<Option<MarkupContent>, LspError>> + Send
    {
        async move {
            let h = self.request::<HoverRequest>(HoverParams {
                text_document_position_params: pos_params(path, pos),
                ..Default::default()
            }).await?;
            Ok(h.map(normalize_hover))   // collapse MarkedString | str | MarkupContent
        }
    }
}
impl<T: Lsp + Supports<HoverRequest>> HoverExt for T {}
```

This is a deliberate simplification over the Python version, which coupled the
capability mechanism and the ergonomic method into one mixin. Here the mechanism
is the uniform `request::<R>`; ergonomics are an independent, opt-in layer.

---

## 6. Declaring "a client with specific capabilities"

All three scenarios are unified on `Supports<R>`; they differ only in *where*
the capability proof appears: at the type definition, at a generic bound, or in
a trait-object bundle.

### 6.1 Scenario A — define a concrete client (capability set fixed at compile time)

```rust
client! {
    Pyright {
        language: python,
        servers: [
            stdio("pyright-langserver", ["--stdio"]),
            container("ghcr.io/lsp-client/pyright:latest"),
        ],
        capabilities: [
            HoverRequest, GotoDefinition, References, Rename,
            PublishDiagnostics,           // server → client notification
            WorkspaceConfiguration,       // server → client request
        ],
    }
}
```

The macro expands to:

- `impl Supports<HoverRequest> for Pyright {}` (one per listed capability),
- an aggregate that calls `<R as Negotiate>::declare(&mut caps)` for each
  capability in sequence (this *replaces* the Python `super()` chain — the macro
  expands the chain into an explicit static sequence over a single shared
  `ClientCapabilities` builder),
- an aggregate `check_server()` calling `<R as Negotiate>::check(server)?` for
  each.

### 6.2 Scenario B — generic code constrained to "any client with these caps"

```rust
async fn navigate<C>(c: &C, path: &Url, pos: Position) -> Result<(), LspError>
where
    C: Lsp + Supports<HoverRequest> + Supports<GotoDefinition>,
{
    let _ = c.request::<HoverRequest>(/* … */).await?;
    let _ = c.request::<GotoDefinition>(/* … */).await?;
    Ok(())
}
```

`where C: Supports<HoverRequest> + Supports<GotoDefinition>` reads literally as
"a client supporting hover and definition" — something the Python version cannot
express statically.

**Capability bundles** (optional convenience) collapse long bound lists, using
the standard-library `Read + Write` blanket-impl idiom:

```rust
pub trait Navigation:
    Lsp + Supports<HoverRequest> + Supports<GotoDefinition> + Supports<References> {}

impl<T> Navigation for T
where T: Lsp + Supports<HoverRequest> + Supports<GotoDefinition> + Supports<References> {}

async fn navigate<C: Navigation>(c: &C) { /* … */ }   // single bound
```

### 6.3 Scenario C — runtime client selection (multi-language dispatch)

`request::<R>` is a generic method → the AFIT trait is **not object-safe** → it
cannot go behind `Box<dyn>`. The design offers three Rust-native answers,
ordered by preference:

**Preferred: `enum` + static dispatch** when the client set is closed and small:

```rust
enum AnyClient { Pyright(Client<Pyright>), RustAnalyzer(Client<RustAnalyzer>) }

impl AnyClient {
    async fn hover(&self, p: &Url, pos: Position) -> Result<Option<MarkupContent>, LspError> {
        match self {
            AnyClient::Pyright(c)      => c.hover(p, pos).await,
            AnyClient::RustAnalyzer(c) => c.hover(p, pos).await,
        }
    }
}
```

Zero-cost, exhaustiveness-checked, no object-safety problem.

**When the client set is open/plugin-like: a narrow object-safe bundle.** Only
this thin layer pays the `Box<dyn Future>` cost; the generic main path is
untouched:

```rust
pub trait DynNavigation: Send + Sync {
    fn hover(&self, p: &Url, pos: Position)
        -> Pin<Box<dyn Future<Output = Result<Option<MarkupContent>, LspError>> + Send + '_>>;
    fn definition(&self, p: &Url, pos: Position)
        -> Pin<Box<dyn Future<Output = Result<GotoDefinitionResponse, LspError>> + Send + '_>>;
}

// blanket impl: any generic client satisfying Navigation gets the dyn version
impl<T: Navigation + Send + Sync> DynNavigation for T {
    fn hover(&self, p: &Url, pos: Position) -> Pin<Box<dyn Future<…> + Send + '_>> {
        Box::pin(async move { Lsp::request::<HoverRequest>(self, /* … */).await.map(/* … */) })
    }
    /* … */
}

let c: Box<dyn DynNavigation> = match lang {
    Language::Python => Box::new(pyright),
    Language::Rust   => Box::new(rust_analyzer),
};
c.hover(&uri, pos).await?;   // runtime dispatch; capability still type-guaranteed
```

> The `trait-variant` crate (maintained by rust-lang) can auto-derive such an
> object-safe `dyn`-compatible variant from a native AFIT trait, avoiding the
> hand-written bridge. It is the modern, non-`async_trait` answer when the dyn
> layer is needed. Pulled in only where Scenario C is required.

**Principle:** `Box<dyn Future>` appears only where genuinely unavoidable
(runtime polymorphism over an open client set), never as the default.

---

## 7. Client Lifecycle

Python's `async with` does construction + teardown in one construct. Rust splits
this into explicit async connect + typestate + RAII.

### 7.1 Connect: builder + typestate

```rust
let client = Pyright::connect()
    .workspace(cwd)
    .server(ServerChoice::Auto)          // user → local → container fallback
    .on_notification::<PublishDiagnostics>(|p| async move { /* … */ })
    .on_request::<WorkspaceConfiguration>(|p| async move { Ok(/* … */) })
    .await?;                             // performs initialize handshake + negotiation
```

`connect()` returns `Client<Pyright>`, and **only that type implements `Lsp`**.
The un-connected builder has no `request` method. "Cannot send requests before
initialization" is enforced by the type system, not a runtime check.

During connect:

1. Resolve transport via fallback (§9).
2. Spawn the background dispatch task.
3. `build_capabilities()` (macro aggregate) → send `initialize`.
4. On the response, run `check_server()` — gated by `cfg!(debug_assertions)` so
   release builds skip it at zero cost (mirrors the Python `__debug__` gate),
   surfaced as a `Result`, not a panic/assert.
5. Send `initialized`.

### 7.2 Document sync: scoped closure (semantically closest, async-safe)

`Drop` cannot be async, so we cannot send `didClose` from a destructor. Use a
scope closure instead:

```rust
client.with_files(&["a.py", "b.py"], |scope| async move {
    scope.request::<HoverRequest>(/* … */).await
}).await?;
// after the closure, didClose is awaited for files whose refcount hit zero
```

Reference-counted document state is `Arc<Mutex<HashMap<Url, usize>>>` (mirrors
the Python `DocumentStateManager`): `with_files` opens (refcount++) on entry and
closes (refcount--, didClose at 0) on exit.

### 7.3 Shutdown

```rust
client.shutdown().await?;   // drain pending requests → `shutdown` request → `exit` notification
```

A `Drop` impl provides best-effort fallback (spawn shutdown onto the runtime),
but explicit `shutdown().await` is the recommended path.

---

## 8. Server → Client Messages

Method names arrive as strings over the wire. Starting a connection returns a
not-yet-initialized client plus a server-event receiver. The caller starts the
event loop before the LSP `initialize` request, so server-to-client requests
that arrive during initialization can be answered instead of blocking the
handshake:

```rust
let (initializing, mut server_events) = Pyright::connect()
    .workspace(cwd)
    .start()
    .await?;

tokio::spawn(async move {
    while let Some(event) = server_events.recv().await {
        match event {
            ServerEvent::Notification { method, params } => { /* … */ }
            ServerEvent::Request { method, params, responder } => {
                responder.respond(result_json).await?;
            }
        }
    }
});

let client = initializing.initialize().await?;
```

Internally: the background reader routes responses to pending client requests
and sends server-to-client notifications/requests into an `mpsc` channel.
Reverse requests carry a one-shot responder that writes the JSON-RPC response
back to the same channel writer.

- Unhandled reverse **request** → respond with a JSON-RPC error (an unhandled
  request would block the server).
- Unhandled **notification** → drop or log it in the caller-owned event loop.

---

## 9. Transport + Fallback

The single place that requires a trait object (runtime backend choice):

```rust
pub trait Transport: Send + Sync {
    fn available(&self) -> impl Future<Output = Result<(), TransportError>> + Send;
    fn spawn(self: Box<Self>, ws: &Workspace)
        -> impl Future<Output = Result<Channel, TransportError>> + Send;
}

struct Stdio     { program: String, args: Vec<String>, ensure_installed: Option<InstallHook> }
struct Container { image: String, backend: Oci /* docker | podman | nerdctl */ }
struct Socket    { addr: SocketAddrOrPath }
```

Prioritized fallback (mirrors the Python `_iter_candidate_servers`):

```
user-provided  →  local-if-available  →  container (if enabled)  →  local-with-auto-install
```

```rust
async fn resolve(choice: ServerChoice, defaults: DefaultServers, ws: &Workspace)
    -> Result<Channel, TransportErrors>   // aggregates all failures (≈ ExceptionGroup)
```

### JSON-RPC channel

On top of `Channel`:

- `Content-Length` framing (a `tokio_util::codec` codec or a hand-written
  framed reader).
- `HashMap<RequestId, oneshot::Sender<Response>>` correlates responses to
  awaiting callers and lets shutdown drain in-flight requests.
- `mpsc::Receiver<ServerEvent>` exposes server-to-client notifications and
  reverse requests to caller-owned async control flow.
- Background read/write tasks via `tokio::task`; fan-out with `JoinSet`;
  timeouts via `tokio::time::timeout`.

Container backend reuses the stdio backend to drive the OCI CLI, auto-mounting
workspace folders with host-path-preserving bind mounts. Gated behind a config
flag (default off), matching the reference design.

---

## 10. Mechanism Summary

| Concern | Rust mechanism | When checked |
| --- | --- | --- |
| capability = method + params + result | reuse `lsp_types::Request` (`METHOD`/`Params`/`Result`) | compile time |
| client capability declaration | `Negotiate::declare` (direct write) | compile-time constant data |
| global client policy (`dynamic_registration`, …) | set directly in `build_capabilities`, never in `Negotiate` impls (Rule 2) | compile-time constant data |
| server capability check | `Negotiate::check` → `Result` | runtime, gated by `cfg!(debug_assertions)` |
| "client has capability" | `Supports<R>` marker (macro-generated) | compile time |
| unified call facade | `Lsp::request::<R>() where Self: Supports<R>` | compile-time bound |
| async in traits | native AFIT (`-> impl Future`), **no `#[async_trait]`** | — |
| define a concrete client | `client!` proc-macro | compile time |
| constrain "needs caps X, Y" | `where C: Supports<X> + Supports<Y>` / bundle trait | compile time |
| runtime multi-client dispatch | `enum` static dispatch, or object-safe bundle / `trait-variant` | compile-time bound + runtime dispatch |
| lifecycle | builder `start().await` + event loop + `initialize().await` + typestate + `shutdown().await` | — |
| document sync | `with_files(closure)` scope + refcount | runtime |
| server-to-client messages | `mpsc::Receiver<ServerEvent>` + per-request responder | runtime |
| multi-backend fallback | `Transport` trait object + `resolve()` | runtime |

**One-line summary:** a capability is a *type*; having a capability is a
*type-level proof*; calling is *constrained by that proof*; and dynamic dispatch
is confined to the only inherently dynamic point — choosing a backend, and
optionally selecting a client, at runtime.

---

## 11. Open Decisions (to resolve before implementation)

1. **Call surface:** keep only the uniform `request::<HoverRequest>(params)`
   (minimal, zero boilerplate, manual params), add the ergonomic
   `client.hover(path, pos)` sugar layer, or both?
2. ~~**`Negotiate` impls:** hand-write each capability's `declare`/`check`
   (precise, but volume), or generate common patterns via macro with an escape
   hatch for edge cases?~~ **Resolved:** hand-write. Declaration rules (§5.5)
   keep each impl small; the patch pattern handles the one shared-field case.
3. **`client!` macro vs hand-written impls:** proc-macro as the primary way to
   define a client (concise), and/or also support a fully macro-free path
   (transparent but verbose)?
4. **Runtime dispatch (Scenario C):** first-class now (design the object-safe
   bundle / adopt `trait-variant`), or defer in favor of `enum` static dispatch?
5. **Transport:** build our own three-layer transport (full control, mirrors the
   reference separation), or build on `async-lsp`'s `MainLoop` (less wheel
   reinvention, must adapt to its lifecycle model)?

> Status: minimal implementation exists in `src/`. §5.5 declaration rules are
> implemented and enforced by convention. Remaining open decisions above are
> deferred.
