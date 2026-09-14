# SRT Relativity — Architecture & Roadmap

## Context

`/Users/phoenix/code/srtrelativity` is an empty git repo. We are designing, from scratch, an
open-source, sync-first media relay: it accepts N independent live streams over many protocols,
actively aligns them to a common time base, and emits them over many protocols — Docker-first,
with a web GUI for configuration and live sync telemetry.

The problem it solves: SRT removes jitter *within* a link (TSBPD) but has no notion of
cross-link alignment. Twelve remote cameras arriving over twelve connections land at the switcher
with arbitrary, slowly-drifting relative offsets. Operators fix this by hand and re-fix it every
hour. **Nothing in the open-source media-server world does cross-stream sync. That is the whole
differentiator, and every other decision is subordinate to it.**

### Confirmed requirements

| | |
|---|---|
| **Encoders** | Mixed fleet — Sony, PTZOptics, Marshall, Larix Broadcaster (phones), others. Nothing can be required of the encoder. |
| **Accuracy / scale** | **±50 ms across 12+ streams.** |
| **Receivers** | Any receiver; **OBS and vMix primary**. |
| **Ingest (all in v1)** | **SRT, RTMP, RTSP, RTP, NDI, OMT**, plus audio-only **HTTP audio** (Icecast/Shoutcast/AzuraCast MP3/AAC) |
| **Egress (v1)** | **SRT, RTMP, RTSP, HLS, NDI, OMT** — WebRTC deferred to v2 |
| **Sync-critical egress** | **SRT, NDI, OMT only.** RTMP, RTSP and HLS are best-effort. |
| **Priority order** | SRT primary; RTMP and RTSP close seconds. |
| **Team** | **One engineer.** Timelines below reflect that. |
| **Name / home** | **SRT Relativity** — `github.com/RelentNet/srtrelativity`, public, Apache-2.0, © RelentNet |
| **Deployments** | Venue relays and a main relay, same software. Main tested to **~25 independent cameras**. |
| **First release** | **v0.1 ships with the full GUI** (~5–6 months). No GUI-less alpha. |
| **Build posture** | Own the egress stack — not a wrapper around another media server. |
| **NDI** | Investigate NDI\|HX as the low-CPU path. |
| **Licence** | Permissive (Apache-2.0). |
| **Governing constraint** | **"Sync is still the priority."** |

---

## Read this first: three things that change the plan

### 1. Protocol conversion means this is no longer a byte relay

A passthrough relay can align streams by simply delaying bytes on egress. **It cannot convert
protocols.** The moment SRT→HLS or SRT→OMT is in scope, the core must demux to elementary
streams and re-package — and for NDI/OMT, decode and re-encode.

So the architecture becomes a real media pipeline: **ingest adapters → normalised timed access
units → sync engine → egress adapters**, N + M components rather than N×M conversions. This is
a bigger core than a relay, but it is the only shape that supports the matrix you asked for, and
it actually makes sync *cleaner* — we align in the access-unit domain against a wall-clock
deadline rather than shuffling bytes.

An SRT→SRT fast path that skips re-muxing is still worth keeping, since it is the primary
use case and the lowest-latency, lowest-risk path.

### 2. OMT is the answer to the NDI licensing problem — and you already asked for it

The NDI SDK's licence is incompatible with shipping inside an Apache-2.0 binary; **NDI\|HX is
governed by the Advanced SDK, which is more restrictive still, not less.** So the NDI\|HX
investigation you asked for is worth doing, but expect the licensing answer to be "still a
separate, user-supplied sidecar."

**OMT (Open Media Transport) is permissively licensed and natively supported by vMix** — one of
your two primary targets. That means OMT can link into the core binary where NDI cannot. It is
the strategically correct low-latency LAN transport for this project.

One expectation to set honestly: **OMT is not cheaper than NDI in CPU.** It carries its own
codec, so it needs decode + encode exactly like full NDI. Its advantages are licensing, native
vMix support, and openness — not CPU. The only genuinely near-passthrough LAN option is NDI\|HX
(H.264/HEVC in an NDI wrapper), and that is precisely the one with the worst licence.

**Therefore Phase 0 settles the CPU-offload premise with measurement, not argument:** put 12
streams into vMix and OBS as SRT, as NDI, and as OMT, and measure client CPU each way. If
hardware-decoded SRT is already cheap on the client, the offload rationale weakens and NDI/OMT
become compatibility features rather than performance features. Cheap experiment, decisive answer.

### 3. "Own the egress stack" should mean owning the server, not re-writing RTP and ICE

Building our own RTSP/RTMP/HLS servers is right — it keeps the sync engine in control of
scheduling, which a sidecar could never give us. But writing an RTP stack from scratch would
consume the project, especially solo.

> **We build our own servers, on top of mature MIT-licensed Go libraries** — `gortsplib`
> (RTSP/RTP), `mediacommon` (TS/codec/SEI), `go-rtmp`. We own the architecture, the scheduling,
> and the code paths. We do not own the parts where a bug means a security hole.

That is genuinely "our own stack" — MediaMTX is built on the same libraries.

### And one thing about timeline — this is a solo build

Six ingest protocols, six egress protocols, a sync engine, a GUI and production hardening is
realistically **12–16 months solo**. That number is not a reason to cut more scope; it is a
reason to change what "done" means at each step.

> **For a solo build the phases are not milestones toward a distant v1.0 — each one is a
> release.** Phase 1 ships as **v0.1**, a genuinely useful product that does the thing nothing
> else does, at roughly month 5–6. Everything after it is additive.

That is the difference between a project that gets used and one that disappears for a year and
a half. It also means every phase boundary must leave the product in a shippable state — no
half-built protocol left behind a feature flag.

---

## The core technical truth (this determines the sync design)

**SRT carries no absolute time.** The SRT header timestamp is microseconds since socket start on
the *sender's* free-running clock. No wall-clock field, no RTCP-SR equivalent, no NTP in the
handshake.

So "synchronize using NTP" cannot mean "read NTP out of the stream." NTP disciplines *our* clock
so measurements and delay lines are stable; each sender's time relationship is recovered
separately. That yields a hierarchy — **and which tier a stream lands on depends on the protocol
it arrives over, which is why multi-protocol ingest is a sync feature, not just convenience:**

| Tier | Source | Accuracy | Available from |
|---|---|---|---|
| **A1** | **RTCP Sender Reports** (RTSP/RTP ingest) — SR maps RTP timestamp → 64-bit NTP wall clock. The standard RTP inter-stream sync mechanism. | **±5–50 ms, automatic** | Any NTP-synced RTSP camera: PTZOptics, Marshall, Sony |
| **A2** | Per-frame timecode in NDI (100 ns units) / OMT equivalent | ±1 frame if the source clock is disciplined | NDI/OMT-native sources |
| **A3** | Timecode in the elementary stream: H.264 `pic_timing` SEI clock timestamps, HEVC `time_code` SEI (payload 136), vendor `user_data_unregistered` SEI | ±0–1 frame | Model-dependent — **audit per camera, never assume** |
| **A4** | **Relay anchors** — wall-clock anchors embedded by an upstream relay (venue → main) | Venue alignment preserved exactly; a few ms against other sources | Streams pushed from another relay |
| **B** | Arrival timestamping at the relay, corrected for RTT/2 and configured TSBPD latency, PCR-tracked for drift | ±10–40 ms of *transmission* time | Always. SRT, RTMP. |
| **C** | Operator trim — a persisted per-stream offset dialled against a clapper | As good as the operator | A human, once per rig |

You noted RTSP quality is often worse than SRT or NDI on this gear, and that SRT is primary.
Both things are true at once, and they do not conflict: **RTSP's value here is as a *timing*
source, not necessarily a media source.** A camera can send high-quality media over SRT while its
RTSP/RTCP stream is used only to establish that camera's wall-clock anchor.

That is worth building deliberately:

> **"Timing companion" mode:** a stream's media comes from SRT; its anchor is measured from the
> same camera's RTSP/RTCP-SR endpoint. Tier A1 accuracy with Tier-SRT picture quality.

That is a genuinely novel capability, it costs little once RTSP ingest exists, and it targets
your exact fleet. It is flagged as such in the roadmap.

**The hard truth to state plainly in the docs:** Tier B aligns when packets were *sent*, not when
light hit the sensor. Two encoder models can differ by 200–600 ms of internal capture-to-transmit
latency, and no network measurement recovers that. Tier A is the automatic route; **Tier C trim
is a first-class MVP feature, not a fallback**; Tier B is what makes a one-time trim survive an
eight-hour show.

