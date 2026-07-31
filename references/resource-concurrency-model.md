# Resource Concurrency Model

Use this reference to design the concrete implementation for device occupancy, physical resource conflict handling, and hardware submit queue arbitration in AI chip diagnostic frameworks.

## Design Goal

Prevent multiple tests from accidentally contending for the same physical resource while keeping the framework small enough to build first.

Core rule:

```text
Tests compete for scheduler leases over LogicalDevices.
Tests do not directly compete for ringbuffers or descriptors; UMD/HAL owns those details.
```

Default MVP architecture:

```text
TestSpec / Policy
  -> declare shared/exclusive LogicalDevice locks
LogicalDevice Registry
  -> name physical and virtual/domain nodes
LeaseManager / Scheduler
  -> grant exclusive/shared leases; MVP Scheduler may be only a synchronous lease gate
Device WorkQueue
  -> serialize tasks per logical device when needed
HAL Session
  -> owns diagnostic context and coarse submit API
UMD Session, when present
  -> owns low-level device context, firmware descriptors, rings, and doorbells
SubmitQueueManager
  -> allocate descriptors, update rings, ring doorbells, attribute completions
Thin kernel module
  -> enforce the thinnest required memory/IRQ services, such as contiguous allocation/mmap and IRQ event delivery
Hardware
```

Default split:

```text
framework owns LogicalDevice registry, scheduling facade, and cross-process leases
HAL owns diagnostic submit semantics and wraps UMD when present
UMD owns firmware descriptor mechanics, rings, doorbells, polling/IRQ wait, completion attribution, cancel, and drain when present
kernel owns only the required privileged memory/IRQ services; in ultra-thin mode it does not own DMA or IOMMU mapping
```

For the MVP, keep the public API name `Scheduler` if that helps future extensibility, but implement it as a small wrapper around a cross-process `LeaseManager`. It should expand lock policy, acquire leases, run one testcase synchronously, release leases, and write lease metadata. It should not use a global CLI lock, because multiple CLI instances may safely run when their LogicalDevice leases do not conflict.

Defer queueing, priority/fairness, worker pools, DAG dispatch, and automatic suite-level parallelism until the framework actually needs them.

Move submit/wait/cancel/drain into the kernel only as a future heavy-kmod mode when user-space UMD/HAL ownership cannot provide enough crash safety, interrupt ownership, multi-process arbitration, or reset safety.

## LogicalDevice-First Model

Expose everything the user can select, inspect, or lock as a LogicalDevice. A LogicalDevice may be physical, virtual, or a domain.

Examples:

```text
TPU_0                 physical AI chip with HAL backend
SWITCH_0              physical switch ASIC with optional HAL backend
FABRIC_0              virtual/domain device for shared fabric occupancy
PCIE_DOMAIN_0         virtual/domain device for PCIe subtree occupancy
RESET_DOMAIN_0        virtual/domain device for reset blast radius
POWER_DOMAIN_0        virtual/domain device for power budget occupancy
THERMAL_DOMAIN_0      virtual/domain device for thermal coupling
TELEMETRY_0           virtual/domain device for sensor/log channel
BMC_0                 physical or service device for BMC/IPMI/Redfish access
```

Recommended schema:

```yaml
logical_devices:
  TPU_0:
    kind: physical
    type: ai_chip
    has_hal_backend: true
    parents: [FABRIC_0, PCIE_DOMAIN_0, RESET_DOMAIN_0, POWER_DOMAIN_0]

  SWITCH_0:
    kind: physical
    type: fabric_switch
    has_hal_backend: true
    parents: [FABRIC_0]

  FABRIC_0:
    kind: virtual
    type: fabric_domain
    has_hal_backend: false
    children: [TPU_0, TPU_1, TPU_2, TPU_3, TPU_4, TPU_5, TPU_6, TPU_7, SWITCH_0]
```

MVP lease modes:

```text
exclusive       no other exclusive/shared lease may overlap
shared          compatible with other shared leases
```

Conflict rules:

```text
exclusive vs exclusive -> conflict
exclusive vs shared    -> conflict
shared vs shared       -> allowed
```

