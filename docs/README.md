# Documentation index

Five audiences, five entry points. Start with the row that matches why you are here.

| If you are… | Read |
| :--- | :--- |
| **using** ReUnite on a phone | [JOINING.md](JOINING.md) — no terminal, no setup, no account |
| **running** it across several computers | [SETUP.md](SETUP.md) — toolchain, firewall, discovery, several nodes on one laptop |
| **building** it for a phone or laptop | [MOBILE.md](MOBILE.md) — Android, iOS and macOS builds, and the manual Xcode steps iOS needs |
| **demoing** it | [DEMO.md](DEMO.md) — a scripted walkthrough: multi-hop relay, private networks, kick voting, ghosting, zones |
| **changing** it | [ARCHITECTURE.md](ARCHITECTURE.md) — wire formats, routing, crypto, the threat model, and what it deliberately does not protect against |
| **taking it over** | [HANDOVER.md](HANDOVER.md) — the whole project in one document: how it works, what is verified, what is not, and what to do first |

## Elsewhere in the repository

| Path | What it is |
| :--- | :--- |
| [`../plan.md`](../plan.md) | The original product and architecture plan. Code comments cite it by section. |
| [`../phase/`](../phase/) | The build plan, one file per phase, plus the **deviations register** — every place the code deliberately departs from `plan.md`, and why. |
| [`../phase/phase-2e-hardware-verification.md`](../phase/phase-2e-hardware-verification.md) | Where the project actually is. The only phase that cannot be done from a laptop. |
| [`../mobile/README.md`](../mobile/README.md) | The Flutter app's own structure and test notes. |
| [`../web/README.md`](../web/README.md) | The marketing site: map model, heat layer, theming, and the Google Maps key. |

## One command

```bash
./scripts/check.sh          # cargo test + cargo build --release + flutter analyze + flutter test
```
