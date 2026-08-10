---
name: ai-chip-diag-architect
description: Design and iteratively refine AI chip hardware diagnostic and test frameworks. Use when Codex needs to architect or review systems with kernel module access, UMD/user-mode driver boundaries, C++ HAL layers, Python orchestration, external tool adapters such as FIO/stress tools, device/topology management, thread/process scheduling, platform policies, telemetry, diagnostics, burn-in, manufacturing tests, or hardware test framework decomposition.
---

# AI Chip Diag Architect

Use this skill to design or evolve a clean AI chip diagnostic framework. Keep the framework layered, contract-driven, and testable without hardware where possible.

## Operating Principles

- Start from contracts before implementation: define `Device`, `Topology`, `TestSpec`, `Policy`, `TestResult`, and `Artifact` schemas first.
- Keep kernel modules narrow: prefer the thinnest viable provider before adding BAR, MMIO, config space, reset, telemetry, or high-level test logic. In bring-up designs, this may be only contiguous physical memory allocation/mmap plus IRQ request/event delivery, with DMA programming intentionally left to user mode.
- Treat UMD (User Mode Driver) as a low-level user-space driver layer below HAL when present. UMD owns firmware descriptor formats, queue/ring management, doorbells, DMA submit mechanics, completion attribution, polling/IRQ wait, timeout, cancel, and drain. HAL may wrap UMD, but Python should not call UMD descriptor APIs directly.
- Put diagnostic semantics in C++ HAL: hide UMD APIs, ioctl, sysfs, vendor SDK, register offsets, chip stepping differences, and topology quirks from Python.
- Keep Python as orchestration: scheduling, policy loading, external tool execution, report generation, retry, timeout, and workflow composition.
- Use platform policy files for SKU/topology/threshold/concurrency differences instead of scattering platform conditionals through tests.
- Prefer a TPU-target-first MVP for accelerator diagnostics: expose top-level target devices such as `TPU_0`, `TPU_1`, SSDs, NICs, and BMC/system devices. Keep PCIe, ISI, PMU, DDP, memory-controller, and similar TPU-internal blocks as private HAL modules/capabilities unless they must be independently selected, locked, reset, upgraded, or reported.
- Treat the top-level device name as the canonical test target. Policy files should pin stable names such as `TPU_0` or `TPU_1` to real hardware using locator fields such as `pci_bdf`, `devnode`, `sysfs_path`, `serial`, `slot`, or BMC path. HAL discovery should return observed top-level devices and topology facts; CLI targets, testcase config, leases, HAL calls, and result reports should use the canonical name, not raw BDFs.
- Model resource occupancy explicitly, but keep the MVP simple: most per-TPU tests lock the top-level TPU device. Add separate domain resources such as fabric, PCIe domain, reset domain, power domain, or telemetry domain only when tests truly share or contend for them.
- Treat concurrent submit paths as first-class resources: tests should acquire device/resource leases before touching hardware, while UMD/HAL owns descriptor allocation, ring updates, doorbells, polling, completion attribution, cancel, and drain.
- For strongly coupled multi-device diagnostics such as 8-chip stress, fabric stress, collective bandwidth, thermal stress, or synchronized power stress, Python should acquire the full top-level device and shared-domain lease set and call one coarse HAL atomic test. HAL/UMD should own internal multi-device workers, start barriers, submit/completion attribution, timeout, cancel, drain, and cleanup. Python fan-out is acceptable for independent per-device tests and external tool adapters.
- For the first Python MVP, prefer a compact config-driven runner instead of a fully decomposed framework. A reasonable MVP is `main.py`, `config.py`, `runner.py`, and a `tests/` testcase package: entrypoint, test config parser, sequential runner/report writer, cross-process lease helper, `Runner.run_tool(...)`, and thin testcase modules. Python testcases such as `skucheck` and `connectivity` compose HAL atomic tests and compare results with policy/criteria; they are not HAL atomic test names. Split out `scheduler.py`, `lease_manager.py`, `tool_adapter.py`, `result_store.py`, or richer testcase framework files only when complexity proves it is needed.
- Prefer structured APIs and structured outputs: JSON/YAML/protobuf/result objects over ad hoc log scraping where tools allow it.
- Make every layer mockable: provide fake HAL and dry-run tool adapters early.

