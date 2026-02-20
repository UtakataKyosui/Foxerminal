# Foxerminal 実現可能性分析: Loco コア流用アプローチ

## 概要

Foxerminal（ターミナルネイティブアプリケーションフレームワーク）を、既存の Rust Web フレームワーク **Loco** をベースに構築する可能性を分析する。

## Loco とは

Loco は「Rails for Rust」を標榜する Web アプリケーションフレームワーク。以下のスタックで構成される:

- **Axum** — HTTP ルーティング / サーバー
- **SeaORM** — 非同期 ORM
- **Tokio** — 非同期ランタイム
- **sidekiq-rs ベースの Worker** — バックグラウンドジョブ
- **CLI (clap)** — コード生成 / タスク実行

## Foxerminal が必要とする機能と Loco の対応

| Foxerminal の要件 | Loco の対応 | 流用可否 |
|---|---|---|
| Ratatui (UI Layer) | なし（HTTP レスポンスが UI） | ❌ 新規実装 |
| Axum (Routing Engine) | Axum ベース | ⚠️ HTTP ルーティングはそのまま使えない。概念の変換が必要 |
| SeaORM (Async ORM) | 統合済み | ✅ そのまま流用 |
| Tokio (Async Runtime) | 統合済み | ✅ そのまま流用 |
| Cron & Job Scheduler | Worker + スケジューラ | ✅ そのまま流用 |
| Plugin System | Initializer trait | ⚠️ 拡張が必要だが土台になる |
| Event Bus | なし | ❌ 新規実装 |
| View Registration | なし | ❌ 新規実装 |

## 流用可能な部分（Loco の下半分）

### そのまま使えるもの
- **SeaORM 統合**: DB 接続プール、マイグレーション管理、モデル生成
- **Worker / Job Queue**: バックグラウンドジョブの定義・実行・リトライ
- **Scheduler / Cron**: 定期実行タスク
- **Config / Environment**: 環境ごとの設定管理（development / production / test）
- **CLI 基盤**: clap ベースのコマンドライン生成

### 拡張すれば使えるもの
- **Initializer**: Loco の Initializer は起動時フック。Plugin trait に拡張可能
- **App trait**: ライフサイクル管理の骨格は流用可能。routes() の代わりに views() を定義

## 差し替えが必要な部分（Loco の上半分）

### HTTP Server → Ratatui Event Loop
- `boot` 関数が `axum::Server` を起動する部分を、Ratatui の `Terminal::draw()` ループに差し替え
- `crossterm` のイベントポーリングと Tokio の非同期タスクを統合

### Controller → View trait
- Loco の Controller は `async fn(State, Request) -> Response` 型の HTTP ハンドラー
- Foxerminal では `fn render(&mut self, frame: &mut Frame, area: Rect)` 型の TUI ビュー
- 根本的に異なるため、新規設計が必要

### HTTP Router → Screen Router
- URL パスベースのルーティング → 画面名ベースの遷移
- ただし Axum の Router 構造を「画面遷移テーブル」として再解釈する余地はある

### Middleware → Event Middleware
- HTTP ミドルウェア（CORS、認証、ロギング）→ キー入力フィルタ、画面遷移ガード等
- 概念的には類似するが、実装は別物

### EventBus（新規）
- Loco にはイベントバスの概念がない
- `tokio::sync::broadcast` をベースに新規実装が必要

## 結論

| 観点 | 評価 |
|---|---|
| 全体の流用率 | 約 60-70%（インフラ層） |
| 新規実装の規模 | 約 30-40%（TUI 層 + イベントバス） |
| ゼロから作る場合との比較 | 大幅にショートカット可能 |
| 技術的リスク | Loco の HTTP 依存を剥がす作業のコスト |
| 推奨アプローチ | Loco をフォークし、Web レイヤーを剥がして TUI レイヤーを載せる |

## 次のステップ

Loco のソースコードを詳細調査し、具体的にどのモジュール / ファイルを:
1. そのまま残すか
2. 修正して流用するか
3. 完全に差し替えるか

をマッピングする。
