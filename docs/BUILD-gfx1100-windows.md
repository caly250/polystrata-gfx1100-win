# Building `strata-poly` for gfx1100 on Windows 11

Reproducible build recipe for the AMD HIP backend of PolyStrata on Windows, as
used to produce the binaries in [`engine/`](../engine/).

Measured on: Radeon PRO W7800 (gfx1100), Windows 11, Intel i5-11400, 32 GB RAM.

---

## What you need

| Requirement | Notes |
|---|---|
| Windows 10 or 11, 64-bit | |
| AMD Radeon RX 7900 XT/XTX, PRO W7800, or another **gfx1100** card | gfx1101/gfx1200/gfx1201/gfx1030 also supported upstream, but the prebuilt binary here is gfx1100-only |
| Current **AMD Software: Adrenalin Edition** driver | The *only* thing you install yourself |
| **Visual Studio 2022 C++ build tools** | Required — the compile fails without them |
| Python 3.12 | Setup installs it for your user account if missing |
| ~90 GB free disk | ~7 GB ROCm wheels + ~25 GB build tree + ~23 GB model |

### What you do NOT need

- ❌ **No system ROCm / HIP SDK install.** Setup downloads AMD's TheRock wheels into `.rocm-win`.
- ❌ **No admin rights.**
- ❌ **No CUDA Toolkit.**
- ❌ **No NVIDIA card.**

> The engine's `BUILD.json` on this machine recorded `"rocm": "10.2.0a20260930"`,
> installed by setup. The system also happened to have ROCm 7.2 at
> `C:\Program Files\AMD\ROCm\7.2`, and **it was not used** for this build.

---

## Step 1 — Clone

```bat
git clone --depth 1 https://github.com/VecSzn/PolyStrata.git
cd PolyStrata
```

## Step 2 — Check the machine first (recommended)

```bat
py -3.12 setup.py --check
```

This reads your hardware and prints which model sizes fit. It installs nothing.
On the reference machine it printed:

```
Your AMD GPUs:
  GPU 0: AMD Radeon PRO W7800, 30 GB VRAM - can be used
[ok] GPU: AMD Radeon PRO W7800, 30.0 GB VRAM, gfx1100 (AMD: docs/AMD_HIP.md)
[ok] RAM: 32 GB
[ok] CPU: 11th Gen Intel(R) Core(TM) i5-11400 @ 2.60GHz (AVX-512)

Qwen3.6-35B-A3B UD-Q4_K_M: EXPERIMENTAL, measured on one AMD card on Windows,
  fits by setup's estimate (~10 GB of RAM on this card)
```

## Step 3 — Install

```bat
py -3.12 setup.py --family qwen36 --model UD-Q4_K_M --backend hip ^
   --context 131072 --vision no --yes
```

What setup does, in order:

1. **Installs Python packages** (numpy, cmake, ninja, pillow, psutil, …)
2. **Downloads llama.cpp source** at the pinned revision (`3cf0325` on 2026-11-06)
3. **Installs ROCm into `.rocm-win`** (~7 GB) from `nightly.repo.amd.com`:
   ```
   python -m pip install --index-url https://nightly.repo.amd.com/rocm/whl-next/ \
       rocm[libraries,devel,device-gfx1100]==10.2.0a20260930
   ```
4. **Compiles `strata-poly`** for gfx1100 → `engine-poly\strata-poly.exe`
5. **Downloads the model** (~23 GB) — or use `--gguf-dir` to point at a file you already have
6. **Extracts the tokenizer** into `packs\`
7. **Writes the start script** `run-qwen36-ud-q4_k_m.bat`

### Timing on the reference machine

| Step | Duration |
|---|---|
| Python packages | ~2 min |
| ROCm wheels | ~8 min |
| **Engine compile** | **~6 min** (16:06 → 16:12) |
| Model download | varies (use your own downloader, see below) |
| Tokenizer extraction | ~2 min |

> Different from the docs' "10–20 minutes" for the compile: this machine
> finished in about 6. Faster CPU, warm pip cache.

---

## Step 4 — Getting the model file faster

The built-in downloader pulls from Hugging Face and **repeatedly failed here**
with `SSL: UNEXPECTED_EOF_WHILE_READING` and stalled at 0 bytes/s. Measured speeds
from this network on 2026-11-06:

| Source | Single-connection speed |
|---|---:|
| huggingface.co | 0.6 MB/s |
| hf-mirror.com | 0.4 MB/s |
| **modelscope.cn** | **9.3 MB/s** |

Use a multi-threaded downloader against ModelScope:

```
https://modelscope.cn/models/unsloth/Qwen3.6-35B-A3B-MTP-GGUF/resolve/master/Qwen3.6-35B-A3B-UD-Q4_K_M.gguf
```

Put it at `<data-dir>\models\qwen36-UD-Q4_K_M\Qwen3.6-35B-A3B-UD-Q4_K_M.gguf` and
**delete any leftover `.part` file**, then run setup with `--gguf-dir`:

```bat
py -3.12 setup.py --family qwen36 --model UD-Q4_K_M --backend hip ^
   --context 131072 --vision no --gguf-dir "E:\PolyStrata-data\models\qwen36-UD-Q4_K_M" ^
   --yes --no-start
