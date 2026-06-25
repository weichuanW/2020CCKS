# 产业视角：Agent Trajectory / Trace Evaluation

> Date: 2026-06-24
>
> Focus: 当前产业界如何做 LLM agent 的 trajectory、trace、span、tool-call、session 级评估；覆盖产品文档、博客、视频/课程、论文与标准。
>
> Relation to previous notes: 本文是 `agentic-evaluation-memory-skill-roadmap.md` 的产业补充。前文偏学术论文与研究脉络；本文偏生产系统、工具链、CI、online monitoring、human feedback、judge calibration 和落地 roadmap。

## 0. Executive Summary

产业界对 agent trajectory evaluation 的观点已经比较清晰：

1. **Trace-first**：先把 agent run 记录成结构化 trace/span/session，否则无法稳定 debug、eval、monitor。
2. **Final answer 不够**：agent 可能最终答对，但中间路径低效、重复、危险、违反政策或产生副作用。
3. **Eval 要挂到 trace 上**：score 不只是最终样本分数，还要能挂在 trace、span、tool call、retriever、guardrail、session、thread 上。
4. **Deterministic + LLM judge + human label 混合**：工具 schema、路径匹配、成本延迟用确定性指标；语义质量、任务完成、root cause 用 LLM judge；高风险样本用人工校准。
5. **CI + online monitoring 闭环**：离线 golden dataset 用于 regression gate；线上 trace 采样打分，失败样本回流 annotation queue 和 regression set。
6. **标准化正在形成**：OpenTelemetry GenAI semantic conventions、OpenInference、OpenLLMetry/Traceloop 正在成为跨框架 trace 互操作基础。
7. **产业目标不是 leaderboard，而是发布治理**：每次 prompt/model/tool/agent graph 改动都要能回答：质量是否退化、成本是否超预算、是否引入新风险、失败根因在哪里。

一个典型生产闭环是：

```text
Instrument agent with OTel/OpenInference
  -> store traces in LangSmith / Braintrust / Phoenix / Langfuse / Weave / HoneyHive
  -> run offline eval on golden datasets in CI
  -> deploy with online async scoring and alerts
  -> send ambiguous or high-risk traces to human annotation
  -> calibrate judge and update regression set
  -> optimize prompts, tool schemas, routing, memory, skills, or reward models
```

---

## 1. 产业界的核心问题定义

### 1.1 产业界为什么关心 trajectory，而不是只看 final output

在生产 agent 中，final output 往往无法暴露以下问题：

- 调用了错误工具但最后靠猜测答对。
- 工具参数含幻觉字段或危险默认值。
- 重复调用同一个工具形成 loop，成本和延迟异常。
- 先执行了有副作用的错误操作，再补偿性修正。
- 违反业务政策但最终状态看起来正确。
- 检索、记忆读写或 handoff 发生错误，下游模型掩盖了问题。
- 多 agent 系统中真正失败的是上游 planner 或某个 sub-agent，而不是最终 responder。

因此产业界更常把一次 agent run 拆成：

```text
session / thread / trace
  -> agent span
    -> planner / router span
    -> LLM span
    -> tool span
    -> retriever span
    -> memory read/write span
    -> guardrail span
    -> evaluator span
```

每一层都可以被评分、聚合、报警和回放。

### 1.2 产业界更看重的指标

与学术 benchmark 相比，产业界更关注：

- **Regression**：新版本是否比旧版本差。
- **Reliability**：同一任务多次运行是否稳定。
- **Cost/latency**：token、工具调用、p95 latency、重试次数是否可控。
- **Safety/policy**：是否违反业务流程、隐私、合规、权限。
- **Debuggability**：能否定位第一个关键失败 span。
- **Human calibration**：LLM judge 是否被人工 gold labels 校准。
- **Release governance**：能否把 eval 变成 CI gate 和上线监控。

---

## 2. 产业工具与平台地图

