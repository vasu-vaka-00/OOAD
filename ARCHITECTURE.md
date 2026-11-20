# Insurance Policy and Claims Processing System - Architecture Specification

## 1. System Overview
The **Insurance Policy and Claims Processing System** is an enterprise-grade mobile application and backend designed to streamline the management of insurance policies and automate claim submissions, reviews, and adjudications.

## 2. Architecture Layers

```
+-----------------------------------------------------------+
|                      Mobile Client                        |
|               (React Native + Expo + TypeScript)          |
|  - Auth State / Secure Store                              |
|  - React Navigation (Stack / Tabs)                        |
|  - Axios API Client + Interceptors                        |
+-----------------------------+-----------------------------+
                              | HTTPS (REST + JWT)
+-----------------------------v-----------------------------+
|                      Backend API                          |
|             (Node.js + Express.js + TypeScript)           |
|  - Middlewares (Helmet, CORS, Morgan, Auth Guards)        |
|  - Controllers & Input Validation (Zod)                   |
|  - Service Layer (Business Logic & Workflow Engine)       |
|  - Data Access Layer (Prisma ORM)                         |
+-----------------------------+-----------------------------+
                              | TCP / SQL
+-----------------------------v-----------------------------+
|                      Database                             |
|                    (PostgreSQL)                           |
|  - Users & Roles (POLICYHOLDER, AGENT, ADJUSTER, ADMIN)   |
|  - Policies (Auto, Health, Home, Life)                    |
|  - Claims & Claim History Tracking                        |
|  - Claim Documents Metadata                               |
+-----------------------------------------------------------+
```

## 3. Technology Stack Justification
- **Frontend (Mobile)**:
  - **React Native with Expo**: Cross-platform deployment (iOS and Android), rapid feedback loop, fast refresh, and managed native module ecosystem.
  - **TypeScript**: End-to-end type safety, preventing runtime exceptions across complex insurance models.
  - **React Navigation**: Flexible screen transitions, native stack navigation, and role-based route guard architecture.
  - **Axios**: Standardized HTTP client with request/response interceptors for automatic JWT authentication header injection and token expiration handling.

- **Backend**:
  - **Node.js + Express**: High-throughput asynchronous event-driven runtime ideal for RESTful API services.
  - **TypeScript**: Strict type boundaries between controllers, DTOs, and services.
  - **Prisma ORM**: Declarative schema migrations, type-safe queries, zero string-based SQL queries, automated connection pooling.
  - **PostgreSQL**: ACID compliance for insurance financial data, relational integrity between policies, claims, and document attachments.
  - **JWT (JSON Web Tokens)**: Stateless authentication allowing horizontal scaling of API nodes.

## 4. Role-Based Access Control (RBAC)
- `POLICYHOLDER`: View own policies, submit claims, upload supporting claim documents, track claim status.
- `AGENT`: Issue quotes, initiate policy proposals, view assigned clients.
- `CLAIMS_ADJUSTER`: Review submitted claims, inspect documents, approve/reject claims, record review notes and approved amounts.
- `ADMIN`: Full system audit, user management, policy catalogue management.

