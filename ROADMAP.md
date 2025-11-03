# 🗺️ Roadmap de Développement - Boutique Macramé E-commerce

**Stack**: Next.js 16 + Medusa.js 2.0 + PostgreSQL 17 + Stripe
**Budget**: €3.79/mois (production)
**Timeline MVP**: 8 semaines
**Timeline Production-Ready**: 9-10 semaines

---

## 📋 Vue d'ensemble

Ce roadmap suit une approche **backend-first** optimale pour 2025 :
1. ✅ Backend Medusa fournit l'API dont Next.js dépend
2. ✅ Admin panel inclus pour tester les produits avant le frontend
3. ✅ Next.js starter officiel Medusa accélère le frontend (économie 3 semaines)
4. ✅ Intégrations externes (Stripe, Resend, R2) en dernier

---

## 🚀 Phase 0 : Préparation Environnement (2-3 jours)

### Objectif
Setup environnement de développement local + Git repository

### Prérequis Système
```bash
# Vérifier versions installées
node --version   # v24.11.0+ requis
pnpm --version   # 9.x requis
docker --version # 20.x+ requis
git --version    # 2.x+ requis
```

### Checklist Installation

- [ ] **Node.js 24 LTS**
```bash
# Via nvm (recommandé)
nvm install 24
nvm use 24
nvm alias default 24
```

- [ ] **pnpm 9.x**
```bash
npm install -g pnpm@latest
pnpm --version  # Devrait afficher 9.x
```

- [ ] **Docker Desktop**
  - Télécharger depuis https://www.docker.com/products/docker-desktop
  - Vérifier que Docker est lancé : `docker ps`

- [ ] **Git + GitHub**
```bash
git config --global user.name "Ton Nom"
git config --global user.email "ton.email@example.com"
```

- [ ] **IDE Setup** (VS Code recommandé)
  - Extensions : ESLint, Prettier, Prisma, Docker
  - Settings Prettier/ESLint pour auto-format

### Structure Projet
```bash
# Créer le repository principal
mkdir boutique-macrame
cd boutique-macrame
git init

# Structure recommandée (mono-repo)
boutique-macrame/
├── backend/          # Medusa.js
├── frontend/         # Next.js 16
├── docker-compose.yml
├── .env.example
├── .gitignore
└── README.md
```

### Configuration Git
```bash
# .gitignore
node_modules/
.env
.env.local
.next/
dist/
build/
*.log
.DS_Store
```

### Livrable Phase 0
✅ Node 24 + pnpm 9 + Docker installés
✅ Repository Git initialisé
✅ Structure projet créée

**Temps estimé**: 2-3 jours (incluant téléchargements et configuration)

---

## 🔧 Phase 1 : Backend Medusa.js (5-7 jours)

### Objectif
API Medusa fonctionnelle avec PostgreSQL + données de test

### Installation Medusa

```bash
cd boutique-macrame

# Installation via create-medusa-app
npx create-medusa-app@latest

# Répondre aux prompts :
# ? What's the name of your project? → backend
# ? Which database provider would you like to use? → PostgreSQL
# ? Would you like to configure your database connection? → Skip (on va utiliser Docker)
# ? Would you like to install the Next.js storefront? → No (on le fera après)
```

### Setup PostgreSQL avec Docker

```bash
# Créer docker-compose.yml à la racine
```

**docker-compose.yml** :
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:17-alpine
    container_name: medusa-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: medusa_db
      POSTGRES_USER: medusa
      POSTGRES_PASSWORD: medusa_dev_password
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U medusa"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    container_name: medusa-redis
    restart: unless-stopped
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

volumes:
  postgres_data:
  redis_data:
```

```bash
# Lancer PostgreSQL + Redis
docker-compose up -d

# Vérifier que les services sont UP
docker-compose ps
```

### Configuration Backend

**backend/.env** :
```bash
# Database
DATABASE_URL=postgresql://medusa:medusa_dev_password@localhost:5432/medusa_db

# Redis
REDIS_URL=redis://localhost:6379

# JWT
JWT_SECRET=supersecret_change_in_production
COOKIE_SECRET=supersecret_change_in_production

# Admin
MEDUSA_ADMIN_ONBOARDING_TYPE=default

# Server
PORT=9000
```

### Migrations et Setup

```bash
cd backend

# Installer dépendances avec pnpm
pnpm install

# Créer fichier .npmrc pour compatibilité Medusa
echo "node-linker=hoisted" > .npmrc

# Réinstaller avec hoisting
pnpm install

# Lancer migrations Prisma
pnpm prisma migrate dev

# Créer utilisateur admin
pnpm medusa user -e admin@boutique-macrame.fr -p admin123
```

### Lancer Medusa

```bash
# Mode développement
pnpm dev

# Medusa devrait être accessible sur :
# - API: http://localhost:9000
# - Admin: http://localhost:9000/app
```

### Accéder à Medusa Admin

1. Ouvrir http://localhost:9000/app
2. Se connecter avec : `admin@boutique-macrame.fr` / `admin123`
3. Explorer l'interface admin

### Seed Données de Test

**backend/src/scripts/seed.ts** :
```typescript
import { MedusaContainer } from "@medusajs/framework"

export default async function seedProducts(container: MedusaContainer) {
  const productService = container.resolve("productService")

  const products = [
    {
      title: "Suspension Murale Bohème",
      handle: "suspension-murale-boheme",
      description: "Magnifique suspension en macramé fait main, parfaite pour décorer votre intérieur",
      status: "published",
      variants: [
        {
          title: "Default",
          prices: [{ amount: 4900, currency_code: "eur" }],
          inventory_quantity: 10
        }
      ],
      images: [
        "https://images.unsplash.com/photo-1615484477778-ca3b77940c25?w=500"
      ]
    },
    {
      title: "Porte-Plante en Macramé",
      handle: "porte-plante-macrame",
      description: "Support pour plante suspendu en macramé artisanal",
      status: "published",
      variants: [
        {
          title: "Default",
          prices: [{ amount: 3500, currency_code: "eur" }],
          inventory_quantity: 15
        }
      ],
      images: [
        "https://images.unsplash.com/photo-1557821552-17105176677c?w=500"
      ]
    },
    {
      title: "Tenture Murale Grande Taille",
      handle: "tenture-murale-grande",
      description: "Grande tenture murale en macramé, pièce unique",
      status: "published",
      variants: [
        {
          title: "Default",
          prices: [{ amount: 12900, currency_code: "eur" }],
          inventory_quantity: 5
        }
      ],
      images: [
        "https://images.unsplash.com/photo-1572635196184-84e35138cf62?w=500"
      ]
    }
  ]

  for (const product of products) {
    await productService.create(product)
  }

  console.log("✅ 3 produits de test créés")
}
```

```bash
# Lancer le seed
pnpm medusa exec ./src/scripts/seed.ts
```

### Tests API Backend

```bash
# Tester l'API Products
curl http://localhost:9000/store/products

