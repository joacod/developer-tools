# SSH over Tailscale for VPS

Recommended follow-up guide to move VPS SSH access from the public internet to your Tailscale network while keeping normal OpenSSH users, keys, and permissions.

| Item | Value |
| --- | --- |
| Run from | Local machine and VPS |
| Prerequisites | Completed [`VPS security`](../vps-security/README.md) setup, Tailscale installed and connected on both machines, and the [restricted tailnet policy](./vps-access-controls.md) ready to apply |
| Outcome | SSH works only over Tailscale while your existing OpenSSH users and keys stay the same |

## What this setup does

This guide is for standard OpenSSH over the Tailscale network.

It does not use Tailscale SSH.

Use this path if you want to:

- Keep separate Linux users such as `personal` and `hermes`
- Keep separate SSH private keys for each user
- Keep your normal `sudo` and `non-sudo` user permissions
- Remove public SSH exposure from port `22`

For this repo's Dokploy and Hermes VPS flow, this is the intended SSH setup after the base hardening guide.

## Before you start

This guide assumes:

- the base hardening steps in [`VPS security`](../vps-security/README.md) are already done
- Tailscale is installed and connected on the VPS
- Tailscale is installed and connected on your local machine
- the restricted tailnet policy in [VPS access controls](./vps-access-controls.md) will be applied and verified before public SSH is removed
- your current SSH access still works before you change the firewall rules

If you still need the basic Tailscale setup, start with the [Tailscale guide](./README.md).

## Update local SSH aliases

> **Run as:** your user on your `local machine`

Replace `HostName YOUR_VPS_IP` in your local SSH aliases with the VPS `MagicDNS hostname`.

Example:

```sshconfig
Host vps-personal
  HostName my-vps
  User personal
  IdentityFile ~/.ssh/personal_vps
  IdentitiesOnly yes

Host vps-hermes
  HostName my-vps
  User hermes
  IdentityFile ~/.ssh/hermes_vps
  IdentitiesOnly yes
```

Only the `HostName` value changes for this migration. Your users and private keys stay the same.

If you do not use MagicDNS, use the VPS Tailscale IP instead.

After an OS reinstall, the VPS is a new Tailscale identity. An old entry can retain the original name, so the new machine may receive a suffix. Use the name and address shown for the connected replacement, not the expired entry. Follow the [rebuild checklist](../vps-security/agent-setup.md) to verify its new SSH host key and remove stale records safely.

## Check the current VPS state

> **Run as:** your admin user on the VPS

Verify SSH, Tailscale, and the current firewall rules before changing anything.

```sh
sudo systemctl status ssh
tailscale status
tailscale ip -4
sudo ufw status verbose
```

What to confirm before continuing:

- `sudo systemctl status ssh`: the service shows `active (running)`
- `tailscale status`: the VPS appears connected to your tailnet and does not show a disconnected state
- `tailscale ip -4`: you get a Tailscale IPv4 address you can use to sanity-check the machine identity if needed
- `sudo ufw status verbose`: `ufw` is active and you can see the current `OpenSSH` rule that will be removed later

If any of these checks look wrong, stop there and fix that problem before changing the firewall rules.

## Test SSH over Tailscale first

> **Run as:** your user on your `local machine`

Do not continue until both SSH aliases work over Tailscale.

```sh
ssh vps-personal
ssh vps-hermes
```

## Allow SSH from Tailscale

> **Run as:** your admin user on the VPS

Allow incoming traffic on the Tailscale interface.

```sh
sudo ufw allow in on tailscale0
```

This allows tailnet traffic broadly. Tailscale also installs its own packet-filter rules, so UFW is not a substitute for tailnet access controls. Review grants/ACLs to decide which users and devices may reach the server.

Optionally allow `41641/udp` in both host and provider firewalls for direct Tailscale connections. This is Tailscale transport, not public SSH; blocked inbound UDP can still use relays. Use the actual listen port if customized.

## Apply tailnet access controls first

> **Run as:** your user in the Tailscale admin dashboard

Do not remove public SSH access until the restricted tailnet policy is applied and verified.

Complete [VPS access controls](./vps-access-controls.md) first:

- assign `tag:server` to the VPS and leave your Mac untagged
- disable key expiry on the VPS and keep normal expiry on the Mac
- replace the default allow-all grant with the restricted policy
- save the policy and open a new `ssh vps-personal` connection from the Mac

