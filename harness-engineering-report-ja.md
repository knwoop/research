# ハーネスエンジニアリング：科学的根拠と実践

## エグゼクティブサマリー

ハーネスエンジニアリングは、AIコーディングエージェントを大規模に信頼性高く動作させるための環境、制約、フィードバックループを設計するソフトウェア開発の新興分野である。この用語は2026年初頭に正式に定義された。主にMitchell Hashimoto（HashiCorpの共同創設者）とOpenAIのRyan Lopopoloによる公表がきっかけとなった。Lopopoloは5ヶ月間の内部実験を報告し、3人のエンジニアチームがCodexエージェントを使用して、手動でのソースコード記述ゼロで約100万行のプロダクションコードを構築した。

本レポートでは、ハーネスエンジニアリングを巡る科学的根拠、実践的手法、批判的視点を包括的に分析する。主な知見は以下の通り：(1) ハーネスのみの改善で、モデルアップグレードに匹敵またはそれを上回る性能向上が得られる（例：LangChainのモデル変更なしでの13.7ポイントのベンチマーク改善）、(2) エンジニアの役割をコード作成者から環境設計者へ根本的に変える、(3) 検証、レガシーコードベースへの適用、モデル能力とハーネスの洗練度の間の緊張関係など、未解決の課題が多数残っている。

---

## 目次

