# Server maintenance

Routine OS maintenance for an Ubuntu VPS, focused on stability, security, and a clean disk.

| Item | Value |
| --- | --- |
| OS | Ubuntu (supported release) |
| Run from | VPS |
| Prerequisites | `sudo` access as a non-root user |
| Outcome | Updated packages, a deliberate reboot decision, and verified security controls |

> **Run as:** `youruser` on the VPS. Commands use `sudo`; keep one session open in case a reboot is needed.

## Scope

This covers the operating system only: packages, reboot state, disk and log hygiene, and a quick check that the security baseline from the [setup guide](./README.md) is still in place.

It does not cover application or container updates (Docker images, deployed services, app dependencies), release upgrades, or backup verification. Handle those separately.

Run this every 2 to 4 weeks, or monthly when `unattended-upgrades` is enabled. Automatic updates handle security patches between runs; this pass exists for kernels, non-security packages, and the reboot decision.

Applying this with an agent? Use the [agent-assisted checklist](./agent-maintenance.md).

## 1. See what will change

```sh
sudo apt update
apt list --upgradable
```

- Review the list before upgrading. This is exactly what `apt upgrade` would install.
- If `apt update` prints lines starting with `W:` or `E:`, fix the repository or network problem first. Do not upgrade against a broken package index.
- Packages reported as **kept back** are usually phased updates. Leave them; do not force them with `full-upgrade`.

Check whether a reboot is already pending:

```sh
[ -f /var/run/reboot-required ] && cat /var/run/reboot-required.pkgs 2>/dev/null
```

## 2. Apply updates

```sh
sudo apt upgrade
```

- Do not add `-y`. Read the list, then confirm.
- `apt upgrade` applies security and regular updates together.
- If asked about a configuration file, keep your current version unless you know the package needs the new default. When unsure, stop and inspect the file.
- Accept service restarts so they pick up the updated libraries.
- Do not use `full-upgrade` or `dist-upgrade` for routine maintenance.

After the upgrade, check whether services still run against old libraries:

```sh
sudo needrestart -b
```

If it lists services, restart them or reboot during a maintenance window. Ubuntu Server usually installs `needrestart`; skip this check if it is missing.

## 3. Clean up

Remove packages that are no longer needed:

```sh
sudo apt autoremove
```

Review the list first. Decline if it would remove a kernel you still boot, a package tied to a running service, or anything you do not recognize.

Clear the package cache and check what is using disk:

```sh
sudo apt clean
df -h /
df -i /
sudo journalctl --disk-usage
```

- `df -i` catches inode exhaustion, which `df -h` does not.
- If the journal is large, trim old entries with `sudo journalctl --vacuum-time=2weeks`. This deletes logs, so keep at least the current maintenance window.
- If Docker is installed, inspect `sudo docker system df`. Do not prune images or volumes as routine cleanup; rollbacks and active data may depend on them.

## 4. Reboot when required

A kernel or low-level library update does nothing until the server restarts. Check what needs it:

```sh
cat /var/run/reboot-required.pkgs
```

Then reboot deliberately:

```sh
sudo reboot
```

- Reboot during a maintenance window, not during a deploy.
- The SSH session drops. Wait, then reconnect. If SSH is reachable only over a private network such as Tailscale, that service must come back before you can log in.
- After the reboot, repeat the checks in step 5.

## 5. Verify security and health

```sh
sudo ufw status verbose
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication|kbdinteractiveauthentication"
systemctl is-active unattended-upgrades
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer --no-pager
sudo fail2ban-client status sshd
tailscale status
sudo dpkg --audit
sudo systemctl --failed
sudo ss -tulnp
df -h
free -h
```

Skip `fail2ban` and `tailscale` when they are not installed.

What good looks like:

- firewall active with default deny incoming and no unexpected public ports
- SSH still reports `permitrootlogin no`, `passwordauthentication no`, and `kbdinteractiveauthentication no`
- `unattended-upgrades` is `active`, and both APT timers exist
- the Fail2ban `sshd` jail is active when Fail2ban is installed
- `dpkg --audit` prints nothing, and `systemctl --failed` lists no units
- disk and memory have headroom

If something looks wrong, read recent high-priority boot messages:

```sh
sudo journalctl -p 3 -xb
```

## Stay on a supported release

```sh
grep PRETTY_NAME /etc/os-release
```

Check the output against Ubuntu's release cycle. When standard support ends, plan an LTS-to-LTS upgrade as its own project, with backups and a maintenance window. Do not treat it as routine maintenance.

## Avoid

- unnecessary PPAs and third-party repositories unless they are truly needed and trustworthy
- packages from Ubuntu Backports unless you explicitly need a newer version
- `full-upgrade`, `dist-upgrade`, or release upgrades as part of normal upkeep
- `-y` on `upgrade` or `autoremove` before reviewing the list
- disabling `unattended-upgrades` or deleting `/var/lib/dpkg/lock*` to get past a lock; let the running operation finish
- `ufw reset` on a live server
- pruning Docker images or volumes as routine cleanup
- running an unsupported Ubuntu release

## Summary

- review upgrades, then apply them without `-y`
- check `needrestart` and restart services that still use old libraries
- clean packages, logs, and disk space deliberately
- reboot when required, then verify login, services, and firewall
- keep automatic security updates enabled and stay on a supported release
