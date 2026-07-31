# Debian Packaging and Runtime Layout

Use this reference when designing how the diagnostic framework is built, installed, upgraded, and called by an upper-layer CLI.

## Packaging Layer

Add a first-class packaging/release layer:

```text
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
  logrotate/
    ai-diag
build/
  CMakeLists.txt
  pyproject.toml
  scripts/
    build_deb.sh
    collect_artifacts.sh
    package_version.py
```

Kconfig is only for kernel/module compile-time configuration. It is not a replacement for Debian packaging.

## Install Layout

Install artifacts into predictable locations:

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

Optional kernel-module files:

```text
/usr/src/ai-diag-mem-<version>/
/lib/modules/<kernel>/extra/ai_diag_mem.ko
/etc/udev/rules.d/99-ai-diag.rules
```

## Package Modes

Support two modes when possible:

```text
no-kmod
  -> CLI + Python framework + HAL + policies/specs
  -> use VFIO, vendor SDK, sysfs, BMC, and external tools only

dkms-kmod
  -> everything in no-kmod
  -> add ai_diag_mem DKMS source and install hooks
  -> build/load thin DMA memory module on target kernel
```

Prefer `no-kmod` for early development, fake HAL, external tool tests, inventory, and telemetry that does not require custom DMA buffers.

Prefer `dkms-kmod` when the framework needs custom DMA-safe buffer allocation/mapping and existing drivers/VFIO cannot provide it cleanly.

## Debian Control Guidance

Typical dependencies:

```text
Depends:
  python3,
  python3-yaml,
  python3-click or python3-argparse equivalent,
  libstdc++6,
  pciutils,
  nvme-cli,
  fio
```

Add these only for kmod mode:

```text
Depends:
  dkms,
  linux-headers-generic or matching platform kernel headers
```

Use `Recommends` or separate packages for large optional tools such as stress tools, vendor field diagnostics, IB tools, or burn-in payloads.

## Maintainer Scripts

`postinst` should:

- Register/build/install DKMS module when kmod mode is enabled.
- Run `depmod` when installing a prebuilt ko.
- Load `ai_diag_mem` if policy allows automatic loading.
- Install or reload udev rules.
- Create log/config directories with expected ownership and permissions.
- Run a lightweight install smoke check such as `ai-diag --version` and `ai-diag doctor --no-hardware`.

`prerm` should:

- Stop monitor services.
- Unload `ai_diag_mem` only if safe and unused.
- Remove DKMS module registration for package removal.

`postrm` should:

- Clean generated runtime state only on purge.
- Preserve logs and user config unless purging.

## CLI Contract

The upper-layer CLI should call one stable entry point:

```text
/usr/bin/ai-diag
```

Recommended commands:

```text
ai-diag --version
ai-diag doctor
ai-diag inventory --json
ai-diag run --spec quick
ai-diag run --spec burnin --policy platform_x
ai-diag report show <run-id>
```

Keep the CLI independent of source-tree paths. Resolve installed policies/specs through `/usr/share/ai-diag` and user overrides through `/etc/ai-diag`.

## Build Flow

Recommended build stages:

1. Build HAL shared library with CMake.
2. Build Python package and pybind module.
3. Build or stage optional DKMS source for `ai_diag_mem`.
4. Validate policy/spec schemas.
5. Assemble Debian package.
6. Install package in a clean container or VM.
7. Run smoke tests:
   - `ai-diag --version`
   - `ai-diag doctor --no-hardware`
   - `ai-diag inventory --fake-hal --json`

## Release Artifacts

Publish:

- `.deb`
- build manifest with git SHA, build timestamp, toolchain versions
- symbols or debug package for HAL
- schema version list
- default policy/spec bundle version
- install smoke-test log

Keep package versioning aligned across CLI, HAL, Python bindings, policy schema, and optional kmod UAPI.
