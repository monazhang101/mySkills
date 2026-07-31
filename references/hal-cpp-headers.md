# C++ HAL Header Blueprint

Use this as the default starting point for an implementation-ready C++17 HAL. Keep these headers stable enough for Python bindings and test mocks, but do not treat the C++ ABI as stable across shared-library boundaries unless a separate C ABI is added.

## Design Rules

- Use C++17, RAII, `std::chrono`, `std::optional`, `std::variant`, and `std::shared_ptr`.
- Return `Result<T>` instead of throwing across HAL boundaries. Internal implementations may throw only if they catch and convert to `Error`.
- Use `DeviceId` as the stable handle in public APIs. Do not expose raw file descriptors, ioctl structs, register offsets, or vendor SDK handles.
- Make service interfaces mockable: pure virtual interfaces, factory functions, no global singleton requirement.
- Keep Python binding coarse: expose `IHal`, `IDeviceManager`, `ITelemetryService`, and `IPrimitiveRunner` first.
- Keep units explicit in field names: `_c`, `_mv`, `_watts`, `_bytes`, `_bps`, `_us`.

## Suggested Header Files

```text
hal/include/aidiag/
  api.h
  error.h
  result.h
  ids.h
  dma.h
  device.h
  topology.h
  telemetry.h
  primitive.h
  platform.h
  hal.h
```

## `api.h`

```cpp
#pragma once

#if defined(_WIN32)
#  if defined(AIDIAG_HAL_BUILD)
#    define AIDIAG_API __declspec(dllexport)
#  else
#    define AIDIAG_API __declspec(dllimport)
#  endif
#else
#  define AIDIAG_API __attribute__((visibility("default")))
#endif

#define AIDIAG_HAL_VERSION_MAJOR 0
#define AIDIAG_HAL_VERSION_MINOR 1
#define AIDIAG_HAL_VERSION_PATCH 0
```

## `error.h`

```cpp
#pragma once

#include <cstdint>
#include <string>

namespace aidiag {

enum class ErrorDomain {
    kNone,
    kUserConfig,
    kPolicy,
    kFramework,
    kHal,
    kKernel,
    kDriver,
    kVendorSdk,
    kTool,
    kHardware,
    kTimeout,
    kUnsupported,
};

enum class ErrorCode : int32_t {
    kOk = 0,
    kUnknown = 1,
    kInvalidArgument = 2,
    kNotFound = 3,
    kPermissionDenied = 4,
    kUnavailable = 5,
    kTimeout = 6,
    kUnsupported = 7,
    kIoError = 8,
    kProtocolError = 9,
    kDeviceUnhealthy = 10,
};

struct Error {
    ErrorDomain domain = ErrorDomain::kNone;
    ErrorCode code = ErrorCode::kOk;
    std::string message;
    std::string detail;

    static Error Ok() { return {}; }
    explicit operator bool() const { return code != ErrorCode::kOk; }
};

}  // namespace aidiag
```

## `result.h`

```cpp
#pragma once

#include <optional>
#include <utility>

#include "aidiag/error.h"

namespace aidiag {

template <typename T>
class Result {
public:
    Result(T value) : value_(std::move(value)) {}
    Result(Error error) : error_(std::move(error)) {}

    bool ok() const { return value_.has_value() && !error_; }
    explicit operator bool() const { return ok(); }

    const T& value() const { return *value_; }
    T& value() { return *value_; }
    const Error& error() const { return error_; }

private:
    std::optional<T> value_;
    Error error_;
};

template <>
class Result<void> {
public:
    Result() = default;
    Result(Error error) : error_(std::move(error)) {}

    bool ok() const { return !error_; }
    explicit operator bool() const { return ok(); }
    const Error& error() const { return error_; }

private:
    Error error_;
};

}  // namespace aidiag
```

## `ids.h`

