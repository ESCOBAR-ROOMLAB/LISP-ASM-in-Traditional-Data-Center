# LAB — LISP IP Mobility ASM in a Traditional DC

CML lab: `LAB_LISP_IP_Mobility_ASM_in_Traditional_DC_1`
Platform: Cat9000v (xTRs, IOS-XE 17.18), Cat8000v (ORs, MS/MR, branch), IOSv (CRs, core), IOSvL2 (access), Alpine (hosts)

## Scenario

DC01 has a scheduled outage and its hosts must be migrated to DC02 on short notice. Hosts move to their **equivalent VLAN** at DC02, but DC02 uses a **different IP subnet**, and there is **no L2 DCI** between the sites. Re-addressing every host is not an option inside the window.

LISP IP Mobility in **Across Subnets Mode (ASM)** lets a host keep its IP address after landing in a subnet that does not contain it. This is a **cold migration** design: the host is moved, not live-vMotioned.

## Topology

```
              CORP_USER-01 / GUEST_USER-01  (10.253.12.12, VRF 100 / 200)
                              |
                      BRANCH01-ITR-01  (RLOC 12.3.3.1)
                              |
   MS_MR-01 ---------------  L3 CORE  (BGP AS 699)
   MS 10.10.10.1          /           \
   MR 10.10.10.100   DC01-OR-01/02   DC02-OR-01/02     <- underlay only, no LISP
                         |                 |
                     DC01-CR-01        DC02-CR-01       <- transit: global + VRF 100/200
                         |                 |
                  DC01-xTR-01/02     DC02-xTR-01/02     <- enterprise gateway + LISP xTR
                  10.10.1.1/.2       10.20.2.1/.2          (RLOCs)
                         |                 |
                     DC01-AS-01        DC02-AS-01       <- no link between them
```

| | DC01 | DC02 |
|---|---|---|
| VLAN 10 — VRF 100 (CORP), IID 100 | 192.168.10.0/24 | 192.168.20.0/24 |
| VLAN 20 — VRF 200 (GUEST), IID 200 | 192.168.10.0/24 | 192.168.20.0/24 |
| SVIs / HSRP VIP | .252, .253 / .254 | .252, .253 / .254 |
| vMAC (grp 10 / grp 20) | `0200.0010.000a` / `0200.0020.0014` | identical |

| Host | IP | VRF | Home |
|---|---|---|---|
| CORP_SRV-01 | 192.168.10.100/24, gw .254 | 100 | DC01 |
| GUEST_SRV-01 | 192.168.10.100/24, gw .254 | 200 | DC01 (saved topology has it already migrated to DC02) |
| GUEST_SRV-02 | 192.168.20.101/24, gw .254 | 200 | DC02 |

VRFs 100 and 200 deliberately reuse the same addresses. Identify the VRF from the VLAN or port, never from the IP.

## Concepts

**EID and RLOC.** LISP splits a host's identity (EID — its IP) from its location (RLOC — the loopback of the xTR it sits behind). The mapping system (MS/MR) stores which RLOC currently serves each EID, so the EID can move without renumbering.

**Dynamic EID.** Each xTR defines both sites' host ranges as dynamic-EID prefixes (`DC01_*` = 192.168.10.0/25, `DC02_*` = 192.168.20.0/25) and applies both to its SVI with `lisp mobility`. That makes the migration work in either direction. The /25 keeps the SVIs and VIP (.250–.254) out of the detection range; without it, each xTR registers its neighbor's SVIs as mobile hosts.

**Detection.** When a packet arrives on a `lisp mobility` SVI from a source that belongs to a dynamic-EID prefix but not to the SVI's own subnet, the xTR learns the host, installs a local /32 host route pointing at the SVI, and registers the /32 with the Map-Server under its own RLOC.

**Registration and notification.** The /32 is more specific than the home site's /24, so it wins everywhere. The Map-Server notifies the previous registrant at the old site, which stops claiming the host. Remote ITRs holding a stale cache entry are corrected by Solicit-Map-Requests (SMR), LISP Publications (on supported IOS-XE versions) or on cache expiry.

**Reaching the gateway.** The host still uses its old gateway, 192.168.10.254, which does not exist at DC02. Two things cover this:

- The **same vMAC per VLAN at both sites**, so a host whose ARP cache still holds the gateway entry keeps forwarding without re-ARPing.
- **Proxy ARP** on the SVI, for a host whose cache is empty. Proxy ARP only answers when the xTR has a **route in the VRF** to the requested address. Without that route the host ARPs forever and never gets a reply. The CR advertises a default route to the xTRs to provide the needed route.

**Host-route advertisement into the DC core.** The xTRs redistribute LISP host routes (/32 only) into the per-VRF OSPF, so the local core learns where the migrated host now lives:

```
ip prefix-list LISP-HOST-ROUTES seq 5 permit 192.168.0.0/16 ge 32
route-map ADV-HOST-ROUTES permit 10
 match ip address prefix-list LISP-HOST-ROUTES
!
router ospf 100 vrf 100
 redistribute lisp route-map ADV-HOST-ROUTES
router ospf 200 vrf 200
 redistribute lisp route-map ADV-HOST-ROUTES
```

