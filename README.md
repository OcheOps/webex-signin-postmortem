# Wi-Fi IPv6 "half-up": one app broken, everything else fine

Notes from debugging a Fedora laptop where Cisco Webex would not sign in on
a home Wi-Fi network, while Chrome, WhatsApp, and other apps worked
normally on the same network.

Sharing this because the symptom (one specific app fails to connect while
everything else works) is easy to blame on the app, but the root cause was
the local network handing out a broken IPv6 configuration.

## Symptom

- Webex login screen shows: **"Failed to connect to the server. Sorry
  about that. Try again later."**
- Every other app on the laptop works over the same Wi-Fi.
- Switching to mobile hotspot makes Webex log in immediately.

## Diagnosis

Compared IPv4 and IPv6 reachability to a Webex login endpoint:

```sh
$ dig +short A idbroker.example.com
203.0.113.10
203.0.113.11

$ dig +short AAAA idbroker.example.com
2001:db8::10
2001:db8::11

$ curl -4 -sI --max-time 5 https://idbroker.example.com | head -1
HTTP/2 404

$ curl -6 -sI --max-time 5 https://idbroker.example.com | head -1
# (nothing — times out)
```

IPv4 to that host worked; IPv6 didn't.

Checked the laptop's IPv6 state:

```sh
$ ip -6 addr show scope global
# (empty — no global IPv6 address on the Wi-Fi interface)

$ ip -6 route show default
default via fe80::1 dev wlp3s0 proto ra metric 20600 pref medium
```

That is the smoking gun. The Wi-Fi router is broadcasting Router
Advertisements ("I speak IPv6, send v6 traffic to me"), so the laptop
installs a v6 default route toward it. But the router never delivers a
global v6 prefix, so the laptop has no v6 source address to send from.

## Why Webex, and not Chrome / WhatsApp?

- DNS returns both A (IPv4) and AAAA (IPv6) records for many services.
- Modern network stacks run **Happy Eyeballs**: try IPv6 first, fall back
  to IPv4 after ~300 ms if the v6 attempt doesn't complete.
- Chromium's built-in resolver (used by Electron apps like Webex) is more
  aggressive about preferring IPv6 than curl or Firefox.
- Big consumer apps mostly hit CDNs (Cloudflare, Akamai, Google) that
  either don't publish AAAA records on this ISP's DNS or return v4 first.
  Webex's identity/login endpoints publish AAAA prominently, so its login
  flow runs straight into the black hole and fails before the v4 fallback
  helps.
- glibc's `/etc/gai.conf` (`precedence ::ffff:0:0/96 100`) has no effect
  here — Chromium ignores it and does its own address selection.

## Fix 1 — Laptop only (surgical)

Disable IPv6 on the specific broken Wi-Fi profile via NetworkManager. This
touches nothing else — other Wi-Fi profiles (office, coffee shops) are
separate profiles and get their own defaults when the laptop joins them
for the first time.

```sh
# Replace <WIFI-NAME> with the SSID (or profile name from `nmcli connection show`)
sudo nmcli connection modify "<WIFI-NAME>" ipv6.method disabled
sudo nmcli connection down "<WIFI-NAME>" && sudo nmcli connection up "<WIFI-NAME>"
```

To revert later (e.g. if the ISP starts delivering IPv6 properly):

```sh
sudo nmcli connection modify "<WIFI-NAME>" ipv6.method auto
sudo nmcli connection down "<WIFI-NAME>" && sudo nmcli connection up "<WIFI-NAME>"
```

### Pros
- Fixes Webex immediately.
- Zero risk of breaking anything else.
- Roams cleanly — other Wi-Fi profiles are untouched.

### Cons
- Every device in the house has to be fixed individually.
- Guests joining the Wi-Fi still hit the same broken IPv6.

## Fix 2 — Router (household-wide)

Two acceptable end states on the router's web admin. Router shown here is
a generic ISP-provided ONT; other models have similar menus.

### Option A — Turn off IPv6 RA on the LAN
Recommended if the ISP is not actually delivering IPv6.

1. Log in to the router admin page (typically `http://192.168.100.1` or
   `http://192.168.1.1`). ISPs often issue two accounts — a limited user
   account and a super-admin account; the WAN pages usually need the
   super-admin.
2. **LAN → DHCP Server → IPv6** → disable **RA (Router Advertisement)**.
3. Apply, reboot the router.

Result: every device on the LAN gets IPv4 only, automatically. Webex and
anything else with the same issue starts working.

### Option B — Ask the WAN for a real IPv6 prefix
Only useful if the ISP actually supports IPv6.

1. **WAN → WAN Configuration** → open the active internet WAN entry.
2. Set **IPv6 connection type** to **DHCPv6** and enable
   **Prefix Acquisition = DHCPv6-PD**.
3. Apply, reboot.

If the WAN pulls a prefix, IPv6 works for the whole house. If not, ISP
doesn't actually deliver IPv6 to residential yet — fall back to Option A.

### FAQ — will disabling RA hide the Wi-Fi from guests?

No. SSID broadcast and Wi-Fi authentication are 802.11 (Layer 2). RAs are
Layer 3 IPv6 config, negotiated *after* a device has already associated to
the AP. The Wi-Fi is still visible and joinable — guests just get IPv4
only.

## Fix 3 — ISP (upstream)

The real fix is that the router should not be advertising IPv6 as an
option if the ISP has not delegated a prefix to it. Either turn off RA
upstream, or start delivering a v6 prefix. See `isp-message.md` for a
short template.

## References

- Happy Eyeballs (RFC 8305): https://datatracker.ietf.org/doc/html/rfc8305
- glibc address selection (`/etc/gai.conf`, RFC 3484 / 6724).
- Chromium's built-in DNS resolver and address sorting is separate from
  glibc — it does not honour `gai.conf`.
