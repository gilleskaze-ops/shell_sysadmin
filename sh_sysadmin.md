# Le Shell `sh` — Guide du Sysadmin Linux

> `sh` = Bourne Shell — le standard POSIX, disponible sur **tous** les systèmes Unix/Linux.
> Tout ce qui est écrit ici fonctionne sur n'importe quel serveur, sans exception.

---

## 1. Le Shebang — Toujours en première ligne

```sh
#!/bin/sh
```

Dit au système : "exécute ce fichier avec `sh`".
Sans ça, le système ne sait pas comment interpréter ton script.

```sh
chmod +x mon_script.sh   # Rendre le script exécutable
./mon_script.sh          # L'exécuter
```

---

## 2. Variables

```sh
# Déclaration — PAS d'espaces autour du =
NOM="Alice"
AGE=30

# Lecture
echo $NOM
echo "Bonjour $NOM, tu as $AGE ans"

# Accolades — bonne pratique pour éviter l'ambiguïté
echo "${NOM}_backup"    # → Alice_backup
echo "$NOM_backup"      # → vide ! Bash cherche la variable NOM_backup

# Variables en lecture seule
readonly VERSION="1.0"

# Supprimer une variable
unset NOM
```

### Variables spéciales — à connaître absolument

```sh
$0    # Nom du script
$1    # Premier argument passé au script
$2    # Deuxième argument
$@    # Tous les arguments
$#    # Nombre d'arguments
$?    # Code de retour de la dernière commande (0 = succès)
$$    # PID du script en cours
$!    # PID du dernier processus lancé en arrière-plan
```

```sh
# Exemple concret
./mon_script.sh serveur1 8080

# Dans le script :
echo $0   # → ./mon_script.sh
echo $1   # → serveur1
echo $2   # → 8080
echo $#   # → 2
```

---

## 3. Les Guillemets — Règle fondamentale

```sh
# Guillemets doubles " " → les variables sont interprétées
NOM="Alice"
echo "Bonjour $NOM"    # → Bonjour Alice

# Guillemets simples ' ' → TOUT est littéral, rien n'est interprété
echo 'Bonjour $NOM'    # → Bonjour $NOM

# Backtick ` ` ou $() → exécute une commande et récupère le résultat
DATE=`date`            # Ancienne syntaxe
DATE=$(date)           # Syntaxe moderne, préférable
echo "Nous sommes le $DATE"
```

---

## 4. Arithmétique

```sh
# Dans sh, on utilise expr (pas de $(()) garanti en sh pur)
expr 5 + 3        # → 8
expr 10 - 4       # → 6
expr 6 \* 7       # → 42  (attention : * doit être échappé)
expr 15 / 4       # → 3   (division entière)
expr 15 % 4       # → 3   (modulo)

# Stocker le résultat
RESULTAT=$(expr 5 + 3)
echo $RESULTAT    # → 8
```

---

## 5. Conditions — `if / elif / else`

```sh
#!/bin/sh

SCORE=75

if [ $SCORE -ge 90 ]; then
    echo "Excellent"
elif [ $SCORE -ge 70 ]; then
    echo "Bien"
else
    echo "À améliorer"
fi
```

### Les opérateurs de comparaison — POSIX

```sh
# Nombres
-eq    # égal        (equal)
-ne    # différent   (not equal)
-lt    # inférieur   (less than)
-le    # inf ou égal (less or equal)
-gt    # supérieur   (greater than)
-ge    # sup ou égal (greater or equal)

# Chaînes de caractères
=      # égal
!=     # différent
-z     # chaîne vide (zero length)
-n     # chaîne non vide (non-zero)

# Fichiers — très utilisés en sysadmin
-f     # est un fichier régulier
-d     # est un dossier (directory)
-e     # existe (exist)
-r     # est lisible (readable)
-w     # est modifiable (writable)
-x     # est exécutable
-s     # existe et n'est pas vide (size > 0)
```

