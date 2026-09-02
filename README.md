# Fixes for MacBook 12" on Omarchy 4

Getting the **12-inch Retina MacBook** (MacBook8,1) fully working on **Omarchy 4**: built-in keyboard,
touchpad, and suspend. Two kernel parameters and one small sleep hook.

## The machine

| | |
|---|---|
| Model | **MacBook8,1** — MacBook (Retina, 12-inch, Early 2015) |
| Board ID | `Mac-BE0E8AC46FE800CC` |
| CPU | Intel Core M-5Y31 (Broadwell-Y), 2C/4T |
| RAM / SSD | 8 GB · Apple SSD AP0256H 256 GB (NVMe) |
| GPU | Intel HD Graphics 5300 |
| Topcase | Apple SPI `APP000D` via Intel Wildcat Point-LP GSPI `00:15.4` |
| Wi-Fi | Broadcom BCM4350 `14e4:43a3` |
| Boot ROM | `481.0.0.0.0` |

Verified on **Omarchy 4.0.2**, kernel **7.1.9-arch1-2**, limine + UKI, LUKS root.

## Symptoms this fixes

- Built-in **keyboard and touchpad completely dead** — including at the LUKS passphrase prompt.
- `dmesg` shows repeated `applespi ... spi_sync failed` / `-110` timeouts.
- `/proc/bus/input/devices` has no `Apple SPI Touchpad`.
- IRQ 21 (`0000:00:15.4`) stuck at 0 in `/proc/interrupts`.
- Suspend wedges or fails to resume properly.
- Keyboard backlight **stays lit for the whole suspend**.

## Root cause

- `applespi` binds fine to `spi-APP000D:00`, but the GSPI controller at `00:15.4` routes
  transfers through the DesignWare DMA engine at `00:15.0` (`dw_dmac_pci`, **built into the
  kernel**, not a module).
- Those DMA transfers **never complete** → every SPI message times out → topcase is dead.
- Blacklisting the DMA driver's initcall leaves **no DMA channel**, so `pxa2xx_spi` falls back
  to **PIO**, which works.
- Not an IRQ-routing problem, not the 6.12 VT-d regression — which is why `irqpoll`,
  `irqfixup`, `pci=nomsi`, `acpi_osi=!Darwin` and `pcie_aspm=off` all failed for everyone.

## The fix

Create `/etc/limine-entry-tool.d/macbook81-spi-fix.conf`:

```sh
KERNEL_CMDLINE[default]+=" initcall_blacklist=dw_pci_driver_init"   # keyboard + touchpad
KERNEL_CMDLINE[default]+=" mem_sleep_default=s2idle"                # suspend
```

Then:

```sh
sudo limine-update   # rebuilds the UKI and /boot/limine.conf
sudo reboot
```

- A ready-to-copy file with comments is in [`conf/macbook81-spi-fix.conf`](conf/macbook81-spi-fix.conf).
- Keep an external USB keyboard plugged in through the first reboot, in case the fix doesn't take.
- Omarchy's installer already writes `MODULES=(applespi spi_pxa2xx_platform spi_pxa2xx_pci)` to
  `/etc/mkinitcpio.conf.d/macbook_spi_modules.conf`, so **no mkinitcpio change is needed** — this
  is what gives you the keyboard at the LUKS prompt.
- The drop-in is kernel-version-independent; the UKI is rebuilt automatically on kernel updates.

## Keyboard backlight on suspend

The machine suspends correctly with the fix above, but the **keyboard backlight stays lit** the
whole time. Two things have to both be missing for that, and here they are:

- `applespi` registers the backlight as an LED class device but never sets
  **`LED_CORE_SUSPENDRESUME`**, and `applespi_suspend()` turns off only the caps-lock LED. Nothing
  in the kernel zeroes the backlight.
