# Manazir Ali

**manazir.dev@gmail.com** | **072 780 4951** | [LinkedIn: in/mnzr](https://linkedin.com/in/mnzr) | [GitHub: mnzralee](https://github.com/mnzralee) | [www.manazir.dev](https://www.manazir.dev)

---

## Professional Summary

Full-Stack Software Engineer who single-handedly architected and shipped an enterprise blockchain platform from the ground up — 529,000+ lines of production code across 7 repositories, spanning Hyperledger Fabric smart contracts, 14 TypeScript microservices, React/Next.js applications, and a 4-node Kubernetes cluster deployed across 3 continents. Concurrently serving as AI Solutions Architect for a New Zealand-based technology firm. Proven track record of taking complex distributed systems from zero to production with enterprise-grade security, observability, and reliability.

---

## Experience

### Full-Stack Software Engineer — GX Coin Protocol
*Full-Time | January 2025 – Present | Sri Lanka*

- Solely architected and built an enterprise blockchain fintech platform from scratch — 529,000+ lines of source code, 3,678 commits across 7 repositories, owning the complete stack from smart contracts to global infrastructure

- Designed and implemented Version 3 protocol architecture with 7 Hyperledger Fabric smart contracts in Go (38 chaincode functions) covering identity management, tokenomics, governance, lending, and tax workflows with RBAC, endorsement policies, and Chaincode-as-a-Service deployments

- Built 14 TypeScript/Node.js microservices following Clean Architecture, CQRS, and Event-Driven patterns with transactional outbox, circuit breakers, and exponential backoff ensuring zero data loss across distributed blockchain write operations

- Led monolith decomposition of a 43,936-line service into focused bounded contexts with 121 use cases, 90+ domain error classes, and zero TypeScript compilation errors

- Architected Next.js 15 wallet and admin dashboards with TypeScript strict mode, BFF security patterns, real-time blockchain state sync via TanStack Query, schema validation with Zod, and biometric KYC using MediaPipe Face Mesh for real-time face liveness detection

- Engineered PostgreSQL HA cluster (3-replica) with 150+ tables, 470+ performance-tuned indexes, PgBouncer connection pooling, and Redis Sentinel for resilient caching with automatic failover

- Deployed and managed a 4-node Kubernetes (K3s) cluster spanning 3 continents with Cloudflare GeoDNS, zero-trust networking, automated SSL, and enterprise-grade infrastructure built at zero cost

- Led platform security hardening: zero-trust NetworkPolicies, 3-tier rate limiting, CORS protection, and endpoint-level RBAC on 12+ sensitive endpoints — reduced overall CVSS risk exposure by 60%. Responded to live production security incidents and resolved critical Out-of-Memory server crashes

- Implemented enterprise observability stack (Prometheus, Grafana, Loki) with 63 dashboard panels, 15 alert rules, and centralized structured logging achieving sub-2-minute mean-time-to-detect for production incidents

- Created 260,000+ lines of technical documentation including 6 educational lectures, architecture decision records, operational runbooks, and comprehensive API specifications

### AI Solutions Architect — Tectonix AI
*Contract (Remote) | November 2025 – Present | Auckland, New Zealand*

- Driving end-to-end AI architecture vision and technical foundations for a New Zealand-based firm specializing in advanced AI solutions and next-generation software architecture

- Partnering with executives on portfolio strategy, aligning go-to-market offerings with core AI competencies and long-term product roadmap

- Designing reusable technical blueprints bridging cutting-edge AI/ML stacks with pragmatic business use-cases for clients across regions

- Coordinating cross-border delivery teams on solution handoffs, documentation, and governance to meet enterprise expectations

### Web Developer
*Freelance | April 2024 – Present | Sri Lanka*

- Delivering custom WordPress solutions (ThemeCo Cornerstone, Advanced Custom Fields, bespoke components) for educational institutions, charity organizations, and business clients

- Integrating donation features and payment gateway flows, implementing structured content with custom post types for non-technical stakeholder management

- Optimizing site performance through image compression, caching strategy, accessibility-first semantic markup, and SEO schema implementation

### Course Instructor — NEOGRAENIUM
*Part-Time | October 2023 – July 2024 | Sri Lanka*

- Designed and delivered hands-on curricula covering Arduino, Microbit, and Lego Robotics, fostering foundational programming and electronics thinking for students

- Facilitated project-based learning experiences emphasizing problem decomposition, iterative prototyping, and creative problem-solving

---

## Projects

### GX Coin Marketing Website
*Production Launch | September 2025*

- Designed and shipped the public-facing marketing website for the GX Coin Protocol using Next.js App Router, TypeScript, Tailwind CSS, and MDX for content management

- Implemented validator onboarding flows, tokenomics education pages, and audience-segmented conversion funnels with WCAG AA accessibility compliance and Core Web Vitals optimization

### DineEase — Smart Restaurant Platform
*Software Development Group Project | October 2024 – March 2025*

- Developed a full-stack restaurant reservation and meal pre-ordering platform using Next.js and Firebase with real-time notifications, user authentication, and Firestore security rules

- Integrated AR Meal Preview (WebXR/Three.js prototype) and ML-driven personalized recommendations, enhancing customer engagement through immersive dining experiences

### EcomSL — Full-Stack E-Commerce Platform
*Personal Project | In Progress*

- Architected a production-grade e-commerce platform as a Turborepo monorepo with 3 Next.js 15/Express apps and 6 shared packages (@repo/auth, @repo/db, @repo/types, @repo/logger, eslint-config, typescript-config), fully containerized via Docker Compose with PostgreSQL 16

- Built a custom authentication system from scratch replacing Clerk — JWT access/refresh token pairs with HS256 signing (jose), SHA-256 token hashing, automatic session rotation with race-condition grace windows, token-family-based theft detection and revocation, escalating account lockout (1→5→15→30→60 min), and a comprehensive audit trail logging 18 event types

- Designed a 12-model PostgreSQL schema via Prisma ORM covering products with size/color variants, customers with email verification flows, order lifecycle with a 7-state finite state machine (pending→confirmed→processing→shipped→delivered→cancelled→refunded), a headless CMS (hero slides, homepage sections, dynamic pages), and a full RBAC admin system with 3 roles across 10 resources

- Implemented PayHere payment gateway integration with server-side HMAC hash generation, idempotent webhook processing for payment confirmations/chargebacks/failures, and transactional email notifications (welcome, order confirmation, status updates, password reset) via Nodemailer

- Developed the customer storefront with server-side data fetching (ISR with 60s revalidation), Zustand-persisted cart with variant-aware deduplication, multi-step checkout flow (cart→shipping→payment), React Hook Form with Zod schema validation, and a responsive mobile-first UI; plus an admin dashboard with ShadCN UI/Radix components, TanStack React Table with pagination, Recharts analytics, dark mode theming, and role-gated content management

### Meat Shop POS — Business Management & Accounting System
*Freelance Project | In Progress*

- Built a production POS and financial management system for a meat retail business using Next.js 16 (App Router), React 19, TypeScript, Supabase (PostgreSQL + Auth), Tailwind CSS 4, and shadcn/ui — deployed on Vercel with 16 database tables, 14 RESTful API endpoints, and 40+ React components

- Implemented double-entry bookkeeping with a cash ledger, accounts receivable/payable tracking, end-of-day cash reconciliation with physical count vs. calculated balance discrepancy detection, and automated P&L calculations (revenue, COGS, gross profit margin, operating expenses, net profit)

- Engineered a complete sales lifecycle supporting cash and credit transactions with partial upfront payments, variant-aware inventory tracking (kg/piece units with fractional quantities), supplier stock receipt management with per-unit cost capture, and transaction locking after daily closure to ensure data immutability

- Developed a real-time staff dashboard with 11 animated metric cards (Framer Motion), an admin dashboard with transaction approval workflows, Recharts analytics, full audit logging of all CRUD operations, Zod schema validation across 15+ forms, and automated Excel daily report generation via xlsx

### Real-Time Event Ticketing System
*OOP Coursework | October 2024 – November 2024*

- Engineered a multithreaded event ticketing system using Angular, Spring Boot, and MySQL with advanced producer-consumer concurrency (synchronized monitor, wait/notify backpressure, starvation mitigation)

- Implemented real-time WebSocket log streaming and a live dashboard for concurrent ticket management under high-demand conditions

*More projects at [github.com/mnzralee](https://github.com/mnzralee) and [manazir.dev](https://www.manazir.dev)*

---

## Technical Skills

| Category | Technologies |
|----------|-------------|
| **Languages** | TypeScript, JavaScript, Go, Java, Python, SQL, HTML, CSS, Bash |
| **Frontend** | React, Next.js 15, Angular, Tailwind CSS, ShadCN UI, TanStack Query, Zod |
| **Backend** | Node.js, Express.js, NestJS, Spring Boot, Prisma ORM, REST APIs, gRPC |
| **Databases** | PostgreSQL, Redis, MySQL, Firebase, PgBouncer |
| **DevOps & Infra** | Kubernetes (K3s), Docker, Prometheus, Grafana, Loki, GitHub Actions, Nginx, Cloudflare |
| **Architecture** | Microservices, CQRS, Event-Driven, Clean Architecture, Domain-Driven Design, BFF Pattern |
| **Blockchain** | Hyperledger Fabric, Smart Contracts (Chaincode/Go), CCAAS, Distributed Ledger Technology |
| **Security** | Zero-Trust Networking, JWT Authentication, RBAC, CORS, TLS/SSL, API Security |
| **Testing** | Vitest, Testcontainers, Pact (Contract Testing), JUnit, Integration Testing |
| **AI & Automation** | Claude Code, Multi-Agent Orchestration, Prompt Engineering, AI-Augmented Development |
| **Tools** | Git, VS Code, IntelliJ, Figma, Agile/Scrum, WordPress |

---

## Education

**BSc (Hons) in Computer Science** | *Expected Graduation: 2027*
University of Westminster, UK | Informatics Institute of Technology (IIT), Sri Lanka

- **Relevant Coursework:** Object-Oriented Programming, Full-Stack Web Development, Data Structures & Algorithms, Database Systems, Multithreading & Concurrency

**GCE Advanced Level — Physical Science** | *2022*
Isipathana College

- **Results:** ICT — A, Combined Mathematics — C, Physics — S, English — A

---

## Certifications

- Go (Golang) — LinkedIn Learning
- Completed technical certifications through Moratuwa University, LinkedIn Learning, and HackerRank

---

## Activities & Leadership

**Hejaaz Int'l MUN Club** — *Treasurer* | August 2017 – August 2019
- Managed club finances, including budgeting, fundraising, event hosting, and financial reporting

**Hejaaz Int'l Sports Meet** — *Captain* | August 2018 – August 2019
- Led 100+ students to victory, coordinating training and daily practices

---

## Languages

English, Sinhala, Tamil

---

## Referees

**Imani Randhuli**
MBA (PIM-USJ Reading), BA (Hons) TESL (Kelaniya)
Lecturer, Informatics Institute of Technology
imani.r@iit.ac.lk | 076 149 0885

**Kushan Bhareti**
Visiting Lecturer, Informatics Institute of Technology
Director, Inforwaves (Pvt) Ltd.
kushan.b@iit.ac.lk | 076 781 4281
