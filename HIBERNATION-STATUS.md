# Hibernation on MacBook8,1 — open issue

**Status: does not work. Not fixed. PRs welcome.**

Suspend (`s2idle`) works perfectly — see the [README](README.md). Hibernation does not, and the
investigation cost **two forced power-offs and one black-screen boot**. No filesystem damage
resulted, but **do not retry casually.** Use `/sys/power/pm_test` (below) before ever triggering a
real hibernate.

Tested on Omarchy 4.0.2, kernel 7.1.9-arch1-2, limine + UKI, LUKS root, btrfs, 8 GB RAM.

## What is actually wrong

The hibernate machinery itself works. One attempt got all the way in:

```
PM: hibernation: Allocated 2639208 kbytes in 0.51 seconds (5174.91 MB/s)
PM: hibernation: Normal pages needed: 611097 + 1024, available pages: 1460356
ACPI: PM: Waking up from system sleep state S4      <- it really did enter S4
usb usb1: root hub lost power or was reset         <- hardware really did power down
```

The image is built, ACPI S4 is entered, the USB root hubs genuinely lose power — and then **the
Apple firmware powers the machine back on about one second later.** It will not stay off. The
kernel then resumes cleanly, so nothing is broken; the machine just refuses to remain in S4.

## Attempts

| # | `zram0` | `HibernateMode` | Result |
|---|---|---|---|
| 1 | on, prio 100 | `platform` (default) | **Hang** at `PM: hibernation: hibernation entry`. Forced power-off |
| 2 | **off** | `platform` | Image built, **entered S4**, firmware re-powered ~1s later, kernel resumed cleanly |
| 3 | off | **`shutdown`** | **Hang** at `hibernation entry`. Forced power-off → black-screen boot → second power-off |

## Findings

- **`zram0` at priority 100 causes the hang.** It sits above the swapfile's priority 0. With zram
  on, hibernate hangs outright at `hibernation entry`; with it off, attempt 2 reached S4.
  `swapoff /dev/zram0` helps but **does not stick** — zram's systemd unit re-enables it across a
  sleep cycle. A durable fix needs the priority lowered or the zram swap dropped entirely.
- **`HibernateMode=shutdown` made it worse.** It hung where the default `platform` had at least
  reached S4. The usual "set `HibernateMode=shutdown` on a Mac" advice does not apply here.
- **No driver is at fault.** `/sys/power/pm_test` passed at **every** level — `freezer`,
  `devices`, `platform`, `processors`, `core` — including
  `Disabling non-boot CPUs ... Enabling non-boot CPUs`. `core` is the deepest level; it does
  everything except write the image and power off. So the freeze, device-suspend and CPU paths
  are all healthy. `brcmfmac`, `applespi` and `i915` froze and thawed without complaint.
- **Therefore the SPI-detach designs are solving a non-problem.** Do **not** build an initramfs
  `spi-detach` hook, an `applespi-rebind.service`, or a detach-on-hibernate sleep hook. The
  `pm_test` results rule out the premise those are built on.
- **The hangs revealed nothing** beyond "it stopped here", because journald cannot flush to disk
  once hibernation starts. Add `no_console_suspend` to the kernel command line before
  investigating further.

## Prime suspect

`/proc/acpi/wakeup` on this machine:

```
ARPT      S4    *enabled   pci:0000:01:00.0     <- Broadcom Wi-Fi, armed for S4
XHC1      S3    *enabled   pci:0000:00:14.0     <- USB
LID0      S4    *enabled   platform:PNP0C0D:00  <- lid
```

The Wi-Fi card is armed as an **S4** wake source and Wi-Fi was connected during every attempt.
A wake source that was never disarmed is consistent with the firmware re-powering the machine
about a second after S4 entry.

Also relevant: Apple firmware on this model is documented to auto-power-on shortly after
hibernating when a USB or Thunderbolt device is attached. Unplugging external USB before testing
is worth doing regardless.

## Use `pm_test` before any real attempt

`/sys/power/pm_test` runs the hibernate path up to a chosen phase, waits 5 seconds, then resumes —
with **no image write and no power-off**. It is far cheaper than a forced power-off.

```sh
# levels: freezer -> devices -> platform -> processors -> core (deepest)
echo devices | sudo tee /sys/power/pm_test
sudo systemctl hibernate          # returns on its own after ~5s
journalctl -k -b | grep -iE 'hibernation (entry|exit)|Suspending console|non-boot CPUs'

echo none | sudo tee /sys/power/pm_test   # ALWAYS reset, or real hibernate will not work
```

`pm_test` is runtime-only and resets to `none` on reboot.

## If you want to pick this up

In this order:

1. **Disarm the S4 wake sources.** `echo ARPT | sudo tee /proc/acpi/wakeup` (it toggles), and
   consider `XHC1`. Verify with `grep S4 /proc/acpi/wakeup`. Runtime-only — needs a systemd unit
   to persist across reboots.
2. **Make zram's exclusion durable** — lower its priority below the swapfile, or drop the zram
   swap device entirely. A manual `swapoff` will not survive a sleep cycle.
3. **Re-test with `pm_test` first**, then a real attempt with the **default `platform`** mode
   (attempt 2's configuration, which got furthest) — not `shutdown`.
4. **Add `no_console_suspend`** so a hang leaves usable console output.
5. Unplug external USB/Thunderbolt devices before testing.

## Prerequisites (already satisfied — not the problem)

- Swapfile `/swap/swapfile`, 7.7 GB, **priority 0** (non-negative, so systemd accepts it as a
  hibernation target).
- `resume=/dev/mapper/root` and `resume_offset=1945620` on the kernel command line.
- `/sys/power/resume` = `253:0`, `/sys/power/state` includes `disk`.
- `resume` hook present in `HOOKS`, **after** `encrypt` (the LUKS device must exist before the
  resume hook can read the image). Omarchy ships this via
  `/etc/mkinitcpio.conf.d/omarchy_resume.conf`.

## Safety notes

- Take a snapshot before experimenting (`snapper -c root create -d "pre-hibernate"`).
- After any forced power-off, check for damage: `sudo btrfs device stats /`,
  `journalctl -k -b | grep -i 'btrfs.*error'`, `systemctl --failed`, and confirm no stale
  `/sys/firmware/efi/efivars/HibernateLocation-*` variable remains.
- A partially written image plus a resume attempt is the scenario that can corrupt a filesystem.
  None occurred here, but it is the real risk of repeated attempts.
