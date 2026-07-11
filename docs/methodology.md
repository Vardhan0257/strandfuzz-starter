# Methodology

## Harness design rules
1. One harness per library, in `harnesses/<library>/`, fully self-contained.
2. Harness must call the library's real public API - not a stub, not a simplified
   reimplementation. Document exactly which entry point(s) are targeted and why they're
   attacker-reachable (network input? file input from an untrusted source? both?).
3. **Prove the harness actually reaches meaningful code before trusting any fuzzing result.**
   Run it once under afl-cov or equivalent and eyeball the coverage report. A harness that
   only exercises initialization code is worthless - this is the single most common way
   fuzzing projects silently fail.
4. Compile with ASAN/UBSAN where the QEMU target supports it - bugs found are much stronger
   evidence with sanitizer confirmation than a bare segfault.

## Fuzzing loop
- Espressif QEMU fork for the emulation-loop targets.
- AFL++ as primary; libFuzzer as fallback for libraries easier to harness as a pure
  in-process fuzz target (compiled for host, per Espressif's own Jan 2026 host-based
  fuzzing guidance - this is often FASTER than QEMU-in-the-loop, consider it seriously
  per library rather than defaulting to QEMU every time).
- Track and log: executions/sec, edge coverage % over time, time-to-first-crash (or
  time-fuzzed-with-zero-crashes, which is also a real, reportable result).
- Run each campaign multiple times (aim for a handful of repeated runs where feasible) -
  fuzzing has real run-to-run variance and a single lucky/unlucky run isn't a result.

## Triage
1. Every crash -> minimize the reproducing input (afl-tmin or equivalent).
2. Classify: real bug vs. QEMU emulation artifact. Espressif's QEMU fork has known gaps
   in peripheral fidelity (WiFi/BT stacks, some hardware crypto paths) - if a crash only
   reproduces in emulation and never on real hardware, say so explicitly, don't discard
   it silently, log it as an emulator-artifact finding.
3. Confirmed real bug -> reproduce on real ESP32/ESP8266 hardware -> write a minimal PoC.
4. Responsible disclosure: contact the library maintainer (and Espressif if it's an
   official component) before any public writeup. Standard 90-day-style disclosure norms
   unless the maintainer requests otherwise.

## Metrics to report, always, regardless of outcome
- Libraries attempted, libraries successfully harnessed (these can differ - log why if a
  library resists harnessing)
- Edge/code coverage % achieved per library
- Total fuzzing CPU-hours per library
- Crashes found, deduplicated crash count, confirmed-real vs. emulator-artifact count
- CVEs filed, disclosure timeline
