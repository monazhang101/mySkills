# Thin Kernel Module Boundary

Use this reference when deciding how much belongs in the kernel module versus UMD and the C++ HAL.

## Recommendation

Prefer the thinnest viable kernel module. There are two common MVP shapes.

Default thin-kmod mode:

```text
kernel module
  -> DMA-safe memory allocation
  -> IOMMU mapping / device-visible DMA address or IOVA
  -> mmap buffer to user space
  -> free and optional cache sync

C++ HAL
  -> PCI discovery and logical naming
  -> BAR/MMIO mapping and register access
  -> PCI config space access
  -> DMA engine programming
  -> polling completion
  -> diagnostic primitive semantics
```

Ultra-thin bring-up mode, when the platform owner explicitly says the ko should not own DMA or IOMMU:

```text
kernel module
  -> contiguous physical memory allocation
  -> mmap buffer to user space
  -> free on explicit request or file release
  -> request_irq / release_irq
  -> poll/read/eventfd-style IRQ event delivery

UMD / user-mode driver
  -> firmware descriptor construction
  -> command/completion ring ownership
  -> DMA buffer layout and descriptor address fields
  -> BAR/MMIO doorbell or firmware mailbox writes
  -> polling, IRQ wait, timeout, cancel, drain, and completion attribution

C++ HAL
  -> diagnostic semantics and stable APIs for Python
  -> device discovery, topology, telemetry, primitive result conversion
  -> wraps UMD instead of exposing descriptor mechanics to Python
```

Do not put high-level diagnostics, platform policy, topology interpretation, register semantics, or test orchestration in the kernel module.

## Terminology

- DMA is device-driven memory access. The device reads or writes host memory without CPU copy loops.
- IOMMU is the device-side address translation and protection layer. The address written into a device DMA register is often an IOVA, not a raw CPU physical address.
- IOVA is the IO virtual address that a device uses when IOMMU translation is enabled. It is not the same thing as a user-space CPU virtual address.
- UMD means User Mode Driver: a low-level user-space driver that may own firmware descriptors, queues, MMIO doorbells, and DMA submit mechanics.
- BAR/MMIO is the PCIe device register/window mapping that the CPU uses to control the device.
- PCI config space is the standardized PCIe discovery/configuration area containing vendor ID, device ID, BAR info, link capability, MSI/MSI-X capability, AER capability, and power management capability.

## Interpret "Only Allocate Physical Memory"

When a stakeholder says the ko layer should "only allocate physical memory", translate that into:

Ultra-thin interpretation:

```text
Allocate contiguous physical memory and return:
  - a handle for lifetime management
  - an mmap offset so UMD can obtain a CPU virtual pointer
  - a device_addr that UMD can place in firmware descriptors for this MVP
  - an addr_kind that says whether device_addr is PHYS, BUS, IOVA, or FW_VA
  - size, alignment, and cache/sync metadata
```

Default DMA-safe interpretation:

```text
Allocate DMA-safe memory and return:
  - a handle for lifetime management
  - a CPU mapping usable by UMD/HAL, usually via mmap
  - a device-visible DMA address / IOVA for device registers
  - size and cache/sync metadata
```

Do not let UMD place a user-space CPU virtual address in a firmware descriptor DMA address field. On IOMMU systems, the device-visible address may differ from host physical memory. Even when IOMMU is disabled, CPU virtual address, host physical address, bus address, and firmware-visible address are still different concepts.

## Minimal UAPI

```c
#define AI_DIAG_GET_VERSION     _IOR(AI_DIAG_IOC_MAGIC, 0x00, struct ai_diag_version)
#define AI_DIAG_ATTACH_DEVICE   _IOWR(AI_DIAG_IOC_MAGIC, 0x01, struct ai_diag_attach_device)
#define AI_DIAG_CONTIG_ALLOC    _IOWR(AI_DIAG_IOC_MAGIC, 0x20, struct ai_diag_contig_alloc)
#define AI_DIAG_CONTIG_FREE     _IOW(AI_DIAG_IOC_MAGIC, 0x21, struct ai_diag_contig_free)
#define AI_DIAG_IRQ_REQUEST     _IOWR(AI_DIAG_IOC_MAGIC, 0x30, struct ai_diag_irq_request)
#define AI_DIAG_IRQ_RELEASE     _IOW(AI_DIAG_IOC_MAGIC, 0x31, struct ai_diag_irq_release)
```

