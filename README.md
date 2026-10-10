# OpenJarvisProject (OJP)

A generic, FOSS, local-first Jarvis-like AI agent — for Linux and Android,
runs on your own hardware, no account, no cloud dependency required.

Loosely inspired by [Omarchy](https://omarchy.org)'s philosophy (an
opinionated, cohesive, batteries-included experience) but applied to a
personal AI agent instead of a desktop environment: install it, and it
works, with your own local model ("brain") and your own local memory
database, on whatever device you run it on — desktop, laptop, or phone.

> The project is named for recognition, not for the specific Marvel
> character. See [`vision.md`](vision.md) for naming reasoning and the
> trademark consideration that went into it.

## Status

Early architecture/planning stage. No code exists yet. This repo exists
to document decisions as they're made, so the project has a stable
shared understanding before implementation starts, and so later
contributors don't have to reconstruct *why* a decision was made from
scratch.

## Start here

- [`vision.md`](vision.md) — what this project is, why it exists,
  naming, and the persona/branding model
- [`architecture/`](architecture/) — the technical decisions made so
  far: model compression, runtime language split, cross-device sync,
  cloud hosting plans
- [`specs.md`](specs.md) — how this repo relates to the separate `OJPs`
  spec repo (for third-party implementers) and to MCP

## Repo layout in this org

- **`docs`** (this repo) — human-facing documentation: what, why, how
- [**`OJPs`**](https://github.com/OpenJarvisProject/OJPs) —
  Nostr-NIPs-style numbered spec repo, for anyone implementing a
  compatible client/backend/skill against a stable protocol, not for
  general reading
- [**`core`**](https://github.com/OpenJarvisProject/core) — the
  reference runtime (Rust + `llama.cpp`). Early: a CPU/Vulkan inference
  smoke test only so far, not yet an agent
- Model-tooling (pruning/quantization pipeline) repo doesn't exist yet

## License

Not yet decided. Do not assume any particular license applies to this
repo's content until a `LICENSE` file is added.
