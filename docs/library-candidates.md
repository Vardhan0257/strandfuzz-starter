# Library candidate tracking

Status values: `unverified` -> `verified-open` (no known prior fuzzing coverage found) ->
`ready-to-harness` -> `harnessed` -> `fuzzing` -> `done`

Do NOT move a library past `unverified` until you've personally checked:
- [ ] OSS-Fuzz project list (github.com/google/oss-fuzz/tree/master/projects) - is the
      upstream library already continuously fuzzed there?
- [ ] NVD/CVE search for the library name - any recent CVEs, and how were they found?
- [ ] GitHub issues/PRs on the library repo mentioning "fuzz", "AFL", "libFuzzer", "ASAN"
- [ ] General web search for "<library name> fuzzing" / "<library name> CVE 2025 2026"

| Library | Source | Type | Status | Notes |
|---|---|---|---|---|
| LittleFS | ESP-IDF component | Filesystem | unverified | Less scrutinized than FatFs post-CVE-2026-6688 spotlight - check anyway |
| ESP-MODBUS | Official Espressif component | Protocol parser | unverified | Industrial protocol, untrusted-input parsing, less mainstream |
| mDNS component | ESP-IDF | Network service discovery | unverified | Network-reachable - strongest threat model of this list if it pans out |
| espressif/json_parser | ESP Component Registry | JSON parser | unverified | Classic fuzzing target class |
| zlib/libpng wrapper (idf-extra-components) | Espressif wrapper around upstream libs | Compression/image, wrapper layer | unverified | Upstream almost certainly in OSS-Fuzz already - do NOT fuzz upstream, only feasible if you scope specifically to the ESP32 glue/wrapper code, same bug pattern as CVE-2026-6688 |

Add rows as you find more candidates via `components.espressif.com` and
`github.com/espressif/idf-extra-components`.
