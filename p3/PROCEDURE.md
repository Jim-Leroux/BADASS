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

Clic droit sur le fond du projet > **Start all nodes** (ou sélectionner chaque nœud > Start). Attendre que les sept nœuds soient verts.

Ouvrir ensuite la console de chaque nœud (clic droit > Console). C'est dans cette console que l'on colle la config à l'étape 5.

## 5. Configurer chaque équipement

Les configs complètes et commentées sont dans `p3/` (underlay, OSPF, BGP EVPN, VXLAN/bridge). Ouvrir le fichier, tout copier, le coller dans la console du nœud correspondant, puis valider.

Le nom du fichier host ne reprend pas le nom du nœud GNS3 : `_fwong-2_host` va dans `host_fwong-1`, pas dans le routeur `_fwong-2`.

| Nœud GNS3 | Fichier à coller | Rôle |
|---|---|---|
| `_fwong-1` | `p3/_fwong-1` | Route reflector |
| `_fwong-2` | `p3/_fwong-2` | Leaf / VTEP |
| `_fwong-3` | `p3/_fwong-3` | Leaf / VTEP |
| `_fwong-4` | `p3/_fwong-4` | Leaf / VTEP |
| `host_fwong-1` | `p3/_fwong-2_host` | Host 20.1.1.1 |
| `host_fwong-2` | `p3/_fwong-3_host` | Host 20.1.1.2 |
| `host_fwong-3` | `p3/_fwong-4_host` | Host 20.1.1.3 |

Interfaces dans le conteneur : `eth0`, `eth1`, `eth2` (GNS3 les nomme e0, e1, e2).

## 6. Vérifier étape par étape (captures attendues pages 11-14 du sujet)

Aller dans l'ordre. Ne pas passer à l'étape suivante si une vérification échoue.

### 6.1 Underlay (OSPF)

Sur le RR (`_fwong-1`) :

```
ip -br addr
vtysh -c "show ip ospf neighbor"
```

Attendu : `eth0`/`eth1`/`eth2` en `10.1.1.1/30`, `10.1.1.5/30`, `10.1.1.9/30`, loopback `1.1.1.1/32`. Trois voisins OSPF en **Full** (environ 40 s).

Sur un leaf (`_fwong-2`) :

```
vtysh -c "show ip route"
ping -c1 1.1.1.1
ping -c1 1.1.1.4
```

Attendu : les loopbacks des autres routeurs (`1.1.1.1`, `1.1.1.3`, `1.1.1.4`) apprises par OSPF. Les deux pings répondent.

### 6.2 BGP

Sur le RR :

```
vtysh -c "show bgp summary"
```

Attendu : voisins `1.1.1.2`, `1.1.1.3`, `1.1.1.4` en **Established**. Un voisin bloqué en `Active` ou `Connect` indique un problème OSPF (loopback injoignable).

### 6.3 VXLAN local

Sur un leaf :

```
ip -d link show vxlan10
brctl show br0
```

Attendu : `vxlan10` avec `id 10`, `local 1.1.1.X` (`.2` / `.3` / `.4` selon le leaf), `dstport 4789`, `nolearning`. Le bridge `br0` contient `eth1` et `vxlan10`.

### 6.4 Routes EVPN

Sur un leaf, **avant** tout trafic des hosts :

```
vtysh -c "show bgp l2vpn evpn"
```

Attendu : routes **type 3** (`[3]:[0]:[32]:[1.1.1.X]`, `RT:1:10`) pour chaque VTEP, et **aucune** route type 2.

Puis, depuis `host_fwong-1`, envoyer du trafic (`ping -c1 20.1.1.2`). Relancer `show bgp l2vpn evpn` sur un leaf : une route **type 2** (`[2]:[0]:[48]:[MAC]`) apparaît (MAC apprise automatiquement, sans IP) et est reflétée aux autres leafs par le RR.

### 6.5 Bout en bout

Sur `host_fwong-1` :

```
ping -c3 20.1.1.2
ping -c3 20.1.1.3
```

Attendu : 0 % de perte. Le trafic entre hosts de leafs différents passe par le RR, encapsulé VXLAN.

Sur un leaf, la MAC distante est visible avec :

```
bridge fdb show dev vxlan10
```

### 6.6 Capture Wireshark

Clic droit sur un lien RR–leaf > Start capture. Filtre :

```
ospf || udp.port==4789
```

Attendu : paquets OSPF Hello, et l'ICMP encapsulé VXLAN avec **VNI 10**.

### Dépannage

| Symptôme | Cause probable |
|---|---|
| Pas de voisin OSPF | interface pas `up`, ou mauvaise adresse `/30` |
| BGP en `Active` | loopback du voisin injoignable (OSPF) |
| BGP up, pas de routes EVPN | `advertise-all-vni` absent, ou `vxlan10` créé après le démarrage de FRR : `vtysh -c "clear bgp *"` |
| Pas de route type 2 | host silencieux, ou `eth1` absent de `br0` |
| Routes OK mais le ping échoue | `ip -d link show vxlan10` : vérifier `local` et `nolearning` |

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
