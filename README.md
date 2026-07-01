# AI in the Metaverse
This project aims to study how AI agents behave in a dynamic virtual society. The foundation are Large Language Model (LLM)-driven agents capable of autonomously acting and planning. Agents interact with each other, and the environment is governed by a custom-designed set of rules (i.e., constitution) mimicking a modern human society. By blending computational, sociological, and legal perspectives, the project seeks to establish a versatile testbed for interdisciplinary research in AI-driven societies, virtual law, and emergent social behavior. Consequently, this project offers participants from diverse academic disciplines the opportunity to gain insights into the intersection of both; technical AI research as well as broader societal research.

The initial phase (project week) will focus on constructing the foundational infrastructure, including 3D world modeling, backend systems, and legal frameworks while setting the stage for ongoing exploration of societal dynamics in a simulated environment.

## Project Description

- Within the context of a modern large-scale society, we simulate a small section where agents are controlled by Large Language Models (LLMs). The framework of this virtual society (motivational drivers, system dynamics) is carefully crafted and can be changed to test how simulated agents behave under certain circumstances.
- As we cannot simulate every agent in such a society, most systemic societal functions (e.g. law enforcement) are explicitly programmed. Only the "roles" of societal members, which we want to specifically observe, are implemented as language-model-controlled agents
- Due to resource constraints, the world size is fitted for a two-digit amount of agents (no huge city). Non-essential aspects of a simulated world of this size (e.g., long-distance transportation infrastructure) are omitted as they do not add value to the simulation results.
- We utilize differently complex (size, architecture) language models as controllers for the agents to observe their capabilities/competencies in navigating the challenges of a society
- Additionally, users (humans) can join and interact with the simulation and its agents (VR integration, website).

# LLM-Backend

This wiki documents the design, architecture, and implementation of the LLM backend used in the project.

## 1. Introduction

### Purpose and Scope

The LLM backend is responsible for enabling decision-making by AI agents. It operates as an independent service that receives structured state updates from the simulation, constructs model-compatible prompts, performs inference using large language models, and returns structured action decisions. The backend is designed to support experimentation with agent reasoning, memory, personality, and model behavior, and ignores the details of world simulation and rendering.

The primary goal of the backend is to provide a reliable, interpretable, and extensible interface between a simulated environment and large language models acting as agents within that environment.

### Communication Pattern

The backend follows a strict request–response interaction model:

* **Input:** structured JSON describing the current world state, agent state, available actions, and contextual metadata.
* **Processing:** prompt construction, memory integration, model inference, and structured action extraction.
* **Output:** structured JSON containing the agent’s selected action(s), formatted for direct consumption by the simulation.

All communication with the backend occurs over HTTP, and all inputs and outputs are expressed as JSON. The backend does not maintain implicit state across requests beyond what is explicitly stored in its memory subsystem. The frequency of input is determined by the game engine.

## 2. Backend Architecture Overview (High-Level)

The LLM backend is structured as a modular service composed of clearly separated components responsible for request handling, prompt construction, memory management, model interaction, and structured action extraction. The diagram below shows the main components of the LLM backend, and how they interact at a high level.

<img width="1494" height="550" alt="image" src="https://github.com/user-attachments/assets/52ae18ee-6091-44fa-86e2-3ffd10dfcf30" />

* **API Handler (Flask):** entry and exit point for all backend requests
* **Prompter:** constructs structured prompts from input data and memory
* **Memory:** stores and retrieves short-term and long-term agent context
* **Tool Builder:** derives tool/function schemas from available actions
* **LLM Client:** interfaces with external LLM providers via HTTP APIs

### Request Lifecycle

In one request cycle, the backend:

* receives a state update,
* loads memory,
* builds a constrained prompt,
* invokes the appropriate model,
* parses a structured action,
* updates memory,
* returns the decision.

### Memory System

#### Memory Types

The backend distinguishes between **short-term** and **long-term** memory.

Short-term memory stores **recent interaction messages** and is used directly during prompt construction. Characteristics:

* Stored as a rolling window of recent messages.
* Injected into the prompt to preserve local conversational and decision context.
* Subject to truncation and summarization.

Short-term memory is intended to capture _what just happened_.

Long-term memory stores **summarized information** derived from earlier short-term interactions. Characteristics:

* Stored as free-form text summaries.
* Appended over time; never truncated automatically.
* Injected into prompts as high-level contextual background,
* Summarized by the same LLM API Call request logic, by making a request to its agent's LLM Provider.

#### File-Based Storage Layout

Memory is stored **per agent** on disk inside the `memory_store/` directory. For each agent, the following files are maintained:

* `chatlog_window.jsonl` - short-term memory window used for prompting and compaction.
* `chatlog_full.jsonl` - append-only complete interaction log for analysis and debugging.\
  This file is never truncated or summarized.
* `longterm.txt` - long-term memory summaries accumulated over time.

#### Memory Read Path (During a Request)

During each request lifecycle:

1. Long-term memory is loaded in full.
2. Short-term memory is loaded up to a configured maximum number of messages.
3. Both are passed to the prompter for prompt construction.

At this stage:

* memory is read-only,
* no mutation occurs until after a valid model response is obtained.

This guarantees that failed or partial requests do not corrupt the memory state.

#### Memory Write Path (After a Response)

Once a valid response is produced by the LLM:

1. A compact representation of the current game state is logged as a **user message**.
2. The model’s decision (or tool call) is logged as an **assistant message**.
3. Both messages are appended to:
   * the full log (analysis),
   * the short-term window (prompting).