# Devrait retourner les 3 produits en JSON
```

### Checklist Phase 1

- [ ] PostgreSQL + Redis lancés via Docker
- [ ] Medusa backend installé et configuré
- [ ] Migrations Prisma appliquées
- [ ] Utilisateur admin créé
- [ ] Medusa Admin accessible sur http://localhost:9000/app
- [ ] 3+ produits de test créés
- [ ] API `/store/products` retourne les produits

### Livrable Phase 1
✅ API Medusa fonctionnelle sur port 9000
✅ Admin panel accessible et opérationnel
✅ Base de données PostgreSQL avec données de test
✅ Possibilité d'ajouter/modifier produits via Admin

**Temps estimé**: 5-7 jours

---

## 🎨 Phase 2 : Frontend Next.js 16 (7-10 jours)

### Objectif
Storefront Next.js avec pages essentielles et intégration API Medusa

### Installation Next.js Starter

```bash
cd boutique-macrame

# Cloner le starter officiel Medusa
git clone https://github.com/medusajs/nextjs-starter-medusa.git frontend

cd frontend

# Installer dépendances avec pnpm
pnpm install
```

### Configuration Frontend

**frontend/.env.local** :
```bash
# Medusa Backend
NEXT_PUBLIC_MEDUSA_BACKEND_URL=http://localhost:9000
NEXT_PUBLIC_BASE_URL=http://localhost:8000

# Region par défaut
NEXT_PUBLIC_DEFAULT_REGION=fr

# Medusa Publishable Key (à récupérer dans Admin)
NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY=pk_xxxxx

# Revalidation secret (pour ISR on-demand)
REVALIDATE_SECRET=dev_secret_123
```

### Récupérer la Publishable Key

1. Ouvrir Medusa Admin : http://localhost:9000/app
2. Aller dans **Settings** → **Publishable API Keys**
3. Créer une nouvelle clé : "Storefront Key"
4. Copier la clé `pk_01xxxxx`
5. L'ajouter dans `.env.local`

### Lancer Next.js

```bash
cd frontend

# Mode développement avec Turbopack
pnpm dev

# Frontend accessible sur http://localhost:8000
```

### Structure Pages (App Router Next.js 16)

```
frontend/src/app/
├── [countryCode]/
│   ├── page.tsx                    # Home (liste produits)
│   ├── products/
│   │   └── [handle]/
│   │       └── page.tsx            # Product Detail
│   ├── cart/
│   │   └── page.tsx                # Cart
│   ├── checkout/
│   │   └── page.tsx                # Checkout
│   ├── order/
│   │   └── [id]/
│   │       └── confirmed/
│   │           └── page.tsx        # Order Confirmation
│   └── account/
│       ├── page.tsx                # Account Dashboard
│       └── orders/
│           └── page.tsx            # Order History
└── layout.tsx
```

### Développement Progressif des Pages

#### 1. Page Home (Product Listing) - PRIORITÉ 1

**app/[countryCode]/page.tsx** :
```typescript
import { Suspense } from "react"
import { getProductsList } from "@lib/data/products"
import ProductGrid from "@modules/products/components/product-grid"

export default async function HomePage({
  params,
}: {
  params: { countryCode: string }
}) {
  const { products } = await getProductsList({
    limit: 12,
    region_id: params.countryCode,
  })

  return (
    <div className="content-container">
      <h1 className="text-3xl font-bold mb-8">
        Créations Artisanales en Macramé
      </h1>

      <Suspense fallback={<div>Chargement produits...</div>}>
        <ProductGrid products={products} />
      </Suspense>
    </div>
  )
}
```

**Test** : Vérifier que les 3 produits s'affichent correctement

#### 2. Page Product Detail - PRIORITÉ 2

**app/[countryCode]/products/[handle]/page.tsx** :
```typescript
import { notFound } from "next/navigation"
import { getProductByHandle } from "@lib/data/products"
import ProductActions from "@modules/products/components/product-actions"

export default async function ProductPage({
  params,
}: {
  params: { handle: string; countryCode: string }
}) {
  const product = await getProductByHandle(params.handle)

  if (!product) {
    notFound()
  }

  return (
    <div className="product-page">
      <div className="grid grid-cols-1 md:grid-cols-2 gap-8">
        {/* Image */}
        <div className="aspect-square relative">
          <img
            src={product.thumbnail || "/placeholder.jpg"}
            alt={product.title}
            className="object-cover w-full h-full"
          />
        </div>

        {/* Info */}
        <div>
          <h1 className="text-3xl font-bold mb-4">{product.title}</h1>
          <p className="text-2xl font-semibold mb-4">
            {product.variants[0].calculated_price_string}
          </p>
          <p className="text-gray-600 mb-6">{product.description}</p>

          {/* Add to Cart - Client Component */}
          <ProductActions product={product} region={params.countryCode} />
        </div>
      </div>
    </div>
  )
}
```

**Test** : Cliquer sur un produit depuis la home et vérifier les détails

#### 3. Component Add to Cart - PRIORITÉ 3

**modules/products/components/product-actions/index.tsx** :
```typescript
"use client"

import { useState } from "react"
import { addToCart } from "@lib/data/cart"

