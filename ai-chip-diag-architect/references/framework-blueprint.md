# AI Chip Diagnostic Framework Blueprint

## Goals

Design a hardware diagnostic framework for AI accelerator platforms by making the C++ HAL the stable semantic layer and keeping Python focused on orchestration. When a platform provides a UMD/User Mode Driver, treat UMD as the low-level firmware descriptor and queue owner below HAL.

## Suggested Repository Layout

Use the compact Python MVP layout as the default starting point. The fuller framework split below is a growth path, not the first implementation target.

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
      atomic_test.h
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
      atomic_test_runner.cpp
    bindings/
      pybind_module.cpp

  framework/
    aidiag/
      main.py
      config.py
      runner.py
      tests/
        __init__.py
        connectivity_check.py
        nvme_fio.py

  schemas/
    test_config.schema.yaml

  examples/
    connectivity_check.yaml
    production_sequence.yaml

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

Split the Python MVP only when real pressure appears:

- `runner.py` -> `scheduler.py` / `lease_manager.py` when lease acquisition, heartbeats, stale cleanup, wait policy, or concurrency handling make the runner hard to read. A small cross-process lease gate should live in `runner.py` for the MVP.
- `runner.py` -> `tool_adapter.py` and `parsers/` when external tool wrappers and parsers become repetitive.
- `runner.py` -> `result_store.py` when JSON report writing grows into querying, upload, or multi-format reporting.
- `config.py` -> `policy_engine.py` / `logical_device_registry.py` when SKU, topology, threshold, lock group, and support rules outgrow simple config helpers.
- `tests/` -> richer testcase support modules when common testcase validation, retry, skip, or subresult logic becomes repetitive.

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

class IAtomicTest {
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
hal.run_atomic_test("memory_selftest", device="CHIP_0", args={})
hal.reset_device("CHIP_0", mode="flr")
```

For tightly synchronized multi-device diagnostics, expose one coarse HAL atomic test instead of making Python own per-device worker threads:

```python
hal.run_group_atomic_test(
    "chip_stress",
    devices=["CHIP_0", "CHIP_1", "CHIP_2", "CHIP_3", "CHIP_4", "CHIP_5", "CHIP_6", "CHIP_7"],
    args={"duration_s": 600, "start_barrier": True, "fail_policy": "stop_all_on_first_error"},
)
```

Python decides the `devices` list from CLI/config/policy and must acquire leases for every target device and affected shared domain before calling HAL. HAL does not choose which devices participate; it validates the provided set and executes concurrently over exactly that set. HAL/UMD should own the internal workers, start barrier, submit/completion attribution, timeout, cancel, drain, and cleanup for these strongly coupled tests. Python fan-out is still appropriate for independent per-device tests and external tools.

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

For the MVP, use a plain module-level testcase interface. Do not introduce a `TestCase` base class until repeated validation, retry, skip, planning, or parsing logic makes it worthwhile.

```python
name = "connectivity_check"

required_tools: list[str] = []
required_hal_caps: list[str] = [
    "device_status",
    "isi_presence",
    "pcie_link_status",
    "power_connection_status",
]

def run(context, devices: list[str], options: dict) -> list[dict]:
    ...
```

Runner context should provide:

- `ctx.hal`
- `ctx.config`
- `ctx.runner`
- `ctx.lease(...)`
- `ctx.run_atomic_test(...)`
- `ctx.run_tool(...)`
- `ctx.log_dir`

The first testcase package may use a simple registry in `tests/__init__.py`:

```python
TESTCASES = {
    "connectivity_check": run_connectivity_check,
    "nvme_fio": run_nvme_fio,
}
```

Keep external tool execution in `Runner.run_tool(...)` for the MVP. It should own argv construction or validation, timeout, environment, cwd, stdout/stderr capture, process cleanup, log paths, parser callback execution, and artifact registration. Split it into `tool_adapter.py` only when several tool wrappers make `runner.py` hard to read.

## Policy Model

Example:

```yaml
platform: platform_x
expected:
  accelerator: 8
  nvme: 16
  nic: 8
logical_devices:
  CHIP_0:
    name: CHIP_0
    type: ai_chip
    slot: OAM0
    locator:
      pci_bdf: "0000:65:00.0"
      devnode: "/dev/ai-chip0"
    parents: [FABRIC_0, PCIE_DOMAIN_0, RESET_DOMAIN_0, POWER_DOMAIN_0]
  CHIP_1:
    name: CHIP_1
    type: ai_chip
    slot: OAM1
    locator:
      pci_bdf: "0000:66:00.0"
      devnode: "/dev/ai-chip1"
    parents: [FABRIC_0, PCIE_DOMAIN_0, RESET_DOMAIN_0, POWER_DOMAIN_0]
  FABRIC_0:
    name: FABRIC_0
    type: fabric_domain
    parents: []
    locator: {}
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

`logical_devices.<id>.name` is the canonical LogicalDevice ID. It should be stable across runs and should be the identifier used by CLI targets, testcase config, lease requests, HAL atomic calls, and result reports. `locator` pins that logical name to observed hardware such as BDF, devnode, sysfs path, serial, slot, or BMC path. HAL discovery should return observed hardware facts; the Python framework should match observed devices to policy locators before presenting canonical names.

## Future Scheduler Model

Represent tests as a DAG of tasks. Each task declares LogicalDevice locks. LogicalDevices include physical nodes such as chips, SSDs, NICs, and switches, plus virtual/domain nodes such as fabrics, PCIe domains, reset domains, power domains, thermal domains, BMC channels, and telemetry channels.

Examples:

- `chip_mem(CHIP_0)` locks `CHIP_0`.
- `fabric_stress(CHIP_0..CHIP_7)` locks `FABRIC_0_FULL` after expansion to `FABRIC_0`, `SWITCH_0`, and the eight chips.
- `nvme_fio(NVME_3)` locks `NVME_3` and possibly `NUMA_0`.
- `pcie_reset(CHIP_0)` locks `PCIE_DOMAIN_0`, `RESET_DOMAIN_0`, and `CHIP_0`.
- BMC/IPMI/Redfish operations lock `BMC_0`.

Keep the public orchestration abstraction named `Scheduler` only if the framework later needs queueing, worker pools, priority, fairness, or DAG dispatch. In the MVP, keep this behavior as a synchronous lease gate in `runner.py`:

```text
Runner
  -> expand locks and lock groups
  -> acquire lease
  -> testcase run(context, devices, options)
  -> release lease
```

The MVP lease gate should support only `shared` and `exclusive` modes, acquire locks in deterministic LogicalDevice ID order, reject or wait on conflicting locks, and record lease metadata in results. It must be cross-process visible so multiple CLI instances can run concurrently when their LogicalDevice leases do not conflict. Do not use a global CLI lock for this model.

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

## Design Lessons

- Preserve useful testcase contract ideas such as support checks, required tools, timeout, status, and artifacts, but keep the MVP testcase API as a plain module-level `run(context, devices, options)` function.
- Keep platform semantics in C++ HAL and policy files where possible, not scattered through Python testcase code.
- Preserve multi-instance execution patterns for independent accelerator tests and FIO tests, but express concurrency through resource locks. For a clean HAL/UMD-based chip stack, prefer one coarse HAL atomic test for strongly coupled multi-chip stress.
- Preserve background monitoring for BMC/sensors/telemetry, but make it a first-class monitor service.
- Avoid letting Python directly depend on low-level kernel details. Use HAL bindings.
- Avoid log-only parsing when structured output is available.
