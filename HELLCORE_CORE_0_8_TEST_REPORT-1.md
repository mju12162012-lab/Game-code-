# HELLCORE CORE 0.8 — Build and validation report

Target: x86-64 freestanding C++20 kernel, native UEFI PE32+ loader.

## Implemented in source

- Retained the CORE 0.6 AHCI one-sector DMA read/write prototype and E1000 descriptor-ring TX/RX prototype.
- Expanded xHCI beyond halt/reset and Enable Slot probing.
- Added command/event ring setup, ERST, cycle-state handling, event polling, DCBAA, per-slot input/output contexts, and EP0 transfer rings.
- Added connected-port reset, Enable Slot, Address Device, and EP0 control-IN transfer handling.
- Added GET_DESCRIPTOR requests for the USB device descriptor and configuration descriptor header.
- Added Evaluate Context to update EP0 maximum packet size from `bMaxPacketSize0`.
- Added serial counters for reset, slot, addressing, descriptor, and enumeration stages.
- Updated QEMU validation marker to CORE 0.8.

## Build and host checks actually run

Command:

```sh
make clean all efi esp check
nm -u build/kernel.elf
```

Results:

- Freestanding x86-64 kernel compilation and static ELF64 link: PASS.
- Native UEFI loader compilation and PE32+ link: PASS.
- EFI system partition directory generation: PASS.
- ACPI parser tests: PASS.
- PCI resource allocator and PCI class tests: PASS.
- PMM host tests: PASS.
- ELF and UEFI PE32+ format/ABI checks: PASS.
- `nm -u build/kernel.elf`: no undefined symbols.

## QEMU/UEFI runtime validation

**BLOCKED — NOT RUN.** Executed:

```sh
./scripts/validate-boot.sh
```

The clean build and host checks passed, after which the script exited with status 2:

```text
BOOT BLOCKED: qemu-system-x86_64 is not installed. Install qemu-system-x86 and ovmf, then rerun.
```

No QEMU process ran, no OVMF firmware was loaded, and no serial boot log was produced. No firmware execution, kernel execution, AHCI operation, E1000 packet transfer, or xHCI enumeration is claimed.

## Driver limitations and next work

### xHCI

- The new enumeration path is compile-verified only; hardware behavior remains unverified.
- It fetches the device descriptor and the 9-byte configuration descriptor header. It does not fetch/parse the complete configuration tree or issue SET_CONFIGURATION.
- No hub support, HID keyboard/mouse, interrupt endpoints, hotplug, interrupt-driven event processing, slot cleanup, or comprehensive error recovery.
- DMA structures must be identity-mapped and below 4 GiB. There is no DMA allocator or IOMMU support.

### AHCI

- Synchronous single-sector DMA prototype only.
- No multi-sector batching, NCQ, interrupt-driven completion, timeout recovery, hotplug, block layer, or filesystem.

### E1000

- Basic descriptor-ring TX/RX prototype only.
- No network stack, interrupt path, comprehensive error recovery, or broad controller revision support.

### Platform

- PCI BAR reassignment is not automatic; validated host-bridge apertures are not discovered.
- Bootstrap VMM and driver DMA/MMIO remain limited below 4 GiB.
- No physical hardware test was performed.

## Required next validation gates

1. Install QEMU x86-64 and OVMF, run `scripts/validate-boot.sh`, and retain `build/logs/serial.log`.
2. Validate xHCI command completions, port reset, Address Device, and descriptor transfers against the QEMU `qemu-xhci` plus USB keyboard.
3. Extend xHCI to parse full configuration/interface/endpoint descriptors, issue SET_CONFIGURATION, configure interrupt-IN endpoints, and implement HID keyboard/mouse.
4. Test AHCI sector write/readback only against the disposable QEMU test disk.
5. Test E1000 TX/RX with QEMU packet capture or a loopback peer.

A successful build is not a successful boot. CORE 0.8 is a build-verified development milestone, not a runtime-verified hardware release.
