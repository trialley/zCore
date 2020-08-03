make: *** No rule to make target 'debug'.  Stop.
root@ubuntu-vm:~/Desktop/zCore# cd zCore && make debug mode=release accel=1 linux=1
echo Building zCore kenel
Building zCore kenel
cargo build -Z build-std=core,alloc --target x86_64.json --release --features zircon
   Compiling kernel-hal-bare v0.1.0 (/root/Desktop/zCore/kernel-hal-bare)
   Compiling zcore v0.1.0 (/root/Desktop/zCore/zCore)
warning: Linking two modules of different data layouts: '' is 'e-m:e-p270:32:32-p271:32:32-p272:64:64-i64:64-f80:128-n8:16:32:64-S128' whereas 'zcore.dxsimi4h-cgu.3' is 'e-m:e-i64:64-f80:128-n8:16:32:64-S128'


warning: 1 warning emitted

    Finished release [optimized] target(s) in 17.88s
cd ../rboot && make build
make[1]: Entering directory '/root/Desktop/zCore/rboot'
cargo build -Z build-std=core,alloc --target x86_64-unknown-uefi --release
    Finished release [optimized] target(s) in 0.04s
make[1]: Leaving directory '/root/Desktop/zCore/rboot'
mkdir -p target/x86_64/release/esp/EFI/zCore target/x86_64/release/esp/EFI/Boot
cp ../rboot/target/x86_64-unknown-uefi/release/rboot.efi target/x86_64/release/esp/EFI/Boot/BootX64.efi
cp rboot.conf target/x86_64/release/esp/EFI/Boot/rboot.conf
cp ../prebuilt/zircon/x64/bringup.zbi target/x86_64/release/esp/EFI/zCore/fuchsia.zbi
cp target/x86_64/release/zcore target/x86_64/release/esp/EFI/zCore/zcore.elf
qemu-system-x86_64 -smp 1 -machine q35 -cpu Haswell,+smap,-check,-fsgsbase -drive if=pflash,format=raw,readonly,file=../rboot/OVMF.fd -drive format=raw,file=fat:rw:target/x86_64/release/esp -drive format=qcow2,file=target/x86_64/release/disk.qcow2,id=disk,if=none -device ich9-ahci,id=ahci -device ide-drive,drive=disk,bus=ahci.0 -serial mon:stdio -m 4G -nic none -device isa-debug-exit,iobase=0xf4,iosize=0x04 -display none -nographic -s -S
qemu-system-x86_64: -device ide-drive,drive=disk,bus=ahci.0: warning: 'ide-drive' is deprecated, please use 'ide-hd' or 'ide-cd' instead


# zCore

[![CI](https://github.com/rcore-os/zCore/workflows/CI/badge.svg?branch=master)](https://github.com/rcore-os/zCore/actions)
[![Docs](https://img.shields.io/badge/docs-alpha-blue)](https://rcore-os.github.io/zCore/zircon_object/)
[![Coverage Status](https://coveralls.io/repos/github/rcore-os/zCore/badge.svg?branch=master)](https://coveralls.io/github/rcore-os/zCore?branch=master)

Reimplement [Zircon][zircon] microkernel in safe Rust as a userspace program!

## Dev Status

🚧 Working In Progress

- 2020.04.16: Zircon console is working on zCore! 🎉

## Getting started

Environments：

* [Rust toolchain](http://rustup.rs)
* [QEMU](https://www.qemu.org)
* [Git LFS](https://git-lfs.github.com)

Clone repo and pull prebuilt fuchsia images:

```sh
git clone https://github.com/rcore-os/zCore --recursive
cd zCore
git lfs pull
```

For users in China, there's a mirror you can try:

```sh
git clone https://github.com.cnpmjs.org/rcore-os/zCore --recursive
```

Prepare Alpine Linux rootfs:

```sh
make rootfs
```

Run native Linux program (Busybox):

```sh
cargo run --release -p linux-loader /bin/busybox [args]
```

Run native Zircon program (shell):

```sh
cargo run --release -p zircon-loader prebuilt/zircon/x64
```

Run Zircon on bare-metal (zCore):

```sh
cd zCore && make run mode=release [graphic=on] [accel=1]
```

Build and run your own Zircon user programs:

```sh
# See template in zircon-user
cd zircon-user && make zbi mode=release

# Run your programs in zCore
cd zCore && make run mode=release user=1
```

To debug, set `RUST_LOG` environment variable to one of `error`, `warn`, `info`, `debug`, `trace`.

## Testing

Run Zircon official core-tests:

```sh
cd zCore && make test mode=release [accel=1] test_filter='Channel.*'
```

Run all (non-panicked) core-tests for CI:

```sh
pip3 install pexpect
cd script && python3 core-tests.py
```

Check `test-result.txt` for results.

## Components

### Overview

![](./docs/structure.svg)

[zircon]: https://fuchsia.googlesource.com/fuchsia/+/master/zircon/README.md
[kernel-objects]: https://github.com/PanQL/zircon/blob/master/docs/objects.md
[syscalls]: https://github.com/PanQL/zircon/blob/master/docs/syscalls.md

### Hardware Abstraction Layer

|                           | Bare Metal | Linux / macOS     |
| :------------------------ | ---------- | ----------------- |
| Virtual Memory Management | Page Table | Mmap              |
| Thread Management         | `executor` | `async-std::task` |
| Exception Handling        | Interrupt  | Signal            |

