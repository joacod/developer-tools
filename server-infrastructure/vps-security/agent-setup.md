# Agent-assisted VPS setup

Use this checklist for a fresh Ubuntu VPS or an OS reinstall. The [human setup guide](./README.md) supplies the baseline commands; [SSH over Tailscale](../tailscale/ssh-over-tailscale-for-vps.md) supplies the private-network migration, and [VPS access controls](../tailscale/vps-access-controls.md) supplies the dashboard policy checkpoint.

This is an operational checklist, not an unattended installer. Execute each phase only after its preceding checks pass.

## Inputs and boundaries

Resolve these from the user's request, existing local SSH configuration, and trusted provider console. Ask only for missing decisions:

- Target VPS and whether it is fresh or hosts existing workloads.
- Intended admin username and local alias, creating them if this is a first setup.
- Whether the user wants to create a dedicated SSH key or reuse one, and its local path.
- Whether a tailnet already exists, which account/network to use, and the server's actual name/address after enrollment.
- Whether an authenticated Tailscale admin browser session and an authorized browser or computer-use tool are available for dashboard changes.
- Intended public services and an independent recovery path.
- Whether sudo requires a password or passwordless sudo is explicitly wanted.

Keep actual usernames, addresses, hostnames, emails, fingerprints, keys, authentication links, and console output out of repository changes. Examples use placeholders. Never read or transmit the private SSH key. Never overwrite an existing key or replace the whole local SSH configuration.

Follow the execution environment's approval rules for access changes, credentials, device removal, and reboots. Do not infer permission to delete devices, change tags, broaden tailnet access, disable expiry, or reboot a working application from a documentation task.

## 1. Inspect and preserve recovery

Read-only checks on the VPS:

```sh
whoami
hostname
cat /etc/os-release
ip -brief address
ss -tulnp
ufw status verbose
getent passwd youruser
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

Run privileged inspection from the trusted root console or with sudo. On the local machine, check whether an SSH configuration, intended alias, and dedicated key already exist. Inspect only the relevant config block and public-key metadata when present; a first setup may have none of them. Back up existing files before changing them, including SSH snippets, sudoers snippets, firewall configuration, and local `config`/`known_hosts`.

Keep the initial session open. Verify an independent provider recovery console or rescue procedure; a browser terminal that depends on public SSH/root login is not independent recovery.

## 2. Create or restore the admin user and SSH key

Use the [user steps](./README.md#add-user) to create the intended non-root account if it does not exist. Verify an existing account before modifying its groups or access.

Before handling keys, ask: "Do you already have a dedicated SSH key you want to use for this VPS, or should we create one?" If the user has already answered, keep that decision. Resolve the chosen path and, for a new key, the passphrase preference before generation. Recommend a passphrase and let the user enter it securely; do not collect it in chat or put it into commands.

- **New key:** follow [Generate SSH key](./README.md#generate-ssh-key) on the local machine. Choose an unused path, stop if either key file already exists, and explain which file is private and which is public.
- **Existing key:** verify the user-selected path and `.pub` fingerprint. Do not infer permission to use a key merely because it exists, or request the private-key contents. If the public file is missing, resolve that before proceeding; do not generate over the existing private key.

Copy the chosen public key to the intended VPS account. Preserve unrelated authorized keys on an existing system. Set `.ssh` to `700`, `authorized_keys` to `600`, and both to the account's ownership. Add the local SSH alias if absent, or update only its block if present.

Default to the human guide's password-protected sudo account. Let the user enter their password securely. Passwordless sudo means compromise of that account's SSH access permits root access without another password, and in an agent-driven workflow it removes the pause before a destructive command. Do not silently choose this trade-off.

A locked password plus sudo-group membership alone will not make sudo usable: an explicit, validated sudoers rule is needed. A `NOPASSWD` rule is therefore a deliberate choice, not a way to paper over a locked account. If `passwd -S youruser` reports `L`, set a password before removing any `NOPASSWD` rule:

1. Keep a root session or provider console available.
2. Set the password first: `sudo passwd youruser`. The user types it; never handle or echo it.
3. Find and remove the `NOPASSWD` drop-in only after the password is set: `sudo grep -rl NOPASSWD /etc/sudoers /etc/sudoers.d/`.
4. Validate with `sudo visudo -cf /etc/sudoers`.
5. In an open session, run `sudo -k`, then `sudo -v` and `sudo id`. It must prompt and accept.
6. Confirm `sudo -l` shows no `NOPASSWD` for the account.

If a temporary `NOPASSWD` rule was needed to bootstrap, remove it in the same session after the password works. Do not leave it behind.

Verify sudoers changes with `visudo -cf` before relying on them.

## 3. Verify the SSH host key

For a first setup, verify the server fingerprint before trusting it. An OS reinstall normally changes the server host key, so verify the replacement again. Obtain its fingerprint through the trusted provider console and compare that with the key presented over SSH before accepting it.

`ssh-keyscan` can collect a candidate public host key, but cannot establish its authenticity. Do not disable host-key checking or blindly remove a warning. After an independent match, back up `known_hosts` and replace only the relevant stale entries. Keyscan output can include comment/banner lines; parse key records rather than assuming the first line is a key.

Open a separate local connection with the intended key and account. Verify `id` and working sudo in a session you keep open: `sudo -k`, then `sudo -v` and `sudo id`. For an intentionally passwordless account, `sudo -n id` is enough. A cached timestamp does not prove the password works. Continue only when both work.

## 4. Harden OpenSSH and maintain packages

Follow [SSH hardening](./README.md#harden-ssh). Check snippet precedence, run `sshd -t`, verify effective values using `sshd -T`, reload SSH, and open another new connection. Add an `AllowUsers` restriction only after identifying every legitimate SSH account.

Install UFW, unattended upgrades, and Fail2ban from Ubuntu's repositories. Use the journal-backed SSH jail on systems without an auth log. Verify periodic APT settings and timers. Inspect available upgrades before applying them; allow an active unattended upgrade to finish instead of deleting package locks or interrupting dpkg.

Record any reboot requirement. Reboot a fresh, unused VPS when authorized by the setup scope; agree a maintenance window for existing applications.

## 5. Enroll and test Tailscale before closing public SSH

Install Tailscale from its [official Linux instructions](https://tailscale.com/docs/install/linux) for the detected distribution. Use ordinary OpenSSH over Tailscale; do not enable Tailscale SSH as a substitute for this workflow.

Ask which Tailscale account/network to use if it is not already established. For a first setup without a tailnet, follow [First setup](../tailscale/README.md#first-setup) and let the user complete account registration. Otherwise authenticate into the intended existing tailnet. Device enrollment can require a fresh identity-provider login even when the admin console is already signed in. Do not create a duplicate account or embed authentication tokens in repository files.

Inspect `tailscale status` and `tailscale ip -4`. Reinstalls create new device identities; an expired record may reserve the old hostname and cause a suffix. Set the intended local SSH alias to the connected server's actual name or address, preserving other aliases. Independently verify the same new host key over this path.

Prove initial connectivity with a fresh `ssh myvps` and working sudo over Tailscale while public SSH still works. Keep the initial session and an independent provider recovery console available throughout the remaining steps.

### Tailscale access-controls checkpoint

This is an explicit human-in-the-loop checkpoint. The remaining configuration is dashboard-only and global to the tailnet. It cannot be completed from the VPS terminal because the policy is not stored on the VPS, tag ownership is part of the central policy, and key expiry is changed from the admin console. `tailscale status` only shows that the client is connected; it does not prove the policy is in place.

Complete these steps in order:

1. Enroll the VPS and the local machine in Tailscale.
2. Confirm initial private connectivity.
3. Complete the [VPS access controls](../tailscale/vps-access-controls.md) checkpoint:
   - Open the [Access controls](https://login.tailscale.com/admin/acls) page and define `tag:server` with `autogroup:owner` as its tag owner.
   - Open the [Machines](https://login.tailscale.com/admin/machines) page and assign `tag:server` to the VPS; leave the Mac untagged.
   - Check the VPS key-expiry setting and disable expiry if it is enabled. Keep expiry enabled on the Mac.
   - Replace the default allow-all grant with the restricted policy and save.
4. Verify a fresh OpenSSH connection from the Mac after the policy is applied.
5. Only then remove public SSH access. Follow the [firewall migration](../tailscale/ssh-over-tailscale-for-vps.md#allow-ssh-from-tailscale): permit tailnet traffic, remove every public SSH allow/limit rule for IPv4 and IPv6, and leave web ports closed unless an actual public service needs them. Preserve other legitimate workload rules.
6. Continue with the existing final verification in phase 6.

If an authenticated Tailscale admin browser session and an authorized browser or computer-use tool are available, the agent may guide or perform the dashboard configuration with explicit user approval. Otherwise, stop at this checkpoint and direct the user to [VPS access controls](../tailscale/vps-access-controls.md). Do not ask for credentials, guess at settings, or use unapproved automation.

Before continuing, ask the user to confirm that the VPS is tagged `tag:server`, its key expiry is disabled, and the restrictive policy is saved. Never assume the dashboard changes succeeded because `tailscale status` or `tailscale ping` works. Do not remove public SSH access until the resulting private SSH connection has been verified.

For a VPS reinstall, the replacement is a new Tailscale identity. Check its actual name or address, tag, and key-expiry setting after enrollment; do not assume the old device's configuration carries over.

Review the provider firewall too, especially before Docker publishes ports. Optional inbound Tailscale UDP permits direct connections; it does not expose SSH. Tailnet grants/ACLs, rather than UFW alone, decide which tailnet peers should reach services.

## 6. Verify persistence and clean up retired identities

Run the [sanity checks](./sanity-check.md), then verify:

- A new local `ssh myvps` uses the intended user/key and sudo works.
- Root and password authentication are rejected on the reachable private path; public IPv4 SSH fails. Test public IPv6 from an IPv6-capable client or label that check unverified.
- SSH, Tailscale, firewall, and Fail2ban are healthy; APT timers are enabled.
- `dpkg --audit` is clear and there are no unexplained failed units.
- After an approved reboot, repeat login, sudo, service, firewall, and kernel/reboot checks.

For a reinstall, only after the replacement works, match any retired tailnet entry by its old address and last-seen/OS information, then obtain the required removal confirmation. Never delete a machine solely because it is offline or has a similar name. Keep the working alias aligned with the replacement if names are later changed.

Finish with a brief report: the connection command, sudo model, public exposure, checks actually performed, backup locations, and any remaining recovery, account-MFA, tailnet-policy, provider-firewall, or backup work. Do not claim unperformed checks passed.
