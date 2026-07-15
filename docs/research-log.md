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

[``c
//if (func_code & MB_FUNC_ERROR) {
//    exception = (mb_exception_t)buf[MB_PDU_DATA_OFF];
//    return exception;
//}  ]


# Day 5:
Malformed network response
        ↓
PDU length = 1
        ↓
Exception function code = 0x83
        ↓
Missing exception-code byte
        ↓
Parser still accesses buf[1]
        ↓
Observed buf[1] = 0x00
        ↓
0x00 interpreted as MB_EX_NONE
        ↓
Malformed response reported as successful
        ↓
Application receives 0x0000 with error = 0


## End-to-End Mechanism Confirmation: Truncated Exception PDU

**Date:** 2026-07-15  
**Status:** Confirmed  
**Environment:** ESP32, ESP-IDF v5.5.4, ESP-Modbus TCP master  
**Test application:** `test_apps/mb_tcp_e2e_repro`  
**Controlled peer:** `tools/truncated_exception_server.py`

### Objective

The previous end-to-end test confirmed that a truncated Modbus/TCP exception response was reported as a successful parameter read. However, the returned application value was `0x0000`, while the controlled sink-level test returned the poisoned sentinel value `0xAA`.

This experiment was performed to determine the value of `buf[1]` when the truncated response reached the exception-processing logic and to explain the difference between the sink-level and end-to-end observations.

### Test Input

The controlled Modbus/TCP peer returned the following eight-byte response:

```text
00 04 00 00 00 02 01 83



Add a new section at the **bottom** of `docs/research-log.md`. Do not replace your previous sink-level or E2E evidence. Paste this:

````markdown
## End-to-End Mechanism Confirmation: Truncated Exception PDU

**Date:** 2026-07-15  
**Status:** Confirmed  
**Environment:** ESP32, ESP-IDF v5.5.4, ESP-Modbus TCP master  
**Test application:** `test_apps/mb_tcp_e2e_repro`  
**Controlled peer:** `tools/truncated_exception_server.py`

### Objective

The previous end-to-end test confirmed that a truncated Modbus/TCP exception response was reported as a successful parameter read. However, the returned application value was `0x0000`, while the controlled sink-level test returned the poisoned sentinel value `0xAA`.

This experiment was performed to determine the value of `buf[1]` when the truncated response reached the exception-processing logic and to explain the difference between the sink-level and end-to-end observations.

### Test Input

The controlled Modbus/TCP peer returned the following eight-byte response:

```text
00 04 00 00 00 02 01 83
````

The Modbus Application Protocol header declared:

```text
Length: 0x0002
Unit identifier: 0x01
```

Therefore, the received Modbus PDU contained only one byte:

```text
83
```

`0x83` represents function code `0x03` with the Modbus exception bit set. A valid Modbus exception response requires an additional exception-code byte. That required byte was intentionally omitted.

### Instrumentation

Temporary diagnostic instrumentation was added to the exception-processing path in:

```text
modbus/mb_objects/mb_master.c
```

The trace recorded:

* Received function code
* Logical PDU length
* `buf[0]`
* `buf[1]`

The diagnostic output was:

```text
MB_E2E_TRACE: func_code=0x83, pdu_len=1, buf[0]=0x83, buf[1]=0x00
```

The same result was observed for all three tested parameter reads.

### Observed Application Behavior

Immediately after each trace, the ESP-Modbus master reported the parameter read as successful:

```text
CHAR #0 MB_hold_reg-0 (Data) value = (0x0000) parameter read successful.
CHAR #0, value: 0x0, expected: 0x1111, error = 0.

CHAR #1 MB_hold_reg-1 (Data) value = (0x0000) parameter read successful.
CHAR #1, value: 0x0, expected: 0x2222, error = 0.

CHAR #2 MB_hold_reg-2 (Data) value = (0x0000) parameter read successful.
CHAR #2, value: 0x0, expected: 0x3333, error = 0.
```

The Unity test failed because the returned values did not match the expected values, even though the Modbus API reported no error:

```text
1 Tests 1 Failures 0 Ignored
FAIL
```

### Confirmed Mechanism

The trace confirms the following execution sequence:

1. The Modbus/TCP master received a one-byte exception PDU containing only `0x83`.

2. The parser correctly observed a logical PDU length of one:

```text
pdu_len=1
```

3. Despite the missing exception-code byte, the exception-processing path accessed:

```c
buf[1]
```

4. The byte at that location was observed as:

```text
buf[1]=0x00
```

5. The value `0x00` was interpreted as `MB_EX_NONE`.

6. The malformed and truncated exception response was therefore propagated as a successful parameter read rather than rejected as an invalid response.

7. The application received a value of `0x0000` while the returned error status was zero.

### Explanation of the Sink-Level and End-to-End Difference

The sink-level and end-to-end results are consistent with the same missing length validation.

In the controlled sink-level test, the backing buffer was deliberately initialized with the sentinel value `0xAA`. The invalid access to `buf[1]` therefore produced:

```text
0xAA
```

In the real end-to-end network path, the logical PDU still contained only one valid byte, but the byte observed at `buf[1]` was:

```text
0x00
```

This caused the exception value to evaluate to `MB_EX_NONE`, resulting in the malformed response being reported as successful.

The difference between `0xAA` and `0x00` does not contradict the two experiments. Both demonstrate that the exception-processing path accesses `buf[1]` without first verifying that the logical PDU contains the required exception-code byte.

### Confirmed Finding

ESP-Modbus does not verify that an exception PDU contains the required exception-code byte before accessing `buf[1]`.

A truncated Modbus/TCP exception response containing only an exception function code can therefore cause the parser to consume a byte outside the logical PDU boundary. In the reproduced end-to-end execution, the observed byte was `0x00`, which was interpreted as `MB_EX_NONE`. As a result, the malformed response was silently reported as a successful parameter read with a returned value of `0x0000`.

### Impact

An application using the ESP-Modbus TCP master may be unable to reliably distinguish a malformed or truncated exception response from a successful Modbus operation.

This can cause corrupted or incomplete network responses to be silently accepted and can expose invalid or default parameter values to application logic while the API reports success.

This finding is characterized as a protocol robustness and input-validation defect. The confirmed impact is silent acceptance of a malformed truncated response. Memory disclosure has not been demonstrated.

### Suggested Validation

Before reading the exception-code byte, the implementation should verify that the logical PDU contains at least the function-code byte and the required exception-code byte.

The exception path should reject a truncated PDU before accessing `buf[1]`.

### Separate Unconfirmed Observation

A Guru Meditation double-exception crash occurred after test completion and was reproducible during multiple runs.

The crash has not been established as a consequence of the missing exception-PDU length validation. It is therefore recorded only as a separate, unconfirmed observation and is not included in the confirmed impact of this finding.

````

Then save the file and run:

```powershell
cd D:\projects\HARDWARE\strandfuzz-starter\strandfuzz

git diff -- docs\research-log.md
````

Review the diff. If the new section looks correct:

```powershell
git add docs\research-log.md

git commit -m "Confirm E2E truncated exception stale-byte mechanism"

git tag e2e-mechanism-confirmed-v1

git status
```

One precision correction: your professor called it a “stale-value mechanism,” but your current evidence proves that `buf[1]` was `0x00` outside the **logical PDU boundary**. It does **not yet prove where that zero originated**—reused buffer contents, initialization, or another transport behavior. The research log above deliberately avoids claiming “stale memory” as proven. That distinction matters in a vendor report.
