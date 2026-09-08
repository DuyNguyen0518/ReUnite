# ReUnite — Takeover Guide

*Written 2026-09-08, revised 2026-09-09, by reading every file in the repository and running
every check it has. Everything marked "verified" below was actually executed on this machine;
everything else is sourced from the code or the existing docs and labelled as such.*

*The 2026-09-09 revision folded in a cleanup pass: the Dart suite was repaired, clippy taken to
zero, and the documentation reconciled with the code. Items struck through in §16 were fixed in
that pass; §13 gained the concurrency note below.*

---

## Part I — What you are taking over

### 1. In one paragraph

ReUnite is an **offline peer-to-peer emergency mesh network**. Every phone or laptop running it
is a node: it discovers neighbours over Bluetooth LE and/or Wi-Fi, relays traffic for the nodes
it can hear, and carries chat, GPS positions, an in-network SOS, one-byte panic codes, and
safe/unsafe area reports across the resulting mesh. There is no server, no account, no cell
tower and no internet anywhere in the design. A **Rust core** owns all protocol logic; a
**Flutter app** and a **terminal client** are two thin shells over the same core; **Kotlin and
Swift** own the Bluetooth radios because mobile operating systems will not let a Rust library
have them.

### 2. Provenance

| | |
|---|---|
| Your fork (`origin`) | `github.com/DuyNguyen0518/ReUnite` |
| Original (`upstream`) | `github.com/somerandomguy-coder/ReUnite` |
| Active period | 2026-08-29 → 2026-08-31 (three days) |
| Commits | 42, across 11 merged PRs |
| Contributors | 5–6 identities; effectively 3 people writing code |
| Size | ~21,600 lines: Rust 8,495 · Dart 4,229 · Kotlin+Swift 1,837 · Web 3,549 · Markdown 3,509 |

This was built at hackathon pace. That matters for how you present it (Part IV) and for what
you should expect to find (Part III).

> **The `phase/` directory was removed in `9da2b70`.** It held the build plan — one file per
> phase — plus the deviations register and the hardware-verification ladder. The plan itself is
> spent history, but two things in it were not, so they are reproduced in this document: the
> **invariants table** (§17) and the **verification ladder** (Appendix B). Everything else is
> recoverable with `git show ac2d337:phase/<file>`.

**Who wrote what** — this is the most important thing to have straight before you put it on a
profile, and it is good news for you:

| Author | Contribution |
|---|---|
| **DuyNguyen0518 (you)** | `plan.md` and the whole `phase/` plan; **Phase 1 in full** — the Rust mesh core: identity, crypto, packets, routing, SOS, panic codes, H3 zones, ghosting, battery; **Phase 2** — the Flutter app running on the real core over FFI; the native BLE transport seam (`external.rs`) and the Kotlin/Swift radios; **Phases 2A–2E** — build integrity, the safe/unsafe zone redesign, the iOS BLE fix, `MultiTransport`, `duty.rs`; and every document in `docs/` and `phase/`. **+14,579 / −1,122 lines across 6 commits.** |
| somerandomguy-coder | Initial Flutter scaffolding and Android chat/SOS UI; the C-FFI first cut; the `DatabaseStore` / SQL schema commit; the interactive OpenStreetMap + GeoJSON work; the entire `web/` marketing site; battery badges on the map. |
| Tuong Vy Vu / vutuongvy | macOS-vs-iPhone build split, Podfile and Swift build fixes. |
| Nam Le, Trieu Bao Huynh | Repo owner account and merge commits; little or no code of their own. |

You are the principal author of the engine and of the project's architecture. Say so plainly —
and equally plainly that the UI polish, the map screen and the marketing site were teammates'.

### 3. The mental model

```
        meshcli (terminal)              Flutter app (mobile/)
              │                                  │
              │  Command in / Event out          │  dart:ffi, JSON in / JSON out
              │                                  │
              └────────────► meshcore ◄──────────┴── meshffi (5 C functions + 5 BLE ones)
                                 │
                    Transport trait (3 methods)
                    ┌────────────┼─────────────┐
                  UDP        External        BLE (Linux)
                (Wi-Fi)    (Kotlin/Swift        via bluer
                            own the radio)
```

The single most important structural fact: **`meshcore` contains no UI code and `meshcli`
contains no protocol code.** Everything about the mesh — routing, encryption, peer ranking,
zone consensus, ghost detection, duty cycling — happens in Rust. Both front ends only display
what the core reports. Preserve that. The Dart layer is explicitly documented as forbidden from
growing protocol logic, and that rule is why two entirely different UIs behave identically.

---

## Part II — How it actually works

### 4. Repository map