export default function ProductActions({ product, region }) {
  const [loading, setLoading] = useState(false)
  const [success, setSuccess] = useState(false)

  async function handleAddToCart() {
    setLoading(true)

    try {
      await addToCart({
        variantId: product.variants[0].id,
        quantity: 1,
        countryCode: region,
      })

      setSuccess(true)
      setTimeout(() => setSuccess(false), 2000)
    } catch (error) {
      console.error("Erreur ajout panier:", error)
    } finally {
      setLoading(false)
    }
  }

  return (
    <button
      onClick={handleAddToCart}
      disabled={loading}
      className="bg-black text-white px-8 py-3 rounded-md hover:bg-gray-800 disabled:opacity-50"
    >
      {loading ? "Ajout..." : success ? "✓ Ajouté !" : "Ajouter au panier"}
    </button>
  )
}
```

**Test** : Ajouter un produit au panier et vérifier dans DevTools Network

#### 4. Page Cart - PRIORITÉ 4

**app/[countryCode]/cart/page.tsx** :
```typescript
import { getCart } from "@lib/data/cart"
import CartItems from "@modules/cart/components/items"
import CartTotals from "@modules/cart/components/totals"
import CheckoutButton from "@modules/cart/components/checkout-button"

export default async function CartPage() {
  const cart = await getCart()

  if (!cart || cart.items.length === 0) {
    return (
      <div className="content-container py-12 text-center">
        <h1 className="text-2xl font-bold mb-4">Votre panier est vide</h1>
        <a href="/" className="text-blue-600 hover:underline">
          Continuer vos achats
        </a>
      </div>
    )
  }

  return (
    <div className="content-container py-12">
      <h1 className="text-3xl font-bold mb-8">Panier</h1>

      <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
        {/* Items */}
        <div className="lg:col-span-2">
          <CartItems items={cart.items} />
        </div>

        {/* Summary */}
        <div className="lg:col-span-1">
          <CartTotals cart={cart} />
          <CheckoutButton />
        </div>
      </div>
    </div>
  )
}
```

**Test** : Accéder à `/cart` et vérifier que les produits ajoutés s'affichent

### Checklist Phase 2

- [ ] Next.js starter installé et lancé sur port 8000
- [ ] Variables d'environnement configurées
- [ ] Page Home affiche la liste des produits
- [ ] Page Product Detail fonctionne
- [ ] Bouton "Ajouter au panier" fonctionnel
- [ ] Page Cart affiche les produits ajoutés
- [ ] Navigation entre pages fluide

### Livrable Phase 2
✅ Frontend Next.js opérationnel
✅ Intégration API Medusa validée
✅ User peut naviguer et ajouter au panier
✅ Base du storefront fonctionnelle

**Temps estimé**: 7-10 jours

---

## 💳 Phase 3 : Paiements Stripe (4-5 jours)

### Objectif
Checkout fonctionnel avec paiements Stripe en mode test

### Prérequis Stripe

1. Créer compte Stripe : https://dashboard.stripe.com/register
2. Activer mode Test
3. Récupérer les clés API de test

### Configuration Stripe Backend

**backend/.env** (ajouter) :
```bash
# Stripe
STRIPE_API_KEY=sk_test_xxxxx
STRIPE_WEBHOOK_SECRET=whsec_xxxxx
```

```bash
cd backend

# Installer le plugin Stripe Medusa
pnpm add @medusajs/medusa/payment-stripe
```

**backend/medusa-config.ts** (ajouter) :
```typescript
import { loadEnv, Modules } from "@medusajs/framework/utils"

loadEnv(process.env.NODE_ENV || "development", process.cwd())

export default {
  // ... config existante

  modules: [
    {
      resolve: "@medusajs/medusa/payment-stripe",
      options: {
        apiKey: process.env.STRIPE_API_KEY,
      },
    },
  ],
}
```

```bash
# Relancer Medusa pour charger le module
pnpm dev
```

### Configuration Stripe Frontend

**frontend/.env.local** (ajouter) :
```bash
# Stripe
NEXT_PUBLIC_STRIPE_KEY=pk_test_xxxxx
```

### Webhooks Stripe Local

```bash
# Installer Stripe CLI
# macOS
brew install stripe/stripe-cli/stripe

# Windows
# Télécharger depuis https://github.com/stripe/stripe-cli/releases

# Login Stripe
stripe login

# Lancer webhook forwarding
stripe listen --forward-to localhost:9000/webhooks/stripe

# Copier le webhook secret affiché (whsec_xxxxx)
# L'ajouter dans backend/.env comme STRIPE_WEBHOOK_SECRET
```

### Page Checkout avec Stripe Payment Element

**app/[countryCode]/checkout/page.tsx** :
```typescript
import { getCart } from "@lib/data/cart"
import CheckoutForm from "@modules/checkout/components/checkout-form"
import { redirect } from "next/navigation"

export default async function CheckoutPage() {
  const cart = await getCart()

  if (!cart || cart.items.length === 0) {
    redirect("/cart")
  }

  return (
    <div className="content-container py-12">
      <h1 className="text-3xl font-bold mb-8">Paiement</h1>

      <CheckoutForm cart={cart} />
    </div>
  )
}
```

**modules/checkout/components/checkout-form/index.tsx** :
```typescript
"use client"

import { loadStripe } from "@stripe/stripe-js"
import { Elements } from "@stripe/react-stripe-js"
import PaymentElement from "./payment-element"

const stripePromise = loadStripe(
  process.env.NEXT_PUBLIC_STRIPE_KEY!
)

export default function CheckoutForm({ cart }) {
  return (
    <Elements
      stripe={stripePromise}
      options={{
        clientSecret: cart.payment_session?.data?.client_secret,
      }}
    >
      <div className="grid grid-cols-1 lg:grid-cols-2 gap-8">
        <PaymentElement />

        <div>
          {/* Order Summary */}
          <h2 className="text-xl font-bold mb-4">Résumé</h2>
          {/* Cart items + totals */}
        </div>
      </div>
    </Elements>
  )
}
```

### Tests Paiements

```bash
# Carte test Stripe (succès)
Numéro: 4242 4242 4242 4242
Date: 12/34
CVC: 123
ZIP: 12345

