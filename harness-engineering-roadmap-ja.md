# ハーネスエンジニアリング ロードマップ

ハーネスエンジニアリングの導入、学習、業界進化に関する包括的ロードマップ。OpenAI、Stripe、Anthropic、LangChain、ETH Zurich等30以上のソースからの調査研究に基づく。

---

## 目次

1. [パートI：チーム導入ロードマップ](#パートi-チーム導入ロードマップ)
2. [パートII：学習ロードマップ](#パートii-学習ロードマップ)
3. [パートIII：業界進化ロードマップ](#パートiii-業界進化ロードマップ)

---

## パートI: チーム導入ロードマップ

### 概要

ハーネスエンジニアリングの導入は段階的モデルに従う。実践者からの重要原則：**シンプルに始め、失敗後にのみ設定し、たゆまず反復する**（<a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer</a>）。実際のエージェント失敗に遭遇する前に理想的なハーネスを事前設計することは、最も一般的なアンチパターンの一つである。

---

### フェーズ0：基盤構築（第1週）

**目標**：AIエージェント導入の前提条件を確立する。

| アクション | 詳細 |
|-----------|------|
| コードベースの監査 | テストカバレッジ、リンティングルール、CI/CDの成熟度、ドキュメント品質を評価 |
| CI/CDの構築 | すべてのPRで自動ビルド、テスト、リンティングが実行されることを確認 |
| エージェントツールの選択 | OpenAI Codex、Claude Code、Cursor等 |
| サンドボックスの作成 | エージェント実行を本番システムと認証情報から分離 |

**重要指標**：CIパイプラインが全コミットで意味のあるテストカバレッジとともにグリーンで実行される。

**なぜ重要か**：OpenAIの実験は、エージェントが厳格な境界と予測可能な構造を持つ環境で最も効果的であることを示した。CI/CDとテストなしにはフィードバックループがなく、フィードバックなしではハーネスエンジニアリングは機能しない。

---

### フェーズ1：個人開発者（第2〜3週）

**目標**：1人のエンジニアがAIコーディングエージェントで生産的になる。

**時間投資**：1〜2時間のセットアップ（<a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode</a>）

| アクション | 詳細 |
|-----------|------|
| CLAUDE.md / AGENTS.mdの作成 | 60行以下に抑える。エージェントがコードから**推論できない**情報のみを含める：ビルドコマンド、自明でないツール、落とし穴（<a href="https://addyosmani.com/blog/agents-md/" target="_blank">Addy Osmani</a>） |
| プレコミットフックの追加 | コミットがCIに到達する前にフォーマット、リント、型チェック |
| 命名規則の確立 | 一貫したファイル/関数/変数の命名がエージェントの混乱を減らす |
| 失敗ログの開始 | エージェントがミスするたびに、何が間違ったか、どのような修正を適用したかを記録 |

**避けるべきアンチパターン**：
- `/init`でAGENTS.mdを自動生成すること — ETH Zurichがパフォーマンスを約2%低下させ、コストを20%以上増加させることを証明した（<a href="https://arxiv.org/html/2602.11988v1" target="_blank">ETH Zurich</a>）
- 包括的なコードベース概要を書くこと — エージェントはコードを読んで構造を発見できる

**重要指標**：エージェントが手動コード介入なしで、シンプルで明確に定義されたタスク（バグ修正、小さな機能）を正常に完了する。

---

### フェーズ2：小規模チーム（第4〜6週）

**目標**：2〜5人のエンジニアチームがエージェントと効果的に協力する。

**時間投資**：1〜2日のセットアップ（<a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode</a>）

| アクション | 詳細 |
|-----------|------|
| CI強制のアーキテクチャ制約 | 依存関係の方向、命名規則、モジュール境界を強制するカスタムリンターを追加。エラーメッセージに修復指示を含める（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ/OpenAI</a>） |
| 共有プロンプトライブラリ | チームが検証したタスクテンプレート（機能実装、バグ修正、リファクタリング、テスト作成） |
| 構造化された進捗追跡 | セッション間のタスク完了追跡にJSON（Markdownではなく）を使用 — モデルはJSONを上書きしにくい（<a href="https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents" target="_blank">Anthropic</a>） |
| サブエージェント戦略 | サブエージェントを「コンテキストファイアウォール」として使用 — リサーチ、探索、実装を別々のコンテキストウィンドウに分離（<a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer</a>） |
| レビュープロセスの適応 | 1行ずつのレビューはスケールしないことを受け入れる。アーキテクチャのゲートキーピングと自動品質チェックへ移行 |
| Documentation-as-Code | 設計ドキュメントをリポジトリ内に保持し、リンターで検証、機械的にクロスリンク |

**避けるべきアンチパターン**：
- 利用可能なすべてのMCPサーバーをインストールすること（コンテキストウィンドウを溢れさせる）
- エージェントの変更のたびにフルテストスイートを実行すること（選択的でコンテキスト効率の良いテストを使用 — 失敗のみを表示）
- 過度に細かいツールアクセス制御

**重要指標**：チームが許容可能な品質で一貫したエージェントスループット（例：エンジニア1人あたり1日2以上のPR）を達成する。

---

### フェーズ3：プロダクションレベル（第7〜12週）

**目標**：無人エージェント運用をサポートする成熟したハーネスインフラ。

**時間投資**：1〜2週間のセットアップ（<a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode</a>）

| アクション | 詳細 |
|-----------|------|
| カスタムミドルウェア | 検証ミドルウェア（事前完了チェックリスト、ループ検出、コンテキスト注入）を構築（<a href="https://blog.langchain.com/improving-deep-agents-with-harness-engineering/" target="_blank">LangChain</a>） |
| 可観測性の統合 | エージェントをログ、メトリクス、トレースに接続し、動作の監視、バグの再現、修正の検証を可能にする（<a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ/OpenAI</a>） |
| エントロピー管理エージェント | 定期的な「ガベージコレクション」エージェント（日次/週次）をスケジュールし、ドキュメントのドリフト、アーキテクチャ違反、パターンの不整合を検出（<a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode</a>） |
| ライフサイクルフック | ツールコール前後のアクション自動化：型チェック、フォーマット、ビルド検証。エラーのみを表示（<a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer</a>） |
| サンドボックス実行環境 | 各エージェントタスクのための事前ウォーム・隔離環境（Stripeは10秒スピンアップの「devbox」を使用）（<a href="https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents" target="_blank">Stripe</a>） |
| セキュリティ強化 | 敵対的脅威モデリング、最小限の認証情報アクセス、サンドボックスネットワーク分離、機密操作のための人間介入（<a href="https://www.helpnetsecurity.com/2026/03/05/securing-autonomous-ai-agents/" target="_blank">Help Net Security</a>） |
| 監視ダッシュボード | エージェント成功率、タスクあたりのコスト、コンテキストウィンドウ使用率、エントロピー指標を追跡 |

**重要指標**：エージェントが無人モードで運用可能 — タスクを非同期で提出、PRがCIを通過し、インタラクションなしで人間のレビュー準備完了。

---

### フェーズ4：組織規模（4〜6ヶ月以上）

**目標**：ハーネスエンジニアリングが組織能力となる。

| アクション | 詳細 |
|-----------|------|
| ハーネスをプラットフォーム化 | チームがカスタマイズする内部ハーネステンプレートを構築 — 現在のサービステンプレートに類似（<a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Böckeler/Fowler</a>） |
| マルチエージェントオーケストレーション | 特化エージェントの並列展開：コーディング、テスト、レビュー、クリーンアップ（<a href="https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report" target="_blank">Anthropic Trends</a>） |
| エージェント支援レビュー | エージェント生成PRのAI自動コードレビュー、人間はアーキテクチャ決定に集中 |
| コスト最適化 | 高価モデル（Opus）をオーケストレーション/計画に、安価モデル（Sonnet/Haiku）をサブエージェントタスクに使用 |
| チーム横断設定共有 | 実戦検証済みハーネス設定を組織全体に配布 |
| 継続的ハーネス反復 | ハーネスを生きたシステムとして扱う — モデル更新のたびにハーネス調整が必要な場合がある（<a href="https://www.philschmid.de/agent-harness-2026" target="_blank">Philipp Schmid</a>） |

**重要指標**：組織全体のエージェント導入と測定可能な生産性向上（例：OpenAIの約10倍の速度推定、Stripeの週1,300以上のPR）。

---

### 導入ロードマップ概観

```
第1週           第2-3週           第4-6週           第7-12週          4-6ヶ月以上
┌──────────┐   ┌──────────┐    ┌──────────┐    ┌──────────────┐  ┌──────────────┐
│フェーズ0  │──▶│フェーズ1  │───▶│フェーズ2  │───▶│ フェーズ3    │─▶│ フェーズ4    │
│基盤構築   │   │個人開発者 │    │小規模チーム│   │プロダクション│  │組織規模      │
│           │   │          │    │          │    │              │  │              │
│• CI/CD   │   │• AGENTS.md│   │• リンター │    │• ミドルウェア │  │• プラット    │
│• テスト  │   │• フック    │   │• 共有     │    │• 可観測性    │  │  フォーム化  │
│• サンド  │   │• 命名規則  │   │  プロンプト│   │• エントロピー│  │• マルチ     │
│  ボックス│   │• 失敗ログ  │   │• サブ     │    │  管理       │  │  エージェント│
│• ツール  │   │           │   │  エージェント│  │• サンドボックス│ │• AIレビュー │
│  選択    │   │           │   │           │   │• セキュリティ │  │• コスト最適化│
└──────────┘   └──────────┘    └──────────┘    └──────────────┘  └──────────────┘
   1-2時間       1-2時間         1-2日           1-2週間           継続的
```

---

## パートII: 学習ロードマップ

### 概要

ハーネスエンジニアリングは複数の分野から知見を引き出す：ソフトウェアアーキテクチャ、DevOps、可観測性、AI/MLエンジニアリング。この学習パスは基礎概念から上級専門化まで整理されている。

---

### レベル1：基礎（1〜2週間）

**目標**：ハーネスエンジニアリングとは何か、なぜ重要かを理解する。

#### 学ぶべきコア概念

| トピック | 主要リソース |
|---------|-------------|
| ハーネスエンジニアリングとは？ | <a href="https://openai.com/index/harness-engineering/" target="_blank">OpenAIブログ記事</a> — 基礎となる記事 |
| プロンプト vs コンテキスト vs ハーネスエンジニアリング | <a href="https://madplay.github.io/en/post/harness-engineering" target="_blank">MadPlay — Beyond Prompts and Context</a> |
| ハーネスのメタファーとその限界 | <a href="https://www.futureofbeinghuman.com/p/what-we-miss-when-we-talk-about-ai-harnesses" target="_blank">Andrew Maynard — What We Miss</a> |
| 「大きなモデル vs 大きなハーネス」論争 | <a href="https://www.latent.space/p/ainews-is-harness-engineering-real" target="_blank">Latent Space — Is HE Real?</a> |

#### ハンズオン演習
1. 個人プロジェクトまたは小さなコードベースを選ぶ
2. 1つのAIコーディングエージェント（Claude Code、Codex、Cursor）をセットアップ
3. エージェントで10のタスクを試行し、すべての失敗を記録
4. 上位5つの失敗に対処するAGENTS.mdを作成
5. 同じタスクを再実行し、改善を測定

---

### レベル2：コアプラクティス（2〜4週間）

**目標**：3つの柱をマスターする — コンテキストエンジニアリング、アーキテクチャ制約、エントロピー管理。

#### コンテキストエンジニアリング

| トピック | 主要リソース |
|---------|-------------|
| 効果的なAGENTS.mdファイル | <a href="https://addyosmani.com/blog/agents-md/" target="_blank">Addy Osmani — Stop Using /init</a> |
| ETH Zurichのコンテキストファイル研究 | <a href="https://arxiv.org/html/2602.11988v1" target="_blank">arXiv論文</a> |
| 構造化された進捗追跡 | <a href="https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents" target="_blank">Anthropic — Effective Harnesses</a> |
| コンテキストロットと劣化 | <a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer — Skill Issue</a> |

**重要な洞察**：コンテキストは量より質。より少なく、より的を絞った情報が包括的なドキュメントを上回る。

#### アーキテクチャ制約

| トピック | 主要リソース |
|---------|-------------|
| 教育メカニズムとしてのカスタムリンター | <a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ — OpenAIのアプローチ</a> |
| 構造テスト（ArchUnitスタイル） | <a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Böckeler — 実践的推奨事項</a> |
| レイヤード依存関係の強制 | OpenAIのTypes → Config → Repo → Service → Runtime → UIパターン |

**重要な洞察**：制約が生産性を高める。エージェントができることを制限すると、行き止まりの探索がなくなる。

#### エントロピー管理

| トピック | 主要リソース |
|---------|-------------|
| ガベージコレクションエージェント | <a href="https://www.nxcode.io/resources/news/harness-engineering-complete-guide-ai-agent-codex-2026" target="_blank">NxCode完全ガイド</a> |
| ゴールデンプリンシプルと機械的ルール | <a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">InfoQ — OpenAI実験</a> |

#### ハンズオン演習
1. 修復メッセージ付きの3つのカスタムリントルールをプロジェクトに追加
2. 依存関係の方向を強制する構造テストをセットアップ
3. ドキュメントのドリフトを特定するスケジュール「クリーンアップエージェント」を作成

---

### レベル3：上級ツール（2〜4週間）

**目標**：サブエージェント、フック、ミドルウェア、MCP統合をマスターする。

| トピック | 主要リソース |
|---------|-------------|
| コンテキストファイアウォールとしてのサブエージェント | <a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer — 詳細解説</a> |
| MCPサーバー統合 | <a href="https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents" target="_blank">Stripe — 400以上のMCPツール</a> |
| ライフサイクルフック | <a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer — フックパターン</a> |
| ミドルウェアパターン | <a href="https://blog.langchain.com/improving-deep-agents-with-harness-engineering/" target="_blank">LangChain — ミドルウェアアプローチ</a> |
| 編集ツール設計（hashline） | <a href="https://blog.can.ac/2026/02/12/the-harness-problem/" target="_blank">Can Bölük — The Harness Problem</a> |

#### ハンズオン演習
1. エージェントがループに陥った時に検出するミドルウェアを構築
2. 完了前検証フックを作成
3. マルチステップ機能のためのサブエージェントワークフローを設定

---

### レベル4：プロダクションエンジニアリング（4週間以上）

**目標**：プロダクショングレードのハーネスインフラを設計・運用する。

| トピック | 主要リソース |
|---------|-------------|
| エージェントの可観測性 | <a href="https://www.infoq.com/news/2026/02/openai-harness-engineering-codex/" target="_blank">OpenAI — テレメトリ統合</a> |
| サンドボックスと分離 | <a href="https://developers.openai.com/codex/concepts/sandboxing" target="_blank">OpenAI — Codexサンドボックス</a>、<a href="https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents" target="_blank">Stripe devbox</a> |
| セキュリティと信頼エンジニアリング | <a href="https://www.helpnetsecurity.com/2026/03/05/securing-autonomous-ai-agents/" target="_blank">Help Net Security — セキュリティブループリント</a> |
| Codex App Serverアーキテクチャ | <a href="https://www.infoq.com/news/2026/02/opanai-codex-app-server/" target="_blank">InfoQ — App Server詳細</a> |
| マルチエージェント協調 | <a href="https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report" target="_blank">Anthropicトレンドレポート</a> |

#### ハンズオン演習
1. エージェントタスク用のサンドボックス実行環境を構築
2. エージェント成功率、コスト、コンテキスト使用率を追跡する可観測性ダッシュボードを構築
3. エージェントワークフローの敵対的脅威モデリングを実装

---

### 必読リスト

| 優先度 | リソース | 理由 |
|--------|----------|------|
| 1 | <a href="https://openai.com/index/harness-engineering/" target="_blank">OpenAI — Harness Engineering</a> | すべての始まりとなった基礎的記事 |
| 2 | <a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Böckeler / Fowler — Harness Engineering</a> | 最もバランスの取れた批判的分析 |
| 3 | <a href="https://www.humanlayer.dev/blog/skill-issue-harness-engineering-for-coding-agents" target="_blank">HumanLayer — Skill Issue</a> | 最も実践的なハンズオンガイド |
| 4 | <a href="https://arxiv.org/html/2602.11988v1" target="_blank">ETH Zurich — コンテキストファイル評価</a> | 主要な実証研究 |
| 5 | <a href="https://blog.langchain.com/improving-deep-agents-with-harness-engineering/" target="_blank">LangChain — Improving Deep Agents</a> | 厳密なベンチマーク方法論 |
| 6 | <a href="https://stripe.dev/blog/minions-stripes-one-shot-end-to-end-coding-agents" target="_blank">Stripe — Minions</a> | エンタープライズスケールの本番実装 |
| 7 | <a href="https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents" target="_blank">Anthropic — Effective Harnesses</a> | 長時間実行エージェントのパターン |
| 8 | <a href="https://blog.can.ac/2026/02/12/the-harness-problem/" target="_blank">Can Bölük — The Harness Problem</a> | ハーネス＞モデルアップグレードの実証的証明 |
| 9 | <a href="https://www.latent.space/p/ainews-is-harness-engineering-real" target="_blank">Latent Space — Is HE Real?</a> | 論争に関する批判的視点 |
| 10 | <a href="https://www.philschmid.de/agent-harness-2026" target="_blank">Philipp Schmid — Agent Harness 2026</a> | ハーネスに適用された苦い教訓 |

---

## パートIII: 業界進化ロードマップ

### 概要

ハーネスエンジニアリングは2026年2月に出現し、急速に進化している。本セクションでは、現在の証拠、実践者パターン、特定されたギャップに基づいて軌道を予測する。

---

### 2026年上半期：出現（現在の状態）

**ステータス**：用語が定義され、アーリーアダプターが結果を実証し、基礎パターンが結晶化。

| 起きていること | 証拠 |
|--------------|------|
| 用語の形式化 | Mitchell HashimotoとOpenAIが2026年2月に公開 |
| 初の本番ケーススタディ | OpenAI（約100万行）、Stripe（週1,300以上のPR） |
| 初の実証研究 | ETH Zurichコンテキストファイル研究、LangChainベンチマーク、Hashline実験 |
| 初の批判的分析 | Böckeler/Fowler、Latent Space、Maynardの哲学的批判 |
| カンファレンスでの認知 | AIE Europeが「世界初のハーネスエンジニアリングトラック」を開催 |
| ツールサポートの出現 | Claude Code、Codex、Cursorすべてがコアパターンをサポート |

**未解決**：標準ベンチマークなし、グリーンフィールドバイアス、機能検証のギャップ、ブラウンフィールド導入が不明確。

---

### 2026年下半期：統合（予測）

**現在の軌道に基づく予想される展開**：

| トレンド | 根拠 |
|---------|------|
| **標準ベンチマークの出現** | Terminal Bench 2.0が出発点；専用ハーネス評価フレームワークの登場を期待 |
| **ハーネスをプラットフォーム化した製品** | Böckelerの予測：ハーネスがサービステンプレートの後継に（<a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Fowler</a>） |
| **ブラウンフィールド適応パターン** | 最大のギャップ — レガシーコードベースの後付けに関する集中的な研究を期待 |
| **セキュリティフレームワークの成熟** | エージェント対応環境のための敵対的脅威モデルが標準に |
| **AI支援コードレビュー** | 98%増のPRでの人間の1行ずつレビューはスケールしない — 自動レビューが必要に（<a href="https://www.ignorance.ai/p/the-emerging-harness-engineering" target="_blank">Ignorance.ai</a>） |
| **マルチエージェントアーキテクチャ** | 単一エージェントから特化エージェントチーム（コーディング、テスト、レビュー、クリーンアップ）へ（<a href="https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report" target="_blank">Anthropic</a>） |

---

### 2027年：成熟（予測）

| トレンド | 根拠 |
|---------|------|
| **技術スタックの統合** | 組織がハーネスの可用性が優れた少数のスタックに収束（<a href="https://martinfowler.com/articles/exploring-gen-ai/harness-engineering.html" target="_blank">Böckeler</a>） |
| **機能検証の解決** | ハーネスが構造を制約するが振る舞いを検証しない現在のギャップが解消 |
| **コードベーストポロジーの標準化** | 保守性に優しい構造がデフォルトアーキテクチャパターンに |
| **ハーネスの脆弱性の低減** | フレームワークがよりモデル非依存になり、モデル更新時の破損が減少 |
| **非エンジニアの導入** | ビジネスチームが成熟したハーネスインフラを使用して独立してツールを構築（<a href="https://tessl.io/blog/8-trends-shaping-software-engineering-in-2026-according-to-anthropics-agentic-coding-report" target="_blank">Anthropic</a>） |

---

### 2028年以降：新しい均衡（推測的）

| 可能性 | 根拠 |
|--------|------|
| **ハーネスエンジニアリングがCS教育のコアカリキュラムに** | この分野が耐久性を証明すれば、大学がソフトウェアアーキテクチャと並んで教える |
| **モデルとハーネスの共進化** | モデルのトレーニングが特定のハーネスパターン向けに最適化され、現在の分離が曖昧に |
| **エージェントネイティブなソフトウェアパラダイム** | エージェントファースト開発のために設計された新しいプログラミング言語、フレームワーク、アーキテクチャ |
| **苦い教訓の実現** | Schmidの分析：一般的な手法が手作りのハーネスに勝ち、モデルの改善とともに分野が劇的に簡素化 |
| **あるいは：ハーネスが堀（モート）になる** | Cursorの500億ドルの評価：モデルがコモディティ化しても環境エンジニアリング層が持続的価値を捕捉 |

---

### 業界進化の概観

```
2026年上半期         2026年下半期         2027年              2028年以降
出現                統合                成熟                新しい均衡
──────────────────────────────────────────────────────────────────────────

用語の定義           標準ベンチマーク      技術スタック         エージェント
初のケーススタディ    ハーネスを           統合                ネイティブ
初の研究             プラットフォーム化    機能検証            パラダイム
批判的分析           ブラウンフィールド    トポロジー           モデル-ハーネス
カンファレンス       パターン             標準化              共進化
ツールサポート       セキュリティ         ハーネス             コアカリキュラム？
                    フレームワーク        回復力              あるいは苦い教訓が
                    AIレビュー           非エンジニア          すべてを簡素化
                    マルチエージェント     導入

├── 現在地
```

---

### 主要な不確実性

以下の問いが軌道を形成する：

1. **モデルの改善がハーネスを陳腐化させるか？** 苦い教訓は一般的な計算が手作りの知識に勝つことを示唆。しかし現在の証拠はハーネスとモデルの改善が競合ではなく相補的であることを示す。

2. **ブラウンフィールドの導入は機能するか？** すべての主要な成功事例はグリーンフィールド。レガシーコードベースが恩恵を受けられなければ、この分野の範囲は大幅に狭まる。

3. **誰が価値を捕捉するか — モデルプロバイダーかハーネスビルダーか？** Cursorの500億ドルの評価 vs 「すべての秘密のソースはモデルにある」は未解決の経済的問題。

4. **標準が出現するか、断片化が続くか？** Codex App ServerとMCPはエージェント統合への競合するアプローチ。業界は収束する必要がある。

5. **セキュリティリスクはどうスケールするか？** エージェントの自律性 + 認証情報アクセス + デプロイメントパスウェイ = 導入とともに拡大する攻撃面。

---

## まとめ：今すべきこと

この分野がどう進化するかに関係なく、以下のアクションはエビデンスに裏付けられ、リスクが低い：

1. **今日からエージェントの失敗を記録する** — これがハーネス改善の原材料
2. **最小限の手作りAGENTS.mdを書く** — 60行以下、推論不可能な情報のみ
3. **修復メッセージ付きカスタムリンターを追加する** — OpenAIとStripeの事例による最もレバレッジの高い投資
4. **コンテキスト分離にサブエージェントを使用する** — コンテキストロットの防止が証明済み
5. **テストスイートをコンテキスト効率良くする** — 通過出力を抑制、失敗のみを表示
6. **ハーネスを生きたシステムとして扱う** — エージェントの失敗を反復し、最初から完璧に設計しようとしない

証拠は一貫して示している：**エージェントパフォーマンスのボトルネックは環境設計であり、モデルのインテリジェンスではない**。今日ハーネスエンジニアリングに投資することは、明日どのモデルやツールが出現するかに関係なく、持続的な競争優位性を構築する。
