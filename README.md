# protos/ — authoring guide for AI agents

These `.proto` files are the single source of truth for the wasm task contract.
Two codegen passes consume them:

1. **swift-protobuf** (`--swift_out`) → `<base>.pb.swift` — the message/enum types.
2. **[protoc-gen-flow-kit](https://github.com/mahainc/protoc-gen-flow-kit)**
   (`--flow-kit_out`) → `<base>.fk.pb.swift` — FlowKit/TaskWasm convenience
   session wrappers, emitted **only** for files that declare a `service` block.
   Files without a `service` are silently skipped (the `.pb.swift` already
   covers them).

Both are wired into the repo `Makefile`:

```sh
make proto            # swift-protobuf for all targets
make proto-flow-kit   # builds + runs the flow-kit plugin for service-bearing protos
```

`make proto-flow-kit` first does `swift build --package-path tools/protoc-gen-flow-kit -c release`,
then runs `protoc` with `--plugin=protoc-gen-flow-kit=.../release/protoc-gen-flow-kit`
and `--flow-kit_out=<same dir as --swift_out>`. You do not invoke `protoc`
by hand — edit the proto, run the make target.

---

## What flow-kit generates per service

For a file with `option swift_prefix = "Tcg"` and `service TcgService { rpc Scan(...) ... }`,
the plugin emits into `Tcg.fk.pb.swift`:

- **`enum TcgMethod: String`** — one case per rpc, raw value
  `"<package>.<Service>/<Method>"` (the wasm dispatch key). Every rpc gets a
  case, even skipped ones.
- **`struct Tcg`** namespace on the wasm host (`wasm.tcg.scan(...)`) — one typed
  wrapper func per rpc that builds the request message, serializes it, and
  dispatches through `run(method:args:)`. Skipped rpcs get no wrapper.
- **`argString` extensions** on opted-in enums — maps each Swift case to the
  string the Rust task layer expects on the wire.

### Naming derivation (don't fight it)
- **Type prefix** = `option swift_prefix` if set, else PascalCase of the proto
  `package`'s last segment. This is the `Tcg` in `TcgMethod`. **Always set
  `swift_prefix` explicitly** — every service proto here does.
- **Func prefix** = type prefix with a lowercased first letter (`tcg`), used for
  the host accessor `wasm.tcg.<method>(...)`.
- **rpc case name** = mechanical lowerCamel of the rpc name, unless overridden
  (see `swift_case` below).

---

## Writing a service proto — checklist

1. `syntax = "proto3";`
2. `option swift_prefix = "<Pascal>";` (required — drives every generated name).
3. Define `<Method>Request` / `<Method>Response` message pairs.
4. Declare `service <Pascal>Service { rpc <Method>(<Method>Request) returns (<Method>Response); }`.
5. If you need any codegen knob (below), `import "codegen.proto";` and annotate.
6. Comment every message, field, rpc, and enum value — units, purpose, wire
   context. swift-protobuf 1.35.1 drops file-level comments, so `make proto`
   re-injects them via `scripts/inject_file_header.py`; per-field comments are
   preserved and surface in the generated Swift docs.

A request/response pair is required even when empty — the rpc signature must be
well-formed. The rpc *is* the contract; hosts that adopt the generated session
skip per-action discovery.

---

## codegen.proto options (the only knobs)

`codegen.proto` (proto2, package `asyncify.codegen`) defines custom option
extensions the plugin reads off descriptors. **None of these change the wire
shape or are read by Rust** — they only steer Swift codegen. `import "codegen.proto";`
to use them.

### Method options — on an `rpc`

```proto
rpc Init(InitRequest) returns (InitResponse) {
  option (asyncify.codegen.swift_case) = "providerInit";
}
```

| Option | Type | Effect |
|--------|------|--------|
| `skip_flow_kit` | bool | Emit **no** typed wrapper. The `<Prefix>Method` case is still emitted (dispatch by method-name still works). Use for rpcs whose typed wrapper would be wire-incorrect — e.g. `stream` rpcs that own their own host driver (`Chat` in openai.proto / livescore.proto). |
| `swift_case` | string | Override the enum case + dispatch name. Use when mechanical lowerCamel collides with a Swift keyword — `Init → "providerInit"`. |
| `task_only` | bool | Emit only the `*Task()` raw variant, no typed unpack wrapper. Use for rpcs returning `asyncify.task.Task` that fire-and-forget an OTel event with no useful payload. |

### Enum options — on an `enum`

```proto
enum AnalyticsWindow {
  option (asyncify.codegen.gen_arg_string) = true;
  option (asyncify.codegen.unrecognized_arg_string) = "3M";
  ANALYTICS_WINDOW_UNSPECIFIED = 0 [(asyncify.codegen.arg_string) = "ALL_TIME"];
  ONE_DAY  = 1 [(asyncify.codegen.arg_string) = "1D"];
  THREE_MONTHS = 4 [(asyncify.codegen.arg_string) = "3M"];
}
```

| Option | Scope | Effect |
|--------|-------|--------|
| `gen_arg_string` | enum | Emit an `argString` extension mapping each case to its Rust wire string. **Mechanical default per case** = strip the SCREAMING_SNAKE type prefix (`BREAKDOWN_FACET_BY_SET → "BY_SET"`); a `*_UNSPECIFIED` / bare `UNSPECIFIED` case defaults to `""`. |
| `unrecognized_arg_string` | enum | Wire string for swift-protobuf's synthetic `.UNRECOGNIZED` case. Defaults to `""`; set when the fallback should be a real value. |
| `arg_string` | enum value | Per-value override when the label doesn't follow strip-prefix — `ONE_DAY → "1D"`, `MAGIC_THE_GATHERING → "MTG"`. |

**Only add `gen_arg_string` when the Rust parser actually expects the
strip-prefix/uppercase spelling.** If Rust expects abbreviations the mechanical
rule can't produce (e.g. Condition's `NM/LP/MP/HP/DMG`), leave `gen_arg_string`
off — see the `Condition` enum in `tcg.proto` for that case.

---

## Conventions baked into the plugin

- **`account_id` fields are dropped from wrapper params.** The wasm module fills
  them from its cached session (`rust/src/task/collectr/session.rs`); the host
  never supplies them. Name a field `account_id` to get this behavior.
- **Generated file is a separate sibling**, never a replacement for a
  hand-written `<X>Session.swift`. Adopting the generated file and keeping the
  hand-written one in the same module is a **collision** — both declare the same
  `<Prefix>Method` enum and conflicting func names. Adopt one or the other.

---

## Current service protos

`aiart` · `blobstore` · `calories` · `homedecor` · `inpaint` · `livescore` ·
`openai` · `tcg` · `user` · `vision` · `visual`.

Non-service protos (types, engine, flows, task, codegen, otel, mail, music,
browser, it) are message-only and get `--swift_out` only.
