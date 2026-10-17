# Agent-Assisted T2 Touch ID Setup

This guide starts after Ubuntu is installed and boots on an Intel Mac with an
Apple T2 chip. It covers the current Touch ID setup, not a complete Ubuntu
installation manual.

The current and tested Touch ID implementation is
[`jmurth1234/t2-touchid-linux`](https://github.com/jmurth1234/t2-touchid-linux).
Its installer, macOS export helpers, PAM helpers, provisioning utilities, and
systemd units come from that upstream checkout. This repository contains
documentation only; it does not contain those tools.

## Before starting

If Ubuntu/T2 kernel installation is not complete, follow the
[t2linux Ubuntu installation guide](https://wiki.t2linux.org/distributions/ubuntu/installation/)
and its [pre-install guide](https://wiki.t2linux.org/guides/preinstall/) first.
Keep macOS installed and bootable as a recovery environment.

Before any disk or partition operation, inspect the actual device and
partition layout, explain the proposed change and its consequences, make sure
the user has a current backup, and obtain explicit confirmation. Never infer
device names or partition numbers from another Mac.

On Ubuntu, confirm that the machine is booted with the T2-supported kernel and
that the T2 USB-network interface is present. At minimum, inspect:

```bash
uname -r
ip -brief link
```

Use the current t2linux post-install instructions to resolve missing kernel
support. Do not proceed to Touch ID setup if the T2 USB-network interface or
kernel support is unavailable.

Keep Linux password authentication available throughout setup. Never
permanently store, log, or pass a macOS password in command arguments or
environment variables.

## 1. Prepare and export Touch ID state in macOS

Boot macOS before configuring Touch ID on Linux. Confirm that macOS remains
bootable, the macOS password works, and the intended enrolled finger unlocks
the Mac.

On macOS, clone the
[upstream `t2-touchid-linux` repository](https://github.com/jmurth1234/t2-touchid-linux)
and follow its current README instructions for the export helpers in that
checkout's `tools/macos/` directory. Do not assume these helpers exist in this
repository or invent helper commands.

Transfer the exported keybag and Catacomb material privately to Ubuntu. These
exports are sensitive private data: never commit, upload, paste into chat, or
otherwise publish them. If an export is missing or invalid, stop and resolve
it using the upstream instructions.

## 2. Discover the T2 USB-NCM and BridgeOS path on Ubuntu

Discover the interface on the actual machine; do not assume its name. `t2_ncm`
was observed on the test Mac, but other names are possible.

Inspect the interfaces, IPv6 addresses, routes, and neighbors:

```bash
ip -details link
ip -6 addr
ip -6 route
ip -6 neigh
nmcli device
```

Identify and record separately:

1. the actual T2 USB-NCM interface;
2. Linux's IPv6 link-local address on that interface;
3. the BridgeOS IPv6 link-local peer on that interface.

The Linux-side address is not the BridgeOS peer. Do not guess a peer from the
Linux address or interface MAC. If the peer cannot be established, stop and
follow the upstream Linux bring-up troubleshooting before continuing.

The T2 USB-NCM/BridgeOS connection is a private transport path, separate from
Wi-Fi, ordinary DNS, and `/etc/resolv.conf`. Do not change DNS configuration
to fix a T2 transport problem. A normal systemd-resolved entry such as
`nameserver 127.0.0.53` is not evidence of a T2 connectivity failure.

## 3. Install and configure the current upstream implementation

On Ubuntu, clone the upstream repository and inspect its current installation
instructions before running privileged code:

```bash
git clone https://github.com/jmurth1234/t2-touchid-linux.git
cd t2-touchid-linux
git rev-parse HEAD
```

Record the checked-out revision for troubleshooting. Run the following from
that upstream checkout, not this documentation repository:

Before executing it, read the upstream README's prerequisites, installation,
and recovery sections; inspect the installer and its effects; and understand
the configuration values discovered for this machine. The upstream project
describes its proven configurations and distro-specific behavior. Do not
assume that a successful upstream installation on Arch/Omarchy makes every
installer step safe on Ubuntu.

```bash
sudo ./install.sh
```

Follow the installer and upstream README to configure
`/etc/t2-touchid.conf` with values discovered on this Mac, including the T2
USB-NCM interface, BridgeOS peer, numeric macOS user ID, and corresponding
special bag identifier. Do not copy machine-specific values from this
repository or another Mac.

The current upstream setup orders T2 bridge networking, BridgeOS biometric
port discovery, SEP transport, and keybag loading. Use the upstream units and
diagnostics; do not add guessed systemd overrides or assume a separate
biometric-port service unit exists.

## 4. Provision, load, and unlock keybag state

These are separate operations. Follow the current upstream README and
installer for any additional validation or required files.

1. Place the extracted keybag at the upstream-documented location with the
   required root ownership and restrictive permissions. The current upstream
   README documents `/var/lib/t2-touchid/user.kb` with root ownership and mode
   `0600`.
2. From the upstream checkout, provision the exported Catacomb. For example:

   ```bash
   sudo t2-touchid-provision-catacomb /private/path/t2-touchid-catacomb.tar.gz
   ```

   Replace the example path with the actual private transfer path. Do not
   expose the archive or its contents.
3. Load the keybag using the current upstream service:

   ```bash
   sudo systemctl start t2-keybag-load.service
   ```

   Confirm the unit completed successfully and follow upstream diagnostics if
   it fails. Do not try to unlock a keybag that has not loaded.
4. Unlock the loaded handles using the upstream helper:

   ```bash
   sudo t2-keybag-unlock
   ```

   This helper is installed by the upstream project; it is not a file in this
   repository. The Linux and macOS passwords are separate credentials even if
   they happen to be identical. Enter the macOS password only at the
   interactive prompt.
5. Confirm that the upstream readiness checks and `fprintd` inventory show
   Touch ID usable before changing PAM.

## 5. Verify direct fingerprint operation before PAM

Run `fprintd` tests before installing or changing PAM integration:

```bash
fprintd-list "$USER"
fprintd-verify -f any "$USER"
```

Run `fprintd-verify` separately for each control:

- an enrolled finger returns a successful match;
- a deliberately unenrolled finger returns `verify-no-match`.

If either control fails, stop and diagnose the T2 network, BridgeOS
reachability, SEP transport, keybag/Catacomb state, and `fprintd` using the
upstream troubleshooting instructions. Do not proceed to PAM.

## 6. Stop before PAM changes unless an Ubuntu procedure is verified

The upstream `tools/install-pam.sh` currently expects Arch-style
`/etc/pam.d/system-auth` ordering and can install Omarchy-specific lock-screen
stacks. It is **not a generic Ubuntu PAM installer**. Do not run
`tools/install-pam.sh` or `tools/rollback-pam.sh` as part of this Ubuntu
procedure.

The test machine has demonstrated sudo and GNOME lock-screen fingerprint
authentication on Ubuntu, but this repository does not record a reproducible,
Ubuntu-specific PAM installation procedure or rollback path. Do not infer one
from the upstream Arch/Omarchy helper. This guide therefore stops before PAM:
without a separately verified procedure for the installed Ubuntu release and
actual PAM service files, make no PAM changes and do not continue to PAM
authentication testing.

Before any PAM change, keep a working root recovery shell open, verify the
Linux password, and preserve the original files. Any Ubuntu-specific rollback
must be verified for the actual changes made. Keep password authentication as
a recovery path; never remove it as a troubleshooting step.

Only after a verified Ubuntu-specific procedure has been established and
applied may an agent perform these checks:

1. `sudo -k` followed by `sudo -v` accepts a valid enrolled finger and rejects
   an invalid one;
2. Linux password fallback still works;
3. the GNOME lock screen accepts the enrolled finger and retains password
   fallback.

Do not remove password authentication as a troubleshooting step.

## 7. Verify reboot and keyring behavior

Reboot and verify that T2 networking and BridgeOS discovery become ready in
the expected startup order, SEP transport and keybag loading complete, and
fingerprint authentication works again. Record upstream service status and
boot journal output if anything fails; see [TROUBLESHOOTING.md](./TROUBLESHOOTING.md).

The current upstream README documents `t2-bridge-network.service`,
`t2-sep-transport.service`, `t2-keybag-load.service`, and
`t2-touchid-post-reboot.service`. Verify the installed unit names and inspect
their boot status rather than assuming units from another implementation:

```bash
systemctl status t2-bridge-network.service t2-sep-transport.service \
  t2-keybag-load.service t2-touchid-post-reboot.service --no-pager
journalctl -b --no-pager
```

The tested workflow uses the Linux password after reboot to log in and unlock
the GNOME login keyring. Once logged in, Touch ID can be used for sudo and
lock-screen unlock. Fingerprint authentication does not supply the plaintext
password required to unlock the login keyring. Do not weaken keyring security
merely to remove a prompt.

## 8. Treat suspend/resume as separate QA

Do not enable or change suspend behavior merely to make Touch ID work. Test
suspend/resume separately after normal post-reboot behavior is established.
The documented Touch ID failure after resume was reproduced with the previous
implementation; it is not a confirmed failure of the current
`jmurth1234/t2-touchid-linux` setup.

## Stop conditions

Stop and diagnose rather than continuing if:

- T2 kernel support, the USB-NCM interface, or the BridgeOS peer is unavailable;
- macOS exports are missing, invalid, or cannot be transferred privately;
- upstream installation, Catacomb provisioning, keybag loading, or unlocking
  fails;
- either positive or negative direct `fprintd` verification fails;
- password authentication or PAM rollback cannot be confirmed;
- a proposed change could weaken security or risk existing macOS data.

Keep macOS available as a recovery environment and keep password
authentication working throughout the process.
