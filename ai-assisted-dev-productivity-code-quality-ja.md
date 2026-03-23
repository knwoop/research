# AI支援ソフトウェア開発：生産性、コード品質、開発者の認知

## 実証研究の包括的サーベイ（2023年–2026年）

---

## エグゼクティブサマリー

GitHub Copilotを筆頭とするAIコーディングアシスタント、さらにはCursorやClaude Codeなどのエージェント型ツールの急速な普及は、前例のない規模の実証研究を促している。本レポートは、2023年から2026年にかけて発表された対照実験、大規模観察分析、系統的文献レビュー、業界調査の知見を統合する。対象とする学術会議・ジャーナルは、ICSE、FSE、NeurIPS、IEEE TSE、TOSEM、CHI、CACM、DORAを含む。

**主要な知見：**

- **生産性向上は実在するが不均一。** Microsoft、Accenture、Googleでの3件の大規模RCTは、Copilot利用者のタスク処理量が21〜26%増加したことを示し、経験の浅い開発者が最大の恩恵（35〜39%の高速化）を受けた。一方、METR 2025年のRCTでは、経験豊富なオープンソース開発者がAIツール使用時に19%*遅く*なった — 開発者の自己評価と専門家の予測の双方に反する結果であった。
- **コード品質は諸刃の剣。** GitHubの202名規模のRCTでは、可読性（+3.6%）、信頼性（+2.9%）、保守性（+2.5%）に小さいが統計的に有意な改善が報告された。逆に、GitClearの2億1100万行のコード分析では、コード重複が48%急増、コードチャーンが劇的に増加し、リファクタリングが変更行の25%から10%未満に低下した（2021年から2024年）。
- **セキュリティ脆弱性が蔓延。** Copilot生成コード733スニペットの実証研究では、27.3%にセキュリティ上の弱点が含まれ、43のCWEカテゴリにまたがり、うち8つが2023年CWE Top-25に該当した。
- **技術的負債が蓄積。** 経験豊富な開発者は、Copilot導入後にオリジナルコード生産性が19%低下し、レビュー作業量が6.5%増加した。AI生成コードがリポジトリ基準を満たすためにより多くの修正作業を必要とするためである。
- **スキル形成が阻害される。** Anthropicの2026年RCTでは、AI支援を受けた学習者の理解度評価が17%低くなり、デバッグスキルで最大の差が生じた。コード生成を委任した開発者のスコアは40%未満だったのに対し、概念的な問いかけにAIを活用した開発者は65%以上を記録した。
- **信頼の較正が不適切。** Stack Overflowの2025年調査では、84%の開発者がAIツールを使用しているが、信頼しているのはわずか29%（2024年の43%から低下）。一方、59%の開発者が完全に理解していないAI生成コードを使用していることを認めている。

---

## 目次