Keep the current session and an independent provider recovery console available. A working `tailscale status` does not prove the dashboard changes were saved.

## Remove public SSH exposure

> **Run as:** your admin user on the VPS

Your base VPS guide opens public SSH with `OpenSSH`. This step removes those public SSH rules after you already verified the Tailscale path works.

First list the current numbered rules:

```sh
sudo ufw status numbered
```

What to look for:

- any public SSH rule, including `OpenSSH`, `22/tcp`, or a custom SSH port, with either `ALLOW` or `LIMIT`
- the rule numbers attached to those public SSH entries

You will use those rule numbers in the next step.

> **Important:** After you delete the first rule, the remaining rule numbers can change. Run `sudo ufw status numbered` again before deleting the next `OpenSSH` rule so you do not remove the wrong entry.

Delete the `OpenSSH` rules you see there.

```sh
sudo ufw delete RULE_NUMBER
```

What to do here:

- replace `<RULE_NUMBER>` with the actual numbers shown by `sudo ufw status numbered`
- if you have both IPv4 and IPv6 `OpenSSH` rules, delete both

Then verify the firewall state.

```sh
sudo ufw status verbose
```

What to confirm before moving on:

- `sudo ufw status verbose` no longer shows public `OpenSSH` rules
- `sudo ufw status verbose` shows the `tailscale0` allow rule

## Final verification

> **Run as:** your user on your `local machine`

Verify the Tailscale path works and the public IP path no longer works.

```sh
ssh vps-personal
ssh vps-hermes
ssh personal@YOUR_PUBLIC_IP
```

Expected result:

- `ssh vps-personal` works
- `ssh vps-hermes` works
- `ssh personal@YOUR_PUBLIC_IP` fails

The public-IP failure will often look like `Operation timed out` because `ufw` is dropping the incoming connection. Test public IPv6 too when the local machine has IPv6 connectivity. `No route to host` from a client without IPv6 is not evidence that the server firewall blocked the connection.

What this verifies:

- both local aliases now work over Tailscale with their existing users and keys
- SSH is no longer reachable from the public internet on the VPS public IP

## Rollback

> **Run as:** your admin user on the VPS while you still have an open SSH session

If something goes wrong, re-open public SSH access temporarily.

```sh
sudo ufw limit OpenSSH
sudo ufw reload
```

What to confirm after rollback:

- `sudo ufw reload` completes without errors
- `sudo ufw status verbose` shows the `OpenSSH` rules again
- you can reconnect using the old public SSH path if needed; the provider firewall must also permit it

This restores network reachability only. Root/password login stays disabled. If Tailscale access has already failed, use an independent provider console or rescue mode to recover; a web terminal backed by SSH may also be blocked.

## Notes

- Your SSH login model does not change: OpenSSH still decides which Linux user you become and what permissions that user has
- Fail2ban is still fine to keep installed, but it matters much less for SSH once public port `22` is no longer exposed

## Device expiry and old machines

Expiry prevents a device from authenticating; it does not remove its inventory record. After verifying the replacement, identify the retired device by its old address, last-seen time, and OS, then use **Machines > … > Remove**. Removing the old record does not wipe the VPS. Rejoining requires authentication again. Confirm the exact retired identity before removal.

Keep an independent recovery path for a server whose SSH is private. The standard setup here disables key expiry on the tagged VPS so a server reachable only over the tailnet cannot lock itself out; treat that as an explicit availability/security trade-off and keep normal expiry on the Mac. Check the VPS expiry setting after tagging and after a reinstall, because tagging an already-authenticated machine does not change it automatically. If you intentionally leave expiry enabled on a private-only server, plan reauthentication before it expires. Do not run `tailscale up --force-reauth` through the only working connection without recovery access.

## Official references

- [Protect your SSH servers using Tailscale](https://tailscale.com/docs/reference/ssh-over-tailscale)
- [Set up MagicDNS](https://tailscale.com/docs/features/magicdns)
- [Remove a device](https://tailscale.com/docs/features/access-control/device-management/how-to/remove)
- [Key expiry](https://tailscale.com/docs/features/access-control/key-expiry)
- [Using Tailscale with your firewall](https://tailscale.com/kb/1181/firewalls)
