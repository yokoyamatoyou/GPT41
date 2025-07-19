# Long-Context Chat Application — Detailed Specification

## 1. 目的

OpenAI GPT-4.1 系列モデル (最大100万トークン) を API 経由で利用し、Web版ChatGPTに近いユーザー体験 (会話履歴・題名自動生成・ファイル添付・Webブラウズ・ストリーミング表示など) を提供するアプリケーションを構築する。コーディングは Jules エージェントで自動化・補助する想定。

## 2. モデル選択UI

| オプション        | コンテキスト長       | コスト(相対) | 想定用途                       |
| ----------------- | ------------------ | ----------- | ------------------------------ |
| GPT-4.1           | 1,000,000 tokens   | 高          | リサーチ・長編ドキュメント・大規模コード解析 |
| GPT-4.1-mini      | 256,000 tokens     | 低          | 日常QA・短〜中程度コンテキスト          |

- フロントエンドにドロップダウン「使用モデル」を配置し、上記2つを選択可能とする。
- 選択状態はスレッド単位で保持 (途中変更時はユーザへ警告)。

## 3. 主要機能一覧

- ストリーミング表示: API stream=true で分割受信し、逐次レンダリング。
- 会話メモリ: 全メッセージを保存し、必要に応じ要約(Summarize)でスライディングウィンドウに圧縮。
- 題名自動生成: スレッド開始から n=5 メッセージで title をモデルに生成させ、サイドバー表示。
- ファイル添付: PDF/Markdown/コードファイルなどをアップロードしテキスト抽出、要約＋全文または抜粋をコンテキストへ。
- Webブラウズ: 外部検索API (Bing/GCS) でHTML抽出し、GPTに要約指示。ユーザには引用付きで返却。
- トークン & コスト表示: 送信時点の総トークン・推定課金額をUIヘッダーに表示。

## 4. アーキテクチャ概要

```
┌──────────────┐        ┌──────────────┐
│  Frontend    │ ⇄ WS  │   Gateway     │
│  (React/Vue) │      │  (FastAPI)    │
└────┬─────────┘        └┬─────────────┘
     │                     │
     │ REST/WS             │ OpenAI API (chat/completions)
     ▼                     ▼
┌──────────────┐        ┌──────────────┐
│  MongoDB      │        │  Worker(JobQ) │
│  (threads)    │        │  file parse   │
└──────────────┘        └──────────────┘
```

- Gateway: 認証、レート制限、ストリーミング転送。
- Worker: ファイル抽出・Web fetch・要約処理。Julesによる自動タスク生成を想定。
- DB: threads, messages, files, search_cache コレクション。

## 5. エンドポイント設計 (一部抜粋)

| Method | Path              | 説明                                                            |
| ------ | ----------------- | ------------------------------------------------------------- |
| POST   | /api/chat         | body: {thread_id, model, messages[], stream} → streaming JSON chunk |
| POST   | /api/thread       | 新規スレッド作成 (model, system_prompt)                              |
| GET    | /api/thread/:id   | 履歴取得                                                      |
| POST   | /api/upload       | multipart form (file) → {file_id}                             |
| POST   | /api/search       | {query} → 要約付き検索結果                                       |
| POST   | /api/summarize    | {thread_id} → messages要約＆DB更新                                 |

## 6. 会話履歴 & 要約アルゴリズム

- 各スレッドにフル履歴と要約履歴の2層を保持。
- トークン数がモデル上限の70%超過時、古いチャンクをsummarizeエンドポイントで圧縮。assistant_summary roleで保存。
- 要約粒度はtoken約2kごとに一段階要約。

## 7. ファイル処理フロー

1. /uploadで保存
2. WorkerがMIME type判定→テキスト抽出(pdfminer, Pygments等)
3. 抽出テキストが10k token超ならTL;DR要約を分割生成
4. filesコレクションに格納し、メッセージに<file ref>タグを付与

## 8. Webブラウザモジュール

- 検索API(Bing v9 or SerpAPI): top 5 links
- 各ページをReadabilityで本文抽出
- GPTに「次のHTMLを日本語で700字以内に要約し、3箇条書きで引用元タイトルとURLを添えて」と指示

## 9. コスト管理

- price_per_1k_token をenvで管理
- リクエストごとにprompt_tokens+completion_tokensを集計
- 月次利用をダッシュボード表示。アラート閾値設定

