# BADASS


## 1️⃣ Installer GNS3 et Docker dans la VM

```bash
sudo add-apt-repository ppa:gns3/ppa
sudo apt update
sudo apt install gns3-gui docker.io
sudo usermod -aG docker $USER
sudo reboot
```

---

## 2️⃣ Récupérer le projet

```bash
# Cloner le repo GitHub
git clone git@github.com:Jim-Leroux/BADASS.git

cd BADASS/P1

# Décompresser le ZIP contenant les fichiers et images Docker
unzip BADASS_P1.zip
```

---

## 3️⃣ Importer les images Docker

```bash
# Restaurer les images Docker depuis les fichiers tar
docker load -i routeur_msharifi.tar
docker load -i host_jileroux-1.tar

# Vérifier que les images sont présentes
docker images
```

---

## 4️⃣ Lancer GNS3

* Ouvrir GNS3 dans la VM.
* Aller dans **Edit → Preferences → Docker Containers** et vérifier que les images `routeur_msharifi` et `host_jileroux-1` apparaissent.

---

## 5️⃣ Ajouter les containers dans le projet

1. Glisser-déposer les containers dans le projet.
2. Ajouter un lien entre le routeur et le host.
3. Ouvrir les terminaux auxiliaire de FRR & Alpine → `shell` pour vérifier que les machines fonctionnent et que les daemons sont actifs :

```bash
ps
```

* Pas besoin de config IP pour P1, tout est déjà dans les fichiers de config.

---

✅ Avec ce process, **il suffit de cloner, unzip, restaurer les images et lancer GNS3**. Tout est prêt à l’usage.

