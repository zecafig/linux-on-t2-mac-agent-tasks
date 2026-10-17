# TROUBLESHOOTING.md

# Real Failures and Verified Findings

This file records problems encountered during installation and QA.

The current and tested Touch ID implementation is
[`jmurth1234/t2-touchid-linux`](https://github.com/jmurth1234/t2-touchid-linux).
The installer, units, helpers, and authoritative recovery procedures are in
that upstream repository, not this documentation repository. Follow its
current troubleshooting instructions before attempting ad-hoc changes.

---

## 1. Standard Ubuntu live ISO freezes on T2 Mac

### Symptom

The standard Ubuntu ISO froze at the live-environment splash screen with SQUASHFS/xz errors.

### Resolution

Use the T2-specific Ubuntu image from the t2linux project.

Tested image:

```text
ubuntu-26.04-7.1.8-t2-resolute.iso
```

---

## 2. T2 NCM interface has no IPv6 link-local address

### Symptom

The Touch ID transport cannot reach BridgeOS because the T2 USB-NCM interface has no usable IPv6 link-local address.

First set `T2_NCM_IF` to the interface actually discovered on this Mac. The
test machine used `t2_ncm`; do not assume that name on another machine.

```bash
T2_NCM_IF="replace-with-discovered-interface"
```

Check:

```bash
ip -6 addr show dev "$T2_NCM_IF"
ip -6 route show dev "$T2_NCM_IF"
ip -6 neigh show dev "$T2_NCM_IF"
```

The interface may exist while the T2 network path is not ready.

### Important distinction

The Linux-side link-local address is **not** the BridgeOS peer.

Both must be discovered:

- Linux-side link-local address;
- BridgeOS link-local peer.

Do not invent the peer address from the interface MAC.

### Diagnostics

```bash
ip -details link show dev "$T2_NCM_IF"
ip -6 addr show dev "$T2_NCM_IF"
ip -6 route show dev "$T2_NCM_IF"
ip -6 neigh show dev "$T2_NCM_IF"
nmcli device show "$T2_NCM_IF"
```

Then:

```bash
systemctl status t2-bridge-network.service --no-pager
journalctl -b --no-pager | grep -Ei 't2-bridge-network|t2_ncm|bridgeos|touchid|sep|remotexpc'
```

Do not modify `/etc/resolv.conf` as a response to a T2 IPv6 connectivity problem. The T2 link is a separate private network path.

---

## 3. BridgeOS peer cannot be discovered

### Symptom

The discovered T2 USB-NCM interface exists, but no BridgeOS IPv6 neighbor
appears.

Check:

```bash
ip -6 neigh show dev "$T2_NCM_IF"
```

Verify that the T2 interface is present, administratively up, has its
link-local IPv6 address, and that `t2-bridge-network.service` is active.

Do not use the Linux-side link-local address as the BridgeOS peer.

If the peer remains unavailable, stop before modifying Touch ID, PAM, or SEP configuration.

---

## 4. Touch ID service starts before the T2 network is ready

### Symptom

The current Touch ID stack reports that the discovered T2 USB-NCM interface
has no usable IPv6 link-local address or BridgeOS is not reachable.

### Interpretation

The T2 network path was not ready when SEP transport attempted to start.

A working interface appearing later does not prove that the original startup
sequence was correct.

### Current upstream approach

The current workflow orders the relevant components:

```text
T2 bridge network
        ↓
BridgeOS peer verification
        ↓
biometric BridgeXPC port refresh
        ↓
SEP transport
        ↓
keybag loading
        ↓
fprintd readiness
```

Do not assume the biometric port refresh has its own service unit or recreate
this ordering with arbitrary systemd overrides. Check the installed upstream
units and current upstream troubleshooting instructions first.

### Diagnostics

```bash
systemctl status t2-bridge-network.service --no-pager
systemctl status t2-sep-transport.service --no-pager
systemctl status t2-keybag-load.service --no-pager
systemctl status fprintd.service --no-pager
systemctl list-dependencies t2-sep-transport.service --no-pager
```

Then:

```bash
journalctl -b --no-pager | grep -Ei 't2-bridge-network|t2-sep-transport|t2-keybag-load|t2-touchid|fprintd|remotexpc|biometr'
```

---

## 5. Fingerprint is visible but not usable

If:

```bash
fprintd-list "$USER"
```

does not show the expected enrolled fingerprint, do not immediately modify PAM.

First inspect the current `t2-touchid-linux` configuration and identity state.

Follow the upstream provisioning/reconciliation procedure rather than manually editing fingerprint databases.

---

## 6. Catacomb provisioning, keybag loading, or unlocking fails

Treat these as distinct stages: Catacomb provisioning validates and installs
the exported Catacomb; `t2-keybag-load.service` loads the keybag; and
`t2-keybag-unlock` unlocks the loaded handles. Do not retry unlock as a
substitute for a failed provisioning or load step.

Check:

```bash
systemctl status t2-keybag-load.service --no-pager
sudo ls -la /run/t2-touchid
```

Recovery:

```bash
sudo t2-keybag-unlock
```

Run the unlock helper only after confirming that the current boot's
`t2-keybag-load.service` succeeded. If provisioning or loading failed, collect
the upstream diagnostics and stop before PAM changes.

The Linux and macOS passwords are separate credentials.

Never put either password in command-line arguments, environment variables, shell history, persistent plaintext files, or logs.

---

## 7. Historical Touch ID failure after automatic suspend

Automatic suspend was temporarily enabled during QA of the previous
implementation.

The previous implementation reproduced a Touch ID failure after resume even when `t2_ncm`, IPv6 addressing, `fprintd`, and the Touch ID service appeared healthy.

Restarting `fprintd` and the previous implementation's Touch ID service did
not recover it. A reboot restored Touch ID.

This finding belongs to the previous implementation. It has not been
reproduced with the current `jmurth1234/t2-touchid-linux` implementation.

> **Do not enable automatic suspend until current-implementation behavior has been verified.**

Test suspend/resume independently after normal operation is verified.

---

## 8. Fingerprint login does not unlock GNOME keyring

Fingerprint verification does not inherently provide the user's plaintext password to `pam_gnome_keyring`.

The tested workflow is to log in with the password after reboot so the GNOME
login keyring unlocks, then use Touch ID for sudo and lock-screen unlock.

Do not make the keyring passwordless or weaken Chrome password storage merely to hide the prompt.

---

## 9. GDM password-first restriction was not confirmed

A previous configuration attempted to require a password after restart and display:

```text
Password required after restart
```

The tested login screen continued to display the fingerprint prompt.

Therefore the custom banner and GDM fingerprint restriction were not confirmed.

**Status: UNRESOLVED / DEFERRED**

Do not modify a working PAM/T2/Touch ID configuration merely to force this behavior.

---

## 10. `fprintd` reports an unspecified device error

Logs may contain:

```text
Error receiving instruction data: Got empty data
```

and:

```text
Device reported an error during verify: An unspecified error occurred!
```

These messages were recorded during earlier QA; do not assume they identify a
current-implementation failure without reproducing them. Use them as symptoms
to investigate, not as a diagnosis.

Do not immediately delete fingerprints or reinstall PAM.

Collect:

```bash
systemctl status fprintd --no-pager
fprintd-list "$USER"
systemctl status t2-bridge-network.service --no-pager
systemctl status t2-sep-transport.service --no-pager
systemctl status t2-keybag-load.service --no-pager
systemctl list-dependencies t2-sep-transport.service --no-pager
```

Then:

```bash
journalctl -b --no-pager | grep -Ei 't2-touchid|t2_ncm|fprintd|biometr|remotexpc|sep|keybag|catacomb'
```

Pay particular attention to startup ordering.

---

## 11. PAM changes cause authentication problems

Never install PAM integration before direct fingerprint verification passes.

Positive control:

```bash
fprintd-verify -f any "$USER"
```

An enrolled finger must match.

Negative control must produce:

```text
verify-no-match
```

using an unenrolled finger.

The upstream `tools/install-pam.sh` expects Arch-style
`/etc/pam.d/system-auth` ordering and may install Omarchy-specific lock-screen
stacks. Do not use it as a generic Ubuntu PAM procedure. Only proceed with a
PAM change after confirming that the procedure supports the installed Ubuntu
release and actual PAM service files, and after preparing a verified rollback
path with a root shell open.

The upstream `tools/rollback-pam.sh` is for restoring changes made by its
installer; do not assume it can roll back a different Ubuntu-specific PAM
change. Use the rollback procedure corresponding to the actual edit.

Do not remove password authentication as a troubleshooting step.

---

## 12. macOS Touch ID exports are missing or invalid

Boot macOS and confirm:

- password authentication works;
- the intended Touch ID finger is enrolled;
- the intended finger works in macOS.

Run the upstream export helpers from the `t2-touchid-linux` checkout on
macOS, following that project's current instructions:

```text
tools/macos/
```

Transfer the resulting files privately.

Never commit or publish exported material.

If the export is incomplete or invalid, stop rather than constructing replacement files manually.

---

## 13. Never use `/etc/resolv.conf` to fix T2 Touch ID

The T2 Touch ID connection uses its own private IPv6 path.

A normal systemd-resolved configuration such as:

```text
nameserver 127.0.0.53
```

in `/etc/resolv.conf` is not evidence of a Touch ID networking problem.

Check the actual T2 path:

```bash
T2_NCM_IF="replace-with-discovered-interface"
ip -6 addr show dev "$T2_NCM_IF"
ip -6 route show dev "$T2_NCM_IF"
ip -6 neigh show dev "$T2_NCM_IF"
```

Use `resolvectl status` only when diagnosing general DNS behavior.

Do not replace `/etc/resolv.conf` with a static DNS configuration as a response to a T2 Touch ID failure.

---

## Troubleshooting philosophy

1. Reproduce the failure.
2. Record the exact symptom.
3. Check the relevant subsystem before changing it.
4. Inspect startup ordering when the failure occurs after boot.
5. Check T2 network readiness before changing fingerprint configuration.
6. Avoid speculative fixes.
7. Prefer reversible mitigations.
8. Verify the mitigation.
9. Document both failure and recovery.

A workaround is not automatically a fix.

## Final safety rules

> **Keep macOS available.**

> **Keep password authentication working.**

> **Do not commit macOS keybags, Catacomb archives, credentials, or other private T2/Touch ID state.**

> **Do not modify PAM until both positive and negative fingerprint controls pass.**

> **Do not unload a T2 kernel module after it has registered SEP DMA. Reboot before replacing it.**

> **Do not perform destructive or irreversible Touch ID identity operations without explicit acknowledgement and a verified recovery path.**
