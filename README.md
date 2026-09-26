# Webex "Failed to connect to the server" on Fedora — a debugging post-mortem

Notes from debugging a Fedora laptop where the Cisco Webex desktop client
would not sign in on a home Wi-Fi network, while Chrome, WhatsApp, and
other apps worked normally on the same network.

The first diagnosis (IPv6 half-up) turned out to be wrong. The real cause
was smaller-path MTU on the ISP + a fragile bundled TLS stack in an older
Webex build. Documenting both because the wrong turn is instructive.

## Symptom

- Webex sign-in screen: **"Failed to connect to the server. Sorry about
  that. Try again later."**
- Every other app on the laptop works over the same Wi-Fi.
- Switching to mobile hotspot makes Webex log in immediately.

## First diagnosis (wrong) — IPv6 half-up

The Wi-Fi router advertised an IPv6 default route (`fe80::1 via wlp3s0`)
but never delivered a global IPv6 prefix. `ip -6 addr show scope global`
was empty. That is a real problem in its own right — some apps do time
out on the v6 attempt before falling back to v4 — and it was tempting to
call it done.

Applied fix (still applied, still a reasonable state for this network):

```sh
sudo nmcli connection modify "<WIFI-NAME>" ipv6.method disabled
sudo nmcli connection down "<WIFI-NAME>" && sudo nmcli connection up "<WIFI-NAME>"
```

Result: Webex still failed. IPv6 was a real but unrelated problem.

## Real diagnosis — Path MTU + Webex's bundled TLS

The Webex client log had the actual error the whole time:

```
Failed to create HTTP request
uri: https://u2c.wbx2.com
exception: Error in SSL handshake
clientErrorCode: 167772294
```

Two things stacked:

1. **Path MTU is smaller than the Wi-Fi interface MTU.** The route cache
   showed `mtu 1480` toward Webex hosts while the Wi-Fi interface was set
   to 1500. `ping -M do -s 1472` returned `sendmsg: Message too long` —
   the kernel had already learned the path couldn't carry 1500-byte
   packets. Whenever Webex sent a large TLS ClientHello, the packet would
   need PMTU discovery to shrink; if ICMP `fragmentation-needed` is
   blocked upstream (very common on residential ISPs), the packet just
   vanishes silently and the handshake stalls out. Classic MTU black
   hole.
2. **Older Webex build with a fragile bundled TLS config.** Webex 46.8
   ships an `/opt/Webex/lib/openssl.cnf` that references only a
   `fips_sect` provider without `activate = 1`, and no default provider
   fallback. Under some execution paths this leaves the bundled libssl
   with no active crypto provider, and handshakes fail immediately with
   an SSL-handshake error even when the network is fine. Newer Webex
   builds ship a corrected config.

System `curl` and `openssl s_client` succeed to the same hosts because
they use the system TLS stack and honour the kernel's PMTU cache. The
Webex client uses its own bundled libcurl + libssl, and doesn't.

## Fixes worth applying if you hit this

### 1. Match Wi-Fi MTU to the real path MTU

```sh
sudo nmcli connection modify "<WIFI-NAME>" 802-11-wireless.mtu 1480
sudo nmcli connection down "<WIFI-NAME>" && sudo nmcli connection up "<WIFI-NAME>"
```

Revert later with `... 802-11-wireless.mtu 0`. This costs ~1.3% overhead
per packet — noise, not felt.

### 2. Upgrade Webex, don't just reinstall the same version

`sudo dnf upgrade -y webex` — or download the latest `.rpm` from
https://www.webex.com/downloads.html and install it. Any release newer
than 46.8 ships a corrected openssl.cnf.

### 3. Fall back to the web app

https://web.webex.com/ works in Chrome and uses the browser's TLS stack,
which honours PMTU and doesn't have the bundled-config problem. In this
case that's what I ended up using — the desktop client's fragility isn't
worth chasing further when the web app does the job.

## Lessons

- **Don't let the first suspicious clue drown out the actual error
  message.** The client log said "Error in SSL handshake" from the very
  first failure. IPv6 was a real but unrelated defect on the network;
  building the story around it delayed finding the real cause.
- **Reproduce with system tools first.** `curl -v` and `openssl s_client`
  against the same hostnames the client is failing on will tell you
  whether the problem is your network or the app's bundled stack. If
  system tools succeed and the client fails, the fault is in the app.
- **MTU black holes are quiet.** Any app that pushes large TLS handshakes
  (video conferencing, VPN clients, some game launchers) can trip on
  them. The kernel route cache (`ip route get <ip>`) is where to look.

## References

- Happy Eyeballs (RFC 8305): https://datatracker.ietf.org/doc/html/rfc8305
- Path MTU Discovery blackhole detection: https://datatracker.ietf.org/doc/html/rfc4821
- OpenSSL 3 provider activation:
  https://docs.openssl.org/3.0/man5/config/#provider-configuration
