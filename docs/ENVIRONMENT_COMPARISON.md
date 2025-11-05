# Comparaison des Environnements Dev vs Prod

> **💡 Astuce :** Pour un meilleur affichage, utilisez l'aperçu Markdown de VS Code :
> **Raccourci :** `Ctrl+Shift+V` (Windows/Linux) ou `Cmd+Shift+V` (Mac)

---

## 📊 Vue d'Ensemble

### 🔵 DÉVELOPPEMENT (Local)
```
Commande      : docker compose up -d
Fichier .env  : .env ou .env.local
Services      : Postgres + Redis + Medusa (tous en Docker)
Base de données : PostgreSQL (Docker local)
Cache Redis   : Redis (Docker local)
Volumes       : Hot-reload activé
Ports exposés : 5432, 6379, 9000
```

### 🔴 PRODUCTION (Hetzner VPS)
```
Commande      : docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
Fichier .env  : .env.production (renommé en .env sur le serveur)
Services      : Medusa uniquement (Docker)
Base de données : Supabase (managed, externe)
Cache Redis   : Vercel KV (managed, externe)
Volumes       : Aucun (pas de hot-reload)
Ports exposés : 9000 uniquement
```

---

## 🔧 Variables d'Environnement

### ✅ Variables Communes (Dev + Prod)

```bash
# Environment
NODE_ENV                    # dev: development  |  prod: production

# Port
MEDUSA_PORT                 # dev: 9000         |  prod: 9000

# Secrets
JWT_SECRET                  # dev: supersecret ⚠️  |  prod: [OPENSSL GENERATED] ✅
COOKIE_SECRET               # dev: supersecret ⚠️  |  prod: [OPENSSL GENERATED] ✅
```

### 🔵 Variables UNIQUEMENT en DEV

```bash
# PostgreSQL (Container Docker)
POSTGRES_DB=medusa-store           # Nom de la base de données
POSTGRES_USER=postgres             # Utilisateur PostgreSQL
POSTGRES_PASSWORD=postgres         # Mot de passe PostgreSQL
POSTGRES_PORT=5432                 # Port exposé (pour pgAdmin, DBeaver, etc.)

# Redis (Container Docker)
REDIS_PORT=6379                    # Port exposé (pour RedisInsight, etc.)

# Note : DATABASE_URL est construit automatiquement en dev
# → postgresql://postgres:postgres@postgres:5432/medusa-store
```

### 🔴 Variables UNIQUEMENT en PROD

```bash
# Supabase PostgreSQL (Externe)
DATABASE_URL=postgresql://postgres.[PROJECT-ID]:[PASSWORD]@[REGION].pooler.supabase.com:6543/postgres

# Vercel KV Redis (Externe)
REDIS_URL=redis://default:[TOKEN]@[REGION].kv.vercel-storage.com:6379

# URLs Publiques (HTTPS obligatoire!)
MEDUSA_BACKEND_URL=https://api.yourdomain.com
STORE_CORS=https://yourdomain.com,https://www.yourdomain.com
ADMIN_CORS=https://admin.yourdomain.com

# Optionnel
DISABLE_MEDUSA_ADMIN=false         # true pour désactiver l'admin
```

---

## 🏗️ Architecture des Services

### 🔵 Développement (Local Docker)

```
┌─────────────────────────────────────────────────┐
│           Docker Compose Environment            │
│  (Auto-charge docker-compose.override.yml)      │
│                                                  │
│  ┌──────────────┐  ┌──────────┐  ┌──────────┐  │
│  │ PostgreSQL   │  │  Redis   │  │  Medusa  │  │
│  │ (Container)  │  │(Container│  │(Container│  │
│  │              │  │          │  │          │  │
│  │ Port: 5432   │  │Port: 6379│  │Port: 9000│  │
│  └──────┬───────┘  └────┬─────┘  └────┬─────┘  │
│         │               │             │         │
│         └───────────────┴─────────────┘         │
│           Docker Network: medusa_network        │
└─────────────────────────────────────────────────┘
                    │
                    ▼
         Accessible sur localhost:
         • PostgreSQL: localhost:5432
         • Redis:      localhost:6379
         • Medusa:     localhost:9000

Connexions internes (dans Docker):
  Medusa → PostgreSQL : postgres:5432
  Medusa → Redis      : redis:6379
```

### 🔴 Production (Hetzner VPS)