| 工具/组织 | 核心观点 | Trace / trajectory 表示 | Eval 覆盖 | CI / online monitoring |
|---|---|---|---|---|
| LangSmith / LangChain / AgentEvals | agent 改进循环从 trace 开始；评估消息序列、工具调用和图路径 | LangSmith run/trace；AgentEvals 用 OpenAI message dict 或 LangChain messages；LangGraph 记录 graph trajectory | strict/unordered/subset/superset trajectory match；tool args；LLM-as-judge；latency/cost；human annotation | `evaluate()`、pytest、GitHub Actions、online evaluators、annotation queues |
| OpenAI Agents / Evals | 调试先看 traces，再沉淀 datasets/eval runs；trace grading 给决策、工具、handoff、guardrail 打分 | Agents SDK tracing：model calls、tool calls、handoffs、guardrails、custom spans | trace grading、tool/handoff/guardrail score、model-based grader、structured eval | eval runs、trace grading、observability integrations |
| Anthropic | agent eval 应定义 task、trial、grader、transcript/trace、harness；要组合 code/model/human graders | transcript / trace / trajectory 作为审查对象 | task success、tool use、policy、rubric、人类 review；强调读 transcript 校验 grader | eval harness、human review、grader calibration |
| Braintrust | eval、trace、production scoring、human review 是闭环 | nested trace/span/thread；span 可为 task/tool/score/automation | custom scorer、LLM judge、span/trace score、latency/cost/error、human review | `bt eval`、GitHub Action、pytest、online scoring |
| W&B Weave | instrument everything；把 trace、eval、dataset、feedback 放入实验系统 | agent -> session -> turn -> LLM/tool spans；OTel spans | custom scorer、LLM judge、hallucination/toxicity/relevance、token/cost/latency、feedback | `weave.Evaluation`、production monitors/signals |
| Arize Phoenix / OpenInference | 标准化 AI trace；span kind 表达 LLM/agent/tool/retriever/evaluator | OpenInference on OTel；span tree with LLM/AGENT/TOOL/RETRIEVER/GUARDRAIL/EVALUATOR | Tool Selection、Tool Invocation、RAG evals、LLM judge、latency/token/cost | Phoenix datasets/experiments、server evals、trace annotations |
| Langfuse | OTel-native 开源/自托管；score 可挂 trace、observation、session、dataset run | trace + observation/span + session；SDK/OTel/LangChain/Vercel AI/LlamaIndex | LLM judge、code evaluator、manual labels、user feedback、自定义 numeric/categorical score | dataset experiments、GitHub Action、online evaluators、regression gate |
| HoneyHive | wide-event model；把 trace/log/metric/feedback 统一为 event | root session event；child model/tool/chain events | code evaluator、LLM-assisted evaluator、human evaluator、step/session eval | `evaluate()`、`compare_runs()`、CI regression、online eval |
| Humanloop | eval 是 LLM app 的单元测试；online/offline evaluator 一体 | agent execution log + nested LLM/tool traces | AI evaluator、code evaluator、human evaluator、PII/safety、factuality、tone、latency/cost | datasets、online monitoring、quality gates |
| Confident AI / DeepEval | pytest-style agent eval；分 end-to-end、trajectory、component/span 三级 | `@observe` trace；LLM/tool/retriever/sub-agent spans | Task Completion、Tool Correctness、Argument Correctness、Step Efficiency、Plan Adherence、Plan Quality、RAG/safety | `deepeval test run`、pytest、CI reports |
| Ragas | 从 RAG eval 扩展到 agent/tool metrics，适合作为 metric library | `MultiTurnSample`，可含 reference tool calls 和 reference outcome | ToolCallAccuracy、ToolCallF1、AgentGoalAccuracy、Topic Adherence | 嵌入 CI、MLflow、LangSmith、Weave、Phoenix |
| Helicone | 低侵入 proxy/gateway observability，强在 cost/latency/session analytics | session id/path/name；distributed parent-child traces | latency、cost、token、success proxy、feedback、custom properties、request score | production monitoring、alerts、score/feedback API |
| AgentOps.ai | agent replay/debug/cost/session monitoring | session / waterfall / decorators: trace, agent, tool, operation | token/cost/latency、tool usage、errors、custom metrics | online dashboard、session replay；CI eval 相对弱 |
| LlamaIndex | 框架层 instrumentation，导出到 Phoenix/Langfuse/Weave/OTel 后端 | dispatcher、event handler、span handler；workflow/agent/tool events | 自身偏 instrumentation；eval 常组合 Phoenix/Ragas/DeepEval/Langfuse | OTel exporter，CI 通常由外部平台承担 |
| Microsoft AutoGen / Magentic-One / AgentEval | 离线 benchmark、可重复 run、完整 logs、multi-agent debugging | console logs、agent message JSON、artifact、multi-agent traces | task completion、criteria utility、root-cause、step constraints | AutoGenBench、Docker 隔离、重复运行、tabulate |
| Patronus AI / Percival / TRAIL | agent trace debugging 和错误 taxonomy 很关键 | structured agent traces，支持自定义 failure taxonomy | reasoning/planning/system execution errors、RAG/safety evaluators | trace debugger、prompt fix suggestions、human taxonomy |

