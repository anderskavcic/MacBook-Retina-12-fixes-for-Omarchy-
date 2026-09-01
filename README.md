# MacBook8,1 on Omarchy 4

Getting the **12-inch Retina MacBook** fully working on **Omarchy 4**: built-in keyboard,
touchpad, and suspend. Two kernel parameters. Nothing else.

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

## Agent skill

An agent skill is included for applying this in a controlled way:
[`skills/macbook81-spi-fix/SKILL.md`](skills/macbook81-spi-fix/SKILL.md).

- **Verify → confirm → apply → hand off the reboot.** It never reboots the machine itself.
- **Hard-gated on `MacBook8,1`** and on limine being present; refuses to run on anything else, and
  will not improvise a GRUB or systemd-boot variant.
- Reads current state first and stops if the fix is already applied.
- Requires **explicit confirmation** before writing, and checks the parameters actually reached
  `/boot/limine.conf` before telling you to reboot.
- **Wi-Fi and hibernation are out of scope** by design.
- `.claude/skills/macbook81-spi-fix` is a symlink to it, so Claude Code picks it up automatically
  when this repo is cloned. Copy either path into `~/.claude/skills/` to make it available
  system-wide.

## Verify

```sh
grep -o 'initcall_blacklist=dw_pci_driver_init' /proc/cmdline    # present
sudo dmesg | grep pxa2xx        # "no DMA channels available, using PIO"
sudo dmesg | grep applespi      # "modeswitch done."
grep -c 'Apple SPI' /proc/bus/input/devices                      # 2 (Keyboard + Touchpad)
cat /sys/power/mem_sleep                                         # [s2idle] deep
```

Plus, by hand: type on the built-in keyboard, move the touchpad, `systemctl suspend` and wake it
with the **built-in** keyboard, and close/open the lid.

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
  does not occur on this model. **No sleep hook is needed at all.**
- ❌ **Do not use `mem_sleep_default=deep`.** The claim that S3 is required here was never
  substantiated; `s2idle` works.
- ❌ **Do not install `macbook12-spi-driver-dkms`.** The in-tree `applespi` is what loads. The
  DKMS package is inert and just fails to build noisily.
- ❌ **Do not install `broadcom-sta` / `broadcom-wl`** for the Wi-Fi. Wrong driver family (see below).

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
- This repo adds the suspend finding (no sleep hook: the upstream hook is harmful), the
  hibernation investigation, and the Wi-Fi/WPA3 limitation and a SKILL for agents.

## Licence

MIT — see [LICENSE](LICENSE).
