# AI-Assisted Software Development: Productivity, Code Quality, and Developer Cognition

## A Comprehensive Survey of Empirical Research (2023–2026)

---

## Executive Summary

The rapid adoption of AI coding assistants — led by GitHub Copilot and increasingly by agentic tools such as Cursor and Claude Code — has prompted an unprecedented wave of empirical research. This report synthesizes findings from controlled experiments, large-scale observational analyses, systematic literature reviews, and industry surveys published between 2023 and 2026, drawing on venues including ICSE, FSE, NeurIPS, IEEE TSE, TOSEM, CHI, CACM, and DORA.

**Key findings include:**

- **Productivity gains are real but uneven.** Three large-scale RCTs across Microsoft, Accenture, and Google show 21–26% increases in task throughput for Copilot users, with the greatest benefits accruing to less experienced developers (up to 35–39% speedup). However, the METR 2025 RCT found that experienced open-source developers actually took 19% *longer* with AI tools — a result that contradicted both developer self-estimates and expert forecasts.
- **Code quality is a double-edged sword.** GitHub's own 202-developer RCT reported small but statistically significant improvements in readability (+3.6%), reliability (+2.9%), and maintainability (+2.5%). Conversely, GitClear's analysis of 211 million lines of code found that code duplication surged 48%, code churn increased dramatically, and refactoring declined from 25% to under 10% of changed lines between 2021 and 2024.
- **Security vulnerabilities are prevalent.** An empirical study of 733 Copilot-generated code snippets found 27.3% contained security weaknesses spanning 43 CWE categories, with eight mapping to the 2023 CWE Top-25.
- **Technical debt is accumulating.** Experienced developers face a 19% drop in original-code productivity and a 6.5% increase in review workload after Copilot adoption, as AI-generated code requires more rework to meet repository standards.
- **Skill formation is impaired.** Anthropic's 2026 RCT found that AI-assisted learners scored 17% lower on comprehension assessments, with debugging skills showing the largest gap. Developers who delegated code generation scored below 40%, while those who used AI for conceptual inquiry scored 65% or higher.
- **Trust is miscalibrated.** Stack Overflow's 2025 survey found 84% of developers use AI tools, but only 29% trust them — down from 43% in 2024. Meanwhile, 59% of developers admit to using AI-generated code they don't fully understand.

---

## Table of Contents

