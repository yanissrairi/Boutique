# Configuration Docker - Medusa Backend

## 🏗️ Architecture

### Environnement DEV (Local)
```
┌─────────────────────────────────────┐
│  Docker Compose (Local)             │
│  ┌───────────┐  ┌───────┐  ┌─────┐ │
│  │  Medusa   │──│ Redis │  │ PG  │ │
│  │  Backend  │  │ Local │  │Local│ │
│  └───────────┘  └───────┘  └─────┘ │
└─────────────────────────────────────┘
```

### Environnement PROD (Hetzner VPS)
```
┌──────────────────┐     ┌──────────────┐
│  Hetzner VPS     │     │   Supabase   │
│  ┌────────────┐  │────▶│  PostgreSQL  │
│  │   Medusa   │  │     └──────────────┘
│  │   Backend  │  │
│  └────────────┘  │     ┌──────────────┐
│                  │────▶│  Vercel KV   │
└──────────────────┘     │    Redis     │
                         └──────────────┘
```

---

## 📦 Fichiers de Configuration

| Fichier | Description | Chargement |
|---------|-------------|------------|
| `docker-compose.yml` | Configuration de base (Postgres + Redis + Medusa) | Auto (dev + prod) |
| `docker-compose.override.yml` | Overrides DEV (ports exposés, hot-reload) | Auto (dev uniquement) |
| `docker-compose.prod.yml` | Overrides PROD (désactive Postgres/Redis) | Manuel (prod) |
| `backend/Dockerfile` | Image Docker Medusa multi-stage (Node.js 24-alpine) | Build automatique |
| `.env` | Variables d'environnement actives | Requis |
| `.env.example` | Template pour développement local | Documentation |
| `.env.production.example` | Template pour production (Hetzner VPS) | Documentation |

---

## 🚀 Démarrage Rapide

### Développement Local

```bash
# 1. Copier le fichier d'environnement
cp .env.example .env

# 2. Démarrer tous les services (Medusa + Postgres + Redis)
docker compose up -d

# 3. Vérifier les logs
docker compose logs -f medusa

# 4. Accéder à Medusa
# Backend API: http://localhost:9000
# Admin: http://localhost:9000/app
```

**Services disponibles en DEV :**
- Medusa Backend : `localhost:9000`
- PostgreSQL : `localhost:5432`
- Redis : `localhost:6379`

### Production (Hetzner VPS)

```bash
# 1. Copier le template de production
cp .env.production.example .env

# 2. Générer des secrets forts
openssl rand -base64 32  # Pour JWT_SECRET
openssl rand -base64 32  # Pour COOKIE_SECRET

# 3. Éditer .env et remplir les variables de production:
# - DATABASE_URL (depuis Supabase)
# - REDIS_URL (depuis Vercel KV)
# - JWT_SECRET et COOKIE_SECRET (générés ci-dessus)
# - MEDUSA_BACKEND_URL, STORE_CORS, ADMIN_CORS (vos domaines)

# 4. Démarrer UNIQUEMENT Medusa (pas de Postgres/Redis)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 5. Vérifier les logs
docker compose logs -f medusa
```

**Services en PROD :**
- ✅ Medusa Backend : `localhost:9000` (via Docker)
- ❌ PostgreSQL : Supabase (externe)
- ❌ Redis : Vercel KV (externe)

---

## 🔧 Configuration Supabase

### 1. Créer un Projet Supabase

