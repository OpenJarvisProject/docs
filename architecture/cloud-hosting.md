# Cloud hosting (planned, optional)

## The plan

A paid cloud tier that only runs lightweight models (~3B parameters) on
the maintainer's **own home hardware** to start (not rented cloud
GPUs), funded donation-based until there's enough revenue for real
infrastructure. Model choice for this tier is flexible. Core pitch:
genuinely more private than mainstream AI products — **no chat saves,
no logs of what the AI is used for.**

This is entirely optional infrastructure on top of the core project —
nothing about the free/local/self-hosted side of OpenJarvisProject
depends on this tier existing.

## Flagged risks — unresolved, should be addressed before real money changes hands

- **Network isolation**: a public paid service should not run on the
  same LAN/host as any other home infrastructure. Use a separate VLAN
  or a reverse tunnel (Tailscale Funnel, Cloudflare Tunnel) instead of
  directly port-forwarding the home IP, so a bug in the AI-serving code
  can't become a path into other home devices.
- **Residential ISP terms of service**: most residential plans
  technically prohibit running commercial/paid servers on a home
  connection — worth checking the actual ISP's terms before launch.
- **Payment/business/tax**: accepting real payments (even
  donation-framed) eventually raises a real business-entity and tax
  question — not addressed yet.
- **"No logs" has to be an architecture decision, not a promise**: if
  this ever grows beyond home hardware into rented GPU capacity, the
  upstream provider's own infrastructure may log at a layer this
  project doesn't control — the privacy claim needs to be
  independently true at whatever layer is actually running it.
- **Abuse/legal exposure**: a genuinely no-log, low-friction paid
  inference API is also attractive for abuse (illegal content, spam),
  and "we truly can't see what anyone does with it" is real legal
  exposure for a sole operator — worth deciding deliberately rather
  than backing into it once everything else already exists.
