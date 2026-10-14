# Procédure P3 — BGP EVPN avec Route Reflector

Prérequis : images `fwong_host` et `fwong_routeur` déjà buildées et déclarées en templates GNS3 (voir P1). Le template `fwong_routeur` doit avoir au moins 3 adaptateurs réseau (le RR en a besoin de 3, un par leaf).

## 1. Créer le projet

Nouveau projet GNS3 `P3_fwong`.

## 2. Poser les nœuds (topologie page 11 du sujet)

- 1 RR : `_fwong-1`
- 3 leafs (VTEP) : `_fwong-2`, `_fwong-3`, `_fwong-4`
- 3 hosts : `host_fwong-1`, `host_fwong-2`, `host_fwong-3`

## 3. Câbler

- `_fwong-1` e0 <-> `_fwong-2` e0
- `_fwong-1` e1 <-> `_fwong-3` e0
- `_fwong-1` e2 <-> `_fwong-4` e0
- `_fwong-2` e1 <-> `host_fwong-1`
- `_fwong-3` e1 <-> `host_fwong-2`
- `_fwong-4` e1 <-> `host_fwong-3`

## 4. Démarrer tous les nœuds

## 5. Configurer chaque équipement

Les configs complètes et commentées sont dans les fichiers de `p3/` (underlay, OSPF, BGP EVPN, VXLAN/bridge). Dans la console de chaque nœud, coller le contenu du fichier du même nom :
- RR : `_fwong-1` (voisins BGP dynamiques via `bgp listen range 1.1.1.0/24`)
- leafs : `_fwong-2`, `_fwong-3`, `_fwong-4`
- hosts : `_fwong-2_host` (20.1.1.1), `_fwong-3_host` (20.1.1.2), `_fwong-4_host` (20.1.1.3)

Interfaces dans le conteneur : `eth0`, `eth1`, `eth2` (GNS3 les nomme e0, e1, e2).

## 6. Vérifier étape par étape (captures attendues pages 11-14 du sujet)

- `show ip route` sur un leaf : loopbacks des autres routeurs visibles via OSPF.
- `show bgp summary` : session iBGP up avec le RR.
- `show bgp l2vpn evpn` : routes **type 3** visibles dès le départ (même sans host actif), routes **type 2** apparaissent dès qu'un host devient actif (MAC appris automatiquement, sans IP).
- Ping entre deux hosts connectés à des leafs différents → doit passer via le RR.
- Capture Wireshark sur un lien RR–leaf : paquets OSPF hello + ICMP encapsulé VXLAN.

## 7. Documenter (déjà fait : les fichiers sont commentés)

Commenter les configs FRR (loopback, OSPF, BGP EVPN) et les scripts vxlan/bridge de chaque équipement, à sauvegarder dans `p3/` avec le login dans le nom (ex. `_fwong-1`, `_fwong-2`, `_fwong-2_host`...).

## 8. Exporter

File > Export portable project > cocher **Include base images** > enregistrer sous `p3/P3.gns3project`.

## 9. Vérifier l'arborescence finale (modèle page 16 du sujet)

```
p3/
├── P3.gns3project
├── _fwong-1
├── _fwong-2
├── _fwong-2_host
├── _fwong-3
├── _fwong-3_host
├── _fwong-4
└── _fwong-4_host
```
