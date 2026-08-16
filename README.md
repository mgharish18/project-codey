# project-codey

## Overview

**project-codey** is a template for a Windows‑based, GPU‑accelerated **local LLM** development environment that integrates with **Visual Studio Code** and **GitHub Copilot**.  It showcases how to host local model (Gemma 4 12B + GPT‑OSS 20B) using **Ollama** and use it as a primary coding assistant with **Visual Studio Code**.  

## Table of Contents
- [Overview](#overview)
- [System Requirements](#system-requirements)
- [Installation & Configuration](#installation--configuration-powershell)
- [Architecture Diagram](#architecture-diagram)
- [Validation Checklist](#validation-checklist)
- [Performance Metrics](#performance-metrics)
- [Troubleshooting](#troubleshooting)
- [Alternative Tools & Models](#alternative-tools--models)
- [License](#license)
- [Feedback & Contributions](#feedback--contributions)

Feel free to copy, tweak, or extend this repository to suit your own local‑AI workflow.

---

## System Requirements

| Component | Minimum | Recommended | Notes |
|-----------|---------|-------------|-------|
| OS | Windows 10 / 11 (64‑bit) | Same | Also supported with Linux and MacOS |
| GPU | Nvidia GTX 10xx+ (supports CUDA 11.8) | RTX 5050 Ti 16 GB | Ensure the latest driver is installed from NVIDIA |
| CPU | Dual‑core | Quad‑core or better | Needed for local inference when GPU is busy |
| RAM | 4 GB | 8 GB+ | For running VS Code + Ollama |
| Disk | 10 GB free | 30 GB+ | 20 GB is minimum for large models |

> **Tip:** Use the `nvidia-smi` command from a terminal or PowerShell to verify driver and GPU availability and `Windows Task Manager` to check real-time GPU vRAM usage while thile the mode is running.

---

## Installation & Configuration (PowerShell)

1. **Open PowerShell as Administrator** (required for installing the system‑wide Ollama binary).

2. **Ollama Installation & Model Hosting**
   ```powershell
   # Install the Ollama binary using the official PowerShell script
   irm https://ollama.com/install.ps1 | iex
   ```
   This single line downloads the installer from the official source and executes it.
   For more information on installing Ollama for Windows, see the official
   download page: https://ollama.com/download/windows

3. Pull the desired models.  The GPU has 16 GB VRAM, so we selected models that
   stay comfortably below that limit:
    * **[Gemma 4 12B](https://ollama.com/library/gemma4:12b)** – Google's open-weight models designed to deliver frontier-level performance at each size. They are well-suited for reasoning, agentic workflows, coding, and multimodal understanding.
    * **[GPT‑OSS 20B](https://ollama.com/library/gpt-oss:20b)** – OpenAI’s open-weight models designed for powerful reasoning, agentic tasks, and versatile developer use cases.
   
   ```powershell
   ollama pull gemma4:12b
   ollama pull gpt-oss:20b
   ```

4. Verify models are available:
   ```powershell
   ollama list
   ```
   The output should list `gemma4:12b` and `gpt-oss:20b`.

5. **Optional** – Verify a model by generating a short prompt:
   ```powershell
   ollama run gemma4:12b "What is the capital of India?"
   ```
   If a response appears, the model is working.
---

## Architecture Diagram

```mermaid
flowchart TD
    A[VS Code] -->|Chat Prompt| B[GitHub Copilot Extension]
    B -->|Local‑LLM request| C[Ollama daemon (localhost:11434)]
    C -->|Inference| D[GPU (NVIDIA RTX 5050 Ti)]
    D -->|Result| C
    C -->|Response| B
    B -->|Chat reply| A
```

---

## Validation Checklist

| Step | Command / Action | Expected Result |
|------|-----------------|----------------|
|1|`ollama run gemma4:12b --prompt "Hello, world!"`|Returns a generated text string without errors|
|2|`ollama run gpt-oss:20b --prompt "Define a function in Python"`|Shows a reasonable Python function|
|3|`nvidia-smi --query-gpu=memory.used,gpu_utilization --format=csv`|Displays current memory usage and GPU utilization (should spike above 10 % during inference) |
|4|VS Code → Copilot Chat → type `Explain recursion in C++`|Local model replies with an explanation|
|5|Measure latency: `Measure-Command { ollama run gemma4:12b --prompt "Test" }`|Approximately X ms (capture for comparison) |

---

## Performance Metrics

1. **Latency** – Time from sending a prompt to receiving the first token.
   ```powershell
   Measure-Command { ollama run gemma4:12b --prompt "Describe the solar system." }
   ```
   Record the *Milliseconds* value.

2. **Throughput** – Tokens per second (TPS). Use a large prompt:
   ```powershell
   $prompt = "a" * 2000  # 2000 characters
   $start = Get-Date
   $response = ollama run gemma4:12b --prompt $prompt
   $duration = (Get-Date) - $start
   $tokens = ($response | ConvertFrom-Json).total_tokens
   "TPS: $( [int]($tokens / $duration.TotalSeconds) )"
   ```

3. **GPU Utilization** – Use `nvidia-smi` during inference and plot the values. Example:
   ```powershell
   nvidia-smi --query-gpu=utilization.gpu --format=csv -l 1 > gpu_log.csv
   ```

4. **Memory Footprint** – Monitor the `memory.used` column in `nvidia-smi` while running large prompts.

> **Tip:** Store the metric results in a CSV or simple text file to track over time.

---

## Troubleshooting

| Symptom | Likely Cause | Quick Fix |
|---------|--------------|-----------|
| Ollama daemon not starting | Port 11434 already in use | Kill the process (`Stop-Process -Id (Get-Process -Name "ollama").Id`) or change port via `--port` |
| Copilot never connects | `github.copilot.experimental.local.enabled` is false | Enable it in `settings.json` and restart VS Code |
| Model responds with an error | GPU driver outdated | Update the NVIDIA driver to the newest version and reboot |
| Very long latency | High CPU activity | Close other heavy applications, consider reducing prompt size |
| Token limit exceeded | Prompt longer than model context | Shorten prompt or switch to a larger‑context model if available |

---

## Alternative Tools & Models

| Tool / Model | GPU Fit | Typical Latency | Token Throughput | Notes |
|--------------|---------|----------------|-----------------|-------|
| **Llama 2 70B (Meta)** | ✅ (requires >24 GB VRAM) | High (500 ms+) | Low (~10 TPS) | Requires 70 B VRAM, not suitable for 16 GB GPU |
| **Mistral 7B** | ✅ (≈8 GB VRAM) | Low (~100 ms) | High (~40 TPS) | Excellent for 16 GB GPUs |
| **Qwen‑1.5‑14B** | ✅ (≈10 GB VRAM) | Medium (~200 ms) | Good (~20 TPS) | Open‑source, good multi‑lingual |
| **OpenAI gpt‑4o** | ❌ (cloud) | Varies | N/A | Requires internet, but very low latency |
| **ChatGPT‑4** | ❌ | N/A | N/A | Cloud‑only |

> Use the quick‑start guides in each section to evaluate which model best fits your hardware and latency requirements.

---

## License

This project is licensed under the MIT License. See the **LICENSE** file for details.

---

## Feedback & Contributions

Feel free to open issues or pull requests. Contributions that improve the installation scripts, add new model recipes, or document more use‑cases are welcome!