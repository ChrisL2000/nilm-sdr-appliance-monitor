---
name: nilm-sdr-appliance-monitor
description: "Monitor and fingerprint appliances via NILM and SDR EMI."
version: 0.1.0
author: ChrisL2000, Hermes Agent
license: MIT
platforms: [linux]
metadata:
  hermes:
    tags: [NILM, SDR, EMI, energy-monitoring, smart-home]
    related_skills: [openhue, grafana-observability-ops]
---

# NILM + SDR Appliance Monitor Skill

Non-intrusively monitor a 120/220 V 60 Hz residential service and fingerprint
individual appliances by fusing two modalities: (A) a metering AFE for
current/voltage harmonic fingerprints (the METER — quantifies watts, RMS, pf,
THD, harmonics) and (B) an SDR for radiated + conducted EMI fingerprints (a
DETECTOR — identifies WHICH appliance runs, no power/energy). Design state and
full detail live in `references/spec.md`.

## When to Use

- Resuming or extending this project after a break.
- Adding appliance-level energy or EMI monitoring to a home lab.
- Designing isolation, sample-rate, or interface-bandwidth tradeoffs for NILM.

Don't use for: plain smart-plug on/off control (use `openhue` / Tasmota), or
generic Grafana dashboarding (use `grafana-observability-ops`).

## Locked Design Decisions

- AFE: ADE9000-class metering front-end (7 ADC, Class 0.2, on-chip DSP, 32 kS/s
  waveform buffer resampled to 128 pt/cycle for >=50-harmonic FFT) on
  ESP32/STM32. Budget path: ATM90E32 / ADE9153A / MCP39F511.
- Isolation is mandatory on every signal path: clamp CT + burden for current;
  magnetic (non-switching) AC step-down transformer for the voltage reference;
  isolated coupling network for conducted EMI. Nothing touches mains directly.
- Architecture: on-device/host compute -> feature vectors, NOT raw continuous
  streaming. 6 circuits @ 1 Hz = ~0.9 kB/s.
- SDR receiver tiers: characterize with RTL-SDR V3 first; upgrade to RSP1A/B or
  Airspy HF+ Discovery only if conducted signatures prove discriminating.
- Two-evening calibration runs FIRST and gates the receiver upgrade + permanent
  install. See `references/spec.md` section 8.

## Prerequisites

- Host: any Linux box; Raspberry Pi 4 (2 GB) is the floor, a retired x86 is
  comfortable (SDR load is single-digit % of one core, ~300-600 MB RAM).
- SDR: RTL-SDR V3 + a shielded near-field probe set (primary). Optional discone
  for whole-home ambient. SMA adapters for N/BNC antennas.
- Software: SoapySDR (+ osmosdr/SDRplay/airspyhf backends), GNU Radio or
  scipy, Python. Downstream: existing MQTT -> InfluxDB -> Grafana.

## Procedure (build order)

1. Two-evening calibration — radiated first, zero mains work.
   Evening 1: RTL-SDR V3 + loop/probe near each appliance; spectrogram
   on-minus-off diff, tuner mode 30 MHz-1 GHz (direct-sampling 500 kHz-28.8 MHz
   as a bonus check only). Evening 2 (conditional, only if 1 is insufficient):
   Tier-1 conducted coupling network at one outlet, 150 kHz-30 MHz.
   Output = appliance x band x discriminability table. Gate the next steps on it.
2. Tier-1 conducted coupling network (only if Evening 2 needed). BOM and
   topology in `references/spec.md` section 5. Safety rules are non-negotiable.
3. AFE board (ADE9000-class) — spec after calibration justifies the build.
4. Host: run calibration on the existing box; add a dedicated Pi only for a
   permanent install (RF PSU isolation + reboot decoupling).

## Quick Reference (verified numbers)

- Coupling cap 47 nF: 56 kOhm @ 60 Hz / 22.6 Ohm @ 150 kHz / 3.4 Ohm @ 1 MHz;
  ~2.1 mA leakage @ 120 V (under GFCI 5 mA); use 33 nF @ 230 V.
- Sampling -> harmonic ceiling @ 60 Hz: 6 kS/s -> 50th, 25 kS/s -> 200th,
  100 kS/s -> 800th. Sweet spot = 50th harmonic (6-25 kS/s).
- Feature vector: scalars 32 B/ch, 2-50 harmonic vector 392 B/ch.
- Raw waveform: 2ch 16-bit @ 25 kS/s = 100 kB/s; SDR raw I/Q: V3 4.8 MB/s,
  RSP1A 12.1 MB/s, Airspy HF+ 3.1 MB/s.
- SDR input ceiling: +10 dBm = 0.71 Vrms into 50 Ohm.

## Pitfalls

- A 50/60 Hz isolation transformer is NOT an HF coupler — its leakage inductance
  and winding capacitance kill 150 kHz-30 MHz. Isolation and RF-passing must
  come from different elements (class-Y cap + earthed box).
- ADC dynamic range beats sample rate for RF fingerprinting
  (8/14/18-bit -> 48/84/110 dB).
- Self-interference: the board/host PSU radiates into the bands you listen to.
  Keep the antenna remote, enclosure shielded, ferrites on DC lines.
- RF is a presence/likelihood feature, not metrology — fingerprints drift with
  antenna geometry and line impedance; re-baseline and weight RF as
  weak-but-orthogonal.
- Single CT sized for a 200 A service cannot resolve a 50 mA charger; use
  per-circuit CTs or multi-range clamps.
- RTL-SDR direct-sampling mode has weak dynamic range and AM bleed-through in
  the low band; drive Evening-1 verdicts off tuner-mode 30 MHz-1 GHz.

## Verification

- Calibration table produced and saved (see `references/spec.md` section 8).
- Coupling network: leakage < 5 mA (GFCI), -61 dB mains remnant at the pad,
  clamp + 20 dB pad present, box and host earthed before any mains work.
- Feature vectors flowing into MQTT -> InfluxDB -> Grafana at ~Kbps scale, not raw.

## References

- `references/spec.md` — full v0.1 specification (objective, isolation posture,
  AFE decisions, bandwidth architecture, Tier-1 BOM, SDR tiers + host, and the
  two-evening calibration). This is the single source of truth for the design.
