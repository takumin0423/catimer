# 技術スタックと開発の進め方

この文書には、使う技術と、日々の開発の進め方をまとめる。後から覆しにくい決定は、理由と一緒に[docs/adr/](./adr/)に書いている。ここではそれを参照するだけにする。バージョンは2026年11月時点のもの。

## 開発の進め方

- **役割分担**：`crates/catimer-core`（ドメインと永続化）は、人が書く。AIは、テストケースの提案、レビュー、Rustのイディオムの解説、ヒントにとどめる。配線（Tauriのコマンド、プラグイン、トレイ、ビルド、CI）とフロントエンドは、AIに任せてよい。
- **作る順番**：端から端まで通る薄い機能を、1本ずつ縦に足していく。1本ごとに親issueを作る。その下に、人が書くドメインの子issue（`ready-for-human`）と、AIに任せる配線・UIの子issue（`ready-for-agent`）を置く。
- **テスト**：`catimer-core`は、TDDで`cargo test`を回す。concept.mdの例（「毎週月曜日のタスクを水曜日に完了」など）は、そのままテストにする。フロントエンドのテストとE2Eは、MVPでは持たない。ロジックが出てきたら、そのとき入れる。
- **学び**：リポジトリには学びのメモを置かない。PRの説明とコミットメッセージに残し、まとまったものはリポジトリの外に書く。
- **配布**：MVPの間は、自分でローカルビルドしたアプリを使う。署名、公証、自動アップデート、App Storeは扱わない。

## ツールのバージョン固定

| 対象 | 方法 |
|---|---|
| Node.js | mise（`mise.toml`） |
| pnpm | `package.json`の`packageManager` |
| Rust | `rust-toolchain.toml`（パッチ番号まで固定。`rustfmt`と`clippy`を含める）。上げるときはPRで明示的に上げる |
| Tauri | 2.x（3.0はalphaなので追わない） |

## Rust

| 用途 | 選んだもの | 補足 |
|---|---|---|
| 構成 | Cargo workspace（`crates/catimer-core`と`src-tauri`） | [ADR-0006](./adr/0006-core-crate-without-tauri.md) |
| DB | sqlx 0.9（SQLiteを同梱。`migrate`、`chrono`） | [ADR-0004](./adr/0004-sqlx-for-sqlite.md)。DBファイルは`app_data_dir()`に置く |
| 日付・時刻 | chrono | 期日は`NaiveDate`。終了予定時刻は壁時計の時刻で持つ（[ADR-0002](./adr/0002-timer-from-end-time.md)） |
| 型の生成 | ts-rs | [ADR-0005](./adr/0005-ts-rs-for-type-generation.md) |
| エラー | thiserror | コマンドのエラーは`{kind, message}`に`Serialize`するenumにする。anyhowは`setup`とテストでだけ使う |
| 非同期 | `tauri::async_runtime`（tokio） | ロックは`std::sync::Mutex`。ロックを持ったまま`.await`するときだけtokioのMutexを使う |
| ログ | `log`とtauri-plugin-log | `catimer-core`は`log`だけに依存する |
| テスト | `cargo test`、必要ならinsta | |
| Lint・整形 | rustfmt（既定の設定）、clippy | `[workspace.lints.clippy]`に`all = "warn"`、`unwrap_used`、`dbg_macro`、`todo`を入れる。pedanticは入れない |

## macOS とメニューバー

| 用途 | 選んだもの | 補足 |
|---|---|---|
| メニューバー | Tauriの`TrayIcon` | 猫はテンプレート画像で描く。残り時間は`set_title`で出す。トレイは必ずメインスレッドで更新する |
| ポップオーバー | tauri-nspanel | フルスクリーンのアプリの上にも出せ、フォーカスを奪わない |
| Dock | `set_activation_policy`（`Accessory`と`Regular`） | ウィンドウを開いている間だけ`Regular`にする |
| 通知 | tauri-plugin-notification | 音は付けない。開発中（`tauri dev`）は「ターミナル」からの通知として出る |
| 音 | NSSound（objc2-app-kit） | 音のファイルは`bundle.resources`で同梱する。試し聞きも同じコマンドで鳴らす |
| 二重起動の防止 | tauri-plugin-single-instance | `RunEvent::Reopen`も扱う |

## フロントエンド

| 用途 | 選んだもの | 補足 |
|---|---|---|
| 基盤 | React 19、TypeScript 7（tsgo）、Vite 8 | |
| 状態 | Rustからのスナップショットを`useSyncExternalStore`で購読 | [ADR-0007](./adr/0007-push-snapshots-to-webviews.md)。TanStack Queryは使わない |
| スタイル | CSS Modules、CSS custom properties | UIのコンポーネントライブラリは使わない |
| Lint | Oxlint（型を使うLintも有効にする） | 型を使うLintはTypeScript 7以上が必要 |
| 整形 | Oxfmt | ベータ版であることは承知のうえで使う |

## CI

PRごとに、GitHub ActionsのmacOSランナーで次を実行する。

- Rust：`cargo fmt --check`、`cargo clippy --all-targets -- -D warnings`、`cargo test`、`cargo sqlx prepare --check`
- フロントエンド：`oxlint`、`oxfmt --check`、`tsc`

アプリのビルド（tauri-action）と、依存の脆弱性やライセンスの検査（cargo-deny）は、MVPの後に回す。
