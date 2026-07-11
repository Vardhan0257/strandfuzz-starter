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


