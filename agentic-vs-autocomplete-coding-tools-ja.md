# エージェント型コーディングツール vs. オートコンプリート型アシスタント

## AI支援ソフトウェア開発における次のパラダイムシフトの実証的エビデンス（2024年–2026年）

---

## エグゼクティブサマリー

AIコーディングアシスタントの風景はパラダイムシフトを迎えている。第一世代ツール — GitHub Copilot、Codeium、TabNine — はインラインのオートコンプリートエンジンとして動作し、単一ファイル内で次の数行のコードを提案していた。第二世代の**エージェント型コーディングツール** — Claude Code、Cursor Agent、Devin、OpenAI Codex、GitHub Copilot Agent Mode — はリポジトリレベルで動作する：マルチステップのタスクを計画し、複数ファイルを編集し、ターミナルコマンドを実行し、テストを走らせ、失敗に対して自律的に反復する。

本レポートは、学術論文（MSR '26、ICSE '25、NeurIPS 2024–2025）、業界レポート（Anthropic、DORA、Sonar、Pragmatic Engineer）、約100万件のエージェント型プルリクエストの大規模分析に基づき、生産性、コード品質、ベンチマーク性能、コスト、リスクにわたる2つのパラダイムの比較に関する新興の実証的エビデンスを統合する。

**主要な知見：**

- **ベンチマークのギャップは巨大。** トップエージェントはSWE-bench Verified（単独バグ修正）で74–81%を達成するが、複雑な実世界ベンチマークでは11–23%に崩壊する：FeatureBench（マルチファイル機能開発）、SWE-Bench Pro（長期的タスク）、SWE-EVO（リリース間のソフトウェア進化）。
- **エージェント型ツールは速度を向上させるが品質を低下させる。** 806のCursor導入リポジトリの差分の差分研究で、統計的に有意だが*一時的な*速度向上と、*持続的な*静的解析警告とコード複雑度の増加が確認された（He et al., MSR '26）。
- **PR受け入れ率はエージェントとタスクにより劇的に変動。** 7,156件のエージェント型PRにおいて、OpenAI Codexが77.9%でリード、Claude Codeが71.9%、Devinが61.6%。タスクタイプが支配的要因：ドキュメントPRの受け入れ率は82.1%、機能は66.1%。
- **エージェントは人間とは異なるリファクタリングを行う。** Claude Codeはリファクタリングの91%をアノテーションに集中；人間はExtract MethodやMove Classなどの構造的変更に分散。Cursor Agentのみがコードスメルを有意に増加させた（Ottenhof et al., MSR '26）。
- **経験豊富な開発者が急速に採用。** 調査対象エンジニアの95%がAIを週次で使用；55%が定期的にエージェントを使用；Claude Codeが46%の支持率で最も使用されるツール（Pragmatic Engineer, 2026）。
- **安全性は未解決の問題。** OpenAgentSafetyは安全性脆弱タスクの51–73%で安全でない行動を発見。業界は「制限された自律性」に収束中。
- **コストが新たなボトルネック。** エージェント型タスクは高い分散で数百万トークンを消費し（実行間で最大10倍）、価格は従量制ハイブリッドモデルに移行。

---

## 目次