Memory updates occur **only after** successful inference and output parsing. 

#### Memory Compaction and Summarization

To prevent the unbounded growth of short-term memory, the backend applies a **summarization policy**.

**Trigger Condition**

When the number of messages in the short-term window reaches a configured threshold, summarization is triggered automatically.

**Compaction Procedure**

The compaction process proceeds as follows:

1. A batch of older short-term messages is selected.
2. These messages are summarized using the same LLM infrastructure.
3. The summary is appended to long-term memory.
4. The short-term window is rewritten to keep only the most recent messages.

The full chat log remains untouched.

**Failure Handling**

If summarization fails:

* the request lifecycle continues,
* memory is left unchanged,
* a warning is logged.

Summarization is our best-effort optimization, not a critical path, so in some sense can be skipped. 

#### Configuration Parameters

Memory behavior is controlled via centralized configuration values, including:

* maximum number of short-term messages injected into prompts,
* summarization trigger threshold,
* number of messages retained after compaction,
* summarization model parameters (temperature, token limits).

### LLM Client Abstraction

The abstraction enables flexible experimentation with models and deployment setups. The backend is designed to support:

* multiple LLM providers,
* heterogeneous models across agents,
* rapid switching between local and remote deployments.

The LLM client abstraction fulfills it by defining a **common interface** that all providers must implement.

At a high level, the backend treats all LLMs as black boxes that accept a prompt (string), optional tool/function definitions, and return either free-form text or a structured tool/function call.

#### Client Interface

All LLM clients inherit from a common abstract base class that defines the minimal contract required by the backend. Each client must support:

* initialization with a base URL and model identifier,
* text generation via a `generate()` method.

Some clients additionally support:

* structured tool/function calling via a dedicated method.

#### Tool / Function Calling Support

Tool calling is implemented exclusively in the OpenAI-compatible client. When enabled:

1. the backend constructs a tool schema from the available actions,
2. the schema is sent together with the prompt,
3. the model is required to select exactly one tool,
4. the backend parses the returned function name and arguments.

This approach ensures:

* only valid actions can be selected,
* output is structurally constrained,
* hallucinated actions are prevented by design.

If no tool call is returned:

* a fallback mechanism attempts to extract structured JSON from the model’s text output.

#### Project Files Structure

* `utils/vllm`- Code files related to experiments with vllm on the aim2 server
* `full_backend/`- Folder for all main project-related code files
  * `main.py` - everything assembles here; you need to run this file to start the backend
  * `llm_client.py` - llm provider script; provider abstraction (Ollama vs OpenAI-compatible, incl. tool calling)
  * `memory.py` - script with memory implementation helper functions; file-based memory (STM/LTM/logs)
  * `config.py` - script containing configuration settings; routing + limits (provider endpoints, agent→model mapping, memory thresholds)
  * `prompter.py` - prompt builder script; prompt construction + memory wrapping + summary prompt
  * `test_server.py` - small test script to test the connection
* `main-old-implementation-all-files/` - Folder with all previous cohorts' main branch files
* `run_backend.sh` - bash file to start the backend (must be run on the server aim2, can be adapted)

## 3. Prompt Engineering & Decision Logic

The agent’s behavior is governed by a strictly structured system prompt generated for every decision cycle. The prompt is designed to minimize hallucinations, enforce action validity, and ensure goal-aligned behavior under survival and economic constraints `full_backend/prompter.py`.

The prompt is composed of the following sections:

### **System Instruction**

Located at (`prompter.py` → `generate_full_prompt()`) and defines global behavioral rules and priorities. The agent must select exactly one valid action based solely on the provided game state. Survival (hunger avoidance) is always prioritized over economic gain. The agent is forbidden from assuming, inventing, or inferring any actions, items, locations, or states not explicitly present in the input JSON. 

### **Action & Range Constraints**

Explicit rules enforce that actions may only be taken on items listed in the agent's inventory or `vicinity.objects`. Location interaction is strictly gated by the `range` field, allowing entry or interaction only when the range is `"next_to"`. Invalid actions are disallowed by construction. 

### **Conditional Trading / Negotiation Mode**

When trade-related dialogue is detected (`prompter.py` → `input_has_trade_talk()`), a negotiation appendix is injected into the prompt (prompter.py → TRADING_APPENDIX). In this mode, the agent evaluates trades using expected value, survival risk penalties, dynamic reservation pricing, and trust-adjusted future value discounting. Responses are constrained to a strict JSON schema.

### **Agent Personality**

Each agent is assigned a predefined personality profile that influences communication style and decision tendencies, while remaining subordinate to survival and rule constraints (`prompter.py` → `AGENT_PERSONALITIES()`).

**Memory Context**

Long-term summarized memory and short-term interaction transcripts may be prepended to the prompt to preserve behavioral continuity across turns without violating prompt determinism.

**Current Game State**

The complete raw game state JSON is embedded verbatim, serving as the sole source of truth for decision-making.

#### **Example Input Game State**

To make the prompt structure and action constraints concrete, an example of the current input game state format is provided by the Godot integration layer. This sample illustrates the exact JSON structure supplied to the agent during a decision cycle, including available actions, agent state, vicinity objects, and location ranges.

**Decision Boundary**

The prompt ends with a clearly marked decision section, instructing the model to output only the next valid action or, in negotiation mode, a structured trade decision.