---

## 3. 标准与 Trace Schema

### 3.1 OpenTelemetry GenAI semantic conventions

OpenTelemetry 正在把 GenAI 和 agent 运行纳入通用 observability 体系。关键属性包括：

- `gen_ai.operation.name`
  - `chat`
  - `text_completion`
  - `embeddings`
  - `invoke_agent`
  - `execute_tool`
  - `retrieval`
  - `invoke_workflow`
- `gen_ai.provider.name`
- `gen_ai.request.model`
- `gen_ai.response.model`
- `gen_ai.usage.input_tokens`
- `gen_ai.usage.output_tokens`
- `gen_ai.conversation.id`
- `gen_ai.tool.name`
- `gen_ai.tool.call.id`
- `gen_ai.tool.type`

生产建议：

- prompt、messages、tool args、tool results 默认不要无差别写入 trace；应脱敏、采样或用外部对象引用。
- tool call 应作为独立 span，便于计算 latency、error、side effect 和参数质量。
- evaluator 自身也应作为 span，记录 judge model、judge prompt、score、confidence、explanation 和版本。

### 3.2 OpenInference

OpenInference 是建立在 OpenTelemetry 上的 AI-specific semantic convention。常用 span kind：

- `LLM`
- `AGENT`
- `CHAIN`
- `TOOL`
- `RETRIEVER`
- `RERANKER`
- `EMBEDDING`
- `GUARDRAIL`
- `EVALUATOR`
- `PROMPT`

它的价值在于：不同框架产生的 trace 可以被 Phoenix、Langfuse、Weave、LlamaIndex 等工具较一致地渲染和评估。

### 3.3 OpenLLMetry / Traceloop

OpenLLMetry 基于 OpenTelemetry 自动 instrument OpenAI、Anthropic、LangChain、vector DB 等组件。对已有 Datadog、Honeycomb、New Relic、Grafana 的团队，它提供了接入现有 APM 的低成本路径。

### 3.4 推荐的生产 trace schema

```text
Trace / Session
- trace_id
- session_id / conversation_id
- user_cohort / environment / release_sha
- agent_name / agent_version
- prompt_template_name / prompt_version
- model / model_version / sampling_config
- task_id / scenario_id / eval_suite_id
- policy_version / tool_registry_version

Span
- span_id / parent_span_id
- span_kind: AGENT / LLM / TOOL / RETRIEVER / GUARDRAIL / EVALUATOR
- operation_name: invoke_agent / chat / execute_tool / retrieval
- input_ref / output_ref
- token_usage / latency / cost
- tool_name / tool_args_ref / tool_result_ref
- state_before_ref / state_after_ref / state_diff_ref
- error_type / exception / retry_count
- safety_flags / pii_redaction_status

Evaluation
- evaluator_name / evaluator_version
- evaluator_type: code / llm_judge / human / reward_model
- score_dimension: success / side_effect / repetition / policy / groundedness / efficiency / safety
- score_value
- confidence / abstain
- explanation_ref
- human_label_id / annotator_role
- calibration_set_version
```

---

## 4. Metric Taxonomy

### 4.1 End-to-end outcome metrics

- Task Completion
- Goal Accuracy
- final state correctness
- scenario success
- pass/fail
- pass^k / repeated-run consistency

适合：

- deterministic API/app tasks
- customer-service workflows
- code-generated tasks with tests
- web tasks with state checks

代表：

- tau-bench `pass^k`
- AppWorld state-based unit tests
- WebArena functional correctness
- DeepEval Task Completion
- Ragas AgentGoalAccuracy

### 4.2 Tool selection metrics

评估是否调用了正确工具，是否不该调用工具时避免调用。

代表：

- Phoenix Tool Selection
- DeepEval Tool Correctness
- Ragas ToolCallAccuracy
- TRAJECT-Bench tool selection

### 4.3 Tool invocation / argument metrics

评估参数是否：

- schema valid
- complete
- semantically correct
- non-hallucinated
- safe
- policy-compliant

代表：

- Phoenix Tool Invocation
- DeepEval Argument Correctness
- TRAJECT-Bench argument correctness

