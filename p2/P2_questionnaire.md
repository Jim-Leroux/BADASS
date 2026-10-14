# Fiche de révision — P2 : Discovering a VXLAN

## 1. Concepts de base

**Q : Qu'est-ce qu'un VXLAN ?**
R : VXLAN (Virtual eXtensible LAN, RFC 7348) est une technologie d'overlay réseau qui encapsule des trames Ethernet (couche 2) dans des paquets UDP (couche 3/4), pour faire transiter un réseau L2 par-dessus une infrastructure L3 (l'underlay). Ça permet de faire croire à deux machines qu'elles sont sur le même segment Ethernet alors qu'elles sont séparées par un réseau routé.

**Q : Pourquoi utiliser VXLAN plutôt qu'un VLAN classique ?**
R : Un VLAN 802.1Q est limité à 4094 IDs et reste cantonné à un même domaine de diffusion L2. VXLAN utilise un identifiant sur 24 bits (le VNI), donc jusqu'à ~16 millions de réseaux possibles, et peut traverser un réseau routé (donc s'étendre sur plusieurs datacenters/sites), pas juste un switch local.

**Q : Que veut dire VNI ?**
R : VXLAN Network Identifier — l'équivalent d'un VLAN ID mais pour VXLAN, sur 24 bits. Chez nous : VNI = 10. Il doit être identique sur tous les VTEPs qui partagent le même overlay.

**Q : Que veut dire VTEP ?**
R : VXLAN Tunnel Endpoint — l'élément (ici, chacun de nos routeurs) qui encapsule les trames Ethernet entrantes en paquets UDP/VXLAN vers l'autre VTEP, et désencapsule les paquets VXLAN entrants pour les remettre sous forme de trames Ethernet vers l'hôte local.

