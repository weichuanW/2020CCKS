# Agentic Evaluation, Memory Stabilization, and Skill Evolution Reading Notes

> Date: 2026-06-24
>
> Scope: Recent research on agentic evaluation, especially trajectory/trace/path-level evaluation; memory stabilization and consolidation for agents; and skill evolution / lifelong self-improving agents.
>
> Note: Several 2026 works are recent preprints. Treat their empirical conclusions as trend signals rather than settled benchmarks.

## 0. Overall Synthesis

The three research lines are converging:

> Trajectory is the raw material; evaluation provides reward and diagnosis; memory stabilizes useful experience; skill evolution compresses experience into reusable capability.

Modern agent systems can be viewed as a loop:

```text
Agent executes a task
  -> produces trajectory / trace
  -> trajectory evaluation diagnoses success, failure, safety, efficiency, and error location
  -> memory stabilization decides what to store and how to prevent drift or pollution
  -> skill evolution distills successes and failures into reflections, principles, or skills
  -> the next agent run retrieves memory and skills to improve behavior
```

The core trend is a shift from:

```text
Did the agent finish?
```

to:

```text
Did the agent finish through a correct, safe, efficient, explainable, reproducible, and reusable path?
```

## Industry Addendum

This note is complemented by an industry-focused survey:

- [Industry view: Agent trajectory / trace evaluation](industry-agent-trajectory-evaluation.md)

That addendum focuses on how production teams operationalize trajectory evaluation with LangSmith, OpenAI Agents, Anthropic eval methodology, Braintrust, W&B Weave, Arize Phoenix/OpenInference, Langfuse, HoneyHive, Humanloop, DeepEval, Ragas, LlamaIndex, AutoGen, Patronus AI, Helicone, and AgentOps.

The main industry pattern is:

```text
OpenTelemetry/OpenInference tracing
  -> offline golden eval and CI gate
  -> online async scoring and monitoring
  -> human annotation and judge calibration
  -> failed-trace regression set
  -> prompt/tool/agent/memory/skill optimization
```

---

## 1. Agentic Evaluation and Trajectory Evaluation

### 1.1 AgentBench: Evaluating LLMs as Agents, 2023 / ICLR 2024

- **Research problem**: Can LLMs act as general-purpose interactive agents across environments such as OS, database, knowledge graph, WebShop, ALFWorld, and Mind2Web?
- **Method**: Builds 8 environments with a unified observation-action interface to test multi-turn reasoning, decision making, and instruction following.
- **Trajectory representation/evaluation**: Records multi-turn interaction trajectories, but evaluation mainly relies on task-specific final rewards, success rates, F1, or environment scores.
- **Key findings**: GPT-4 and Claude-style models significantly outperform open models; long-horizon reasoning and instruction following remain key bottlenecks.
- **Limitations**: It can identify that an agent failed, but not where or why the trajectory failed; safety and efficiency of the path are not first-class metrics.
- **Technical role**: Establishes an early systematic LLM-as-agent benchmark and motivates later process-aware benchmarks such as AgentBoard.

### 1.2 WebArena, 2023 / ICLR 2024

- **Research problem**: Can web agents complete realistic, reproducible, long-horizon tasks on fully functional websites?
- **Method**: Provides self-hosted e-commerce, forum, GitLab, and CMS websites with 812 tasks.
- **Trajectory representation/evaluation**: Browser action traces can be saved, but scoring mainly checks final web/database state or final answer correctness.
- **Key findings**: The best GPT-4-based agent in the original study achieved about 14.4% success, far below human performance around 78%.
- **Limitations**: Final-state evaluation can miss redundant actions, unsafe intermediate steps, side effects, and poor recovery behavior.
- **Technical role**: Moves web-agent evaluation from toy environments to realistic websites; later works such as AgentRewardBench and TRACE address its process-evaluation gap.

### 1.3 AgentBoard, 2024 / NeurIPS Datasets and Benchmarks

