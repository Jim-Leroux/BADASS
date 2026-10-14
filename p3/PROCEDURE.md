# Procédure P3 — BGP EVPN avec Route Reflector

Prérequis : images `eflaquet_host` et `eflaquet_routeur` déjà buildées et déclarées en templates GNS3 (voir P1). Le template `eflaquet_routeur` doit avoir au moins 3 adaptateurs réseau (le RR en a besoin de 3, un par leaf).

## 1. Créer le projet

Nouveau projet GNS3 `P3_eflaquet`.

## 2. Poser les nœuds (topologie page 11 du sujet)

- 1 RR : `_eflaquet-1`
- 3 leafs (VTEP) : `_eflaquet-2`, `_eflaquet-3`, `_eflaquet-4`
- 3 hosts : `host_eflaquet-1`, `host_eflaquet-2`, `host_eflaquet-3`

## 3. Câbler

- `_eflaquet-1` e0 <-> `_eflaquet-2` e0
- `_eflaquet-1` e1 <-> `_eflaquet-3` e0
- `_eflaquet-1` e2 <-> `_eflaquet-4` e0
- `_eflaquet-2` e1 <-> `host_eflaquet-1`
- `_eflaquet-3` e1 <-> `host_eflaquet-2`
- `_eflaquet-4` e1 <-> `host_eflaquet-3`

## 4. Démarrer tous les nœuds, entrer dans `vtysh` sur chaque routeur

## 5. Adressage underlay + loopbacks

Un `/30` par lien RR–leaf, une loopback `/32` par routeur (sert de router-id BGP et de next-hop iBGP) :

- `_eflaquet-1` : lo `1.1.1.1/32`, e0 `10.1.1.1/30`, e1 `10.1.1.5/30`, e2 `10.1.1.9/30`
- `_eflaquet-2` : lo `1.1.1.2/32`, e0 `10.1.1.2/30`
- `_eflaquet-3` : lo `1.1.1.3/32`, e0 `10.1.1.6/30`
- `_eflaquet-4` : lo `1.1.1.4/32`, e0 `10.1.1.10/30`

## 6. OSPF (underlay) — sur chaque routeur

```
router ospf
 network 10.1.1.0/24 area 0
 network 1.1.1.0/24 area 0
```

Vérifier que chaque leaf voit les loopbacks des autres via `show ip route`.

## 7. BGP EVPN — sur le RR (`_eflaquet-1`)

```
router bgp 1
 bgp router-id 1.1.1.1
 neighbor LEAFS peer-group
 neighbor LEAFS remote-as 1
 neighbor LEAFS update-source lo
 neighbor 1.1.1.2 peer-group LEAFS
 neighbor 1.1.1.3 peer-group LEAFS
 neighbor 1.1.1.4 peer-group LEAFS
 address-family l2vpn evpn
  neighbor LEAFS activate
  neighbor LEAFS route-reflector-client
  advertise-all-vni
 exit-address-family
```

## 8. BGP EVPN — sur chaque leaf (exemple `_eflaquet-2`)

```
router bgp 1
 bgp router-id 1.1.1.2
 neighbor 1.1.1.1 remote-as 1
 neighbor 1.1.1.1 update-source lo
 address-family l2vpn evpn
  neighbor 1.1.1.1 activate
  advertise-all-vni
 exit-address-family
```

Adapter le router-id pour `_eflaquet-3` (1.1.1.3) et `_eflaquet-4` (1.1.1.4).

## 9. VXLAN piloté par EVPN — sur chaque leaf (hors vtysh, shell du conteneur)

Contrairement à P2, plus de `remote`/`group` statique : c'est le control-plane BGP EVPN qui apprend les MAC distantes. Exemple pour `_eflaquet-2` :

```
ip link add vxlan10 type vxlan id 10 dstport 4789 local 1.1.1.2 nolearning
ip link add br0 type bridge
ip link set vxlan10 master br0
ip link set e1 master br0
ip link set vxlan10 up
ip link set e1 up
ip link set br0 up
```

Répéter sur `_eflaquet-3` (local `1.1.1.3`) et `_eflaquet-4` (local `1.1.1.4`).

## 10. Host

Sur chaque host, juste une IP sur l'interface overlay, par ex. `host_eflaquet-1` : `ip addr add 30.1.1.1/24 dev e1`.

## 11. Vérifier étape par étape (captures attendues pages 11-14 du sujet)

- `show ip route` sur un leaf : loopbacks des autres routeurs visibles via OSPF.
- `show bgp summary` : session iBGP up avec le RR.
- `show bgp l2vpn evpn` : routes **type 3** visibles dès le départ (même sans host actif), routes **type 2** apparaissent dès qu'un host devient actif (MAC appris automatiquement, sans IP).
- Ping entre deux hosts connectés à des leafs différents → doit passer via le RR.
- Capture Wireshark sur un lien RR–leaf : paquets OSPF hello + ICMP encapsulé VXLAN.

## 12. Documenter

Commenter les configs FRR (loopback, OSPF, BGP EVPN) et les scripts vxlan/bridge de chaque équipement, à sauvegarder dans `p3/` avec le login dans le nom (ex. `_eflaquet-1`, `_eflaquet-2`, `_eflaquet-2_host`...).

## 13. Exporter

File > Export portable project > cocher **Include base images** > enregistrer sous `p3/P3.gns3project`.

## 14. Vérifier l'arborescence finale (modèle page 16 du sujet)

```
p3/
├── P3.gns3project
├── _eflaquet-1
├── _eflaquet-2
├── _eflaquet-2_host
├── _eflaquet-3
├── _eflaquet-3_host
├── _eflaquet-4
└── _eflaquet-4_host
```
