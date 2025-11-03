# Stack Technique - Boutique Macramé 2025

## 📦 Technologies Utilisées

---

## 🎨 Frontend

### Next.js
**Version**: 16.0.1 (Stable)
**Type**: The React Framework for Production
**Licence**: MIT
**Site**: https://nextjs.org

**Rôle**: Framework principal pour le frontend, SSR et commerce

**Caractéristiques Next.js 16**:
- **Turbopack (Stable)**: 2-5x faster builds, 5-10x faster Fast Refresh
- **React 19.2 Support**: React Compiler intégré (auto-memoization)
- **use cache Directive**: Caching granulaire de pages/composants/fonctions
- **Partial Pre-Rendering (PPR)**: Instant navigation avec cache intelligent
- **Layout Deduplication**: Shared layouts téléchargés 1x au lieu de Nx
- **Incremental Prefetching**: Seules les parties non-cached sont préchargées
- **proxy.ts**: Remplacement de middleware.ts (Node.js runtime)
- **Enhanced Dev Logging**: Visibilité complète des temps de build

**Installation**:
```bash
npx create-next-app@latest --app
# Ou avec Medusa starter
npx create-medusa-app@latest
```

**Configuration** (`next.config.ts`):
```typescript
import type { NextConfig } from 'next'

const config: NextConfig = {
  reactCompiler: true,           // React 19 Compiler (auto-memoization)
  turbo: {                        // Turbopack config
    resolveAlias: {
      '@': './src'
    }
  },
  experimental: {
    ppr: 'incremental',           // Partial Pre-Rendering
    reactCompiler: true,
  },
  images: {
    domains: ['pub-xxxxxxxxxx.r2.dev'],  // Cloudflare R2 bucket
    formats: ['image/avif', 'image/webp']
  }
}

export default config
```

