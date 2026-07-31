# SQL Visualizer — SELECT 文のデータフロー可視化

SELECT 文専用の SQL パーサーを **Rust のパーサーコンビネータで自作**し、
クエリを「集合がどう形作られ、絞られ、選択されるか」の**データフロー図**として
ブラウザ上に描くプロジェクト。パーサーは WASM としてブラウザ内で動く。

## 起動方法

前提: Rust(rustup)、Node.js + pnpm、wasm-pack

```sh
# 初回のみ: WASM ビルドの準備
rustup target add wasm32-unknown-unknown
cargo install wasm-pack

# フロントエンドの起動
cd frontend
pnpm install
pnpm build:wasm   # Rust → WASM(Rust 側を変更したら再実行)
pnpm dev          # http://localhost:5173
```

CLI で AST を確認することもできる:

```sh
cargo run -- "FROM users u JOIN orders o ON u.id = o.user_id SELECT u.name"
```

## 全体像

SQL を編集すると、ブラウザ内の Rust パーサー(WASM)が AST →
フローグラフに変換し、React Flow が描画する。

```mermaid
sequenceDiagram
    actor U as ユーザー
    participant FE as フロントエンド<br>(React + React Flow)
    participant W as WASM<br>(wasm-api)
    participant P as parser<br>(Rust)
    participant E as explain<br>(Rust)

    U->>FE: SQL を編集
    FE->>W: explain(sql) ※デバウンス後
    W->>P: 字句解析 → 構文解析
    Note over P: 句順序自由の SELECT 文を<br>論理評価順の AST に正規化
    P-->>W: AST (Query)
    W->>E: AST → フローグラフ変換
    Note over E: JOIN の合流・列の系譜(output/used)<br>実行順タイムラインを計算
    E-->>W: FlowGraph
    W-->>FE: JSON
    FE->>FE: ELK で自動レイアウト
    FE-->>U: データフロー図
```

より詳細なシーケンス(モジュール単位)は
[docs/02_architecture.md](./docs/02_architecture.md) を参照。

## 構成

### バックエンド(Rust ワークスペース)

パーサーは外部ライブラリに頼らず、`Parser<I, T>` 型と
Functor / Applicative / Alternative / Monad のコンビネータで組み上げている。
入力型がジェネリックなので、**字句解析(文字列)と構文解析(トークン列)が
同じコンビネータを共有**する。

| クレート | 役割 |
| --- | --- |
| [kernel/](./kernel/) | コンビネータの核。`Parser<I, T>`・型クラス・`many0` / `sep_by1` など |
| [basic-parser/](./basic-parser/) | 汎用の基本パーサー(`char` / `digit` / `string` / …) |
| [tokenizer/](./tokenizer/) | SQL 字句解析。文字列 → `Vec<Token>`([仕様](./docs/04_tokenizer_spec.md)) |
| [parser/](./parser/) | 構文解析。トークン列 → AST。**句順序自由化**はここ([文法](./docs/05_grammar_spec.md)) |
| [explain/](./explain/) | AST → フローグラフ変換。列の系譜・実行順タイムラインの計算 |
| [wasm-api/](./wasm-api/) | wasm-bindgen で `parse()` / `explain()` を公開 |

### WASM API

`wasm-pack` で `frontend/src/wasm/pkg` にビルドされ、ブラウザ内で実行される。

| 関数 | 返り値(JSON) | 用途 |
| --- | --- | --- |
| `explain(sql)` | FlowGraph(ノード・エッジ・実行順) | 可視化の本命 API |
| `parse(sql)` | AST | デバッグ用 |

FlowGraph の契約(ノード種別・列の系譜・タイムライン)は
[docs/06_api_design.md](./docs/06_api_design.md) を参照。

### フロントエンド([frontend/](./frontend/))

Vite + React + TypeScript。キャンバスは @xyflow/react(React Flow)、
自動レイアウトは elkjs、スタイリングは Tailwind CSS + Base UI。
「グラフ変換 → レイアウト → ハイライト」を純粋関数として分離している。
デザイン(kawaii × 見やすい)の詳細は
[docs/07_ui_design.md](./docs/07_ui_design.md) を参照。

## ドキュメント

| ドキュメント | 内容 |
| --- | --- |
| [01_overview.md](./docs/01_overview.md) | プロジェクトのゴールとスコープ |
| [02_architecture.md](./docs/02_architecture.md) | アーキテクチャ・データの流れ(詳細シーケンス図) |
| [03_roadmap.md](./docs/03_roadmap.md) | 開発ロードマップ(フェーズ別チェックリスト・現在地) |
| [04_tokenizer_spec.md](./docs/04_tokenizer_spec.md) | 字句解析の仕様 |
| [05_grammar_spec.md](./docs/05_grammar_spec.md) | 文法(EBNF)・句順序自由化・AST |
| [06_api_design.md](./docs/06_api_design.md) | WASM API と FlowGraph 契約 |
| [07_ui_design.md](./docs/07_ui_design.md) | UI 設計・描画ルール・デザイントークン |

## 開発コマンド

```sh
cargo test --workspace      # Rust 側のテスト
cargo clippy --workspace    # Lint
cargo run -- "<SQL>"        # AST の確認

cd frontend
pnpm build:wasm             # Rust → WASM
pnpm dev                    # 開発サーバー
pnpm build                  # 型チェック + プロダクションビルド
node scripts/wasm-smoke.mjs # WASM 経由の動作確認
```

## 謝辞

可視化 UI のアーキテクチャとインタラクション(グラフ変換・レイアウト・
ハイライトの分離、上流経路のハイライト、エッジ上のパーティクル演出など)は
[Liam ERD](https://github.com/liam-hq/liam)
(Apache-2.0, Copyright (c) 2024 ROUTE06, Inc.)を参考にしています。
ソースコードの流用はしていませんが、設計から多くを学びました。