```
Cargo.toml                  workspace: meshcore, meshcli, meshffi
crates/
  meshcore/                 the mesh. 3,400 lines of library + 1,250 of tests
    src/node.rs             (1,811) the actor: one task owns all state; Command/Event API
    src/router.rs           neighbours, learned routes, dedupe cache, link filter
    src/packet.rs           Frame / Packet wire format, TTL, path recording, signatures
    src/beacon.rs           Beacon v1: the 27-byte BLE-advertisement codec (no_std-ready)
    src/zones.rs            (542) H3 safe/unsafe aggregation, verdicts, consensus counts
    src/net.rs              networks: keys, epochs, members, invites, kick tally, re-key
    src/crypto.rs           Ed25519, X25519 sealed boxes, ChaCha20-Poly1305, HKDF
    src/identity.rs         UUID → hashed NodeId + persistent keypairs
    src/duty.rs             adaptive beacon/scan cadence ladder (pure function)
    src/status.rs           one-byte panic codes (no_std-ready)
    src/store.rs            JSON/JSONL persistence + an unused SQL schema constant
    src/transport/          udp.rs · external.rs · multi.rs · ble_linux.rs
    tests/                  38 tests: mesh.rs (26), external_transport.rs (5),
                            udp_seeded_unicast.rs (3), db_test.rs (1), bridge.rs (3)
  meshcli/                  the `meshnet` terminal client: clap args + a line REPL + tables
  meshffi/                  C ABI bridge: JSON in, JSON out (lib.rs 475, dto.rs 460)
mobile/
  lib/services/mesh_service.dart   (861) the app's one connection to the core
  lib/bridge/mesh_ffi.dart         raw dart:ffi bindings + a stub fallback
  lib/bridge/ble_radio.dart        MethodChannel/EventChannel to the native radio
  lib/features/                    chat · map (radar + OSM) · emergency · networks
  android/.../BleMesh.kt   (554)   Android BLE: peripheral + central at once
  android/.../FrameCodec.kt        length-prefixed chunking/reassembly per device
  ios/Runner/BleMesh.swift (513)   the same, in CoreBluetooth
  macos/Runner/BleMeshCentral.swift (256)  macOS: central role only
  test/                            24 Dart tests, 3 files
web/                        static marketing site + a simulated Sydney-flood signal map
docs/                       README (index) · ARCHITECTURE · SETUP · MOBILE · DEMO · JOINING
                            · HANDOVER (this file)
scripts/                    check.sh · build_ffi.sh · ble_gateway.py · autostart/install.sh
plan.md                     the original product/architecture plan
```

### 5. Identity and cryptography

- **Node id** = first 8 bytes of SHA-256 over a UUID generated on first launch. Deliberately
  *not* a MAC hash — modern OSes randomise MACs and hide them from userspace.
- **Ed25519** signs every packet. Receivers verify before acting, using the key learned from
  that node's `Hello`. Relays forward without verifying, since a relay may not know the key yet.
- **X25519 sealed box** (ephemeral key → shared secret → HKDF-SHA256 → ChaCha20-Poly1305)
  delivers a network's symmetric key to exactly one invited member.
- **ChaCha20-Poly1305** encrypts all network traffic inside an `Envelope`.
- `identity.json`, `contacts.json`, `networks.json`, `zones.json` and `messages/<net>.jsonl`
  live under `~/.meshnet` (override with `--home`).

### 6. Two wire formats, and why

**`Frame` / `Packet`** (bincode, up to 8 KB) is the connection-oriented format:

```
Frame  { magic, version, link_from, instance, packet }
Packet { id, origin, dest, sent_at_ms, body, sig,   ← signed
         ttl, path }                                ← rewritten by every relay
```

The signature covers everything *except* `ttl` and `path`, because those are exactly what a
relay must change and nothing else. `Body` is `Hello`, `Ping`/`Pong`, `Envelope` or `Invite`;
inside an `Envelope` sits a `NetPayload` (`Chat`, `Direct`, `Gps`, `Members`, `KickVote`, `Ack`,
`Status`, `Sos`, `Zone`) that only network members can decrypt. A relay carries someone's SOS
without being able to read it.

**Beacon v1** (`beacon.rs`) is the second format: hand-packed, fixed-layout, allocation-free,
sized for the **27 usable bytes** of a BLE manufacturer-specific advertisement.

```
header  (4)  ver/type | flags(SOS,GPS,STATUS,RELAY) | battery | seq
presence(23) node(8) lat_e7(4) lon_e7(4) status(1) hops(1) ttl(1)
zone    (24) origin(8) cell(8) verdict(1) consensus(1) radius_m(2)
```

It is encoded, decoded and byte-exactly round-trip tested — **but nothing ever puts it on a
radio.** That is deferred work (phase 2C.4), and there is a hard security constraint attached:
an advertisement carries no signature, so a beacon may only ever be a *discovery hint* that
triggers a GATT connection. If a beacon were allowed to set SOS state, anyone with a BLE radio
could broadcast a forged SOS attributed to any node id.

### 7. Routing

Path-recorded flooding with learned reverse routes:

1. **Dedupe** — 128-bit packet ids, last 4,096 remembered; repeats dropped. This is what stops
   broadcast storms in a crowded room.
2. **TTL** — default 8, decremented per relay. SOS gets 12.
3. **Route learning** — a packet arriving from neighbour `N` after `h` hops teaches "to reach
   `origin`, send to `N`, cost `h`". Shorter wins; routes expire at 120 s, neighbours at 30 s.