**Pourquoi Next.js 16 pour e-commerce**:
- ✅ **Starter Medusa officiel** (gain 3 semaines développement)
- ✅ **Performance 2-5x** (Turbopack vs Webpack)
- ✅ **Layout Deduplication** (crucial pour 50+ product links)
- ✅ **use cache** (réduit DB load pour product listings)
- ✅ **React 19 Compiler** (0% re-renders inutiles)
- ✅ **Production-ready** (utilisé par milliers d'e-commerce)
- ✅ **SEO optimal** (PPR + SSR + ISR)
- ✅ **Image Optimization** (next/image avec Cloudflare R2)
- ✅ **Server Actions** (cart management type-safe)

**Breaking Changes (from 15.x)**:
- Node.js 20.9.0+ requis (18.x deprecated) - **Recommandé: Node.js 24 LTS**
- TypeScript 5.8+ requis
- `params`, `searchParams` nécessitent `await` (async)
- `cookies()`, `headers()` nécessitent `await`
- AMP support retiré

---

### TanStack Query
**Version**: v5.x (Medusa 2.0 starter uses v4.22.0)
**Type**: Client-Side Data Fetching & State Management
**Licence**: MIT
**Site**: https://tanstack.com/query

**Rôle**: Gestion des interactions client et cache côté client (complémentaire aux Server Components)

**⚠️ Important**: TanStack Query est **complémentaire** à Next.js 16 Server Components, pas un remplacement:
- **Server Components + `use cache`**: Initial data, SEO pages, données static/semi-static
- **TanStack Query**: Interactions client, optimistic updates, données temps réel

**Cas d'usage e-commerce spécifiques**:
1. **Gestion du panier** (optimistic updates pour feedback instantané)
2. **Recherche/Filtres** (debouncing, évite appels serveur excessifs)
3. **Inventaire temps réel** (polling pour stock disponible)
4. **Session client** (données persistantes entre navigations)

**Caractéristiques**:
- Cache intelligent avec invalidation automatique
- Optimistic updates (UX instant pour cart/wishlist)
- Retry automatique sur erreur
- Background refetching
- DevTools intégrés

**Installation**:
```bash
npm install @tanstack/react-query @tanstack/react-query-devtools
```

**Configuration** (`app/providers.tsx`):
```typescript
'use client'

import { QueryClient, QueryClientProvider } from '@tanstack/react-query'
import { ReactQueryDevtools } from '@tanstack/react-query-devtools'
import { useState } from 'react'

export function Providers({ children }: { children: React.ReactNode }) {
  const [queryClient] = useState(() => new QueryClient({
    defaultOptions: {
      queries: {
        staleTime: 1000 * 60 * 5,     // 5 minutes
        gcTime: 1000 * 60 * 30,        // 30 minutes
        retry: 2,
        refetchOnWindowFocus: false
      }
    }
  }))

  return (
    <QueryClientProvider client={queryClient}>
      {children}
      <ReactQueryDevtools initialIsOpen={false} />
    </QueryClientProvider>
  )
}
```

**Architecture Hybride** (Server Components + TanStack Query):
```typescript
// ✅ Server Component (SEO, initial data)
// app/products/[id]/page.tsx
import { unstable_cache as cache } from 'next/cache'

const getProduct = cache(
  async (id: string) => {
    return await medusa.products.retrieve(id)
  },
  ['product'],
  { revalidate: 3600 } // 1 hour
)

export default async function ProductPage({ params }: { params: { id: string } }) {
  const product = await getProduct(params.id)

  return (
    <div>
      <ProductDetails product={product} />
      <AddToCartButton productId={product.id} /> {/* Client Component */}
    </div>
  )
}

// ✅ Client Component avec TanStack Query (cart interactions)
// components/add-to-cart-button.tsx
'use client'

import { useMutation, useQueryClient } from '@tanstack/react-query'
import { addToCart } from '@/actions/cart'

export function AddToCartButton({ productId }: { productId: string }) {
  const queryClient = useQueryClient()

  const { mutate, isPending } = useMutation({
    mutationFn: (quantity: number) => addToCart(productId, quantity),

    // Optimistic update (UI instant)
    onMutate: async (quantity) => {
      await queryClient.cancelQueries({ queryKey: ['cart'] })

      const previousCart = queryClient.getQueryData(['cart'])
      queryClient.setQueryData(['cart'], (old: any) => ({
        ...old,
        items: [...old.items, { productId, quantity }]
      }))

      return { previousCart }
    },

    // Rollback si erreur
    onError: (err, variables, context) => {
      if (context?.previousCart) {
        queryClient.setQueryData(['cart'], context.previousCart)
      }
    },

    // Sync avec serveur
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['cart'] })
    }
  })

  return (
    <button onClick={() => mutate(1)} disabled={isPending}>
      {isPending ? 'Ajout...' : 'Ajouter au panier'}
    </button>
  )
}
```

**Utilisation avec medusa-react** (hooks pré-configurés):
```typescript
import { useCart, useCreateLineItem } from 'medusa-react'

function CartComponent() {
  const { cart, isLoading } = useCart()
  const createLineItem = useCreateLineItem(cart?.id)

  // Hooks built on TanStack Query
}
```

---

### React 19
**Version**: 19.2 (Stable)
**Type**: UI Library
**Licence**: MIT
**Site**: https://react.dev

**Rôle**: Construction des composants UI avec optimisations automatiques

**Nouvelles features React 19 (critiques pour e-commerce)**:

**1. React Compiler** (auto-optimization):
- Memoization automatique sans `useMemo`/`useCallback`
- 38% d'amélioration de performance moyenne
- Élimine re-renders inutiles automatiquement
- Activé via `reactCompiler: true` dans Next.js 16

**2. Actions API** (mutations simplifiées):
- Gestion automatique des états pending/error/success
- Parfait pour cart mutations, checkout forms
- Élimine patterns `useEffect` + `fetch` manuels
- Server Actions directement appelables depuis forms

**3. Nouveaux Hooks E-commerce**:

**useActionState** (gestion état des actions):
```typescript
'use client'
import { useActionState } from 'react'
import { addToCart } from '@/actions/cart'

function AddToCartForm({ productId }: { productId: string }) {
  const [state, formAction, isPending] = useActionState(addToCart, null)

  return (
    <form action={formAction}>
      <input type="hidden" name="productId" value={productId} />
      <button disabled={isPending}>
        {isPending ? 'Ajout...' : 'Ajouter au panier'}
      </button>
      {state?.error && <p className="error">{state.error}</p>}
    </form>
  )
}
```

**useFormStatus** (statut soumission):
```typescript
'use client'
import { useFormStatus } from 'react'

function SubmitButton() {
  const { pending, data } = useFormStatus()

  return (
    <button disabled={pending}>
      {pending ? 'Traitement...' : 'Commander'}
    </button>
  )
}
```

**useOptimistic** (updates instantanées):
```typescript
'use client'
import { useOptimistic } from 'react'

function Cart({ items }: { items: CartItem[] }) {
  const [optimisticItems, addOptimisticItem] = useOptimistic(
    items,
    (state, newItem: CartItem) => [...state, newItem]
  )

  async function handleAddToCart(item: CartItem) {
    // UI update instantané
    addOptimisticItem(item)

    // Appel serveur (rollback automatique si erreur)
    await addToCartAction(item)
  }

  return (
    <div>
      {optimisticItems.map(item => (
        <CartItem key={item.id} item={item} />
      ))}
    </div>
  )
}
```

**4. Autres features utilisées**:
- **React Server Components** (stable)
- **use() hook** pour async data
- **Ref as Prop** (plus de forwardRef)
- **Transitions améliorées**
- **Automatic batching**
- **Better hydration errors**

**Installation**:
```bash
npm install react@latest react-dom@latest
# React 19.2 installé automatiquement avec Next.js 16
```

**Pourquoi React 19 pour e-commerce**:
- ✅ **React Compiler** (38% perf boost, 0 config)
- ✅ **useOptimistic** (instant cart feedback)
- ✅ **Actions API** (forms simplifiés)
- ✅ **Server Components** (SEO + performance)
- ✅ **useActionState** (checkout flow simplifié)

---

### TypeScript
**Version**: 5.8+ (Latest: 5.9)
**Type**: Langage de programmation
**Licence**: Apache 2.0
**Site**: https://www.typescriptlang.org

**Rôle**: Type safety pour tout le projet avec support Next.js 16

**Configuration Next.js 16** (`tsconfig.json`):
```json
{
  "compilerOptions": {
    // Build & Module
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "jsx": "preserve",                    // Next.js transforme le JSX

    // Type Checking
    "strict": true,
    "noEmit": true,
    "isolatedModules": true,
    "skipLibCheck": true,

    // Module Resolution
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,

    // Performance
    "incremental": true,                  // Faster subsequent builds

    // Path Mapping
    "paths": {
      "@/*": ["./src/*"]                  // @ alias pour imports
    },

    // Next.js Plugin
    "plugins": [
      { "name": "next" }                  // Type-safe routing
    ]
  },
  "include": [
    "next-env.d.ts",
    ".next/types/**/*.ts",                // Next.js generated types
    "**/*.ts",
    "**/*.tsx"
  ],
  "exclude": [
    "node_modules"
  ]
}
```

**Fonctionnalités Next.js 16**:
- **Type-safe routing**: `.next/types` génère types pour routes
- **@ alias**: Imports simplifiés (`@/components/Button`)
- **Plugin Next.js**: Auto-completion pour Link href
- **Incremental builds**: Recompilation rapide

**Exemple type-safe routing**:
```typescript
import Link from 'next/link'

// ✅ Type-safe: Next.js génère types pour toutes les routes
<Link href="/products/123">Product</Link>

// ❌ TypeScript error si route n'existe pas
<Link href="/invalid-route">Invalid</Link>

// ✅ Auto-completion pour routes dynamiques
<Link href={`/products/${productId}`}>Product</Link>
```

**Exemple @ alias**:
```typescript
// ❌ Avant: imports relatifs compliqués
import { Button } from '../../../components/ui/Button'

// ✅ Après: imports avec @ alias
import { Button } from '@/components/ui/Button'
```

---

### Tailwind CSS
**Version**: 4.0 (Stable - Janvier 2025)
**Type**: CSS Framework (CSS-first configuration)
**Licence**: MIT
**Site**: https://tailwindcss.com

**Rôle**: Styling et design system avec configuration CSS native

**⚠️ Breaking Changes v4.0:**
- ❌ Plus de `tailwind.config.js` → Configuration **CSS-first**
- ❌ Plus de `npx tailwindcss init`
- ✅ Nouveau: `@theme` pour variables CSS
- ✅ Nouveau: `@utility` pour custom utilities
- ✅ Built-in import support (plus besoin de postcss-import)

**Installation Next.js 16**:
```bash
npm install tailwindcss @tailwindcss/postcss postcss
```

**Configuration PostCSS** (`postcss.config.mjs`):
```javascript
const config = {
  plugins: {
    '@tailwindcss/postcss': {}
  }
}

export default config
```

**Configuration CSS-first** (`app/globals.css`):
```css
@import "tailwindcss";

/* Theme customization avec @theme */
@theme {
  /* Colors - Palette macramé */
  --color-primary: #8B7355;      /* Beige macramé */
  --color-secondary: #F5E6D3;    /* Crème */
  --color-accent: #6B5744;       /* Marron foncé */
  --color-neutral-50: #fafafa;
  --color-neutral-900: #171717;

  /* Typography */
  --font-display: "Satoshi", "sans-serif";
  --font-sans: "Inter", "sans-serif";

  /* Breakpoints */
  --breakpoint-3xl: 1920px;

  /* Spacing */
  --spacing-18: 4.5rem;

  /* Animations */
  --ease-fluid: cubic-bezier(0.3, 0, 0, 1);
  --ease-snappy: cubic-bezier(0.2, 0, 0, 1);
}

/* Custom utilities avec @utility */
@utility card-shadow {
  box-shadow: 0 4px 6px -1px rgb(0 0 0 / 0.1),
              0 2px 4px -2px rgb(0 0 0 / 0.1);
}

@utility product-card {
  @apply rounded-lg overflow-hidden card-shadow
         transition-transform hover:scale-105;
}
```

**Nouvelles fonctionnalités Tailwind 4.0**:
1. **3D Transforms**: `rotate-x-45`, `rotate-y-90`
2. **@starting-style**: Transitions enter/exit sans JavaScript
3. **Cascade layers**: Meilleure organisation CSS
4. **color-mix()**: Mélange de couleurs natif
5. **@property**: Custom properties enregistrées

**Exemple 3D Transform** (Hero macramé):
```tsx
<div className="rotate-y-12 rotate-x-6 transition-transform hover:rotate-y-0">
  <img src="/macrame-hero.jpg" alt="Macramé" />
</div>
```

**Exemple @starting-style** (Animation entrée produit):
```css
@starting-style {
  .product-card {
    opacity: 0;
    transform: translateY(2rem);
  }
}

.product-card {
  opacity: 1;
  transform: translateY(0);
  transition: all 0.3s var(--ease-fluid);
}
```

**Intégration Next.js 16** (`app/layout.tsx`):
```typescript
import './globals.css'

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="fr">
      <body className="bg-secondary text-neutral-900 antialiased">
        {children}
      </body>
    </html>
  )
}
```

**Utilisation dans composants**:
```tsx
// Produit macramé avec design system
export function ProductCard({ product }: { product: Product }) {
  return (
    <div className="product-card bg-white p-6">
      <img
        src={product.image}
        alt={product.title}
        className="w-full h-64 object-cover rounded-md"
      />
      <h3 className="mt-4 font-display text-xl text-primary">
        {product.title}
      </h3>
      <p className="mt-2 text-neutral-600">{product.price}€</p>
      <button className="mt-4 w-full bg-accent text-white py-2 rounded-md
                         transition-colors hover:bg-primary">
        Ajouter au panier
      </button>
    </div>
  )
}
```

**Migration depuis v3.x**:
- Convertir `tailwind.config.js` → `@theme` en CSS
- `colors.blue[500]` → `--color-blue-500`
- `theme.extend.fontFamily` → `--font-*`
- Plugins → `@utility` ou `@layer`

**Pourquoi Tailwind 4.0 pour e-commerce**:
- ✅ **CSS-first**: Configuration plus simple et native
- ✅ **Performance**: Built-in optimizations
- ✅ **3D transforms**: Animations produits immersives
- ✅ **@starting-style**: Transitions fluides
- ✅ **Cascade layers**: Meilleure organisation du CSS

---

### Zod
**Version**: 3.x
**Type**: Schema Validation
**Licence**: MIT
**Site**: https://zod.dev

**Rôle**: Validation des données et type inference

**Installation**:
```bash
npm install zod
```

**Exemple d'utilisation**:
```typescript
import { z } from 'zod'

export const ProductSchema = z.object({
  name: z.string().min(1).max(200),
  price: z.number().positive(),
  description: z.string().max(2000),
  images: z.array(z.string().url()).min(1)
})

type Product = z.infer<typeof ProductSchema>
```

---

### Turbopack
**Version**: Stable (included in Next.js 16)
**Type**: Rust-powered Bundler
**Licence**: MIT
**Site**: https://turbo.build/pack

**Rôle**: Bundler ultra-rapide pour Next.js 16

**Caractéristiques**:
- **2-5x faster builds** vs Webpack
- **5-10x faster Fast Refresh** (HMR)
- **Incremental compilation** (cache intelligent)
- **Support natif TypeScript**
- **CSS Modules, Sass, PostCSS** intégrés

**Performance (Medium e-commerce app: 150 routes, 2,500 modules)**:
```
Cold Start:
- Webpack: 8.2s
- Turbopack: 0.9s
→ 9.1x faster

Hot Reload (HMR):
- Webpack: 200-500ms
- Turbopack: 20-50ms
→ 10x faster
```

**Configuration** (`next.config.ts`):
```typescript
import type { NextConfig } from 'next'

const config: NextConfig = {
  turbo: {
    resolveAlias: {
      '@': './src',
      '@components': './src/components',
      '@lib': './src/lib'
    },
    rules: {
      '*.svg': {
        loaders: ['@svgr/webpack'],
        as: '*.js'
      }
    }
  }
}

export default config
```

**Activation automatique** (Next.js 16):
```bash
# Dev (Turbopack activé par défaut)
npm run dev

# Build production (Turbopack)
npm run build
```

---

## 🔧 Backend

### Medusa.js
**Version**: 2.x (Latest: 2.11.2)
**Type**: Headless Commerce Platform
**Licence**: MIT
**Site**: https://medusajs.com

**Rôle**: Backend e-commerce complet avec admin panel

**Caractéristiques**:
- Admin dashboard React inclus
- API REST + GraphQL
- Gestion produits, commandes, clients
- Intégrations paiements (Stripe, PayPal)
- Multi-currency & multi-region
- Extensible via plugins
- Medusa Cache (2.2x faster APIs - built-in)

**Installation**:
```bash
npx create-medusa-app@latest
```

**⚠️ Compatibilité Next.js 16 + React 19**:
- **Starter officiel**: Supporte Next.js 15 + React 18
- **Pour Next.js 16 + React 19**: Utiliser `@medusajs/js-sdk` directement avec TanStack Query
- `medusa-react` est **déprécié** (remplacé par JS SDK)

**Intégration Next.js 16**:
```typescript
// lib/medusa.ts
import Medusa from "@medusajs/js-sdk"

export const sdk = new Medusa({
  baseUrl: process.env.MEDUSA_BACKEND_URL || "http://localhost:9000",
  publishableKey: process.env.NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY
})
```

```typescript
// Client component avec TanStack Query
'use client'
import { useQuery } from '@tanstack/react-query'
import { sdk } from '@/lib/medusa'

function Products() {
  const { data } = useQuery({
    queryKey: ['products'],
    queryFn: async () => {
      const { products } = await sdk.store.product.list()
      return products
    }
  })
}
```

**Architecture modulaire**:
- Product Module (produits, variants, catégories)
- Cart Module (panier, checkout)
- Order Module (commandes, fulfillment)
- Customer Module (clients, auth)
- Payment Module (Stripe integration)
- Region Module (zones géo, taxes)

---

### Node.js
**Version**: 24.x LTS (Krypton) - Latest: 24.11.0
**Type**: Runtime JavaScript
**Licence**: MIT
**Site**: https://nodejs.org

**Rôle**: Environnement d'exécution pour Medusa backend et Next.js 16

**Pourquoi Node.js 24 LTS**:
- ✅ **Active LTS** jusqu'en avril 2028 (vs avril 2026 pour v20)
- ✅ **Next.js 16 compatible** (minimum: 20.9.0+ - recommandé: 24.x)
- ✅ **Medusa.js compatible** (supporte Node.js 20+, optimisé pour 24.x)
- ✅ **V8 14.x** : Performance JavaScript améliorée
- ✅ **Security enhancements** : Runtime plus sécurisé
- ✅ **Pas Bun** : Next.js 16 ne supporte pas Bun runtime (novembre 2025)

**Installation**:
```bash
# Via nvm (recommandé)
nvm install 24
nvm use 24

# Vérifier version
node --version  # v24.11.0+
```

**Alternatives considérées**:
- ❌ **Node.js 20 LTS**: Support se termine avril 2026 (trop proche)
- ❌ **Bun**: Next.js 16 incompatible + Medusa compatibilité partielle (2025)
- ⚠️ **Node.js 25 Current**: Pas LTS, expérimental (fin support avril 2026)

---

### Prisma
**Version**: 6.x (Latest: 6.18.0)
**Type**: ORM (Object-Relational Mapping)
**Licence**: Apache 2.0
**Site**: https://www.prisma.io

**Rôle**: Interaction type-safe avec PostgreSQL pour extensions et services personnalisés

**Note importante**: Medusa.js utilise son propre data layer, Prisma est pour vos services/extensions personnalisés

**Nouveautés Prisma 6 (2025)**:
- ✅ **Prisma Config (GA)**: `prisma.config.ts` pour configuration TypeScript (obligatoire v7)
- ✅ **Multi-schema (GA)**: Organisation du schéma en plusieurs fichiers
- ✅ **Uint8Array pour Bytes**: Remplace `Buffer` (breaking change)
- ✅ **Prisma Studio amélioré**: Interface GUI modernisée
- ✅ **Performance optimisée**: Requêtes plus rapides avec nouveau query compiler
- ✅ **TypeScript 5.1+**: Support complet des dernières fonctionnalités TS

**Exigences**:
- Node.js 18.18.0+ (✅ Node 24 compatible)
- TypeScript 5.1.0+ (✅ TypeScript 5.8+ compatible)
- PostgreSQL 12+ (✅ compatible)

**Caractéristiques**:
- Schema-first approach avec type safety complète
- Type-safe queries auto-générées
- Migrations automatiques avec Prisma Migrate
- Prisma Studio (GUI) pour visualisation données
- Support TypeScript/ESM natif
- Connection pooling et caching intégrés

**Schema exemple**:
```prisma
// schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

generator client {
  provider = "prisma-client-js"
  output   = "../src/generated/prisma"
}

model CustomProduct {
  id          String   @id @default(uuid())
  title       String
  handle      String   @unique
  description String?
  metadata    Json?
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

**Breaking Changes v5 → v6**:
```typescript
// ❌ Avant (v5): Buffer
const data = Buffer.from([1, 2, 3, 4])

// ✅ Après (v6): Uint8Array
const data = Uint8Array.from([1, 2, 3, 4])
```

**Commandes utiles**:
```bash
npx prisma migrate dev    # Créer migration
npx prisma generate       # Générer client
npx prisma studio         # Ouvrir GUI
```

---

## 🗄️ Base de Données

### PostgreSQL
**Version**: 17.x (Latest: 17.5)
**Type**: RDBMS (Relational Database)
**Licence**: PostgreSQL License (open-source)
**Site**: https://www.postgresql.org

**Rôle**: Stockage des données produits, commandes, clients pour Medusa.js

**Nouveautés PostgreSQL 17 (2025)**:
- ✅ **JSON_TABLE()**: Requêtes JSON comme tables relationnelles (parfait pour metadata produits)
- ✅ **2x Write Throughput**: Performance doublée pour opérations d'écriture haute concurrence
- ✅ **Streaming I/O**: Scans séquentiels plus rapides pour analytics/reporting
- ✅ **transaction_timeout**: Contrôle durée max des transactions
- ✅ **Improved Query Planning**: CTE optimisés, skipping checks inutiles
- ✅ **Better Logical Replication**: Migrations slots + subscriptions améliorées

**Pourquoi PostgreSQL 17 pour e-commerce**:
- ✅ **JSON_TABLE**: Queries complexes sur metadata produits (couleurs, tailles, attributs)
- ✅ **JSONB Performance**: Plus rapide pour filtres/recherche sur attributs variables
- ✅ **2x Write Speed**: Gère pics de commandes (Black Friday, promotions)
- ✅ **Prisma 6 Compatible**: Prisma Postgres basé sur PostgreSQL 17
- ✅ **Medusa Compatible**: Fully supported par Medusa.js v2
- ✅ **Strong Backward Compatibility**: Migration depuis v16 sans breaking changes

**Compatibilité**:
- Node.js 24 LTS ✅
- Prisma 6.x ✅
- Medusa.js 2.x ✅

**Caractéristiques**:
- ACID compliant (transactions fiables)
- JSON/JSONB support avec JSON_TABLE()
- Full-text search (pg_trgm)
- Row-level security
- Extensions riches (uuid-ossp, pg_trgm)
- Streaming replication

**Configuration locale** (Docker):
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: medusa
      POSTGRES_PASSWORD: medusa
      POSTGRES_DB: medusa_db
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    command:
      - "postgres"
      - "-c"
      - "max_connections=200"
      - "-c"
      - "shared_buffers=256MB"

volumes:
  postgres_data:
```

**Extensions recommandées**:
```sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";      -- UUIDs
CREATE EXTENSION IF NOT EXISTS "pg_trgm";        -- Recherche floue/similaire
CREATE EXTENSION IF NOT EXISTS "btree_gin";      -- Index GIN optimisés
```

**Exemple JSON_TABLE** (Requête metadata produits):
```sql
-- Produits avec metadata JSON (couleurs, matériaux, dimensions)
SELECT
  p.id,
  p.title,
  jt.color,
  jt.material,
  jt.width_cm
FROM products p
CROSS JOIN JSON_TABLE(
  p.metadata,
  '$'
  COLUMNS (
    color TEXT PATH '$.color',
    material TEXT PATH '$.material',
    width_cm NUMERIC PATH '$.dimensions.width'
  )
) AS jt
WHERE jt.color = 'beige'
  AND jt.material = 'coton';

-- Résultat:
--  id  |      title       | color | material | width_cm
-- -----+------------------+-------+----------+----------
--  123 | Suspension Boho  | beige | coton    |    40
--  456 | Tenture Moderne  | beige | coton    |    60
```

---

## 💳 Paiements

### Stripe
**Version**: Node.js 19.x, @stripe/stripe-js 8.x
**Type**: Payment Gateway
**Site**: https://stripe.com

**Rôle**: Traitement des paiements sécurisés

**Pourquoi Stripe pour ce projet**:
- **Coûts les plus bas**: 1.4% pour cartes EU (vs Mollie 1.8%, PayPal 2.9%)
- **Intégration Medusa native**: Support officiel `@medusajs/medusa/payment-stripe`
- **Startup-friendly**: Pas de frais mensuels, setup simple
- **Croissance internationale**: Excellents taux pour cartes non-EU (2.9%)
- **Documentation exceptionnelle**: API moderne, composants React prêts

**Coûts** (France 2025):
- 1.4% + 0.25€ par transaction (cartes européennes)
- 2.9% + 0.25€ (cartes non-européennes)
- +3.25% pour cartes internationales
- +2% pour conversion de devises
- Pas de frais mensuels fixes

**Alternatives considérées**:
| Provider | Coûts EU | Coûts non-EU | Intégration Medusa | Meilleur pour |
|----------|----------|--------------|-------------------|---------------|
| **Stripe** | 1.4% + €0.25 ✅ | 2.9% + €0.25 ✅ | Officielle ✅ | Startups, PME internationales |
| Mollie | 1.8% + €0.25 | 3.25% + €0.25 | Plugin tiers | PME EU (NL, BE) + méthodes locales |
| PayPal | 2.9% + $0.30 | 2.9% + $0.30 | Plugin tiers | B2C consumer-first |
| Adyen | €0.11 + % | €0.11 + % | Plugin tiers | Grandes entreprises |

**Installation**:
```bash
npm install stripe @stripe/stripe-js
```

**Configuration Medusa.js 2.x** (`medusa-config.ts`):
```typescript
import { defineConfig } from "@medusajs/framework/utils"

module.exports = defineConfig({
  modules: [
    {
      resolve: "@medusajs/medusa/payment",
      options: {
        providers: [
          {
            resolve: "@medusajs/medusa/payment-stripe",
            id: "stripe",
            options: {
              apiKey: process.env.STRIPE_API_KEY,
            },
          },
        ],
      },
    },
  ],
})
```

**Configuration générique**:
```typescript
// Backend (server-side)
import Stripe from 'stripe'

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
  apiVersion: '2024-12-18.acacia', // API Acacia (releases mensuelles sans breaking changes)
  maxNetworkRetries: 2,
  timeout: 80000,
  appInfo: {
    name: 'Boutique-Macrame',
    version: '1.0.0',
  }
})

// Frontend (client-side)
import { loadStripe } from '@stripe/stripe-js'

const stripePromise = loadStripe(process.env.VITE_STRIPE_PUBLISHABLE_KEY!)
```

**Environnement** (`.env`):
```bash
# Medusa.js
STRIPE_API_KEY=sk_test_51J...

# Générique
STRIPE_SECRET_KEY=sk_test_51J...
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_51J...
```

**Features utilisées**:
- Payment Intents (3D Secure / DSP2)
- Checkout Sessions
- Webhooks (confirmations de paiement)
- Customer Portal (gestion abonnements)
- Invoicing (factures automatiques)
- Payment Element (composant React intégré)

**Note sur les versions Acacia**:
Stripe utilise depuis septembre 2024 un nouveau modèle de versioning:
- **Releases mensuelles** (ex: 2024-10-28.acacia → 2024-12-18.acacia): Pas de breaking changes
- **Releases biannuelles** (ex: Acacia → prochaine famille): Peuvent contenir des breaking changes
- Upgrade sécurisé au sein d'une même famille (Acacia)

---

## 📧 Emails

### Resend
**Version**: SDK latest (Node.js, @react-email/components)
**Type**: Email Service (Developer-First)
**Site**: https://resend.com

**Rôle**: Envoi emails transactionnels (confirmations commande, expédition, réinitialisation mot de passe)

**Pourquoi Resend pour ce projet**:
- **React Email natif**: Créé et maintenu par l'équipe Resend (composants React type-safe)
- **Stack alignment**: Next.js 16 + React 19 + TypeScript → templates emails même stack
- **Developer Experience**: API moderne, TypeScript-first, documentation exceptionnelle
- **FREE tier suffisant**: 3,000 emails/mois (couvre besoins petite boutique: 500-2000 emails/mois)
- **Hot reload**: Développement emails avec Fast Refresh comme composants web
- **Pas de vendor lock-in**: Migration vers Postmark simple si besoins deliverability augmentent

**Coûts** (2025):
- **Free**: 3,000 emails transactionnels/mois + 1,000 contacts marketing (unlimited sends)
- **Pro**: $20/mois pour 50,000 emails transactionnels
- **Scale**: $0.65-$0.90 per 1k emails (volume discount automatique)

**Pourquoi PAS Brevo** (malgré entreprise française):
- ❌ **Deliverability problématique**: 79.8% (20% emails perdus = customer service nightmare)
- ❌ **400 commandes perdues** par mois à 2k volume (inacceptable pour transactional)
- ❌ Drag-and-drop editor inadapté pour développeurs React
- ❌ Marketing-focused (CRM features non nécessaires pour petite boutique)
- ✅ Avantage RGPD (entreprise française) mais pas au prix de 20% échec

**Alternatives considérées** (comparaison email transactionnel 2025):
| Provider | Free Tier | Coûts (per 1k) | Deliverability | React Email | Meilleur pour |
|----------|-----------|----------------|----------------|-------------|---------------|
| **Resend** | 3k/mois ✅ | $0.65-$0.90 | Inconnu (nouveau) | Natif ✅✅✅ | Devs React, startups, stack moderne |
| Postmark | 500 emails | $15/mois (10k) | **93.8%** ✅ | Manuel | Serious business, deliverability critique |
| Brevo | 9k/mois | $15/20k | **79.8%** ❌ | Non | Marketing automation, EU CRM |
| SendGrid | 3k/2 mois | $2.50 | 98-99% | Manuel | Volume élevé, features marketing |
| AWS SES | 3k/mois (1 an) | $0.10 | Good | Manuel | AWS-heavy, infrastructure teams |
| Mailgun | 1k/mois | $0.80 | Good | Manuel | Developers, bulk sending |

**Plan de migration** (si deliverability devient problème):
- **Monitoring**: Surveiller bounce rate (<5% acceptable)
- **Seuil de switch**: Si >5% bounce → Migrer vers Postmark ($15/mois, 93.8% deliverability)
- **Temps migration**: 2-4 heures (APIs similaires)
- **Coût annuel**: $0 avec Resend vs $180 avec Postmark (économie si deliverability OK)

**Installation**:
```bash
npm install resend @react-email/components
```

**Configuration** (`lib/email.ts`):
```typescript
import { Resend } from 'resend'

export const resend = new Resend(process.env.RESEND_API_KEY)

// Environnement (.env)
// RESEND_API_KEY=re_...
```

**Utilisation** (Server Action):
```typescript
// actions/send-order-confirmation.ts
'use server'

import { resend } from '@/lib/email'
import { OrderConfirmation } from '@/emails/order-confirmation'

export async function sendOrderConfirmation(order: Order) {
  const { data, error } = await resend.emails.send({
    from: 'Boutique Macramé <orders@boutique-macrame.com>',
    to: order.customer.email,
    subject: `Commande #${order.display_id} confirmée`,
    react: OrderConfirmation({ order }),
    // Optional: tags pour analytics
    tags: [
      { name: 'category', value: 'order_confirmation' }
    ]
  })

  if (error) {
    console.error('Email send failed:', error)
    throw new Error('Failed to send order confirmation')
  }

  return data
}
```

**Templates React** (`emails/order-confirmation.tsx`):
```tsx
import {
  Html,
  Head,
  Body,
  Container,
  Section,
  Row,
  Column,
  Text,
  Button,
  Hr,
  Tailwind
} from '@react-email/components'

