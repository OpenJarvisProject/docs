# Model compression

Goal: run genuinely useful local models on cheap/weak hardware without
losing much quality — the core enabling tech behind "runs on your own
devices, no cloud, no heavy setup."

## The five techniques (stacked together)

1. **Pruning** — remove ~40% of low-contribution neurons/heads/layers.
   ~40% smaller, negligible quality loss alone.
2. **Factorization** — replace a big weight matrix `W` with two small
   ones, `A(k×256) × B(256×output)`. `k=128` ≈ 2-3x smaller, ~95%
   quality, slightly slower (two matmuls instead of one).
3. **Quantization** — lower bit precision. FP32→INT4 ≈ 4x smaller,
   ~95% quality. INT4→INT2 ≈ 8x smaller, ~90% quality (aggressive —
   stack with the other techniques rather than relying on quantization
   alone).
4. **Smart/layer-offloaded loading** — don't load the whole model into
   RAM/VRAM; keep active layers resident, prefetch the next layer while
   computing the current one, keep the rest on disk/system RAM. This is
   what lets a 70B-class model run in 4-8GB instead of 70GB.
5. **Specialized models (router + experts)** — one always-on tiny
   router (~1B, ~256MB) decides what's needed, then loads a task-
   specific specialist on demand (e.g. 3B coding, 7B reasoning) instead
   of running one giant generalist model for everything.

**Combined example** (illustrative, not a guarantee): 70B FP32 (280GB)
→ INT2 quant (8.75GB) → 40% pruning (5.25GB) → k=128 factorization
(3.5GB) → learned quant (2.8GB) → structured pruning (2.2GB) ≈ **2.2GB
on disk, 3-4GB resident**, ~127x smaller, ~5-10% quality loss.

## Target size/latency table

| Model | Disk | RAM on load | Use case | Speed |
|---|---|---|---|---|
| 1B | 256MB | 256MB | quick/1-shot | <1s |
| 2.5B | 500MB | 500MB | better reasoning | 2-5s |
| 3B | 300MB | 300MB | coding help baseline | 3-8s |
| 7B | 1GB | 1-2GB | main assistant/complex reasoning | 5-15s |
| 70B | 2-3GB | 4-8GB (offloaded) | deep/overnight tasks | 20-60s |

## Quality/efficiency presets

- **High quality**: INT4, 20% pruning, k=256 → 1.5-2x larger than
  target, ~98% quality.
- **Balanced (default)**: INT4, 40% pruning, k=128 → hits the table
  above, ~92-95% quality.
- **Extreme/CPU-only**: INT2, 60% pruning, k=64 → 0.5x the table sizes,
  ~85-90% quality.

## Pruning/publishing pipeline

Pruning (and any fine-tuning/distillation recovery pass) happens
**once, offline, by the maintainer** — never on an end user's device.
Proper structured pruning needs calibration-data forward passes plus
usually a short fine-tune to recover quality: realistically GPU-hours
to GPU-days per base model, infeasible as a per-install step even on a
real desktop, let alone phone-class hardware.

This runs on the maintainer's **RX 580** (otherwise idle), with
finished pruned+quantized weights uploaded to a **Hugging Face account
under the org** (or a self-hosted mirror later, if HF's logging/ToS
ever conflicts with this project's privacy positioning). End-user
devices only ever download and run a finished artifact. Quantization
alone is cheap enough to redo locally in minutes via `llama.cpp`'s
`quantize` tool, but ships pre-baked by default so a fresh install has
zero wait.

**RX 580 ROCm caveat**: current ROCm releases have dropped clean
support for the RX 580's chip generation (gfx803/Polaris) — it still
mostly works via `HSA_OVERRIDE_GFX_VERSION`, but it's fragile/
unofficial and could break on a ROCm upgrade. `burn`'s `wgpu`(Vulkan)
backend (see [`runtime.md`](runtime.md)) is the candidate fallback,
since Vulkan compute doesn't have ROCm's generation cutoffs — not yet
tried, just identified as the likely fix if ROCm support fully breaks.

## The free ultra-light tier doubles as the router

Two separate-sounding goals turn out to be the same model:

- The router (always-on, sized for the weakest target devices) needs to
  be as small as possible.
- A free, no-payment tier — a 1B-1.5B base model pruned down to ≤512MB
  — for basic automated tasks (e.g. a self-running RSS feed) without
  needing the full paid/self-hosted stack.

**These should be the same artifact**, not two separately maintained
models — one build serves both roles, cutting the number of small-model
pipelines to maintain from two to one. This isn't really new scope: the
balanced preset above already puts a 1B model at ~256MB, comfortably
under the 512MB free-tier target; a 1.5B base at similar compression
lands roughly 380-500MB.

One open nuance, not yet tested: router and standalone-assistant are
different jobs with possibly different quality bars — a router just
needs fast intent classification, a standalone free assistant needs to
actually complete small tasks. Validate the same build against both use
cases rather than assuming one size fits both.

## Real target hardware

Not hypothetical — these are actual machines this project is being
built against:

- **Main desktop** (i5 + 16GB RAM, AMD RX 580): can run 7B+3B+1B
  simultaneously (~1.5GB total), 10GB+ headroom for OS/apps. Also the
  pruning workhorse, see above.
- **A converted Chromebook** (Mediatek MT8183, 8-core @ 1.99GHz, Mali
  GPU, Debian aarch64, ~3.79GiB RAM total, already running
  `fht-compositor` as its Wayland compositor): the real "weak CPU,
  ~4GB RAM" case. Target here is roughly a 1B model (~256MB) for
  responsive basic tasks — a 1B router + ~1.5-2.5B main model fits
  comfortably (≈750MB-1.3GB resident) inside its RAM budget. This
  machine is also a natural dogfooding target for the compositor's
  AI-owned-input-seat work (see `sync-and-devices.md`), on genuinely
  low-end ARM hardware, not just the AI side.
