# Specs, docs, and MCP

This repo (`docs`) is for human-facing documentation: what the project
is, why it exists, and how its architecture works. It changes often and
is meant to be read start-to-finish by a newcomer.

## OJPs

A separate repo, [`OJPs`](https://github.com/OpenJarvisProject/OJPs)
("OpenJarvisProject Proposals"), holds a Nostr-NIPs-style set of
numbered, implementation-agnostic markdown specs — for anyone building
a *compatible* third-party client, backend, or skill against a stable
protocol, without needing to read reference code. Same split Nostr
(NIPs vs. no real docs wiki) and Matrix (spec.matrix.org vs. Element's
own docs) both use, for the same reason: a stable public spec and a
friendly getting-started guide serve different audiences and change at
different rates.

Current OJPs (all `Draft` — see [`OJP-1`](https://github.com/OpenJarvisProject/OJPs/blob/main/ojp-1.md)
for what that status actually means):

- **OJP-2**: the core client-daemon API — how any UI (desktop app,
  Android app, a third party's own client) talks to the local agent
  process at all
- **OJP-3**: the CRDT sync wire protocol and device-pairing handshake
- **OJP-4**: the memory/second-brain DB schema
- **OJP-5**: a model distribution/provenance format (base lineage,
  compression preset used, benchmarks, license — so third parties can
  verify or publish their own compatible pruned/quantized models)
- **OJP-6**: the avatar/persona config format (VRM-based)
- **OJP-7**: the wake-word model packaging format
- **OJP-9**: the router dispatch contract between the always-on router
  and on-demand specialist models
- **OJP-10**: non-normative recommended hardware tiers

## MCP

For agent-to-tool invocation specifically, this project adopts
**[MCP](https://modelcontextprotocol.io)** (Model Context Protocol)
rather than inventing an equivalent from scratch — see **OJP-8** for
the adoption itself and the one project-specific extension
(per-tool resource-cost hints, so a lightweight router model doesn't
invoke something too expensive for it to run). MCP's scope is narrow —
just tool invocation — so it doesn't make OJPs redundant; everything
else listed above still needs its own spec regardless.
