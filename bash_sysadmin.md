# Le Shell `bash` — Guide du Sysadmin Linux

> `bash` = Bourne Again Shell — tout ce que `sh` fait, **plus** des fonctionnalités
> puissantes pour l'automatisation, le scripting avancé et la productivité au quotidien.
> Shebang à toujours mettre : `#!/bin/bash`

---

## 1. Ce que bash ajoute par rapport à sh — Vue d'ensemble

```
sh   →  Variables, if, for, while, fonctions, redirections
bash →  Tout sh + tableaux, arithmetic $(( )), [[ ]], 
         process substitution, here-strings, brace expansion,
         fonctions avancées, history, autocomplétion...
```

---

## 2. Arithmétique — La vraie syntaxe bash

En `sh` on utilisait `expr`. En `bash`, on a `$(( ))` — bien plus lisible et puissant.

```bash
# Opérations de base
echo $((5 + 3))       # → 8
echo $((10 - 4))      # → 6
echo $((6 * 7))       # → 42  (pas besoin d'échapper *)
echo $((15 / 4))      # → 3   (division entière)
echo $((15 % 4))      # → 3   (modulo)
echo $((2 ** 8))      # → 256 (puissance — inexistant en sh !)

# Stocker dans une variable
RESULTAT=$((100 / 4))
echo $RESULTAT         # → 25

# Incrémenter — très utilisé dans les boucles
COMPTEUR=0
((COMPTEUR++))         # → 1
((COMPTEUR+=5))        # → 6
((COMPTEUR*=2))        # → 12

# Calcul dans une condition directement
if (( $SCORE >= 90 )); then
    echo "Excellent"
fi
```

---

## 3. Tableaux (Arrays) — Inexistant en sh

C'est l'une des fonctionnalités les plus importantes de bash pour le sysadmin.

```bash
# Déclaration
SERVEURS=("web01" "web02" "web03" "db01")
PORTS=(80 443 3306 5432)

# Accès par index (commence à 0)
echo ${SERVEURS[0]}    # → web01
echo ${SERVEURS[2]}    # → web03

# Modifier un élément
SERVEURS[1]="web02-backup"

# Ajouter un élément
SERVEURS+=("cache01")

# Nombre d'éléments
echo ${#SERVEURS[@]}   # → 5

# Tous les éléments
echo ${SERVEURS[@]}    # → web01 web02-backup web03 db01 cache01

# Itérer sur un tableau — pattern ultra-courant en sysadmin
for SERVEUR in "${SERVEURS[@]}"; do
    echo "Ping de $SERVEUR..."
    ping -c 1 "$SERVEUR" > /dev/null 2>&1 && echo "OK" || echo "KO"
done

# Tableau d'indices
echo ${!SERVEURS[@]}   # → 0 1 2 3 4
```

### Tableaux associatifs (clé → valeur) — comme un dictionnaire

```bash
# Déclaration obligatoire avec declare -A
declare -A CONFIG
CONFIG["host"]="192.168.1.10"
CONFIG["port"]="5432"
CONFIG["db"]="production"
CONFIG["user"]="admin"

# Accès
echo ${CONFIG["host"]}    # → 192.168.1.10
echo ${CONFIG["port"]}    # → 5432

# Itérer sur les clés
for CLE in "${!CONFIG[@]}"; do
    echo "$CLE = ${CONFIG[$CLE]}"
done

# Cas concret : mapper des services à leurs ports
declare -A SERVICES
SERVICES["nginx"]=80
SERVICES["postgresql"]=5432
SERVICES["redis"]=6379

for SERVICE in "${!SERVICES[@]}"; do
    PORT=${SERVICES[$SERVICE]}
    ss -tulnp | grep ":$PORT" > /dev/null && \
        echo "$SERVICE (port $PORT) : ACTIF" || \
        echo "$SERVICE (port $PORT) : INACTIF"
done
```

---

## 4. Les doubles crochets `[[ ]]` — La condition bash

En `sh` on utilisait `[ ]`. En bash, `[[ ]]` est plus puissant et plus sûr.

```bash
# Comparaison de chaînes avec patterns (glob)
FICHIER="access_log_2024.txt"
if [[ "$FICHIER" == *.txt ]]; then
    echo "C'est un fichier texte"
fi

if [[ "$FICHIER" == access_* ]]; then
    echo "C'est un log d'accès"
fi

# Regex — impossible en sh !
IP="192.168.1.42"
if [[ "$IP" =~ ^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Adresse IP valide"
fi

# Pas besoin de guillemets pour éviter les erreurs
VARIABLE=""
if [[ -z $VARIABLE ]]; then    # Sûr même sans guillemets
    echo "Variable vide"
fi
# En sh : if [ -z "$VARIABLE" ]  ← les guillemets étaient obligatoires

# ET / OU plus lisibles
if [[ $AGE -ge 18 && $AGE -le 65 ]]; then    # bash
    echo "En âge de travailler"
fi

if [[ $STATUT == "actif" || $STATUT == "standby" ]]; then
    echo "Serveur disponible"
fi
```