Acquire locks in a deterministic global order, such as sorted LogicalDevice ID, to avoid deadlocks.

Use lock groups to reduce policy repetition:

```yaml
lock_groups:
  FABRIC_0_FULL:
    exclusive:
      - FABRIC_0
      - SWITCH_0
      - TPU_0
      - TPU_1
      - TPU_2
      - TPU_3
      - TPU_4
      - TPU_5
      - TPU_6
      - TPU_7
```

## Policy Schema

Each test declares the LogicalDevices it touches. The scheduler expands `${device}` and lock groups before execution.

```yaml
tests:
  gpumem:
    parallelism: per_device
    locks:
      exclusive:
        - ${device}
      shared:
        - TELEMETRY_0

  dma_submit:
    parallelism: per_device
    locks:
      exclusive:
        - ${device}
      shared:
        - TELEMETRY_0

  fabric_stress_8chip:
    parallelism: inside_hal
    locks:
      exclusive_groups:
        - FABRIC_0_FULL
      shared:
        - TELEMETRY_0

  pcie_reset:
    parallelism: sequential_by_domain
    locks:
      exclusive:
        - PCIE_DOMAIN_0
        - RESET_DOMAIN_0
        - ${device}
```

## Optional ResourceGraph Model

Add a richer ResourceGraph only when LogicalDevice locks become too repetitive or when topology-derived expansion is needed across fabrics, PCIe domains, reset domains, power domains, or multi-node systems.

Represent contended objects as resources with stable IDs:

```text
chip:<id>                 accelerator/GPU/AI chip
hbm:<id>                  device-local memory
engine:<device>:<engine>  compute/copy/DMA/media engine
ring:<device>:<queue>     hardware command ring or queue
descriptor_pool:<id>      descriptor allocator pool
fabric:<id>               NVLink/NoC/switch fabric
switch:<id>               fabric switch ASIC
pcie_domain:<id>          PCIe root/domain/switch subtree
reset_domain:<id>         FLR/hot-reset/bus-reset scope
irq:<num>                 interrupt line or MSI-X vector
mmap_region:<bar>:<range> mapped BAR/MMIO range
dma_iova:<range>          DMA-visible address range
bmc                       global BMC/IPMI/Redfish channel
telemetry:<id>            read-only sensors/log streams
numa:<id>                 CPU/NUMA locality resource
```

Treat `ring`, `descriptor_pool`, `irq`, `mmap_region`, and `dma_iova` as HAL/kernel internal resources unless a test directly stresses those low-level objects.

## Python Framework Interfaces

Define explicit scheduler contracts.

```python
from dataclasses import dataclass
from enum import Enum

class LeaseMode(str, Enum):
    EXCLUSIVE = "exclusive"
    SHARED = "shared"

@dataclass(frozen=True)
class LogicalDeviceId:
    value: str

@dataclass(frozen=True)
class DeviceLockRequest:
    device: LogicalDeviceId
    mode: LeaseMode

@dataclass
class Lease:
    lease_id: str
    owner_test_id: str
    locks: list[DeviceLockRequest]
    deadline_ms: int

class LogicalDeviceRegistry:
    def get(self, device_id: str) -> LogicalDevice: ...
    def children_of(self, device_id: str) -> list[LogicalDeviceId]: ...
    def expand_lock_group(self, group_id: str) -> list[DeviceLockRequest]: ...
    def expand_test_policy(self, test_name: str, targets: list[str], args: dict) -> list[DeviceLockRequest]: ...

class LeaseManager:
    def try_acquire(self, owner_test_id: str, requests: list[DeviceLockRequest], timeout_ms: int) -> Lease: ...
    def release(self, lease: Lease) -> None: ...
    def heartbeat(self, lease: Lease) -> None: ...
```

Scheduler pseudocode:

```python
class Scheduler:
    def run_test(self, test, target_devices):
        requests = self.devices.expand_test_policy(test.name, target_devices, test.args)

        lease = self.lease_manager.try_acquire(
            owner_test_id=test.id,
            requests=requests,
            timeout_ms=test.queue_timeout_ms,
        )

        try:
            return test.run(lease=lease)
        finally:
            self.lease_manager.release(lease)
```

