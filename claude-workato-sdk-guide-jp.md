# Workato SDK × Claude Code — カスタムコネクター開発ガイド

SDK CLI インストール → デプロイ → API 仕様書からの自動生成まで

---

## 目次

1. [はじめに](#1-はじめに)
2. [SDK CLI のインストール](#2-sdk-cli-のインストール)
3. [プロジェクトの構成](#3-プロジェクトの構成)
4. [認証情報の設定](#4-認証情報の設定)
5. [Claude Code でのコネクター開発](#5-claude-code-でのコネクター開発)
6. [OpenAPI 仕様書からコネクターを自動生成](#6-openapi-仕様書からコネクターを自動生成)
7. [Workato へのデプロイ](#7-workato-へのデプロイ)
8. [トラブルシューティング](#8-トラブルシューティング)
9. [参考リンク](#9-参考リンク)

---

## 1. はじめに

このドキュメントは、Workato カスタムコネクターを **Claude Code** を使ってローカルで開発するためのセットアップガイドです。  
SDK CLI のインストールから、コネクターの実装・テスト・デプロイまでを一通り説明します。

### このガイドでわかること

- Workato SDK CLI（`workato-connector-sdk` gem）のインストール方法
- プロジェクトテンプレート（CLAUDE.md / Agents / Skills）の構成と使い方
- Claude Code からのデプロイ手順（`workato push`）
- API 仕様書（OpenAPI）からコネクターを自動生成するワークフロー

### 前提条件

- Ruby 2.7.x / 3.0.x / 3.1.x（推奨: `rbenv` または `rvm` で管理）
- Bundler 2.x
- Git
- Claude Code CLI（`npm install -g @anthropic-ai/claude-code`）

> **日本データセンターの場合**  
> 日本リージョンの Workato を使用している場合、push 先は `https://app.jp.workato.com` になります。後述のデプロイ手順で正しい URL を指定してください。

---

## 2. SDK CLI のインストール

### 2-1. Ruby バージョン確認

```bash
$ ruby -v
# ruby 2.7.6p219 (2022-04-12) [arm64-darwin22]

# バージョンが古いか Ruby がない場合は rbenv/rvm でインストール
$ rbenv install 2.7.6
$ rbenv global 2.7.6
```

### 2-2. SDK gem のインストール

```bash
$ gem install workato-connector-sdk

# Windows の場合は追加で必要
$ gem install tzinfo-data

# インストール確認
$ workato help
```

> **Mac で `charlock_holmes` のインストールに失敗する場合**  
> ICU ライブラリが必要です。以下を実行してから再試行してください。
> ```bash
> $ brew install icu4c
> $ gem install charlock_holmes -- --with-icu-dir=$(brew --prefix icu4c)
> ```

### 2-3. プロジェクトテンプレートを取得

```bash
$ git clone https://github.com/tatsuki-workato/claude-workato-sdk.git
$ cd claude-workato-sdk
```

### 2-4. 新しいコネクタープロジェクトの作成

```bash
# SDK gem でプロジェクトをスキャフォールド
$ workato new ./my-connector
# プロンプトで「1 - secure（推奨）」を選択

# テンプレートの設定ファイルをコピー
$ cp /path/to/claude-workato-sdk/CLAUDE.md my-connector/
$ cp -r /path/to/claude-workato-sdk/.claude my-connector/

$ cd my-connector
$ bundle install
```

> **⚠️ セキュリティ重要**  
> `workato new` 実行時に生成される `master.key` は必ず `.gitignore` に追加してください。このキーで認証情報を復号できます。Git にコミットしないよう注意してください。

---

## 3. プロジェクトの構成

このテンプレートには `CLAUDE.md`、3 つの Subagent、3 つの Skill が含まれています。  
Claude Code が起動するたびに `CLAUDE.md` が読み込まれ、Agents と Skills が必要に応じて呼び出されます。

### 3-1. ディレクトリ構成

```
my-connector/
├── connector.rb              # コネクター本体（Workato DSL Ruby コード）
├── CLAUDE.md                 # Claude Code へのプロジェクト指示書
├── .claude/
│   ├── agents/
│   │   ├── workato-sdk-pro.md    # SDK 実装専門 Agent
│   │   ├── workato-cli.md        # CLI 実行専門 Agent
│   │   └── openapi-analyzer.md   # OpenAPI 解析 Agent
│   └── skills/
│       ├── workato-conventions/  # DSL コーディング規約
│       ├── rspec-patterns/       # テストパターン集
│       └── openapi-to-workato/   # OpenAPI→Workato 変換ルール
├── fixtures/                 # テスト用 JSON 入出力
├── spec/                     # RSpec ユニットテスト
├── tape_library/             # VCR カセット（録音済み HTTP）
├── settings.yaml.enc         # 暗号化済み認証情報
├── master.key                # ⚠️ 絶対に Git コミット禁止
└── Gemfile                   # gem 依存関係
```

### 3-2. Agents の役割

| Agent | 役割 | 呼び出すとき |
|---|---|---|
| `workato-sdk-pro` | `connector.rb` の実装・レビュー専門 | actions / triggers / auth / schema を書くとき |
| `workato-cli` | CLI コマンド実行専門 | `workato exec` / `push` / `oauth2` のデバッグやフィクスチャ生成 |
| `openapi-analyzer` | OpenAPI スペックを解析して実装計画を返す | 大きなスペックを渡すときに先に実行 |

### 3-3. Skills の役割

| Skill | 内容 | 自動ロードのタイミング |
|---|---|---|
| `workato-conventions` | DSL の命名規則・HTTP ヘルパー・スキーマパターン | `connector.rb` を書く・レビューするとき |
| `rspec-patterns` | RSpec + VCR のテストボイラープレート集 | spec ファイルを書く・レビューするとき |
| `openapi-to-workato` | OpenAPI の型・認証・エンドポイントを Workato DSL に変換するルール集 | OpenAPI スペックを変換するとき |

> **💡 Skills の自動ロード**  
> `workato-sdk-pro` は 3 つの Skill をすべて自動的にプリロードします。`connector.rb` に変更を加えるときは常に `workato-sdk-pro` が参照されます。

---

## 4. 認証情報の設定

### 4-1. settings.yaml.enc に認証情報を追加

```bash
# Mac / Linux
$ EDITOR=nano workato edit settings.yaml.enc

# Windows
$ set EDITOR=notepad
$ workato edit settings.yaml.enc
```

エディタが開いたら YAML 形式で認証情報を入力します。

```yaml
# 接続が1つの場合
client_id: your_client_id
client_secret: your_client_secret

# 複数の接続（有効・無効のテスト用）
valid_connection:
  client_id: valid_id
  client_secret: valid_secret
invalid_connection:
  client_id: wrong_id
  client_secret: wrong_secret
```

### 4-2. 接続テスト

```bash
# OAuth2 Authorization Code フロー
$ workato oauth2
# → ブラウザが開くのでログインして認証

# API Key / custom_auth / basic_auth
$ workato exec test --verbose
```

> **🔑 CI/CD での master.key 管理**  
> GitHub Actions などの CI 環境では `master.key` をファイルではなく環境変数で渡します。
> ```
> WORKATO_CONNECTOR_MASTER_KEY=<key の内容>
> ```

---

## 5. Claude Code でのコネクター開発

### 5-1. Claude Code の起動

```bash
$ cd my-connector
$ claude
# → CLAUDE.md が自動で読み込まれる
```

### 5-2. 新しいアクションを追加する

Claude Code に自然言語で指示するだけで、`workato-sdk-pro` Agent が `connector.rb` を実装します。

```
# Claude Code への指示例
"GET /customers エンドポイントを使って顧客を検索する
 search_customers アクションを実装してください。
 name と limit パラメータを受け付けます"
```

実装後は `workato-cli` Agent が以下のサイクルを実行します。

1. フィクスチャ作成（`fixtures/actions/search_customers/input.json`）
2. ローカル実行で動作確認（`workato exec`）
3. 出力を fixtures に保存（`--output` オプション）
4. RSpec テスト作成
5. VCR カセット録音（`VCR_RECORD_MODE=once`）
6. 全テスト通過確認（`bundle exec rspec`）

### 5-3. よく使う CLI コマンド

これらのコマンドは**端末で直接実行する以外に、Claude Code への自然言語の指示でも実行できます**。`workato-cli` Agent がコマンドを組み立てて代わりに実行してくれます。

```
# Claude Code への指示例
"search_customers アクションを input.json で実行して結果を確認してください"
"テストを全部走らせてください"
"DEV 環境に push してください"
```

コマンドのリファレンスとして使いたい場合や、オプションを細かく制御したい場合は直接実行してください。

| コマンド | 説明 |
|---|---|
| `workato exec actions.NAME.execute --input=FILE --verbose` | アクションの execute ラムダをローカル実行 |
| `workato exec actions.NAME.input_fields` | input フィールドスキーマを確認 |
| `workato exec triggers.NAME.poll --input=FILE --verbose` | トリガーのポーリングをローカル実行 |
| `workato exec test --verbose` | 接続テストを実行（HTTP リクエスト詳細付き） |
| `workato oauth2` | OAuth2 Authorization Code フローを実行 |
| `bundle exec rspec` | 全ユニットテストを実行 |
| `VCR_RECORD_MODE=once bundle exec rspec FILE` | VCR カセットを新規録音 |
| `workato generate schema --json=FILE` | JSON レスポンスから Workato スキーマを生成 |
| `workato push --api-token=TOKEN --environment=URL` | Workato ワークスペースにデプロイ |

---

## 6. OpenAPI 仕様書からコネクターを自動生成

API 仕様書（OpenAPI 3.x / Swagger 2.x）がある場合、Claude Code がそれを読み込んで `connector.rb` を自動的に実装します。

### 6-1. 小さいスペック（数百行以下）

スペックファイルをプロジェクトルートに置いて、そのまま指示します。

```
# openapi.yaml をプロジェクトルートに配置してから...

# Claude Code への指示例
"openapi.yaml を読んで connector.rb を実装してください"

# workato-sdk-pro が直接スペックを解析して実装します
```

### 6-2. 大きいスペック（エンドポイントが多数）

先に `openapi-analyzer` Agent でスペックを分析・整理させてから実装に進みます。

```
# Step 1: openapi-analyzer にスペックを解析させる
"openapi-analyzer で openapi.yaml を解析して
 実装すべきアクション・トリガーの計画を作成してください"

# → Agent が以下を返す:
#   - 認証タイプと必要な connection フィールド
#   - 実装すべきアクション一覧（エンドポイントのマッピング）
#   - object_definitions として切り出すスキーマ
#   - pick_lists として定義する enum 値

# Step 2: 計画をもとに実装
"計画に従って workato-sdk-pro で connector.rb を実装してください"
```

### 6-3. 自動変換される主な対応表

| OpenAPI | Workato DSL |
|---|---|
| `securitySchemes` / `oauth2` / `authorizationCode` | `type: 'oauth2'`（Authorization Code Grant） |
| `securitySchemes` / `oauth2` / `clientCredentials` | `type: 'custom_auth'`（acquire でトークン取得） |
| `securitySchemes` / `http` / `bearer` | `type: 'custom_auth'`（Bearer ヘッダーを apply） |
| `components/schemas` の `$ref` | `object_definitions` に切り出し |
| `string` + `enum: [...]` | `pick_lists` + `control_type: 'select'` |
| `string` + `format: date-time` | `type: 'date_time'` |
| `array` of objects | `type: 'array', of: 'object', properties: [...]` |
| `readOnly` フィールド | `input_fields` から除外 |

---

## 7. Workato へのデプロイ

### 7-1. デプロイ前チェックリスト

- [ ] `bundle exec rspec` がすべて PASS している
- [ ] `test` ラムダのレスポンスが 5,000 文字以下
- [ ] トークン認証の場合 `refresh_on: [401]` が設定されている
- [ ] 全 URL は `base_uri` で定義され、アクション内にハードコードしていない
- [ ] `master.key` が `.gitignore` に含まれ、Git にコミットされていない
- [ ] `README.md` にこのコネクターのセットアップ手順が書かれている

### 7-2. workato push コマンド

`workato push` も**Claude Code への指示で実行できます**。`workato-cli` Agent がオプションを確認しながら実行します。

```
# Claude Code への指示例
"DEV 環境に push してください"
"JP データセンターの DEV にバージョンノート付きで push してください"
```

直接実行する場合のコマンド：

```bash
$ workato push \
  --api-token=<Workato API トークン> \
  --environment=https://app.jp.workato.com \  # JP データセンター
  --folder=<フォルダ ID> \
  --notes="search_customers アクションを追加"
```

**データセンター別の environment URL**

| リージョン | URL |
|---|---|
| 日本 (JP) | `https://app.jp.workato.com` |
| 米国 (US) | `https://app.workato.com` |
| 欧州 (EU) | `https://app.eu.workato.com` |
| シンガポール (SG) | `https://app.sg.workato.com` |
| オーストラリア (AU) | `https://app.au.workato.com` |

> **💡 API トークンの取得方法**  
> Workato の Settings → API clients から API クライアントを作成し、トークンを発行してください。Push には Connector SDK スコープの権限が必要です。

### 7-3. 環境プロモーション

```
ローカル開発
    ↓ workato exec でテスト
DEV workspace
    ↓ workato push でアップロード
UAT
    ↓ レシピで動作確認
PROD
    ↓ ライフサイクル管理で昇格
```

> **⚠️ コネクターバージョンは不変**  
> Workato でリリース済みのコネクターバージョンは変更できません。変更後は常に新しいバージョンとして `push` してください。

---

## 8. トラブルシューティング

| エラー | 原因と対処 |
|---|---|
| `401 Unauthorized`（exec 実行時） | アクセストークンが期限切れ。oauth2 コネクターは `workato oauth2` を、custom_auth は `workato exec test` を実行してトークンを更新してください |
| `VCR::Errors::UnhandledHTTPRequestError` | VCR カセットが未録音か、トークンが変わってマッチしなくなっています。`VCR_RECORD_MODE=once bundle exec rspec <ファイル>` で再録音してください |
| `master.key not found` | `master.key` が見つかりません。`WORKATO_CONNECTOR_MASTER_KEY` 環境変数を設定するか、`--key` オプションでパスを指定してください |
| push で `403` エラー | API トークンに必要なスコープがありません。Connector SDK スコープを持つトークンか確認してください |
| `undefined method`（connector.rb） | DSL の構文エラーか、Ruby `def` を使っています（lambda を使ってください）。`workato exec test --debug` でスタックトレースを確認してください |
| `charlock_holmes` のインストール失敗 | ICU ライブラリが必要です。Mac: `brew install icu4c` を実行後、`gem install charlock_holmes -- --with-icu-dir=$(brew --prefix icu4c)` を試してください |

---

## 9. 参考リンク

- [Connector SDK ドキュメント](https://docs.workato.com/developing-connectors/sdk.html)
- [SDK CLI リファレンス](https://docs.workato.com/developing-connectors/sdk/cli.html)
- [SDK gem（GitHub）](https://github.com/workato/workato-connector-sdk)
- [このプロジェクトテンプレート](https://github.com/tatsuki-workato/claude-workato-sdk)
- [Claude Code ドキュメント](https://code.claude.com/docs)
