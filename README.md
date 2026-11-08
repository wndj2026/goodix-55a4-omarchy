# Goodix 27c6:55a4 fingerprint reader on Omarchy / Arch

Makes the `Shenzhen Goodix Technology Co.,Ltd. Goodix FingerPrint Device`
(`USB 27c6:55a4`, found in many Lenovo ThinkBook / ThinkPad / IdeaPad laptops)
work on [Omarchy](https://omarchy.org/) (Arch + Hyprland), including `sudo`,
polkit, and lock-screen auth.

## Status

Working. This exact setup drives a `GF3208_RTSEC_APP_10062` unit that no
upstream `libfprint` release supports as of 2026.

## Why this exists

- Stock `libfprint` / `libfprint-git` has no driver for `27c6:55a4`:
  `fprintd-list` says `No devices available`, so
  `omarchy setup security fingerprint` can never enroll.
- Goodix ships no Linux driver; the sensor is Windows-provisioned
  (PSK + `GF32xx_RTSEC_APP_*` firmware).
- Community drivers exist but need help on Arch, and this sensor revision
  (`GF3208/10062`) trips up forks aimed at `55b4`/`GF3268` units.

## What worked

1. **[Hydrogell/goodix-27c6-55a4-fingerprint-linux](https://github.com/Hydrogell/goodix-27c6-55a4-fingerprint-linux)** —
   a patch on `TheWeirdDev/libfprint#55b4-experimental` built specifically for
   the `55a4`, with per-sensor finger-detect thresholds and correct `0xd0`
   image-ready handling. Installed to `/opt/libfprint-goodix` with a systemd
   drop-in so **only** `fprintd` uses it; the system `libfprint` stays stock
   (clean across `omarchy update`).
2. A udev rule keeping USB autosuspend **off** for the sensor. This unit resets
   its USB connection on sleep; with `power/control=auto` enroll/verify kept
   dying with `device was disconnected`. Note: Arch has **no `plugdev` group**,
   so Debian-style rules referencing it fail open — use `TAG+="uaccess"`.
3. Arch build deps the READMEs don't all mention: `glib2-devel` (provides
   `glib-mkenums`), `cmake`, `doctest`, plus an `opencv4 → opencv5` pkg-config
   shim (Arch ships OpenCV 5; the driver asks for `opencv4`).
4. **`min_match` 5 → 4** in the sigfm matcher (the rebuild script does this
   by default; `GOODIX_MIN_MATCH=5` restores upstream). With 5, `fprintd-verify`
   on this unit captured image after image for ~10 s and never returned a
   result; with 4 (plus a firm, steady press) it returns `verify-match`. This is what fixed the
   "flaky verify".
5. PAM wiring identical to `omarchy setup security fingerprint`
   (`sudo`, `polkit-1`, `omarchy-lock-fingerprint`) — see below.

Also tried and abandoned: `GuNanOvO/goodix-55x4-linux` (targets `55b4`/
`GF3268`; patched locally for the `GF3208` firmware prefix and the missing
`0xd0` routing, but enroll hung at image capture on this unit).

## Install

```bash
git clone <this-repo>
cd goodix-55a4-omarchy
./goodix-55a4-rebuild
```

Then enroll (16 touches, firm presses, vary position, full lift between touches;
re-run if a pass dies with `device removed` — the sensor resets USB mid-flow):

```bash
fprintd-enroll -f right-index-finger
fprintd-verify                      # confirm it matches
omarchy setup security fingerprint  # wires sudo/polkit/lock PAM
```

`goodix-55a4-rebuild` is also installed to `~/.local/bin` with a desktop entry
("Rebuild Fingerprint Driver") so it can be re-run from the launcher after a
system update breaks the sensor. Enrolled prints survive rebuilds.

### Wire PAM (sudo / polkit / lock screen)

`omarchy setup security fingerprint` re-enrolls, then runs its own verify, and
**aborts before wiring PAM if that verify fails** — its wrapper can report
`verify-unknown-error` even when plain `fprintd-verify` matches. So check
`fprintd-verify` yourself first. If it prints `verify-match` but the omarchy
script keeps failing, wire PAM by hand (keep a root shell open while testing;
the line is `sufficient`, so a failed scan falls back to the password):

```bash
sudo sed -i '1i auth      sufficient pam_fprintd.so' /etc/pam.d/sudo
sudo sed -i '/pam_fprintd\.so/i auth      [success=1 default=ignore] pam_exec.so quiet /usr/bin/omarchy-hw-laptop-closed' /etc/pam.d/sudo
sudo -k && sudo true   # touch the sensor
```

When verifying, press firmly and hold still for 2–3 s; don't tap.

## Troubleshooting

| Symptom | Likely cause / fix |
|---|---|
| `No devices available` | Stock libfprint; (re)run the rebuild script |
| `Invalid device firmware: GF3208…` | Old `55b4`-only driver; use the Hydrogell-based build here |
| `enroll-unknown-error` + `device removed` | USB reset race; wait ~10 s, retry. Check `power/control` is `on`: `cat /sys/bus/usb/devices/*/power/control` for the `27c6` device |
| `fprintd-verify` captures for ~10 s, then `verify-unknown-error` + `device removed` | Matcher too strict for this unit. Rebuild (default now `min_match=4`). The USB disconnect is a side effect of the verify being killed mid-capture, not the cause |
| Rebuild fails at `libfprint and libfprint-git are in conflict` | You have `libfprint-git`; the script no longer asks pacman for plain `libfprint` |
| `verify-no-match` on the right finger | Weak template or strict matcher; delete (`fprintd-delete`) and re-enroll with better coverage |
| `glib-mkenums` missing at build | `sudo pacman -S glib2-devel` (handled by the script) |
| `opencv4 not found` at build | Arch OpenCV 5 shim (handled by the script) |

## Caveats

- Reverse-engineered community driver with a public, hardcoded PSK: transport
  encryption only, no host authentication. Fine for laptop login; don't rely on
  it alone for high-security use.
- Field-calibrated on very few units. If capture is unreliable, the upstream
  knob is `GOODIX55X4_TOUCH_DIFF=200`; if matching is too strict, lower `min_match`
  further (default here is already 4; set `GOODIX_MIN_MATCH=3` and rebuild —
  lower values raise the false-accept rate).
- Dual-boot Windows rewrites the sensor PSK on its boots; the provision step in
  `install.sh` plus a PSK auto-fix service handles the round-trip.

## Credits

- [Hydrogell/goodix-27c6-55a4-fingerprint-linux](https://github.com/Hydrogell/goodix-27c6-55a4-fingerprint-linux) — the driver
- [TheWeirdDev/libfprint](https://github.com/TheWeirdDev/libfprint) (`55b4-experimental` base)
- [goodix-fp-linux-dev/goodix-fp-dump](https://github.com/goodix-fp-linux-dev/goodix-fp-dump) — PSK provisioning research
- [GuNanOvO/goodix-55x4-linux](https://github.com/GuNanOvO/goodix-55x4-linux) — tried along the way