In MVP, this `Scheduler` is deliberately synchronous: it is not required to own a task queue or dispatch pool. Its main job is to make lease acquisition mandatory before any testcase reaches HAL, UMD, external tools, BAR/MMIO, reset, DMA submit, or shared telemetry/BMC paths.

Conflict check:

```python
def conflicts(existing: DeviceLockRequest, requested: DeviceLockRequest) -> bool:
    if existing.device != requested.device:
        return False
    return LeaseMode.EXCLUSIVE in (existing.mode, requested.mode)
```

Expand lock groups before conflict checking. Without expansion, a lease on `FABRIC_0_FULL` will not conflict with a request for `TPU_0` unless `TPU_0` is materialized as an acquired lock.

Device-local work queue:

```python
class LogicalDeviceExecutor:
    def __init__(self):
        self.queues: dict[str, asyncio.Queue] = {}

    async def submit(self, device_id: str, task):
        await self.queues[device_id].put(task)

    async def worker(self, device_id: str):
        while True:
            task = await self.queues[device_id].get()
            try:
                await task.run()
            finally:
                self.queues[device_id].task_done()
```

Use the device-local queue for same-LogicalDevice serialization when needed. Use the LeaseManager for cross-device occupancy such as fabric, PCIe domain, reset, power, thermal, BMC, or telemetry.

## Multi-Device Stress Boundary

For strongly coupled multi-device diagnostics, Python should acquire the full lease set and call one coarse HAL primitive. Examples include 8-chip stress, fabric stress, collective bandwidth, thermal stress, synchronized power stress, and tests that require aligned start/stop timing.

Recommended flow:

```text
Python Scheduler
  -> acquire exclusive CHIP_0..CHIP_7 plus affected FABRIC/POWER/THERMAL domains
  -> call HAL run_primitive("chip_stress", devices=[...], lease=token, options={...})
  -> collect structured per-device results and artifacts

HAL/UMD
  -> open sessions
  -> create per-device workers and queues
  -> synchronize start barrier
  -> submit work, attribute completions, monitor errors
  -> apply fail policy
  -> timeout, cancel, drain, cleanup
```

Python fan-out is acceptable for independent per-device tests and external tool adapters. DGX `gpustress` follows a Python multi-instance MODS pattern, but treat that as a tool-adapter/legacy shape rather than the preferred HAL/UMD boundary for tightly synchronized chip diagnostics.

## HAL Interfaces

Expose session-scoped operations. A session must carry an active scheduler lease.

```cpp
struct DeviceId {
    std::string value;  // Stable logical ID such as TPU_0, FABRIC_0, or BMC_0.
};

struct LeaseToken {
    std::string lease_id;
    std::string owner_test_id;
    uint64_t expires_at_ms;
};

struct SessionOptions {
    bool exclusive;
    uint32_t flags;
};

class DeviceSession {
public:
    virtual DeviceId device() const = 0;
    virtual LeaseToken lease() const = 0;
    virtual void Close() = 0;
};

class HalDeviceManager {
public:
    virtual std::vector<DeviceId> Discover() = 0;
    virtual Topology GetTopology() = 0;
    virtual std::unique_ptr<DeviceSession> OpenSession(
        const DeviceId& device,
        const LeaseToken& lease,
        const SessionOptions& options) = 0;
};
```

Submit API:

```cpp
using JobId = uint64_t;
using QueueId = uint32_t;
using BufferId = uint64_t;

struct Descriptor {
    uint32_t opcode;
    uint32_t flags;
    uint64_t src_iova;
    uint64_t dst_iova;
    uint64_t bytes;
    uint64_t user_cookie;
};

struct SubmitOptions {
    uint64_t timeout_ms;
    bool interrupt_on_completion;
    bool allow_queueing;
};

struct JobStatus {
    JobId job_id;
    int rc;
    uint32_t hw_status;
    uint64_t completion_timestamp_ns;
};

class SubmitQueue {
public:
    virtual JobId Submit(
        DeviceSession& session,
        QueueId queue,
        const Descriptor& descriptor,
        const std::vector<BufferId>& buffers,
        const SubmitOptions& options) = 0;

    virtual JobStatus Wait(JobId job, uint64_t timeout_ms) = 0;
    virtual void Cancel(JobId job) = 0;
    virtual void Drain(DeviceSession& session, QueueId queue) = 0;
};
```

