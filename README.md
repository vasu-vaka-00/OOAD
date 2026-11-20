# Assurance: Insurance Policy and Claims Processing System

[![Build Status](https://img.shields.io/badge/Build-Passing-brightgreen.svg)]()
[![Automated Tests](https://img.shields.io/badge/Tests-325%20Passed-brightgreen.svg)]()
[![TypeScript](https://img.shields.io/badge/TypeScript-Strict%205.x-blue.svg)]()
[![Node.js](https://img.shields.io/badge/Node.js-v20%20LTS-green.svg)]()
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-v16%20ACID-blue.svg)]()
[![Expo](https://img.shields.io/badge/Expo-SDK%2052-black.svg)]()
[![Android ID](https://img.shields.io/badge/Android-com.insurance.claims-orange.svg)]()

A complete, enterprise-grade full-stack mobile application and RESTful microservice designed to digitize the complete insurance lifecycle—from policy discovery, underwriting application, and automated issuance to indemnity claim filing, adjuster investigation, evidence verification, and payout settlement.

---

## Workspace Root Runner

This root directory provides convenience workspace commands:

```bash
# Run strict TypeScript compilation check across backend & mobile
npm run typecheck

# Run ESLint quality checks across backend & mobile
npm run lint

# Start backend in development mode
npm run backend:dev

# Start Expo development server for mobile client
npm run mobile:start
```

---

## Detailed System Documentation

All project source code and comprehensive documentation reside in the `insurance-policy-claims-system/` directory:

- [**Master Project README**](./insurance-policy-claims-system/README.md)
- [**Setup & Local Development Guide**](./insurance-policy-claims-system/docs/SETUP_GUIDE.md)
- [**REST API Documentation & Schemas**](./insurance-policy-claims-system/docs/API_DOCUMENTATION.md)
- [**Database Architecture & 15 Prisma Models (ERD)**](./insurance-policy-claims-system/docs/DATABASE_DESIGN.md)
- [**Quality Assurance & Testing Report (325 Tests Passed)**](./insurance-policy-claims-system/docs/TESTING_REPORT.md)
- [**Production Deployment & Mobile APK Build Guide**](./insurance-policy-claims-system/docs/DEPLOYMENT_GUIDE.md)

---

## Pre-Configured Demo Accounts

| Role | Email | Password | Scope & Access Permissions |
|---|---|---|---|
| **Administrator** | `admin@assurance.com` | `AdminPass123!` | System dashboard, application underwriting adjudication, claim payouts, product creation, audit logs |
| **Insurance Agent** | `agent.smith@assurance.com` | `AgentPass123!` | Caseload review, assigned customer management, claim investigations, internal underwriting notes |
| **Customer** | `customer@assurance.com` | `CustomerPass123!` | Policy purchasing, claim filing, evidence document upload, status tracking, notifications |
