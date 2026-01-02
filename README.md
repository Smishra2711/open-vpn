OPENWRT + OPENVPN FAILOVER SETUP
================================

QUICK CHECKLIST (RUN FIRST)
---------------------------

☐ VPN is OFF
☐ Run: ip route
☐ A default route via WAN exists
☐ LAN → WAN forwarding is allowed
☐ Masquerading is enabled on WAN
☐ Client device can ping router IP
☐ Client device uses router as gateway

If ANY box fails → internet will work via SSH but NOT via LAN devices


IMPORTANT NOTE (YOUR SYMPTOM)
-----------------------------

If:
- Internet works when pinging from OpenWrt via SSH
- Internet FAILS on phones / laptops / LAN ports

Then:
- Routing is correct
- Firewall forwarding or NAT is BROKEN

This is NOT an ISP or VPN issue.


NETWORK TOPOLOGY
----------------

Laptop / Phone / LAN devices
        ↓
   OpenWrt (dual-channel router, multiple LAN ports)
        ↓
      ONT
        ↓
       ISP

Optional:
OpenWrt → OpenVPN → GCP


ROOT CAUSE
----------

OpenVPN modifies routing and/or firewall state.
When VPN goes down:
- Default route may be missing OR
- LAN → WAN forwarding is blocked

OpenWrt itself still has internet (SSH works),
but LAN devices are isolated.


STEP 1: VERIFY DEFAULT ROUTE EXISTS
----------------------------------

VPN OFF

Run on OpenWrt (SSH):

ip route

EXPECTED:
default via <WAN-GATEWAY> dev wan metric 10

If NO default route exists:
- WAN is misconfigured
- Internet will never work for LAN clients


STEP 2: VERIFY WAN CONFIGURATION
--------------------------------

Edit /etc/config/network

config interface 'wan'
    option proto 'dhcp'
    option device 'eth0.2'
    option defaultroute '1'
    option metric '10'


STEP 3: VERIFY LAN CONFIGURATION (DUAL LAN ROUTERS)
---------------------------------------------------

For routers with multiple LAN ports:

config interface 'lan'
    option proto 'static'
    option ipaddr '192.168.1.1'
    option netmask '255.255.255.0'
    option device 'br-lan'

Verify:
- ALL LAN ports are part of br-lan
- Devices receive IP from router via DHCP

Run:
brctl show

Expected:
br-lan includes all LAN interfaces


STEP 4: VERIFY CLIENT GATEWAY
-----------------------------

On a connected device:

☐ IP address is 192.168.1.x
☐ Gateway is 192.168.1.1
☐ DNS is router or valid external DNS

If gateway is wrong → routing will fail


STEP 5: FIREWALL – THIS IS CRITICAL
----------------------------------

LuCI → Network → Firewall

Zone: lan
- Input: ACCEPT
- Output: ACCEPT
- Forward: ACCEPT

Zone: wan
- Masquerading: ENABLED
- Input: REJECT
- Output: ACCEPT
- Forward: REJECT

Forwarding rules:
- lan → wan  (MUST EXIST)

Without masquerading + forwarding:
- SSH ping works
- LAN devices fail


STEP 6: VERIFY FIREWALL FROM CLI
--------------------------------

Run:

iptables -t nat -L -n | grep MASQUERADE

Expected:
MASQUERADE  all  --  192.168.1.0/24  0.0.0.0/0


STEP 7: CONFIGURE VPN INTERFACE (NO DEFAULT ROUTE)
--------------------------------------------------

config interface 'vpn'
    option proto 'none'
    option device 'tun0'
    option defaultroute '0'
    option metric '20'


STEP 8: FIX OPENVPN CLIENT CONFIG
--------------------------------

REMOVE any of the following:

redirect-gateway def1
push "redirect-gateway def1"
push "redirect-gateway"


STEP 9: INSTALL POLICY-BASED ROUTING
-----------------------------------

opkg update
opkg install pbr luci-app-pbr


STEP 10: CONFIGURE PBR
---------------------

Services → Policy Routing

Enable PBR

Policy:
- Name: Route LAN via VPN
- Source: 192.168.1.0/24
- Interface: vpn

Behavior:
- VPN up → traffic via VPN
- VPN down → WAN fallback automatically


STEP 11: VPN FIREWALL ZONE
-------------------------

Zone: vpn
- Masquerading: ENABLED
- Input: REJECT
- Output: ACCEPT
- Forward: REJECT

Forwardings:
- lan → vpn
- vpn → wan (optional)


STEP 12: FINAL VERIFICATION
--------------------------

VPN OFF:
☐ LAN devices have internet
☐ ip route shows WAN default route

VPN ON:
☐ Traffic routes via VPN
☐ WAN still present as backup

Run:
ip route
ip rule
pbr status


COMMON FAILURE PATTERN (YOUR CASE)
----------------------------------

✓ OpenWrt can reach internet (SSH ping works)
✗ LAN devices cannot

Cause:
- Missing NAT (MASQUERADE)
- Missing lan → wan forwarding
- Firewall zone mismatch after VPN down


RECOMMENDED BEST PRACTICE
-------------------------

- Never allow OpenVPN to own default route
- Use PBR for all VPN routing
- Keep WAN routing permanent
- Treat VPN as optional overlay


END OF FILE
-----------