4. **Unicast when known, flood when not.**
5. **Store-and-forward** — direct messages sit in an outbox and re-send under *fresh* packet ids
   (so dedupe doesn't eat the retry) every 15 s for 2 minutes, until an `Ack` returns.

Peers are ranked GPS distance → hops → latency, ghosts last. RSSI is carried and displayed but
is always `None` on UDP; only a radio can fill it in.

### 8. Networks, invites, kick voting

`[default]` is the public lobby — its key is derived from a constant in the source, so it is
functionally public, and it exists so that public traffic goes through the *same* encrypted code
path as private traffic instead of a second one. A private network is a name + a random 32-byte
key + a member list. `--kick` broadcasts a signed ballot; every member tallies independently; at
`votes >= members/2` the **lowest-id remaining member** — a deterministic choice everyone
computes identically, with no elected leader — generates a new key, seals it to each remaining
member, and broadcasts the new roster. Old epochs are kept so in-flight packets still open.

### 9. Emergency signals

- **In-network SOS.** A flag inside the signed `Hello`, plus an immediate `Sos` payload at TTL
  12. It is *deliberately isolated* from the OS emergency-call path so testing can never dial
  real emergency services. Both the CLI and the app say so unmissably. **Do not ever wire this
  to a real emergency service.**
- **Panic codes.** One byte on the wire; the English text lives only in the renderer and never
  travels. (See §16 for drift here.)
- **Ghosting.** A peer with no route and no neighbour entry is not deleted — it is drawn dimmed
  at its last known fix with the age of that fix. A dead battery must not look like never having
  existed.

### 10. Safe / unsafe zones

Raw coordinates would be a broadcast storm with no useful aggregate, so a report snaps to an
**H3 cell** (resolution 8, ~460 m edge) and only `{cell, verdict, radius_m}` travels. The design
decisions here are the most defensible in the codebase and are worth being able to recite:

- **One bit, not a five-point scale.** "Is this a 2 or a 3?" has no answer at 3 a.m. in a flooded
  street; "safe — yes or no?" does. Averaging a scale also turned two 4s and two 0s into
  "moderate", a sentence nobody said.
- **The reporter picks the radius**, because the cell size was never theirs to choose.
- **Both vote counts travel separately, never blended.** "5 say safe" and "5 say safe, 4 say
  unsafe" must not render identically.
- **A tie reads unsafe**, and anything that isn't an explicit `safe` byte decodes as unsafe. A
  corrupt byte must never clear a hazard.
- **One report per node per cell**, latest wins — otherwise one node manufactures a consensus.
- **Re-gossip is bounded at 16 own reports**, one per 5-second tick, round-robin, so a late
  joiner converges. Eviction stops republishing but does not withdraw the vote.

### 11. The duty cycle

`duty.rs` is a pure function from conditions to a cadence — no clock, no radio, no randomness,
which is why it is testable at all.

| Alone for | Hello | Scan |
|---|---|---|
| peers present, or any SOS | 3 s | low latency |
| 0–1 min | 3 s | low latency |
| 1–5 min | 10 s | balanced |
| 5–20 min | 30 s | low power, 5 s window / 30 s |
| > 20 min | 60 s | low power, 5 s window / 60 s |

Below 15 % battery it drops one further rung. Any frame from anyone — *including one it cannot
decrypt* — resets to the top rung, because being slow to notice a rescuer costs more than a few
beacons. Intervals are jittered ±20 % so twenty phones that started together don't collide on
air forever.

### 12. The transport seam

Three methods: `send_broadcast`, `send_to`, `recv`. Everything above is radio-agnostic.

- **`UdpTransport`** — IPv4 multicast `239.42.13.7:47474` for discovery, limited broadcast as a
  fallback, unicast for routed traffic, plus learned addresses and `--peer` seeds. This is what
  ships and what the tests exercise.
- **`ExternalTransport`** — no I/O at all: a pair of queues the platform drains and feeds. This
  is how Kotlin/Swift own the radio while Rust owns the protocol. Bluetooth device ids are
  mapped to synthetic `127.0.0.0/8` addresses so the router's `SocketAddr` keying survives
  untouched.
- **`MultiTransport`** — every radio at once. Succeeds if *any* child accepted the frame; a dead
  radio costs that radio, never the node. Each child gets its own pump task rather than a
  `select!`, because cancelling a `recv` mid-await is how frames go missing.
- **`BleLinuxTransport`** — real BLE on Linux via `bluer`, same GATT UUIDs as the phones.

**Why not BLE on laptops:** `btleplug` and friends implement the BLE *central* role only.
Advertising as a *peripheral* isn't portably exposed on macOS or Windows from userspace, so a
laptop mesh over BLE cannot be assembled from available libraries. UDP proves the identical
routing/crypto/CLI and leaves the radio swappable. This is a good answer to give when someone
asks "why is this Wi-Fi if it's a Bluetooth mesh?"

### 13. The mobile app

The FFI contract is small on purpose — **JSON in, JSON out**:

```
mesh_start(config_json) → whoami | error       mesh_ble_drain() → frames to transmit
mesh_command(cmd_json)  → reply json           mesh_ble_inject(json)
mesh_poll_event(ms)     → event json           mesh_ble_rssi(json)
mesh_status_table()     → the panic codes      mesh_ble_peer_lost(device)
mesh_free(ptr)                                 mesh_stop()
```

`mesh_poll_event` blocks up to `timeout_ms`, so Dart drives it without spinning. `MeshService`
(861 lines) is the only Dart that talks to it: it starts the core, pumps commands, drains
events, republishes to widgets — and contains no protocol logic by design.

Four tabs: **Chat**, **Peers** (compass/radar first, interactive OSM map second — deliberately,
because a phone in a disaster has no tiles and no internet), **Emergency** (slide-to-SOS, panic
buttons sourced from the core, zone reporter, heatmap), **Networks** (create/invite/switch/kick
plus a radio diagnostics panel).

The native radios (`BleMesh.kt`, `BleMesh.swift`) play **peripheral and central simultaneously**,
share UUIDs with the Linux transport, and chunk frames behind a 4-byte length prefix with
per-device reassembly buffers.

Platform constraints you will run into and cannot code around:
- **iOS has neither UDP multicast nor broadcast** without a restricted Apple entitlement this
  build doesn't hold. On iOS, Wi-Fi reaches only addresses it was explicitly given.
- **A backgrounded iPhone is invisible to Android.** iOS moves 128-bit service UUIDs into the
  advertisement's overflow area, which non-Apple centrals cannot read.
- **Some budget Android chipsets have no BLE peripheral role** (`advertising failed with code
  5`). Two such phones can never discover each other.

### 14. The web site

`web/` is a static marketing site plus a full-screen "signal map" — a canvas heatmap over Google
Maps with a simulated Hawkesbury–Nepean flood scenario (415 nodes, 16 SOS signals). It shares no
build tooling with the app. **The data is entirely simulated and the page says so.** It needs a
Google Maps API key in `web/assets/js/config.js` (gitignored; `config.example.js` is the
template) and deploys via Netlify, which injects the key at build time. `web/README.md` is
unusually good — it explains the heatmap lobe-hashing and the zoom re-pegging in real detail.

---

## Part III — The state you are inheriting

### 15. Verified, on this machine, today

| Check | Result |
|---|---|
| `cargo test --workspace` | ✅ **38 passed, 0 failed** |
| `cargo build --release` | ✅ builds |
| `cargo clippy --workspace --all-targets` | ✅ **0 warnings** (was 14) |
| `flutter analyze` | ✅ **No issues found** |
| `flutter test` (24 tests) | ✅ **24 passed, 0 failed** (was 14 passed, 10 failed — fixed, below) |
| `./scripts/check.sh` | ✅ **all checks passed** |
| Two nodes meshing over an actual Bluetooth radio | ⚠️ **never once attempted, by anyone** |

**The Dart suite was red when this document was first written; it has since been fixed.**
Recorded here because the cause is instructive. Every failure was:

```
MissingPluginException(No implementation found for method isSupported on channel reunite/ble)
```

Cause: commit `68df131` ("temp 1", from the macOS/iPhone split branch) widened
`BleRadio.isAvailable` to include macOS:

```dart
// mobile/lib/bridge/ble_radio.dart:23
static bool get isAvailable => Platform.isAndroid || Platform.isIOS || Platform.isMacOS;
```

`flutter test` runs on the host VM with no plugin registrar, so the `reunite/ble` MethodChannel
throws `MissingPluginException`. `BleRadio` catches only `PlatformException`, which
`MissingPluginException` is not, so it escapes to `MeshService.init`'s catch, sets `startError`,
and the app renders its startup-error screen — failing all ten widget/integration tests.

**Fixed** by adding an `on MissingPluginException` clause to each of the eight `BleRadio`
methods, each returning the same answer it already gives for a platform with no radio. That
restores the class's own stated contract — "a missing radio costs that radio, never the node" —
which was being violated on any platform without a registrar. `./scripts/check.sh` is green
again.

> `phase/README.md` and `phase-2e` both claimed the suite was green. They were true when
> written and were broken by a later merge that nobody re-ran the tests after. This is worth
> mentioning in an interview as a concrete example of why you'd want CI on a project like
> this — and adding CI is still on the list below.

### 16. Known gaps, drift and honest limits

Ranked by what I'd fix first. Items 1–4 are the project's real frontier; the security list is
already documented candidly in `docs/ARCHITECTURE.md`, which is to the original author's credit.

1. ~~**The Dart test suite is red.**~~ **Fixed** — see §15.
2. **Bluetooth has never been run on real hardware.** Every claim about BLE behaviour in this
   project was established by *reading code*. `external_transport.rs` proves the layers above the radio are
   transport-agnostic — it is a pair of in-memory queues and touches no radio.
   **Appendix B** is a stop-at-first-failure ladder with a diagnosis table; follow it exactly
   when you get two phones. **This is the single highest-value thing you can do to this
   project.**
3. **The mesh stops when the app leaves the screen.** No Android foreground service, no iOS
   state restoration. This is the largest gap between this and something usable in a real
   emergency.
4. **The "offline" map fetches tiles from `tile.openstreetmap.org`** — blank without internet, in
   an offline-first app. The compass/radar fallback is correct and correctly the default, but the
   map needs a bundled MBTiles pack to be honest.
5. **`DatabaseStore` is a facade.** `store.rs` exports a 60-line `REUNITE_SQL_SCHEMA` constant
   and a `DatabaseStore` struct whose every method forwards to the JSON file functions. No
   SQLite, no `rusqlite` dependency. `db_test.rs` (1 test) tests the forwarding. The deviations
   register (D4) is honest that storage is JSON; the commit message ("finalize DatabaseStore
   persistence layer... 100% real db_test suite") is not. **Do not describe this as a database
   layer anywhere on a profile.**
6. **Status-code drift.** `status.rs` defines eight constants (`SAFE`, `MEDICAL`, `SUPPLIES`,
   `TRAPPED`, `MOVING`, `SHELTER`, `HAZARD`, `NONE`) but commit `6eba821` cut `TABLE` to three
   rows — and mapped code `0x02` (`MEDICAL`) to the name `"sos"` and the text "🚨 SOS Emergency",
   colliding with the actual in-network SOS flag.

   **Partly fixed.** The docs no longer promise seven codes, and `render.rs`'s usage line no
   longer tells you to type `--status medical` — a name the parser has rejected ever since the
   cut, so the tool's own help was instructing you to type something it refuses. **Still open,
   and it is a product decision, not a cleanup:** three displayed codes or seven? Five constants
   (`SUPPLIES`, `TRAPPED`, `MOVING`, `SHELTER`, and the `MEDICAL`/`sos` naming collision) are
   dead weight until someone answers. It was flagged as an open question for the product owner
   — which is now you.
7. ~~**Docs cite a `proposal.md` that isn't in the repo.**~~ **Fixed** — the dead citation is gone; the requirement it cited still stands in the text.
8. **`meshcore` is not `no_std`** despite `plan.md` requiring it. `beacon.rs`, `status.rs` and
   `duty.rs` are written std-free to keep the eventual split cheap; everything else uses `tokio`,
   `socket2` and `std::fs`. Deviation D1; Phase 3 work.
9. **Security limits, as documented and as I confirm them:** traffic analysis is possible
   (origin, dest and network id are in clear so relays can route); keys sit unencrypted at rest
   in `identity.json` / `networks.json`; identities are free, so Sybil vote-stuffing beats the
   kick threshold; old-epoch packets stay decryptable by design; `[default]` is public. Every one
   of these is written down in `docs/ARCHITECTURE.md` under "What this does not protect against".
   **That candour is an asset — keep it, cite it, don't quietly delete it.**
10. **The web site's Maps API key was shared in plaintext at some point** (`web/README.md` says
    so and tells you to rotate it). Confirm it's rotated before any public deploy.
11. ~~**Repo hygiene: 12 stale remote branches.**~~ **Done** — `origin` is now your own fork
    (`DuyNguyen0518/ReUnite`) with a single `main`, and the original is `upstream`. The 13 old
    branches still exist on `upstream`, which is the right place for them. Two commits are still
    named "temp 1" and "Edit something here", but they are merged history and not worth a rewrite.

12. **The Dart suite cannot be run twice at once against one working tree.** Two concurrent
    `flutter test` invocations race on the shared `mobile/.dart_tool` build directory; one dies
    with *"the file was deleted or moved while the tool was running. Try running `flutter
    clean`"*, and — worse — the other can execute a **stale compilation snapshot**, reporting
    results for code that is no longer on disk. That is not hypothetical: it happened during the
    2026-09-09 cleanup and reproduced a fixed suite as still-failing, which cost an hour of
    chasing a bug that had already been repaired.

    Verified by running two suites simultaneously: one exits 1 on the build directory, the other
    passes 24/24. **It is not the mesh ports** — `app_test.dart` binds 47651 and
    `peers_test.dart` 47652, and those are distinct and never exchange datagrams. Nothing needs
    fixing in the tests. What needs fixing is the habit: **run the suite once, sequentially, and
    when you add CI (§19) do not let two jobs share a checkout.** If you ever see a test result
    that contradicts the code in front of you, re-run it alone before believing it.

13. **No `LICENSE` file, and three different answers about the licence.** `README.md` says
    "MIT / Apache 2.0", `Cargo.toml` says `MIT`, and there is no licence file at all — so
    legally the repository is "all rights reserved" regardless of what the README claims. This
    is a decision for you and your teammates, not a cleanup: pick one, add the file, and make
    the README and `Cargo.toml` agree with it. (All three crates now inherit the workspace
    licence field, which `meshcore` previously did not.)

### 17. Invariants — do not "simplify" these away

Each of these was a deliberate decision with a failure mode behind it. A future change that
looks like a cleanup can undo one without noticing. This table is reproduced in full here
because its original home, `phase/phase-2e-hardware-verification.md`, was deleted in `9da2b70`.

| Invariant | Why | Where |
| :--- | :--- | :--- |
| SOS and status live **inside** the Ed25519 signature of `Hello` | So no relay can clear someone's SOS or forge a status on their behalf | `packet.rs`, `node.rs` |
| `ttl` and `path` sit **outside** the signature | Every relay must rewrite them and nothing else | `packet.rs` |
| An unsigned advertisement may never set peer state | Otherwise a forged SOS costs one BLE radio | `beacon.rs` |
| A zone tie resolves to **unsafe** | A contested area is not a safe area | `zones.rs` `Zone::verdict` |
| Anything that is not an explicit `safe` byte decodes as unsafe | A corrupt byte must never clear a hazard | `zones.rs` `Verdict::from_wire` |
| Both zone vote counts travel separately, never blended | "5 say safe" ≠ "5 say safe, 4 say unsafe" | `zones.rs`, `ZoneDto` |
| No automatic, unattended safety verdict anywhere | An earlier build auto-reported "safe" from GPS every 2 minutes, manufacturing false consensus | `mesh_service.dart` `_autoShareLocation` |
| Only a state the platform actually reported may accuse the radio | `unknown` ≠ "Bluetooth is off"; the previous build sent people to check a correct setting | `mesh_service.dart` `bleErrorForRadioState`, tested |
| One dead radio never takes down the node | A phone with Bluetooth off must still mesh over Wi-Fi | `transport/multi.rs` |
| Never back off the duty cycle during an SOS | That is the moment to spend the battery | `duty.rs`, tested |
| Compass/Grid is the **default** map view | A phone in a disaster has no tiles and no internet | `map_screen.dart` tab order, tested |
| The re-gossip ring is bounded at 16, and eviction does not withdraw the report | Other nodes are still counting that vote | `zones.rs` `record_own` |

### 18. Commands you'll use

```bash
# everything the project can check, in one command
./scripts/check.sh              # or: check.sh rust | check.sh dart

# terminal node
cargo build --release
./target/release/meshnet --name laptop-a
# then, at the [default] > prompt:  --peers · --routes · --sos start · --heatmap show · --help

# three nodes on one laptop (fake radio range)
./target/release/meshnet --home /tmp/a --name alice --port 47001 --no-multicast --no-broadcast --peer 127.0.0.1:47002
# ...see docs/SETUP.md §8 and docs/DEMO.md for the full A—B—C relay demo

# build the native core for a target
./scripts/build_ffi.sh macos | android | ios

# run the app
cd mobile && flutter run -d macos        # or -d <phone>
```

iOS needs two manual Xcode steps that cannot be scripted: link the `.a`, then set **Dead Code
Stripping = No** and add `-all_load`. Nothing in the Swift references the C entry points — Dart
finds them with `dlsym` — so without that the linker strips them and the app shows the
startup-error screen. `scripts/build_ffi.sh` prints this reminder; `docs/MOBILE.md` explains it.

### 19. Your first week

| Day | Do this |
|---|---|
| 1 | Read `docs/ARCHITECTURE.md`, then §16–§17 and Appendix B of this document. Run `./scripts/check.sh`. |
| 1 | ~~Fix the `MissingPluginException`~~ — already done; the suite is green. Verify with `./scripts/check.sh`. |
| 2 | Run `docs/DEMO.md` end to end on one laptop — three nodes, multi-hop relay, a private network the relay can't read, kick voting, ghosting, zones. This is the fastest way to *feel* the protocol. |
| 3 | Add CI (GitHub Actions: `cargo test`, `cargo build --release`, `cargo clippy -- -D warnings`, `flutter analyze --fatal-warnings`, `flutter test`). The green-suite claim went stale in a merge; CI is why that stops happening. **Give the Dart job its own checkout and never run two `flutter test` jobs against one working tree** — see §16.12. |
| 4–5 | **Two phones.** Walk the ladder in Appendix B rung by rung. Write down what you actually observe, whatever it is — a measured failure is worth more than an untested assumption. |
| Then | Background execution (Android foreground service + iOS state restoration). It's the difference between a demo and a usable tool. |

Then, and only then, Beacon v1 on the air — and read the security constraint in phase 2C.4
before writing a line of it.

---

## Part IV — Putting it on your profile

### 20. What to call it

Keep **ReUnite** as the product name — it's short, pronounceable, and says what it's for. What
matters is the *tagline*, because that's what gets read. Lead with the engineering, not the
mission; the mission is the hook, not the claim.

> **ReUnite — an offline peer-to-peer mesh network for disasters**
> *Rust protocol core · Flutter app · native BLE on Android, iOS and Linux*

Good variants depending on the surface:

| Surface | Line |
|---|---|
| GitHub repo description | `Offline P2P emergency mesh: Rust protocol core, C-FFI bridge, Flutter app, native BLE on Android/iOS/Linux` |
| CV bullet header | `ReUnite — offline mesh networking stack (Rust · Flutter · Kotlin/Swift)` |
| LinkedIn / portfolio card | `A phone-to-phone mesh network that works with no cell tower, no Wi-Fi and no internet` |

Avoid: "disaster relief app", "emergency SOS app", "life-saving communication platform". They
read as a hackathon pitch deck and they undersell the part that's actually hard — which is the
protocol, not the product idea.

**Already done:** `origin` is `DuyNguyen0518/ReUnite` and the original is `upstream`, so the
repo you link is under your own name. Keep the upstream link and the team credit in the README.

### 21. What to include

**Fix these before you link anyone:**

1. Green tests (§15) — a red suite on a repo whose README advertises "full automated tests" is
   the worst possible first impression.
2. A CI badge. It's the cheapest credibility signal on GitHub.
3. **A 30–60 second demo GIF at the top of the README.** This project's biggest problem as a
   portfolio piece is that nobody can run it — it needs two devices and a native toolchain.
   A terminal recording of three nodes relaying a message with the middle node dropping out is
   *reproducible on one laptop* and shows the protocol working. Record it with `asciinema` or a
   screen capture from `docs/DEMO.md`. If you get two phones, a video of a real SOS crossing
   Bluetooth in airplane mode is worth more than everything else on this list combined.
4. A **status section** in the README, above the fold: what's built and tested, what's built and
   unverified, what's not built. You already have this material — it's §15 and §16 of this
   document. Moving an honest status table to the front converts your biggest weakness (nothing
   has run on a radio) into your strongest signal (you know exactly what you have and haven't
   proven). A first version is already in the README's Testing section.
5. An **architecture diagram**. One image: UI shells → core → transport trait → three radios.
   `docs/ARCHITECTURE.md` has the ASCII version; render it properly.
6. Trim the README's install guide. The current one is a step-by-step for a non-technical user
   installing on a phone, which is the wrong audience for a GitHub landing page. Move it to
   `docs/` and give the README: what it is, the demo GIF, the architecture, the status table,
   the design decisions, then a short quickstart.

**Keep, prominently:**

- `docs/ARCHITECTURE.md` and its **"What this does not protect against"** section. Publishing
  your own threat model is rare in junior portfolios and reads as senior.
- **Appendix B** of this document — a verification ladder with a diagnosis table, written for a
  stranger arriving cold. A handover artefact like this is a genuinely unusual thing for a
  candidate to have written, and it is evidence of exactly the discipline the project's
  weakest area needs.
- The **deviations register** — nine documented cases of "the plan said X, reality is Y, here's
  why and who owns it" — was in the deleted `phase/README.md`. It is a strong
  project-management artefact and worth reviving in some form; recover it with
  `git show ac2d337:phase/README.md`.

**Be explicit about credit.** One line in the README: *"Team project. I designed and built the
Rust protocol core, the FFI bridge, the Flutter–core integration and the native BLE transport;
[teammates] built the chat and map UI and the marketing site."* Then link your six commits. This
costs you nothing and protects you from the one question that can sink an interview.

### 22. How recruiters will read it

**Non-technical recruiters and HR screens** will mostly see: *disaster tech, Rust, Flutter,
Bluetooth, team project, three days.* The mission is memorable and they will remember the repo.
They will not evaluate the code. What helps here is the tagline, the demo GIF, and a clean README.

**Hiring managers and engineers** — the people who decide — will react roughly like this:

*What genuinely impresses:*

- **Choosing Rust for a protocol core and actually keeping the boundary clean.** `meshcore` has
  no UI code; two totally different front ends drive it through one `Command`/`Event` API. That
  is a real architectural instinct, not a tutorial pattern.
- **Cross-language systems work.** Rust ↔ C ABI ↔ Dart FFI ↔ Kotlin/Swift MethodChannels, with
  the radio deliberately living outside Rust because mobile OSes won't share the Bluetooth stack.
  Very few junior portfolios have a working FFI boundary at all.
- **Applied cryptography with correct instincts.** Ed25519 signatures with `ttl`/`path` outside
  the signed view; X25519 sealed boxes for key delivery; epochs on re-key; and — the tell — a
  written list of what the scheme *doesn't* protect against.
- **Design decisions with reasons attached.** The 27-byte beacon budget derived from the BLE AD
  structure. The five-point safety scale replaced by one bit because nobody can answer a scale
  under stress. A zone tie resolving to unsafe. The duty-cycle ladder resetting on any frame,
  even an undecryptable one, because missing a rescuer costs more than beacons. These are the
  answers that make an interview go well, and they are all already written down in your own
  commits.
- **Tests that assert behaviour, not coverage.** `an_sos_never_backs_off_however_alone_it_is`,
  `a_contested_cell_reads_unsafe_rather_than_splitting_the_difference`,
  `emergency_payloads_are_unreadable_outside_the_network`. Named like specifications.
- **Knowing what you haven't proven.** The line "swiftc -typecheck passing means the Swift is
  valid, not that it works" is the single most senior sentence in the repository.

*What they will probe, and what you should have ready:*

| Their question | Your answer |
|---|---|
| "Has this ever run on real hardware?" | "The Rust core and the UI are tested end to end over an in-memory transport, 38 + 24 tests. Bluetooth between two phones has never been run — I wrote the verification ladder for it and it's the next thing I'd do. Everything I claim about BLE behaviour is inspection, and I've labelled it as such in the repo." **Never bluff this one.** The honest answer is stronger than the bluff, and the repo already tells the truth in writing. |
| "Three days and 14k lines — how much of this is yours, and how much is generated?" | Expect this. Answer directly, name the six commits, and be ready to explain any file on screen. The best defence is being able to derive the 27-byte beacon budget or the kick threshold live. If AI tooling was involved, say so plainly — that's now unremarkable; being unable to explain your own code is what isn't. |
| "Why is this Wi-Fi if it's a Bluetooth mesh?" | §12. The `btleplug` peripheral-role answer is a good one and shows you hit a real platform limit and routed around it without abandoning the abstraction. |
| "What would you do differently?" | Have three ready. Mine: hardware-in-the-loop from day one instead of at the end; CI from the first commit (the test suite went red in a merge and nobody noticed); and either build SQLite properly or don't ship a `DatabaseStore` facade that implies it. |

*What will count against it, and what to do:*

- **Nothing has been verified on a radio.** Unavoidable — but you convert it from a weakness to a
  strength by leading with the status table rather than letting them discover it. Better still,
  borrow two phones and close phase 2E. It's the highest-leverage day of work available to you.
- **Three-day hackathon timeline.** Some reviewers discount hackathon repos on sight. Your
  counter-evidence is the planning trail: gated phases with a deviations register, not a weekend
  sprint. That trail now lives only in git history (`git show ac2d337:phase/README.md`), which
  is worth remembering before you rely on it in an interview.
- **The `DatabaseStore` / SQL schema commit and the "seven panic codes" doc drift.** Small, but
  a careful reviewer *will* find at least one of them and it costs more than fixing them would.
  Fix them.
- **The mission framing can read as naïve** if the README leads with "life-saving". Lead with the
  systems work; let the mission be the reason it exists.

**Where this lands you:** this is well above the median junior/new-grad portfolio project.
Backend/systems, embedded/IoT, and mobile-platform teams will find something they recognise as
real work — particularly anyone doing Rust, FFI boundaries, or BLE. It is not a
web-CRUD-shop portfolio piece, and that's fine; it's aimed at better roles than those.

---

## Appendix A — quick reference

| Thing | Where |
|---|---|
| GATT service UUID | `a1b2c3d4-e5f6-7890-1234-56789abcdef0` (RX `…f1`, TX `…f2`) — must match in `BleMesh.kt`, `BleMesh.swift`, `ble_linux.rs`, `ble_gateway.py` |
| UDP discovery | multicast `239.42.13.7`, port `47474` |
| Protocol version | `Frame` VERSION = 4; Beacon VERSION = 1 |
| Default TTL / SOS TTL | 8 / 12 |
| Dedupe cache | 4,096 packet ids |
| Neighbour / route timeout | 30 s / 120 s |
| Zone resolution / TTL / ring | H3 res 8 / 6 hours / 16 own reports |
| State directory | `~/.meshnet` (`--home` to override); macOS app library at `~/.reunite/lib` |
| Run one command to check everything | `./scripts/check.sh` |

---

## Appendix B — the hardware verification ladder

*Preserved from `phase/phase-2e-hardware-verification.md`, deleted in `9da2b70`. This is the
only part of the phase plan that describes work still to be done, and it is the highest-value
day of work available to the project. Nothing in it has been carried out.*

**Before changing any BLE code, understand this:** nothing in the test suite touches a radio.
`crates/meshcore/tests/external_transport.rs` exercises the BLE path through
`ExternalTransport`, which is a pair of in-memory queues with a `pump()` shuttling frames
between them. That proves everything *above* the radio is transport-agnostic. It says nothing
about whether CoreBluetooth and Android's BLE stack interoperate. `swiftc -typecheck` passing
means the Swift is valid, not that it works.

### Build and install

```bash
./scripts/build_ffi.sh android      # writes .so into mobile/android/app/src/main/jniLibs
cd mobile && flutter run -d <android-device>
```

`jniLibs` is gitignored and starts absent. A plain `flutter build apk` without this step
produces an app that compiles and then reports *"the mesh core did not start"*. For iOS, see
§18 — the two manual Xcode steps are not optional.

### The ladder

Both phones: **airplane mode on, Bluetooth on**, app open and **on screen**, within a few
metres. Run in order and **stop at the first rung that does not happen**. Each rung rules out
everything above it.

1. **Radio panel.** Networks tab on both phones reads `Bluetooth state: on`, `Connected peers: 0`.
2. **Android advertises.** `adb logcat -s ReUniteBle` shows `advertising as a mesh node`.
3. **Android scans.** Same log shows `scanning for mesh peers`, and no `scan failed with code N`.
4. **Each phone sees the other's advertisement at all.** Install **nRF Connect** on both and
   filter on `a1b2c3d4-e5f6-7890-1234-56789abcdef0`.
5. **GATT connects and the service resolves.** Log shows `connecting to <id>`, then
   `<id> is a mesh peer`.
6. **A frame crosses.** The peer appears in the Peers list on at least one phone.
7. **Both directions.** The roles are symmetric by design; verify it, do not assume it.
8. **A frame larger than one MTU crosses intact** — send a message over 500 bytes, forcing
   chunking and reassembly.

**Rung 4 matters most.** It is the only test that separates *"the radios cannot see each other"*
from *"our code cannot see them"*, and those need completely different fixes.

### Diagnosis

| Stops at | Most likely cause | Where to look |
| :--- | :--- | :--- |
| 1, says `unknown` for over a second | The platform never pushed a state | `AppDelegate.swift` (`case "state"`), `BleMesh.swift` `onState`, `mesh_service.dart` `case 'radio_state'` |
| 1, says `off` or `unauthorized` | Genuine — the OS said so; not the app guessing | Phone settings |
| 1, says `unsupported` on a phone with Bluetooth | `isSupported` is wrong on this device | `MainActivity.kt` / `AppDelegate.swift` |
| 2, `advertising failed with code N` | 1 = data too large, 2 = too many advertisers, 4 = internal, **5 = no peripheral role on this chipset** | `BleMesh.kt` `startAdvertising`. Code 5 is a hardware limit: that phone can only ever be a central, and two such phones can never find each other. |
| 3, `scan failed with code 2` | App registration failed — on Android 12+ almost always the *Nearby devices* runtime permission | `AndroidManifest.xml` declares `BLUETOOTH_SCAN` with `neverForLocation`; check it was granted at runtime |
| 4, nRF Connect sees **neither** | Neither is advertising. Advertising is broken, not discovery. | `BleMesh.kt` `startAdvertising`, `BleMesh.swift` `peripheralManagerDidUpdateState` |
| 4, nRF Connect sees **both**, app sees neither | Radios are fine; **our scan or filter is wrong** | `BleMesh.kt` `startScanning` (`ScanFilter`), `BleMesh.swift` `scanForPeripherals(withServices:)` |
| 4, sees Android but not iOS | Classic iOS symptom. Confirm the app is foregrounded. | `BleMesh.swift` `peripheralManagerDidUpdateState` |
| 5, `connecting to <id>` repeats forever | Connection never completes; check the reconnect throttle is in effect | `BleMesh.swift` `connecting` map; `BleMesh.kt` `connectTo` |
| 5, `has no mesh service; disconnecting` | Service discovery found the device but not our GATT service | UUIDs must match across `BleMesh.kt`, `BleMesh.swift`, `transport/ble_linux.rs` |
| 6, connected but no peer appears | Frames cross the radio but do not reach the core | `mesh_service.dart` `case 'frame'`, then `mesh_ble_inject` in `crates/meshffi/src/lib.rs` |
| 8, short messages work, long ones do not | Chunking or reassembly; the iOS write flow-control path is the suspect | `BleMesh.swift` `pumpWrites` / `pumpNotifications`, `FrameCodec.kt` reassembler |

> **A backgrounded iPhone is invisible to Android by design.** iOS moves 128-bit service UUIDs
> into the advertisement's *overflow area*, which non-Apple centrals cannot read. Any "it works
> until I lock the screen" result traces to this. It is an Apple platform constraint, not a bug,
> and the fix is background modes plus Beacon v1 in manufacturer data — not a change to discovery.

### Then verify the multi-radio and duty-cycle behaviour

- **Both radios at once.** Wi-Fi *and* Bluetooth on, both phones on one hotspot: the peer appears
  **once**, not twice. Dedupe is by `NodeId` in the router.
- **One radio down.** Turn Bluetooth off on one phone: it must keep meshing over Wi-Fi and say
  why Bluetooth is gone. It must **not** show the startup-error screen. Then repeat with Wi-Fi.
- **The duty cycle eases off.** Leave one phone alone 25 minutes; `adb logcat` should show the
  cadence drop through `balanced` to `low_power`. Bring the other into range: back to the fast
  rate within one interval.
- **An SOS never backs off.** Raise an SOS on the lone phone, wait past 5 minutes, confirm the
  cadence stays at 3 s.
- **Measure the battery.** `plan.md` targets **< 5 %/hour idle**. Publish the measured figure
  even if it misses — the target does not move to meet the measurement. Expect Android and iOS
  to differ substantially: CoreBluetooth has no scan-mode knob, so only the *window* applies.

### Rules for this work

- **Do not fix by guessing.** Every rung has a log line. If a change is made without a failing
  rung pointing at it, it is a guess — and guesses in this layer are how the original iOS bug
  survived from the first commit until it was found by inspection.
- **Some defects only a running radio will reveal.** Five were found by reading code. Five is
  what inspection found; it is not necessarily what exists.
- **Record what you observe**, so the next person does not have to run it again to find out.