1. [Productivity Impact: Evidence from Controlled Experiments](#1-productivity-impact-evidence-from-controlled-experiments)
2. [Code Quality and Defect Rates](#2-code-quality-and-defect-rates)
3. [Security Vulnerabilities in AI-Generated Code](#3-security-vulnerabilities-in-ai-generated-code)
4. [Maintainability and Technical Debt](#4-maintainability-and-technical-debt)
5. [Developer Cognition, Comprehension, and Skill Formation](#5-developer-cognition-comprehension-and-skill-formation)
6. [Moderating Factors](#6-moderating-factors)
7. [Benchmarks and the Evaluation Gap](#7-benchmarks-and-the-evaluation-gap)
8. [Methodological Landscape and Limitations](#8-methodological-landscape-and-limitations)
9. [Conclusions and Open Questions](#9-conclusions-and-open-questions)
10. [Sources](#10-sources)

---

## 1. Productivity Impact: Evidence from Controlled Experiments

### 1.1 The Foundational Copilot RCT (Peng et al., 2023)

The earliest rigorous evidence came from <a href="https://arxiv.org/abs/2302.06590" target="_blank">Peng et al. (2023)</a>, who conducted a controlled experiment in which professional programmers were tasked with implementing an HTTP server in JavaScript. Developers with GitHub Copilot access completed the task **55.8% faster** than the control group. The study identified heterogeneous effects: less experienced, higher-workload, and older developers benefited most, suggesting that AI assistants partially compensate for gaps in domain knowledge or working memory.

### 1.2 Three-Company Field Experiments (Cui et al., 2024)

The most comprehensive industry evidence comes from <a href="https://papers.ssrn.com/sol3/papers.cfm?abstract_id=4945566" target="_blank">Cui, Demirer, Jaffe, Musolff, Peng, and Salz (2024)</a>, published in *Management Science*. The authors ran randomized controlled trials at Microsoft (1,663 developers), Accenture, and an anonymous Fortune 100 electronics manufacturer, totalling **4,867 developers**. Key findings:

- An average **26.08% increase** in weekly completed pull requests (SE: 10.3%) for the treatment group.
- Less experienced developers saw the largest gains: **35–39% speedup**, compared with **8–16%** for senior developers.
- The tool version used was based on GPT-3.5, suggesting that modern models may yield different effect sizes.

### 1.3 Google Enterprise RCT (Paradis et al., 2024)

<a href="https://arxiv.org/html/2410.12944v2" target="_blank">Paradis et al. (2024)</a> conducted a randomized controlled trial with **96 full-time Google engineers** who completed a standardized code-modification task (updating 10 files, 474 lines). The treatment group completed the task in approximately 96 minutes versus 114 minutes — roughly **21% faster**. The main effect was significant without controls (p=0.038) but lost significance when adjusting for developer and task characteristics (p=0.086), highlighting the importance of covariates.

### 1.4 The METR Surprise: AI Makes Experienced Developers Slower (Becker et al., 2025)

The most counterintuitive finding came from <a href="https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/" target="_blank">METR's 2025 RCT</a>. Sixteen experienced open-source developers (averaging 5 years on their respective projects, which had 22k+ stars and 1M+ lines of code) completed 246 real issues randomly assigned to allow or disallow AI tools (primarily Cursor Pro with Claude 3.5/3.7 Sonnet).

**Result:** AI usage increased task completion time by **19%** (CI: +2% to +39%). Developers had predicted a 24% speedup; even after experiencing the slowdown, they still believed AI had accelerated their work by 20%.

Five contributing factors were identified:
1. Friction from tool context-switching
2. Hallucinations requiring verification
3. Over-reliance on imperfect suggestions
4. Learning curve with unfamiliar AI interfaces
5. Quality and documentation standards exceeding AI output capability

Importantly, the quality of submitted PRs was similar in both conditions, ruling out quality degradation as an explanation.

### 1.5 The Productivity Paradox (Xu et al., 2025)

<a href="https://arxiv.org/abs/2510.10165" target="_blank">Xu, Medappa, Tunc, Vroegindeweij, and Fransoo (2025)</a> analyzed open-source projects before and after Copilot's introduction. While overall productivity metrics rose, the gains concentrated among less experienced developers, masking a **19% drop in original-code productivity** for core (experienced) developers and a **6.5% increase in code review workload**. The paper argues that short-term aggregate productivity gains obscure growing maintenance burdens on scarce expert resources.

### 1.6 Systematic Review Evidence (Mohamed et al., 2025)

<a href="https://arxiv.org/abs/2507.03156" target="_blank">Mohamed, Assi, and Guizani (2025)</a> reviewed 37 peer-reviewed studies (2014–2024) and found consistent benefits in "minimized code search, accelerated development, and automation of trivial tasks." However, they also documented concerns around cognitive offloading, reduced team collaboration, and inconsistent code quality effects. Critically, 92% of studies measured at least two SPACE framework dimensions, but only 14% extended beyond three, and 64% used exploratory rather than confirmatory methodologies. The authors called for "longitudinal and team-based evaluations."

### 1.7 CACM Perspective: Measuring Copilot's Impact (Ziegler et al., 2024)

<a href="https://cacm.acm.org/research/measuring-github-copilots-impact-on-productivity/" target="_blank">Ziegler et al. (2024)</a>, published in *Communications of the ACM*, analyzed 2,631 survey responses matched to IDE telemetry. They found that suggestion acceptance rate (21.2–23.5%) was the strongest predictor of perceived productivity. 73% of developers reported maintaining flow state more effectively, and 87% reported preserved mental effort on repetitive tasks. The study also noted that developers sometimes valued even erroneous suggestions, complicating simple acceptance-rate metrics.

---

## 2. Code Quality and Defect Rates

### 2.1 GitHub's 202-Developer Code Quality RCT (GitHub Research, 2025)

<a href="https://github.blog/news-insights/research/does-github-copilot-improve-code-quality-heres-what-the-data-says/" target="_blank">GitHub Research (February 2025)</a> conducted a randomized controlled trial with 202 developers (5+ years of experience) who wrote API endpoints. Code was blind-reviewed by 25 expert evaluators. Results:

| Metric | Improvement | p-value |
|---|---|---|
| Readability | +3.62% | 0.003 |
| Reliability | +2.94% | 0.01 |
| Maintainability | +2.47% | 0.041 |
| Conciseness | +4.16% | 0.002 |
| Unit test pass (all 10) | +53.2% likelihood | <0.01 |
| Reviewer approval | +5% | 0.014 |

The researchers hypothesized that Copilot freed cognitive resources, allowing developers to focus on quality refinement rather than achieving baseline functionality. Critics, however, note that the study was conducted by GitHub itself and that the controlled task (API endpoint creation) may not generalize to more complex development scenarios.

### 2.2 ClassEval: Class-Level Code Generation (Du et al., ICSE 2024)

<a href="https://dl.acm.org/doi/10.1145/3597503.3639219" target="_blank">Du et al. (ICSE 2024)</a> introduced **ClassEval**, the first class-level code generation benchmark: 100 classes, 410 methods, and 33.1 test cases per class on average. All tested LLMs performed substantially worse on class-level generation compared with method-level tasks. GPT-4 outperformed the third-ranked model (WizardCoder) by 25.4% in class-level Pass@1, but absolute performance remained modest. The study underscores a significant gap between the benchmark numbers commonly reported (method-level HumanEval) and real-world development scenarios involving interdependent code structures.

### 2.3 Code Generation Error Taxonomy (Wang et al., ICSE 2025)

<a href="https://arxiv.org/abs/2406.08731" target="_blank">Wang et al. (ICSE 2025)</a> conducted the first systematic taxonomy of LLM code generation errors. Analyzing 557 errors from six LLMs on HumanEval, they found that models frequently produce "non-trivial, multi-line code generation errors in various locations and with various root causes." The taxonomy spans both semantic and syntactic error dimensions, and the authors identified significant challenges in locating and fixing these errors — suggesting that automated program repair for LLM-generated code remains an open problem.

### 2.4 DORA 2024: The Delivery Stability Trade-off (Google, 2024)

<a href="https://dora.dev/research/2024/dora-report/" target="_blank">Google's 2024 DORA Report</a>, the industry's flagship DevOps survey, found that for every 25% increase in AI adoption:

- Code review speed increased by **3.1%**
- Code quality increased by **3.4%**
- But delivery stability **decreased by 7.2%**
- And delivery throughput **decreased by 1.5%**
- 39% of respondents reported little to no trust in AI-generated code

The report theorizes that faster code generation without corresponding improvements in testing and review practices contributes to the stability decrease.

---

## 3. Security Vulnerabilities in AI-Generated Code

### 3.1 Large-Scale Security Weakness Analysis (Fu et al., 2023–2024)

<a href="https://dl.acm.org/doi/10.1145/3716848" target="_blank">Fu et al. (TOSEM, 2024)</a> conducted the most comprehensive security analysis to date, examining **733 code snippets** from GitHub projects generated by Copilot (672), CodeWhisperer (38), and Codeium (23), using CodeQL, Bandit, and ESLint.

**Key findings:**
- **27.3%** of all snippets contained security weaknesses (200/733)
- Python: 29.5% affected; JavaScript: 24.2% affected
- **628 total vulnerabilities** across 43 CWE categories
- Average of 3 weaknesses per affected snippet; 51% contained multiple issues
- Top 3 CWEs: CWE-330 (Insufficient Randomness, 18.15%), CWE-94 (Code Injection, 9.87%), CWE-79 (XSS, 9.55%)
- Eight CWEs aligned with the 2023 CWE Top-25 (37.1% of all detections)

**Copilot Chat fix effectiveness:**
- Default "/fix" command: 19.3% fix rate
- Basic prompt: 31.8%
- Enhanced prompt with static analysis warnings: **55.5%**

The finding that providing static analysis context improves fix rates substantially suggests that integrating security tools into AI workflows could meaningfully reduce vulnerability propagation.

### 3.2 Vulnerability Replication Rates

Earlier work by <a href="https://arxiv.org/abs/2204.04741" target="_blank">Asare et al. (2022)</a>, examining C/C++ vulnerabilities, found that Copilot replicated the original vulnerable code approximately **33% of the time** while replicating fixed code at only a 25% rate, indicating a bias toward reproducing known vulnerability patterns.

---

## 4. Maintainability and Technical Debt

### 4.1 GitClear: 211 Million Lines of Evidence (2024–2025)

<a href="https://www.gitclear.com/ai_assistant_code_quality_2025_research" target="_blank">GitClear's 2025 research</a> analyzed **211 million changed lines of code** from 2020 to 2024 across repositories owned by Google, Microsoft, Meta, and enterprise corporations. The findings reveal systematic shifts in code composition:

| Metric | 2020/2021 | 2024 | Change |
|---|---|---|---|
| Refactored ("moved") lines | 24.1% (2020) | 9.5% | −61% |
| Copy/pasted (cloned) lines | 8.3% (2020) | 12.3% | +48% |
| Newly added lines | 39% (2020) | 46% | +18% |
| Code revised within 2 weeks | 5.5% (2020) | 7.9% | +44% |
| Refactoring share of changes | 25% (2021) | <10% | −60% |

The report documents that in 2024, copy/pasted lines exceeded moved lines for the first time in the dataset's history, and code duplication blocks rose eightfold. These metrics are consistent with an accumulation of technical debt: more code is being added, less is being refactored, and more is being thrown away shortly after creation.

### 4.2 The Experienced Developer Burden (Xu et al., 2025)

As detailed in Section 1.5, <a href="https://arxiv.org/abs/2510.10165" target="_blank">Xu et al. (2025)</a> quantified the redistribution of work after Copilot adoption. The core tension is between "short-term productivity gains and long-term system sustainability": while less experienced developers produce more code, the burden of ensuring that code meets quality standards falls disproportionately on senior developers, whose own original-code output declines by 19%.

### 4.3 Code Smells in LLM-Generated Code (Ghosh Paul et al., 2025)

<a href="https://arxiv.org/html/2510.03029v1" target="_blank">Ghosh Paul, Zhu, and Bayley (2025)</a> compared Java code generated by four LLMs (Gemini Pro, ChatGPT, Codex, Falcon) against professional human-written solutions across 1,000 tasks.

**Key findings:**
- LLM-generated code showed **42.28%–84.97% higher code smell incidence** than human reference solutions (average: 63.34%)
- Implementation smells: +73.35% vs. human baselines
- Design smells: +21.42%
- Worst performance on advanced topics: encapsulation (+138.53%), array handling (+101.88%), OOP (+101.88%)
- Strong positive correlation (0.9653) between cyclomatic complexity and smell frequency — LLMs degrade more dramatically than humans as tasks grow complex

### 4.4 Sonar 2026 State of Code Survey

<a href="https://www.sonarsource.com/blog/state-of-code-developer-survey-report-the-current-reality-of-ai-coding/" target="_blank">Sonar's 2026 survey</a> (N=1,149 developers) found that AI-generated code now accounts for **42% of all committed code**, expected to rise to 65% by 2027. While 93% of developers report positive effects (improved documentation: 57%; test coverage: 53%), **88% also cite negative impacts**: code that "looks correct but isn't reliable" (53%) or is "unnecessary and duplicative." Perhaps most strikingly, **96% do not fully trust AI-generated code, but only 48% always verify it before committing** — a "verification gap" that may amplify technical debt accumulation.

---

## 5. Developer Cognition, Comprehension, and Skill Formation

### 5.1 AI Impairs Skill Formation (Shen & Tamkin, Anthropic, 2026)

<a href="https://arxiv.org/html/2601.20245v1" target="_blank">Shen and Tamkin (2026)</a> from Anthropic conducted a randomized controlled experiment with 52 participants learning the Python Trio library for asynchronous programming. The AI-assisted group scored **17% lower** on skill assessments (Cohen's d = 0.738, p = 0.010), with no statistically significant productivity gains.

The study identified six distinct AI interaction patterns with dramatically different outcomes:

| Pattern | N | Quiz Score Range |
|---|---|---|
| AI Delegation | 4 | 24–39% |
| Progressive AI Reliance | 4 | 24–39% |
| Iterative AI Debugging | 4 | 24–39% |
| Generation-Then-Comprehension | 2 | 65–86% |
| Hybrid Code-Explanation | 3 | 65–86% |
| Conceptual Inquiry | 7 | 65–86% |

The critical distinction: developers who used AI for **conceptual inquiry** (asking "why" and "how" rather than requesting code) scored 65%+ and retained understanding, while those who **delegated code generation** scored below 40%. The control group encountered a median of 3 errors versus 1 for the AI group, suggesting that encountering and resolving errors is itself a learning mechanism.

### 5.2 Enterprise Developer Experience (Weisz et al., CHI 2025)

<a href="https://dl.acm.org/doi/10.1145/3706599.3706670" target="_blank">Weisz et al. (CHI EA '25)</a> studied 669 IBM developers using watsonx Code Assistant through surveys and usability testing. Notable findings:

- Developers prioritized **code understanding** (71.9%) over code generation (55.6%)
- Only **2–4% accepted generated outputs verbatim**; 23–37% used outputs for "learning/inspiration"
- 42.6% felt **less effective** with the tool; 57.4% felt more effective
- Users compared the AI to "an intern" requiring supervision
- Explicit **deskilling anxiety** was reported, alongside stigma about "being seen with generated code in PRs"

### 5.3 The Trust Calibration Crisis

Multiple sources document a growing trust paradox:

- **Stack Overflow 2025 Survey**: 84% of developers use AI tools, but only **29% trust them** (down from 43% in 2024) (<a href="https://stackoverflow.blog/2026/02/18/closing-the-developer-ai-trust-gap/" target="_blank">Stack Overflow, 2026</a>)
- **Clutch 2025 Survey** (N=800): **59% of developers** use AI-generated code they don't fully understand (<a href="https://clutch.co/resources/devs-use-ai-generated-code-they-dont-understand" target="_blank">Clutch, 2025</a>)
- **DORA 2024**: 39% of respondents reported little to no trust in AI-generated code
- **Sonar 2026**: 96% don't fully trust AI output, but only 48% always verify

<a href="https://arxiv.org/html/2509.13253v1" target="_blank">Amoozadeh et al. (2024)</a> found that students in operating systems courses exhibited lower trust in AI assistants than CS2 students, and all participants trusted code comprehension features more than code generation features — suggesting trust is appropriately calibrated to perceived difficulty rather than blanketly assigned.

---

## 6. Moderating Factors

The literature consistently identifies several factors that moderate AI's impact on productivity and quality:

### 6.1 Developer Experience

This is the most robust moderating factor across studies:
- **Cui et al. (2024)**: Less experienced developers gained 35–39% speedup; senior developers only 8–16%
- **METR (2025)**: Experienced open-source maintainers were 19% slower with AI
- **Xu et al. (2025)**: Experienced developers bore disproportionate review and maintenance burden
- **Shen & Tamkin (2026)**: AI usage pattern (delegation vs. inquiry) predicted skill outcomes more than raw experience

<a href="https://arxiv.org/html/2504.13903v1" target="_blank">A 2025 study</a> found that developer experience does not predict *whether* developers adopt AI tools, but shapes the *roles* they assign: experienced developers treat AI as a "junior colleague" or "content generator," while novices treat it as a "teacher."

### 6.2 Task Complexity

- **ClassEval (ICSE 2024)**: LLM performance degrades sharply on class-level vs. method-level tasks
- **Ghosh Paul et al. (2025)**: Code smell incidence increases with cyclomatic complexity, and the gap between human and LLM code widens as complexity rises
- **IaC-Eval (NeurIPS 2024)**: GPT-4 achieved only 19.36% pass@1 on infrastructure code vs. 86.6% on simpler Python benchmarks
- **SWE-Bench Pro (2025)**: Top models achieve 23% on complex real-world tasks vs. 70%+ on isolated bug fixes

### 6.3 Code Domain and Language

- **Fu et al. (2024)**: Python (29.5%) showed higher vulnerability rates than JavaScript (24.2%); web applications dominated JavaScript vulnerabilities while utility tools showed highest density in Python
- **NeurIPS benchmarks**: Performance varies dramatically by domain — near-90% on HumanEval Python, 19% on infrastructure code, and under 40% on research code implementation

### 6.4 Organizational Context

- **Sonar 2026**: Enterprises investing in governance produce higher-quality AI-assisted code; SMBs gain speed but feel pain in verification and rework
- **Weisz et al. (CHI 2025)**: Organizational culture (endorsement vs. stigma around AI use) significantly affects adoption patterns
- **DORA 2024**: Teams without robust testing mechanisms saw the largest stability decreases

---

## 7. Benchmarks and the Evaluation Gap

### 7.1 The Benchmark Saturation Problem

HumanEval, the foundational benchmark for code generation, is nearing saturation: top models achieve near or above 90% on Python problems. Yet performance on more realistic benchmarks tells a different story:

| Benchmark | Top Performance | Scope |
|---|---|---|
| HumanEval (Python) | ~95% | 164 single-function problems |
| ClassEval (ICSE 2024) | GPT-4: ~65% | 100 classes, 410 methods |
| IaC-Eval (NeurIPS 2024) | GPT-4: 19.36% | 458 infrastructure scenarios |
| SWE-bench Verified | Claude 4.5: 74.4% | Real GitHub issues (isolated) |
| SWE-Bench Pro (2025) | ~23% | 1,865 complex multi-file tasks |
| FeatureBench (2026) | Claude 4.5: 11.0% | Complex feature development |
| ResearchCodeBench (NeurIPS 2025) | <40% | Novel ML research implementation |

This progression from near-perfect scores on simple benchmarks to below-40% on complex tasks illuminates the gap between what AI can do in controlled settings and what real-world development demands.

### 7.2 Emerging Evaluation Frameworks

Recent benchmarks have expanded beyond functional correctness:
- **SWE-EVO (2025)**: Evaluates agents across long-horizon software evolution scenarios spanning multiple commits
- **FeatureBench (2026)**: Tests complex feature development requiring architectural decisions
- **DPAI Arena (JetBrains, 2025)**: Evaluates full-lifecycle developer tasks including code review, test generation, and static analysis

---

## 8. Methodological Landscape and Limitations

### 8.1 Study Design Distribution

<a href="https://arxiv.org/abs/2507.03156" target="_blank">Mohamed et al.'s (2025) SLR</a> found that 64% of studies used exploratory methodologies, and only 14% examined more than three dimensions of the SPACE productivity framework. The field suffers from several systematic limitations:

1. **Short time horizons**: Most RCTs measure hours or days, not the weeks-to-months timeline over which technical debt manifests
2. **Controlled tasks vs. real work**: Many studies use standardized tasks (HTTP servers, API endpoints) that may not generalize to complex legacy codebases
3. **Selection effects**: Field experiments often rely on voluntary adoption, introducing selection bias
4. **Proxy metrics**: Pull request counts, lines of code, and acceptance rates are imperfect proxies for genuine productivity
5. **Conflict of interest**: Several high-profile positive studies were conducted by tool vendors (GitHub, Microsoft)
6. **Missing dimensions**: Communication, collaboration, and Activity dimensions of the SPACE framework remain significantly underexplored

### 8.2 The Hawthorne Effect and Self-Report Bias

The METR study powerfully demonstrated self-report bias: developers believed AI made them faster even when objective measurements showed a 19% slowdown. This finding casts doubt on the many survey-based studies reporting productivity improvements, including Ziegler et al.'s CACM study, which relied heavily on self-reported perceptions matched to IDE telemetry.

### 8.3 The Need for Longitudinal Research

Perhaps the most critical gap is the absence of longitudinal studies tracking:
- Technical debt accumulation over quarters and years
- Developer skill trajectories with sustained AI use
- Team-level collaboration dynamics
- Codebase health metrics over full product lifecycles

GitClear's five-year analysis is the closest approximation, but it uses observational data that cannot establish causation. The field urgently needs longitudinal RCTs.

---

## 9. Conclusions and Open Questions

### What the Evidence Supports

1. **AI assistants accelerate routine coding tasks**, particularly for less experienced developers working on well-defined, method-level problems.
2. **Productivity gains diminish or reverse** for experienced developers working on complex, real-world tasks in large codebases.
3. **Code quality impacts are mixed**: small improvements in readability and correctness at the function level coexist with increased duplication, reduced refactoring, and higher code churn at the repository level.
4. **Security vulnerabilities are a systemic risk**, with roughly one-quarter of AI-generated code containing weaknesses. Integration with static analysis tools can improve but not eliminate the problem.
5. **Technical debt is accumulating** through increased code volume, reduced refactoring, and higher rework rates, with the maintenance burden falling disproportionately on experienced developers.
6. **How developers use AI matters more than whether they use it**: conceptual inquiry preserves learning; code delegation impairs it.
7. **Trust is poorly calibrated** in both directions: many developers use code they don't understand, while declining aggregate trust may cause others to underutilize genuinely helpful capabilities.

### Open Questions

- **What is the long-term (multi-year) trajectory of technical debt** in AI-heavy codebases?
- **Can organizational practices** (mandatory code review, static analysis integration, paired AI use) mitigate the quality trade-offs?
- **How will agentic coding tools** (which already average 47 tool calls per session and handle multi-file edits in 78% of sessions) change the productivity and quality calculus?
- **What training and onboarding practices** can help developers adopt AI in skill-preserving ways?
- **How do team dynamics and collaboration patterns** change when individual developers produce code at 2–3x the previous rate?

The evidence is clear that AI coding assistants are neither a universal productivity multiplier nor an unmitigated risk. Their impact depends profoundly on who uses them, how they are used, and what organizational guardrails are in place. The most productive path forward is not to adopt or reject these tools wholesale, but to invest in understanding and shaping the conditions under which they produce net-positive outcomes.

---

## 10. Sources

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
