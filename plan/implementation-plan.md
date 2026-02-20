# Foxerminal 実装計画

**作成日**: 2026-02-20
**ベース**: Loco v0.16.4 フォーク
**参照**: `research/01-feasibility-analysis.md`, `research/02-loco-source-analysis.md`, `research/03-ratatui-ecosystem.md`

---

## 戦略概要

Loco をフォークし、feature flag (`with-tui` / `with-http`) で HTTP 層と TUI 層を切り替え可能にする。
インフラ層（ORM, Worker, Scheduler, Cache, Storage, Mailer）は Loco をそのまま流用し、
プレゼンテーション層（Controller → View, HTTP Router → Screen Router）を Ratatui ベースで新規実装する。

```
フォーク戦略:

  Loco (upstream)
    │
    ├── そのまま維持 ─────── インフラ層 (55%)
    ├── feature-gate 追加 ── コア trait 層 (16%)
    └── 新規実装 ──────────── TUI 層 (29%)
```

---

## Phase 0: プロジェクト基盤構築

**目的**: Loco フォークとワークスペースの初期設定

### 0-1. Loco フォーク & リネーム

- Loco リポジトリをフォーク
- クレート名を変更:
  - `loco-rs` → `foxerminal-core`
  - `loco-gen` → `foxerminal-gen`
  - `loco-cli` → `foxerminal-cli` (後の Phase で対応)
- ライセンス確認（Loco は Apache-2.0 / MIT デュアルライセンス）

### 0-2. Feature Flag 設計

```toml
[features]
default = ["with-tui", "cli", "with-db", "cache_inmem", "bg_sqlt"]

# プレゼンテーション層（排他的に選択）
with-tui = ["dep:ratatui", "dep:crossterm"]
with-http = ["dep:axum", "dep:axum-extra", "dep:tower", "dep:tower-http"]

# 既存の Loco feature flags（そのまま維持）
auth_jwt = ["dep:jsonwebtoken"]
cli = ["dep:clap"]
with-db = ["dep:sea-orm", "dep:sea-orm-migration", "dep:sqlx"]
cache_inmem = ["dep:moka"]
cache_redis = ["dep:bb8-redis", "dep:bb8"]
bg_redis = ["dep:redis", "dep:ulid"]
bg_pg = ["dep:sqlx", "dep:ulid"]
bg_sqlt = ["dep:sqlx", "dep:ulid"]
```

**重要**: 現在 Loco では `axum` がハード依存。これを `with-http` feature に移動するのが Phase 1 の中核作業。

### 0-3. TUI 依存クレートの選定

**コア依存（Foxerminal 自体が使うもの）:**

| 用途 | クレート | バージョン |
|------|----------|-----------|
| TUI レンダリング | `ratatui` | latest |
| ターミナルバックエンド (デフォルト) | `crossterm` | latest |
| イベントバス | `tokio::sync::broadcast` | (Tokio 内蔵) |
| ボイラープレート削減 | `ratatui-macros` | latest |

**バックエンド feature flags（差し替え可能）:**

| バックエンド | クレート | feature flag | 用途 |
|-------------|----------|-------------|------|
| ターミナル | `crossterm` | `backend-crossterm` (default) | 通常の TUI |
| ブラウザ (WASM) | `ratzilla` | `backend-ratzilla` | Web dashboard mode |
| GPU | `ratatui-wgpu` | `backend-wgpu` | 高速描画 |

**エコシステム互換（ユーザーが View 内で自由に使うもの）:**

Foxerminal はこれらをバンドルしない。View trait が Ratatui の `Frame` を直接公開するため、
ユーザーが自身の `Cargo.toml` に追加すればそのまま動作する。

| 分類 | 代表的なクレート |
|------|-----------------|
| テキスト入力 | `tui-input`, `tui-textarea`, `edtui`, `ratatui-code-editor` |
| エフェクト | `tachyonfx` |
| ウィジェット | `tui-tree-widget`, `tui-scrollview`, `ratatui-image`, `tui-popup`, `rat-widget` |
| ユーティリティ | `ansi-to-tui`, `color-to-tui`, `tui-syntax-highlight` |

