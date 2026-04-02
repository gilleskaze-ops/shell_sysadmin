# 📦 Package Management sur Debian/Ubuntu
## Guide complet du SysAdmin prudent

> **Niveau** : Débutant → Intermédiaire  
> **OS cible** : Debian / Ubuntu (et dérivés : Mint, Pop!OS, Kali…)  
> **Objectif** : Maîtriser `apt` et les outils associés sans jamais casser son serveur

---

## Table des matières

1. [Comment fonctionne APT ?](#1-comment-fonctionne-apt-)
2. [Les sources de paquets (repositories)](#2-les-sources-de-paquets-repositories)
3. [Mettre à jour les listes de paquets](#3-mettre-à-jour-les-listes-de-paquets)
4. [Inspecter avant d'agir](#4-inspecter-avant-dagir)
5. [Installer un paquet avec prudence](#5-installer-un-paquet-avec-prudence)
6. [Mettre à jour les paquets installés](#6-mettre-à-jour-les-paquets-installés)
7. [Simuler une opération avec --dry-run](#7-simuler-une-opération-avec---dry-run)
8. [Supprimer un paquet proprement](#8-supprimer-un-paquet-proprement)
9. [Gérer les dépendances orphelines](#9-gérer-les-dépendances-orphelines)
10. [Verrouiller un paquet (apt hold)](#10-verrouiller-un-paquet-apt-hold)
11. [Historique des actions apt](#11-historique-des-actions-apt)
12. [Outils de diagnostic avancés](#12-outils-de-diagnostic-avancés)
13. [Checklist du SysAdmin avant toute opération](#13-checklist-du-sysadmin-avant-toute-opération)
14. [Glossaire](#14-glossaire)

---

## 1. Comment fonctionne APT ?

### L'architecture en couches

APT (*Advanced Package Tool*) est le gestionnaire de paquets de haut niveau de Debian/Ubuntu. Il repose sur une pile de couches :

```
[Tu tapes]   apt install nginx
                    |
             [APT résout les dépendances]
                    |
             [dpkg installe le .deb]
                    |
             [Le paquet est installé sur le système]
```

| Outil | Rôle | Niveau |
|-------|------|--------|
| `apt` | Interface haut niveau, résout les dépendances | Tu l'utilises au quotidien |
| `apt-get` | Ancienne interface, plus scriptable | Scripts et automatisation |
| `dpkg` | Installe/supprime un fichier `.deb` directement | Bas niveau, pas de résolution de dépendances |
| `snap` / `flatpak` | Gestionnaires alternatifs, sandboxés | Applications isolées |

> ⚠️ **Règle d'or** : n'utilise **jamais** `dpkg` pour installer manuellement un paquet sauf si tu sais exactement ce que tu fais. `apt` gère les dépendances ; `dpkg` non.

### Le cycle de vie d'un paquet

```
Repository (internet)
        |
   apt update          ← télécharge les listes de paquets disponibles
        |
   apt install foo     ← résout les dépendances + télécharge + installe
        |
   dpkg -i foo.deb     ← (sous le capot) installe le fichier .deb
        |
   Paquet installé     ← visible dans dpkg --list
```

---

## 2. Les sources de paquets (repositories)

### Où APT va chercher les paquets ?

APT consulte des **dépôts** (repositories) : des serveurs qui hébergent des paquets `.deb`. La liste de ces dépôts est stockée dans deux endroits :

```
/etc/apt/sources.list              ← fichier principal
/etc/apt/sources.list.d/*.list    ← fichiers additionnels (PPAs, dépôts tiers)
```

### Anatomie d'une ligne de dépôt

```
deb https://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
 |         |                           |      |
 |         |                           |      └─ Sections (main, restricted, universe, multiverse)
 |         |                           └─ Nom de code de la release (jammy = Ubuntu 22.04)
 |         └─ URL du dépôt
 └─ Type : deb (binaires) ou deb-src (sources)
```

### Les sections expliquées

| Section | Contenu | Support |
|---------|---------|---------|
| `main` | Logiciels libres officiellement supportés | ✅ Canonical |
| `restricted` | Drivers non-libres mais supportés | ✅ Canonical |
| `universe` | Logiciels libres maintenus par la communauté | ⚠️ Communauté |
| `multiverse` | Logiciels non-libres | ❌ Aucun support officiel |

> 💡 **Bonne pratique** : sur un serveur de production, reste sur `main` et `restricted`. Évite `universe` et surtout `multiverse` sauf nécessité absolue.

### Vérifier tes sources

```bash
cat /etc/apt/sources.list
ls /etc/apt/sources.list.d/
```

### Les PPAs : à manipuler avec précaution

Un PPA (*Personal Package Archive*) est un dépôt non-officiel hébergé sur Launchpad (Ubuntu). Ils permettent d'accéder à des versions plus récentes de logiciels, mais :

- ❌ Pas de revue de sécurité officielle
- ❌ Peuvent écraser des paquets système critiques
- ❌ Peuvent devenir orphelins (plus maintenus)

```bash
# Ajouter un PPA (à utiliser avec précaution)
sudo add-apt-repository ppa:auteur/nom-du-ppa

# Lister les PPAs installés
ls /etc/apt/sources.list.d/

# Supprimer un PPA
sudo add-apt-repository --remove ppa:auteur/nom-du-ppa
```

> ⚠️ **Règle SysAdmin** : sur un serveur de production, **évite les PPAs**. Sur une machine de dev, c'est acceptable mais documente ce que tu fais.

---

## 3. Mettre à jour les listes de paquets

### `apt update` : la commande de départ obligatoire

```bash
sudo apt update
```

**Ce que fait cette commande :**
- Contacte chaque dépôt listé dans `sources.list`
- Télécharge les fichiers d'index (`Packages.gz`) qui listent les paquets disponibles et leurs versions
- Met à jour le cache local dans `/var/lib/apt/lists/`

**Ce qu'elle ne fait PAS :**
- Elle n'installe rien
- Elle ne met à jour aucun paquet installé

> 🔑 **Règle fondamentale** : toujours lancer `apt update` avant `apt install` ou `apt upgrade`. Sinon APT travaille sur des informations potentiellement obsolètes.

### Lire la sortie de `apt update`

```
Hit:1 https://archive.ubuntu.com/ubuntu jammy InRelease
Get:2 https://archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB]
Get:3 https://archive.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
Fetched 229 kB in 2s (114 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
```

| Préfixe | Signification |
|---------|---------------|
| `Hit` | Le dépôt n'a pas changé depuis la dernière mise à jour |
| `Get` | Nouvelles données téléchargées |
| `Ign` | Ignoré (souvent normal, ex: fichiers de traduction manquants) |
| `Err` | Erreur ! Le dépôt est inaccessible ou corrompu |

> ⚠️ Si tu vois des lignes `Err`, **ne continue pas**. Résous le problème d'abord : dépôt inaccessible, clé GPG manquante, problème réseau.

### Vérifier l'authenticité des paquets (GPG)

APT vérifie la signature cryptographique de chaque dépôt. Si une clé manque :

```
W: GPG error: https://... InRelease: The following signatures couldn't be verified
   because the public key is not available: NO_PUBKEY ABCDEF123456
```

**Solution :**

```bash
# Importer la clé manquante
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys ABCDEF123456

# Méthode moderne (recommandée)
curl -fsSL https://example.com/repo.key | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/repo.gpg
```

> 💡 **Pourquoi c'est important ?** Sans vérification GPG, un attaquant pourrait faire passer un paquet malveillant pour un paquet légitime (attaque man-in-the-middle).

---

## 4. Inspecter avant d'agir

### `apt list --installed` : voir ce qui est installé

```bash
apt list --installed
```

Filtre sur un paquet précis :

```bash
apt list --installed | grep nginx
```

### `apt list --upgradable` : la commande clé avant tout upgrade

```bash
apt list --upgradable
```

**Exemple de sortie :**

```
Listing... Done
curl/jammy-updates 7.81.0-1ubuntu1.15 amd64 [upgradable from: 7.81.0-1ubuntu1.14]
libcurl4/jammy-updates 7.81.0-1ubuntu1.15 amd64 [upgradable from: 7.81.0-1ubuntu1.14]
openssl/jammy-updates 3.0.2-0ubuntu1.14 amd64 [upgradable from: 3.0.2-0ubuntu1.13]
```

**Comment lire cette sortie :**

```
nginx/jammy-updates 1.24.0-1ubuntu1 amd64 [upgradable from: 1.22.0-1ubuntu1]
  |        |              |            |              |
  |        |              |            |              └─ Version actuellement installée
  |        |              |            └─ Architecture (amd64 = 64 bits)
  |        |              └─ Nouvelle version disponible
  |        └─ Dépôt source de la mise à jour
  └─ Nom du paquet
```

> 🔍 **Bonne pratique SysAdmin** : avant un `apt upgrade`, passe en revue cette liste. Identifie les paquets critiques (kernel, openssl, libc6) qui nécessitent une attention particulière.

### `apt show` : inspecter un paquet avant installation

```bash
apt show nginx
```

Sortie typique :

```
Package: nginx
Version: 1.24.0-1ubuntu1
Priority: optional
Section: web
Maintainer: Ubuntu Developers
Installed-Size: 87.0 kB
Depends: nginx-core (<< 1.24.0-1ubuntu1.1~) | nginx-full (<< 1.24.0-1ubuntu1.1~) | ...
Homepage: https://nginx.net
Description: small, powerful, scalable web/proxy server
```

**Ce qu'il faut vérifier :**
- `Depends` : quelles dépendances seront installées ?
- `Version` : est-ce la version attendue ?
- `Maintainer` : est-ce un paquet officiel ?
- `Installed-Size` : combien d'espace sera utilisé ?

### `apt-cache policy` : comparer les versions disponibles

```bash
apt-cache policy nginx
```

```
nginx:
  Installed: 1.22.0-1ubuntu1
  Candidate: 1.24.0-1ubuntu1
  Version table:
 *** 1.24.0-1ubuntu1 500
        500 https://archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages
     1.22.0-1ubuntu1 100
        100 /var/lib/dpkg/status
```

**Lecture :**
- `Installed` : version actuellement sur le système
- `Candidate` : version qu'APT installerait si tu lances `apt install nginx`
- `500` / `100` : priorité du dépôt (plus c'est élevé, plus il est prioritaire)

> 💡 **Cas d'usage** : tu veux savoir si un paquet vient du dépôt officiel ou d'un PPA. `apt-cache policy` te le dit précisément.

### `dpkg -l` : lister les paquets installés via dpkg

```bash
dpkg -l
dpkg -l | grep nginx
dpkg -l | grep "^ii"   # seulement les paquets correctement installés
```

**Les états dpkg :**

```
ii  nginx    1.24.0   amd64   ...   ← installé et fonctionnel
rc  foo      1.0.0    amd64   ...   ← supprimé mais fichiers de config restants
un  bar      <aucun>  <aucun> ...   ← non installé
```

| Code | Signification |
|------|---------------|
| `ii` | Installed, OK |
| `rc` | Removed, Config files remain |
| `un` | Unknown, Not installed |
| `hi` | Hold, Installed (paquet verrouillé) |

---

## 5. Installer un paquet avec prudence

### La séquence correcte

```bash
# 1. Mettre à jour les listes
sudo apt update

# 2. Inspecter le paquet avant installation
apt show nom-du-paquet

# 3. Simuler l'installation (voir section 7)
sudo apt install --dry-run nom-du-paquet

# 4. Installer si tout est OK
sudo apt install nom-du-paquet
```

### Lire ce qu'APT va faire avant de confirmer

Quand tu lances `apt install`, APT affiche toujours un résumé avant de demander confirmation :

```
The following additional packages will be installed:
  libgd3 libwebp7 nginx-common

The following NEW packages will be installed:
  libgd3 libwebp7 nginx nginx-common

0 upgraded, 4 newly installed, 0 to remove and 12 not upgraded.
Need to get 891 kB of archives.
After this operation, 2,789 kB of additional disk space will be used.
Do you want to continue? [Y/n]
```

**À analyser :**
- **"additionally installed"** : des dépendances vont être installées — tu as vu ça ?
- **"upgraded"** : est-ce que ça va mettre à jour un paquet déjà présent ?
- **"to remove"** : est-ce que l'installation va supprimer quelque chose ?! ⚠️
- **Disk space** : as-tu assez d'espace ?

> ⚠️ **Alerte rouge** : si la ligne `to remove` n'est pas à `0`, **stoppe et analyse**. Cela signifie qu'il y a un conflit de dépendances et qu'APT va supprimer un paquet existant pour résoudre le conflit. C'est rarement ce que tu veux sur un serveur de production.

### Installer une version spécifique

```bash
# Installer une version précise
sudo apt install nginx=1.22.0-1ubuntu1

# Voir les versions disponibles
apt-cache showpkg nginx
```

> 💡 **Bonne pratique** : en production, épingle les versions critiques plutôt que d'installer "la dernière". Ça évite les surprises lors des mises à jour automatiques.

### Répondre automatiquement (scripts)

```bash
# -y : répond "oui" automatiquement (attention !)
sudo apt install -y nginx

# Mieux dans les scripts : utiliser DEBIAN_FRONTEND
DEBIAN_FRONTEND=noninteractive sudo apt install -y nginx
```

> ⚠️ Le `-y` automatique est pratique dans les scripts mais **dangereux à la main** : il bypasse la phase de confirmation où tu aurais pu détecter un problème.

---

## 6. Mettre à jour les paquets installés

### `apt upgrade` vs `apt full-upgrade` : quelle différence ?

| Commande | Comportement |
|----------|-------------|
| `apt upgrade` | Met à jour les paquets sans jamais en supprimer ni en installer de nouveaux |
| `apt full-upgrade` | Peut installer ET supprimer des paquets pour résoudre les dépendances |
| `apt dist-upgrade` | Alias de `full-upgrade` (ancienne syntaxe) |

> 🔑 **Règle SysAdmin** : sur un serveur de production, préfère **toujours** `apt upgrade`. Il est conservateur. `full-upgrade` peut surprendre en supprimant des paquets.

### Procédure de mise à jour sécurisée

```bash
# Étape 1 : mettre à jour les listes
sudo apt update

# Étape 2 : voir ce qui va être mis à jour
apt list --upgradable

# Étape 3 : simuler la mise à jour (voir section 7)
sudo apt upgrade --dry-run

# Étape 4 : si le résumé est satisfaisant, procéder
sudo apt upgrade
```

### Les mises à jour de sécurité uniquement

Sur un serveur, tu peux vouloir n'appliquer que les patches de sécurité sans toucher aux autres paquets :

```bash
# Installer unattended-upgrades
sudo apt install unattended-upgrades

# Appliquer manuellement seulement les mises à jour de sécurité
sudo unattended-upgrade --dry-run
sudo unattended-upgrade
```

Ou plus simplement :

```bash
# N'upgrader que les paquets du dépôt security
sudo apt upgrade -o Dir::Etc::SourceList=/etc/apt/sources.list.d/ubuntu-security.list
```

### Mettre à jour le kernel : attention particulière

```bash
apt list --upgradable | grep linux-image
```

Une mise à jour du kernel nécessite un **reboot** pour être effective. Sur un serveur :

1. Planifie une fenêtre de maintenance
2. Préviens les utilisateurs
3. Fais un snapshot/backup avant
4. Redémarre avec `sudo reboot`
5. Vérifie que le bon kernel est actif : `uname -r`

> ⚠️ Ne jamais `apt full-upgrade` un serveur de production sans tester d'abord sur un environnement de staging.

---

## 7. Simuler une opération avec `--dry-run`

### Le filet de sécurité du SysAdmin prudent

L'option `--dry-run` (ou `-s` pour *simulate*) dit à APT de **simuler l'opération sans rien modifier** sur le système. C'est ton meilleur ami avant toute opération risquée.

```bash
# Syntaxes équivalentes
sudo apt install --dry-run nginx
sudo apt install -s nginx
sudo apt-get install --dry-run nginx
```

### Exemple de sortie `--dry-run`

```bash
sudo apt install --dry-run nginx
```

```
NOTE: This is only a simulation!
      apt needs root privileges for real execution.
      Keep also in mind that locking is deactivated,
      so don't depend on the relevance to the real current situation!
Inst libgd3 (2.3.0-2ubuntu2 Ubuntu:22.04/jammy [amd64])
Inst libwebp7 (1.2.2-2 Ubuntu:22.04/jammy [amd64])
Inst nginx-common (1.24.0-1ubuntu1 Ubuntu:22.04/jammy-updates [all])
Inst nginx (1.24.0-1ubuntu1 Ubuntu:22.04/jammy-updates [amd64])
Conf libgd3 (2.3.0-2ubuntu2 Ubuntu:22.04/jammy [amd64])
Conf libwebp7 (1.2.2-2 Ubuntu:22.04/jammy [amd64])
Conf nginx-common (1.24.0-1ubuntu1 Ubuntu:22.04/jammy-updates [all])
Conf nginx (1.24.0-1ubuntu1 Ubuntu:22.04/jammy-updates [amd64])
```

**Lecture des lignes :**
- `Inst` : sera **installé**
- `Conf` : sera **configuré**
- `Remv` : sera **supprimé** ← ⚠️ à surveiller impérativement
- `Upgrd` : sera **mis à jour**

### `--dry-run` sur un upgrade

```bash
sudo apt upgrade --dry-run
```

```
Reading package lists... Done
Building dependency tree... Done
The following packages will be upgraded:
  curl libcurl4 openssl
3 upgraded, 0 newly installed, 0 to remove and 0 not upgraded.
Inst curl [7.81.0-1ubuntu1.14] (7.81.0-1ubuntu1.15 Ubuntu:22.04/jammy-updates [amd64])
Inst libcurl4 [7.81.0-1ubuntu1.14] (7.81.0-1ubuntu1.15 Ubuntu:22.04/jammy-updates [amd64])
Inst openssl [3.0.2-0ubuntu1.13] (3.0.2-0ubuntu1.14 Ubuntu:22.04/jammy-updates [amd64])
```

> 🔑 **Règle absolue** : avant tout `apt upgrade` ou `apt full-upgrade` sur un serveur de production, **toujours passer par `--dry-run`** et analyser chaque ligne `Remv`.

### Combiner avec `grep` pour filtrer

```bash
# Voir seulement ce qui sera supprimé
sudo apt upgrade --dry-run | grep "^Remv"

# Voir seulement ce qui sera installé
sudo apt install --dry-run paquet | grep "^Inst"
```

---

## 8. Supprimer un paquet proprement

### Trois niveaux de suppression

```bash
# Niveau 1 : supprime le paquet mais garde les fichiers de configuration
sudo apt remove nginx

# Niveau 2 : supprime le paquet ET les fichiers de configuration
sudo apt purge nginx

# Niveau 3 : purge + suppression des dépendances devenues inutiles
sudo apt purge nginx && sudo apt autoremove
```

### Quelle commande choisir ?

| Situation | Commande recommandée |
|-----------|---------------------|
| Je veux juste désinstaller temporairement | `apt remove` |
| Je veux supprimer proprement et définitivement | `apt purge` |
| Je réinstalle plus tard avec la même config | `apt remove` (garde la config) |
| Je change de logiciel définitivement | `apt purge` + `apt autoremove` |

> ⚠️ **Piège classique** : `apt remove nginx` sur un serveur Ubuntu peut laisser des fichiers de config dans `/etc/nginx/`. Si tu réinstalle nginx plus tard, ces anciens fichiers seront réutilisés — parfois source de bugs mystérieux.

### Simuler une suppression

```bash
sudo apt remove --dry-run nginx
sudo apt purge --dry-run nginx
```

Vérifie toujours la ligne **"to remove"** — parfois supprimer un paquet entraîne une cascade de suppressions de dépendances qui peuvent affecter d'autres services.

---

## 9. Gérer les dépendances orphelines

### Qu'est-ce qu'un paquet orphelin ?

Quand tu installes `nginx`, APT installe aussi des dépendances (`libgd3`, `libwebp7`, etc.). Si tu supprimes `nginx`, ces dépendances restent sur le système mais ne servent plus à rien.

### `apt autoremove` : nettoyer les orphelins

```bash
# Voir ce qui serait supprimé (simulation)
sudo apt autoremove --dry-run

# Supprimer les paquets orphelins
sudo apt autoremove
```

> ⚠️ **Ne jamais lancer `autoremove` sans `--dry-run` d'abord.** Parfois, APT considère comme "orphelin" un paquet que tu utilises mais que tu as installé manuellement sans qu'aucun autre paquet ne le "réclame".

### `apt-mark` : contrôler le statut d'un paquet

```bash
# Marquer un paquet comme "installé manuellement" (ne sera pas autoremoved)
sudo apt-mark manual nom-du-paquet

# Marquer comme "installé automatiquement" (sera candidat à l'autoremove)
sudo apt-mark auto nom-du-paquet

# Voir le statut de tous les paquets
apt-mark showmanual
apt-mark showauto
```

> 💡 **Bonne pratique** : quand tu installes un outil que tu veux garder indéfiniment (comme `htop`, `vim`, `curl`), marque-le comme `manual`. Ainsi `autoremove` ne le touchera jamais.

### `deborphan` : outil spécialisé pour trouver les orphelins

```bash
sudo apt install deborphan
deborphan           # liste les bibliothèques orphelines
deborphan --all     # liste tous les paquets orphelins
```

---

## 10. Verrouiller un paquet (`apt hold`)

### Pourquoi verrouiller un paquet ?

Scénario réel : tu as testé que `nginx 1.22.0` fonctionne avec ton application. La version `1.24.0` est disponible mais tu ne veux pas qu'elle soit installée automatiquement avant d'avoir testé la compatibilité.

```bash
# Verrouiller un paquet (ne sera plus mis à jour)
sudo apt-mark hold nginx

# Vérifier les paquets verrouillés
apt-mark showhold

# Déverrouiller
sudo apt-mark unhold nginx
```

### Méthode alternative avec `dpkg`

```bash
# Verrouiller
echo "nginx hold" | sudo dpkg --set-selections

# Vérifier
dpkg --get-selections | grep hold
```

> 💡 **Cas d'usage Cloud** : verrouiller la version du kernel sur un serveur critique, ou verrouiller `docker-ce` à une version précise validée par ton équipe.

---

## 11. Historique des actions apt

### Le log d'APT : ton journal de bord

Toutes les actions APT sont logguées dans :

```
/var/log/apt/history.log       ← actions récentes (lisible)
/var/log/apt/term.log          ← sortie terminal des opérations
/var/log/dpkg.log              ← log bas niveau dpkg
```

### Lire l'historique

```bash
cat /var/log/apt/history.log
```

Exemple de sortie :

```
Start-Date: 2024-03-15  14:23:41
Commandline: apt install nginx
Requested-By: gilles (1000)
Install: nginx:amd64 (1.24.0-1ubuntu1), nginx-common:amd64 (1.24.0-1ubuntu1, automatic)
End-Date: 2024-03-15  14:23:55

Start-Date: 2024-03-16  09:12:03
Commandline: apt remove nginx
Requested-By: gilles (1000)
Remove: nginx:amd64 (1.24.0-1ubuntu1)
End-Date: 2024-03-16  09:12:08
```

### Filtrer l'historique

```bash
# Voir les 50 dernières lignes
tail -50 /var/log/apt/history.log

# Chercher toutes les installations de nginx
grep -A5 "Install.*nginx" /var/log/apt/history.log

# Voir qui a fait quoi
grep "Requested-By" /var/log/apt/history.log
```

> 💡 **Bonne pratique SysAdmin** : en cas de problème sur un serveur, **commence toujours par consulter `/var/log/apt/history.log`**. C'est souvent là que tu trouveras qu'un collègue a installé quelque chose la veille qui a causé le bug.

### `apt-history` (outil tiers pratique)

```bash
sudo apt install apt-history
apt-history install    # voir tous les installs
apt-history rollback   # voir les rollbacks possibles
```

---

## 12. Outils de diagnostic avancés

### `apt-cache` : interroger le cache local

```bash
# Rechercher un paquet par nom ou description
apt-cache search nginx

# Afficher les détails complets d'un paquet
apt-cache show nginx

# Afficher les dépendances
apt-cache depends nginx

# Afficher les paquets qui dépendent de ce paquet (dépendances inverses)
apt-cache rdepends nginx

# Afficher les statistiques du cache
apt-cache stats
```

### `dpkg-query` : interroger la base dpkg

```bash
# Lister les fichiers installés par un paquet
dpkg-query -L nginx

# Savoir quel paquet a installé un fichier
dpkg-query -S /etc/nginx/nginx.conf

# Afficher l'état d'un paquet
dpkg-query -s nginx
```

> 💡 **Très utile** : `dpkg-query -S /chemin/vers/fichier` pour savoir quel paquet "possède" un fichier donné. Pratique pour déboguer des conflits.

### `apt-file` : trouver quel paquet contient un fichier

```bash
sudo apt install apt-file
sudo apt-file update

# Trouver quel paquet fournit un fichier
apt-file search libssl.so
apt-file search /usr/bin/curl
```

> 💡 **Cas d'usage** : tu as une erreur `error while loading shared libraries: libfoo.so.1`. `apt-file search libfoo.so.1` te dit quel paquet installer.

### Vérifier l'intégrité des paquets installés

```bash
# Vérifier qu'aucun paquet n'est cassé
sudo dpkg --audit

# Reconfigurer un paquet mal configuré
sudo dpkg --reconfigure nginx

# Forcer la reconfiguration de tous les paquets
sudo dpkg --configure -a
```

### `debsums` : vérifier l'intégrité des fichiers installés

```bash
sudo apt install debsums

# Vérifier tous les fichiers d'un paquet
debsums nginx

# Vérifier tout le système (long !)
debsums --all
```

> 💡 **Cas de sécurité** : si tu suspectes une compromission, `debsums` peut détecter des fichiers système modifiés par rapport à leur état d'origine.

---

## 13. Checklist du SysAdmin avant toute opération

### ✅ Avant `apt install`

```
□ sudo apt update                          # listes à jour
□ apt show <paquet>                        # inspecter le paquet
□ apt-cache policy <paquet>               # vérifier la source et version
□ sudo apt install --dry-run <paquet>     # simuler
□ Analyser : que va-t-il être installé/supprimé ?
□ Vérifier l'espace disque : df -h
□ Si production : faire un snapshot/backup
□ sudo apt install <paquet>               # procéder
```

### ✅ Avant `apt upgrade`

```
□ sudo apt update
□ apt list --upgradable                   # voir ce qui va changer
□ Identifier les paquets critiques : kernel, openssl, libc6, docker
□ sudo apt upgrade --dry-run              # simuler
□ Analyser toutes les lignes "Remv"
□ Si kernel mis à jour : planifier un reboot
□ Si production : fenêtre de maintenance planifiée
□ Si production : snapshot/backup avant
□ sudo apt upgrade
□ sudo reboot (si nécessaire)
□ Vérifier que les services sont up : systemctl status
```

### ✅ Avant `apt purge` / `apt remove`

```
□ sudo apt purge --dry-run <paquet>       # simuler
□ Vérifier la cascade de suppressions
□ Sauvegarder les fichiers de config si besoin (/etc/<paquet>/)
□ Vérifier les dépendances inverses : apt-cache rdepends <paquet>
□ sudo apt purge <paquet>
□ sudo apt autoremove --dry-run           # voir les orphelins
□ sudo apt autoremove
```

### ✅ Commandes de diagnostic en cas de problème

```bash
# Système cassé ? Paquets à moitié installés ?
sudo dpkg --configure -a
sudo apt install -f                  # -f = --fix-broken

# Nettoyer le cache
sudo apt clean                       # supprime tous les .deb en cache
sudo apt autoclean                   # supprime seulement les .deb obsolètes

# Vérifier l'espace disque
df -h
du -sh /var/cache/apt/archives/      # taille du cache apt
```

### ✅ Bonnes pratiques générales

```
□ Ne jamais faire apt upgrade sans apt update avant
□ Toujours --dry-run sur un serveur de production
□ Logger toutes tes actions (history.log le fait automatiquement)
□ Utiliser apt-mark hold sur les paquets critiques versionnés
□ Éviter les PPAs sur les serveurs de production
□ Utiliser apt-mark manual sur les outils que tu installe volontairement
□ Faire des snapshots avant les mises à jour majeures
□ Tester sur staging avant production
□ Garder un œil sur les mises à jour de sécurité : /var/log/apt/history.log
```

---

## 14. Glossaire

| Terme | Définition |
|-------|-----------|
| **APT** | Advanced Package Tool — gestionnaire de paquets haut niveau de Debian/Ubuntu |
| **dpkg** | Debian Package Manager — outil bas niveau qui installe les fichiers `.deb` |
| **Repository (dépôt)** | Serveur distant qui héberge des paquets `.deb` |
| **PPA** | Personal Package Archive — dépôt non-officiel hébergé sur Launchpad |
| **Dépendance** | Paquet nécessaire au fonctionnement d'un autre paquet |
| **Orphelin** | Paquet installé automatiquement comme dépendance mais dont le paquet parent a été supprimé |
| **Cache APT** | Copie locale des fichiers `.deb` téléchargés, dans `/var/cache/apt/archives/` |
| **GPG** | GNU Privacy Guard — système de signature cryptographique pour vérifier l'authenticité des paquets |
| **Hold** | Verrou APT qui empêche la mise à jour d'un paquet spécifique |
| **Dry-run** | Simulation d'une opération sans rien modifier sur le système |
| **sources.list** | Fichier listant les dépôts APT configurés sur le système |
| **InRelease** | Fichier signé GPG qui décrit le contenu d'un dépôt |
| **dist-upgrade** | Alias de `full-upgrade` — mise à jour pouvant installer ou supprimer des paquets |
| **autoremove** | Commande qui supprime les paquets orphelins devenus inutiles |
| **purge** | Suppression d'un paquet incluant ses fichiers de configuration |
| **debsums** | Outil qui vérifie l'intégrité des fichiers installés par rapport aux checksums officiels |
| **apt-file** | Outil qui permet de chercher quel paquet fournit un fichier donné |

---

*Document généré pour la formation Cloud/AWS — Phase Linux SysAdmin*  
*Référence : Debian/Ubuntu package management best practices*