Reset API:

```cpp
enum class ResetMode {
    FunctionLevelReset,
    HotReset,
    SecondaryBusReset,
    ChipReset,
};

class ResetController {
public:
    virtual void Quiesce(DeviceSession& session) = 0;
    virtual void Drain(DeviceSession& session) = 0;
    virtual void Reset(DeviceSession& session, ResetMode mode) = 0;
    virtual void Reinitialize(DeviceSession& session) = 0;
};
```

## Submit Queue Manager Implementation

The submit queue manager owns ring mutation and descriptor lifetime.

```cpp
class SubmitQueueManager {
private:
    std::mutex queue_lock_;
    RingBuffer ring_;
    DescriptorPool descriptor_pool_;
    CompletionTable completions_;
    std::unordered_map<JobId, JobContext> jobs_;

public:
    JobId Submit(DeviceSession& session, QueueId queue, const Descriptor& desc,
                 const std::vector<BufferId>& buffers, const SubmitOptions& options) {
        std::lock_guard<std::mutex> g(queue_lock_);

        ValidateLease(session.lease(), queue);
        ValidateBuffersOwnedBySession(session, buffers);

        auto slot = descriptor_pool_.Allocate();
        auto job = CreateJobContext(session, queue, slot, buffers, options);

        WriteDescriptor(slot, desc);
        MemoryBarrierBeforeDoorbell();
        ring_.Push(slot);
        RingDoorbell(queue);

        jobs_.emplace(job.job_id, job);
        return job.job_id;
    }
};
```

Required invariants:

```text
One descriptor belongs to exactly one in-flight job.
One job belongs to exactly one session.
One session belongs to exactly one active lease.
Ring head/tail updates are serialized or atomic.
Doorbell writes happen after descriptor visibility is guaranteed.
Completion lookup uses job/session identity, not only interrupt timing.
```

## Kernel Module Responsibilities

Keep the default kernel module narrow. It is authoritative only for resources that user space cannot safely own: DMA-safe buffer allocation, IOMMU mapping, mmap ownership, cache sync when needed, and buffer cleanup.

Default thin-kmod UAPI:

```c
#define ACD_IOCTL_GET_VERSION      _IOR('A', 0x01, struct acd_version)
#define ACD_IOCTL_OPEN_DEVICE      _IOWR('A', 0x02, struct acd_open_device)
#define ACD_IOCTL_DMA_ALLOC        _IOWR('A', 0x03, struct acd_dma_alloc)
#define ACD_IOCTL_DMA_FREE         _IOW('A', 0x04, struct acd_dma_free)
#define ACD_IOCTL_DMA_SYNC         _IOW('A', 0x05, struct acd_dma_sync)
```

Versioned UAPI structs:

```c
struct acd_dma_alloc {
    __u32 size;
    __u32 version;
    __u64 flags;
    __u64 bytes;
    __u64 handle;
    __u64 dma_addr;      /* device-visible DMA address / IOVA */
    __u64 mmap_offset;   /* offset passed to mmap */
    __u64 reserved[8];
};
```

Future heavy-kmod mode, not the default:

```c
#define ACD_IOCTL_SUBMIT           _IOWR('A', 0x20, struct acd_submit)
#define ACD_IOCTL_WAIT             _IOWR('A', 0x21, struct acd_wait)
#define ACD_IOCTL_CANCEL           _IOW('A', 0x22, struct acd_cancel)
#define ACD_IOCTL_DRAIN            _IOW('A', 0x23, struct acd_drain)
```

Add these only if HAL/user-space submit cannot satisfy crash safety, interrupt ownership, multi-process arbitration, or reset safety requirements.