```sh
# Exemples sysadmin concrets
if [ -f "/etc/nginx/nginx.conf" ]; then
    echo "Nginx est installé"
fi

if [ -d "/var/log/app" ]; then
    echo "Le dossier de logs existe"
else
    mkdir -p /var/log/app
fi

if [ -z "$1" ]; then
    echo "Erreur : aucun argument fourni"
    exit 1
fi
```

### Opérateurs logiques

```sh
# ET logique
if [ $AGE -ge 18 ] && [ $AGE -le 65 ]; then   # bash
if [ $AGE -ge 18 -a $AGE -le 65 ]; then        # sh POSIX

# OU logique
if [ $STATUT = "actif" ] || [ $STATUT = "pending" ]; then   # bash
if [ $STATUT = "actif" -o $STATUT = "pending" ]; then       # sh POSIX
```

---

## 6. Boucles

### `for` — itérer sur une liste

```sh
# Sur une liste fixe
for SERVEUR in web01 web02 web03; do
    echo "Vérification de $SERVEUR..."
    ping -c 1 $SERVEUR
done

# Sur des fichiers
for FICHIER in /var/log/*.log; do
    echo "Taille de $FICHIER : $(du -sh $FICHIER)"
done

# Avec seq (génère des nombres)
for i in $(seq 1 10); do
    echo "Ligne $i"
done
```

### `while` — tant que la condition est vraie

```sh
# Boucle classique
COMPTEUR=1
while [ $COMPTEUR -le 5 ]; do
    echo "Iteration $COMPTEUR"
    COMPTEUR=$(expr $COMPTEUR + 1)
done

# Lire un fichier ligne par ligne — très utile en sysadmin
while read LIGNE; do
    echo "Traitement : $LIGNE"
done < /etc/hosts

# Boucle infinie avec sortie conditionnelle
while true; do
    if ping -c 1 monserveur.com > /dev/null 2>&1; then
        echo "Serveur en ligne"
        break
    fi
    echo "En attente..."
    sleep 5
done
```

### `until` — jusqu'à ce que la condition soit vraie (inverse de while)

```sh
until [ -f "/tmp/deploy_done" ]; do
    echo "Déploiement en cours..."
    sleep 2
done
echo "Déploiement terminé !"
```

---

## 7. `case` — Alternative au if/elif

```sh
#!/bin/sh

ACTION=$1

case "$ACTION" in
    start)
        echo "Démarrage du service..."
        ;;
    stop)
        echo "Arrêt du service..."
        ;;
    restart)
        echo "Redémarrage..."
        ;;
    status|info)           # Plusieurs valeurs possibles avec |
        echo "Statut du service"
        ;;
    *)                     # Défaut (comme else)
        echo "Usage: $0 {start|stop|restart|status}"
        exit 1
        ;;
esac
```

---

## 8. Fonctions

```sh
#!/bin/sh

# Définition
verifier_service() {
    SERVICE=$1
    if systemctl is-active --quiet "$SERVICE"; then
        echo "$SERVICE est actif"
        return 0    # Succès
    else
        echo "$SERVICE est inactif"
        return 1    # Échec
    fi
}

# Appel
verifier_service nginx
verifier_service mysql

# Récupérer le code de retour
verifier_service nginx
if [ $? -eq 0 ]; then
    echo "Tout va bien"
fi
```

---

## 9. Redirections et Pipes

```sh
# Redirection de sortie
echo "log entry" > fichier.txt      # Écrase
echo "log entry" >> fichier.txt     # Ajoute (append)

# Redirection des erreurs
commande 2> erreurs.txt             # Stderr vers fichier
commande > out.txt 2>&1            # Stdout ET stderr vers même fichier
commande > /dev/null 2>&1          # Silence total (jeter toute sortie)

# Entrée depuis un fichier
commande < fichier.txt

# Pipes — connecter des commandes
cat access.log | grep "500" | wc -l
ps aux | grep nginx | grep -v grep
```