```cpp
#pragma once

#include <cstdint>
#include <optional>
#include <string>

namespace aidiag {

struct DeviceId {
    std::string value;  // Stable logical ID such as CHIP_0, NVME_3, NIC_1.
};

struct PciAddress {
    uint16_t domain = 0;
    uint8_t bus = 0;
    uint8_t device = 0;
    uint8_t function = 0;

    std::string ToBdf() const;
};

struct PhysicalId {
    std::optional<PciAddress> pci;
    std::string sysfs_path;
    std::string serial;
    std::string fru;
};

}  // namespace aidiag
```

## `device.h`

```cpp
#pragma once

#include <cstdint>
#include <map>
#include <optional>
#include <string>
#include <vector>

#include "aidiag/ids.h"

namespace aidiag {

enum class DeviceType {
    kUnknown,
    kAccelerator,
    kMemory,
    kSwitch,
    kPcieBridge,
    kNvme,
    kNic,
    kCpu,
    kDimm,
    kFan,
    kPsu,
    kBmc,
    kSensor,
};

enum class DeviceHealth {
    kUnknown,
    kOk,
    kDegraded,
    kFault,
    kAbsent,
};

enum class Capability {
    kTelemetry,
    kEventSubscribe,
    kReset,
    kPcieLinkCheck,
    kMemorySelfTest,
    kDmaLoopback,
    kStress,
    kFirmwareQuery,
    kInventory,
};

struct DeviceInfo {
    DeviceId id;
    DeviceType type = DeviceType::kUnknown;
    DeviceHealth health = DeviceHealth::kUnknown;
    PhysicalId physical;

    std::string logical_name;   // Human-readable alias. Usually same as id.value.
    std::string slot;           // OAM1, SXM3, PCIeSlot4, DriveBay7.
    std::string model;
    std::string vendor;
    std::string firmware_version;
    std::optional<int> numa_node;

    std::vector<Capability> capabilities;
    std::map<std::string, std::string> attributes;
};

struct DeviceFilter {
    std::optional<DeviceType> type;
    std::optional<DeviceHealth> min_health;
    std::vector<DeviceId> only;
    bool include_absent = false;
};

class IDeviceManager {
public:
    virtual ~IDeviceManager() = default;

    virtual Result<std::vector<DeviceInfo>> Discover(const DeviceFilter& filter = {}) = 0;
    virtual Result<DeviceInfo> Get(const DeviceId& id) = 0;
    virtual Result<std::vector<DeviceId>> ListByType(DeviceType type) = 0;
    virtual Result<void> Refresh() = 0;
};

}  // namespace aidiag
```

## `dma.h`

```cpp
#pragma once

#include <cstddef>
#include <cstdint>
#include <memory>

#include "aidiag/ids.h"
#include "aidiag/result.h"

namespace aidiag {

enum class DmaDirection {
    kBidirectional,
    kToDevice,
    kFromDevice,
};

struct DmaBufferOptions {
    size_t bytes = 0;
    DmaDirection direction = DmaDirection::kBidirectional;
    bool coherent = true;
    uint64_t alignment_bytes = 4096;
};

class IDmaBuffer {
public:
    virtual ~IDmaBuffer() = default;

    virtual void* CpuPtr() = 0;
    virtual uint64_t DmaAddress() const = 0;  // Device-visible DMA address / IOVA.
    virtual size_t SizeBytes() const = 0;
    virtual Result<void> SyncForCpu() = 0;
    virtual Result<void> SyncForDevice() = 0;
};

class IDmaAllocator {
public:
    virtual ~IDmaAllocator() = default;

    virtual Result<std::shared_ptr<IDmaBuffer>> Allocate(const DeviceId& id,
                                                         const DmaBufferOptions& options) = 0;
};

}  // namespace aidiag
```

## `topology.h`

