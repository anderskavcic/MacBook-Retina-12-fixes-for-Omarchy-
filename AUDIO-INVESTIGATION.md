# Audio on MacBook8,1 — diagnosis and plan

**Status: diagnosed, not fixed. No changes made to the machine.**

Investigated 2026-09-01 on Omarchy 4.0.2, kernel 7.1.9-arch1-2. Symptom: no sound from the
built-in speakers, e.g. playing a YouTube video in Chrome.

## Diagnosis: it's not Chrome, and it's not PipeWire

The audio stack is healthy end-to-end. During playback the PCM reaches `state: RUNNING`, the
default sink is the CS4208 card, volume is 80% and unmuted. The problem is one layer below all
of that.

**This machine has no code path to the speakers at all.** Here's the chain:

`journalctl -k`, at codec probe:

```
snd_hda_codec_cs420x hdaudioC1D0: CS4208: picked fixup  for codec SSID 106b:0000
snd_hda_codec_cs420x hdaudioC1D0: autoconfig for CS4208: line_outs=1 (0x1d) type:speaker
snd_hda_codec_cs420x hdaudioC1D0:    speaker_outs=0
snd_hda_codec_cs420x hdaudioC1D0:    hp_outs=1 (0x10)
```

The BIOS pin table (`/sys/class/sound/hwC1D0/init_pin_configs`) says the only internal speaker
is **pin `0x1d`** (`0x90100110`); the analog pins `0x11`–`0x14` are `0x400000f0` — **not
connected**. And pin `0x1d` is an **8-channel digital** pin whose only source is digital
converter `0x0a`. The HDA generic parser can't assign an analog DAC to a digital pin, so it
silently drops the speaker path and builds exactly one PCM: `CS4208 Analog` on DAC `0x02` →
**headphone jack pin `0x10`**.

Confirmed live, while a tone was actually playing:

| pin | role | pin-ctl during playback |
|---|---|---|
| `0x10` | headphone jack | `0xc0` — **OUT HP (enabled)** |
| `0x11`–`0x14` | (unconnected) | `0x00` |
| `0x1d` | internal speaker | `0x00` |

Every sound this machine makes goes out the headphone jack. YouTube in Chrome is doing
everything right.

**Root cause:** upstream `patch_cirrus`'s CS4208 quirk table has entries for
`106b:5e00/6c00/7100/7200/7b00` — MacBookPro11,2, MacMini7,1, MacBookAir6,1/6,2,
MacBookPro12,1. This codec's SSID is **`106b:6400`**, which matches nothing, so it falls
through to the generic `CS4208_GPIO0` default, which does not build the speaker path. Same
shape as the SPI bug: correct driver, no entry for this machine.

### Reference values from this machine

```
Codec           Cirrus Logic CS4208, vendor 0x10134208, revision 0x100401
Subsystem       0x106b6400
Card            card 1 [PCH], HDA Intel PCH at 00:1b.0 (Wildcat Point-LP)
power_save      10   (must be 0 — see gotchas)
init_pin_configs
  0x10 0x002b4020   HP jack
  0x11 0x400000f0   not connected
  0x12 0x400000f0   not connected
  0x13 0x400000f0   not connected
  0x14 0x400000f0   not connected
  0x18 0x00ab9030   mic jack
  0x19 0x90a60100   internal mic
  0x1d 0x90100110   internal speaker (digital, source = converter 0x0a)
```

Note: `/proc/asound/card1/codec#0` shows *different*, misleading pin defaults (four analog
"Speaker" pins at `0x9017001x`). Those are the CS4208's silicon power-on defaults, which
reappear after the codec's first D3cold transition — Apple's firmware pin table is lost and
never rewritten. `init_pin_configs` is the authoritative view, and it is what the parser used.

## What the research turned up

