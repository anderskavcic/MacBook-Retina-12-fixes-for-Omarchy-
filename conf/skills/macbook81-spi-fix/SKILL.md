---
name: macbook81-spi-fix
description: Apply the MacBook8,1 (MacBook Retina 12-inch, Early 2015) keyboard, touchpad and suspend fix on Omarchy 4 — two kernel parameters via a limine drop-in. Use when the built-in keyboard and/or touchpad are dead on a 12-inch MacBook, when applespi times out with -110, when Apple SPI Touchpad is missing from /proc/bus/input/devices, or when asked to apply the MacBook8,1 SPI fix. Verifies hardware first, requires explicit confirmation before writing, and never reboots on its own. Does NOT cover Wi-Fi or hibernation.
---

# MacBook8,1 SPI fix (Omarchy 4)

Applies two kernel parameters that make the built-in keyboard, touchpad and suspend work.
**Verify → confirm → apply → hand off the reboot.** Never reboot the machine yourself.

## Scope

- **In scope:** built-in keyboard, touchpad, suspend (s2idle), on **Omarchy 4** with **limine + UKI**.
- **Out of scope — do not attempt, even if asked as part of this skill:**
  - **Wi-Fi.** It works out of the box. Its only limitation (no WPA3/SAE) is fixed router-side.
  - **Hibernation.** Known broken on this model, costs forced power-offs. See the repo README.

---

## Step 0 — Hard gate. Refuse if this is not the right machine.

```sh
cat /sys/class/dmi/id/product_name    # must be exactly: MacBook8,1
cat /sys/class/dmi/id/board_name      # expect: Mac-BE0E8AC46FE800CC
lspci -nn | grep -E '00:15\.(0|4)'    # expect 8086:9ce0 (DMA) and 8086:9ce6 (GSPI)
```

- If `product_name` is **not** `MacBook8,1` → **STOP.** Tell the user this fix is model-specific,
  do not apply anything, and do not improvise a variant for their hardware.
- Also confirm the bootloader is limine, or **STOP** and say so:

```sh
command -v limine-update && ls /etc/limine-entry-tool.d/
```

Do not translate the parameters to GRUB or systemd-boot. That path is untested.

---

## Step 1 — Assess current state (read-only). Report before changing anything.

```sh
grep -o 'initcall_blacklist=dw_pci_driver_init' /proc/cmdline   # already applied?
grep -o 'mem_sleep_default=s2idle'              /proc/cmdline
cat /etc/limine-entry-tool.d/macbook81-spi-fix.conf 2>/dev/null # drop-in exists?
grep -c -i 'apple spi' /proc/bus/input/devices                  # 2 = both alive, 1 = touchpad dead
cat /sys/power/mem_sleep
ls /usr/lib/systemd/system-sleep/                               # must NOT contain macbook81-applespi
cat /etc/mkinitcpio.conf.d/macbook_spi_modules.conf 2>/dev/null
```

- If both parameters are already on `/proc/cmdline` **and** input devices = 2 → **already fixed.**
  Report that and stop. Do not rewrite the file.
- If `/usr/lib/systemd/system-sleep/macbook81-applespi` exists → **flag it for removal** (see
  "Never do these", below). That is a real bug, not a leftover.

---

## Step 2 — Show the exact change and get explicit confirmation

Show the user the literal file you intend to write and where. Then **wait for explicit
confirmation.** Do not proceed on implied consent, silence, or a general "fix my laptop".

File: `/etc/limine-entry-tool.d/macbook81-spi-fix.conf`

```sh
KERNEL_CMDLINE[default]+=" initcall_blacklist=dw_pci_driver_init"
KERNEL_CMDLINE[default]+=" mem_sleep_default=s2idle"
```

State plainly, before they answer:
- This edits the **bootloader configuration** and requires a reboot to take effect.
- If it goes wrong, the built-in keyboard may be unavailable at the LUKS prompt →
  **tell them to plug in an external USB keyboard before rebooting.**
- Rollback is `rm` the file + `limine-update` + reboot.