### Comparaison `[ ]` vs `[[ ]]`

```
[ ]   →  POSIX, sh compatible, plus limité, guillemets obligatoires
[[ ]] →  bash uniquement, supporte regex (=~), glob (==), && ||
          plus sûr avec les variables vides ou avec espaces
```

---

## 5. Brace Expansion — Générer des séquences

```bash
# Séquences de nombres
echo {1..10}           # → 1 2 3 4 5 6 7 8 9 10
echo {01..10}          # → 01 02 03 04 05 06 07 08 09 10
echo {1..10..2}        # → 1 3 5 7 9  (pas de 2)

# Séquences de lettres
echo {a..z}            # → a b c d ... z
echo {A..Z}

# Combinaisons — très puissant !
echo serveur{1..5}     # → serveur1 serveur2 serveur3 serveur4 serveur5
echo web{01..03}.{prod,dev}.com
# → web01.prod.com web01.dev.com web02.prod.com ...

# Créer des dossiers en masse
mkdir -p /var/log/app/{2023,2024}/{01,02,03,04,05,06,07,08,09,10,11,12}

# Sauvegarder un fichier rapidement
cp nginx.conf{,.bak}   # → copie nginx.conf vers nginx.conf.bak
```

---

## 6. Substitution de commandes et Process Substitution

```bash
# Substitution classique — capturer la sortie d'une commande
DATE=$(date '+%Y-%m-%d')
NB_PROCESSUS=$(ps aux | wc -l)
IP_LOCALE=$(hostname -I | awk '{print $1}')

echo "Rapport du $DATE — $NB_PROCESSUS processus actifs sur $IP_LOCALE"

# Process substitution <() — traiter la sortie comme un fichier
# Utile pour comparer deux listes sans fichiers temporaires
diff <(ls /etc/nginx/sites-available/) <(ls /etc/nginx/sites-enabled/)

# Comparer les utilisateurs de deux serveurs
diff <(ssh serveur1 cat /etc/passwd) <(ssh serveur2 cat /etc/passwd)
```

---

## 7. Here-String et Here-Document

```bash
# Here-String <<< — passer une chaîne comme entrée
grep "root" <<< "root:x:0:0:root:/root:/bin/bash"

# Compter des mots dans une chaîne directement
wc -w <<< "un deux trois quatre"    # → 4

# Here-Document << — bloc de texte multiligne
# Très utilisé pour générer des configs ou envoyer des emails
cat << EOF > /etc/nginx/sites-available/monsite.conf
server {
    listen 80;
    server_name monsite.com;
    root /var/www/monsite;
    index index.html;
}
EOF

# Avec tiret <<- pour ignorer les tabulations
mysql -u root -p << EOF
    CREATE DATABASE IF NOT EXISTS production;
    CREATE USER 'appuser'@'localhost' IDENTIFIED BY 'motdepasse';
    GRANT ALL ON production.* TO 'appuser'@'localhost';
EOF
```

---

## 8. Manipulation de chaînes — Puissant et unique à bash

```bash
CHAINE="Hello World Linux 2024"

# Longueur
echo ${#CHAINE}                  # → 22

# Extraction de sous-chaîne ${var:debut:longueur}
echo ${CHAINE:6:5}               # → World
echo ${CHAINE:6}                 # → World Linux 2024

# Supprimer un préfixe
FICHIER="access_log_2024.txt"
echo ${FICHIER#access_}          # → log_2024.txt  (supprime le plus court)
echo ${FICHIER##*_}              # → 2024.txt       (supprime le plus long)

# Supprimer un suffixe
echo ${FICHIER%.txt}             # → access_log_2024
echo ${FICHIER%%_*}              # → access

# Remplacer
echo ${CHAINE/Linux/Unix}        # → Hello World Unix 2024  (première occurrence)
echo ${CHAINE//l/L}              # → HeLLo WorLd Linux 2024 (toutes les occurrences)

# Changer la casse
echo ${CHAINE,,}                 # → hello world linux 2024  (tout en minuscule)
echo ${CHAINE^^}                 # → HELLO WORLD LINUX 2024  (tout en majuscule)
echo ${CHAINE^}                  # → Hello world linux 2024  (première lettre)

# Valeur par défaut si variable vide
echo ${VAR:-"valeur_defaut"}     # Si VAR vide → valeur_defaut
echo ${VAR:="valeur_defaut"}     # Si VAR vide → assigne ET retourne valeur_defaut
```

