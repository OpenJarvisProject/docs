# Runtime architecture

## Language split

- **Rust** — everything around inference: agent loop, tool dispatch,
  networking, the memory/second-brain store, router logic, config,
  cross-platform packaging/app shell. Consistent with the maintainer's
  established preference across related projects (a pure-Rust package
  manager, a Rust trading app, Rust coreutils replacements in a sibling
  distro project, a Rust Wayland compositor fork).
- **C++ via FFI (`llama.cpp`)** — the actual tensor/kernel engine.
  Reason: `llama.cpp` has years of community-optimized GGUF/
  quantization/ARM-NEON work that pure-Rust engines (`candle`, `burn`)
  don't yet match, which matters a lot for the low-end ARM targets this
  project cares about (see
  [`model-compression.md`](model-compression.md)'s hardware section).
  This mirrors how Ollama (Go), GPT4All, LM Studio, and Jan already
  work: wrap the mature C++ inference core, write the application in
  whatever language you actually want. The binding crate actually used
  is [`llama-crab`](https://crates.io/crates/llama-crab) — actively
  maintained, safe high-level API, CPU/Metal/CUDA/Vulkan/ROCm/OpenCL/
  KleidiAI backend features.
- **`burn.dev`** (pure-Rust ML framework) — a narrow, targeted role, not
  a general replacement for the above:
  1. GPU-backend portability via its `wgpu`/Vulkan backend, as a
     fallback where ROCm support is degrading on older AMD cards (see
     the RX 580/ROCm note in `model-compression.md`) — Vulkan compute
     doesn't have ROCm's chip-generation cutoffs.
  2. A possible all-Rust alternative to Python/PyTorch for maintainer-
     side training/fine-tuning, if avoiding Python even on the
     maintainer's own machine turns out to matter.

### Open question: pruning-tooling language

Published pruning research (SparseGPT, Wanda, LLM-Pruner, etc.) ships
as PyTorch reference implementations.

- **Option A (current default)**: keep Python/PyTorch for the one-time,
  maintainer-side pruning/fine-tuning step only. It never runs on an end
  user's device, so it costs nothing against the "lightweight for
  users" goal — just reuse existing published code directly.
- **Option B**: go all-Rust via `burn`, reimplementing the pruning
  algorithms from their papers by hand. More consistent stack end to
  end, but real, nontrivial extra engineering with no existing
  off-the-shelf port to lean on.

Not yet decided — revisit before the pruning pipeline is actually
built.

## Validation methodology

Before any pruned model is tested, the runtime itself gets validated in
isolated stages, one variable at a time, so a later problem can be
attributed to a single cause. Planned to run on the maintainer's RX 580
machine:

1. **CPU-only, stock unmodified model** — ideally an already
   pre-quantized GGUF of the same model family planned for pruning
   later, so the file-format/loading path is proven too, not just the
   architecture. No GPU driver involved yet; confirms the Rust↔
   `llama.cpp` FFI, model loading, tokenization, and generation loop
   work at all.
2. **Add GPU acceleration as its own isolated step**, same stock model.
   Try `llama.cpp`'s Vulkan backend before ROCm on the RX 580 — the
   same chip-generation support cutoff flagged for ROCm likely applies
   to ROCm here too, and Vulkan avoids re-fighting it on a second tool.
3. **Only then substitute the project's own pruned/quantized checkpoint**
   on the same harness from step 2 — anything that breaks here is
   attributable to the compression pipeline, not the runtime.

**Status**: Stages 1 and 2 are done, live on the real RX 580 machine —
see [`OpenJarvisProject/core`](https://github.com/OpenJarvisProject/core).
Stage 2 produced byte-identical output to Stage 1, confirming
correctness across the CPU→Vulkan backend swap. That repo's README also
documents two real build gotchas hit along the way (a broken default
Vulkan driver, and a gap in `llama.cpp`'s own Vulkan CMake wiring) —
useful if you hit the same thing on similar hardware. Stage 3 (a
project-produced pruned model) hasn't happened yet, since the pruning
pipeline itself doesn't exist yet.
