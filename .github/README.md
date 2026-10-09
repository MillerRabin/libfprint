# libfprint + FocalTech FT9366 driver

An open-source [libfprint](https://gitlab.freedesktop.org/libfprint/libfprint) driver for the
**FocalTech FT9366 fingerprint sensor behind a Realtek RTS5811 USB bridge (`2808:a658`)**, found
in ASUS VivoBook laptops and others:

```
ID 2808:a658 Realtek USB2.0 Finger Print Bridge FocalTech Fingerprint Device
```

Until now the only way to use this sensor on Linux was FocalTech's closed-source libfprint
build, which is stuck at libfprint 1.94.4 and needs binary patching on current distributions.
This driver is written from scratch, contains no vendor code, and works with current libfprint
and fprintd.

The driver is in `libfprint/drivers/ft9366.c` (USB protocol, finger detection, capture) and
`libfprint/drivers/ft9366-match.{c,h}` (image processing and matching). Everything else is
upstream libfprint. Branch `ft9366` is upstream `master` plus the driver commits.

## Status

Enroll, verify and identify work through fprintd, including `sudo` via `pam_fprintd`.

| | Result |
|---|---|
| Enrolled finger, live | 20/20 and 18/20 accepted in two test sessions |
| Other fingers, live | 0/45 accepted |
| Recorded touches, across sessions | 82 % accepted, 0 of 1,764 other-finger comparisons accepted |

This is new code tested on one laptop. Keep a password fallback (`pam_fprintd` as
`sufficient`) and treat it as convenience, not as a security boundary, until it has seen more
hardware and more people. Reports from other `2808:a658` machines are very welcome.

How it works:

- **Capture**: waits for the finger to settle, keeps the best of up to 4 frames, rejects empty
  frames, and recalibrates the sensor if its finger-detect baseline was taken with a finger on it.
- **Matching**: the sensor sees only ~3×4 mm of finger, too little for libfprint's minutiae
  matcher. Instead: SIFT-style keypoints and descriptors, a pose found by keypoint pairs that
  agree in position and orientation, then a check that the ridges actually line up.
- **Enrollment**: 20 touches, near-duplicates rejected, registered into one template so a later
  touch can match across several enrolled touches.

## Install

### Arch Linux

```sh
git clone https://github.com/tattwamashi/libfprint-ft9366-pkg
cd libfprint-ft9366-pkg
makepkg -si
```

It builds the official libfprint 1.94.100 release with this driver as a patch, with Arch's build
options and every upstream driver, and replaces the `libfprint` package. Prebuilt packages are
on that repository's [releases](https://github.com/tattwamashi/libfprint-ft9366-pkg/releases).
An AUR package (`libfprint-ft9366`) will follow once AUR registration reopens.

Then:

```sh
fprintd-enroll          # 20 touches; place the finger as you would to unlock
fprintd-verify
```

Fingerprint for `sudo`, with the password as fallback:

```sh
sudo sed -i '1a auth\t\tsufficient\tpam_fprintd.so max-tries=3 timeout=10' /etc/pam.d/sudo
```

### From source

```sh
meson setup build -Ddrivers=ft9366 -Ddoc=false -Dgtk-examples=false -Dintrospection=false
ninja -C build
```

## Tips

- The sensor often sits on the power button. A light touch is enough, and a firm press may
  power the laptop off.
- The sensor is tiny: enroll with the finger placed the way you will actually touch it, shifting
  a millimetre or two between touches.
- If a scan fails, lift the finger completely before trying again.

## Credits

- **V8V88V8V88**: the original FT9366 driver this work builds on (USB protocol, finger detection,
  capture), and the [ft9366-rek](https://github.com/V8V88V8V88/ft9366-rek) protocol notes.
- **Tattwamashi Nayak**: capture reliability, the matcher, enrollment, packaging.
- The **libfprint** project.

License: LGPL-2.1-or-later, as libfprint.