### 4.4 Path / trajectory matching metrics

评估工具序列、消息序列、graph node path 是否符合预期。

匹配模式：

- strict exact path
- unordered set match
- subset match
- superset match
- dependency/order satisfaction
- DFA valid path

代表：

- LangChain AgentEvals
- CORE
- TRAJECT-Bench

### 4.5 Step-level / span-level metrics

对每个 span 单独打分：

- first critical failure step
- retry/loop
- latency/cost outlier
- hallucinated observation
- invalid state transition
- unsafe side effect
- guardrail failure

代表：

- TRAIL
- AgentRx
- Braintrust span score
- Langfuse observation score
- HoneyHive event score
- DeepEval span metrics

### 4.6 Efficiency and reliability metrics

- total steps
- tool call count
- duplicate tool call count
- loop depth
- p50/p95 latency
- token cost
- retry count
- timeout rate
- error rate
- pass^k consistency

### 4.7 Safety and policy metrics

- PII leakage
- prompt injection susceptibility
- unsafe tool call
- unauthorized action
- policy violation
- harmful side effect
- escalation correctness
- refusal correctness

### 4.8 Human and judge calibration metrics

- judge precision / recall
- false positive rate on failed trajectories
- abstain rate
- human disagreement rate
- pairwise preference consistency
- calibration drift over time

---

## 5. Judge Calibration and Human Labeling

### 5.1 产业共识

AgentRewardBench、Anthropic、Braintrust、LangSmith 等资料共同指向一点：

> LLM judge 有用，但不能直接相信；必须用 human labels、domain rubrics 和回归集校准。

### 5.2 校准原则

1. **按任务域校准 judge**
   - 不要一个全局 judge 打所有任务。
   - Web、客服、API、桌面、代码、检索任务需要不同 rubrics。

2. **优先优化 precision**
   - 如果 judge 用于 reward model 或自动上线，false positive 代价很高。
   - 把失败 trajectory 判成成功，会污染后续训练和 regression set。

3. **引入 abstain / escalation**
   - cheap judge -> strong judge -> human review。
   - 低 confidence 或高风险 trace 自动进入人工队列。

4. **多维度打分**
   - 不要只用 overall score。
   - 至少拆成 success、tool correctness、argument correctness、policy、side effect、efficiency、safety、adaptivity。

5. **人工读 trace 校验 judge**
   - 定期审查 high-score failures 和 low-score successes。
   - 保留 judge prompt、judge model、judge output explanation。

### 5.3 推荐标注粒度

- trace-level：是否完成任务、是否违反政策、是否有副作用。
- span-level：哪一步首次出错、错误类型、严重度。
- pairwise：两个 trajectory 哪个更好。
- rubric score：reliability、security、instruction adherence、plan optimality。
- free-text RCA：根因描述，用于聚类和 prompt/tool 改进。

### 5.4 标注闭环

```text
production trace sample
  -> automatic clustering / deduplication
  -> LLM pre-label
  -> domain expert review
  -> disagreement resolution
  -> gold label set
  -> judge calibration
  -> CI regression suite
  -> reward model / verifier training
```

---

## 6. CI, Online Monitoring, and Optimization Roadmap

### Phase 0: 定义任务和风险

- 列出 top 20-50 个真实用户任务。
- 为每类任务定义 success、failure、side effect、policy violation。
- 区分：
  - 可 final-state check 的任务。
  - 必须 trajectory check 的任务。
  - 必须人工审查的高风险任务。

### Phase 1: 统一 trace

- 接入 OpenTelemetry + OpenInference。
- 统一 span kind：AGENT / LLM / TOOL / RETRIEVER / GUARDRAIL / EVALUATOR。
- 每条 trace 绑定：
  - release SHA
  - prompt version
  - model version
  - tool registry version
  - policy version
- 内容脱敏或用对象引用存储。

### Phase 2: 建立 offline eval harness

- 建 golden dataset：
  - 真实生产失败。
  - 人工构造边界条件。
  - 合成 hard negatives。
  - 核心 happy paths。
- 每个任务至少一个 deterministic grader。
- 开放式任务增加 LLM judge，但初期不 block。
- CI 先跑小型 smoke suite。

### Phase 3: Human labels and judge calibration

- 建 annotation queue。
- 每周抽样 production traces 和 CI failures。
- 校准 judge precision、recall、abstain rate。
- 对高风险维度使用 cascaded judge + human escalation。

### Phase 4: Online monitoring

