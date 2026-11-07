# ドメインと永続化は、Tauriに依存しないcrateに分ける

Rust側はCargo workspaceにし、`crates/catimer-core`と`src-tauri`の2つに分ける。

- `crates/catimer-core`：Tauriに依存しない。ドメイン（タスク、タイマーの状態遷移、繰り返し設定）と、sqlxによる永続化を置く。
- `src-tauri`：薄いアダプタ。コマンド、イベント、スケジューラ、トレイ、ポップオーバー、通知、音を置く。

`cargo test -p catimer-core`がTauriやWebKitをビルドせずに済むので、TDDの回転が速くなる。また、ドメインがTauriのAPIに触れていないことを、コンパイラが保証してくれる。`catimer-core`のコードは人が書く（CLAUDE.md）ので、学ぶ範囲をこのcrateに区切る意味もある。

## Consequences

- ts-rsの`derive`は`catimer-core`の型に付ける。
- `target/`はworkspaceのルートにできる。
- 時刻に依存するロジックは、現在時刻を引数で受け取る純粋関数にする。時刻を差し替えるための抽象（`Clock`）は、スケジューラとの境目にだけ置く。
