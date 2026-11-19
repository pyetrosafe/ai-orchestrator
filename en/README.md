# OpenCode: Cloud Orchestrator + Local Ollama

This repository provides a streamlined configuration for running OpenCode in a hybrid workflow:

- **Cloud Orchestrator**: analyzes the request, splits the work into microtasks, and audits the results.
- **Local Worker with Ollama**: executes one small task at a time, validates the change, and returns a standardized report.
- **Local fallback**: saves the latest result to `.opencode/state/last-result.md`, allowing the workflow to continue or diagnose issues even when the subagent response is empty.

The GPU is used sequentially: the worker never receives two microtasks in parallel.

## How the workflow works

1. `/orquestrar <request>` sends the request to the primary `orchestrator` agent.
2. The orchestrator performs the minimum necessary reading and creates a plan of tasks, subtasks, and microtasks.
3. Each microtask is delegated to `local-worker` one at a time.
4. The worker changes only the authorized scope, runs a validation, and writes the final contract to `last-result.md`.
5. The orchestrator checks the result:
   - `DONE`: advances to the next microtask.
   - `FAILED` or `BLOCKED`: sends a surgical correction.
   - After three consecutive failures on the same microtask, stops the cycle and asks for guidance.

Internal worker format:

```text
TASK
STATUS: DONE | FAILED | BLOCKED
CHANGED_FILES
VERIFICATION
ISSUE_FOUND
SUGGESTED_NEXT_STEP
```

## Repository files

```text
.
├── opencode.json                     # Active configuration: providers, agents, and permissions
├── README.md                         # Installation and hardware calibration guide
├── agents/
│   ├── orchestrator.md               # Primary Cloud agent
│   └── local-worker.md               # Local Ollama executor
├── commands/
│   └── orquestrar.md                 # Command that starts the hybrid workflow
```

The global directory used by OpenCode is normally `~/.config/opencode/`. On Windows, it corresponds to `C:\Users\<user>\.config\opencode\`.

The same structure can also be installed in a specific project:

```text
your-project/
├── opencode.json
└── .opencode/
    ├── agents/
    │   ├── orchestrator.md
    │   └── local-worker.md
    ├── commands/
    │   └── orquestrar.md
    └── state/
        └── last-result.md
```

## Current configuration

- The `orchestrator` agent is primary and can delegate exclusively to `local-worker`.
- `local-worker` is configured to use `ollama/gemma4:e4b#fast`.
- Local calls are limited to 15 steps per microtask.
- The Ollama capabilities configured in this installation include a context of up to 64k tokens.
- The interface is configured to request permissions interactively.

Before using it on another machine, adjust the Cloud provider/model selected by OpenCode and verify the models available in Ollama.

## Installation

### 1. Install and run Ollama

```bash
ollama pull gemma4:e4b
ollama serve
```

Verify the default endpoint:

```text
http://localhost:11434
```

### 2. Copy the configuration

For global configuration on Windows:

```powershell
Copy-Item .\opencode.json "$env:USERPROFILE\.config\opencode\opencode.json"
Copy-Item .\cli.json "$env:USERPROFILE\.config\opencode\cli.json"
Copy-Item .\agents\* "$env:USERPROFILE\.config\opencode\agents\" -Recurse -Force
Copy-Item .\commands\* "$env:USERPROFILE\.config\opencode\commands\" -Recurse -Force
```

For a project installation, copy `opencode.json` to the project root and copy the `agents` and `commands` directories to `.opencode/`.

### 3. Adjust the Cloud model

In OpenCode, select the Cloud model you want to use as the primary agent. The configuration can also point directly to a standardized model in the `orchestrator` profile, depending on the installed OpenCode version.

## Usage

From the project root:

```bash
opencode
```

---
> **Select the desired Cloud** model (a free model is also an option), for example `DeepSeek V4.1 Flash Free`. Enter `/models` to view the available models, or filter for free models by typing `Free` in the search field.
---

Select the `orchestrator` agent or run it directly:

```text
/orquestrar Implement CPF validation in the registration form and add unit tests.
```

OpenCode will present the plan before delegations. The local executor will respond in a short, objective format, without detailed reasoning or long logs.

## Suggested hardware requirements

The configuration files were initially calibrated for entry-level requirements, such as an RTX 3070 Ti 8 GB GPU with Ollama 64k. Below is a comparison of hardware profiles for different usage levels:

### Configuration Comparison by Profile

