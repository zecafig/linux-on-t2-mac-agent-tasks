# TESTED.md

# Tested Hardware / QA Matrix

This file records what was actually exercised on the test machine.

## Test machine

- MacBook Pro `MacBookPro15,2`
- Intel Core i7-8559U
- 16 GB RAM
- Intel Iris Plus 655
- 1 TB Apple SSD `AP1024J`
- Apple T2
- Touch Bar
- Touch ID
- macOS Sequoia `15.7.9`
- Ubuntu `26.04.1 LTS`
- GNOME 50
- Wayland
- Kernel `7.2.6-1-t2-resolute`

## Installation state

| Area | Result | Notes |
|---|---|---|
| macOS preservation | PASS | Existing APFS installation preserved |
| APFS resize | PASS | macOS container reduced to ~700 GB |
| Linux free space | PASS | ~300 GB created after APFS resize |
| Ubuntu installation | PASS | Manual installation into the free space |
| Apple EFI reuse | PASS | Existing EFI reused; not formatted |
| Ubuntu boot | PASS | Starts from Apple's Startup Manager |
| macOS boot | PASS | macOS remains available |

## Hardware / desktop testing

| Area | Result | Notes |
|---|---|---|
| Wi-Fi | PASS | Tested |
| Bluetooth phone | PASS | Tested |
| Bluetooth microphone/audio | PASS | Tested |
| Internal speakers | PASS | Tested |
| Internal microphone | PASS | Tested |
| Samsung external display | PASS | Tested |
| Samsung monitor speakers | PASS | Tested |
| Internal camera | PASS | Tested |
| Chrome | PASS | Tested |
| Passkey QR authentication | PASS | Tested |
| Google Meet | PASS | Internal/Bluetooth mic, speakers, camera, Wi-Fi/WebRTC |
| Touch Bar controls | PASS | Volume, playback, brightness, Escape |
| Keyboard backlight | PASS | Tested |
| Display brightness | PASS | Tested |
| Touch ID / GNOME unlock | PASS | Tested |
| Touch ID / sudo | PASS | Tested |
| 3.5 mm headphone output | PASS | Tested |
| 3.5 mm headset microphone | NOT FULLY TESTED | Still open |
| Trackpad | NOT FULLY TESTED | Formal testing pending |
| USB-C hub | PASS / PARTIAL | Flash drive, HDMI, keyboard, mouse, hotplug tested; individual ports pending |
| Battery / charging | PASS | Tested |
| Automatic suspend | DO NOT ENABLE | See suspend/Touch ID finding below |

## Touch ID

### Working

Touch ID works with:

- GNOME lock-screen unlock
- `sudo`
- fingerprint enrollment through the configured `fprintd` interface

The Touch ID service is configured to bind the enrolled finger to the Linux user.

### GDM login / keyring behavior

The current tested behavior is:

1. After reboot, GDM still offers fingerprint authentication.
2. Fingerprint login succeeds.
3. Fingerprint login does not unlock the GNOME login keyring.
4. Password login does unlock the keyring.
5. After password login, Chrome does not prompt for the keyring password.
6. After password login, fingerprint works for sudo.
7. After password login, fingerprint works for lock-screen unlock.

Operationally, the security-preserving procedure is:

> **After reboot, log in with the password once. Then use Touch ID for subsequent authentication.**

A GDM configuration was attempted to display:

> **Password required after restart**

However, the tested login screen still displayed the fingerprint prompt:

> **Swipe your right index finger across the fingerprint reader**

The intended custom banner was therefore **not confirmed to work**.

A separate configuration intended to disable fingerprint authentication at the GDM login screen is also present, but fingerprint login remains available. Both the custom banner and GDM fingerprint restriction are **not yet solved** and should not be presented as completed fixes.

## Suspend / resume finding

Automatic suspend was temporarily enabled during QA specifically to test suspend/resume behavior.

The test reproduced a Touch ID failure after resume.

Observed after resume:

- `t2_ncm` could be `UP` with an IPv6 link-local address;
- `fprintd-list` could still see the enrolled fingerprint;
- `kait2en-t2-touchid.service` could remain active;
- Touch ID communication could nevertheless time out with a BiometricKit advertisement/connection timeout.

Restarting `fprintd` did not recover Touch ID.

Restarting `kait2en-t2-touchid.service` did not recover Touch ID.

Rebooting restored Touch ID.

### Current rule

> **KEEP AUTOMATIC SUSPEND DISABLED.**

The suspend test was intentionally performed to reproduce the limitation. It should not be enabled as part of the normal tested configuration.

## Remaining tests

- Formal trackpad testing
- 3.5 mm headset microphone
- Individual USB-C ports
- Further power-management testing without enabling automatic suspend
- GDM password-first configuration, if pursued later
