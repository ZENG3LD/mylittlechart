# Architecture

## Components

Three separate repositories ship together as a system:

```
mylittlechart (this repo)        — Desktop client. All local, no required server.
    |
    +--- zengeld-server (hosted) — Cloud API: accounts, sync. Not yet public.
    |
    +--- mylittlechart-landing   — Marketing site at mylittlechart.org. Not yet public.
```

The desktop client can operate fully standalone. Network calls to the hosted server are optional. The local Agent API (`127.0.0.1:17420`) is always available regardless of network connectivity.

---

## Client internals

```
┌─────────────────────────────────────────────────────────────┐
│                   chart-app-vello (binary)                   │
│   winit event loop · multi-window AppState · save_all()     │
└────────────┬────────────────────────┬───────────────────────┘
             │                        │
      ┌──────▼──────┐         ┌───────▼───────┐
      │  chart-app  │         │ zengeld-server │
      │  tick()     │         │ Agent API      │
      │  action queue│        │ :17420         │
      └──────┬──────┘         └───────────────┘
             │
      ┌──────▼──────────────────────────┐
      │           chart                  │
      │  panel_app · rendering · profile │
      │  indicators layout · drawings    │
      └──┬──────────┬───────────────────┘
         │          │
  ┌──────▼──┐  ┌────▼──────┐
  │indicators│  │  panels   │
  │ 480+     │  │ DOM/FP/VP │
  │ rayon    │  │ tape/etc  │
  └──────────┘  └────┬──────┘
                     │
              ┌──────▼──────┐
              │  live-data  │
              │  DataBridge │
              │  ConnPool   │
              │  bar cache  │
              └──────┬──────┘
                     │
              ┌──────▼──────┐
              │ digdigdig3  │
              │ 18 exchange │
              │ connectors  │
              └─────────────┘
```

### Render pipeline

- Dedicated GPU thread holds the Vello surface.
- Main (app) thread builds scenes in parallel using `thread::scope` per window.
- Double-buffered scenes: one being built, one being rendered.
- Indicator computation runs on a rayon thread pool — parallelized across all active indicators.
- The render thread drains the Agent API command queue each frame.

### Data flow

1. `digdigdig3` connectors maintain WebSocket feeds per exchange.
2. `live-data::DataBridge` receives updates and writes them into shared bar/trade/orderbook stores.
3. `chart-app` reads from stores each tick and pushes updated data to chart panels.
4. `indicators` crate computes outputs from bar series on each new bar.
5. `zengeld-server` reads bar cache and indicator snapshots via `Arc<AgentState>`.

### State ownership

- `AppState` (in `chart-app-vello`) is the canonical owner of all user-level data.
- Windows push typed `Action` enums into a queue; `chart-app` drains the queue in `about_to_wait()`.
- `AgentState` is `Arc`-shared between the axum server and the render thread. Updates use `RwLock` for reads and a `Mutex<Vec<AgentCommand>>` queue for writes.

### Render backends

| Backend | Crate | Default |
|---|---|---|
| Vello GPU | `vello-context` | Yes |
| Vello CPU | `vello-cpu-context` | Fallback |
| Vello Hybrid | `vello-hybrid-context` | No |
| TinySkia | `tiny-skia-context` | No |
| wgpu Instanced | `instanced-context` | No (experimental) |

The backend is selected at startup based on GPU availability and configured preference.

### Alert delivery

```
alert fires
    │
    ├── toast_tx (unbounded channel) → UI event loop → ToastNotification rendered
    │
    ├── Telegram Bot API (async reqwest) → sendMessage or sendPhoto per subscriber
    │
    └── HTTP Webhook (async reqwest) → POST JSON payload
```

All channels run concurrently in a background Tokio task. A failed channel does not block the others.

### Profile and encryption

User profile (presets, watchlists, notification settings) is stored in a JSON file, optionally encrypted at rest.

Encryption: passphrase → PBKDF2-HMAC-SHA256 (600K iterations, 16-byte salt) → master key → HKDF-SHA256 (info=`"mylittlechart-vault-v1"`) → 32-byte AES-256-GCM key.

Blob format: `[12-byte nonce][ciphertext][16-byte GCM tag]`.

---

## External dependencies (key)

| Dependency | Role |
|---|---|
| `digdigdig3` | Exchange connectors (WebSocket + REST) — local path in dev, crates.io in CI |
| `uzor` | 2D UI framework built on Vello |
| `gate4agent` | Agent key management primitives |
| `rayon` | Data-parallel indicator computation |
| `tokio` | Async runtime for live data + Agent API server |
| `axum` | Agent API HTTP server |
| `reqwest` | Alert delivery HTTP client (Telegram, Webhook) |
| `aes-gcm` | AES-256-GCM for profile encryption |
| `pbkdf2` + `hkdf` | Key derivation for the vault |
| `serde` / `serde_json` | Serialization |