In production the DC core router is an **SD-WAN WAN Edge**, which then advertises the /32 into **OMP** in the matching service VPN, so the rest of the SD-WAN fabric follows the host as well. Note that in the lab there is just the demonstration of LISP IGP-Assist (redistribution of mobile host routes into an IGP). The lab does not focuses on SD-WAN or proper design of the CR.

**Encapsulation.** The C9Kv xTRs encapsulate in **VXLAN** (UDP 4789). Every xTR must match, so the branch Cat8000v is configured explicitly with `encapsulation vxlan` under the global `router lisp` → `service ipv4`.

**Macro-segmentation.** The xTRs are the enterprise gateway; the ORs only extend the underlay (in production a firewall pair sits between xTR and OR). VRFs map 1:1 to instance-ids and are never merged.

## Testing the lab

Mirror access ports are pre-provisioned on both access switches, so a host moves to the **same port number** on the other DC's switch:

| Port | DC01-AS-01 | DC02-AS-01 |
|---|---|---|
| Gi0/0 | CORP_SRV-01, VLAN 10 | CORP_SRV-01, VLAN 10 |
| Gi0/1 | GUEST_SRV-01, VLAN 20 | GUEST_SRV-01, VLAN 20 |
| Gi0/2 | GUEST_SRV-02, VLAN 20 | GUEST_SRV-02, VLAN 20 |

### Procedure — CORP_SRV-01, DC01 → DC02

0. **Let the host speak.** Detection needs a packet *from* the host. Usually VMs are chatty and send packets wihtout manual intervention, but for this lab is usually required to send the first packet manually:
   ```sh
   ping -c 3 10.253.12.12
   ```

1. **Start traffic.** From `CORP_USER-01`:
   ```sh
   ping 192.168.10.100
   ```
2. **Baseline.** Run the verification commands below. The host should be registered with DC01's RLOCs (10.10.1.1 / 10.10.1.2).
3. **Unplug.** In CML, delete the link `CORP_SRV-01 eth0 ↔ DC01-AS-01 Gi0/0`.
4. **Replug.** Create a new link `CORP_SRV-01 eth0 ↔ DC02-AS-01 Gi0/0`.
5. **Let the host speak.** Detection needs a packet *from* the host. If the user ping has not recovered on its own, send one from the server:
   ```sh
   ping -c 3 10.253.12.12
   ```
6. **Verify.** Run the verification commands again. Everything that pointed at DC01 should now point at DC02 (10.20.2.1 / 10.20.2.2).

Expect a short loss window at the move. This is a cold migration, not a zero-loss one.

To test the reverse direction, move `GUEST_SRV-01` from `DC02-AS-01 Gi0/1` back to `DC01-AS-01 Gi0/1`, using VRF / IID 200 in the commands.

## Verification

Use IID and VRF `100` for CORP_SRV-01, `200` for GUEST_SRV-01. Examples below use 100.

### xTRs at the new DC

```
show lisp instance-id 100 ipv4 database
```
Confirms the mobile host has been discovered: `192.168.10.100/32` appears as a dynamic-EID entry with this site's locators.

```
show ip route vrf 100
show ip route vrf 100 192.168.10.100
```
Confirms the host route has been populated in the RIB: a `/32` LISP route pointing out the local mobility SVI.

### MS/MR

```
show lisp instance-id 100 ipv4 server
```
Confirms the host-route registration is correct: `192.168.10.100/32` is registered with the **new DC's xTR RLOCs**, replacing the old ones. The /32 still falls under site DC01's `eid-record` (192.168.10.0/24, `accept-more-specifics`), but it is registered by DC02's xTRs — this only works because all sites share the same authentication key.

### Branch xTR

```
show lisp instance-id 100 ipv4 map-cache
```
Confirms the RLOC for the host route has been updated: the `192.168.10.100/32` entry now lists the DC02 locators. If it still shows DC01 while the ping runs, the cache has not been refreshed yet — the continuous ping makes the old site answer with an SMR.

### DC Core Router (new DC)

```
show ip route vrf 100
show ip route vrf 100 192.168.10.100
```
Confirms the host route has been redistributed into the IGP running in the core: `192.168.10.100/32` as an OSPF external route (`O E2` by default) learned from the local xTRs. In production this router is an SD-WAN WAN Edge that advertises the host route into OMP in the corresponding VPN.

### When something does not converge

| Symptom | Check |
|---|---|
| Host never appears in `database` | Is the host sending anything? Detection needs a packet from the host on the SVI. |
| Echo requests reach the host but no replies | The host is ARPing for its gateway and proxy ARP is not answering: `show ip interface Vlan10 \| include Proxy` and `show ip route vrf 100 192.168.10.100` on the new xTR. |
| Ping fails in one direction only | Encapsulation mismatch. Check the destination UDP port on a capture: 4789 VXLAN, 4341 LISP data. All xTRs must use the same one. |
| MS shows old RLOCs | Old xTR still holds the host. This can happen if the MS cannot notify the xTR of the new RLOC.