```cpp
#pragma once

#include <cstdint>
#include <map>
#include <string>
#include <vector>

#include "aidiag/device.h"
#include "aidiag/ids.h"

namespace aidiag {

enum class LinkType {
    kUnknown,
    kPcie,
    kChipToChip,
    kChipToSwitch,
    kNvlinkLike,
    kNetwork,
    kManagement,
    kPower,
    kThermal,
};

struct LinkInfo {
    LinkType type = LinkType::kUnknown;
    DeviceId src;
    DeviceId dst;
    std::string name;
    uint32_t width = 0;
    double speed_gbps = 0.0;
    bool active = false;
    std::map<std::string, std::string> attributes;
};

struct Topology {
    std::vector<DeviceInfo> devices;
    std::vector<LinkInfo> links;
};

class ITopologyService {
public:
    virtual ~ITopologyService() = default;

    virtual Result<Topology> GetTopology() = 0;
    virtual Result<std::vector<DeviceId>> Neighbors(const DeviceId& id, LinkType type) = 0;
    virtual Result<std::vector<DeviceId>> DevicesInResetDomain(const DeviceId& id) = 0;
    virtual Result<std::string> ResourceDomain(const DeviceId& id, const std::string& domain_kind) = 0;
};

}  // namespace aidiag
```

## `telemetry.h`

```cpp
#pragma once

#include <chrono>
#include <cstdint>
#include <functional>
#include <map>
#include <memory>
#include <string>
#include <variant>
#include <vector>

#include "aidiag/ids.h"
#include "aidiag/result.h"

namespace aidiag {

using MetricValue = std::variant<int64_t, double, bool, std::string>;

struct MetricSample {
    std::string name;
    MetricValue value;
    std::string unit;  // C, mV, W, bytes, errors, percent.
    std::chrono::system_clock::time_point timestamp;
    std::map<std::string, std::string> labels;
};

struct TelemetrySnapshot {
    DeviceId device;
    std::vector<MetricSample> metrics;
};

struct TelemetrySelector {
    DeviceId device;
    std::vector<std::string> metric_names;
};

struct Event {
    DeviceId device;
    std::string type;
    std::string severity;
    std::string message;
    std::chrono::system_clock::time_point timestamp;
    std::map<std::string, std::string> attributes;
};

class ISubscription {
public:
    virtual ~ISubscription() = default;
    virtual void Cancel() = 0;
};

using EventCallback = std::function<void(const Event&)>;

class ITelemetryService {
public:
    virtual ~ITelemetryService() = default;

    virtual Result<TelemetrySnapshot> Read(const TelemetrySelector& selector) = 0;
    virtual Result<std::vector<std::string>> ListMetrics(const DeviceId& device) = 0;
    virtual Result<std::shared_ptr<ISubscription>> Subscribe(const DeviceId& device,
                                                             EventCallback callback) = 0;
};

}  // namespace aidiag
```

## `primitive.h`

```cpp
#pragma once

#include <chrono>
#include <map>
#include <string>
#include <vector>

#include "aidiag/ids.h"
#include "aidiag/result.h"
#include "aidiag/telemetry.h"

namespace aidiag {

enum class TestStatus {
    kPass,
    kFail,
    kSkip,
    kError,
    kTimeout,
    kUnsupported,
};

struct Artifact {
    std::string name;
    std::string path;
    std::string mime_type;
    std::map<std::string, std::string> attributes;
};

struct PrimitiveRequest {
    std::string name;  // memory_selftest, dma_loopback, pcie_link_check, reset_check.
    DeviceId device;
    std::map<std::string, std::string> options;
    std::chrono::seconds timeout{0};
};

struct PrimitiveResult {
    std::string name;
    DeviceId device;
    TestStatus status = TestStatus::kError;
    Error error;
    std::string diagnosis;
    std::chrono::system_clock::time_point start_time;
    std::chrono::system_clock::time_point end_time;
    std::vector<MetricSample> metrics;
    std::vector<Artifact> artifacts;
};

class IPrimitiveRunner {
public:
    virtual ~IPrimitiveRunner() = default;

    virtual Result<std::vector<std::string>> ListPrimitives(const DeviceId& device) = 0;
    virtual Result<PrimitiveResult> Run(const PrimitiveRequest& request) = 0;
};

}  // namespace aidiag
```