interface OrderConfirmationProps {
  order: {
    display_id: string
    id: string
    items: Array<{ title: string; quantity: number; price: number }>
    total: number
    shipping_address: {
      first_name: string
      address_1: string
      city: string
      postal_code: string
    }
  }
}

export function OrderConfirmation({ order }: OrderConfirmationProps) {
  return (
    <Html>
      <Head />
      <Tailwind>
        <Body className="bg-neutral-50 font-sans">
          <Container className="mx-auto py-8 px-4 max-w-2xl">
            {/* Header */}
            <Section className="bg-white rounded-lg shadow-sm p-6 mb-4">
              <Text className="text-2xl font-bold text-primary mb-2">
                Merci pour votre commande!
              </Text>
              <Text className="text-neutral-600">
                Commande <strong>#{order.display_id}</strong>
              </Text>
            </Section>

            {/* Order Items */}
            <Section className="bg-white rounded-lg shadow-sm p-6 mb-4">
              <Text className="text-lg font-semibold mb-4">Articles commandés</Text>
              {order.items.map((item, index) => (
                <Row key={index} className="mb-3">
                  <Column className="w-2/3">
                    <Text className="m-0">{item.title}</Text>
                    <Text className="m-0 text-sm text-neutral-500">
                      Quantité: {item.quantity}
                    </Text>
                  </Column>
                  <Column className="w-1/3 text-right">
                    <Text className="m-0 font-semibold">
                      {(item.price / 100).toFixed(2)}€
                    </Text>
                  </Column>
                </Row>
              ))}

              <Hr className="my-4" />

              <Row>
                <Column className="w-2/3">
                  <Text className="m-0 font-bold">Total</Text>
                </Column>
                <Column className="w-1/3 text-right">
                  <Text className="m-0 font-bold text-lg">
                    {(order.total / 100).toFixed(2)}€
                  </Text>
                </Column>
              </Row>
            </Section>

            {/* Shipping Address */}
            <Section className="bg-white rounded-lg shadow-sm p-6 mb-4">
              <Text className="text-lg font-semibold mb-2">Adresse de livraison</Text>
              <Text className="m-0">{order.shipping_address.first_name}</Text>
              <Text className="m-0">{order.shipping_address.address_1}</Text>
              <Text className="m-0">
                {order.shipping_address.postal_code} {order.shipping_address.city}
              </Text>
            </Section>

            {/* CTA Button */}
            <Section className="text-center">
              <Button
                href={`https://boutique-macrame.com/orders/${order.id}`}
                className="bg-primary text-white px-6 py-3 rounded-md font-semibold no-underline"
              >
                Suivre ma commande
              </Button>
            </Section>

            {/* Footer */}
            <Section className="mt-8">
              <Text className="text-center text-sm text-neutral-500">
                Questions? Contactez-nous à{' '}
                <a href="mailto:contact@boutique-macrame.com" className="text-primary">
                  contact@boutique-macrame.com
                </a>
              </Text>
            </Section>
          </Container>
        </Body>
      </Tailwind>
    </Html>
  )
}

