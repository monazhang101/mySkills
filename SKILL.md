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
- Use a session-scoped HAL resource boundary. Prefer `DeviceManager` owning one `HalSession` for discovery, BAR/MMIO mappings, DMA buffers, future work queues, and HAL backend selection (`iHal`/`pHal`). Pass the session into atomic tests through `TestInfo` when a test needs low-level resources; keep Python and testcase config from passing raw CPU pointers, DMA addresses, queue descriptors, or backend handles.
- Keep Python as orchestration: scheduling, policy loading, external tool execution, report generation, retry, timeout, and workflow composition.
- Use platform policy files for SKU/topology/threshold/concurrency differences instead of scattering platform conditionals through tests.
- Prefer an ATLAS-device-first MVP for accelerator diagnostics: expose each physical TPU card as a canonical device such as `ATLAS_0`, `ATLAS_1`, `ATLAS_2`, or `ATLAS_3`. The top device `name` should be `ATLAS_n`, while its `type` remains `TPU`. PCIe, PMU, ISI, DDP, and DMC are diagnostic modules owned by the ATLAS/TPU parent object.
- Treat the top-level target name as the canonical test target. Policy files should pin stable names such as `ATLAS_0` or `ATLAS_1` to real hardware using locator fields such as `slot`, `position`, `devnode`, `sysfs_path`, or `serial`; BDF is kept as a separate observed field and should not be duplicated inside `locator`. HAL discovery should return observed top-level targets and topology facts; CLI targets, testcase config, HAL calls, and result reports should use the canonical target name, not raw BDFs.
- MVP concurrency decision: do not require Python/testcase code to acquire explicit device/resource leases. Use object mutexes first; in the C++ MVP, name the per-target `BaseDevice::run_atomic_test()` lock `atomic_test_mutex_` so it is not confused with a whole-TPU device lock. Keep HQC/FW queue management dormant until the FW/HAL ownership boundary is explicitly confirmed by the user or project owner.
- Treat HQC command submission as a likely future concurrency boundary when internal tests are confirmed to use HQC commands. Keep `CmdQueueMgm` available as a dormant core abstraction, but do not wire it into `TPUDevice` or child modules until the FW/HAL ownership boundary is confirmed. Once real commands are in scope, it can own descriptor allocation, ring head/tail updates, FW notification, completion attribution, timeout, cancel, and drain.
- Use object-level mutexes to keep tests on the same `TPUDevice`, `PCIeModule`, `PMUModule`, `ISIModule`, `DDPModule`, or `DMCModule` object sequential. In code, prefer `atomic_test_mutex_` for this per-target lock; reserve device/per-TPU names for locks that serialize all modules under one physical TPU. Keep session-level resource locks inside HAL for shared mappings, DMA allocation/free, PMU IPC/mailboxes, and future work queues. If two module objects later share one HQC path, they may submit concurrently only through the confirmed FW queue/UMD layer or a reconnected `CmdQueueMgm`; add a parent TPU or shared-executor mutex only for non-HQC paths or multi-command sequences that must not interleave.
- TODO: review every non-HQC path before enabling broad parallel execution. Direct BAR/MMIO register access, PMU IPC/mailboxes, VFIO interrupt setup, reset/link retrain, GPIO-index paths, device-memory allocation/free, and multi-command stateful sequences may need HAL-internal serialization or quiesce/drain sequencing.
- For strongly coupled multi-device diagnostics such as 8-chip stress, fabric stress, collective bandwidth, thermal stress, or synchronized power stress, Python should call one coarse HAL atomic test. HAL/UMD should own internal multi-device workers, start barriers, submit/completion attribution, timeout, cancel, drain, and cleanup. Python fan-out is acceptable for independent per-device tests and external tool adapters.
- For the first Python MVP, prefer a compact config-driven runner instead of a fully decomposed framework. A reasonable MVP is `main.py`, `config.py`, `runner.py`, and a `tests/` testcase package: entrypoint, test config parser, sequential runner/report writer, `Runner.run_tool(...)`, and thin testcase modules. Python testcases such as `skucheck` and `connectivity` compose HAL atomic tests and compare results with policy/criteria; they are not HAL atomic test names. Split out `tool_adapter.py`, `result_store.py`, or richer testcase framework files only when complexity proves it is needed.
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
  -> DeviceManager owns PCI scan, policy locator matching, BAR mmap, top-level TPUDevice/ATLAS device creation, DeviceTree topology facts, telemetry, low-level diagnostic atomic tests
  -> DeviceManager owns one HalSession; HalSession owns backend choice, mapping lifetimes, DMA-buffer allocation/free, and future queue/session state
  -> wraps UMD when present
  -> directly owns BAR/MMIO/config access, DMA engine programming, descriptor submit, and polling only when no separate UMD exists
  -> TPUDevice represents the ATLAS parent device and owns PCIe, PMU, ISI[8], and DDP[4] module objects; each DDP owns DMC[3] controller module objects. PCIe/DMA/PHY tests live on PCIeModule. SoC-named tests stay on the parent device only as routed operations through PMU, PCIe, or GPIO index paths; do not create a fake SoC register window or SoC target by default.
  -> pybind11/C ABI/gRPC bindings as needed

