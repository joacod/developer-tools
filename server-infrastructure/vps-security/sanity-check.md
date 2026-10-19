# VPS sanity check

Final verification steps for the Ubuntu VPS setup.

## Check SSH config is active

> **Run as:** `youruser` on the VPS

Verify the active SSH settings.

```sh
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication|kbdinteractiveauthentication"
```

Expected output:

- `permitrootlogin no`
- `passwordauthentication no`
- `kbdinteractiveauthentication no`

## Check SSH is running

> **Run as:** `youruser` on the VPS

Verify the SSH service is running.

```sh
sudo systemctl status ssh
```

Expected result:

- `active (running)`

## Check firewall status

> **Run as:** `youruser` on the VPS

Verify `ufw` is active and using the expected defaults.

```sh
sudo ufw status verbose
```

Expected output:

- `Status: active`
- Default incoming policy set to deny
- Default outgoing policy set to allow
- Before Tailscale migration: a public SSH rule (`OpenSSH`) is present
- After migration: `tailscale0` is allowed and no public SSH allow/limit rule remains
- Rules include `80/tcp` and `443/tcp` if you enabled HTTP and HTTPS

## Check open ports

> **Run as:** `youruser` on the VPS

Review listening ports.

```sh
sudo ss -tulnp
```

What to check:

- Port `22` for SSH
- Ports `80` and `443` only if you enabled them
- No unexpected listening ports

## Check unattended upgrades

> **Run as:** `youruser` on the VPS

Verify unattended upgrades are running.

```sh
sudo systemctl status unattended-upgrades
```

Expected result:

- `active (running)`

## Check your user permissions

> **Run as:** `youruser` on the VPS

Verify the current user is in the `sudo` group.

```sh
id
```

Expected output:

- Includes `sudo`

## Check the sudo model

> **Run as:** `youruser` on the VPS

Confirm the account has a usable password and that passwordless sudo is not accidental.

```sh
passwd -S youruser
sudo -v && { sudo -l | grep -i nopasswd || echo "no NOPASSWD rules"; }
```

`sudo -v` authenticates first, so a failed login cannot be mistaken for "no rules". It may prompt for the password, which is expected.

Expected output:

- `passwd -S` reports `P` for a usable password. `L` means locked, which only works when passwordless sudo is deliberate.
- `sudo -l` prints `no NOPASSWD rules`, or only rules you intentionally configured for a specific account.

If the output differs, stop and review `/etc/sudoers` and `/etc/sudoers.d/`. A locked password plus a leftover `NOPASSWD` rule is the common cause; the [agent setup checklist](./agent-setup.md) has the switch procedure.

## Check `.ssh` permissions

> **Run as:** `youruser` on the VPS

Verify the `.ssh` directory and `authorized_keys` file permissions.

```sh
ls -ld ~/.ssh
ls -l ~/.ssh
```

Expected output:

- `.ssh` shows `drwx------` (`700`)
- `authorized_keys` shows `-rw-------` (`600`)

## Check root login is blocked

> **Run as:** your user on your `local machine`

Verify direct SSH login as `root` is blocked. Use the reachable Tailscale hostname after migration, or the public address during bootstrap. Confirm the non-root login works first; a network timeout does not prove root authentication is disabled.

```sh
ssh -o BatchMode=yes -o IdentitiesOnly=yes -i ~/.ssh/vps_server root@YOUR_VPS_TAILSCALE_HOST
```

Expected result:

- `Permission denied (publickey)`

## Check password login is blocked

> **Run as:** your user on your `local machine`

Verify password authentication is blocked.

```sh
ssh -o PreferredAuthentications=password -o PubkeyAuthentication=no -o NumberOfPasswordPrompts=1 myvps
```

Expected result:

- `Permission denied` without a password prompt

If a password prompt appears, cancel it and inspect the effective server configuration. Do not use `BatchMode=yes` as proof that passwords are disabled: it prevents the client from asking for a password even when the server permits one.

## Check updates and persistence

> **Run as:** `youruser` on the VPS

```sh
systemctl list-timers apt-daily.timer apt-daily-upgrade.timer
sudo apt-config dump | grep -E "APT::Periodic::(Update-Package-Lists|Unattended-Upgrade)"
sudo dpkg --audit
sudo systemctl --failed
```

Expect both periodic settings to be `1`, active timers, no incomplete packages, and no unexplained failed units. After an approved reboot, repeat a fresh `ssh myvps`, verify `sudo`, and check SSH, Tailscale, the firewall, and Fail2ban again.

## What this verifies

If all checks pass, you have verified:

- SSH hardening is active
- Root login is disabled
- Password login is disabled
- The firewall is active
- Automatic security updates are enabled
- `youruser` has the expected permissions
- The sudo model is deliberate: password-protected, or explicitly passwordless
- Only expected ports are open

## Final verdict

These checks establish the configured baseline. Account security, tailnet access policy, provider firewall rules, and backup recovery still need separate review.