## 10. セキュリティ & 運用

- JWT or OAuth2でユーザ認証
- ファイルストレージは署名付きURL
- Web fetchはCORS & 同一オリジン制限
- Julesの生成コードはCIテスト + human-in-the-loopレビュー

## 11. 今後の拡張案

- 音声入力/読み上げ (Web Speech API)
- プロンプトテンプレート共有ギャラリー
- チーム共有 (スレッドを他ユーザへ共有リンク)
- メタデータ付き検索 (Vector DB + embeddings)

---

### Appendix: Julesへのタスク指示例

```yaml
- name: implement_stream_chat
  description: |
    Create a Gateway endpoint POST /api/chat that proxies to OpenAI streaming.
    Requirements:
      - Support text/event-stream.
      - Timeout 120s.
      - Verify user quota.
```

## 7. MVP Local Storage & Deployment Strategy

### 7.1 Conversation History (ローカル保存)

- 保存場所: アプリのルート直下 ./chat_history/<thread_id>/YYYYMMDD_HHMMSS.json 形式でディレクトリ自動生成
- データ構造: { "threadId", "title", "createdAt", "messages": [ {"role","content","ts"}, … ] }
- 実装方針: IChatHistoryStore 抽象インターフェースを定義し、MVPでは FileSystemChatHistoryStore を実装。クラウド移行時は S3ChatHistoryStore や DynamoDBChatHistoryStore を差し替え。

### 7.2 APIキー管理

- 取得方法: 環境変数 OPENAI_API_KEY を参照
- .env.example をリポジトリに同梱し、開発者は .env をローカル作成
- 取得失敗時は起動時に明確にエラーを表示

### 7.3 移行容易性（クラウド対応）

1. 依存性注入: ストレージ層・設定層を DIコンテナ経由で提供
2. コンテナ化: Dockerfileを用意し、volumeマウントで chat_historyディレクトリを永続化
3. CI/CD: GitHub Actions→コンテナビルド→任意のクラウド（GKE, ECS, Cloud Run）へデプロイ可能に
4. 設定の切替: ENV=local/cloud で設定ファイルを読み分け、ストア実装をバインド

- Julesエージェントによる実装時は、IChatHistoryStoreとOPENAI_API_KEY取得ロジックをテンプレート化し、クラウド環境用のストアをTODOコメントとして残す

## 12. HTML Diagram & Presentation Support

### 12.1 Diagram Generation (MermaidJS)

- Syntax: ユーザはメッセージやファイル内に mermaid コードブロックを記述
- レンダリング: フロントエンドで MermaidJS (browser build) を import('mermaid') で動的ロードし、可視化
- CLI依存なし: Mermaid CLIは使用しない。ブラウザJSだけで描画するため、外部Node実行環境を要求しない
- エクスポート: 右上メニューからSVG/PNG保存可能
- ユースケース: ER図、フローチャート、シーケンス図、ガントチャート

#### 12.1.1 Mermaid Syntax Rules (GPT出力ガイドライン)

