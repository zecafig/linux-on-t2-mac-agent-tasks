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

## Before booting the USB: change Startup Security Utility settings (in macOS)

T2 Macs refuse to boot Linux media by default. This setting cannot be changed
from the normal macOS desktop or from the Option-key Startup Manager. It lives
in **Startup Security Utility**, which is only available from macOS Recovery.
Do this before trying to boot the installer USB
([t2linux pre-install guide](https://wiki.t2linux.org/guides/preinstall/#disable-secure-boot),
[Apple documentation](https://support.apple.com/en-us/102522)):

1. Shut down the Mac completely.
2. Power on and immediately hold **Command (⌘)-R** until the Apple logo or
   spinning globe appears (macOS Recovery).
3. If asked, select your user and enter its administrator password.
4. In the menu bar choose **Utilities → Startup Security Utility**.
5. Click **Enter macOS Password**, choose an administrator account, and
   authenticate again.
6. Set **Secure Boot** to **No Security**.
7. Set **Allow Boot Media** to **Allow booting from external or removable
   media**.
8. Close the utility, restart, and hold **Option (⌥)** to open Startup
   Manager. Pick the orange **EFI Boot** entry (if there are two, try the
   rightmost first).

Notes:

- This is the setting users report as "can't change boot config": it is
  not in System Settings and not in the Option boot menu. Use Recovery.
- If Startup Security Utility is greyed out or asks for a password, a
  firmware password may be set; turn it off first
  ([Apple](https://support.apple.com/en-us/102384)).
- Use the built-in or a wired Apple keyboard; wireless and PC keyboards
  often fail to register the Command-R key combination.
- If Recovery reports error **-1008F**, the Mac has not reached Startup
  Security Utility; this is a Recovery-startup problem, not the Linux USB
  boot-security setting. Apple documents **Option-Command-R** as another
  startup combination for Internet Recovery on Intel Macs. If the error
  persists, check whether **Activation Lock** is involved: if you can boot
  macOS, turn off Find My Mac in System Settings; otherwise, the owner can
  remove the Mac from their account through
  [iCloud Find Devices](https://www.icloud.com/find). A previously owned Mac
  may require its previous owner to remove Activation Lock. See
  [Apple's Recovery guidance](https://support.apple.com/en-us/102518) and
  [Activation Lock guidance](https://support.apple.com/en-us/102541).
- If you see "A software update is required to use this startup disk",
  Secure Boot is still being enforced. Re-check step 6, or try the other
  orange EFI entry.
- Afterwards you may set "Disallow booting from external or removable media"
  again; the internal Linux install keeps booting from Startup Manager.
  Keep Secure Boot at No Security, as Linux will not boot otherwise.
- Intel T2 Macs only. Apple silicon Macs are not supported by this guide.

## Installation journey

This playbook does not replace the Ubuntu installation guide. Install Ubuntu
with the T2-supported kernel using the
[t2linux Ubuntu guide](https://wiki.t2linux.org/distributions/ubuntu/installation/)
first, then use `INSTALL.md` for the agent-assisted Touch ID setup.

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

The current and tested Touch ID implementation is
[`jmurth1234/t2-touchid-linux`](https://github.com/jmurth1234/t2-touchid-linux).
The upstream repository supplies the installer, macOS export helpers, PAM
helpers, provisioning utilities, and systemd units; this repository contains
documentation only. See [INSTALL.md](./INSTALL.md) for the current setup
sequence.

On the test machine, the T2 NCM interface required a NetworkManager
connection bound explicitly to `t2_ncm` so that it received an IPv6
link-local address. Discover the actual interface on other Macs.

The resulting setup supports:

- GNOME lock-screen fingerprint unlock
- `sudo` fingerprint authentication

The GNOME keyring behaves differently depending on how the session starts:

- Password login unlocks the GNOME keyring automatically.
- Fingerprint authentication does not provide the plaintext password required
  by `pam_gnome_keyring`, so Chrome can subsequently request the keyring
  password if the keyring was not unlocked by password login.

For security, the project does **not** use passwordless keyrings and does not recommend disabling Chrome's secure password storage.

The current tested operational model is:

> **After reboot, use the password once. After that, Touch ID can be used for sudo and lock-screen authentication.**

The GDM password-first restriction and login-screen banner remain unresolved.
Use password login after reboot for the tested keyring workflow; do not alter
PAM merely to force an unverified GDM policy.

## Important power-management finding

**Keep automatic suspend disabled until current-implementation behavior is verified.**

During QA of the previous Touch ID implementation, automatic suspend was
temporarily enabled specifically to test suspend/resume behavior. That test
reproduced a Touch ID failure after resume. This was not reproduced with the
current `jmurth1234/t2-touchid-linux` implementation; its suspend/resume
behavior on this machine has not yet been verified.

The upstream README reports `s2idle` suspend/resume verified on its proven
configurations and says `deep` remains broken there. Neither result verifies
suspend/resume on this Ubuntu installation.

With the previous implementation, the T2 network interface could remain
healthy and the fingerprint could remain visible to `fprintd`, while the
Touch ID daemon reported that BiometricKit was not advertised and timed out.

Restarting `fprintd` and the previous implementation's Touch ID service did
not recover the sensor. A reboot restored Touch ID.

Therefore:

> **Do not enable automatic suspend until the current implementation's behavior has been tested.**

See `TROUBLESHOOTING.md` and `TESTED.md`.

## Documentation

- `INSTALL.md` — the ordered agent-assisted Touch ID setup.
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
- [Current Touch ID implementation: `jmurth1234/t2-touchid-linux`](https://github.com/jmurth1234/t2-touchid-linux)
- [Ubuntu](https://ubuntu.com/)

## Thanks to

Area31 Hackerspace, Slackware-br (SJ), and BHack friends.

I've been learning and drinking beer with these guys since 1998.

## Special thanks

A very special thank you to my **Caverna brothers**. ❤️

I've had the enormous pleasure of working with you guys, and that makes this whole QA journey even more enjoyable.

Love you all.
## License

This repository's original material is dedicated to the public domain under **CC0**.

CC0 applies only to original material in this repository. It does not change the licenses or ownership of upstream projects, software, documentation, trademarks, or other third-party material referenced here.
