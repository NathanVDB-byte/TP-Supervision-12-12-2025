# TLS et Traefik

## Génération du certificat auto-signé

Un certificat auto-signé a été généré pour le domaine `app.localhost` avec la commande suivante :

```bash
openssl req -x509 -nodes -newkey rsa:2048 \
  -keyout certs/app.localhost.key \
  -out certs/app.localhost.crt \
  -days 365 \
  -subj "/CN=app.localhost"
```

**Explications :**
- `-x509` : génère un certificat auto-signé (pas de CA).
- `-nodes` : n'encrypte pas la clé privée.
- `-newkey rsa:2048` : génère une clé RSA 2048 bits.
- `-days 365` : validité du certificat sur 365 jours.
- `-subj "/CN=app.localhost"` : le Common Name est `app.localhost`.

## Emplacement des certificats

Les fichiers sont stockés dans la structure du projet :

```
/srv/docker/supervision/
├── certs/
│   ├── app.localhost.crt      # Certificat public
│   └── app.localhost.key      # Clé privée
├── docker-compose.yml
└── traefik/
    └── dynamic.yml
```

Ils sont montés en lecture seule dans le conteneur Traefik :

```yaml
volumes:
  - "./certs/app.localhost.crt:/certs/app.localhost.crt:ro"
  - "./certs/app.localhost.key:/certs/app.localhost.key:ro"
```

## Configuration Traefik

### Entrypoints

Traefik est configuré avec deux entrypoints :

- `web` : port 80 (HTTP).
- `websecure` : port 443 (HTTPS).

### Fichier dynamique (traefik/dynamic.yml)

```yaml
tls:
  certificates:
    - certFile: /certs/app.localhost.crt
      keyFile: /certs/app.localhost.key
```

Traefik lit ce fichier et charge le certificat au démarrage.

### Router de l'application

Dans `docker-compose.yml`, le service `app` est configuré ainsi :

```yaml
app:
  labels:
    - "traefik.enable=true"
    - "traefik.http.routers.app.rule=Host(`app.localhost`)"
    - "traefik.http.routers.app.entrypoints=websecure"
    - "traefik.http.routers.app.tls=true"
    - "traefik.http.services.app.loadbalancer.server.port=80"
```

## Flux de requête TLS

1. **Client appelle** `http://app.localhost:80`
2. **Traefik reçoit** sur l'entrypoint `web` (port 80)
3. **Redirection HTTP → HTTPS** : grâce à la directive `--entrypoints.web.http.redirections.entrypoint.to=websecure`
4. **Client reconnecte** en HTTPS : `https://app.localhost:443`
5. **Traefik établit TLS** avec le certificat `app.localhost.crt`
6. **Router `app` route** la requête décryptée vers le conteneur `app` en HTTP interne (port 80)
7. **Réponse retournée** et retransmise en HTTPS au client

## Vérification

Pour vérifier le certificat sur la machine client :

```bash
# Afficher le certificat
openssl s_client -connect app.localhost:443 -servername app.localhost </dev/null 2>/dev/null | openssl x509 -noout -text

# Vérifier le CN (Common Name)
openssl s_client -connect app.localhost:443 -servername app.localhost </dev/null 2>/dev/null | openssl x509 -noout -subject
# Doit afficher : subject=CN = app.localhost
```

## Sécurité

- Le certificat est **auto-signé** : c'est normal pour un environnement de test/dev.
- La clé privée est **montée en lecture seule** dans Traefik.
- La redirection HTTP → HTTPS est **transparente** pour l'utilisateur.
