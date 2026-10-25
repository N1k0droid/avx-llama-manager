<div align="center">

# avx-llama-manager

**A PowerShell toolkit to build, update, run, and manage [llama.cpp](https://github.com/ggml-org/llama.cpp) on Windows, with optional CUDA and CPU profiles from AVX-only to AVX2/FMA.**

[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Platform: Windows](https://img.shields.io/badge/platform-Windows%2010%2F11-0078D6.svg?logo=windows)](#requirements)
[![PowerShell](https://img.shields.io/badge/PowerShell-5.1%2B-5391FE.svg?logo=powershell)](#requirements)

</div>

---

The toolkit uses double-click `.bat` launchers and does not require a separate manager runtime or a permanent PowerShell execution-policy change.

> **Tested on:** Windows 11 Pro · Visual Studio 2022 · CUDA 13.x · NVIDIA Ampere GPU (compute capability 8.6) · Intel Xeon E5-2697 v2 (AVX, no AVX2/FMA)

## Contents

- [Scope](#scope)
- [Features](#features)
- [Requirements](#requirements)
- [Installation](#installation)
- [Configuration](#configuration)
- [Launchers](#launchers)
- [Model management](#model-management)
- [System tray](#system-tray)
- [Project layout](#project-layout)
- [Known limitations](#known-limitations)
- [Contributing](#contributing)
- [License](#license)

## Scope

`avx-llama-manager` is for Windows users who want a `llama.cpp` build suited to their CPU, optionally with an NVIDIA GPU. Its AVX-only profile is particularly useful for older workstations without AVX2/FMA; newer CPUs and CPU-only builds are supported too.

## Features

- Guided `llama.cpp` builds for AVX-only, AVX2/FMA, Native, or custom CPU instruction sets; optional CUDA and BoringSSL.
- Update, automatic backup, rollback, and manual restore.
- System-tray server controls, live VRAM status, logs, crash notifications, and a single-instance guard.
- Browser-based Web UI, OpenAI-compatible API, configurable server settings, named launch profiles, and optional start at Windows logon.
- Optional KV cache settings (quantized K/V, KV cache in system RAM) globally or per model, for longer contexts on limited VRAM.
- GGUF discovery and Hugging Face downloads; automatic or manual benchmarks and model presets.
- VRAM fit pre-check with optional CPU-offload confirmation, plus manual and idle model unload.
- Registration of manually downloaded GGUF files and cleanup of unreferenced Hugging Face cache blobs.

## Requirements

| Component | Purpose | Installation |
|---|---|---|
| Windows PowerShell 5.1 | Running the `.bat` launchers | Included with Windows 10/11 |
| [Git](https://git-scm.com/download/win) | Clones and updates `llama.cpp` | `winget install Git.Git` |
| [CMake](https://cmake.org/download/) | Configures the build | `winget install Kitware.CMake` |
| Visual Studio 2022 or later | Provides the MSVC C++ toolchain | `winget install Microsoft.VisualStudio.2022.Community` |
| **Desktop development with C++** workload | Required Visual Studio build workload | Add it from Visual Studio Installer |
| [NVIDIA CUDA Toolkit](https://developer.nvidia.com/cuda-downloads) | Optional GPU acceleration | NVIDIA installer or `winget install Nvidia.CUDA` |

`1.Build.bat` checks the required tools before cloning or building and reports anything missing. The launchers use Windows PowerShell (`powershell.exe`), even if PowerShell 7 is installed. An NVIDIA GPU and CUDA are optional; without them, you can build for CPU-only inference. With several CUDA toolkits installed, the build uses the one whose `nvcc` was detected (passed to CMake as `-T cuda=<path>`), as long as Visual Studio has the integration for that version.

> [!WARNING]
> If Microsoft Defender **Controlled folder access** is enabled, do not install the package in a protected folder (Documents, Desktop, ...): the compiler and git are blocked from writing there, the build fails with `C1083: Cannot open compiler generated file`, and git can lose its index. `1.Build.bat` detects this and lists the programs to allow. Blocked programs appear in Event Viewer under *Microsoft-Windows-Windows Defender/Operational*, event 1123.

## Installation

Clone the repository, or download it as a ZIP from GitHub and extract it. Put it outside protected folders such as Documents if Defender's Controlled folder access is on:

```powershell
git clone https://github.com/N1k0droid/avx-llama-manager.git
cd avx-llama-manager
```

1. Double-click **`1.Build.bat`**. Press Enter at the installation-path prompt to create `llama.cpp\` inside the project folder.
2. Choose the CPU profile; enable CUDA and confirm compute capability if using an NVIDIA GPU. Enable BoringSSL if you want to use the Hugging Face download helper. Wait for compilation.
3. Double-click **`3.Start.bat`** → **Start llama-server**. The tray icon appears, and the Web UI opens when the server is ready.
4. Use **`4.Models.bat`** to download a GGUF or register one you already have. Right-click the tray icon → **Load model**.

The default OpenAI-compatible API base URL is:

```text
http://127.0.0.1:5001/v1
```

> [!NOTE]
> A GGUF copied into a folder may not automatically appear in the router's model list. If it does not, use `4.Models.bat` → **Register local file**.

## Configuration

### CPU instruction set

Choose the build profile for the CPU that will **run** the binaries:

| Option | Profile | Use when |
|---|---|---|
| 1 | SSE4.2 + AVX | The CPU supports AVX but not AVX2/FMA |
| 2 | SSE4.2 + AVX + AVX2 + FMA | The CPU supports AVX2 and FMA |
| 3 | Native (`-DGGML_NATIVE=ON`) | The build stays on the current computer |
| 4 | Custom | You want to provide your own CMake CPU flags |

> [!WARNING]
> An AVX2/FMA build can fail with an illegal-instruction error on CPUs without those instructions. Native builds are not necessarily portable to another CPU.

### CUDA and SSL

**CUDA** enables NVIDIA GPU acceleration; the build wizard detects compute capability when possible and lets you change it. **BoringSSL** enables Hugging Face downloads through the compiled `llama.cpp` tools and requires internet access during configuration.

### Models folder

The initial default is the **project root**, not the bundled `models\` directory. You can select that directory or another folder and save it as the default or use it for the current run. The server's models folder and the Hugging Face cache location are separate settings.

The router finds models in two places, and lists each one with its source in `logs\server.log` (`Available models`):

| Source | Where | Model id |
|---|---|---|
| `models_dir` | `model.gguf` directly in the models folder | `model` |
| `models_dir` | One subfolder per model: `my-model\model.gguf`, or every shard plus an optional `mmproj*.gguf` | `my-model` (the subfolder name) |
| `cache` | The Hugging Face cache (`HF_HOME\hub`, or the default under your user profile) | `org/repo:QUANT` |

Files deeper in the models folder are not seen. **Register local file** fixes that by giving the model its own subfolder.

### Hugging Face environment

Open **`3.Start.bat` → Hugging Face environment variables** to view, set, or remove these variables for your Windows user account:

| Variable | Value | Use |
|---|---|---|
| `HF_HOME` | A writable cache root, e.g. `D:\AI\huggingface` | Sets the Hugging Face cache root; the model cache is under `HF_HOME\hub` |
| `HF_TOKEN` | Your Hugging Face access token | Accesses gated/private repositories for which you have permission |

You can alternatively set user-level variables in PowerShell (replace the example values):

```powershell
[Environment]::SetEnvironmentVariable('HF_HOME', 'D:\AI\huggingface', 'User')
[Environment]::SetEnvironmentVariable('HF_TOKEN', '<your-token>', 'User')
```

To remove one, use the menu or set its value to `$null` with `SetEnvironmentVariable`. **Close and reopen the launchers and restart the tray/server** after a change so new processes inherit it. Avoid placing a real token in shell history, logs, or screenshots; the `3.Start.bat` menu is preferable. `HF_TOKEN` is not needed for public models that do not require authentication. If a downloaded model does not appear in **Load model**, check the configured models folder against the cache location or use **Register local file**.

### Server settings

Open **`3.Start.bat` → Custom options** to adjust a one-off launch or save new defaults:

| Setting | Description |
|---|---|
| `--host`, `--port` | Listening address and port; defaults to `127.0.0.1:5001` |
| `--models-dir`, `--models-max` | Models folder and maximum number of loaded models |
| `-ngl`, `-c` | GPU-offloaded layers (`auto`, `all`, or a number) and context size |
| `-fa` | Flash attention (`on`, `off`, or `auto`) |
| `-ctk`, `-ctv` | KV cache type for K and V; default `f16`. `q8_0` roughly halves KV memory; a quantized V requires `-fa on` |
| KV offload | `on` (default) keeps the KV cache in VRAM; `off` passes `-nkvo` and keeps it in system RAM |
| `-t` | CPU threads; `0` (default) lets `llama-server` decide |
| Extra args | Free-form `llama-server` flags appended last; empty by default |
| `--api-key`, `--offline` | Optional API authentication and locally cached content only |
| `--fit`, `--fit-target`, `--fit-ctx` | Automatic VRAM fitting, per-GPU headroom, and minimum fit context |
| Fit pre-check | Test model fit before tray-based loading; enabled by default |
| Idle unload | Unload inactive models after the selected number of minutes; `0` disables it |

If `-ngl` is fixed, `--fit` cannot adjust GPU layers; choose `auto` if you want automatic adjustment. With an explicit `-c`, `--fit` keeps the context size and offloads layers to the CPU instead. **Fit pre-check**, **idle unload**, and **launch profiles** are manager settings, not `llama-server` command-line flags. The KV settings, threads, and extra args are passed only when changed, so the default command line is the same as without them. Extra args win over every per-model preset, because `llama-server` gives the command line the highest priority. Before exposing the server beyond localhost, review the listening address, API key, and network access.

### Launch profiles

A profile is a named copy of the launch settings, stored in `config\profiles\<name>.json` (not tracked by git). Use **`3.Start.bat` → Launch profiles** to start the server with a profile without changing the defaults, make a profile the default, save the current defaults as a profile, or delete one. **Custom options** can also save its values as a profile. To use an example, copy a file from `config\examples\` into `config\profiles\` and adjust it.

### Profiles and per-model presets: which one wins

A **preset** belongs to one model (`4.Models.bat` → Presets): context size, flash attention, KV cache and extra args for that model only. A **profile** applies to every model. The tray always runs in one of two modes:

| How the server was started | Mode | Context, flash attention and KV cache come from |
|---|---|---|
| `3.Start.bat` → 1, or auto-start at logon | **Auto** | the model's preset; the launch defaults fill in what it does not set |
| `3.Start.bat` → Launch profiles → 2, or tray → **Profile** → a profile | **Profile** | the profile, for every model; presets are ignored |
| `3.Start.bat` → 3 (Custom options) | **Custom** | the values entered, for every model |
| tray → **Profile** → **Auto** | **Auto** | as in the first row, with the saved launch defaults |

- Changing the profile from the tray restarts the server, so the loaded model has to be loaded again.
- The mode is not remembered: the next start without a profile is Auto again.
- Always applied to every model, in both modes: `-ngl`, `--fit*`, `-t`, and the profile's or defaults' extra args. A preset's own extra args apply to its model in Auto mode only.
- The current mode shows in the tray icon tooltip and in `logs\server.log` (`settings mode: ...`).

This works through `llama-server`'s precedence (command line > model section of `models.ini` > `[*]`): in Auto mode the manager writes the defaults to `[*]` and each preset to its model's section; in Profile mode it puts the profile's values on the command line.

### Long context on limited VRAM

A model that nearly fills the GPU leaves little room for the KV cache, which grows linearly with the context. In order of preference:

1. **Quantize the KV cache:** `-ctk q8_0 -ctv q8_0` with `-fa on` uses about half the memory of `f16` with little quality loss. `q4_0` saves more but costs more quality.
2. **Use a smaller weight quantization** to leave VRAM for the KV cache.
3. **Keep the KV cache in system RAM** (KV offload `off`, `-nkvo`): all weights stay in VRAM and the context is limited by system RAM, but attention then runs on the CPU. Generation and prompt processing slow down as the context grows, especially on AVX-only CPUs without FMA and with older memory. On the reference machine below, generation dropped from about 36 to 7 tokens/s at an empty context and to 3.4 tokens/s at 16k. Benchmark before relying on it.
4. For mixture-of-experts models, offloading expert tensors to the CPU (`--n-cpu-moe N` or `-ot` via extra args) usually beats `-nkvo`. `--fit` already does this automatically when needed. Add `-ub 2048 -b 2048` to the extra args: with experts in system RAM, larger batches let the GPU process the prompt, which tripled prompt speed on the reference machine.

Two things to check before tuning:

- **Dense models must fit entirely in VRAM.** Layers that overflow to the CPU cost far more than their share: on the reference machine, a 27B model with 13 of 65 layers on the CPU generated about 4 tokens/s instead of about 35.
- **Some models need much less KV memory than others.** Hybrid-attention models (for example Qwen3.5 and later, with full attention on one layer in four) use a fraction of the KV memory per token of a classic transformer of the same size. Measure with `llama-fit-params -m <model> -c <ctx> -ctk q8_0 -ctv q8_0 -fa on -fitp on`, which prints the estimated memory per device without loading the model.

`--fit` measures the VRAM actually free at load time, so memory used by the desktop and monitors is already accounted for: `--fit-target` is only the margin for what grows afterwards, and the default 1024 MiB is usually enough. `config\examples\24gb-gpu-long-context.json` is a starting point for a 24 GB GPU: `-fa on`, `q8_0` K/V, and a 32k context. On Windows, set **CUDA - Sysmem Fallback Policy** to **Prefer No Sysmem Fallback** in the NVIDIA Control Panel. Otherwise an overfull GPU silently spills into shared system memory instead of failing at load time; on the reference machine this cut speed by about two thirds without any error.

Reference machine for the figures above: Xeon E5-2697 v2 (AVX only), 256 GB DDR3-1600, RTX 3090 24 GB, NVMe storage.

Any `llama-server` option can also be set through its `LLAMA_ARG_*` environment variable (for example `LLAMA_ARG_CACHE_TYPE_K`). The tray passes the environment on to the server, but command-line flags take precedence.

## Launchers

Double-click the `.bat` files to use the menus. Each launches its corresponding PowerShell script; `tray.ps1` is launched by `3.Start.bat` or auto-start, not directly.

### 1.Build.bat — First build

Checks prerequisites, asks for an installation path, then clones and builds `llama.cpp`:

| Prompt | What to do |
|---|---|
| Installation path | Press Enter for `llama.cpp\` inside this project, or enter another path |
| CPU profile | Choose AVX-only, AVX2/FMA, Native, or Custom for the target CPU; see [CPU instruction set](#cpu-instruction-set) |
| CUDA | Enable for an NVIDIA GPU build; leave disabled for CPU-only inference |
| Compute capability | Accept the detected value or specify the target NVIDIA GPU's value |
| BoringSSL | Enable for downloads via the Hugging Face helper |

Use `2.Manage.bat` rather than rerunning this launcher to update an existing checkout.

### 2.Manage.bat — Builds and backups

| Menu item | What it does |
|---|---|
| Update | Checks upstream changes and that the `llama.cpp` checkout is clean (rebuilds a missing git index; offers to discard leftover changes, default no), backs up the current build, pulls changes, and rebuilds; offers to restore the backup if rebuilding fails |
| Backup | Saves compiled binaries in `backups\`; the latest five backups are retained |
| Restore | Selects a backup to replace the current compiled binaries |

### 3.Start.bat — Server and settings

| Menu item | What it does |
|---|---|
| 1. Start llama-server | Starts the tray-managed server with saved defaults and opens the Web UI when ready |
| 2. Auto-start | Enables or removes a per-user Windows Task Scheduler task at logon; this start path does not open the browser |
| 3. Custom options | Prompts for server and fit settings; Enter keeps a value; choose whether to save the settings before starting |
| 4. Change models folder | Selects a folder for this run or saves it as the new default |
| 5. Change listening port | Changes the TCP listening port (1–65535) |
| 6. Hugging Face environment variables | Views, sets, or removes user-level `HF_HOME` and `HF_TOKEN`; machine-level values are read-only |
| 7. Restore default launch settings | Resets `config\launch-defaults.json`; does not remove models or backups |
| 8. Start with debug console | Starts as in option 1, but keeps a console open with the live server log |
| 9. Set idle-unload timeout | Sets minutes of inactivity before model unload; `0` disables it |
| 10. Launch profiles | Shows, starts with, saves, deletes, or sets as default a named profile; see [Launch profiles](#launch-profiles) |

**Custom options:** prompts for the parameters in [Server settings](#server-settings), plus the manager's fit pre-check, and can save the result as a profile. **Auto-start** requires an interactive user logon: it is not a Windows service. The `.bat` launchers use a process-local PowerShell execution-policy bypass; they do not permanently change the system policy.

### 4.Models.bat — GGUF tools

| Menu item | What it does |
|---|---|
| 1. Scan and list models | Shows GGUF files, sizes, estimated router aliases, benchmarks, and saved presets |
| 2. Download from Hugging Face | Downloads a repository or selected quantization using `llama-cli -hf`; the helper may also load the model once |
| 3. Scan and benchmark | Select a model; choose Automatic to compare flash attention on/off or Manual to select `llama-bench` parameters |
| 4. Presets | List, create, update, or delete per-model flash-attention, context, KV cache and extra-args overrides, for models in the models folder and in the Hugging Face cache; set the global flash-attention default. Presets apply in Auto mode (see [Profiles and per-model presets](#profiles-and-per-model-presets-which-one-wins)) |
| 5. Register local file | Gives a GGUF the router cannot see its own subfolder in the models folder: regular files are moved, symlinks (Hugging Face cache, LM Studio) become hard links so the original stays intact. Shards and companion files (`mmproj`, MTP/draft) move together |
| 6. Clean cache | Shows unreferenced Hugging Face blobs and asks for confirmation before removing them |

For **Automatic** benchmarking, the faster flash-attention setting can be saved as a per-model preset; this keeps the preset's other values. Presets generate `config\models.ini`; verify estimated aliases against the `Available models` line in `logs\server.log`. When `models.ini` exists, the tray regenerates it at each server start from the active launch settings: the context size, flash attention, and KV settings go into its `[*]` section instead of the command line, so per-model presets can override them.

## Model management

Use `4.Models.bat` for the options above. A download identifier may include a quantization, for example:

```text
unsloth/gemma-3-12b-it-GGUF:Q4_K_M
```

Omit `:QUANT` to let `llama.cpp` choose a default file. If you downloaded a `.gguf` manually and it is missing from **Load model**, use **Register local file**. Cache cleanup requires confirmation and does not intentionally remove referenced snapshot files.

## System tray

`tray.ps1` starts `llama-server.exe`, prevents a second tray-managed instance, and writes output to `logs\server.log`. Logs rotate above 10 MB; `server.log.old` keeps the previous log.

Right-click the tray icon:

| Menu item | What it does |
|---|---|
| VRAM status | Shows free/total VRAM, refreshed when the menu opens |
| Show console/logs | Opens a window with the live server log |
| Launch WebUI | Opens the server Web UI in a browser |
| Open models folder | Opens the configured models directory |
| Model: ... | Informational line under VRAM status: the loaded model, its state and context size, refreshed when the menu opens |
| Load model | Lists models from the router (the loaded one is ticked) and loads one, after an optional VRAM fit check |
| Unload model | Unloads models currently known to the router |
| Profile | Switches between **Auto** (each model's preset) and a saved profile (its values for every model); restarts the server. The active one is ticked |
| Start / Restart / Stop | Controls `llama-server` without closing the tray manager. The browser opens only on the tray's first start; after a restart or a profile change, a notification asks to reload the Web UI page |
| Exit | Closes the tray manager |

### VRAM fit check

When enabled, **Load model** runs `llama-fit-params` first. If the model fits, loading continues; if CPU offload is needed, the manager asks for confirmation; if the model cannot fit, loading is stopped. When `llama-fit-params` is unavailable, the pre-check is skipped rather than blocking loading.

### Idle unload and crash alerts

Set the inactivity timeout with `3.Start.bat` option 9; `0` disables it. The manager unloads models after the configured idle period. Unexpected `llama-server` exits are logged with their exit code and shown as a Windows notification; intentional stops, restarts, and exits do not trigger an alert.

## Project layout

```text
avx-llama-manager\
  1.Build.bat
  2.Manage.bat
  3.Start.bat
  4.Models.bat
  scripts\
    common.ps1
    build.ps1
    manage.ps1
    start.ps1
    models.ps1
    tray.ps1
  assets\
    tray.ico
  config\            generated settings and presets
    profiles\        named launch profiles (not tracked)
    examples\        example profiles to copy into profiles\
  backups\           compiled-build backups
  llama.cpp\         default clone location, created on first build
  models\            optional models folder
  logs\              server.log and rotated log
```

## Known limitations

- Flash attention may improve or reduce performance depending on the model; benchmark before saving a preset.
- A quantized V cache requires flash attention on. KV cache in system RAM (`-nkvo`) runs attention on the CPU and slows down as the context grows.
- The VRAM pre-check requires a recent `llama.cpp` build containing `llama-fit-params`.
- BoringSSL-enabled builds require internet access during configuration.
- Auto-start is a per-user scheduled task, not a Windows service.
- Router aliases follow `llama-server`'s `--models-dir` rules; for Hugging Face cache paths they are a heuristic. Verify them against `Available models` in the log before relying on per-model presets. Models found only in a Hugging Face cache outside the models folder are not listed by `4.Models.bat`.
- Hugging Face cache cleanup may overestimate recoverable space when snapshots use hardlinks.
- Other hardware and toolchain combinations may require additional troubleshooting.

## Contributing

Issues and pull requests are welcome. Include your Windows version, CPU/GPU, Visual Studio, CUDA and PowerShell versions, selected build profile, and relevant `logs\server.log` excerpts. Remove secrets before sharing.

## License

[MIT](LICENSE). This project is not affiliated with [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp).