- **Research problem**: Success rate alone is too coarse for multi-turn agents; how can intermediate capability be measured?
- **Method**: Provides an analytical board across tasks such as ALFWorld, ScienceWorld, BabyAI, Jericho, PDDL, WebShop, WebArena, and tool-use tasks.
- **Trajectory representation/evaluation**: Uses subgoals and trajectory visualization to compute progress rate and diagnose partial completion.
- **Metrics**: Success Rate, Progress Rate, Grounding Accuracy, long-horizon analysis, difficulty breakdown, sub-skill analysis.
- **Key findings**: Progress rate separates agents that all fail final success but differ in how close they get.
- **Limitations**: Subgoal annotation is costly and task-dependent; safety, efficiency, and error recovery are still incomplete.
- **Technical role**: Transitional work from outcome-only evaluation to process-aware evaluation.

### 1.4 tau-bench, 2024

- **Research problem**: Can customer-service agents reliably interact with users, follow domain policies, and use APIs?
- **Method**: Simulates a user-agent-tool setting with domain APIs, policy documents, hidden database state, and LLM-simulated users.
- **Trajectory representation/evaluation**: Records dialogue turns, tool calls, and database changes; scoring compares final DB state to the target state.
- **Key metric**: `pass^k`, which measures whether the same task succeeds consistently across independent trials.
- **Key findings**: Strong function-calling agents can show nontrivial single-run success but poor repeated reliability.
- **Limitations**: Policy violations or risky intermediate steps may be missed if final state is correct.
- **Technical role**: Adds reliability and user interaction as central evaluation dimensions.

### 1.5 AppWorld, 2024 / ACL

- **Research problem**: Can API/coding agents perform realistic digital tasks across multiple applications rather than simple API chains?
- **Method**: Provides 9 simulated apps, 457 APIs, about 106 synthetic users, and 750 tasks requiring code generation and iterative API use.
- **Trajectory representation/evaluation**: Records code execution, API calls, and environment responses; scoring uses state-based unit tests to check both goal completion and collateral damage.
- **Metrics**: Task Goal Completion and Scenario Goal Completion.
- **Key findings**: GPT-4o solved roughly 49% of normal tasks and 30% of challenge tasks in reported settings.
- **Limitations**: Still mostly final-state oriented; it does not fully distinguish efficient paths from risky paths that eventually self-correct.
- **Technical role**: A strong realistic API-agent environment, later used by skill-library and RL self-improvement work.

### 1.6 OSWorld, 2024 / NeurIPS

- **Research problem**: Can multimodal agents operate real desktop environments like humans?
- **Method**: Provides real Ubuntu, Windows, and macOS environments with 369 open-ended computer tasks.
- **Trajectory representation/evaluation**: Trajectories include screenshots, accessibility trees, mouse/keyboard actions, and final machine state; evaluation uses execution scripts.
- **Key findings**: Human success was around 72%, while the best models were around 12% in the original report; GUI grounding and operational knowledge are major bottlenecks.
- **Limitations**: Environment reproducibility is complex; low-level actions introduce noise; path quality remains under-evaluated.
- **Technical role**: Extends agent evaluation from web tasks to general computer-use agents.

### 1.7 WorkArena and WorkArena++, 2024

- **Research problem**: Can agents automate enterprise knowledge-work workflows in browser-based software?
- **Method**: Uses ServiceNow tasks. WorkArena focuses on atomic tasks; WorkArena++ adds 682 compositional tasks requiring planning and reasoning.
- **Trajectory representation/evaluation**: BrowserGym provides HTML, accessibility tree, screenshots, and action logs; oracle traces can be generated.
- **Metrics**: Success rate and skill-category breakdowns.
- **Key findings**: Current models remain far from human-level performance on compositional enterprise tasks.
- **Limitations**: Focuses on one enterprise platform; generated traces are useful for training, but path quality evaluation is still limited.
- **Technical role**: Moves web-agent benchmarks toward enterprise workflow automation.

### 1.8 AgentGym and AgentEvol, 2024 / ACL 2025

- **Research problem**: How can the community evaluate and train generalist agents across many interactive environments?
- **Method**: Provides 14 environments, 89 tasks, AgentEval, AgentTraj, and AgentTraj-L.
- **Trajectory representation/evaluation**: Uses unified ReAct-style trajectories with observation, thought/action, feedback, and reward.
- **Training method**: AgentEvol uses exploration trajectories and reward-weighted supervised learning.
- **Key findings**: Agent trajectory data improves open-source agents; self-evolution can further improve performance.
- **Limitations**: Reward quality determines learning quality; cross-environment score aggregation can hide important details.
- **Technical role**: Bridges benchmark, trajectory dataset, and agent self-improvement.

