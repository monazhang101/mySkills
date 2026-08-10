# UMD and HAL Boundary

Use this reference when a platform has a User Mode Driver (UMD) that directly operates firmware descriptors, command queues, completion queues, doorbells, and DMA submit mechanics.

## Core Position

UMD is a low-level user-space driver. HAL is the diagnostic semantic layer above it.

```text
Python framework
  -> test orchestration, policy, scheduling, reports

C++ HAL
  -> stable diagnostic APIs and result semantics
  -> wraps UMD and hides descriptor/register/FW details from Python

UMD / user-mode driver
  -> firmware protocol, descriptors, command rings, completion rings
  -> BAR/MMIO doorbells or firmware mailbox writes
  -> DMA submit mechanics, polling, IRQ wait, timeout, cancel, drain

kernel module
  -> ultra-thin privileged services: contiguous memory allocation/mmap and IRQ event delivery
```

Do not expose UMD descriptor APIs directly to Python testcases. Python should call HAL operations such as `run_atomic_test("pcie_dma_data_transfer", "TPU_0", options)`.

## UMD Responsibilities

UMD owns mechanism-level hardware control:

- Open and close device sessions.
- Map or access BAR/MMIO or a firmware command portal through the platform-approved path.
- Allocate host-visible buffers through the ko, VFIO, vendor SDK, or existing driver.
- Partition buffers into descriptor rings, completion rings, payload buffers, and scratch buffers.
- Construct firmware descriptors according to the FW ABI.
- Maintain command ring head/tail, descriptor ownership, sequence IDs, and queue state.
- Issue memory barriers before doorbell/mailbox writes.
- Ring doorbells or submit mailbox commands.
- Poll completion memory or wait for IRQ/eventfd notifications.
- Attribute completions to the correct session, queue, descriptor, and test.
- Implement timeout, cancel, drain, quiesce, and reset preparation.
- Convert FW status codes into structured low-level errors.

UMD should not own platform policy, testcase orchestration, final diagnostic classification, or report generation.

## HAL Responsibilities

HAL owns diagnostic semantics:

- Device discovery and logical naming.
- Topology and domain mapping.
- Telemetry readout and event normalization.
- Stable atomic APIs such as `memory_selftest`, `pcie_dma_data_transfer`, `compute_smoke`, `pcie_link_status_check`, and `reset_check`.
- Conversion from UMD/FW errors into framework error domains.
- Artifact creation for descriptor dumps, completion logs, telemetry snapshots, and FW traces.
- A mockable boundary for Python bindings and fake HAL tests.

HAL can live in the same shared library as UMD during early bring-up, but preserve the conceptual boundary in headers and APIs.

## Descriptor Address Rule

UMD must not put a user-space CPU virtual pointer into a firmware descriptor DMA address field.

```text
CPU virtual address:
  pointer returned by mmap, usable only by the UMD process

device_addr:
  address the device/FW can use in descriptors
  may be PHYS, BUS, IOVA, or FW_VA depending on platform mode
```

The memory provider should return both CPU and device views:

```cpp
enum class AddressKind {
    kPhys,
    kBus,
    kIova,
    kFwVa,
};

struct DeviceMemory {
    void* cpu_va;
    uint64_t device_addr;
    AddressKind addr_kind;
    size_t bytes;
    uint64_t handle;
};
```

Descriptor construction should only use `device_addr`:

```cpp
desc.src_addr = buffer.device_addr + src_offset;
desc.dst_addr = buffer.device_addr + dst_offset;
desc.completion_addr = buffer.device_addr + completion_offset;
```

When IOMMU is disabled for MVP, `device_addr` may be a physical or bus address. When IOMMU/VFIO/SMMU support is added later, `device_addr` can become an IOVA without changing queue or descriptor construction logic.

## Ultra-Thin Kmod Flow

```text
1. UMD opens /dev/tpu_diag.
2. UMD requests contiguous memory allocation.
3. ko returns handle, mmap_offset, host_phys_addr, device_addr, addr_kind, and cache metadata.
4. UMD mmaps the handle and initializes descriptor/completion/data regions.
5. UMD fills FW descriptors using device_addr plus offsets.
6. UMD writes MMIO doorbell or FW mailbox.
7. FW/device executes the command and writes completion.
8. UMD polls completion or waits on ko IRQ event.
9. HAL converts UMD result into PrimitiveResult/TestResult.
10. UMD frees buffers or ko cleans them up on file release.
```

## Suggested UMD API

```cpp
class IUmdDevice {
public:
    virtual Result<void> Open(const DeviceId& id) = 0;
    virtual Result<DeviceMemory> AllocateMemory(size_t bytes, MemoryOptions options) = 0;
    virtual Result<QueueHandle> CreateQueue(const QueueOptions& options) = 0;
    virtual Result<JobId> SubmitDescriptor(QueueHandle queue, const FwDescriptor& desc) = 0;
    virtual Result<Completion> WaitCompletion(JobId job, Timeout timeout) = 0;
    virtual Result<void> Cancel(JobId job) = 0;
    virtual Result<void> Drain(QueueHandle queue) = 0;
    virtual Result<void> Close() = 0;
};
```

HAL should expose coarser APIs over this:

```cpp
hal.run_atomic_test("pcie_dma_data_transfer", DeviceId{"TPU_0"}, options);
hal.run_atomic_test("memory_selftest", DeviceId{"TPU_0"}, options);
```

## Open Design Checks

Ask these before freezing the ABI:

- Does FW descriptor require PHYS, BUS, IOVA, FW_VA, or a handle/offset pair?
- Is IOMMU disabled, passthrough/identity-mapped, or fully translating on the target platform?
- Is the device coherent with CPU caches, or must UMD/ko/FW perform flush/invalidate?
- How is BAR/MMIO or the firmware command portal exposed to UMD?
- Are completions primarily polled, IRQ-driven, or both?
- What is the reset protocol for in-flight descriptors?
- What cleanup is required if the UMD process crashes while descriptors are in flight?

## Recommended MVP

1. Keep ko to contiguous allocation/mmap plus IRQ request/event delivery.
2. Implement a UMD memory provider returning `cpu_va`, `device_addr`, and `addr_kind`.
3. Implement UMD queue create, descriptor submit, completion wait, timeout, cancel, and drain.
4. Wrap UMD in HAL primitives; keep Python coarse and descriptor-free.
5. Preserve `device_addr`/`addr_kind` so future IOMMU/VFIO/SMMU support swaps only the memory provider.
