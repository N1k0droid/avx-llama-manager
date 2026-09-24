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
- Browser-based Web UI, OpenAI-compatible API, configurable server settings, and optional start at Windows logon.
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

`1.Build.bat` checks the required tools before cloning or building and reports anything missing. The launchers use Windows PowerShell (`powershell.exe`), even if PowerShell 7 is installed. An NVIDIA GPU and CUDA are optional; without them, you can build for CPU-only inference.

## Installation

Clone the repository (once published under this name):

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
| `--api-key`, `--offline` | Optional API authentication and locally cached content only |
| `--fit`, `--fit-target`, `--fit-ctx` | Automatic VRAM fitting, per-GPU headroom, and minimum fit context |
| Fit pre-check | Test model fit before tray-based loading; enabled by default |
| Idle unload | Unload inactive models after the selected number of minutes; `0` disables it |

If `-ngl` is fixed, `--fit` cannot adjust GPU layers; choose `auto` if you want automatic adjustment. **Fit pre-check** and **idle unload** are manager settings, not `llama-server` command-line flags. Before exposing the server beyond localhost, review the listening address, API key, and network access.

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
| Update | Checks upstream changes, backs up the current build, pulls changes, and rebuilds; offers to restore the backup if rebuilding fails |
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

**Custom options:** prompts for the parameters in [Server settings](#server-settings), plus the manager's fit pre-check. **Auto-start** requires an interactive user logon: it is not a Windows service. The `.bat` launchers use a process-local PowerShell execution-policy bypass; they do not permanently change the system policy.

### 4.Models.bat — GGUF tools

| Menu item | What it does |
|---|---|
| 1. Scan and list models | Shows GGUF files, sizes, estimated router aliases, benchmarks, and saved presets |
| 2. Download from Hugging Face | Downloads a repository or selected quantization using `llama-cli -hf`; the helper may also load the model once |
| 3. Scan and benchmark | Select a model; choose Automatic to compare flash attention on/off or Manual to select `llama-bench` parameters |
| 4. Presets | List, create, update, or delete per-model flash-attention/context overrides; set the global flash-attention default |
| 5. Register local file | Makes a manually downloaded GGUF discoverable through a router-compatible Hugging Face-style layout |
| 6. Clean cache | Shows unreferenced Hugging Face blobs and asks for confirmation before removing them |

For **Automatic** benchmarking, the faster flash-attention setting can be saved as a per-model preset. Presets generate `config\models.ini`; verify estimated aliases against the `Available models` line in `logs\server.log`.

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
| Load model | Lists models from the router and loads one, after an optional VRAM fit check |
| Unload model | Unloads models currently known to the router |
| Start / Restart / Stop | Controls `llama-server` without closing the tray manager |
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
  backups\           compiled-build backups
  llama.cpp\         default clone location, created on first build
  models\            optional models folder
  logs\              server.log and rotated log
```

## Known limitations

- Flash attention may improve or reduce performance depending on the model; benchmark before saving a preset.
- The VRAM pre-check requires a recent `llama.cpp` build containing `llama-fit-params`.
- BoringSSL-enabled builds require internet access during configuration.
- Auto-start is a per-user scheduled task, not a Windows service.
- Router aliases estimated from local cache paths may differ from server aliases; verify them in the log before relying on per-model presets.
- Hugging Face cache cleanup may overestimate recoverable space when snapshots use hardlinks.
- Other hardware and toolchain combinations may require additional troubleshooting.

## Contributing

Issues and pull requests are welcome. Include your Windows version, CPU/GPU, Visual Studio, CUDA and PowerShell versions, selected build profile, and relevant `logs\server.log` excerpts. Remove secrets before sharing.

## License

[MIT](LICENSE). This project is not affiliated with [ggml-org/llama.cpp](https://github.com/ggml-org/llama.cpp).
