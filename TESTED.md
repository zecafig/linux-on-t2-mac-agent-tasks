# TESTED.md

# Tested Hardware / QA Matrix

This file records what was actually exercised on the test machine.

Do not mark a feature as PASS because an installation procedure exists. A PASS requires an actual test on the documented machine.

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

## Touch ID

### Current implementation

The current and tested implementation is
[`jmurth1234/t2-touchid-linux`](https://github.com/jmurth1234/t2-touchid-linux).
On the test machine, the implementation is installed and in use. The upstream
project supplies the installer, export helpers, provisioning utilities, PAM
helpers, and systemd units; they are not part of this documentation repository.

| Area | Result |
|---|---|
| macOS Touch ID enrollment | TESTED |
| macOS keybag and Catacomb export | TESTED |
| Private transfer of exports | TESTED |
| T2 USB-NCM discovery | TESTED |
| BridgeOS IPv6 peer discovery | TESTED |
| Upstream `t2-touchid-linux` installation | TESTED |
| T2 bridge network and startup ordering | TESTED |
| Biometric BridgeXPC port discovery before SEP transport | TESTED |
| Keybag loading and unlock | TESTED |
| Positive `fprintd-verify` control | TESTED |
| Negative `fprintd-verify` control | TESTED |
| sudo fingerprint and password fallback | TESTED |
| GNOME lock-screen fingerprint and password fallback | TESTED |
| Reboot recovery and post-boot operation | TESTED |
| Current implementation suspend/resume on this Ubuntu installation | NOT YET TESTED |
| Ubuntu-specific PAM procedure used on the test machine | NOT RECORDED |
| Installed upstream revision used on the test machine | NOT RECORDED |

The installed upstream commit was not recorded. Capture it from the tested
installation before relying on version-specific upstream instructions or
reproducing its setup.

### Direct verification and Ubuntu PAM scope

Before any PAM changes, run the following command separately for each direct
`fprintd` control:

```bash
fprintd-verify -f any "$USER"
```

The enrolled finger must match and an unenrolled finger must return
`verify-no-match`. The current upstream `tools/install-pam.sh` expects
Arch-style `/etc/pam.d/system-auth` ordering and may install Omarchy-specific
lock-screen stacks. It is not a generic Ubuntu PAM procedure; do not run it on
Ubuntu unless upstream explicitly supports the installed Ubuntu release and
the helper's preflight confirms the actual PAM stack.

Sudo and GNOME lock-screen fingerprint authentication have been tested on
this Ubuntu machine, but the Ubuntu-specific PAM installation procedure and
exact upstream commit used for that setup are not recorded here. Keep password
authentication as fallback and use a verified root recovery/rollback path
before any PAM changes.

When validating the already configured test machine, test:

```bash
sudo -k
sudo -v
```

Password authentication must remain available as fallback.

### Login and keyring behavior

The tested workflow is password-first after reboot:

1. Password login after reboot unlocks the GNOME login keyring.
2. After password login, Touch ID works for sudo.
3. After password login, Touch ID works for GNOME lock-screen unlock.

Fingerprint authentication does not provide the plaintext password required
to unlock the login keyring. Do not weaken keyring security merely to remove a
prompt.

### Historical suspend / resume finding

Automatic suspend was temporarily enabled during QA of the previous
implementation.

The previous implementation reproduced a Touch ID failure after resume. `t2_ncm`, IPv6 addressing, `fprintd`, and the Touch ID service could appear healthy while biometric communication still timed out. Restarting the relevant services did not recover Touch ID; a reboot restored it.

This finding belongs to the previous implementation and must not be
attributed to the current `t2-touchid-linux` implementation. Suspend/resume
with the current implementation has not yet been tested on this Ubuntu
installation.

The current upstream README reports `s2idle` suspend/resume verification on
its proven configurations and says `deep` remains broken there. That upstream
evidence does not verify suspend/resume on this Ubuntu installation.

> **Keep automatic suspend disabled on this Ubuntu installation until its behavior has been explicitly verified here.**

## Remaining tests

- current implementation suspend/resume behavior on this Ubuntu installation
- capture the installed upstream commit and document the Ubuntu-specific PAM
  procedure used by the tested machine
- formal trackpad testing
- 3.5 mm headset microphone
- individual USB-C ports
- other supported power-management testing
