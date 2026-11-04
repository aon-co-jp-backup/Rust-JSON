# Rust-JSON

Rustで書かれた、寛容(lenient)モードと厳密(strict)モードの両方を1つの
クレートから提供するJSON処理ライブラリ。パース結果は常に標準の
`serde_json::Value`——新しいデータモデルは導入しない。

## これは何か

- **寛容モード(`parse`)**: JSON5/JSONCスタイルの拡張構文を受理する
  JSONスーパーセット——トレイリングコンマ・コメント(`//`・`/* */`)・
  クォート無しキー・シングルクォート文字列。
- **厳密モード(`parse_strict`)**: RFC 8259準拠のJSONのみを受理
  (内部的に`serde_json::from_str`へ委譲)。
- **`extract_path`**: ドット/ブラケット記法(`stats.damage`・
  `bonuses[1]`・`items[2].name`)によるサーバー側の部分抽出。
- **`unsafe`コード不使用・外部パースクレート非依存**
  (`serde_json`は値モデル/厳密モードの委譲先としてのみ使用)。

## なぜ「ハイブリッド」なのか

既存のRust JSONエコシステム(`serde_json`・`simd-json`・`sonic-rs`・
`json5`・`json_five`・`json-commons`・`valq`等)をGoogle・GitHub・
crates.ioで調査した上で、良い部分を組み合わせて設計した(詳細は
`CLAUDE.md`参照)。要点:

- SIMD最適化(`simd-json`/`sonic-rs`)による高速化は、`unsafe`コードとの
  トレードオフを伴うため意図的に採用しない。
- 寛容パース(`json5`系)と厳密パース(`serde_json`)を、別々のクレートを
  自分で組み合わせるのではなく、1つのクレートの2つの関数として提供する。
- パス抽出(`extract_path`)は`json-commons`の`get_path()`と同じ発想——
  独立に収斂した設計であることを調査で確認済み。

## 使用例

```rust
use rust_json::{parse, parse_strict, extract_path, to_string};

// 寛容モード: コメント・トレイリングコンマ・クォート無しキーを受理
let value = parse(r#"{
    // 装備アイテム
    name: 'longsword',
    stats: { damage: 12, bonuses: [1, 2, 3,] },
}"#).unwrap();

// 厳密モード: 上と同じ入力は拒否される(RFC 8259のみ受理)
assert!(parse_strict(r#"{name: 'longsword'}"#).is_err());

// 部分抽出
let damage = extract_path(&value, "stats.damage");

// 常に厳密JSONとして出力
let canonical = to_string(&value);
```

## ビルド・テスト

```bash
cargo test
```

## 関連プロジェクト

