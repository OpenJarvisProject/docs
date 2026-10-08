# Sync and devices

## Per-device instances, not client-server

Every device (desktop, laptop, phone, however weak) runs its own full,
independent, 100% offline-capable instance of the runtime — explicitly
**not** one instance on a home machine that other devices call into
(that design would also kill real multitasking across devices: one
shared instance means one device's heavy task blocks everyone else's).
Each device picks model tiers sized to its own hardware from
[`model-compression.md`](model-compression.md)'s table.

This is validated as realistic by existing prior art, not speculative:
it's exactly how `llama.cpp` cross-compiled via the Android NDK already
works in apps like MLC Chat, Private LLM, and ChatterUI — fully
offline, in-app, no server.

## The second-brain / memory database

Requirements: token-efficient retrieval (never dump the whole DB into
context), fully offline-first per device, syncs between devices with no
central server/single point, and setup has to be easy/semi-automatic
for the end user.

- **Retrieval**: standard local RAG — a small local embedding model
  (tens of MB) plus a lightweight local vector index. `sqlite-vec`
  (pure C, embeds into SQLite, has Rust bindings) is the concrete
  candidate — keeps the memory store a single local SQLite file, which
  satisfies offline-first for free.
- **Cross-device sync without a central point**: a CRDT (Conflict-free
  Replicated Data Type) problem, not something to hand-roll. Planned
  pairing: **Automerge** (a Rust-native CRDT library, built by the team
  that coined "local-first software" for exactly this scenario — every
  device has a full local copy, edits offline, merges automatically
  later) for the data layer, plus **Iroh** (a Rust P2P networking
  library built for NAT traversal/relay without running your own
  server) for transport. This pairing is already a known, idiomatic
  pattern in the Rust local-first community, not a novel invention.
- **Pairing UX**: Syncthing's device-linking model (scan a QR/short
  code to add a device, then opportunistic sync whenever reachable) is
  the UX to copy for "semi-automatic." Note Syncthing itself is
  block-level file sync, not CRDT/structured-data merge — a UX
  reference only, not the actual tool.

### LAN discovery is not a security boundary

mDNS/LAN broadcast is for *discovery only* — finding an address — and
must never be relied on for security, since sync has to stay safe even
on a hostile network (public WiFi, etc.). All sync traffic — LAN, Iroh,
or a VPN mesh — gets encrypted and mutually authenticated using a key
established once during device pairing, the same general approach
WireGuard itself uses (the Noise Protocol Framework), regardless of
which network path carried it. Minor hardening note: mDNS announcements
are visible to everyone on the same network, so broadcast an opaque
device identifier, not a human-readable name.

### Remote control, decoupled from any specific mesh tool

Controlling one device's instance from another (not just background
memory sync, but live request/response) should work over "any reachable
IP" — a LAN mDNS address, an Iroh P2P session, or a VPN-mesh virtual IP
(Tailscale, Netbird, plain WireGuard, Headscale, whatever) all just look
like a normal address once connected. This means no per-vendor
integration code is needed: whichever mesh tool the user runs (or none)
is their own infrastructure choice, entirely outside the app. Worth
noting for anyone choosing one to recommend: Netbird's control/
management plane is itself open-source and self-hostable, unlike
Tailscale's closed-SaaS coordination server (Headscale is an OSS
reimplementation of just that piece) — a better fit for this project's
own self-hosting philosophy if a recommendation is ever made.

## Android: zero Google dependency

Hard requirement: not just "doesn't call GMS at runtime" — the
manifest itself must not reference any Google component, even unused
ones. A real bug surfaced this: an earlier prototype's voice-interaction
config hardcoded `android:recognitionService` to a Google component.
That attribute is mandatory and resolved at assistant-registration time
even though the code never actually invoked it — on a device without
the Google app (a de-Googled ROM, GrapheneOS, etc.), this can silently
keep an app out of Settings → Default apps → Digital assistant app.

**Fix**: implement a no-op `RecognitionService` stub inside the app
itself and point `recognitionService` at that instead — removes the
only real GMS touchpoint. Payoff beyond fixing the bug: a genuinely
zero-GMS app also qualifies for **F-Droid** distribution, a natural
extra channel alongside or instead of the Play Store.

## Android: default assistant app + wake word

Becoming Android's assist-gesture handler (replacing "Hey Google") uses
the platform's `VoiceInteractionService` API — an existing prototype
already implements this correctly (service + session-service
declarations, `BIND_VOICE_INTERACTION` permission, `supportsAssist`).
Android requires the user to manually pick it in Settings → Default
apps → Digital assistant app; it can't be silently seized, by platform
design.

Wake-word detection uses openWakeWord-style tiny ONNX models (hundreds
of KB to a few MB) — a separate, already-solved, tiny-footprint problem,
unrelated in scale to the LLM router/main-model compression work above.
The stock openWakeWord phrase, "Hey Jarvis," conveniently doubles as the
project's own on-brand default; per-user custom wake words (e.g. a
user's own personalized name for their instance) are a planned
customization option, not the shipped default.