Kernel-side protection:

```text
per-device mutex              DMA mapping context open/close
per-client mutex              DMA allocation list and mmap list
idr/xarray                    buffer handle lookup
refcount_t                    buffers and mappings
```

Required cleanup:

```text
file release -> free DMA buffers owned by client
file release -> revoke mmap ownership for buffers owned by client
```

Thin-kmod crash caveat:

```text
Because submit/ring ownership lives in HAL, the framework must prevent DMA buffers from being freed while hardware may still access them. Prefer running HAL submit through a supervised process or HAL daemon that can quiesce/drain/reset affected queues before releasing buffers. If this cannot be guaranteed, consider the future heavy-kmod submit mode.
```

## Reset and Destructive Operation Protocol

Any reset, link retrain, firmware reload, power transition, or fabric reconfiguration requires an exclusive lease over its blast radius.

Protocol:

```text
1. Scheduler acquires exclusive `RESET_DOMAIN_0` and affected device leases.
2. Scheduler blocks new submits for affected LogicalDevices.
3. HAL calls Quiesce() for affected sessions, delegating to UMD when UMD owns the queues.
4. SubmitQueueManager drains or cancels in-flight jobs.
5. UMD/HAL verifies no owned descriptors remain active.
6. Kernel retains only memory/mmap/IRQ ownership unless heavy-kmod mode is enabled.
7. HAL performs reset/retrain/reconfigure.
8. HAL reinitializes queue state and telemetry.
9. Scheduler releases lease.
```

Reject policy:

```text
submit during reset lease -> EBUSY or queued by scheduler
reset while shared lease exists -> denied
reset while in-flight jobs exist -> drain, cancel, or denied based on policy
```

## Error Model

Return structured errors rather than relying only on log parsing.

```cpp
enum class DiagError {
    Ok,
    LeaseConflict,
    LeaseExpired,
    DeviceBusy,
    QueueFull,
    DescriptorExhausted,
    InvalidSession,
    InvalidBufferOwner,
    SubmitTimeout,
    CompletionMismatch,
    ResetBlocked,
    HardwareFault,
    DriverFault,
};
```

Every result should include:

```json
{
  "test_id": "gpumem.001",
  "lease_id": "lease-123",
  "session_id": "session-456",
  "device_id": "CHIP_0",
  "locks": ["CHIP_0", "FABRIC_0", "SWITCH_0"],
  "job_id": 99,
  "queue_id": 0,
  "descriptor_id": 12,
  "rc": "SubmitTimeout"
}
```

## Suggested Code Layout

```text
diag/
  schemas/
    device.schema.yaml
    topology.schema.yaml
    test_policy.schema.yaml
    result.schema.yaml
  policies/
    platform_a.yaml
  python/
    diag_runner/
      scheduler.py
      lease_manager.py
      logical_device_registry.py
      logical_device_executor.py
      process_manager.py
      tool_adapter.py
      result_store.py
  hal/
    include/
      acd/device_manager.h
      acd/device_session.h
      acd/submit_queue.h
      acd/reset_controller.h
    src/
      device_manager.cpp
      submit_queue_manager.cpp
      reset_controller.cpp
  kernel/
    acd_ioctl.h
    acd_main.c
    acd_dma.c
    acd_mmap.c
  tests/
    gpumem.py
    dma_submit.py
    pcie_reset.py
```

## Implementation Checklist

- Define physical and virtual LogicalDevices.
- Require every testcase to declare LogicalDevice locks before execution.
- Validate policies in dry-run mode.
- Implement lease conflict detection before running any test.
- Serialize same-logical-device tasks with a device work queue.
- Lock shared physical/domain LogicalDevices with scheduler leases.
- Centralize hardware queue submission in HAL by default.
- Track session/job/descriptor/buffer ownership in HAL, and buffer/mmap ownership in the thin kernel module.
- Enforce DMA buffer cleanup in kernel file release; handle submit crash safety through supervised HAL ownership or future heavy-kmod mode.
- Require exclusive leases for reset and fabric reconfiguration.
- Return structured job/resource errors.