詳細: `research/03-ratatui-ecosystem.md`

### 成果物
- [ ] フォーク済みリポジトリ
- [ ] `Cargo.toml` の feature flag 定義完了
- [ ] `cargo check --features with-tui --no-default-features` が通る状態（スタブ）

---

## Phase 1: コアの脱 HTTP 化

**目的**: Loco コアから Axum ハード依存を除去し、feature-gate で分離する
**対象ファイル**: `app.rs`, `boot.rs`, `errors.rs`, `config.rs`, `lib.rs`

### 1-1. `src/errors.rs` の分離

**現状**: 25 個のエラーバリアントのうち 7 個が Axum 依存

```rust
// Before
pub enum Error {
    Axum(axum::http::Error),
    JsonRejection(JsonRejection),
    CustomError(StatusCode, ErrorDetail),
    // ...
}

// After
pub enum Error {
    #[cfg(feature = "with-http")]
    Axum(axum::http::Error),
    #[cfg(feature = "with-http")]
    JsonRejection(JsonRejection),
    #[cfg(feature = "with-http")]
    CustomError(StatusCode, ErrorDetail),
    #[cfg(feature = "with-http")]
    InvalidHeaderValue(InvalidHeaderValue),
    #[cfg(feature = "with-http")]
    InvalidHeaderName(InvalidHeaderName),
    #[cfg(feature = "with-http")]
    InvalidMethod(InvalidMethod),
    #[cfg(feature = "with-http")]
    AxumFormRejection(FormRejection),

    #[cfg(feature = "with-tui")]
    TuiRender(String),
    #[cfg(feature = "with-tui")]
    TuiEvent(String),
    // ... 残りの 18 バリアントはそのまま
}
```

### 1-2. `src/app.rs` — Hooks trait の分離

**現状**: 16 メソッド中 7 つが HTTP 依存

```rust
#[async_trait]
pub trait Hooks: Send {
    // ===== 共通メソッド（変更なし） =====
    fn app_version() -> String;
    fn app_name() -> &'static str;
    async fn boot(mode: StartMode, environment: &Environment, config: Config) -> Result<BootResult>;
    fn init_logger(ctx: &AppContext) -> Result<bool>;
    async fn load_config(env: &Environment) -> Result<Config>;
    async fn initializers(ctx: &AppContext) -> Result<Vec<Box<dyn Initializer>>>;
    async fn before_run(app_context: &AppContext) -> Result<()>;
    async fn after_context(ctx: AppContext) -> Result<AppContext>;
    async fn connect_workers(ctx: &AppContext, queue: &Queue) -> Result<()>;
    fn register_tasks(tasks: &mut Tasks);
    async fn on_shutdown(ctx: &AppContext);

    // ===== HTTP 専用（feature-gate） =====
    #[cfg(feature = "with-http")]
    async fn serve(app: AxumRouter, ctx: &AppContext, serve_params: &ServeParams) -> Result<()>;
    #[cfg(feature = "with-http")]
    async fn before_routes(ctx: &AppContext) -> Result<AxumRouter<AppContext>>;
    #[cfg(feature = "with-http")]
    async fn after_routes(router: AxumRouter, ctx: &AppContext) -> Result<AxumRouter>;
    #[cfg(feature = "with-http")]
    fn middlewares(ctx: &AppContext) -> Vec<Box<dyn MiddlewareLayer>>;
    #[cfg(feature = "with-http")]
    fn routes(ctx: &AppContext) -> AppRoutes;

    // ===== TUI 専用（新規） =====
    #[cfg(feature = "with-tui")]
    fn views(ctx: &AppContext) -> ViewRegistry;
    #[cfg(feature = "with-tui")]
    fn keybindings(ctx: &AppContext) -> KeybindingMap;
    #[cfg(feature = "with-tui")]
    async fn before_render(ctx: &AppContext) -> Result<()>;

    // ===== DB（既存の feature-gate） =====
    #[cfg(feature = "with-db")]
    async fn truncate(ctx: &AppContext) -> Result<()>;
    #[cfg(feature = "with-db")]
    async fn seed(ctx: &AppContext, path: &Path) -> Result<()>;
}
```

