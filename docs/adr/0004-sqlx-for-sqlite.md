# SQLiteはRustからsqlxで扱う

DBはSQLiteで、Rust側（ADR-0001）からsqlxで読み書きする。SQLはそのまま書き、`query!`／`query_as!`でコンパイル時に検査する。マイグレーションは`migrate!`でバイナリに埋め込み、起動時に流す。Tauriが内部で使っているtokioとそのまま組み合わせられる。また、SQLを知っていれば書ける点も決め手になった。

## Considered Options

- **rusqlite + rusqlite_migration**：同期処理で単純。YaakやGitButlerなど、成熟したTauriアプリの多くが使っている。ただし、SQLのコンパイル時検査がない。sqlx 0.9とは、リンクするネイティブのSQLiteのバージョンが衝突するので、同じプロジェクトでは併用できない。
- **diesel**：スキーマから型安全なDSLを組み立てる。TypeScriptのDrizzleに最も近い。ただし、ジェネリクスが重くてコンパイルエラーが難解で、ビルドも遅い。テーブルが数個しかないこのアプリでは、学習コストに見合わない。
- **sea-orm／toasty**：テーブル数個のアプリにはORMが重い。toastyは1.0前でもある。
- **tauri-plugin-sql**：JavaScriptからSQLを実行するためのプラグインで、ADR-0001に反する。

## Consequences

- スキーマやクエリを変えたら`cargo sqlx prepare`を実行し、`.sqlx/`をコミットする。CIでは`cargo sqlx prepare --check`で、更新し忘れを検出する。
- sqlxは、WALモードも`create_if_missing`も既定で有効にしない。接続するときに自分で設定する。
- SQLiteでは、NULL許容の推論が外れることがある。外れたら`"col!"`や`"col?"`の別名で補う。