1. コードブロックは必ず mermaid で開始し、``` で終了
2. ダイアグラムタイプはgraph TD, sequenceDiagram, classDiagram, erDiagram, ganttなど正式シンタックスに限定
3. ノードIDは英数字と_のみ。スペースはラベルに"..."を使い、IDには使用しない
4. 特殊文字(<, >, &, :)はMermaidで許可された場合のみ使用。SVGエスケープ不要
5. コメントは%%プレフィックス
6. GPTはエラー出力禁止。不正構文を生成しないこと
7. 複雑図の場合はセクション分割やサブグラフ(subgraph)を活用

### 12.2 Diagram Generation (ネイティブSVGモード — 外部ライブラリなし)

- 動機: OpenAI API以外の外部依存を完全に排除したい場合
- 切替方法: 環境変数 DIAGRAM_RENDERER=native を設定
- プロンプト設計: システムプロンプトでGPT-4.1に以下を指示

  あなたはSVG図ジェネレータです。与えられた図の説明を、枠線・矢印・テキストを含む<svg>マークアップ(viewBox="0 0 800 600")として返してください。色指定はnone、黒線のみ、フォントはmonospace。

- レンダリング: GPTの応答にsvgコードブロックが含まれている場合、フロントエンドはinnerHTMLで挿入し直接表示
- エクスポート: download="diagram.svg"付き<a>タグで保存可能
- 制限事項: レイアウト自動化は行われないため、ノード位置はGPTに計算させる必要あり（複雑図は難易度↑）

### 12.3 HTML Presentation Builder (Reveal.js)

- ライブラリ: Reveal.js (Markdown対応) を組み込み、/presentルートで表示
- 入力: Markdownスレッドまたはアップロードした.mdをスライドに変換
- 機能:
  1. テーマ選択(black/white/league等)
  2. ノートモード&全画面モード
  3. Export HTML/PDFボタン(reveal.js print-pdfオプション)
- CLI: Julesタスクgenerate_presentation_from_thread→Markdownスライドファイルを出力しPR化

### 12.4 API & UI

| Method | Path                      | 説明                                |
| ------ | ------------------------- | --------------------------------- |
| POST   | /api/diagram/preview      | {code, renderer} → SVG data URL   |
| POST   | /api/presentation/build   | {markdown, theme} → HTML ZIP      |

### 12.5 クラウド対応

- Serverless: HTMLスライドは静的ビルド→Cloud Storage(CDN)へ配置
- Access Control: 署名付きURLで限定共有(期限付き)

- MVPでは renderer=mermaid と renderer=native の両方をサポートし、後者は完全にライブラリレスでオフライン対応

## 13. External Dependencies & Offline Mode

### 13.1 Mandatory External Service

- OpenAI API(GPT-4.1 / GPT-4.1-mini)

### 13.2 Optional / Pluggable Services(無効化可能)

| 機能        | 依存サービス                           | デフォルト設定       | 無効化方法                          |
| --------- | -------------------------------- | ------------- | ------------------------------ |
| Web検索・要約  | Bing Web Search API / Google CSE | OFF (MVP) | ENABLE_WEB_SEARCH=false 環境変数 |
| クラウドストレージ | AWS S3 / GCP Storage             | OFF           | STORAGE_DRIVER=local         |
| メール通知     | SendGrid API                     | OFF           | ENABLE_EMAIL=false           |

- MVPではすべてOFF。将来クラウド移行時にfeature-flagで有効化

### 13.3 クライアント側ライブラリ(ローカルバンドル)

| ライブラリ         | 用途       | オンライン依存      | バンドル方法                                   |
| ------------- | -------- | ------------ | ------------------------------------------ |
| MermaidJS     | 図のレンダリング | なし (CDN不使用) | npm install mermaid; Vite/webpackでembed.min.jsを同梱 |
| Reveal.js     | スライド生成   | なし           | npm install reveal.js; dist/を静的配信          |

- これらは node_modules からビルド成果物にコピーして配信するため、オフライン環境でも動作。
- CSS/フォントもプロジェクト内にベンドル（Google Fontsは使用しない）

### 13.4 サーバ側ライブラリ(OSS)

| ライブラリ            | 言語     | 役割           | オンライン依存 |
| ---------------- | ------ | ------------ | ------- |
| pdfminer.six     | Python | PDF → テキスト抽出 | なし      |
| Pygments         | Python | シンタックスハイライト  | なし      |
| readability-lxml | Python | HTML→本文抽出    | なし      |

- これらはPyPIから取得し、Dockerイメージにキャッシュ。同じバージョンで再現可能

### 13.5 ビルド & 配布

1. npm run build でフロントエンドを完全静的アセットにバンドル。/dist内にMermaid/Revealを同梱
2. Python/FastAPIサーバ用のDockerfileはpip installで依存をvendorし、--no-index --find-links=/wheelsオプションでオフライン再ビルドに対応
3. Single-file binaryオプション: PyInstallerでスタンドアロン実行ファイルを出力しCLIツールとして配布可能

### 13.6 オフライン動作テスト手順

- (1) インターネット遮断環境を想定し、/etc/hostsで0.0.0.0 0.0.0.0を追加 or --offline Wi-Fiモード
- (2) OPENAI_API_KEYもダミーに置き換え、期待通り起動失敗するか確認
- (3) ENABLE_WEB_SEARCH=falseで図・スライド・ファイル要約がローカルで完結することを確認
- (4) Mermaid, Revealのレンダリングが200 OK(networkタブに外部リクエスト無し)であることを確認

- 以上により、OpenAI API以外の外部ネットワーク依存を排除し、MVPが完全ローカルで動作することを保証する。クラウド移行時はfeature-flagを切り替え、必要な外部サービスを段階的に有効化できる設計。