## Default Architecture

Use this decomposition unless the repo or user gives stronger constraints:

```text
kernel module
  -> default MVP: versioned ioctl/mmap UAPI for the thinnest required privileged services
  -> common ultra-thin mode: contiguous physical memory allocation/mmap plus IRQ request/event delivery
  -> optional later: IOMMU mapping, reset, telemetry, or kernel-mediated BAR/config access

UMD / user-mode driver, when platform provides one
  -> firmware descriptor ownership, command/completion rings, BAR/MMIO doorbells, DMA submit mechanics
  -> queue lifecycle, polling/IRQ wait, timeout, cancel, drain, and crash-aware cleanup

C++ HAL
  -> DeviceManager owns PCI scan, policy locator matching, BAR mmap, top-level TPUDevice creation, topology facts, telemetry, low-level diagnostic atomic tests
  -> wraps UMD when present
  -> directly owns BAR/MMIO/config access, DMA engine programming, descriptor submit, and polling only when no separate UMD exists
  -> TPUDevice owns internal modules such as PCIe, ISI, PMU, DDP, and memory controller without exposing them as Python-visible devices
  -> pybind11/C ABI/gRPC bindings as needed

Python framework
  -> later target: CLI, testcase registry, top-level device registry, lease-aware Scheduler facade, enhanced cross-process LeaseManager, process manager
  -> complete target: policy engine, external tool adapters, monitor service, result store
  -> Python MVP: main.py entrypoint, config.py parser, runner.py cross-process lease + run_tool + sequential execution + reports, tests/ thin testcase modules

Specs and policies
  -> run specs, platform policies, thresholds, expected inventory, resource locks, concurrency rules

Packaging and release
  -> build HAL/Python/kernel artifacts, install runtime layout, DKMS/no-kmod modes, deb packaging
```

## Layer Boundaries

