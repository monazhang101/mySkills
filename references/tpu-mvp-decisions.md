# TPU Diagnostic MVP Decisions

Use this reference for the current TPU diagnostic framework direction. These project-local decisions override older generic logical-device-first guidance.

## Current Direction

The MVP is built around two priorities:

1. Keep the kernel module thin.
2. Put device discovery, policy mapping, BAR mapping, and TPU object creation behind a C++ HAL `DeviceManager`.

## Thin Kernel Module

Prefer an ultra-thin kernel module when privileged help is required:

- Allocate contiguous or DMA-safe memory.
- Expose `mmap` for user-space access.
- Return a device-visible address plus an explicit address kind such as `PHYS`, `BUS`, `IOVA`, or `FW_VA`.
- Provide IRQ request/release and event delivery only when polling is insufficient.

Do not put diagnostic semantics, SKU policy, topology interpretation, register decoding, DMA descriptor construction, or test orchestration in the kernel module.

## Device Model

Use stable HAL target names as Python/CLI-visible targets:

- `ATLAS_0`, `ATLAS_1`, and later `SSD_0`, `NIC_0`, `BMC_0`, etc.
- For the current TPU MVP, `TPUDevice` is the ATLAS parent device object and owns child module target objects for PCIe, PMU, ISI, DDP, and DMC.
- Canonical child target names are hierarchical: `ATLAS_0.PCIE_0`, `ATLAS_0.PMU_0`, `ATLAS_0.ISI_0`, `ATLAS_0.DDP_0`, and `ATLAS_0.DDP_0.DMC_0_0`.
- PCIe/DMA/PHY atomic tests are registered on `PCIeModule`, because PCIe is an explicitly modeled diagnostic module.
- SoC-named atomic tests stay registered on the parent ATLAS/TPU device only as routed operations through PMU, PCIe, or GPIO index paths.
- PMU, ISI, DDP, and DMC atomic tests are registered on their own module target objects.
- Generic helpers such as register read/write, MMIO read/write, pattern generation, and data compare are pure utility APIs, not target devices and not atomic tests.
- Ignore UART, OpenOCD, and XDE helper placement for now.

`DeviceContext` is the physical PCI device context:

- Keep `DeviceContext` generic: `bdf`, `vendor_id`, `device_id`, `mapped_bar_base`, and `bar_size`.
- Do not put module register windows such as PMU/ISI/DDP `reg_base`, `reg_offset`, or `reg_size` into `DeviceContext`.
- `DeviceContext` is filled during `DeviceManager.discover()` after PCI scan, policy match, and BAR mmap.

Register-window ownership:

- Register windows belong to the device/module object that uses them.
- `TPUDevice` should not create fake SoC register windows by default. If a SoC-named test exists, route it through PMU, PCIe, or GPIO-index mechanisms.
- `PCIeModule`, `PMUModule`, `ISIModule`, `DDPModule`, and `DMCModule` hold their own `reg_base_`, `reg_offset_`, and `reg_size_`.
- Keep `reg_size_` with `reg_base_` so `common::reg::read/write` can perform range checks.
- DDP owns three DMC controller module objects; DMC register-window attributes belong to each `DMCModule`.
- Do not add `memory_reg_base_` or unrelated memory-controller register attributes until real MC register atomic tests are identified.
- Treat device-memory or DMEM windows separately from register windows; use names such as `dev_mem_base` or `dmem_window_base` only when a real data window exists.

`DeviceManager.discover()` should:

- Scan PCI devices.
- Filter supported TPU VID/DID/revision values.
- Apply platform policy to map BDF/slot/serial to canonical names such as `ATLAS_0`.
- mmap required BAR spaces.
- Construct `TPUDevice` objects and let each TPU construct its PCIe/PMU/ISI/DDP/DMC module targets.
- Register every runnable target in one target registry.
- Return a `DeviceTree` containing ATLAS/TPU devices plus PCIe/PMU/ISI/DDP/DMC module entries. Top entries have `type=TPU`; child entries use module types such as `PCIE_MODULE` or `DMC_MODULE`.
- Keep BDF as a separate observed field. Do not duplicate BDF inside `locator`; use locator for stable placement facts such as slot, position, serial, or devnode.
- Do not expose module `reg_offset`, `reg_size`, or raw mapped BAR virtual addresses in the returned `DeviceTree`; those are internal members of the HAL module objects created during discovery.
- Be called explicitly by CLI/Python before atomic tests run; discovery is not a background side effect.

Common utility boundary:

- `common::args` parses `TestArgs`.
- `common::reg` operates on register windows and uses `reg_base + reg_offset + reg_size`.
- `common::devmem` operates on device-memory/DMEM byte ranges, not control/status registers.
- `common::pattern` only generates and compares simple payload patterns; current formats are `zero`, `random`, and `incremental`.

## HAL Atomic Tests

HAL atomic tests are low-level hardware facts or primitive operations. Do not expose suite names such as `skucheck` as HAL atomic tests.

Good atomic test names:

- `identify`
- `read_fru`
- `bar_probe`
- `register_block_probe`
- `pcie_link_status_check`
- `isi_linkup`
- `pmu_reg_read`
- `error_counter_snapshot`
- `pcie_dma_data_transfer`

Use `snake_case` for external atomic test names because those names appear in CLI, config, Python calls, logs, and reports. Keep C++ implementation functions in normal C++ style, for example `PcieDmaDataTransfer`. The registration line connects the two names:

```cpp
_add_test("pcie_dma_data_transfer",
          [this](const TestArgs& args) { return PcieDmaDataTransfer(args); });
```

Keep headers declaration-only. Put one `_add_test(...)` line per atomic test directly inside the device/module constructor body so the constructor reads like that target's test catalog. Put atomic test steps directly in the matching implementation file: `TPUDevice.cpp` for TPU/PCIe/DMA/SoC tests, `PMUModule.cpp` for PMU tests, `ISIModule.cpp` for ISI tests, and `DDPModule.cpp` for DDP tests. Do not create separate `*TestCases.cpp` files for the current pseudocode layout.

Merge equivalent operations into one atomic test and express variants as arguments:

```python
hal.run_atomic_test("ATLAS_0.PCIE_0", "pcie_dma_data_transfer", {"direction": "h2d"})
hal.run_atomic_test("ATLAS_0.PCIE_0", "pcie_dma_data_transfer", {"direction": "d2h"})
hal.run_atomic_test("ATLAS_0.PCIE_0", "pcie_dma_data_transfer", {"direction": "both"})
hal.run_atomic_test("ATLAS_0.ISI_0", "isi_linkup", {})
```

For a CLI-first MVP, prefer a generic one-shot shape before adding per-test custom parsers:

```bash
aidiag atomic --device ATLAS_0.PCIE_0 --test pcie_dma_data_transfer --arg direction=h2d --arg size_bytes=4096
```

HAL returns observed facts and metrics. Python applies policy, thresholds, and pass/fail criteria.

## Python Testcases

Python composes atomic tests into user-facing testcases:

- `skucheck`: supported-system and TPU basic configuration/version/inventory check, aligned with DGX-style expected-vs-observed policy comparison.
- `connectivity`: link and power-path sanity checks, including ISI physical presence, PCIe speed/width at required topology depths, and short power workload sustain checks.

`skucheck` should call atomic tests such as `identify`, `read_fru`, `bar_probe`, `register_block_probe`, and `error_counter_snapshot`, then compare results with SKU policy.

`connectivity` should call atomic tests such as `pcie_link_status_check`, `isi_linkup`, and short `power_sanity` or workload probes, then compare with connectivity criteria.