1. [生産性への影響：対照実験からのエビデンス](#1-生産性への影響対照実験からのエビデンス)
2. [コード品質と欠陥率](#2-コード品質と欠陥率)
3. [AI生成コードのセキュリティ脆弱性](#3-ai生成コードのセキュリティ脆弱性)
4. [保守性と技術的負債](#4-保守性と技術的負債)
5. [開発者の認知、理解、スキル形成](#5-開発者の認知理解スキル形成)
6. [調整要因](#6-調整要因)
7. [ベンチマークと評価のギャップ](#7-ベンチマークと評価のギャップ)
8. [方法論的景観と限界](#8-方法論的景観と限界)
9. [結論と未解決の課題](#9-結論と未解決の課題)
10. [参考文献](#10-参考文献)

---

## 1. 生産性への影響：対照実験からのエビデンス

### 1.1 基礎的なCopilot RCT（Peng et al., 2023）

最初の厳密なエビデンスは、<a href="https://arxiv.org/abs/2302.06590" target="_blank">Peng et al. (2023)</a>の対照実験である。プロのプログラマーにJavaScriptでHTTPサーバーを実装させたところ、GitHub Copilotを使用したグループは対照群より**55.8%速く**タスクを完了した。異質的効果が確認され、経験の浅い開発者、作業負荷の高い開発者、年齢の高い開発者ほど恩恵が大きく、AIアシスタントがドメイン知識やワーキングメモリの不足を部分的に補完することが示唆された。

### 1.2 3社フィールド実験（Cui et al., 2024）

最も包括的な産業界のエビデンスは、<a href="https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4945566" target="_blank">Cui, Demirer, Jaffe, Musolff, Peng, Salz (2024)</a>による*Management Science*掲載論文である。Microsoft（1,663名）、Accenture、匿名のFortune 100電子機器メーカーの3社で計**4,867名**の開発者を対象にRCTを実施した。主な結果：

- 処置群の週次プルリクエスト完了数が平均**26.08%増加**（SE: 10.3%）
- 経験の浅い開発者で最大の効果：**35〜39%の高速化**（シニア開発者は**8〜16%**）
- 使用されたツールはGPT-3.5ベースであり、現在のモデルでは効果量が異なる可能性がある

### 1.3 Google企業内RCT（Paradis et al., 2024）

<a href="https://arxiv.org/html/2410.12944v2" target="_blank">Paradis et al. (2024)</a>は、**96名のGoogleフルタイムエンジニア**を対象に、標準化されたコード修正タスク（10ファイル、474行の更新）でRCTを実施した。処置群は約96分、対照群は約114分でタスクを完了 — 約**21%高速化**。ただし、開発者やタスクの特性を調整すると主効果の統計的有意性が失われた（p=0.086）点は、共変量の重要性を浮き彫りにしている。

### 1.4 METRの衝撃：AIは経験豊富な開発者を遅くする（Becker et al., 2025）

最も反直感的な知見は、<a href="https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/" target="_blank">METRの2025年RCT</a>からもたらされた。16名の経験豊富なオープンソース開発者（各プロジェクトで平均5年の経験、リポジトリは平均22,000+スター、100万行以上のコード）が、246件の実際のイシューをAI使用/不使用にランダムに割り当てて実施した（主にCursor ProとClaude 3.5/3.7 Sonnetを使用）。

**結果：** AI使用によりタスク完了時間が**19%増加**（CI: +2%〜+39%）。開発者は24%の高速化を予測していたが、遅延を経験した後でさえ、20%速くなったと信じ続けていた。

5つの寄与因子が特定された：
1. ツールのコンテキスト切り替えによる摩擦
2. ハルシネーションの検証コスト
3. 不完全な提案への過度の依存
4. 不慣れなAIインターフェースの学習曲線
5. AI出力を超えるコード品質・ドキュメント基準

重要な点として、両条件でのPR品質は同程度であり、品質低下は説明要因から除外された。

### 1.5 生産性のパラドックス（Xu et al., 2025）

<a href="https://arxiv.org/abs/2510.10165" target="_blank">Xu, Medappa, Tunc, Vroegindeweij, Fransoo (2025)</a>は、Copilot導入前後のオープンソースプロジェクトにおける開発者の活動を分析した。全体的な生産性指標は上昇したものの、その効果は経験の浅い開発者に集中し、コア（経験豊富な）開発者のオリジナルコード生産性の**19%低下**とコードレビュー作業量の**6.5%増加**が隠蔽されていた。「短期的な生産性向上と長期的なシステム持続可能性」の根本的な対立を指摘している。

### 1.6 系統的レビューのエビデンス（Mohamed et al., 2025）

<a href="https://arxiv.org/abs/2507.03156" target="_blank">Mohamed, Assi, Guizani (2025)</a>は37本の査読済み研究（2014年–2024年）をレビューし、「コード検索の最小化、開発の加速、定型タスクの自動化」における一貫した効果を確認した。一方で、認知的オフローディング、チーム協調の減少、コード品質効果の不一致に関する懸念も記録された。研究の92%がSPACEフレームワークの少なくとも2次元を測定したが、3次元以上を対象としたのはわずか14%、64%が探索的手法を用いていた。著者らは「縦断的かつチームベースの評価」の必要性を訴えた。

### 1.7 CACMの視点：Copilotの影響測定（Ziegler et al., 2024）

<a href="https://cacm.acm.org/research/measuring-github-copilots-impact-on-productivity/" target="_blank">Ziegler et al. (2024)</a>は*Communications of the ACM*に掲載された論文で、2,631件の調査回答をIDEテレメトリとマッチングして分析した。提案受入率（21.2〜23.5%）が認知された生産性の最も強い予測因子であった。73%の開発者がフロー状態の維持が容易になったと報告し、87%が反復作業での精神的負荷の軽減を報告した。

---

## 2. コード品質と欠陥率

### 2.1 GitHubの202名規模コード品質RCT（GitHub Research, 2025）

<a href="https://github.blog/news-insights/research/does-github-copilot-improve-code-quality-heres-what-the-data-says/" target="_blank">GitHub Research (2025年2月)</a>は、5年以上の経験を持つ202名の開発者でRCTを実施した。APIエンドポイントを作成させ、25名の専門家がブラインドレビューした。

| 指標 | 改善率 | p値 |
|---|---|---|
| 可読性 | +3.62% | 0.003 |
| 信頼性 | +2.94% | 0.01 |
| 保守性 | +2.47% | 0.041 |
| 簡潔性 | +4.16% | 0.002 |
| ユニットテスト全合格 | +53.2%の可能性 | <0.01 |
| レビュアー承認 | +5% | 0.014 |

研究者らは、Copilotが認知リソースを解放し、基本的な機能実現よりも品質改善に注力できるようになったと仮説を立てた。ただし、本研究がGitHub自身により実施された点と、標準化されたタスク（APIエンドポイント作成）が複雑な開発シナリオに一般化されない可能性は批判の対象となっている。

### 2.2 ClassEval：クラスレベルのコード生成（Du et al., ICSE 2024）

<a href="https://dl.acm.org/doi/10.1145/3597503.3639219" target="_blank">Du et al. (ICSE 2024)</a>は、初のクラスレベルコード生成ベンチマーク**ClassEval**を提案した：100クラス、410メソッド、クラスあたり平均33.1テストケース。テストされた全てのLLMは、メソッドレベルのタスクと比較してクラスレベルの生成で著しく低いパフォーマンスを示した。GPT-4は第3位のモデル（WizardCoder）をクラスレベルPass@1で25.4%上回ったが、絶対的なパフォーマンスは依然として控えめであった。

### 2.3 コード生成エラーの分類法（Wang et al., ICSE 2025）

<a href="https://arxiv.org/abs/2406.08731" target="_blank">Wang et al. (ICSE 2025)</a>は、LLMコード生成エラーの初の体系的分類法を構築した。HumanEvalにおける6つのLLMの557件のエラーを分析し、モデルが「様々な場所と根本原因を持つ、非自明な複数行のコード生成エラー」を頻繁に生成することを明らかにした。

### 2.4 DORA 2024：デリバリー安定性のトレードオフ（Google, 2024）

<a href="https://dora.dev/research/2024/dora-report/" target="_blank">Googleの2024年DORAレポート</a>によると、AI導入が25%増加するごとに：

- コードレビュー速度が**3.1%向上**
- コード品質が**3.4%向上**
- しかしデリバリー安定性は**7.2%低下**
- デリバリースループットは**1.5%低下**
- 回答者の39%がAI生成コードに対する信頼が低いまたは全くないと報告

テストとレビュープラクティスの改善を伴わない高速なコード生成が安定性低下の原因と推測されている。

---

## 3. AI生成コードのセキュリティ脆弱性

### 3.1 大規模セキュリティ脆弱性分析（Fu et al., 2023–2024）

<a href="https://dl.acm.org/doi/10.1145/3716848" target="_blank">Fu et al. (TOSEM, 2024)</a>は、これまでで最も包括的なセキュリティ分析を実施した。GitHubプロジェクトからCopilot（672件）、CodeWhisperer（38件）、Codeium（23件）で生成された**733件のコードスニペット**を、CodeQL、Bandit、ESLintで分析した。

**主要な知見：**
- 全スニペットの**27.3%**にセキュリティ上の弱点（200/733）
- Python: 29.5%に影響; JavaScript: 24.2%に影響
- 43のCWEカテゴリにわたる**628件の脆弱性**
- 影響を受けたスニペットあたり平均3件の弱点; 51%が複数の問題を含有
- 上位3つのCWE: CWE-330（不十分なランダム性, 18.15%）、CWE-94（コードインジェクション, 9.87%）、CWE-79（XSS, 9.55%）
- 8つのCWEが2023年CWE Top-25に該当（全検出の37.1%）

**Copilot Chatの修正効果：**
- デフォルト"/fix"コマンド: 19.3%の修正率
- 基本的なプロンプト: 31.8%
- 静的解析警告を含む拡張プロンプト: **55.5%**

### 3.2 脆弱性の再現率

<a href="https://arxiv.org/abs/2204.04741" target="_blank">Asare et al. (2022)</a>のC/C++脆弱性に関する研究では、Copilotが元の脆弱なコードを約**33%の確率**で再現し、修正済みコードの再現率はわずか25%であった。既知の脆弱性パターンを再現する傾向が示唆されている。

---

## 4. 保守性と技術的負債

### 4.1 GitClear: 2億1100万行のエビデンス（2024–2025）

<a href="https://www.gitclear.com/ai_assistant_code_quality_2025_research" target="_blank">GitClearの2025年調査</a>は、2020年から2024年にかけて、Google、Microsoft、Meta、エンタープライズ企業のリポジトリから**2億1100万行の変更コード**を分析した。

| 指標 | 2020/2021年 | 2024年 | 変化 |
|---|---|---|---|
| リファクタリング（移動）行 | 24.1%（2020） | 9.5% | −61% |
| コピー/ペースト（クローン）行 | 8.3%（2020） | 12.3% | +48% |
| 新規追加行 | 39%（2020） | 46% | +18% |
| 2週間以内に修正されたコード | 5.5%（2020） | 7.9% | +44% |
| 変更に占めるリファクタリング比率 | 25%（2021） | <10% | −60% |

2024年には、データセット史上初めてコピー/ペースト行が移動行を上回り、コード複製ブロックは8倍に増加した。これは技術的負債の蓄積を示唆している。

### 4.2 経験豊富な開発者の負担（Xu et al., 2025）

セクション1.5で詳述した通り、<a href="https://arxiv.org/abs/2510.10165" target="_blank">Xu et al. (2025)</a>はCopilot導入後の作業再分配を定量化した。経験の浅い開発者がより多くのコードを生成する一方、そのコードの品質保証の負担は経験豊富な開発者に不均衡に帰着し、彼らのオリジナルコード生産量は19%減少した。

### 4.3 LLM生成コードのコードスメル（Ghosh Paul et al., 2025）

<a href="https://arxiv.org/html/2510.03029v1" target="_blank">Ghosh Paul, Zhu, Bayley (2025)</a>は、4つのLLM（Gemini Pro、ChatGPT、Codex、Falcon）が生成したJavaコードを、1,000タスクのプロフェッショナルな人間コードと比較した。

**主要な知見：**
- LLM生成コードは人間のリファレンスと比較して**42.28%〜84.97%高いコードスメル発生率**（平均: 63.34%）
- 実装スメル: +73.35%
- 設計スメル: +21.42%
- 高度なトピックで最悪のパフォーマンス: カプセル化（+138.53%）、配列処理（+101.88%）、OOP（+101.88%）
- 循環的複雑度とスメル頻度の間に強い正の相関（0.9653）

### 4.4 Sonar 2026年コード状態調査

<a href="https://www.sonarsource.com/blog/state-of-code-developer-survey-report-the-current-reality-of-ai-coding/" target="_blank">Sonarの2026年調査</a>（N=1,149名）によると、AI生成コードは現在コミットされた全コードの**42%**を占め、2027年までに65%に達すると予想されている。93%の開発者がポジティブな効果を報告する一方、**88%がネガティブな影響も指摘**：「正しく見えるが信頼できないコード」（53%）や「不必要で重複したコード」。最も顕著なのは、**96%がAI出力を完全に信頼していないにもかかわらず、コミット前に常に検証するのは48%のみ**という「検証ギャップ」である。

---

## 5. 開発者の認知、理解、スキル形成

### 5.1 AIはスキル形成を阻害する（Shen & Tamkin, Anthropic, 2026）

<a href="https://arxiv.org/html/2601.20245v1" target="_blank">Shen, Tamkin (2026)</a>はAnthropicにおいて、52名の参加者がPythonのTrioライブラリを学習するRCTを実施した。AI支援グループはスキル評価で**17%低いスコア**（Cohen's d = 0.738, p = 0.010）を記録し、生産性の有意な向上は見られなかった。

6つの異なるAIインタラクションパターンが特定された：

| パターン | 人数 | クイズスコア範囲 |
|---|---|---|
| AI委任 | 4 | 24–39% |
| 段階的AI依存 | 4 | 24–39% |
| 反復的AIデバッグ | 4 | 24–39% |
| 生成後理解 | 2 | 65–86% |
| ハイブリッドコード・説明 | 3 | 65–86% |
| 概念的問いかけ | 7 | 65–86% |

重要な区別：**概念的問いかけ**（コードの要求ではなく「なぜ」「どのように」を質問）を行った開発者は65%以上を記録し理解を保持した。一方、**コード生成を委任**した開発者は40%未満であった。

### 5.2 企業における開発者エクスペリエンス（Weisz et al., CHI 2025）

<a href="https://dl.acm.org/doi/10.1145/3706599.3706670" target="_blank">Weisz et al. (CHI EA '25)</a>は、IBMの669名の開発者を対象にwatsonx Code Assistantの利用を調査した。

- 開発者はコード生成（55.6%）よりも**コード理解**（71.9%）を優先
- 生成出力をそのまま受け入れたのは**わずか2〜4%**; 23〜37%が「学習・インスピレーション」のために使用
- 42.6%がツールの使用で**効率が低下**したと感じた
- AIを「監督が必要なインターン」と比喩する声
- **デスキリング（スキル喪失）への不安**が明示的に報告された

### 5.3 信頼の較正危機

複数の情報源が、拡大する信頼のパラドックスを記録している：

- **Stack Overflow 2025年調査**: 84%の開発者がAIツールを使用するが、**信頼しているのはわずか29%**（2024年の43%から低下）(<a href="https://stackoverflow.blog/2026/02/18/closing-the-developer-ai-trust-gap/" target="_blank">Stack Overflow, 2026</a>)
- **Clutch 2025年調査**（N=800）: **59%の開発者**が完全に理解していないAI生成コードを使用 (<a href="https://clutch.co/resources/devs-use-ai-generated-code-they-dont-understand" target="_blank">Clutch, 2025</a>)
- **DORA 2024**: 39%の回答者がAI生成コードに対する信頼が低いまたは全くないと報告
- **Sonar 2026**: 96%がAI出力を完全に信頼していないが、常に検証するのは48%のみ

<a href="https://arxiv.org/html/2509.13253v1" target="_blank">Amoozadeh et al. (2024)</a>は、OSコースの学生がCS2の学生より低い信頼を示し、全参加者がコード生成機能よりもコード理解機能を信頼していることを発見した。

---

## 6. 調整要因

文献は一貫して、AIの影響を調整するいくつかの要因を特定している。

### 6.1 開発者の経験

研究間で最も堅固な調整要因：
- **Cui et al. (2024)**: 経験の浅い開発者は35〜39%の高速化; シニア開発者はわずか8〜16%
- **METR (2025)**: 経験豊富なオープンソースメンテナはAI使用で19%遅延
- **Xu et al. (2025)**: 経験豊富な開発者がレビューと保守の不均衡な負担を負った
- **Shen & Tamkin (2026)**: AI利用パターン（委任vs.問いかけ）が経験よりもスキル結果を予測

<a href="https://arxiv.org/html/2504.13903v1" target="_blank">2025年の研究</a>は、開発者の経験がAIツール*採用の有無*を予測しないが、*割り当てる役割*を形成することを発見した：経験豊富な開発者はAIを「ジュニアの同僚」や「コンテンツ生成器」として扱い、初心者は「教師」として扱う。

### 6.2 タスクの複雑さ

- **ClassEval (ICSE 2024)**: クラスレベルvs.メソッドレベルでLLMパフォーマンスが急激に低下
- **Ghosh Paul et al. (2025)**: 循環的複雑度に伴いコードスメルが増加し、人間とLLMのギャップが拡大
- **IaC-Eval (NeurIPS 2024)**: GPT-4はインフラコードでpass@1がわずか19.36%（単純なPythonベンチマークでは86.6%）
- **SWE-Bench Pro (2025)**: トップモデルは複雑な実世界タスクで23%（単独バグ修正では70%以上）

### 6.3 コードドメインと言語

- **Fu et al. (2024)**: Python（29.5%）はJavaScript（24.2%）より脆弱性率が高い
- **NeurIPSベンチマーク**: ドメインにより劇的にパフォーマンスが変動 — HumanEval Pythonでほぼ90%、インフラコードで19%、研究コード実装で40%未満

### 6.4 組織的コンテキスト

- **Sonar 2026**: ガバナンスに投資するエンタープライズはより高品質なAI支援コードを生産; SMBは速度を享受するが検証と再作業で苦しむ
- **Weisz et al. (CHI 2025)**: 組織文化（AI使用の推奨vs.汚名）が採用パターンに大きく影響
- **DORA 2024**: 堅牢なテストメカニズムを欠くチームで最大の安定性低下

---

## 7. ベンチマークと評価のギャップ

### 7.1 ベンチマーク飽和の問題

コード生成の基礎的ベンチマークであるHumanEvalは飽和に近づいている。トップモデルはPython問題で90%近くまたはそれ以上を達成する。しかし、より現実的なベンチマークでは異なる様相を呈する：

| ベンチマーク | トップパフォーマンス | 範囲 |
|---|---|---|
| HumanEval (Python) | ~95% | 164の単一関数問題 |
| ClassEval (ICSE 2024) | GPT-4: ~65% | 100クラス、410メソッド |
| IaC-Eval (NeurIPS 2024) | GPT-4: 19.36% | 458のインフラシナリオ |
| SWE-bench Verified | Claude 4.5: 74.4% | 実際のGitHubイシュー（単独） |
| SWE-Bench Pro (2025) | ~23% | 1,865の複雑なマルチファイルタスク |
| FeatureBench (2026) | Claude 4.5: 11.0% | 複雑な機能開発 |
| ResearchCodeBench (NeurIPS 2025) | <40% | 新規ML研究の実装 |

### 7.2 新興の評価フレームワーク

最近のベンチマークは機能的正確性を超えて拡大：
- **SWE-EVO (2025)**: 複数コミットにまたがる長期的ソフトウェア進化シナリオでエージェントを評価
- **FeatureBench (2026)**: アーキテクチャ上の意思決定を必要とする複雑な機能開発をテスト
- **DPAI Arena (JetBrains, 2025)**: コードレビュー、テスト生成、静的解析を含む開発ライフサイクル全体を評価

---

## 8. 方法論的景観と限界

### 8.1 研究デザインの分布

<a href="https://arxiv.org/abs/2507.03156" target="_blank">Mohamed et al. (2025)のSLR</a>は、64%の研究が探索的手法を用い、SPACEフレームワークの3次元以上を検討したのはわずか14%であることを明らかにした。本分野にはいくつかの体系的な限界がある：

1. **短い時間軸**: ほとんどのRCTが数時間〜数日を測定し、技術的負債が顕在化する数週間〜数ヶ月のタイムラインには対応していない
2. **統制タスクvs.実業務**: 多くの研究が標準化タスク（HTTPサーバー、APIエンドポイント）を使用し、複雑なレガシーコードベースへの一般化に疑問が残る
3. **選択効果**: フィールド実験は自発的な採用に依存することが多く、選択バイアスが生じる
4. **代理指標**: プルリクエスト数、コード行数、受入率は生産性の不完全な代理である
5. **利益相反**: 著名なポジティブな研究のいくつかはツールベンダー（GitHub、Microsoft）により実施された
6. **欠落する次元**: SPACEフレームワークのコミュニケーションと活動の次元が大幅に未探索

### 8.2 ホーソン効果と自己報告バイアス

METRの研究は自己報告バイアスを力強く実証した：開発者は客観的測定が19%の遅延を示した場合でもAIが速度を向上させたと信じていた。この知見は、ZieglerらのCACM研究を含む、自己報告された生産性改善に依存する多くの調査ベースの研究に疑問を投げかける。

### 8.3 縦断的研究の必要性

最も重要なギャップは以下を追跡する縦断的研究の欠如である：
- 四半期・年単位での技術的負債の蓄積
- 持続的なAI使用による開発者のスキル軌跡
- チームレベルの協調ダイナミクス
- 製品ライフサイクル全体にわたるコードベースの健全性指標

GitClearの5年間の分析が最も近い近似であるが、因果関係を確立できない観察データを使用している。

---

## 9. 結論と未解決の課題

### エビデンスが支持すること

1. **AIアシスタントは定型的なコーディングタスクを加速する**。特に、明確に定義されたメソッドレベルの問題に取り組む経験の浅い開発者に有効。
2. **生産性向上は経験豊富な開発者には減少または逆転する**。大規模コードベースでの複雑な実世界タスクにおいて顕著。
3. **コード品質への影響は混在**：関数レベルでの可読性と正確性の小さな改善は、リポジトリレベルでの重複増加、リファクタリング減少、コードチャーン増加と共存。
4. **セキュリティ脆弱性は体系的リスク**。AI生成コードの約4分の1に弱点が含まれる。静的解析ツールとの統合は問題を改善するが排除はできない。
5. **技術的負債が蓄積**。コード量の増加、リファクタリングの減少、再作業率の上昇を通じて進行し、保守負担は経験豊富な開発者に不均衡に帰着。
6. **AIの使い方が使用の有無より重要**：概念的な問いかけは学習を保持し、コード委任はそれを阻害する。
7. **信頼は双方向に不適切に較正されている**：多くの開発者が理解していないコードを使用する一方、全体的な信頼の低下が有用な機能の過小利用を招く可能性がある。

### 未解決の課題

- AI多用コードベースにおける**長期的（複数年）技術的負債の軌跡**はどうなるか？
- **組織的プラクティス**（必須コードレビュー、静的解析統合、ペアAI使用）は品質トレードオフを緩和できるか？
- **エージェント型コーディングツール**（セッションあたり平均47回のツール呼び出し、78%のセッションでマルチファイル編集を処理）は生産性と品質の計算をどう変えるか？
- 開発者がスキルを保持する形でAIを採用するための**トレーニングとオンボーディングプラクティス**は何か？
- 個々の開発者が従来の2〜3倍の速度でコードを生産する場合、**チームダイナミクスと協調パターン**はどう変化するか？

エビデンスは、AIコーディングアシスタントが普遍的な生産性乗数でも、緩和されないリスクでもないことを明確に示している。その影響は、誰が使うか、どのように使うか、どのような組織的ガードレールが設けられているかに深く依存する。最も生産的な道は、これらのツールを一括して採用または拒否することではなく、ネットポジティブな結果を生み出す条件を理解し形成することに投資することである。

---

## 10. 参考文献

1. <a href="https://arxiv.org/abs/2302.06590" target="_blank">Peng, S., Kalliamvakou, E., Cihon, P., & Demirer, M. (2023). "The Impact of AI on Developer Productivity: Evidence from GitHub Copilot." arXiv:2302.06590</a>
2. <a href="https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4945566" target="_blank">Cui, Z.K., Demirer, M., Jaffe, S., Musolff, L., Peng, S., & Salz, T. (2024). "The Effects of Generative AI on High-Skilled Work: Evidence from Three Field Experiments with Software Developers." Management Science.</a>
3. <a href="https://arxiv.org/html/2410.12944v2" target="_blank">Paradis, E., Grey, K., Madison, Q., et al. (2024). "How Much Does AI Impact Development Speed? An Enterprise-Based Randomized Controlled Trial." Google Research.</a>
4. <a href="https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/" target="_blank">Becker, J., Rush, N., Barnes, E., & Rein, D. (2025). "Measuring the Impact of Early-2025 AI on Experienced Open-Source Developer Productivity." METR.</a>
5. <a href="https://arxiv.org/abs/2510.10165" target="_blank">Xu, F., Medappa, P.K., Tunc, M.M., Vroegindeweij, M., & Fransoo, J.C. (2025). "AI-Assisted Programming Decreases the Productivity of Experienced Developers by Increasing the Technical Debt and Maintenance Burden." arXiv:2510.10165</a>
6. <a href="https://arxiv.org/abs/2507.03156" target="_blank">Mohamed, A., Assi, M., & Guizani, M. (2025). "The Impact of LLM-Assistants on Software Developer Productivity: A Systematic Literature Review." arXiv:2507.03156</a>
7. <a href="https://cacm.acm.org/research/measuring-github-copilots-impact-on-productivity/" target="_blank">Ziegler, A. et al. (2024). "Measuring GitHub Copilot's Impact on Productivity." Communications of the ACM, 67(3).</a>
8. <a href="https://github.blog/news-insights/research/does-github-copilot-improve-code-quality-heres-what-the-data-says/" target="_blank">GitHub Research (2025). "Does GitHub Copilot Improve Code Quality? Here's What the Data Says."</a>
9. <a href="https://dl.acm.org/doi/10.1145/3597503.3639219" target="_blank">Du, X. et al. (2024). "Evaluating Large Language Models in Class-Level Code Generation." ICSE 2024.</a>
10. <a href="https://arxiv.org/abs/2406.08731" target="_blank">Wang, Z. et al. (2025). "Towards Understanding the Characteristics of Code Generation Errors Made by Large Language Models." ICSE 2025.</a>
11. <a href="https://dora.dev/research/2024/dora-report/" target="_blank">Google (2024). "Accelerate State of DevOps Report 2024." DORA.</a>
12. <a href="https://dl.acm.org/doi/10.1145/3716848" target="_blank">Fu, Y., Liang, P., Tahir, A., et al. (2024). "Security Weaknesses of Copilot-Generated Code in GitHub Projects: An Empirical Study." ACM TOSEM.</a>
13. <a href="https://arxiv.org/abs/2204.04741" target="_blank">Asare, O. et al. (2022). "Is GitHub's Copilot as Bad as Humans at Introducing Vulnerabilities in Code?" arXiv:2204.04741</a>
14. <a href="https://www.gitclear.com/ai_assistant_code_quality_2025_research" target="_blank">GitClear (2025). "AI Copilot Code Quality: 2025 Data Suggests 4x Growth in Code Clones."</a>
15. <a href="https://arxiv.org/html/2510.03029v1" target="_blank">Ghosh Paul, D., Zhu, H., & Bayley, I. (2025). "Investigating The Smells of LLM Generated Code." Oxford Brookes University.</a>
16. <a href="https://www.sonarsource.com/blog/state-of-code-developer-survey-report-the-current-reality-of-ai-coding/" target="_blank">SonarSource (2026). "State of Code Developer Survey Report 2026."</a>
17. <a href="https://arxiv.org/html/2601.20245v1" target="_blank">Shen, J.H. & Tamkin, A. (2026). "How AI Impacts Skill Formation." Anthropic. arXiv:2601.20245</a>
18. <a href="https://dl.acm.org/doi/10.1145/3706599.3706670" target="_blank">Weisz, J.D. et al. (2025). "Examining the Use and Impact of an AI Code Assistant on Developer Productivity and Experience in the Enterprise." CHI EA '25.</a>
19. <a href="https://stackoverflow.blog/2026/02/18/closing-the-developer-ai-trust-gap/" target="_blank">Stack Overflow (2026). "Mind the Gap: Closing the AI Trust Gap for Developers."</a>
20. <a href="https://clutch.co/resources/devs-use-ai-generated-code-they-dont-understand" target="_blank">Clutch (2025). "Blind Trust in AI: Most Devs Use AI-Generated Code They Don't Understand."</a>
21. <a href="https://arxiv.org/html/2509.13253v1" target="_blank">Amoozadeh, A. et al. (2024). "Evolution of Programmers' Trust in Generative AI Programming Assistants."</a>
22. <a href="https://arxiv.org/html/2504.13903v1" target="_blank">"From Teacher to Colleague: How Coding Experience Shapes Developer Perceptions of AI Tools." (2025)</a>
23. <a href="https://neurips.cc/virtual/2024/poster/97835" target="_blank">IaC-Eval (NeurIPS 2024). "A Code Generation Benchmark for Cloud Infrastructure-as-Code Programs."</a>
24. <a href="https://proceedings.neurips.cc/paper_files/paper/2024/file/6a059625a6027aca18302803743abaa2-Paper-Datasets_and_Benchmarks_Track.pdf" target="_blank">EvoCodeBench (NeurIPS 2024). "An Evolving Code Generation Benchmark."</a>
25. <a href="https://neurips.cc/virtual/2025/poster/121850" target="_blank">ResearchCodeBench (NeurIPS 2025). "Benchmarking LLMs on Implementing Novel Machine Learning Research Code."</a>
26. <a href="https://arxiv.org/html/2509.16941" target="_blank">SWE-Bench Pro (2025). "Can AI Agents Solve Long-Horizon Software Engineering Tasks?"</a>
27. <a href="https://arxiv.org/html/2602.10975v1" target="_blank">FeatureBench (2026). "Benchmarking Agentic Coding for Complex Feature Development."</a>
28. <a href="https://resources.anthropic.com/hubfs/2026%20Agentic%20Coding%20Trends%20Report.pdf?hsLang=en" target="_blank">Anthropic (2026). "2026 Agentic Coding Trends Report."</a>
29. <a href="https://www.anthropic.com/research/AI-assistance-coding-skills" target="_blank">Anthropic (2026). "How AI Assistance Impacts the Formation of Coding Skills."</a>
30. <a href="https://arxiv.org/html/2603.13724" target="_blank">"Testing with AI Agents: An Empirical Study of Test Generation Frequency, Quality, and Coverage." (2026)</a>
31. <a href="https://www.sciencedirect.com/science/article/pii/S0164121225002687" target="_blank">"The Evolution of Technical Debt from DevOps to Generative AI: A Multivocal Literature Review." Journal of Systems and Software (2025).</a>
32. <a href="https://cacm.acm.org/opinion/redefining-the-software-engineering-profession-for-ai/" target="_blank">"Redefining the Software Engineering Profession for AI." Communications of the ACM (2025).</a>
33. <a href="https://cacm.acm.org/practice/toward-effective-ai-support-for-developers/" target="_blank">"Toward Effective AI Support for Developers." Communications of the ACM (2025).</a>
34. <a href="https://github.blog/news-insights/research/research-quantifying-github-copilots-impact-in-the-enterprise-with-accenture/" target="_blank">GitHub (2024). "Research: Quantifying GitHub Copilot's Impact in the Enterprise with Accenture."</a>
35. <a href="https://arxiv.org/html/2501.02684v1" target="_blank">"Towards Decoding Developer Cognition in the Age of AI Assistants." (2025)</a>
36. <a href="https://stemeducationjournal.springeropen.com/articles/10.1186/s40594-025-00537-3" target="_blank">"The Impact of AI-Assisted Pair Programming on Student Motivation, Programming Anxiety, Collaborative Learning, and Programming Performance." International Journal of STEM Education (2025).</a>
37. <a href="https://arxiv.org/html/2601.20112" target="_blank">"Usage, Effects and Requirements for AI Coding Assistants in the Enterprise: An Empirical Study." (2026)</a>
