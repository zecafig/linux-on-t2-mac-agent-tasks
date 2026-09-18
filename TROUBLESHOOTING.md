# TROUBLESHOOTING.md

# Real Failures and Verified Findings

This file records problems encountered during the installation and QA process.

The purpose is not to hide failures. A real installation guide should preserve them because they are often the most useful information for the next person.

## 1. Standard Ubuntu live ISO freezes on T2 Mac

### Symptom

The standard Ubuntu ISO froze at the live-environment splash screen with SQUASHFS/xz errors.

### Resolution

Use the T2-specific Ubuntu image from the t2linux project.

Tested image:

`ubuntu-26.04-7.1.8-t2-resolute.iso`

The ISO was written to USB and independently verified byte-for-byte against the ISO-sized range on the USB.

### Upstream

- T2-Ubuntu: https://github.com/t2linux/T2-Ubuntu
- t2linux: https://t2linux.org/

---

## 2. T2 NCM interface had no IPv6 link-local address

### Symptom

`t2-touchid` reported that it could not reach the sensor because `t2_ncm` had no IPv6 link-local address.

The interface existed but NetworkManager did not initially have the expected connection associated with it.

### Verified configuration

The existing `Apple T2 Bridge` NetworkManager profile was bound to `t2_ncm`:

```bash
sudo nmcli connection modify "Apple T2 Bridge" connection.interface-name t2_ncm
sudo nmcli connection up "Apple T2 Bridge"
```

Afterward:

```bash
ip -6 addr show dev t2_ncm
```

showed a link-local IPv6 address.

Touch ID then became available.

---

## 3. Fingerprint was visible but not bound to the Linux user

### Symptom

The T2 Touch ID service could see the enrolled macOS finger but `fprintd-list` did not initially show a fingerprint bound to the Linux user.

### Resolution

Create the local Touch ID configuration from the package template:

```bash
sudo mkdir -p /etc/kait2en
sudo cp /usr/share/kait2en/t2-touchid.conf /etc/kait2en/
```

Set the binding:

```text
T2_TOUCHID_BIND_USER=<your-linux-username>
```

Restart the service:

```bash
sudo systemctl restart kait2en-t2-touchid.service
```

The service then reported the enrolled finger as bound to the Linux user.

`fprintd-list` subsequently showed:

```text
Fingerprints for user <your-linux-username>
- #0: right-index-finger
```

---

## 4. Touch ID fails after automatic suspend

### This is the important one

**KEEP AUTOMATIC SUSPEND DISABLED.**

Automatic suspend was already intended to remain disabled in the normal configuration.

During QA, it was temporarily enabled specifically to test suspend/resume behavior.

### Reproduction

After automatic suspend and resume:

- Touch ID authentication timed out.
- `t2_ncm` could still be up.
- `t2_ncm` could still have its IPv6 link-local address.
- `fprintd` could still see the enrolled finger.
- `kait2en-t2-touchid.service` remained active.

The Touch ID daemon reported:

```text
t2-touchid: cannot reach the sensor: BiometricKit was not advertised: connection timed out
```

### Recovery attempts

These did **not** restore Touch ID:

```bash
sudo systemctl restart fprintd
```

or:

```bash
sudo systemctl restart kait2en-t2-touchid.service
```

A reboot restored Touch ID.

### Mitigation

Disable automatic suspend and leave it disabled.

Do not treat this as a reason to make arbitrary changes to the T2 transport or unload kernel modules. The failure was reproduced and documented rather than hidden behind an unverified workaround.

---

## 5. GNOME keyring is not unlocked by fingerprint login

### Symptom

After logging into GNOME using fingerprint, Chrome can ask for the GNOME keyring password.

After logging in with the account password, Chrome does not ask for the keyring password.

### Evidence

The GNOME PAM configuration includes `pam_gnome_keyring`.

During fingerprint login the journal reported:

```text
gkr-pam: no password is available for user
```

This is consistent with the important distinction between password authentication and fingerprint authentication: fingerprint verification does not inherently provide the user's plaintext password to `pam_gnome_keyring`.

### Security decision

Do **not**:

- make the keyring passwordless;
- switch Chrome to insecure/basic password storage;
- install extra software merely to hide the prompt.

### Current operational solution

After reboot:

> **Log in with the password once.**

The GNOME keyring is then unlocked, Chrome starts without the keyring prompt, and fingerprint authentication remains available for:

- sudo;
- lock-screen unlock;
- subsequent authentication.

This behavior is intentionally documented rather than "fixed" by weakening security.

---

## 6. GDM fingerprint-login restriction and custom login banner are not yet working

### Current configuration

The following file exists:

```text
/etc/dconf/db/local.d/00-login-fingerprint
```

with:

```text
[org/gnome/login-screen]
enable-fingerprint-authentication=false
```

A separate login-message configuration was attempted:

```text
/etc/dconf/db/local.d/01-login
```

with:

```text
[org/gnome/login-screen]
banner-message-enable=true
banner-message-text='Password required after restart'
```

### Observed result

The tested GDM screen did **not** show the intended custom message.

Instead, the screen displayed the fingerprint prompt:

> **Swipe your right index finger across the fingerprint reader**

After pressing `Esc`, the user could enter the password, and fingerprint authentication remained available as an alternative.

Therefore:

- the custom banner is **not confirmed working**;
- the GDM fingerprint restriction is **not working**;
- fingerprint login itself remains functional;
- the password-first keyring behavior remains a separate, working security-preserving procedure.

### Status

**UNRESOLVED / DEFERRED**

Do not change the working PAM, `fprintd`, `t2-touchid`, sudo, or lock-screen configuration merely to force this behavior.

The desired future state is:

> Password required after reboot → GNOME keyring unlocked → fingerprint available for sudo and lock-screen authentication.

The GDM-only enforcement and login message will be investigated later as separate controlled changes.

---

## Troubleshooting philosophy

When a failure occurs:

1. Reproduce it.
2. Record the exact symptom.
3. Check the relevant subsystem before changing it.
4. Avoid speculative fixes.
5. Prefer a reversible mitigation.
6. Verify the mitigation.
7. Document both the failure and the successful recovery.

A workaround is not automatically a fix.

And a test that breaks something is still valuable QA if it tells us what not to enable.

## Final warning for this configuration

> **KEEP AUTOMATIC SUSPEND DISABLED.**

The suspend test is what exposed the reproducible Touch ID failure after resume. The normal configuration should leave automatic suspend disabled.