### 1.9 AgentRewardBench, 2025

- **Research problem**: Can LLM judges reliably evaluate full web-agent trajectories?
- **Method**: Provides 1,302 expert-reviewed trajectories from five web benchmarks and four LLM agents.
- **Trajectory representation/evaluation**: Full web trajectories are judged for success, side effects, and repetitive behavior.
- **Key findings**: No single LLM judge dominates across all benchmarks; rule-based evaluation often underreports agent success.
- **Limitations**: Mostly web-agent focused; expert labels are expensive and partially subjective.
- **Technical role**: Meta-evaluation of trajectory evaluators and reward models.

### 1.10 TRAIL: Trace Reasoning and Agentic Issue Localization, 2025

- **Research problem**: Can models read complex agent traces and localize or classify errors?
- **Method**: Provides 148 human-annotated long traces with 841 errors across reasoning, planning/coordination, and system execution.
- **Trace representation/evaluation**: Uses OpenTelemetry/OpenInference-style structured traces and asks models to identify error spans and types.
- **Key findings**: Even strong long-context models perform poorly; best reported joint accuracy is around 11%.
- **Limitations**: Small but expensive dataset; traces are extremely long; it evaluates debugging ability more than task success.
- **Technical role**: Connects agent evaluation with observability, telemetry, and production debugging.

### 1.11 CORE: Full-Path Evaluation of LLM Agents Beyond Final State, 2025

- **Research problem**: Final state can hide unsafe, inefficient, or incorrectly ordered tool-use paths.
- **Method**: Encodes valid tool-use paths with deterministic finite automata.
- **Trajectory representation/evaluation**: Compares generated tool-call sequences to reference paths.
- **Metrics**: Path Correctness, Path Correctness-Kendall Tau Composite, Prefix Criticality, Harmful-Call Rate, Efficiency.
- **Key findings**: Final-state evaluation can miss harmful intermediate calls, skipped preconditions, and compensating error pairs.
- **Limitations**: Requires DFA/reference-path construction; open environments have many equivalent valid paths.
- **Technical role**: Representative of reference-based path-level evaluation.

### 1.12 TRACE / Beyond the Final Answer, 2025

- **Research problem**: How can reasoning/tool-use trajectories be evaluated without annotating all valid gold trajectories?
- **Method**: Builds an evidence bank over previous steps and uses it for reference-free trajectory evaluation.
- **Trajectory representation/evaluation**: Evaluates efficiency, hallucination, and adaptivity using accumulated evidence.
- **Key findings**: Evidence organization makes trajectory judging more reliable than naive LLM-as-judge.
- **Limitations**: Still depends on LLM judges; safety and compliance dimensions are less developed.
- **Technical role**: Complements CORE by reducing dependence on hand-coded reference paths.

### 1.13 TRAJECT-Bench, 2025 / 2026

- **Research problem**: Can tool agents select the right tools, fill arguments correctly, order calls properly, and satisfy dependencies?
- **Method**: Provides production-style APIs and controlled tool-use trajectories of different breadth/depth.
- **Trajectory representation/evaluation**: Compares predicted JSON tool-call trajectories against gold trajectories.
- **Metrics**: Tool selection, argument correctness, order/dependency satisfaction, trajectory exact match, final answer accuracy.
- **Key findings**: Common failures include similar-tool confusion, parameter-blind selection, and sharp degradation for long trajectories.
- **Limitations**: Gold trajectories can penalize valid alternative solutions.
- **Technical role**: Makes tool-use trajectory fidelity a first-class benchmark target.

---

## 2. Memory Stabilization and Consolidation

### 2.1 Generative Agents, 2023

- **Research problem**: How can LLM-driven agents maintain believable long-term behavior in an open simulated world?
- **Memory structure**: Memory stream storing observations, reflections, and plans.
- **Mechanism**: Retrieval combines relevance, recency, and importance; high accumulated importance triggers reflection.
- **Stabilization contribution**: Reflection compresses events into higher-level insights and improves behavioral consistency.
- **Limitations**: No explicit forgetting; memory grows without bound; reflection can amplify errors.
- **Technical role**: Establishes the event-stream + retrieval + reflection + planning memory pattern.