---

## 10. Commandes essentielles du Sysadmin

### Fichiers et répertoires
```sh
ls -la              # Liste avec détails et fichiers cachés
find / -name "*.log" -mtime +7    # Fichiers .log de plus de 7 jours
find /var -size +100M             # Fichiers de plus de 100MB
du -sh /var/log/*   # Taille de chaque sous-dossier
df -h               # Espace disque des partitions
```

### Processus
```sh
ps aux              # Tous les processus
ps aux | grep nginx # Chercher un processus
kill -15 1234       # Arrêt propre (SIGTERM)
kill -9 1234        # Arrêt forcé (SIGKILL)
top                 # Supervision en temps réel
```

### Réseau
```sh
ping -c 4 google.com           # Test connectivité
curl -I https://monsite.com    # Headers HTTP
wget https://url/fichier       # Télécharger un fichier
ss -tulnp                      # Ports ouverts et services
netstat -tulnp                 # Alternative à ss
```

### Logs
```sh
tail -f /var/log/syslog        # Suivre un log en temps réel
tail -n 100 /var/log/nginx/error.log   # 100 dernières lignes
grep "ERROR" /var/log/app.log  # Chercher des erreurs
journalctl -u nginx -f         # Logs systemd en temps réel
```

### Permissions
```sh
chmod 755 script.sh     # rwxr-xr-x
chmod +x script.sh      # Ajouter l'exécution
chown user:group fichier
sudo commande            # Exécuter en root
```

---

## 11. Gestion des erreurs — Indispensable en production

```sh
#!/bin/sh

# Arrêter le script à la première erreur
set -e

# Afficher chaque commande avant de l'exécuter (debug)
set -x

# Les deux combinés
set -ex

# Vérifier le succès d'une commande
cp fichier.txt /backup/ || echo "Erreur lors de la copie !"

# Exit codes — convention universelle
# 0 = succès
# 1 = erreur générale
# 2 = mauvaise utilisation du script
exit 0
```

---

## 12. Script type Sysadmin — Exemple complet

```sh
#!/bin/sh
# surveillance_disque.sh
# Alerte si l'utilisation disque dépasse un seuil

SEUIL=80
DATE=$(date '+%Y-%m-%d %H:%M:%S')
LOG="/var/log/surveillance.log"

verifier_disque() {
    PARTITION=$1
    USAGE=$(df "$PARTITION" | tail -1 | awk '{print $5}' | tr -d '%')

    if [ "$USAGE" -ge "$SEUIL" ]; then
        echo "[$DATE] ALERTE : $PARTITION utilisation ${USAGE}%" >> "$LOG"
        return 1
    else
        echo "[$DATE] OK : $PARTITION utilisation ${USAGE}%" >> "$LOG"
        return 0
    fi
}

# Vérifier les partitions principales
for PARTITION in / /var /home; do
    if [ -d "$PARTITION" ]; then
        verifier_disque "$PARTITION"
    fi
done
```

---

## Résumé — Ce que tout Sysadmin doit maîtriser en `sh`

| Concept | Pourquoi c'est crucial |
|---|---|
| Variables et `$?` | Savoir si une commande a réussi ou échoué |
| Guillemets `' "` | Éviter les bugs d'interprétation |
| `if / case` | Logique conditionnelle dans les scripts |
| Boucles `for / while` | Automatiser des tâches répétitives |
| Fonctions | Organiser et réutiliser le code |
| Redirections `> >> 2>` | Gérer les logs et les erreurs |
| `set -e` | Ne jamais laisser un script continuer après une erreur |
| Commandes système | `ps`, `df`, `find`, `grep`, `tail`, `kill` |

---

> **Prochaine étape →** `bash_sysadmin.md` : tout ce que `sh` ne peut pas faire
