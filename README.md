# Supermicro IPMI Fan Curve

A small temperature-based fan controller and reusable Ansible role for
Supermicro motherboards with compatible two-zone IPMI controls. Runs locally
on the managed server; no BMC network credentials are needed.

Verified BMC interface: **X12SCZ-F**, BIOS 3.5, BMC 1.08.06, Ubuntu with systemd.
Other boards and firmware are not verified. The role rejects unlisted boards;
extend `supermicro_fan_supported_boards` only after verifying command support
and the fan-zone mapping on that hardware.

## Fan Zones

| Zone | Headers | Configuration |
|---|---|---|
| `0` | Numbered FAN1-FAN4 headers | `CPU_PWM` |
| `1` | FANA/FANB | `FRONT_PWM` |

**A PWM value controls a zone, not one connector.** Adding a low-speed case fan
to the numbered zone makes the CPU-only minimum unsafe for that fan. The minimum
must suit the slowest fan in the zone. Header labels and SDR names must be
checked on the actual board; an unused header reports `No Reading`.

The [LANCOOL III example](docs/X12SCZ-F.md) has wiring, a host-specific curve
and measured RPM.

## Role Default Curve

| CPU temperature | Zone 0 | Zone 1 |
|---|---:|---:|
| Up to 54 C | 50% | 50% |
| 55-75 C | 60% | 70% |
| 76-80 C | 80% | 90% |
| 81-85 C | 90% | 100% |
| 86 C and above | 100% | 100% |

Ansible uses this fallback only when a host has no `supermicro_fan_curve` override.
The LANCOOL III example shows a host profile for a specific machine.
The controller has no built-in curve: it rejects an empty configuration file.
Validate minimum RPM, temperatures and BMC events for every connected fan.
The controller also enforces 100% at 90 C regardless of configured temperature
boundaries; the final curve stage must request 100% on both zones.

The role and `fans.yml` leave BMC alarm thresholds unchanged unless inventory
sets `supermicro_fan_thresholds`. The role reads before writing and verifies
changed values; check mode does not write.

- Reads CPU temperature every five seconds; rising demand applies immediately.
- Reduces one stage at a time after 30 seconds below a four-degree hysteresis boundary.
- Reasserts PWM every 30 seconds without re-entering Full Speed on each update.
- Three consecutive failed temperature reads request Full Speed; the request
  repeats every 30 seconds until a valid reading returns.
- Startup and controlled shutdown request Full Speed. systemd also attempts Full
  Speed after unexpected service termination, then restarts the controller.
- IPMI calls have an eight-second timeout. A dead BMC cannot be controlled by software;
  fail-safe writes are best-effort, not a substitute for hardware thermal protection.

## Deploy With Ansible

Controller requirements: Ansible Core 2.14+ and SSH access with sudo.
Target requirements: compatible Supermicro BMC, Debian/Ubuntu, Bash and systemd.
Ansible installs the distribution's `ipmitool` package and manages only these files:

```text
/usr/local/sbin/supermicro-fan-curve
/etc/default/supermicro-fan-curve
/etc/systemd/system/supermicro-fan-curve.service
```

For `smicro`, copy the inventory example and edit its address and SSH account.
The role supplies the default curve. Private inventory and live host variables
are ignored by Git.

```bash
cp inventory/hosts.example inventory/hosts
# Edit inventory/hosts with the smicro address and SSH account.
ansible-playbook fans.yml --diff --check
ansible-playbook fans.yml --diff
```

On a fresh host, check mode may stop before packages or the service exist.
The normal run installs `ipmitool` and the service.

For another host, set its inventory alias. Put curve or fan-threshold overrides
in `inventory/host_vars/<alias>.yml` if needed.
Host overrides use the `supermicro_fan_*` variables in
[`roles/supermicro_fan_curve/defaults/main.yml`](roles/supermicro_fan_curve/defaults/main.yml).
Set `supermicro_fan_curve` to a list of `upper_c`, `cpu_pwm`, `front_pwm` stages.
Temperature bounds must increase, PWM must not decrease, and the final values
must both be 100. Always validate the minimum against every connected fan.