## `platform.h`

```cpp
#pragma once

#include <map>
#include <optional>
#include <string>
#include <vector>

#include "aidiag/device.h"
#include "aidiag/result.h"

namespace aidiag {

struct PlatformInfo {
    std::string name;
    std::string sku;
    std::string board_id;
    std::string chassis_serial;
    std::string policy_file;
    std::map<std::string, std::string> attributes;
};

struct Threshold {
    std::string metric;
    std::optional<double> min;
    std::optional<double> max;
    std::string unit;
};

struct ResourceLock {
    std::string name;  // chip:CHIP_0, bmc, pcie_domain:root0.
    bool exclusive = true;
};

class IPlatformService {
public:
    virtual ~IPlatformService() = default;

    virtual Result<PlatformInfo> GetPlatformInfo() = 0;
    virtual Result<DeviceId> MapPhysicalToLogical(const PhysicalId& physical, DeviceType type) = 0;
    virtual Result<std::vector<Threshold>> GetThresholds(const DeviceId& id) = 0;
    virtual Result<std::vector<ResourceLock>> GetResourceLocks(const std::string& operation,
                                                               const DeviceId& id) = 0;
};

}  // namespace aidiag
```

## `hal.h`

```cpp
#pragma once

#include <map>
#include <memory>
#include <string>

#include "aidiag/api.h"
#include "aidiag/dma.h"
#include "aidiag/device.h"
#include "aidiag/platform.h"
#include "aidiag/primitive.h"
#include "aidiag/telemetry.h"
#include "aidiag/topology.h"

namespace aidiag {

struct HalConfig {
    std::string policy_file;
    std::string log_dir;
    bool use_kernel_transport = true;
    bool use_vendor_sdk = true;
    bool dry_run = false;
    std::map<std::string, std::string> options;
};

class IHal {
public:
    virtual ~IHal() = default;

    virtual IDeviceManager& Devices() = 0;
    virtual ITopologyService& Topology() = 0;
    virtual ITelemetryService& Telemetry() = 0;
    virtual IDmaAllocator& Dma() = 0;
    virtual IPrimitiveRunner& Primitives() = 0;
    virtual IPlatformService& Platform() = 0;

    virtual Result<void> Initialize(const HalConfig& config) = 0;
    virtual Result<void> Shutdown() = 0;
};

AIDIAG_API std::shared_ptr<IHal> CreateHal();
AIDIAG_API std::shared_ptr<IHal> CreateFakeHal();

}  // namespace aidiag
```

## Python Binding Shape

Expose the same service boundaries without leaking C++ implementation details:

```python
hal = aidiag.create_hal()
hal.initialize({"policy_file": "policies/platform_x.yaml"})

devices = hal.devices.discover({"type": "accelerator"})
topology = hal.topology.get_topology()
snapshot = hal.telemetry.read({"device": "CHIP_0", "metric_names": ["temp_c", "power_watts"]})
result = hal.primitives.run({
    "name": "memory_selftest",
    "device": "CHIP_0",
    "options": {"pattern": "walking_ones", "duration_s": "60"},
    "timeout_s": 120,
})
```

## Implementation Notes

- Implement `CreateFakeHal()` first so Python runner and policy logic can be tested without hardware.
- Implement `sysfs_backend` before kernel transport when possible. This quickly validates discovery, PCI identity, NUMA, NVMe, and hwmon telemetry.
- Keep vendor SDK calls behind backend classes; do not let them appear in public headers.
- Keep register-level tests as HAL primitives, not Python scripts.
- Convert all backend errors into `ErrorDomain` and `ErrorCode` at the HAL boundary.
- Document thread safety: service methods should be safe for concurrent calls unless a method explicitly says otherwise; callbacks must not be invoked while holding internal locks.