The speakers are **four MAX98357 class-D amps on a 4-channel TDM stream** —
`converter 0x0a → mixer 0x25 → pin 0x1d`, format `0x4013` (4ch / 44.1 kHz / S16). That's not
inference: [Shawn3DB](https://github.com/leifliddy/macbook8-1-audio-driver-test/issues/2)
dual-booted macOS, pulled `AppleHDA.kext`'s `layout100` / PathMapID 31, and published it — for
codec `0x10134208`, **SSID `106b:6400`**. Literally this machine. Codec revision matches this
one exactly (`0x100401`).

As of 2026 there are **three independently confirmed working MacBook8,1 setups**. The two
worth the time:

| | Shawn3DB | thomas-shirley |
|---|---|---|
| Source | [test-repo issue #2](https://github.com/leifliddy/macbook8-1-audio-driver-test/issues/2) (zip attached) | [macbook8.1-speaker-driver](https://github.com/thomas-shirley/macbook8.1-speaker-driver) |
| Scope | 1 patched module (`snd-hda-codec-cs420x`) + `power_save=0` | 3 patched modules (adds `snd-hda-intel`, `snd-hda-codec-generic`) + `single_cmd=1` + PipeWire EQ chain + resume-recovery service |
| Install | manual `.ko` swap | DKMS, one `install.sh` |
| Jack switching | none — pick output in Sound settings | systemd unit handles it |
| Validated on | Mint, kernel 6.17 | Ubuntu 24.04/26.04, kernel 6.17, incl. microphone |

Three gotchas that will otherwise cost an evening:

- **`power_save` must be 0.** It's `10` here (Arch's default). At non-zero, runtime PM resets
  converter `0x0a`'s stream-id mid-playback and audio dies after ~2s. Good news: no TLP is
  installed on this machine, which is what silently re-enables it for other people.
- **Never test with `speaker-test` sine.** The digital path soft-mutes a perfectly static tone
  after ~3 seconds. Test with music or speech.
- **Codec GPIO 4/5 are a red herring** — documented dead end, speakers play with them low.

The Omarchy project has [PR #6921](https://github.com/omacom/omarchy/pull/6921) adding this
driver as an install leaf — and it **explicitly skips MacBook8,1** as unsupported. That gap is
now closed by the work above, and nobody has told them.

## The plan

**Step 0 — one thing needed from the user.** Plug in headphones and play the YouTube video.
Sound is expected. If headphones are also silent, the whole diagnosis above changes and it
starts over.

**Step 1 — prerequisites** (not installed): `dkms`, `linux-headers`. Currently absent, so
nothing can build yet.

**Step 2 — prove the path, cheaply.** Take Shawn3DB's single module, build it against 7.1.9,
install it alongside a backup of the stock `.ko.zst`, add `options snd_hda_intel power_save=0`,
reboot, play music. One module, one modprobe line, rollback is restoring the `.bak` and
rebooting. It was written for kernel 6.17; 7.1.9 uses the same `sound/hda/codecs/cirrus`
layout, so small or no build fixes are expected.

**Step 3 — decide on the full stack.** If step 2 gives sound, that's working speakers but
manual output switching. thomas-shirley's DKMS stack adds jack switching, EQ tuning and resume
recovery — but it's Ubuntu-shaped (`apt`, `update-initramfs`) and needs porting to
pacman/mkinitcpio, and it patches `snd-hda-intel` itself. Worth it only once step 2 has proven
the hardware path on this kernel.

**Step 4 — the thing to flag hardest: this collides with the suspend fix.** Both reporters hit
it. On the old-tree driver, *any* s2idle suspend kills the speakers until a full reboot — ACPI
`_PS3` drives the amp-enable GPIO low and nothing on the OS side brings it back; only the EFI
boot path revives the amps. thomas-shirley's stack works around a related S3 clock-fault latch
with a resume service. This repo's headline claim is *"suspend is solid, s2idle works."*
Adding speakers may cost that, so suspend/resume has to be part of the acceptance test, not an
afterthought.

**Step 5 — write it up.** Unlike the SPI fix, there is no kernel-parameter version of this; it
is unavoidably an out-of-tree module, which cuts against this repo's minimal-change framing and
its explicit "do NOT install DKMS drivers" rule. That distinction deserves to be stated plainly
in the README rather than glossed. Then a hard-gated
`macbook81-cs4208-audio` skill in the same shape as the SPI one, and — if it works — a comment
on Omarchy PR #6921 telling them 8,1 is solvable.

Risk note: unlike the keyboard fix, nothing here can lock the machine. Worst case is no audio
at all until the stock module is restored and the machine rebooted.

## Sources

- [Shawn3DB's MacBook8,1 writeup](https://github.com/leifliddy/macbook8-1-audio-driver-test/issues/2)
- [thomas-shirley/macbook8.1-speaker-driver](https://github.com/thomas-shirley/macbook8.1-speaker-driver)
- [leifliddy/macbook12-audio-driver #57](https://github.com/leifliddy/macbook12-audio-driver/issues/57)
- [Omarchy PR #6921 — Install the CS4208 speaker driver on 12-inch MacBooks](https://github.com/omacom/omarchy/pull/6921)
- [Running Omarchy on the MacBook 12"](https://kuroshi.net/blog/2026/04/17/omarchy-macbook-12.html)
