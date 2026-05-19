---
description: 'Linux distro guides: Arch, CentOS, Debian, Fedora'
applyTo: '**'
---


# Arch Linux Administration Guidelines

Use these instructions when writing guidance, scripts, or documentation for Arch Linux systems.

## Platform Alignment

- Emphasize the rolling-release model and the need for full upgrades.
- Confirm current kernel and recent package changes when troubleshooting.
- Prefer official repositories and the Arch Wiki for authoritative guidance.

## Package Management

- Use `pacman -Syu` for full system upgrades; avoid partial upgrades.
- Inspect packages with `pacman -Qi`, `pacman -Ql`, and `pacman -Ss`.
- Mention AUR helpers only with explicit cautions and PKGBUILD review reminders.

## Configuration & Services

- Keep configuration under `/etc` and avoid editing files in `/usr`.
- Use systemd drop-ins in `/etc/systemd/system/<unit>.d/`.
- Use `systemctl` and `journalctl` for service control and logs.

## Security

- Note reboot requirements after kernel or core library upgrades.
- Recommend least-privilege `sudo` usage and minimal packages.
- Call out firewall tooling expectations (nftables/ufw) explicitly.

## Deliverables

- Provide commands in copy-paste-ready blocks.
- Include validation steps after changes.
- Offer rollback or cleanup steps for risky operations.

---


# CentOS Administration Guidelines

Use these instructions when producing guidance, scripts, or documentation for CentOS environments.

## Platform Alignment

- Identify CentOS version (Stream vs. legacy) and tailor commands.
- Prefer `dnf` for Stream/8+ and `yum` for CentOS 7.
- Use RHEL-compatible terminology and paths.

## Package Management

- Verify repositories with GPG checks enabled.
- Use `dnf info`/`yum info` and `dnf repoquery` for package details.
- Use `dnf versionlock` or `yum versionlock` for stability where needed.
- Call out EPEL dependencies and how to enable/disable them safely.

## Configuration & Services

- Place service environment files in `/etc/sysconfig/` when required.
- Use systemd drop-ins for overrides and `systemctl` for control.
- Prefer `firewalld` (`firewall-cmd`) unless explicitly using `iptables`/`nftables`.

## Security

- Keep SELinux in enforcing mode whenever possible.
- Use `semanage`, `restorecon`, and `setsebool` for policy adjustments.
- Reference `/var/log/audit/audit.log` for denials.

## Deliverables

- Provide commands in copy-paste-ready blocks.
- Include verification steps after changes.
- Offer rollback steps for risky operations.

---


# Debian Linux Administration Guidelines

Use these instructions when writing guidance, scripts, or documentation intended for Debian-based systems.

## Platform Alignment

- Favor Debian Stable defaults and long-term support expectations.
- Call out the Debian release (`bookworm`, `bullseye`, etc.) when relevant.
- Prefer official Debian repositories before suggesting third-party sources.

## Package Management

- Use `apt` for interactive commands and `apt-get` for scripts.
- Inspect packages with `apt-cache policy`, `apt show`, and `dpkg -l`.
- Use `apt-mark` to track manual vs. auto-installed packages.
- Document any apt pinning in `/etc/apt/preferences.d/` and explain why.

## Configuration & Services

- Store configuration under `/etc` and avoid modifying `/usr` files directly.
- Use systemd drop-ins in `/etc/systemd/system/<unit>.d/` for overrides.
- Prefer `systemctl` and `journalctl` for service control and logs.
- Use `ufw` or `nftables` for firewall guidance; state which is expected.

## Security

- Account for AppArmor profiles and mention adjustments if needed.
- Recommend least-privilege `sudo` use and minimal package installs.
- Include verification commands after security changes.

## Deliverables

- Provide commands in copy-paste-ready blocks.
- Include validation steps after changes.
- Offer rollback steps for destructive actions.

---


# Fedora Administration Guidelines

Use these instructions when writing guidance, scripts, or documentation for Fedora systems.

## Platform Alignment

- State the Fedora release number when relevant.
- Prefer modern tooling (`dnf`, `systemctl`, `firewall-cmd`).
- Note the fast release cadence and confirm compatibility for older guidance.

## Package Management

- Use `dnf` for installs and updates, and `dnf history` for rollback.
- Inspect packages with `dnf info` and `rpm -qi`.
- Mention COPR repositories only with clear support caveats.

## Configuration & Services

- Use systemd drop-ins in `/etc/systemd/system/<unit>.d/`.
- Use `journalctl` for logs and `systemctl status` for service health.
- Prefer `firewalld` unless using `nftables` explicitly.

## Security

- Keep SELinux enforcing unless the user requests permissive mode.
- Use `semanage`, `setsebool`, and `restorecon` for policy changes.
- Recommend targeted fixes instead of broad `audit2allow` rules.

## Deliverables

- Provide commands in copy-paste-ready blocks.
- Include verification steps after changes.
- Offer rollback steps for risky operations.