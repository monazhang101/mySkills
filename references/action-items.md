# AI Chip Diag Action Items

Use this file as the living action-item tracker for the AI chip diagnostic framework. Keep module-specific follow-ups here so Python framework, HAL, UMD, kernel, packaging, and testcase design decisions can be reviewed without scattering TODOs across reference files.

When a module design becomes stable, move durable guidance into that module's dedicated reference file. Keep unresolved questions and next-step decisions here.

## 1. Python Framework

Current design reference: `python-framework-mvp.md`.

Status: MVP structure agreed. Remaining work is to make config, result, lease, and first testcase contracts precise enough for implementation.

### 1.1 Define `test_config.yaml` Schema

Goal: make the user-facing test config precise enough that CLI, MFG station, and daemon entrypoints can all run the same sequences.

Decisions to record:

- Required top-level fields: `sequence`, `tests`, `devices`, optional `platform`, `defaults`, `report`.
- Device entry shape: `name`, `type`, `parents`, `locator`, optional `slot`; `name` is the canonical top-level target ID used by CLI targets, testcase config, leases, HAL atomic calls, and result reports.
- Locator match keys and precedence, for example `pci_bdf`, `devnode`, `sysfs_path`, `serial`, `slot`, or BMC path; HAL discovery returns observed hardware facts, and the Python framework matches observed hardware to policy locators before presenting canonical device names.
- Sequence entry shape: `name`, `devices`, `options`, `timeout_s`, `continue_on_fail`, `tags`.
- Per-test config shape: `options`, `thresholds`, `locks`, `timeout_s`, `retry`.
- How device templates are expanded, for example `${device}`.
- Default values and invalid-config error behavior.

Expected output:

```text
schemas/test_config.schema.yaml
examples/connectivity_check.yaml
examples/production_sequence.yaml
```

### 1.2 Define HAL `atomic_test` Catalog

Goal: agree on the first set of HAL atomic tests that Python testcases can compose.

Decision: first implementation focuses on single-device `run_atomic_test(...)`. Reserve `run_group_atomic_test(...)` as a future extension for strongly synchronized multi-device diagnostics. Python will choose and lease the target device list; HAL will validate and execute concurrently over exactly the devices provided.

Initial candidates:

- `identify`
- `read_fru`
- `bar_probe`
- `register_block_probe`
- `pcie_link_status_check`
- `isi_link_up`
- `power_sanity`
- `topology_snapshot`
- `telemetry_snapshot`
- `memory_get_inventory`
- `memory_selftest`
- `error_counter_snapshot`
- `pcie_dma_data_transfer`

For each atomic test, record:

- Name and purpose.
- Whether it is single-device only or may later support group execution.
- Required target type, usually top-level `TPU` for TPU atomic tests. Use `PCIE_DOMAIN`, `POWER_DOMAIN`, or other domain targets only when the domain must be independently selected or locked.
- Input options.
- Output fields and metrics.
- Error codes and failure meanings.
- Whether it is read-only, disruptive, reset-sensitive, or stress-like.

Expected output:

```text
hal_atomic_tests.md
```

### 1.3 Define Result Status and Error Model

Goal: make atomic test, testcase, and sequence reports consistent.

Decisions to record:

- Status enum: `pass`, `fail`, `skip`, `error`, `timeout`, `unsupported`.
- Error domains: `user_config`, `framework`, `lease`, `hal`, `umd`, `kernel`, `tool`, `hardware`.
- Required result fields: `name`, `type`, `device/devices`, `status`, `start_time`, `end_time`, `duration_s`, `metrics`, `error_code`, `diagnosis`, `artifacts`, `subresults`.
- How HAL atomic results are embedded inside testcase results.
- How sequence-level exit code is computed.

Expected output:

```text
result_model.md
```

### 1.4 Define Lease Semantics

Goal: keep resource locking simple but explicit enough to protect shared device/domain resources.

Decision: MVP leases are cross-process, but the implementation stays as a simple lease gate inside `runner.py`. Enhanced `LeaseManager` behavior can come later.

Decisions to record:

- Lock modes: `shared` and `exclusive`.
- MVP lease owner format, for example `run_id:testcase:device`.
- Lock expansion for `${device}`.
- Domain examples: `PCIE_DOMAIN_0`, `POWER_DOMAIN_0`, `RESET_DOMAIN_0`, `FABRIC_0`, `TELEMETRY_0`.
- Minimal cross-process coordination mechanism, timeout behavior, owner metadata, and failure behavior.
- Later enhancement criteria for moving lease code from `runner.py` into `lease_manager.py`, such as heartbeats, stale cleanup, queueing, priority, or richer concurrency.

Expected output:

```text
lease_model_mvp.md
```

### 1.5 Write `connectivity_check.py` Testcase Template

Goal: make the first real testcase the reference style for all future Python testcases.

The testcase should demonstrate:

- Gate check with `identify` or another cheap status/identity atomic test.
- Lease use through `ctx.lease(...)`.
- Composing multiple HAL atomic tests.
- Threshold comparison from config.
- Aggregate result with `subresults`.
- Clean diagnosis strings for field/debug use.

Expected atomic calls:

```python
ctx.run_atomic_test("identify", device=device)
ctx.run_atomic_test("isi_link_up", device=device, args={"links": "all"})
ctx.run_atomic_test("pcie_link_status_check", device=device, args={"depth": "all"})
ctx.run_atomic_test("power_sanity", device=device, args={"workload": "short"})
```

Expected output:

```text
aidiag/tests/connectivity_check.py
examples/connectivity_check.yaml
```

### Python Framework Review Order

Recommended order for the next design pass:

1. HAL `atomic_test` catalog.
2. Result and error model.
3. Lease semantics.
4. `test_config.yaml` schema.
5. `connectivity_check.py` template.

The HAL/UMD boundary should be reviewed before finalizing the atomic test catalog, because atomic test outputs depend on what HAL can reliably discover, read, or execute through UMD.

## 2. HAL

Status: To be expanded during HAL design review.

Initial placeholders:

- Define HAL module split: device manager, topology manager, telemetry manager, atomic test runner, error/result manager, UMD adapter.
- Define public C++ API and Python binding naming, especially `run_atomic_test`.
- Define first atomic test catalog inputs/outputs with structured result fields.
- Define how HAL maps UMD/FW/kernel errors into framework error domains.
- Define fake HAL behavior for Python MVP and CI.

## 3. UMD

Status: To be expanded during UMD design review.

Initial placeholders:

- Finalize UMD three-block diagram: Memory & Address Manager, Queue/Descriptor Manager, Doorbell/Completion/IRQ Manager.
- Define memory object contract: `cpu_va`, `device_addr`, `addr_kind`, size, handle, cache attributes.
- Define queue/session ownership and descriptor lifecycle.
- Define completion attribution, timeout, cancel, drain, and reset preparation.
- Define process-crash cleanup expectations.

## 4. Kernel Module

Status: To be expanded during kernel boundary review.

Initial placeholders:

- Confirm ultra-thin kmod scope: contiguous physical memory allocation, mmap, request_irq/event delivery.
- Confirm whether kmod returns PHYS, BUS, IOVA, or FW_VA as `device_addr`.
- Define versioned UAPI structs with `size`, `version`, `flags`, and reserved fields.
- Define file-release cleanup semantics.

## 5. Packaging And Release

Status: To be expanded when MVP install/run path is ready.

Initial placeholders:

- Define runtime install layout for CLI, Python package, HAL shared library, UMD library, optional kmod, configs, schemas, reports, logs.
- Decide deb packaging modes: no-kmod and DKMS-kmod if needed.
- Define smoke test after install.
