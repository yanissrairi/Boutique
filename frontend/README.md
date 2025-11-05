# Frontend Next.js - Medusa Storefront

Next.js 16 storefront for Medusa e-commerce backend.

## 🚀 Quick Start

### Development (Docker)

```bash
# From project root
docker compose up -d

# Frontend will be available at http://localhost:3000
# Backend API at http://localhost:9000
```

### Production (Vercel)

The frontend is deployed on Vercel in production, not Docker.

**Environment Variables (Vercel Dashboard):**
- `NEXT_PUBLIC_MEDUSA_BACKEND_URL`: https://api.yourdomain.com
- `NEXT_PUBLIC_BASE_URL`: https://yourdomain.com

## 📋 Prerequisites for Docker

### 1. **Create `next.config.ts`** (REQUIRED)

```bash
cp next.config.ts.example next.config.ts
```

⚠️ **CRITICAL**: The `output: 'standalone'` setting is **required** for Docker builds.

Without it, the Docker build will fail because `.next/standalone` won't exist.

### 2. **Create Health Check Endpoint** (REQUIRED)

```bash
mkdir -p app/api/health
cp app/api/health/route.ts.example app/api/health/route.ts
```

This endpoint is used by Docker health checks to verify the application is running.

### 3. **Package Manager**

This project uses **pnpm**. Make sure you have a `pnpm-lock.yaml` file.

```bash
pnpm install
```

## 🐳 Docker Configuration

### Development
- **Hot Reload**: ✅ Turbopack Fast Refresh enabled
- **Port**: 3000
- **Backend**: http://medusa:9000 (internal Docker network)

### Production (Not Used - Vercel Deployment)
- **Standalone Build**: Minimal production image
- **Multi-stage**: deps → builder → runner
- **Security**: Non-root user (nextjs:nodejs)
- **Size**: ~150MB (vs ~1GB without standalone)

## 🏗️ Project Structure

```
frontend/
├── app/                          # Next.js App Router
│   ├── api/
│   │   └── health/
│   │       └── route.ts         # Health check endpoint (REQUIRED)
│   ├── layout.tsx
│   └── page.tsx
├── public/                       # Static assets
├── next.config.ts                # Next.js config (REQUIRED with output: 'standalone')
├── package.json
├── pnpm-lock.yaml
├── Dockerfile                    # Multi-stage Docker build
└── .dockerignore
```

## ⚙️ Configuration

### Next.js Config (`next.config.ts`)

```typescript
import type { NextConfig } from 'next'

const config: NextConfig = {
  output: 'standalone',        // ← REQUIRED for Docker
  reactCompiler: true,         // React 19 auto-memoization
  experimental: {
    ppr: 'incremental',        // Partial Pre-Rendering
  },
  images: {
    domains: ['your-cdn.com'],
    formats: ['image/avif', 'image/webp'],
  },
}

export default config
```

### Environment Variables

**Development (`.env.local`):**
```bash
NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
NEXT_PUBLIC_BASE_URL=http://localhost:3000
```

**Production (Vercel Dashboard):**
```bash
NEXT_PUBLIC_MEDUSA_BACKEND_URL=https://api.yourdomain.com
NEXT_PUBLIC_BASE_URL=https://yourdomain.com
```

## 🔧 Scripts

```bash
# Development
pnpm dev              # Start dev server (Turbopack)
pnpm build            # Build for production
pnpm start            # Start production server

# Docker
docker compose up -d  # Start all services (from project root)
```

## 📦 Tech Stack

- **Framework**: Next.js 16.0.1
- **React**: 19.2
- **TypeScript**: 5.8+
- **Package Manager**: pnpm
- **Build Tool**: Turbopack (dev) / Next.js Compiler (prod)

## 🚀 Deployment

### Vercel (Production)
1. Connect Git repository to Vercel
2. Set environment variables in Vercel Dashboard
3. Deploy automatically on git push

### Docker (Development Only)
```bash
docker compose up -d
```

## 📚 Resources

- [Next.js Documentation](https://nextjs.org/docs)
- [Medusa Documentation](https://docs.medusajs.com)
- [Docker Setup Guide](../docs/DOCKER_SETUP.md)
