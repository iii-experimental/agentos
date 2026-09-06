# AgentOS — agent guide

## What this is

AgentOS is an "agent operating system" built on the [iii engine](https://github.com/iii-hq/iii). It ships **65 narrow workers** (64 Rust + 1 Python), each a standalone binary owning one domain. Workers hold **no shared in-process state** — they coordinate only through iii primitives over the engine WebSocket (`ws://localhost:49134`, override with `III_WS_URL`).

Three primitives are the entire protocol:
- **Worker** — one binary per domain, connects via `register_worker(ws_url, InitOptions::default())`.
- **Function** — a named handler a worker registers: `iii.register_function(RegisterFunction::new_async("agent::chat", ...))`. Namespaced `worker::verb`.
- **Trigger** — binds a Function to HTTP / cron / pub-sub.

The only inter-worker call is `iii.trigger(TriggerRequest { function_id, payload, action: None, timeout_ms: None })`. Fire-and-forget = the same call inside `tokio::spawn`. See `ARCHITECTURE.md` for the full primitive flow and worker→namespace map.

## Common commands

```bash
cargo build --workspace --release           # all workers + both surface crates
cargo test --workspace --release            # full Rust suite (~1,300 tests)
cargo test -p agentos-core <test_name>      # one worker / one test
cargo check --workspace --all-targets       # fast pre-commit gate, matches CI

iii --config config.yaml                    # engine (terminal 1)
bash scripts/dev-up.sh                      # spawn every target/release/agentos-* worker (terminal 2)
bash scripts/dev-up.sh --build              # cargo build --release first
bash scripts/dev-up.sh stop                 # kill workers started by dev-up

cargo run --release -p agentos-tui          # chat-first terminal UI
cargo run --release -p agentos-cli -- ...   # CLI (binary name: `agentos`)

npm install && npm run test:e2e             # vitest e2e vs. live stack (AGENTOS_API_KEY in .env)
npm run test:e2e:smoke                      # health + chat_completions only
docker compose up --build                   # engine + all workers in one container
```

Ports: engine WebSocket **49134**, `iii-http` REST **3111**, streaming **3112**. `cp .env.example .env` and set `ANTHROPIC_API_KEY` before running — without it `llm-router` falls back to mocks.

## Layout

```
workers/<name>/          64 Rust workers + embedding/ (Python). Each: src/main.rs (300–2000 LOC,
                         5–10 registered functions), Cargo.toml, iii.worker.yaml
crates/cli, crates/tui   Surfaces. Speak HTTP to iii-http:3111. Register NO functions.
tests/                   Rust integration tests (workspace-level)
e2e/                     vitest suite against a live engine + workers
hands/<name>/HAND.toml    Agent personas → hand-runner worker
integrations/<name>.toml  MCP server connection configs → mcp-client worker
agents/<name>/            Markdown agent templates
workflows/<name>.yaml     Workflow definitions → workflow worker
plugin/                   Reusable agent/command/skill/hook bundles
config.yaml               iii engine boot config; config/ holds per-module overrides
```

`hands/`, `integrations/`, `agents/`, `workflows/` are **declarative config, not workers** — they configure workers that register functions.

## Adding or changing a worker

1. Create `workers/<name>/` with `src/main.rs`, `Cargo.toml` (`.workspace = true` for version/edition/license and shared deps), and `iii.worker.yaml`.
2. Add the path to `members` in the root `Cargo.toml`.
3. `iii.worker.yaml` is CI-enforced: keys `iii: v1`, `name` (must equal the folder name), `language` (`rust`|`python`), `deploy` (`binary`|`image`), `manifest`, and `bin` (cargo bin name) for binary deploys.
4. Naming: folder `agent-core` → cargo package `agentos-core` → binary `agentos-core` (what `iii.worker.yaml: bin` names).

## Hard constraints (CI blocks on these)

- **No worker may register `sandbox::*`** — reserved for the builtin iii-sandbox. AgentOS sandboxing is the `wasm-sandbox` worker under `wasm::*`. CI greps `workers/` and `crates/`.
- **`iii.worker.yaml` must parse and its `name` must match the folder** — every `workers/*/`.
- e2e smoke asserts ≥30 functions register across core workers and that ports 49134 and 3111 open.
- Prefer iii's atomic `state::update` / `stream::update` (`UpdateOp::set|increment|append`, nested shallow-merge) over `state::list` + `state::set` for lists/counters.

## Versioning

`iii-sdk` is pinned exactly: `=0.11.6` (Rust workspace), `0.11.6` (Node, e2e only), `>=0.11.6` (Python embedding worker); engine `v0.11.6`. Bump all four in lockstep. Workspace `version` is `0.0.1` (pre-1.0, deliberate).

## CI

`.github/workflows/ci.yml` on every PR: `cargo check` + `cargo build --release` + `cargo test --workspace --release`; `iii.worker.yaml` validation; the `sandbox::*` grep guard; e2e smoke (no LLM key); e2e full (gated on `AGENTOS_API_KEY`). `.github/workflows/vercel-deploy.yml` fires a Vercel deploy hook on `main` pushes touching `website/**`.

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

When the user types `/graphify`, use the installed graphify skill or instructions before doing anything else.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- Dirty graphify-out/ files are expected after hooks or incremental updates; dirty graph files are not a reason to skip graphify. Only skip graphify if the task is about stale or incorrect graph output, or the user explicitly says not to use it.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
