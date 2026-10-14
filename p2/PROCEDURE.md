# Procédure P2 — VXLAN statique puis dynamique multicast

Prérequis : images `eflaquet_host` et `eflaquet_routeur` déjà buildées et déclarées en templates GNS3 (voir P1).

## 1. Créer le projet

Nouveau projet GNS3 `P2_eflaquet` (ou dupliquer le projet P1).

## 2. Poser les nœuds

- 1 switch Ethernet natif GNS3 : `Switch_eflaquet`
- 2 routeurs (template `eflaquet_routeur`) : `routeur-eflaquet-1`, `routeur-eflaquet-2`
- 2 hosts (template `eflaquet_host`) : `host-eflaquet-1`, `host-eflaquet-2`

## 3. Câbler (topologie page 9 du sujet)

- `Switch_eflaquet` e0 <-> `routeur-eflaquet-1` e0
- `Switch_eflaquet` e1 <-> `routeur-eflaquet-2` e0
- `routeur-eflaquet-1` e1 <-> `host-eflaquet-1`
- `routeur-eflaquet-2` e1 <-> `host-eflaquet-2`

## 4. Démarrer tous les nœuds

## 5. Mode statique (unicast)

Ouvrir la console de chaque équipement et exécuter le script correspondant déjà écrit dans `p2/` :
- `routeur-eflaquet-1` → contenu de `eflaquet-1_s`
- `routeur-eflaquet-2` → contenu de `eflaquet-2_s`
- `host-eflaquet-1` → contenu de `_eflaquet-1_host`
- `host-eflaquet-2` → contenu de `_eflaquet-2_host`

## 6. Vérifier le mode statique

- `ping 30.1.1.2` depuis `host-eflaquet-1`.
- Clic droit sur le lien `Switch_eflaquet` <-> `routeur-eflaquet-1` > Start capture, filtrer `udp.port==4789` dans Wireshark : on doit voir le VNI 10 et l'ICMP encapsulé.
- `brctl showmacs br0` sur les deux routeurs.

## 7. Basculer en mode dynamique multicast

Sur chaque routeur, nettoyer l'ancienne interface avant de rejouer le script `_g` :

```
ip link del vxlan10
ip link del br0
```

Puis exécuter :
- `routeur-eflaquet-1` → contenu de `eflaquet-1_g`
- `routeur-eflaquet-2` → contenu de `eflaquet-2_g`

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
├── eflaquet-1_s
├── eflaquet-1_g
├── eflaquet-2_s
├── eflaquet-2_g
├── _eflaquet-1_host
└── _eflaquet-2_host
```
