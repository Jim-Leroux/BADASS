# Procédure P1 — Créer et exporter le projet GNS3

Prérequis : être dans la VM (Linux), avec Docker et GNS3 installés (`sudo apt install docker.io gns3-server gns3-gui`, ajouter l'utilisateur au groupe `docker` puis redémarrer la session).

## 1. Builder les deux images

```
docker build -t eflaquet_host ./p1/eflaquet_host
docker build -t eflaquet_routeur ./p1/eflaquet_routeur
```

## 2. Déclarer les images comme templates GNS3

Edit > Preferences > Docker containers > New > "Existing image" :
- `eflaquet_host` : 1 adaptateur réseau suffit pour l'instant.
- `eflaquet_routeur` : mettre au moins 3 adaptateurs (besoin de 2 en P2, 3 pour le RR en P3), ça ne gêne pas de les avoir en trop dès P1.

## 3. Créer le projet

Nouveau projet GNS3, nom `P1_eflaquet`.

## 4. Poser et renommer les nœuds

Glisser un nœud de chaque template, renommer avec le login imposé par le sujet :
- `host_eflaquet-1`
- `routeur_eflaquet`

## 5. Câbler

Relier les deux nœuds par un lien Ethernet (une interface de chaque côté suffit pour cette partie).

## 6. Démarrer et vérifier

Start les deux nœuds, ouvrir les consoles :
- Les deux machines doivent être accessibles via GNS3 (console).
- Sur `host_eflaquet-1` : `ps` doit montrer les process busybox actifs.
- Sur `routeur_eflaquet` : `ps` doit montrer `zebra`, `bgpd`, `ospfd`, `isisd`.

## 7. Compléter la doc

Ajouter des commentaires dans `Dockerfile` et `daemons.conf` expliquant chaque choix (contrainte explicite du sujet : "add the configuration files with comments").

## 8. Exporter

File > Export portable project > cocher **Include base images** > enregistrer sous `p1/P1.gns3project`.

## 9. Vérifier l'arborescence finale

```
p1/
├── P1.gns3project
├── eflaquet_host/
│   └── Dockerfile
└── eflaquet_routeur/
    ├── Dockerfile
    └── daemons.conf
```