```

Setup verifies the SHA-256 and skips the download.

---

## Step 5 — Start

```bat
run-qwen36-ud-q4_k_m.bat
```

Or headless:

```bat
python serve\server.py --engine strata --config strata-qwen36-ud-q4_k_m.json --port 8080
```

First load takes **about 60 seconds** (reading 22.7 GB of weights and placing
18.7 GiB of experts on the card). The server prints `ready:` when done:

```
[strata] still starting (61 s) - please wait ...
ready: http://127.0.0.1:8080/v1  (context 131072 tokens)
```

---

## Pitfalls hit during this build

### 1. `pip` cannot reach PyPI

```
ConnectionResetError(10054) ... /simple/numpy/
```

**Fix:** configure a domestic mirror before running setup.

```bat
py -3.12 -m pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
py -3.12 -m pip config set global.trusted-host pypi.tuna.tsinghua.edu.cn
```

### 2. `--prompt-batch` above 4096 makes the engine refuse to start

```
strata-poly: a prompt batch is 1 to 4096 tokens
```

Valid range is **1–4096**. Anything larger is a hard startup failure, not a warning.

### 3. `--max-context 262144` silently spills to shared memory on a 30 GB card

No error, no warning — it just gets **2.2× slower**. The KV cache costs ~20 KB/token,
so 256K is 5 GiB, which looks affordable next to 18.7 GiB of experts. But the
prompt-batch activation buffer pushes the total past the card's budget and Windows
spills the overflow into GTT.

**Always verify with performance counters after changing the context:**

```powershell
Get-Counter '\GPU Adapter Memory(*)\Dedicated Usage','\GPU Adapter Memory(*)\Shared Usage' |
  Select-Object -ExpandProperty CounterSamples |
  Where-Object {$_.CookedValue -gt 100MB} |
  Select-Object Path,@{n='GB';e={[math]::Round($_.CookedValue/1GB,2)}}
```

Shared memory should stay around **0.4 GB**. If it is gigabytes, you are spilling.

### 4. Every reply is empty

```
the reply reached max tokens while still thinking, so it has no answer
```

**Fix:** add `"reasoning_budget_tokens": 4096` to the model config JSON.

### 5. Verification helpers

```bat
engine-poly\strata-device.exe --list-devices
engine-poly\strata-device.exe --selftest
```

With `.rocm-win\Lib\site-packages\_rocm_sdk_devel\bin` on `PATH`.

---

## Rebuilding after `git pull`

```bat
UPDATE.bat
```

Or rerun setup; it recompiles the engine when the source changed. Engine options
you added by hand to `"args"` are **not** carried over — setup rewrites them, so
re-add them afterwards.

---

## Optional: hipBLASLt tuning table

The engine uses `tools/hip/gfx1100-hipblaslt-<version>.txt` **only when the
version matches the installed library exactly**. This machine reported
`hipblaslt_version: 100500`, matching `tools/hip/gfx1100-hipblaslt-100500.txt`.

Confirm what you have:

```bat
type engine-poly\BUILD.json
```

If the table does not match, the engine says so and prefills through plain
hipBLAS — roughly **45% slower on very long prompts**, with identical answers.
Keep the table and the library paired.

---

## Architectural notes for gfx1100

- gfx1100 is **RDNA3**: it has `v_dot4_i32_iu8`, wave32 and 64 KiB LDS, but **no
  WMMA matrix cores**. The `STRATA_PF_FUSED` / `STRATA_HIP_WMMA` speed switches
  that help RDNA4 (gfx1200/gfx1201) are **not available here**.
- `STRATA_SH_STREAM` (shared-expert stream fork) is **off by default on HIP**.
  On an RX 6800 under Windows it was 12–22% *slower* when on.
- `STRATA_SCORES_THREAD=1` enables the exact long-prompt QSA scorer on gfx1100
  (opt-in). Reported elsewhere as 10–23% faster on 130K–240K prompts.
- Images always go through the CPU encoder on AMD, and on Windows they are off
  entirely.