- Kernel module API to UMD/HAL: use stable UAPI structs with `size`, `version`, `flags`, and reserved fields. For an ultra-thin bring-up kmod, expose only `GET_VERSION`, contiguous memory allocation/free, `mmap`, and IRQ request/wait/release. If IOMMU is intentionally out of scope, return a `device_addr` plus `addr_kind` such as physical, bus, IOVA, or FW virtual address; UMD must fill firmware descriptors from `device_addr`, never from a user-space CPU virtual pointer.
- UMD API to HAL: expose session-scoped operations such as `open_device()`, `alloc_memory()`, `create_queue()`, `submit_descriptor()`, `wait_completion()`, `cancel()`, `drain()`, and `close()`. Keep firmware descriptor layout, ring indices, doorbell offsets, MSI/MSI-X details, and completion parsing inside UMD.
- HAL low-level access: if no separate UMD exists, keep BAR/MMIO access, PCI config space access, DMA descriptor programming, doorbell writes, and polling completion in C++ HAL when user-space access is safe through VFIO, sysfs, vendor SDK, or a controlled mapping. If UMD exists, HAL should call UMD and keep descriptor/register details out of HAL's public Python-facing API.
- HAL API to Python: expose operations such as `discover()`, `get_topology()`, `read_telemetry(device)`, `run_atomic_test(test_name, device, args)`, `run_group_atomic_test(test_name, devices, args)` when needed, `reset_device(device, mode)`, and `subscribe_events(device)`. HAL atomic tests are registered on the target device, so do not prefix test names with the device type. Prefer hardware-fact or primitive-operation names such as `identify`, `read_fru`, `bar_probe`, `register_block_probe`, `pcie_link_status_check`, `isi_link_up`, `pmu_read_sensor`, `memory_get_inventory`, `error_counter_snapshot`, or `pcie_dma_data_transfer`. Do not expose suite names such as `skucheck` as HAL atomic tests.
- HAL submit API: expose session-scoped `submit_job()`, `wait_job()`, `cancel_job()`, and `drain_queue()` operations when tests program hardware work queues. HAL can implement these directly or delegate to UMD. Attach `test_id`, `session_id`, `device_id`, `queue_id`, descriptor identity, timeout, and artifact paths to every job.
- HAL multi-device atomic API: expose coarse multi-device operations for tightly synchronized diagnostics, for example `run_group_atomic_test("tpustress", devices=[...], args={...})`. The active lease token must cover every affected chip and shared domain. Keep per-device workers, start barriers, queue submit mechanics, fail-fast policy, timeout, cancel, drain, and cleanup inside HAL/UMD for these tests.
- Python to external tools: for the MVP, keep this in `Runner.run_tool(...)`; it owns argv construction, timeout, environment, log paths, process cleanup, and parsers. Split to a `ToolAdapter` abstraction later when external tool wrappers become repetitive.
- Policy to runner: for the default MVP, express locks as top-level target device IDs with `shared` or `exclusive` modes, for example `TPU_0`, `TPU_1`, `SSD_0`, `NIC_0`, or `BMC_0`. Add domain locks such as `FABRIC_0`, `PCIE_DOMAIN_0`, `RESET_DOMAIN_0`, `POWER_DOMAIN_0`, or `TELEMETRY_0` only when a testcase genuinely touches shared resources.
- Runner to HAL: acquire top-level device/resource leases before submit; reject or wait on conflicting tests before they reach BAR/MMIO, reset, DMA engine, ringbuffer, or descriptor operations. For MVP, `runner.py` expands locks, uses a simple cross-process lease gate, wraps testcase execution, releases leases, and records lease metadata. Keep ring and descriptor arbitration in UMD/HAL, and keep the default kernel module outside this path except for the thinnest required memory/IRQ services.
- Packaging to runtime: install CLI, HAL shared library, Python package, policies, specs, schemas, tools, udev/systemd/logrotate files, and optional DKMS kernel module in predictable filesystem locations.

## Design Workflow

1. Identify target devices: accelerators, HBM, switches, PCIe links, NVMe, NIC/IB, CPU, DIMM, fans, PSU, BMC/HMC, sensors.
2. Define top-level target devices and topology: map physical devices such as `TPU_0`, `TPU_1`, `SSD_0`, `NIC_1`, or `BMC_0` to stable canonical `name` values; use `locator` fields to pin each physical device to BDF/devnode/sysfs/serial/slot/BMC access paths. Keep TPU-internal blocks such as PCIe, ISI, PMU, DDP, and memory controller inside `TPUDevice` unless they need independent target/lock/lifecycle semantics.
3. Define lightweight lock semantics: each testcase declares `shared` and `exclusive` locks over top-level devices and required shared domains; use lock groups for repeated sets such as all chips under one fabric. Implement a simple cross-process lease gate in `runner.py` so multiple CLI instances may run concurrently when leases do not conflict. Add a formal `LeaseManager`, `Scheduler`, or full resource graph only when lock expansion becomes too repetitive or imprecise.
4. Define schemas and result model before writing tests.
5. Build a minimal Python runner with fake HAL using the compact MVP layout when appropriate: `main.py`, `config.py`, `runner.py`, and `tests/`. Validate inventory, config, lease expansion, one HAL atomic testcase, one external-tool testcase, logs, and result JSON.
6. Add external tool tests first when useful, especially FIO/NVMe, because they validate orchestration without custom kernel work.
7. Add C++ HAL discovery and telemetry next.
8. Add UMD or HAL submit queue manager when tests program hardware queues. Centralize descriptor allocation, ring tail/head updates, doorbells, completion attribution, timeout, cancel, drain, reset coordination, and process-crash cleanup.
9. Add the thinnest kernel module only when contiguous memory allocation/mmap, IRQ request/event delivery, IOMMU mapping, privileged mapping, or lifecycle cleanup cannot be safely obtained from VFIO, vendor SDKs, or existing drivers.
10. Add packaging early enough that the CLI, HAL, policies, specs, and optional kmod can be installed and smoke-tested as a deb.
11. Expand to stress, burn-in, thermal, multi-device, and multi-node tests once resource scheduling is solid.