# Carte test (refus)
Numéro: 4000 0000 0000 0002
```

### Flow Complet à Tester

1. Ajouter produit au panier
2. Aller sur `/cart`
3. Cliquer "Passer commande"
4. Entrer adresse de livraison
5. Entrer carte test Stripe
6. Valider paiement
7. Vérifier redirection vers `/order/[id]/confirmed`
8. Vérifier email confirmation (si Resend configuré)

### Checklist Phase 3

- [ ] Plugin Stripe installé dans Medusa
- [ ] Stripe CLI webhook forwarding actif
- [ ] Clés Stripe configurées (frontend + backend)
- [ ] Page Checkout avec Stripe Payment Element
- [ ] Paiement test avec carte 4242 fonctionne
- [ ] Webhook Stripe reçu et traité
- [ ] Commande créée dans Medusa Admin
- [ ] Page Order Confirmation s'affiche

### Livrable Phase 3
✅ Paiements Stripe opérationnels en mode test
✅ User peut finaliser un achat complet
✅ Commandes visibles dans Medusa Admin

**Temps estimé**: 4-5 jours

---

## 📧 Phase 4 : Emails avec Resend (2-3 jours)

### Objectif
Envoi automatique d'emails de confirmation de commande

### Setup Resend

1. Créer compte : https://resend.com
2. Vérifier email
3. Créer API key
4. FREE tier : 100 emails/jour

### Installation Resend

```bash
cd backend
pnpm add resend react-email
```

**backend/.env** (ajouter) :
```bash
# Resend
RESEND_API_KEY=re_xxxxx
FROM_EMAIL=boutique@votredomaine.fr  # ou noreply@resend.dev
```

### Email Template avec React Email

**backend/src/emails/order-confirmation.tsx** :
```tsx
import {
  Body,
  Container,
  Head,
  Heading,
  Html,
  Img,
  Link,
  Preview,
  Text,
} from "@react-email/components"

interface OrderConfirmationProps {
  orderNumber: string
  customerName: string
  total: string
  items: Array<{
    title: string
    quantity: number
    price: string
  }>
}

export default function OrderConfirmation({
  orderNumber,
  customerName,
  total,
  items,
}: OrderConfirmationProps) {
  return (
    <Html>
      <Head />
      <Preview>Confirmation de commande #{orderNumber}</Preview>
      <Body style={main}>
        <Container style={container}>
          <Heading style={h1}>
            Merci pour votre commande !
          </Heading>

          <Text style={text}>
            Bonjour {customerName},
          </Text>

          <Text style={text}>
            Nous avons bien reçu votre commande <strong>#{orderNumber}</strong>.
          </Text>

          <Heading as="h2" style={h2}>
            Articles commandés
          </Heading>

          {items.map((item, index) => (
            <div key={index} style={item}>
              <Text>{item.quantity}x {item.title} - {item.price}</Text>
            </div>
          ))}

          <Text style={total}>
            <strong>Total : {total}</strong>
          </Text>

          <Text style={text}>
            Vous recevrez un email de confirmation d'expédition dès que votre commande sera envoyée.
          </Text>

          <Link href="https://votresite.fr/account/orders" style={button}>
            Voir ma commande
          </Link>
        </Container>
      </Body>
    </Html>
  )
}

const main = {
  backgroundColor: "#f6f9fc",
  fontFamily: '-apple-system,BlinkMacSystemFont,"Segoe UI",Roboto,"Helvetica Neue",Ubuntu,sans-serif',
}

const container = {
  backgroundColor: "#ffffff",
  margin: "0 auto",
  padding: "20px 0 48px",
  marginBottom: "64px",
}

const h1 = {
  color: "#333",
  fontSize: "24px",
  fontWeight: "bold",
  margin: "40px 0",
  padding: "0",
}
```

### Service Email

**backend/src/services/email.ts** :
```typescript
import { Resend } from "resend"
import OrderConfirmation from "../emails/order-confirmation"

const resend = new Resend(process.env.RESEND_API_KEY)

export async function sendOrderConfirmation(order: any) {
  try {
    const { data, error } = await resend.emails.send({
      from: process.env.FROM_EMAIL!,
      to: order.email,
      subject: `Confirmation de commande #${order.display_id}`,
      react: OrderConfirmation({
        orderNumber: order.display_id.toString(),
        customerName: `${order.customer.first_name} ${order.customer.last_name}`,
        total: `${order.total / 100}€`,
        items: order.items.map((item: any) => ({
          title: item.title,
          quantity: item.quantity,
          price: `${item.unit_price / 100}€`,
        })),
      }),
    })

    if (error) {
      console.error("Erreur envoi email:", error)
      return { success: false, error }
    }

    console.log("✅ Email envoyé:", data?.id)
    return { success: true, data }
  } catch (error) {
    console.error("Erreur Resend:", error)
    return { success: false, error }
  }
}
```

### Subscriber Medusa (envoi auto après commande)

**backend/src/subscribers/order-placed.ts** :
```typescript
import { SubscriberConfig } from "@medusajs/framework"
import { sendOrderConfirmation } from "../services/email"

export default async function orderPlacedHandler({
  event: { data },
  container,
}) {
  const orderService = container.resolve("orderService")

  // Récupérer la commande complète
  const order = await orderService.retrieve(data.id, {
    relations: ["items", "customer"],
  })

  // Envoyer l'email
  await sendOrderConfirmation(order)
}

export const config: SubscriberConfig = {
  event: "order.placed",
}
```

### Test Email

```bash
# Relancer Medusa
cd backend
pnpm dev