### 1-3. `src/app.rs` — Initializer trait の分離

```rust
#[async_trait]
pub trait Initializer: Sync + Send {
    fn name(&self) -> String;
    async fn before_run(&self, app_context: &AppContext) -> Result<()>;
    async fn check(&self, app_context: &AppContext) -> Result<Option<Check>>;

    #[cfg(feature = "with-http")]
    async fn after_routes(&self, router: AxumRouter, ctx: &AppContext) -> Result<AxumRouter>;

    #[cfg(feature = "with-tui")]
    async fn after_setup(&self, ctx: &AppContext) -> Result<()>;
}
```

### 1-4. `src/boot.rs` — StartMode の拡張

```rust
pub enum StartMode {
    #[cfg(feature = "with-http")]
    ServerOnly,
    #[cfg(feature = "with-http")]
    ServerAndWorker,

    #[cfg(feature = "with-tui")]
    TuiOnly,
    #[cfg(feature = "with-tui")]
    TuiAndWorker,

    WorkerOnly { tags: Option<Vec<String>> },
    All,
}
```

`boot.rs` の `create_context()` はそのまま流用。`run_app()` と `start()` に TUI 分岐を追加。

### 1-5. `src/config.rs` — TUI 設定の追加

```rust
pub struct Config {
    pub logger: Logger,
    #[cfg(feature = "with-http")]
    pub server: Server,
    #[cfg(feature = "with-tui")]
    pub tui: TuiConfig,
    #[cfg(feature = "with-db")]
    pub database: Database,
    pub cache: CacheConfig,
    pub queue: Option<QueueConfig>,
    pub auth: Option<Auth>,
    pub workers: Workers,
    pub mailer: Option<Mailer>,
    pub initializers: Option<Initializers>,
    pub settings: Option<serde_json::Value>,
    pub scheduler: Option<scheduler::Config>,
}

#[cfg(feature = "with-tui")]
pub struct TuiConfig {
    pub tick_rate_ms: u64,          // イベントループのティックレート（デフォルト: 250）
    pub frame_rate_ms: u64,         // 描画フレームレート（デフォルト: 16 ≈ 60fps）
    pub mouse_capture: bool,        // マウスイベントのキャプチャ
    pub paste_capture: bool,        // ペーストイベントのキャプチャ
    pub default_view: String,       // 初期表示ビュー
}
```

### 1-6. `src/lib.rs` と `src/prelude.rs` の条件分岐

```rust
// src/lib.rs
#[cfg(feature = "with-http")]
pub mod controller;
#[cfg(feature = "with-tui")]
pub mod tui;

// src/prelude.rs
#[cfg(feature = "with-http")]
pub use crate::controller::*;
#[cfg(feature = "with-tui")]
pub use crate::tui::prelude::*;
```

### 成果物
- [ ] `cargo check --features with-tui,with-db,cli` がコンパイル通過
- [ ] `cargo check --features with-http,with-db,cli` で既存の Loco 動作が維持される
- [ ] Axum がハード依存ではなくなっている

---

## Phase 2: TUI コアシステムの実装

**目的**: Foxerminal の心臓部となる TUI 固有モジュールを実装する

### エコシステム互換性の設計原則（全サブタスク共通）

Ratatui エコシステム（1,686+ crates）との互換性を最優先とする。
詳細は `research/03-ratatui-ecosystem.md` を参照。