### 2.2 MemoryBank, 2023 / AAAI 2024

- **Research problem**: How can long-term dialogue agents remember user experiences, preferences, and personality?
- **Memory structure**: Conversation logs, daily summaries, global summaries, user personality summaries.
- **Mechanism**: Uses the Ebbinghaus forgetting curve; recalled memories get reinforced.
- **Stabilization contribution**: Explicit selective forgetting and reinforcement.
- **Limitations**: Forgetting is heuristic; conflict resolution, safety, and error propagation are underdeveloped.
- **Technical role**: Moves from general memory streams toward personalized long-term memory.

### 2.3 MemGPT, 2023

- **Research problem**: How can fixed-context LLMs handle very long conversations and documents?
- **Memory structure**: OS-style hierarchy: main context as RAM, archival/recall storage as disk.
- **Mechanism**: The model uses function calls to read, write, and search memory.
- **Stabilization contribution**: Treats context windows as scarce resources and introduces virtual context management.
- **Limitations**: Correct memory use depends on model discipline; truthfulness and access control are not central.
- **Technical role**: A foundational virtual-memory approach for LLM agents.

### 2.4 A-Mem, 2025

- **Research problem**: Fixed-schema memory systems are too rigid; can agents organize and evolve memory dynamically?
- **Memory structure**: Zettelkasten-style atomic notes with content, tags, keywords, context, embeddings, and links.
- **Mechanism**: New memories create links to old memories and can trigger updates to old memory contexts/tags.
- **Stabilization contribution**: Dynamic linking improves multi-hop association and long-term organization.
- **Limitations**: LLM rewriting of old notes can introduce semantic drift; provenance is weak.
- **Technical role**: Moves from flat vector stores to networked agentic memory.

### 2.5 Mem0, 2025

- **Research problem**: How can production agents maintain long-term personalized memory with low latency and token cost?
- **Memory structure**: Base Mem0 stores salient facts with embeddings; Mem0g adds graph memory with entities and relations.
- **Mechanism**: Dynamically extracts, updates, deletes, and retrieves memories; graph variant handles entity relations.
- **Stabilization contribution**: CRUD-style conflict handling reduces duplication and contradictions; large latency/token reductions.
- **Limitations**: Full context can still be more accurate; extraction errors can pollute memory.
- **Technical role**: Engineering-oriented synthesis of MemoryBank, A-Mem, and MemGPT ideas.

### 2.6 MEM1, 2025

- **Research problem**: Can agents use constant-size memory across long-horizon tasks?
- **Memory structure**: A compact internal state updated every turn.
- **Mechanism**: Reinforcement learning teaches the model to preserve relevant information and discard irrelevant content.
- **Stabilization contribution**: Bounded memory and reduced token use.
- **Limitations**: Compression errors are hard to audit and can persist.
- **Technical role**: Shifts from external memory stores to learned memory compression.

### 2.7 SSGM, 2026

- **Research problem**: How can evolving memory avoid semantic drift, privacy leakage, and memory poisoning?
- **Memory structure**: Mutable active graph, immutable episodic ledger, read/write governance gates.
- **Mechanism**: Write validation, read filtering, temporal freshness, provenance checks, and periodic reconciliation.
- **Stabilization contribution**: Treats memory integrity, safety, and drift control as first-class concerns.
- **Limitations**: Mostly conceptual; added governance increases latency and complexity.
- **Technical role**: Moves the field from "remember more" toward "remember safely and auditably."

### 2.8 CraniMem, 2026

- **Research problem**: How can long-horizon agents avoid unbounded growth and distractor pollution?
- **Memory structure**: Goal-conditioned gate, bounded episodic buffer, long-term knowledge graph.
- **Mechanism**: Gates write decisions; periodically consolidates high-utility episodic traces into the graph.
- **Stabilization contribution**: Bounded memory, utility-based pruning, and improved noise robustness.
- **Limitations**: Experiments are limited; latency can be high.
- **Technical role**: Cognitive-inspired gated and bounded consolidation pipeline.

### 2.9 Auto-Dreamer, 2026