- Under `deep`/S3 the topcase would lose power and go dark as a side effect. Under **`s2idle`** —
  which the fix above mandates — it stays powered. **This repo's own suspend fix is what makes the
  problem visible**, which is why nobody upstream has hit it on this model.

Omarchy ships `/usr/lib/systemd/system-sleep/keyboard-backlight`, which looks like it would help
and cannot: it is installed mode **`644`**, and `systemd-sleep` runs only *executables* — it skips
non-executable files silently, with no warning. Its one branch matches `hibernate`, never
`suspend`, so it would be the wrong hook even with the bit set.

The fix saves and zeroes the level on the way down and restores it on the way up, reusing
Omarchy's own `omarchy-brightness-keyboard`:

```sh
sudo install -m 0755 conf/macbook81-kbd-backlight /usr/lib/systemd/system-sleep/
```

- Ready-to-copy file: [`conf/macbook81-kbd-backlight`](conf/macbook81-kbd-backlight).
- **`install -m 0755`, never `cp`.** That mode bit is the entire reason Omarchy's own hook is inert.
- No reboot needed. It takes effect on the next suspend.
- It restores the level you had, not a fixed one. Suspend at 0 and it stays 0 on wake.
- Rollback: `sudo rm /usr/lib/systemd/system-sleep/macbook81-kbd-backlight`.
- It writes a brightness value and nothing else. It does **not** unbind `applespi` — see
  [Do NOT](#do-not), which forbids a different hook entirely.

Two implementation details make this reliable on this driver rather than merely likely:
`applespi_suspend()` calls `applespi_drain_writes()`, so the zero queued by `pre` is flushed to the
hardware before the freeze; and `applespi_resume()` resets `have_bl_level = 0`, clearing the
redundant-write cache so the `post` restore actually reaches the hardware instead of being dropped
as a no-op.

## Agent skill

An agent skill is included for applying this in a controlled way:
[`conf/skills/macbook81-spi-fix/SKILL.md`](conf/skills/macbook81-spi-fix/SKILL.md).

- **Verify → confirm → apply → hand off the reboot.** It never reboots the machine itself.
- **Hard-gated on `MacBook8,1`** and on limine being present; refuses to run on anything else, and
  will not improvise a GRUB or systemd-boot variant.
- Reads current state first and stops if the fix is already applied.
- Requires **explicit confirmation** before writing, and checks the parameters actually reached
  `/boot/limine.conf` before telling you to reboot.
- **Wi-Fi, audio and hibernation are out of scope** by design.
- `.claude/skills/macbook81-spi-fix` is a symlink to it, so Claude Code picks it up automatically
  when this repo is cloned. To make it available system-wide, copy the real directory —
  `cp -r conf/skills/macbook81-spi-fix ~/.claude/skills/` — not the symlink.

## Verify

```sh
grep -o 'initcall_blacklist=dw_pci_driver_init' /proc/cmdline    # present
sudo dmesg | grep pxa2xx        # "no DMA channels available, using PIO"
sudo dmesg | grep applespi      # "modeswitch done."
grep -c 'Apple SPI' /proc/bus/input/devices                      # 2 (Keyboard + Touchpad)
cat /sys/power/mem_sleep                                         # [s2idle] deep
ls -l /usr/lib/systemd/system-sleep/macbook81-kbd-backlight      # mode must be 0755
```

Plus, by hand: type on the built-in keyboard, move the touchpad, `systemctl suspend` and wake it
with the **built-in** keyboard, and close/open the lid. The keyboard backlight should go dark
as the machine suspends and come back at the level you left it.

> One gotcha reading the wake evidence: on a **successful** wake from the built-in keyboard,
> `/sys/bus/spi/devices/spi-APP000D:00/power/wakeup_count` stays **`0`**. `applespi` never calls
> `pm_wakeup_event()`, so the wake isn't attributed to the SPI device. Check
> `/sys/power/pm_wakeup_irq` instead — **`9`** (the ACPI SCI) is the correct signature, because
> the topcase signals via **ACPI GPE 0x1C**, not a Linux IRQ. Don't read `0` as failure.

## Do NOT

- ❌ **Do not install the detach-on-suspend `system-sleep` hook** that the upstream gist
  recommends (`modprobe -r applespi` + unbind `00:15.4` before suspend). Tested here: it
  **breaks wake-on-built-in-keyboard**, because a device with no driver bound can't raise a wake
  event — only an external USB keyboard could wake the machine. The wedge it exists to prevent
  does not occur on this model. This bans *unbinding*, not sleep hooks as a category — the
  backlight hook above writes a brightness value and touches no driver binding, so it is safe.
- ❌ **Do not use `mem_sleep_default=deep`.** The claim that S3 is required here was never
  substantiated; `s2idle` works.
- ❌ **Do not install `macbook12-spi-driver-dkms`.** The in-tree `applespi` is what loads. The
  DKMS package is inert and just fails to build noisily.
- ❌ **Do not install `broadcom-sta` / `broadcom-wl`** for the Wi-Fi. Wrong driver family (see below).

## Open issue: audio

**No sound from the internal speakers.** Not fixed here. Headphones work.

The speakers are four MAX98357 class-D amps on a 4-channel TDM stream — digital converter `0x0a`
→ pin `0x1d`, **not** the analog DACs. Mainline `patch_cirrus` has no quirk for codec SSID
`106b:6400`, so the HDA parser finds no usable speaker path and routes everything to the
headphone jack. There is no kernel-parameter fix; it needs an out-of-tree codec module.

Full diagnosis, the three known-working community drivers, and the plan:
**[AUDIO-INVESTIGATION.md](AUDIO-INVESTIGATION.md)**.

> Note for anyone applying one of those drivers: reports say suspend kills the speakers until
> reboot. That collides with the `s2idle` fix above, so test suspend as part of the acceptance
> criteria, not afterwards.

## Open issue: hibernation

Suspend (s2idle) is solid. **Hibernate does not work on this model** and is not fixed here.

- The machinery works — the image builds and the machine genuinely enters ACPI S4 — but the
  **Apple firmware powers it back on ~1 second later.** It will not stay off.
- Cost of investigating: two forced power-offs and one black-screen boot. No filesystem damage,
  but **do not retry casually** — use `/sys/power/pm_test` first.
- Full write-up, everything tried, and how to pick it up safely:
  **[HIBERNATION-STATUS.md](HIBERNATION-STATUS.md)**. PRs welcome.

## Wi-Fi

Wi-Fi works out of the box on Omarchy — no driver work needed. One limitation:

- The BCM4350's firmware (`7.35.180.133`, dated **Nov 2015**) **cannot associate with WPA3/SAE**,
  including **WPA2/WPA3 transition mode**. There is no newer firmware.
- It presents as *"the password is rejected"* — but the password is never tested. Association
  fails before the 4-way handshake and NetworkManager just re-asks for secrets.
- Tell them apart in the log: a real bad password reaches `Associated with <bssid>` and then
  `4-Way Handshake failed`. WPA3 never reaches `Associated with` at all — you get
  `CTRL-EVENT-ASSOC-REJECT bssid=00:00:00:00:00:00 status_code=16`.
- **Fix is router-side only:** set the SSID to **WPA2-Personal** (PMF `Disable`). No driver
  option works — verified including `feature_disable=0x7FFFFFFF`, i.e. every feature bit off.

## Credits

- Root cause diagnosed by **matthiasjg**, independently confirmed by **jpumfrey**, in
  [basecamp/omarchy#1954](https://github.com/basecamp/omarchy/issues/1954).
- This repo adds the suspend finding (the upstream detach hook is harmful), the
  keyboard-backlight-on-suspend fix and its root cause, the hibernation investigation, the
  Wi-Fi/WPA3 limitation, and a SKILL for agents.

## Licence

MIT — see [LICENSE](LICENSE).
