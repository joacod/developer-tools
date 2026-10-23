# Tailscale VPS access controls

Restrict tailnet access so your Mac can reach the VPS, while the VPS cannot initiate connections back to the Mac. This keeps normal OpenSSH over Tailscale and uses a small, dashboard-managed policy.

| Item | Value |
| --- | --- |
| Run from | Tailscale admin dashboard |
| Prerequisites | Tailscale installed and connected on the Mac and VPS, with access to an Owner, Admin, or Network admin account |
| Outcome | Mac can reach the VPS; VPS cannot initiate tailnet connections; VPS uses `tag:server` with key expiry disabled |

## What this configures

For the standard personal Mac + VPS setup:

- The Mac can initiate connections to the VPS.
- The VPS cannot initiate connections to the Mac over the tailnet.
- The VPS uses `tag:server` and has key expiry disabled.
- The Mac remains untagged with normal key expiry.
- Access controls are configured centrally in the admin dashboard, not on the VPS.
- Standard OpenSSH remains the login path; this does not enable Tailscale SSH.

This is a global tailnet policy, not a per-device configuration file, and nothing is stored on the VPS. The VPS terminal can show client status, but the tailnet policy, tag ownership, and key-expiry setting are managed in the admin dashboard, not from the server. `tailscale status` working does not prove this policy is in place.

Apply and verify this before removing public SSH access. See [SSH over Tailscale for VPS](./ssh-over-tailscale-for-vps.md) for the firewall migration that follows.

## Setup

1. Open the [Access controls](https://login.tailscale.com/admin/acls) page.
2. Define `tag:server` with `autogroup:owner` as its tag owner:

   ```json
   "tagOwners": {
     "tag:server": ["autogroup:owner"]
   }
   ```

   For a fresh tailnet, the complete policy in step 7 also includes this definition.

3. Save the policy with this `tagOwners` addition while preserving the existing access policy. Keep the current allow-all grant in place for now; the VPS is not tagged yet, so the restricted grant would not match it.
4. Open the [Machines](https://login.tailscale.com/admin/machines) page.
5. Assign `tag:server` to the VPS. Open the VPS row's menu, choose **Edit tags**, add `tag:server`, and save. Leave the Mac untagged. Tagging gives the VPS a service identity instead of a user identity.
6. Check the VPS key-expiry setting on the same row. If the menu shows **Disable key expiry**, select it. If it shows **Enable key expiry**, expiry is already disabled. Keep expiry enabled on the Mac. Tagging an already-authenticated machine does not automatically change its existing key-expiry setting.
7. Only after the VPS is tagged and its key-expiry setting is confirmed, replace the default allow-all rule (the starter grant, or the equivalent `acls` rule in an older tailnet) with a restricted policy. For a fresh, two-device personal tailnet, use the complete policy below:

   ```json
   {
     "tagOwners": {
       "tag:server": ["autogroup:owner"]
     },
     "grants": [
       {
         "src": ["autogroup:owner"],
         "dst": ["tag:server"],
         "ip": ["*"]
       }
     ]
   }
   ```

   For an existing tailnet, preserve unrelated legitimate rules and merge the `tagOwners` and `grants` entries instead of replacing the entire policy.

   This example intentionally omits Tailscale SSH rules so the setup keeps using standard OpenSSH.

8. Save. Tailscale applies policy changes automatically; no VPS reboot or service restart is necessary. From the Mac, open a new OpenSSH connection:

   ```sh
   ssh vps-personal
   ```

   Keep public SSH and an independent recovery console available until this new connection succeeds. An existing session, `tailscale status`, or `tailscale ping` does not prove the policy is active.

## What the policy does

- `tagOwners` designates the tailnet owner as the tag owner for `tag:server`.
- `grants` allows devices owned by the tailnet owner to connect to devices tagged `tag:server`. Because `autogroup:owner` selects all owner devices, adding another device you own would also authorize it. This is not an immutable Mac-only rule.
- `ip: ["*"]` allows all protocols and ports on the server, not just SSH. That keeps Serve-based services such as the Dokploy panel reachable. Narrow it to specific ports only after confirming every service you need; a `tcp:22`-only rule would also block those Serve-based services.
- Access is denied by default. No rule allows the VPS to initiate connections to the Mac or any other tailnet device. Adding a rule in the other direction would change that property.
- Tailscale grants control network reachability only. OpenSSH still decides which Linux user and key can log in.
- Tailscale policy does not replace the VPS firewall or the provider firewall. Keep both restrictive.

## Verify

1. Confirm the VPS row in **Machines** shows `tag:server`.
2. Confirm the VPS menu offers **Enable key expiry** (meaning expiry is currently disabled) and the Mac menu offers **Disable key expiry** (meaning normal expiry is enabled).
3. From the Mac, close the old session and open a new one with `ssh vps-personal`. Confirm the intended user, key, and sudo work.
4. Optionally, from the VPS, try to reach a known listening service on the Mac over the tailnet; it should fail under this policy. If the Mac has no listener, rely on the policy's deny-by-default direction rather than a connection test. Do not treat `tailscale ping` or `tailscale status` as proof of application access, because they can succeed even when grants block a connection.

If the new connection fails, use the admin dashboard or an independent provider console to restore the previous policy. Do not remove public SSH rules until the private path is verified.

## Official references

- [Grants](https://tailscale.com/docs/features/access-control/grants)
- [Tags](https://tailscale.com/docs/features/tags)
- [Key expiry](https://tailscale.com/docs/features/access-control/key-expiry)
- [Tailnet policy file syntax](https://tailscale.com/docs/reference/syntax/policy-file)
