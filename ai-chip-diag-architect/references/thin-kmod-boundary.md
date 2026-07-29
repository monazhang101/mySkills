# Thin Kernel Module Boundary

Use this reference when deciding how much belongs in the kernel module versus the C++ HAL.

## Recommendation

Prefer the thinnest viable kernel module:

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

Do not put high-level diagnostics, platform policy, topology interpretation, register semantics, or test orchestration in the kernel module.

## Terminology

- DMA is device-driven memory access. The device reads or writes host memory without CPU copy loops.
- IOMMU is the device-side address translation and protection layer. The address written into a device DMA register is often an IOVA, not a raw CPU physical address.
- BAR/MMIO is the PCIe device register/window mapping that the CPU uses to control the device.
- PCI config space is the standardized PCIe discovery/configuration area containing vendor ID, device ID, BAR info, link capability, MSI/MSI-X capability, AER capability, and power management capability.

## Interpret "Only Allocate Physical Memory"

When a stakeholder says the ko layer should "only allocate physical memory", translate that into:

```text
Allocate DMA-safe memory and return:
  - a handle for lifetime management
  - a CPU mapping usable by HAL, usually via mmap
  - a device-visible DMA address / IOVA for device registers
  - size and cache/sync metadata
```

Do not expose or rely on guessed CPU physical addresses. On IOMMU systems, the device-visible address may differ from host physical memory.

## Minimal UAPI

```c
#define AI_DIAG_GET_VERSION     _IOR(AI_DIAG_IOC_MAGIC, 0x00, struct ai_diag_version)
#define AI_DIAG_ATTACH_DEVICE   _IOWR(AI_DIAG_IOC_MAGIC, 0x01, struct ai_diag_attach_device)
#define AI_DIAG_DMA_ALLOC       _IOWR(AI_DIAG_IOC_MAGIC, 0x20, struct ai_diag_dma_alloc)
#define AI_DIAG_DMA_FREE        _IOW(AI_DIAG_IOC_MAGIC, 0x21, struct ai_diag_dma_free)
#define AI_DIAG_DMA_SYNC        _IOW(AI_DIAG_IOC_MAGIC, 0x22, struct ai_diag_dma_sync)
```

Suggested fields:

```c
struct ai_diag_dma_alloc {
    __u32 size;
    __u32 version;
    __u64 flags;
    __u64 bytes;
    __u64 handle;
    __u64 dma_addr;     /* device-visible DMA address / IOVA */
    __u64 mmap_offset;  /* offset passed to mmap */
    __u64 reserved[8];
};
```

## HAL Interfaces

```cpp
class IDmaBuffer {
public:
    virtual ~IDmaBuffer() = default;
    virtual void* CpuPtr() = 0;
    virtual uint64_t DmaAddress() const = 0;  // IOVA/device-visible address.
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

Add IRQ/event delivery only when polling is insufficient for long-running tests, error notification, or performance-sensitive flows.

Add reset only when user-space reset paths are unreliable or require privileged sequencing that should not live in HAL.

Add telemetry only when telemetry cannot be obtained from vendor SDK, hwmon, sysfs, BMC, or safe MMIO reads.

## Preferred MVP

1. Try no-kmod or VFIO mode first.
2. Add `ai_diag_mem.ko` only for DMA-safe buffer allocation/mapping.
3. Keep DMA engine programming and polling in HAL.
4. Add IRQ/event/reset later based on measured need.