- **Research problem**: How can multi-session agent experience be offline-consolidated into compact reusable memory?
- **Memory structure**: Fast online writer plus slow offline consolidator over a typed memory bank.
- **Mechanism**: Region rewriting replaces a memory region with a compact replacement set; trained with GRPO.
- **Stabilization contribution**: Provenance-grounded offline consolidation reduces redundancy and memory bloat.
- **Limitations**: Missed writes cannot easily be recovered; excessive abstraction may drop useful details.
- **Technical role**: A learned "sleep-time consolidation" upgrade to reflection and summarization.

### 2.10 MemSkill, 2026

- **Research problem**: Can agents learn not only memory content but also reusable memory-management skills?
- **Memory structure**: Trace-specific memory bank plus shared memory skill bank.
- **Mechanism**: A controller selects memory skills; an executor constructs memory; a designer evolves skills from hard cases.
- **Stabilization contribution**: Evolves the meta-operations of remembering, not only remembered content.
- **Limitations**: Complex system; depends on LLM designer quality; skills may overfit benchmark failure modes.
- **Technical role**: Memory management as skill evolution.

---

## 3. Skill Evolution and Lifelong Agents

### 3.1 Reflexion, 2023

- **Research problem**: Can agents improve from failures without parameter updates?
- **Method**: Generates natural-language reflections from evaluator feedback and stores them in episodic memory.
- **Skill form**: Short reflections, not structured skills.
- **Key findings**: Strong gains on ALFWorld, HotPotQA, HumanEval, and related tasks.
- **Limitations**: Memory is short and can be polluted; best suited for retryable tasks.
- **Technical role**: Starting point for experience-to-reflection approaches.

### 3.2 Voyager, 2023

- **Research problem**: Can an LLM agent autonomously explore Minecraft and accumulate reusable skills?
- **Method**: Successful task solutions become executable JavaScript skills indexed by description embeddings.
- **Skill form**: Executable code plus natural-language descriptions.
- **Key findings**: More unique items, faster tech-tree milestones, and transfer to new worlds.
- **Limitations**: Depends on code execution, APIs, and clear environment feedback.
- **Technical role**: Moves from reflection to executable skill libraries.

### 3.3 ExpeL, 2023 / AAAI 2024

- **Research problem**: Can agents abstract cross-task experience from successful and failed trajectories?
- **Method**: Distills insights/rules and retrieves similar successful trajectories.
- **Skill form**: Natural-language insights plus trajectory retrieval.
- **Key findings**: Generally outperforms ReAct/Act baselines.
- **Limitations**: Insights can overgeneralize; validation is weak.
- **Technical role**: Bridges Reflexion and later principle-library approaches such as EvolveR.

### 3.4 AgentGym / AgentEvol, 2024 / ACL 2025

- **Research problem**: Can agents self-evolve across many environments using their own trajectories?
- **Method**: Collects trajectories and performs reward-weighted supervised learning.
- **Skill form**: No explicit skill library; experience is absorbed into model parameters.
- **Key findings**: Self-evolution improves cross-environment performance.
- **Limitations**: Requires rewards and training compute; learned capabilities are less interpretable than external skills.
- **Technical role**: Moves from in-context skill use to parameterized self-evolution.

### 3.5 LifelongAgentBench, 2025

- **Research problem**: How can agents be evaluated for continual learning, retention, and transfer?
- **Method**: Provides DB, OS, and KG environments with skill-dependent sequential tasks.
- **Skill form**: Benchmark-focused; evaluates replay and group self-consistency rather than proposing a full skill library.
- **Key findings**: Naive experience replay is limited by irrelevant history and context length.
- **Limitations**: More technical than open-ended life simulation; less focus on motivation or autonomy.
- **Technical role**: Provides a systematic benchmark for lifelong agent learning.

### 3.6 Experience-driven Lifelong Learning / StuLife, 2025

- **Research problem**: How can lifelong learning become open-ended, long-term, and autonomous?
- **Method**: Defines Experience Exploration, Long-term Memory, Skill Learning, and Knowledge Internalization.
- **Skill form**: Facts, events, reflections, skills, and strategies as long-term growth artifacts.
- **Key findings**: Even strong models perform poorly in long-term life simulation, suggesting the problem is far from solved.
- **Limitations**: New benchmark with complex metrics; standardization is still evolving.
- **Technical role**: Expands lifelong learning from technical tasks to life-like long-term agency.