# Faire un achat complet sur le frontend
# Vérifier dans la console backend : "✅ Email envoyé: xxx"
# Vérifier réception email
```

### Checklist Phase 4

- [ ] Resend API key configurée
- [ ] Template email créé avec React Email
- [ ] Service email implémenté
- [ ] Subscriber order.placed configuré
- [ ] Email de test reçu après commande
- [ ] Email contient les bonnes informations (produits, total)

### Livrable Phase 4
✅ Emails de confirmation automatiques
✅ Templates professionnels
✅ 100 emails/jour gratuits (FREE tier)

**Temps estimé**: 2-3 jours

---

## 🧪 Phase 5 : Tests E2E Critiques (3-4 jours)

### Objectif
Valider les parcours utilisateur critiques avec Playwright

### Installation Playwright

```bash
cd frontend
pnpm add -D @playwright/test
pnpm exec playwright install
```

**playwright.config.ts** :
```typescript
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './tests',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',

  use: {
    baseURL: 'http://localhost:8000',
    trace: 'on-first-retry',
  },

  projects: [
    {
      name: 'chromium',
      use: { ...devices['Desktop Chrome'] },
    },
  ],

  webServer: {
    command: 'pnpm dev',
    url: 'http://localhost:8000',
    reuseExistingServer: !process.env.CI,
  },
})
```

### Test 1 : Browse to Cart

**tests/e2e/browse-to-cart.spec.ts** :
```typescript
import { test, expect } from '@playwright/test'

