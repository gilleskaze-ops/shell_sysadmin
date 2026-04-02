# Outils du Sysadmin & AWS Solutions Architect

> Deux catégories : les **outils Unix** (le shell les connecte entre eux)
> et les **outils Cloud/DevOps** (l'écosystème moderne du Solutions Architect).
> Pour chaque outil : ce qu'il fait, pourquoi il compte, un use case réel.

---

## PARTIE 1 — Les Outils Unix "Connectables"

> Philosophie Unix : chaque outil fait **une seule chose, très bien**.
> Le shell les connecte avec des pipes `|` pour créer des pipelines puissants.

---

### `grep` — Chercher des patterns

```bash
# Syntaxe
grep [options] "pattern" fichier

# Use case 1 : trouver toutes les erreurs 500 dans un log nginx
grep "\" 500 " /var/log/nginx/access.log

# Use case 2 : chercher récursivement un mot dans tous les fichiers de config
grep -r "max_connections" /etc/

# Use case 3 : afficher les lignes qui NE contiennent PAS un pattern
grep -v "200" access.log    # Tout sauf les succès

# Use case 4 : compter les occurrences
grep -c "ERROR" /var/log/app.log

# Use case 5 : afficher le numéro de ligne
grep -n "CRITICAL" /var/log/syslog

# Use case 6 : regex étendue
grep -E "40[0-9]|500" access.log    # Tous les codes 4xx et 500
```

**Options indispensables**

| Option | Rôle |
|---|---|
| `-i` | Insensible à la casse |
| `-r` | Récursif dans les dossiers |
| `-v` | Inverser (exclure le pattern) |
| `-c` | Compter les lignes qui matchent |
| `-n` | Afficher les numéros de ligne |
| `-o` | Afficher seulement la partie qui matche |
| `-E` | Regex étendue (ERE) |
| `-l` | Afficher seulement les noms de fichiers |

---

### `awk` — Extraire et manipuler des colonnes

> `awk` traite chaque ligne comme un tableau de colonnes séparées par un délimiteur.
> C'est un mini-langage de programmation à lui seul.

```bash
# Syntaxe
awk '{action}' fichier
awk -F "délimiteur" '{action}' fichier

# Les variables automatiques
$0   # La ligne entière
$1   # Première colonne
$2   # Deuxième colonne
$NF  # Dernière colonne (Number of Fields)
NR   # Numéro de la ligne courante

# Use case 1 : extraire les IPs d'un log nginx (colonne 1)
awk '{print $1}' access.log

# Use case 2 : extraire IP et code HTTP (colonnes 1 et 9)
awk '{print $1, $9}' access.log

# Use case 3 : filtrer les lignes avec un code 500
awk '$9 == "500" {print $0}' access.log

# Use case 4 : calculer la taille totale des réponses
awk '{sum += $10} END {print "Total bytes:", sum}' access.log

# Use case 5 : parser un fichier CSV avec un délimiteur personnalisé
awk -F',' '{print $1, $3}' utilisateurs.csv

# Use case 6 : afficher les lignes entre deux patterns
awk '/START/,/END/' fichier.log

# Use case 7 : rapport complet depuis un log
awk '
{
    total++
    if ($9 ~ /^2/) ok++
    if ($9 ~ /^4/) err4xx++
    if ($9 ~ /^5/) err5xx++
}
END {
    print "Total requêtes :", total
    print "Succès 2xx     :", ok
    print "Erreurs 4xx    :", err4xx
    print "Erreurs 5xx    :", err5xx
}' access.log
```

---

### `sed` — Modifier du texte à la volée

> `sed` = stream editor. Il lit ligne par ligne et applique des transformations.
> Indispensable pour modifier des fichiers de config en script.

```bash
# Syntaxe de base
sed 's/ancien/nouveau/' fichier        # Remplace première occurrence par ligne
sed 's/ancien/nouveau/g' fichier       # Remplace toutes les occurrences

# Use case 1 : changer une valeur dans un fichier de config
sed -i 's/max_connections = 100/max_connections = 500/' /etc/mysql/my.cnf
# -i = in-place (modifie le fichier directement)

# Use case 2 : toujours faire un backup avant modification
sed -i.bak 's/port=8080/port=443/' /etc/app/config.properties
# Crée config.properties.bak automatiquement

# Use case 3 : supprimer les lignes vides
sed '/^$/d' fichier.txt

# Use case 4 : supprimer les commentaires
sed '/^#/d' /etc/nginx/nginx.conf

# Use case 5 : afficher seulement les lignes 10 à 20
sed -n '10,20p' access.log

# Use case 6 : insérer une ligne après un pattern
sed '/\[database\]/a host = localhost' config.ini

# Use case 7 : déploiement — personnaliser un template de config
# Template : config.template contient {{APP_NAME}} et {{PORT}}
sed -e 's/{{APP_NAME}}/monapp/g' \
    -e 's/{{PORT}}/8080/g' \
    config.template > config.final
```

---

### `cut` — Découper par délimiteur ou position

```bash
# Use case 1 : extraire le nom d'utilisateur depuis /etc/passwd
cut -d':' -f1 /etc/passwd

# Use case 2 : extraire les 10 premiers caractères de chaque ligne
cut -c1-10 fichier.txt

# Use case 3 : extraire plusieurs colonnes d'un CSV
cut -d',' -f1,3,5 rapport.csv

# Use case 4 : extraire l'extension d'un nom de fichier
echo "access_log_2024.txt" | cut -d'.' -f2    # → txt

# Use case 5 : combiné avec d'autres outils
grep "ERROR" app.log | cut -d' ' -f1,2         # Date + heure des erreurs
```

---

### `sort` et `uniq` — Trier et dédoublonner

```bash
# sort
sort fichier.txt              # Tri alphabétique
sort -n fichier.txt           # Tri numérique
sort -rn fichier.txt          # Tri numérique décroissant
sort -t',' -k3 -n data.csv    # Trier par la 3ème colonne CSV
sort -u fichier.txt           # Trier ET dédoublonner

# uniq (le fichier doit être trié au préalable)
uniq fichier.txt              # Supprimer les doublons consécutifs
uniq -c fichier.txt           # Compter les occurrences
uniq -d fichier.txt           # Afficher seulement les doublons

# Use case classique sysadmin : top 10 des IPs les plus actives
awk '{print $1}' access.log | sort | uniq -c | sort -rn | head -10

# Use case : trouver les utilisateurs connectés plusieurs fois
last | awk '{print $1}' | sort | uniq -c | sort -rn
```

---

### `find` — Chercher des fichiers avec critères

```bash
# Use case 1 : trouver tous les logs de plus de 7 jours
find /var/log -name "*.log" -mtime +7

# Use case 2 : trouver et supprimer les vieux logs (avec confirmation)
find /var/log -name "*.log" -mtime +30 -exec ls -lh {} \;
find /var/log -name "*.log" -mtime +30 -delete

# Use case 3 : trouver les fichiers de plus de 100MB
find / -size +100M -type f 2>/dev/null

# Use case 4 : trouver les fichiers avec des permissions dangereuses
find / -perm -4000 -type f 2>/dev/null    # Fichiers SUID (risque sécurité)
find /var/www -perm -o+w -type f          # Fichiers accessibles en écriture par tous

# Use case 5 : trouver et archiver en une commande
find /var/log -name "*.log" -mtime +7 | xargs tar -czf old_logs.tar.gz

# Use case 6 : trouver les fichiers modifiés dans les dernières 24h
find /etc -mtime -1 -type f
```

---

### `xargs` — Passer des résultats comme arguments

```bash
# Use case 1 : supprimer tous les fichiers .tmp trouvés
find /tmp -name "*.tmp" | xargs rm -f

# Use case 2 : pinguer une liste de serveurs depuis un fichier
cat serveurs.txt | xargs -I {} ping -c 1 {}

# Use case 3 : lancer en parallèle (-P)
cat urls.txt | xargs -P 5 -I {} curl -o /dev/null -s {}
# Télécharge 5 URLs en même temps

# Use case 4 : combiner find et grep pour chercher dans des fichiers
find /etc -name "*.conf" | xargs grep -l "ssl"
# Trouve tous les .conf qui mentionnent ssl
```

---

### `curl` — Requêtes HTTP et test d'APIs

```bash
# Use case 1 : tester qu'un endpoint répond
curl -I https://monsite.com                    # Headers seulement
curl -o /dev/null -s -w "%{http_code}" https://monsite.com   # Code HTTP seul

# Use case 2 : appel API REST avec JSON
curl -X POST https://api.exemple.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer TOKEN" \
  -d '{"name": "Alice", "role": "admin"}'

# Use case 3 : télécharger un fichier
curl -O https://exemple.com/fichier.tar.gz
curl -L -o output.tar.gz https://url-avec-redirect.com   # Suit les redirections

# Use case 4 : tester un health check en boucle
while true; do
    STATUS=$(curl -o /dev/null -s -w "%{http_code}" https://monapp.com/health)
    echo "$(date) — Status: $STATUS"
    [ "$STATUS" != "200" ] && echo "ALERTE — Service dégradé !"
    sleep 30
done

# Use case 5 : uploader vers S3 (sans AWS CLI)
curl -X PUT -T fichier.txt \
  -H "Host: mon-bucket.s3.amazonaws.com" \
  "https://mon-bucket.s3.amazonaws.com/fichier.txt"
```

---

### `ssh` et `scp` — Connexion et transfert distants

```bash
# Connexion de base
ssh user@192.168.1.10
ssh -i ~/.ssh/ma_cle.pem ubuntu@ec2-xxx.compute.amazonaws.com

# Use case 1 : exécuter une commande sur un serveur distant sans s'y connecter
ssh user@serveur "df -h && free -m"

# Use case 2 : exécuter un script local sur un serveur distant
ssh user@serveur 'bash -s' < mon_script_local.sh

# Use case 3 : boucle sur plusieurs serveurs
for SERVEUR in web01 web02 web03; do
    echo "=== $SERVEUR ==="
    ssh user@$SERVEUR "systemctl status nginx | grep Active"
done

# Use case 4 : copier des fichiers vers/depuis un serveur
scp fichier.txt user@serveur:/var/www/html/
scp -r /local/dossier user@serveur:/remote/dossier
scp -i ~/.ssh/cle.pem user@serveur:/etc/nginx/nginx.conf ./backup_nginx.conf

# Use case 5 : tunnel SSH (port forwarding)
# Accéder à une BDD distante comme si elle était locale
ssh -L 5432:localhost:5432 user@serveur-prod
# Maintenant tu peux te connecter sur localhost:5432 → ça pointe vers la prod
```

---

### `rsync` — Synchronisation de fichiers

```bash
# Use case 1 : backup local
rsync -av /var/www/html/ /backup/www/
# -a = archive (préserve permissions, dates, liens)
# -v = verbose

# Use case 2 : backup vers serveur distant
rsync -avz /var/www/html/ user@backup-server:/backups/www/
# -z = compression pendant le transfert

# Use case 3 : synchronisation avec suppression des fichiers obsolètes
rsync -av --delete /source/ /destination/

# Use case 4 : dry-run (voir ce qui serait fait sans faire)
rsync -av --dry-run /source/ /destination/

# Use case 5 : exclure des dossiers
rsync -av --exclude='node_modules' --exclude='.git' /mon-projet/ user@serveur:/deploy/
```

---

### `tar` — Archiver et compresser

```bash
# Créer une archive compressée
tar -czf archive.tar.gz /dossier/    # gzip
tar -cjf archive.tar.bz2 /dossier/  # bzip2 (plus lent, plus compact)

# Extraire
tar -xzf archive.tar.gz
tar -xzf archive.tar.gz -C /destination/

# Use case 1 : backup journalier avec date
tar -czf "/backup/www_$(date +%Y%m%d).tar.gz" /var/www/html/

# Use case 2 : backup et transfert en une commande (sans fichier intermédiaire)
tar -czf - /var/www/ | ssh user@backup "cat > /backup/www_$(date +%Y%m%d).tar.gz"

# Lister le contenu sans extraire
tar -tzf archive.tar.gz
```

---

### `jq` — Parser du JSON en ligne de commande

> Indispensable dès qu'on utilise AWS CLI ou des APIs REST — tout retourne du JSON.

```bash
# Use case 1 : extraire les IPs privées de toutes les instances EC2
aws ec2 describe-instances | jq '.Reservations[].Instances[].PrivateIpAddress'

# Use case 2 : filtrer les instances en état "running"
aws ec2 describe-instances \
  | jq '.Reservations[].Instances[] | select(.State.Name=="running") | .InstanceId'

# Use case 3 : formater un JSON illisible
cat response.json | jq '.'

# Use case 4 : extraire une valeur précise
curl -s https://api.exemple.com/status | jq '.services.database.status'

# Use case 5 : créer un tableau depuis des résultats
aws s3api list-buckets | jq '[.Buckets[].Name]'

# Use case 6 : combiner avec un script bash
aws ec2 describe-instances | jq -r '.Reservations[].Instances[].PrivateIpAddress' \
  | while read IP; do
      echo "Vérification de $IP..."
      ssh -o ConnectTimeout=5 ubuntu@$IP "uptime" 2>/dev/null || echo "$IP injoignable"
  done
```

---

## PARTIE 2 — Outils Cloud & DevOps

---

### AWS CLI — La télécommande d'AWS

```bash
# Installation
pip install awscli
aws configure    # Renseigne Access Key, Secret, Region

# Use case 1 : déployer un fichier sur S3
aws s3 cp mon-app.jar s3://mon-bucket/releases/
aws s3 sync /var/www/html/ s3://mon-site-static/

# Use case 2 : lister les instances EC2 qui tournent
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[].Instances[].[InstanceId,PrivateIpAddress,Tags[?Key=='Name'].Value|[0]]" \
  --output table

# Use case 3 : créer un snapshot EBS automatique (dans un cron)
VOLUME_ID="vol-0123456789abcdef0"
aws ec2 create-snapshot \
  --volume-id $VOLUME_ID \
  --description "Backup automatique $(date +%Y-%m-%d)"

# Use case 4 : lire les logs CloudWatch en temps réel
aws logs tail /aws/lambda/ma-fonction --follow

# Use case 5 : script de déploiement Lambda complet
zip -r function.zip . && \
aws lambda update-function-code \
  --function-name ma-fonction \
  --zip-file fileb://function.zip && \
echo "Déploiement Lambda réussi !"
```

---

### Terraform — Infrastructure as Code

> **Principe** : tu décris l'état désiré de ton infrastructure dans des fichiers `.tf`.
> Terraform calcule ce qui doit être créé, modifié ou supprimé, puis l'applique.

```hcl
# Use case : créer un serveur web sur AWS avec réseau et sécurité

# main.tf
provider "aws" {
  region = "eu-west-1"
}

resource "aws_instance" "web" {
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  key_name      = "ma-cle-ssh"

  tags = {
    Name        = "web-server"
    Environment = "production"
  }
}

resource "aws_security_group" "web_sg" {
  name = "web-sg"

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

```bash
# Workflow Terraform
terraform init      # Télécharger les providers
terraform plan      # Voir ce qui va être créé/modifié (dry-run)
terraform apply     # Appliquer les changements
terraform destroy   # Détruire toute l'infrastructure

# Use case réel : créer 10 serveurs identiques
# Il suffit de changer count = 10 — impossible à faire manuellement sans erreur
resource "aws_instance" "web" {
  count         = 10
  ami           = "ami-12345678"
  instance_type = "t3.micro"
  tags = { Name = "web-${count.index + 1}" }
}
```

**Pourquoi c'est crucial en Cloud Architecture**
- Ton infrastructure devient du code versionnable dans Git
- Reproductible : recréer un environnement identique en une commande
- Auditables : on voit exactement ce qui a changé et quand

---

### Docker — Conteneurisation

> **Principe** : packager une application avec TOUTES ses dépendances dans un conteneur.
> Ce qui tourne sur ton laptop tourne identiquement sur n'importe quel serveur AWS.

```dockerfile
# Use case : Dockerfile pour une app Node.js
FROM node:18-alpine

WORKDIR /app
COPY package*.json ./
RUN npm install --production

COPY . .
EXPOSE 3000

CMD ["node", "server.js"]
```

```bash
# Construire et lancer
docker build -t mon-app:v1.0 .
docker run -d -p 80:3000 --name mon-app mon-app:v1.0

# Use case : docker-compose pour app + base de données
# docker-compose.yml
version: '3'
services:
  app:
    build: .
    ports: ["80:3000"]
    environment:
      DB_HOST: postgres
    depends_on: [postgres]
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data
volumes:
  pgdata:

# Lancer toute la stack
docker-compose up -d

# Use case sysadmin : voir les logs d'un conteneur en temps réel
docker logs -f mon-app

# Entrer dans un conteneur pour déboguer
docker exec -it mon-app /bin/sh
```

---

### Kubernetes — Orchestration de conteneurs

> **Principe** : Docker gère UN conteneur. Kubernetes gère des MILLIERS de conteneurs
> en production — avec scaling automatique, self-healing, load balancing.

```yaml
# Use case : déployer une app avec 3 réplicas et load balancer
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mon-app
spec:
  replicas: 3                  # 3 instances de l'app
  selector:
    matchLabels:
      app: mon-app
  template:
    spec:
      containers:
      - name: mon-app
        image: mon-app:v1.0
        ports:
        - containerPort: 3000
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: mon-app-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 3000
  selector:
    app: mon-app
```

```bash
# Commandes essentielles
kubectl apply -f deployment.yaml        # Déployer
kubectl get pods                        # État des conteneurs
kubectl get services                    # Services et IPs
kubectl logs -f pod/mon-app-xxx         # Logs en temps réel
kubectl scale deployment mon-app --replicas=10   # Scaler à 10 instances
kubectl rollout undo deployment mon-app          # Rollback si problème
```

---

### Ansible — Configuration Management

> **Principe** : configurer des dizaines/centaines de serveurs de manière identique,
> sans y aller un par un en SSH. Idempotent — tu peux lancer 10 fois, même résultat.

```yaml
# Use case : installer et configurer nginx sur 50 serveurs
# playbook_nginx.yml
---
- name: Configurer les serveurs web
  hosts: webservers       # Groupe de serveurs défini dans l'inventaire
  become: yes             # sudo

  tasks:
    - name: Installer nginx
      apt:
        name: nginx
        state: present
        update_cache: yes

    - name: Copier la configuration
      template:
        src: nginx.conf.j2
        dest: /etc/nginx/nginx.conf
      notify: Redémarrer nginx

    - name: Ouvrir le port 80
      ufw:
        rule: allow
        port: '80'

  handlers:
    - name: Redémarrer nginx
      service:
        name: nginx
        state: restarted
```

```bash
# Lancer le playbook sur tous les serveurs
ansible-playbook -i inventaire.ini playbook_nginx.yml

# Use case : vérifier l'uptime de tous les serveurs en une commande
ansible all -i inventaire.ini -m command -a "uptime"
```

---

### Git — Versioning

```bash
# Use case 1 : workflow standard (feature branch)
git checkout -b feature/nouveau-script
# ... tu codes ...
git add mon_script.sh
git commit -m "feat: script de monitoring disque"
git push origin feature/nouveau-script
# Puis Pull Request → review → merge

# Use case 2 : voir qui a modifié une ligne et quand
git blame /etc/nginx/nginx.conf

# Use case 3 : revenir en arrière sur un fichier
git checkout HEAD~1 -- mon_script.sh    # Version d'avant le dernier commit

# Use case 4 : tags pour les releases
git tag -a v1.2.0 -m "Version 1.2.0 — ajout monitoring"
git push origin v1.2.0

# Use case 5 : stash — mettre de côté des modifications
git stash          # Sauvegarder sans committer
git stash pop      # Récupérer
```

---

### Python + boto3 — Automatisation AWS

> boto3 est le SDK AWS pour Python. Plus puissant que AWS CLI pour la logique complexe.

```python
# Use case 1 : lister toutes les instances EC2 et leur état
import boto3

ec2 = boto3.client('ec2', region_name='eu-west-1')
response = ec2.describe_instances()

for reservation in response['Reservations']:
    for instance in reservation['Instances']:
        print(f"ID: {instance['InstanceId']} | "
              f"État: {instance['State']['Name']} | "
              f"IP: {instance.get('PrivateIpAddress', 'N/A')}")

# Use case 2 : backup automatique de tous les volumes EBS
import boto3
from datetime import datetime

ec2 = boto3.client('ec2')
volumes = ec2.describe_volumes()['Volumes']

for volume in volumes:
    snapshot = ec2.create_snapshot(
        VolumeId=volume['VolumeId'],
        Description=f"Backup automatique {datetime.now().strftime('%Y-%m-%d')}"
    )
    print(f"Snapshot créé : {snapshot['SnapshotId']}")

# Use case 3 : envoyer une alerte sur SNS si CPU > 80%
import boto3

sns = boto3.client('sns')
sns.publish(
    TopicArn='arn:aws:sns:eu-west-1:123456789:alertes',
    Subject='ALERTE CPU élevé',
    Message='Le serveur web01 dépasse 80% de CPU depuis 5 minutes'
)
```

---

### CI/CD avec GitHub Actions — Automatisation complète

> **Principe** : à chaque `git push`, un pipeline se déclenche automatiquement —
> tests, build Docker, déploiement sur AWS. Zéro intervention manuelle.

```yaml
# Use case : pipeline complet test → build → deploy sur AWS
# .github/workflows/deploy.yml

name: Deploy to AWS

on:
  push:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Lancer les tests
        run: npm test

  build-and-deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Configurer AWS
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: eu-west-1

      - name: Build et push image Docker
        run: |
          docker build -t mon-app:${{ github.sha }} .
          docker tag mon-app:${{ github.sha }} 123456789.dkr.ecr.eu-west-1.amazonaws.com/mon-app:latest
          docker push 123456789.dkr.ecr.eu-west-1.amazonaws.com/mon-app:latest

      - name: Déployer sur ECS
        run: |
          aws ecs update-service \
            --cluster production \
            --service mon-app \
            --force-new-deployment
```

---

### Prometheus + Grafana — Monitoring & Observabilité

> **Prometheus** collecte les métriques. **Grafana** les visualise.
> Équivalent self-hosted de CloudWatch + CloudWatch Dashboards.

```yaml
# Use case : alerter si un serveur est down depuis 5 minutes
# alerting_rules.yml
groups:
  - name: infrastructure
    rules:
      - alert: ServeurDown
        expr: up == 0
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Serveur {{ $labels.instance }} injoignable"
          description: "{{ $labels.instance }} est down depuis plus de 5 minutes"

      - alert: CPUElevé
        expr: cpu_usage_percent > 85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "CPU élevé sur {{ $labels.instance }}"
```

---

## Récapitulatif — Quand utiliser quoi

| Besoin | Outil |
|---|---|
| Chercher dans des logs | `grep`, `awk` |
| Modifier des fichiers de config | `sed` |
| Trouver des fichiers suspects | `find` |
| Tester une API / un endpoint | `curl` |
| Se connecter à un serveur | `ssh` |
| Synchroniser des fichiers | `rsync` |
| Parser du JSON (AWS CLI) | `jq` |
| Automatiser AWS | `AWS CLI`, `Python boto3` |
| Créer/détruire de l'infrastructure | `Terraform` |
| Packager une application | `Docker` |
| Gérer des conteneurs en production | `Kubernetes` |
| Configurer des serveurs en masse | `Ansible` |
| Versionner du code/infra | `Git` |
| Automatiser les déploiements | `GitHub Actions` |
| Surveiller l'infrastructure | `Prometheus + Grafana` |

---

## Le pipeline ultime — Tout enchaîné

```
Developer git push
      ↓
GitHub Actions (CI/CD)
  → Tests automatiques
  → docker build
  → docker push → ECR (registry AWS)
      ↓
Terraform (si nouvelle infra nécessaire)
  → Crée VPC, EC2, RDS, Load Balancer
      ↓
Ansible (si config serveur nécessaire)
  → Installe dépendances, configure nginx
      ↓
AWS ECS/EKS
  → Déploie le nouveau conteneur
  → Rolling update (zéro downtime)
      ↓
Prometheus + Grafana
  → Surveille CPU, RAM, erreurs HTTP
  → Alerte si anomalie détectée
      ↓
AWS CloudWatch + jq + bash
  → Logs centralisés
  → Scripts d'analyse automatique
```

> Tout ce pipeline repose sur une fondation solide : **Linux + Bash + outils Unix**.
> Un Solutions Architect qui comprend chaque brique de ce pipeline
> peut concevoir, déboguer et optimiser n'importe quelle infrastructure Cloud.
