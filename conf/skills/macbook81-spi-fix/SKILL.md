---
name: macbook81-spi-fix
description: Apply the MacBook8,1 (MacBook Retina 12-inch, Early 2015) keyboard, touchpad and suspend fixes on Omarchy 4 — two kernel parameters via a limine drop-in, plus a sleep hook that turns the keyboard backlight off during suspend. Use when the built-in keyboard and/or touchpad are dead on a 12-inch MacBook, when applespi times out with -110, when Apple SPI Touchpad is missing from /proc/bus/input/devices, when the keyboard backlight stays lit through suspend, or when asked to apply the MacBook8,1 SPI fix. Verifies hardware first, requires explicit confirmation before writing, and never reboots on its own. Does NOT cover Wi-Fi, audio or hibernation.
---

# MacBook8,1 SPI fix (Omarchy 4)

Applies two kernel parameters that make the built-in keyboard, touchpad and suspend work, plus a
`system-sleep` hook that turns the keyboard backlight off while suspended.
**Verify → confirm → apply → hand off the reboot.** Never reboot the machine yourself.

## Scope

- **In scope:** built-in keyboard, touchpad, suspend (s2idle), and the keyboard backlight during
  suspend, on **Omarchy 4** with **limine + UKI**.
- **Out of scope — do not attempt, even if asked as part of this skill:**
  - **Wi-Fi.** It works out of the box. Its only limitation (no WPA3/SAE) is fixed router-side.
  - **Hibernation.** Known broken on this model, costs forced power-offs. See the repo README.
  - **Audio.** Internal speakers need an out-of-tree codec driver. Separate problem, separate fix.

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
cat /etc/mkinitcpio.conf.d/macbook_spi_modules.conf 2>/dev/null
ls -l /usr/lib/systemd/system-sleep/   # macbook81-kbd-backlight present and 0755?
                                       # must NOT contain macbook81-applespi
```

- If both parameters are already on `/proc/cmdline` **and** input devices = 2 → the
  kernel-parameter half is **already applied**. Do not rewrite the drop-in, and skip Steps 2, 3a
  and 4. **Do not stop here** — go to Step 3b. The backlight hook is a separate change and is
  usually missing on a machine that already has the parameters.
- Only when the parameters **and** a `0755` `macbook81-kbd-backlight` are both in place is there
  nothing to do. Report that and stop.
- If `/usr/lib/systemd/system-sleep/macbook81-applespi` exists → **flag it for removal** (see
  "Never do these", below). That is a real bug, not a leftover.
- `macbook81-kbd-backlight` present and mode `0755` → the backlight fix is applied. Present but
  mode `0644` → it is **inert**; `systemd-sleep` runs only executables and skips the rest silently.
  Fix the mode, do not rewrite the file.
- Omarchy's own `keyboard-backlight` hook is normally mode `0644` and hibernate-only. Leave it
  alone. It is inert and harmless, and ours supersedes it.

---

## Step 2 — Show the exact change and get explicit confirmation

Show the user the literal file you intend to write and where. Then **wait for explicit
confirmation.** Do not proceed on implied consent, silence, or a general "fix my laptop".

There are two files. Show both, and say which needs a reboot.

File 1 — `/etc/limine-entry-tool.d/macbook81-spi-fix.conf` (Step 3a, **needs a reboot**):

```sh
KERNEL_CMDLINE[default]+=" initcall_blacklist=dw_pci_driver_init"
KERNEL_CMDLINE[default]+=" mem_sleep_default=s2idle"
```

File 2 — `/usr/lib/systemd/system-sleep/macbook81-kbd-backlight` (Step 3b, **no reboot**):

```sh
#!/bin/bash
case "$1" in
  pre)  /usr/bin/omarchy-brightness-keyboard --no-osd off ;;
  post) /usr/bin/omarchy-brightness-keyboard --no-osd restore ;;
