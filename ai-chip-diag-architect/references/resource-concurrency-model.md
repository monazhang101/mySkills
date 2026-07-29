# Resource Concurrency Model

Use this reference to design the concrete implementation for device occupancy, physical resource conflict handling, and hardware submit queue arbitration in AI chip diagnostic frameworks.

## Design Goal

Prevent multiple tests from accidentally contending for the same physical resource.

Core rule:

```text
Tests compete for scheduler leases.
Tests do not directly compete for ringbuffers, descriptors, reset domains, DMA engines, or shared fabrics.
```

Target architecture:

```text
TestSpec / Policy
  -> declare required resources
ResourceGraph
  -> map logical devices to shared physical resources
LeaseManager / Scheduler
  -> grant exclusive/shared leases
Device WorkQueue
  -> serialize tasks per logical device when needed
HAL Session
  -> owns device context and submit API
SubmitQueueManager
  -> allocate descriptors, update rings, ring doorbells, attribute completions
Thin kernel module
  -> enforce DMA-safe buffer allocation, IOMMU mapping, mmap ownership, and buffer cleanup
Hardware
```

Default split:

```text
framework owns scheduling and leases
HAL owns sessions, descriptors, rings, doorbells, polling, completion attribution, cancel, and drain
kernel owns DMA-safe buffer allocation/mapping only
```

Move submit/wait/cancel/drain into the kernel only as a future heavy-kmod mode when user-space HAL ownership cannot provide enough crash safety, interrupt ownership, or reset safety.

## Resource Model

Represent all contended objects as resources with stable IDs.

Recommended resource namespaces:

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

Most testcase policies should declare semantic resources such as `chip`, `hbm`, `engine`, `ring`, `descriptor_pool`, `fabric`, `pcie_domain`, `reset_domain`, `bmc`, and `numa`. Treat `irq`, `mmap_region`, and `dma_iova` as HAL/kernel internal resources unless a test directly stresses those low-level objects.

Lease modes:

```text
exclusive       no other exclusive/shared lease may overlap
shared          compatible with other shared leases
group_exclusive exclusive over a computed resource group, e.g. all devices in one fabric
```

Conflict rules:

```text
exclusive vs exclusive -> conflict
exclusive vs shared    -> conflict
shared vs shared       -> allowed
```

## Policy Schema

Each test declares the resources it touches. The scheduler expands `${device}` and topology-derived variables before execution.

```yaml
tests:
  gpumem:
    parallelism: per_device
    resources:
      exclusive:
        - chip:${device}
        - hbm:${device}
        - engine:${device}:mem
      shared:
        - telemetry:${device}

  dma_submit:
    parallelism: per_device
    resources:
      exclusive:
        - chip:${device}
        - engine:${device}:dma0
        - ring:${device}:dma0
        - descriptor_pool:${device}:dma0
      shared:
        - telemetry:${device}

  fabric_link:
    parallelism: per_fabric
    resources:
      group_exclusive:
        - fabric:${fabric_id}
        - switch:${fabric_switches}
      shared:
        - chip:${fabric_devices}

  pcie_reset:
    parallelism: sequential_by_domain
    resources:
      exclusive:
        - pcie_domain:${domain}
        - reset_domain:${domain}
        - chip:${device}
```

## Resource Graph Schema

Build a topology-derived graph at discovery time.

```yaml
devices:
  CHIP_0:
    bdf: "0000:81:00.0"
    resources:
      - chip:CHIP_0
      - hbm:CHIP_0
      - pcie_domain:0000
      - reset_domain:pcie_root_0
      - fabric:fabric_0
      - engine:CHIP_0:dma0
      - ring:CHIP_0:dma0
      - descriptor_pool:CHIP_0:dma0
      - telemetry:CHIP_0

  CHIP_1:
    bdf: "0000:82:00.0"
    resources:
      - chip:CHIP_1
      - hbm:CHIP_1
      - pcie_domain:0000
      - reset_domain:pcie_root_0
      - fabric:fabric_0
      - engine:CHIP_1:dma0
      - ring:CHIP_1:dma0
      - descriptor_pool:CHIP_1:dma0
      - telemetry:CHIP_1

groups:
  fabric:fabric_0:
    members:
      - chip:CHIP_0
      - chip:CHIP_1
      - switch:SWITCH_0

  reset_domain:pcie_root_0:
    members:
      - chip:CHIP_0
      - chip:CHIP_1
      - pcie_domain:0000
```

