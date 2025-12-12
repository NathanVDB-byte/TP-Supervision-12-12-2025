# CI/CD Docker avec Registry privée

## Objectif

Mettre en place un pipeline simple et local de build, tagging et déploiement d'une image Docker vers une registry privée, permettant un cycle de développement rapide sans dépendre d'une registry publique.

## Infrastructure du pipeline

### 1. Registry privée locale

La registry Docker privée est déployée comme un conteneur dans `docker-compose.yml` :

```yaml
registry:
  image: registry:2
  container_name: registry
  restart: always
  ports:
    - "5000:5000"
  volumes:
    - "registry-data:/var/lib/registry"
  networks:
    - secure_back
```

Elle écoute sur le port 5000 en local (`localhost:5000`).

### 2. Image de l'application

L'application est un serveur Nginx simple :

```dockerfile
# app/Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/index.html
```

```html
<!-- app/index.html -->
<html><body><h1>App supervision</h1></body></html>
```

## Étapes du pipeline

### Étape 1 : Build l'image localement

```bash
docker build -t app:latest ./app
```

Crée une image nommée `app:latest` basée sur Nginx Alpine.

### Étape 2 : Tag pour la registry privée

```bash
docker tag app:latest localhost:5000/app:latest
```

Prépare l'image pour la registry en ajoutant le préfixe `localhost:5000/`.

### Étape 3 : Push vers la registry privée

```bash
docker push localhost:5000/app:latest
```

Envoie l'image vers la registry locale. Celle-ci la stocke dans son volume `registry-data`.

### Étape 4 : Utilisation dans docker-compose

Dans `docker-compose.yml`, le service app utilise **uniquement** l'image de la registry :

```yaml
app:
  image: localhost:5000/app:latest
  depends_on:
    - registry
  networks:
    - secure_back
    - secure_front
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.app.rule=Host(`app.localhost`)"
    - "traefik.http.routers.app.entrypoints=websecure"
    - "traefik.http.routers.app.tls=true"
    - "traefik.http.services.app.loadbalancer.server.port=80"
```

### Étape 5 : Déploiement

Lancer ou mettre à jour l'application :

```bash
docker compose up -d
```

Docker tire l'image depuis la registry privée et crée le conteneur.

## Cycle d'itération (Rebuild / Redeploy)

En cas de modification du code (ex. `app/index.html`) :

```bash
# 1. Rebuild l'image
docker build -t app:latest ./app

# 2. Retag pour la registry
docker tag app:latest localhost:5000/app:latest

# 3. Repush
docker push localhost:5000/app:latest

# 4. Redéployer (force recreate pour utiliser la nouvelle image)
docker compose up -d --force-recreate
```

Cela garantit que le conteneur utilise la version à jour de l'image.

## Schéma du pipeline

```
┌─────────────────────────────────────────────────────────┐
│                  Code Source                             │
│         (./app/Dockerfile, ./app/index.html)            │
└────────────────────────┬────────────────────────────────┘
                         │
                         ▼
            ┌────────────────────────────┐
            │   docker build             │
            │   -t app:latest ./app      │
            └────────────┬───────────────┘
                         │
                         ▼
            ┌────────────────────────────────────┐
            │   docker tag                       │
            │   localhost:5000/app:latest        │
            └────────────┬──────────────────────┘
                         │
                         ▼
        ┌─────────────────────────────────────────┐
        │   docker push                           │
        │   localhost:5000/app:latest             │
        └────────┬────────────────────────────────┘
                 │
                 ▼
    ┌──────────────────────────────────────────────┐
    │  Registry privée (localhost:5000)             │
    │  Stocke : registry-data/...                  │
    └─────────────┬───────────────────────────────┘
                  │
                  ▼
    ┌──────────────────────────────────────────────┐
    │  docker-compose.yml                          │
    │  image: localhost:5000/app:latest            │
    └─────────────┬───────────────────────────────┘
                  │
                  ▼
    ┌──────────────────────────────────────────────┐
    │  docker compose up -d --force-recreate       │
    │  Crée/met à jour le conteneur app            │
    └──────────────────────────────────────────────┘
```

## Avantages de cette approche

| Avantage | Détail |
|----------|--------|
| **Indépendance** | Pas de dépendance à Docker Hub ou autre registry publique. |
| **Vitesse** | Les images sont locales, push/pull très rapides. |
| **Contrôle** | Stockage centralisé dans `registry-data/`, facilite les sauvegardes. |
| **Isolement** | Environnement de test fermé, idéal pour le dev. |
| **Réutilisabilité** | La même image peut être utilisée par plusieurs services. |

## Rôle de la registry

- **Point central** : unique source de vérité pour les images de l'application.
- **Découplage CI/CD** : sépare le build (création d'image) du déploiement (création de conteneurs).
- **Versionning** : permet de tagger différentes versions (`:latest`, `:v1.0`, etc.).
- **Distribution** : les autres hôtes/services peuvent tirer la même image de manière cohérente.

## Vérification

Lister les images dans la registry :

```bash
curl -s http://localhost:5000/v2/_catalog | jq .
# Doit afficher : {"repositories":["app"]}

curl -s http://localhost:5000/v2/app/tags/list | jq .
# Doit afficher : {"name":"app","tags":["latest"]}
```