1. **透過的な Frame アクセス**: View::render() は Ratatui の `Frame` をそのまま渡す。独自の描画抽象化を挟まない。これにより全サードパーティウィジェット（tui-textarea, ratatui-image, tui-tree-widget 等）が `frame.render_widget()` でそのまま動作する
2. **レンダリングフック**: 描画後に `Buffer` を操作するフックポイントを設け、tachyonfx 等のエフェクトライブラリが介入可能にする
3. **バックエンド差し替え**: feature flag でバックエンド（crossterm / ratzilla / wgpu）を切り替え可能にする。ratzilla 対応で "Web dashboard mode" が実現できる
4. **イベント互換性**: crossterm の `KeyEvent` をそのまま配送し、tui-input / terminput 等の入力ライブラリと直接互換にする
5. **バンドルしない**: エコシステムのウィジェットを Foxerminal にバンドルしない。ユーザーが必要なものを `Cargo.toml` に追加すればそのまま動く

### 2-1. EventBus（イベントバス）

README の "Everything is an Event" 哲学の根幹。

```
src/tui/
  event/
    mod.rs          -- EventBus, AppEvent enum
    bus.rs          -- broadcast ベースの配信エンジン
    handler.rs      -- イベントハンドラ trait
```

```rust
// src/tui/event/mod.rs

/// フレームワーク全体の統一イベント型
#[derive(Clone, Debug)]
pub enum AppEvent {
    // ターミナルイベント
    Key(crossterm::event::KeyEvent),
    Mouse(crossterm::event::MouseEvent),
    Resize(u16, u16),
    Paste(String),

    // アプリケーションイベント
    ViewChanged { from: String, to: String },
    Tick,

    // インフライベント
    JobCompleted { name: String, result: Result<(), String> },
    CronTriggered { name: String },
    DbEvent { table: String, operation: String },

    // ユーザー定義
    Custom(Box<dyn std::any::Any + Send + Sync>),
}

/// イベントバス — tokio::sync::broadcast ベース
pub struct EventBus {
    sender: broadcast::Sender<AppEvent>,
}

impl EventBus {
    pub fn new(capacity: usize) -> Self;
    pub fn emit(&self, event: AppEvent);
    pub fn subscribe(&self) -> broadcast::Receiver<AppEvent>;
}
```

### 2-2. Terminal Manager（ターミナル管理）

Ratatui + crossterm のセットアップ・ティアダウンを一元管理。

```
src/tui/
  terminal.rs       -- Terminal の初期化・復元・描画ループ
```

```rust
pub struct TerminalManager {
    terminal: Terminal<CrosstermBackend<Stdout>>,
    config: TuiConfig,
}

impl TerminalManager {
    pub fn new(config: &TuiConfig) -> Result<Self>;
    pub fn restore(&mut self) -> Result<()>;  // パニック時にもターミナル復元
    pub fn draw<F>(&mut self, f: F) -> Result<()>
    where
        F: FnOnce(&mut Frame);
}
```

**重要**: `panic::set_hook` でパニック時のターミナル復元を保証する。

### 2-3. View trait & ViewRegistry（ビューシステム）

Loco の Controller に相当する TUI 層の中核。

```
src/tui/
  view/
    mod.rs          -- View trait, ViewRegistry
    registry.rs     -- ビューの登録・検索
    lifecycle.rs    -- ビューのライフサイクルフック
```

