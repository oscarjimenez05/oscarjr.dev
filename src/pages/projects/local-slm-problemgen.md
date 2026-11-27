---
layout: '../../layouts/Project.astro'
title: Local SLM Problem Generation
date: '2026-03-15'
priority: 9
description: "Replacing expensive cloud LLM APIs with local Small Language Models, deterministic validation pipelines, and a human-in-the-loop review interface for CS education."
badge: "Research"
tags: ["Spring Boot", "Java", "Ollama / SLMs", "ACM SIGCSE"]
---

## Publication

O. Jimenez Rodriguez. "Generating Contextual Programming Exercises via Large Language Models and Microservice Architectures." In *Proceedings of the 57th ACM Technical Symposium on Computer Science Education (SIGCSE 2026)*, St. Louis, MO, 2026. [doi:10.1145/3770761.3777153](https://doi.org/10.1145/3770761.3777153).

## The Core Problem
Introductory Computer Science courses burn through a lot of practice problems. Writing fresh coding exercises, edge-case unit tests, and tailored feedback for every topic takes hours of faculty labor. 

Cloud LLM APIs (like GPT-4) can generate exercises on demand, but relying on proprietary cloud endpoints for university coursework brings recurring API bills, rate limits, data-privacy headaches, and zero guarantee that the generated code actually compiles or tests what the professor intended.

During my **Summer 2025 Research Fellowship** at Ithaca College (advised by Prof. Toby Dragon), we set out to answer a simple question: *Can a locally hosted Small Language Model (SLM) replace cloud APIs to generate reliable, curriculum-aligned programming problems?*

---

## System Architecture
Instead of a fragile Python script glued to an API key, I engineered a containerized, full-stack microservice ecosystem designed to integrate into an existing educational platform (**TECMap**).

```text
┌──────────────────────┐       ┌───────────────────────────────────────────┐
│   React Frontend /   │──────▶│         Spring Boot Microservice          │
│   Instructor UI      │       │  (AIController / DataController / H2 SQL) │
└──────────────────────┘       └─────────────────────┬─────────────────────┘
                                                     │
                         ┌───────────────────────────┴───────────────────────────┐
                         ▼                                                       ▼
        ┌─────────────────────────────────┐                     ┌─────────────────────────────────┐
        │      Local Ollama Engine        │                     │   OpenAI Cloud Fallback / Eval  │
        │  (Custom Modelfile Pipelines)   │                     │   (Benchmark Baseline)          │
        └─────────────────────────────────┘                     └─────────────────────────────────┘

```

### The Stack

- **Backend Core:** Java & **Spring Boot** `(MicroserviceApp)`, exposing REST endpoints via `AIController` and `DataController` with relational persistence `(AssessmentItem`, `AssessmentRecord`, `LearningMaterial)` backed by an embedded SQL database.
- **Local Inference Engine:** Containerized **Ollama** `(Dockerfile.ollama)` running specialized, task-isolated **Modelfiles** `(cs-topicScanner`, `cs-problemGenerator`, `cs-problemSolver`, `cs-syntaxChecker`, `cs-problemGrader`, `cs-feedbackGenerator)`.
- **Frontend & Reviewer Portal:** A React client `(frontend)` served via Nginx alongside a dedicated Spring Boot + Thymeleaf **Instructor Verification App** `(reviewer)` for human-in-the-loop benchmarking and approval.
- **Orchestration:** Multi-container Docker Compose topology wiring the backend, frontend, local inference engine, and review portal together.

---

## The CHASE Multi-Agent Refinement Loop
Small Language Models (4-bit quantized, ≤8B parameters running on a standard laptop with an Intel Core i7 and 16GB RAM) are fast and private, but left unchecked, they tend to generate trivial problems or hallucinate broken constraints.

To push algorithmic depth without sacrificing solvability, we adapted a **CHASE-style multi-agent refinement loop** using specialized Ollama Modelfiles:

```text
┌───────────────────┐     ┌───────────────────┐     ┌───────────────────────┐
│     Generator     │────▶│       Hider       │────▶│       Verifier        │
│ Qwen2.5-Coder 7B  │     │ Qwen2.5-Coder 1.5B│     │    Granite-3.3 2B     │
│  (Seed Problem)   │     │ (Obscure Context) │     │ (Solvability/Filters) │
└───────────────────┘     └─────────▲─────────┘     └───────────┬───────────┘
                                    │                           │
                          If Too Easy (Loop)                    ▼
                          ┌─────────┴─────────┐     ┌───────────────────────┐
                          │     Extractor     │◀────│         Zoner         │
                          │ Qwen2.5-Coder 3B  │     │   Qwen2.5-0.5B        │
                          │ (Clean Artifacts) │     │ (Rate Difficulty 1–5) │
                          └───────────────────┘     └───────────────────────┘
                               If Target Met
```

1. **Generator `(Qwen2.5-Coder 7B)`:** Produces the initial seed problem `(Temp: 0.4, Top_p: 0.92)`.
2. **Hider `(Qwen2.5-Coder 1.5B)`:** Rewrites the prompt to obscure direct hints and surface-level shortcuts, moving key operational facts into a hidden `CONTEXT` block.
3. **Verifier `(Granite-3.3 2B)`:** Runs deterministically `(Temp: 0, Top_p: 1)` to verify that the problem is still solvable given `CONTEXT` and that no excluded topics leaked in.
4. **Zoner `(Qwen2.5-0.5B)`:** Rates the difficulty from `1–5`. If it falls short of the target difficulty, it kicks the problem back to the **Hider** for another refinement pass.
5. **Extractor `(Qwen2.5-Coder 3B)`:** Strips internal reasoning and formatting artifacts before handing the final problem to the user.

---

## Human-in-the-Loop Verification

Even with automated syntax and solvability checks, AI-generated coursework should never go straight to students unreviewed.

We built an administrative **Verification & Review Interface** `(ReviewController)` that allowed a Computer Science educator to blindly evaluate **150 generated problems** (50 per method, spanning CS1 to Advanced Algorithms) across Clarity, Depth, Overall Quality, and Topic Accuracy.

---

## Benchmarking & Key Findings (ACM SIGCSE 2026)

We benchmarked two local strategies—**Default** (single-shot `Qwen2.5-Coder 7B` at `Temp: 1.3)` and the **CHASE Loop**—against a **GPT-4.1** cloud baseline:

| Method | Avg. Latency | Clarity (1–5) | Depth (1–5) | Overall Quality (1–5) | Topic Adherence |
| --- | --- | --- | --- | --- | --- |
| **Default (Local Single-Shot)** | `15.9s` | **4.8** | 3.5 | **4.5** | ~50% |
| **CHASE (Local Multi-Agent)** | `132.4s` | 3.2 | **4.1** | 2.9 | **~85%** |
| **GPT-4.1 (Cloud Baseline)** | `3.2s` | 4.6 | 3.9 | 4.4 | — |

### What We Learned

- **Simple Beats Complex (For Now):** The single-shot local model `(Default)` actually beat GPT-4.1 in both **Clarity (4.8 vs. 4.6)** and **Overall Quality (4.5 vs. 4.4)**. Chaining small 4-bit models together amplified their quirks rather than canceling them out.
- **The Cost of Refinement & Leakage:** While the CHASE loop successfully pushed algorithmic **Depth (4.1)** and **Topic Adherence (~85%)**, it was **~18× slower** and suffered from *answer-key leakage*—where the `Extractor` occasionally failed to strip the solution out of the final problem text.
- **Mode Collapse Under High Temperature:** Even at `Temp = 1.3`, small models lacked true creative variety—asking for *"Sorting Algorithms"* repeatedly yielded Selection Sort across almost every trial.

We presented these findings at the **ACM SIGCSE 2026** Undergraduate Student Research Competition in St. Louis ([DOI: 10.1145/3770761.3777153](https://dl.acm.org/doi/10.1145/3770761.3777153)).

---

## Conference Poster

[![SIGCSE 2026 Research Poster](/images/local-slm-poster.png)](/images/local-slm-poster.png)