// Preview text (apparaît dans inbox avant ouverture)
OrderConfirmation.PreviewProps = {
  order: {
    display_id: '1234',
    id: 'order_abc123',
    items: [
      { title: 'Suspension Macramé Boho', quantity: 1, price: 4500 },
      { title: 'Tenture Murale Moderne', quantity: 2, price: 6500 }
    ],
    total: 17500,
    shipping_address: {
      first_name: 'Marie Dupont',
      address_1: '12 Rue des Artisans',
      city: 'Paris',
      postal_code: '75001'
    }
  }
} as OrderConfirmationProps
```

**Développement local** (avec preview):
```bash
# Démarrer React Email dev server
npx react-email dev

# Ouvre http://localhost:3000
# Hot reload des templates emails
```

**Features React Email**:
- **Composants**: Html, Head, Body, Container, Section, Row, Column, Text, Button, Hr, Link, Img
- **Tailwind CSS**: Support natif (pas besoin d'inlining manuel)
- **TypeScript**: Type-safe props pour tous les composants
- **Preview**: Props de preview pour développement
- **Dark mode**: Support automatique
- **Responsive**: Email clients compatibility gérée automatiquement

---

## 🖼️ Assets & CDN

### Image Storage & CDN

**RECOMMANDATION 2025**: Cloudflare R2 + Next.js Image Component

#### Comparaison des Solutions

| Critère | **Cloudflare R2 + next/image** ✅ | Cloudinary | R2 + Cloudflare Images |
|---------|-----------------------------------|------------|------------------------|
| **Storage FREE** | 10GB/mois | 25GB/mois | 10GB/mois |
| **Bandwidth FREE** | ♾️ Illimité (zero egress) | 25GB/mois | ♾️ Illimité (zero egress) |
| **Transformations** | Illimité (Next.js natif) | Inclus | 5k/mois gratuit |
| **Après FREE tier** | $0.015/GB storage | $99/mois (Plus plan) | +$0.50/1k transforms |
| **CDN** | Cloudflare (330+ villes) | Cloudinary | Cloudflare (330+ villes) |
| **Format auto** | WebP/AVIF (Next.js) | WebP/AVIF | WebP/AVIF |
| **Vendor lock-in** | ❌ Non (S3-compatible) | ⚠️ Oui | ❌ Non (S3-compatible) |
| **Stack alignment** | ✅ Next.js natif | Dependency externe | Moderate |
| **Setup complexity** | Faible | Très faible | Moyenne |

**Justification choix R2**:
- ✅ **Zero egress fees**: Bandwidth illimitée = scalabilité infinie sans coût
- ✅ **FREE tier suffisant**: 10GB storage pour ~100-500 produits (~2-5GB réel)
- ✅ **Stack alignment**: Utilise Next.js 16 Image Component (déjà installé)
- ✅ **No vendor lock-in**: S3-compatible, migration facile
- ✅ **Open-source friendly**: Même stratégie que Dub.co (Cloudinary → R2 migration)
- ⚠️ **Trade-off**: Pas d'AI features (background removal, generative fill) - non nécessaires pour macramé

#### Cloudflare R2 + Next.js Image
**Version**: R2 API latest, Next.js 16.0.1
**Type**: Object Storage S3-compatible + Image Optimization
**Site**: https://developers.cloudflare.com/r2/

**Coûts**:
- **FREE tier**: 10GB storage + 10M Class A ops (write) + 1M Class B ops (read) + ZERO egress
- **Payant**: $0.015/GB storage, $4.50/million Class A, $0.36/million Class B, $0 egress

**Configuration R2**:
```bash
# 1. Créer bucket via Cloudflare Dashboard
# 2. Configurer domaine personnalisé (ex: cdn.macrame-boutique.com)
# 3. Activer accès public ou signed URLs
```

**Configuration Next.js** (`next.config.ts`):
```typescript
import type { NextConfig } from 'next'

const config: NextConfig = {
  images: {
    remotePatterns: [
      {
        protocol: 'https',
        hostname: 'pub-xxxxx.r2.dev', // Ou domaine custom
        port: '',
        pathname: '/macrame-products/**',
      },
    ],
    formats: ['image/avif', 'image/webp'], // Auto-optimisation
    deviceSizes: [640, 750, 828, 1080, 1200],
    imageSizes: [16, 32, 48, 64, 96, 128, 256, 384],
  },
}