Suggested fields:

```c
enum ai_diag_addr_kind {
    AI_DIAG_ADDR_PHYS = 1,
    AI_DIAG_ADDR_BUS  = 2,
    AI_DIAG_ADDR_IOVA = 3,
    AI_DIAG_ADDR_FW_VA = 4,
};

struct ai_diag_contig_alloc {
    __u32 size;
    __u32 version;
    __u64 flags;
    __u64 bytes;
    __u64 handle;
    __u64 host_phys_addr; /* for bring-up/debug; UMD should not depend on this forever */
    __u64 device_addr;    /* address UMD writes into firmware descriptors */
    __u32 addr_kind;      /* enum ai_diag_addr_kind */
    __u32 cache_policy;
    __u64 mmap_offset;  /* offset passed to mmap */
    __u64 reserved[8];
};

struct ai_diag_irq_request {
    __u32 size;
    __u32 version;
    __u32 irq_index;
    __u32 flags;
    __s32 event_fd;      /* optional eventfd; -1 if using read/poll */
    __u64 reserved[8];
};
```

If the project later adds IOMMU mapping, keep the same conceptual contract and return `device_addr` with `addr_kind = AI_DIAG_ADDR_IOVA`. UMD descriptor construction should not need to change.

## HAL Interfaces

```cpp
class IDmaBuffer {
public:
    virtual ~IDmaBuffer() = default;
    virtual void* CpuPtr() = 0;
    virtual uint64_t DeviceAddress() const = 0;  // PHYS/BUS/IOVA/FW_VA depending on addr_kind.
    virtual std::string AddressKind() const = 0;
    virtual size_t SizeBytes() const = 0;
};

class IDmaAllocator {
public:
    virtual ~IDmaAllocator() = default;
    virtual Result<std::shared_ptr<IDmaBuffer>> Allocate(DeviceId id, size_t bytes) = 0;
};

class IMmioRegion {
public:
    virtual ~IMmioRegion() = default;
    virtual Result<uint32_t> Read32(uint64_t offset) = 0;
    virtual Result<void> Write32(uint64_t offset, uint32_t value) = 0;
};

class IPciAccess {
public:
    virtual ~IPciAccess() = default;
    virtual Result<std::shared_ptr<IMmioRegion>> MapBar(DeviceId id, int bar) = 0;
    virtual Result<uint32_t> ReadConfig32(DeviceId id, uint16_t offset) = 0;
};
```

## When to Add More Kernel Capability

Add kernel-mediated BAR/config access only if VFIO, sysfs, vendor SDK, or controlled user-space mapping is not safe or not available.

Add IRQ/event delivery when polling is insufficient for long-running tests, error notification, or performance-sensitive flows. In the ultra-thin model, IRQ request/event delivery is allowed early while descriptor and DMA mechanics remain in UMD.

Add reset only when user-space reset paths are unreliable or require privileged sequencing that should not live in HAL.

Add telemetry only when telemetry cannot be obtained from vendor SDK, hwmon, sysfs, BMC, or safe MMIO reads.

## Preferred MVP

1. Try no-kmod or VFIO mode first.
2. If platform guidance requires an ultra-thin ko, add only contiguous memory allocation/mmap plus IRQ request/event delivery.
3. Keep DMA engine programming, firmware descriptors, doorbells, polling, completion attribution, cancel, and drain in UMD or HAL.
4. Preserve `device_addr`/`addr_kind` in UAPI even when IOMMU is disabled, so IOMMU/VFIO/SMMU support can replace the memory provider later.
5. Add reset, telemetry, BAR/config mediation, or IOMMU mapping later based on measured need.
