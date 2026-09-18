# INSTALL.md

# Agent-Assisted Installation Protocol

This document describes how an AI agent should assist a human installing Linux on an Apple T2 Mac.

The agent is an assistant, not an autonomous installer.

## Core rule

> **Inspect → interpret → show user → confirm → act → verify**

The human must remain in control of the machine and especially of destructive operations.

## Safety rules

### Never blindly execute destructive operations

Before modifying disks, partitions, filesystems, boot configuration, firmware-related settings, or other potentially irreversible state:

1. Inspect the current state.
2. Identify the target precisely.
3. Show the user the relevant command output.
4. Explain:
   - what will change;
   - what will not change;
   - what could go wrong;
   - how the result will be verified.
5. Ask for explicit confirmation.
6. Execute.
7. Verify the result.
8. Stop if verification does not match expectations.

### Never hard-code discovered values

Do not assume:

- `/dev/nvme0n1`
- partition numbers
- usernames
- filesystem UUIDs
- network interface names
- mount points
- disk sizes
- boot entries

Discover them first.

## Suggested task sequence

### TASK 001 — Identify the Mac

Collect model, CPU, memory, storage, T2 status, macOS version, firmware information, and relevant security state.

### TASK 002 — Identify the internal disk

Confirm the physical internal disk and distinguish it from USB/external media.

### TASK 003 — Build the partition safety map

Record the EFI System Partition, APFS container, existing volumes, and free space.

Do not modify anything until the user understands the map.

### TASK 004 — Verify security / firmware state

Check relevant macOS security settings and T2 prerequisites.

### TASK 005 — Prepare macOS

Make the required macOS changes while preserving the existing installation.

### TASK 006 — Create Linux free space

Resize the APFS container only after inspection and explicit confirmation.

### TASK 007 — Verify free space

Boot the Linux installer and independently verify that the expected free space exists.

### TASK 008 — Prepare the Linux target

Create the Linux partition(s) only in the previously confirmed free space.

### TASK 009 — Preserve the Apple EFI partition

Reuse the existing EFI System Partition where appropriate.

**Do not format the existing Apple EFI partition unless there is a specific, verified reason.**

### TASK 010 — Install Ubuntu

Use the T2-compatible Ubuntu installation media and perform the installation manually where necessary.

### TASK 011 — Verify dual boot

Confirm that macOS remains bootable and Ubuntu boots successfully.

### TASK 012 — Verify the T2 kernel

Confirm that the expected T2-compatible kernel is installed and running.

### TASK 013 — Verify networking

Test Wi-Fi and the T2-specific networking required by relevant hardware.

### TASK 014 — Verify audio

Test internal speakers, internal microphone, external audio, Bluetooth audio, and external-display audio.

### TASK 015 — Verify Bluetooth

Test pairing and actual audio/microphone use rather than merely checking that the Bluetooth adapter appears.

### TASK 016 — Verify camera and external display

Test the internal camera, external displays, hotplug, and external-display audio where applicable.

### TASK 017 — Verify Touch Bar and keyboard

Test Touch Bar controls, keyboard backlight, brightness, media controls, and Escape behavior.

### TASK 018 — Configure Touch ID

Install and configure the supported T2 Touch ID stack.

Verify the T2 NCM interface and IPv6 link-local address when required.

### TASK 019 — Bind the fingerprint to the Linux user

Verify that `fprintd-list` reports the expected enrolled fingerprint.

### TASK 020 — Verify Touch ID

Test Touch ID in the GNOME environment.

### TASK 021 — Verify sudo fingerprint authentication

Test:

```bash
sudo -k
sudo -v
```

Confirm that fingerprint authentication works without weakening the sudo/PAM configuration.

### TASK 022 — Verify GNOME lock-screen fingerprint

Lock the session and verify fingerprint unlock.

### TASK 023 — Verify GNOME keyring behavior

Test both password login and fingerprint login.

Document whether the login method provides the password needed to unlock the GNOME keyring.

Do not weaken keyring security merely to remove a prompt.

### TASK 024 — Verify power management

Test suspend/resume separately from normal operation.

**Important:** automatic suspend must remain disabled on the tested configuration. It was temporarily enabled during QA specifically to reproduce the suspend-related Touch ID failure.

### TASK 025 — Verify normal applications

Test representative workflows such as Chrome, WebRTC/Google Meet, passkeys, external microphones, and external displays.

## STOP conditions

Stop and return control to the user if:

- the target disk cannot be identified confidently;
- partition boundaries differ from the expected map;
- a destructive command would affect existing macOS data;
- verification fails;
- a proposed fix would weaken security;
- the agent cannot distinguish a tested fact from an assumption;
- an upstream procedure conflicts with the actual machine state.

## Evidence discipline

For every important result, record:

- what was tested;
- the machine / configuration;
- the exact observed behavior;
- whether the result was reproducible;
- the mitigation, if any;
- whether the behavior is upstream-supported or merely observed here.

Never turn an untested assumption into installation instructions.

## Current known operational rule

After reboot:

> **Use the password once.**

This unlocks the GNOME keyring. Touch ID can then be used for sudo and lock-screen authentication.

The attempt to make GDM itself reject fingerprint login is intentionally deferred until it can be implemented and verified without disturbing the working Touch ID stack.
