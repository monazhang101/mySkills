# Python Framework MVP

Use this reference to implement the first Python orchestration layer for the AI chip diagnostic tool. The MVP should stay small, but keep the boundaries clean enough that CLI, MFG station, or daemon entrypoints can reuse the same execution engine.

## Core Architecture

Recommended MVP layout:

```text
aidiag/
  main.py
  config.py
  runner.py
  tests/
    __init__.py
    connectivity_check.py
```

Add more testcase files under `tests/` as needed, for example `skucheck.py`, `tpu_mem.py`, `tpu_dma.py`, `nvme_fio.py`, or `cpu_stress.py`.

Layering:

```text
Entry Adapter
  main.py
    -> CLI parsing and command dispatch only

Execution Engine
  runner.py
    -> reusable by CLI, MFG station, daemon, or service API

Test Config
  config.py
    -> parse user test config and expose sequence/options/thresholds/locks

Testcases
  tests/*.py
    -> compose HAL atomic tests and external tools into diagnostic testcases

C++ HAL
  hal.discover()
    -> observed components and topology scan
  hal.run_atomic_test(...)
    -> atomic hardware checks and actions

External Tools
  runner.run_tool(...)
    -> fio, stress-ng, nvme, iostat, ipmitool, vendor utilities
```

## Responsibility Split

`main.py` is a CLI adapter. It should parse user commands, load config, create a `Runner`, and call the proper runner method. Keep it thin because future systems may bypass CLI entirely.

```text
main.py
  -> parse: aidiag discover
  -> parse: aidiag atomic pcie_link_status --device TPU_0
  -> parse: aidiag test connectivity_check --device TPU_0
  -> parse: aidiag run --config production.yaml
  -> call Runner.discover(...)
  -> call Runner.run_atomic_test(...)
  -> call Runner.run_testcase(...)
  -> call Runner.run_sequence(...)
```

`config.py` owns user config parsing. It should not call HAL and should not execute tests.

```text
config.py
  -> load YAML/JSON
  -> validate required keys
  -> return sequence entries
  -> return testcase options
  -> return thresholds
  -> return timeout
  -> return LogicalDevice locks
```

`runner.py` is the reusable execution engine. It exists so CLI, MFG station, daemon, or REST service can all execute diagnostics without duplicating lease, result, report, and helper logic.

```text
runner.py
  -> own HAL handle
  -> own run context
  -> run hardware discovery
  -> run one HAL atomic test
  -> run one testcase
  -> run a configured sequence
  -> provide lease helper
  -> provide external tool helper
  -> normalize and collect results
  -> write report
```

`tests/*.py` contains diagnostic testcase orchestration. A testcase may do gate checks, acquire leases through the runner/context, call several HAL atomic tests, call external tools, and combine results into one diagnostic result.

```text
tests/connectivity_check.py
  -> decide which connectivity checks compose the testcase
  -> call HAL atomic tests through runner/context
  -> aggregate pass/fail and diagnosis
```

## HAL Discovery Interface

For MVP, do not expose `initialize()` as a user-visible CLI flow. A command such as `aidiag discover` should create the HAL client and call `hal.discover(...)`; HAL can lazily do any internal setup needed to scan the system.

Recommended HAL binding shape:

```python
hal.discover() -> dict
hal.get_device(name: str) -> dict
hal.get_devices_by_type(type: str) -> list[str]
```

`discover()` returns observed components only. Do not include absent devices or `presence=false` entries. Missing devices are detected by the `skucheck` testcase by comparing expected topology/config against the discovered result.

For MVP, `name` is the canonical LogicalDevice ID used by CLI targets, testcase config, lease requests, HAL atomic calls, and result reports. The platform policy should pin each stable `name` to real hardware using `locator` fields such as `pci_bdf`, `devnode`, `sysfs_path`, `serial`, `slot`, or BMC path. HAL discovery should return observed hardware facts. The Python framework should match observed devices against policy locators and resolve the canonical `name`; do not assign `TPU_0`, `TPU_1`, and similar names from raw discovery order.