test.describe('Browse to Cart Flow', () => {
  test('user can browse products and add to cart', async ({ page }) => {
    // 1. Aller sur la home
    await page.goto('/')

    // 2. Vérifier que des produits s'affichent
    const products = page.locator('[data-testid="product-card"]')
    await expect(products).toHaveCount(3)

    // 3. Cliquer sur le premier produit
    await products.first().click()

    // 4. Vérifier URL produit
    await expect(page).toHaveURL(/\/products\//)

    // 5. Cliquer "Ajouter au panier"
    await page.click('button:has-text("Ajouter au panier")')

    // 6. Vérifier succès
    await expect(page.locator('button:has-text("✓ Ajouté !")')).toBeVisible()

    // 7. Aller au panier
    await page.click('a[href="/cart"]')

    // 8. Vérifier produit dans le panier
    await expect(page.locator('[data-testid="cart-item"]')).toHaveCount(1)
  })
})
```

### Test 2 : Complete Checkout (Mode Test Stripe)

**tests/e2e/checkout.spec.ts** :
```typescript
import { test, expect } from '@playwright/test'

test.describe('Checkout Flow', () => {
  test('user can complete purchase with test card', async ({ page }) => {
    // Setup: Ajouter produit au panier
    await page.goto('/')
    await page.locator('[data-testid="product-card"]').first().click()
    await page.click('button:has-text("Ajouter au panier")')
    await page.click('a[href="/cart"]')

    // 1. Aller au checkout
    await page.click('button:has-text("Passer commande")')
    await expect(page).toHaveURL(/\/checkout/)

    // 2. Remplir adresse
    await page.fill('input[name="email"]', 'test@example.com')
    await page.fill('input[name="first_name"]', 'John')
    await page.fill('input[name="last_name"]', 'Doe')
    await page.fill('input[name="address_1"]', '123 Test St')
    await page.fill('input[name="city"]', 'Paris')
    await page.fill('input[name="postal_code"]', '75001')
    await page.selectOption('select[name="country_code"]', 'FR')

    // 3. Remplir Stripe Payment Element (test card)
    const stripeFrame = page.frameLocator('iframe[name*="stripe"]')
    await stripeFrame.locator('input[name="cardnumber"]').fill('4242424242424242')
    await stripeFrame.locator('input[name="exp-date"]').fill('1234')
    await stripeFrame.locator('input[name="cvc"]').fill('123')
    await stripeFrame.locator('input[name="postal"]').fill('12345')

    // 4. Valider paiement
    await page.click('button:has-text("Payer")')

    // 5. Attendre redirection confirmation
    await page.waitForURL(/\/order\/.*\/confirmed/, { timeout: 15000 })

    // 6. Vérifier page confirmation
    await expect(page.locator('h1:has-text("Commande confirmée")')).toBeVisible()
    await expect(page.locator('text=/Numéro de commande : #\d+/')).toBeVisible()
  })
})
```

### Lancer les Tests

```bash
# Lancer tous les tests
pnpm playwright test

# Lancer en mode UI (debug)
pnpm playwright test --ui

# Lancer un test spécifique
pnpm playwright test tests/e2e/checkout.spec.ts
```

### Checklist Phase 5

- [ ] Playwright installé et configuré
- [ ] Test "Browse to Cart" passe
- [ ] Test "Complete Checkout" passe avec carte test
- [ ] Rapport HTML généré
- [ ] Tests lancés en CI (optionnel)

### Livrable Phase 5
✅ Tests E2E critiques validés
✅ Confiance dans le parcours utilisateur
✅ Détection automatique des régressions

**Temps estimé**: 3-4 jours

---

## 🚀 Phase 6 : Déploiement Production (5-7 jours)

### Objectif
Site en production accessible au public avec paiements réels

### Ordre de Déploiement (CRITIQUE)

**1. Database FIRST → 2. Backend → 3. Frontend → 4. Services Externes**

### 6.1 Supabase PostgreSQL (Database)

1. **Créer projet Supabase**
   - Aller sur https://supabase.com
   - Créer nouveau projet : "boutique-macrame-prod"
   - Région : Europe (West) pour la France
   - Plan : FREE (500MB always-on)

2. **Récupérer DATABASE_URL**
   ```
   Settings → Database → Connection String → URI

   Format:
   postgresql://postgres:[PASSWORD]@db.[PROJECT_REF].supabase.co:5432/postgres
   ```

3. **Migrer schéma Prisma**
   ```bash
   cd backend

   # Ajouter DATABASE_URL production dans .env.production
   DATABASE_URL=postgresql://postgres:xxx@db.xxx.supabase.co:5432/postgres

   # Lancer migrations
   pnpm prisma migrate deploy
   ```

4. **Vérifier tables**
   - Ouvrir Supabase Dashboard → Table Editor
   - Vérifier que les tables Medusa sont créées

### 6.2 Upstash Redis (Cache + Rate Limiting)

1. **Créer database Upstash**
   - Aller sur https://upstash.com
   - Créer nouvelle database Redis
   - Région : Europe (Ireland)
   - Plan : FREE (10k commands/jour)

2. **Récupérer REDIS_URL**
   ```
   Format:
   redis://default:[PASSWORD]@[ENDPOINT]:6379
   ```

### 6.3 Hetzner VPS (Backend Medusa)

1. **Créer serveur Hetzner**
   - Type : CAX11 (ARM)
   - OS : Ubuntu 24.04 LTS
   - Localisation : Falkenstein (Allemagne)
   - Prix : €3.79/mois

2. **SSH Setup**
   ```bash
   # Se connecter au serveur
   ssh root@[IP_SERVER]

   # Mettre à jour système
   apt update && apt upgrade -y

   # Installer Docker
   curl -fsSL https://get.docker.com -o get-docker.sh
   sh get-docker.sh

   # Installer Docker Compose
   apt install docker-compose-plugin -y
   ```

3. **Déployer Backend**
   ```bash
   # Sur le serveur Hetzner
   mkdir -p /opt/medusa
   cd /opt/medusa

   # Créer docker-compose.yml
   nano docker-compose.yml
   ```

   **docker-compose.yml production** :
   ```yaml
   version: '3.8'

   services:
     medusa:
       image: node:24-alpine
       container_name: medusa-backend
       restart: unless-stopped
       working_dir: /app
       environment:
         NODE_ENV: production
         DATABASE_URL: ${DATABASE_URL}
         REDIS_URL: ${REDIS_URL}
         JWT_SECRET: ${JWT_SECRET}
         COOKIE_SECRET: ${COOKIE_SECRET}
         STRIPE_API_KEY: ${STRIPE_API_KEY}
         STRIPE_WEBHOOK_SECRET: ${STRIPE_WEBHOOK_SECRET}
         RESEND_API_KEY: ${RESEND_API_KEY}
         PORT: 9000
       volumes:
         - ./backend:/app
       ports:
         - "9000:9000"
       command: sh -c "pnpm install && pnpm build && pnpm start"
       healthcheck:
         test: ["CMD", "curl", "-f", "http://localhost:9000/health"]
         interval: 30s
         timeout: 10s
         retries: 3

     caddy:
       image: caddy:2-alpine
       container_name: caddy-reverse-proxy
       restart: unless-stopped
       ports:
         - "80:80"
         - "443:443"
       volumes:
         - ./Caddyfile:/etc/caddy/Caddyfile
         - caddy_data:/data
         - caddy_config:/config

   volumes:
     caddy_data:
     caddy_config:
   ```

   **Caddyfile** (SSL automatique) :
   ```
   api.votredomaine.fr {
       reverse_proxy medusa:9000

       encode gzip

       header {
           Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
           X-Content-Type-Options "nosniff"
           X-Frame-Options "DENY"
           Referrer-Policy "strict-origin-when-cross-origin"
       }

       log {
           output file /var/log/caddy/access.log
       }
   }
   ```

   **Variables d'environnement (.env)** :
   ```bash
   DATABASE_URL=postgresql://postgres:xxx@db.xxx.supabase.co:5432/postgres
   REDIS_URL=redis://default:xxx@xxx.upstash.io:6379
   JWT_SECRET=[générer secret fort]
   COOKIE_SECRET=[générer secret fort]
   STRIPE_API_KEY=sk_live_xxxxx
   STRIPE_WEBHOOK_SECRET=whsec_xxxxx
   RESEND_API_KEY=re_xxxxx
   FROM_EMAIL=boutique@votredomaine.fr
   ```

4. **Déployer le code backend**
   ```bash
   # Sur votre machine locale
   cd backend
   tar -czf backend.tar.gz .
   scp backend.tar.gz root@[IP_SERVER]:/opt/medusa/

   # Sur le serveur
   cd /opt/medusa
   tar -xzf backend.tar.gz -C backend/
   rm backend.tar.gz

   # Lancer les containers
   docker compose up -d

   # Vérifier logs
   docker compose logs -f medusa
   ```

5. **Tester API production**
   ```bash
   curl https://api.votredomaine.fr/health
   # Devrait retourner {"status": "ok"}
   ```

### 6.4 Vercel (Frontend Next.js)

1. **Connecter repository GitHub**
   - Push code frontend sur GitHub
   - Aller sur https://vercel.com
   - Import project depuis GitHub

2. **Configuration Vercel**
   ```
   Framework Preset: Next.js
   Root Directory: frontend/
   Build Command: pnpm build
   Output Directory: .next
   Install Command: pnpm install
   Node Version: 24.x
   ```

3. **Variables d'environnement Vercel**
   ```bash
   NEXT_PUBLIC_MEDUSA_BACKEND_URL=https://api.votredomaine.fr
   NEXT_PUBLIC_BASE_URL=https://votresite.fr
   NEXT_PUBLIC_DEFAULT_REGION=fr
   NEXT_PUBLIC_MEDUSA_PUBLISHABLE_KEY=pk_live_xxxxx
   NEXT_PUBLIC_STRIPE_KEY=pk_live_xxxxx
   REVALIDATE_SECRET=[secret fort]
   ```

4. **Deploy**
   - Cliquer "Deploy"
   - Attendre build (~3-5 min)
   - Vérifier site : `https://votresite.vercel.app`

5. **Custom Domain**
   - Settings → Domains
   - Ajouter `votresite.fr`
   - Configurer DNS (A record vers Vercel IPs)

### 6.5 Configuration Stripe Production

1. **Webhooks Production**
   - Aller sur Stripe Dashboard
   - Developers → Webhooks
   - Add endpoint : `https://api.votredomaine.fr/webhooks/stripe`
   - Events : `payment_intent.succeeded`, `payment_intent.payment_failed`
   - Copier webhook secret `whsec_xxxxx`

2. **Mettre à jour backend .env**
   ```bash
   STRIPE_WEBHOOK_SECRET=whsec_xxxxx
   ```

3. **Redémarrer Medusa**
   ```bash
   docker compose restart medusa
   ```

### 6.6 Cloudflare R2 (Images - Optionnel Phase 7)

Pour l'instant, laisser images en local (`/public/`) ou Unsplash URLs.
Migration vers R2 dans Phase 7 (optimisations).

### Tests Production Complets

#### Checklist Tests Production

- [ ] **API Backend**
  ```bash
  curl https://api.votredomaine.fr/store/products
  # Devrait retourner produits
  ```

- [ ] **Frontend Vercel**
  - Ouvrir https://votresite.fr
  - Vérifier home page charge
  - Vérifier images s'affichent
  - Vérifier navigation fonctionne

- [ ] **Purchase Flow Complet**
  1. Ajouter produit au panier
  2. Aller au checkout
  3. Entrer vraie adresse
  4. Utiliser carte test Stripe : `4242 4242 4242 4242`
  5. Valider paiement
  6. Vérifier page confirmation
  7. Vérifier email reçu
  8. Vérifier commande dans Medusa Admin

- [ ] **SSL/HTTPS**
  - Vérifier cadenas vert sur frontend et backend
  - Tester https://www.ssllabs.com/ssltest/

- [ ] **Performance**
  - PageSpeed Insights : https://pagespeed.web.dev/
  - Objectif : Score > 90 mobile et desktop

- [ ] **Monitoring**
  - Vérifier logs backend : `docker compose logs -f`
  - Vérifier logs Vercel : Dashboard → Logs

### Configuration DNS Finale

**Pour votresite.fr** (frontend) :
```
Type: A
Host: @
Value: 76.76.21.21 (Vercel IP)

Type: CNAME
Host: www
Value: cname.vercel-dns.com
```

**Pour api.votredomaine.fr** (backend) :
```
Type: A
Host: api
Value: [IP_SERVEUR_HETZNER]
```

### Checklist Phase 6

- [ ] Supabase PostgreSQL configuré et migrations appliquées
- [ ] Upstash Redis configuré
- [ ] Backend Medusa déployé sur Hetzner VPS
- [ ] Caddy reverse proxy avec SSL actif
- [ ] Frontend Next.js déployé sur Vercel
- [ ] DNS configuré et propagé
- [ ] Stripe webhooks production configurés
- [ ] Achat test complet réussi avec carte test
- [ ] Email confirmation reçu
- [ ] Site accessible en HTTPS

### Livrable Phase 6
✅ Site en production accessible au public
✅ Backend API sécurisé avec SSL
✅ Paiements Stripe en mode production (test cards)
✅ Monitoring et logs actifs

**Temps estimé**: 5-7 jours

---

## 🎯 Phase 7 : Optimisations (Optionnel - 3-5 jours)

### Objectif
Optimiser performance, SEO, et migrer images vers Cloudflare R2

### 7.1 Migration Images vers R2

**Pourquoi migrer maintenant** :
- Images locales `/public/` chargent le bundle Next.js
- R2 = CDN global + 0€ egress + compression automatique

#### Setup Cloudflare R2

1. **Créer bucket R2**
   - Aller sur Cloudflare Dashboard
   - R2 → Create Bucket : "boutique-macrame-images"
   - FREE : 10GB storage

2. **Créer API Token**
   - R2 → Manage R2 API Tokens
   - Créer token avec permissions Read/Write
   - Copier Access Key ID et Secret Access Key

3. **Configuration Backend**
   ```bash
   cd backend
   pnpm add @aws-sdk/client-s3
   ```

   **backend/.env** (ajouter) :
   ```bash
   R2_ACCOUNT_ID=xxxxx
   R2_ACCESS_KEY_ID=xxxxx
   R2_SECRET_ACCESS_KEY=xxxxx
   R2_BUCKET_NAME=boutique-macrame-images
   R2_PUBLIC_URL=https://pub-xxxxx.r2.dev
   ```

4. **Service Upload R2**
   ```typescript
   // backend/src/services/r2-upload.ts
   import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3"

   const s3Client = new S3Client({
     region: "auto",
     endpoint: `https://${process.env.R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
     credentials: {
       accessKeyId: process.env.R2_ACCESS_KEY_ID!,
       secretAccessKey: process.env.R2_SECRET_ACCESS_KEY!,
     },
   })

   export async function uploadToR2(
     file: Buffer,
     filename: string,
     contentType: string
   ) {
     const command = new PutObjectCommand({
       Bucket: process.env.R2_BUCKET_NAME,
       Key: filename,
       Body: file,
       ContentType: contentType,
     })

     await s3Client.send(command)

     return `${process.env.R2_PUBLIC_URL}/${filename}`
   }
   ```

5. **Mettre à jour next.config.ts**
   ```typescript
   images: {
     domains: ['pub-xxxxx.r2.dev'],
     formats: ['image/avif', 'image/webp']
   }
   ```

### 7.2 SEO Optimization

**app/[countryCode]/layout.tsx** :
```typescript
import type { Metadata } from 'next'