export default config
```

**Upload vers R2** (Server Action):
```typescript
'use server'

import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3'

const r2Client = new S3Client({
  region: 'auto',
  endpoint: `https://${process.env.CLOUDFLARE_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: {
    accessKeyId: process.env.R2_ACCESS_KEY_ID!,
    secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
  },
})

export async function uploadProductImage(formData: FormData) {
  const file = formData.get('image') as File
  const buffer = Buffer.from(await file.arrayBuffer())

  const key = `macrame-products/${Date.now()}-${file.name}`

  await r2Client.send(new PutObjectCommand({
    Bucket: 'macrame-boutique',
    Key: key,
    Body: buffer,
    ContentType: file.type,
  }))

  return `https://cdn.macrame-boutique.com/${key}`
}
```

**Affichage optimisé** (Composant React):
```typescript
import Image from 'next/image'

export function ProductImage({ src, alt }: { src: string; alt: string }) {
  return (
    <Image
      src={src} // URL R2: https://cdn.macrame-boutique.com/macrame-products/...
      alt={alt}
      width={1200}
      height={1200}
      sizes="(max-width: 768px) 100vw, (max-width: 1200px) 50vw, 33vw"
      quality={85}
      placeholder="blur"
      blurDataURL="data:image/jpeg;base64,/9j/4AAQSkZJRg..." // Généré au build
      className="object-cover"
    />
  )
}
```

**Installation dépendances**:
```bash
npm install @aws-sdk/client-s3
```

**Variables d'environnement** (`.env`):
```bash
CLOUDFLARE_ACCOUNT_ID=your_account_id
R2_ACCESS_KEY_ID=your_access_key
R2_SECRET_ACCESS_KEY=your_secret_key
NEXT_PUBLIC_R2_PUBLIC_URL=https://cdn.macrame-boutique.com
```

#### Alternative: Cloudinary (Si besoin AI features)
**Version**: SDK 2.x, next-cloudinary 6.x
**Coûts**: 25GB storage + 25GB bandwidth gratuit, puis $99/mois

**Migration path**:
1. **Phase 1** (Actuelle): R2 + next/image (FREE, suffit pour 95% cas usage)
2. **Phase 2** (Si besoin): Ajouter Cloudflare Images pour transformations complexes (+$5/mois)
3. **Phase 3** (Si besoin AI): Migrer vers Cloudinary pour background removal, generative fill

**Références**:
- Dub.co migration case study: https://dub.co/blog/image-hosting-r2
- Cloudflare R2 pricing: https://developers.cloudflare.com/r2/pricing/
- Next.js Image docs: https://nextjs.org/docs/app/api-reference/components/image

---

## 🛡️ Sécurité

### Security Headers (Native Next.js)

**RECOMMANDATION 2025**: Configuration native Next.js (sans Helmet)

**Pourquoi PAS Helmet**:
- ❌ **Incompatible avec Next.js**: Helmet est un middleware Express (`app.use()`) qui ne fonctionne pas avec Next.js serverless/edge runtime
- ❌ **Architecture différente**: Next.js utilise `next.config.ts` pour headers statiques + `proxy.ts` pour headers dynamiques
- ❌ **Dependency inutile**: Next.js fournit toutes les fonctionnalités nativement
- ✅ **Alternative**: Utiliser `headers()` dans `next.config.ts` (zero dependency, official support, production-ready)

#### Comparaison des Solutions

| Solution | Compatibilité Next.js 16 | Dependencies | Maintenance | Recommandé pour |
|----------|-------------------------|--------------|-------------|-----------------|
| **Native headers()** ✅ | ✅ Parfait | 0 | Officiel | Tous projets |
| next-secure-headers | ✅ Bon | 1 | Communauté | Preset configurations |
| Nosecone (Arcjet) | ✅ Excellent | 1 | Active (2025) | Multi-framework |
| Helmet | ❌ Incompatible | 1 | Express only | Express/Node.js only |

#### Configuration Native Next.js 16

**Configuration** (`next.config.ts`):
```typescript
import type { NextConfig } from 'next'

const config: NextConfig = {
  // ... autres configs (reactCompiler, turbo, images, etc.)

  async headers() {
    return [
      {
        // Appliquer à toutes les routes
        source: '/(.*)',
        headers: [
          // Content Security Policy (CSP)
          {
            key: 'Content-Security-Policy',
            value: [
              "default-src 'self'",
              "script-src 'self' 'unsafe-inline' 'unsafe-eval' https://js.stripe.com", // Stripe Payment Element
              "style-src 'self' 'unsafe-inline'", // Tailwind CSS inline styles
              "img-src 'self' data: https: blob: https://cdn.macrame-boutique.com", // R2 CDN
              "font-src 'self' data:",
              "connect-src 'self' https://api.stripe.com", // Stripe API
              "frame-src 'self' https://js.stripe.com", // Stripe iframe
              "object-src 'none'",
              "base-uri 'self'",
              "form-action 'self'",
              "frame-ancestors 'none'", // Remplace X-Frame-Options
              "upgrade-insecure-requests"
            ].join('; ')
          },

          // HTTP Strict Transport Security (HSTS)
          {
            key: 'Strict-Transport-Security',
            value: 'max-age=63072000; includeSubDomains; preload' // 2 ans
          },

          // Prevent MIME type sniffing
          {
            key: 'X-Content-Type-Options',
            value: 'nosniff'
          },

          // Referrer Policy
          {
            key: 'Referrer-Policy',
            value: 'strict-origin-when-cross-origin'
          },

          // Permissions Policy (remplace Feature-Policy)
          {
            key: 'Permissions-Policy',
            value: [
              'camera=()',
              'microphone=()',
              'geolocation=()',
              'interest-cohort=()' // Disable FLoC
            ].join(', ')
          }
        ]
      }
    ]
  }
}

export default config
```

**Headers spécifiques e-commerce**:
- ✅ **Stripe CSP**: `script-src` + `frame-src` + `connect-src` autorisent Stripe Payment Element
- ✅ **R2 Images CDN**: `img-src` autorise Cloudflare R2 custom domain
- ✅ **Tailwind inline styles**: `style-src 'unsafe-inline'` requis pour Tailwind CSS
- ✅ **HSTS Preload**: Force HTTPS permanent (2 ans) pour SEO et sécurité
- ✅ **No clickjacking**: `frame-ancestors 'none'` empêche iframe malveillants

#### CSP avec Nonces (Optionnel - Plus Sécurisé)

Si besoin de CSP stricte sans `'unsafe-inline'`, utiliser `proxy.ts`:

**Configuration** (`proxy.ts`):
```typescript
import { NextRequest, NextResponse } from 'next/server'

export function proxy(request: NextRequest) {
  // Générer nonce unique par requête
  const nonce = Buffer.from(crypto.randomUUID()).toString('base64')
  const isDev = process.env.NODE_ENV === 'development'

  const cspHeader = `
    default-src 'self';
    script-src 'self' 'nonce-${nonce}' 'strict-dynamic' https://js.stripe.com ${isDev ? "'unsafe-eval'" : ''};
    style-src 'self' 'nonce-${nonce}';
    img-src 'self' data: https: blob: https://cdn.macrame-boutique.com;
    font-src 'self' data:;
    connect-src 'self' https://api.stripe.com;
    frame-src 'self' https://js.stripe.com;
    object-src 'none';
    base-uri 'self';
    form-action 'self';
    frame-ancestors 'none';
    upgrade-insecure-requests;
  `

  const cspValue = cspHeader.replace(/\s{2,}/g, ' ').trim()

  const requestHeaders = new Headers(request.headers)
  requestHeaders.set('x-nonce', nonce)
  requestHeaders.set('Content-Security-Policy', cspValue)

  const response = NextResponse.next({
    request: { headers: requestHeaders }
  })

  response.headers.set('Content-Security-Policy', cspValue)
  return response
}

// Matcher: exclure API routes, static files
export const config = {
  matcher: [
    {
      source: '/((?!api|_next/static|_next/image|favicon.ico).*)',
      missing: [
        { type: 'header', key: 'next-router-prefetch' },
        { type: 'header', key: 'purpose', value: 'prefetch' }
      ]
    }
  ]
}
```

**Utilisation nonce dans composants**:
```tsx
import { headers } from 'next/headers'

export default async function Page() {
  const nonce = (await headers()).get('x-nonce')

  return (
    <html>
      <head>
        <script nonce={nonce} src="/analytics.js" />
        <style nonce={nonce}>{`body { margin: 0; }`}</style>
      </head>
      <body>...</body>
    </html>
  )
}
```

#### Alternatives (Si Preset Configurations Souhaité)

**next-secure-headers** (Helmet-like pour Next.js):
```bash
npm install next-secure-headers
```

```typescript
import { createSecureHeaders } from 'next-secure-headers'

const config: NextConfig = {
  async headers() {
    return [{
      source: '/(.*)',
      headers: createSecureHeaders({
        contentSecurityPolicy: {
          directives: {
            defaultSrc: "'self'",
            scriptSrc: ["'self'", "https://js.stripe.com"],
            // ...
          }
        },
        forceHTTPSRedirect: [true, { maxAge: 63072000, includeSubDomains: true }]
      })
    }]
  }
}
```

**Nosecone** (Arcjet - Modern 2025):
```bash
npm install @nosecone/next
```

```typescript
import { defaults } from '@nosecone/next'

const config: NextConfig = {
  async headers() {
    return [{
      source: '/(.*)',
      headers: defaults({
        csp: {
          directives: {
            scriptSrc: ["'self'", "https://js.stripe.com"]
          }
        }
      })
    }]
  }
}
```

**Validation Security Headers**:
- https://securityheaders.com (Score A+ recommandé)
- https://observatory.mozilla.org
- Chrome DevTools → Network → Response Headers

**Note CVE-2025-29927**: Next.js middleware avait une vulnérabilité d'authorization bypass. Configuration via `next.config.ts` est plus sûre que middleware pour headers de sécurité.

---

### Rate Limiting (Next.js)

**RECOMMANDATION 2025**: Upstash Rate Limit + Vercel KV

**Pourquoi PAS express-rate-limit**:
- ❌ **Incompatible avec Next.js**: express-rate-limit utilise `app.use()` (Express middleware) qui ne fonctionne pas avec Next.js serverless/edge runtime
- ❌ **Architecture différente**: Next.js API routes et Server Actions nécessitent approche edge-compatible
- ❌ **Pas de state partagé**: Next.js déploie sur edge functions distribuées sans mémoire partagée
- ✅ **Alternative**: Utiliser Upstash Rate Limit + Redis pour state distribué

#### Comparaison des Solutions

| Solution | Next.js 16 Edge | FREE Tier | Algorithms | Bot Detection | Recommandé pour |
|----------|----------------|-----------|------------|---------------|-----------------|
| **Upstash Rate Limit** ✅ | ✅ Parfait | 10k commands/day | Sliding, Fixed, Token | ❌ | Tous projets, startups |
| Arcjet | ✅ Excellent | 5k req/month | Sliding, Fixed | ✅ | Security-focused, paid |
| Vercel WAF | ✅ Built-in | Pro plan only | Automatic | ✅ | Enterprise (>$20/user) |
| Custom Redis | ✅ Possible | Depends | Manual | ❌ | High maintenance |
| express-rate-limit | ❌ Incompatible | N/A | Fixed, Sliding | ❌ | Express/Node.js only |

#### Configuration Upstash Rate Limit (Next.js 16)

**FREE Tier Upstash**:
- 10,000 commands/day (suffisant pour petite boutique)
- 256MB Redis storage
- Global edge locations
- Sliding window algorithm

**Installation**:
```bash
npm install @upstash/ratelimit @upstash/redis
```

**Setup Vercel KV** (ou Upstash Redis):
```bash
# Via Vercel Dashboard: Storage → Create KV Database
# Ou via Upstash Console: https://console.upstash.com

# Variables d'environnement automatiques:
# KV_REST_API_URL=https://...
# KV_REST_API_TOKEN=...
```

**Configuration globale** (`lib/rate-limit.ts`):
```typescript
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

// Redis client (Vercel KV ou Upstash)
export const redis = Redis.fromEnv()

// Rate limiters par use case
export const rateLimiters = {
  // API générale: 100 req/15min
  api: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(100, '15 m'),
    analytics: true,
    prefix: 'ratelimit:api'
  }),

  // Login: 5 tentatives/15min (anti-brute force)
  auth: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(5, '15 m'),
    analytics: true,
    prefix: 'ratelimit:auth'
  }),

  // Checkout: 10 commandes/heure (anti-spam)
  checkout: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(10, '1 h'),
    analytics: true,
    prefix: 'ratelimit:checkout'
  }),

  // Contact form: 3 messages/heure
  contact: new Ratelimit({
    redis,
    limiter: Ratelimit.slidingWindow(3, '1 h'),
    analytics: true,
    prefix: 'ratelimit:contact'
  })
}

