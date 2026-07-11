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
