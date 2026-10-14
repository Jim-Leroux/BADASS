# P1 — Questions probables en soutenance

**Pourquoi Alpine ?**
Léger, rapide à démarrer dans GNS3, et busybox est natif dedans — coche direct la contrainte "busybox ou équivalent".

**C'est quoi busybox ?**
Un seul binaire qui regroupe plein d'utilitaires Unix (ls, ping, ifconfig, sh…). Utile sur une image minimaliste.

**Pourquoi FRR et pas Quagga ?**
FRR est le fork actif de Quagga (Quagga n'est plus maintenu). Le sujet accepte l'un ou l'autre — FRR fait tourner zebra, bgpd, ospfd, isisd.

**C'est quoi zebra ?**
Le daemon central de FRR : il gère la table de routage du noyau et fait le lien entre les protocoles (BGP, OSPF, IS-IS…) et le kernel. C'est lui, la "gestion du routage" demandée par le sujet.

**Différence entre BGP, OSPF et IS-IS ?**
- OSPF et IS-IS : protocoles de routage interne (IGP), à l'intérieur d'un même AS.
- BGP : protocole entre AS différents (EGP), utilisé aussi ici en interne pour l'EVPN (partie 3).

**Pourquoi aucune IP configurée par défaut dans les images ?**
Consigne explicite du sujet : les images doivent être réutilisables telles quelles, l'adressage dépend de la topologie (P2 VXLAN, P3 BGP EVPN) donc il se fait après, pas dans le Dockerfile.

**Comment tu actives bgpd/ospfd/isisd sans leur donner d'IP ?**
Dans `daemons.conf` (copié vers `/etc/frr/daemons.conf`) on les passe à `yes` — les daemons démarrent sans config réseau, l'adressage se fera en P2/P3.

**Pourquoi `tini` comme entrypoint ?**
Sous Alpine il n'y a pas de systemd. `tini` sert de PID 1 minimal : il reap les processus zombies et lance `docker-start`, le script fourni par le paquet FRR qui démarre zebra puis chaque daemon activé.

**Comment tu vérifies que ça marche dans GNS3 ?**
Console sur le routeur → `ps` doit montrer zebra/bgpd/ospfd/isisd tournants.

**Pourquoi le nom des images/dossiers contient ton login ?**
Convention imposée par le sujet, pour identifier clairement le rendu de chaque étudiant (`fwong_host`, `fwong_routeur`).

**GNS3 vs Docker, quel rôle pour chacun ?**
Docker construit et fait tourner les images (le contenu des machines). GNS3 orchestre la topologie réseau : il crée les nœuds à partir de ces images, les câble entre eux, gère les consoles.

**Pourquoi 2 interfaces sur le routeur et 1 sur le host ?**
Le host n'a besoin que de se connecter au routeur. Le routeur, lui, devra relier plusieurs segments (underlay/overlay en P2, autre routeur en P3) — donc au moins 2 interfaces dès P1.

**C'est quoi le fichier `.gns3project` exporté ?**
Un export "portable" (ZIP) de tout le projet GNS3 — topologie + images Docker embarquées (option "Include base images") — pour que le rendu soit rejouable tel quel par un correcteur sans qu'il rebuild rien.
