# Harness Design for Long-Running Agent Coding: A Deep Dive

## Executive Summary

As AI coding agents scale from single-turn code completion to multi-hour autonomous application development, the **harness**—the orchestration infrastructure surrounding the model—has emerged as the primary determinant of output quality. Anthropic's engineering team demonstrated this through a GAN-inspired multi-agent architecture (Planner → Generator → Evaluator) that produced fully functional full-stack applications where solo agents consistently failed. A retro game maker built by the harness in 6 hours ($200) delivered working physics, AI integration, and coherent UI, while a solo agent's 20-minute, $9 attempt produced broken core mechanics. This report examines the architecture patterns, context management strategies, evaluator design, cost tradeoffs, and evolving landscape of harness engineering for long-running coding agents.

---

## Table of Contents

1. [Introduction: The Problem Space](#1-introduction-the-problem-space)
2. [The Generator-Evaluator Architecture](#2-the-generator-evaluator-architecture)
3. [The Planner-Generator-Evaluator Pipeline](#3-the-planner-generator-evaluator-pipeline)
4. [Context Management for Long-Running Tasks](#4-context-management-for-long-running-tasks)
5. [Evaluator Design and Quality Assurance](#5-evaluator-design-and-quality-assurance)
6. [Cost and Performance Tradeoffs](#6-cost-and-performance-tradeoffs)
7. [Comparison with Other Multi-Agent Frameworks](#7-comparison-with-other-multi-agent-frameworks)
8. [The Evolving Harness: How Design Shifts with Model Capability](#8-the-evolving-harness-how-design-shifts-with-model-capability)
9. [Conclusions](#9-conclusions)
10. [Sources](#10-sources)

---

## 1. Introduction: The Problem Space

Traditional approaches to agentic coding hit performance ceilings that cannot be overcome by model improvements alone. Two persistent failure modes define this ceiling:

### 1.1 Context Coherence Degradation

Large language models exhibit measurable performance degradation as their context windows fill—a phenomenon researchers have formally studied as **"context rot."** Chroma's 2025 research tested 18 frontier models (including GPT-4.1, Claude Opus 4, and Gemini 2.5) and found that <a href="https://www.trychroma.com/research/context-rot" target="_blank">every model exhibited performance degradation at every input length increment tested</a>. Performance drops exceeding 50% were observed well before context windows were technically exhausted, with the "effective context window" often being much smaller than the advertised token limit.

For coding agents, this manifests as what Anthropic's engineers termed **"context anxiety"**—models prematurely concluding work as they approach perceived context limits. Earlier Claude models (like Sonnet 4.5) were particularly susceptible, often <a href="https://www.anthropic.com/engineering/harness-design-long-running-apps" target="_blank">declaring projects finished without accurately assessing remaining work</a>.

### 1.2 Self-Evaluation Bias

When asked to evaluate their own outputs, AI agents consistently overestimate quality. As Anthropic's team observed, agents "respond by confidently praising the work even when, to a human observer, the quality is obviously mediocre." This makes single-agent systems fundamentally unable to perform reliable quality assurance on their own code—a critical flaw for long-running tasks where errors compound.

### 1.3 The Harness as the Solution

These twin problems cannot be solved by improving model capability alone. They require **architectural solutions**—systems that manage context across sessions, separate generation from evaluation, and provide external feedback loops. This infrastructure is what the field now calls the **agent harness**: the complete system that governs how the agent operates, including prompts, tool integrations, agent collaboration structures, feedback mechanisms, and everything external to the model itself.

As one analysis puts it: <a href="https://businessengineer.ai/p/the-harness-as-the-agentic-moat" target="_blank">"The shift from scaling to deployment is the shift from 'how good is the model?' to 'how well can you harness it?'"</a>

---

## 2. The Generator-Evaluator Architecture

### 2.1 GAN-Inspired Design

The central architectural innovation in Anthropic's harness draws inspiration from **Generative Adversarial Networks (GANs)**. In traditional GANs, a generator creates outputs while a discriminator evaluates them, with both improving through adversarial pressure. Applied to coding agents, this principle separates the act of code generation from the act of code evaluation into distinct agents with <a href="https://www.freecodecamp.org/news/how-to-apply-gan-architecture-to-multi-agent-code-generation/" target="_blank">separate context windows, preventing inherited blind spots</a>.

The key insight: **"the same context that created the code is the one evaluating it"** in single-agent systems, so there is no adversarial pressure or fresh perspective to catch issues like placeholder tests, phantom dependencies, or ambiguous implementations.

### 2.2 How It Works

The generator-evaluator pattern structures the development pipeline so that review is architecturally mandatory:

| Component | Role | Key Behavior |
|-----------|------|-------------|
| **Generator** | Produces code, implements features | Works in structured sprints, follows conventions |
| **Evaluator** | Tests and grades output | Uses browser automation, grades against criteria |
| **Orchestrator** | Manages flow | Does not advance pipeline without evaluator approval |

The evaluator operates with **fresh context**, independent from the generator's reasoning process. This enables genuinely skeptical assessment—the evaluator has no investment in the code it's reviewing and no knowledge of the shortcuts or trade-offs the generator made during implementation.

### 2.3 The Iteration Loop

Rather than a single pass, the generator-evaluator pair operates in iterative cycles:

1. Generator produces output (code, design, feature implementation)
2. Evaluator assesses against predefined criteria
3. If criteria are not met, evaluator provides detailed feedback
4. Generator receives feedback and revises
5. Loop continues until quality thresholds are met or iteration cap is reached

This pattern is also known across the industry as the <a href="https://docs.google.com/architecture/choose-design-pattern-agentic-ai-system" target="_blank">evaluator-optimizer, generator-verifier, critic loop, or reflection loop</a>. Anthropic's own "Building Effective Agents" guide identifies the <a href="https://www.anthropic.com/research/building-effective-agents" target="_blank">evaluator-optimizer workflow</a> as particularly effective "when there are clear evaluation criteria and when iterative refinement provides measurable value."

### 2.4 Convergence Controls

Uncontrolled iteration loops risk infinite refinement and runaway costs. Practical implementations require:

- **Iteration caps**: Maximum 3-5 iterations per feedback loop
- **Token budgets**: Roughly 50k tokens per phase with hard ceilings
- **Explicit signal protocols**: State transitions use clear signals like `PLAN_APPROVED` or `NO-GO`
- **Fallback behavior**: When caps are reached, the system either escalates to human review or returns the best result with a quality warning

### 2.5 Frontend Design Application

Anthropic first applied the generator-evaluator pattern to **frontend design quality**, transforming subjective assessment into measurable standards through four grading criteria:

- **Design quality**: Coherence across visual elements
- **Originality**: Detection of generic patterns and AI-generated defaults
- **Craft**: Technical execution (typography, spacing, color harmony)
- **Functionality**: Usability independent of aesthetics

The evaluator used <a href="https://playwright.dev/" target="_blank">Playwright</a> to interact with live pages rather than reviewing static code. Over 5-15 iterations, generators produced increasingly distinctive designs, occasionally pivoting entirely toward novel aesthetic directions that would never emerge from a single-pass approach.

---

## 3. The Planner-Generator-Evaluator Pipeline

### 3.1 The Three-Agent Architecture

Building on the generator-evaluator foundation, Anthropic developed a **three-agent pipeline** that addressed distinct gaps in autonomous application development:

#### The Planner

The Planner agent was added to address a critical gap: when given raw prompts, models would **underscope tasks**—starting to build before fully thinking through what they were making, producing thin, feature-poor applications. The Planner:

- Expands brief user prompts into detailed product specifications
- Emphasizes deliverables and ambitious scope over implementation details
- Identifies opportunities to integrate AI features naturally
- Produces specifications covering 16+ features across multiple functional areas

The Planner focuses on **what** to build, not **how**—intentionally avoiding granular technical details that could cascade errors into implementation.

#### The Generator

The Generator implements features systematically using a full-stack technology stack (React, Vite, FastAPI, PostgreSQL). Key behaviors:

- Works in **structured sprints**, targeting one feature set at a time
- **Negotiates sprint contracts** with the Evaluator before writing code
- Performs self-evaluation before handing off to QA
- Maintains clean, committable code at each sprint boundary

#### The Evaluator

The Evaluator represents the most innovative component. Rather than reviewing static code, it:

- Uses **Playwright browser automation** to interact with the live running application
- Clicks through features, tests API endpoints, and probes database states
- Grades each sprint against predefined criteria: product depth, functionality, visual design, and code quality
- Provides detailed, actionable feedback when sprints fail quality thresholds

### 3.2 Sprint Contracts

A crucial innovation is the **sprint contract**—a negotiated agreement between the Generator and Evaluator before any code is written for a given sprint. The contract:

- Defines what "done" looks like for each deliverable
- Specifies verification methods the Evaluator will use
- Bridges the gap between high-level user stories and testable implementation
- Prevents the Generator from declaring work complete prematurely

This is analogous to the <a href="https://www.anthropic.com/engineering/harness-design-long-running-apps" target="_blank">definition-of-done practice in agile software teams</a>, but enforced architecturally rather than through team norms.

### 3.3 Inter-Agent Communication

Communication between agents flows through **structured artifacts** rather than direct message passing:

| Artifact | Purpose | Format |
|----------|---------|--------|
| Product specification | Planner → Generator | Detailed document with features, user stories |
| Sprint contract | Generator ↔ Evaluator | Negotiated deliverables and verification criteria |
| Sprint results | Generator → Evaluator | Committed code + running application |
| Feedback report | Evaluator → Generator | Graded assessment with specific failure details |
| Progress tracking | All agents | Git history, progress files |

This approach reflects a broader principle in multi-agent systems: <a href="https://addyosmani.com/blog/code-agent-orchestra/" target="_blank">using artifact systems where specialized agents create outputs that persist independently</a>, with lightweight references passing between coordinator and workers. File-based coordination minimizes the "game of telephone" effect where information degrades through multiple agent handoffs.

### 3.4 Results: Retro Game Maker

The three-agent pipeline was tested by building a 2D retro game maker:

| Configuration | Duration | Cost | Outcome |
|---------------|----------|------|---------|
| Solo agent (Opus 4.5) | 20 min | $9 | Superficially functional UI; core gameplay mechanics broken |
| Full harness (Opus 4.5) | 6 hr | $200 | Polished application with working physics, AI content generation, coherent visual design |

The solo version produced code that looked reasonable on inspection but failed fundamental end-to-end tests. The harness-generated version represented a **qualitative leap**—not just more code, but fundamentally more reliable and feature-complete software.

---

## 4. Context Management for Long-Running Tasks

### 4.1 The Core Challenge

Long-running coding tasks inevitably exceed a single context window. When agents must work across multiple sessions, each new session starts with no memory of previous work—creating coordination problems <a href="https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents" target="_blank">similar to engineering teams working shifts without handoff documentation</a>.

Two primary failure patterns emerge:

- **Over-ambition**: Agents attempt to complete entire projects at once, exhausting context mid-implementation and leaving features half-built
- **Premature completion**: New agent instances observe partial progress and incorrectly declare the task finished

### 4.2 Context Resets vs. Compaction

Anthropic's research identified a critical finding: **complete context resets outperform summarization** for models exhibiting context anxiety. While the Claude Agent SDK includes context compaction capabilities, compaction alone proved insufficient—it doesn't always pass perfectly clear instructions to the next agent.

The solution combines both approaches strategically:

| Strategy | When to Use | Tradeoffs |
|----------|-------------|-----------|
| **Full context reset** | Between major work sessions; with models prone to context anxiety | Loses nuanced context but prevents degradation |
| **Context compaction** | Within a session approaching limits; with newer models less prone to anxiety | Preserves some context but may introduce confusion |
| **Structured artifacts** | Always, as complement to either approach | Adds overhead but provides reliable state transfer |

Opus 4.6 largely eliminated context anxiety behavior, allowing the removal of aggressive context reset strategies. But the principle remains: <a href="https://www.working-ref.com/en/reference/anthropic-harness-design-philosophy-evolution" target="_blank">context management strategy must be calibrated to the specific model's behavior</a>.

### 4.3 The Initializer-Coding Agent Pattern

Anthropic's earlier harness (pre-three-agent architecture) solved multi-session coordination with a dual-agent approach:

**Initializer Agent** (first session only):
- Creates an `init.sh` script for development environment setup
- Establishes a `claude-progress.txt` file for session-to-session work tracking
- Generates a comprehensive **JSON feature list** with 200+ specific, testable requirements, all marked as `passes: false`
- Makes initial git commits documenting the project setup

**Coding Agent** (subsequent sessions):
- Reads progress files and git history immediately upon starting
- Works on a **single feature** per session
- Commits changes with descriptive messages
- Updates progress documentation before finishing
- Leaves code in mergeable, well-documented states

The JSON feature list proved particularly important. Using JSON format rather than Markdown was more resilient—<a href="https://www.zenml.io/llmops-database/long-running-agent-harness-for-multi-context-software-development" target="_blank">models less frequently corrupt structured data formats</a>. Marking all features as "failing" prevented agents from one-shotting applications and ensured incremental progress.

### 4.4 Session Startup Protocol

Effective agents follow a structured onboarding ritual at the start of each session:

1. Execute `pwd` to confirm working directory
2. Read git logs and progress files
3. Select the next highest-priority incomplete feature
4. Execute `init.sh` to start the development server
5. Perform basic end-to-end verification before starting new work

This final verification step—testing core functionality before implementing new features—prevented cascading failures from broken foundations. It costs tokens but saves far more by catching regressions early.

### 4.5 Advanced Context Engineering

Research on terminal-native coding agents reveals five integrated strategies for managing context across long sessions, as documented in <a href="https://arxiv.org/html/2603.05344v1" target="_blank">a comprehensive study of AI coding agent architecture</a>:

1. **Dynamic system prompt construction**: Modular prompt sections assembled conditionally by priority, supporting provider-specific variants
2. **Tool result optimization**: Intelligent summarization per tool type, large output offloading, and truncation hints to prevent context bloat
3. **Dual-memory architecture**: Episodic memory (long-term, cross-session) plus working memory (current conversation), both injected strategically
4. **Context-aware reminders**: Event-driven injections that counteract instruction fade-out in long conversations, triggering on specific conditions
5. **Adaptive compaction**: Progressive five-stage compression as token budget depletes—from summarization through selective truncation to emergency full-conversation compression

### 4.6 Context Rot: The Quantitative Reality

Chroma's research provides hard numbers on context degradation:

- **All 18 tested frontier models** exhibit context rot at every input length increment
- Performance degradation accelerates beyond ~30,000 tokens even in models with much larger windows
- The "effective context window" is often far smaller than the advertised limit (< 256k tokens for most models)
- **Claude models exhibit the lowest hallucination rates** among tested models and tend to abstain when uncertain
- **GPT models show the highest hallucination rates**, generating confident but incorrect responses under distracting context

Counterintuitively, models perform better on **shuffled haystacks** (random sentence reordering) than logically coherent ones, suggesting structural patterns influence attention mechanisms at longer lengths.

For practitioners, this means: <a href="https://blog.jetbrains.com/research/2025/12/efficient-context-management/" target="_blank">isolating search into subagents with their own context windows, returning precise results rather than whole files, discarding exploration traces, using compact diffs, and applying proactive compression</a>.

---

## 5. Evaluator Design and Quality Assurance

### 5.1 Why Separate Evaluation Matters

The generator-evaluator separation is not merely an architectural preference—it addresses a fundamental limitation. In production AI systems, <a href="https://www.infoq.com/articles/evaluating-ai-agents-lessons-learned/" target="_blank">using a separate judge model to reduce self-grading bias is considered a best practice</a>. When the same model generates and evaluates, it cannot provide the adversarial pressure needed to catch its own blind spots.

### 5.2 Grading Criteria Design

Effective evaluators require structured, specific rubrics rather than vague quality assessments. Anthropic's frontend design evaluator used four distinct dimensions:

| Criterion | What It Measures | Why It Matters |
|-----------|-----------------|----------------|
| Design quality | Coherence across visual elements | Catches disjointed UI that "works" but looks amateur |
| Originality | Absence of generic AI patterns | Detects copy-paste templates and default aesthetics |
| Craft | Typography, spacing, color harmony | Evaluates professional-level polish |
| Functionality | Usability independent of appearance | Ensures the app actually works for users |

The principle extends to code evaluation: <a href="https://www.confident-ai.com/blog/definitive-ai-agent-evaluation-guide" target="_blank">grade each dimension with an isolated judge rather than using one evaluation to assess everything</a>. Specific, measurable standards (not "good code" but "handles error states for all API calls") produce more reliable assessment.

### 5.3 Browser-Based Testing

A defining feature of Anthropic's evaluator is its use of **Playwright browser automation** for end-to-end testing. Rather than reviewing code statically, the evaluator:

- Opens the running application in a real browser
- Navigates through user workflows
- Tests API endpoints and database states
- Takes screenshots for visual verification
- Interacts with UI elements as a real user would

This approach catches bugs that unit tests and code review consistently miss. Anthropic found that <a href="https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents" target="_blank">agents consistently failed to identify end-to-end bugs without browser automation tools, even when unit tests passed</a>.

However, browser-based testing has limitations. Notably, "browser-native alert modals aren't visible through the Playwright MCP," causing features relying on modals to be consistently more buggy. Understanding these tool limitations is essential for calibrating evaluator expectations.

### 5.4 Evaluator Calibration

Calibrating evaluators is a multi-iteration process requiring careful attention to several dynamics:

**Prompt sensitivity**: Anthropic discovered that including phrases like "museum quality designs" subtly steered generator outputs toward specific aesthetic convergences. The language used to describe quality standards influences not just what the evaluator accepts, but what the generator aims for.

**Skepticism calibration**: Evaluators must be tuned through multiple iteration cycles to achieve appropriate skepticism. Too lenient, and the evaluator rubber-stamps mediocre work. Too strict, and the system enters infinite refinement loops or fails to make progress.

**Few-shot examples**: Providing concrete examples of what constitutes high, medium, and low scores improves consistency. <a href="https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents" target="_blank">LLM-as-judge graders should be closely calibrated with human experts</a> to ensure minimal divergence between human and model grading.

### 5.5 The Digital Audio Workstation Case Study

When Anthropic tested the harness with Opus 4.6 to build a Digital Audio Workstation:

- **Duration**: 3 hours 50 minutes
- **Cost**: $124.70
- **Sprint decomposition was removed** (the improved model didn't need it)

The QA evaluator caught meaningful gaps that would have gone undetected by the generator alone: non-functional timeline interactions, missing effect visualizations, and incomplete audio recording features. This demonstrated that <a href="https://www.anthropic.com/engineering/harness-design-long-running-apps" target="_blank">even with improved models, evaluator agents add value at capability boundaries</a>.

---

## 6. Cost and Performance Tradeoffs

### 6.1 The Token Economics of Multi-Agent Systems

Multi-agent harnesses are significantly more expensive than solo agents. Anthropic's results demonstrate the scale:

| System | Task | Duration | Cost | Token Multiplier |
|--------|------|----------|------|-----------------|
| Solo agent | Retro Game Maker | 20 min | $9 | 1× |
| Full harness | Retro Game Maker | 6 hr | $200 | ~22× |
| Simplified harness (Opus 4.6) | Digital Audio Workstation | 3 hr 50 min | $124.70 | ~14× |

Anthropic's multi-agent research system provides additional context: <a href="https://www.anthropic.com/engineering/multi-agent-research-system" target="_blank">agents consume about 4× more tokens than chat interactions, and multi-agent systems use about 15× more tokens than chats</a>.

The most dangerous economic trap is **quadratic token growth**: in multi-turn conversations, LLMs charge for every input token sent in every turn, causing costs to accumulate rapidly. Research indicates that <a href="https://online.stevens.edu/blog/hidden-economics-ai-agents-token-costs-latency/" target="_blank">an unconstrained agent can cost $5-8 per task to solve a single software engineering issue</a>.

### 6.2 When the Cost Is Justified

The harness cost is justified when:

- **Quality requirements are high**: The solo agent's broken game mechanics versus the harness's fully functional application represents a qualitative difference, not just quantitative
- **Tasks exceed single-context capacity**: Complex applications with 200+ features cannot be reliably built in one session
- **Errors are expensive to fix later**: Catching bugs during development (via the evaluator) is far cheaper than debugging production failures
- **The task sits at the model's capability boundary**: External QA adds value primarily for tasks that push beyond what the model can reliably do alone

Conversely, for simple scripts, prototypes, or small scoped changes, the harness overhead is not justified—<a href="https://www.freecodecamp.org/news/how-to-apply-gan-architecture-to-multi-agent-code-generation/" target="_blank">single-pass generation is sufficient when speed matters more than thoroughness</a>.

### 6.3 Cost Optimization Strategies

Several strategies can dramatically reduce multi-agent costs:

**Model routing**: Not every agent task requires the most expensive model. Using cheaper models for planning, formatting, and simple evaluation while reserving premium models for complex reasoning can reduce costs significantly. Anthropic's own research system uses <a href="https://www.anthropic.com/engineering/multi-agent-research-system" target="_blank">Opus 4 as the lead agent with Sonnet 4 subagents</a>, balancing capability against cost.

**Context optimization**: <a href="https://thenewstack.io/a-guide-to-token-efficient-data-prep-for-llm-workloads/" target="_blank">Poor data serialization consumes 40-70% of available tokens through unnecessary formatting overhead</a>. CSV outperforms JSON by 40-50% for tabular data, and custom compact formats can achieve even greater efficiency.

**Caching**: Provider-level prompt caching for repeated system instructions and knowledge bases avoids re-sending static content. RAG retrieves only relevant chunks rather than entire documents.

**Trajectory reduction**: Research on <a href="https://arxiv.org/pdf/2509.23586" target="_blank">AgentDiet shows input token reduction of 39.9-59.7%</a> with cost savings of 21.1-35.9% while maintaining performance within -1.0% to +2.0%.

**Iteration caps**: Bounding refinement loops prevents runaway costs. Maximum 3 iterations per GAN loop with explicit convergence signals provides a practical ceiling.

### 6.4 The Scaling Insight

A key finding from Anthropic's multi-agent research: **token usage alone explains 80% of the variance in performance**. More tokens (distributed across more agents with separate context windows) directly correlates with better outcomes. However, upgrading models provides a "larger performance gain than doubling the token budget"—meaning model capability and token budget are complementary levers, not substitutes.

---

## 7. Comparison with Other Multi-Agent Frameworks

### 7.1 Framework Landscape

The multi-agent framework landscape in 2025-2026 includes several major approaches, each with distinct architectural philosophies:

| Framework | Architecture | Best For | Learning Curve |
|-----------|-------------|----------|---------------|
| **Anthropic Harness** | GAN-inspired generator-evaluator with sprint contracts | Long-running app development, quality-critical code | High (custom design) |
| **LangGraph** | Graph-based workflow with state machines | Complex orchestration with decision points | Medium-High |
| **CrewAI** | Role-based agents mimicking org structures | Business workflows, team-like coordination | Low (20 lines to start) |
| **AutoGen** | Conversational agent collaboration | Dynamic, dialogue-driven workflows | Medium |
| **MetaGPT** | SOP-based software company simulation | Full software development lifecycle | Medium |
| **OpenAI Codex** | Sandboxed agent loop with AGENTS.md | Autonomous coding in isolated environments | Medium |

### 7.2 Anthropic's Approach: What's Unique

Anthropic's harness design is distinguished by several features not found in general-purpose frameworks:

**Architectural enforcement of evaluation**: The evaluator is not optional or advisory—the orchestrator will not advance the pipeline without the evaluator's approval signal. This makes quality assurance a structural guarantee rather than a best practice.

**Sprint contract negotiation**: No other framework implements pre-implementation agreements between generator and evaluator agents. This mirrors professional agile practices and prevents the common failure mode of generators declaring work "done" by their own standards.

**Browser-based live testing**: While other frameworks support testing, Anthropic's evaluator interacts with the live running application through Playwright, testing as an end-user would rather than examining code or test output.

**Progressive scaffolding removal**: Anthropic explicitly designs harnesses with the expectation that components will be removed as models improve. This stands in contrast to frameworks that add capabilities incrementally.

### 7.3 LangGraph

<a href="https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen" target="_blank">LangGraph</a> reached v1.0 in late 2025 and became the default runtime for all LangChain agents. Its graph-based architecture:

- Represents agent workflows as directed graphs with nodes (processing steps) and edges (transitions)
- Supports sophisticated state management with checkpointing and recovery
- Excels at parallel processing with multiple decision points
- Provides the most fine-grained control over execution flow

LangGraph's strength is **explicitness**—every possible state transition is defined in the graph. This makes debugging and monitoring straightforward but requires more upfront design work than conversation-based approaches.

### 7.4 CrewAI

<a href="https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen" target="_blank">CrewAI</a> adopts a role-based model inspired by organizational structures. Agents are defined by roles, goals, and backstories—making it intuitive for teams modeling business processes. Its event-driven Flows system enables sophisticated coordination with the lowest learning curve among major frameworks.

### 7.5 MetaGPT

<a href="https://github.com/FoundationAgents/MetaGPT" target="_blank">MetaGPT</a> explicitly models a software company, with agents playing roles of product manager, architect, project manager, and engineer. It follows **Standardized Operating Procedures (SOPs)** encoded as prompt sequences, producing complete development artifacts from requirements through implementation. Its paper AFlow was accepted for oral presentation at ICLR 2025, and the framework evolved into MGX—a no-code development platform.

### 7.6 OpenAI Codex

<a href="https://developers.openai.com/codex/guides/agents-md" target="_blank">OpenAI Codex</a> takes a different approach, focusing on sandboxed execution with strict isolation:

- **AGENTS.md files** provide machine-readable instructions (~100 lines) injected into context
- **Sandboxed execution** constrains what agents can access (files, network, commands)
- **Per-worktree isolation** gives each task its own copy of the repository
- Post-trained models (GPT-5 Codex) are optimized specifically for their harness

A key difference: <a href="https://www.nxcode.io/resources/news/what-is-harness-engineering-complete-guide-2026" target="_blank">frontier coding models are post-trained on their harnesses</a>, meaning Claude performs best in Claude Code's harness and Codex performs best in its own environment. This creates a tight coupling between model and infrastructure.

### 7.7 Anthropic's Multi-Agent Research System

Beyond coding, Anthropic built a <a href="https://www.anthropic.com/engineering/multi-agent-research-system" target="_blank">multi-agent research system</a> using an orchestrator-worker pattern. A lead agent (Opus 4) coordinates Sonnet 4 subagents for parallel information retrieval. Key findings:

- Outperformed single-agent Opus 4 by **90.2%** on research evaluations
- **Parallel tool calling cut research time by up to 90%** for complex queries
- The system works mainly because it helps "spend enough tokens to solve the problem"
- Domains requiring shared context or real-time coordination remain poor fits

### 7.8 The Emerging Orchestration Tiers

<a href="https://addyosmani.com/blog/code-agent-orchestra/" target="_blank">Addy Osmani's analysis</a> identifies three tiers of agent orchestration:

| Tier | Scope | Example |
|------|-------|---------|
| **Tier 1** | In-process subagents and teams (single terminal) | Claude Code subagents |
| **Tier 2** | Local orchestrators (3-10 agents, visual dashboards) | Conductor, custom harnesses |
| **Tier 3** | Cloud async agents (overnight, CI-integrated) | GitHub Copilot Coding Agent, Claude Code Web |

Most productive teams in 2026 use all three tiers: interactive work locally, parallel sprints through orchestrators, and overnight backlog drainage via cloud agents.

---

## 8. The Evolving Harness: How Design Shifts with Model Capability

### 8.1 The Central Principle

Anthropic articulated a principle that defines the future of harness engineering: <a href="https://www.anthropic.com/engineering/harness-design-long-running-apps" target="_blank">"The space of interesting harness combinations doesn't shrink as models improve. Instead, it moves."</a>

This means better models don't eliminate the need for harnesses—they shift what harnesses need to do. Components that compensated for model weaknesses become unnecessary, while new architectural possibilities emerge at higher complexity levels.

### 8.2 Three Stages of Evolution

Anthropic's harness evolved across three distinct stages, each corresponding to model capability improvements:

**Stage 1: Initializer + Coding Agent (Sonnet 4.5, Nov 2025)**

The earliest harness addressed basic multi-session coordination. Sonnet 4.5 exhibited strong context anxiety, so the harness used:
- Complete context resets between sessions
- JSON feature lists to prevent one-shotting
- Structured progress files for session handoff
- Explicit behavioral constraints in prompts

**Stage 2: Planner + Generator + Evaluator (Opus 4.5)**

With a more capable model, the harness expanded to three agents:
- Sprint decomposition broke work into manageable chunks
- Sprint contracts negotiated between Generator and Evaluator
- Playwright-based live testing by the Evaluator
- 10 sprints covering 16+ features

**Stage 3: Simplified Harness (Opus 4.6)**

The improved model required less scaffolding:
- Sprint decomposition was **removed entirely**
- Evaluation was reduced to end-stage assessment
- The model could sustain coherent work across a two-hour build without sprint boundaries
- Cost dropped from $200 to $124.70 for comparable complexity

### 8.3 The Scaffolding Removal Principle

Each harness component encodes an **assumption about model limitations**. As models strengthen, those assumptions require systematic re-evaluation. Anthropic's approach:

1. Build the harness with all components needed for the current model
2. As models improve, **systematically remove components** one at a time
3. Measure impact of each removal on output quality
4. Retain only components that still demonstrably improve outcomes

This process revealed that sprint decomposition was "load-bearing" for Opus 4.5 but unnecessary for Opus 4.6. The evaluator, however, continued to add value across both models—suggesting that <a href="https://www.working-ref.com/en/reference/anthropic-harness-design-philosophy-evolution" target="_blank">external evaluation addresses a more fundamental limitation than planning decomposition</a>.

### 8.4 The "Bitter Lesson" Applied to Harnesses

Drawing from Rich Sutton's "Bitter Lesson" in machine learning, <a href="https://www.philschmid.de/agent-harness-2026" target="_blank">rigid hand-coded control flows become obsolete with each model release</a>. Companies like Vercel, Manus, and LangChain repeatedly refactored their agent systems, demonstrating that over-engineering the control flow makes systems fragile.

The practical implication: **design harnesses to be modular and deletable**. Assume every component will eventually be removed, and structure the system so removal is straightforward rather than requiring a rewrite.

### 8.5 The Human-Written Context Advantage

A surprising finding from multi-agent coding research: <a href="https://addyosmani.com/blog/code-agent-orchestra/" target="_blank">AI-generated AGENTS.md files provide no benefit (~3% reduction in success) while increasing inference costs by 20%</a>. Developer-written context, however, improves outcomes by ~4%.

This suggests that human judgment about what context is important—which conventions matter, what gotchas exist, which architectural decisions carry weight—remains superior to automated context generation. The harness benefits from human curation of its configuration, even as agents handle more of the execution.

### 8.6 Future Directions

Several open questions define the next frontier of harness design:

- **Asynchronous agent coordination**: Current systems mostly wait for all agents to complete before proceeding. Asynchronous execution could enable concurrent work but introduces coordination challenges.
- **Cross-domain generalization**: Harnesses optimized for web development may not transfer to scientific research, financial modeling, or embedded systems.
- **Agent-to-Agent protocols**: Standards like <a href="https://www.ibm.com/think/topics/agent2agent-protocol" target="_blank">Google's A2A protocol</a> and Anthropic's MCP are enabling interoperability between agent systems.
- **Self-improving harnesses**: Using agent execution data as training signal to improve future harness configurations.
- **Compound AI systems**: Multi-model routing where different cognitive tasks use different models optimized for cost and capability.

---

## 9. Conclusions

### 9.1 Key Takeaways

1. **The harness is the architecture**. Model capability alone does not determine outcomes—the execution environment, evaluation mechanisms, and coordination infrastructure are equally important. LangChain demonstrated this concretely by improving a coding agent from 52.8% to 66.5% performance by changing only the harness, not the model.

2. **Separate generation from evaluation**. Self-evaluation bias is a fundamental limitation of single-agent systems. The GAN-inspired generator-evaluator pattern provides structurally guaranteed quality assurance that iterative self-critique cannot match.

3. **Manage context as a first-class design constraint**. Context rot is real, measurable, and affects every frontier model. Effective harnesses use structured artifacts, session startup protocols, and strategic resets rather than relying on raw context window capacity.

4. **Design for deletion**. Every harness component encodes an assumption about model limitations. As models improve, components should be systematically tested for continued relevance and removed when no longer load-bearing.

5. **Quality gates are the bottleneck, not generation**. In multi-agent systems, the ability to verify output reliably matters more than the ability to generate it quickly. Browser-based testing, structured rubrics, and calibrated evaluators create the feedback loops that turn raw generation into reliable software.

6. **Cost follows quality requirements**. Multi-agent harnesses cost 10-20× more than solo agents, but the quality difference can be categorical (broken vs. functional). The decision framework is straightforward: use harnesses when quality requirements justify the cost, solo agents when speed matters more.

7. **The harness design space moves, not shrinks**. Better models don't eliminate the need for thoughtful orchestration—they enable new possibilities at higher complexity levels while making previously essential scaffolding optional.

### 9.2 Practical Recommendations

For practitioners building long-running coding agents:

- **Start simple**: Use the evaluator-optimizer pattern before building full three-agent pipelines
- **Use structured artifacts**: JSON feature lists, git history, and progress files over context compaction
- **Test with the browser**: Playwright-based end-to-end testing catches bugs that code review and unit tests miss
- **Set iteration caps**: 3-5 loops maximum with explicit convergence signals
- **Calibrate evaluators**: Use few-shot examples matching your quality standards; invest multiple iteration cycles in tuning skepticism levels
- **Route models by task**: Use cheaper models for planning and formatting, premium models for complex reasoning
- **Write AGENTS.md/CLAUDE.md yourself**: Human-curated context outperforms AI-generated context
- **Measure and remove**: Systematically test each harness component's continued value as models improve

---

## 10. Sources

1. <a href="https://www.anthropic.com/engineering/harness-design-long-running-apps" target="_blank">Anthropic Engineering - Harness Design for Long-Running Application Development</a>
2. <a href="https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents" target="_blank">Anthropic Engineering - Effective Harnesses for Long-Running Agents</a>
3. <a href="https://www.anthropic.com/research/building-effective-agents" target="_blank">Anthropic Research - Building Effective Agents</a>
4. <a href="https://www.anthropic.com/engineering/multi-agent-research-system" target="_blank">Anthropic Engineering - How We Built Our Multi-Agent Research System</a>
5. <a href="https://www.working-ref.com/en/reference/anthropic-harness-design-philosophy-evolution" target="_blank">Working-Ref - Anthropic's Harness Design Philosophy: From Multi-Agent to Single-Agent Evolution</a>
6. <a href="https://www.freecodecamp.org/news/how-to-apply-gan-architecture-to-multi-agent-code-generation/" target="_blank">freeCodeCamp - How to Apply GAN Architecture to Multi-Agent Code Generation</a>
7. <a href="https://www.nxcode.io/resources/news/what-is-harness-engineering-complete-guide-2026" target="_blank">NxCode - What Is Harness Engineering? Complete Guide for AI Agent Development (2026)</a>
8. <a href="https://www.trychroma.com/research/context-rot" target="_blank">Chroma Research - Context Rot: How Increasing Input Tokens Impacts LLM Performance</a>
9. <a href="https://businessengineer.ai/p/the-harness-as-the-agentic-moat" target="_blank">Business Engineer - The Harness as the Agentic Moat</a>
10. <a href="https://www.zenml.io/llmops-database/long-running-agent-harness-for-multi-context-software-development" target="_blank">ZenML - Long-Running Agent Harness for Multi-Context Software Development</a>
11. <a href="https://addyosmani.com/blog/code-agent-orchestra/" target="_blank">Addy Osmani - The Code Agent Orchestra: What Makes Multi-Agent Coding Work</a>
12. <a href="https://www.philschmid.de/agent-harness-2026" target="_blank">Philipp Schmid - The Importance of Agent Harness in 2026</a>
13. <a href="https://arxiv.org/html/2603.05344v1" target="_blank">arXiv - Building AI Coding Agents for the Terminal: Scaffolding, Harness, Context Engineering, and Lessons Learned</a>
14. <a href="https://www.datacamp.com/tutorial/crewai-vs-langgraph-vs-autogen" target="_blank">DataCamp - CrewAI vs LangGraph vs AutoGen: Choosing the Right Multi-Agent AI Framework</a>
15. <a href="https://github.com/FoundationAgents/MetaGPT" target="_blank">GitHub - MetaGPT: The Multi-Agent Framework</a>
16. <a href="https://developers.openai.com/codex/guides/agents-md" target="_blank">OpenAI Developers - Custom Instructions with AGENTS.md (Codex)</a>
17. <a href="https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents" target="_blank">Anthropic Engineering - Demystifying Evals for AI Agents</a>
18. <a href="https://playwright.dev/" target="_blank">Playwright - Fast and Reliable End-to-End Testing for Modern Web Apps</a>
19. <a href="https://blog.jetbrains.com/research/2025/12/efficient-context-management/" target="_blank">JetBrains Research - Cutting Through the Noise: Smarter Context Management for LLM-Powered Agents</a>
20. <a href="https://online.stevens.edu/blog/hidden-economics-ai-agents-token-costs-latency/" target="_blank">Stevens Institute - The Hidden Economics of AI Agents: Managing Token Costs and Latency Trade-offs</a>
21. <a href="https://arxiv.org/pdf/2509.23586" target="_blank">arXiv - Improving the Efficiency of LLM Agent Systems through Trajectory Reduction</a>
22. <a href="https://www.ibm.com/think/topics/agent2agent-protocol" target="_blank">IBM - What Is Agent2Agent (A2A) Protocol?</a>
23. <a href="https://www.confident-ai.com/blog/definitive-ai-agent-evaluation-guide" target="_blank">Confident AI - AI Agent Evaluation: The Definitive Guide</a>
24. <a href="https://www.infoq.com/articles/evaluating-ai-agents-lessons-learned/" target="_blank">InfoQ - Evaluating AI Agents in Practice: Benchmarks, Frameworks, and Lessons Learned</a>
25. <a href="https://thenewstack.io/a-guide-to-token-efficient-data-prep-for-llm-workloads/" target="_blank">The New Stack - A Guide to Token-Efficient Data Prep for LLM Workloads</a>
