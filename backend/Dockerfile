# =============================================================================
# Medusa Backend Dockerfile
# Multi-stage build for optimized production image
# =============================================================================
# Node.js 24.x LTS (Krypton) - Active LTS until April 2028
# Compatible with Medusa v2.x (requires Node.js v20+)
# =============================================================================

# -----------------------------------------------------------------------------
# Stage 1: Dependencies
# -----------------------------------------------------------------------------
FROM node:24-alpine AS deps

WORKDIR /app

# Install dependencies for native modules (Medusa requirements)
RUN apk add --no-cache \
    libc6-compat \
    python3 \
    make \
    g++ \
    && rm -rf /var/cache/apk/*

# Copy package files
COPY package*.json ./

# Install production dependencies only
RUN npm ci --only=production --omit=dev && \
    npm cache clean --force

# -----------------------------------------------------------------------------
# Stage 2: Builder
# -----------------------------------------------------------------------------
FROM node:24-alpine AS builder

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install ALL dependencies (including devDependencies for build)
RUN npm ci && \
    npm cache clean --force

# Copy source code
COPY . .

# Build Medusa (TypeScript → JavaScript)
# Remove "|| true" to fail on build errors in production
RUN npm run build

# -----------------------------------------------------------------------------
# Stage 3: Production Runtime
# -----------------------------------------------------------------------------
FROM node:24-alpine AS runner

WORKDIR /app

# Install runtime dependencies only
RUN apk add --no-cache \
    dumb-init \
    && rm -rf /var/cache/apk/*

# Create non-root user for security
RUN addgroup --system --gid 1001 medusa && \
    adduser --system --uid 1001 medusa

# Copy production dependencies from deps stage
COPY --from=deps --chown=medusa:medusa /app/node_modules ./node_modules

# Copy built application from builder stage
COPY --from=builder --chown=medusa:medusa /app ./

# Switch to non-root user
USER medusa

# Expose Medusa port
EXPOSE 9000

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
  CMD node -e "require('http').get('http://localhost:9000/health', (r) => {process.exit(r.statusCode === 200 ? 0 : 1)})"

# Use dumb-init to handle signals properly
ENTRYPOINT ["/usr/bin/dumb-init", "--"]

# Start Medusa
CMD ["npm", "start"]