- [open-runo](https://github.com/aon-co-jp/open-runo)・
  [poem-cosmo-tauri](https://github.com/aon-co-jp/poem-cosmo-tauri) —
  旧居場所(`open-runo-rjson`→`open-runo-rustjson`)。当面は参照実装として
  残置。
- [open-raid-z](https://github.com/aon-co-jp/open-raid-z)・
  [aruaru-db](https://github.com/aon-co-jp/aruaru-db) — ZFS互換・
  ACID互換のデータ層でのJSON処理を、今後本クレートへ置き換える移行先。

## ライセンス

Apache-2.0 OR MIT

## API 方針: REST 全廃・gRPC / Connect へ (2026-11-04)

### 日本語
- **REST は全廃する。新しい API は REST で作らない。** ブラウザや `curl` から呼ぶ API は **Connect の単純呼び出し**(`POST /<パッケージ>.<Service>/<Method>`、`content-type: application/json`、本文は JSON)、サービス間は **gRPC / gRPC-web**(RPoem の管理API `open_runo.admin.v1.Admin` と同じ流儀)。ストリーミングは、必要になってから。
- **既存の REST は、触るときに順に移す。** 移したら旧 REST の経路は消す。移行が終わるまで残るが、**新規には追加しない**。
- **JSON は RS-JSON に統一**する(読み込みは `from_str_secure` / `parse_secure` でサイズ・深さ・重複キーの上限つき、書き出しは `rust_json::fast`)。ブラウザの WASM に `prost` は入れない(サイズが増えるため)。
- **例外(自分たちの API ではなく、相手の規格が決めているもの)**: ACME、OAuth/OIDC のコールバック、第三者(GitHub など)からの Webhook、ヘルスチェック、静的ファイル配信、OTLP など。これらは規格どおりの HTTP を受ける。
- 参考実装: `RPoem`(`open_runo.admin.v1.Admin`)、`open-easy-web`(`server/src/connect.rs`、`proto/open_easy_web/admin/v1/admin.proto`)。
- ※ このリポジトリのコードが既に方針どおりかどうかは、各リポジトリの現状による。この節は方針の記録であり、コードの変更を意味しない。

### English
- **REST is being abolished. New APIs are not built as REST.** APIs called from browsers or `curl` use **Connect unary calls** (`POST /<package>.<Service>/<Method>`, `content-type: application/json`, JSON body); service-to-service calls use **gRPC / gRPC-web** (the same style as RPoem's admin API `open_runo.admin.v1.Admin`). Streaming only when it is actually needed.
- **Existing REST endpoints are migrated one by one, when they are touched.** The old REST route is removed once migrated. Until migration is finished they remain, but **no new ones are added**.
- **JSON is unified on RS-JSON**: reading with `from_str_secure` / `parse_secure` (size, depth and duplicate-key limits), writing with `rust_json::fast`. `prost` is not put into browser WASM (it would grow the binary).
- **Exceptions (shapes dictated by someone else's standard, not our own API)**: ACME, OAuth/OIDC callbacks, webhooks from third parties (e.g. GitHub), health checks, static file serving, OTLP, and so on. These accept plain HTTP as the standard requires.
- Reference implementations: `RPoem` (`open_runo.admin.v1.Admin`) and `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Note: whether the code in a given repository already follows this policy depends on that repository. This section records the policy; it does not mean the code has been changed.

### Deutsch (KI-Übersetzung, redaktionell überarbeitet; noch nicht von Muttersprachlern geprüft)
- **REST wird vollständig abgeschafft; neue APIs entstehen nicht mehr als REST.** APIs, die aus dem Browser oder per `curl` aufgerufen werden, verwenden **unäre Connect-Aufrufe** (`POST /<Paket>.<Service>/<Methode>`, `content-type: application/json`, JSON-Body). Zwischen Diensten kommen **gRPC bzw. gRPC-web** zum Einsatz, im selben Stil wie die Admin-API von RPoem (`open_runo.admin.v1.Admin`). Streaming wird erst ergänzt, wenn es tatsächlich gebraucht wird.
- **Bestehende REST-Endpunkte werden schrittweise umgestellt, und zwar immer dann, wenn man ohnehin an ihnen arbeitet.** Sobald ein Endpunkt umgestellt ist, wird die alte REST-Route entfernt. Bis zum Abschluss der Migration bleiben die übrigen bestehen, es werden aber **keine neuen mehr hinzugefügt**.
- **JSON wird einheitlich über RS-JSON verarbeitet**: Gelesen wird mit `from_str_secure` bzw. `parse_secure` (mit Obergrenzen für Größe, Verschachtelungstiefe und doppelte Schlüssel), geschrieben mit `rust_json::fast`. `prost` wird nicht in das Browser-WASM aufgenommen, da es die Datei vergrößern würde.
- **Ausnahmen (Schnittstellen, deren Form ein fremder Standard vorgibt und die nicht zu unserer eigenen API gehören)**: ACME, OAuth/OIDC-Callbacks, Webhooks von Drittanbietern (z. B. GitHub), Health-Checks, die Auslieferung statischer Dateien, OTLP und Ähnliches. Diese nehmen, wie vom jeweiligen Standard verlangt, gewöhnliches HTTP entgegen.
- Referenzimplementierungen: `RPoem` (`open_runo.admin.v1.Admin`) und `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Hinweis: Ob der Code eines einzelnen Repositorys diese Richtlinie bereits erfüllt, hängt vom jeweiligen Repository ab. Dieser Abschnitt dokumentiert lediglich die Richtlinie; er besagt nicht, dass der Code schon angepasst wurde.

### Русский (перевод выполнен с помощью ИИ и отредактирован; носителями языка пока не проверен)
- **От REST мы полностью отказываемся; новые API больше не создаются в стиле REST.** API, к которым обращаются из браузера или через `curl`, используют **унарные вызовы Connect** (`POST /<пакет>.<Service>/<Method>`, `content-type: application/json`, тело запроса — JSON). Для взаимодействия между сервисами применяются **gRPC и gRPC-web** — в том же стиле, что и административный API RPoem (`open_runo.admin.v1.Admin`). Потоковая передача данных добавляется лишь тогда, когда в ней действительно возникает необходимость.
- **Существующие REST-эндпоинты переводятся поэтапно — по мере того, как с ними приходится работать.** После перевода старый REST-маршрут удаляется. Пока миграция не завершена, оставшиеся эндпоинты сохраняются, но **новые не добавляются**.
- **Работа с JSON унифицирована на основе RS-JSON**: чтение — через `from_str_secure` или `parse_secure` (с ограничениями на размер, глубину вложенности и повторяющиеся ключи), запись — через `rust_json::fast`. `prost` в браузерный WASM не включается, поскольку это увеличило бы размер файла.
- **Исключения (интерфейсы, форму которых определяет чужой стандарт, а не наш собственный API)**: ACME, обратные вызовы OAuth/OIDC, вебхуки сторонних сервисов (например, GitHub), проверки работоспособности, раздача статических файлов, OTLP и тому подобное. Такие интерфейсы принимают обычный HTTP, как того требует соответствующий стандарт.
- Эталонные реализации: `RPoem` (`open_runo.admin.v1.Admin`) и `open-easy-web` (`server/src/connect.rs`, `proto/open_easy_web/admin/v1/admin.proto`).
- Примечание: соответствует ли код конкретного репозитория этой политике, зависит от самого репозитория. Данный раздел лишь фиксирует политику и не означает, что код уже изменён.
