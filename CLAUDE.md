# 開発方針・開発環境ルール(Rust-JSON)

## このリポジトリの役割(2026-07-17、RJSON/RustJSONから改称・独立)

`Rust-JSON`は、以前`open-runo`/`poem-cosmo-tauri`内のクレート
(`open-runo-rjson`→`open-runo-rustjson`)として存在していた、
石塚正浩(aon CEO)発案・Claude実装のJSON拡張仕様を、**独立リポジトリ
として切り出し、改称したもの**。作業ドライブは`F:\open-runo\Rust-JSON`、
VPS上は`/root/Rust-JSON`、GitHubは
[aon-co-jp/Rust-JSON](https://github.com/aon-co-jp/Rust-JSON)。
今後の読み書き・pushはすべてこのリポジトリを正とする。

## 設計方針: 「良いとこ取りのハイブリッド」(2026-07-17、EN/JP調査済み)

ユーザー指示により、既存のRust JSON エコシステムをGoogle・GitHub・
crates.ioで調査した上で、ゼロから作るのではなく良い部分を組み合わせる
方針で設計した。調査結果の要約:

| クレート | 強み | 弱み |
|---|---|---|
| `serde_json` | 事実上の標準、`Value`型で動的操作も可能、追加依存なし | 拡張構文(コメント・トレイリングカンマ等)は非対応 |
| `simd-json`/`sonic-rs` | SIMDで`serde_json`比1.5〜4倍高速 | `unsafe`コード依存、`simd-json`はx86限定の高速パスあり |
| `json5`/`json_five` | JSON5(コメント・トレイリングカンマ・クォート無しキー)に対応 | 厳密JSONとは別クレート、両方使うには自分で組み合わせる必要 |
| `json-commons` | JSONC対応+ドット記法のパス抽出(`get_path()`) | この設計が`extract_path`(本クレートが独自に先行実装していたもの)と同じ発想であることを確認できた=収斂進化の裏付け |
| `valq` | 深い階層のJSONから値を抽出するマクロ | マクロベース、本クレートのような汎用ランタイムAPIではない |

**採用したハイブリッド設計**:
- **`unsafe`不使用・外部パースクレート非依存**を維持(`#![deny(unsafe_code)]`
  相当、`[lints.rust] unsafe_code = "deny"`をCargo.tomlに明記)。
  SIMD最適化(`simd-json`/`sonic-rs`相当の速度)は意図的に追わない——
  高速化には`unsafe`コードとのトレードオフが伴うことを調査で確認済みで
  あり、このエコシステムが他の場所(WebSocketフレーミング・multipart
  パース・gRPC Protocol Buffersコーデック)で一貫して採用している
  「プロトコル/データ形状の層は手書き・`unsafe`不使用」という方針を
  JSONだけ特別扱いしない。
- **`parse()`(寛容モード)と`parse_strict()`(RFC 8259厳密モード)の
  両方を1つのクレートから提供**——`json5`/`serde_json`のように別々の
  クレートを自分で組み合わせる必要が無い、という点をハイブリッドの核に
  した。`parse_strict`は`serde_json::from_str`にそのまま委譲する
  (厳密JSONの定義そのものが`serde_json`の受理範囲と一致するため、
  別の厳密パーサーを自作する必要が無い)。
- **`extract_path`(ドット/ブラケット記法のパス抽出)を維持**——
  `json-commons`の`get_path()`と同じ発想が既に独立して実装されていた
  ことを調査で確認、この設計の妥当性の裏付けとした。

## ビルド・テスト

```bash
cargo test
```

外部パーシングクレートへの依存が無いため、追加のセットアップは不要。

## 関連プロジェクト・移行元

- 旧居場所: `open-runo`(`crates/open-runo-rustjson`)・
  `poem-cosmo-tauri`(同名クレート)——**今後はこのリポジトリへ一本化**。
  旧クレートは撤去せず当面残置(open-runo-db/aruaru-db等が参照している
  ため)、実際にこのクレートへの依存に切り替わった後で撤去を検討する。
- **`open-raid-z`/`aruaru-db`**: ZFS互換・ACID互換のデータ層でこれまで
  素の`serde_json`を直接使っていた箇所を、今後`Rust-JSON`
  (`rust_json::parse`/`parse_strict`/`extract_path`)へ置き換える
  (ユーザー指示、2026-07-17)。進捗は`aruaru-db`側CLAUDE.mdの
  HANDOFFを参照。
- **`open-runo`/`poem-cosmo-tauri`**: `open-runo-db`の`rustjson`
  feature経由でDB層バリデーションに使用中(移行元の実装、参照実装)。

## 運用ルール

- 開発中はこの`CLAUDE.md`を、コード変更のコミット/pushと必ず一緒にpush
  する。
- 確認不要でそのまま実装を進める(このエコシステム共通の運用ルール)。
- README・PORTING.mdは日本語版のみ当面維持し、必要に応じて多言語版を
  追加する(旧クレート側で既に10ヶ国語展開していた実績があるため、
  要望があれば同様に展開可能)。

## 現状

- `src/lib.rs`: `parse`(寛容)・`parse_strict`(厳密)・`to_string`・
  `extract_path`。テスト29件(旧25件+`parse_strict`関連4件)。
- 依存: `serde_json`・`thiserror`のみ。`unsafe_code = "deny"`。

## HANDOFF

- **2026-07-17 リポジトリ新設・ハイブリッド設計・`parse_strict`追加**:
  旧`open-runo-rustjson`のコードをベースに、独立リポジトリとして
  新設。EN/JP調査(`simd-json`/`sonic-rs`/`json5`/`json_five`/
  `json-commons`/`valq`)を踏まえ、`parse_strict`(RFC 8259厳密モード、
  `serde_json::from_str`に委譲)を追加——「寛容+厳密の両方を1クレートで
  提供する」という差別化ポイントを実装。テスト・ビルド確認は次回HANDOFF
  エントリで更新。
  次にすべきこと: (1) `aruaru-db`/`open-raid-z`側のJSON使用箇所を
  `Rust-JSON`に置き換える移行作業、(2) GitHubへの初回push、(3) VPS
  `/root/Rust-JSON`へのクローン。

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