```rust
/// TUI ビューの基本 trait
#[async_trait]
pub trait View: Send {
    /// このビューのルートパス（例: "/", "/monitor", "/settings"）
    fn route(&self) -> &'static str;

    /// ビュー名（デバッグ・ログ用）
    fn name(&self) -> &'static str;

    /// 状態の初期化（ビューがアクティブになったとき）
    async fn on_mount(&mut self, ctx: &AppContext) -> Result<()> { Ok(()) }

    /// 状態のクリーンアップ（ビューが非アクティブになったとき）
    async fn on_unmount(&mut self, ctx: &AppContext) -> Result<()> { Ok(()) }

    /// キーイベント処理。Handled を返すとイベント伝播が止まる
    fn handle_key(&mut self, key: KeyEvent, ctx: &AppContext) -> EventResult {
        EventResult::Ignored
    }

    /// 毎フレーム描画
    /// Frame を直接公開するため、全 Ratatui サードパーティウィジェットがそのまま使える:
    ///   frame.render_widget(my_tui_textarea, area);
    ///   frame.render_widget(my_ratatui_image, area);
    ///   frame.render_stateful_widget(my_tree, area, &mut state);
    fn render(&mut self, frame: &mut Frame, area: Rect, ctx: &AppContext);

    /// 描画後にバッファに適用するエフェクトを返す（tachyonfx 等のエフェクトライブラリ用）
    /// デフォルトは空。tachyonfx を使う場合にオーバーライドする
    fn post_render_effects(&mut self, _ctx: &AppContext) -> Vec<Box<dyn FnOnce(&mut Buffer, Rect)>> {
        vec![]
    }

    /// Tick イベント（定期更新）
    async fn on_tick(&mut self, ctx: &AppContext) -> Result<()> { Ok(()) }
}

pub enum EventResult {
    Handled,
    Ignored,
    Navigate(String),  // 別ビューへの遷移
    Quit,
}

/// ビューのレジストリ
pub struct ViewRegistry {
    views: HashMap<String, Box<dyn View>>,
    current: String,
    history: Vec<String>,
}
```

### 2-4. Screen Router（画面遷移）

URL パスベースのルーティングを TUI の画面遷移に転用。

```
src/tui/
  router/
    mod.rs          -- Router 本体
    transition.rs   -- 遷移アニメーション（オプション）
```

```rust
pub struct ScreenRouter {
    registry: ViewRegistry,
    guards: Vec<Box<dyn RouteGuard>>,  // 遷移ガード（認証チェック等）
}

/// 遷移ガード（Middleware に相当）
#[async_trait]
pub trait RouteGuard: Send + Sync {
    async fn can_navigate(
        &self,
        from: &str,
        to: &str,
        ctx: &AppContext,
    ) -> Result<bool>;
}

impl ScreenRouter {
    pub fn navigate(&mut self, path: &str) -> Result<()>;
    pub fn back(&mut self) -> Result<()>;
    pub fn current(&self) -> &str;
}
```

### 2-5. Keybinding Manager（キーバインド管理）

```
src/tui/
  keybinding.rs     -- キーバインドの定義・マッチング
```

```rust
pub struct KeybindingMap {
    bindings: Vec<Keybinding>,
}

pub struct Keybinding {
    pub key: KeyEvent,
    pub action: String,          // アクション名
    pub context: Option<String>, // 特定ビューでのみ有効
    pub description: String,     // ヘルプ表示用
}
```

### 成果物
- [ ] `src/tui/` モジュール一式
- [ ] EventBus が `tokio::sync::broadcast` で動作
- [ ] View trait で画面描画が可能
- [ ] Screen Router で `navigate("/path")` による画面遷移が動作
- [ ] ターミナル初期化・復元が安全に動作

---

## Phase 3: プラグインシステム

**目的**: Loco の Initializer を拡張し、Foxerminal の Plugin システムを実装

### 3-1. Plugin trait

Loco の `Initializer` を拡張し、README に記載した Plugin 仕様を実現する。

```
src/tui/
  plugin/
    mod.rs          -- Plugin trait, PluginContext
    registry.rs     -- プラグインの登録・管理
    lifecycle.rs    -- ライフサイクル管理
```

```rust
#[async_trait]
pub trait Plugin: Send + Sync {
    fn name(&self) -> &'static str;

    /// プラグインの初期化。PluginContext を通じてフレームワークにリソースを登録する
    async fn register(&self, ctx: &mut PluginContext) -> Result<()>;

    /// アプリケーション起動前のフック
    async fn on_boot(&self, app_ctx: &AppContext) -> Result<()> { Ok(()) }

    /// アプリケーション終了時のフック
    async fn on_shutdown(&self, app_ctx: &AppContext) -> Result<()> { Ok(()) }

    /// イベントハンドリング
    async fn on_event(&self, event: &AppEvent, app_ctx: &AppContext) -> Result<()> { Ok(()) }

    /// ヘルスチェック（Loco の doctor 連携）
    async fn check(&self, app_ctx: &AppContext) -> Result<Option<Check>> { Ok(None) }
}
```