1. Aller sur [supabase.com](https://supabase.com)
2. Créer un nouveau projet
3. Choisir une région proche de votre VPS (ex: `eu-central-1`)
4. Noter le mot de passe de la base de données

### 2. Récupérer l'URL de Connexion

**Dashboard → Project Settings → Database → Connection String**

Format Session Pooler (recommandé) :
```
postgresql://postgres.[PROJECT-ID]:[PASSWORD]@aws-0-eu-central-1.pooler.supabase.com:6543/postgres
```

### 3. Configurer dans .env

```bash
DATABASE_URL=postgresql://postgres.abcdefgh:votre-password@aws-0-eu-central-1.pooler.supabase.com:6543/postgres
```

---

## 🔧 Configuration Vercel KV

### 1. Créer un KV Store

1. Aller sur [vercel.com](https://vercel.com)
2. Storage → Create Database → KV
3. Choisir une région proche (ex: `Frankfurt, Germany`)
4. Créer le store

### 2. Récupérer les Variables

**Dashboard → Storage → KV → [Votre Store] → .env.local**

Vercel fournit :
- `KV_URL` → Utiliser cette variable pour Medusa
- `KV_REST_API_URL` (non utilisé par Medusa)
- `KV_REST_API_TOKEN` (non utilisé par Medusa)

### 3. Configurer dans .env

```bash
REDIS_URL=redis://default:AbC123...xyz@eu1-abc-123.kv.vercel-storage.com:6379
```

**Note :** Utilisez `KV_URL` de Vercel comme valeur pour `REDIS_URL`

---

## 📝 Variables d'Environnement

### Développement (Local)

```bash
# Node
NODE_ENV=development

# PostgreSQL (Docker local)
POSTGRES_DB=medusa-store
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
POSTGRES_PORT=5432

# Redis (Docker local)
REDIS_PORT=6379

# Medusa
MEDUSA_PORT=9000
MEDUSA_BACKEND_URL=http://localhost:9000
STORE_CORS=http://localhost:3000
ADMIN_CORS=http://localhost:7001

# Secrets (dev seulement)
JWT_SECRET=supersecret
COOKIE_SECRET=supersecret
```

### Production (Hetzner)

```bash
# Node
NODE_ENV=production

# Supabase PostgreSQL
DATABASE_URL=postgresql://postgres.[PROJECT-ID]:[PASSWORD]@[REGION].pooler.supabase.com:6543/postgres

# Vercel KV Redis
REDIS_URL=redis://default:[TOKEN]@[REGION].kv.vercel-storage.com:6379

# Medusa
MEDUSA_PORT=9000
MEDUSA_BACKEND_URL=https://api.votredomaine.com
STORE_CORS=https://votredomaine.com
ADMIN_CORS=https://admin.votredomaine.com

# Secrets (FORTS en production!)
JWT_SECRET=[GENERATED_WITH_OPENSSL]
COOKIE_SECRET=[GENERATED_WITH_OPENSSL]

# Optionnel
DISABLE_MEDUSA_ADMIN=false
```

---

## 🛠️ Commandes Utiles

### Développement

```bash
# Démarrer les services
docker compose up -d

# Arrêter les services
docker compose down

# Voir les logs
docker compose logs -f medusa
docker compose logs -f postgres
docker compose logs -f redis

# Reconstruire l'image Medusa
docker compose build medusa

# Redémarrer Medusa seul
docker compose restart medusa

# Accéder au shell Medusa
docker compose exec medusa sh

# Accéder à PostgreSQL
docker compose exec postgres psql -U postgres -d medusa-store

# Nettoyer tout (⚠️ supprime les données!)
docker compose down -v
```

### Production

```bash
# Démarrer (Medusa uniquement)
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# Arrêter
docker compose -f docker-compose.yml -f docker-compose.prod.yml down

# Logs
docker compose -f docker-compose.yml -f docker-compose.prod.yml logs -f medusa

# Rebuild
docker compose -f docker-compose.yml -f docker-compose.prod.yml build medusa

# Restart
docker compose -f docker-compose.yml -f docker-compose.prod.yml restart medusa
```

---

## 🐛 Dépannage

### Problème : Port déjà utilisé

```bash
# Trouver le processus qui utilise le port 9000
lsof -i :9000  # macOS/Linux
netstat -ano | findstr :9000  # Windows

# Changer le port dans .env
MEDUSA_PORT=9001
```

### Problème : Erreur de connexion à la base de données

```bash
# Vérifier que PostgreSQL est démarré
docker compose ps

# Vérifier les logs PostgreSQL
docker compose logs postgres

# Tester la connexion
docker compose exec medusa sh
# Puis dans le container:
nc -zv postgres 5432
```

### Problème : Redis non disponible

```bash
# Vérifier que Redis est démarré
docker compose ps

# Tester la connexion Redis
docker compose exec redis redis-cli ping
# Devrait retourner: PONG
```

### Problème : Medusa ne démarre pas en production

```bash
# Vérifier les variables d'environnement
docker compose -f docker-compose.yml -f docker-compose.prod.yml config

# Vérifier que DATABASE_URL et REDIS_URL sont définis
docker compose exec medusa env | grep -E 'DATABASE_URL|REDIS_URL'

# Tester la connexion Supabase
psql "postgresql://postgres.[PROJECT-ID]:[PASSWORD]@[REGION].pooler.supabase.com:6543/postgres"

# Tester la connexion Vercel KV
redis-cli -u "redis://default:[TOKEN]@[REGION].kv.vercel-storage.com:6379" PING
```

---

## 🔒 Sécurité Production

### Checklist Avant Déploiement

- [ ] Générer des secrets JWT et Cookie forts (`openssl rand -base64 32`)
- [ ] Configurer les CORS correctement (uniquement vos domaines)
- [ ] Utiliser HTTPS pour `MEDUSA_BACKEND_URL`
- [ ] Configurer un reverse proxy (Nginx/Caddy) devant Medusa
- [ ] Activer SSL/TLS sur Supabase
- [ ] Restreindre l'accès à la base de données Supabase par IP
- [ ] Configurer des sauvegardes automatiques Supabase
- [ ] Surveiller les logs avec un service de monitoring

### Configuration Nginx (Exemple)

```nginx
server {
    listen 80;
    server_name api.votredomaine.com;

    # Redirect to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name api.votredomaine.com;

    ssl_certificate /etc/letsencrypt/live/api.votredomaine.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.votredomaine.com/privkey.pem;

    location / {
        proxy_pass http://localhost:9000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

---

## 📊 Monitoring

### Health Check

```bash
# Vérifier la santé de Medusa
curl http://localhost:9000/health

# Devrait retourner: {"status":"ok"}
```

### Docker Health Status

```bash
# Voir le statut de santé des containers
docker compose ps
```

---

## 📚 Ressources

- [Documentation Medusa](https://docs.medusajs.com)
- [Documentation Supabase](https://supabase.com/docs)
- [Documentation Vercel KV](https://vercel.com/docs/storage/vercel-kv)
- [Docker Compose Reference](https://docs.docker.com/compose/compose-file/)