## Python Framework Interfaces

Define explicit scheduler contracts.

```python
from dataclasses import dataclass
from enum import Enum
from typing import Iterable

class LeaseMode(str, Enum):
    EXCLUSIVE = "exclusive"
    SHARED = "shared"
    GROUP_EXCLUSIVE = "group_exclusive"

@dataclass(frozen=True)
class ResourceRef:
    namespace: str
    identifier: str

@dataclass(frozen=True)
class ResourceRequest:
    resource: ResourceRef
    mode: LeaseMode

@dataclass
class Lease:
    lease_id: str
    owner_test_id: str
    resources: list[ResourceRequest]
    deadline_ms: int

class ResourceGraph:
    def resources_for_device(self, logical_device: str) -> set[ResourceRef]: ...
    def expand_group(self, resource: ResourceRef) -> set[ResourceRef]: ...
    def expand_test_policy(self, test_name: str, device: str, args: dict) -> list[ResourceRequest]: ...

class LeaseManager:
    def try_acquire(self, owner_test_id: str, requests: list[ResourceRequest], timeout_ms: int) -> Lease: ...
    def release(self, lease: Lease) -> None: ...
    def heartbeat(self, lease: Lease) -> None: ...
```

Scheduler pseudocode:

```python
class Scheduler:
    def run_test(self, test, target_devices):
        requests = []
        for dev in target_devices:
            requests.extend(self.resource_graph.expand_test_policy(test.name, dev, test.args))

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

Conflict check:

```python
def conflicts(existing: ResourceRequest, requested: ResourceRequest) -> bool:
    if existing.resource != requested.resource:
        return False
    return LeaseMode.EXCLUSIVE in (existing.mode, requested.mode) or \
           LeaseMode.GROUP_EXCLUSIVE in (existing.mode, requested.mode)
```

Expand `group_exclusive` requests before conflict checking, or have `LeaseManager` call `ResourceGraph.expand_group()` internally. Without expansion, a lease on `fabric:fabric_0` will not conflict with a request for `chip:CHIP_0` even when that chip belongs to the fabric group.

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

Use the device-local queue for same-logical-device serialization. Use the LeaseManager for shared physical resources that span multiple logical devices.

## HAL Interfaces

Expose session-scoped operations. A session must carry an active scheduler lease.

```cpp
struct DeviceId {
    std::string value;  // Stable logical ID such as CHIP_0.
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
1. Scheduler acquires exclusive reset_domain lease.
2. Scheduler blocks new submits for affected resources.
3. HAL calls Quiesce() for affected sessions.
4. SubmitQueueManager drains or cancels in-flight jobs.
5. HAL verifies no owned descriptors remain active.
6. Kernel retains only DMA buffer/mmap ownership unless heavy-kmod mode is enabled.
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
  "resources": ["chip:CHIP_0", "hbm:CHIP_0", "ring:CHIP_0:dma0"],
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
      resource_graph.py
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

- Define logical devices and topology-derived shared resources.
- Require every testcase to declare resource requests before execution.
- Validate policies in dry-run mode.
- Implement lease conflict detection before running any test.
- Serialize same-logical-device tasks with a device work queue.
- Lock shared physical resources with scheduler leases.
- Centralize hardware queue submission in HAL by default.
- Track session/job/descriptor/buffer ownership in HAL, and buffer/mmap ownership in the thin kernel module.
- Enforce DMA buffer cleanup in kernel file release; handle submit crash safety through supervised HAL ownership or future heavy-kmod mode.
- Require exclusive leases for reset and fabric reconfiguration.
- Return structured job/resource errors.
