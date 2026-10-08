# Specs, docs, and MCP

This repo (`docs`) is for human-facing documentation: what the project
is, why it exists, and how its architecture works. It changes often and
is meant to be read start-to-finish by a newcomer.

## OJPs (planned, not yet created)

A separate repo, `OJPs` ("OpenJarvisProject Proposals"), will hold a
Nostr-NIPs-style set of numbered, implementation-agnostic markdown
specs — for anyone building a *compatible* third-party client, backend,
or skill against a stable protocol, without needing to read reference
code. Same split Nostr (NIPs vs. no real docs wiki) and Matrix
(spec.matrix.org vs. Element's own docs) both use, for the same reason:
a stable public spec and a friendly getting-started guide serve
different audiences and change at different rates.

Candidate content for OJPs, once the relevant architecture is further
along:

- the CRDT sync wire format and device-pairing handshake
- the memory/second-brain DB schema
- a model distribution/provenance format (base lineage, compression
  preset used, benchmarks, license — so third parties can verify or
  publish their own compatible pruned/quantized models)
- the avatar/persona config format (VRM-based)
- the wake-word model packaging format
- **the core client-daemon API** — how any UI (desktop app, Android
  app, a third party's own client) talks to the local agent process at
  all

## MCP

For agent-to-tool invocation specifically, this project is leaning
toward adopting **[MCP](https://modelcontextprotocol.io)** (Model
Context Protocol) rather than inventing an equivalent from scratch.
MCP's scope is narrow — just tool invocation — so it doesn't make OJPs
redundant; everything listed above still needs its own spec regardless.

There's also a smaller case for *extending* MCP via a small OJP rather
than using it as-is: MCP's base spec has no notion of per-tool resource
cost, which matters for a project whose whole premise is running on
weak hardware (e.g. "this tool is expensive, only call it from the big
model, not the router").
