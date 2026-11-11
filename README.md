# NILM + SDR Appliance Monitor

Non-intrusive residential appliance monitoring and fingerprinting for 120/220 V
60 Hz service, fusing two modalities:

- **AFE metering** (ADE9000-class) — current/voltage harmonic fingerprints. The
  *meter*: quantifies watts, RMS, power factor, THD, and harmonic vectors.
- **Software-defined radio** — radiated + conducted EMI fingerprints. The
  *detector*: identifies *which* appliance is running (no power/energy).

This repo is a **Hermes Agent skill** (procedural knowledge) plus the project's
design specification. It is not yet built hardware.

## Contents

- `SKILL.md` — the loaded skill: locked design decisions, procedure, verified
  numbers, pitfalls, verification.
- `references/spec.md` — full v0.1 specification (single source of truth):
  objective, isolation posture, AFE decision, bandwidth/interface architecture,
  Tier-1 conducted coupling network BOM, SDR receiver tiers + host sizing, and
  the two-evening calibration that runs first.
- `README.md` — this file.

## Status

v0.1 — design + front-end spec locked. Awaiting hardware calibration. No mains
work done. The two-evening calibration gates the receiver upgrade and any
permanent install.

## Key decisions (summary)

- Isolation is mandatory on every signal path: clamp CT, magnetic step-down
  transformer (voltage ref), isolated coupling network (conducted EMI). Nothing
  touches mains directly.
- Compute-on-device, ship feature vectors — never stream raw continuously
  (6 circuits @ 1 Hz is ~0.9 kB/s).
- SDR starts with an RTL-SDR V3 + shielded near-field probe set; upgrade to
  RSP1A/B or Airspy HF+ Discovery only if conducted signatures prove
  discriminating.
- SDR host floor: Raspberry Pi 4 (2 GB). The service runs headless on Linux;
  feature vectors publish to MQTT -> InfluxDB -> Grafana.

## Safety

This project involves residential mains. The isolation rules in
`references/spec.md` section 2 and the Tier-1 safety rules in section 5 are
non-negotiable. Treat any coupling-network output as live, earth box and host,
and verify against a published reference (Jim Brown K9YC "Power Line Noise and
RFI") before touching mains.

## License

MIT