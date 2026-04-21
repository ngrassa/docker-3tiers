# 🐳 docker-3tiers

Application web déployée en architecture **3 niveaux** avec Docker Compose.  
Elle illustre l'isolation réseau entre les couches **frontend (nginx)**, **application (Flask/Python)** et **base de données (MySQL)**.

---

## 📋 Table des matières

- [Architecture](#-architecture)
- [Prérequis](#-prérequis)
- [Structure du projet](#-structure-du-projet)
- [Démarrage rapide](#-démarrage-rapide)
- [Services](#-services)
- [Réseaux & isolation](#-réseaux--isolation)
- [Volumes](#-volumes)
- [Endpoints disponibles](#-endpoints-disponibles)
- [Connectivité réseau](#-connectivité-réseau)
- [Commandes utiles](#-commandes-utiles)
- [Test de l'architecture réseau](#-test-de-larchitecture-réseau)

---

## 🏗 Architecture

```
          ┌──────────────────────────────┐
          │      HÔTE (localhost)        │
          │        port 80 → nginx       │
          └──────────────┬───────────────┘
                         │ HTTP
                         ▼
          ┌──────────────────────────────┐
          │   réseau : frontend_net      │
          │                              │
          │  ┌──────────┐  ┌──────────┐  │
          │  │  nginx   │─►│  app     │  │
          │  │:80       │  │Flask:8080│  │
          │  └──────────┘  └────┬─────┘  │
          └───────────────────┬─┘────────┘
                              │
          ┌───────────────────▼──────────┐
          │   réseau : backend_net       │
          │                              │
          │              ┌──────────┐    │
          │              │    db    │    │
          │              │MySQL:3306│    │
          │              └──────────┘    │
          └──────────────────────────────┘
```

| Couche | Service | Image | Port interne | Réseau(x) |
|--------|---------|-------|-------------|-----------|
| Frontend | `nginx` | nginx:alpine | 80 | frontend_net |
| Application | `app` | python:3.11-slim (build local) | 8080 | frontend_net + backend_net |
| Base de données | `db` | mysql:8 | 3306 | backend_net |

---

## ✅ Prérequis

- [Docker](https://docs.docker.com/get-docker/) ≥ 20.x
- [Docker Compose](https://docs.docker.com/compose/install/) ≥ 1.29 (ou `docker compose` v2)

> **WSL2 (Windows)** : si vous obtenez l'erreur `docker-credential-wincred.exe not found`, éditez `~/.docker/config.json` et supprimez la clé `credsStore` :
> ```json
> { "auths": { "https://index.docker.io/v1/": { "auth": "..." } } }
> ```

---

## 📁 Structure du projet

```
docker-3tiers/
├── docker-compose.yml          # Orchestration des services
├── nginx/
│   └── nginx.conf              # Configuration reverse proxy nginx
├── app/
│   ├── Dockerfile              # Image Python/Flask personnalisée
│   ├── app.py                  # Application Flask (routes / health check)
│   └── monsite/
│       └── index.html          # Page HTML servie par Flask
├── monsite/
│   └── index.html              # Page HTML statique (référence)
└── test-app-3tiers/
    ├── docker-compose.yml      # Environnement de test isolation réseau
    └── README.md               # Notes de tests de connectivité
```

---

## 🚀 Démarrage rapide

```bash
# 1. Cloner le dépôt
git clone https://github.com/ngrassa/docker-3tiers.git
cd docker-3tiers

# 2. Construire et lancer tous les services
docker-compose up -d --build

# 3. Vérifier que les 3 conteneurs tournent
docker-compose ps

# 4. Ouvrir l'application dans le navigateur
http://localhost
```

Pour arrêter :

```bash
docker-compose down
```

Pour arrêter et supprimer les données persistantes :

```bash
docker-compose down -v
```

---

## 🔧 Services

### 🌐 nginx — Reverse proxy frontal

```yaml
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
  depends_on:
    - app
  networks:
    - frontend_net
```

Point d'entrée unique de l'application. Il reçoit les requêtes HTTP sur le port **80** et les transfère vers le serveur Flask (`app:8080`) via un `proxy_pass`.

**Configuration nginx (`nginx/nginx.conf`) :**
```nginx
events {}

http {
  upstream app_backend {
    server app:8080;
  }

  server {
    listen 80;
    location / {
      proxy_pass http://app_backend;
    }
  }
}
```

---

### ⚙️ app — Serveur d'application Flask

```yaml
app:
  build: ./app
  environment:
    DB_HOST: db
    DB_USER: root
    DB_PASSWORD: root
    DB_NAME: appdb
  networks:
    - frontend_net
    - backend_net
  depends_on:
    - db
```

Serveur Python/Flask construit depuis l'image `python:3.11-slim`. Il expose le port **8080**, sert le site HTML et dispose d'un endpoint `/health`.  
Connecté aux **deux réseaux** : c'est la passerelle entre nginx et la base de données.

**Dockerfile :**
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . .
RUN pip install flask
EXPOSE 8080
CMD ["python", "app.py"]
```

**Routes applicatives (`app/app.py`) :**

| Route | Description |
|-------|-------------|
| `GET /` | Sert la page `index.html` depuis `monsite/` |
| `GET /health` | Retourne `{"status": "ok"}` (health check) |

---

### 🗄️ db — Base de données MySQL

```yaml
db:
  image: mysql:8
  environment:
    MYSQL_ROOT_PASSWORD: root
    MYSQL_DATABASE: appdb
  volumes:
    - db_data:/var/lib/mysql
  networks:
    - backend_net
```

Instance MySQL 8. Isolée dans le réseau `backend_net` : non accessible depuis l'extérieur ni depuis nginx. Les données sont persistées dans le volume `db_data`.

| Variable | Valeur |
|----------|--------|
| `MYSQL_ROOT_PASSWORD` | `root` |
| `MYSQL_DATABASE` | `appdb` |
| Port interne | `3306` |

---

## 🌐 Réseaux & isolation

```yaml
networks:
  frontend_net:   # nginx ↔ app
  backend_net:    # app ↔ db
```

Deux réseaux **bridge** distincts assurent l'isolation entre les couches :

| Source | Destination | Accès | Raison |
|--------|------------|-------|--------|
| `nginx` | `app` | ✅ Oui | réseau `frontend_net` partagé |
| `app` | `db` | ✅ Oui | réseau `backend_net` partagé |
| `nginx` | `db` | ❌ Non | aucun réseau commun |
| `db` | `nginx` | ❌ Non | aucun réseau commun |

> **Principe clé :** Docker isole les réseaux bridge entre eux. Deux conteneurs ne peuvent communiquer que s'ils appartiennent au même réseau. `app`, connecté aux deux réseaux, est le seul pont entre les couches.

---

## 💾 Volumes

```yaml
volumes:
  db_data:    # Persistance des données MySQL entre les redémarrages
```

Le volume `db_data` garantit que les données MySQL survivent à un `docker-compose down`. Seul un `docker-compose down -v` les supprime.

---

## 🔗 Endpoints disponibles

| URL | Service | Description |
|-----|---------|-------------|
| `http://localhost/` | nginx → app | Page d'accueil HTML |
| `http://localhost/health` | nginx → app | Health check JSON |

---

## 📡 Connectivité réseau

```
nginx ──(frontend_net)──► app ──(backend_net)──► db
  │                                                │
  └────────── ✗ ISOLÉS (pas de route directe) ────┘
```

Vérification des pings depuis les conteneurs :

```bash
# ✅ nginx peut joindre app (frontend_net)
docker exec <nginx_container> ping -c 2 app

# ✅ app peut joindre db (backend_net)
docker exec <app_container> ping -c 2 db

# ❌ nginx NE PEUT PAS joindre db
docker exec <nginx_container> ping -c 2 db
# → ping: db: Name or service not known
```

---

## 🛠 Commandes utiles

```bash
# Voir les logs de tous les services
docker-compose logs -f

# Logs d'un service spécifique
docker-compose logs -f app

# Reconstruire l'image app après modification
docker-compose up -d --build app

# Inspecter les réseaux créés
docker network ls
docker network inspect docker-3tiers_frontend_net

# Accéder au shell du conteneur app
docker-compose exec app bash

# Accéder à MySQL
docker-compose exec db mysql -u root -proot appdb
```

---

## 🧪 Test de l'architecture réseau

Le dossier `test-app-3tiers/` contient un environnement de test minimal (nginx + ubuntu + mysql) pour valider l'isolation réseau sans image personnalisée :

```bash
cd test-app-3tiers
docker-compose up -d

# Test connectivité app → nginx ✅
docker exec app-3tiers_app_1 ping -c 2 app-3tiers_nginx_1

# Test connectivité app → db ✅
docker exec app-3tiers_app_1 ping -c 2 app-3tiers_db_1

# Test isolation nginx → db ❌
docker exec app-3tiers_nginx_1 ping -c 2 app-3tiers_db_1
```

---

## 📝 Licence

Projet pédagogique — libre d'utilisation à des fins d'apprentissage.