Changed installation files are backed up on the target before replacement.
The service is enabled at boot and starts after `rc-local.service`. It does not
rewrite `/etc/rc.local`, flash firmware, or stop applications. Only the configured
lower-critical/non-recoverable fan thresholds are managed; other thresholds and
hysteresis are not changed.
Remove old fixed-PWM startup commands from your own configuration once ownership
has been transferred; do not run competing fan controllers.

## Inspect And Operate

Run these commands on the managed server:

```bash
sudo ipmitool sensor get 'CPU Temp'
sudo ipmitool sdr type Fan
sudo ipmitool sensor get FAN2
sudo ipmitool sensor get FAN3
sudo ipmitool sensor get FAN4
sudo ipmitool sensor get FANA
sudo ipmitool sel elist last 20
sudo systemctl status supermicro-fan-curve --no-pager
sudo journalctl -fu supermicro-fan-curve
```

For continuous temperature, RPM and PWM readings, run this on the managed server:

```bash
while true; do
  clear
  date
  sudo ipmitool sdr type Temperature
  sudo ipmitool sdr type Fan
  for zone in 0 1; do
    hex=$(sudo ipmitool raw 0x30 0x70 0x66 0x00 "$zone") || continue
    hex=$(printf '%s' "$hex" | tr -d '[:space:]')
    printf 'Zone %s PWM: %d%%\n' "$zone" "$((16#$hex))"
  done
  sleep 10
done
```

Sensor output reports RPM, not requested PWM. SEL records fan threshold assertions
and deassertions. Repeated stop/recover events indicate a bad minimum, a stalled
fan or a hardware problem; do not silence them by disabling alarms.

Read fan mode and requested zone PWM without changing them:

```bash
sudo ipmitool raw 0x30 0x45 0x00
sudo ipmitool raw 0x30 0x70 0x66 0x00 0x00
sudo ipmitool raw 0x30 0x70 0x66 0x00 0x01
```

Values are hexadecimal percentages: `32` means 50%, `64` means 100%.
The controller uses these writes; do not run manual writes against an active controller:

| Command | Meaning on verified hardware |
|---|---|
| `ipmitool raw 0x30 0x45 0x01 0x01` | Select BMC Full Speed mode before manual zone writes |
| `ipmitool raw 0x30 0x70 0x66 0x01 0x00 0x32` | Set the numbered/CPU zone to 50% |
| `ipmitool raw 0x30 0x70 0x66 0x01 0x01 0x32` | Set the peripheral zone to 50% |

Emergency fallback, at the cost of noise:

```bash
sudo systemctl disable --now supermicro-fan-curve
sudo ipmitool raw 0x30 0x45 0x01 0x01
```

Validate the generated configuration without touching the BMC:

```bash
sudo /usr/local/sbin/supermicro-fan-curve --check
```

Success exits with status 0 and no output.

For persistent curve changes, update Ansible variables and rerun the playbook.
Hand edits in `/etc/default/supermicro-fan-curve` will be replaced by Ansible.

## Checks

```bash
bash -n roles/supermicro_fan_curve/files/supermicro-fan-curve
ansible-playbook fans.yml --diff --syntax-check
```

These syntax checks do not touch real fans.
After deployment check real RPM/temperatures over several reassert cycles and
rerun Ansible expecting `changed=0`. Idle checks do not establish cooling under full load.

LANCOOL III load measurements and validation limits are in the
[hardware reference](docs/X12SCZ-F.md).

## Vendor References

- [Supported fan modes in the X12SCZ family](https://www.supermicro.com/en/support/faqs/faq.php?faq=39466)
- [Numbered versus peripheral fan groups on Supermicro boards](https://www.supermicro.com/en/support/faqs/faq.php?faq=13433)
- [Supermicro warning about model-specific raw fan commands](https://www.supermicro.com/en/support/faqs/faq.php?faq=39476)

The raw command interface is OEM- and firmware-specific, not a portable IPMI
standard. Supermicro recommends its own SMCIPMITool/IPMICFG for unsupported
raw-command behavior. Do not assume a different motherboard uses these bytes.