### What "aligning" manipulates

Essentially every receiver — OBS (ffmpeg-backed Media Source), vMix, VLC, ffplay, hardware
decoders — treats each input as an independent source with its own decoder and jitter buffer.
**None lock presentation across separate sources by PTS.** What moves alignment is controlling
*when data leaves our egress*, per path.

> **The sync engine computes a global wall-clock presentation deadline per access unit. Each
> egress path subtracts its own measured pipeline latency. Scheduling is per (stream × path).**

Path latencies differ enormously — SRT passthrough ≈ 0 added, NDI/OMT 1–3 frames of codec
latency, HLS seconds — so this generalisation is mandatory, and it must exist from the first line
of the sync engine or Phase 4 becomes a rewrite. It applies even to best-effort paths, which
still need a *measured* latency figure to display, just not a defended one.

**Audio:** the per-stream offset applies identically to every track in a stream, so intra-stream
A/V sync is preserved by construction. This is a one-line invariant that must be tested, not
assumed.

### Drift

Encoder drift (±20–50 ppm each, up to ~100 ppm relative ≈ **0.36 s/hour**) shifts relative
offsets. Corrected by **slewing** the delay (rate-limited, e.g. ≤1 ms/s); step only on (re)join.
On passthrough paths this is free — one AU in, one AU out, time-shifted. On decoded paths
(NDI/OMT) it is not: those need occasional frame drop/repeat and audio resampling.

---

### Sync groups and independent streams

Aligning every stream on the box to one master would make every camera wait for the slowest one,
including cameras in unrelated shows. So alignment is scoped:

- **Sync group:** a set of streams aligned to each other, with its own master and its own
  alignment cost. Two shows on one relay are two groups and never affect each other.
- **Independent stream:** belongs to no group. Pure relay: **zero added alignment delay**, no
  master, no offset, no tier badge. A single camera pushing to YouTube is independent.
- **New streams default to independent.** Adding a stream to a group is a deliberate act. This
  matches the self-serve case (nationcam cameras are single-camera restreams) and means nobody
  pays alignment delay they didn't ask for.
- The scheduler still handles independent streams — they just get a deadline of "now + path
  latency" instead of a group deadline — so there is one code path, not two.
- Moving a stream between groups, or out of one, re-anchors that stream only.

---

## Architecture

```
┌────────────────────────── srtrelativity (Go) ─────────────────────────────────────────┐
│                                                                                        │
│  INGEST ADAPTERS  (the measurement point — every adapter emits an anchor estimate)    │
│  ┌───────┐┌───────┐┌───────┐┌───────┐┌───────┐┌───────┐                              │
│  │  SRT  ││ RTMP  ││ RTSP  ││  RTP  ││  NDI  ││  OMT  │   + "timing companion":       │
│  │  (B)  ││  (B)  ││ (A1!) ││ (A1!) ││ (A2)  ││ (A2)  │     media from X, anchor      │
│  └───┬───┘└───┬───┘└───┬───┘└───┬───┘└───┬───┘└───┬───┘     from the same camera's    │
│      └───────┴────────┴────────┴────────┴───────┘          RTSP/RTCP endpoint        │
│                              │                                                         │
│                    normalised timed ACCESS UNITS                                       │
│              {track, dts, pts, anchor(wall-clock), keyframe, bitstream}                │
│                              ▼                                                         │
│              ┌───────────────────────────────────────────┐                            │
│              │              SYNC ENGINE                   │◀── NTP-disciplined         │
│              │  anchor estimation per tier (A1/A2/A3/B/C) │    monotonic clock         │
│              │  master selection · drift fit · slew ctrl  │                            │
│              │  → GLOBAL PRESENTATION DEADLINE per AU     │                            │
│              └───────────────────┬───────────────────────┘                            │
│                                  ▼                                                     │
│         ┌────────────────────────────────────────────────────────┐                    │
│         │  SCHEDULER — per (stream × path): release at            │                    │
│         │  deadline − path.Latency()   · buffer caps · drop policy│                    │
│         └───┬────────────────────────┬───────────────────┬───────┘                    │
│             ▼                        ▼                   ▼                             │
│   ┌──────────────────┐   ┌──────────────────────┐  ┌──────────────────┐              │
│   │ PASSTHROUGH      │   │ REMUX (no decode)    │  │ DECODE + ENCODE  │              │
│   │ SRT→SRT fast path│   │ SRT · RTMP · RTSP ·  │  │ NDI · OMT        │              │
│   │ ~0 added latency │   │ HLS                  │  │ heavy · see tiers│              │
│   │ ✅ SYNC GUARANTEE│   │ codec-compat checked │  │ ✅ SYNC GUARANTEE│              │
│   │                  │   │ best-effort (no sync)│  │                  │              │
│   └──────────────────┘   └──────────────────────┘  └──────────────────┘              │
│                                                     NDI = optional sidecar (licence)  │
│                                                     OMT = in-core (permissive)        │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                 │
│  │ REST + WS API│ │ embedded UI  │ │ /metrics     │ │ config.yaml  │                 │
│  └──────────────┘ └──────────────┘ └──────────────┘ └──────────────┘                 │
└────────────────────────────────────────────────────────────────────────────────────────┘
```

### The two interfaces the whole product hangs on

```go
type AccessUnit struct {
    Track    *Track         // video|audio, codec, init params
    PTS, DTS time.Duration  // source timebase, normalised to ns
    Anchor   time.Time      // estimated wall-clock capture/send time; zero if unknown
    Tier     SyncTier       // A1|A2|A3|B|C — provenance of Anchor, shown in the UI
    Keyframe bool
    Data     []byte         // codec-native compressed bitstream
}

type Ingest interface { Tracks() []*Track; Read() (*AccessUnit, error); Stats() Stats }
type Egress interface { Start([]*Track) error; Write(*AccessUnit) error; Latency() time.Duration }
```

Everything else — six ingest protocols, seven egress protocols — is an implementation of one of
these two. Getting them right is the single highest-leverage design decision in the project.

### The sync contract — which egress paths carry a guarantee

Sync only matters when the consumer **combines multiple streams**. A switcher cutting between
cameras needs alignment. A viewer watching one HLS feed never sees two streams at once, so there
is nothing to align. Scoping the guarantee to where it is meaningful is free, and it removes the
expensive half of every path that doesn't need it.

| Egress | Contract | Rationale |
|---|---|---|
| **SRT** | **±50 ms guaranteed** | The primary path into vMix/OBS. This is the product. |
| **NDI** | **±50 ms guaranteed** | Same role, LAN transport. |
| **OMT** | **±50 ms guaranteed** | Same role, LAN transport, permissive licence. |
| RTSP | Best-effort | Consumers are usually single-stream recorders or monitors. |
| RTMP | Best-effort | Feeds a CDN — a single program feed a viewer watches alone. |
| HLS | Best-effort | Inherently 2–6 s and segment-aligned; cross-stream sync is meaningless here. |

**What the guarantee costs, and what best-effort saves.** Writing an HLS segmenter on top of
`mediacommon` is routine. Making it hold a deterministic latency budget under load — measuring
per-path latency, compensating for it, bounding its variance, and re-proving all of that in the
test rig every release — is where the months go. Best-effort paths are ordinary remuxes: they get
correctness tests, but they leave the sync rig entirely and never need a latency budget defended.

**Two consequences that shape v1:**

1. **Until Phase 4, SRT is the only sync-critical egress.** The test rig gates exactly one path
   for most of the project's life. That is a large simplification for a solo build.
2. **The UI must label this honestly.** A best-effort path shows a measured latency figure and an
   explicit *no sync guarantee* badge. The failure mode we are avoiding is an operator assuming
   the HLS feed is aligned because everything else is.

**RTSP ingest is unaffected by this and stays high-priority.** RTSP *egress* is best-effort;
RTSP *ingest* is a Tier A1 sync source and one of the most valuable items in the plan. Do not
conflate the two — they are different work with different value.

### Codec compatibility is a first-class concern

Not every codec survives every egress. This must be a real capability table enforced at config
validation and surfaced in the UI, not discovered at showtime:

| Egress | H.264 | HEVC | AV1 | AAC | Opus | Notes |
|---|---|---|---|---|---|---|
| SRT (TS) | ✅ | ✅ | ✅ | ✅ | ⚠️ | The permissive one |
| RTSP (RTP) | ✅ | ✅ | ✅ | ✅ | ✅ | |
| RTMP | ✅ | ⚠️ enhanced-RTMP only | ❌ | ✅ | ❌ | Legacy constraints bite here |
| HLS | ✅ | ✅ | ⚠️ | ✅ | ⚠️ | fMP4 needed for HEVC |
| NDI / OMT | decode | decode | decode | decode | decode | Always full decode + encode |
| *(v2)* WebRTC | ✅ | ⚠️ browser-dependent | ⚠️ | ❌ | ✅ | Would force an AAC→Opus transcode |

**Deferring WebRTC keeps v1 transcode-free.** WebRTC cannot carry AAC, so it would have required
an audio transcode pipeline — the only transcoding anywhere in the product, and a whole subsystem
rather than just another protocol. With it in v2, **v1 does no codec work at all except the
unavoidable decode+encode on the NDI/OMT paths.** That is worth more than the six to eight weeks
it saves.

### Tech stack

Go, decisively — the MIT-licensed Go media ecosystem is precisely the set of parts this needs,
and they interoperate.

| Layer | Choice | Why |
|---|---|---|
| Core | **Go, single binary** (+ optional NDI sidecar) | One deployment unit; goroutine-per-connection fits; passthrough of 12×20 Mbps is trivial. |
| SRT | **`datarhei/gosrt`**, cgo **`libsrt`** as named fallback | Pure Go → static, easy cross-compile. Risk: no socket groups/bonding, encryption coverage unverified. Phase 0 decides; kept behind a ~6-method interface so swapping is a day. |
| TS / codec / SEI | **`bluenviron/mediacommon`** (MIT) | TS reader/writer, H.264/HEVC/AAC parsing, SEI. **Do not hand-roll.** |
| RTSP + RTP + RTCP | **`bluenviron/gortsplib`** (MIT) | Client *and* server. Gives RTCP SR — i.e. gives us Tier A1. |
| RTMP | `yutopp/go-rtmp` (MIT) | Ingest and egress. |
| HLS | `mediacommon` + our own segmenter | Best-effort path. LL-HLS later if wanted. |
| OMT | **`libomt` via cgo, in-core** | Permissive licence allows linking. Costs the `FROM scratch` image → distroless/slim. |
| NDI | **Optional sidecar**, user-supplied SDK | Licence incompatible with Apache-2.0. **NDI\|HX is worse, not better.** |
| Audio transcode | **None in v1** | Deferred with WebRTC. See the codec table. |
| Snapshots | One-shot `ffmpeg` binary in the image (LGPL build) | Only runs when the snapshot button is pressed. Adds ~80 MB to the image; doesn't touch the relayed video. |
| Preview player | `hls.js` (Apache-2.0) against our own HLS output | Preview only, labelled not time-aligned. |
| Web UI | **React 19 + Tailwind v4 + hls.js + uPlot**, Vite, `embed.FS` | Same toolchain as nationcam's `web-next`. uPlot handles thousands of live points at ~40 KB. |
| API contract | OpenAPI spec for `/api/v1` | Generated clients for nationcam and others. |
| Time | **chrony** (host, or optional compose service) | No NTP code of our own. |
| Config | **`config.yaml` is the source of truth**; API writes it atomically | No database. Git-committable. Satisfies "YAML *and* GUI". |
| Auth | **OIDC relying party** (`coreos/go-oidc` + `x/oauth2`) against Logto or any provider; offline local admin via `scs` + `argon2id`; stdlib CSRF | No custom auth logic. See the Authentication section. |
| Metrics | `prometheus/client_golang` on `/metrics` | |
| Licence | **Apache-2.0**; NDI stays outside it | |

### Hardware

Through v0.3 the relay only repackages video, so **the network is the constraint, not the box.**
12 cameras at 10 Mbps is 120 Mbps in; pushing each to YouTube needs another 120 Mbps of upload
plus SRT's retransmission overhead (25% by default). CPU is a fraction of one core; a 2 s
alignment buffer is ~2.5 MB per stream; a watched HLS preview ~15 MB.

**Venue relay (v0.1–v0.3)** — also the Phase 0 test machine. Buy one now.

| Part | Recommendation | Why |
|---|---|---|
| CPU | Ryzen 7 mini-PC, 8 cores (7840HS / 8845HS class) | Plenty of headroom |
| RAM | 16 GB | 8 works; 32 is wasted |
| Storage | 500 GB NVMe | Config, snapshots, logs. Never SD cards or USB sticks |
| Network | **Two 2.5GbE ports, Intel preferred** | One to internet/remote cameras, one to the production LAN. No USB NICs |
| Power | Small UPS | |
| OS | Ubuntu 24.04 LTS or Debian 13, Docker, host chrony | Linux only in production |

**Main relay (~25 independent cameras)** — a server, not a workstation.

| Part | Recommendation | Why |
|---|---|---|
| Compute | 4–8 cores, 16 GB | 25 cameras at 4 Mbps ≈ 100 Mbps in, ~100 Mbps out to YouTube |
| Network | 1 Gbps, **flat-rate or unmetered bandwidth** | ~35 TB/month outbound at this load. Hyperscaler egress pricing (~$0.05–0.09/GB) turns that into roughly $1,700–3,100/month for bandwidth alone |
| Location | Region near the cameras | Distance raises RTT, which forces higher SRT latency |
| Viewers | **Never served directly** — YouTube or a CDN | Viewer bandwidth scales with audience, not cameras |

**NDI/OMT box (v1.0)** — **don't buy until Phase 0 measures it.** Likely: 16-core desktop CPU
(Ryzen 9 9950X class) for NDI/OMT encoding, an Intel Arc (A380 class) GPU for hardware decode,
32 GB, 10GbE card and switch. 12×1080p60 is 1.5–1.9 Gbps on the wire.

**Multi-resolution transcoding (v2, optional service)** — a GPU box of its own, sized by
cameras × renditions. Only needed if viewers use nationcam's own player instead of YouTube.

**Pitfalls:**
- **CGNAT** (Starlink, 5G, some ISPs): nothing can connect in to a venue. Venues push out to main.
- **Wi-Fi** for camera ingest at a venue: don't.
- **Heat:** mini-PCs throttle in hot cabinets and trucks.
- **No internet at a venue is fine for sync** — cameras syncing to the relay's chrony agree on time
  even with no upstream. No GPS clock needed.
- **UDP buffers:** SRT at scale needs larger `net.core.rmem_max`/`wmem_max`; the docs ship values.
- **Raspberry Pi 5:** fine for testing or 1–2 cameras, not production.
- **Develop on the Mac, measure on Linux:** Docker Desktop's VM distorts UDP performance and
  blocks NDI/OMT discovery, and `tc netem` is Linux-only.
- **Keep a cold spare:** config export makes swapping a unit a five-minute job. Automatic failover
  is Phase 5.

### Explicit non-goals

**No transcoding of any kind in v1** except the unavoidable decode+encode on NDI/OMT paths — no
resolution or bitrate ladders, no audio conversion. The only decoding elsewhere is the on-demand
snapshot: one keyframe per button press. Also no recording/DVR, no encoder, no clustering, no
WebRTC, no PTZ. The GUI's HLS preview player reuses our own HLS output and does no transcoding.

---

## Deployment topology: venue and main

There is **one piece of software**. "Venue" and "main" are deployment roles, not different
products and not modes in the code:

- **Venue relay:** on site, on the production LAN, next to the cameras (and often vMix/OBS).
- **Main relay:** central — cloud or a data centre — reachable from the internet.

**Both run fully on their own.** A venue relay with no main is a complete on-site product. A main
relay with no venues takes cameras directly (phones, remote cameras, nationcam cameras) and
publishes them anywhere. **The venue must never depend on the main:** if the uplink dies, on-site
production carries on untouched.

### How they work together

**Chaining is free.** Every relay speaks SRT in and out, so a venue stream published to a main
relay is just an SRT output on one box feeding an SRT input on the other. That works in v0.1 with
no extra code.

What needs deliberate design on top of that:

**1. "Push to relay" publication.** A publication type on the venue's stream page: enter the main
relay's URL and an access token once, then toggle per stream. Behind the toggle, the venue calls
the main relay's ordinary `POST /api/v1/streams` (idempotent on `external_id`), receives the
streamid and passphrase, and starts an SRT caller. **No new protocol** — it reuses the public API.
On the main relay these streams appear labelled with the venue's name. Calling *out* means it
works from behind Starlink/5G CGNAT.