### 3.7 EvolveR, 2025

- **Research problem**: Can agents form a closed loop of online interaction, offline distillation, online retrieval, and RL update?
- **Method**: Distills trajectories into strategic principles; retrieves principles during online interaction; updates policy with RL.
- **Skill form**: Principle or strategy library.
- **Key findings**: Improves multi-hop QA/search-style tasks over strong baselines.
- **Limitations**: Principle quality is hard to guarantee; wrong principles can amplify errors.
- **Technical role**: Combines ExpeL-style principle libraries with AgentEvol-style self-evolution.

### 3.8 SAGE, 2025 / ACL 2026

- **Research problem**: Can RL train agents to generate and use skill libraries reliably?
- **Method**: Sequential Rollout over chains of similar tasks; Skill-integrated Reward rewards both task success and skill quality/use.
- **Skill form**: Dynamically accumulated skill library across related tasks.
- **Key findings**: On AppWorld, improves Scenario Goal Completion while reducing interaction steps and generated tokens.
- **Limitations**: Depends on scored environments and similar task chains.
- **Technical role**: Combines Voyager-style skill libraries with GRPO/RL.

### 3.9 AutoSkill, 2026

- **Research problem**: How can stable user preferences, style, and workflows be converted into reusable skills?
- **Method**: Extracts recurring patterns from conversations and feedback into editable, versioned `SKILL.md` files.
- **Skill form**: Markdown skill artifacts for personalization and workflow conventions.
- **Key findings**: Demonstrates feasibility of automatic personalized skill formation.
- **Limitations**: Automatic validation is weak; privacy and preference overfitting are risks.
- **Technical role**: Moves from task skills to personalized lifelong skills.

### 3.10 SkillX, 2026

- **Research problem**: Can reusable SkillKBs be automatically built and transferred across agents?
- **Method**: Distills successful trajectories into planning, functional, and atomic skills.
- **Skill form**: Hierarchical Skill Knowledge Base.
- **Key findings**: Improves success and efficiency on AppWorld, BFCL-v3, and tau2-Bench-style tasks.
- **Limitations**: Requires strong models to extract high-quality skills; bad skills can transfer pollution.
- **Technical role**: Moves from per-agent skill libraries to plug-and-play SkillKBs.

### 3.11 MUSE-Autoskill, 2026

- **Research problem**: How can skills become long-lived, testable, maintainable, and transferable software assets?
- **Method**: Provides a lifecycle for skill creation, memory, management, evaluation, and refinement.
- **Skill form**: Skill packages with `SKILL.md`, scripts, tests, resources, and per-skill memory.
- **Key findings**: Skills significantly improve task success on SkillsBench; generated skills show transfer potential.
- **Limitations**: Benchmark remains small; test coverage, safety, dependencies, and conflicts remain engineering challenges.
- **Technical role**: A more mature engineering form of skill-library systems.

---

## 4. Time-Dimension Roadmap

### 2023: Agent Prototype Stage

**Keywords**: ReAct, reflection, memory stream, early skill libraries.

- Generative Agents: memory stream + reflection + planning.
- Reflexion: failed trajectory -> verbal reflection.
- Voyager: successful trajectory -> executable skill.
- ExpeL: multiple trajectories -> insights/rules.
- AgentBench and WebArena: early systematic agent benchmarks.

**Central question**: Can agents act, remember, and improve from feedback?

### 2024: Realistic Environments and Process Awareness

**Keywords**: realistic environments, progress rate, state-based evaluation, reliability.

- AgentBoard: progress rate.
- tau-bench: user-agent-tool interaction and `pass^k`.
- AppWorld: multi-app API world.
- OSWorld: real computer environment.
- WorkArena: enterprise web workflows.
- AgentGym: benchmark + trajectory data + self-evolution.

**Central question**: Can agents operate reliably in realistic web, API, computer, and enterprise settings?

### 2025: Trajectory Becomes a First-Class Object

**Keywords**: trajectory-level evaluation, trace debugging, path correctness, lifelong benchmark, memory compression.

