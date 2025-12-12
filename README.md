# TP Supervision Docker - README complet

## 📋 Table des matières

- [Description du projet](#description-du-projet)
- [Architecture globale](#architecture-globale)
- [Prérequis](#prérequis)
- [Installation rapide](#installation-rapide)
- [Structure du projet](#structure-du-projet)
- [Détails des composants](#détails-des-composants)
- [Commandes essentielles](#commandes-essentielles)
- [Accès aux services](#accès-aux-services)
- [Dépannage](#dépannage)
- [Fichiers de documentation](#fichiers-de-documentation)

---

## Description du projet

Ce projet implémente une **infrastructure de supervision Docker complète** incluant :

- ✅ **TLS auto-signé** : certificat HTTPS pour `app.localhost` exposé via Traefik
- ✅ **Reverse proxy** : Traefik avec routage HTTP/HTTPS et redirection automatique
- ✅ **Stack de monitoring** : cAdvisor → Prometheus → Grafana (3 métriques : CPU, RAM, Disk I/O, Uptime)
- ✅ **CI/CD local** : registry privée Docker locale avec pipeline build/tag/push
- ✅ **Sécurité réseau** : 3 réseaux Docker isolés (front, back, monitoring)
- ✅ **Scan de vulnérabilités** : intégration Trivy/Docker Scout

**Technologies :** Docker, Docker Compose, Traefik, Nginx, Prometheus, Grafana, cAdvisor

---

## Architecture globale

```
┌─────────────────────────────────────────────────────────────────┐
│                     Internet / Client                            │
│              (app.localhost, monitoring.localhost)              │
└──────────────────────┬──────────────────────────────────────────┘
                       │
                       ▼
        ┌──────────────────────────────┐
        │   Traefik (Reverse Proxy)    │
        │   Ports 80 (→443), 443       │
        │   Certificat auto-signé      │
        └──────────────────────────────┘
                   │              │
      ┌────────────┘              └──────────────┐
      ▼                                          ▼
┌──────────────────┐              ┌──────────────────────┐
│   App (Nginx)    │              │   Grafana            │
│   secure_back    │              │   secure_front       │
│   secure_front   │              │   monitoring_net     │
└──────────────────┘              └──────────────────────┘
      │                                          │
      │                                          ▼
      │                           ┌──────────────────────┐
      │                           │   Prometheus         │
      │                           │   monitoring_net     │
      │                           │   (pas d'exposition) │
      │                           └──────────────────────┘
      │                                          │
      │                                          ▼
      │                           ┌──────────────────────┐
      │                           │   cAdvisor           │
      │                           │   monitoring_net     │
      │                           │   (pas d'exposition) │
      │                           └──────────────────────┘
      │
      ▼
┌──────────────────────┐
│   Registry Privée    │
│   (localhost:5000)   │
│   secure_back        │
└──────────────────────┘
```

---

## Prérequis

- **Linux/Mac/Windows (avec WSL2)** avec Docker et Docker Compose installés
- **OpenSSL** (pour générer les certificats)
- **Git** (pour pousser vers le dépôt)
- **Accès root/sudo** (pour les commandes Docker)

Vérifier l'installation :

```bash
docker --version
docker-compose --version
openssl version
```

---

## Installation rapide

### 1. Cloner ou créer le dépôt

```bash
# Créer le dossier du projet
mkdir -p /srv/docker/supervision
cd /srv/docker/supervision

# Initialiser Git (si c'est un nouveau dépôt)
git init
git config user.name "Ton Nom"
git config user.email "ton@email.com"
```

### 2. Créer la structure des dossiers

```bash
mkdir -p certs traefik prometheus app
```

### 3. Générer le certificat auto-signé

```bash
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout certs/app.localhost.key \
  -out certs/app.localhost.crt \
  -days 365 \
  -subj "/CN=app.localhost"
```

### 4. Créer les fichiers de configuration

**traefik/dynamic.yml :**

```yaml
tls:
  certificates:
    - certFile: /certs/app.localhost.crt
      keyFile: /certs/app.localhost.key
```

**prometheus/prometheus.yml :**

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: "cadvisor"
    static_configs:
      - targets: ["cadvisor:8080"]
```

**app/Dockerfile :**

```dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

**app/index.html :**

```html
<html><body><h1>App supervision</h1></body></html>
```

### 5. Créer docker-compose.yml

Voir le fichier `docker-compose.yml` à la racine du projet pour la configuration complète.

### 6. Démarrer le stack

```bash
# Démarrer la registry et l'app
docker compose up -d registry
sleep 3

# Builder et pusher l'app vers la registry
docker build -t app:latest ./app
docker tag app:latest localhost:5000/app:latest
docker push localhost:5000/app:latest

# Démarrer tout le stack
docker compose up -d
```

### 7. Ajouter les hosts locaux

**Sur Linux/Mac, modifier `/etc/hosts` :**

```bash
echo "127.0.0.1  app.localhost monitoring.localhost" | sudo tee -a /etc/hosts
```

**Sur Windows, modifier `C:\Windows\System32\drivers\etc\hosts` :**

```
127.0.0.1  app.localhost monitoring.localhost
```

### 8. Vérifier que tout fonctionne

```bash
# Vérifier les conteneurs
docker ps

# Accéder à l'app
curl -vk https://app.localhost/

# Affichera : App supervision
```

---

## Structure du projet

```
/srv/docker/supervision/
│
├── README.md                    # Ce fichier
├── docker-compose.yml           # Configuration des services
├── tls.md                       # Documentation TLS/Traefik
├── cicd.md                      # Documentation CI/CD
├── security.md                  # Documentation sécurité réseau
│
├── certs/
│   ├── app.localhost.crt        # Certificat public
│   └── app.localhost.key        # Clé privée
│
├── traefik/
│   └── dynamic.yml              # Config TLS Traefik
│
├── prometheus/
│   └── prometheus.yml           # Config scrape cAdvisor
│
├── app/
│   ├── Dockerfile               # Image Nginx
│   └── index.html               # Page simple
│
└── .gitignore                   # Fichiers à ignorer
```

---

## Détails des composants

### Traefik (Reverse Proxy)

**Rôle :** Expose les services en HTTPS avec certificat auto-signé.

**Ports :**
- 8088 (HTTP) → redirige vers 443
- 8443 (HTTPS) → expose l'app et Grafana

**Configuration :**
- Lecture du Docker socket pour découvrir les services
- Chargement du certificat custom depuis `traefik/dynamic.yml`
- Routers pour `app.localhost` et `monitoring.localhost`

**Vérifier :**

```bash
docker logs traefik | grep -E "Router|Certificate"
curl -vk https://app.localhost/
```

---

### App (Nginx)

**Rôle :** Application simple exposée via Traefik.

**Image :** `localhost:5000/app:latest` (depuis registry privée)

**Ports :**
- 80 (interne, accès uniquement via Traefik)

**Réseaux :**
- `secure_back` (access à registry)
- `secure_front` (accessible par Traefik)

**Vérifier :**

```bash
docker logs app
curl -s http://localhost/  # Depuis Traefik en interne
```

---

### Registry Privée

**Rôle :** Stockage local des images Docker (CI/CD).

**Ports :**
- 5000 (local uniquement)

**Stockage :**
- Volume `registry-data:/var/lib/registry`

**Utilisation :**

```bash
docker build -t app:latest ./app
docker tag app:latest localhost:5000/app:latest
docker push localhost:5000/app:latest

# Lister les images
curl -s http://localhost:5000/v2/_catalog | jq .
```

---

### Prometheus

**Rôle :** Scrape les métriques de cAdvisor.

**Ports :**
- 9090 (interne, pas d'exposition à l'hôte)

**Réseau :**
- `monitoring_net` (isolé)

**Datasource pour Grafana :**
- URL : `http://prometheus:9090`
- Type : Prometheus

**Vérifier :**

```bash
# Depuis l'extérieur (doit échouer)
curl http://localhost:9090  # ❌

# Depuis Grafana (doit passer)
docker exec -it grafana wget -qO- http://prometheus:9090
```

---

### cAdvisor (Google)

**Rôle :** Collecte les métriques de tous les conteneurs.

**Ports :**
- 8080 (interne, pas d'exposition à l'hôte)

**Réseau :**
- `monitoring_net` (isolé)

**Métriques exposées :**
- `container_cpu_usage_seconds_total` (CPU)
- `container_memory_usage_bytes` (RAM)
- `container_fs_reads_bytes_total` / `container_fs_writes_bytes_total` (Disk I/O)
- `container_start_time_seconds` (pour calculer uptime)

---

### Grafana

**Rôle :** Dashboard de visualisation.

**Ports :**
- 3000 (interne, exposé uniquement via Traefik en HTTPS)

**Accès :**
- URL : `https://monitoring.localhost`
- Login : `admin`
- Password : `azerty`

**Réseaux :**
- `monitoring_net` (accès à Prometheus)
- `secure_front` (accès par Traefik)

**Configuration :**

1. Ajouter datasource Prometheus :
   - Settings → Datasources → Add Prometheus
   - URL : `http://prometheus:9090`
   - Save & Test

2. Importer dashboard :
   - Import → Upload JSON File
   - Charger `dashboard.json`
   - Sélectionner datasource Prometheus

---

## Commandes essentielles

### Démarrage / Arrêt

```bash
# Démarrer le stack
docker compose up -d

# Arrêter le stack
docker compose down

# Redémarrer un service
docker compose restart grafana

# Voir les logs
docker logs traefik
docker logs app
docker logs prometheus
```

### Gestion des images

```bash
# Builder l'app
docker build -t app:latest ./app

# Tagger pour la registry
docker tag app:latest localhost:5000/app:latest

# Pusher dans la registry
docker push localhost:5000/app:latest

# Lancer le cycle complet (build/tag/push/redeploy)
docker build -t app:latest ./app && \
docker tag app:latest localhost:5000/app:latest && \
docker push localhost:5000/app:latest && \
docker compose up -d --force-recreate
```

### Vérification de l'isolation réseau

```bash
# Lister les réseaux
docker network ls

# Inspecter un réseau
docker network inspect secure_back

# Tester l'isolation
docker exec -it app ping prometheus  # Doit échouer ❌
docker exec -it app curl http://prometheus:9090  # Doit échouer ❌
docker exec -it traefik wget -qO- http://app:80  # Doit passer ✓
```

### Nettoyage

```bash
# Supprimer les conteneurs et volumes
docker compose down -v

# Nettoyer les images inutilisées
docker image prune -a

# Nettoyer tout (conteneurs, images, volumes, réseaux)
docker system prune -a --volumes
```

---

## Accès aux services

| Service | URL | Authentification | Port |
|---------|-----|-----------------|------|
| **App** | `https://app.localhost` | Aucune | 443 |
| **Grafana** | `https://monitoring.localhost` | `admin:azerty` | 443 |
| **Prometheus** | `http://prometheus:9090` (interne) | Aucune | 9090 |
| **cAdvisor** | `http://cadvisor:8080` (interne) | Aucune | 8080 |
| **Registry** | `http://localhost:5000` (local) | Aucune | 5000 |

---

## Dépannage

### Les certificats ne sont pas reconnus

**Symptôme :** Erreur SSL au navigateur.

**Solution :**

```bash
# Vérifier que les fichiers existent
ls -l certs/app.localhost.*

# Vérifier le contenu du certificat
openssl x509 -in certs/app.localhost.crt -text -noout | grep -E "Subject:|CN="
```

### Traefik retourne 404

**Symptôme :** `404 page not found` en HTTPS.

**Solution :**

```bash
# Vérifier que les hosts sont définis localement
cat /etc/hosts | grep localhost  # Linux/Mac
type C:\Windows\System32\drivers\etc\hosts | findstr localhost  # Windows

# Vérifier les routers Traefik
docker logs traefik | grep -i router

# Vérifier les labels du service
docker inspect app | jq '.[0].Config.Labels'
```

### Prometheus ne scrape pas cAdvisor

**Symptôme :** Pas de métriques dans Grafana.

**Solution :**

```bash
# Vérifier que cAdvisor est accessible
docker exec -it prometheus wget -qO- http://cadvisor:8080/metrics | head

# Vérifier la config Prometheus
docker exec -it prometheus cat /etc/prometheus/prometheus.yml
```

### Grafana ne trouve pas Prometheus

**Symptôme :** "connection refused" sur la datasource.

**Solution :**

```bash
# Vérifier que Prometheus tourne
docker ps | grep prometheus

# Tester depuis Grafana
docker exec -it grafana wget -qO- http://prometheus:9090

# Recréer la datasource dans Grafana (Settings → Datasources)
```

### L'app ne démarre pas (image introuvable)

**Symptôme :** `image not found` ou `failed to pull image`.

**Solution :**

```bash
# Vérifier que la registry tourne
docker ps | grep registry

# Builder et pusher l'image
docker build -t app:latest ./app
docker tag app:latest localhost:5000/app:latest
docker push localhost:5000/app:latest

# Relancer le compose
docker compose up -d app
```

---

## Fichiers de documentation

Ce projet inclut 3 fichiers de documentation détaillée :

### **tls.md**

Explique :
- Comment générer le certificat auto-signé
- L'emplacement des fichiers de certificat
- Le fonctionnement du router TLS dans Traefik
- Le flux complet de requête HTTPS

### **cicd.md**

Explique :
- Le pipeline CI/CD local (build → tag → push → deploy)
- Le rôle de la registry privée
- Les commandes essentielles
- Un schéma visuel du pipeline

### **security.md**

Explique :
- L'architecture des 3 réseaux Docker
- Les règles d'isolation entre réseaux
- Comment tester l'isolement
- Le bloquage du ping (ICMP)
- Les scans de vulnérabilités (Trivy, Docker Scout)

---

## Workflow complet d'utilisation

### 1. Premier démarrage

```bash
cd /srv/docker/supervision
docker compose up -d
```

### 2. Configurer Grafana

- Ouvrir `https://monitoring.localhost`
- Login : `admin:azerty`
- Ajouter datasource Prometheus (`http://prometheus:9090`)
- Importer le dashboard `dashboard.json`

### 3. Mettre à jour l'app

```bash
# Modifier app/index.html
echo '<h1>App v2</h1>' > app/index.html

# Rebuild / push / redeploy
docker build -t app:latest ./app && \
docker tag app:latest localhost:5000/app:latest && \
docker push localhost:5000/app:latest && \
docker compose up -d --force-recreate
```

### 4. Pousser vers GitHub

```bash
git add .
git commit -m "TP supervision Docker 12-12-2025"
git remote add origin https://github.com/TON_USER/TP-Supervision-12-12-2025.git
git branch -M main
git push -u origin main
```

---

## Checklist de validation

- [ ] Docker et Docker Compose installés
- [ ] Certificats générés dans `certs/`
- [ ] Services démarrent sans erreur (`docker ps`)
- [ ] App accessible sur `https://app.localhost`
- [ ] Grafana accessible sur `https://monitoring.localhost`
- [ ] Dashboard Grafana affiche des métriques
- [ ] Registry fonctionne (`curl http://localhost:5000/v2/_catalog`)
- [ ] Isolation réseau testée et fonctionnelle
- [ ] Fichiers `.md` présents dans le dépôt
- [ ] Tout poussé sur GitHub

---

## Support et contributions

Pour des questions ou améliorations :

1. Consulter les fichiers `tls.md`, `cicd.md`, `security.md`
2. Vérifier les logs Docker : `docker logs <service>`
3. Tester manuellement les commandes du dépannage

---

## Licence

Projet pédagogique (TP Docker).