1. [パラダイム分類：オートコンプリート vs. エージェント](#1-パラダイム分類オートコンプリート-vs-エージェント)
2. [ベンチマーク環境と複雑性の崖](#2-ベンチマーク環境と複雑性の崖)
3. [生産性のエビデンス](#3-生産性のエビデンス)
4. [コード品質とPR受け入れ](#4-コード品質とpr受け入れ)
5. [コスト、レイテンシ、トークンエコノミクス](#5-コストレイテンシトークンエコノミクス)
6. [リスク、安全性、障害モード](#6-リスク安全性障害モード)
7. [ガバナンス：ヒューマン・イン・ザ・ループから制限された自律性へ](#7-ガバナンスヒューマンインザループから制限された自律性へ)
8. [軌跡と展望](#8-軌跡と展望)
9. [結論](#9-結論)
10. [参考文献](#10-参考文献)

---

## 1. パラダイム分類：オートコンプリート vs. エージェント

### 1.1 AIコーディングツールの3つのレイヤー

現在のランドスケープは3つの異なる補完的レイヤーとして理解される（<a href="https://www.faros.ai/blog/best-ai-coding-agents-2026" target="_blank">Faros AI, 2026</a>；<a href="https://redmonk.com/kholterhoff/2025/12/22/10-things-developers-want-from-their-agentic-ides-in-2025/" target="_blank">RedMonk, 2025</a>）：

| レイヤー | 例 | 範囲 | インタラクションモデル |
|---|---|---|---|
| **オートコンプリート** | Copilot（インライン）、Codeium、TabNine | 単一ファイル、行レベル | 受動的：提案→受入/拒否 |
| **エージェント型IDE** | Cursor Agent、Copilot Agent Mode、Windsurf | マルチファイル、タスクレベル | 対話的：計画→編集→テスト→反復 |
| **自律エージェント** | Claude Code、Devin、OpenAI Codex | リポジトリレベル、ワークフローレベル | 委任型：割当→自律実行→レビュー |

主要なアーキテクチャ上の差別化要因は**ツール使用ループ**である。オートコンプリートツールは現在のファイルコンテキストに条件付けられたテキスト補完を生成する。エージェント型ツールは計画-実行-観察サイクルで動作する：任意のファイルを読み、複数の場所に書き込み、シェルコマンドを実行し、テストを走らせ、エラー出力を解釈し、タスクが完了するかリソース制限に達するまで反復する。

### 1.2 エージェント型ツールのアーキテクチャ解剖

<a href="https://openai.com/index/introducing-codex/" target="_blank">OpenAI Codex</a>は自律エージェントアーキテクチャを例示する：各タスクはリポジトリがプリロードされた専用クラウドサンドボックスで実行され、ファイルシステムアクセス、ターミナル実行、git操作を備えている。

<a href="https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf?hsLang=en" target="_blank">Anthropicの2026年エージェント型コーディングトレンドレポート</a>は行動の変化を記録：2026年Q1のClaude Codeセッションの**78%がマルチファイル編集**を含み（2025年Q1の34%から上昇）、平均セッション長は4分から23分に延長、セッションあたりのツール呼び出しは平均**47回**。

### 1.3 収束仮説

かつて明確だった分類は曖昧になりつつある。GitHub Copilotはオートコンプリートからエージェントモードに移行した。Cursorは同一IDEでオートコンプリート（Tab）とエージェント（Agent/Composer）の両モードを備える。区別はますます*どのツール*かではなく*どのモード*かの問題となり、実証的な問いは同じツール内でエージェントモードがオートコンプリートモードと比較してネットポジティブな結果を提供するかどうかである。

---

## 2. ベンチマーク環境と複雑性の崖

### 2.1 ベンチマーク階層

この領域で最も注目すべき実証的知見は**複雑性の崖**である：タスクが単独バグ修正から現実的なマルチファイル開発作業に移行するにつれ、エージェントのパフォーマンスが急落する。

| ベンチマーク | 年 | タスク数 | 平均修正ファイル数 | トップパフォーマンス | 範囲 |
|---|---|---|---|---|---|
| <a href="https://www.swebench.com/" target="_blank">SWE-bench Verified</a> | 2024 | 500 | 1.7 | ~81%（Claude Opus 4.5） | 単独GitHubイシュー解決 |
| <a href="https://labs.scale.com/leaderboard/swe_bench_pro_public" target="_blank">SWE-Bench Pro</a> | 2025 | 1,865 | マルチファイル | ~23%（GPT-5 / Opus 4.1） | 複雑なマルチファイルタスク |
| <a href="https://arxiv.org/html/2512.18470v1" target="_blank">SWE-EVO</a> | 2025 | 48 | 20.9 | ~21%（GPT-5） | リリース間のソフトウェア進化 |
| <a href="https://arxiv.org/html/2602.10975v1" target="_blank">FeatureBench</a> | 2025 | 200 | 15.7 | 12.5%（GPT-5.1 Codex） | 複雑な機能開発 |
| <a href="https://dpaia.dev/" target="_blank">DPAI Arena</a> | 2025 | 140+ | 可変 | 可変 | エンタープライズJava/Springライフサイクルタスク |

### 2.2 FeatureBench：機能開発のギャップ（Zhou et al., 2025）

<a href="https://arxiv.org/html/2602.10975v1" target="_blank">Zhou et al. (2025)</a>は24のPythonリポジトリから200タスクのFeatureBenchを導入した。結果は**63パーセントポイントのギャップ**を露呈：

- **Claude Opus 4.5 + Claude Code**：11.0%解決（SWE-benchでは74.4%）
- **GPT-5.1-Codex**：12.5%解決
- 全モデルが**100万以上の入力トークン**を消費し最小限の解決

タスクの複雑性は劇的に高い：790のソリューション行（SWE-benchでは33）、15.7の修正ファイル（vs. 1.7）、62.7の失敗→成功テストポイント（vs. 9.1）。

### 2.3 SWE-EVO：長期的ソフトウェア進化（Thai et al., 2025）

<a href="https://arxiv.org/html/2512.18470v1" target="_blank">Thai et al. (2025)</a>は7つの成熟したPythonプロジェクトのリリースノートから48タスクを構築。GPT-5はSWE-EVOで**21%**のみを達成（SWE-bench Verifiedでは65%）— 44パーセントポイントの低下。

### 2.4 Terminal-BenchとDPAI Arena

**Terminal-Bench**（Stanford/Laude Institute, 2025）は実際のサンドボックス化されたコマンドライン環境でエージェントを評価。Terminal-Bench 2.0でOpus 4.5は~59%、GPT-5.1 Codex Maxは~58.1%。

<a href="https://blog.jetbrains.com/blog/2025/10/28/introducing-developer-productivity-ai-arena-an-open-platform-for-ai-coding-agents-benchmarks/" target="_blank">DPAI Arena</a>（JetBrains, 2025年10月）はLinux Foundation管理の初のベンダー中立ベンチマークプラットフォームで、140以上のエンタープライズグレードのJava/Springタスクを含む。

---

## 3. 生産性のエビデンス

### 3.1 METR RCT：経験豊富な開発者がAIで遅くなる（Becker et al., 2025）

<a href="https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/" target="_blank">METRの2025年RCT</a>は、エージェント型ツールの影響に関する最高品質の因果的エビデンスである。16名の経験豊富な開発者が246件の実際のイシューをAIツール（主に**Cursor ProとClaude 3.5/3.7 Sonnet** — エージェント型構成）使用/不使用で実施。結果：AIで**19%遅延**（CI: +2%〜+39%）。

### 3.2 Cursorの速度-品質トレードオフ（He et al., MSR '26）

<a href="https://arxiv.org/abs/2511.04427" target="_blank">He, Miller, Agarwal, Kästner, Vasilescu (MSR '26)</a>は**806のCursor導入リポジトリ**を対象にした差分の差分設計による初の因果研究を実施。

**主要な知見：**
- **速度**：統計的に有意で大きいが**一時的な**開発速度の増加
- **品質**：静的解析警告とコード複雑度の**持続的な**増加
- 速度向上は蓄積された複雑性が開発を減速させるにつれて消失
- 品質保証が早期Cursor採用者にとって「重要なボトルネック」

### 3.3 Pragmatic Engineerサーベイ（2026年3月）

<a href="https://newsletter.pragmaticengineer.com/p/ai-tooling-2026" target="_blank">Pragmatic Engineer</a>が**906名のソフトウェアエンジニア**を調査（経験中央値：11〜15年）：

- **95%**がAIツールを少なくとも週次で使用
- **55%**が定期的にエージェントを使用（スタッフ+エンジニア：63.5%）
- **56%**がエンジニアリング作業の70%以上をAIで実施と報告
- **Claude Code**が最も使用されるツール（46%の支持率）、Cursor（19%）、GitHub Copilot（9%）

### 3.4 Anthropicのエージェント型コーディングトレンドレポート（2026）

<a href="https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf?hsLang=en" target="_blank">Anthropic (2026)</a>は、開発者が作業の~60%でAIを使用するが完全に委任できるのは0–20%のタスクのみと報告。AI支援作業の約**27%**は以前は実施されなかったタスク。

ケーススタディ：
- **Rakuten**：Claude Codeが1250万行コードベースで99.9%の数値精度で7時間で完了
- **TELUS**：コード出荷30%高速化、50万時間以上を節約
- **Zapier**：89%のAI採用率、800以上の内部エージェント展開

### 3.5 Devinのパフォーマンス進化（Cognition, 2025）

<a href="https://cognition.ai/blog/devin-annual-performance-review-2025" target="_blank">Cognitionの2025年レビュー</a>は、DevinのPRマージ率が34%から**67%**に倍増、問題解決速度4倍向上、リソース消費半減を報告。脆弱性修復：人間の30分に対しDevinは1.5分（20倍）。

---

## 4. コード品質とPR受け入れ

### 4.1 AIDevデータセット：93万件のエージェント型プルリクエスト

<a href="https://arxiv.org/html/2602.09185" target="_blank">AIDevデータセット</a>は5つのエージェントから116,211リポジトリ、72,189開発者にわたる**932,791件のエージェント型プルリクエスト**を収集。

### 4.2 エージェント別PR受け入れ率（Pinna et al., MSR '26）

<a href="https://arxiv.org/html/2602.08915v1" target="_blank">Pinna, Gong, Williams, Sarro (MSR '26)</a>がAIDevデータセットから7,156件のPRを分析：

| エージェント | 受け入れ率 | 分析PR数 |
|---|---|---|
| OpenAI Codex | 77.9% | 2,002 |
| Cursor | 74.5% | 569 |
| Claude Code | 71.9% | 139 |
| GitHub Copilot | 68.0% | 2,194 |
| Devin | 61.6% | 2,252 |

**タスクタイプが支配的要因**：タスクカテゴリ間の29%の受け入れギャップがエージェント間の差異を上回る。Claude Codeはドキュメント（92.3%）と機能（72.6%）でリード；Cursorはバグ修正（80.4%）に秀でる。

### 4.3 エージェント型PRが拒否される理由（Nakashima et al., MSR '26）

<a href="https://arxiv.org/html/2602.04226v1" target="_blank">Nakashima et al. (MSR '26)</a>は654件の拒否PRを分析し、**7つのエージェント専用拒否モード**を特定：AI生成コードへの不信、過大なPR、実験目的の提出を含む。重要な発見：**拒否PRの67.9%がレビュアーフィードバックなし**。

### 4.4 エージェントのリファクタリング方法（Ottenhof et al., MSR '26）

<a href="https://arxiv.org/html/2601.20160" target="_blank">Ottenhof, Penner, Hindle, Lutellier (MSR '26)</a>が5つのエージェントと人間開発者のリファクタリングパターンを比較：

| エージェント | 平均リファクタリング/コミット | 主要パターン | コードスメル変化 |
|---|---|---|---|
| Claude Code | 762.73 | アノテーション（91%） | +35.5（有意でない） |
| Copilot | 100.74 | 混合 | +2.22（有意でない） |
| Cursor Agent | — | 混合 | **有意な増加**（p=0.013） |
| Codex | — | 混合 | −1.2（有意でない） |
| 開発者 | 15.34 | 構造的（Extract Method, Move Class） | +2.43 |

重要な知見：**エージェントと人間は根本的に異なるリファクタリングを行う**。Claude Codeの762リファクタリング/コミットは印象的に見えるが、91%がアノテーション追加であり、技術的負債を削減する構造的改善ではない。

---

## 5. コスト、レイテンシ、トークンエコノミクス

### 5.1 トークン消費パターン

<a href="https://openreview.net/forum?id=1bUeVB3fov" target="_blank">エージェント型タスクのトークン消費に関する研究</a>は複数の経済的課題を明らかにする：

- **入力トークンが支配的**：キャッシュ使用時でも入力トークンがコストの大部分を占める
- **高い分散**：複雑なタスクはシンプルな実行の**最大10倍**のトークンを消費
- **二次的成長**：自己修正ループでの多ターン会話は単一線形パスの50倍のトークンを消費しうる

### 5.2 価格モデルの変化

業界は定額サブスクリプションからハイブリッドモデルに移行（<a href="https://smcleod.net/2025/04/the-cost-of-agentic-coding/" target="_blank">McLeod, 2025</a>）：

- パワーユーザーはベースサブスクリプションの**2〜5倍の月額コスト**を報告
- 制約のないエージェントが単一SWE-benchイシューを解決するのに**$5–8/タスク**
- <a href="https://galileo.ai/blog/hidden-cost-of-agentic-ai" target="_blank">Gartnerは2027年までにエージェント型AIプロジェクトの40%以上が本番前にキャンセルされると予測</a>

---

## 6. リスク、安全性、障害モード

### 6.1 OpenAgentSafety（2025）

<a href="https://arxiv.org/abs/2507.06134" target="_blank">OpenAgentSafety</a>は8つのリスクカテゴリにわたる350以上のマルチターンタスクでエージェントの安全性を評価。安全性脆弱タスクにおいて**51.2%（Claude Sonnet 3.7）から72.7%（o3-mini）の安全でない行動**を検出。

### 6.2 エージェント特有の障害モード

1. **暴走編集**：単一セッションで数十ファイルを修正するエージェントがコードベース全体に相関エラーを導入しうる
2. **コンテキストウィンドウ枯渇**：フルコンテキストを消費したエージェントが一貫性を失い、ランダムな編集を行う
3. **エスカレーション連鎖**：自分が導入したエラーを修正しようとする際に修正ループが損傷を増幅
4. **不可視の回帰**：マルチファイル変更がエージェントが検査しなかったファイルの機能を破壊しうる
5. **デバッグの不透明性**：15ファイルにわたる500行のdiffの各変更の推論を追跡することが困難

### 6.3 セキュリティリスク

<a href="https://www.darkreading.com/application-security/coders-adopt-ai-agents-security-pitfalls-lurk-2026" target="_blank">セキュリティ研究</a>によると、AI生成コードの40–62%にセキュリティ脆弱性が含まれ、エージェント展開を計画する企業の83%が従来のセキュリティツールが自律的コード実行向けに設計されていないことを発見。

---

## 7. ガバナンス：ヒューマン・イン・ザ・ループから制限された自律性へ

### 7.1 制限された自律性パターン

2026年の主要なガバナンスパターンは**制限された自律性**（<a href="https://www.bunnyshell.com/guides/agentic-development/" target="_blank">Bunnyshell, 2026</a>；<a href="https://www.weforum.org/stories/2026/03/ai-agent-autonomy-governance/" target="_blank">世界経済フォーラム, 2026</a>）：エージェントが事前定義された制限内で動作し、高リスクの決定には人間へのエスカレーションパスが必須であり、包括的な監査証跡を持つ。

設計原則：
- **権限ティア**：低リスク操作（ファイル読み取り、テスト実行）は自律；高リスク操作（ファイル削除、本番へのプッシュ）は承認が必要
- **リソース制限**：トークン予算、時間制限、ファイル変更上限で暴走セッションを防止
- **監査証跡**：全てのツール呼び出し、ファイル読み取り、コマンド実行をログ記録

### 7.2 開発者の監視に関する実証的エビデンス

<a href="https://arxiv.org/html/2602.02345" target="_blank">Rahman et al. (MSR '26)</a>は、**エージェント型PRの90.6%がレビューコメントをゼロ件受け取った**ことを発見し、人間の監視が実際に行われているか、多くのエージェント型PRがラバースタンプされているかという疑問を提起。

---

## 8. 軌跡と展望

### 8.1 マルチエージェントの未来

Anthropicのレポートはマルチエージェントアーキテクチャの出現を記録：**Planner**がコードベースを探索しタスクを作成、**Worker**が割り当てられたタスクを独立して実行、**Judge**が各サイクル終了時に継続するかどうかを判断。

### 8.2 IDEからオーケストレーションへ

軌跡は明確：
- **2023年**：開発者はより良いオートコンプリートを求めた
- **2024年**：開発者はマルチファイル編集を求めた
- **2025年**：開発者はワークフロー全体をエージェントに委任
- **2026年**：組織がプロジェクト横断で複数エージェントを調整

### 8.3 収束の道

GitHub Copilotの進化 — オートコンプリート（2021）→エージェントモード（2025）→自律コーディングエージェント（2025–2026）— は収束パスを示す。オートコンプリートで始まったツールがエージェント機能を追加し、自律エージェントで始まったツールがIDE統合を追加している。

しかし、実証的エビデンスは収束が根本的な制限により制約されることを示唆：
- **コンテキストウィンドウの制限**が非常に大きなコードベースの全体的推論を妨げる
- **品質劣化**がエージェントの自律性に比例して拡大
- **トークンエコノミクス**が長時間のエージェントセッションを高額にする
- **人間の監視**が依然として必要だが、コード量がレビュー能力を超えると希薄化しうる

---

## 9. 結論

### エビデンスが支持すること

1. **エージェント型ツールは漸進的改善ではなく真のパラダイムシフトを表す。** ファイル横断での計画、実行、テスト、反復の能力は質的に異なるワークフローを可能にする — が、実証的な利益はまだ一義的ではない。

2. **複雑性の崖が中心的課題。** エージェントは単独バグ修正をほぼ解決（SWE-benchで~80%）したが、機能開発（~11%）、ソフトウェア進化（~21%）、長期タスクでは崩壊する。

3. **速度と品質は緊張関係にある。** Cursorの差分の差分研究（MSR '26）が最も強い因果的エビデンスを提供：エージェント型ツールは一時的な速度向上をもたらすが、持続的な品質劣化を伴う。

4. **エージェント選択はタスク固有であるべき。** 単一のエージェントが全カテゴリで支配的ではない。Claude Codeはドキュメントと機能でリード；Cursorはバグ修正に秀でる；Codexは最高の全体的受け入れ率。

5. **エージェントは人間と異なるリファクタリングを行う。** 高いリファクタリング量をアーキテクチャの改善と同一視すべきではない。

6. **安全性とガバナンスは前提条件であり後付けではない。** 51–73%の安全でない行動率とPRの90.6%がレビューコメントゼロという状況は体系的リスクである。

7. **コストが新たな制約。** トークン消費がタスクの複雑性に対して超線形に増大し、最も高性能なエージェント構成は高額である。

### 未解決の課題

- **複雑性の崖は克服可能か** — より良いアーキテクチャ（階層的計画、永続的メモリ）を通じて、あるいはエージェントが維持できる推論に根本的な限界があるのか？
- **どのような組織的プラクティス**が品質-速度トレードオフを緩和するか？
- **価格はどう進化するか** — 競争の激化と推論コストの低下に伴い？
- **マルチエージェント調整はその約束を果たすか** — あるいは調整オーバーヘッドが生産性向上を消費するか？

エージェント型コーディングパラダイムは実在し急速に成熟している。しかし、エビデンスは一貫して、自律性の増加がリスクの増加、コストの増加、ガバナンスの必要性の増加を伴うことを示す。最も恩恵を受ける組織は、エージェントの自律性を最大化する組織ではなく、品質要件、セキュリティ制約、人間のレビュー能力に正確に較正する組織である。

---

## 10. 参考文献

1. <a href="https://arxiv.org/abs/2511.04427" target="_blank">He, H. et al. (2026). "Speed at the Cost of Quality: How Cursor AI Increases Short-Term Velocity and Long-Term Complexity." MSR '26.</a>
2. <a href="https://arxiv.org/html/2602.10975v1" target="_blank">Zhou, Q. et al. (2025). "FeatureBench: Benchmarking Agentic Coding for Complex Feature Development."</a>
3. <a href="https://arxiv.org/html/2512.18470v1" target="_blank">Thai, M.V.T. et al. (2025). "SWE-EVO: Benchmarking Coding Agents in Long-Horizon Software Evolution."</a>
4. <a href="https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/" target="_blank">Becker, J. et al. (2025). "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity." METR.</a>
5. <a href="https://arxiv.org/html/2602.08915v1" target="_blank">Pinna, G. et al. (2026). "Comparing AI Coding Agents: A Task-Stratified Analysis of Pull Request Acceptance." MSR '26.</a>
6. <a href="https://arxiv.org/html/2602.04226v1" target="_blank">Nakashima, S. et al. (2026). "Why Agentic-PRs Get Rejected." MSR '26.</a>
7. <a href="https://arxiv.org/html/2601.20160" target="_blank">Ottenhof, L. et al. (2026). "How do Agents Refactor: An Empirical Study." MSR '26.</a>
8. <a href="https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf?hsLang=en" target="_blank">Anthropic (2026). "2026 Agentic Coding Trends Report."</a>
9. <a href="https://newsletter.pragmaticengineer.com/p/ai-tooling-2026" target="_blank">Orosz, G. (2026). "AI Tooling for Software Engineers in 2026." The Pragmatic Engineer.</a>
10. <a href="https://cognition.ai/blog/devin-annual-performance-review-2025" target="_blank">Cognition (2025). "Devin's 2025 Performance Review."</a>
11. <a href="https://openai.com/index/introducing-codex/" target="_blank">OpenAI (2025). "Introducing Codex."</a>
12. <a href="https://openai.com/index/introducing-gpt-5-3-codex/" target="_blank">OpenAI (2026). "Introducing GPT-5.3-Codex."</a>
13. <a href="https://code.visualstudio.com/blogs/2025/02/24/introducing-copilot-agent-mode" target="_blank">GitHub (2025). "Introducing GitHub Copilot Agent Mode."</a>
14. <a href="https://www.swebench.com/" target="_blank">SWE-bench Leaderboards.</a>
15. <a href="https://labs.scale.com/leaderboard/swe_bench_pro_public" target="_blank">Scale Labs. "SWE-Bench Pro Leaderboard."</a>
16. <a href="https://arxiv.org/abs/2506.17208" target="_blank">"Dissecting the SWE-Bench Leaderboards." (2025)</a>
17. <a href="https://blog.jetbrains.com/blog/2025/10/28/introducing-developer-productivity-ai-arena-an-open-platform-for-ai-coding-agents-benchmarks/" target="_blank">JetBrains (2025). "Introducing DPAI Arena."</a>
18. <a href="https://dpaia.dev/" target="_blank">DPAI Arena.</a>
19. <a href="https://arxiv.org/abs/2507.06134" target="_blank">OpenAgentSafety (2025).</a>
20. <a href="https://arxiv.org/html/2602.02345" target="_blank">Rahman, S. et al. (2026). "A Task-Level Evaluation of AI Agents in Open-Source Projects." MSR '26.</a>
21. <a href="https://arxiv.org/html/2602.09185" target="_blank">AIDev Dataset (2026).</a>
22. <a href="https://www.faros.ai/blog/best-ai-coding-agents-2026" target="_blank">Faros AI (2026). "Best AI Coding Agents for 2026."</a>
23. <a href="https://redmonk.com/kholterhoff/2025/12/22/10-things-developers-want-from-their-agentic-ides-in-2025/" target="_blank">RedMonk (2025). "10 Things Developers Want from their Agentic IDEs."</a>
24. <a href="https://www.gitclear.com/ai_assistant_code_quality_2025_research" target="_blank">GitClear (2025). "AI Copilot Code Quality: 2025 Data."</a>
25. <a href="https://smcleod.net/2025/04/the-cost-of-agentic-coding/" target="_blank">McLeod, S. (2025). "The Cost of Agentic Coding."</a>
26. <a href="https://openreview.net/forum?id=1bUeVB3fov" target="_blank">"How Do Coding Agents Spend Your Money?"</a>
27. <a href="https://www.sitepoint.com/ai-coding-tools-cost-analysis-roi-calculator-2026/" target="_blank">SitePoint (2026). "AI Coding Tools ROI Calculator."</a>
28. <a href="https://galileo.ai/blog/hidden-cost-of-agentic-ai" target="_blank">Galileo (2026). "The Hidden Costs of Agentic AI."</a>
29. <a href="https://www.darkreading.com/application-security/coders-adopt-ai-agents-security-pitfalls-lurk-2026" target="_blank">Dark Reading (2026). "Security Pitfalls in AI Agents."</a>
30. <a href="https://www.bunnyshell.com/guides/agentic-development/" target="_blank">Bunnyshell (2026). "Agentic Development."</a>
31. <a href="https://www.weforum.org/stories/2026/03/ai-agent-autonomy-governance/" target="_blank">World Economic Forum (2026). "Governance Is Key for AI Agents."</a>
32. <a href="https://claude.com/blog/eight-trends-defining-how-software-gets-built-in-2026" target="_blank">Anthropic (2026). "Eight Trends Defining How Software Gets Built."</a>
33. <a href="https://arxiv.org/html/2601.07136v1" target="_blank">"A Large-Scale Study on Multi-Agent AI Systems." (2026)</a>
34. <a href="https://2026.msrconf.org/details/msr-2026-technical-papers/44/Are-We-All-Using-Agents-Now-An-Empirical-Study-of-Core-and-Peripheral-Developers-Us" target="_blank">"Are We All Using Agents Now?" MSR '26.</a>
35. <a href="https://arxiv.org/html/2601.16839" target="_blank">"AI Builds, We Analyze." (2026)</a>
36. <a href="https://arxiv.org/html/2511.00872v1" target="_blank">"Empirical Evaluation of Agent Frameworks." (2025)</a>
37. <a href="https://dora.dev/research/2024/dora-report/" target="_blank">Google (2024). "DORA Report 2024."</a>
38. <a href="https://www.sonarsource.com/blog/state-of-code-developer-survey-report-the-current-reality-of-ai-coding/" target="_blank">SonarSource (2026). "State of Code 2026."</a>
