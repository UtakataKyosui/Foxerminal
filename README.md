# Foxerminal

> The Full-Stack Async Terminal Framework
> Forked from [Loco](https://github.com/loco-rs/loco), specialized for TUI application development in Rust.

Foxerminal is not just a TUI toolkit.

It is a **terminal-native application framework** that takes Loco's battle-tested infrastructure
(ORM, background jobs, scheduler, config, CLI) and replaces the HTTP presentation layer
with a Ratatui-based TUI system.

**Stack:**

- Ratatui (UI Layer)
- SeaORM (Async ORM) — from Loco
- Tokio (Async Runtime) — from Loco
- Background Workers & Cron Scheduler — from Loco
- First-class Plugin System — extended from Loco's Initializer

```
Loco (Web Framework for Rust)
  └─ fork ─→ Foxerminal (TUI Framework for Rust)
               HTTP layer stripped, TUI layer added
```

---

## Origin: Why Fork Loco?

Loco provides a Rails-like experience for Rust web applications.
Its infrastructure layer — database integration, background jobs, cron scheduling,
configuration management, and CLI scaffolding — is entirely HTTP-independent.

Foxerminal exploits this by:

1. **Keeping** Loco's infrastructure layer as-is (~55% of codebase)
2. **Modifying** core traits (`Hooks`, `Initializer`) with feature flags (~16%)
3. **Replacing** the HTTP presentation layer with a TUI system (~29%)

This means Foxerminal inherits Loco's production-tested ORM integration, worker system,
and configuration management without reimplementing them.

```
What comes from Loco:          What Foxerminal adds:
├── SeaORM integration         ├── Ratatui View system
├── Background Workers         ├── EventBus (broadcast-based)
├── Cron Scheduler             ├── Screen Router
├── Cache (inmem / Redis)      ├── Keybinding Manager
├── Storage (S3/Azure/GCP)     ├── Plugin System
├── Mailer                     ├── Terminal lifecycle management
├── Config / Environment       └── TUI test utilities
├── CLI scaffolding
├── Auth (JWT)
└── Logger / Validation / Hash
```

---

## Philosophy

> Everything is an Event.

UI input, view transitions, database writes, background jobs, cron triggers —
all flow through a unified event-driven architecture.

Foxerminal treats the terminal as a first-class application runtime,
not just a rendering surface.

---

## Architecture Overview

```
┌────────────────────────────────────────────────────┐
│                    Ratatui                          │  Declarative View Layer
├────────────────────────────────────────────────────┤
│  View System / Screen Router / EventBus / Plugins  │  Application Core (new)
├────────────────────────────────────────────────────┤
│  SeaORM / Workers / Scheduler / Cache / Storage    │  Infrastructure (from Loco)
├────────────────────────────────────────────────────┤
│                    Tokio                            │  Async Runtime
└────────────────────────────────────────────────────┘
```

Foxerminal follows a **microkernel architecture**:

- The core is minimal.
- All features live in plugins.

### Feature Flag Architecture

```toml
[features]
with-tui  = ["dep:ratatui", "dep:crossterm"]   # TUI mode (default)
with-http = ["dep:axum", "dep:tower-http"]      # HTTP mode (Loco compat)
```

The same codebase can target terminal or web by switching feature flags.

---

## Plugin System (Core Feature)

Plugins are first-class citizens.

Each plugin can provide:

- Views (Ratatui components)
- Screen routes
- Database models & migrations
- Scheduled jobs (Cron)
- Background workers
- Event listeners
- CLI extensions

---

### Plugin Trait

```rust
#[async_trait]
pub trait Plugin: Send + Sync {
    fn name(&self) -> &'static str;

    async fn register(&self, ctx: &mut PluginContext) -> Result<()>;

    async fn on_boot(&self, app_ctx: &AppContext) -> Result<()> { Ok(()) }
    async fn on_shutdown(&self, app_ctx: &AppContext) -> Result<()> { Ok(()) }
    async fn on_event(&self, event: &AppEvent, app_ctx: &AppContext) -> Result<()> { Ok(()) }
}
```

---

### PluginContext

```rust
pub struct PluginContext<'a> {
    pub views: &'a mut ViewRegistry,
    pub router: &'a mut ScreenRouter,
    pub scheduler: &'a mut Scheduler,
    pub event_bus: &'a EventBus,
    pub shared_store: &'a mut SharedStore,
    pub keybindings: &'a mut KeybindingMap,
}
```

This gives plugins controlled access to:

* View registration
* Screen routing
* Scheduling
* Events
* Shared state
* Keybindings

---

## View System

```rust
#[async_trait]
pub trait View: Send {
    fn route(&self) -> &'static str;
    fn name(&self) -> &'static str;

    async fn on_mount(&mut self, ctx: &AppContext) -> Result<()> { Ok(()) }
    async fn on_unmount(&mut self, ctx: &AppContext) -> Result<()> { Ok(()) }

    fn handle_key(&mut self, key: KeyEvent, ctx: &AppContext) -> EventResult;
    fn render(&mut self, frame: &mut Frame, area: Rect, ctx: &AppContext);

    async fn on_tick(&mut self, ctx: &AppContext) -> Result<()> { Ok(()) }
}

pub enum EventResult {
    Handled,
    Ignored,
    Navigate(String),
    Quit,
}
```

Register views in plugins:

```rust
async fn register(&self, ctx: &mut PluginContext) -> Result<()> {
    ctx.views.register(Box::new(MonitorView::new()));
    ctx.keybindings.bind(KeyCode::Char('m'), "navigate:/monitor");
    Ok(())
}
```

---

## Ratatui Ecosystem Compatibility

Foxerminal is designed to be fully compatible with the Ratatui ecosystem (1,686+ crates).
The `View::render()` method exposes Ratatui's `Frame` directly — no wrapper, no abstraction.
Every third-party widget works out of the box.

```rust
use tui_textarea::TextArea;
use ratatui_image::StatefulImage;
use tui_tree_widget::{Tree, TreeState};

fn render(&mut self, frame: &mut Frame, area: Rect, ctx: &AppContext) {
    let chunks = Layout::vertical([
        Constraint::Length(3),
        Constraint::Min(10),
        Constraint::Length(8),
    ]).split(area);

    // tui-textarea: just works
    frame.render_widget(&self.textarea, chunks[0]);

    // ratatui-image: just works
    frame.render_stateful_widget(self.image.clone(), chunks[1], &mut self.image_state);

    // tui-tree-widget: just works
    frame.render_stateful_widget(self.tree.clone(), chunks[2], &mut self.tree_state);
}
```

### Effects (tachyonfx)

Views can apply post-render effects for animations and transitions:

```rust
fn post_render_effects(&mut self, _ctx: &AppContext) -> Vec<Box<dyn FnOnce(&mut Buffer, Rect)>> {
    vec![Box::new(|buf, area| {
        // tachyonfx effects operate on the Buffer directly
    })]
}
```

### Backend Flexibility

Switch rendering backends via feature flags:

```toml
[features]
backend-crossterm = []   # Default: terminal
backend-ratzilla = []    # Browser via WASM
backend-wgpu = []        # GPU-accelerated
```

### Ecosystem Highlights

| Category | Crates |
|----------|--------|
| Text Editing | `tui-textarea`, `edtui`, `ratatui-code-editor` |
| Data Display | `tui-tree-widget`, `tui-widget-list`, `tui-piechart`, `rat-widget` |
| Media | `ratatui-image`, `ratatui-splash-screen` |
| Navigation | `tui-menu`, `ratatui-explorer`, `tui-popup` |
| Effects | `tachyonfx` |
| Input | `tui-input`, `terminput`, `tui-prompts` |
| Utilities | `ratatui-macros`, `ansi-to-tui`, `tui-syntax-highlight` |

---

## Database Extension (SeaORM) — from Loco

Plugins can ship their own models and migrations:

```rust
ctx.shared_store.insert(CreateMonitoringTable);
```

Auto-applied at startup.

Supported databases:

* SQLite
* Postgres
* MySQL

---

## Cron & Background Jobs — from Loco

```rust
ctx.scheduler.add_cron(
    "0 */5 * * * *",
    async move {
        cleanup().await;
    }
);
```

Or spawn background workers:

```rust
ctx.scheduler.spawn(async move {
    background_task().await;
});
```

No external worker required. Workers run alongside the TUI via Tokio,
sharing the same `AppContext` (DB, cache, storage).

---

## Event-Driven Core

All subsystems communicate through an event bus:

```rust
#[derive(Clone, Debug)]
pub enum AppEvent {
    Key(KeyEvent),
    Mouse(MouseEvent),
    Resize(u16, u16),
    ViewChanged { from: String, to: String },
    Tick,
    JobCompleted { name: String, result: Result<(), String> },
    CronTriggered { name: String },
    Custom(Box<dyn Any + Send + Sync>),
}
```

Plugins can subscribe:

```rust
ctx.event_bus.subscribe(|event| async move {
    match event {
        AppEvent::ViewChanged { to, .. } => {
            tracing::info!("Navigated to: {}", to);
        }
        _ => {}
    }
});
```

One mental model. Zero chaos.

---

## Plugin Loading Strategy

### Compile-time Plugins (Recommended)

Use Cargo feature flags:

```toml
[features]
monitoring = []
git = []
docker = []
```

Safe, idiomatic Rust.

### Runtime Plugins (Advanced)

Dynamic loading via `libloading`:

```
plugins/libmonitoring.so
```

For future marketplace support.

---

## Suggested Project Structure

```
src/
  app.rs              # Hooks implementation
  views/
    dashboard.rs
    settings.rs
  models/
  jobs/
  plugins/
    monitoring/
    git/
    docker/
config/
  development.yaml
  production.yaml
  test.yaml
```

---

## Security & Capability Model (Planned)

Future capability-based permission system:

```rust
ctx.require_capability(Capability::Database);
```

Enables sandboxed plugin execution.

---

## Why Foxerminal?

Foxes are:

* Intelligent
* Adaptive
* Silent but powerful
* Masters of tunnels

Just like terminal applications.

---

## Roadmap

* [ ] Core fork with `with-tui` / `with-http` feature flags
* [ ] EventBus, View system, Screen Router
* [ ] Plugin system (extending Loco Initializer)
* [ ] CLI scaffolding tool (`cargo foxerminal`)
* [ ] Actor-based state engine
* [ ] Hot reload
* [ ] Plugin marketplace
* [ ] Web dashboard mode
* [ ] Distributed mode
* [ ] Capability sandboxing

---

## License

Foxerminal is forked from [Loco](https://github.com/loco-rs/loco) and inherits its licensing.

Licensed under either of:

* Apache License, Version 2.0 ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
* MIT License ([LICENSE-MIT](LICENSE-MIT) or http://opensource.org/licenses/MIT)

at your option.