- 线上异步 evaluator，不阻塞用户请求。
- 对高风险流量全量评分，对普通流量抽样。
- dashboard 跟踪：
  - success proxy
  - policy violation
  - tool error rate
  - repeated tool calls
  - cost/token drift
  - latency drift
  - judge score drift
  - human escalation rate
- 异常 trace 自动进入 regression dataset。

### Phase 5: Trajectory-aware optimization

- 引入 CORE 风格 path metrics：
  - valid path
  - harmful calls
  - prefix criticality
  - efficiency
- 引入 TRAIL/AgentRx 风格 root-cause taxonomy。
- 用 human-labeled + judge-filtered traces 训练 verifier / reward model。
- 优化：
  - prompt
  - tool schema
  - tool routing
  - memory retrieval
  - skill library
  - agent graph
  - model selection
  - RL / DPO / RFT

### Phase 6: Mature release governance

- 每个 evaluator 有：
  - owner
  - version
  - calibration set
  - expected operating range
  - known failure modes
- 每次模型、prompt、tool、agent graph 升级自动跑 full eval。
- 高风险 agent 上线前必须通过：
  - deterministic checks
  - calibrated judge
  - human spot review
  - canary monitoring

---

## 7. 与前一份学术路线图的合并关系

### 7.1 学术论文给出 metric 和 benchmark 原型

- AgentRewardBench -> judge calibration 和 human-labeled trajectories。
- TRAIL -> trace debugging 和 error taxonomy。
- CORE -> formal path safety/efficiency metrics。
- TRACE -> reference-free reasoning trajectory judge。
- TRAJECT-Bench -> tool selection、arguments、order/dependency metrics。
- tau-bench -> pass^k reliability。
- AppWorld/WebArena/OSWorld/WorkArena -> realistic task environments。

### 7.2 产业工具把这些能力产品化

- LangSmith / AgentEvals -> trajectory matching 和 annotation queue。
- Braintrust / HoneyHive / Langfuse -> CI regression + online scoring。
- Phoenix / OpenInference -> OTel span schema 和 tool invocation eval。
- DeepEval / Ragas -> agent metric library。
- Weave / Helicone / AgentOps -> production traces、cost、latency、session replay。
- Humanloop / Patronus -> human feedback、安全评估和错误 taxonomy。
- AutoGen / AgentEval / AgentRx -> multi-agent benchmark 和 root-cause debugging。

### 7.3 当前最佳实践组合

一个较成熟的工程组合通常是：

```text
OTel/OpenInference instrumentation
  + trace store / observability platform
  + deterministic graders for state and tool schema
  + LLM judges for semantic and trajectory quality
  + human annotation for calibration
  + CI regression gate
  + online async scoring
  + failure trace replay and root-cause taxonomy
```

---

## 8. Concrete Resource List

### 8.1 Product docs and technical guides