Minimum discovery result:

```python
{
    "devices": [
        {
            "name": "TPU_0",
            "type": "TPU",
            "slot": "OAM0",
            "locator": {
                "pci_bdf": "0000:81:00.0",
                "devnode": "/dev/tpu0"
            }
        }
    ],
    "topology": [
        {
            "parent": "CPU_0",
            "child": "PCIE_SWITCH_0",
            "type": "pcie"
        },
        {
            "parent": "PCIE_SWITCH_0",
            "child": "TPU_0",
            "type": "pcie"
        }
    ]
}
```

Field meaning:

```text
name      -> canonical LogicalDevice ID and normal test target, for example TPU_0
type      -> TPU, SSD, CPU, PCIE_SWITCH, BMC, etc.
slot      -> simple human-facing location string; may be empty if unknown
locator   -> machine-facing access info used to match policy to observed hardware, such as pci_bdf, devnode, sysfs path, serial, slot, or BMC path
topology  -> parent/child edges used by CLI tree output and topology-aware tests
```

Do not put firmware version, serial number, PCIe link status, health, or telemetry into `discover()`. Use `run_atomic_test("identify", device)` for identity details, `skucheck.py` for expected-vs-observed inventory/version checks, and `connectivity_check.py` for link/power/connection health.

## HAL Atomic Test Interface

Use `atomic_test` terminology at the Python/HAL boundary.

Recommended HAL binding shape:

```python
hal.run_atomic_test(
    test_name: str,
    device: str,
    args: dict | None = None,
    timeout_s: int | None = None,
) -> dict
```

Examples:

```python
hal.run_atomic_test("identify", "TPU_0")
hal.run_atomic_test("isi_presence", "TPU_0")
hal.run_atomic_test("pcie_link_status", "TPU_0")
hal.run_atomic_test("power_connection_status", "TPU_0")
hal.run_atomic_test("memory_selftest", "TPU_0", args={"pattern": "walking_ones"})
hal.run_atomic_test("dma_loopback", "TPU_0", args={"bytes": 1048576})
```

HAL owns the hardware facts and low-level semantics. Python must not directly access registers, firmware descriptors, BAR/MMIO, UMD queue APIs, or kernel ioctls.

For MVP, keep one `device` per atomic test call. If a future HAL operation truly requires synchronized multi-device execution, add a separate group API later, such as `hal.run_group_atomic_test(...)`, instead of complicating the first `run_atomic_test(...)` contract.

Reserve the group atomic test shape for later:

```python
hal.run_group_atomic_test(
    test_name: str,
    devices: list[str],
    args: dict | None = None,
    timeout_s: int | None = None,
) -> dict
```

Python decides the target device list from CLI/config/policy, acquires leases covering every target and affected shared domain, and passes the list to HAL. HAL does not choose which devices to stress. It validates the given devices and runs the atomic test concurrently over exactly that set. This is intended for strongly synchronized diagnostics such as multi-TPU stress, fabric stress, collective bandwidth, thermal stress, or synchronized power stress, where HAL/UMD should own internal workers, start barriers, fail policy, timeout, cancel, drain, and cleanup. Do not implement this for the MVP unless a real synchronized multi-device test requires it.

Atomic test result should be structured:

```python
{
    "name": "pcie_link_status",
    "device": "TPU_0",
    "status": "pass",
    "metrics": {
        "speed_gtps": 32,
        "width": 16
    },
    "error_code": "OK",
    "diagnosis": "PCIe link is x16 Gen5",
    "artifacts": []
}
```

## Connectivity Check Example

The user config defines when and how to run `connectivity_check`:

```yaml
sequence:
  - name: connectivity_check
    devices: ["TPU_0", "TPU_1"]

tests:
  connectivity_check:
    timeout_s: 120
    options:
      expected_pcie_width: 16
      expected_pcie_speed_gtps: 32
    locks:
      shared:
        - "${device}"
        - "PCIE_DOMAIN_0"
        - "POWER_DOMAIN_0"
      exclusive: []
```

CLI entry:

```text
aidiag test connectivity_check --device TPU_0
```

Execution flow:

```text
main.py
  -> parse CLI args
  -> load config
  -> create Runner
  -> Runner.run_testcase("connectivity_check", devices=["TPU_0"])

runner.py
  -> load connectivity_check testcase
  -> create context
  -> pass config/options to testcase
  -> collect result
  -> write report

tests/connectivity_check.py
  -> with context.lease(["TPU_0", "PCIE_DOMAIN_0", "POWER_DOMAIN_0"], shared=True)
  -> context.hal.run_atomic_test("isi_presence", "TPU_0")
  -> context.hal.run_atomic_test("pcie_link_status", "TPU_0")
  -> context.hal.run_atomic_test("power_connection_status", "TPU_0")
  -> compare metrics against config thresholds
  -> return one connectivity_check result
```

Important boundary:

```text
connectivity_check.py decides what to check.
HAL decides how each atomic hardware check is performed.
runner.py provides the reusable execution harness.
main.py only adapts CLI input into runner calls.
```

## Runner API Shape

Keep `Runner` small, but make it the only Python execution engine used by CLI and future entrypoints.

```python
class Runner:
    def __init__(self, config: dict, hal: object, tests_package: str = "aidiag.tests"): ...

    def discover(self) -> dict: ...

    def run_atomic_test(
        self,
        test_name: str,
        device: str,
        args: dict | None = None,
        timeout_s: int | None = None,
    ) -> dict: ...

    def run_testcase(
        self,
        name: str,
        devices: list[str],
        options: dict | None = None,
    ) -> list[dict]: ...

    def run_sequence(self, sequence: list[dict]) -> list[dict]: ...

    def lease(
        self,
        shared: list[str] | None = None,
        exclusive: list[str] | None = None,
        owner: str | None = None,
        timeout_s: int | None = None,
    ): ...

    def run_tool(
        self,
        argv: list[str],
        timeout_s: int,
        log_path: str,
        env: dict | None = None,
        cwd: str | None = None,
    ) -> dict: ...

    def write_report(self, results: list[dict], output_path: str) -> None: ...
```

`Runner.run_atomic_test()` is a Python convenience wrapper around `hal.run_atomic_test()`. The HAL API remains the source of hardware truth.

## Testcase API Shape

Each testcase module should expose a stable name and a `run()` function. Keep the interface plain for MVP.

`tests/connectivity_check.py`:

```python
name = "connectivity_check"

def run(context, devices: list[str], options: dict) -> list[dict]: ...
```

The `context` object should provide:

```text
context.hal
context.config
context.runner
context.lease(...)
context.run_atomic_test(...)
context.run_tool(...)
context.log_dir
```

Avoid making testcase modules own global HAL clients or global lease state. They should use the context passed by the runner.

`context` is not special Python syntax. It is a normal object created by the runner for the current diagnostic run. The purpose is to bundle shared services so testcase functions do not need long parameter lists or their own global clients.

Conceptually:

```python
context = RunnerContext(
    hal=hal,
    config=config,
    runner=runner,
    log_dir=log_dir,
)

connectivity_check.run(context, devices=["TPU_0"], options={...})
```

Many Python projects shorten `context` to `ctx` by convention, but this is only a naming convention. `ctx` is not short for `connectivity_check`.

```python
def run(ctx, devices: list[str], options: dict) -> list[dict]:
    ...
```

For clarity in design docs and early code, prefer the full name:

```python
def run(context, devices: list[str], options: dict) -> list[dict]:
    ...
```

## Config API Shape

`config.py` should be boring and predictable:

```python
def load_config(path: str) -> dict: ...
def get_sequence(config: dict, only_test: str | None = None, devices: list[str] | None = None) -> list[dict]: ...
def get_test_config(config: dict, test_name: str) -> dict: ...
def get_options(config: dict, test_name: str) -> dict: ...
def get_thresholds(config: dict, test_name: str, device: str | None = None) -> dict: ...
def get_timeout_s(config: dict, test_name: str) -> int: ...
def get_locks(config: dict, test_name: str, device: str) -> dict: ...
```

Keep SKU-specific rules in config at first. Split `config.py` into a richer `policy_engine.py` only when platform variants, lock groups, expected inventory, thresholds, and support rules become too large for one config helper.

## CLI Modes

Support four CLI modes from the start:

```text
aidiag discover
  -> calls Runner.discover()

aidiag atomic <atomic_test_name> --device <device>
  -> calls Runner.run_atomic_test()

aidiag test <testcase_name> --device <device>
  -> calls Runner.run_testcase()

aidiag run --config <config.yaml>
  -> calls Runner.run_sequence()
```

This gives bring-up engineers a direct low-level path for quick checks while preserving full testcase and sequence execution.

## Lease Guidance

For MVP, testcase code may choose what to lease, but it must use the runner/context lease helper. Do not reimplement lease mechanics inside each testcase.

The MVP `LeaseManager` must provide cross-process protection so multiple CLI instances, MFG station workers, or daemon entrypoints cannot accidentally touch conflicting LogicalDevices at the same time. Keep the implementation small: shared/exclusive locks over expanded LogicalDevice IDs, deterministic acquisition order, owner metadata, timeout behavior, and best-effort stale lease cleanup are enough. Do not add queueing, priority, fairness, worker pools, or DAG scheduling for the MVP.

The cross-process lease helper may live inside `runner.py` initially. Split it into `lease_manager.py` only when heartbeats, stale cleanup, wait policy, persistence, or concurrent execution make `runner.py` hard to read.

Recommended pattern:

```python
with context.lease(shared=["TPU_0", "PCIE_DOMAIN_0", "POWER_DOMAIN_0"], owner=name):
    isi = context.run_atomic_test("isi_presence", "TPU_0")
    pcie = context.run_atomic_test("pcie_link_status", "TPU_0")
    power = context.run_atomic_test("power_connection_status", "TPU_0")
```

## Result and Report Guidance

Every atomic test and testcase should return a structured dictionary with at least:

```text
name
type: atomic_test or testcase
device or devices
status: pass, fail, skip, error, timeout, unsupported
start_time
end_time
duration_s
metrics
error_code
diagnosis
artifacts
```

`connectivity_check.py` should return both the aggregate result and enough sub-results to explain failures:

```python
{
    "name": "connectivity_check",
    "type": "testcase",
    "device": "TPU_0",
    "status": "fail",
    "diagnosis": "PCIe width mismatch",
    "subresults": [
        {"name": "isi_presence", "status": "pass"},
        {"name": "pcie_link_status", "status": "fail", "metrics": {"width": 8}},
        {"name": "power_connection_status", "status": "pass"}
    ]
}
```

## When To Split More Modules

Keep the MVP compact. Split only when there is pressure from real code:

```text
runner.py -> lease_manager.py
  when cross-process lease acquisition, heartbeats, stale cleanup, wait policy, persistence, or concurrent execution make runner.py hard to read

runner.py -> result_store.py
  when report generation grows beyond JSON/text summaries

runner.py -> tool_adapter.py
  when external tools need dedicated parsers and cleanup policies

config.py -> policy_engine.py
  when SKU/topology/threshold/lock rules become more than simple config lookups

tests/common.py -> richer testcase helpers
  when common testcase validation, retry, skip, or subresult logic becomes repetitive
```

Do not split just to make the architecture look complete. Split when it removes repeated code or isolates a real lifecycle boundary.