// Helper: Get client identifier (IP or user ID)
export function getIdentifier(request: Request): string {
  const forwarded = request.headers.get('x-forwarded-for')
  const ip = forwarded ? forwarded.split(',')[0] : 'anonymous'
  return ip
}
```

**Protection API Routes** (`app/api/products/route.ts`):
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { rateLimiters, getIdentifier } from '@/lib/rate-limit'

export async function GET(request: NextRequest) {
  const identifier = getIdentifier(request)

  // Check rate limit
  const { success, limit, remaining, reset } = await rateLimiters.api.limit(identifier)

  if (!success) {
    return NextResponse.json(
      { error: 'Rate limit exceeded. Please try again later.' },
      {
        status: 429,
        headers: {
          'X-RateLimit-Limit': limit.toString(),
          'X-RateLimit-Remaining': remaining.toString(),
          'X-RateLimit-Reset': reset.toString()
        }
      }
    )
  }

  // Process request
  const products = await getProducts()
  return NextResponse.json(products, {
    headers: {
      'X-RateLimit-Limit': limit.toString(),
      'X-RateLimit-Remaining': remaining.toString()
    }
  })
}
```

**Protection Server Actions** (`actions/auth.ts`):
```typescript
'use server'

import { rateLimiters, getIdentifier } from '@/lib/rate-limit'
import { headers } from 'next/headers'

export async function loginAction(email: string, password: string) {
  // Get identifier from headers
  const headersList = await headers()
  const forwarded = headersList.get('x-forwarded-for')
  const identifier = forwarded?.split(',')[0] || 'anonymous'

  // Check rate limit
  const { success } = await rateLimiters.auth.limit(identifier)

  if (!success) {
    return {
      error: 'Too many login attempts. Please try again in 15 minutes.'
    }
  }

  // Process login
  const user = await authenticateUser(email, password)
  return { user }
}
```

**Protection via Proxy** (`proxy.ts`):
```typescript
import { NextRequest, NextResponse } from 'next/server'
import { rateLimiters, getIdentifier } from '@/lib/rate-limit'

export async function proxy(request: NextRequest) {
  const identifier = getIdentifier(request)

  // Rate limit all API routes
  if (request.nextUrl.pathname.startsWith('/api/')) {
    const { success } = await rateLimiters.api.limit(identifier)

    if (!success) {
      return NextResponse.json(
        { error: 'Rate limit exceeded' },
        { status: 429 }
      )
    }
  }

  // Rate limit Server Actions (detect via Next-Action header)
  if (request.headers.get('Next-Action')) {
    const { success } = await rateLimiters.api.limit(identifier)

    if (!success) {
      return NextResponse.json(
        { error: 'Rate limit exceeded' },
        { status: 429 }
      )
    }
  }

  return NextResponse.next()
}

export const config = {
  matcher: [
    '/api/:path*',
    '/((?!_next/static|_next/image|favicon.ico).*)'
  ]
}
```

**Exemple use case e-commerce** (Checkout protection):
```typescript
// app/checkout/actions.ts
'use server'

import { rateLimiters } from '@/lib/rate-limit'
import { headers } from 'next/headers'

export async function createOrderAction(cartId: string) {
  const headersList = await headers()
  const ip = headersList.get('x-forwarded-for')?.split(',')[0] || 'anonymous'

  // Rate limit: 10 commandes/heure max
  const { success, reset } = await rateLimiters.checkout.limit(ip)

  if (!success) {
    const resetDate = new Date(reset)
    return {
      error: `Too many orders. Please try again at ${resetDate.toLocaleTimeString()}`
    }
  }

  // Process order
  const order = await medusa.checkout.complete(cartId)
  return { order }
}
```

#### Alternative: Arcjet (Security Suite Complète)

**Si besoin bot detection + rate limiting + attack protection**:

```bash
npm install @arcjet/next
```

```typescript
import arcjet, { detectBot, slidingWindow } from '@arcjet/next'

const aj = arcjet({
  key: process.env.ARCJET_KEY!,
  rules: [
    detectBot({ mode: 'LIVE', block: ['AUTOMATED'] }),
    slidingWindow({
      mode: 'LIVE',
      interval: '15m',
      max: 100
    })
  ]
})

export async function GET(req: NextRequest) {
  const decision = await aj.protect(req)

  if (decision.isDenied()) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }

  // Process request
}
```

**Coûts Arcjet**:
- FREE: 5,000 requests/month
- Pro: $25/month (100k requests)

#### Note: Medusa Backend Rate Limiting

**Medusa.js utilise Express** sous le capot → `express-rate-limit` POURRAIT fonctionner pour Medusa API.

Mais **Medusa 2.x a rate limiting intégré** via configuration:

```typescript
// medusa-config.ts
module.exports = {
  projectConfig: {
    redis_url: process.env.REDIS_URL,
    // Medusa built-in rate limiting
  }
}
```

**Séparation des concerns**:
- **Next.js frontend**: Upstash Rate Limit (edge-compatible)
- **Medusa backend**: Built-in ou express-rate-limit (si nécessaire)

**Validation**:
- Test: `curl -I https://your-app.com/api/products` (check `X-RateLimit-*` headers)
- Monitor: Upstash Dashboard → Analytics

---

### bcrypt
**Version**: 5.x
**Type**: Password Hashing
**Licence**: MIT

**Rôle**: Hachage sécurisé des mots de passe

**Installation**:
```bash
npm install bcrypt
npm install -D @types/bcrypt
```

**Utilisation**:
```typescript
import bcrypt from 'bcrypt'

// Hash password
const saltRounds = 12
const hashedPassword = await bcrypt.hash(password, saltRounds)

// Vérifier password
const isValid = await bcrypt.compare(inputPassword, hashedPassword)
```

---

## 📊 Monitoring & Logging

### Sentry
**Version**: SDK latest
**Type**: Error Tracking & Performance Monitoring
**Site**: https://sentry.io

**Rôle**: Tracking erreurs production et performance

**Coûts**:
- Free: 5K events/mois
- Team: 26$/mois (50K events)

**Installation**:
```bash
npm install @sentry/node @sentry/react
```

**Configuration Backend**:
```typescript
import * as Sentry from '@sentry/node'

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,  // 10% des transactions
  integrations: [
    new Sentry.Integrations.Http({ tracing: true }),
    new Sentry.Integrations.Postgres()
  ]
})
```

**Configuration Frontend**:
```typescript
import * as Sentry from '@sentry/react'

Sentry.init({
  dsn: process.env.VITE_SENTRY_DSN,
  integrations: [
    new Sentry.BrowserTracing(),
    new Sentry.Replay()
  ],
  tracesSampleRate: 0.1,
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0
})
```

---

### Winston
**Version**: 3.x
**Type**: Logging Library
**Licence**: MIT

**Rôle**: Logs structurés backend

**Installation**:
```bash
npm install winston
```

**Configuration**:
```typescript
import winston from 'winston'

const logger = winston.createLogger({
  level: process.env.LOG_LEVEL || 'info',
  format: winston.format.combine(
    winston.format.timestamp(),
    winston.format.errors({ stack: true }),
    winston.format.json()
  ),
  transports: [
    new winston.transports.File({ filename: 'error.log', level: 'error' }),
    new winston.transports.File({ filename: 'combined.log' }),
    new winston.transports.Console({
      format: winston.format.simple()
    })
  ]
})

// Utilisation
logger.info('Order created', { orderId: order.id, total: order.total })
logger.error('Payment failed', { error, customerId })
```

---

## 🧪 Testing

### Vitest
**Version**: 2.x
**Type**: Unit Testing Framework
**Licence**: MIT
**Site**: https://vitest.dev

**Rôle**: Tests unitaires et d'intégration

**Installation**:
```bash
npm install -D vitest @vitest/ui
```

**Configuration** (`vitest.config.ts`):
```typescript
import { defineConfig } from 'vitest/config'

export default defineConfig({
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: './tests/setup.ts',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'json', 'html']
    }
  }
})
```

