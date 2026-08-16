# A silent thermal watchdog on Linux: reading temps from sysfs, not lm-sensors

**Date:** 2026-08-16
**Tags:** `linux`, `monitoring`, `sysfs`, `automation`

## The problem

I wanted a thermal watchdog for my laptop: alert when CPU/GPU/NVMe/Wi-Fi run hot, stay quiet otherwise, and cost nothing to run. `lm-sensors` is the usual answer, but it's a userspace layer over the same kernel interface — and for a script, the raw interface is simpler and has zero install dependencies.

## What I tried

First I read the kernel's thermal zones directly:

```bash
for z in /sys/class/thermal/thermal_zone*; do
  echo "$(basename $z): $(cat $z/type) = $(( $(cat $z/temp) / 1000 ))C"
done
```

That works, but zone numbering is *not* stable or meaningful — on this machine `thermal_zone18` is `iwlwifi_1` and `thermal_zone19` is `x86_pkg_temp`, while zones 3–17 are `SEN0..SEND` mystery sensors from the ACPI table. Two gotchas:

- **Zone indices drift between boots** — match on `type`, never on the number.
- **`iwlwifi_1` was the trap** — I enumerated zones with a loop variable and hit the classic variable-swap bug (using the *type* where the *path* was needed), so the Wi-Fi temp silently read as the CPU's. The tell: the alert printed the same number twice.

NVMe and fans don't appear in the thermal zones at all — they live under hwmon:

```bash
for h in /sys/class/hwmon/hwmon*; do echo "$(basename $h): $(cat $h/name)"; done
# hwmon3: nvme   hwmon4: thinkpad   hwmon8: coretemp  ...
```

Fans are plain integers on this box: `hwmon4/fan1_input` → `3262` RPM, `fan2_input` → `2979` RPM.

## What worked

A watchdog script that is **silent when healthy** — no "all good" spam — and only prints an alert when a threshold trips:

```bash
$ python3 thermal_watch.py; echo "exit=$?"     # healthy: prints nothing, exit 0
$ THERMAL_TEST=1 python3 thermal_watch.py       # forced alert branch
🌡️ THERMAL ALERT
🔴 CPU pkg: 110.0°C  (CRITICAL, crit 95)
🔴 GPU: 110.0°C  (CRITICAL, crit 95)
🔴 NVMe: 90.0°C  (CRITICAL, crit 75)
🔴 Wi-Fi: 110.0°C  (CRITICAL, crit 95)
Fans: fan1 3264 RPM | fan2 2983 RPM
```

`THERMAL_TEST=1` is a test override that keeps the real thresholds (crit 95/95/75/95) but forces every temperature past them, so the alert path can be exercised on demand — and the output wording matches production exactly. Fan reads are alert-only: they appear in alerts but a read failure never blocks the alert itself.

## Takeaway

Linux exposes every temperature and fan as a plain file under `/sys/class/thermal` and `/sys/class/hwmon` — no daemon needed, but match zones by `type`, not by index, and add a `THERMAL_TEST` env override so you can prove your alert path works before it has to.