## MVP And Future Ideas

- MVP: schemas for top-level `Device`, `Topology`, `Policy`, `TestSpec`, and `TestResult`; compact Python runner with fake HAL, DeviceManager-backed discovery, policy locator matching, config-driven sequence execution, `tests/` testcase modules, simple cross-process lease helper, `Runner.run_tool(...)`, dry-run mode, and structured report.
- Future idea: real `nvme_fio` testcase via `Runner.run_tool(...)`; split out `ToolAdapter` later only when external tool wrappers become repetitive.
- Current HAL direction: implement C++ `DeviceManager.discover()` first. It scans PCI devices, applies policy to map BDF/slot/serial to canonical TPU names, mmaps BAR space, creates `TPUDevice` instances, and exposes a Python binding. Then expose a small `run_atomic_test(...)` catalog. Reserve `run_group_atomic_test(...)` for synchronized multi-device diagnostics when needed.
- Future idea: ultra-thin kernel module or VFIO/existing-driver path for contiguous memory allocation/mmap and optional IRQ event delivery. Preserve `device_addr`/`addr_kind` even if IOMMU is disabled for the MVP.
- Future idea: enhance the lease implementation or split a formal `Scheduler` / `LeaseManager` facade out of `runner.py` if queueing, priority, heartbeats, stale cleanup, or richer concurrency requires it.
- Future idea: UMD/HAL submit queue manager for hardware queue tests: session ownership, descriptor allocator, ringbuffer arbitration, doorbell, completion attribution, drain/cancel/reset, and cleanup.
- Future idea: deb packaging with no-kmod and DKMS-kmod modes, runtime layout, udev, and install smoke test.
- Future idea: chip memory, DMA, PCIe, thermal, reset, burn-in, multi-device, and multi-node suites.

## Reference Loading

Read `references/tpu-mvp-decisions.md` when the user asks about the current TPU diagnostic MVP, target-device model, DeviceManager responsibilities, HAL atomic test scope, `skucheck`, `connectivity`, or why PCIe/ISI/PMU/DDP are not exposed as logical tree nodes.

Read `references/python-framework-mvp.md` when the user asks to simplify the Python layer, design the MVP Python framework, decide whether CLI entry logic and runner logic should be merged, place testcase wrappers, define HAL `run_atomic_test` vs Python `run_tool`, or design CLI/testcase/sequence execution.

Read `references/action-items.md` when the user asks what design or implementation details remain to be finalized, wants to track follow-up tasks across Python framework, HAL, UMD, kernel, packaging, or needs to revisit config schema, HAL atomic test catalog, result/error model, lease semantics, or the first connectivity testcase template.

Read `references/resource-concurrency-model.md` when the user asks about device/resource leases, device occupancy, concurrent tests, resource conflicts, shared physical resources, submit queues, ringbuffers, descriptors, scheduler policy, or concurrency models. Apply the TPU MVP override from `references/tpu-mvp-decisions.md`: internal TPU blocks are not logical tree nodes by default.

Read `references/thin-kmod-boundary.md` when the user asks whether the kernel module can be smaller, what belongs in ko vs HAL, or how DMA, IOMMU, BAR/MMIO, and PCI config space should be split.

Read `references/umd-hal-boundary.md` when the user asks about UMD, user-mode drivers, firmware descriptors, doorbells, command/completion rings, DMA responsibility moving to user space, IRQ wait paths, or whether UMD can replace HAL.

Read `references/packaging-deb.md` when the user asks how to package the diagnostic tool as a deb, design install paths, DKMS/no-kmod modes, udev/systemd integration, postinst/prerm behavior, or release artifacts.