export const metadata: Metadata = {
  title: 'Boutique Macramé - Créations Artisanales Fait Main',
  description: 'Découvrez nos créations uniques en macramé : suspensions murales, porte-plantes, tentures. Fait main avec amour en France.',
  keywords: 'macramé, artisanal, fait main, décoration, bohème, suspension murale',
  openGraph: {
    type: 'website',
    locale: 'fr_FR',
    url: 'https://votresite.fr',
    title: 'Boutique Macramé - Créations Artisanales',
    description: 'Créations uniques en macramé fait main',
    images: [
      {
        url: 'https://votresite.fr/og-image.jpg',
        width: 1200,
        height: 630,
      }
    ],
  },
  twitter: {
    card: 'summary_large_image',
    title: 'Boutique Macramé',
    description: 'Créations artisanales en macramé',
  },
}
```

**robots.txt** (public/robots.txt) :
```
User-agent: *
Allow: /

Sitemap: https://votresite.fr/sitemap.xml
```

**Sitemap dynamique** (app/sitemap.ts) :
```typescript
import { getProductsList } from '@lib/data/products'

export default async function sitemap() {
  const { products } = await getProductsList({ limit: 100 })

  const productUrls = products.map((product) => ({
    url: `https://votresite.fr/products/${product.handle}`,
    lastModified: product.updated_at,
    changeFrequency: 'weekly' as const,
    priority: 0.8,
  }))

  return [
    {
      url: 'https://votresite.fr',
      lastModified: new Date(),
      changeFrequency: 'daily' as const,
      priority: 1,
    },
    ...productUrls,
  ]
}
```

### 7.3 Performance Optimization

**Lazy Loading Components** :
```typescript
import dynamic from 'next/dynamic'