If a `macbook81-spi-fix.conf` already exists with different content, show a diff and ask before
overwriting. Never silently clobber it.

---

## Step 3 — Apply

Only after explicit confirmation. Write the file with the commented version from
`conf/macbook81-spi-fix.conf` in this repo if available, otherwise the two lines above.

```sh
sudo install -m 0644 /dev/stdin /etc/limine-entry-tool.d/macbook81-spi-fix.conf <<'EOF'
KERNEL_CMDLINE[default]+=" initcall_blacklist=dw_pci_driver_init"
KERNEL_CMDLINE[default]+=" mem_sleep_default=s2idle"
EOF

sudo limine-update
```

Then confirm the parameters actually reached the boot entries **before** telling the user to reboot:

```sh
sudo grep -o 'initcall_blacklist=dw_pci_driver_init' /boot/limine.conf | head -1
sudo grep -o 'mem_sleep_default=s2idle'              /boot/limine.conf | head -1
```

- If `limine-update` fails or the parameters are absent from `/boot/limine.conf`, **stop, do not
  tell the user to reboot,** and report the error.
- No `mkinitcpio` change is needed — Omarchy's installer already ships
  `MODULES=(applespi spi_pxa2xx_platform spi_pxa2xx_pci)`. If that file is missing, say so; that
  is what provides the keyboard at the LUKS prompt.

---

## Step 4 — Hand off the reboot. Do not reboot yourself.

Tell the user:

- Reboot when ready; **keep an external USB keyboard plugged in** through the first reboot.
- After reboot, ask them to run Step 5 or invoke this skill again to verify.

---

## Step 5 — Post-reboot verification

Automated:

```sh
grep -o 'initcall_blacklist=dw_pci_driver_init' /proc/cmdline   # present
sudo dmesg | grep pxa2xx                                        # "no DMA channels available, using PIO"
sudo dmesg | grep applespi                                      # "modeswitch done."
grep -c -i 'apple spi' /proc/bus/input/devices                  # 2
cat /sys/power/mem_sleep                                        # [s2idle] deep
grep SPIT /proc/acpi/wakeup                                     # SPIT S3 *enabled
```

Needs the user's hands — ask, don't assume:
- Built-in keyboard types, touchpad moves.
- Built-in keyboard works at the **LUKS passphrase prompt**.
- `systemctl suspend`, then wake with the **built-in** keyboard.
- Close and open the lid.

**Reading the wake evidence correctly** — this trips people up:
- On a **successful** built-in-keyboard wake, `spi-APP000D:00/power/wakeup_count` stays **`0`**.
  `applespi` never calls `pm_wakeup_event()`. **Do not report `0` as a failure.**
- Check `/sys/power/pm_wakeup_irq` instead — **`9`** (the ACPI SCI) is the correct signature. The
  topcase signals via **ACPI GPE 0x1C**, not a Linux IRQ.
- IRQ 21 (`0000:00:15.4`) in `/proc/interrupts` is an **activity counter, not a health signal** —
  it only moves when the devices are physically touched. A flat count proves nothing.
- One `Received corrupted packet (crc mismatch)` during probe is benign.

---

## Rollback

```sh
sudo rm /etc/limine-entry-tool.d/macbook81-spi-fix.conf
sudo limine-update
# reboot (hand off to the user)
```

---

## Never do these

- ❌ **Never create `/usr/lib/systemd/system-sleep/macbook81-applespi`** or any detach-on-suspend
  hook that does `modprobe -r applespi` / unbinds `0000:00:15.4`. It **breaks
  wake-on-built-in-keyboard** — a device with no driver bound cannot raise a wake event. Tested
  and removed. If you find one on the machine, recommend deleting it.
- ❌ **Never set `mem_sleep_default=deep`.** `s2idle` is correct here.
- ❌ **Never install `macbook12-spi-driver-dkms`.** The in-tree `applespi` is what loads.
- ❌ **Never reboot the machine yourself.** Always hand that to the user.
- ❌ **Never touch Wi-Fi or hibernation** from this skill.
- ❌ **Never apply this to hardware that is not `MacBook8,1`.**