Python framework
  -> later target: CLI, testcase registry, top-level device registry, process manager
  -> complete target: policy engine, external tool adapters, monitor service, result store
  -> Python MVP: main.py entrypoint, config.py parser, runner.py run_tool + sequential execution + reports, tests/ thin testcase modules

Specs and policies
  -> run specs, platform policies, thresholds, expected inventory, concurrency hints when needed

Packaging and release
  -> build HAL/Python/kernel artifacts, install runtime layout, DKMS/no-kmod modes, deb packaging
```

## Layer Boundaries

- Kernel module API to UMD/HAL: use stable UAPI structs with `size`, `version`, `flags`, and reserved fields. For an ultra-thin bring-up kmod, expose only `GET_VERSION`, contiguous memory allocation/free, `mmap`, and IRQ request/wait/release. If IOMMU is intentionally out of scope, return a `device_addr` plus `addr_kind` such as physical, bus, IOVA, or FW virtual address; UMD must fill firmware descriptors from `device_addr`, never from a user-space CPU virtual pointer.
- UMD API to HAL: expose session-scoped operations such as `open_device()`, `alloc_memory()`, `create_queue()`, `submit_descriptor()`, `wait_completion()`, `cancel()`, `drain()`, and `close()`. Keep firmware descriptor layout, ring indices, doorbell offsets, MSI/MSI-X details, and completion parsing inside UMD.
- HAL low-level access: if no separate UMD exists, keep BAR/MMIO access, PCI config space access, DMA descriptor programming, doorbell writes, and polling completion in C++ HAL when user-space access is safe through VFIO, sysfs, vendor SDK, or a controlled mapping. If UMD exists, HAL should call UMD and keep descriptor/register details out of HAL's public Python-facing API.
- HAL session and DMA ownership: model low-level resources as session-scoped C++ objects. `HalSession` should expose narrow operations such as `scan_pci_devices()`, `mmap_bar_space()`, `alloc_dma_buffer()`, `submit_job()`/`wait_job()` later, and `clear()`/`reset()`. `DmaBuffer` should be move-only and RAII-managed, carrying the CPU mapping, size, device-visible address, backend handle, and `addr_kind` (`physical`, `bus`, `iova`, `phal_handle`, etc.). Free buffers through the creating `HalSession`; do not let Python or tests fabricate device addresses or release handles directly.
- HAL API to Python: expose operations such as `discover()`, `get_topology()`, `read_telemetry(device)`, `run_atomic_test(target_name, test_name, args)`, `run_group_atomic_test(test_name, devices, args)` when needed, `reset_device(device, mode)`, and `subscribe_events(device)`. `discover()` should produce a `DeviceTree` result: top-level ATLAS entries are devices, and PCIe/PMU/ISI/DDP/DMC children are modules. HAL atomic tests are registered on the target object, so do not prefix test names with the target type when the target already scopes it. Prefer hardware-fact or primitive-operation names such as `identify`, `read_fru`, `bar_probe`, `register_block_probe`, `pcie_link_status_check`, `isi_linkup`, `pmu_reg_read`, `dmc_status_check`, `error_counter_snapshot`, or `pcie_dma_data_transfer`. Do not expose suite names such as `skucheck` as HAL atomic tests.
- HAL submit API: expose session-scoped `submit_job()`, `wait_job()`, `cancel_job()`, and `drain_queue()` operations when tests program hardware work queues. HAL can implement these directly or delegate to UMD. Attach `test_id`, `session_id`, `device_id`, `queue_id`, descriptor identity, timeout, and artifact paths to every job. Keep queue mutexes and completion attribution inside HAL/UMD/`CmdQueueMgm`; `atomic_test_mutex_` serializes same-target atomic test entry, while queue locks serialize shared ring/head-tail/doorbell state.
- HAL multi-device atomic API: expose coarse multi-device operations for tightly synchronized diagnostics, for example `run_group_atomic_test("tpustress", devices=[...], args={...})`. Keep per-device workers, start barriers, queue submit mechanics, fail-fast policy, timeout, cancel, drain, and cleanup inside HAL/UMD for these tests.
- Python to external tools: for the MVP, keep this in `Runner.run_tool(...)`; it owns argv construction, timeout, environment, log paths, process cleanup, and parsers. Split to a `ToolAdapter` abstraction later when external tool wrappers become repetitive.
- Policy to runner: for the default MVP, keep policy focused on platform mapping, expected inventory, thresholds, and testcase options. Use canonical target paths such as `ATLAS_0.PCIE_0`, `ATLAS_0.PMU_0`, `ATLAS_0.DDP_2.DMC_2_1`, and `ATLAS_0.ISI_7`. Do not require user-facing resource lock declarations.
- Runner to HAL: call HAL atomic tests directly. Keep same-target atomic test serialization in the current framework through `BaseDevice::atomic_test_mutex_`. Keep HQC ring/descriptor arbitration, PMU IPC arbitration when present, completion attribution, timeout, cancel, drain, and reset coordination inside the confirmed FW interface layer, UMD, or a later reconnected `CmdQueueMgm`. Keep the default kernel module outside this path except for the thinnest required memory/IRQ services.
- Packaging to runtime: install CLI, HAL shared library, Python package, policies, specs, schemas, tools, udev/systemd/logrotate files, and optional DKMS kernel module in predictable filesystem locations.

## Design Workflow

1. Identify target devices: accelerators, HBM, switches, PCIe links, NVMe, NIC/IB, CPU, DIMM, fans, PSU, BMC/HMC, sensors.
2. Define top-level devices and topology: map physical TPU cards to stable canonical names such as `ATLAS_0` through `ATLAS_3`, with `type=TPU` and separate observed fields such as BDF/vendor/device. Use `locator` fields for stable placement facts such as slot and position. Keep PCIe, PMU, ISI[8], DDP[4], and DMC[3 per DDP] inside the ATLAS/TPU parent object as diagnostic modules.
3. Define internal MVP concurrency semantics: same-object atomic tests are serialized by `BaseDevice::atomic_test_mutex_`. `HalSession` owns shared resource lifetimes and any HAL-internal locks for DMA buffers, BAR mappings, PMU IPC, and future work queues. Do not wire HQC-backed command queues into device/module objects by default; `CmdQueueMgm` stays dormant until FW queue ownership is confirmed. Non-HQC paths are tracked as TODO review items and get HAL-side serialization only when a real test needs it.
4. Define schemas and result model before writing tests.
5. Build a minimal Python runner with fake HAL using the compact MVP layout when appropriate: `main.py`, `config.py`, `runner.py`, and `tests/`. Validate inventory, config, one HAL atomic testcase, one external-tool testcase, logs, and result JSON.
6. Add external tool tests first when useful, especially FIO/NVMe, because they validate orchestration without custom kernel work.
7. Add C++ HAL discovery and telemetry next.
8. Add UMD or HAL submit queue manager only after tests are confirmed to program hardware/FW queues. Centralize descriptor allocation, ring tail/head updates, FW notification, completion attribution, timeout, cancel, drain, reset coordination, and process-crash cleanup at that confirmed lower layer.
9. Add the thinnest kernel module only when contiguous memory allocation/mmap, IRQ request/event delivery, IOMMU mapping, privileged mapping, or lifecycle cleanup cannot be safely obtained from VFIO, vendor SDKs, or existing drivers.
10. Add packaging early enough that the CLI, HAL, policies, specs, and optional kmod can be installed and smoke-tested as a deb.
11. Expand to stress, burn-in, thermal, multi-device, and multi-node tests once resource scheduling is solid.

## MVP And Future Ideas

- MVP: schemas for top-level `Device`, `DeviceTree`, `Policy`, `TestSpec`, and `TestResult`; compact Python runner with fake HAL, DeviceManager-backed discovery, policy locator matching, config-driven sequence execution, `tests/` testcase modules, `Runner.run_tool(...)`, dry-run mode, and structured report.
- Future idea: real `nvme_fio` testcase via `Runner.run_tool(...)`; split out `ToolAdapter` later only when external tool wrappers become repetitive.
- Current HAL direction: implement C++ `DeviceManager.discover()` first. It scans PCI devices through `HalSession`, applies policy to map BDF/slot/position/serial to canonical ATLAS device names, mmaps BAR space through the session, creates `TPUDevice` parent instances, and lets each parent create its PCIe, PMU, DDP/DMC, and ISI module objects with internal register-window offsets. It registers the full device tree and exposes a Python binding. Discovery output should not expose module register offsets or raw mapped BAR virtual addresses; those remain HAL object internals. Then expose a small `run_atomic_test(...)` catalog that receives `HalSession` via `TestInfo` only when low-level resources are needed. Reserve `run_group_atomic_test(...)` for synchronized multi-device diagnostics when needed.
- Current C++ layout: keep public headers under `include/diag/`. Put framework primitives such as `BaseDevice`, `Common`, `TestInfo`, and `CmdQueueMgm` in `include/diag/core` with implementations in `src/core`; top-level device ownership such as `DeviceManager` and `TPUDevice` in `include/diag/device` and `src/device`; child modules such as PCIe, PMU, ISI, DDP, and DMC in `include/diag/module` and `src/module`; pybind code in `bindings`; examples in `examples`; platform policy placeholders in `policies`.
- Current DMA direction: keep DMA memory allocation and address interpretation in `HalSession`/HAL. `pcie_dma_data_transfer` and similar tests should request a `DmaBuffer`, program or sketch transfer intent using its `device_addr` plus offsets, and report `addr_kind`, `backend_handle` when useful, and allocation size. Do not accept raw device-memory addresses from CLI/Python except as a future explicit expert/debug mode with policy validation.
- Future idea: ultra-thin kernel module or VFIO/existing-driver path for contiguous memory allocation/mmap and optional IRQ event delivery. Preserve `device_addr`/`addr_kind` even if IOMMU is disabled for the MVP.
- Future idea: add a HAL-side scheduler or supervised HAL daemon only if queueing, priority, crash recovery, multi-process arbitration, or richer concurrency requires it.
- Current queue direction: keep a lightweight `CmdQueueMgm` class in `core` as an unconnected placeholder. Do not pass it through `TPUDevice`, `BaseDevice`, PCIe, PMU, ISI, DDP, or DMC constructors while the FW interaction is out of the active diag framework scope. When queue work becomes real, attach queue managers to `HalSession` or the confirmed UMD/HAL backend rather than to individual module objects. The class should still expose `submit()`, `poll_complete()`, `wait()`, and `drain()` as the future HAL-side concurrency contract around descriptor ownership, ring head/tail updates, FW notification, completion consumption, queue locking, timeout, cancel, and drain. Reconnect it only after the owner of HQC/IPC descriptor packing and completion parsing is confirmed. If future work mentions queue manager casually, first preserve the dormant state and ask whether the FW boundary has changed before wiring it back.
- Future idea: review and classify non-HQC paths once real tests exist; add narrow HAL-internal mutexes, parent-device sequencing, or quiesce/drain/reset guards only for paths proven to be unsafe.
- Future idea: deb packaging with no-kmod and DKMS-kmod modes, runtime layout, udev, and install smoke test.
- Future idea: chip memory, DMA, PCIe, thermal, reset, burn-in, multi-device, and multi-node suites.

## Reference Loading

Read `references/tpu-mvp-decisions.md` when the user asks about the current TPU diagnostic MVP, target-device model, DeviceManager responsibilities, HAL atomic test scope, `skucheck`, `connectivity`, or why PCIe/ISI/PMU/DDP are not exposed as logical tree nodes.

Read `references/python-framework-mvp.md` when the user asks to simplify the Python layer, design the MVP Python framework, decide whether CLI entry logic and runner logic should be merged, place testcase wrappers, define HAL `run_atomic_test` vs Python `run_tool`, or design CLI/testcase/sequence execution.

Read `references/action-items.md` when the user asks what design or implementation details remain to be finalized, wants to track follow-up tasks across Python framework, HAL, UMD, kernel, packaging, or needs to revisit config schema, HAL atomic test catalog, result/error model, concurrency semantics, or the first connectivity testcase template.

Read `references/resource-concurrency-model.md` only when the user explicitly asks to revisit the older explicit lease/resource-policy approach. For current TPU MVP concurrency questions, prefer the SKILL.md conclusion: `atomic_test_mutex_` for same-target serialization first, dormant `CmdQueueMgm`, and no Python-visible LeaseManager by default.

Read `references/thin-kmod-boundary.md` when the user asks whether the kernel module can be smaller, what belongs in ko vs HAL, or how DMA, IOMMU, BAR/MMIO, and PCI config space should be split.

Read `references/umd-hal-boundary.md` when the user asks about UMD, user-mode drivers, firmware descriptors, doorbells, command/completion rings, DMA responsibility moving to user space, IRQ wait paths, or whether UMD can replace HAL.

Read `references/packaging-deb.md` when the user asks how to package the diagnostic tool as a deb, design install paths, DKMS/no-kmod modes, udev/systemd integration, postinst/prerm behavior, or release artifacts.

Read `references/local-env.md` when local command discovery fails on Mona's Windows workstation, especially for Python.
