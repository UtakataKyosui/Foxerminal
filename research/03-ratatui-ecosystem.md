# Ratatui エコシステム調査

**調査日**: 2026-02-20
**エコシステム規模**: 1,686+ crates, 18.4k GitHub stars, 17.4M+ downloads
**参照**: [awesome-ratatui](https://github.com/ratatui/awesome-ratatui), [Third-party Widgets](https://ratatui.rs/showcase/third-party-widgets/)

---

## 1. エコシステム分類

### 1-1. ウィジェットライブラリ（25+）

Foxerminal の View 内で直接使えるべきもの。

| クレート | 用途 | Foxerminal での活用場面 |
|----------|------|----------------------|
| `tui-textarea` / `ratatui-textarea` | テキストエディタ | 設定画面、入力フォーム |
| `ratatui-code-editor` | シンタックスハイライト付きエディタ | コードビューア、ログ |
| `ratatui-image` | 画像表示 (sixel/kitty/iTerm2) | ダッシュボード、プレビュー |
| `ratatui-explorer` | ファイルエクスプローラ | ファイル管理プラグイン |
| `tui-tree-widget` | ツリー表示 | ディレクトリ、階層データ |
| `tui-scrollview` | スクロール可能ビュー | 長いコンテンツ |
| `tui-term` | 擬似ターミナル | 組み込みシェル |
| `tui-logger` | ログ表示 | デバッグ、モニタリング |
| `tui-menu` | ネスト可能メニュー | ナビゲーション |
| `tui-big-text` | 大きなテキスト | スプラッシュ、ヘッダー |
| `tui-nodes` | ノードグラフ | パイプライン可視化 |
| `tui-piechart` | 円グラフ | ダッシュボード |
| `tui-checkbox` | チェックボックス | 設定画面 |
| `tui-widget-list` | ステートフルリスト | 一覧画面全般 |
| `tui-popup` | ポップアップ | 確認ダイアログ |
| `tui-dialog` | 入力ダイアログ | プロンプト |
| `tui-slider` | スライダー | 設定調整 |
| `throbber-widgets-tui` | ローディング表示 | 非同期処理中の表示 |
| `ratatui-splash-screen` | スプラッシュ画面 | 起動時表示 |
| `tui-prompts` | インタラクティブプロンプト | ウィザード形式の入力 |
| `tui-rain` | 雨エフェクト | デコレーション |
| `edtui` | Vim ライクエディタ | 高機能テキスト編集 |
| `rat-widget` | データ入力/テーブル/ダイアログ | 業務アプリ全般 |
| `ratatui-fretboard` | フレットボード表示 | 音楽アプリ |

### 1-2. エフェクト・アニメーション

| クレート | 用途 | 統合の考慮点 |
|----------|------|-------------|
| `tachyonfx` | シェーダーライクなエフェクト | レンダリングパイプラインへのフック必要 |

tachyonfx は Ratatui の `Buffer` を直接操作してエフェクトを適用する。
Foxerminal の描画ループで `Frame::buffer_mut()` へのアクセスを提供する必要がある。

### 1-3. 入力処理

| クレート | 用途 | 統合の考慮点 |
|----------|------|-------------|
| `tui-input` | ヘッドレス入力処理 | EventBus のキーイベントと連携 |
| `terminput` | 入力バックエンド抽象化 | crossterm 以外のバックエンドも視野に |

### 1-4. ユーティリティ

| クレート | 用途 |
|----------|------|
| `ratatui-macros` | ボイラープレート削減マクロ |
| `ansi-to-tui` | ANSI カラーコード → Ratatui Text 変換 |
| `color-to-tui` | カラー値のパース・変換 |
| `tui-syntax-highlight` | シンタックスハイライト |

### 1-5. 代替バックエンド

| クレート | 用途 | 統合の考慮点 |
|----------|------|-------------|
| `ratzilla` | WASM バックエンド（ブラウザで TUI） | Foxerminal の "Web dashboard mode" に直結 |
| `ratatui-wgpu` | GPU バックエンド | 高速描画 |
| `soft_ratatui` | ソフトウェアレンダリング | GPU なし環境 |
| `egui-ratatui` | egui 統合 | ハイブリッド UI |
| `ratatui-uefi` | UEFI 環境 | 組み込み |

### 1-6. フレームワーク（競合・参考）

| クレート | アーキテクチャ | Foxerminal との関係 |
|----------|--------------|-------------------|
| `tui-realm` | Elm / React パターン | 参考: コンポーネントモデル |
| `tui-react` | React パターン | 参考: 宣言的 UI |
| `widgetui` | Bevy ライク | 参考: ECS パターン |
| `rat-salsa` | イベントキュー + タスク + タイマー | 参考: イベントシステム |

---

## 2. Foxerminal の設計への影響

### 2-1. View trait は Ratatui の `Frame` を直接公開すべき

エコシステムのウィジェットは全て `Frame` と `Rect` を受け取る `Widget` trait か `StatefulWidget` trait を実装している。

```rust
// Ratatui の標準パターン
impl Widget for MyWidget {
    fn render(self, area: Rect, buf: &mut Buffer);
}
impl StatefulWidget for MyWidget {
    type State = MyState;
    fn render(self, area: Rect, buf: &mut Buffer, state: &mut Self::State);
}
```

Foxerminal の `View::render()` が `Frame` を直接渡す設計であれば、
全てのサードパーティウィジェットが `frame.render_widget()` でそのまま使える。

```rust
// Foxerminal の View 内でサードパーティウィジェットをそのまま使う
fn render(&mut self, frame: &mut Frame, area: Rect, ctx: &AppContext) {
    // tui-textarea がそのまま使える
    frame.render_widget(&self.textarea, chunks[0]);

    // ratatui-image がそのまま使える
    frame.render_widget(self.image.clone(), chunks[1]);

    // tui-tree-widget がそのまま使える
    frame.render_stateful_widget(tree, chunks[2], &mut self.tree_state);
}
```

**設計原則**: View trait は Ratatui の上に薄いレイヤーだけを追加する。
描画 API に独自の抽象化を挟まない。

### 2-2. tachyonfx 統合にはレンダリングフックが必要

tachyonfx は描画後の `Buffer` にエフェクトを適用する。
Foxerminal のレンダリングパイプラインにフックポイントを設ける:

```rust
// レンダリングパイプライン
loop {
    terminal.draw(|frame| {
        // 1. View の描画
        current_view.render(frame, area, &ctx);

        // 2. エフェクトの適用（tachyonfx 等）
        for effect in &mut active_effects {
            effect.process(frame.buffer_mut(), area, duration);
        }
    })?;
}
```

View trait にオプショナルなエフェクトフックを追加:

```rust
pub trait View: Send {
    // ... 既存メソッド ...

    /// 描画後にバッファに適用するエフェクトを返す
    fn effects(&mut self) -> Vec<Box<dyn Effect>> {
        vec![]
    }
}
```

### 2-3. バックエンド抽象化で ratzilla (WASM) 対応

Foxerminal は crossterm をデフォルトバックエンドとするが、
ratzilla (WASM) をバックエンドとして差し替え可能にすれば、
README の Roadmap にある "Web dashboard mode" が実現できる。

```toml
[features]
backend-crossterm = ["dep:crossterm"]        # デフォルト: ターミナル
backend-ratzilla = ["dep:ratzilla"]          # オプション: ブラウザ
backend-wgpu = ["dep:ratatui-wgpu"]          # オプション: GPU
```

### 2-4. プラグインとしてのウィジェットパッケージング

エコシステムのウィジェットを Foxerminal プラグインとしてラップし、
ビュー・キーバインド・イベントハンドリングをセットで提供する:

```rust
/// ratatui-explorer を Foxerminal プラグインとしてラップした例
pub struct FileExplorerPlugin;

#[async_trait]
impl Plugin for FileExplorerPlugin {
    fn name(&self) -> &'static str { "file-explorer" }

    async fn register(&self, ctx: &mut PluginContext) -> Result<()> {
        // ビュー登録（ratatui-explorer を内部で使用）
        ctx.views.register(Box::new(FileExplorerView::new()));

        // キーバインド登録
        ctx.keybindings.bind(KeyCode::Char('e'), "navigate:/explorer");

        Ok(())
    }
}

struct FileExplorerView {
    explorer: ratatui_explorer::FileExplorer,  // サードパーティウィジェットをそのまま保持
}

impl View for FileExplorerView {
    fn render(&mut self, frame: &mut Frame, area: Rect, ctx: &AppContext) {
        // サードパーティウィジェットをそのまま描画
        frame.render_widget(&self.explorer, area);
    }

    fn handle_key(&mut self, key: KeyEvent, ctx: &AppContext) -> EventResult {
        // サードパーティウィジェットにイベント委譲
        self.explorer.handle(key);
        EventResult::Handled
    }
}
```

### 2-5. 入力ライブラリとの互換性

`tui-input` や `terminput` は独自の入力処理を提供する。
Foxerminal の EventBus は crossterm の `KeyEvent` をそのまま配送するため、
これらのライブラリは View 内で自由に使える:

```rust
use tui_input::Input;

struct SearchView {
    input: Input,  // tui-input をそのまま使用
}

impl View for SearchView {
    fn handle_key(&mut self, key: KeyEvent, ctx: &AppContext) -> EventResult {
        // tui-input に直接委譲
        self.input.handle_event(&crossterm::event::Event::Key(key));
        EventResult::Handled
    }
}
```

---

## 3. 設計原則まとめ

| 原則 | 内容 |
|------|------|
| **透過的な Frame アクセス** | View::render() は Ratatui の Frame をそのまま渡す。独自の描画抽象化を挟まない |
| **レンダリングフック** | 描画前後にフックポイントを設け、tachyonfx 等のエフェクトライブラリが介入可能にする |
| **バックエンド差し替え** | feature flag でバックエンド（crossterm / ratzilla / wgpu）を切り替え可能にする |
| **プラグインラッピング** | サードパーティウィジェットを Plugin としてパッケージングするパターンを公式に提供 |
| **イベント互換性** | crossterm の KeyEvent をそのまま使い、tui-input 等の入力ライブラリと直接互換にする |
| **Buffer 直接アクセス** | エフェクト系ライブラリのために Frame::buffer_mut() へのアクセスパスを確保する |
