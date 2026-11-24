## Can Kepler Consume Energy Consumption From a File?

### Short Answer

**No first-class "read energy/power from a file" backend exists today.** However, there
is a clean, supported workaround: use the `hwmon` backend with a redirected sysfs root
(`--host.sysfs`) and have an external process maintain a fake hwmon-style directory tree.

### Why There Is No Direct File Source

Kepler has exactly three CPU meter backends, dispatched in
`buildCPUMeter()` (`internal/device/cpu_power_meter.go:96-143`) and validated in
`config/config.go:990-1002` (`validCpuMeters = {rapl, hwmon, fake}`):

| Value   | Constructor                                 | Reads from                        |
| ------- | ------------------------------------------- | --------------------------------- |
| `rapl`  | `NewCPUPowerMeter(cfg.Host.SysFS, ...)`     | `<sysfs>/class/powercap/intel-rapl*` |
| `hwmon` | `NewHwmonPowerMeter(cfg.Host.SysFS, ...)`   | `<sysfs>/class/hwmon`             |
| `fake`  | `NewFakeCPUMeter(cfg.Dev.FakeCpuMeter.Zones, ...)` | synthetic (in-memory, random) |

- Config key: `cpu.preferredMeters` (`config/config.go:61,299`), default `["rapl", "hwmon"]`.
- There is **no** generic file-based source.
- There is **no** plugin / exec / command mechanism (the only `os/exec` usage is in
  `test/e2e/*`, not runtime).
- There is **no** general external input. Redfish is HTTP-to-BMC only
  (`internal/platform/redfish/`) and does **not** feed the CPU energy-zone pipeline; it
  also requires a BMC endpoint plus a credentials config file.

#### The `fake` meter does not help

`internal/device/fake_cpu_power_meter.go` is purely synthetic and fixed at startup:

- `Energy()` computes an in-memory value with a random component
  (`fake_cpu_power_meter.go:52-60`); it never reads a file.
- The `path` field is a cosmetic label only — returned by `Path()`, never opened.
- `WithFakePath()` only rewrites that label string and is not wired through `buildCPUMeter`.
- Zones/increments/maxEnergy are set once in `NewFakeCPUMeter`; `Init()` is a no-op.
- It cannot be driven externally at runtime.

### The Viable Workaround: Fake hwmon Tree + Redirected sysfs Root

The one lever available is `host.sysfs` (`--host.sysfs`, default `/sys`,
`config/config.go:53,291,500`). It is only validated as a readable directory
(`canReadDir`, `config/config.go:960-967`), so a fake directory is accepted. Both `rapl`
and `hwmon` resolve their base path from it. `hwmon` is the simpler target.

#### Minimal fake hwmon tree (energy-based, recommended)

```
testrootstock/class/hwmon/hwmon0/name           # e.g. "externalmeter"
testrootstock/class/hwmon/hwmon0/energy1_input  # cumulative microjoules, monotonically increasing
testrootstock/class/hwmon/hwmon0/energy1_label  # optional, zone name e.g. "package"
```

#### Run Kepler pointed at it

```yaml
cpu:
  preferredMeters: ["hwmon"]
host:
  sysfs: /testrootstock
  procfs: /proc          # MUST stay real — needed for pod/process attribution
```

Or via flags:

```bash
kepler --host.sysfs=/path/to/fakeroot --config kepler.yaml
```

Kepler reads `energy1_input` on every monitor interval (default 5s,
`config/config.go:365`) and derives power as deltaEnergy / deltaTime — exactly how RAPL
works.

### Discovery / Parsing Rules (So the Fake Tree Is Accepted)

From `internal/device/hwmon_power_meter.go`:

- Base path: `filepath.Join(sysfsPath, "class", "hwmon")` (`:95,278`).
- `Zones()` iterates `hwmonN` sub-dirs (`:287-320`); `discoverZones` uses a 3-tier strategy
  (`:322-388`), using the first tier that yields zones:
  1. **Energy (preferred, cumulative µJ):** matches regex `^energy(\d+)_(.+)$`
     (`:392-416`); requires `energyN_input` (`:479-483`); read as µJ
     (`hwmonEnergyZone.Energy()`, `:1057-1070`). `MaxEnergy()` returns 0 for hwmon
     energy zones (`:1072-1076`). Optional `energyN_label` sets the zone name;
     optional `energyN_enable` (0 disables, `:813-836`).
  2. **Power (instantaneous µW):** regex `^power(\d+)_(.+)$`; needs `powerN_average`
     or `powerN_input` (`:438-446`); read as µW (`hwmonPowerZone.Power()`, `:994-1009`).
  3. **Voltage x Current fallback:** `inN_input` (mV) x `currN_input` (mA)
     (`:510-637,1083-1113`).
- Chip name resolution (`getChipName`, `:846-891`): tries a `device` symlink, then a
  `name` file, then the dir name. Provide a plain `name` file to keep it simple.
- File reads use a raw `unix.Read` of up to 128 bytes (`sysReadFile`, `:924-947`); value
  is `TrimSpace` + `ParseUint`, so each file must contain a plain unsigned integer.

### Critical Requirements & Caveats

- `energy1_input` must be a **cumulative, monotonically increasing** counter in
  **microjoules**. (Power is computed from deltas by the monitor.)
- Alternatively use `power1_input` as an **instantaneous microwatt** value if you have
  power rather than cumulative energy.
- This only fakes the **node/zone energy input**. The attribution engine
  (process -> container -> pod, split by CPU time from `/proc`) runs normally and is
  unaffected.
- `host.procfs` must still point at a real `/proc`.
- RAPL can likewise be redirected via `--host.sysfs`, but you must faithfully reproduce
  the `intel-rapl` powercap layout (`name`, `energy_uj`, `max_energy_range_uj`, path
  containing `/intel-rapl:`, see `rapl_sysfs_power_meter.go:231-252`). hwmon is the
  recommended, simpler path.

### Bottom Line

| Capability                                            | Status                         |
| ----------------------------------------------------- | ------------------------------ |
| First-class "read energy from arbitrary file" backend | Not available today            |
| Plugin / exec / command power source                  | Not available                  |
| HTTP external source                                  | Redfish only (BMC, not a file) |
| `fake` meter driven from a file at runtime            | Not possible (synthetic/random)|
| Fake hwmon tree + `--host.sysfs` redirect             | **Works today** (workaround)   |

### Key File References

- Backend dispatch: `internal/device/cpu_power_meter.go:96-143`
- Valid meters / defaults: `config/config.go:358-360,990-1002`
- hwmon reader: `internal/device/hwmon_power_meter.go:95,287-388,924-947,994-1009,1057-1076`
- RAPL reader: `internal/device/rapl_sysfs_power_meter.go:53-54,231-252`
- Fake meter: `internal/device/fake_cpu_power_meter.go:52-95,116-162`
- sysfs config: `config/config.go:53,291,500,960-967`
- Monitor interval: `config/config.go:365`
- Backend docs: `docs/user/configuration.md:263-296`
