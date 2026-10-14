# Procédure P2 — VXLAN statique puis dynamique multicast

Prérequis : images `fwong_host` et `fwong_routeur` déjà buildées et déclarées en templates GNS3 (voir P1).

## 1. Créer le projet

Nouveau projet GNS3 `P2_fwong` (ou dupliquer le projet P1).

## 2. Poser les nœuds

- 1 switch Ethernet natif GNS3 : `Switch_fwong`
- 2 routeurs (template `fwong_routeur`) : `routeur-fwong-1`, `routeur-fwong-2`
- 2 hosts (template `fwong_host`) : `host-fwong-1`, `host-fwong-2`

## 3. Câbler (topologie page 9 du sujet)

- `Switch_fwong` e0 <-> `routeur-fwong-1` e0
- `Switch_fwong` e1 <-> `routeur-fwong-2` e0
- `routeur-fwong-1` e1 <-> `host-fwong-1`
- `routeur-fwong-2` e1 <-> `host-fwong-2`

## 4. Démarrer tous les nœuds

## 5. Mode statique (unicast)

Ouvrir la console de chaque équipement et exécuter le script correspondant déjà écrit dans `p2/` :
- `routeur-fwong-1` → contenu de `fwong-1_s`
- `routeur-fwong-2` → contenu de `fwong-2_s`
- `host-fwong-1` → contenu de `_fwong-1_host`
- `host-fwong-2` → contenu de `_fwong-2_host`

## 6. Vérifier le mode statique

- `ping 30.1.1.2` depuis `host-fwong-1`.
- Clic droit sur le lien `Switch_fwong` <-> `routeur-fwong-1` > Start capture, filtrer `udp.port==4789` dans Wireshark : on doit voir le VNI 10 et l'ICMP encapsulé.
- `brctl showmacs br0` sur les deux routeurs.

## 7. Basculer en mode dynamique multicast

Sur chaque routeur, nettoyer l'ancienne interface avant de rejouer le script `_g` :

```
ip link del vxlan10
ip link del br0
```

Puis exécuter :
- `routeur-fwong-1` → contenu de `fwong-1_g`
- `routeur-fwong-2` → contenu de `fwong-2_g`

Les hosts ne changent pas.

## 8. Vérifier le mode dynamique

- `ip -d link show vxlan10` doit afficher `group 239.1.1.1`.
- Capture Wireshark : trafic ARP/multicast vers `239.1.1.1` visible en plus de l'ICMP (comme page 10 du sujet).

## 9. Exporter

File > Export portable project > cocher **Include base images** > enregistrer sous `p2/P2.gns3project`.

## 10. Vérifier l'arborescence finale

```
p2/
├── P2.gns3project
├── fwong-1_s
├── fwong-1_g
├── fwong-2_s
├── fwong-2_g
├── _fwong-1_host
└── _fwong-2_host
```