- AgentRewardBench: evaluating trajectory judges.
- TRAIL: trace error localization.
- CORE: DFA-based full-path evaluation.
- TRACE: reference-free trajectory evaluation.
- TRAJECT-Bench: tool trajectory benchmark.
- MEM1: constant-memory agents.
- LifelongAgentBench: continual-learning evaluation.
- EvolveR and SAGE: experience-driven self-evolution.

**Central question**: Was the path correct, safe, efficient, adaptive, and reusable?

### 2026: Stabilization, Governance, and Skill Assetization

**Keywords**: memory governance, offline consolidation, skill lifecycle, agent-native memory.

- SSGM: memory stability and safety governance.
- CraniMem: gated and bounded memory.
- Auto-Dreamer: offline memory consolidation.
- MemSkill: evolving memory skills.
- AutoSkill: personalized skill self-evolution.
- SkillX: hierarchical SkillKB.
- MUSE-Autoskill: skill lifecycle management.

**Central question**: How can long-term memory and skill libraries remain bounded, auditable, safe, transferable, and maintainable?

---

## 5. Technique-Dimension Roadmap

### 5.1 Evaluation Line

```text
Final answer / success rate
  -> progress rate / subgoal tracking
  -> reliability metrics such as pass^k
  -> trajectory-level judging
  -> trace debugging and issue localization
  -> formal path evaluation such as DFA / valid path
  -> evaluation-as-reward for training
```

Representative works:

- AgentBench, WebArena: final success.
- AgentBoard: progress rate.
- tau-bench: `pass^k`.
- AgentRewardBench, TRACE: trajectory judge.
- TRAIL: trace debugging.
- CORE, TRAJECT-Bench: path/tool-call correctness.
- AgentGym, SAGE, EvolveR: evaluation signals become training signals.

### 5.2 Memory Line

```text
Raw memory stream
  -> summary + reflection
  -> forgetting / reinforcement
  -> virtual context / paging
  -> structured notes / graph memory
  -> learned compression / constant memory
  -> offline consolidation
  -> memory governance / safety / drift control
  -> evolving memory skills
```

Representative works:

- Generative Agents: memory stream.
- MemoryBank: forgetting curve.
- MemGPT: virtual memory.
- A-Mem, Mem0: structured and graph memory.
- MEM1: learned compact state.
- Auto-Dreamer: offline consolidation.
- SSGM: governance.
- MemSkill: memory operation skills.

Core tension:

> Remembering more improves recall but increases bloat, drift, and pollution; remembering less improves efficiency but risks losing essential evidence.

### 5.3 Skill Evolution Line

```text
Failure reflection
  -> experience insight
  -> executable skill
  -> principle / strategy library
  -> hierarchical skill library
  -> skill retrieval + composition
  -> skill verification / unit tests
  -> RL-trained skill generation and usage
  -> lifecycle-managed skill asset
```

Representative works:

- Reflexion: reflection.
- ExpeL: insight.
- Voyager: executable skill.
- EvolveR: principle library.
- SkillX: hierarchical SkillKB.
- MUSE-Autoskill: skill lifecycle.
- SAGE: RL-trained skill use.
- AutoSkill: personalized skill artifacts.

Core tension:

> Abstract skills transfer better but are harder to verify; concrete skills are more reliable but less reusable.

---

## 6. Recommended Reading Order

### 6.1 Foundations

1. Generative Agents
2. Reflexion
3. Voyager
4. ExpeL
5. AgentBench
6. WebArena

### 6.2 Realistic Environments and Evaluation Expansion

7. AgentBoard
8. tau-bench
9. AppWorld
10. OSWorld
11. WorkArena / WorkArena++
12. AgentGym

### 6.3 Trajectory and Trace Evaluation

13. AgentRewardBench
14. TRAIL
15. CORE
16. TRACE / Beyond the Final Answer
17. TRAJECT-Bench

### 6.4 Memory Stabilization

18. MemoryBank
19. MemGPT
20. A-Mem
21. Mem0
22. MEM1
23. SSGM
24. CraniMem
25. Auto-Dreamer
26. MemSkill

### 6.5 Skill Evolution

27. LifelongAgentBench
28. Experience-driven Lifelong Learning / StuLife
29. EvolveR
30. SAGE
31. AutoSkill
32. SkillX
33. MUSE-Autoskill

