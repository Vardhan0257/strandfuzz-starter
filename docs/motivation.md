# Motivation and honest novelty framing

## The problem
ESP32/ESP8266 chips are embedded in a huge share of cheap consumer IoT. A bug in a vendored
third-party library (filesystem, protocol parser, JSON parser) doesn't get fixed just because
the library gets patched upstream — it stays baked into every product that already shipped
with it. One bug, many silent victims.

## What is NOT novel here (say this explicitly, always)
- Coverage-guided fuzzing of embedded C code inside QEMU-family emulators — well established
  (P2IM, HALucinator, Firm-AFL, Para-rehosting, GDBFuzz, and Espressif's own Jan 2026 fuzzing
  guidance).
- The general "vendored library supply-chain risk" framing — CVE-2026-6688 (FatFs wrapper
  buffer overflow) already popularized this for ESP32 specifically, and did so via an
  LLM-assisted audit (GitHub Copilot, "auto" mode), NOT classic fuzzing. Do not cite this CVE
  as precedent for a fuzzing methodology — it isn't one. Cite it only as evidence the target
  class (vendored ESP32 libraries) is live and currently yielding real bugs.

## What IS the actual claimed contribution
Pick ONE of these framings before writing anything, and be consistent:

1. **Breadth study (baseline, WOOT-tier):** first systematic, multi-library security
   evaluation of commonly-vendored ESP32/ESP8266 libraries via emulation fuzzing +
   hardware validation. Contribution = coverage table, confirmed CVEs, methodology
   artifacts (harnesses, corpora, configs) released for reproducibility.

2. **Harness-auto-generation (stretch, RAID/DIMVA-tier):** a tool that automatically
   generates fuzzing harnesses for arbitrary newly-vendored libraries, evaluated on
   auto-vs-manual coverage, generalization rate, and engineer-hours saved. This is the
   one thing that could genuinely push this above WOOT — see roadmap.md.

3. **Comparative methodology study (stretch, RAID/DIMVA-tier):** rigorous head-to-head of
   QEMU/AFL++ fuzzing vs. Espressif's host-based fuzzing recommendation vs. LLM-assisted
   auditing (Copilot-style), same libraries, same time budget, real metrics. Nobody has
   done this rigorously — the runZero work is an anecdotal blog post, not a controlled study.

## Venue tiering (do not inflate this when talking to anyone)
- **Primary, realistic:** USENIX WOOT — rewards demonstrated exploits over theory.
- **Secondary, plausible if breadth is large (15-20+ libraries):** RAID, DIMVA, ACSAC.
- **Aspirational only:** CCS/S&P/USENIX Security main track — needs framing #2 or #3 above
  executed well, or an unusually high number of significant CVEs. Not a baseline expectation.
- **Safe floor:** domestic/regional venues (IEEE INDICON/ANTS, ACM ISEC/ICDCN) as a guaranteed
  fallback if there's ever a hard publication deadline again — not needed now since there
  isn't one.
