# Project Setup and Local Development Guide

## Prerequisites
- **Node.js**: v18 or higher (v20+ recommended)
- **npm**: v9 or higher
- **PostgreSQL**: v14 or higher (local or managed instance)
- **Expo Go App**: Installed on physical iOS/Android device OR iOS Simulator / Android Emulator

---

## 1. Backend Setup

1. **Navigate to the backend directory:**
   ```bash
   cd insurance-policy-claims-system/backend
   ```

2. **Configure Environment Variables:**
   ```bash
   cp .env.example .env
   ```
   *Edit `.env` to configure your PostgreSQL connection string and JWT secret.*

3. **Install Dependencies:**
   ```bash
   npm install
   ```

4. **Initialize Prisma Client & Database Migrations:**
   ```bash
   npx prisma generate
   # When PostgreSQL is active:
   npx prisma migrate dev --name init
   ```

5. **Start Development Server:**
   ```bash
   npm run dev
   ```
   *Backend will start on `http://localhost:5000` (or configured PORT).*

---

## 2. Mobile Client Setup

1. **Navigate to the mobile directory:**
   ```bash
   cd insurance-policy-claims-system/mobile
   ```

2. **Configure Environment Variables:**
   ```bash
   cp .env.example .env
   ```
   *Set `EXPO_PUBLIC_API_URL` to your backend machine IP address (e.g. `http://192.168.1.X:5000`) for device testing, or `http://localhost:5000` for simulator.*

3. **Install Dependencies:**
   ```bash
   npm install
   ```

4. **Start Expo Development Server:**
   ```bash
   npm run start
   ```
   *Scan the QR code with Expo Go (Android) or Camera app (iOS) to launch.*

---

## 3. Verification & Code Quality Commands

### Backend:
```bash
npm run typecheck    # Verify TypeScript types
npm run lint         # Check ESLint rules
npm run build        # Compile to dist/
```

### Mobile:
```bash
npm run typecheck    # Verify TypeScript types
npm run lint         # Check ESLint rules
```

