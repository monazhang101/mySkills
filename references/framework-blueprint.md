# AI Chip Diagnostic Framework Blueprint

## Goals

Design a hardware diagnostic framework for AI accelerator platforms that improves on DGX-style packaging by making the C++ HAL the stable semantic layer and keeping Python focused on orchestration. When a platform provides a UMD/User Mode Driver, treat UMD as the low-level firmware descriptor and queue owner below HAL.

## Suggested Repository Layout

```text
ai-diag/
  kernel/
    ai_diag_mem/
      ai_diag_uapi.h
      Kconfig
      ioctl.c
      dma.c

  umd/
    include/aidiag_umd/
      device.h
      memory.h
      queue.h
      fw_descriptor.h
    src/
      device.cpp
      memory_provider.cpp
      queue_manager.cpp
      fw_protocol.cpp

  hal/
    include/aidiag/
      device.h
      topology.h
      telemetry.h
      test_primitive.h
      platform.h
      result.h
      error.h
    src/
      kernel_transport.cpp
      vendor_sdk_backend.cpp
      sysfs_backend.cpp
      device_manager.cpp
      topology_service.cpp
      platform_policy.cpp
      telemetry_service.cpp
      primitive_runner.cpp
    bindings/
      pybind_module.cpp

  framework/
    aidiag/
      cli.py
      context.py
      runner.py
      registry.py
      scheduler.py
      lease_manager.py
      logical_device_registry.py
      device_manager.py
      process_manager.py
      monitor.py
      result_store.py
      policy_engine.py
      tool_adapter.py
      parsers/
        fio.py
        stress_ng.py
        vendor_diag.py
      tests/
        inventory.py
        chip_smoke.py
        chip_mem.py
        chip_dma.py
        chip_stress.py
        pcie.py
        nvme_fio.py
        thermal.py
        reset.py

  policies/
    generic.yaml
    platform_x.yaml

  specs/
    quick.yaml
    production.yaml
    burnin.yaml

  tools/
  packaging/
    debian/
      control
      rules
      changelog
      postinst
      prerm
      postrm
      ai-diag.install
      ai-diag.dkms
    systemd/
      ai-diag-monitor.service
    udev/
      99-ai-diag.rules
  reports/
  logs/
```

## Kernel Module Contract

Keep the module small. Prefer a thin `ai_diag_mem` module that provides only the privileged services that cannot safely live in user mode. Use VFIO or existing vendor drivers instead when they already provide safe user-space BAR/config/DMA access.

If platform guidance says ko should not own DMA or IOMMU, use an ultra-thin kmod:

```text
ko
  -> contiguous physical memory allocation
  -> mmap to UMD
  -> request_irq / IRQ event delivery

UMD
  -> firmware descriptors, command/completion rings, DMA submit mechanics, doorbells, polling/IRQ wait
```

Preserve a `device_addr` plus `addr_kind` contract so IOMMU/VFIO/SMMU support can later return an IOVA without changing UMD descriptor construction.

Typical operations:

- Version query.
- Device attach/open by BDF if needed for mapping or IRQ context.
- Contiguous memory allocation for ultra-thin bring-up mode, or DMA-capable buffer allocation in default thin-kmod mode.
- Optional IOMMU mapping and device-visible DMA address/IOVA return.
- `mmap` of buffer to UMD/HAL user space.
- Buffer free and optional sync operations.
- Optional request_irq / IRQ event delivery.

Keep these in UMD/HAL when safe:

- PCI discovery and logical naming.
- BAR/MMIO mapping and register access.
- PCI config space read and limited policy-controlled writes.
- DMA engine descriptor programming, firmware descriptor construction, doorbell writes, status polling, and timeout handling.
- Device-specific diagnostic semantics.

Defer these until needed:

- Reset operations.
- Telemetry snapshots.
- Kernel-mediated BAR/config access.

UAPI rules:

- Version every struct.
- Include `size`, `version`, `flags`, and reserved fields.
- Return explicit error codes that HAL maps to framework result categories.
- Avoid encoding platform policy in the module.

Minimal ioctl set:

