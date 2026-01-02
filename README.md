# OpenWrt + OpenVPN Failover Setup

## Goal

- **VPN ON** → Route traffic via VPN  
- **VPN OFF / VPN fails** → Internet continues via ISP (WAN)  
- No LAN outage when VPN is down


---

## Quick Checklist (Run First)

> If internet works via SSH but fails on LAN devices, this checklist will catch it.

- [ ] VPN is **OFF**
- [ ] Run `ip route` on OpenWrt
- [ ] Default route via **WAN** exists
- [ ] LAN → WAN forwarding is allowed
- [ ] Masquerading (NAT) is enabled on WAN
- [ ] Client device can ping router IP
- [ ] Client device uses router as default gateway

❗ If **any** item fails → OpenWrt may have internet, but LAN devices will not.


---

## Important Symptom Explained

**Observed behavior**
- Internet works when pinging from OpenWrt via SSH
- Internet fails when using phones / laptops / LAN ports

**Meaning**
- Routing on OpenWrt is correct
- Firewall forwarding or NAT is broken

This is **not** an ISP issue and **not** a VPN tunnel issue.


---

## Network Topology
