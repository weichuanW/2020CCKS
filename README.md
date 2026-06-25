# Agentic Evaluation Research Notes

Research reading notes and paper collection on **agentic evaluation**, **trajectory/trace evaluation**, **memory stabilization**, and **skill evolution** for LLM agents.

## Contents

| File | Description |
|---|---|
| [agentic-evaluation-memory-skill-roadmap.md](./agentic-evaluation-memory-skill-roadmap.md) | Academic survey covering benchmarks, trajectory evaluation, memory systems, and lifelong skill evolution (English) |
| [industry-agent-trajectory-evaluation.md](./industry-agent-trajectory-evaluation.md) | Industry-focused survey on production trace evaluation, tooling, and operational workflows (中文) |
| [papers/](./papers/) | Downloaded PDF copies of referenced papers; see [papers/manifest.md](./papers/manifest.md) for the full list |

## Core Theme

Modern agent systems can be viewed as a loop:

```text
Agent executes a task
  -> produces trajectory / trace
  -> evaluation diagnoses success, failure, safety, and efficiency
  -> memory stabilization decides what to store and how to prevent drift
  -> skill evolution distills experience into reusable capability
  -> the next run retrieves memory and skills to improve behavior
```

The research trend is shifting from *"Did the agent finish?"* to *"Did the agent finish through a correct, safe, efficient, explainable, and reusable path?"*

## Topics Covered

- **Evaluation**: AgentBench, WebArena, AgentBoard, AgentRewardBench, TRAIL, CORE, TRACE, TRAJECT-Bench, and related benchmarks
- **Memory**: MemGPT, Mem0, A-Mem, SSGM, CraniMem, Auto-Dreamer, MemSkill, and related work
- **Skill evolution**: Reflexion, Voyager, ExpeL, SAGE, AutoSkill, SkillX, MUSE-Autoskill, and related work
- **Industry tooling**: LangSmith, Braintrust, Arize Phoenix, Langfuse, W&B Weave, OpenTelemetry / OpenInference, and more

## Repository Layout

```text
.
├── README.md
├── agentic-evaluation-memory-skill-roadmap.md
├── industry-agent-trajectory-evaluation.md
└── papers/
    ├── manifest.md
    └── *.pdf
```

## Notes

- Several 2026 papers are recent preprints; treat their empirical conclusions as trend signals rather than settled benchmarks.
- PDFs in `papers/` are publicly accessible copies for offline reading. Sources and links are listed in `papers/manifest.md`.

## License

Papers remain the property of their respective authors and publishers. Research notes in this repository are provided for personal reference.
