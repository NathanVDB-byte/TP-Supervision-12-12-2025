# Sécurité réseau avancée

## Architecture des réseaux

### Les 3 réseaux Docker créés

```bash
docker network create secure_front
docker network create secure_back
docker network create monitoring_net
```

| Réseau | Rôle | Services |
|--------|------|----------|
| `secure_front` | Front-end, reverse proxy, exposition publique | Traefik, Grafana |
| `secure_back` | Back-end interne, données sensibles | App, Registry, DB (optionnelle) |
| `monitoring_net` | Monitoring et supervision (isolation) | cAdvisor, Prometheus, Grafana (aussi) |

### Topologie des services

```
┌─────────────────────────────────────────────────────────────┐
│                     secure_front                             │
│  ┌──────────────┐              ┌──────────────┐              │
│  │   Traefik    │              │   Grafana    │              │
│  │ (80/443)     │              │ (3000)       │              │
│  └──────────────┘              └──────────────┘              │
└─────────────────────────────────────────────────────────────┘
          │                               │
          │ Route HTTP/HTTPS             │
          ▼                               ▼
┌─────────────────────────────────────────────────────────────┐
│                     secure_back                              │
│  ┌──────────────┐              ┌──────────────┐              │
│  │     App      │              │   Registry   │              │
│  │  (Nginx)     │              │  (5000)      │              │
│  └──────────────┘              └──────────────┘              │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                   monitoring_net                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐        │
│  │   cAdvisor   │  │ Prometheus   │  │   Grafana    │        │
│  │  (8080)      │  │  (9090)      │  │ (3000) [alt] │        │
│  └──────────────┘  └──────────────┘  └──────────────┘        │
└─────────────────────────────────────────────────────────────┘
```

**Points clés :**

- Traefik est **seul** sur `secure_front` pour recevoir le trafic extérieur.
- L'app est sur `secure_back` ET `secure_front` (pour être routée par Traefik).
- Registry est isolée sur `secure_back` (accès direct bloqué).
- Monitoring (Prometheus, cAdvisor) est **complètement isolé** sur `monitoring_net`.
- Grafana apparaît sur deux réseaux pour avoir accès à Prometheus ET être exposée via Traefik.

## Règles d'isolation

### 1. Accès inter-réseaux bloqués par défaut

```bash
# Vérifier que secure_back et monitoring_net ne communiquent pas
docker exec -it app ping prometheus        # ❌ Doit échouer
docker exec -it app curl http://prometheus:9090  # ❌ Doit échouer
docker exec -it prometheus ping app         # ❌ Doit échouer
```

### 2. Pas d'exposition de ports sensibles

- Prometheus : **aucun port** exposé vers l'hôte (accessible uniquement via `monitoring_net`).
- cAdvisor : **aucun port** exposé vers l'hôte.
- Registry : port 5000 local uniquement, accès depuis `secure_back`.

Configuration en exemple :

```yaml
prometheus:
  image: prom/prometheus
  # AUCUN ports: [...] → pas d'exposition à l'hôte
  networks:
    - monitoring_net

cadvisor:
  image: gcr.io/cadvisor/cadvisor:latest
  # AUCUN ports: [...] → pas d'exposition à l'hôte
  networks:
    - monitoring_net
```

### 3. Accès à Grafana sécurisé

Grafana est exposé **uniquement via Traefik** en HTTPS :

```yaml
grafana:
  labels:
    - "traefik.http.routers.monitoring.rule=Host(`monitoring.localhost`)"
    - "traefik.http.routers.monitoring.entrypoints=websecure"
    - "traefik.http.routers.monitoring.tls=true"
  networks:
    - monitoring_net
    - secure_front  # Accessible à Traefik uniquement
```

Accès : `https://monitoring.localhost` (avec authentification `admin:azerty`).

## Vérification de l'isolation

### Inspecter un réseau

```bash
docker network inspect secure_back
# Affiche les conteneurs connectés, leurs IPs, etc.

docker network inspect monitoring_net
```

### Tester la séparation

```bash
# Test 1 : App ne voit pas Prometheus
docker exec -it app sh -c "ping -c 1 prometheus" || echo "✓ Isolé"

# Test 2 : App ne voit pas cAdvisor
docker exec -it app sh -c "curl -s http://cadvisor:8080" || echo "✓ Isolé"

# Test 3 : Prometheus ne voit pas App
docker exec -it prometheus sh -c "curl -s http://app" || echo "✓ Isolé"

# Test 4 : Traefik atteint l'App (sur secure_front)
docker exec -it traefik sh -c "curl -s http://app:80" && echo "✓ Accessible"
```

## Bloquage du ping (ICMP) dans un conteneur

Pour bloquer la requête ping au niveau d'un conteneur, ajouter une règle iptables via un script d'entrypoint :

```bash
#!/bin/bash
iptables -A INPUT -p icmp --icmp-type echo-request -j DROP
exec "$@"
```

Montrer dans le Dockerfile :

```dockerfile
COPY entrypoint.sh /
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

Résultat :

```bash
docker exec -it app ping localhost  # ❌ Timeout (ICMP bloqué)
curl http://app:80                  # ✓ OK (TCP/80 passe)
```

## Scan de vulnérabilités

### Avec Trivy

```bash
# Installer Trivy (si pas déjà fait)
curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin

# Scanner l'image
trivy image localhost:5000/app:latest
```

### Avec Docker Scout

```bash
docker scout cves localhost:5000/app:latest
```

### Exemple de rapport

```
localhost:5000/app:latest (debian:11 - Debian GNU/Linux 11)
─────────────────────────────────────────────────────────────
LOW        CVE-2023-12345  (openssl:1.1.1)      [CVSS: 4.5]
MEDIUM     CVE-2023-45678  (curl:7.68)          [CVSS: 6.1]
HIGH       CVE-2023-90123  (libc:2.31)          [CVSS: 8.2]
```

Les vulnérabilités peuvent être adressées en :
- Mettant à jour l'image de base (ex. `alpine:latest` au lieu de `debian:11`).
- Appliquant des correctifs via `apt update && apt upgrade`.

## Impact global de l'isolation

| Aspect | Impact |
|--------|--------|
| **Surface d'attaque** | ✓ Réduite : services internes non accessibles de l'extérieur |
| **Mouvements latéraux** | ✓ Limités : une app compromise ne peut accéder à Prometheus |
| **Confidentialité** | ✓ Renforcée : données de monitoring isolées du back-end |
| **Débogage** | ✓ Facilité : séparation claire front/back/monitoring |
| **Scalabilité** | ✓ Améliorée : chaque plan peut évoluer indépendamment |

## Checklist de sécurité

- [x] Trois réseaux Docker créés et configurés.
- [x] Services isolés sur les bons réseaux.
- [x] Ports sensibles (Prometheus, cAdvisor) non exposés à l'hôte.
- [x] Accès à Grafana uniquement via Traefik/HTTPS.
- [x] Règles ICMP appliquées dans les conteneurs.
- [x] Scan de vulnérabilités effectué et documenté.
- [x] Vérification d'isolation testée et passée.