const ProductReviews = dynamic(() => import('./ProductReviews'), {
  loading: () => <p>Chargement avis...</p>,
  ssr: false, // Ne charger que côté client
})
```

**Image Optimization** :
```typescript
import Image from 'next/image'

<Image
  src={product.thumbnail}
  alt={product.title}
  width={500}
  height={500}
  priority={false} // lazy load
  placeholder="blur"
  blurDataURL="data:image/..."
/>
```

**Font Optimization** (app/layout.tsx) :
```typescript
import { Inter } from 'next/font/google'

const inter = Inter({
  subsets: ['latin'],
  display: 'swap',
  preload: true,
})

export default function RootLayout({ children }) {
  return (
    <html lang="fr" className={inter.className}>
      {children}
    </html>
  )
}
```

### 7.4 Analytics (Google Analytics 4)

```bash
pnpm add @next/third-parties
```

**app/layout.tsx** :
```typescript
import { GoogleAnalytics } from '@next/third-parties/google'

export default function RootLayout({ children }) {
  return (
    <html>
      <body>{children}</body>
      <GoogleAnalytics gaId="G-XXXXXXXXXX" />
    </html>
  )
}
```

### Checklist Phase 7

- [ ] Images migrées vers Cloudflare R2
- [ ] Metadata SEO configurés sur toutes les pages
- [ ] Sitemap.xml généré dynamiquement
- [ ] robots.txt configuré
- [ ] Lazy loading implémenté
- [ ] Google Analytics configuré
- [ ] PageSpeed score > 90

### Livrable Phase 7
✅ Site optimisé pour SEO et performance
✅ Images servies depuis CDN global (R2)
✅ Analytics tracking actif

**Temps estimé**: 3-5 jours

---

## 📊 Résumé Timeline

| Phase | Durée | Effort | Priorité |
|-------|-------|--------|----------|
| **Phase 0 : Préparation** | 2-3 jours | Faible | 🔴 Critique |
| **Phase 1 : Backend Medusa** | 5-7 jours | Moyen | 🔴 Critique |
| **Phase 2 : Frontend Next.js** | 7-10 jours | Élevé | 🔴 Critique |
| **Phase 3 : Paiements Stripe** | 4-5 jours | Moyen | 🔴 Critique |
| **Phase 4 : Emails Resend** | 2-3 jours | Faible | 🟡 Important |
| **Phase 5 : Tests E2E** | 3-4 jours | Moyen | 🟡 Important |
| **Phase 6 : Déploiement Prod** | 5-7 jours | Élevé | 🔴 Critique |
| **Phase 7 : Optimisations** | 3-5 jours | Moyen | 🟢 Optionnel |
| **TOTAL MVP (Phase 0-4)** | **20-28 jours** | **~4-6 semaines** | - |
| **TOTAL Production (Phase 0-6)** | **28-39 jours** | **~6-8 semaines** | - |
| **TOTAL Optimisé (Phase 0-7)** | **31-44 jours** | **~6-9 semaines** | - |

---

## 🎯 Milestones Clés

### ✅ Milestone 1 : Backend API Functional (Fin Phase 1)
- Medusa API répond sur `http://localhost:9000`
- Admin panel accessible et fonctionnel
- 3+ produits de test créés

### ✅ Milestone 2 : Frontend MVP (Fin Phase 2)
- User peut naviguer dans le catalogue
- User peut ajouter au panier
- Frontend communique avec backend API

### ✅ Milestone 3 : Checkout Functional (Fin Phase 3)
- User peut finaliser un achat avec Stripe (test)
- Webhooks Stripe fonctionnels
- Commandes créées dans Medusa Admin

### ✅ Milestone 4 : Production Live (Fin Phase 6)
- Site accessible publiquement avec HTTPS
- Paiements réels possibles
- Monitoring actif

---

## 💡 Conseils Pratiques

### ⚠️ Erreurs Fréquentes à Éviter

1. **Ne PAS développer tout le frontend d'un coup**
   - Valider l'intégration API sur la home AVANT de faire les autres pages
   - Approche incrémentale : Home → Product → Cart → Checkout

2. **Ne PAS skip les tests de paiement**
   - Toujours tester avec carte test Stripe AVANT la production
   - Vérifier que les webhooks sont reçus

3. **Ne PAS déployer frontend avant backend**
   - Ordre critique : Database → Backend → Frontend
   - Sinon = erreurs 404 API sur le frontend

4. **Ne PAS oublier les webhooks production**
   - Stripe webhook URL doit pointer vers backend production
   - Vérifier secret webhook mis à jour

5. **Ne PAS utiliser clés API dev en production**
   - Toujours utiliser clés production (Stripe, Resend, etc.)
   - Jamais commiter de clés dans Git

### 🚀 Accélérateurs

- **Utiliser le Next.js starter officiel Medusa** = économie 3 semaines
- **Docker dès le début** = environnement identique dev/prod
- **pnpm au lieu de npm** = installations 3x plus rapides
- **Turbopack (Next.js 16)** = HMR 10x plus rapide

### 📚 Ressources Utiles

- **Medusa Docs** : https://docs.medusajs.com
- **Next.js 16 Docs** : https://nextjs.org/docs
- **Stripe Docs** : https://stripe.com/docs
- **Supabase Docs** : https://supabase.com/docs
- **Playwright Docs** : https://playwright.dev

---

## 🎉 Conclusion

Ce roadmap vous guide étape par étape pour construire une boutique e-commerce moderne et performante en **8 semaines**.

**Points clés** :
- ✅ Backend-first approach (Medusa API d'abord)
- ✅ Utilisation des starters officiels (gain 3 semaines)
- ✅ Tests progressifs à chaque phase
- ✅ Déploiement méthodique (Database → Backend → Frontend)
- ✅ Budget minimal : €3.79/mois en production

**Prochaine étape** : [Phase 0 - Préparation Environnement](#-phase-0--préparation-environnement-2-3-jours)

Bon développement ! 🚀