**Q : Quel port UDP utilise VXLAN ?**
R : Le port de destination standard est **4789** (celui qu'on précise avec `dstport 4789`).

## 2. Underlay vs Overlay

**Q : Quelle est la différence entre underlay et overlay dans notre topologie ?**
R :
- **Underlay** = le réseau physique/réel qui transporte les paquets encapsulés : `eth0` des routeurs, adressé en `10.1.1.0/24`, relié via `Switch_wil`.
- **Overlay** = le réseau virtuel vu par les hosts, comme s'ils étaient sur le même câble : `eth1` des routeurs + hosts, adressé en `30.1.1.0/24`, transporté à l'intérieur du tunnel VXLAN.

**Q : Pourquoi les hosts n'ont-ils aucune configuration VXLAN ?**
R : Parce que l'encapsulation/désencapsulation est faite par les routeurs (VTEPs). Les hosts n'ont besoin que d'une IP sur le réseau overlay — pour eux, tout se passe comme sur un LAN normal.

## 3. Les commandes, expliquées

```sh
ip addr add 10.1.1.1/24 dev eth0
```
→ Donne une IP au VTEP sur l'interface underlay, nécessaire pour que les paquets UDP/VXLAN puissent être routés/switchés jusqu'à l'autre VTEP.

```sh
ip link add name vxlan10 type vxlan id 10 dev eth0 local 10.1.1.1 remote 10.1.1.2 dstport 4789
```
→ Crée l'interface virtuelle VXLAN :
- `id 10` : le VNI
- `dev eth0` : interface underlay utilisée pour envoyer/recevoir les paquets encapsulés
- `local` : IP source de ce VTEP
- `remote` : IP du VTEP distant unique → **mode statique/unicast**, le tunnel est figé vers une seule destination
- `dstport 4789` : port UDP standard VXLAN

```sh
ip link add name vxlan10 type vxlan id 10 dev eth0 group 239.1.1.1 dstport 4789
```
→ Variante **dynamique/multicast** : au lieu d'un `remote` fixe, on rejoint un groupe multicast. Les VTEPs se découvrent automatiquement via ce groupe (utile si on a plus de 2 VTEPs, pas besoin de configurer un tunnel par paire).

```sh
ip link add br0 type bridge
ip link set eth1 master br0
ip link set vxlan10 master br0
```
→ Crée un bridge Linux et y rattache deux interfaces :
- `eth1` : le côté local (vers le host)
- `vxlan10` : le côté distant (le tunnel)
Le bridge fait le pont entre les deux mondes : une trame arrivant sur `eth1` peut ressortir encapsulée sur `vxlan10`, et inversement.

## 4. Statique vs Multicast

**Q : Quelle est la vraie différence entre le mode statique et le mode dynamique/multicast ?**
R :
- **Statique (`remote`)** : on indique explicitement l'IP du VTEP distant. Simple, mais il faut une entrée par pair de VTEPs (ne scale pas bien avec beaucoup de nœuds).
- **Dynamique (`group`)** : les VTEPs rejoignent un groupe multicast commun. La découverte des autres VTEPs et l'apprentissage des adresses MAC distantes se font dynamiquement via ce groupe, sans lister chaque pair manuellement.

**Q : Le groupe 239.1.1.1, c'est quoi ?**
R : Une adresse IP multicast (plage 239.0.0.0/8 = "administratively scoped", usage local/privé). Tous les VTEPs qui veulent participer au même VXLAN en mode dynamique doivent rejoindre le même groupe.

## 5. Vérification / preuve de fonctionnement

**Q : Comment prouver que le VXLAN fonctionne ?**
R : Trois preuves demandées par le sujet :
1. `ping 30.1.1.2` depuis un host → doit répondre (connectivité overlay de bout en bout).
2. Une capture Wireshark/tcpdump sur l'interface underlay (`eth0`) montrant un paquet UDP port 4789 contenant le VNI 10 et, à l'intérieur, le ping ICMP entre les IP overlay.
3. `bridge fdb show` / `brctl showmacs br0` sur les routeurs → montre les adresses MAC apprises sur le bridge, certaines locales (`is local? yes`, apprises directement sur `eth1`) et d'autres distantes (apprises via `vxlan10`, donc via le tunnel).

**Q : Si le ping ne passe pas, où chercher ?**
R : Dans l'ordre :
1. `ip a` sur underlay et overlay → les IP sont-elles bien posées, interfaces `up` ?
2. `ip -d link show vxlan10` → l'interface existe, le VNI et le remote/group sont corrects ?
3. `bridge link` → `eth1` et `vxlan10` sont bien `master br0` ?
4. Le firewall/iptables ne bloque pas UDP 4789 ?
5. Les deux routeurs ont bien la même config (VNI identique, bon remote/group) ?

## 6. Questions pièges possibles à l'oral

**Q : Pourquoi le VNI doit-il être identique sur les deux VTEPs mais pas les adresses `local`/`remote` ?**
R : Le VNI identifie l'overlay (le "réseau logique") — s'il diffère, les deux VTEPs encapsulent/désencapsulent dans des tunnels différents et ne se comprennent pas. Les IP `local`/`remote`, elles, décrivent juste où envoyer/recevoir physiquement les paquets — elles sont forcément différentes puisque ce sont deux machines distinctes.

**Q : Est-ce que le bridge `br0` fait du routage ou du switching ?**
R : Du switching (couche 2) : il relaie des trames Ethernet entre `eth1` et `vxlan10` en apprenant les adresses MAC, exactement comme un switch physique — pas de décision de routage IP à ce niveau.

**Q : Pourquoi la config ne survit pas à un redémarrage du conteneur ?**
R : Ces commandes `ip link`/`ip addr` sont appliquées à chaud dans le namespace réseau du conteneur, elles ne sont pas écrites dans un fichier de config persistant (comme `/etc/network/interfaces`) ni lancées par un script de démarrage (pas de systemd sur Alpine). D'où l'intérêt de garder les scripts de config pour pouvoir les rejouer.
