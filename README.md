# System Metrics Agent — Conteneurisation, Compose et CI/CD

Projet réalisé dans le cadre du cours « Éléments du DevOps » (Master).
Dépôt : https://github.com/yabowilfred/metrics-agent-devops

## 1. Présentation et architecture
Agent Python (psutil) qui collecte CPU / RAM / charge système et envoie les métriques en HTTP
à une API FastAPI. Deux services, **une seule image Docker** :

- `api` : `uvicorn app.api:app` (port 8000), routes `/health`, `/metrics`, `/metrics/latest`
- `agent` : `python -m app.agent`, envoie vers `http://api:8000/metrics` via le réseau `metrics-net`

```
[agent] --POST /metrics--> [api] <-- GET /health, /metrics/latest
        (réseau Docker metrics-net)
```

## 2. Prérequis
Docker, Docker Compose v2, Git, un compte GitHub et un compte Docker Hub.

## 3. Lancer en développement (Dockerfile.dev)
```bash
cp .env.example .env
docker compose up --build        # docker-compose.override.yml chargé automatiquement (hot-reload)
docker build -f Dockerfile.dev -t metrics-agent:dev .
docker run --rm -v "$(pwd):/app" metrics-agent:dev pytest -q
```

## 4. Lancer en production
Build local de l'image de production :
```bash
docker compose -f docker-compose.yaml -f docker-compose.build.yaml up --build -d
```
Depuis Docker Hub (sans build, sans code source) :
```bash
docker compose -f docker-compose.yaml pull
docker compose -f docker-compose.yaml up -d
docker compose ps
curl http://localhost:8000/health
curl http://localhost:8000/metrics/latest
```

## 5. Pipeline CI/CD
Fichier `.github/workflows/ci-cd.yml`, déclenché sur push et pull request vers `main` :
checkout → build de l'image de production → tests pytest → login Docker Hub → push (`latest` + SHA du commit).
Le login et le push n'ont lieu que sur un push vers `main`, et seulement si les tests passent.
Secret requis : `DOCKERHUB_TOKEN` (access token Docker Hub, droits Read & Write).

![Pipeline vert](docs/pipeline-vert.png)
![Pipeline rouge sur une PR avec un test volontairement faux](docs/pipeline-rouge.png)
![Détail : le test en échec bloque le login et le push](docs/pipeline-rouge-detail.png)

## 6. Images Docker Hub
https://hub.docker.com/r/yabowilfred/metrics-agent (tags `latest` et SHA du commit, 65,21 Mo compressée)

![Tags Docker Hub](docs/dockerhub-tags.png)

## 7. Choix techniques et difficultés
- Base `python:3.12-slim` : légère et compatible avec les wheels de psutil.
- Image unique pour `api` et `agent` (commande surchargée dans Compose) : un seul artefact versionné, versions alignées.
- Multi-stage : les dépendances sont installées dans un venv copié dans l'image finale (pas de pytest ni d'outils de build en production). Utilisateur non-root (uid 10001), HEALTHCHECK en Python (pas de curl dans l'image slim).
- `procps` installé car `collector.py` appelle la commande `uptime`, absente de l'image slim.
- La suite de tests n'était pas dans le dépôt fourni : ajout d'un test minimal sur `/health` et d'un `pytest.ini`.
- Le HEALTHCHECK de l'image, prévu pour l'API, était hérité par le service agent et le marquait `unhealthy` : il est désactivé pour ce service dans `docker-compose.yaml`.
- Entre conteneurs, `127.0.0.1` n'est plus valable : `METRICS_ENDPOINT` vaut `http://api:8000/metrics`. L'ordre de démarrage est géré par `depends_on` avec `condition: service_healthy`.
- Plusieurs erreurs de pipeline corrigées pas à pas (nom d'image vide, secret manquant, token en lecture seule) : visibles dans l'historique des runs.
- Secrets : jamais committés (`.env` ignoré, `.env.example` seul versionné, token dans les secrets GitHub).

## 8. Preuves de fonctionnement
![Conteneurs actifs](docs/compose-ps.png)
![Appels /health et /metrics/latest](docs/health-metrics.png)