**2. Keeping sync across the hop.** A group aligned at the venue would lose that alignment if the
main relay had to re-measure it from arrival times (Tier B, ±10–40 ms). So relay-to-relay links
**carry the venue's wall-clock anchors inside the stream**: a private MPEG-TS data stream mapping
PTS to wall-clock time — effectively our own RTCP Sender Report, embedded in the TS.
- The main relay recognises it and treats those streams as a new tier, **A4 (relay anchors)**.
- Relative alignment between the venue's cameras is preserved **exactly**, because every anchor
  came from the same clock.
- Against streams arriving at main from elsewhere, accuracy is bounded by the NTP agreement
  between the two boxes — a few milliseconds with a shared upstream.
- It works across any number of hops. It is added only on relay-to-relay links; outputs to vMix,
  OBS and platforms stay plain.

**3. Where alignment happens** depends on where the switcher is:

| Production style | Cameras | Switcher | Alignment |
|---|---|---|---|
| On-site | Venue LAN | vMix/OBS at the venue | **Venue relay.** Main is optional (distribution, archive feeds). |
| Remote production | Venue LAN | vMix/OBS at HQ or in the cloud | Venue aligns and pushes with anchors; **main preserves** them into its group |
| Mixed | Venue cameras **plus** phones/remote cameras connecting straight to main | vMix/OBS at HQ | **Main relay**, one group: venue streams on Tier A4, direct streams on their own tiers |
| Self-serve (nationcam) | Independent cameras anywhere | None — YouTube / web player | None: independent streams on main |

**4. Bandwidth:** the venue uplink carries every forwarded stream plus SRT's retransmission
overhead. Forwarding is per-stream toggled for exactly this reason.

### Later: managing venues from main

Seeing and controlling venue relays from the main GUI (health, alerts, remote config) needs a
connection *initiated by the venue*, since venues are often unreachable from outside. That is
the multi-node work in Phase 5/v2. The same outbound link makes a venue relay the **site agent**
that v2 PTZ needs for cameras behind NAT.

---

## API-first, and integrating with nationcam

The relay must be drivable entirely by software so sites like nationcam.com can offer self-serve
streaming on top of it. nationcam currently attempts this against datarhei Restreamer/Core, and
its code shows exactly what the replacement has to do better:

| nationcam today (Restreamer/Core) | Relay API |
|---|---|
| Live status by **polling** every process's `/state`, one call each | **One event stream** (WebSocket) for the GUI; **webhooks** for nationcam's backend |
| Fakes a Restreamer UI **metadata blob** so processes show up in the UI | GUI and API are the same API — nothing to fake |
| Filters processes by **ID prefix** to find its own | `external_id` + `labels` on every stream; list/filter by them; create is idempotent on `external_id` |
| Streams exist only in Restreamer, **drifting** from nationcam's cameras table | nationcam keeps ownership in its own Postgres; relay holds a stable id and echoes `external_id` |
| Pull-only RTSP; **no stream keys**, no push ingest | Per-stream generated credentials (SRT streamid + passphrase, RTMP key), rotate endpoint, connection string in the create response |
| No per-output status for YouTube etc. | Outputs are first-class resources with state in the event stream |
| No thumbnails used | Snapshot API — nationcam can call it on its own schedule for site thumbnails; the relay stays on-demand |
| One shared API key, also accepted **in the query string** | Scoped tokens (per-stream / per-action), header only |

**Division of responsibility:** nationcam owns users, accounts and who-owns-which-camera. The
relay owns media. The relay does not grow a user/tenant model; it trusts its API clients within
the scope of their token. That keeps multi-tenancy in the one place that already has a database.

**HLS for public players:** nationcam currently proxies HLS through its own API with a host
allow-list. The relay can instead issue short-lived signed HLS URLs so browsers play directly;
the proxy remains an option.

---

## Authentication

**Principle: write no authentication logic of our own.** Every security-critical job is handed to
an established, maintained library or to a real identity provider.

The relay does not need user management — sign-up, password reset, email verification, 2FA,
social login. nationcam already has all of that in **Logto**. So the relay is an **OIDC relying
party**: it trusts an identity provider rather than becoming one.

| Job | Library / service | Why this one |
|---|---|---|
| Users, sign-up, password reset, 2FA, social login | **Logto** (already running for nationcam) — or any OIDC provider | Don't run a second identity system. Provider-agnostic: Keycloak, Authentik, Zitadel, Auth0, or better-auth's OIDC provider plugin all work unchanged. |
| GUI single sign-on (authorization code + PKCE) | **`golang.org/x/oauth2`** (BSD-3) | The Go team's own OAuth2 client. |
| Verifying ID/access tokens, JWKS fetch + key rotation | **`coreos/go-oidc` v3** (Apache-2.0) | Used by Kubernetes, Dex and Argo CD. Handles discovery, caching and rotation correctly. |
| API access for nationcam's backend | **Logto machine-to-machine apps** (client credentials) with scopes, verified by `go-oidc` | nationcam gets scoped, expiring tokens from Logto. **No custom API-key system needed for integration.** |
| Offline local admin login | **`alexedwards/scs` v2** sessions (MIT) + **`alexedwards/argon2id`** hashing (MIT) | Works with no internet at a venue. scs gives secure cookie defaults, idle/absolute timeouts and session renewal on login; argon2id is OWASP's first-choice password hash. |
| CSRF protection | **Go stdlib `net/http.CrossOriginProtection`** | Built into Go since 1.25 — no dependency at all. |
| Standalone API tokens (deployments with no identity provider) | Random 256-bit token, only its SHA-256 stored, constant-time compare, scopes | A standard pattern, not new crypto. Only for boxes with no OIDC provider. |

**Authorization:** permissions come from token scopes and roles (e.g. `streams:read`,
`streams:write`, `outputs:toggle`, `admin`). Every API route declares the scope it needs; the
GUI is subject to the same checks. If per-camera rules get complex (PTZ in v2), adopt **casbin**
rather than growing custom rule logic.

**Considered and rejected:**
- **better-auth** — TypeScript; needs a Node runtime and its own database beside a Go binary.
- **aarondl/authboss** — the closest embedded Go equivalent to better-auth, and well maintained,
  but it brings registration, recovery and user storage we'd have to wire up and not use, and it
  doesn't verify bearer tokens from an external provider — the part integration actually needs.
- **Ory Kratos** — excellent, but a separate service with its own database, filling the exact
  role Logto already fills.

---

## Time: NTP, layered

We don't write NTP code. We use **chrony**, in two layers:

1. **Upstream:** the relay host disciplines its clock against a consistent, good source
   (e.g. `time.cloudflare.com` with NTS, or a regional pool). Remote cameras should use the
   **same** upstream, so their clocks and ours share a reference.
2. **Local:** chrony on the relay host also **serves NTP on UDP 123** to cameras on the same LAN.
   Pointing LAN cameras at the relay gives every device one reference over a sub-millisecond LAN
   path — better than each camera syncing independently over the internet. This is what makes
   Tier A1 (RTCP-SR anchoring) accurate.

**Docker caveat:** containers share the host kernel's clock and cannot set it without the
`SYS_TIME` capability.

**Decision:**
- **Production default: chrony installed on the host.** The docs ship a ready `chrony.conf`
  (upstream servers + LAN `allow` rule). The relay only reads chrony's status. Clock discipline
  stays with the OS, where it belongs, and survives the relay container restarting.
- **Quick-start only:** an optional `chrony` service in `docker-compose.yml` (with `SYS_TIME` and
  host networking) for trying the relay out without touching the host. Marked as not for
  production, because a container adjusting the host clock is surprising to whoever runs the host.
- On startup the relay checks for a synchronised clock. If none, it raises a critical alert and
  the Ports/Settings page shows exactly what to install.

Either way, disable competing time daemons (e.g. `systemd-timesyncd`). The relay reads chrony's
tracking data (offset, stratum, reachability) and shows it in the alerts bar. Losing sync is a
**critical** alert, because every sync measurement becomes untrustworthy.

---

## Audio

Modelled on Restreamer's audio handling, minus the transcoding:

| Capability | Cost | Phase |
|---|---|---|
| **Show incoming audio specs** per track — codec, sample rate, channels, bitrate | Free: already parsed from the stream | 1 |
| **Choose which audio track** a stream uses (e.g. Sony multi-channel) | Free: track selection, no decode | 1 |
| **Replace audio with another stream's audio**, e.g. camera video + a mixer's audio feed | Remux only. **Aligned by the sync engine**, since both streams already have wall-clock anchors — something Restreamer can't do | 3 |
| **Insert silence** where there's no audio (YouTube expects an audio track) | Pre-encoded silent AAC frames repeated, so no encoder needed | 3 |
| **Internet radio as the audio** — an MP3 or AAC stream from Icecast, Shoutcast or AzuraCast, e.g. music under a beach cam | New audio-only **HTTP audio ingest**. MP3/AAC frames are remuxed, never re-encoded | 3 |
| Audio level meters | Needs audio decode | v2 |
| Resample / re-encode audio (e.g. Opus or PCM sources into RTMP) | Transcode | v2 |

**Limit to state in the UI:** replacement audio must already be in a codec the output accepts.
If it isn't, the GUI says so rather than silently producing a broken output.

### Internet radio (MP3/AAC over HTTP)

What makes it work, and the three honest limits:

- **Codec:** MP3 is carried by MPEG-TS, so SRT, HLS and NDI/OMT outputs are fine. RTMP/FLV can
  carry MP3 too, but **some platforms only accept AAC** — which destinations take MP3 (YouTube,
  Facebook, others) gets verified in Phase 3, and the GUI warns per destination. An AAC radio
  stream avoids the question entirely.
- **No automatic sync.** An Icecast stream carries no timestamps, and servers often burst tens
  of seconds of buffered audio on connect. Its delay can't be measured, only trimmed by ear.
  For music under a camera that doesn't matter; for anything needing lip sync, it isn't the tool.
- **Slow clock drift.** The radio server's clock and the camera's clock run at slightly different
  rates. Without re-encoding, the only way to keep them together over hours is to occasionally
  drop or repeat one audio frame (~26 ms). At typical drift that's roughly one tiny, usually
  inaudible adjustment every few minutes.
- **Reconnects:** radio streams drop and come back. The relay inserts silence while it reconnects
  so the output — and a YouTube stream on top of it — never breaks.

### Multi-language audio (e.g. English + Spanish)

A stream carries an ordered list of **audio tracks**, each with a language code, a display name
and a default flag. HLS publishes them as alternate audio renditions (`EXT-X-MEDIA TYPE=AUDIO`),
and hls.js gives the embedded player a language selector — on nationcam's site or anywhere else.

Where each language can come from:

| Source of the second language | Cost | Phase |
|---|---|---|
| **A separate audio track** in the same incoming stream (encoder sends English on track 1, Spanish on track 2) | Free: remux | 3 (preview player selector: 1) |
| **A separate feed** — a Spanish commentator on their own SRT stream, or an HTTP audio stream | Remux. Aligned by the sync engine plus operator trim, same as audio replacement | 3 |
| **Channels inside one track** (e.g. a 4-channel track: channels 1–2 English, 3–4 Spanish) | Needs decode, channel split and re-encode — transcoding | v2 |

**Ask encoder operators for separate tracks, not separate channels.** It's the difference between
free and transcoding.

Which outputs can carry more than one audio track:

| Output | Multiple languages? |
|---|---|
| **HLS** | ✅ Alternate renditions with a player selector |
| **SRT** (MPEG-TS) | ✅ Multiple audio tracks; vMix/OBS choose which to use |
| **RTSP** | ✅ Multiple audio tracks |
| **RTMP** (YouTube, Facebook) | ⚠️ One audio track in practice — the output picks which language to send. A second RTMP output can carry the other |
| **NDI / OMT** | ⚠️ One audio stream per source — the output picks the language (verify multichannel options in Phase 4) |

---

## Ancillary data: preserve, never process

