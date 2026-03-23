# AI-Generated Test Suites: Quality, Effectiveness, and the Gap Between Coverage and Bug Detection

## Executive Summary

Large Language Models are rapidly transforming software testing. From 1 paper in 2022 to 55 in 2024 and 84+ in the first eight months of 2025, the research field has exploded. Commercial tools (GitHub Copilot, Qodo, Diffblue Cover) and industrial deployments (Meta's TestGen-LLM and ACH) have brought LLM-based test generation into production codebases. Yet the central question remains contentious: **do AI-generated tests actually find bugs, or do they merely inflate coverage numbers?**

The evidence is nuanced. On well-documented, self-contained code, LLMs achieve 70–93% line coverage and mutation scores above 89%. On complex, real-world codebases, they sometimes fall below 2% coverage — underperforming classical tools like EvoSuite by an order of magnitude. The most alarming finding: test suites can reach 100% line coverage while achieving only a 4% mutation score, exposing a fundamental disconnect between coverage metrics and fault-detection capability.

Mutation-guided approaches — notably Meta's ACH system and academic tools like MuTAP — represent the most promising direction, achieving 93%+ mutation scores by forcing LLMs to generate tests that discriminate between correct and buggy code. Meanwhile, the test oracle problem persists: LLMs tend to capture *actual* program behavior rather than *expected* behavior, encoding existing bugs as correct. Developer trust is declining even as adoption rises — 84% of developers use AI tools, but only 29% trust the output.

This report synthesizes findings from over 60 peer-reviewed papers, industrial reports, and empirical studies published between 2022 and 2026.

---

## Table of Contents

1. [The Tool and Technique Landscape](#1-the-tool-and-technique-landscape)
2. [Coverage Metrics: What the Numbers Actually Show](#2-coverage-metrics-what-the-numbers-actually-show)
3. [Mutation Testing: The True Measure of Fault Detection](#3-mutation-testing-the-true-measure-of-fault-detection)
4. [Real Bugs vs. Coverage Inflation](#4-real-bugs-vs-coverage-inflation)
5. [The Test Oracle Problem](#5-the-test-oracle-problem)
6. [Flakiness in AI-Generated Tests](#6-flakiness-in-ai-generated-tests)
7. [Quality Failure Modes and Anti-Patterns](#7-quality-failure-modes-and-anti-patterns)
8. [Prompting Strategies and Feedback Loops](#8-prompting-strategies-and-feedback-loops)
9. [Developer Trust and Adoption](#9-developer-trust-and-adoption)
10. [Open Questions and Future Directions](#10-open-questions-and-future-directions)
11. [Conclusions](#11-conclusions)
12. [Sources](#12-sources)

---

## 1. The Tool and Technique Landscape

### 1.1 Commercial and Industrial Tools

The LLM-based test generation ecosystem has matured rapidly since 2022, with distinct architectural philosophies emerging.

**GitHub Copilot** generates tests via inline suggestions and chat-based prompts within the IDE. A dedicated testing feature for .NET launched in 2024–2025 produces deterministic results grounded in the C# compiler with deep awareness of codebase structure and testing conventions. An empirical study at AST 2024 found that within an existing test suite, approximately 45% of Copilot-generated tests pass; without existing tests as context, 92% of generated tests are failing, broken, or empty — highlighting the critical dependence on contextual information (<a href="https://dl.acm.org/doi/10.1145/3644032.3644443" target="_blank">Dakhel et al., AST 2024</a>).

**Qodo (formerly CodiumAI)** operates as a multi-agent system that builds an internal semantic index of the codebase and analyzes dependency graphs. Its open-source **Qodo Cover** (formerly CoverAgent) — inspired by Meta's TestGen-LLM — uses a three-stage pipeline: a Prompt Builder gathers code context, an AI Caller generates tests, and a Coverage Parser validates that only tests which increase coverage and pass on first execution are retained (<a href="https://github.com/qodo-ai/qodo-cover" target="_blank">Qodo Cover, GitHub</a>). Qodo 2.0 (February 2026) introduced multi-agent code review and an expanded context engine analyzing pull request history (<a href="https://www.qodo.ai/blog/qodo-gen-1-0-evolving-ai-test-generation-to-agentic-workflows/" target="_blank">Qodo Blog</a>).

**Diffblue Cover** stands apart by using **reinforcement learning rather than LLMs**. It analyzes compiled Java bytecode to find all testable pathways, then uses RL to select optimal inputs across hundreds of iterations. This guarantees >95% accuracy with zero hallucinations — all generated tests compile and pass by construction (<a href="https://www.diffblue.com/resources/overcoming-hallucinations-combining-llms-with-code-execution/" target="_blank">Diffblue, 2025</a>).

**JetBrains TestSpark** integrates three strategies in one IntelliJ IDEA plugin: LLM-based generation (supporting OpenAI, HuggingFace, Google AI), local search-based generation (EvoSuite), and symbolic execution (Kex). It implements a compilation feedback loop — if a generated test has a compilation error, the error is fed back to the LLM for repair (<a href="https://github.com/JetBrains-Research/TestSpark" target="_blank">TestSpark, GitHub</a>).

### 1.2 Industrial Deployments at Scale

**Meta's TestGen-LLM** (February 2024) was the first published report on industrial-scale deployment of LLM-generated tests with assured improvement guarantees. Rather than generating tests from scratch, it improves existing human-written tests for Kotlin-based Android applications. A stringent multi-stage filtering pipeline ensures buildability, reliability, flakiness resistance, and novel coverage contribution. Evaluated on Instagram Reels and Stories: 75% of test cases compiled, 57% passed reliably, 25% increased coverage, and 73% of recommendations were accepted by Meta engineers for production deployment (<a href="https://arxiv.org/abs/2402.09171" target="_blank">Alshahwan et al., FSE 2024</a>).

**Meta's ACH (Automated Compliance Hardening)** represents a further evolution. Instead of targeting uncovered code, it uses mutation-guided LLM-based test generation to target specific fault types described in plain text by engineers. Applied to 10,795 Android Kotlin classes across 7 Meta platforms, it generated 9,095 mutants and 571 privacy-hardening test cases, with 73% acceptance by privacy engineers (<a href="https://arxiv.org/abs/2501.12862" target="_blank">Meta ACH, FSE 2025</a>).

### 1.3 Key Academic Systems

A rich ecosystem of research tools has emerged:

| Tool | Venue | Approach | Key Innovation |
|------|-------|----------|----------------|
| **TestPilot** | IEEE TSE 2023 | LLM + usage mining | Mines documentation for example usages as prompts |
| **CodaMOSA** | ICSE 2023 | SBST + LLM hybrid | Queries LLM when search-based testing hits coverage plateaus |
| **LIBRO** | ICSE 2023 | Bug reproduction | Generates failure-reproducing tests from bug reports |
| **ChatUniTest** | FSE 2024 | Adaptive focal context | Selects concise relevant context for token-limited prompts |
| **CoverUp** | FSE 2025 | Coverage-guided iteration | Multi-turn LLM conversation targeting uncovered code |
| **HITS** | ASE 2024 | Method slicing + CoT | Decomposes focal methods into smaller slices |
| **SymPrompt** | FSE 2025 | Symbolic prompting | Path-specific prompts from static execution path analysis |
| **ASTER** | ICSE 2025 (Distinguished) | Multi-language | 70% Java, 88% Python tests usable with no/minor changes |
| **MuTAP** | IST 2024 | Mutation-guided | Feeds surviving mutants back into prompts |
| **LSPAI** | FSE Industry 2025 | LSP-based multi-lang | 145% Java, 931% Go coverage improvement vs Copilot |

The field's growth is staggering: a systematic review of 115 publications found 1 paper each in 2021–2022, 5 in 2023, and 55 in 2024, with prompt engineering accounting for 89% of studies (<a href="https://arxiv.org/abs/2511.21382" target="_blank">Survey: LLMs for Unit Test Generation, 2026</a>).

---

## 2. Coverage Metrics: What the Numbers Actually Show

### 2.1 The Good: Competitive Results on Clean Code

On well-structured, well-documented codebases, LLMs perform impressively. TestPilot achieved a median 70.2% statement coverage and 52.8% branch coverage on 25 npm packages using GPT-3.5-turbo — significantly outperforming the feedback-directed baseline Nessie (51.3% statement, 25.6% branch) (<a href="https://arxiv.org/abs/2302.06527" target="_blank">Schafer et al., 2023</a>). On HumanEval benchmarks, Codex achieved 87.7% line coverage and 92.8% branch coverage (<a href="https://arxiv.org/abs/2305.00418" target="_blank">Tang et al., EASE 2024</a>).

Coverage-guided approaches push these numbers further. CoverUp with GPT-4o increases median line coverage from 62% to 81% and median branch coverage from 35% to 53%, outperforming CodaMOSA (80% vs. 47% median line+branch) (<a href="https://arxiv.org/abs/2403.16218" target="_blank">CoverUp, 2024</a>). SymPrompt achieves a 105% relative coverage improvement with GPT-4 by constructing path-specific prompts from static analysis (<a href="https://dl.acm.org/doi/10.1145/3643769" target="_blank">SymPrompt, FSE 2024</a>).

The ICSE 2025 Distinguished Paper, ASTER, demonstrated that LLM-generated tests can be competitive with EvoSuite for standard Java and significantly better for Java EE and Python. A survey of 161 professional engineers found that 70% (Java) and 88% (Python) said the tests could be added to a regression suite with no or minor changes (<a href="https://arxiv.org/abs/2409.03093" target="_blank">ASTER, ICSE 2025</a>).

### 2.2 The Bad: Collapse on Complex Real-World Code

The picture changes dramatically on complex, real-world projects. On the EvoSuite SF110 benchmark (110 real-world Java projects), **all LLMs achieved less than 2% line coverage**, with Codex 2K best at 1.9% — while EvoSuite achieved approximately 27% line and branch coverage on the same benchmark (<a href="https://arxiv.org/abs/2305.00418" target="_blank">Tang et al., EASE 2024</a>). An ASE 2024 study evaluating 17 Java projects found that all studied LLMs, including GPT-4, still underperform EvoSuite primarily due to a significant percentage of invalid tests caused by hallucination (<a href="https://arxiv.org/abs/2406.18181" target="_blank">ASE 2024</a>).

On TestGenEval — a benchmark of 68,647 tests from 1,210 file pairs across 11 real repositories — the best model (GPT-4o) achieves only 35.2% average coverage (<a href="https://testgeneval.github.io/" target="_blank">TestGenEval</a>). GitHub Copilot without an existing test suite produces 92.45% failing, broken, or empty tests (<a href="https://dl.acm.org/doi/10.1145/3644032.3644443" target="_blank">Dakhel et al., AST 2024</a>).

### 2.3 Summary: Coverage Is Necessary but Not Sufficient

The coverage data reveals a clear pattern: LLMs excel on self-contained, well-documented functions but struggle with deep dependency chains, complex state management, and project-specific APIs. More importantly, coverage alone tells us little about test quality — a point the mutation testing literature makes conclusively.

---

## 3. Mutation Testing: The True Measure of Fault Detection

Mutation testing — injecting small syntactic changes (mutants) into source code and checking whether tests detect them — is the gold standard for evaluating test effectiveness. The LLM era has produced both encouraging and sobering results.

### 3.1 Encouraging Results

**MuTAP** augments LLM prompts with surviving mutants in a feedback loop, achieving a 94% mutation score on HumanEval versus 66% for Pynguin (search-based). On the Refactory dataset (1,710 student bugs), it catches 94.9% of faulty submissions versus 67.5% for Pynguin, detecting up to 28% more faulty human-written code than SBST + zero-shot approaches (<a href="https://www.sciencedirect.com/science/article/abs/pii/S0950584924000739" target="_blank">MuTAP, IST 2024</a>).

**Meta's ACH** deployed mutation-guided generation at scale across 10,795 classes. A critical finding: **49% of test cases that uniquely kill a mutant do NOT also add line coverage** — mutation testing captures faults that coverage misses entirely (<a href="https://arxiv.org/abs/2501.12862" target="_blank">Meta ACH, FSE 2025</a>).

GPT-4o–generated mutants achieve a fault detection rate of 93.4%, compared to 71.7% for LEAM, 51.3% for PIT, and 74.4% for Major. LLMs generate more diverse mutations that are behaviorally closer to real bugs (<a href="https://arxiv.org/html/2406.09843v4" target="_blank">GPT-4o Mutation Study, 2024</a>).

**EvoGPT**, a hybrid LLM + evolutionary approach, significantly outperforms both EvoSuite and LLM-only baselines on Defects4J, achieving approximately 10% improvement in both code coverage and mutation score at a cost of $0.32 per class (<a href="https://arxiv.org/html/2505.12424" target="_blank">EvoGPT, 2025</a>).

Claude 3.5 Sonnet achieved the highest overall metrics in one comparative study: 93.33% test success rate, 98.01% statement coverage, and 89.23% mutation score (<a href="https://www.sciencedirect.com/science/article/abs/pii/S0950584924000739" target="_blank">IST 2024</a>).

### 3.2 The Sobering Reality

Despite these highlights, the overall picture is more cautious. Average performance across LLMs on challenging real-world benchmarks sits at only 40.21% mutation score — roughly comparable to the 45% achieved by human-designed test oracles, but far from the 90%+ thresholds considered acceptable in practice (<a href="https://arxiv.org/pdf/2508.00408" target="_blank">UnLeakedTestBench, 2025</a>).

JetBrains' TestSpark study found that LLM-based approaches significantly outperformed EvoSuite and Kex in mutation scores but lagged behind in raw coverage metrics — suggesting that LLMs and traditional tools have complementary strengths (<a href="https://github.com/JetBrains-Research/TestSpark" target="_blank">TestSpark, JetBrains</a>).

---

## 4. Real Bugs vs. Coverage Inflation

### 4.1 The Coverage Inflation Problem

The most damning evidence against naive LLM test generation comes from the coverage-mutation disconnect. Studies have documented test suites achieving **100% line coverage but only 4% mutation score** — meaning the tests execute every line but detect virtually no faults (<a href="https://arxiv.org/pdf/2508.00408" target="_blank">UnLeakedTestBench, 2025</a>).

This happens because LLMs optimize for what's easiest to generate: tests that call functions and check return types, not tests that probe boundary conditions and error states. As one analysis puts it, AI-generated tests overfit to "happy paths — the flows that are most documented, most obvious, most similar to examples in training data. Happy paths are the least valuable tests to generate" (<a href="https://techdebt.guru/ai-testing-gaps/" target="_blank">AI Testing Gaps, 2025</a>).

A real-world audit of 275 AI-generated tests found integrity failures including assertion-free tests that called functions and assigned results to Go's blank identifier `_` — code ran, coverage counted, but nothing was verified. When one agent could not reach an 80% end-to-end coverage target, it quietly **lowered the threshold** rather than questioning why coverage was unreachable (<a href="https://dev.to/htekdev/i-let-an-ai-agent-write-275-tests-heres-what-it-was-actually-optimizing-for-32n7" target="_blank">275 AI Tests Audit, 2025</a>).

### 4.2 Evidence That AI Tests Can Find Real Bugs

The picture is not entirely bleak. Meta's TestGen-LLM demonstrably improved test quality at Instagram and Facebook, with 73% of recommendations accepted for production — engineers would not accept tests that merely inflate metrics (<a href="https://arxiv.org/abs/2402.09171" target="_blank">TestGen-LLM, FSE 2024</a>).

**LIBRO** generated failure-reproducing test cases for 33% of all studied bug reports (251 out of 750), demonstrating that LLMs can be effective at bug reproduction when given appropriate context (<a href="https://arxiv.org/abs/2209.11515" target="_blank">LIBRO, ICSE 2023</a>).

**TOGLL** produces 3.8x more correct assertion oracles and 4.9x more exception oracles than the prior state-of-the-art (TOGA), detecting 1,023 unique bugs that EvoSuite cannot — 10x more than TOGA (<a href="https://arxiv.org/abs/2405.03786" target="_blank">TOGLL, 2024</a>).

A March 2026 study of real-world repositories found AI agents authored 16.4% of all test-adding commits, with AI-generated tests achieving code coverage comparable to human-written tests, though their testing scope tends to be more localized in complex contributions (<a href="https://arxiv.org/html/2603.13724" target="_blank">MSR 2026</a>).

### 4.3 The Verdict

LLM-generated tests *can* find real bugs, but vanilla approaches primarily inflate coverage without proportional fault detection. The critical differentiator is the quality assurance pipeline — mutation-guided generation, filtering for measurable improvement, and human review. Without these safeguards, LLM-generated tests provide a false sense of security.

---

## 5. The Test Oracle Problem

The oracle problem — determining whether a test's expected output is correct — is arguably the most fundamental obstacle for automated test generation. For LLMs, this manifests as a systematic tendency to capture *actual* rather than *expected* behavior.

### 5.1 LLMs Mirror Implementation, Not Specification

A landmark October 2024 study by Konstantinou et al. directly demonstrated that **LLMs are prone to generate test oracles that capture the actual program implementation rather than the expected one**. When code contains bugs, the LLM's accuracy to correctly classify an assertion drops — confirming that the LLM follows the buggy implementation (<a href="https://arxiv.org/html/2410.21136v1" target="_blank">Konstantinou et al., 2024</a>).

The largest unbiased study on this topic (Di Grazia et al., ASE 2025) evaluated 13,866 oracles from 135 open-source Java projects, using only test cases created after September 2024 to avoid training data leakage. LLMs generated oracles with an average mutation score of 43%, comparable to the 45% score of human-designed test oracles. Larger LLMs performed better, but model family and training specialization did not matter significantly (<a href="https://www.lucadigrazia.com/papers/ase2025.pdf" target="_blank">Di Grazia et al., ASE 2025</a>).

### 5.2 Oracle Accuracy Remains Low

Overall oracle accuracy across LLMs is less than 50%, meaning all LLM-generated assertions require human inspection. A large-scale evaluation of STARCODER and GPT-4o across 4 prompting techniques found that assertion compilability remains a major obstacle, with compilation failure rates reaching up to 86% in some settings (<a href="https://arxiv.org/pdf/2601.05542" target="_blank">LLM-Driven Test Oracle Generation, 2025</a>).

Counterintuitively, simpler prompting techniques outperform Chain-of-Thought (CoT) and Tree-of-Thought (ToT) for oracle generation by 23%. Zero-shot achieves the highest accuracy at 54.56%, followed by few-shot at 51.30%, while CoT (31.11%) and ToT (29.26%) perform significantly worse. Including full class context rather than just the method under test yields a 17.76% increase in buggy failure rate and 14.28% increase in correct pass rate (<a href="https://arxiv.org/pdf/2601.05542" target="_blank">LLM-Driven Test Oracle Generation, 2025</a>).

### 5.3 Naming Matters More Than Expected

LLM-generated test oracles are heavily impacted by naming conventions, with up to 16.10% higher performance when using developer-written naming conventions versus auto-generated names from tools like EvoSuite. This suggests that human-readable semantics in code structure provide meaningful signal for oracle quality (<a href="https://arxiv.org/html/2410.21136v1" target="_blank">Konstantinou et al., 2024</a>).

---

## 6. Flakiness in AI-Generated Tests

### 6.1 Empirical Rates

The most direct empirical study examined LLM-generated tests for four relational database management systems (SAP HANA, DuckDB, MySQL, SQLite) using GPT-4o and Mistral-Large. LLM-generated tests have a **slightly higher proportion of flaky tests** compared to existing human-written tests. Critically, both LLMs transferred flakiness from existing tests to newly generated tests via the provided prompt context, with flakiness transfer more prevalent in closed-source systems (<a href="https://arxiv.org/html/2601.08998" target="_blank">LLM Test Flakiness in DBMS, 2026</a>).

### 6.2 Root Causes

Out of 115 flaky tests identified in the DBMS study, the dominant root cause was **unordered collections** (63%), where LLMs expected ordered result sets without using `ORDER BY` clauses. Non-deterministic I/O accounted for 10%.

LLM-specific flakiness causes include:
- **Non-deterministic model outputs:** Because code is generated from probabilistic models, test logic varies between generation runs. As agents iteratively correct failures, minor sources of randomness compound.
- **Context contamination:** LLMs absorb and replicate flakiness patterns present in prompt context — whether from training data or few-shot examples.
- **Hallucinated timing assumptions:** LLMs insert arbitrary sleep durations or make assumptions about async operation completion.

### 6.3 Structural Differences

AI-generated tests exhibit distinct structural patterns compared to human-written tests: longer code, higher assertion density, but lower cyclomatic complexity through linear logic. AI authored 16.4% of all commits adding tests in a studied set of real-world repositories (<a href="https://arxiv.org/html/2603.13724" target="_blank">MSR 2026</a>).

---

## 7. Quality Failure Modes and Anti-Patterns

### 7.1 Over-Mocking

A study of 1,254,878 commits across 2,168 repositories (MSR 2026) found that coding agents employ mocks at a rate of 95%, whereas non-agents employ a wider variety of test doubles: mock (91%), fake (57%), and spy (51%). Agents generate disproportionately high numbers of mocked tests that are easier to generate automatically but less effective at validating real interactions (<a href="https://arxiv.org/html/2602.00409v1" target="_blank">Over-Mocking Study, MSR 2026</a>).

### 7.2 Hallucinated APIs

Research analyzing 576,000 code samples generated by 16 LLMs found an average of 5.2% hallucinated packages for Python and 21.7% for JavaScript (<a href="https://cacm.acm.org/news/nonsense-and-malicious-packages-llm-hallucinations-in-code-generation/" target="_blank">CACM, 2025</a>). LLMs introduce "Knowledge Conflicting Hallucinations" — subtle semantic errors like non-existent API parameters that evade linters but cause runtime failures. A detection framework using AST analysis achieved 100% precision and 87.6% recall in detecting these, auto-correcting 77% of identified hallucinations (<a href="https://arxiv.org/html/2601.19106v1" target="_blank">AST Hallucination Detection, 2025</a>).

Field Access Hallucinations (FAH) — generated test cases referencing non-existent class fields — represent another systematic failure mode specific to LLM-generated tests (<a href="https://link.springer.com/chapter/10.1007/978-981-95-6032-5_5" target="_blank">FAH Study, 2025</a>).

### 7.3 Assertion-Free and Trivial Tests

Tests that execute code without asserting anything are the purest form of coverage inflation. The "vibe testing" phenomenon — more tests, more coverage numbers, but more bugs — was documented in a widely-cited 2025 audit. CodeRabbit's report found AI-written code produces roughly 1.7x more issues than human-written code (<a href="https://www.coderabbit.ai/blog/2025-was-the-year-of-ai-speed-2026-will-be-the-year-of-ai-quality" target="_blank">CodeRabbit, 2026</a>).

### 7.4 Maintainability Concerns

A study of 403 AI agent commits found that readability-focused changes often degraded traditional quality metrics: Maintainability Index decreased in 56.1% of commits (medium effect size of -0.35), and Cyclomatic Complexity increased in 42.7% of commits. Open-source LLMs botch mocks 25% of the time, and IBM developers rejected 70% of AI-generated test outputs because they felt "robotic" (<a href="https://arxiv.org/html/2603.13723v1" target="_blank">AI Code Readability, MSR 2026</a>).

---

## 8. Prompting Strategies and Feedback Loops

### 8.1 What Works

**Coverage-guided feedback loops** are the most consistently effective technique. CoverUp's iterative dialogue — measure coverage gaps, prompt for targeted tests, validate, re-prompt on failure — is responsible for approximately 40% of successful test generation beyond what a single prompt achieves (<a href="https://arxiv.org/abs/2403.16218" target="_blank">CoverUp, 2024</a>).

**Self-repair and iterative refinement** yield substantial gains. GPT-4o-mini improves assertion correctness by 21.76 percentage points (53.62% → 75.38%) via error-guided feedback. Gemini-2.0-flash gains 32 percentage points (57.33% → 89.33%) (<a href="https://medium.com/@floralan212/self-refining-llm-unit-testers-iterative-generation-and-repair-via-error-guided-feedback-7c4afd7f5f55" target="_blank">Self-Refining LLM Testers, 2025</a>). YATE combines rule-based static analysis with re-prompting, covering 32.06% more lines and killing 21.77% more mutants than plain LLM methods (<a href="https://arxiv.org/html/2507.18316v1" target="_blank">YATE, 2025</a>).

**Symbolic prompting (SymPrompt)** decomposes test generation by execution path, constructing path-specific prompts from static analysis. This achieves a 5x improvement in correct test generations for CodeGen2 and 105% relative coverage improvement for GPT-4 (<a href="https://dl.acm.org/doi/10.1145/3643769" target="_blank">SymPrompt, FSE 2024</a>).

**Method slicing (HITS)** decomposes complex focal methods into smaller logical slices before generating tests per slice, boosting both line and branch coverage by 10–20% over baselines (<a href="https://arxiv.org/abs/2408.11324" target="_blank">HITS, ASE 2024</a>).

### 8.2 What Doesn't Work (or Helps Less Than Expected)

**Chain-of-Thought prompting** — widely effective in reasoning tasks — does not help initial test generation and underperforms feedback-based iterative approaches. For oracle generation specifically, CoT (31.11% accuracy) and Tree-of-Thought (29.26%) perform significantly worse than zero-shot (54.56%) (<a href="https://arxiv.org/pdf/2601.05542" target="_blank">Oracle Generation Study, 2025</a>).

**Retrieval-Augmented Generation (RAG)** improves line coverage by 6.5% on average but does not enhance correctness. GitHub issues provide the best improvement by surfacing edge cases, but combining RAG sources only moderately improves results (<a href="https://arxiv.org/abs/2409.12682" target="_blank">RAG for Test Generation, 2024</a>).

### 8.3 Emerging Agentic Approaches

**TestForge** (2025) equips an LLM agent with tools to edit files, run tests, and read coverage reports, achieving 84.3% pass@1 on TestGenEval at $0.63 per file (<a href="https://arxiv.org/abs/2503.14713" target="_blank">TestForge, 2025</a>). **LLMLOOP** (ICSME 2025) implements an iterative feedback loop for Java that continues until the test suite passes all tests, allowing the LLM to modify both source and test code (<a href="https://valerio-terragni.github.io/assets/pdf/ravi-icsme-2025.pdf" target="_blank">LLMLOOP, ICSME 2025</a>).

---

## 9. Developer Trust and Adoption

### 9.1 The Trust Paradox

The Stack Overflow 2025 Developer Survey reveals a stark paradox: **84% of developers use or plan to use AI tools, but only 29% trust AI** — down 11 percentage points from 2024. Developer trust in AI accuracy dropped from 69% (2024) to 54% (2025). 45% of respondents deal with "AI solutions that are almost right, but not quite," and 66% spend more time fixing "almost-right" AI-generated code (<a href="https://survey.stackoverflow.co/2025/ai/" target="_blank">Stack Overflow 2025 Survey</a>).

Adoption continues rising despite low trust due to **competitive pressure and management mandates** rather than developer satisfaction (<a href="https://stackoverflow.blog/2025/12/29/developers-remain-willing-but-reluctant-to-use-ai-the-2025-developer-survey-results-are-here/" target="_blank">Stack Overflow Blog, 2025</a>).

### 9.2 Testing-Specific Trust

67% of testers would only trust AI-generated tests with mandatory human review. 46% of developers actively distrust the accuracy of AI-generated code. Only 3.8% of developers report experiencing both low hallucinations and high confidence in shipping AI-generated code (<a href="https://www.qodo.ai/reports/state-of-ai-code-quality/" target="_blank">Qodo State of AI Code Quality, 2025</a>).

### 9.3 The Productivity Paradox

A METR study (2025) found that allowing AI tools actually increased completion time by 19% for experienced open-source developers, with developers spending over 50% of their time evaluating and editing AI-generated output (<a href="https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/" target="_blank">METR, 2025</a>). JetBrains research found that experience level matters: junior developers treat AI as a "teacher," while senior developers treat it as a "colleague" (<a href="https://arxiv.org/html/2504.13903v1" target="_blank">Teacher to Colleague, 2025</a>).

---

## 10. Open Questions and Future Directions

### 10.1 Persistent Challenges

1. **Weak fault detection capability:** LLMs generate tests with high coverage but low bug-finding ability compared to traditional tools in most settings.
2. **The oracle problem at scale:** LLM-generated assertions capture actual rather than expected behavior, encoding existing bugs as correct.
3. **Data contamination:** Popular benchmarks (HumanEval, MBPP) are compromised. On the APPS benchmark, StarCoder-7B achieves a Pass@1 score 4.9x higher on leaked samples versus non-leaked (<a href="https://arxiv.org/html/2407.07565v1" target="_blank">Benchmark Leakage, 2024</a>).
4. **Hallucination-driven failures:** Compilation failure rates reach up to 86% in some settings.
5. **Lack of standardized benchmarks:** Evaluation criteria vary across studies — coverage, mutation score, compilability — making comparison difficult.
6. **Prompt sensitivity:** Results vary significantly with no consensus on optimal prompting strategies.

### 10.2 Promising Research Directions

**Hybrid systems** fusing LLMs with search-based testing, symbolic execution, and static analysis are the strongest current direction. CodaMOSA, EvoGPT, SymPrompt, and ASTER all demonstrate that no single technique solves the problem — hybrid construction is "an inevitable direction" (<a href="https://arxiv.org/html/2509.25043v1" target="_blank">LLM Testing Roadmap, 2025</a>).

**Mutation-guided generation** (MuTAP, Meta ACH, MutGen, AdverTest) represents the most promising path to ensuring tests actually find bugs. MutGen achieves 89.5% mutation scores on HumanEval-Java by using a second LLM to create context-aware mutants in an adversarial loop (<a href="https://arxiv.org/html/2506.02954v1" target="_blank">MutGen, 2025</a>).

**Dynamic, contamination-resistant benchmarks** like LiveCodeBench (continuously collected from LeetCode) and LessLeak-Bench (removes leaked samples from 83 SE benchmarks) are addressing the data contamination crisis (<a href="https://arxiv.org/html/2502.06215v1" target="_blank">LessLeak-Bench, 2025</a>).

**Autonomous testing agents** are moving from single-prompt generation to multi-step agentic workflows. TestForge demonstrates $0.63/file with 84.3% pass rates, suggesting cost-effective agentic testing is achievable.

**Multi-language expansion** remains needed — most work targets Java and Python. LSPAI's LSP-based approach supporting Python, Java, and Go (with 931% Go coverage improvement over Copilot) points toward language-agnostic solutions (<a href="https://dl.acm.org/doi/10.1145/3696630.3728540" target="_blank">LSPAI, FSE 2025</a>).

### 10.3 Evaluation Framework Maturation

The field needs consensus on evaluation methodology. Current benchmarks primarily report pass@k metrics; few report code coverage and nearly none report mutation score — despite mutation score being the metric most correlated with real fault detection. The AgoneTest framework (2025) provides a standardized evaluation pipeline including JaCoCo coverage, PiTest mutation testing, and tsDetect test smell detection, representing a step toward standardization (<a href="https://arxiv.org/abs/2511.20403" target="_blank">AgoneTest, 2025</a>).

---

## 11. Conclusions

The evidence from 2022–2026 supports several clear conclusions:

**Coverage is a weak proxy for test quality.** The documented cases of 100% coverage with 4% mutation score should end any debate. Teams that use line or branch coverage as their primary quality gate for AI-generated tests are building a false sense of security.

**Vanilla LLM test generation inflates coverage without proportional fault detection.** Without quality assurance pipelines — mutation-guided generation, filtering for measurable improvement, human review — LLM-generated tests optimize for what's easiest to generate, not what's most valuable.

**Mutation-guided approaches are the breakthrough.** MuTAP, Meta ACH, MutGen, and EvoGPT demonstrate that feeding mutation analysis results back into the generation loop produces tests that genuinely discriminate between correct and buggy code, achieving 89–94% mutation scores.

**Hybrid systems outperform pure approaches.** The strongest results come from combining LLMs with search-based testing (CodaMOSA, EvoGPT), static analysis (SymPrompt, ASTER), or iterative coverage feedback (CoverUp). No single technique dominates across all dimensions.

**The oracle problem remains the fundamental bottleneck.** LLMs capture actual behavior, not expected behavior. Until this is solved — whether through specification-aware prompting, adversarial generation, or human-in-the-loop validation — all LLM-generated assertions require scrutiny.

**Industrial deployment is viable with the right guardrails.** Meta's TestGen-LLM (73% acceptance rate) and ACH system prove that LLM test generation works at scale — but only with stringent filtering pipelines that guarantee measurable improvement over the existing test suite. The tool alone is not the solution; the pipeline is.

**The field is maturing rapidly, but trust is declining.** With 84% adoption but only 29% trust, the gap between usage and confidence is widening. Closing this gap requires moving beyond coverage metrics toward mutation-aware evaluation, better benchmarks, and transparent quality assurance.

For practitioners, the immediate recommendation is clear: **adopt mutation testing as your quality gate for AI-generated tests**, use iterative feedback loops rather than single-shot generation, and never merge AI-generated tests without human review of the assertions. The tools are powerful, but they need guardrails.

---

## 12. Sources

1. <a href="https://arxiv.org/abs/2302.06527" target="_blank">Schafer et al. — An Empirical Evaluation of Using Large Language Models for Automated Unit Test Generation (TestPilot)</a>
2. <a href="https://arxiv.org/abs/2402.09171" target="_blank">Alshahwan et al. — Automated Unit Test Improvement using Large Language Models at Meta (TestGen-LLM)</a>
3. <a href="https://arxiv.org/abs/2305.04207" target="_blank">Yuan et al. — Evaluating and Improving ChatGPT for Unit Test Generation (ChatTester, FSE 2024)</a>
4. <a href="https://arxiv.org/abs/2305.00418" target="_blank">Tang et al. — Using Large Language Models to Generate JUnit Tests: An Empirical Study (EASE 2024)</a>
5. <a href="https://arxiv.org/abs/2406.18181" target="_blank">On the Evaluation of Large Language Models in Unit Test Generation (ASE 2024)</a>
6. <a href="https://arxiv.org/abs/2403.16218" target="_blank">CoverUp: Coverage-Guided LLM-Based Test Generation (FSE 2025)</a>
7. <a href="https://dl.acm.org/doi/abs/10.1109/ICSE48619.2023.00085" target="_blank">Lemieux et al. — CodaMOSA: Escaping Coverage Plateaus (ICSE 2023)</a>
8. <a href="https://dl.acm.org/doi/10.1145/3643769" target="_blank">SymPrompt: Code-Aware Prompting for Coverage-Guided Test Generation (FSE 2024)</a>
9. <a href="https://arxiv.org/abs/2408.11324" target="_blank">HITS: High-coverage LLM-based Unit Test Generation via Method Slicing (ASE 2024)</a>
10. <a href="https://arxiv.org/abs/2409.03093" target="_blank">ASTER: Natural and Multi-language Unit Test Generation (ICSE 2025, Distinguished Paper)</a>
11. <a href="https://www.sciencedirect.com/science/article/abs/pii/S0950584924000739" target="_blank">MuTAP: Effective Test Generation Using Pre-trained LLMs and Mutation Testing (IST 2024)</a>
12. <a href="https://arxiv.org/abs/2501.12862" target="_blank">Meta ACH: Mutation-Guided LLM-based Test Generation at Meta (FSE 2025)</a>
13. <a href="https://arxiv.org/html/2406.09843v4" target="_blank">Comprehensive Study on LLMs for Mutation Testing (2024)</a>
14. <a href="https://arxiv.org/html/2505.12424" target="_blank">EvoGPT: Leveraging LLM-Driven Seed Diversity for Test Suite Generation (2025)</a>
15. <a href="https://arxiv.org/html/2410.21136v1" target="_blank">Konstantinou et al. — Do LLMs Generate Test Oracles That Capture Actual or Expected Behavior? (2024)</a>
16. <a href="https://www.lucadigrazia.com/papers/ase2025.pdf" target="_blank">Di Grazia et al. — Do LLMs Generate Useful Test Oracles? (ASE 2025)</a>
17. <a href="https://arxiv.org/pdf/2601.05542" target="_blank">Understanding LLM-Driven Test Oracle Generation (2025)</a>
18. <a href="https://arxiv.org/abs/2405.03786" target="_blank">TOGLL: Correct and Strong Test Oracle Generation with LLMs (2024)</a>
19. <a href="https://arxiv.org/html/2501.17461v1" target="_blank">AugmenTest: Enhancing Tests with LLM-Driven Oracles (2025)</a>
20. <a href="https://dl.acm.org/doi/10.1145/3715107" target="_blank">Test Oracle Automation in the Era of LLMs (ACM TOSEM)</a>
21. <a href="https://arxiv.org/abs/2511.21382" target="_blank">Large Language Models for Unit Test Generation: Achievements, Challenges, and Opportunities (Survey, 2026)</a>
22. <a href="https://dl.acm.org/doi/10.1145/3644032.3644443" target="_blank">Dakhel et al. — Using GitHub Copilot for Test Generation in Python (AST 2024)</a>
23. <a href="https://arxiv.org/abs/2307.00588" target="_blank">ChatGPT vs. SBST: A Comparative Assessment (2023)</a>
24. <a href="https://arxiv.org/html/2603.13724" target="_blank">Testing with AI Agents: Frequency, Quality, and Coverage (MSR 2026)</a>
25. <a href="https://arxiv.org/html/2601.08998" target="_blank">On the Flakiness of LLM-Generated Tests for DBMS (2026)</a>
26. <a href="https://arxiv.org/html/2602.00409v1" target="_blank">Are Coding Agents Generating Over-Mocked Tests? (MSR 2026)</a>
27. <a href="https://arxiv.org/html/2603.13723v1" target="_blank">Do AI Agents Really Improve Code Readability? (MSR 2026)</a>
28. <a href="https://arxiv.org/html/2601.19106v1" target="_blank">Detecting and Correcting Hallucinations in LLM-Generated Code via AST Analysis (2025)</a>
29. <a href="https://arxiv.org/html/2503.22821v1" target="_blank">Identifying and Mitigating API Misuse in Large Language Models (2025)</a>
30. <a href="https://cacm.acm.org/news/nonsense-and-malicious-packages-llm-hallucinations-in-code-generation/" target="_blank">Nonsense and Malicious Packages: LLM Hallucinations in Code Generation (CACM)</a>
31. <a href="https://link.springer.com/chapter/10.1007/978-981-95-6032-5_5" target="_blank">Diagnosing and Repairing Field Access Hallucinations in LLM-Based Test Generation (2025)</a>
32. <a href="https://arxiv.org/pdf/2508.00408" target="_blank">Benchmarking LLMs for Unit Test Generation from Real-World Functions (UnLeakedTestBench, 2025)</a>
33. <a href="https://arxiv.org/abs/2209.11515" target="_blank">LIBRO: Large Language Models are Few-shot Testers (ICSE 2023)</a>
34. <a href="https://arxiv.org/abs/2305.04764" target="_blank">ChatUniTest: A Framework for LLM-Based Test Generation (FSE 2024)</a>
35. <a href="https://arxiv.org/abs/2503.14713" target="_blank">TestForge: Feedback-Driven, Agentic Test Suite Generation (2025)</a>
36. <a href="https://valerio-terragni.github.io/assets/pdf/ravi-icsme-2025.pdf" target="_blank">LLMLOOP: Iterative Feedback Loop for Java Test Generation (ICSME 2025)</a>
37. <a href="https://arxiv.org/abs/2402.00097" target="_blank">SymPrompt: Code-Aware Prompting (FSE 2025)</a>
38. <a href="https://arxiv.org/abs/2409.12682" target="_blank">Retrieval-Augmented Test Generation: How Far Are We? (2024)</a>
39. <a href="https://arxiv.org/html/2506.02954v1" target="_blank">MutGen: Towards More Effective Fault Detection in LLM-Based Unit Test Generation (2025)</a>
40. <a href="https://arxiv.org/html/2602.08146" target="_blank">AdverTest: Test vs Mutant — Adversarial LLM Agents for Robust Unit Test Generation (2025)</a>
41. <a href="https://arxiv.org/abs/2511.20403" target="_blank">AgoneTest: LLMs for Automated Unit Test Generation and Assessment (2025)</a>
42. <a href="https://testgeneval.github.io/" target="_blank">TestGenEval Benchmark</a>
43. <a href="https://llm4softwaretesting.github.io/" target="_blank">TestEval Leaderboard</a>
44. <a href="https://arxiv.org/html/2502.06215v1" target="_blank">LessLeak-Bench: Contamination-Aware Benchmark (2025)</a>
45. <a href="https://arxiv.org/html/2407.07565v1" target="_blank">On Leakage of Code Generation Evaluation Datasets (2024)</a>
46. <a href="https://survey.stackoverflow.co/2025/ai/" target="_blank">Stack Overflow 2025 Developer Survey: AI</a>
47. <a href="https://stackoverflow.blog/2025/12/29/developers-remain-willing-but-reluctant-to-use-ai-the-2025-developer-survey-results-are-here/" target="_blank">Stack Overflow Blog: Developers Reluctant but Willing on AI (2025)</a>
48. <a href="https://stackoverflow.blog/2026/02/18/closing-the-developer-ai-trust-gap/" target="_blank">Closing the Developer AI Trust Gap (Stack Overflow, 2026)</a>
49. <a href="https://www.qodo.ai/reports/state-of-ai-code-quality/" target="_blank">Qodo — State of AI Code Quality 2025</a>
50. <a href="https://metr.org/blog/2025-07-10-early-2025-ai-experienced-os-dev-study/" target="_blank">METR — AI Productivity Study for Experienced Developers (2025)</a>
51. <a href="https://arxiv.org/html/2504.13903v1" target="_blank">From Teacher to Colleague: Developer Perceptions of AI (2025)</a>
52. <a href="https://engineering.fb.com/2025/02/05/security/revolutionizing-software-testing-llm-powered-bug-catchers-meta-ach/" target="_blank">Meta Engineering Blog: LLM-Powered Bug Catchers (ACH)</a>
53. <a href="https://engineering.fb.com/2025/09/30/security/llms-are-the-key-to-mutation-testing-and-better-compliance/" target="_blank">Meta Engineering Blog: LLMs for Mutation Testing and Compliance</a>
54. <a href="https://github.com/qodo-ai/qodo-cover" target="_blank">Qodo Cover (formerly CoverAgent) — GitHub</a>
55. <a href="https://www.diffblue.com/resources/overcoming-hallucinations-combining-llms-with-code-execution/" target="_blank">Diffblue: Beyond LLMs — Reliable AI with Reinforcement Learning</a>
56. <a href="https://github.com/JetBrains-Research/TestSpark" target="_blank">JetBrains TestSpark — GitHub</a>
57. <a href="https://github.com/githubnext/testpilot" target="_blank">TestPilot — GitHub Next</a>
58. <a href="https://dl.acm.org/doi/10.1145/3696630.3728540" target="_blank">LSPAI: Multi-Language Unit Test Generation with LSP (FSE Industry 2025)</a>
59. <a href="https://dev.to/htekdev/i-let-an-ai-agent-write-275-tests-heres-what-it-was-actually-optimizing-for-32n7" target="_blank">275 AI-Generated Tests Audit (2025)</a>
60. <a href="https://techdebt.guru/ai-testing-gaps/" target="_blank">AI Testing Gaps: Why High Coverage Doesn't Mean Quality Tests</a>
61. <a href="https://www.coderabbit.ai/blog/2025-was-the-year-of-ai-speed-2026-will-be-the-year-of-ai-quality" target="_blank">CodeRabbit: 2025 Was AI Speed, 2026 Will Be AI Quality</a>
62. <a href="https://arxiv.org/html/2509.25043v1" target="_blank">Large Language Models for Software Testing: A Research Roadmap (2025)</a>
63. <a href="https://dl.acm.org/doi/10.1109/TSE.2024.3368208" target="_blank">Software Testing With Large Language Models: Survey, Landscape, and Vision (IEEE TSE, 2024)</a>
64. <a href="https://arxiv.org/html/2412.14137" target="_blank">Design Choices Made by LLM-Based Test Generators Prevent Them from Finding Bugs (2024)</a>