### 3-2. PluginContext

```rust
pub struct PluginContext<'a> {
    pub views: &'a mut ViewRegistry,
    pub router: &'a mut ScreenRouter,
    pub scheduler: &'a mut Scheduler,      // Loco のスケジューラをそのまま流用
    pub event_bus: &'a EventBus,
    pub shared_store: &'a mut SharedStore,  // Loco の SharedStore をそのまま流用
    pub keybindings: &'a mut KeybindingMap,
}
```

### 3-3. Loco Initializer との統合

Loco の `Hooks::initializers()` と Foxerminal の `Plugin` を橋渡しする。

```rust
// Plugin を Loco の Initializer としてラップするアダプタ
struct PluginInitializerAdapter(Box<dyn Plugin>);

#[async_trait]
impl Initializer for PluginInitializerAdapter {
    fn name(&self) -> String { self.0.name().to_string() }
    async fn before_run(&self, ctx: &AppContext) -> Result<()> { self.0.on_boot(ctx).await }
    async fn check(&self, ctx: &AppContext) -> Result<Option<Check>> { self.0.check(ctx).await }
}
```

### 成果物
- [ ] Plugin trait 定義
- [ ] PluginContext による各サブシステムへのアクセス
- [ ] Loco Initializer との互換レイヤー
- [ ] サンプルプラグイン（例: SystemMonitor プラグイン）

---

## Phase 4: ブートシーケンス統合

**目的**: Phase 1-3 の成果物を統合し、`cargo run` で TUI アプリが起動する状態にする

### 4-1. TUI メインループ

Loco の `boot.rs::start()` に TUI 起動パスを追加。

```rust
// boot.rs に追加
#[cfg(feature = "with-tui")]
async fn start_tui<H: Hooks>(boot_result: BootResult) -> Result<()> {
    let ctx = boot_result.app_context;
    let event_bus = EventBus::new(256);
    let mut terminal_manager = TerminalManager::new(&ctx.config.tui)?;

    // プラグイン登録
    let plugins = H::plugins(&ctx).await?;
    let mut plugin_ctx = PluginContext::new(/* ... */);
    for plugin in &plugins {
        plugin.register(&mut plugin_ctx).await?;
    }

    // ワーカー起動（バックグラウンド）
    if let Some(queue) = &ctx.queue_provider {
        // Loco のワーカーシステムをそのまま使用
        tokio::spawn(async move { /* worker loop */ });
    }

    // スケジューラ起動（バックグラウンド）
    if boot_result.run_scheduler {
        // Loco のスケジューラをそのまま使用
        tokio::spawn(async move { /* scheduler loop */ });
    }

    // メインイベントループ
    loop {
        // 1. crossterm イベントをポーリング
        // 2. AppEvent に変換して EventBus に emit
        // 3. 現在のビューにイベントを配送
        // 4. ビューを描画
        // 5. Tick イベント処理

        terminal_manager.draw(|frame| {
            router.current_view().render(frame, frame.area(), &ctx);
        })?;

        // crossterm::event::poll + tokio::select! で
        // ターミナルイベントと非同期タスクを統合
        tokio::select! {
            event = crossterm_events.next() => { /* handle terminal event */ }
            _ = tick_interval.tick() => { /* handle tick */ }
            _ = shutdown_signal() => { break; }  // Loco の shutdown_signal を流用
        }
    }

    terminal_manager.restore()?;
    H::on_shutdown(&ctx).await;
    Ok(())
}
```

### 4-2. Tokio + crossterm の統合パターン

TUI イベントループと Tokio の非同期ランタイムの共存が最大の技術的チャレンジ。