```
┌──────────────────────┐
│   Hetzner VPS        │
│                      │         ┌─────────────────────┐
│  ┌────────────────┐ │         │   Supabase          │
│  │     Medusa     │─┼────────▶│   PostgreSQL        │
│  │   (Container)  │ │  HTTPS  │   (EU-Central-1)    │
│  │                │ │         │   Port: 6543        │
│  │   Port: 9000   │ │         │   (Session Pooler)  │
│  └───────┬────────┘ │         └─────────────────────┘
│          │           │
│          │           │         ┌─────────────────────┐
│          │           │         │   Vercel KV         │
│          └───────────┼────────▶│   Redis             │
│                      │  HTTPS  │   (Frankfurt)       │
│  ┌────────────────┐ │         │   Port: 6379        │
│  │  Nginx/Caddy   │ │         └─────────────────────┘
│  │ Reverse Proxy  │ │
│  │   Port: 443    │ │
│  └───────┬────────┘ │
└──────────┼──────────┘
           │
           ▼
    HTTPS Public Access:
    https://api.yourdomain.com

Connexions:
  Medusa → Supabase   : DATABASE_URL (TLS/SSL)
  Medusa → Vercel KV  : REDIS_URL (TLS/SSL)
  Public → Nginx      : HTTPS (Let's Encrypt)
  Nginx  → Medusa     : HTTP localhost:9000
```

---

## 🔒 Sécurité

### 🔵 Développement (Permissif pour faciliter le dev)

```
Secrets          : Valeurs simples (supersecret) ✅ OK en local
HTTPS            : Non activé (HTTP) ✅ OK en local
Ports exposés    : Tous (5432, 6379, 9000) ✅ OK en local
CORS             : Localhost URLs ✅ OK en local
PostgreSQL       : Optimisations dev (fsync=off) ⚠️ Risque perte données
```

### 🔴 Production (Sécurité maximale)

```
✅ Secrets          : Générés avec openssl (32+ caractères)
✅ HTTPS            : Obligatoire via Nginx/Caddy + Let's Encrypt
✅ Ports exposés    : 9000 uniquement (via reverse proxy)
✅ CORS             : Domaines spécifiques uniquement
✅ Base de données  : Supabase avec SSL/TLS
✅ Redis            : Vercel KV avec authentification
```

---

## 🚀 Commandes de Déploiement

### 🔵 Développement

```bash
# 1. Créer l'environnement
cp .env.example .env

# 2. (Optionnel) Personnaliser les valeurs
nano .env

# 3. Démarrer tous les services
docker compose up -d

# 4. Vérifier les logs
docker compose logs -f medusa

# 5. Tester
curl http://localhost:9000/health
# Devrait retourner: {"status":"ok"}

# Accès:
# • API:   http://localhost:9000
# • Admin: http://localhost:9000/app
```

### 🔴 Production

```bash
#──────────────────────────────────────────────
# Sur votre machine LOCALE
#──────────────────────────────────────────────

# 1. Préparer l'environnement
cp .env.production .env.prod-ready

# 2. Générer les secrets
openssl rand -base64 32  # Copier pour JWT_SECRET
openssl rand -base64 32  # Copier pour COOKIE_SECRET

# 3. Éditer et remplir les vraies valeurs
nano .env.prod-ready
# → Remplir DATABASE_URL (Supabase)
# → Remplir REDIS_URL (Vercel KV)
# → Remplir JWT_SECRET et COOKIE_SECRET
# → Remplir les URLs publiques

# 4. Transférer sur le VPS
scp .env.prod-ready user@your-vps:/path/to/project/.env
scp -r . user@your-vps:/path/to/project/

#──────────────────────────────────────────────
# Sur le VPS (SSH)
#──────────────────────────────────────────────

ssh user@your-vps
cd /path/to/project

# 5. Vérifier la configuration
docker compose -f docker-compose.yml -f docker-compose.prod.yml config

# 6. Démarrer en production
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d

# 7. Vérifier les logs
docker compose logs -f medusa

# 8. Tester
curl http://localhost:9000/health
# Devrait retourner: {"status":"ok"}
```

---

## 📋 Checklist de Migration Dev → Prod

### Avant le Déploiement

**Configuration Supabase**
```
☐ Projet créé sur supabase.com
☐ Région choisie proche du VPS (ex: eu-central-1)
☐ Connection string récupérée (Session Pooler)
☐ SSL/TLS activé
☐ (Optionnel) IP du VPS autorisée
```

**Configuration Vercel KV**
```
☐ Store créé sur vercel.com
☐ Région choisie proche du VPS (ex: Frankfurt)
☐ KV_URL récupérée depuis le dashboard
```

**Secrets de Sécurité**
```
☐ JWT_SECRET généré (openssl rand -base64 32)
☐ COOKIE_SECRET généré (openssl rand -base64 32)
☐ Secrets sauvegardés en lieu sûr
```

**Configuration DNS**
```
☐ api.yourdomain.com → IP du VPS
☐ (Optionnel) admin.yourdomain.com → IP du VPS
```

**Reverse Proxy**
```
☐ Nginx ou Caddy installé
☐ Certificat SSL configuré (Let's Encrypt)
☐ Proxy configuré vers localhost:9000
```

### Variables à Modifier