| Resource | Source | Link | Why it matters |
|---|---|---|---|
| How to evaluate your agent with trajectory evaluations | LangSmith | https://docs.langchain.com/langsmith/trajectory-evals | Direct official guide for deterministic and LLM-judge trajectory evals. |
| Agent Evals | LangChain | https://docs.langchain.com/oss/python/langchain/evals | Shows `agentevals` usage for strict/unordered/subset/superset path matching. |
| Evaluate agent workflows | OpenAI | https://developers.openai.com/api/docs/guides/agent-evals | Official OpenAI view on agent eval surfaces and trace-driven debugging. |
| Trace grading | OpenAI | https://developers.openai.com/api/docs/guides/trace-grading | Directly covers grading decisions, tool calls, handoffs, and guardrails. |
| Integrations and observability | OpenAI Agents SDK | https://developers.openai.com/api/docs/guides/agents/integrations-observability | Shows tracing of model calls, tool calls, handoffs, guardrails, custom spans. |
| Demystifying evals for AI agents | Anthropic Engineering | https://www.anthropic.com/engineering/demystifying-evals-for-ai-agents | One of the clearest industry methodology posts on tasks, trials, graders, traces. |
| Evaluate systematically | Braintrust | https://www.braintrust.dev/docs/evaluate | Practical eval/experiment framework with CI integration. |
| Score production traces | Braintrust | https://www.braintrust.dev/docs/evaluate/score-online | Online scoring of production traces. |
| Examine traces | Braintrust | https://www.braintrust.dev/docs/observe/examine-traces | Trace/span inspection and debugging workflow. |
| Trace your agents | W&B Weave | https://docs.wandb.ai/weave/guides/tracking/trace-agents | Agent -> session -> turn -> LLM/tool span model. |
| Weave scorers | W&B Weave | https://docs.wandb.ai/weave/guides/evaluation/scorers | Custom scorers and LLM judges over traces/calls. |
| OpenInference spec | Arize / OpenInference | https://arize-ai.github.io/openinference/spec/ | AI semantic conventions on top of OpenTelemetry. |
| Evaluate an agent | Arize Phoenix | https://arize.com/docs/phoenix/cookbook/evaluation/evaluate-an-agent | Phoenix cookbook for agent evals over traces. |
| Tool invocation evaluator | Arize Phoenix | https://arize.com/docs/phoenix/evaluation/server-evals/pre-built-metrics/tool-invocation | Evaluates tool call parameters, hallucinations, and formatting. |
| Langfuse observability | Langfuse | https://langfuse.com/docs/observability/get-started | Open-source/self-hosted tracing and eval platform. |
| Langfuse experiment CI/CD | Langfuse | https://langfuse.com/docs/evaluation/experiments/experiments-ci-cd | Regression gate for LLM/agent experiments. |
| HoneyHive evaluation intro | HoneyHive | https://docs.honeyhive.ai/v2/evaluation/introduction | Wide-event evaluation model. |
| HoneyHive CI regression | HoneyHive | https://docs.honeyhive.ai/v2/evaluation/ci-regression-detection | Concrete CI regression detection flow. |
| DeepEval agent quickstart | Confident AI / DeepEval | https://deepeval.com/docs/getting-started-agents | Pytest-style agent trajectory evaluation. |
| DeepEval CI/CD | Confident AI | https://documentation.confident-ai.com/llm-evaluation/evaluation-features/unit-testing-in-cicd | Unit testing and CI gate for LLM apps. |
| Ragas agent metrics | Ragas | https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/agents/ | AgentGoalAccuracy, ToolCallAccuracy, ToolCallF1. |
| LlamaIndex observability | LlamaIndex | https://developers.llamaindex.ai/python/framework/module_guides/observability/ | Instrumentation and OTel export for LlamaIndex agents/workflows. |
| AutoGen tracing | Microsoft AutoGen | https://microsoft.github.io/autogen/dev/user-guide/agentchat-user-guide/tracing.html | OTel tracing for multi-agent systems. |

### 8.2 Blogs and thought pieces

| Resource | Source | Link | Why it matters |
|---|---|---|---|
| The Agent Improvement Loop Starts with a Trace | LangChain | https://www.langchain.com/blog/traces-start-agent-improvement-loop | Frames trace as the center of agent improvement loop. |
| What is agent observability? Tracing tool calls, memory, and multi-step reasoning | Braintrust | https://www.braintrust.dev/articles/agent-observability-tracing-tool-calls-memory | Explains why ordinary LLM logs are not enough for agents. |
| What is agent evaluation? | Braintrust | https://www.braintrust.dev/articles/agent-evaluation | Practical taxonomy of task, simulation, success criteria, and trajectory metrics. |
| How to evaluate multi-turn conversations | Braintrust | https://www.braintrust.dev/blog/multi-turn-scoring | Turn-level vs session-level scoring for multi-turn agents. |
| Add Observability to Your Open Agent Spec Agents with Arize Phoenix | Arize | https://arize.com/blog/add-observability-to-your-open-agent-spec-agents-with-arize-phoenix/ | Shows OTel/OpenInference-based tracing and evaluations. |
| How to evaluate tool-calling agents | Arize | https://arize.com/blog/how-to-evaluate-tool-calling-agents/ | Focuses on tool selection and invocation quality. |
| Mastering AI agent observability | W&B | https://wandb.ai/site/articles/ai-agent-observability/ | Industry view of traces, metrics, evaluations, and governance. |
| LLM monitoring | Humanloop | https://humanloop.com/blog/llm-monitoring | Connects production logs, monitoring, evaluations, and CI/CD gates. |
| LLM observability tutorial and best practices | Patronus AI | https://www.patronus.ai/llm-testing/llm-observability | Focus on complete traces, safety evaluators, and diagnostics. |
| AgentEval developer tool | Microsoft AutoGen | https://microsoft.github.io/autogen/0.2/blog/2024/06/21/AgentEval/ | Early multi-agent evaluation methodology. |
| AgentRx framework | Microsoft Research | https://www.microsoft.com/en-us/research/blog/systematic-debugging-for-ai-agents-introducing-the-agentrx-framework/ | Systematic debugging and root-cause analysis for agents. |

