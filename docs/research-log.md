# Research log

Dated entries. One entry per work session, even short ones. This IS your reproducibility
trail and your future paper's evidence base - do not skip this because it feels like
overhead, it is the actual deliverable when there's no CVE yet.

## Template
### YYYY-MM-DD
- **Library:**
- **What I did:**
- **Result:**
- **Next step:**

---

# Day 1:
## 2026-07-11 — ESP-IDF and QEMU environment validation
- Installed ESP-IDF v5.5.4 on Windows.
- Created a minimal ESP32 project named `hello_test`.
- Successfully configured the target as `esp32`.
- Successfully compiled the application and bootloader.
- Installed Espressif QEMU 9.2.2 for Xtensa.
- Diagnosed QEMU startup failure as missing `libiconv-2.dll`.
- Resolved the dependency using the Git for Windows MinGW runtime path.
- Successfully generated the QEMU flash and eFuse images.
- Successfully booted the firmware under ESP32 QEMU.
- Confirmed execution reached and returned from `app_main()`.
**Result:** ESP-IDF → build → firmware image → QEMU → `app_main()` pipeline validated.


# Day 2:
Good find — and the caution in that trace is exactly right, so let me sharpen it further before you go looking at more test folders, because there's a specific technical detail that determines whether this is provable at all with ASan alone.
The bug class you're actually looking at
buf[1] being read without checking *len >= 2 is very unlikely to be an out-of-bounds heap/stack overflow in the classic ASan sense — in embedded C like this, rcv_frame is almost certainly a fixed-size buffer (like the ptemp_buf[MB_TCP_BUFF_MAX_SIZE] array we already saw in port_read_packet), allocated at its max size regardless of how many bytes were actually received. So buf[1] is very likely reading memory that's technically in-bounds but logically stale — leftover data from a previous packet, or uninitialized memory. ASan will not flag this. You need MemorySanitizer (MSan), which specifically tracks uninitialized-value reads, or a manual "poison the buffer between packets" test harness that deliberately fills rcv_frame with a known sentinel pattern (e.g., 0xAA) before each simulated receive, sends your 1-byte [0x83] PDU, and checks whether the returned exception code equals your sentinel byte — if it does, you've proven it's reading stale/uninitialized data, cleanly, without needing MSan tooling at all. That second approach is actually the faster, more portable proof for a WOOT-style writeup: it's a self-contained reproducer anyone can run, not something that depends on your sanitizer build working correctly.
Write this down in your research log now, before you forget the reasoning:Log the candidate bug and the sentinel-poisoning proof strategy before continuing the traceLog the candidate bug and the sentinel-poisoning proof strategy before continuing the traceNow let me actually check that test folder for you instead of relaying instructions through another layer.Reconciled version discrepancies and acknowledged research limitationsReconciled version discrepancies and acknowledged research limitationsMy search didn't get me inside that specific folder — but it surfaced something worth knowing before you go further: this is genuinely useful context, not filler.
Important context: you're auditing recently-rewritten code
The search results show esp-modbus underwent a real architectural rewrite — the older structure (v1.x) was built on the original 2006 FreeMODBUS codebase with files like mbc_tcp_master.c under a freemodbus/ prefix. The current v2.x release (which is what you're tracing — the mb_objects, mbm_check_invoke_handler, port_tcp_utils.c naming you found matches this) is described as supporting multi-instance TCP master/slave objects, which points to an object-oriented rewrite, not the old codebase. This is genuinely good news for your candidate: newer rewritten code has had far less real-world audit time than code descended from a 2006 library, which raises the odds a real bug survived. Keep that framing for your eventual writeup — "recently rewritten, object-oriented v2.x architecture" is a legitimate part of your motivation section, not just a technical detail.
I don't have reliable visibility into the exact current contents of test_apps/unit_tests/mb_controller_common from search — you're faster continuing that yourself with the repo cloned locally now. Do this:
cd esp-modbus
ls test_apps/unit_tests/mb_controller_common
That's instant, no GitHub UI roundtrip. Send me what's actually in there.
While you're at it, one more useful local-clone move
grep -rn "mbm_check_invoke_handler" --include=*.c --include=*.h .
grep -rn "MB_PDU_SIZE_MIN\|MB_PDU_DATA_OFF" test_apps/
That answers the "does any existing test touch this path" question in one shot, across the whole repo, not just the one folder the earlier trace was walking toward. You have the repo on disk now — use it. Send me the actual output of both commands.


# Day3:
### Sink-level reproducer CONFIRMED on real hardware
- Board: ESP32 (COM3, CH9102 USB-serial)
- Test: test_truncated_exception_pdu
- Result: PASS — 5 Tests, 0 Failures, 0 Ignored
- Confirmed: mbm_check_invoke_handler() returns buf[1] (the sentinel 0xAA) when
  called with func_code=0x83, len=1 — i.e., it reads past the caller-declared
  PDU boundary on the exception-response branch.
- Accurate claim (do not overstate): sink-level unchecked read confirmed on
  hardware. NOT yet confirmed: real TCP peer delivering this condition
  end-to-end. NOT yet established: any exploitability beyond stale-byte
  misinterpretation as an exception code.
- Next: end-to-end reachability via crafted Python fake-slave + physical_tests
  master, per plan already in progress.

# Day 4 :
# ESP-Modbus Security Research Log

## Candidate MB-001: Truncated Exception PDU Length Validation

### Status

Sink-level behavior confirmed on ESP32 hardware.

End-to-end network reachability has not yet been confirmed.

Remote exploitability and security impact have not yet been established.

### Environment

- Repository: espressif/esp-modbus
- Branch: research/mb-truncated-exception
- ESP-Modbus version: 2.1.3 development tree
- ESP-IDF version: v5.5.4
- Target: ESP32
- Hardware revision: ESP32-D0WD-V3 revision 3.1
- Serial port: COM3
- Test framework: ESP-IDF Unity fixture tests

### Relevant Function

`mbm_check_invoke_handler()` in:

`modbus/mb_objects/mb_master.c`

The exception-response branch reads:

```c
if (func_code & MB_FUNC_ERROR) {
    exception = (mb_exception_t)buf[MB_PDU_DATA_OFF];
    return exception;
}