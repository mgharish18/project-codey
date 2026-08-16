# project-codey

## Overview

**project-codey** is a template for a Windows‑based, GPU‑accelerated **local LLM** development environment that integrates with **Visual Studio Code** and **GitHub Copilot**.  It showcases how to host an **Ollama** local model (Gemma 4 12B + GPT‑OSS 20B) and use it as a primary coding assistant.  The project includes:

- A professional repository structure (MIT licence, .gitignore, etc.)
- A detailed installation and configuration guide written in PowerShell
- A **Mermaid** architecture diagram that visualises the flow from VS Code → Copilot → Ollama → GPU
- Validations (example prompts, GPU monitoring, latency checks)
- Troubleshooting tips
- Recommended alternative models and tooling
- Performance metrics and how to capture them.

Feel free to copy, tweak, or extend this repository to suit your own local‑AI workflow.

---

## System Requirements

| Component | Minimum | Recommended | Notes |
|-----------|---------|-------------|-------|
| OS | Windows 10 / 11 (64‑bit) | Same | PowerShell 5.1+ |
| GPU | Nvidia GTX 10xx+ (supports CUDA 11.8) | RTX 5050 Ti 16 GB | Ensure the latest driver is installed from NVIDIA.com |
| CPU | Dual‑core | Quad‑core or better | Needed for local inference when GPU is busy |
| RAM | 8 GB | 16 GB+ | For running VS Code + Ollama |
| Disk | 10 GB free | 30 GB+ | 20 GB is minimum for large models |

> **Tip:** Use the `nvidia-smi` command from a terminal or PowerShell to verify driver and GPU availability.

---

## Installation & Configuration (PowerShell)

1. **Open PowerShell as Administrator** (required for installing the system‑wide Ollama binary).
2. Create a new folder if you haven’t already:
   ```powershell
   New-Item -ItemType Directory -Path "C:\Users\starh\Projects\project-codey"
   ```
3. Change to that folder:
   ```powershell
   Set-Location -Path "C:\Users\starh\Projects\project-codey"
   ```
4. Download and install **Ollama**:
   ```powershell
   # Official installer script (public URL)
   Invoke-WebRequest -Uri "https://ollama.com/install.ps1" -OutFile "$env:TEMP\ollama-install.ps1"
   & "$env:TEMP\ollama-install.ps1"
   `nRemove-Item "$env:TEMP\ollama-install.ps1"
   ```
5. Pull the desired models:
   ```powershell
   # Gemma 4 12 B (use the exact model name from Ollama’s catalog)
   ollama pull gemma4:12b

   # GPT‑OSS 20 B (replace with the correct name in the catalog)
   ollama pull gptoss20b:20b
   ```
6. **Start the Ollama daemon** (runs in the background):
   ```powershell
   Start-Process -FilePath "ollama" -ArgumentList "serve" -WindowStyle Hidden
   ```
   Verify it’s running:
   ```powershell
   ollama list
   ```
   You should see `gemma4:12b` and `gptoss20b:20b` listed.
7. **Integrate with VS Code – Copilot Extension**:
   - Install the `GitHub Copilot` extension via the VS Code marketplace.
   - Open **User Settings** (`File → Preferences → Settings`).
   - Search for **Copilot – Local model** and enable the experimental local model integration:
     ```json
     {
       "github.copilot.experimental.local.enabled": true,
       "github.copilot.experimental.local.provider": "ollama",
       "github.copilot.experimental.local.apiUrl": "http://localhost:11434/api/generate",
       "github.copilot.experimental.local.model": "gemma4:12b"
     }
     ```
   - Save the settings.  Restart VS Code.
8. **Verification** – Open a new file, type a comment such as:
   ```text
   // Prompt the local model to confirm it works
   ```
   Then press `<Ctrl+Shift+Enter>` (Copilot chat) or use the `Copilot → Open Chat` command and type: `Hello, world!`.
   The local model should return a coherent reply.  If it fails, see **Troubleshooting**.

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
|2|`ollama run gptoss20b:20b --prompt "Define a function in Python"`|Shows a reasonable Python function|
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