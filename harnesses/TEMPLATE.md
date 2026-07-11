# Harness template — copy this folder structure per library

harnesses/<library-name>/
├── harness.c              # the actual fuzz target, calls the library's real API
├── build.sh               # compiles harness + library for the fuzzing target (host or QEMU)
├── seeds/                 # initial corpus — real, valid inputs for this library
├── NOTES.md               # entry point chosen, why it's attacker-reachable, known limitations

## harness.c skeleton (libFuzzer-style, adapt for AFL++ persistent mode if targeting QEMU)

```c
#include <stdint.h>
#include <stddef.h>
// #include "the_library.h"

int LLVMFuzzerTestOneInput(const uint8_t *data, size_t size) {
    // Call the REAL public API entry point here, not a simplified reimplementation.
    // Example shape only — replace with the actual library call:
    // library_parse(data, size);
    return 0;
}
```

## Before you trust any fuzzing result from this harness:
1. Run `afl-cov` (or equivalent) once and confirm the harness reaches meaningful code,
   not just initialization.
2. Confirm the harness is calling attacker-reachable code — write this down in NOTES.md
   explicitly (e.g., "this parses data received over mDNS UDP packets from the local
   network, attacker model = on-path or malicious LAN device").