---

## 7. High-Value Research Opportunities

### 7.1 Trajectory Evaluation -> Memory Update

Not every trajectory should be written into memory. A good memory system should first judge which steps were successful, which were mistakes, and which patterns are reusable.

Open problems:

- trajectory-aware write filters;
- failure-aware memory updates;
- provenance-grounded memory entries;
- process-level reward for memory consolidation.

### 7.2 Memory Stabilization -> Skill Evolution

A skill library is a high-level compression of memory. It needs provenance, versioning, tests, and rollback to prevent bad skills from becoming permanent.

Open problems:

- skill provenance;
- skill conflict detection;
- skill unit tests;
- skill rollback and deprecation;
- privacy-aware skill extraction.

### 7.3 Trajectory-Level Reward -> Agent Self-Improvement

Final reward is sparse and often misleading. More useful training signals include tool correctness, argument correctness, dependency satisfaction, harmful-call rate, efficiency, and recovery quality.

Open problems:

- calibrated process reward models;
- trajectory-level RL;
- safe exploration with tools;
- equivalence classes of valid trajectories;
- scalable expert trace annotation.

---

## 8. Paper Index

The following papers are referenced in these notes. PDF downloads, when available, are tracked in `papers/manifest.md`.

### Agentic Evaluation and Trajectory Evaluation

- AgentBench: Evaluating LLMs as Agents.
- WebArena: A Realistic Web Environment for Building Autonomous Agents.
- AgentBoard: An Analytical Evaluation Board of Multi-turn LLM Agents.
- tau-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains.
- AppWorld: A Controllable World of Apps and People for Benchmarking Interactive Coding Agents.
- OSWorld: Benchmarking Multimodal Agents for Open-Ended Tasks in Real Computer Environments.
- WorkArena: How Capable are Web Agents at Solving Common Knowledge Work Tasks?
- WorkArena++: Towards Compositional Planning and Reasoning-based Common Knowledge Work Tasks.
- AgentGym: Evaluating and Evolving LLM Agents across Diverse Environments.
- AgentRewardBench: Evaluating Automatic Evaluations of Web Agent Trajectories.
- TRAIL: Trace Reasoning and Agentic Issue Localization.
- CORE: Full-Path Evaluation of LLM Agents Beyond Final State.
- Beyond the Final Answer: Evaluating the Reasoning Trajectories of Tool-Augmented Agents.
- TRAJECT-Bench: A Trajectory-Aware Benchmark for Evaluating Agentic Tool Use.

### Memory Stabilization and Consolidation

- Generative Agents: Interactive Simulacra of Human Behavior.
- MemoryBank: Enhancing Large Language Models with Long-Term Memory.
- MemGPT: Towards LLMs as Operating Systems.
- A-Mem: Agentic Memory for LLM Agents.
- Mem0: Building Production-Ready AI Agents with Scalable Long-Term Memory.
- MEM1: Learning to Synergize Memory and Reasoning for Efficient Long-Horizon Agents.
- Governing Evolving Memory in LLM Agents: SSGM.
- CraniMem: Cranial Inspired Gated and Bounded Memory for Agentic Systems.
- Auto-Dreamer: Learning Offline Memory Consolidation for Language Agents.
- MemSkill: Learning and Evolving Memory Skills for Self-Evolving Agents.

### Skill Evolution and Lifelong Agents

- Reflexion: Language Agents with Verbal Reinforcement Learning.
- Voyager: An Open-Ended Embodied Agent with Large Language Models.
- ExpeL: LLM Agents Are Experiential Learners.
- LifelongAgentBench: Evaluating LLM Agents as Lifelong Learners.
- Building Self-Evolving Agents via Experience-Driven Lifelong Learning.
- EvolveR: Self-Evolving LLM Agents through an Experience-Driven Lifecycle.
- Reinforcement Learning for Self-Improving Agent with Skill Library / SAGE.
- AutoSkill: Experience-Driven Lifelong Learning via Skill Self-Evolution.
- SkillX: Automatically Constructing Skill Knowledge Bases for Agents.
- MUSE-Autoskill: Self-Evolving Agents via Skill Creation, Memory, Management, and Evaluation.

