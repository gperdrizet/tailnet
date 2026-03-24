# Manual setup steps

Some steps in the tailnet setup cannot be fully automated. This document covers
each manual step in the order it appears in the setup sequence from README.md.

## Step 1: Add a DNS A record

Before installing Headscale, add a DNS A record pointing the control server
subdomain to the VPS public IP address. This record must exist and propagate
before Let's Encrypt can issue a TLS certificate on first start.

In your DNS provider, create:

| Type | Name                       | Value            | TTL   |
|------|----------------------------|------------------|-------|
| A    | headscale.perdrizet.org    | <VPS public IP>  | 300   |

Verify propagation before proceeding:

```
dig headscale.perdrizet.org A +short
```

The command should return the VPS public IP. If it does not, wait for the TTL
to expire and try again.

---

## Step 2: Generate a pre-auth key (recommended)

Pre-auth keys let devices register with Headscale without requiring a URL
approval for each one. Generate a key on the VPS after Headscale is installed:

```
headscale preauthkeys create --user siderealyear --expiration 1h
```

Copy the printed key. Pass it to `install-tailscale-client.sh` on each device:

```
sudo bash scripts/install-tailscale-client.sh --auth-key <key>
```

The key expires after 1 hour. Generate a new one if it expires before use.
Once a device has registered, the pre-auth key is no longer needed for that
device.

---

## Step 3: Approve a device node (interactive registration)

If a device was registered without a pre-auth key, the Tailscale client prints
a URL like:

```
https://headscale.perdrizet.org/register/nodekey:XXXX...
```

On the VPS, approve the node using the node key from that URL:

```
headscale nodes register --user siderealyear --key nodekey:XXXX...
```

Verify the node was registered and is active:

```
headscale nodes list
```

The output should show the device with a recent "last seen" timestamp. Repeat
for each device.

---

## Step 4: Approve exit node routes

After running `scripts/configure-exit-node.sh` on the VPS, the exit node
advertises two routes: `0.0.0.0/0` (all IPv4 traffic) and `::/0` (all IPv6
traffic). These must be explicitly approved before client devices can use them.

List all advertised routes:

```
headscale routes list
```

The output will look similar to:

```
ID | Node       | Prefix      | Advertised | Enabled | IsPrimary
1  | gatekeeper | 0.0.0.0/0   | true       | false   | true
2  | gatekeeper | ::/0        | true       | false   | true
```

Enable each route using its ID:

```
headscale routes enable -r 1
headscale routes enable -r 2
```

Confirm both routes are now enabled:

```
headscale routes list
```

The "Enabled" column should show `true` for both rows.

After approval, run `scripts/configure-client.sh` on the desktop and laptop.

---

## Step 5: Configure the phone

The Tailscale mobile app supports custom control servers. Follow these steps:

1. Install the Tailscale app (iOS App Store or Google Play Store).

2. Before logging in, open the app settings:
   - On iOS: tap the Tailscale logo at the top of the home screen, then
     "Settings".
   - On Android: tap the three-dot menu, then "Settings".

3. Find the "Control server" or "Custom control server" option and enter:
   ```
   https://headscale.perdrizet.org
   ```

4. Tap "Log in". The app will open a browser window showing a URL like:
   ```
   https://headscale.perdrizet.org/register/nodekey:XXXX...
   ```

5. On the VPS, approve the phone:
   ```
   headscale nodes register --user siderealyear --key nodekey:XXXX...
   ```

6. The app should show the tailnet devices as peers.

7. To activate the exit node, go to the exit node settings in the app and
   select "gatekeeper" (the VPS). The phone will route all internet traffic
   through the VPS.

---

## MagicDNS hostnames

With MagicDNS enabled in the Headscale config, devices are reachable at
`<system-hostname>.perdrizet.org` in addition to their Tailscale IP.

The hostname used in MagicDNS is the system hostname of the device at the time
of registration. Verify a device's hostname with:

```
hostname
```

Expected MagicDNS hostnames for this tailnet:

| Device           | System hostname | MagicDNS hostname               |
|------------------|-----------------|---------------------------------|
| This machine     | desktop         | desktop.perdrizet.org           |
| Laptop           | laptop          | laptop.perdrizet.org            |
| Phone            | phone           | phone.perdrizet.org             |
| VPS (gatekeeper) | gatekeeper      | gatekeeper.perdrizet.org        |

If a device's system hostname does not match the expected names above, update
`/etc/hostname` and reboot before registering the device with Headscale.

---

## Troubleshooting

**Headscale does not start after install**

Inspect the service logs:
```
journalctl -u headscale -n 50
```

Common causes:
- Port 443 is already in use by another process (e.g. nginx or apache). Stop
  the conflicting service or configure it as a reverse proxy instead.
- The TLS certificate could not be issued because the DNS A record has not
  propagated or port 80 is blocked by the firewall.

**A device does not appear in `headscale nodes list`**

- Confirm `tailscale up --login-server=...` completed without errors on that
  device.
- If using interactive registration, check that the node key was approved on
  the VPS.
- Check the Headscale logs: `journalctl -u headscale -n 50`

**Exit node is active but `curl https://ifconfig.me` does not return the VPS IP**

- Verify the exit node routes are enabled: `headscale routes list`
- Confirm the client has the exit node set: `tailscale status`
- Check that IP forwarding is active on the VPS: `sysctl net.ipv4.ip_forward`
  (should return `1`).

**SSH connection refused over Tailscale IP**

- Verify the SSH daemon is running on the target device: `systemctl status sshd`
- Confirm the Tailscale IP is reachable: `ping <tailscale-ip>`
- Check that the public key was distributed: `cat ~/.ssh/authorized_keys` on
  the target device.
