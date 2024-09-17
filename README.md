# Raspberry Pi Pico 2 Buildroot

How to build:

```bash
# Prepare Repo
git clone https://github.com/Mr-Bossman/pi-pico2-linux
cd pi-pico2-linux
git submodule update --init

# Build Linux Image
make -C buildroot BR2_EXTERNAL="$PWD/" raspberrypi-pico2_defconfig
make -C buildroot

# Build 2nd Stage Bootloader, and flash it and the kernel
# (See notes on Toolchain and SDK below for ENV setup)
make -C psram-bootloader flash-kernel
```

## Designed to work with [SparkFun Pro Micro - RP2350](https://www.sparkfun.com/products/24870)

## Also working on the [Pimoroni Pico 2 Plus](https://shop.pimoroni.com/products/pimoroni-pico-plus-2)

![Image of boot](images/booting.png)


#### NOTES on Toolchain

The [Pico SDK Manual](https://datasheets.raspberrypi.com/pico/raspberry-pi-pico-c-sdk.pdf)
section 2.10 suggests installing and using the "Top of Tree"
[Corev Compiler Toolchain](https://www.embecosm.com/resources/tool-chain-downloads/#corev).

```bash
# Install the Corev Toolchain to /opt/riscv/corev-openhw-gcc-*/
sudo mkdir -p /opt/riscv/
cd /opt/riscv/
sudo tar -xvf /path/to/corev-openhw-gcc*.tar.gz
```

To use this toolchain with the Pico SDK, set the `PICO_TOOLCHAIN_PATH` variable.

```bash
export PICO_TOOLCHAIN_PATH="$(realpath /opt/riscv/corev-openhw-gcc-*)"
```


#### NOTES on SDK

The [Pico SDK](https://github.com/raspberrypi/pico-sdk) is expected to be
downloaded at `~/.pico-sdk/`. To use the SDK and Tools installed to another
location see [this issue](https://github.com/raspberrypi/pico-sdk/pull/1820#issuecomment-2291611448).

In short, `PICO_SDK_PATH` must be set to the path of your SDK install.

```bash
export PICO_SDK_PATH="~/example_sdk/sdk/2.0.0/"

# The SDK should assume these. They weren't necessary for me. 
# export pioasm_DIR=~/example_sdk/tools/2.0.0/pioasm
# export picotool_DIR=~/example_sdk/picotool/2.0.0/picotool
```


### NOTES on Picotool

Picotool is required to load the Linux image onto the SPI flash.

`picotool` can be built without this image loading support. This happens if
cmake can not find libusb 1.0 at buildtime. When this occurs, the help message
of `picotool` will read as follows:

`Tool for interacting with an RP2040/RP2350 binary`

When image loading support has been built successfully, the help message will
mention interacting with boards in BOOTSEL mode:

`Tool for interacting with RP2040/RP2350 device(s) in BOOTSEL mode, or with an RP2040/RP2350 binary`

Suggest manually building `picotool` if having issues with it missing libusb.
It is [available on github](https://github.com/raspberrypi/picotool).


#### NOTES on Atomics
On page 307 of the RP2350 Datasheet MCAUSE register CODE 7 says:
> Store/AMO access fault. A store/AMO failed a PMP check, or
encountered a downstream bus error. Also set if an AMO is attempted on a
region that does not support atomics (on RP2350, anything but SRAM).

Atomics will only work in SRAM, the kernel is located PSRAM, not SRAM.
The `lr` and `sr` atomic load and store will always return error in this region causing most code using them to behave incorrectly.
Most implementations assume `lr` and `sr` will eventually succeed.

### Use on other boards

This only works on the RP2350 RISC-V cores.

If you want to run this on other boards, please change the `PICO_BOARD` variable in `CMakeLists.txt`.
`set(PICO_BOARD sparkfun_promicro_rp2350 CACHE STRING "Board type")`

You will also need to set the psram CS pin with the `SFE_RP2350_XIP_CSI_PIN` macro in `bootloader.c`.
As of now the only psram chip tested is the `APS6404L` and any others may not work.