Closed captions (CEA-608/708, carried in the video's SEI) and SCTE-35 markers pass through the
SRT fast path untouched. Remux paths (RTMP, HLS) can silently drop them. The rule: **preserve
wherever the output format can carry them, never interpret or modify them**, with a test per
output proving what survives. Where a format can't carry them, the output's detail view says so.

---

## Latency budget

```
encoder capture→send      80–600 ms   (encoder-dependent; unrecoverable without Tier A)
ingress latency           SRT 2.5–4 × RTT · RTSP/RTP low · NDI/OMT LAN low
alignment buffer          = spread between streams + margin   ← the cost we add
egress path latency       SRT 20–60 ms · NDI/OMT 1–3 frames · RTMP/RTSP low · HLS seconds
downstream decode buffer  40–200 ms
```

The alignment buffer costs the **spread**, not the sum — the earliest stream waits for the latest.
Streams within 300 ms of each other cost ~350 ms. **The UI shows this live as "alignment cost"**;
it is the number an operator negotiates against.

---

## PTZ control (added after review — yes, and it fits well)

The relay is already the one box that knows every camera: its identity, address, credentials,
health, and **measured video latency**. That makes it the natural place for PTZ control, and one
capability here is genuinely differentiating rather than merely convenient.

### The three things this buys

1. **Control proxy.** One authenticated endpoint drives every camera, regardless of whether it
   speaks VISCA-over-IP, ONVIF, or an HTTP CGI. Operators stop juggling per-vendor tools and
   per-camera credentials.
2. **PTZ bridging into the switcher — the differentiating one.** vMix and OBS can drive PTZ over
   NDI's control channel, but a camera ingested as **SRT has no control channel at all**. If the
   relay presents that camera as an NDI/OMT output *and* bridges the control channel down to
   VISCA on the real camera, **vMix drives PTZ on an SRT camera through the relay**. Nothing else
   does this. It also makes the OMT/NDI egress path worth more than a CPU-offload argument.
3. **Latency-aware control.** We already measure each camera's video latency to ±ms. Nobody can
   send commands back in time, but knowing the number lets us (a) show the operator the real
   control round-trip so they stop over-correcting, and (b) prefer **absolute-position and
   move-by-increment commands over start/stop jog**, which is dramatically better over a
   400 ms link. Jogging blind over high latency is the actual operational pain, and this is a
   real mitigation.

Preset recall across several cameras at once is possible too, but the cameras' own mechanical
travel times dominate, so treat it as convenience, not sync.

### Protocols

| Protocol | Transport | Gear |
|---|---|---|
| **VISCA over IP** | UDP 52381 (Sony, sequence-numbered) · TCP 5678 / UDP 1259 (PTZOptics) | Sony, PTZOptics, Marshall — covers your whole fleet |
| **ONVIF Profile S** | SOAP/XML over HTTP | PTZOptics, Marshall, most IP cameras |
| **HTTP CGI** | Vendor-specific | PTZOptics and others |
| **NDI / OMT control channel** | In-band | Both as a *client* (drive NDI cameras) and as a *server* (the bridge above) |

VISCA-over-IP first — it covers Sony, PTZOptics and Marshall, and it is a small binary protocol.
Per-vendor quirks (Sony's sequence wrapper, PTZOptics' TCP-vs-UDP) mean a thin driver per vendor;
tedious, but bounded and testable without a camera.

### Constraints to be honest about

- **Reachability.** Control needs an IP path *to* the camera. A camera pushing SRT to us as a
  caller from behind NAT gives us no return path. Control relay works when the relay shares a
  network with the camera; genuinely remote sites need a VPN or a small site agent. **The site
  agent is out of scope for v1** — document the limitation instead of half-building it.
- **Security boundary.** PTZ means the relay stores camera credentials and can physically move
  hardware. That needs per-camera authorization (not just "logged in"), and an audit log of who
  moved what. This is the one part of the feature that must not be built lazily.
- **Scope discipline.** This is a control-plane subsystem that touches none of the media
  pipeline — which also makes it **the most parallelizable work in the project**, buildable
  alongside the sync spine without contention.

### Phasing

**Deferred to v2.** A minimal version — VISCA-over-IP driver, `/api/ptz/*`, presets, and a
pan/tilt/zoom panel in the existing UI — is roughly 2–3 weeks on top of infrastructure that
already exists. The NDI/OMT PTZ bridge is the expensive and valuable half, and it depends on the
OMT work in Phase 4.

Nothing in v1 needs to change to accommodate it later: PTZ touches none of the media pipeline,
and the config already carries per-camera identity and credential references. **The only v1
obligation is to not paint ourselves into a corner on auth** — per-camera authorization must be
expressible in the auth model from the start, even with nothing using it yet.

---

## Phase plan

**Each phase is a release, not a milestone.** Solo, the difference matters: v0.1 must be usable
on its own, and every phase boundary must leave the product shippable. Ordering puts
sync-improving work before protocol breadth, because sync is the differentiator and breadth is
the commodity.

| Phase | Ships as | Solo estimate | The point of it |
|---|---|---|---|
| 0 | — | ~1 month | Kill bad assumptions cheaply |
| 1 | **v0.1** | ~5–6 months | **SRT in, aligned SRT out, full API + GUI. The product exists.** |
| 2 | **v0.2** | ~2–3 months | Automatic accuracy — RTSP/RTCP anchoring, timing companion |
| 3 | **v0.3** | ~2–3 months | Interop — RTMP ingest, best-effort RTMP/RTSP/HLS egress |
| 4 | **v1.0** | ~3–4 months | OMT + NDI, the LAN transports |

### Phase 0 — Spike and decide (2–3 weeks, throwaway code)

**Highest-value task first:**

- **Audit the actual camera fleet.** Per model (Sony / PTZOptics / Marshall / Larix): does it emit
  SEI timecode? Does its RTSP server produce sane RTCP SRs? Can it hold NTP, and how well? Does it
  do NDI/OMT, and is the timecode populated? **This table decides how much of the roadmap is even
  necessary.** Assume nothing.
- **Settle the CPU-offload premise with numbers.** 12 streams into vMix and OBS as SRT (hardware
  decode), as NDI, and as OMT — measure client CPU each way. If hardware-decoded SRT is already
  cheap on the client, NDI/OMT become compatibility features rather than performance features and
  their phase priority drops.
- **NDI\|HX licence + feasibility review** — is a near-passthrough H.264-in-NDI path legally
  shippable in any form? Expected answer: only via a user-supplied sidecar.
- **OMT reality check** — `libomt` maturity, API stability, Go/cgo binding effort, whether OMT
  frames expose usable timecode, actual encode cost.
- Does `datarhei/gosrt` hold up? 12 in + 12 out, 1080p30 @10 Mbps, 1 h, `tc netem` (40 ms delay,
  20 ms jitter, 0.5% loss), encryption on.
- Does egress scheduling move alignment across the **receiver matrix**? Inject a known 250 ms
  delay; measure in OBS, vMix, ffplay, VLC. **ffplay is the control** — it separates our error
  from theirs.
- Prove Tier A1 on one real camera: does RTCP-SR-derived alignment land inside ±50 ms?

**Exit criteria:** a completed fleet capability table; a client-CPU comparison table; written
go/no-go on NDI\|HX and on OMT-in-core; zero unexplained SRT drops over 1 h; a commanded 250 ms
delay producing 250 ±20 ms of observed shift in OBS and vMix; RTCP-SR alignment demonstrated
within ±50 ms on real hardware. **Failures here change the stack or the mechanism before real
code exists.**

### Phase 1 — The spine: pipeline, sync engine, SRT in/out, UI

The smallest thing that is genuinely useful and proves the hard part.

- `AccessUnit` / `Ingest` / `Egress` interfaces; scheduler with **per-(stream × path) latency
  subtraction from day one**, even though only one path exists
- SRT ingest (listener/caller) and SRT egress (listener multi-subscriber / caller), plus the
  SRT→SRT passthrough fast path
- Sync engine: Tier B anchors, **persisted Tier C trim**, master selection (explicit or
  auto = latest-arriving), drift fit, slew-limited control, re-anchor on reconnect
- Live metrics: per-stream offset-from-master, drift ppm, sync tier, RTT, loss/retrans, buffer
  fill, bitrate, state, **alignment cost**
- `/api/v1` REST + **event stream** with live GUI parity, OpenAPI spec, `external_id`/labels,
  generated per-stream credentials, scoped tokens
- Sync groups + independent streams; SRT streamid on one shared ingest port and one egress port
- Versioned YAML config + separate `secrets.yaml`
- GUI: command center, stream page (with minimal **preview HLS** — pulled forward from Phase 3,
  not yet offered as a publication), snapshot button, audio specs + track select, groups, ports
  page, alerts bar
- chrony integration and NTP health alerting; `/metrics`, `/healthz`, Docker + compose
- **The measurement rig** — without it no accuracy claim is falsifiable

**Exit criteria:** 12 in / 12 out, 10 Mbps each, 8 h, no restarts, RSS stable · streams skewed
0–500 ms converge within **±20 ms at egress** and hold for 8 h under ±100 ppm simulated drift ·
**end-to-end ±50 ms in OBS and vMix**, characterised for ffplay/VLC/hardware and published as a
latency+variance table · blank config → aligned 3-camera rig in <10 min, GUI only · drop/reconnect
re-anchors and reapplies trim in <5 s · intra-stream A/V sync provably preserved.

### Phase 2 — v0.2, the accuracy release (ingest only, all of it sync work)

Deliberately contains **no egress work at all.** Every item here makes alignment more accurate or
more automatic — which is the differentiator, so it comes before breadth.

- **RTSP/RTP ingest with RTCP-SR anchoring → Tier A1** — the biggest automatic-accuracy win for
  this fleet, and it needs nothing from the camera but NTP
- **"Timing companion" mode: media from SRT, anchor from the same camera's RTSP/RTCP endpoint.**
  Highest-value item in the whole roadmap and unique to this product — SRT picture quality with
  Tier A1 timing.
- Opportunistic SEI timecode (Tier A3) — cheap now that `mediacommon` is already a dependency;
  auto-promote B→A when timecode appears, demote gracefully when it stops
- Sync tier badge and confidence in the UI
- **Venue → main:** "Push to relay" publication (uses main's public API), and relay anchors
  (Tier A4) embedded on relay-to-relay links

**Exit criteria:** three NTP-synced RTSP cameras align automatically within **±1 frame, zero
operator trim** · a camera in timing-companion mode (SRT media + RTSP anchor) matches
native-RTSP anchor accuracy within 10 ms · a mixed rig (SRT + RTSP) holds ±50 ms for 4 h ·
a group aligned at a venue relay stays within **±5 ms of its venue alignment** at the main
relay, under `tc netem` WAN impairment, for 4 h.

### Phase 3 — v0.3, the interop release (all best-effort, no sync budget to defend)

Cheap, low-risk, and it broadens who can use the thing — which for a solo open-source project is
how contributors arrive. None of these paths carry a sync guarantee, so none of them enter the
sync rig; they need correctness tests and a *measured* latency figure to display, nothing more.

- RTMP ingest (Larix, OBS-as-sender) — Tier B
- RTMP egress (to YouTube/Facebook/CDN)
- RTSP server egress
- HLS egress (fMP4, HEVC-capable) as a publication, reusing Phase 1's preview HLS; LL-HLS
  deferred unless asked
- Audio: replace with another stream's audio (sync-aligned), insert silence for YouTube
- HTTP audio ingest (Icecast/Shoutcast/AzuraCast, MP3/AAC) as a replacement audio source;
  verify MP3 acceptance per RTMP platform
- Webhooks for server-to-server event delivery (nationcam's backend)
- Captions / SCTE-35 preservation tests per output
- Codec compatibility table enforced at config validation and shown in the UI
- **Every best-effort path carries an explicit "no sync guarantee" badge in the UI**, with its
  measured latency next to it

**Exit criteria:** each egress path has a published measured latency and variance · the UI makes
the sync contract unambiguous at a glance · a mixed rig (SRT + RTSP + RTMP-from-phone) still
holds ±50 ms on the *SRT* outputs while best-effort paths run alongside · no best-effort path can
degrade a guaranteed one (proven by test, since they share buffers and CPU).

### Phase 4 — OMT and NDI (the LAN transports)

- **OMT ingest and egress, in-core** (permissive licence)
- **NDI ingest and egress as an optional sidecar** — user supplies the SDK, core stays
  Apache-2.0 and NDI-free
- Frame drop/repeat drift handling and audio resampling on decoded paths
- Documented hardware tier, hard stream-count cap, clear UI warning when the box is undersized

Ordered last among protocols because Phase 0's CPU measurements may well reduce its value, and
because it is the only phase that changes the hardware requirement. **If Phase 0 shows a large
client-CPU win, promote this phase ahead of Phase 3.**

**Exit criteria:** 8 streams of 1080p60 SRT-in → OMT-out on documented hardware, holding ±50 ms
against SRT paths for 4 h, with published CPU/GPU/network headroom · same for NDI via sidecar.

### Phase 5 — Hardening and reach

Primary/backup failover with hitless-ish switching · SMPTE timecode generation/rewrite · optional
PCR/PTS rewrite for recorder/muxer downstreams · multi-node distributed mode · adaptive jitter
handling · rotating JSONL stats. Each ships on demand.

### v2 — after v1.0 ships

- **WebRTC egress** via WHEP, plus the AAC→Opus audio transcode it forces. Deferred to keep v1
  transcode-free; purely additive as another `Egress` implementation.
- **PTZ control** (see the PTZ section above) — VISCA/ONVIF drivers, then the NDI/OMT PTZ bridge
  that lets vMix drive PTZ on an SRT-ingested camera. Independent of the media pipeline; the only
  v1 obligation is that the auth model can express per-camera authorization.
- Venue management from main: venue-initiated link for health, alerts and remote config; the same
  link makes the venue relay the site agent for PTZ behind NAT
- Whatever Phase 0–4 measurements prove is actually missing

---

## Top technical risks

| # | Risk | Why it is real | Mitigation |
|---|---|---|---|
| 0 | **Solo build over 12–16 months loses momentum before it ships** | The most likely failure mode of this project is not technical — it is a year of building with nothing usable to show | **Each phase is a release.** v0.1 at ~month 5–6 does the thing nothing else does. Phases 2–4 are additive and independently valuable. No phase boundary leaves half-built work behind a flag. |
| 1 | **Scope vs. the sync priority** | Six ingest + six egress protocols can consume all engineering time; sync quietly becomes a feature rather than the point | Phase 1 proves sync before any breadth. Phase 2 is pure sync work with zero egress. **Governing rule: no *sync-critical* protocol ships until it passes the rig.** Best-effort paths are explicitly outside that promise, which is what makes them cheap. |
| 2 | **Tier B cannot recover encoder capture latency** | Sony vs PTZOptics vs a phone differ by hundreds of ms internally; invisible to the network | Tier C trim as a first-class feature. Timing-companion mode to pull the fleet onto Tier A1. Explicit tier badges in the UI. |
| 3 | **Larix/phone sources are physically unstable** | Thermal throttling and cellular jitter change capture latency *during* a show | Larger buffers and looser expectations for Tier B/C; continuous re-measurement, not one-shot; per-stream tier badge so operators know which streams to distrust. |
| 4 | **NDI licence blocks in-core; NDI\|HX is worse** | Advanced SDK terms are more restrictive than the base SDK | OMT in-core as the strategic LAN transport; NDI confined to an optional sidecar. Phase 0 confirms in writing. |
| 5 | **NDI/OMT CPU and bandwidth** | 12×1080p60 ≈ 1.5–1.9 Gbps + 12 decodes + 12 encodes | Published hardware tier, hard stream cap, UI warning. Phase 0 measures whether the offload is even worth it. |
| 6 | **Codec incompatibility discovered at showtime** | RTMP can't carry HEVC without enhanced-RTMP; HLS needs fMP4 for HEVC | Capability table enforced at config validation, surfaced in the UI, tested in CI. |
| 6b | **Best-effort paths starve a guaranteed one** | HLS segmenting and RTMP egress share CPU and buffers with the SRT path that carries the ±50 ms promise | Per-(stream × path) buffer caps and drop policy; Phase 3 exit criteria explicitly test that best-effort load cannot degrade a guaranteed path. |
| 7 | **Receiver buffers eat alignment, differently per receiver** | OBS/vMix/VLC buffering is opaque and varies | Phase 0 measures across the matrix with ffplay as control; per-receiver settings become published documentation. |
| 8 | **`gosrt` feature/robustness gaps** | Pure-Go reimplementation; no socket groups; encryption coverage unverified | Phase 0 validation; SRT behind a ~6-method interface — the *one* abstraction worth pre-building. |
| 9 | **Path asymmetry biases Tier B anchors** | RTT/2 assumes symmetry; real WAN paths are not | Publish the ±10–40 ms bound rather than hide it. Median filtering cuts variance, never bias. Tier A1 sidesteps it. |
| 10 | **Re-anchor thrash / control oscillation** | Naive loops hunt around the target | Slew-rate limit + deadband + hysteresis on master reselection; step only on join. |
| 11 | **Stalled subscriber grows memory** | An egress that stops draining backs up into RAM | Hard per-(stream × path) buffer cap, documented drop policy, exposed as a metric. |
| 12 | **Accuracy claims unfalsifiable without a rig** | Easy to ship something that *feels* aligned | The rig is Phase 1 scope, not a nice-to-have. |
| 12b | **GUI and API drift apart** | A convenient private endpoint for one screen is how parity erodes, and then nationcam can't do what the GUI does | The GUI consumes only the public `/api/v1`; a CI check fails on any HTTP route missing from the OpenAPI spec. |
| 12c | **Can't log in at a venue with no internet** | If login depends only on an external identity provider, a dead uplink locks the operator out mid-show | A local admin login that works fully offline is mandatory, whatever external identity provider is added. |
| 13 | *(v2)* **PTZ turns the relay into a physical-actuation attack surface** | It stores camera credentials and can move real hardware; a compromised session moves cameras mid-show | Per-camera authorization distinct from general login, audit log of every action, credentials by env-var reference only. **The one part of PTZ that must not be built lazily.** v1 obligation: the auth model must be able to express per-camera scope. |
| 14 | *(v2)* **PTZ control has no return path to NAT'd remote cameras** | A camera pushing SRT as a caller gives us no route back to it | Document the limitation; support control only where the relay can reach the camera. A site agent is a v2 item, not a half-built v1 one. |

---

## Phase 1 implementation detail

### Repo layout

```
go.mod                 # module github.com/RelentNet/srtrelativity
cmd/srtrelativity/main.go
internal/media/        # AccessUnit, Track, codec params — the spine
internal/ingest/       # srt/ now; rtmp/ rtsp/ rtp/ ndi/ omt/ land here
internal/egress/       # srt/ now; rtmp/ rtsp/ hls/ ndi/ omt/ land here (webrtc/ in v2)
internal/sync/         # anchor estimation, drift fit, master selection, slew control
internal/sched/        # per-(stream × path) delay lines, buffer caps, drop policy
internal/srtconn/      # thin interface over gosrt (swap point for libsrt)
internal/api/          # REST + WS + auth
internal/config/       # YAML load/validate/atomic-save + codec capability checks
web/                   # Vite SPA → embed.FS
test/rig/              # the measurement harness
Dockerfile  docker-compose.yml
```

### Config shape (illustrative — protocol fields exist from day one)

```yaml
version: 1                # schema version; v0.2+ migrates older configs instead of rejecting them
server:
  http_addr: ":8080"
  srt_ingest_addr: ":9000"   # ONE port for all SRT ingest, routed by streamid
  srt_egress_addr: ":9100"   # ONE port for all SRT listener outputs, routed by streamid
sync:
  slew_rate_ms_per_s: 1.0
  deadband_ms: 5
  max_buffer_ms: 2000
groups:
  - id: show-a
    master: auto          # auto | <input id>
inputs:
  - id: cam-a
    name: "Camera A — Stage Left"
    group: show-a         # a group id, or omit for an independent stream (no alignment)
    external_id: "nc-video-1234"        # caller-owned id, e.g. nationcam's camera row
    labels: { owner: "logto-sub-abc" }  # free-form, round-tripped untouched
    protocol: srt         # srt|rtmp|rtsp|rtp|ndi|omt   (srt only in Phase 1)
    mode: listener        # listener | caller
    streamid: cam-a
    latency_ms: 200
    credentials: cam-a    # reference into secrets.yaml — never inline
    trim_ms: 0            # Tier C, persisted
    audio:                # ordered tracks; each output uses all it can carry, or a chosen one
      - { source: cam-a, track: 0, language: en, name: English, default: true }
      - { source: spanish-booth, track: 0, language: es, name: Español }  # another stream's audio
    # timing_companion:   # Phase 2 — anchor from RTSP, media from SRT
    #   protocol: rtsp
    #   url: rtsp://cam-a.local/stream
outputs:
  - id: cam-a-srt
    input: cam-a
    protocol: srt
    mode: listener        # served on srt_egress_addr under streamid cam-a
    enabled: true
  - id: cam-a-youtube
    input: cam-a
    protocol: rtmp
    url: rtmp://a.rtmp.youtube.com/live2
    credentials: cam-a-youtube
    enabled: false
```

**Secrets live in a separate `secrets.yaml`** (mode 0600, git-ignored), referenced by id. Env-var
references alone don't work once the API generates per-stream passphrases and stream keys at
runtime. `config.yaml` stays safe to commit.

**Scale note:** a YAML file holds hundreds of streams comfortably. If self-serve pushes that into
thousands, move state to embedded SQLite behind the same config package — not before.

### API surface

**The GUI uses only this API — there is no private GUI endpoint.** Anything the GUI can do, an
API client can do, and vice versa. Versioned under `/api/v1`, described by an OpenAPI spec so
nationcam (or anyone) can generate a client.

```
GET    /api/v1/status                         full snapshot
WS     /api/v1/events                         every state change + telemetry (see below)

GET    /api/v1/streams                        list  (?label=owner:xyz  ?external_id=…)
POST   /api/v1/streams                        create; idempotent on external_id; returns
                                              generated ingest credentials + connection string
GET    /api/v1/streams/{id}
PATCH  /api/v1/streams/{id}                   name, group, labels, latency, audio track, …
DELETE /api/v1/streams/{id}
POST   /api/v1/streams/{id}/credentials/rotate
POST   /api/v1/streams/{id}/trim              { ms }  — applies immediately
POST   /api/v1/streams/{id}/reanchor
POST   /api/v1/streams/{id}/snapshot          grab + decode one keyframe → thumbnail
GET    /api/v1/streams/{id}/snapshot          current thumbnail (JPEG)

GET    /api/v1/streams/{id}/outputs
POST   /api/v1/streams/{id}/outputs           add a push destination (RTMP, SRT caller)
PATCH  /api/v1/outputs/{id}                   { enabled: bool, … }  — the GUI toggles
DELETE /api/v1/outputs/{id}

GET    /api/v1/groups   POST /api/v1/groups   PATCH/DELETE /api/v1/groups/{id}   (master, members)
GET    /api/v1/alerts                         history; POST /api/v1/alerts/{id}/ack
GET    /api/v1/ports                          every port required / in use, with owner and state
GET    /api/v1/system                         CPU, memory, NTP status, version
GET/PUT /api/v1/config                        YAML import/export (validated, atomic)
GET/POST/DELETE /api/v1/webhooks              server-to-server event delivery (Phase 3)

GET    /metrics   /healthz   /readyz
```

**Live parity.** Every mutation — from the GUI, an API client, a hand-edited `config.yaml` being
reloaded, or the engine itself (stream dropped, re-anchored, alert raised) — goes through one
state store that emits an event. The GUI renders from `/api/v1/events`, so a change nationcam
makes via the API appears in an open GUI within a second, with no refresh. Events carry a
monotonically increasing sequence number so a reconnecting client can ask for what it missed.

### UI

**Resembles** datarhei Restreamer's layout — a source card plus a publications list — because that
shape maps one-to-one onto `Ingest` and `Egress`. Not a replica: own visual identity, no
Restreamer branding or assets. The addition that makes it *this* product is that sync is visible
everywhere it applies.

**Stack:** React 19 + Tailwind v4 + hls.js + uPlot, built into `embed.FS` — the same toolchain
nationcam's `web-next` already uses, so components and skills carry across.

**1. Command center (home)**
- **One section per sync group**, each with its own **alignment strip** across the top: master
  at 0 ms, every member a marker at its offset; the strip's width is that group's live
  **alignment cost**. The one element Restreamer doesn't have.
- **Independent streams** in their own section with no strip, no offset and no tier badge —
  just thumbnail, name, state and bitrate. They add zero delay and never affect a group.
- Tiles: snapshot thumbnail (or a placeholder until the first snapshot), name, state, and for
  grouped streams **offset from master** + **sync tier badge**. Click to open the stream page.
- **Add stream** button → wizard: pick ingest protocol → listener/caller → sync group or
  independent → latency → finish on the **connection string to hand the camera operator**
  (copy button; QR for Larix). SRT defaults to the shared ingest port with a generated streamid.

**2. Stream page**
- **Player:** HLS preview via hls.js, labelled **"preview — not time-aligned"** (2–6 s behind).
  For framing, exposure and identity only; operators judge sync in vMix's multiview. Preview HLS
  is generated only while someone is watching. If the browser can't decode the stream's codec
  (HEVC on some machines), say so instead of showing a black player.
- **Snapshot button:** grabs one keyframe, decodes it with a one-shot `ffmpeg` call, and sets or
  replaces the command-center thumbnail. On demand only — no background decode loop.
- **Stats row:** Uptime · kbit/s · FPS · **Offset** · **Tier** (last two only for grouped streams).
- **Sync card** (grouped streams only): group selector, offset and drift sparklines, buffer fill,
  trim control with keyboard nudge (±1 ms / ±1 frame). **Trims apply immediately:** adding delay
  briefly freezes that camera, removing delay jumps it forward at the next keyframe (so the
  picture never breaks up). When the stream has connected viewers, the control warns before
  applying. Automatic drift correction is unaffected — it keeps slewing invisibly. Staging a trim
  until the camera is off-air needs vMix/OBS tally integration and is a later addition.
- **Audio card:** incoming audio track specs (codec, sample rate, channels, bitrate); an ordered
  list of output audio tracks, each with language, name and default flag; add a track from this
  stream, another stream or an HTTP audio feed; insert silence. The player shows a language
  selector when there's more than one track. See the Audio section.
- **Connection string** for this input, with copy and rotate-credentials buttons.
- **Publications card (egress):**
  - **Toggle-only**, no setup: NDI, OMT, HLS, RTSP server path, SRT listener (shared egress port,
    streamid)
  - **Form** needed: push destinations — RTMP (YouTube/Facebook/custom), SRT caller
  - Every row carries a **contract badge**: ✅ sync-guaranteed (SRT, NDI, OMT) or best-effort
    (RTMP, RTSP, HLS), plus its copyable URL where applicable
- **Process details / report** equivalent: per-stream event log (connected, dropped, re-anchored,
  trim changed and by whom).

**3. Groups** — create and rename sync groups, pick each group's master, move streams between
groups or make them independent.

**4. Ports** — every port the relay needs or is using: port, UDP/TCP, what uses it (which stream,
output or system service), direction, and state (listening / in use / **conflict**). Buttons to
copy the equivalent `docker-compose` port mappings and firewall rules (ufw). Conflicts also fail
config validation, so a bad port never reaches a running show.

**5. Settings** — auth, API tokens, webhooks, config YAML import/export, NTP details.

**Alerts bar, bottom of every page** — the latest alerts, colour-coded by severity:
**critical** (red: stream lost, NTP lost, buffer cap hit), **warning** (amber: drift over
threshold, reconnecting, best-effort output failing), **info** (neutral: stream connected,
re-anchored, config changed). Click to open the full history with acknowledge. The same bar
carries CPU / memory, version and **NTP health**, replacing a separate footer. Alerts come
through the same event stream, so API clients and webhooks see exactly what the GUI shows.

### The measurement rig (`test/rig/`)

One source file → N `ffmpeg` senders, each behind its own `tc netem` profile with an injected
known offset → relay → N recorders → compare frame-level burned-in timecode. Emits one number:
worst-case pairwise misalignment over the run. This is what turns "±50 ms" into a test.

Two layers, deliberately separated:
- **Automated (CI gate):** ffmpeg senders → ffmpeg recorders only. Measures *our* error with no
  third-party buffering in the path. Every new protocol adds a row here before it ships.
- **Manual (per release):** the same rig terminating in OBS, vMix, VLC and a hardware decoder, to
  refresh the published compatibility table.

---

## Verification

- `go test ./...` — parsing against captured real-world fixtures from **each camera model in the
  fleet**; sync solver against synthetic skew/drift traces with known answers; codec capability
  matrix enforced in tests.
- `test/rig` under `tc netem`: the 8-hour soak and the ±100 ppm drift run, asserting the Phase 1
  exit criteria numerically.
- Manual: real Sony / PTZOptics / Marshall / Larix sources → relay → **OBS and vMix**, clapper
  test; alignment holds after 4 hours and survives pulling a network cable.
- `docker compose up` on a clean 16 GB host; confirm `/metrics`, `/healthz`, and the UI.