### 8.3 Videos and courses

| Resource | Source | Link | Why it matters |
|---|---|---|---|
| The Agent Development Lifecycle: Build, Test, Deploy, Monitor | LangChain / Interrupt 26 | https://www.youtube.com/watch?v=jWy39wavbjY | High-level product view of trace-centered agent lifecycle. |
| Building and Testing Reliable Agents | LangChain / AI Engineer World Fair | https://www.youtube.com/watch?v=XiySC-d346E | Practical talk on LangGraph/LangSmith and trajectory-aware testing. |
| Evaluating AI Agents | DeepLearning.AI / Arize AI course | https://www.deeplearning.ai/courses/evaluating-ai-agents | Course covering tracing, router/skill evals, trajectory evals, LLM-as-judge, monitoring. |

### 8.4 Papers and benchmarks that connect directly to industry practice

| Paper / benchmark | Link | Industrial relevance |
|---|---|---|
| AgentRewardBench | https://arxiv.org/abs/2504.08942 | Judge calibration with expert-labeled web-agent trajectories. |
| TRAIL | https://arxiv.org/abs/2505.08638 | OpenTelemetry/OpenInference-like trace debugging and error taxonomy. |
| CORE | https://arxiv.org/abs/2509.20998 | Formal path correctness, harmful-call rate, and efficiency metrics. |
| TRACE / Beyond the Final Answer | https://arxiv.org/abs/2510.02837 | Reference-free trajectory evaluation using evidence banks. |
| TRAJECT-Bench | https://arxiv.org/abs/2510.04550 | Tool selection, arguments, order, and dependency metrics at scale. |
| tau-bench | https://arxiv.org/abs/2406.12045 | Customer-service agent reliability with pass^k. |
| AppWorld | https://arxiv.org/abs/2407.18901 | Multi-app API/coding-agent sandbox with state-based tests. |
| WebArena | https://arxiv.org/abs/2307.13854 | Realistic web-agent task environment. |
| WorkArena / WorkArena++ | https://arxiv.org/abs/2403.07718 | Enterprise SaaS workflow automation benchmark. |
| OSWorld | https://arxiv.org/abs/2404.07972 | Computer-use agent benchmark with execution-based checks. |
| AgentEval | https://aclanthology.org/2024.emnlp-main.1219.pdf | Multi-agent application utility criteria generation and quantification. |

---

## 9. Concrete Checklist for a Team Building Agent Trajectory Evaluation

### Week 0 style bootstrap, without calendar commitment

- [ ] Define top real user tasks and risk classes.
- [ ] Pick trace schema: OTel GenAI + OpenInference-compatible.
- [ ] Instrument LLM, tool, retriever, memory, guardrail, evaluator spans.
- [ ] Add deterministic graders for tool schema, final state, and policy.
- [ ] Add LLM judge only after writing rubrics and sampling human labels.
- [ ] Create golden dataset from production failures and hand-authored cases.
- [ ] Add CI smoke eval with baseline comparison.
- [ ] Add online async scoring and dashboards for cost/latency/tool errors.
- [ ] Build annotation queue for ambiguous, high-risk, and regression traces.
- [ ] Version every prompt, model, tool registry, policy, evaluator, and dataset.
- [ ] Periodically promote production failures to regression tests.
- [ ] Track judge drift and recalibrate against human labels.

### Minimal metric set

If only starting with five metrics:

1. task success / goal accuracy
2. tool correctness
3. argument correctness
4. step efficiency / repeated tool calls
5. policy or side-effect violation

If adding five more:

6. pass^k reliability
7. p95 latency
8. token/cost per successful task
9. first critical failure step
10. human-calibrated judge confidence / abstain rate

---

## 10. Bottom Line

The current industry view is not that trajectory evaluation is a separate research artifact. It is becoming the operational backbone of production agents:

```text
trace = observability primitive
trace + evaluator = quality signal
quality signal + CI = release gate
quality signal + online monitor = production safety net
human labels + judge calibration = trusted automation
failed traces + regression set = continuous improvement
```

The next mature agent stacks will treat every agent trajectory as a reusable data asset: debuggable, scoreable, replayable, comparable, annotatable, and eventually trainable.

