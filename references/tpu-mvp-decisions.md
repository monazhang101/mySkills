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

Use top-level devices as Python-visible targets:

- `TPU_0`, `TPU_1`, and later `SSD_0`, `NIC_0`, `BMC_0`, etc.
- For TPU tests, Python targets the TPU device, not internal blocks.
- PCIe, ISI, PMU, DDP, memory controller, and similar blocks are private `TPUDevice` modules/capabilities.
- Do not expose internal TPU blocks in the logical tree unless they need independent target selection, locking, reset, firmware update, lifecycle ownership, or report identity.

`DeviceManager.discover()` should:

- Scan PCI devices.
- Filter supported TPU VID/DID/revision values.
- Apply platform policy to map BDF/slot/serial to canonical names such as `TPU_0`.
- mmap required BAR spaces.
- Construct `TPUDevice` objects with internal block helpers.
- Return a device tree containing top-level target devices and relevant system topology, not every internal register block.

## HAL Atomic Tests

HAL atomic tests are low-level hardware facts or primitive operations. Do not expose suite names such as `skucheck` as HAL atomic tests.

Good atomic test names:

- `identify`
- `read_fru`
- `bar_probe`
- `register_block_probe`
- `pcie_link_status_check`
- `isi_link_up`
- `pmu_read_sensor`
- `memory_get_inventory`
- `error_counter_snapshot`
- `pcie_dma_data_transfer`

Merge equivalent operations into one atomic test and express variants as arguments:

```python
hal.run_atomic_test("pcie_dma_data_transfer", "TPU_0", {"direction": "h2d"})
hal.run_atomic_test("pcie_dma_data_transfer", "TPU_0", {"direction": "d2h"})
hal.run_atomic_test("pcie_dma_data_transfer", "TPU_0", {"direction": "both"})
hal.run_atomic_test("isi_link_up", "TPU_0", {"links": "all"})
```

HAL returns observed facts and metrics. Python applies policy, thresholds, and pass/fail criteria.

## Python Testcases

Python composes atomic tests into user-facing testcases:

- `skucheck`: supported-system and TPU basic configuration/version/inventory check, aligned with DGX-style expected-vs-observed policy comparison.
- `connectivity`: link and power-path sanity checks, including ISI physical presence, PCIe speed/width at required topology depths, and short power workload sustain checks.

`skucheck` should call atomic tests such as `identify`, `read_fru`, `bar_probe`, `register_block_probe`, `memory_get_inventory`, and `error_counter_snapshot`, then compare results with SKU policy.

`connectivity` should call atomic tests such as `pcie_link_status_check`, `isi_link_up`, and short `power_sanity` or workload probes, then compare with connectivity criteria.