```
┌─────────────────────────────────────────────┐
│                 Tokio Runtime                │
│                                             │
│  ┌──────────────┐  ┌────────────────────┐   │
│  │ TUI Event    │  │ Background Tasks   │   │
│  │ Loop (main)  │  │                    │   │
│  │              │  │ ┌────────────────┐ │   │
│  │ crossterm    │  │ │   Workers      │ │   │
│  │ poll ──┐     │  │ └────────────────┘ │   │
│  │        │     │  │ ┌────────────────┐ │   │
│  │ EventBus ◄───┼──┤ │   Scheduler    │ │   │
│  │        │     │  │ └────────────────┘ │   │
│  │   render     │  │ ┌────────────────┐ │   │
│  │              │  │ │   DB queries   │ │   │
│  └──────────────┘  │ └────────────────┘ │   │
│                    └────────────────────┘   │
└─────────────────────────────────────────────┘
```

**ポイント**: `crossterm::event::EventStream`（async 対応）を使い、`tokio::select!` でターミナルイベントとバックグラウンドタスクの完了を同時に待機する。

### 4-3. Graceful Shutdown

Loco の `shutdown_signal()` を流用。加えて TUI 固有のクリーンアップを実行:

1. `on_shutdown` フック実行（プラグイン → アプリ）
2. ワーカーキューの停止
3. DB コネクション切断
4. ターミナルモード復元（raw mode 解除、カーソル表示、alternate screen 解除）

### 成果物
- [ ] `cargo run` で TUI アプリが起動
- [ ] キー入力でビュー遷移が動作
- [ ] バックグラウンドワーカーが TUI と並行で動作
- [ ] Ctrl+C でクリーンにシャットダウン

---

## Phase 5: 開発者体験（DX）

**目的**: Foxerminal を使ってアプリを作る開発者のための CLI ツールとテンプレートを整備

### 5-1. CLI コマンド (`cargo foxerminal`)

Loco の `loco-cli` をベースに、TUI 固有のコマンドを追加。

```
cargo foxerminal new <app-name>        # 新規プロジェクト生成
cargo foxerminal generate view <name>  # ビューのスキャフォールド
cargo foxerminal generate plugin <name> # プラグインのスキャフォールド
cargo foxerminal generate model <name> # モデル生成（Loco と同じ）
cargo foxerminal generate worker <name> # ワーカー生成（Loco と同じ）
cargo foxerminal db migrate            # DB マイグレーション（Loco と同じ）
cargo foxerminal task <name>           # タスク実行（Loco と同じ）
cargo foxerminal doctor                # ヘルスチェック（Loco と同じ）
```

### 5-2. スターターテンプレート

```
starters/
  minimal/         -- 最小構成（TUI のみ、DB なし）
  full/            -- フル構成（TUI + DB + Worker + Scheduler）
  plugin-example/  -- プラグイン開発用テンプレート
```

### 5-3. View スキャフォールド

```rust
// generate view monitor で生成されるコード
use foxerminal::prelude::*;

pub struct MonitorView {
    // state
}

#[async_trait]
impl View for MonitorView {
    fn route(&self) -> &'static str { "/monitor" }
    fn name(&self) -> &'static str { "Monitor" }

    async fn on_mount(&mut self, ctx: &AppContext) -> Result<()> {
        // 初期データロード
        Ok(())
    }

    fn handle_key(&mut self, key: KeyEvent, ctx: &AppContext) -> EventResult {
        match key.code {
            KeyCode::Char('q') => EventResult::Quit,
            KeyCode::Esc => EventResult::Navigate("/".to_string()),
            _ => EventResult::Ignored,
        }
    }

    fn render(&mut self, frame: &mut Frame, area: Rect, ctx: &AppContext) {
        let block = Block::default()
            .title(" Monitor ")
            .borders(Borders::ALL);
        frame.render_widget(block, area);
    }
}
```

### 成果物
- [ ] `cargo foxerminal new` でプロジェクト生成
- [ ] `cargo foxerminal generate view/plugin/model` でコード生成
- [ ] スターターテンプレート 3 種

---