**Exemple test**:
```typescript
import { describe, it, expect } from 'vitest'
import { calculateTotal } from './cart'

describe('calculateTotal', () => {
  it('should calculate correct total with tax', () => {
    const items = [
      { price: 1000, quantity: 2 },  // 20€
      { price: 500, quantity: 1 }     // 5€
    ]
    expect(calculateTotal(items, 0.2)).toBe(30)  // 25€ + 20% TVA
  })
})
```

---

### Playwright
**Version**: 1.x
**Type**: E2E Testing Framework
**Licence**: Apache 2.0
**Site**: https://playwright.dev

**Rôle**: Tests end-to-end automatisés

**Installation**:
```bash
npm install -D @playwright/test
npx playwright install
```

**Test checkout flow**:
```typescript
import { test, expect } from '@playwright/test'

test('complete checkout flow', async ({ page }) => {
  // Ajouter produit au panier
  await page.goto('/products/macrame-wall-hanging')
  await page.click('[data-testid="add-to-cart"]')

  // Aller au checkout
  await page.click('[data-testid="cart-icon"]')
  await page.click('[data-testid="checkout-button"]')

  // Remplir infos
  await page.fill('[name="email"]', 'test@example.com')
  await page.fill('[name="address"]', '123 Rue Example')

  // Vérifier page paiement Stripe
  await expect(page).toHaveURL(/checkout.stripe.com/)
})
```

---

## 🚀 DevOps & Deployment

### Docker
**Version**: 27.x (Latest)
**Type**: Containerization Platform
**Licence**: Apache 2.0
**Site**: https://www.docker.com

**Rôle**: Environnement développement local et production

**⚠️ Deployment Architecture (2025 SOTA)**:

Pour une petite boutique e-commerce, **NE PAS tout self-host**. Architecture hybride optimale :

| Composant | Solution | Coût/mois | Pourquoi |
|-----------|----------|-----------|----------|
| **Frontend** | Vercel | FREE | Edge network global (330+ locations), auto-deploy GitHub |
| **Backend** | Hetzner CAX11 ARM | €3.79 (~$4.44) | Meilleur rapport qualité/prix 2025 (4GB RAM, 20TB bandwidth) |
| **Database** | Supabase | FREE | 500MB always-on, backups automatiques, pas de cold start |
| **KV/Rate Limit** | Upstash Redis | FREE | 10k commands/day, edge-compatible |
| **Images** | Cloudflare R2 | FREE | 10GB storage + bandwidth illimitée (zero egress) |
| **Total** | | **€3.79/mois** | vs Railway $10-20/mois, vs tout self-host (risques) |

**Pourquoi PAS tout sur VPS ?**
- ❌ **Performance**: Vercel edge network >> VPS unique (SEO + conversions)
- ❌ **Risque**: VPS down = site down = ventes perdues
- ❌ **Maintenance**: SSL, backups, monitoring, mises à jour = temps ≠ business
- ❌ **Scalabilité**: VPS fixe vs Vercel auto-scale
- ✅ **Trade-off**: €3.79/mois = prix d'1 café = infrastructure professionnelle

**Dockerfile Backend (Production - Multi-stage)**:
```dockerfile
# Build stage
FROM node:24-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production && \
    npm cache clean --force

COPY . .
RUN npm run build

# Production stage
FROM node:24-alpine

# Security: non-root user
RUN addgroup -g 1001 -S medusa && \
    adduser -S medusa -u 1001

WORKDIR /app

# Copy from builder
COPY --from=builder --chown=medusa:medusa /app/node_modules ./node_modules
COPY --from=builder --chown=medusa:medusa /app/dist ./dist
COPY --from=builder --chown=medusa:medusa /app/package*.json ./

USER medusa

EXPOSE 9000

HEALTHCHECK --interval=30s --timeout=3s --start-period=40s \
  CMD node -e "require('http').get('http://localhost:9000/health', (r) => process.exit(r.statusCode === 200 ? 0 : 1))"

CMD ["npm", "start"]
```

**Docker Compose (Développement Local)**:
```yaml
version: '3.8'

services:
  # PostgreSQL 17 (local dev uniquement - production = Supabase)
  postgres:
    image: postgres:17-alpine
    container_name: medusa-postgres
    environment:
      POSTGRES_DB: medusa_db
      POSTGRES_USER: medusa
      POSTGRES_PASSWORD: medusa
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U medusa"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Redis 7 (Medusa cache + rate limiting local dev)
  redis:
    image: redis:7-alpine
    container_name: medusa-redis
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # Medusa Backend
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: medusa-backend
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: postgresql://medusa:medusa@postgres:5432/medusa_db
      REDIS_URL: redis://redis:6379
      NODE_ENV: development
      JWT_SECRET: supersecret
      COOKIE_SECRET: supersecret
    ports:
      - "9000:9000"
    volumes:
      - ./backend:/app
      - /app/node_modules  # Prevent node_modules override
    command: npm run dev

volumes:
  postgres_data:
  redis_data:
```

**Docker Compose (Production - Hetzner VPS)**:
```yaml
version: '3.8'

services:
  # Redis uniquement (PostgreSQL = Supabase externe)
  redis:
    image: redis:7-alpine
    container_name: medusa-redis
    restart: unless-stopped
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 3s
      retries: 3
    networks:
      - medusa-network

  # Medusa Backend
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: medusa-backend
    restart: unless-stopped
    depends_on:
      redis:
        condition: service_healthy
    environment:
      DATABASE_URL: ${DATABASE_URL}  # Supabase connection string
      REDIS_URL: redis://redis:6379
      NODE_ENV: production
      JWT_SECRET: ${JWT_SECRET}
      COOKIE_SECRET: ${COOKIE_SECRET}
      MEDUSA_BACKEND_URL: ${MEDUSA_BACKEND_URL}
    ports:
      - "9000:9000"
    healthcheck:
      test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:9000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    networks:
      - medusa-network

  # Caddy (Reverse proxy + SSL automatique)
  caddy:
    image: caddy:2-alpine
    container_name: medusa-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_data:/data
      - caddy_config:/config
    networks:
      - medusa-network

volumes:
  redis_data:
  caddy_data:
  caddy_config:

networks:
  medusa-network:
    driver: bridge
```

