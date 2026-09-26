# Draft message to ISP

Short, neutral, no PII. Fill in account number / router MAC when sending.

---

Hello,

I am seeing an IPv6 configuration issue on my home connection that is
breaking a few applications (Cisco Webex is the most obvious one, but
anything that prefers IPv6 over IPv4 is affected).

From the laptop side:

- The router advertises an IPv6 default route to connected devices
  (`fe80::1` via the LAN interface).
- No global IPv6 address is ever assigned — `ip -6 addr show scope global`
  is empty on every device on the network.
- IPv4 works normally to the same destinations.

The result is a "half-up" IPv6 setup: devices install a v6 default route
because they see the RA, but they have no source address, so any app that
prefers v6 (via Happy Eyeballs) hits a timeout before falling back.

Could you confirm one of the following on my account:

1. Is IPv6 actually delivered on this line? If yes, is the router
   configured to request a prefix via DHCPv6-PD, and is the delegation
   succeeding upstream?
2. If IPv6 is not being delivered, please disable IPv6 Router
   Advertisement on the LAN side of the CPE so client devices don't
   attempt v6 at all.

Either fix (delivering a real prefix, or turning off the LAN RA) resolves
the problem. Happy to test on your side or provide diagnostic output from
the device.

Account: <account number>
CPE model / MAC: <router model / MAC>

Thanks.
