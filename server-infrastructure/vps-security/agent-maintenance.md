# Agent-assisted maintenance

Use this checklist for routine maintenance on an Ubuntu VPS that already completed the [baseline setup](./README.md). The [human maintenance guide](./server-maintenance.md) explains the same work; this file adds ordering, approval gates, and stop conditions for an agent.

This is an operational checklist, not an unattended script. Work one phase at a time, show the evidence, and stop at every gate.

## Inputs and boundaries

Resolve from the request, the VPS itself, and the user's existing SSH configuration. Ask only for missing decisions:

- SSH alias or target, and the non-root user with `sudo`
- Whether the VPS hosts live workloads, and whether a reboot is authorized
- Whether Docker, Fail2ban, or Tailscale are installed (discover them; do not assume)
- Whether a deploy or other package operation is in progress

Rules:

- Run on the VPS as the sudo user. Do not run maintenance from a local clone.
- Keep hostnames, addresses, usernames, and command output with private data out of repository changes.
- Do not install, remove, or reconfigure packages beyond this routine.
- Never run `apt full-upgrade`, `dist-upgrade`, `do-release-upgrade`, `ufw reset`, or edit `sshd`, `sudoers`, or firewall rules.
- Never reboot without explicit approval.
- Treat any `E:` line from `apt` or `dpkg` as a stop. Report `W:` lines without acting on them.
- If a check fails, report it. Do not widen scope to fix it.

## Sudo protocol

This is the part that must not be improvised.

Before every privileged phase:

```sh
sudo -n true 2>/dev/null && echo SUDO_READY || echo SUDO_BLOCKED
```

- **`SUDO_READY`:** continue.
- **`SUDO_BLOCKED`:** stop the run. Tell the user: "sudo needs your password. Run `sudo -v` in this same terminal, then tell me to continue." Wait for their confirmation, then re-check once. If it is still blocked, stop and hand back.
- Never ask for, accept, echo, store, or transmit the password. Do not read it from history, environment variables, or files.
- Run privileged commands in the same session or pane the user authenticated in. Sudo timestamps are per terminal; a separate `ssh host 'command'` channel stays blocked even after `sudo -v` elsewhere.
- Re-check before each phase. The default timestamp expires after about 15 minutes, and a long upgrade can outlast it.
- If output stalls at a `[sudo] password for` prompt, stop and hand control back. Do not send input.

Never use `sudo -S`, `SUDO_ASKPASS`, `echo password | sudo ...`, or a `NOPASSWD` sudoers rule. Do not retry a blocked privileged command in a loop.

## 1. Preflight (read-only)

```sh
whoami
hostname
grep PRETTY_NAME /etc/os-release
date -Is
sudo -n true 2>/dev/null && echo SUDO_READY || echo SUDO_BLOCKED
sudo dpkg --audit
systemctl is-active apt-daily-upgrade.service unattended-upgrades.service
df -h /
df -i /
free -h
```

Stop and report when:

- `dpkg --audit` prints anything
- `apt-daily-upgrade` or `unattended-upgrades` is `activating` (wait for it to finish)
- free space on `/` is under about 1 GiB

`apt-daily-upgrade.service` is a oneshot unit: `inactive` between runs is normal. Only `activating` means a run is in progress.

## 2. Inventory (no package changes)

Refresh the package index and list changes:

```sh
sudo apt update
apt list --upgradable
[ -f /var/run/reboot-required ] && cat /var/run/reboot-required.pkgs 2>/dev/null
```

If `apt update` prints `W:` or `E:`, stop and report the repository error. Do not upgrade against a broken index.

Capture the current security and health state:

```sh
sudo ufw status verbose
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication|kbdinteractiveauthentication"
systemctl is-active unattended-upgrades
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer --no-pager
if command -v fail2ban-client >/dev/null; then sudo fail2ban-client status sshd; fi
if command -v tailscale >/dev/null; then tailscale status; fi
sudo systemctl --failed
sudo journalctl -p 3 -xb --no-pager | tail -n 20
sudo journalctl --disk-usage
if command -v docker >/dev/null; then sudo docker system df; fi
```

`command -v` guards keep a missing optional tool from hiding a real error when the tool is installed but broken.

Empty `apt list --upgradable` output means nothing to upgrade. No `/var/run/reboot-required` means no reboot is pending.

Also preview cleanup and restarts:

```sh
sudo apt-get -s autoremove
sudo needrestart -b 2>/dev/null || true
```