| Component | Minimum (Entry) | Recommended (Cost-Effective) | High (Enthusiast / Workstation) |
| :--- | :--- | :--- | :--- |
| **GPU (VRAM)** | **8 GB** (e.g., RTX 3060 8GB, RTX 4060) | **16 GB** (e.g., RTX 4060 Ti 16GB, RTX 4070 Ti Super, RTX 5060 Ti 16GB) | **24 GB to 48 GB+** (e.g., RTX 3090, RTX 4090, RTX 5090, or dual GPU) |
| **RAM** | **16 GB** DDR4/DDR5 | **32 GB** DDR4/DDR5 | **64 GB to 128 GB** DDR5 (prefer *Dual/Quad Channel*) |
| **CPU** | 6 modern cores (e.g., Ryzen 5 5600 / Core i5-12400) | 8 cores (e.g., Ryzen 7 5700X / Core i5-13600K) | 12 to 16+ cores with AVX-512 support (e.g., Ryzen 9 7900/9900X or Core i7/i9) |
| **Storage** | 500 GB NVMe SSD | 1 TB NVMe SSD (PCIe 4.0) | 2 TB+ NVMe SSD (PCIe 4.0/5.0) |
| **Supported Models (100% GPU)** | **2B to 9B** (short/medium context) | **2B to 24B** (with context headroom) | **32B to 70B+** (depending on VRAM and quantization) |

The `gemma4:e4b` model is the alternative currently in use. Other models may require VRAM, context, and variant adjustments. For Ollama 64k context, the following settings may be useful:

```powershell
$env:OLLAMA_FLASH_ATTENTION = "1"
$env:OLLAMA_KV_CACHE_TYPE = "q8_0"
ollama serve
```

### Hardware Calibration for the Entry Profile (RTX 3070 Ti 8 GB + Ollama 64k)

Analyzing some of the models available in Ollama:

| Local Model | Size | Recommendation for `local-worker` |
|---|---|---|
| `qwen3.5:4b` | 3.3 GB | Low-resource alternative, excellent for simple tasks and limited VRAM |
| `gemma4:e4b` | 6.6 GB | Recommended alternative, good performance on complex tasks |
| `deepseek-r1:8b` | 5.2 GB | **Avoid for the worker.** Long-reasoning models generate too much internal text and commonly break short-output contracts. |

### Extra Ollama VRAM-Saving Tips
1. **Keep `small_model` aligned with `local-worker`:** In `opencode.json`, confirm that they match. This prevents Ollama from unloading and reloading different models on an 8 GB GPU during every turn.
2. **Enable Flash Attention and quantized KV Cache in Ollama** (optional, recommended for using 64k without exceeding 8 GB of VRAM):
   - `OLLAMA_FLASH_ATTENTION=1`
   - `OLLAMA_KV_CACHE_TYPE=q8_0`

> ---
> #### KV Cache Configuration (`OLLAMA_KV_CACHE_TYPE`)
>
> Defines the precision of the context history in VRAM. Quantization reduces memory consumption and enables larger context windows (`num_ctx`).
>
> ##### Available Options
>
> | Configuration | VRAM Consumption | Quality Impact | Ideal Use Case |
> | :--- | :--- | :--- | :--- |
> | **`f16`** *(default)* | 100% | None | GPUs with memory headroom or short contexts (4k–8k). |
> | **`q8_0`** | ~50% | Imperceptible | **Recommended default:** balances VRAM savings and precision. |
> | **`q4_0`** | ~25% | Noticeable in long text | Critical low-VRAM scenarios or extreme contexts. |
>
> ##### Recommendations by VRAM Capacity
>
> * **6 GB – 8 GB:** Use **`q8_0`** as the default for 7B–9B models (`Q4_K_M`) with 8k–16k contexts without using RAM. Use **`q4_0`** only for long documents (32k+) at the GPU limit.
> * **10 GB – 12 GB:** Use **`q8_0`** for 12B–14B models for code/chat tasks or for running 7B–8B models with high contexts (32k–64k).
> * **16 GB – 24 GB:** Use **`q8_0`** for 22B–35B models (16k–32k context) and to keep the 32k default consumption under control, which Ollama applies automatically starting at 24 GiB. Use **`f16`** only for ≤14B models with short context.
> * **48 GB or more:** Use **`f16`** for medium-sized models or contexts up to 32k for maximum speed. Use **`q8_0`** for 70B+ models or extreme windows (128k–256k), where `f16` can exceed 40 GB for cache alone.
>
> ##### Requirements and Considerations
>
> * **Flash Attention required:** Quantization only works with `OLLAMA_FLASH_ATTENTION=1` enabled simultaneously.
> * **Model sensitivity:** Architectures with aggressive *Grouped-Query Attention* (GQA) (e.g., smaller variants of the Qwen family) suffer greater degradation with `q4_0`.
> * **Global scope:** The variable applies to every model on the server and requires restarting the Ollama service before it takes effect.
> ---
## Security

- Review the configuration before running it.
- Change permissions only if you understand which tools the orchestrator and worker can access.

## Customization

- Adjust the Cloud agent in `agents/orchestrator.md`.
- Adjust the local worker behavior in `agents/local-worker.md`.
- Adjust the command text and workflow in `commands/orquestrar.md`.
- Adjust models, context limits, variants, and permissions in `opencode.json`.
- Adjust the CLI appearance and behavior in `cli.json`.