esac
```

State plainly, before they answer:
- File 1 edits the **bootloader configuration** and requires a reboot to take effect.
- If it goes wrong, the built-in keyboard may be unavailable at the LUKS prompt →
  **tell them to plug in an external USB keyboard before rebooting.**
- File 2 changes nothing about boot and cannot lock them out; worst case the backlight stays off
  after a wake, and a brightness-up key press fixes it.
- Rollback is `rm` each file (plus `limine-update` + reboot for file 1).

If a `macbook81-spi-fix.conf` already exists with different content, show a diff and ask before
overwriting. Never silently clobber it.

---

## Step 3a — Apply the kernel parameters

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

## Step 3b — Apply the keyboard-backlight sleep hook

Independent of Step 3a: **no reboot needed**, it takes effect on the next suspend. Apply it even
if the kernel parameters were already in place.

Why it is needed: `applespi` registers the backlight LED without `LED_CORE_SUSPENDRESUME`, and
`applespi_suspend()` turns off only the caps-lock LED — so nothing zeroes the backlight. Under
`s2idle` the topcase stays powered, so it stays lit for the whole suspend.

From a clone of this repo (path is relative to the repo root):

```sh
sudo install -m 0755 conf/macbook81-kbd-backlight /usr/lib/systemd/system-sleep/
```

If the repo file is unavailable, write the hook inline — same mode, same name:

```sh
#!/bin/bash
case "$1" in
  pre)  /usr/bin/omarchy-brightness-keyboard --no-osd off ;;
  post) /usr/bin/omarchy-brightness-keyboard --no-osd restore ;;
esac
```

- **`install -m 0755`, never `cp`.** `cp -p` from a mode-`0644` source is exactly how Omarchy's own
  hook ends up inert. Verify the mode after writing.
- Requires `brightnessctl` and `omarchy-brightness-keyboard`. If `omarchy-brightness-keyboard` is
  missing, **stop and say so** rather than open-coding a `brightnessctl` call — device discovery
  belongs in Omarchy's script.
- Rollback: `sudo rm /usr/lib/systemd/system-sleep/macbook81-kbd-backlight`.
- Verify the mode landed before reporting success:
  `ls -l /usr/lib/systemd/system-sleep/macbook81-kbd-backlight`.

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
ls -l /usr/lib/systemd/system-sleep/macbook81-kbd-backlight     # -rwxr-xr-x, root:root
```

After a suspend/resume cycle, the hook leaves proof that it ran — a root-owned save file holding
the pre-suspend level:

```sh
cat /tmp/brightnessctl/leds/spi::kbd_backlight   # the level in effect before suspend
```

Needs the user's hands — ask, don't assume:
- Built-in keyboard types, touchpad moves.
- Built-in keyboard works at the **LUKS passphrase prompt**.
- `systemctl suspend`, then wake with the **built-in** keyboard.
- Close and open the lid.
- Keyboard backlight goes dark as the machine suspends, and returns at the level it was left at.

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

Kernel parameters (needs a reboot):

```sh
sudo rm /etc/limine-entry-tool.d/macbook81-spi-fix.conf
sudo limine-update
# reboot (hand off to the user)
```

Backlight hook (takes effect immediately, no reboot):

```sh
sudo rm /usr/lib/systemd/system-sleep/macbook81-kbd-backlight
```

---

## Never do these

- ❌ **Never create `/usr/lib/systemd/system-sleep/macbook81-applespi`** or any detach-on-suspend
  hook that does `modprobe -r applespi` / unbinds `0000:00:15.4`. It **breaks
  wake-on-built-in-keyboard** — a device with no driver bound cannot raise a wake event. Tested
  and removed. If you find one on the machine, recommend deleting it. This bans *unbinding*, not
  sleep hooks as a category: the Step 3b hook writes a brightness value and touches no binding.
- ❌ **Never set `mem_sleep_default=deep`.** `s2idle` is correct here.
- ❌ **Never install `macbook12-spi-driver-dkms`.** The in-tree `applespi` is what loads.
- ❌ **Never reboot the machine yourself.** Always hand that to the user.
- ❌ **Never touch Wi-Fi, audio or hibernation** from this skill.
- ❌ **Never apply this to hardware that is not `MacBook8,1`.**
