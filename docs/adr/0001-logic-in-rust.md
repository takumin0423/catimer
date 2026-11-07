# ドメインロジックはRust側に置き、Webviewは描画だけを担う

タイマー、期日の通知、繰り返し設定によるタスクの生成は、ウィンドウを閉じていても動き続ける必要がある。Webviewは隠れると処理が絞られ（`backgroundThrottling`）、約5分で止まることがあるうえ、macOSのApp Napも処理を抑える。そこで、ドメインモデル、ユースケース、DB（SQLite）、タイマー、スケジューラ、通知、音はRustで書き、Vite + React + TypeScriptは画面の描画だけを担う。TypeScriptの型はRustから生成する。

## Considered Options

- **Webアプリ**：ブラウザのタブを閉じると、通知も音も止まる。
- **Swift/SwiftUI**：Macとの統合は最良だが、未経験で、Rustを使えない。
- **Electron**：重く、Rustを使う余地がない。
- **RustだけのGUI（egui、iced、gpui）**：Macらしい見た目と操作感を作るのが大変で、UXを重視する方針と合わない。
- **Tauri 2（採用）**：業務で使っているReactとTypeScriptの知識を活かしつつ、中核をRustで書ける。
