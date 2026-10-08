# Vision

## What this is

A generic, FOSS, local-first Jarvis-like AI agent, for both Linux and
Android. Each user's own device(s) host their own model ("brain") and
their own local memory database — not a cloud service you log into.
The goal is zero required technical setup: Claude-like polish, Jarvis-
like capability, usable by someone who has never configured a Linux
service in their life.

Design philosophy loosely follows [Omarchy](https://omarchy.org): an
opinionated, cohesive, batteries-included experience, rather than a
pile of configurable parts the user has to assemble themselves — just
applied to a personal AI agent instead of a desktop environment.

A later idea, not yet started: dotfiles-style backup/import of a user's
own setup and personalization (the UnixPorn pattern — save your config
to your own GitHub, restore or share it from there), so a user's
customized assistant is portable and backed up the same way Linux
ricers already back up their configs.

## Relationship to ZELIA

This project is a fresh start, not a fork. The original `ZELIA` project
(a personal, single-machine voice/text AI agent, Manjaro/Arch-specific)
proved out a lot of real architecture — an agent loop with tool
dispatch, a small/large-brain split, a local memory store, Wayland
input handling across compositors — but is deeply coupled to one
machine's hardware and distro. OpenJarvisProject intends to borrow
*ideas and lessons learned*, not literal code, and build something
portable from the start.

## Naming

The org name is `OpenJarvisProject` (informally "OJP," matching how
[OBS Studio](https://obsproject.com)'s actual GitHub org is
`obsproject`, not `obs` — people say the short form even though the
org handle is the full name).

**Known, accepted trade-off**: "Jarvis" carries real trademark exposure
— Disney/Marvel's J.A.R.V.I.S. (Iron Man's AI, backronym "Just A Rather
Very Intelligent System") led to actual legal pressure on a real
product, [Jarvis.ai](https://www.jasper.ai), which fully rebranded to
Jasper in 2021. This was a deliberate choice, made for the recognition
value of a widely-understood AI-assistant name, weighed against that
risk — not an oversight.

## Persona, not project identity

The *project's* default assistant persona must not be a specific
person's personal name. The precedent: Mycroft AI, Home Assistant, and
similar open assistants ship with a configurable default persona name,
distinct from the project's own brand — the shipped default here should
be similarly neutral, with full customization (name, voice, avatar)
easy for anyone to set up for themselves. A user naming their own
instance something personal (e.g. the original author's own instance,
"Zelia") is a configuration choice, never the shipped default.

- **Default voice**: male-leaning/neutral, not feminine by default,
  specifically because the default must not read as personal.
- **Avatar format**: [VRM](https://vrm.dev) (glTF-based, royalty-free) —
  chosen over Live2D because Live2D's Cubism SDK has a non-free
  commercial license past a revenue threshold, which would conflict
  with a FOSS/paid-hosting project (see
  [`cloud-hosting.md`](architecture/cloud-hosting.md)). Desktop-pet/
  VTuber-style presentation, fully optional, is an explicit goal —
  including letting users treat their own customized instance as a
  companion/"waifu" persona if they choose to.
