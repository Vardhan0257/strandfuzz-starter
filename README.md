# STRANDFUZZ

Systematic vulnerability discovery in third-party libraries vendored into ESP32/ESP8266 firmware SDKs (ESP-IDF, Arduino-ESP32), using Espressif's official QEMU-based emulator for coverage-guided fuzzing, with real-hardware validation before disclosure.

## Status
🟡 Setup phase — no libraries fuzzed yet.

## Why this exists
A single vulnerable vendored library gets silently compiled into a huge number of unrelated downstream IoT products. Finding and disclosing these early has real value independent of any paper outcome. See `docs/motivation.md`.

## Project structure
```
strandfuzz/
├── docs/
│   ├── motivation.md          # the pitch, novelty framing, honest positioning
│   ├── library-candidates.md  # tracked shortlist + verification status per library
│   ├── methodology.md         # harness design rules, triage process, metrics
│   └── research-log.md        # dated log — every session, what ran, what happened
├── harnesses/
│   └── <library-name>/        # one folder per library, self-contained
├── corpus/
│   └── <library-name>/        # seed corpora, kept out of git if large (see .gitignore)
├── triage/
│   └── <library-name>/        # crash artifacts, minimized reproducers, analysis notes
└── .github/workflows/         # CI: build harnesses + short smoke-fuzz on every push
```

## Quick start
See `docs/methodology.md` for the full pipeline. Short version:
1. Pick a library from `docs/library-candidates.md` (status: `ready-to-harness`)
2. Build a harness in `harnesses/<library>/` following the template
3. Fuzz under QEMU with AFL++/libFuzzer (see methodology doc for exact invocation)
4. Any crash → triage/<library>/ → minimize → reproduce on real hardware → responsible disclosure

## Publication note
No fixed deadline. This is published only once there is a real, validated finding worth reporting — see `docs/motivation.md` for the honest venue-tiering (WOOT realistic, main-track CCS/S&P/USENIX aspirational and contingent on either significant findings or the harness-auto-generation extension, see roadmap).
