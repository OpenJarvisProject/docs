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

**RX 580 ROCm caveat — confirmed dead, not just a theoretical risk**:
tested directly (`rocminfo` fails at the HSA runtime level even with
the `HSA_OVERRIDE_GFX_VERSION` workaround). `burn`'s `wgpu` (Vulkan)
backend (see [`runtime.md`](runtime.md)) is the confirmed-working
fallback instead — `wgpu` enumerates this card via Vulkan specifically
(`RADV POLARIS10`, `DiscreteGpu`, preferred over its GL fallback
adapter), and a real tensor computation through it produces a correct
result. See
[`OpenJarvisProject/model-tooling`](https://github.com/OpenJarvisProject/model-tooling).

### Pruning method: whole-layer removal, not unstructured sparsity

Wanda-style unstructured/N:M sparsity (zeroing individual weights) was
considered and rejected — it doesn't shrink tensor shapes, so it
produces no real file-size reduction and no speedup on `llama.cpp`
(no sparse-tensor-core support). The pipeline instead removes entire
transformer decoder blocks (structural/depth pruning), ranking each
layer by cosine similarity between its input and output across
calibration text (the "Block Influence" idea from ShortGPT-style depth
pruning) — high similarity means the layer is acting close to an
identity function and is safe to remove.

**Removal is iterative and perplexity-validated, not a single batch** —
this was a real correctness fix, not a refinement. A first version
measured importance once on the intact model and removed several
layers at once based on that snapshot; tested fairly (preserving the
checkpoint's native dtype), it broke the model outright, producing
fluent-looking complete gibberish rather than degraded-but-functional
output. Removing one layer changes what the next layer receives as
input, which a one-shot ranking never accounts for — sharper on a
small model (24 layers) than on the larger models this technique is
usually demonstrated on. The pipeline now re-measures importance after
every single removal, validates each candidate against a held-out
perplexity check, falls back to the next candidate on failure, and
stops early rather than force a bad removal if nothing remaining
passes.

Confirmed on `Qwen/Qwen2.5-0.5B-Instruct`: targeted removing 6 of 24
layers, correctly stopped at 5 once every remaining candidate failed
validation — 24→19 layers, 954MB→812MB on disk, output stays fluent
and on-topic.

### Stage 3 closed: controlled original-vs-pruned comparison

Both the original and pruned models were converted through the exact
same GGUF pipeline (`convert_hf_to_gguf.py` + a from-source
`llama-quantize` build, both missing from the published Rust crate) to
matched Q4_K_M quantization — using the externally-downloaded original
GGUF as-is would have confounded quantization methodology with layer
count as two variables at once. Results (5 reps/condition, low
variance):

- **CPU**: pruned uses 11.1% less peak RAM, runs 11.6% faster.
- **GPU/Vulkan**: model weights -13.3% VRAM, KV-cache -20.8% (exactly
  matches the 19/24 layer ratio), **compute/scratch buffer unchanged
  (0%)** — that buffer turns out to be driven by `n_ubatch` (max
  micro-batch size), not model depth at all; confirmed by isolating it
  separately: exactly linear (512→298.5MiB, 256→149.25, 128→74.625,
  64→37.3, each exactly half), independent of pruning or model size.
  Net total VRAM savings land at -7.9%, well below the raw weight
  reduction, because that fixed buffer is nearly half of total VRAM for
  a model this small. Practical implication: `n_ubatch` is a real,
  independent lever for cutting GPU memory on weak hardware, unrelated
  to pruning — worth exposing as a real config option later.

### Threshold sensitivity and a real removability cliff

Confirmed the method is deterministic first (identical repeat runs,
zero diff) before trusting any threshold comparison — ruled out CPU
floating-point non-determinism as a confound. A looser
`--max-ppl-ratio` doesn't just gate the final cutoff, it reshapes the
whole greedy trajectory. On this model, 1.5x and 3.0x land at the same
count (5/24) via different paths; 10.0x reaches 7/24 — a real
structural cliff between those thresholds, not a smooth gradient.

### Model size doesn't clearly change removability (yet)

Repeated the identical method on `Qwen/Qwen2.5-1.5B-Instruct` (28
layers): removed 6/28 (21.4%) at the same threshold, almost exactly
matching the 0.5B model's 20.8%. The "bigger models tolerate
proportionally more removal" expectation from depth-pruning literature
(usually shown at much larger scale jumps, e.g. 7B vs 70B) isn't
visible at this modest 3x size difference — worth knowing before
assuming the method scales favorably on model size alone.

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

- **Main desktop** (Ryzen 5 4500, ~23GB RAM, AMD RX 580 — confirmed via
  driver string, not an RX 590 despite the PCI ID being shared across
  the whole Ellesmere/Polaris rebadge family): can run 7B+3B+1B
  simultaneously (~1.5GB total), plenty of headroom for OS/apps. Also
  the pruning workhorse, see above. This is also where
  [`runtime.md`](runtime.md)'s Stage 1/2 validation actually ran —
  see [`OpenJarvisProject/core`](https://github.com/OpenJarvisProject/core).
- **A converted Chromebook** (Mediatek MT8183, 8-core @ 1.99GHz, Mali
  GPU, Debian aarch64, ~3.79GiB RAM total, already running
  `fht-compositor` as its Wayland compositor): the real "weak CPU,
  ~4GB RAM" case. Target here is roughly a 1B model (~256MB) for
  responsive basic tasks — a 1B router + ~1.5-2.5B main model fits
  comfortably (≈750MB-1.3GB resident) inside its RAM budget. This
  machine is also a natural dogfooding target for the compositor's
  AI-owned-input-seat work (see `sync-and-devices.md`), on genuinely
  low-end ARM hardware, not just the AI side.