**Caddyfile** (SSL automatique Let's Encrypt):
```
api.macrame-boutique.com {
    reverse_proxy backend:9000
    encode gzip

    header {
        # Security headers
        Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
        X-Content-Type-Options "nosniff"
        X-Frame-Options "DENY"
        Referrer-Policy "strict-origin-when-cross-origin"
    }

    log {
        output file /var/log/caddy/access.log
    }
}
```

**Setup Production (Hetzner VPS)**:
```bash
# 1. Créer VPS Hetzner CAX11 ARM (€3.79/mois)
# 2. SSH et installer Docker
ssh root@your-vps-ip
apt update && apt upgrade -y
apt install -y docker.io docker-compose

# 3. Cloner repo
git clone https://github.com/your-username/boutique-macrame.git
cd boutique-macrame/backend

# 4. Variables d'environnement
cat > .env.production <<EOF
DATABASE_URL=postgresql://postgres:[password]@[host].supabase.co:5432/postgres
REDIS_URL=redis://redis:6379
JWT_SECRET=$(openssl rand -base64 32)
COOKIE_SECRET=$(openssl rand -base64 32)
MEDUSA_BACKEND_URL=https://api.macrame-boutique.com
EOF

# 5. Démarrer services
docker-compose -f docker-compose.prod.yml up -d

# 6. Vérifier logs
docker-compose logs -f backend
```

**Monitoring (Gratuit)**:
```bash
# Uptime monitoring: UptimeRobot (50 monitors gratuit)
# Logs: docker-compose logs
# Metrics: docker stats

# Backup automatique Supabase (inclus FREE tier)
# Redis data: docker volume backup hebdomadaire
```

**Migration path si trafic augmente**:
1. **Phase 1** (0-1000 visites/jour): Hetzner CAX11 + Supabase FREE ✅
2. **Phase 2** (1000-10k/jour): Upgrade Supabase Pro $25/mois (2GB + backups)
3. **Phase 3** (10k+/jour): Migrer backend vers Railway/Render ($20-50/mois auto-scaling)

---

### GitHub Actions
**Type**: CI/CD Platform
**Licence**: Inclus avec GitHub
**Site**: https://github.com/features/actions

**Rôle**: Pipeline CI/CD automatisé

**Workflow** (`.github/workflows/ci.yml`):
```yaml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - run: npm ci
      - run: npm run test
      - run: npm run build

      - name: Run E2E tests
        run: npx playwright test

      - name: Upload coverage
        uses: codecov/codecov-action@v4
```

---

## 🌐 Hébergement

### Vercel
**Type**: Hosting Platform (Frontend)
**Site**: https://vercel.com

**Caractéristiques**:
- Edge Network global
- Automatic deployments via Git
- Preview deployments pour PR
- Serverless functions
- Analytics intégré

**Coûts**:
- Free: 100GB bandwidth, unlimited sites
- Pro: 20$/mois (1TB bandwidth)

**Configuration** (`vercel.json`):
```json
{
  "buildCommand": "npm run build",
  "outputDirectory": ".output/public",
  "framework": "vite",
  "env": {
    "VITE_MEDUSA_BACKEND_URL": "@medusa_backend_url"
  }
}
```

---

### Railway
**Type**: Hosting Platform (Backend)
**Site**: https://railway.app

**Caractéristiques**:
- Déploiement simplifié
- PostgreSQL managed inclus
- Auto-scaling
- Metrics & logs

**Coûts**:
- Pay-as-you-go: ~5-15$/mois
- Pro: 20$/mois (usage inclus)

**Configuration** (`railway.toml`):
```toml
[build]
builder = "NIXPACKS"

[deploy]
startCommand = "npm run start"
healthcheckPath = "/health"
restartPolicyType = "ON_FAILURE"
```

---

### Neon
**Type**: Serverless PostgreSQL
**Site**: https://neon.tech

**Caractéristiques**:
- Serverless auto-scaling
- Branching (git-like pour DB)
- Backups automatiques
- Point-in-time recovery

**Coûts**:
- Free: 500MB, 1 project
- Pro: 19$/mois (10GB)

---

## 🛠️ Development Tools

### ESLint
**Version**: 9.x
**Type**: Linter JavaScript/TypeScript
**Licence**: MIT

**Installation**:
```bash
npm install -D eslint @typescript-eslint/parser @typescript-eslint/eslint-plugin
```

**Configuration** (`.eslintrc.json`):
```json
{
  "extends": [
    "eslint:recommended",
    "plugin:@typescript-eslint/recommended",
    "plugin:react/recommended",
    "plugin:react-hooks/recommended"
  ],
  "rules": {
    "no-console": "warn",
    "@typescript-eslint/no-unused-vars": "error"
  }
}
```

---

### Prettier
**Version**: 3.x
**Type**: Code Formatter
**Licence**: MIT

**Installation**:
```bash
npm install -D prettier eslint-config-prettier
```

**Configuration** (`.prettierrc`):
```json
{
  "semi": false,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5",
  "printWidth": 100
}
```

---

### Husky
**Version**: 9.x
**Type**: Git Hooks Manager
**Licence**: MIT

**Rôle**: Pre-commit hooks pour qualité code

**Installation**:
```bash
npm install -D husky lint-staged
npx husky init
```

**Configuration** (`.husky/pre-commit`):
```bash
#!/bin/sh
npx lint-staged
```

**lint-staged** (`package.json`):
```json
{
  "lint-staged": {
    "*.{js,ts,tsx}": ["eslint --fix", "prettier --write"],
    "*.{json,md}": ["prettier --write"]
  }
}
```

---

## 📦 Package Managers

### Package Manager

**RECOMMANDATION 2025**: **pnpm 9.x** (avec workaround Medusa)

#### Comparaison npm vs pnpm vs yarn vs bun (2025)

| Critère | npm 10 | **pnpm 9** ✅ | Yarn 1 | Yarn 3 | Bun |
|---------|--------|---------------|---------|---------|-----|
| **Cold Install** | 65s | **38s** (1.7x faster) | 52s | 40s | 19s |
| **Warm Install** | 22s | **7s** (3x faster) | 18s | 9s | 5s |
| **Disk Usage** | 1.2GB | **<300MB** (75% savings) | 1.1GB | ~1GB | ~1GB |
| **Next.js 16** | ✅ | ✅ | ✅ | ⚠️ PnP issues | ❌ Incompatible |
| **Medusa.js** | ✅ | ⚠️ Requires hoisting | ✅ | ⚠️ | ❌ |
| **Monorepo** | ⚠️ Basic | ✅ Excellent | ✅ Good | ✅ Excellent | ⚠️ Experimental |
| **Compatibility** | ✅ Maximum | ⚠️ Good | ✅ Excellent | ⚠️ Moderate | ❌ Limited |
| **Ecosystem** | ✅ Standard | ✅ Growing fast | ✅ Mature | ⚠️ Complex | ❌ Young |

**Pourquoi pnpm pour e-commerce 2025:**
- ✅ **3x plus rapide** que npm (CI/CD faster = moins de downtime)
- ✅ **75% moins d'espace disque** (économie serveur + dev machines)
- ✅ **Strict dependency resolution** (évite phantom dependencies)
- ✅ **Future-proof**: Vercel, Astro, SvelteKit, Turborepo l'utilisent
- ✅ **Monorepo-friendly**: Parfait si vous évoluez vers frontend + backend monorepo

**Installation**:
```bash
# Via npm (ironiquement)
npm install -g pnpm

# Via Node.js Corepack (recommandé)
corepack enable
corepack prepare pnpm@latest --activate

# Vérifier version
pnpm --version  # 9.x
```

**⚠️ Workaround Medusa.js** (obligatoire):

Medusa.js v2 a des problèmes de compatibilité pnpm. Créer `.npmrc` à la racine du projet:

```
# .npmrc
node-linker=hoisted
```

Cela force pnpm à créer une structure `node_modules` flat comme npm, perdant certains avantages pnpm mais gardant la vitesse.

**Alternative si pnpm pose problème:**
- **npm 10.x** ✅ (bundled, maximum compatibility, plus lent)
- **yarn 1.x** ✅ (bon compromis vitesse/compatibilité)
- ❌ **bun** (incompatible Next.js 16)
- ⚠️ **yarn 3+** (Plug'n'Play peut casser outils)

**Scripts package.json**:

**Frontend Next.js 16** (`frontend/package.json`):
```json
{
  "name": "macrame-boutique-frontend",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "next dev --turbopack",
    "build": "next build",
    "start": "next start",
    "lint": "next lint",
    "type-check": "tsc --noEmit",
    "format": "prettier --write \"**/*.{ts,tsx,js,jsx,json,md}\""
  },
  "dependencies": {
    "next": "^16.0.1",
    "react": "^19.2.0",
    "react-dom": "^19.2.0",
    "@medusajs/js-sdk": "latest",
    "@tanstack/react-query": "^5.x"
  },
  "devDependencies": {
    "@types/node": "^24.x",
    "@types/react": "^19.x",
    "typescript": "^5.8.0",
    "eslint": "^9.x",
    "eslint-config-next": "^16.0.1",
    "prettier": "^3.x"
  },
  "packageManager": "pnpm@9.0.0"
}
```

**Backend Medusa.js** (`backend/package.json`):
```json
{
  "name": "macrame-boutique-backend",
  "version": "2.0.0",
  "private": true,
  "scripts": {
    "dev": "medusa dev",
    "build": "medusa build",
    "start": "medusa start",
    "seed": "medusa exec ./src/scripts/seed.ts",
    "migrations:generate": "medusa migrations generate",
    "migrations:run": "medusa migrations run"
  },
  "dependencies": {
    "@medusajs/framework": "^2.x",
    "@medusajs/medusa": "^2.x",
    "@medusajs/medusa/payment-stripe": "^2.x",
    "pg": "^8.x"
  },
  "devDependencies": {
    "@types/node": "^24.x",
    "typescript": "^5.8.0"
  },
  "packageManager": "pnpm@9.0.0"
}
```

**Commandes pnpm courantes**:
```bash
# Install dependencies
pnpm install              # ou pnpm i

# Add dependency
pnpm add <package>        # production
pnpm add -D <package>     # dev dependency
pnpm add -g <package>     # global

# Remove dependency
pnpm remove <package>     # ou pnpm rm

# Update dependencies
pnpm update              # ou pnpm up
pnpm update --latest     # ignorer semver

# Run scripts
pnpm dev                 # run dev script
pnpm run build           # run build script

# Workspace commands (monorepo)
pnpm -r <cmd>            # run command in all workspaces
pnpm --filter frontend dev  # run in specific workspace
```

**Migration depuis npm**:
```bash
# 1. Installer pnpm
npm install -g pnpm

# 2. Importer package-lock.json
pnpm import

# 3. Installer dependencies
pnpm install

# 4. Créer .npmrc si Medusa
echo "node-linker=hoisted" > .npmrc

# 5. Supprimer npm artifacts
rm package-lock.json
rm -rf node_modules
```

**Benchmark réel** (Next.js 16 + 150 dependencies):
```
npm install    → 65s
pnpm install   → 38s (1.7x faster)
yarn install   → 52s

npm ci         → 22s
pnpm install   → 7s (3x faster)
yarn install   → 18s
```

**Pour startup e-commerce**: pnpm économise **~45 minutes/semaine** sur une équipe de 2 devs (10 installs/jour).

---

## 📊 Récapitulatif par Catégorie

### Frontend Core
- Next.js 16 (Framework)
- React 19.2 (UI)
- TypeScript 5.6+ (Language)
- Turbopack (Build)

### Styling
- Tailwind CSS (CSS Framework)

### Data Management
- TanStack Query (State)
- Zod (Validation)

### Backend
- Medusa.js (Commerce)
- Node.js (Runtime)
- Prisma (ORM)

### Database
- PostgreSQL 17

### Payments
- Stripe

### Communications
- Resend (Email)

### Assets
- Cloudflare R2 (Images/CDN)

### Security
- Next.js Security Headers (natif - remplace Helmet)
- Upstash Rate Limiting (remplace Express Rate Limit)
- bcrypt (Passwords)

### Monitoring
- Sentry (Errors)
- Winston (Logs)

### Testing
- Vitest (Unit)
- Playwright (E2E)

### DevOps
- Docker
- GitHub Actions

### Hosting
- Vercel (Frontend - FREE)
- Hetzner CAX11 ARM (Backend - €3.79/mois)
- Supabase (PostgreSQL - FREE tier 500MB)

### Dev Tools
- ESLint
- Prettier
- Husky

---

## 🔄 Version Management Strategy

### Semantic Versioning
Toutes les dépendances suivent semver (`^` pour minor updates):

```json
{
  "dependencies": {
    "react": "^19.0.0",           // Auto-update patch + minor
    "stripe": "^17.0.0",
    "@medusajs/framework": "^2.0.0"
  },
  "devDependencies": {
    "typescript": "^5.6.0",
    "vite": "^6.0.0"
  }
}
```

### Update Strategy
```bash
# Vérifier updates disponibles
npm outdated

# Update patch versions (sûr)
npm update

# Update major (avec précaution)
npx npm-check-updates -u
npm install
npm test  # Vérifier pas de breaking changes
```

---

## 🎯 Prochaines Étapes

1. **Setup projet Next.js 16 + Medusa**:
   ```bash
   # Option 1: Medusa avec Next.js 16 starter (RECOMMANDÉ)
   npx create-medusa-app@latest ma-boutique-macrame
   # Choisir: Next.js 16 storefront + PostgreSQL

   # Option 2: Next.js 16 standalone + Medusa séparé
   npx create-next-app@latest frontend --typescript --app --turbo
   npx create-medusa-app@latest backend
   ```

2. **Configuration Next.js 16**:
   ```typescript
   // next.config.ts
   const config: NextConfig = {
     reactCompiler: true,
     experimental: { ppr: 'incremental' },
     images: { domains: ['pub-xxxxxxxxxx.r2.dev'] }  // R2 bucket
   }
   ```

3. **Configuration environnement**:
   ```bash
   # .env.local (Next.js)
   NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
   NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
   R2_PUBLIC_URL=https://pub-xxxxxxxxxx.r2.dev

   # .env (Medusa backend)
   DATABASE_URL=postgresql://...
   STRIPE_SECRET_KEY=sk_test_...
   JWT_SECRET=...
   ```

4. **Installation dépendances supplémentaires**:
   ```bash
   # Frontend
   cd frontend
   pnpm install @medusajs/js-sdk
   pnpm install @aws-sdk/client-s3  # Pour R2 (S3-compatible)
   pnpm install three @react-three/fiber @react-three/drei
   pnpm install gsap

   # Dev tools
   npm install -D @types/node eslint prettier husky
   ```

5. **Configuration CI/CD**:
   - GitHub Actions (Next.js 16 + Turbopack)
   - Tests automatisés (Vitest + Playwright)
   - Deployment Vercel (frontend) + Hetzner VPS (backend)
