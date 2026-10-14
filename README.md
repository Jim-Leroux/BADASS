# BADASS — BGP At Doors of Autonomous Systems is Simple

Network simulation with GNS3 and Docker: VXLAN, then BGP EVPN with a route reflector.
Everything runs inside a **Linux VM** (required by the subject).

## Contents

| Folder | Contents |
|---|---|
| `p1/` | The 2 Docker images (Alpine host, FRR router) and the GNS3 project `P1.gns3project` |
| `p2/` | Static VXLAN (`*_s`) then multicast (`*_g`), host scripts, `P2.gns3project` |
| `p3/` | BGP EVPN: 1 RR + 3 leafs + 3 hosts, configs `_fwong-*`, `P3.gns3project` |

Each part has a `PROCEDURE.md` describing the GNS3 project setup step by step.

## Requirements (inside the VM)

```
sudo apt install docker.io gns3-server gns3-gui
sudo usermod -aG docker $USER   # then log out and back in
```

## Build the images

```
docker build -t eflaquet_host    ./p1/eflaquet_host
docker build -t eflaquet_routeur ./p1/eflaquet_routeur
```

Declare them in GNS3 (Edit > Preferences > Docker containers > New > Existing image).
The router needs **at least 3 network adapters** (the RR uses 3 in P3).

## P3 at a glance

```
            _fwong-1 (RR, 1.1.1.1)
           eth0  eth1  eth2
            |     |     |
   _fwong-2   _fwong-3   _fwong-4      leafs / VTEPs (1.1.1.2 / .3 / .4)
      |eth1      |eth1      |eth1
   host 20.1.1.1  .2        .3         VXLAN overlay, VNI 10
```

- **Underlay**: one `/30` per RR–leaf link (`10.1.1.0/24`), loopbacks `1.1.1.x/32`, OSPF area 0.
- **Control plane**: iBGP AS 1, `l2vpn evpn` family, the RR reflects routes between leafs.
- **Data plane**: `vxlan10` (VNI 10, `nolearning`) + `br0` on each leaf.

To deploy: create the GNS3 project (see `p3/PROCEDURE.md`), then paste into each node's
console the file with the same name (`p3/_fwong-1` into the RR, and so on).

## How to test P3

Go in order. Do not move to the next step if one fails.

**0. Router image** (inside the VM):
```
docker run --rm --entrypoint sh eflaquet_routeur -c "ip link help 2>&1 | head -3; which vtysh"
```
Must print a `ip link add [link DEV | parentdev NAME]` usage (iproute2) and `/usr/bin/vtysh`.

**1. Underlay** — on the RR:
```
ip -br addr                        # eth0/1/2 = 10.1.1.1/5/9, lo = 1.1.1.1
vtysh -c "show ip ospf neighbor"   # 3 neighbors in Full (~40 s)
```
On a leaf: `ping -c1 1.1.1.1` and `ping -c1 1.1.1.4` must answer.

**2. BGP** — on the RR:
```
vtysh -c "show bgp summary"
```
Neighbors `1.1.1.2`, `1.1.1.3`, `1.1.1.4` must be **Established**.
A neighbor stuck in `Active`/`Connect` means an OSPF problem.

**3. Local VXLAN** — on a leaf:
```
ip -d link show vxlan10            # id 10, local 1.1.1.X, dstport 4789, nolearning
brctl show br0                     # eth1 and vxlan10 in the bridge
```

**4. EVPN routes** — on a leaf, hosts idle:
```
vtysh -c "show bgp l2vpn evpn"
```
**Type 3** routes (`[3]:[0]:[32]:[1.1.1.X]`, `RT:1:10`) for every VTEP, **no type 2** yet.
Make a host send traffic (`ping`): a **type 2** route (`[2]:[0]:[48]:[MAC]`) appears,
then spreads to the other leafs through the RR.

**5. End to end** — on `host_fwong-1`:
```
ping -c3 20.1.1.2
ping -c3 20.1.1.3
```
0% loss expected. Check the remote MAC with `bridge fdb show dev vxlan10` on a leaf.

**6. Wireshark capture** — right-click an RR–leaf link > Start capture, filter
`ospf || udp.port==4789`: OSPF Hello + ICMP encapsulated with **VNI 10**.

### Troubleshooting

| Symptom | Likely cause |
|---|---|
| No OSPF neighbor | interface not `up`, wrong `/30` address |
| BGP in `Active` | neighbor loopback unreachable (OSPF) |
| BGP up, no EVPN routes | `advertise-all-vni` missing, or `vxlan10` created after FRR started: `vtysh -c "clear bgp *"` |
| No type 2 route | host silent, or `eth1` not in `br0` |
| Routes OK but ping fails | `ip -d link show vxlan10`: check `local` and `nolearning` |

## Export

File > Export portable project > tick **Include base images** > `pN/PN.gns3project`.
