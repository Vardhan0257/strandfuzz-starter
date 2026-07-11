# Local setup (run this on your own machine, not a VM without internet)

Do this once. Takes 30-60 min depending on connection speed.

## 1. ESP-IDF toolchain
```bash
mkdir -p ~/esp
cd ~/esp
git clone -b v5.5 --recursive https://github.com/espressif/esp-idf.git
cd esp-idf
./install.sh esp32,esp32c3,esp8266   # pulls the toolchains you need
. ./export.sh                         # run this in every new terminal session
```
Verify:
```bash
idf.py --version
```

## 2. Espressif QEMU fork
```bash
cd ~/esp
git clone https://github.com/espressif/qemu.git
cd qemu
./configure --target-list=xtensa-softmmu,riscv32-softmmu --enable-debug
ninja -C build
```
Verify the binary exists: `./build/qemu-system-xtensa --version`

If you hit missing build deps (ninja, libglib2.0-dev, libpixman-1-dev, etc.), install via
your distro's package manager first — Espressif's QEMU README lists the exact list, check
it if configure fails.

## 3. AFL++
```bash
git clone https://github.com/AFLplusplus/AFLplusplus.git
cd AFLplusplus
make distrib
sudo make install
```

## 4. Sanity check the whole pipeline before touching a real library
Build and run one of ESP-IDF's own `hello_world` examples under QEMU first. If this
doesn't work, nothing downstream will, so don't skip it:
```bash
cd ~/esp/esp-idf/examples/get-started/hello_world
idf.py set-target esp32
idf.py build
~/esp/qemu/build/qemu-system-xtensa -nographic -machine esp32 \
  -drive file=build/hello_world.bin,if=mtd,format=raw
```
You should see the ESP-IDF boot log and "Hello world!" printed. Ctrl-A then X to exit QEMU.

## 5. VS Code
Install the official "Espressif IDF" extension — it wraps most of the `idf.py` commands
into the VS Code UI and handles the environment sourcing for you automatically per-project.

## 6. Real hardware validation setup
- Flash a real board once with the same hello_world to confirm your USB/serial drivers
  work before you need them for a real crash reproduction: `idf.py -p <PORT> flash monitor`
- On Linux, you may need your user in the `dialout` group for serial port access.




## new one after update
# Local setup — Windows (native, no WSL needed)

ESP-IDF ships pre-built QEMU binaries for Windows directly. You do NOT need to build QEMU
from source, and you do NOT need WSL for this step. (Original version of this doc assumed
a Linux-style from-source build — corrected here after checking Espressif's current docs.)

## 1. Install the "Espressif IDF" VS Code extension
- Open VS Code → Extensions (Ctrl+Shift+X) → search "Espressif IDF" → Install
  (publisher: Espressif Systems)
- This is the easiest path on Windows — it wraps the whole setup in a GUI wizard instead
  of raw terminal commands.

## 2. Run the setup wizard
- Ctrl+Shift+P → type "ESP-IDF: Configure ESP-IDF Extension" → Enter
- Choose "Express" install
- Pick the latest stable ESP-IDF version (v5.5 or newer)
- Let it download ESP-IDF itself + the Xtensa/RISC-V toolchains + Python env
  (this step takes a while — real download size, be patient, don't cancel it)

## 3. Install QEMU (pre-built, Windows-native)
Once the extension setup finishes, open the "ESP-IDF Terminal" from the VS Code command
palette (Ctrl+Shift+P → "ESP-IDF: Open ESP-IDF Terminal") — this terminal has the right
environment variables already loaded. Then run:
```
python %IDF_PATH%\tools\idf_tools.py install qemu-xtensa qemu-riscv32
```
Verify it installed:
```
python %IDF_PATH%\tools\idf_tools.py list
```
You should see `qemu-xtensa` and `qemu-riscv32` listed as installed.

## 4. Sanity check — build and boot hello_world under QEMU
Still inside the ESP-IDF Terminal:
```
idf.py create-project hello_test
cd hello_test
idf.py set-target esp32
idf.py build
idf.py qemu monitor
```
You should see the ESP-IDF boot log scroll by. To exit: Ctrl+] (or check the terminal
output for the exact exit key — VS Code's integrated terminal sometimes intercepts Ctrl-A).

If this works, your whole toolchain is confirmed working end to end. Don't touch a real
library until this step succeeds.

## 5. Real hardware validation setup (for later, once you have a harness + a crash)
- Plug in your ESP32/ESP8266 board via USB
- Windows should auto-install the CP210x or CH340 USB-serial driver; if the board doesn't
  show up as a COM port in Device Manager, install the driver manually (search "CP2102
  driver" or "CH340 driver" depending on your board)
- Flash: `idf.py -p COMx flash monitor` (replace COMx with the actual port from Device Manager)

## 6. AFL++ — this DOES need Linux
AFL++ itself is Linux-only (no native Windows build). Two options once you're actually
ready to fuzz (not needed yet — this is a later step):
- **WSL2** for the fuzzing loop specifically, while keeping ESP-IDF/QEMU on native Windows
  for the build+boot steps. This split is fine — QEMU and AFL++ just need to talk to the
  same binary artifact.
- Or run the whole pipeline inside WSL2 once you're comfortable with it, using the
  Linux-native instructions (git clone + install.sh, same idf_tools.py qemu install command
  works identically inside WSL2).
Cross this bridge when you're actually about to fuzz something — not needed for today.
