# Phase 0 — Spike and Decide

**Purpose:** kill the assumptions that could invalidate the architecture, cheaply, before any
production code exists. All code in this phase is throwaway. Target: 2–3 weeks.

**Exit gate:** every table below filled in, and the four decisions at the bottom written down.

---

## 0.1 Fleet capability audit — *do this first*

This single table decides how much of the roadmap is even necessary. **Assume nothing; measure
every cell.** A camera that turns out to emit SEI timecode removes months of inference work for
that stream; one that holds NTP well makes Tier A1 trivial.

| Camera | SRT out | RTMP out | RTSP out | NDI / OMT | SEI timecode? | RTCP SR sane? | NTP sync? | NTP holds to | Audio: tracks or channels? | PTZ protocol |
|---|---|---|---|---|---|---|---|---|---|---|
| Sony ______ | | | | | | | | | | |
| PTZOptics ______ | | | | | | | | | | |
| Marshall ______ | | | | | | | | | | |
| Larix (Android) | | | n/a | n/a | | n/a | | | | n/a |
| Larix (iOS) | | | n/a | n/a | | n/a | | | | n/a |

**How to fill each column**

- **SEI timecode** — capture 10 s of TS, then inspect the video elementary stream for H.264
  `pic_timing` SEI clock timestamps (needs `pic_struct_present_flag` in VUI), HEVC `time_code`
  SEI (payload type 136), or `user_data_unregistered` (type 5) vendor payloads. Record *which*,
  not just yes/no.
- **RTCP SR sane** — connect over RTSP, log Sender Reports, confirm the NTP field is real
  wall-clock and not zero/monotonic. Confirm SRs arrive at a usable rate (≥1 per 5 s).
- **NTP holds to** — leave the camera synced for 24 h, then compare its RTCP NTP field against a
  known-good host. Record the drift, not just "yes".
- **PTZ protocol** — VISCA over IP (which transport and port?), ONVIF Profile S, HTTP CGI.

**Deliverable:** this table, plus captured sample streams from each camera committed as test
fixtures for the parser tests in Phase 1.

---

## 0.2 The CPU-offload question — settle it with numbers

The stated reason for NDI/OMT output is lowering client CPU. That premise is testable in a day,
and the answer reorders Phase 3 vs Phase 4.

| Client | 12× SRT (HW decode) | 12× SRT (SW decode) | 12× NDI | 12× OMT | Network used |
|---|---|---|---|---|---|
| vMix | | | | | |
| OBS | | | | | |

**If hardware-decoded SRT is already cheap on the client, NDI/OMT are compatibility features, not
performance features, and Phase 4 stays where it is. If the win is large, promote Phase 4 ahead
of Phase 3** — Phase 3 is all best-effort interop work and is the easiest thing to postpone.
Note that NDI/OMT at 1080p60 is ~125–160 Mbps *per stream* — record whether the network, not the
CPU, is the real ceiling.

Weigh this against the fact that NDI/OMT are the only egress paths besides SRT that carry a
**sync guarantee**, so Phase 4 is differentiating work while Phase 3 is commodity work.

---

## 0.3 Licence and feasibility reviews

| Question | Answer | Decided |
|---|---|---|
| Can NDI\|HX (H.264/HEVC in an NDI wrapper) ship in *any* form alongside Apache-2.0 code? | | ☐ |
| Advanced SDK terms — what exactly do they forbid? | | ☐ |
| Is `libomt` mature and API-stable enough to depend on? | | ☐ |
| Go/cgo binding effort for `libomt` — days or weeks? | | ☐ |
| Do OMT frames expose usable per-frame timecode (Tier A2)? | | ☐ |
| Actual OMT encode cost per 1080p60 stream | | ☐ |

Expected answer on NDI\|HX: still a user-supplied sidecar. Write down the actual finding either
way — this decision is load-bearing for the licence.

---

## 0.4 SRT stack validation

Does `datarhei/gosrt` hold up, or do we fall back to cgo `libsrt`?

**Test:** 12 inbound + 12 outbound, 1080p30 @ 10 Mbps, 1 hour, encryption on, under
`tc netem` — 40 ms delay, 20 ms jitter, 0.5 % loss.

| Metric | Result | Pass? |
|---|---|---|
| Unexplained drops over 1 h | | ☐ zero |
| RSS stable (no leak) | | ☐ |
| CPU at steady state | | ☐ |
| AES-128 / 192 / 256 all work | | ☐ |
| Caller *and* listener both solid | | ☐ |
| Socket groups / bonding available? | | (expected: no) |

**Main-relay load test:** 25 independent SRT inputs at 4 Mbps, each pushed out over SRT and RTMP,
1 hour, same `tc netem` profile. Record CPU per stream and RSS — this sizes the main relay.

| Metric | Result | Pass? |
|---|---|---|
| Unexplained drops over 1 h | | ☐ zero |
| CPU per stream | | ☐ recorded |
| RSS stable | | ☐ |

---

## 0.5 Does egress scheduling actually move alignment?

The core mechanism. Inject a known **250 ms** delay on one of two identical streams and measure
the observed shift at each receiver. **ffplay is the control** — it has the least added buffering,
so it separates *our* error from *theirs*.

| Receiver | Observed shift | Variance | Settings required | Pass (250 ±20 ms)? |
|---|---|---|---|---|
| ffplay *(control)* | | | | ☐ |
| OBS | | | | ☐ |
| vMix | | | | ☐ |
| VLC | | | | ☐ |

If OBS or vMix fail, the mechanism changes to PTS rewriting and the architecture needs revisiting
before Phase 1 starts.

---

## 0.6 Prove Tier A1 on real hardware

Take one NTP-synced RTSP camera. Derive its wall-clock anchor from RTCP Sender Reports. Compare
against ground truth (a clapper, or a second camera pointed at the same clock).

| Measurement | Result | Pass? |
|---|---|---|
| RTCP-SR-derived anchor error | | ☐ within ±50 ms |
| Stability over 1 h | | ☐ |
| Behaviour when SRs stop arriving | | ☐ degrades cleanly |

This is the experiment that determines whether **timing-companion mode** (media over SRT, anchor
over RTSP) is worth building in Phase 2. If it passes, it is the highest-value item in that phase.

---

## 0.7 PCR skew stability

Log `skew(t) = arrival_wallclock − pcr_time` for 30 minutes on a real SRT stream and fit a line.
The slope is the sender↔relay clock drift; the residual tells us whether Tier B is usable.

| Measurement | Result | Pass? |
|---|---|---|
| Fit residual (RMS) | | ☐ < 5 ms |
| Measured drift slope | | ☐ plausible (±100 ppm) |

---

## Decisions to write down before Phase 1 starts

1. ☐ **SRT stack** — `gosrt`, or cgo `libsrt`?
2. ☐ **Alignment mechanism** — egress scheduling confirmed, or PTS rewrite needed?
3. ☐ **NDI / OMT priority** — does the CPU data justify promoting Phase 4 ahead of Phase 3?
4. ☐ **Timing-companion mode** — worth building in Phase 2, based on 0.6?

Record each with the number that justified it, not just the verdict.
