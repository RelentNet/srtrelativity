# SRT Relativity

A sync-first live media relay. It accepts independent camera streams over SRT, RTMP, RTSP/RTP,
NDI, OMT and HTTP audio, **actively aligns them to a common wall-clock time base**, and publishes
them over SRT, RTMP, RTSP, HLS, NDI and OMT.

Every open-source media server can move streams around. None of them align streams *to each
other*. That is the whole point of this one.

**Status: pre-alpha. Architecture defined, Phase 0 validation next. No usable build yet.**

## The problem

SRT removes jitter *within* a link, but has no notion of alignment *across* links. Twelve remote
cameras arriving over twelve connections land at the switcher with arbitrary, slowly drifting
offsets — often hundreds of milliseconds apart, drifting further by up to ~0.36 s per hour as
encoder clocks diverge. Operators fix this by hand and re-fix it every hour.

This relay measures those offsets continuously and holds them, targeting **±50 ms across 12+
streams**.

## How it works, briefly

SRT carries no absolute time — its timestamps run on each sender's own clock. So alignment is
recovered from the best time source each stream offers:

| Tier | Source | Accuracy |
|---|---|---|
| A1 | RTCP Sender Reports (RTSP/RTP) → NTP wall clock | ±5–50 ms, automatic |
| A2 | NDI / OMT per-frame timecode | ±1 frame |
| A3 | Timecode embedded in the video stream | ±0–1 frame |
| A4 | Timing carried from another SRT Relativity relay | venue alignment preserved |
| B | Arrival timing + network round-trip correction, drift tracking | ±10–40 ms |
| C | Operator trim, saved per stream | as good as the operator |

Streams are aligned within **sync groups**; a stream in no group is an independent relay with no
added delay. Drift is corrected slowly and invisibly.

## The sync contract

Sync only matters where a consumer combines multiple streams, so the guarantee is scoped to where
it means something, and the UI says so plainly:

| Output | Contract |
|---|---|
| **SRT, NDI, OMT** | **±50 ms guaranteed** — these feed vMix/OBS |
| RTMP, RTSP, HLS | Best-effort, measured latency shown, no sync guarantee |

## Venue and main

The same software runs on site (a **venue** relay next to the cameras) and centrally (a **main**
relay on the internet). Each works alone; together, a venue pushes streams to main and the
alignment survives the trip.

## Built to be driven by software

Everything the web GUI does goes through the public, versioned REST API, and every change — from
the GUI, the API or the engine — appears live everywhere through one event stream. That makes it
suitable as the media layer behind self-serve sites such as nationcam.com.

## Design commitments

- **Sync is the priority.** No sync-guaranteed output ships until it passes the test rig.
- **Honest numbers.** Every accuracy and latency figure is measured by a reproducible rig and
  published with its variance.
- **Works with anything.** The SRT output is standard MPEG-TS over SRT. OBS and vMix are the
  primary targets.
- **No transcoding in v1**, except the decode/encode NDI and OMT inherently require.
- **Permissive licence.** NDI's SDK can't live inside Apache-2.0, so NDI is an optional add-on;
  OMT is permissively licensed and built in.

## Planned for v2

PTZ camera control (including driving PTZ on SRT cameras from inside vMix), WebRTC output,
multi-resolution HLS through an optional transcoding service, and managing venue relays from main.

## Documentation

- **[docs/ARCHITECTURE.md](docs/ARCHITECTURE.md)** — full design, phases, hardware and risks
- **[docs/PHASE0.md](docs/PHASE0.md)** — the validation work before code

## Licence

Apache-2.0 — see [LICENSE](LICENSE). Copyright RelentNet.