```diff
# Environment
- NODE_ENV=development
+ NODE_ENV=production

# Database (construit auto en dev)
+ DATABASE_URL=postgresql://postgres.xxx:xxx@xxx.pooler.supabase.com:6543/postgres

# Redis (construit auto en dev)
+ REDIS_URL=redis://default:xxx@xxx.kv.vercel-storage.com:6379

# URLs
- MEDUSA_BACKEND_URL=http://localhost:9000
+ MEDUSA_BACKEND_URL=https://api.yourdomain.com

- STORE_CORS=http://localhost:3000
+ STORE_CORS=https://yourdomain.com

- ADMIN_CORS=http://localhost:7001
+ ADMIN_CORS=https://admin.yourdomain.com

# Secrets
- JWT_SECRET=supersecret
+ JWT_SECRET=[GENERATED_32_CHARS]

- COOKIE_SECRET=supersecret
+ COOKIE_SECRET=[GENERATED_32_CHARS]
```

### Après le Déploiement

**Vérifications Techniques**
```bash
☐ Health check OK
   curl http://localhost:9000/health

☐ Connexion Supabase OK
   docker compose logs medusa | grep -i database

☐ Connexion Vercel KV OK
   docker compose logs medusa | grep -i redis

☐ Admin accessible
   https://admin.yourdomain.com/app

☐ API accessible
   https://api.yourdomain.com/store/products
```

**Monitoring**
```
☐ Logs Docker surveillés
☐ Métriques Supabase configurées
☐ Métriques Vercel KV configurées
☐ Uptime monitoring activé (ex: UptimeRobot)
```

**Sauvegardes**
```
☐ Sauvegardes auto Supabase activées
☐ .env sauvegardé en lieu sûr (1Password, etc.)
☐ Documentation à jour
```

---

## 🔍 Dépannage

### Problème : Medusa ne peut pas se connecter à Supabase

```bash
# 1. Tester la connexion directement
psql "postgresql://postgres.[PROJECT-ID]:[PASSWORD]@[REGION].pooler.supabase.com:6543/postgres"

# 2. Vérifier les logs Medusa
docker compose logs medusa | grep -i "database\|postgres\|connection"

# 3. Vérifier la variable
docker compose exec medusa env | grep DATABASE_URL
```

**Solutions possibles :**
- ✅ Vérifier le format de `DATABASE_URL` (doit commencer par `postgresql://`)
- ✅ Vérifier que l'IP du VPS est autorisée dans Supabase (Settings > Database > Network)
- ✅ Vérifier que SSL est activé (`sslmode=require` si nécessaire)
- ✅ Tester avec `psql` pour isoler le problème

### Problème : Medusa ne peut pas se connecter à Vercel KV

```bash
# 1. Tester avec redis-cli
redis-cli -u "${REDIS_URL}" PING
# Devrait retourner: PONG

# 2. Vérifier les logs
docker compose logs medusa | grep -i "redis\|cache"

# 3. Vérifier la variable
docker compose exec medusa env | grep REDIS_URL
```

**Solutions possibles :**
- ✅ Vérifier le format de `REDIS_URL` (doit commencer par `redis://`)
- ✅ Vérifier que le token est correct (copié depuis Vercel)
- ✅ Tester la connexion avec `redis-cli` directement
- ✅ Vérifier que la région Vercel KV est accessible

### Problème : Erreurs CORS

```bash
# 1. Vérifier les variables CORS
docker compose exec medusa env | grep CORS

# 2. Vérifier les logs
docker compose logs medusa | grep -i cors
```

**Solutions possibles :**
- ✅ Vérifier que les URLs utilisent HTTPS (pas HTTP) en prod
- ✅ Vérifier qu'il n'y a pas d'espaces dans les valeurs
- ✅ Vérifier que les domaines sont séparés par des virgules
- ✅ Exemple correct : `STORE_CORS=https://site.com,https://www.site.com`

### Problème : Services PostgreSQL/Redis démarrent en prod

```bash
# Vérifier que docker-compose.prod.yml est bien chargé
docker compose -f docker-compose.yml -f docker-compose.prod.yml config | grep replicas

# Devrait afficher:
# replicas: 0  (pour postgres et redis)
```

**Solutions possibles :**
- ✅ Vérifier que vous utilisez bien `-f docker-compose.prod.yml`
- ✅ Vérifier le fichier `docker-compose.prod.yml` (lignes 23-24 et 30-31)

---

## 📚 Ressources

### Documentation Officielle
- [Medusa Documentation](https://docs.medusajs.com)
- [Supabase Connection Pooler](https://supabase.com/docs/guides/database/connecting-to-postgres#connection-pooler)
- [Vercel KV Documentation](https://vercel.com/docs/storage/vercel-kv)
- [Docker Compose Override](https://docs.docker.com/compose/extends/)

### Outils Recommandés
- [pgAdmin](https://www.pgadmin.org/) - Client PostgreSQL graphique
- [DBeaver](https://dbeaver.io/) - Client SQL universel
- [RedisInsight](https://redis.com/redis-enterprise/redis-insight/) - Client Redis graphique
- [Another Redis Desktop Manager](https://github.com/qishibo/AnotherRedisDesktopManager) - Alternative Redis

### Monitoring
- [UptimeRobot](https://uptimerobot.com/) - Monitoring uptime gratuit
- [Better Stack](https://betterstack.com/) - Monitoring et logs
- [Sentry](https://sentry.io/) - Error tracking
