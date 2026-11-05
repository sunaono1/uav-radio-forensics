# UAV Radio Forensics — DJI OcuSync DroneID

Passive reception and forensics on UAV radio links. Getting the signal is step one; after that the questions are how much was captured, whether it decoded correctly, and whether the result still holds on another aircraft model.

The tooling and evidence framework come from [luyii-code-1's DroneID research](https://github.com/luyii-code-1/dji-ocusync-droneid-research); provenance and credits are in [CREDITS.md](CREDITS.md).

[English](README.md) | [简体中文](README.zh-CN.md) | [Research plan](RESEARCH.md) | [Progress](docs/PROGRESS.md) | [Evidence ledger](docs/FACTS.md) | [Credits](CREDITS.md)

---

## Research questions

The question that comes up most in low-altitude monitoring work is: what makes you say you detected it, and what makes you say it was that aircraft. Energy on a spectrum does not settle it. Neither does a decoded frame header. A chain that survives review has to answer four things:

| ID | Question | Our bar |
|---|---|---|
| RQ1 | What did passive reception actually capture, and how complete was it | From the capture parameters we can compute the frames expected in that window against the frames observed; missed frames get an attribution (hopping not covered / SNR too low / capture overrun), not "weak signal" |
| RQ2 | Is a decode correct, or merely plausible | Both CRCs must pass (transport CRC24A and logical DJI CRC16). One layer passing makes it a candidate, not a result |
| RQ3 | Does association hold across sessions for the same aircraft | The 4-byte session hash matches between CRYP and INFP; a session refresh invalidates the old key; the conclusion does not rest on a single capture |
| RQ4 | Do the first three survive a different model or firmware | Every conclusion carries model, firmware and capture parameters. Anything without cross-model validation is labelled a single observation |

So the output here is not a "we can decrypt it" claim. It is a method with error bars and a stated scope. The plan, the experiment matrix and the milestones are in [RESEARCH.md](RESEARCH.md).

### Why the split falls here

The original project's public line is two things: session-key predictability, and the AeroScope dongle hardware route. Both are about getting plaintext back. That work is necessary to understand the protocol, but a monitoring deployment, an incident report, or a compliance review does not need plaintext. It needs to know how much was missed, how often it is wrong, whether another model still matches, and whether someone else can recompute the report.

The weight therefore sits on reproducibility and auditability. Every conclusion layer needs its intermediate artefacts (IQ → symbols → LLR → transport block → logical packet → plaintext), and a missing layer caps the conclusion at the layer below. That rule comes from the original project's evidence grading; here it is the main process rather than a note at the end of the docs.

---

## What each project does

| Item | Original (luyii-code-1) | This repository |
|---|---|---|
| O2 PHY / FEC / CRC reference implementation | Built and locally verified | Inherited as a regression baseline |
| O4 logical packet structure (CRYP / INFP) | Built and cross-confirmed | Inherited for packet-layer checks |
| Session-key predictability ("route 2" upstream) | Main research line | Not reproduced, not extended; only its disclosure boundary is discussed, in a forensic context |
| AeroScope dongle hardware route | Third-party demonstration recorded | Out of scope |
| Detection-side method (hit rate, missed-frame attribution, false positives) | Not systematically covered | Main line here |
| Cross-model / cross-firmware boundaries | Listed as pending | Main line here, as an experiment matrix |
| Evidence auditability (artefact retention, third-party recomputation) | Grading rules exist | Promoted to a mandatory artefact spec |

---

## Protocol baseline

Inherited facts, used for alignment. Nothing here is re-derived.

### Common logical packet header

```text
offset  size  meaning
0       1     packet_length_minus_3
1       1     message type / subtype
2       4     ASCII marker (`CRYP`, `INFP`, ...)
6       4     hashcode (session association)
...     ...   type-specific body
last-2  2     DJI CRC16, little-endian
```

Logical length is `packet[0] + 3`. DJI CRC16 uses initial value `0x3692` with reflected polynomial `0x8408`; a complete logical packet leaves a zero remainder.

### CRYP / AA: wrapped session key

```text
AA 13 43 52 59 50 [hashcode]
      C  R  Y  P

offset    size   interpretation
10:74     64     C1 = X || Y, two 32-byte big-endian coordinates
74:106    32     C3
106:122   16     C2 (plaintext length = the 16-byte session key)
122:...   ...    remaining protocol fields / tail
```

C1 satisfies the standard SM2 curve equation (`a`, `b`, `p` are in the upstream docs). C1 from independent sessions lands on the curve. Conceptually:

```text
S    = d × C1
mask = SM2_KDF(S.x || S.y, 16)
note = C2 XOR mask
C3   = SM3(S.x || note || S.y)
```

`d` is the receiver-side private scalar. It is not in this repository and is not something we try to obtain.

### INFP / 87: telemetry encryption

```text
cipher = AES-128-CTR
key    = note (the session key)
IV     = nonce8 || 0x00×8
```

With a same-session `note`, consecutive 87 packets decode into stable field structure: sequence numbers, timestamps, coordinates and altitude stay coherent.

### O2 frame and FEC chain

```text
sample rate       15.36 MS/s
FFT               1024
subcarrier space  15 kHz
active carriers   600, DC excluded
symbols           9
normal CP         72
long CP           80
symbol 3          ZC root 600
symbol 5          ZC root 147
data symbols      1,2,4,6,7,8
```

```text
6 QPSK symbols × 600 carriers × 2 bit = 7200 soft bits
→ Gold descramble
→ LTE reverse rate matching (E=7200, RV=0)
→ LTE Turbo (K=1408, D=1412)
→ 176-byte transport block
→ CRC24A
→ logical payload
→ DJI CRC16
```

A frame counts only when both CRCs pass.

### Sample rate is not negotiable

HackRF raw files are interleaved signed-int8 I/Q, 2 bytes per complex sample. A 20 MS/s capture feeding the 15.36 MS/s detector has to be resampled by `20 × 96 / 125 = 15.36`. Get that wrong and the ZC time scale, CP/FFT alignment, CFO estimate and CRC result all go wrong together — and it looks a lot like a bad signal rather than a bad setting.

---

## Layout

```text
src/o4_packet_tool.py          CRYP/AA structure check; decrypt INFP/87 with a known note
src/o2_droneid_decode.py       O2 PHY/FEC reference decoder
src/droneid_hackrf_scanner.py  HackRF live / replay scanner
tools/remove_turbo_soft.c      TurboFEC adapter
tests/                         unit tests for the packet tool and frequency presets
docs/PROGRESS.md               milestones for this repository
docs/FACTS.md                  facts and evidence ledger (inherited, sourced)
docs/STATUS.md                 research status (inherited, sourced)
RESEARCH.md                    plan: questions, hypotheses, experiment matrix, metrics, milestones
CREDITS.md                     original author and every upstream source
NOTICE                         GPL-3.0 derivative statement and modification record
```

---

## Quick start

Requires Python 3.10+, numpy, scipy, and the OpenSSL CLI.

```bash
python3 -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt

# inspect a complete AA/CRYP packet
python3 src/o4_packet_tool.py '<complete-AA-hex>'

# decrypt a same-session 87 packet when the 16-byte key is already known
python3 src/o4_packet_tool.py --note '<16-byte-note-hex>' '<complete-87-hex>'

# passive 5.8 GHz scan, including 5816.5 MHz
python3 src/droneid_hackrf_scanner.py --band 5g --preset baseline

# both bands, using the upstream Issue #2 frequency list
python3 src/droneid_hackrf_scanner.py --band both --preset issue2

# custom centre frequency (overrides --preset and --band)
python3 src/droneid_hackrf_scanner.py --freqs 2474.5
```

The public tools cover packet validation and the AES step for a known key. No private commercial key material, and no `d`.

### Frequency presets

| `--preset` | Meaning | 2.4 / 5.8 GHz entries |
|---|---|---|
| `baseline` (default) | The upstream scan order, with 5816.5 MHz added to 5.8 GHz | 5 / 4 |
| `issue2` | The frequency list reported in upstream Issue #2 | 4 / 4 |
| `paper2022` | Bender 2022 Table II, duplicates removed; **historical, not an O4 frequency map** | 5 / 7 |

The three presets come from observations made at different times and **must not be merged into one "universal" frequency table**. The 2022 paper's prose mentions a 2474.5 MHz upper bound while Table II repeats 2459.5 MHz, so `paper2022` does not include 2474.5.

The published v3 specification describes an 8-symbol waveform and seven-frequency hopping (2414.5 / 2429.5 / 2444.5 / 5756.5 / 5776.5 / 5796.5 / 5816.5 MHz, with `2459.5` excluded). **The scanner here implements neither v3 PHY nor hop following.** `--dwell` is not synchronisation.

---

## Evidence rules

Order of precedence (inherited, enforced here):

```text
raw IQ
> locally reproduced double-CRC result
> locally reproduced plaintext fields
> independent primary protocol confirmation
> public code
> discussion claims
> hypothesis
```

What follows:

- energy is not DroneID;
- a ZC/CP match is a PHY candidate, not a decode;
- CRC24A passing means the transport block is trustworthy;
- DJI CRC16 passing means the packet boundary and content are trustworthy;
- a valid encrypted packet stays ciphertext until the crypto step is reproduced;
- a third party saying it has an offline decoder proves an implementation exists, not that the method is publicly reproducible.

Artefacts every experiment must leave behind:

```text
candidate.iq          raw candidate segment
symbols_fd.npy        frequency-domain symbols
equalized.npy         after equalisation
llr_7200.npy          soft bits
descrambled.npy       after descrambling
transport_176.bin     transport block
logical_payload.bin   logical payload
decode.json           parameters and verdict
```

---

## Milestones

| Stage | Goal | Done when |
|---|---|---|
| M0 | Environment and replay path working | Upstream tests pass; a given capture reproduces both CRCs consistently |
| M1 | Capture-parameter sensitivity | Three-factor matrix over sample rate / gain / duration, with a hit-rate curve and missed-frame attribution |
| M2 | Frequency coverage and hop following | A minimal follower over the published seven-frequency table, with measured capture rate |
| M3 | Cross-model boundaries | At least one double-CRC-valid sample per model (Air 3S / Mini 4 Pro / Avata 2 and others), firmware recorded |
| M4 | Method write-up | A detection and forensics procedure a third party can recompute from the docs, including false-positive analysis |

Current state: [docs/PROGRESS.md](docs/PROGRESS.md). Nothing past M0 has samples yet.

---

## Publication and privacy

Examples use synthetic or anonymised identifiers and coordinates. Not published: service credentials, session keys, real serial numbers, precise operator or home coordinates, or acquisition metadata that identifies a person.

## Use boundaries

This repository supports interoperability research, spectrum analysis and authorised security research. Use it for passive reception and for testing you are explicitly authorised to perform, within the radio, privacy, aviation and computer-security rules that apply where you are.

DJI, OcuSync, AeroScope and related product names belong to their respective owners. This project is independent and has no affiliation with, or endorsement from, DJI or any third party mentioned here.

---

## Provenance

The tooling and evidence framework come from **[luyii-code-1](https://github.com/luyii-code-1)'s [dji-ocusync-droneid-research](https://github.com/luyii-code-1/dji-ocusync-droneid-research)** (GPL-3.0): the DroneID PHY / logical packet / crypto research framework, the three tools in `src/` (HackRF scanner, O2 decoder, O4 packet tool), the TurboFEC adapter in `tools/remove_turbo_soft.c`, and the bilingual evidence ledger under `docs/`. The protocol's own description lives in that repository and is more complete than here.

What we changed is the research question. The original asks "can this link be decoded". We ask "once it decodes, does it hold up as evidence". Per-file provenance and modification records are in [NOTICE](NOTICE); full credits in [CREDITS.md](CREDITS.md).

---

## Licence and credits

GPL-3.0. Original copyright belongs to luyii-code-1 and the rights holders named in the licence; the derivative parts of this repository are released under GPL-3.0-or-later as well. Per-file provenance and modification records are in [NOTICE](NOTICE).

**Original research, tooling and evidence framework: [luyii-code-1 / dji-ocusync-droneid-research](https://github.com/luyii-code-1/dji-ocusync-droneid-research).** The code and the evidence rules here come from there.

Upstream public work this repository cites:

- **EdwardBlair** — [dji-droneid-spec](https://github.com/EdwardBlair/dji-droneid-spec): O4 protocol, v3 RF/PHY specification, synthetic test vectors
- **alphafox02** — [antsdr_dji_droneid](https://github.com/alphafox02/antsdr_dji_droneid): independent confirmation of CRYP/INFP semantics
- **proto17** — [dji_droneid](https://github.com/proto17/dji_droneid): receiver and hopping discussion
- **King-Of-Knights** — dongle console demonstration in upstream Issue #1, frequency list in Issue #2
- **Bender et al. (2022)** — historical DroneID frequency data
- **Edgesource / Aerial Defence (2024)** — AeroScope upgrade module hardware and protocol whitepaper

Full citations, with comment links and page numbers, are in [CREDITS.md](CREDITS.md).
