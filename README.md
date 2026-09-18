# Linux on T2 Mac — Agent-Assisted Installation Playbook

> **Bring your old Mac to modern Linux.**  
> A human-controlled, AI-agent-assisted installation playbook for Linux on Apple T2 Macs.

This repository documents a real installation and QA journey: Ubuntu GNOME installed alongside macOS on an Intel/T2 MacBook Pro, while preserving the existing macOS installation.

The goal is not to pretend that every T2 Mac behaves identically. The goal is to record **what was actually tested, what worked, what failed, and how failures were handled**.

## Project philosophy

The human remains in control.

The AI agent proposes actions, explains them, and helps interpret results. It should never blindly perform destructive operations.

For destructive or irreversible operations:

1. Inspect.
2. Show the relevant output.
3. Explain what will change and what will not.
4. Ask for explicit confirmation.
5. Execute.
6. Verify.
7. Stop if verification fails.

The guiding workflow is:

> **Inspect → interpret → show user → confirm → act → verify**

Never hard-code device identifiers, partition numbers, usernames, or paths when discovery can be used instead.

This project distinguishes carefully between:

- **Supported upstream** — documented or supported by the relevant upstream project.
- **Tested by us** — actually exercised on this specific machine.
- **Known limitation** — a failure reproduced during testing.
- **Not yet tested** — deliberately left open rather than guessed.

## The machine

Test machine:

- MacBook Pro `MacBookPro15,2`
- Intel Core i7-8559U
- 16 GB RAM
- Intel Iris Plus 655
- 1 TB Apple SSD `AP1024J`
- Apple T2 chip
- Touch Bar / Touch ID
- macOS Sequoia `15.7.9`
- Ubuntu `26.04.1 LTS`
- GNOME 50
- Wayland
- T2 kernel `7.2.6-1-t2-resolute`

macOS was preserved completely.

Ubuntu occupies approximately 300 GB of the internal SSD, while the macOS APFS container remains approximately 700 GB.

## Installation journey

The standard Ubuntu ISO was not usable on this T2 Mac: the live environment froze with SQUASHFS/xz errors.

The installation therefore used the T2-specific Ubuntu image from the t2linux project.

The APFS container was reduced from approximately 1 TB to 700 GB in macOS, creating approximately 300 GB of free space. Ubuntu was then installed manually into that free space while reusing the existing Apple EFI System Partition without formatting it.

The resulting machine boots from Apple's Startup Manager with entries for macOS and Ubuntu.

## QA mindset

This installation is also being treated as a real-world QA exercise.

Rather than stopping when Ubuntu boots, we progressively exercise networking, audio, Bluetooth, camera, Touch Bar, brightness, authentication, external displays, peripherals, power management, suspend/resume, and normal application workflows.

The goal is evidence-based:

> **What actually works on this specific machine?**

Good QA also means documenting failures honestly, reproducing them where possible, separating supported behavior from tested behavior, and avoiding changes merely to make a problem disappear without understanding it.

## Authentication / Touch ID

Touch ID was successfully brought up using `t2-touchid`.

The T2 NCM interface required a NetworkManager connection bound explicitly to `t2_ncm` so that it received an IPv6 link-local address.

The Touch ID service was then configured to bind the enrolled fingerprint to the Linux user.

The resulting setup supports:

- GNOME lock-screen fingerprint unlock
- `sudo` fingerprint authentication
- Fingerprint login at GDM (currently still available)

The GNOME keyring behaves differently depending on how the session starts:

- Password login unlocks the GNOME keyring automatically.
- Fingerprint login does not provide the plaintext password required by `pam_gnome_keyring`, so Chrome can subsequently request the keyring password.

For security, the project does **not** use passwordless keyrings and does not recommend disabling Chrome's secure password storage.

The current tested operational model is:

> **After reboot, use the password once. After that, Touch ID can be used for sudo and lock-screen authentication.**

A future task is to make GDM enforce password authentication after reboot while retaining fingerprint authentication for the running session. This is intentionally left unresolved for later rather than forcing an unverified configuration.

A login-screen banner was also attempted, but the displayed login screen still showed the fingerprint prompt rather than the intended custom message. The banner configuration is therefore **not considered working** and is deferred together with the GDM login-policy investigation.

## Important power-management finding

**Keep automatic suspend disabled.**

During QA, automatic suspend was temporarily enabled specifically to test suspend/resume behavior. That test reproduced a Touch ID failure after resume.

After resume, the T2 network interface could remain healthy and the fingerprint could remain visible to `fprintd`, while the Touch ID daemon reported that BiometricKit was not advertised and timed out.

Restarting `fprintd` and restarting `t2-touchid` did not recover the sensor. A reboot restored Touch ID.

Therefore:

> **Do not enable automatic suspend on this tested configuration. Keep it disabled.**

See `TROUBLESHOOTING.md` and `TESTED.md`.

## Documentation

- `INSTALL.md` — the agent-assisted installation protocol and task structure.
- `TESTED.md` — evidence-based compatibility / QA matrix.
- `TROUBLESHOOTING.md` — failures reproduced during this installation and their verified mitigations.

## Upstream projects and documentation

This project is a companion field guide, not an official replacement for t2linux and is not affiliated with Apple, Ubuntu, or the upstream projects below.

- [t2linux](https://t2linux.org/)
- [T2-Ubuntu](https://github.com/t2linux/T2-Ubuntu)
- [T2 Debian and Ubuntu Kernel](https://github.com/t2linux/T2-Debian-and-Ubuntu-Kernel)
- [t2linux Ubuntu installation guide](https://wiki.t2linux.org/distributions/ubuntu/installation/)
- [t2linux pre-install guide](https://wiki.t2linux.org/guides/preinstall/)
- [t2linux post-install guide](https://wiki.t2linux.org/guides/postinstall/)
- [t2linux Touch ID guide](https://wiki.t2linux.org/guides/touchid/)
- [t2linux state](https://wiki.t2linux.org/state/)
- [t2linux audio configuration](https://wiki.t2linux.org/guides/audio-config/)
- [t2linux Wi-Fi / Bluetooth guide](https://wiki.t2linux.org/guides/wifi-bluetooth/)
- [KAIT2EN](https://kait2en.org/documentation.html)
- [t2-touchid-linux](https://github.com/jmurth1234/t2-touchid-linux)
- [Ubuntu](https://ubuntu.com/)

## Thanks to

Area31 Hackerspace and SJ. 

## Special thanks

A very special thank you to the **Caverna guys**. ❤️

I have an enormous pleasure working with you guys, and that makes this whole journey even more enjoyable.

## License

This repository's original material is dedicated to the public domain under **CC0**.

CC0 applies only to original material in this repository. It does not change the licenses or ownership of upstream projects, software, documentation, trademarks, or other third-party material referenced here.