```c
#define AI_DIAG_GET_VERSION     _IOR(AI_DIAG_IOC_MAGIC, 0x00, struct ai_diag_version)
#define AI_DIAG_ATTACH_DEVICE   _IOWR(AI_DIAG_IOC_MAGIC, 0x01, struct ai_diag_attach_device)
#define AI_DIAG_DMA_ALLOC       _IOWR(AI_DIAG_IOC_MAGIC, 0x20, struct ai_diag_dma_alloc)
#define AI_DIAG_DMA_FREE        _IOW(AI_DIAG_IOC_MAGIC, 0x21, struct ai_diag_dma_free)
#define AI_DIAG_DMA_SYNC        _IOW(AI_DIAG_IOC_MAGIC, 0x22, struct ai_diag_dma_sync)
```

For details, read `thin-kmod-boundary.md`.

## C++ HAL Contract

HAL owns hardware semantics and exposes coarse, stable operations.

For a more implementation-ready header layout, read `hal-cpp-headers.md`.

Core interfaces:

```cpp
struct PhysicalId {
    std::string bdf;
    uint16_t vendor_id;
    uint16_t device_id;
};

struct LogicalId {
    std::string name;
    std::string slot;
};

struct Device {
    PhysicalId physical;
    LogicalId logical;
    std::string type;
    std::string health;
};

class IDeviceManager {
public:
    virtual std::vector<Device> Discover() = 0;
    virtual Device Open(const LogicalId&) = 0;
    virtual Topology GetTopology() = 0;
};

class ITelemetry {
public:
    virtual TelemetrySnapshot Read(const Device&) = 0;
    virtual void Subscribe(const Device&, EventCallback cb) = 0;
};

class ITestPrimitive {
public:
    virtual TestResult MemorySelfTest(const Device&, const MemoryTestOptions&) = 0;
    virtual TestResult DmaLoopback(const Device&, const DmaOptions&) = 0;
    virtual TestResult PcieLinkCheck(const Device&) = 0;
    virtual TestResult ResetCheck(const Device&, ResetType) = 0;
};
```

Python binding should expose coarse methods:

```python
hal.discover()
hal.get_topology()
hal.read_telemetry("CHIP_0")
hal.run_primitive("memory_selftest", device="CHIP_0", options={})
hal.reset_device("CHIP_0", mode="flr")
```

For tightly synchronized multi-device diagnostics, expose one coarse HAL primitive instead of making Python own per-device worker threads:

```python
hal.run_primitive(
    "chip_stress",
    devices=["CHIP_0", "CHIP_1", "CHIP_2", "CHIP_3", "CHIP_4", "CHIP_5", "CHIP_6", "CHIP_7"],
    options={"duration_s": 600, "start_barrier": True, "fail_policy": "stop_all_on_first_error"},
)
```

HAL/UMD should own the internal workers, start barrier, submit/completion attribution, timeout, cancel, drain, and cleanup for these strongly coupled tests. Python fan-out is still appropriate for independent per-device tests and external tools.

## Packaging and Release Contract

Add a packaging layer once the CLI, HAL library, policies, and at least fake/no-kmod runtime can be smoke-tested. Package both `no-kmod` and `dkms-kmod` modes when the kernel module is optional.

Install layout should be predictable:

```text
/usr/bin/ai-diag
/usr/lib/libaidiag_hal.so
/usr/lib/python3/dist-packages/aidiag/
/usr/share/ai-diag/policies/
/usr/share/ai-diag/specs/
/usr/share/ai-diag/schemas/
/usr/share/ai-diag/tools/
/etc/ai-diag/config.yaml
/var/log/ai-diag/
```

For details, read `packaging-deb.md`.

## Python Framework Contracts

Testcase interface:

```python
class TestCase:
    name: str
    component: str
    required_tools: list[str]
    required_hal_caps: list[str]
    locks_exclusive: list[str]
    locks_shared: list[str]
    timeout_s: int

    def is_supported(self, ctx) -> bool: ...
    def plan(self, ctx) -> list[TestTask]: ...
    def run(self, ctx, task: TestTask) -> TestResult: ...
    def parse(self, ctx, artifacts) -> TestResult: ...
```

Runner context should provide:

- `ctx.hal`
- `ctx.devices`
- `ctx.topology`
- `ctx.policy`
- `ctx.tools`
- `ctx.scheduler`
- `ctx.leases`
- `ctx.monitor`
- `ctx.results`
- `ctx.logs`

Tool adapter contract:

```python
spec = ToolSpec(
    name="fio",
    argv=[...],
    timeout_s=360,
    env={},
    cwd=None,
    parser=FioJsonParser(),
)
result = ctx.tools.run(spec)
```

