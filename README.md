# PolyStrata on AMD Radeon PRO W7800 (gfx1100) — Windows 11 benchmark & build recipe

Community benchmark report and reproducible build recipe for running
[PolyStrata](https://github.com/VecSzn/PolyStrata)'s `strata-poly` engine on an
**AMD Radeon PRO W7800 (gfx1100) under Windows 11**.

Every number here was measured on the machine described below, and every number
was taken from the engine's own timing output (`strata serve: prompt N tokens = ... read in M ms`).
Nothing is estimated.

---

## TL;DR

| | |
|---|---|
| **Model** | Qwen3.6-35B-A3B UD-Q4_K_M (35B total / 3B active) |
| **Engine** | `strata-poly` 0.1.45, HIP, built locally for gfx1100 |
| **Experts in VRAM** | **10,496 / 10,496 (100%)** — 18.7 GiB, nothing streamed |
| **Prefill** | **1,910 tok/s** @ 20K prompt, **1,220 tok/s** @ 42K prompt |
| **Decode** | **128–176 tok/s** (MTP draft acceptance 70–93%) |
| **Sweet spot** | **128K context** — zero VRAM spill, ~4 GB headroom |
| **Trap found** | **256K context spills 3.24 GB into shared memory and costs 2.2× prefill** |

---

## Hardware

| Component | Value |
|---|---|
| GPU | AMD Radeon PRO W7800, **gfx1100**, 30,704 MiB VRAM |
| CPU | Intel Core i5-11400 @ 2.60 GHz (AVX-512) |
| RAM | 32 GB |
| OS | Windows 11 |
| Disk | KIOXIA EXCERIA G2 NVMe SSD |
| ROCm | 10.2.0a20260930 (TheRock wheels, installed into `.rocm-win` by setup) |
| hipBLASLt | **100500** — matches the shipped `tools/hip/gfx1100-hipblaslt-100500.txt` tuning table |

> The card has 30 GB of VRAM, not 32 GB. `setup.py --check` reports it as
> "AMD Radeon PRO W7800, 30 GB VRAM - can be used". Windows budgets 29,893 MiB
> of it for the process (`strata: Windows budgets 29893 of this card's 30704 MiB`).

---

## The headline result: all experts fit in VRAM

```
strata-poly: loaded in 62.0 s; 10496 of 10496 routed experts in VRAM (18.7 GiB), 12 CPU threads
...
[strata] done: 8 tokens in 13 s (154.7 tok/s), expert cache 100.0% hit
```

A 35B-A3B model has ~18.7 GiB of routed experts in Q4. On a 30 GB card they all
fit, so **no expert is ever fetched from RAM or the SSD during inference**. This
is why the numbers below are so much higher than the 8 GB / 16 GB reports in the
PolyStrata docs, where only a fraction of the experts fit and the rest stream.

---

## Prefill across context configurations

Measured by setting `--max-context` and `--prompt-batch` in the model config,
restarting the engine, and reading dedicated/shared GPU memory from Windows
performance counters (`\GPU Adapter Memory(*)\Dedicated Usage` / `Shared Usage`).

| `--max-context` | `--prompt-batch` | VRAM dedicated | **VRAM shared** | Prompt tokens | **Prefill** | TTFT | Spill? |
|---:|---:|---:|---:|---:|---:|---:|---|
| 32K | 2048 | 24.70 GB | 0.42 GB | 4,290 | 1,518 tok/s | 2.8 s | no |
| 32K | 2048 | 24.70 GB | 0.42 GB | 20,445 | **1,910 tok/s** | 10.7 s | no |
| 64K | 2048 | 25.39 GB | 0.42 GB | 20,444 | 1,717 tok/s | 11.9 s | no |
| 64K | 2048 | 25.39 GB | 0.43 GB | 41,989 | 1,259 tok/s | 33.3 s | no |
| 96K | 2048 | 26.07 GB | 0.42 GB | 41,989 | 1,243 tok/s | 33.8 s | no |
| 96K | 2048 | 26.07 GB | 0.43 GB | 92,314 | 707 tok/s | 130.6 s | no |
| **128K** | **2048** | **26.77 GB** | **0.43 GB** | 41,989 | **1,220 tok/s** | 34.4 s | no |
| 128K | 2048 | 26.77 GB | 0.43 GB | 92,314 | 694 tok/s | 133.1 s | no |
| **256K** | **4096** | **28.53 GB** | **3.24 GB** | 41,993 | **570 tok/s** | 73.6 s | **YES** |
| 256K | 4096 | 28.53 GB | 3.24 GB | 92,318 | **443 tok/s** | 208.3 s | **YES** |

Raw data: [`data/bench-results.json`](data/bench-results.json)

### The 256K trap

A 256K context **fits in VRAM on paper**: the fp16 KV cache is about 20 KB/token,
so 256K is only 5 GiB, and 18.7 + 5 = 23.7 GiB < 29.2 GiB available. That
calculation is wrong, because it ignores the **prompt batch activation buffer**.
With `--prompt-batch 4096` on top of a 256K KV cache, dedicated VRAM reached
**28.53 GB** and Windows spilled **3.24 GB into shared (GTT) memory**.

Every expert or KV access that lands in that spilled region crosses PCIe, and
prefill collapses:

| | 128K config | 256K config | Difference |
|---|---:|---:|---|
| 42K-token prompt | 1,220 tok/s | 570 tok/s | **2.14× slower** |
| 92K-token prompt | 694 tok/s | 443 tok/s | **1.57× slower** |
| Shared (spilled) memory | 0.43 GB | 3.24 GB | **7.5× more** |

**Do not enable 256K on a 30 GB card with a 4096-token prompt batch.**

### Long context is bounded by attention, not by VRAM

Note that 92K-token prompts run at ~694–707 tok/s **even with zero spill**. The
drop from 1,220 tok/s (42K) to ~700 tok/s (92K) is quadratic attention
complexity, not a memory problem — no configuration fixes it.

---

## Decode

| Scenario | Prompt tokens | Generated | Decode | MTP drafts accepted |
|---|---:|---:|---:|---|
| Short chat | 18 | 32 | 163.0 tok/s | 23 / 26 |
| Short chat | 21 | 256 | 128.6 tok/s | 150 / 212 |
| 4K context | 4,161 | 200 | **152.4 tok/s** | 139 / 149 |
| Long output | 2,080 | 200 | 152.5 tok/s | 126 / 148 |

`strata-poly` runs MTP speculative decoding with the draft block that ships
inside the Unsloth GGUF. Draft acceptance is **70–93%** on ordinary
conversations, and that is the entire source of the 128–176 tok/s decode. The
engine reports it per request:

```
strata serve: prompt 4161 tokens = 0 reused + 4161 read in 1715 ms (2426.9 tok/s),
              200 generated in 1312 ms (152.4 tok/s), drafts accepted 139 of 149
```

---

## Recommended configuration

[`config/strata-qwen36-128k-final.json`](config/strata-qwen36-128k-final.json) —
**128K context, no spill, ~4 GB VRAM headroom left for the desktop.**

```json
{
 "args": [
  "--model", "...\\Qwen3.6-35B-A3B-UD-Q4_K_M.gguf",
  "--max-context", "131072",
  "--expert-profile", "...\\expert-profile.txt",
  "--conversation-cache-mib", "auto",
  "--tail-skip", "6",
  "--prompt-batch", "2048",
  "--kv-whole",
  "--vram-reserve-mib", "1536"
 ],
 "reasoning_budget_tokens": 4096,
 "fit_max_tokens": true,
 "answer_room": true,
 "reasoning_loop_recovery": "stop"
}
```

### Why each non-obvious key is there

| Key | Why |
|---|---|
| `--kv-whole` | On ROCm, VRAM released by `hipMemUnmap`/`hipMemRelease` never returns to the free pool. The attention caches must take their VRAM at load time, or a long prompt is later refused with "no VRAM for the attention cache". Setup writes this on AMD automatically. |
| `--vram-reserve-mib 1536` | Leaves headroom so the Windows desktop and driver never force a spill. |
| `--prompt-batch 2048` | **Max allowed is 4096** (`strata-poly: a prompt batch is 1 to 4096 tokens`). 4096 is what causes the 256K spill. 2048 is the safe large-context choice. |
| `reasoning_budget_tokens: 4096` | **Without it the model thinks until `max_tokens` and returns an empty answer.** See below. |
| `fit_max_tokens: true` | Lets a request that exceeds the window be shortened instead of refused with HTTP 400. |
| `answer_room: true` | Reserves the last quarter of `max_tokens` for the actual answer rather than letting thinking consume it. |

### The empty-answer trap (reproduced here)

On the first run every reply came back empty. The engine log said exactly why:

```
[strata] the reply reached max tokens while still thinking, so it has no answer:
         a thinking budget (reasoning_budget_tokens, in the request or in
         strata-<model>.json for every request) leaves room to answer
```

This is **not a model or engine defect** — it is a configuration gap. Qwen3.6 in
thinking mode will happily consume the whole `max_tokens` budget on reasoning.
Setting `reasoning_budget_tokens` fixes it:

| | Before | After |
|---|---|---|
| `finish_reason` | `length` (cut off mid-thinking) | `stop` |
| Answer content | empty | 139 chars, correct Chinese explanation |
| Thinking | 612 tokens, no answer | 1,955 chars reasoning, then a real answer |

---

## Known limitations on this configuration

- **Conversation cache stays off.** With 32 GB of RAM the engine reports
  `conversation cache off: not enough free RAM to park conversations`. For
  multi-turn agent work every turn re-prefills the whole conversation. 64 GB of
  RAM would fix this.
- **Images are off.** The image encoder is a CUDA program; on AMD under Windows
  it is unavailable (`--vision no`).
- **No `--calibrate`.** Calibration is still NVIDIA-only for the AMD backend.
- **Not bit-identical to CUDA.** PolyStrata does not promise identical greedy
  tokens across backends.
- **EXPERIMENTAL status.** Upstream's docs say the Windows AMD path for
  `strata-poly` has only been run on an RX 9070 XT (Tiel Coder) and an RX 5700 XT.
  **This is, to our knowledge, the first reported run of Qwen3.6-35B-A3B on an
  RX 7900-class (gfx1100) card under Windows.**

---

## Reproducing the build

See [`docs/BUILD-gfx1100-windows.md`](docs/BUILD-gfx1100-windows.md).

The short version:

```bat
git clone https://github.com/VecSzn/PolyStrata.git
cd PolyStrata
py -3.12 setup.py --family qwen36 --model UD-Q4_K_M --backend hip ^
   --context 131072 --vision no --yes
```

Setup installs the TheRock ROCm wheels into `.rocm-win` (~7 GB), compiles
`strata-poly` for gfx1100, and downloads the model. **No system ROCm install,
no admin rights, and no CUDA Toolkit are needed.** Visual Studio 2022's C++ build
tools are required for the compile.

---

## Prebuilt binaries

[`engine/`](engine/) contains the exact engine used for these measurements:

| File | Size | License |
|---|---:|---|
| `strata-poly.exe` | 26.7 MB | MIT (PolyStrata) |
| `amd_comgr.dll` | 124.1 MB | Apache 2.0 with LLVM exceptions |
| `amdhip64_7.dll` | 16.4 MB | MIT (AMD) |
| `rocm_kpack.dll` | 0.2 MB | MIT (AMD) |

License texts are in [`licenses/`](licenses/).

`BUILD.json` records exactly what produced it:

```json
{"source": "local-hip", "backend": "hip", "platform": "windows-x64",
 "version": "0.1.45", "archs": ["gfx1100"],
 "rocm": "10.2.0a20260930", "hipblaslt_version": 100500}
```

> These binaries are built for **gfx1100 only**. The engine checks the card's
> architecture at startup and refuses to run on anything else rather than
> failing later with "invalid device function".

---

## License

The benchmark data, configuration files and documentation in this repository are
released under **MIT**. PolyStrata itself is MIT (© 2026 Niko1221 and the Strata
contributors; © 2026 VecSzn and the PolyStrata contributors). The bundled ROCm
libraries retain their own licenses, reproduced in [`licenses/`](licenses/).

## Acknowledgements

- [PolyStrata](https://github.com/VecSzn/PolyStrata) and [Strata](https://github.com/Niko1221/Strata)
- [Qwen](https://huggingface.co/Qwen) for Qwen3.6-35B-A3B
- [Unsloth](https://huggingface.co/unsloth) for the Dynamic Q4_K_M quantisation and the MTP draft block