---

## 9. Fonctions avancées

```bash
#!/bin/bash

# Fonction avec valeur de retour via echo
obtenir_ip() {
    local INTERFACE=$1    # 'local' → variable locale à la fonction uniquement
    ip addr show "$INTERFACE" | grep "inet " | awk '{print $2}' | cut -d/ -f1
}

# Appel et capture du résultat
IP=$(obtenir_ip eth0)
echo "IP de eth0 : $IP"

# Fonction avec plusieurs valeurs de retour
analyser_log() {
    local FICHIER=$1
    local -a RESULTATS    # tableau local

    RESULTATS[0]=$(grep -c "200" "$FICHIER")   # Nombre de 200
    RESULTATS[1]=$(grep -c "404" "$FICHIER")   # Nombre de 404
    RESULTATS[2]=$(grep -c "500" "$FICHIER")   # Nombre de 500

    echo "${RESULTATS[@]}"
}

# Récupérer les résultats dans un tableau
read -ra STATS <<< "$(analyser_log access_log.txt)"
echo "200: ${STATS[0]} | 404: ${STATS[1]} | 500: ${STATS[2]}"

# Fonction de logging — pattern standard en production
log() {
    local NIVEAU=$1
    local MESSAGE=$2
    local TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    echo "[$TIMESTAMP] [$NIVEAU] $MESSAGE" | tee -a /var/log/mon_script.log
}

log "INFO"  "Démarrage du script"
log "WARN"  "Espace disque faible"
log "ERROR" "Connexion à la base échouée"
```

---

## 10. Gestion des erreurs — Niveau production

```bash
#!/bin/bash

# Les 3 options essentielles — à mettre en début de tout script de prod
set -e          # Arrêt immédiat si une commande échoue
set -u          # Erreur si une variable non définie est utilisée
set -o pipefail # Un pipe échoue si n'importe quelle commande du pipe échoue

# Ou en une ligne
set -euo pipefail

# Trap — exécuter du code quoi qu'il arrive (cleanup, logs)
cleanup() {
    echo "Nettoyage en cours..."
    rm -f /tmp/mon_script_lock
    log "INFO" "Script terminé"
}
trap cleanup EXIT        # Appelé à la sortie (normale ou erreur)
trap cleanup INT TERM    # Appelé si Ctrl+C ou kill

# Gestion fine des erreurs
copier_backup() {
    local SOURCE=$1
    local DEST=$2

    if ! cp "$SOURCE" "$DEST"; then
        log "ERROR" "Échec de la copie de $SOURCE vers $DEST"
        return 1
    fi
    log "INFO" "Copie réussie : $SOURCE → $DEST"
    return 0
}
```

---

## 11. `read` — Interaction et parsing

```bash
# Lire une entrée utilisateur
read -p "Entrez le nom du serveur : " SERVEUR
read -s -p "Entrez le mot de passe : " PASSWORD   # -s = silencieux
echo ""

# Lire avec timeout
read -t 10 -p "Confirmer ? (o/n) : " REPONSE
if [ $? -ne 0 ]; then
    echo "Timeout — action annulée"
fi

# Parser une ligne en plusieurs variables
echo "192.168.1.1 web01 actif" | while read IP NOM STATUT; do
    echo "IP=$IP, Nom=$NOM, Statut=$STATUT"
done

# Lire un fichier CSV
while IFS=',' read -r IP PORT SERVICE; do
    echo "Service $SERVICE → $IP:$PORT"
done < services.csv
```

---

## 12. Parallélisme — Lancer des tâches en arrière-plan

```bash
#!/bin/bash

# Lancer une commande en arrière-plan avec &
ping -c 3 serveur1 &
ping -c 3 serveur2 &
ping -c 3 serveur3 &

# Attendre que tous les processus arrière-plan se terminent
wait
echo "Tous les pings terminés"

# Pattern avancé : contrôler le nombre de jobs parallèles
MAX_JOBS=5
COMPTEUR=0

for SERVEUR in "${SERVEURS[@]}"; do
    verifier_serveur "$SERVEUR" &
    ((COMPTEUR++))

    # Quand on atteint le max, attendre avant de continuer
    if (( COMPTEUR >= MAX_JOBS )); then
        wait
        COMPTEUR=0
    fi
done
wait    # Attendre les derniers jobs
```

---

## 13. `declare` — Typer les variables

```bash
declare -i NOMBRE=42      # Entier (integer)
declare -r CONSTANTE=100  # Lecture seule (readonly)
declare -a TABLEAU        # Tableau indexé (array)
declare -A DICO           # Tableau associatif
declare -x VAR_ENV        # Exporter comme variable d'environnement
declare -l minuscule      # Toujours en minuscule
declare -u MAJUSCULE      # Toujours en majuscule

# Voir toutes les variables et fonctions définies
declare -p               # Variables
declare -f               # Fonctions
```

