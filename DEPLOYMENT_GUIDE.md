# Production Deployment & Mobile APK Generation Guide

A complete, production-ready guide for deploying the **Assurance: Insurance Policy and Claims Processing System** backend to cloud infrastructure and generating standalone Android APKs and Google Play App Bundles (AAB).

---

## Table of Contents
1. [Backend Production Configuration](#1-backend-production-configuration)
2. [Database Connection Pooling & SSL Migration Strategy](#2-database-connection-pooling--ssl-migration-strategy)
3. [Containerization: Production Dockerfile & Docker Compose](#3-containerization-production-dockerfile--docker-compose)
4. [Cloud Hosting Deployment Strategies](#4-cloud-hosting-deployment-strategies)
5. [Production Logging, Security & Health Probes](#5-production-logging-security--health-probes)
6. [Mobile Android APK & Bundle Generation](#6-mobile-android-apk--bundle-generation)
7. [Device Sideloading, ADB Installation & Testing](#7-device-sideloading-adb-installation--testing)

---

## 1. Backend Production Configuration

### 1.1 Production Environment Variables (`.env.production`)
Create a production `.env` file based on `.env.production.example`:

```env
# Node Environment & Service Port
NODE_ENV=production
PORT=5000

# PostgreSQL Database Connection (with connection pooling and SSL)
DATABASE_URL="postgresql://db_user:SUPER_STRONG_PASSWORD@db-host.internal:5432/insurance_prod?schema=public&sslmode=require&connection_limit=25&pool_timeout=10"
DIRECT_URL="postgresql://db_user:SUPER_STRONG_PASSWORD@db-host.internal:5432/insurance_prod?schema=public&sslmode=require"

# Cryptographic Keys (Generate with: openssl rand -base64 64)
JWT_SECRET=4f8b9e2c6d1a5f8e3b7a9c2d1e0f4a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4d3e2f
JWT_EXPIRES_IN=15m
REFRESH_TOKEN_EXPIRES_DAYS=7
BCRYPT_SALT_ROUNDS=12

# Production Rate Limiting
RATE_LIMIT_WINDOW_MS=900000
RATE_LIMIT_MAX_REQUESTS=150
AUTH_RATE_LIMIT_MAX=10

# CORS Whitelist (Comma-separated exact origins)
CLIENT_ORIGIN="https://assurance.yourdomain.com,https://admin.assurance.yourdomain.com"
```

> [!CAUTION]
> Never commit `.env` or `.env.production` containing real secrets to version control. Use cloud secret managers (AWS Secrets Manager, GCP Secret Manager, Vault, or platform environment dashboards).

---

## 2. Database Connection Pooling & SSL Migration Strategy

### 2.1 Managed PostgreSQL Provisioning
For production workloads, use managed PostgreSQL providers:
- **AWS RDS PostgreSQL** (Multi-AZ with automated backups)
- **Supabase / Neon** (Serverless PostgreSQL with integrated PgBouncer connection pooling)
- **DigitalOcean Managed Databases**

### 2.2 Connection Pooling Parameters
When deploying containerized apps that scale horizontally across multiple instances, append connection pooling parameters to `DATABASE_URL`:
- `connection_limit=25`: Limits max connections per backend instance to prevent exhausting database pool.
- `pool_timeout=10`: Aborts hanging pool checkouts after 10 seconds.
- `sslmode=require`: Enforces encrypted transport layer communication between Node.js and PostgreSQL.

### 2.3 Executing Migrations in CI/CD
Never run `prisma migrate dev` in production. Always run `prisma migrate deploy`, which applies pending SQL migration files without altering the migration history table:

```bash
# In your CI/CD deployment pipeline or container release command:
npx prisma migrate deploy
```

---

## 3. Containerization: Production Dockerfile & Docker Compose

### 3.1 Production Multi-Stage Dockerfile
Place this `Dockerfile` in `insurance-policy-claims-system/backend/Dockerfile`:

```dockerfile
# ====================================================================
# STAGE 1: Build & Prisma Compilation
# ====================================================================
FROM node:20-alpine AS builder

WORKDIR /app

# Install build dependencies
COPY package*.json ./
COPY prisma ./prisma/

RUN npm ci
RUN npx prisma generate

# Copy source code and compile TypeScript
COPY tsconfig.json ./
COPY src ./src/
RUN npm run build

# ====================================================================
# STAGE 2: Minimal Production Runtime
# ====================================================================
FROM node:20-alpine AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV PORT=5000

# Install dumb-init for proper PID 1 signal forwarding
RUN apk add --no-cache dumb-init

# Copy production dependencies only
COPY package*.json ./
RUN npm ci --only=production

# Copy compiled artifacts & prisma client from builder stage
COPY --from=builder /app/node_modules/.prisma ./node_modules/.prisma
COPY --from=builder /app/node_modules/@prisma ./node_modules/@prisma
COPY --from=builder /app/prisma ./prisma
COPY --from=builder /app/dist ./dist

# Create unprivileged application user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
RUN mkdir -p uploads && chown -R appuser:appgroup /app
USER appuser

EXPOSE 5000

HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:5000/health || exit 1

ENTRYPOINT ["/usr/bin/dumb-init", "--"]
CMD ["node", "dist/index.js"]
```

### 3.2 Production Docker Compose (`docker-compose.prod.yml`)

```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    container_name: assurance-backend
    restart: always
    ports:
      - "5000:5000"
    environment:
      - NODE_ENV=production
      - PORT=5000
      - DATABASE_URL=postgresql://insurance_user:SecurePassword2026@postgres:5432/insurance_db?schema=public&sslmode=prefer
      - JWT_SECRET=${JWT_SECRET}
      - CLIENT_ORIGIN=${CLIENT_ORIGIN}
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - backend-uploads:/app/uploads

  postgres:
    image: postgres:16-alpine
    container_name: assurance-postgres
    restart: always
    environment:
      POSTGRES_USER: insurance_user
      POSTGRES_PASSWORD: SecurePassword2026
      POSTGRES_DB: insurance_db
    volumes:
      - postgres-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U insurance_user -d insurance_db"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  backend-uploads:
  postgres-data:
```

---

## 4. Cloud Hosting Deployment Strategies

### Option A: Render.com
1. Create a **PostgreSQL** instance on Render (Select Version 16).
2. Create a new **Web Service**, connect your Git repository, and specify `Root Directory: insurance-policy-claims-system/backend`.
3. Set **Build Command**: `npm ci && npx prisma generate && npm run build`.
4. Set **Pre-Deploy Command**: `npx prisma migrate deploy`.
5. Set **Start Command**: `npm start`.
6. Add environment variables from `.env.production`.
7. Configure Health Check Path: `/health`.

### Option B: Railway.app
1. Click **New Project** -> **Provision PostgreSQL**.
2. Click **New Service** -> **GitHub Repo** (point to repository).
3. Under service settings, set root directory to `insurance-policy-claims-system/backend`.
4. Add reference variable `DATABASE_URL: ${{Postgres.DATABASE_URL}}`.
5. Set custom start command: `npx prisma migrate deploy && node dist/index.js`.

### Option C: AWS (ECS Fargate + RDS)
1. Push Docker image to **AWS ECR**.
2. Provision **AWS RDS PostgreSQL 16** inside a private VPC subnet.
3. Deploy an **ECS Fargate Task Definition** with container port 5000 mapped to an **Application Load Balancer (ALB)** with HTTPS certificate from AWS ACM.
4. Set ALB Target Group health check path to `/health` with 200 OK expectation.

---

## 5. Production Logging, Security & Health Probes

### 5.1 Morgan HTTP Logging
In production (`NODE_ENV=production`), the application automatically activates Apache Combined Log format via Morgan:
```
::ffff:192.168.1.1 - - [20/Nov/2026:17:35:10 +0000] "GET /api/customer/dashboard HTTP/1.1" 200 482 "https://assurance.yourdomain.com" "Mozilla/5.0..."
```
This enables seamless ingestion by Datadog, AWS CloudWatch, Logstash, or Graylog.

### 5.2 Automated Health Checks
Load balancers and Kubernetes kubelet probes should be configured to target:
- **HTTP Path**: `/health` (or `/api/health`)
- **Port**: 5000
- **Success Code**: `200 OK`
- If PostgreSQL connectivity is severed, `/health` automatically responds with `503 Service Unavailable` with database latency metrics, alerting uptime monitors immediately.

---

## 6. Mobile Android APK & Bundle Generation

The mobile application is configured using Expo EAS (Expo Application Services) with dual build profiles defined in `mobile/eas.json`:
- **`preview` profile**: Generates a standalone, installable `.apk` file (for internal QA, stakeholders, and manual device testing).
- **`production` profile**: Generates an optimized Android App Bundle (`.aab`) ready for submission to the Google Play Console.

### 6.1 Prerequisites
Install the EAS Command Line Interface globally:
```bash
npm install -g eas-cli
```

Authenticate with your Expo developer account:
```bash
eas login
```

Verify your project configuration:
```bash
cd insurance-policy-claims-system/mobile
npx expo config
```

### 6.2 Generating Standalone Android APK (`preview` Profile)
Run the EAS build command targeting Android with the `preview` profile:

```bash
eas build --platform android --profile preview
```

**What happens during this build:**
1. EAS packages the verified Metro bundle (911 modules in Hermes bytecode).
2. Generates an Android keystore securely in the cloud (or uses your existing keystore).
3. Compiles the native Android application with Gradle.
4. Outputs a direct HTTPS download link to the compiled `app-preview.apk`.

### 6.3 Generating Google Play Store App Bundle (`production` Profile)
When ready for official store deployment:

```bash
eas build --platform android --profile production
```
This generates the optimized `.aab` file that Google Play uses for Dynamic Delivery.

### 6.4 Local Offline APK Build (Using Local Android SDK & Gradle)
If you prefer building without cloud EAS, build locally on a workstation with Android Studio, JDK 17, and Android SDK:

```bash
cd insurance-policy-claims-system/mobile

# 1. Export the verified Hermes production bundle
npx expo export --platform android

# 2. Generate native Android Gradle project
npx expo prebuild --platform android --no-install

# 3. Compile standalone release APK with Gradle
cd android
./gradlew assembleRelease
```
The compiled APK will be located at:
`mobile/android/app/build/outputs/apk/release/app-release.apk`

---

## 7. Device Sideloading, ADB Installation & Testing

Once the `.apk` file is generated, test it on physical Android hardware or emulators:

### Method 1: Sideloading via ADB (Android Debug Bridge)
1. Enable **Developer Options** and **USB Debugging** on your Android smartphone.
2. Connect your device via USB cable.
3. Verify device connection:
   ```bash
   adb devices
   ```
4. Install the APK:
   ```bash
   adb install -r app-preview.apk
   ```

### Method 2: Direct Device Download
1. Upload `app-preview.apk` to cloud storage (Google Drive, Dropbox) or use the direct download URL provided by EAS CLI.
2. Open the link on the Android mobile browser.
3. Download the file, tap **Open**, and when prompted, allow **Install unknown apps** for your browser.
4. Tap **Install** and launch **Assurance**.

### Method 3: One-Click Testing with Expo Orbit
Install [Expo Orbit](https://expo.dev/orbit) on your macOS or Windows workstation to install builds to physical devices or emulators with a single click directly from your system tray.

### Production Network Verification Checklist:
- [ ] Mobile app connects to the remote production backend URL over HTTPS.
- [ ] Login completes with pre-seeded accounts (`admin@assurance.com`, `agent.smith@assurance.com`, `customer@assurance.com`).
- [ ] Customer dashboard loads real metrics.
- [ ] Document camera capture and gallery uploads work seamlessly with Android permissions.
- [ ] Push/In-app notifications update the badge icon.