1. [定義と起源](#1-定義と起源)
2. [OpenAIの内部実験](#2-openaiの内部実験)
3. [コア技術プラクティス](#3-コア技術プラクティス)
4. [科学的根拠と実証結果](#4-科学的根拠と実証結果)
5. [Codex App Serverとインフラストラクチャ](#5-codex-app-serverとインフラストラクチャ)
6. [実践的ツールと導入パターン](#6-実践的ツールと導入パターン)
7. [コミュニティ分析と批判的視点](#7-コミュニティ分析と批判的視点)
8. [ソフトウェアエンジニアリングへの影響](#8-ソフトウェアエンジニアリングへの影響)
9. [結論](#9-結論)
10. [出典](#10-出典)

---

## 1. 定義と起源

### ハーネスエンジニアリングとは何か？

ハーネスエンジニアリングは、AIエージェントが自律的に信頼性の高い作業を行えるようにするための、インフラストラクチャ、制約、フィードバックループを設計する分野である。ThoughtworksのBirgitta Böckelerは、「ハーネス」を「AIエージェントを制御するために使用できるツールとプラクティス」と定義している（<a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Martin Fowler / Thoughtworks</a>）。

この比喩は馬具に由来する — 強力だが予測不可能な力を生産的に導くものという意味である。ソフトウェアエンジニアリングの文脈では、ハーネスはエージェントのランタイム周辺機器を包含する：ツール、設定ポイント、ドキュメント、リンター、アーキテクチャ制約、モデルが環境と信頼性高く対話するための統合メカニズムである（<a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer</a>）。

基本方程式は次の通り：**コーディングエージェント = AIモデル + ハーネス**

### プロンプトエンジニアリング、コンテキストエンジニアリングとの違い

3つのアプローチは入れ子構造の階層を形成する（<a href="https://madplay.github.io/en/post/harness-engineering" target="_blank">MadPlay</a>）：

| アプローチ | 核となる問い | 設計対象 |
|----------|------------|---------|
| プロンプトエンジニアリング | 「何を尋ねるべきか？」 | LLMへの指示テキスト |
| コンテキストエンジニアリング | 「何を見せるべきか？」 | 推論時に可視な全トークン |
| ハーネスエンジニアリング | 「環境をどう設計すべきか？」 | 外部制約とフィードバックシステム |

プロンプトが「右折」という命令だとすれば、コンテキストエンジニアリングは地図と道路標識を提供する。ハーネスエンジニアリングは完全なインフラストラクチャ — 手綱、鞍、柵、そして道路そのものである。

### 用語の起源

2026年2月初頭、Mitchell Hashimoto — HashiCorpの共同創設者でTerraformの開発者 — がこの実践に名前を与えるブログ記事を公開した。彼はこれを「エージェントがミスをするたびに、そのエージェントが二度とそのミスをしないよう解決策をエンジニアリングする」というアイデアと定義した（<a href="https://mitchellh.com/writing/my-ai-adoption-journey" target="_blank">Mitchell Hashimoto</a>）。その直後、OpenAIのRyan Lopopoloが内部実験の詳細な報告を公開し、このコンセプトが業界全体に広まった（<a href="https://openai.com/index/harness-engineering/" target="_blank">OpenAI</a>）。

---

## 2. OpenAIの内部実験

### 5ヶ月間のプロジェクト

OpenAIのハーネスエンジニアリング実験は、この分野で最も引用されるケーススタディである。5ヶ月間にわたり、当初3人（後に7人に増員）のエンジニアチームが、意図的な制約のもとCodexエージェントを使用して内部ベータ製品を構築・出荷した：手動で書かれたソースコードはゼロ行（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ</a>）。

### 主要指標

| 指標 | 値 |
|------|-----|
| コード量 | 約100万行（アプリケーションロジック、インフラ、ツール、ドキュメント、ユーティリティ） |
| プルリクエスト | 約1,500件（オープン＆マージ） |
| チーム規模 | 初期3名、後に7名に拡大 |
| 期間 | 5ヶ月 |
| スループット | エンジニア1人あたり1日3.5 PR |
| 速度推定 | 手動開発の約1/10の時間 |
| 手動記述コード | 0行 |

### 意図的な制約

チームは「手動記述コード0行」を意図的な強制力として選択した。Lopopoloが説明したように、この制約により、エンジニアリング速度を桁違いに向上させるために必要なものを構築することが確実になった。哲学はシンプル：「人間が舵を取る。エージェントが実行する」（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ</a>）。

### ワークフロー

人間とエージェントのインタラクションループは単純明快：エンジニアがタスクを記述し、エージェントを実行し、PRのオープンを許可する。エージェントは開発インフラと直接対話する — PRの自律的なオープン、コード変更の評価、基準が満たされるまでのソリューション反復、テレメトリデータを用いたバグの再現、隔離された開発環境でのパフォーマンス監視（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ</a>）。

---

## 3. コア技術プラクティス

OpenAIのハーネスエンジニアリングフレームワークは3つの柱で構成され、コミュニティが特定したいくつかの補足的プラクティスがある。

### 3.1 コンテキストエンジニアリング

コンテキストエンジニアリングは、AIエージェントが消費する情報環境を構造化する実践である。OpenAIは、システムオブレコードとして扱う構造化された`docs/`ディレクトリ内にドキュメントを階層的に整理した（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ</a>）。

**マニュアルではなくマップとしてのAGENTS.md**：短いAGENTS.mdファイル（約100行）がコンテキストに注入され、主にマップとして機能する。より深い情報源へのポインタを提供する。最初期の教訓の一つ：「Codexには1,000ページの説明書ではなく、マップを渡せ」（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ</a>）。

**静的コンテキストと動的コンテキスト**：コンテキストには静的コンポーネント（リンターで検証されたリポジトリドキュメント、設計仕様）と動的コンポーネント（可観測性データ、ディレクトリ構造マッピング、CI/CDステータス）がある。重要な原則：「リポジトリが唯一の真実のソースでなければならない」（<a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode</a>）。

**Documentation-as-Code**：ドキュメント間のクロスリンク関係はリンターとCI検証により機械的に強制され、ドキュメントがコードベースと一貫性を保つようにする。

### 3.2 アーキテクチャ制約

逆説的に、エージェントができることを制約することで、行き止まりの探索が減り、生産性が向上する（<a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode</a>）。

**レイヤード依存関係**：OpenAIは厳格な依存関係フローを強制する：

```
Types → Config → Repo → Service → Runtime → UI
```

構造テストにより、エージェントがモジュール境界を尊重し、レイヤードアーキテクチャの違反を防止する。

**修復指示付きカスタムリンター**：カスタムリンターは、構造化ログ、スキーマ・型の命名規則、ファイルサイズ制限、プラットフォーム固有の信頼性要件などのルールを強制する。リントがカスタムであるため、エラーメッセージは修復指示をエージェントのコンテキストに直接注入する — 強制と教育の両方の機能を果たす（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ</a>）。

**構造テスト**：ArchUnitスタイルのテストフレームワークがアーキテクチャ境界を機械的に強制し、人間のアーキテクトが全PRをアーキテクチャ準拠のために手動レビューする必要性を置き換える。

### 3.3 エントロピー管理（ガベージコレクション）

時間の経過とともに、エージェント生成コードは不整合を蓄積する — チームがエントロピーと呼ぶもの。解決策には周期的なクリーンアッププロセスが含まれる：

- チームは「ゴールデンプリンシプル」をリポジトリに直接エンコードした — コードベースを読みやすく一貫性のある状態に保つための、意見を持った機械的ルール（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ</a>）。
- 定期的な「ガベージコレクション」エージェントがスケジュール（日次/週次/イベントトリガー）で実行され、ドキュメントの不整合の特定、アーキテクチャ制約違反の検出、パターンの強制を行う（<a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode</a>）。
- 反復的な哲学：「エージェントが苦戦する時、それをシグナルとして扱う：欠けているもの — ツール、ガードレール、ドキュメント — を特定し、フィードバックする」（<a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Martin Fowler / Thoughtworks</a>）。

### 3.4 可観測性の統合

エージェントは包括的なテレメトリ — ログ、メトリクス、分散トレース — を活用して、アプリケーションの動作を監視し、パフォーマンスの異常を特定し、問題を体系的に再現し、デプロイ前に修正を検証する（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ</a>）。

### 3.5 構造化された進捗追跡

Anthropicの研究により、エージェントセッション間で構造化された進捗を維持することが重要であることが判明した。主な知見：

- **MarkdownよりJSON**：「モデルはMarkdownファイルと比較して、JSONファイルを不適切に変更したり上書きしたりする可能性が低い」（<a href="https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents" target="_blank">Anthropic</a>）。
- **Gitベースの追跡**：エージェントは記述的なコミットメッセージで進捗をコミットし、ロールバックと回復を可能にする。
- **機能リストシステム**：包括的なJSONファイルが全ての必要な機能を完了ステータスとともに詳述し、エージェントがプロジェクトの早期完了を宣言することを防ぐ（<a href="https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents" target="_blank">Anthropic</a>）。

### 3.6 バックプレッシャーと検証

成功はエージェントが自身の作業を検証する能力と強く相関する。必須メカニズムには型チェック、ビルド、ユニット/統合テスト、コードカバレッジ、UI対話テストが含まれる。重要な原則：メカニズムはコンテキスト効率が良くなければならない — フルテストスイートの実行はコンテキストウィンドウを溢れさせ、エージェントがタスクフォーカスを失う原因となる。解決策は、通過したテストの出力を抑制し、失敗のみを表示すること（<a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer</a>）。

---

## 4. 科学的根拠と実証結果

### 4.1 LangChain Terminal Bench 2.0実験

ハーネスエンジニアリングのインパクトの最も厳密な実証の一つが、LangChainのTerminal Bench 2.0での実験である。これは機械学習、デバッグ、バイオロジーにわたる89のコーディングタスクを含むベンチマークである（<a href="https://blog.langchain.com/improving-deep-agents-with-harness-engineering/" target="_blank">LangChain</a>）。

**主要結果**：エージェントは52.8%から66.5%に改善（ランク約30位からトップ5へ）、13.7パーセントポイントの向上。基盤モデル（GPT-5.2-Codex）は固定のまま。

**変更内容**：
- 終了前に検証実行をリマインドする`PreCompletionChecklistMiddleware`
- 起動時にディレクトリをマッピングしツールを特定する`LocalContextMiddleware`
- 繰り返しのファイル編集を監視し代替アプローチを提案する`LoopDetectionMiddleware`
- 「推論サンドイッチ」 — 計画/検証には高い推論、実装には中程度の推論

**最も重要な発見**：「モデルは最初のもっともらしい解決策に偏りがある」。テストによる自己検証が不可欠であることが証明された — エージェントは作業の検証のために積極的なプロンプティングと構造的な強制を必要とした（<a href="https://blog.langchain.com/improving-deep-agents-with-harness-engineering/" target="_blank">LangChain</a>）。

### 4.2 Hashline実験

Can Bölükは、編集ツールの設計がLLMコーディングパフォーマンスの主要なボトルネックであることを実証した。16モデル、1回あたり180タスクで新しい「hashline」編集システムを実装した結果（<a href="https://blog.can.ac/2026/02/12/the-harness-problem/" target="_blank">Can Bölük</a>）：

| モデル | 変更前（Patch） | 変更後（Hashline） | 改善 |
|--------|----------------|-------------------|------|
| Grok Code Fast 1 | 6.7% | 68.3% | 約10倍 |
| MiniMax | 約33% | 約66% | 約2倍 |
| Grok 4 Fast | — | — | 出力トークン61%削減 |
| Gemini | — | — | +8%改善 |

**主要な知見**：「Patchはほぼすべてのモデルにとって最悪のフォーマットであり、hashlineはほとんどのモデルでreplaceに匹敵またはそれを上回り、最も弱いモデルが最も恩恵を受ける」。Geminiの+8%改善は「ほとんどのモデルアップグレードが提供するものより大きく、トレーニング計算コストはゼロ」。

### 4.3 ETH Zurichのコンテキストファイル研究

ETH Zurichの研究者が、リポジトリコンテキストファイル（AGENTS.md / CLAUDE.md）がコーディングエージェントのパフォーマンスを実際に向上させるかの初の厳密な実証評価を発表した（<a href="https://arxiv.org/html/2602.11988v1" target="_blank">ETH Zurich - arXiv</a>）。

**方法論**：4つのエージェント-モデルペアを300のSWE-bench Liteタスクと新しい138タスクのAGENTbenchベンチマークでテスト。

**主要な知見**：
- LLM生成コンテキストファイルは成功率を平均約0.5-2%**低下**させた
- 人間が書いたコンテキストファイルはAGENTbenchで**わずかに約4%の改善**を提供
- LLM生成ファイルは推論コストを20-23%増加させた
- すべてのコンテキストファイルがタスク完了に必要なステップ数を増加させた
- コンテキストファイルはリポジトリ概要として効果的に機能しない

**推奨事項**：LLM生成コンテキストファイルは完全に省略すること。人間が書くファイルは、リポジトリのツールに固有の最小限の必須要件のみ — エージェントがコードを読むだけでは発見できない情報のみ — を含めるべきである（<a href="https://arxiv.org/html/2602.11988v1" target="_blank">ETH Zurich - arXiv</a>）。

Addy Osmaniもこの知見を補強した：「AGENTS.mdは、恒久的な設定ではなく、まだ修正していないコードベースの問題のリビングリストとして扱うのが良いメンタルモデルだ」（<a href="https://addyosmani.com/blog/agents-md/" target="_blank">Addy Osmani</a>）。

### 4.4 コンテキストロットと長コンテキスト劣化

Chromaの研究により、LLMのパフォーマンスは長いコンテキスト長で劣化し、質問とコンテキストの意味的類似度が低い場合により急激に劣化することが実証されている。これは、コンテキストプレッシャーの解決策として、より大きなウィンドウではなく、サブエージェントによるより良いコンテキスト分離が必要というハーネスエンジニアリングの原則を検証している（<a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer</a>）。

**ハーネス間のパフォーマンス差異**：Opus 4.6はClaude Codeのハーネスでは33位だが、代替ハーネスでは5位にランクされており、ハーネス環境によってモデルのパフォーマンスが大幅に上下することを示している（<a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer</a>）。

### 4.5 Stripeの本番スケールの証拠

StripeのMinionsシステムは、エンタープライズスケールでのハーネスエンジニアリングの最も強力な証拠を提供している（<a href="https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents" target="_blank">Stripe Dev Blog</a>）：

- **スケール**：週あたり1,300以上のマージ済みPR、人間が書いたコードゼロ
- **コンテキスト**：年間1兆ドル以上の決済量を支えるコード
- **インフラ**：内部「Toolshed」サーバー経由の400以上のMCPツール、10秒スピンアップの事前ウォームされた「devbox」サンドボックス環境
- **品質**：300万テストバッテリー、タスクあたり最大2回のCI実行、多くのテストに自動修正が含まれる
- **哲学**：「フィードバックを左にシフト」 — 開発の後段ではなく、即座に問題を捕捉する

---

## 5. Codex App Serverとインフラストラクチャ

### アーキテクチャ

OpenAIはCodex App Serverアーキテクチャを公開した — Codexエージェントのコアロジックを複数のクライアントサーフェス（CLI、VS Code、ウェブアプリ、macOSデスクトップ、JetBrains、Xcode）から分離する双方向JSON-RPC API（<a href="https://www.infoq.com/news/2026/02/opanai-codex-app-server/" target="_blank">InfoQ</a>）。

### 3つの会話プリミティブ

| プリミティブ | 目的 |
|------------|------|
| **Items** | ライフサイクル状態（開始、ストリーミング、完了）を持つ入出力のアトミック単位 |
| **Turns** | 単一のエージェント作業単位が生成するアイテムのシーケンスグループ |
| **Threads** | 作成、再開、フォーク、アーカイブをサポートする永続セッションコンテナ |

### なぜMCPではないのか？

OpenAIはModel Context Protocolを実験したが、IDE対話に必要なリッチなセッションセマンティクス — ストリーミングdiff、承認ワークフロー、スレッドの永続化 — にはツール指向のモデルでは不十分であることが判明した。Codexはよりシンプルなワークフローのためにはサポートするが、完全忠実度の統合にはApp Serverが推奨される（<a href="https://www.infoq.com/news/2026/02/opanai-codex-app-server/" target="_blank">InfoQ</a>）。

### サンドボックス

Codexはプラットフォームネイティブの強制を使用してサンドボックス内でシェル/ファイルツールを実行する。サンドボックスはCodexが自律的にできること — 変更可能なファイルとコマンドのネットワーク使用可否 — を定義し、エージェントに制限されたワークスペースを提供する（<a href="https://developers.openai.com/codex/concepts/sandboxing" target="_blank">OpenAI Developers</a>）。

---

## 6. 実践的ツールと導入パターン

### 6.1 実装レベル

コミュニティは段階的導入モデルに収束している（<a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode</a>）：

| レベル | 範囲 | コンポーネント | 時間投資 |
|--------|------|--------------|---------|
| レベル1 | 個人開発者 | CLAUDE.md/.cursorrules、プレコミットフック、テストスイート、一貫した命名 | 1-2時間 |
| レベル2 | 小規模チーム | AGENTS.md、CI強制制約、共有プロンプト、ドキュメントasコード | 1-2日 |
| レベル3 | 本番 | カスタムミドルウェア、可観測性統合、エントロピーエージェント、監視ダッシュボード | 1-2週間 |

### 6.2 具体的な設定レバー

HumanLayerチームは6つの主要な設定レバーを特定している（<a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer</a>）：

1. **CLAUDE.md / AGENTS.mdファイル**：システムプロンプトに注入されるリポジトリレベルのマークダウンファイル。60行以下で手作りし（自動生成ではなく）、推論不可能な情報のみを含める。

2. **MCP（Model Context Protocol）サーバー**：ファイルI/OやBash以上にエージェントの能力を拡張する。重要な注意：ツールの説明がシステムプロンプトに注入されるため、信頼できないサーバーは接続しないこと。トレーニングデータに十分表現されているCLIツールを優先する。

3. **スキル**：段階的開示を可能にする再利用可能な知識/ツールバンドル — エージェントは必要な時にのみ特定の指示にアクセスする。

4. **サブエージェント**：「コンテキストファイアウォール」として機能 — 離散的なタスクが隔離されたコンテキストウィンドウで実行され、中間ノイズの蓄積を防ぐ。圧縮された結果のみが親エージェントに返される。

5. **フック**：指定されたライフサイクルイベント（ツールコールの前後）で自動的に実行される。通知、承認、サービス統合、検証に一般的に使用される。

6. **バックプレッシャーと検証**：型チェック、ビルド、テスト、コードカバレッジレポート — コンテキスト効率の良い出力（通過テストを抑制し、失敗のみを表示）。

### 6.3 サポートツールエコシステム

複数のツールがハーネスエンジニアリングパターンをサポートしている：
- **OpenAI Codex**：このコンセプトの発祥プラットフォーム
- **Claude Code**：CLAUDE.md、サブエージェント、フック、スキルを備えたAnthropicの実装
- **Cursor**：編集マージを処理するために別のニューラルネットワークをトレーニング（<a href="https://blog.can.ac/2026/02/12/the-harness-problem/" target="_blank">Can Bölük</a>）
- **Stripe Minions**：意見を持ったオーケストレーションを備えたBlockのオープンソース「goose」エージェントのカスタムフォーク
- **LangChain/LangGraph**：ミドルウェアシステムを備えたフレームワークレベルのハーネス
- **Harbor**：ハーネス設定のベンチマークテスト用オーケストレーション

### 6.4 成功するもの vs 失敗するもの

実践者は明確なやるべきこととやってはいけないことに収束している（<a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer</a>）：

**失敗したこと**：
- 実際の失敗に遭遇する前に理想的なハーネスを設計すること
- 多数のスキル/MCPサーバーを先行的にインストールすること
- 変更のたびにフルテストスイートを実行すること（コンテキストを溢れさせる）
- 過度に細かいツールアクセス制御

**成功したこと**：
- シンプルに始め、エージェントの失敗後にのみ設定すること
- テスト、反復、非効果的な追加の破棄
- 実戦で検証された設定をチーム間で配布すること
- 初回の成功よりも反復速度を優先すること
- 必要な能力を公開し、慎重に削減していくこと

---

## 7. コミュニティ分析と批判的視点

### 7.1 「大きなモデル vs 大きなハーネス」論争

モデル能力が最重要と考える人々と、ハーネスが同等に重要と見なす人々の間に根本的な緊張関係が存在する（<a href="https://www.latent.space/p/ainews-is-harness-engineering-real" target="_blank">Latent Space</a>）：

**大きなモデル派の立場**：
- Boris Cherny（Claude Code）：「すべての秘密のソースはモデルにある」「最も薄いラッパー」
- Noam Brown：推論モデルが複雑なエージェントシステムの必要性を排除した
- Scale AIのSWE-Atlas：ハーネスの選択は「誤差の範囲内のノイズ」に過ぎない
- METRデータ：基本的なスキャフォールドが専門化されたシステムに匹敵する

**大きなハーネス派の立場**：
- Jerry Liu：「モデルハーネスこそがすべて — AIから価値を引き出す最大の障壁は、コンテキストとワークフローをエンジニアリングする自分自身の能力だ」
- Can Bölük：Geminiの+8%改善は「ほとんどのモデルアップグレードが提供するものより大きく、トレーニング計算コストはゼロ」
- Cursorの500億ドルの評価はハーネスエンジニアリングが実質的価値を捕捉していることを示唆
- すべての本番エージェントはモデルに関係なく類似のコアループに収束する

実践的な答えはおそらく相補的な価値を含み、この分野はハーネスエンジニアリングを正当な学問分野として認識する方向に向かっている。

### 7.2 Martin Fowler / Thoughtworksの分析

Martin FowlerのサイトにおけるBirgitta Böckelerの分析が最もバランスの取れた評価を提供している（<a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Martin Fowler / Thoughtworks</a>）：

**強み**：
- スケールでの保守可能なAI生成コードの実践的デモンストレーション
- 信頼には無制限の柔軟性よりも解決空間の制約が必要という認識
- 長期的な内部品質と保守性への具体的なフォーカス

**顕著なギャップ**：「機能と振る舞い」の検証が欠如 — ハーネスはコードの書き方と構造を制約するが、コードがユーザーのニーズを満たすかは検証していない。

**戦略的洞察**：
1. **実行時制約が自律性を可能にする**：逆説的に、AIの信頼性には技術的柔軟性の制限が必要
2. **技術スタックの統合**：人間のコーディングが減少するにつれ、開発者の好みの重要性が低下 — 組織はハーネスの可用性が優れた少数のスタックに収束する可能性
3. **コードベーストポロジーの標準化**：保守性に優しい構造がデフォルトの抽象化になる可能性

### 7.3 「信頼負債」の概念

「信頼負債（Trust Debt）」の概念は根本的なリスクを強調する：指示の曖昧さはすべて、意図せず委任された決定である。AIが曖昧な指示のギャップを自身の判断で埋める時、これらの選択はシステムが予期せず壊れるまで黙って蓄積される（<a href="https://decision.substack.com/p/harness-engineering-how-to-supervise" target="_blank">Decision Substack</a>）。

提案される緩和策は「二重翻訳習慣」：
1. コード生成前に、AIに指示の理解と仮定を説明させる
2. コード生成後に、別のAIにコードが実際に何をするかを説明させる
3. 両方を元の意図と比較する

### 7.4 哲学的批判

Andrew Maynardはハーネスの比喩に埋め込まれた3つの問題のある仮定を特定している（<a href="https://www.futureofbeinghuman.com/p/what-we-miss-when-we-talk-about-ai-harnesses" target="_blank">Future of Being Human</a>）：

1. **クリーンな分離の仮定**：ハーネスは人間が指示しAIが実行することを前提とするが、AIシステムがすでに運用上の判断を行うシナリオを考慮していない
2. **変わらないユーザーの仮定**：AIの展開後にユーザーが変わらないことを前提とするが、対話が本質的に理解を再形成することを見逃している
3. **道具的フレーミング**：AIは「単なるツール」であるという主張を永続化するが、先進技術が認知的景観を積極的に再形成するという証拠がある

### 7.5 セキュリティと自律性の懸念

エージェントのスループットはコードベースのリスクプロファイルを変える。エージェントを生産的にするハーネスは、リポジトリ、ビルドシステム、認証情報、デプロイメントパスウェイに対するレバレッジも与える。具体的な懸念事項：

- エージェントの認証情報アクセスによるシークレットと機密データの漏洩
- 「承認疲れ」 — すべてのアクションに人間の承認を強制すると、焦って「はい」をクリックするようになり、「実際の監視なしの監視の外観」が生まれる（<a href="https://www.helpnetsecurity.com/2026/03/05/securing-autonomous-ai-agents/" target="_blank">Help Net Security</a>）
- 開発の早い段階での敵対的脅威モデリング、隔離されたサンドボックス環境、最小限のツールアクセス、機密アクションのための人間介入の必要性

### 7.6 苦い教訓の適用

Philipp SchmidはRich Suttonの「苦い教訓（Bitter Lesson）」 — 計算を使用する一般的な方法が手動でコード化された人間の知識に常に勝つ — を参照している。裏付ける証拠：Manusは6ヶ月で5回ハーネスをリファクタリングし、LangChainは年に3回エージェントを再アーキテクトし、Vercelはエージェントツールの80%を削除してステップ数とトークン数の削減を達成した。各モデルリリースが異なる構造的アプローチを要求するため、ハーネスは軽量に保つ必要がある（<a href="https://www.philschmid.de/agent-harness-2026" target="_blank">Philipp Schmid</a>）。

### 7.7 ブラウンフィールド問題

成功事例はグリーンフィールドプロジェクトまたはハーネスをゼロから構築するチームが主流である。一貫性のないテストとドキュメントを持つ10年前のコードベースにこれらのテクニックを適用することは、ほぼ未解決のままである。Böckelerは、これがAI以前とAI以後のアプリケーション間の重要な区別を生み出し、非標準化されたレガシーシステムへのハーネスの後付けは収穫逓減をもたらす可能性があると指摘している（<a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Martin Fowler / Thoughtworks</a>）。

---

## 8. ソフトウェアエンジニアリングへの影響

### 8.1 役割の変革

エンジニアの役割は、コード作成者から環境設計者へ根本的にシフトしている。OpenAIが述べたように：「エンジニアリングチームの主な仕事は、エージェントが有用な仕事をできるようにすることになりつつある」（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ</a>）。

エンジニアリング作業は2つの機能に分割される（<a href="https://www.ignorance.ai/p/the-emerging-harness-engineering" target="_blank">Ignorance.ai</a>）：
1. **環境の構築**：構造、ツール、フィードバックメカニズムの作成
2. **作業の管理**：包括的な計画、批判的なレビュー、並列エージェントセッションのオーケストレーション

### 8.2 Anthropicの8つのトレンド

Anthropicの2026年エージェンティックコーディングトレンドレポートは主要なシフトを特定している（<a href="https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report" target="_blank">Tessl</a>）：

1. エンジニアはハンズオンコーディングから監督とレビューへ進化
2. マルチエージェントコラボレーションがシングルエージェントワークフローを置き換え
3. 数時間から数日にわたる拡張期間タスクが可能に
4. エージェントが不確実性を検出し人間の入力を要求するインテリジェントヘルプシーキング
5. プロフェッショナル開発者を超えたユーザーベースの拡大
6. 加速されたデリバリー — 「かつて数週間かかった作業が数日で完了可能」
7. 非エンジニアの導入（ビジネス部門が独立してツールを構築）
8. 二面性のあるセキュリティインパクト（防御能力の向上と新たな攻撃面の同時出現）

### 8.3 レビューのボトルネック

AIエージェントによる生産性向上はレビュープロセスに深刻なボトルネックを生む。チームは98%多いPRマージを報告するが、PRレビュー時間は91%長くなり、1行ずつの人間によるレビューは持続不可能になっている。これはアーキテクチャゲートキーピング（すべてのdiffを読まずに「テイスト」を維持する）とAI自動レビューシステムへの移行を推進している（<a href="https://www.ignorance.ai/p/the-emerging-harness-engineering" target="_blank">Ignorance.ai</a>）。

### 8.4 求められるスキルの進化

エンジニアに求められるコアコンピテンシーがシフトしている：
- システム思考とアーキテクチャ設計
- 仕様記述と意図の明確化
- 可観測性とモニタリング
- 反復速度とフィードバックループ設計
- エージェント監督とハイステークスな技術的判断

ある分析が述べるように：「Vibeコーディングはシンタックスの壁を下げる。しかしシステム思考の壁は下げない」（<a href="https://decision.substack.com/p/harness-engineering-how-to-supervise" target="_blank">Decision Substack</a>）。

### 8.5 並列化が新しいワークフローに

2つの運用モードが出現している（<a href="https://www.ignorance.ai/p/the-emerging-harness-engineering" target="_blank">Ignorance.ai</a>）：
- **アテンド型**：3-4の同時エージェントセッションを積極的に管理
- **アンアテンド型**：開発者がタスクを投稿し、エージェントが監督なしでCIを通じて進行 — 成熟したハーネスインフラが必要

Peter Steinberger（OpenClaw）がアテンド型を体現：5-10のエージェントを同時に実行し、月間6,600以上のコミット、コードを1行ずつ読むことなく出荷。

---

## 9. 結論

### 証拠が示すもの

1. **ハーネスエンジニアリングは測定可能な結果を生み出す**：LangChainの13.7ポイントのベンチマーク改善、Can BölükのGrok Code Fast 1での10倍の改善、OpenAIの100万行のコードベースは、環境設計がエージェントパフォーマンスに過大な影響を持つことを実証している。

2. **ハーネスとモデルの関係は競合ではなく相補的**：「モデルを改善するだけ」でも「ハーネスを改善するだけ」でも単独では不十分。最良の結果は両者の思慮深い統合から生まれる。

3. **コンテキストは量より質が重要**：ETH Zurichの研究で、LLM生成コンテキストファイルがパフォーマンスを低下させながらコストを20%以上増加させるという知見は、重要な実証結果である。より少なく、より的を絞ったコンテキストが包括的なドキュメントを上回る。

4. **制約は逆説的に自律性を高める**：すべてのケーススタディにおいて、エージェントができることを制限することで生産性が向上する — 直感に反するが繰り返し検証された知見。

5. **この分野は初期段階だが収束している**：OpenAI、Stripe、Anthropic、LangChain、独立した開発者の実践者が、大きく異なるコンテキストにもかかわらず類似のパターンに収束している。

### 未解決の課題

1. **機能的検証**：ハーネスはコードの書き方を制約するが、コードがユーザーのニーズを満たすかを完全には検証していない
2. **ブラウンフィールドの導入**：レガシーコードベースへのハーネスエンジニアリングの適用はほぼ未解決
3. **ハーネスの脆弱性**：モデルの更新により既存のハーネス設定が頻繁に壊れ、常に適応が必要
4. **承認疲れ**：人間の監視メカニズムが誤ったセキュリティ感覚を生む可能性
5. **セキュリティへの影響**：エージェントの自律性が敵対的脅威モデリングを必要とする新たな攻撃面を導入
6. **測定基準**：異なるコンテキスト間でハーネスの有効性を比較するための広く受け入れられたベンチマークが存在しない

### まとめ

ハーネスエンジニアリングは、ソフトウェア開発方法の真のパラダイムシフトを代表している — 単なるバズワードではない。証拠はまだ初期段階であり、査読付き学術研究よりも主に実践者からのものだが、環境設計がモデル能力と同等以上にエージェントの有効性を決定することを一貫して示している。この分野は「AIはコードを書けるか？」から「AIに信頼性高くコードを書かせるシステムをどう構築するか？」へ移行しており — その問いは根本的にエンジニアリングの課題である。

---

## 10. 出典

1. <a href="https://openai.com/index/harness-engineering/" target="_blank">OpenAI — Harness engineering: leveraging Codex in an agent-first world</a>
2. <a href="https://openai.com/index/unlocking-the-codex-harness/" target="_blank">OpenAI — Unlocking the Codex harness: how we built the App Server</a>
3. <a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Martin Fowler / Birgitta Böckeler — Harness Engineering</a>
4. <a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ — OpenAI Introduces Harness Engineering</a>
5. <a href="https://www.infoq.com/news/2026/02/opanai-codex-app-server/" target="_blank">InfoQ — OpenAI Publishes Codex App Server Architecture</a>
6. <a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode — Harness Engineering: The Complete Guide (2026)</a>
7. <a href="https://arxiv.org/html/2602.11988v1" target="_blank">ETH Zurich — Evaluating AGENTS.md: Are Repository-Level Context Files Helpful for Coding Agents?</a>
8. <a href="https://arxiv.org/html/2603.05344v1" target="_blank">arXiv — Building AI Coding Agents for the Terminal</a>
9. <a href="https://blog.langchain.com/improving-deep-agents-with-harness-engineering/" target="_blank">LangChain — Improving Deep Agents with Harness Engineering</a>
10. <a href="https://blog.can.ac/2026/02/12/the-harness-problem/" target="_blank">Can Bölük — I Improved 15 LLMs at Coding in One Afternoon</a>
11. <a href="https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents" target="_blank">Stripe Dev Blog — Minions: Stripe's one-shot, end-to-end coding agents</a>
12. <a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer — Skill Issue: Harness Engineering for Coding Agents</a>
13. <a href="https://www.ignorance.ai/p/the-emerging-harness-engineering" target="_blank">Ignorance.ai — The Emerging "Harness Engineering" Playbook</a>
14. <a href="https://decision.substack.com/p/harness-engineering-how-to-supervise" target="_blank">Decision Substack — Harness Engineering: How to Supervise Code You Can't Read</a>
15. <a href="https://cobusgreyling.substack.com/p/the-rise-of-ai-harness-engineering" target="_blank">Cobus Greyling — The Rise of AI Harness Engineering</a>
16. <a href="https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents" target="_blank">Anthropic — Effective Harnesses for Long-Running Agents</a>
17. <a href="https://www.anthropic.com/engineering/claude-code-best-practices" target="_blank">Anthropic — Claude Code: Best Practices for Agentic Coding</a>
18. <a href="https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report" target="_blank">Tessl — Anthropic: 8 Agentic Coding Trends (2026)</a>
19. <a href="https://www.philschmid.de/agent-harness-2026" target="_blank">Philipp Schmid — The Importance of Agent Harness in 2026</a>
20. <a href="https://madplay.github.io/en/post/harness-engineering" target="_blank">MadPlay — Beyond Prompts and Context: Harness Engineering for AI Agents</a>
21. <a href="https://mitchellh.com/writing/my-ai-adoption-journey" target="_blank">Mitchell Hashimoto — My AI Adoption Journey</a>
22. <a href="https://addyosmani.com/blog/agents-md/" target="_blank">Addy Osmani — Stop Using /init for AGENTS.md</a>
23. <a href="https://www.latent.space/p/ainews-is-harness-engineering-real" target="_blank">Latent Space — Is Harness Engineering Real?</a>
24. <a href="https://www.futureofbeinghuman.com/p/what-we-miss-when-we-talk-about-ai-harnesses" target="_blank">Andrew Maynard — What We Miss When We Talk About "AI Harnesses"</a>
25. <a href="https://www.helpnetsecurity.com/2026/03/05/securing-autonomous-ai-agents/" target="_blank">Help Net Security — Engineering Trust: A Security Blueprint for Autonomous AI Agents</a>
26. <a href="https://developers.openai.com/codex/concepts/sandboxing" target="_blank">OpenAI Developers — Codex Sandboxing</a>
27. <a href="https://developers.openai.com/codex/app-server" target="_blank">OpenAI Developers — Codex App Server</a>
28. <a href="https://www.infoq.com/news/2026/03/stripe-autonomous-coding-agents/" target="_blank">InfoQ — Stripe Engineers Deploy Minions</a>
29. <a href="https://www.marktechpost.com/2026/02/25/new-eth-zurich-study-proves-your-ai-coding-agents-are-failing-because-your-agents-md-files-are-too-detailed/" target="_blank">MarkTechPost — New ETH Zurich Study on AGENTS.md Files</a>
30. <a href="https://www.mindstudio.ai/blog/what-is-ai-agent-harness-stripe-minions" target="_blank">MindStudio — What Is an AI Agent Harness? (Stripe's Architecture)</a>