---

## 14. Script complet — Rapport de santé serveur

```bash
#!/bin/bash
# health_check.sh — Rapport de santé d'un serveur Linux
set -euo pipefail

# Configuration
SEUIL_CPU=80
SEUIL_MEM=85
SEUIL_DISQUE=90
RAPPORT="/tmp/health_$(date +%Y%m%d_%H%M%S).txt"
declare -A RESULTATS

# Logging
log() {
    echo "[$(date '+%H:%M:%S')] $1" | tee -a "$RAPPORT"
}

# Vérification CPU
verifier_cpu() {
    local CPU_USAGE
    CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1 | cut -d',' -f1)
    RESULTATS["cpu"]=$CPU_USAGE
    if (( $(echo "$CPU_USAGE > $SEUIL_CPU" | bc -l) )); then
        log "⚠ CPU : ${CPU_USAGE}% (seuil : ${SEUIL_CPU}%)"
        return 1
    fi
    log "✓ CPU : ${CPU_USAGE}%"
}

# Vérification mémoire
verifier_memoire() {
    local MEM_USAGE
    MEM_USAGE=$(free | grep Mem | awk '{printf "%.0f", $3/$2 * 100}')
    RESULTATS["memoire"]=$MEM_USAGE
    if (( MEM_USAGE > SEUIL_MEM )); then
        log "⚠ Mémoire : ${MEM_USAGE}% (seuil : ${SEUIL_MEM}%)"
        return 1
    fi
    log "✓ Mémoire : ${MEM_USAGE}%"
}

# Vérification disques
verifier_disques() {
    local ALERTE=0
    while IFS= read -r LIGNE; do
        local USAGE PARTITION
        USAGE=$(echo "$LIGNE" | awk '{print $5}' | tr -d '%')
        PARTITION=$(echo "$LIGNE" | awk '{print $6}')
        if (( USAGE > SEUIL_DISQUE )); then
            log "⚠ Disque $PARTITION : ${USAGE}% (seuil : ${SEUIL_DISQUE}%)"
            ALERTE=1
        else
            log "✓ Disque $PARTITION : ${USAGE}%"
        fi
    done < <(df -h | tail -n +2)
    return $ALERTE
}

# Vérification services
SERVICES_CRITIQUES=("nginx" "ssh" "cron")
verifier_services() {
    for SERVICE in "${SERVICES_CRITIQUES[@]}"; do
        if systemctl is-active --quiet "$SERVICE" 2>/dev/null; then
            log "✓ Service $SERVICE : actif"
        else
            log "✗ Service $SERVICE : INACTIF"
        fi
    done
}

# Rapport final
generer_rapport() {
    log "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    log "RAPPORT DE SANTÉ — $(hostname)"
    log "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    verifier_cpu    || true
    verifier_memoire || true
    verifier_disques || true
    verifier_services
    log "━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━"
    log "Rapport sauvegardé : $RAPPORT"
}

generer_rapport
```

---

## 15. Résumé — sh vs bash en un coup d'œil

| Fonctionnalité | `sh` | `bash` |
|---|---|---|
| Variables basiques | ✓ | ✓ |
| if / for / while | ✓ | ✓ |
| Fonctions | ✓ | ✓ |
| Redirections | ✓ | ✓ |
| Tableaux indexés | ✗ | ✓ |
| Tableaux associatifs | ✗ | ✓ |
| Arithmétique `$(( ))` | partiel | ✓ |
| Double crochets `[[ ]]` | ✗ | ✓ |
| Regex `=~` | ✗ | ✓ |
| Brace expansion `{1..10}` | ✗ | ✓ |
| Manipulation de chaînes | basique | ✓ |
| `declare` / typage | ✗ | ✓ |
| `set -u` (var non définie) | ✗ | ✓ |
| `pipefail` | ✗ | ✓ |
| Parallélisme avec `wait` | partiel | ✓ |

---

## Ce qu'il faut retenir

```
sh   →  Portabilité maximale, serveurs inconnus, scripts simples
bash →  Ton outil quotidien, scripts complexes, automatisation avancée

En production :  set -euo pipefail  en début de TOUT script bash
En sysadmin :    maîtriser les tableaux, [[ ]], et la manipulation de chaînes
En Cloud :       ces scripts deviennent tes scripts de déploiement,
                 de monitoring, et d'infrastructure
```

---

> **Prochaine étape →** Les permissions Linux, `chmod`, `chown`, `sudo` et `sudoers`
> ou Docker — la conteneurisation comme passerelle vers le Cloud