## Phase 6: テスト基盤 & サンプルアプリ

### 6-1. TUI テストユーティリティ

Loco の `src/testing/` を TUI 向けに再実装。

```rust
// src/tui/testing.rs
pub struct TestTerminal {
    // headless terminal backend for testing
}

impl TestTerminal {
    pub fn new(width: u16, height: u16) -> Self;
    pub fn send_key(&mut self, key: KeyEvent);
    pub fn current_view(&self) -> &str;
    pub fn rendered_text(&self) -> String;  // 描画されたテキストを取得
}
```

### 6-2. サンプルアプリ: System Monitor

Foxerminal の機能を網羅的に使うデモアプリ。

```
examples/
  system-monitor/
    src/
      app.rs          -- Hooks 実装
      views/
        dashboard.rs  -- ダッシュボード（CPU, メモリ, ディスク）
        processes.rs  -- プロセスリスト
        logs.rs       -- ログビューア
      plugins/
        sysinfo.rs    -- システム情報収集プラグイン
      workers/
        metrics.rs    -- メトリクス収集ワーカー
      models/
        metric.rs     -- メトリクスの DB モデル
```

### 成果物
- [ ] ヘッドレステスト環境
- [ ] System Monitor サンプルアプリが動作
- [ ] 各モジュールのユニットテスト

---

## フェーズ間の依存関係

```
Phase 0 ─── プロジェクト基盤
    │
    ▼
Phase 1 ─── コア脱HTTP化
    │
    ▼
Phase 2 ─── TUIコアシステム ──────┐
    │                            │
    ▼                            ▼
Phase 3 ─── プラグインシステム    │
    │                            │
    ▼                            │
Phase 4 ─── ブート統合 ◄─────────┘
    │
    ├──────────┐
    ▼          ▼
Phase 5    Phase 6
  DX       テスト & サンプル
```

Phase 0 → 1 は厳密に直列。Phase 2 と 3 は一部並行可能（EventBus は Plugin より先に必要）。
Phase 4 は 1-3 の全成果物を統合する。Phase 5 と 6 は Phase 4 完了後に並行実行可能。

---

## リスクと対策

| リスク | 影響度 | 対策 |
|--------|--------|------|
| Loco の Axum ハード依存が予想以上に深い | 高 | Phase 1 で早期に検証。困難なら Loco のインフラ層だけを別クレートとして切り出す |
| crossterm と Tokio の統合でデッドロック | 中 | `crossterm::event::EventStream` (async) を使い、`tokio::select!` で統合。blocking な `event::read()` は使わない |
| Loco upstream の更新への追従コスト | 中 | インフラ層のみ定期的にマージ。TUI 層は独自パスなので衝突しにくい |
| 描画パフォーマンス（60fps + async タスク） | 中 | `ratatui` の差分描画に依存。重い処理は必ずバックグラウンドワーカーで実行 |
| feature flag の組み合わせ爆発 | 低 | CI で `with-tui` / `with-http` の両方をテスト。`with-tui` をデフォルトに |

---

## 推定作業量

| Phase | 新規コード量（概算） | 主要作業 |
|-------|---------------------|----------|
| Phase 0 | ~100 行 | フォーク、Cargo.toml 設定 |
| Phase 1 | ~500 行（変更） | feature-gate 追加、条件コンパイル |
| Phase 2 | ~2,000 行 | EventBus, View, Router, Terminal |
| Phase 3 | ~800 行 | Plugin trait, PluginContext, アダプタ |
| Phase 4 | ~600 行 | メインループ、boot 統合 |
| Phase 5 | ~1,500 行 | CLI, テンプレート、コード生成 |
| Phase 6 | ~1,200 行 | テスト基盤、サンプルアプリ |
| **合計** | **~6,700 行** | |

Loco からの流用: ~5,400 行（変更なし）+ ~1,600 行（修正） = ~7,000 行

**ゼロから書いた場合の推定: ~13,000+ 行** → フォーク戦略で約半分に削減。