Skip `needrestart` when the command is missing. In its output, `NEEDRESTART-KSTA: 1` means the running kernel matches the installed one, `2` or `3` means a reboot is needed to run the new kernel, and any `NEEDRESTART-SVC:` lines name services still using old libraries.

Summarize for the user: package count, which updates look security-relevant, reboot status and affected packages, autoremove count, services needing a restart, disk and journal usage, and whether the firewall, SSH settings, automatic updates, Fail2ban, and Tailscale are as expected.

## 3. Approval gate

If the inventory is empty — no packages, no pending reboot, nothing to autoremove, and no services flagged by `needrestart` — report that and skip to phase 7. Do not ask for approval to change nothing.

Otherwise, report the inventory and ask for one explicit approval to apply. State:

- how many packages will change and whether a kernel is included
- whether a reboot will probably be required
- what `autoremove` would remove
- any anomaly from phase 2

Do not proceed on silence. If the user asked only for a check, stop here.

## 4. Apply

Run interactively. Do not wrap `apt` in command substitution or a non-TTY pipeline; prompts need a terminal.

```sh
sudo env NEEDRESTART_MODE=a apt upgrade
```

- Never add `-y`. Confirm the final summary prompt only after the phase 3 approval.
- `NEEDRESTART_MODE=a` restarts services automatically so they pick up updated libraries. Use `NEEDRESTART_MODE=l` if the user wants the list first, then restart the listed services with approval.
- If asked about a configuration file, stop, report the file, and wait for a decision. Do not auto-answer or add `--force-confold`.
- If packages are reported as **kept back**, report them. Do not switch to `full-upgrade`.

Then remove unused packages:

```sh
sudo apt autoremove
```

Show the simulated list from phase 2 first. Stop if it includes a running kernel, a package tied to a workload, or anything the user has not approved. Otherwise review the list and continue without `-y`.

Clear the package cache:

```sh
sudo apt clean
```

Trim the journal only when `journalctl --disk-usage` is large, and only with approval, because it deletes logs:

```sh
sudo journalctl --vacuum-time=2weeks
```

If Docker is installed, report `sudo docker system df`. Do not prune images, containers, or volumes.

## 5. Post-checks

If phase 4 changed anything, re-run the phase 2 security and health commands, plus:

```sh
sudo dpkg --audit
sudo needrestart -b
```

Every previous result must still hold. If nothing was applied, the phase 2 results are the post-check; do not repeat the full inventory just to produce a second identical table. If `needrestart` lists services, report them and restart them only with approval, or let the reboot handle it.

## 6. Reboot gate

If `/var/run/reboot-required` exists, stop and ask for approval. On approval:

```sh
sudo reboot
```

The command may not return before the session drops. That is expected.

After issuing the reboot, stop the run completely. Do not poll, retry, sleep, or run reconnection loops from any session. Tell the user the host is rebooting, that the connection will drop, and to reply when the connection is back.

When the user confirms reconnection:

- re-run phase 2 on the same session
- run the [sanity check](./sanity-check.md) for the login, firewall, and service checks the environment allows, and label anything you could not test as unverified

If the user reports the host did not come back, stop and point them to the provider console. Do not reboot again.

## 7. Report

Finish with a short report:

- host, user, Ubuntu version, start and end time
- packages upgraded, including kernel and security-relevant ones
- `autoremove` count and `apt clean` result
- reboot required before and after, reboot performed or deferred
- `needrestart` findings and services restarted
- before/after table for firewall, SSH settings, `unattended-upgrades`, APT timers, Fail2ban, Tailscale, failed units, disk, and journal size (current state alone when nothing changed)
- anything that failed, drifted, or needs a follow-up, including application or container updates that were out of scope

Do not claim a check passed if it was skipped or blocked.

## Stop conditions

Stop and hand back when:

- sudo is blocked and the user has not confirmed
- a package lock is held or `dpkg` reports an incomplete state
- `apt update` shows repository errors, or upgrades are held back beyond normal phasing
- `autoremove` would remove something unexpected
- a conffile prompt appears
- a post-check regresses: failed unit, changed firewall or SSH setting, service down
- a reboot was issued and the user has not confirmed the connection is back
- the user reports the host did not come back after a reboot
- the work would need a configuration change, package installation, or removal outside this routine

## Out of scope

Application and container updates, release upgrades, backup verification, account changes, and firewall or tailnet policy changes. Point the user to the matching guide instead of doing them here.
