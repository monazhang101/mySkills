---
name: ai-chip-diag-architect
description: Design and iteratively refine AI chip hardware diagnostic and test frameworks. Use when Codex needs to architect or review systems with kernel module access, C++ HAL layers, Python orchestration, external tool adapters such as FIO/stress tools, device/topology management, thread/process scheduling, platform policies, telemetry, diagnostics, burn-in, manufacturing tests, or DGX-like hardware test framework decomposition.
---

# AI Chip Diag Architect

Use this skill to design or evolve a DGX-like but cleaner AI chip diagnostic framework. Keep the framework layered, contract-driven, and testable without hardware where possible.

## Operating Principles

- Start from contracts before implementation: define `Device`, `Topology`, `TestSpec`, `Policy`, `TestResult`, and `Artifact` schemas first.
- Keep kernel modules narrow: prefer a minimal DMA-safe memory allocation/mapping provider before adding BAR, MMIO, config space, IRQ, reset, telemetry, or high-level test logic.
- Put hardware semantics in C++ HAL: hide ioctl, sysfs, vendor SDK, register offsets, chip stepping differences, and topology quirks from Python.
- Keep Python as orchestration: scheduling, policy loading, external tool execution, report generation, retry, timeout, and workflow composition.
- Use platform policy files for SKU/topology/threshold/concurrency differences instead of scattering platform conditionals through tests.
- Prefer structured APIs and structured outputs: JSON/YAML/protobuf/result objects over ad hoc log scraping where tools allow it.
- Make every layer mockable: provide fake HAL and dry-run tool adapters early.

## Default Architecture

Use this decomposition unless the repo or user gives stronger constraints:

```text
kernel module
  -> default MVP: versioned ioctl/mmap UAPI for DMA-safe buffer allocation and IOMMU mapping
  -> optional later: IRQ/event, reset, telemetry, or kernel-mediated BAR/config access

C++ HAL
  -> device discovery, topology, BAR/MMIO/config access, DMA engine programming, polling
  -> telemetry, low-level diagnostic primitives
  -> pybind11/C ABI/gRPC bindings as needed

Python framework
  -> CLI, testcase registry, scheduler, resource manager, process manager
  -> policy engine, external tool adapters, monitor service, result store

Specs and policies
  -> run specs, platform policies, thresholds, expected inventory, concurrency rules

Packaging and release
  -> build HAL/Python/kernel artifacts, install runtime layout, DKMS/no-kmod modes, deb packaging
```

## Layer Boundaries

- Kernel module API to HAL: use stable UAPI structs with `size`, `version`, `flags`, and reserved fields. For the thin-kmod MVP, expose only `GET_VERSION`, device attach/open if needed, `DMA_ALLOC`, `DMA_FREE`, `mmap`, and optional cache sync. Return a device-visible DMA address/IOVA, not a guessed CPU physical address.
- HAL low-level access: keep BAR/MMIO access, PCI config space access, DMA descriptor programming, doorbell writes, and polling completion in C++ HAL when user-space access is safe through VFIO, sysfs, vendor SDK, or a controlled mapping.
- HAL API to Python: expose coarse operations such as `discover()`, `get_topology()`, `read_telemetry(device)`, `run_primitive(name, device, options)`, `reset_device(device, mode)`, and `subscribe_events(device)`.
- Python to external tools: use a `ToolAdapter` abstraction that owns argv construction, timeout, environment, log paths, process cleanup, and parsers.
- Policy to scheduler: express resource locks and concurrency rules declaratively, for example `chip:{device}`, `pcie_domain:{domain}`, `bmc`, `numa:{node}`.
- Packaging to runtime: install CLI, HAL shared library, Python package, policies, specs, schemas, tools, udev/systemd/logrotate files, and optional DKMS kernel module in predictable filesystem locations.

## Design Workflow

1. Identify target devices: accelerators, HBM, switches, PCIe links, NVMe, NIC/IB, CPU, DIMM, fans, PSU, BMC/HMC, sensors.
2. Define logical naming and topology: map BDF/sysfs/vendor identifiers to stable names such as `CHIP_0`, `OAM_1`, `NVME_3`, `NIC_1`.
3. Define schemas and result model before writing tests.
4. Build a minimal Python runner with fake HAL: inventory, policy validation, one testcase, logs, and result JSON.
5. Add external tool tests first when useful, especially FIO/NVMe, because they validate orchestration without custom kernel work.
6. Add C++ HAL discovery and telemetry next.
7. Add the thinnest kernel module only when DMA-safe memory/IOMMU mapping, privileged mapping, or lifecycle cleanup cannot be safely obtained from VFIO, vendor SDKs, or existing drivers.
8. Add packaging early enough that the CLI, HAL, policies, specs, and optional kmod can be installed and smoke-tested as a deb.
9. Expand to stress, burn-in, thermal, multi-device, and multi-node tests once resource scheduling is solid.

## Recommended First Milestones

- Milestone 0: schemas for `Device`, `Topology`, `Policy`, `TestSpec`, `TestResult`.
- Milestone 1: Python runner with fake HAL, dry-run mode, structured report.
- Milestone 2: real `nvme_fio` testcase via `ToolAdapter`.
- Milestone 3: C++ HAL `discover()` and `read_telemetry()`, exposed to Python.
- Milestone 4: thin kernel module or VFIO path for DMA-safe buffer allocation/mapping and `mmap`.
- Milestone 5: resource-aware scheduler and monitor service.
- Milestone 6: deb packaging with no-kmod and DKMS-kmod modes, runtime layout, udev, and install smoke test.
- Milestone 7: chip memory, DMA, PCIe, thermal, reset, and burn-in suites.

## Reference Loading

Read `references/framework-blueprint.md` when the user asks for concrete directory layout, interface examples, policy schema, testcase contracts, or a fuller DGX-inspired architecture.

Read `references/hal-cpp-headers.md` when the user asks to make the C++ HAL API concrete, create header files, review HAL interface boundaries, or design Python bindings around the HAL.

Read `references/thin-kmod-boundary.md` when the user asks whether the kernel module can be smaller, what belongs in ko vs HAL, or how DMA, IOMMU, BAR/MMIO, and PCI config space should be split.

Read `references/packaging-deb.md` when the user asks how to package the diagnostic tool as a deb, design install paths, DKMS/no-kmod modes, udev/systemd integration, postinst/prerm behavior, or release artifacts.