The adapter owns process group cleanup, stdout/stderr capture, timeout, structured parser execution, and artifact registration.

## Policy Model

Example:

```yaml
platform: platform_x
expected:
  accelerator: 8
  nvme: 16
  nic: 8
mapping:
  "0000:65:00.0": CHIP_0
  "0000:66:00.0": CHIP_1
thresholds:
  chip_temp_c_max: 85
  hbm_ecc_uncorrectable_max: 0
  nvme_randread_iops_min: 500000
capabilities:
  kernel_module_required: true
  supports_flr: true
  supports_dma_loopback: true
concurrency:
  max_chip_stress_parallel: 8
  max_fio_parallel: 4
  bmc_commands_serialized: true
lock_groups:
  FABRIC_0_FULL:
    exclusive:
      - FABRIC_0
      - SWITCH_0
      - CHIP_0
      - CHIP_1
      - CHIP_2
      - CHIP_3
      - CHIP_4
      - CHIP_5
      - CHIP_6
      - CHIP_7
locks:
  pcie_reset:
    exclusive:
      - PCIE_DOMAIN_0
      - RESET_DOMAIN_0
      - "{device}"
  bmc:
    exclusive:
      - BMC_0
```

## Scheduler Model

Represent tests as a DAG of tasks. Each task declares LogicalDevice locks. LogicalDevices include physical nodes such as chips, SSDs, NICs, and switches, plus virtual/domain nodes such as fabrics, PCIe domains, reset domains, power domains, thermal domains, BMC channels, and telemetry channels.

Examples:

- `chip_mem(CHIP_0)` locks `CHIP_0`.
- `fabric_stress(CHIP_0..CHIP_7)` locks `FABRIC_0_FULL` after expansion to `FABRIC_0`, `SWITCH_0`, and the eight chips.
- `nvme_fio(NVME_3)` locks `NVME_3` and possibly `NUMA_0`.
- `pcie_reset(CHIP_0)` locks `PCIE_DOMAIN_0`, `RESET_DOMAIN_0`, and `CHIP_0`.
- BMC/IPMI/Redfish operations lock `BMC_0`.

Keep the public orchestration abstraction named `Scheduler` so the framework can later grow queueing, worker pools, priority, fairness, and DAG dispatch. In the MVP, implement it as a synchronous lease-aware facade:

```text
Runner
  -> Scheduler.run_testcase(...)
      -> PolicyEngine expands locks and lock groups
      -> LeaseManager.try_acquire(...)
      -> TestCase.run(...)
      -> LeaseManager.release(...)
```

The MVP `Scheduler` should support only `shared` and `exclusive` modes, acquire locks in deterministic LogicalDevice ID order, reject or wait on conflicting locks, and record lease metadata in results. Its backing `LeaseManager` must be cross-process visible so multiple CLI instances can run concurrently when their LogicalDevice leases do not conflict. Do not use a global CLI lock for this model.

Add queueing, priority/fairness, worker pools, suite-level automatic parallelization, and a richer ResourceGraph only when the test suite needs them.

## Result Model

Every result should include:

- Test name and subtest name.
- Logical and physical device identifiers.
- Start/end timestamps and duration.
- Status: pass, fail, skip, error, timeout, unsupported.
- Error category: framework, policy, tool, HAL, kernel, hardware, user config.
- Numeric/code string error code.
- Human-readable diagnosis.
- Artifacts: logs, JSON, telemetry snapshots, command lines, core dumps.
- Retry history.

## DGX-Derived Lessons

- Keep a testcase base interface similar to DGX's `DCDiagTest`, but make planning and parsing explicit.
- Keep DGX's useful separation of product/HAL/testcase, but move platform semantics into C++ HAL and policy files where possible.
- Preserve multi-instance execution patterns for independent accelerator tests and FIO tests, but express concurrency through resource locks. DGX `gpustress` launches multiple MODS instances from Python, one per GPU; treat that as a legacy/tool-adapter pattern. For a cleaner HAL/UMD-based chip stack, prefer one coarse HAL primitive for strongly coupled multi-chip stress.
- Preserve background monitoring for BMC/sensors/telemetry, but make it a first-class monitor service.
- Avoid letting Python directly depend on low-level kernel details. Use HAL bindings.
- Avoid log-only parsing when structured output is available.
