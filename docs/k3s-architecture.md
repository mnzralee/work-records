# GX Protocol - Complete Kubernetes Architecture & Implementation Report

## Executive Summary

The GX Protocol is a production-grade blockchain-based monetary system deployed on a **4-node K3s Kubernetes cluster** across 3 continents. The architecture implements a three-tier system: Frontend (Next.js), Backend (Node.js microservices with CQRS), and Blockchain (Hyperledger Fabric 2.5).

---

## Infrastructure Overview

### Cluster Topology

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        GX PROTOCOL K3s CLUSTER                                  │
│                        (4 Nodes, 3 Continents)                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐     │
│  │   VPS1 (Malaysia)   │  │    VPS2 (Malaysia)  │  │  VPS3 (Germany)     │     │
│  │   srv1089618        │  │    srv1117946       │  │  srv1092158         │     │
│  │   72.60.210.201     │  │    72.61.116.210    │  │  72.61.81.3         │     │
│  │   ─────────────     │  │    ─────────────    │  │  ─────────────      │     │
│  │   Control-Plane     │  │    Control-Plane    │  │  Control-Plane      │     │
│  │   MainNet Workloads │  │    MainNet Workloads│  │  MainNet Workloads  │     │
│  │   16 CPU / 64GB RAM │  │    16 CPU / 64GB RAM│  │  16 CPU / 64GB RAM  │     │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────┘     │
│                                                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    VPS4 (USA - Phoenix)                                 │   │
│  │                    srv1089624 | 217.196.51.190                          │   │
│  │                    ─────────────────────────                            │   │
│  │                    Control-Plane (Tainted: environment=nonprod)         │   │
│  │                    DevNet + TestNet Workloads                           │   │
│  │                    16 CPU / 64GB RAM                                    │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                 │
│  Total: 64 vCPU | 256GB RAM | 4TB Storage | AlmaLinux 10.0                     │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Why All Control-Plane Nodes?

K3s differs from standard Kubernetes:
- **K3s doesn't taint control-plane nodes by default** - workloads run on all nodes
- **Resource efficient** for small-to-medium clusters (4-20 nodes)
- **VPS4 has a custom taint** (`environment=nonprod:NoSchedule`) for environment isolation
- This is the **intended K3s design pattern** for edge/small deployments

---

## Complete System Architecture Diagram

```
                                    ┌──────────────────────────────────────┐
                                    │           CLOUDFLARE                 │
                                    │      GeoDNS + DDoS Protection        │
                                    └──────────────────┬───────────────────┘
                                                       │
                    ┌──────────────────────────────────┼──────────────────────────────────┐
                    │                                  │                                  │
                    ▼                                  ▼                                  ▼
        ┌───────────────────────┐      ┌───────────────────────┐      ┌───────────────────────┐
        │  admin.gxcoin.money   │      │  devnet.gxcoin.money  │      │   api.gxcoin.money    │
        │    (Admin Portal)     │      │   (Wallet Frontend)   │      │    (Backend API)      │
        └───────────┬───────────┘      └───────────┬───────────┘      └───────────┬───────────┘
                    │                              │                              │
                    └──────────────────────────────┼──────────────────────────────┘
                                                   │
                                                   ▼
┌──────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                    KUBERNETES INGRESS LAYER                                      │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐    │
│  │                              NGINX Ingress Controller                                    │    │
│  │    Routes:                                                                               │    │
│  │    /api/v1/auth, /api/v1/users     → svc-identity:3001                                  │    │
│  │    /api/v1/wallets, /api/v1/transactions → svc-tokenomics:3003                          │    │
│  │    /api/v1/organizations           → svc-organization:3004                              │    │
│  │    /api/v1/loans                   → svc-loanpool:3005                                  │    │
│  │    /api/v1/proposals, /api/v1/votes → svc-governance:3006                               │    │
│  │    /api/v1/admin                   → svc-admin:3002                                     │    │
│  │    /api/v1/government              → svc-government:3009                                │    │
│  │    /api/v1/fees                    → svc-tax:3007                                       │    │
│  │    /socket.io                      → svc-messaging:3008                                 │    │
│  │    /                               → gx-wallet-frontend:3000 | gx-admin-frontend:3000   │    │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                   │
          ┌────────────────────────────────────────┼────────────────────────────────────────┐
          │                                        │                                        │
          ▼                                        ▼                                        ▼
┌──────────────────────┐              ┌──────────────────────┐              ┌──────────────────────┐
│   FRONTEND LAYER     │              │    BACKEND LAYER     │              │  BLOCKCHAIN LAYER    │
│   (Namespace:        │              │    (Namespace:       │              │  (Namespace:         │
│    backend-devnet)   │              │     backend-mainnet) │              │   fabric-devnet)     │
└──────────────────────┘              └──────────────────────┘              └──────────────────────┘
```

---

## Detailed Layer Architecture

### Layer 1: Frontend Applications

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           FRONTEND LAYER                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                 │
│  ┌─────────────────────────────┐    ┌─────────────────────────────┐            │
│  │   GX WALLET FRONTEND        │    │   GX ADMIN FRONTEND         │            │
│  │   ─────────────────────     │    │   ─────────────────────     │            │
│  │   Framework: Next.js 15     │    │   Framework: Next.js 16     │            │
│  │   React 18 + TypeScript     │    │   React + Tailwind v4       │            │
│  │   Port: 3000                │    │   Port: 3000                │            │
│  │   Replicas: 1               │    │   Replicas: 1               │            │
│  │   Node: VPS4 (DevNet)       │    │   Node: VPS4 (DevNet)       │            │
│  │                             │    │                             │            │
│  │   Features:                 │    │   Features:                 │            │
│  │   • User registration       │    │   • User management         │            │
│  │   • Wallet dashboard        │    │   • Treasury management     │            │
│  │   • Send/Receive            │    │   • KYC review queue        │            │
│  │   • Transaction history     │    │   • Governance              │            │
│  │   • Beneficiaries           │    │   • Audit logs              │            │
│  │   • Government portal       │    │   • RBAC configuration      │            │
│  └─────────────────────────────┘    └─────────────────────────────┘            │
│                                                                                 │
│  Images:                                                                        │
│  • 10.43.75.195:5000/gx-wallet-frontend:1.2.0                                  │
│  • 10.43.75.195:5000/gx-admin-frontend:v1.0.2                                  │
│                                                                                 │
│  Resource Allocation (per pod):                                                 │
│  • CPU: 100m request / 500m limit                                              │
│  • Memory: 256Mi request / 512Mi limit                                         │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

### Layer 2: Backend Microservices (CQRS Architecture)

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  BACKEND LAYER (CQRS Pattern)                                   │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                 │
│  ┌───────────────────────────────── API SERVICES (HTTP) ─────────────────────────────────────┐ │
│  │                                                                                           │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐        │ │
│  │  │svc-identity │ │svc-tokenomics│ │svc-organiz. │ │svc-loanpool │ │svc-governance│        │ │
│  │  │   :3001     │ │    :3003    │ │    :3004    │ │    :3005    │ │    :3006    │        │ │
│  │  │ (3 replicas)│ │ (3 replicas)│ │ (3 replicas)│ │ (3 replicas)│ │ (3 replicas)│        │ │
│  │  │             │ │             │ │             │ │             │ │             │        │ │
│  │  │• Auth/Login │ │• Transfers  │ │• Multi-sig  │ │• Loans      │ │• Proposals  │        │ │
│  │  │• Registration│ │• Balances  │ │• Business   │ │• Collateral │ │• Voting     │        │ │
│  │  │• Profiles   │ │• History    │ │• Approvals  │ │• Repayments │ │• Results    │        │ │
│  │  │• KYC        │ │• Treasury   │ │             │ │             │ │             │        │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘        │ │
│  │                                                                                           │ │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                                        │ │
│  │  │  svc-admin  │ │  svc-tax    │ │svc-government│                                        │ │
│  │  │    :3002    │ │    :3007    │ │    :3009    │                                        │ │
│  │  │ (3 replicas)│ │ (3 replicas)│ │ (3 replicas)│                                        │ │
│  │  │             │ │             │ │             │                                        │ │
│  │  │• System ops │ │• Fee calc   │ │• Treasury   │                                        │ │
│  │  │• User mgmt  │ │• Hoarding   │ │• Hierarchy  │                                        │ │
│  │  │• Audit      │ │  tax        │ │• Approvals  │                                        │ │
│  │  └─────────────┘ └─────────────┘ └─────────────┘                                        │ │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                 │
│  ┌───────────────────────────── WORKER PROCESSES ────────────────────────────────────────────┐ │
│  │                                                                                           │ │
│  │  ┌─────────────────────────────────┐    ┌─────────────────────────────────┐              │ │
│  │  │      OUTBOX SUBMITTER           │    │         PROJECTOR               │              │ │
│  │  │      ─────────────────          │    │         ─────────               │              │ │
│  │  │      (2 replicas)               │    │         (1 replica)             │              │ │
│  │  │                                 │    │                                 │              │ │
│  │  │  [PostgreSQL] ──────────────────┼────┼──────────────► [Fabric]         │              │ │
│  │  │  outbox_commands                │    │                                 │              │ │
│  │  │       │                         │    │  [Fabric Events] ───────────────┼─►[PostgreSQL]│ │
│  │  │       ▼                         │    │                                 │  read_models │ │
│  │  │  • Poll for pending commands    │    │  • Listen to chaincode events   │              │ │
│  │  │  • Submit to blockchain         │    │  • Update read models           │              │ │
│  │  │  • FOR UPDATE SKIP LOCKED       │    │  • Checkpoint recovery          │              │ │
│  │  │  • Circuit breaker pattern      │    │  • Ordered processing           │              │ │
│  │  │  • Retry with backoff           │    │                                 │              │ │
│  │  └─────────────────────────────────┘    └─────────────────────────────────┘              │ │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                 │
│  ┌───────────────────────────── DATA LAYER ──────────────────────────────────────────────────┐ │
│  │                                                                                           │ │
│  │  ┌─────────────────────────┐         ┌─────────────────────────┐                         │ │
│  │  │     POSTGRESQL 15       │         │       REDIS 7           │                         │ │
│  │  │     ─────────────       │         │       ───────           │                         │ │
│  │  │     (3-node cluster)    │         │     (3-node cluster)    │                         │ │
│  │  │                         │         │                         │                         │ │
│  │  │  • Database: gx_protocol│         │  • Session cache        │                         │ │
│  │  │  • Storage: 100Gi       │         │  • Rate limiting        │                         │ │
│  │  │  • Port: 5432           │         │  • Pub/Sub              │                         │ │
│  │  │  • User: gx_admin       │         │  • Storage: 20Gi        │                         │ │
│  │  │                         │         │  • Port: 6379           │                         │ │
│  │  └─────────────────────────┘         └─────────────────────────┘                         │ │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                 │
│  Total API Pods: 24 (8 services × 3 replicas)                                                  │
│  Total Worker Pods: 3 (2 outbox + 1 projector)                                                 │
│  Total DB Pods: 6 (3 postgres + 3 redis)                                                       │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

### Layer 3: Hyperledger Fabric Blockchain

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                           HYPERLEDGER FABRIC 2.5 NETWORK                                        │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                 │
│  ┌───────────────────────────── ORDERING SERVICE (Raft) ─────────────────────────────────────┐ │
│  │                                                                                           │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                        │ │
│  │  │orderer0  │ │orderer1  │ │orderer2  │ │orderer3  │ │orderer4  │                        │ │
│  │  │  :7050   │ │  :7050   │ │  :7050   │ │  :7050   │ │  :7050   │                        │ │
│  │  │  :7053   │ │  :7053   │ │  :7053   │ │  :7053   │ │  :7053   │                        │ │
│  │  │  (admin) │ │  (admin) │ │  (admin) │ │  (admin) │ │  (admin) │                        │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘                        │ │
│  │                                                                                           │ │
│  │  Consensus: etcdraft | Fault Tolerance: F=2 (can lose 2 orderers)                        │ │
│  │  MSP: OrdererMSP | Channel: gxchannel-devnet                                             │ │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                 │
│  ┌───────────────────────────── PEER NETWORK ────────────────────────────────────────────────┐ │
│  │                                                                                           │ │
│  │           ORGANIZATION 1 (Org1MSP)              ORGANIZATION 2 (Org2MSP)                 │ │
│  │  ┌─────────────────────────────────┐   ┌─────────────────────────────────┐               │ │
│  │  │                                 │   │                                 │               │ │
│  │  │  ┌───────────┐  ┌───────────┐  │   │  ┌───────────┐  ┌───────────┐  │               │ │
│  │  │  │ peer0-org1│  │ peer1-org1│  │   │  │ peer0-org2│  │ peer1-org2│  │               │ │
│  │  │  │  :7051    │  │  :7051    │  │   │  │  :7051    │  │  :7051    │  │               │ │
│  │  │  │  :7052    │  │  :7052    │  │   │  │  :7052    │  │  :7052    │  │               │ │
│  │  │  │ (anchor)  │  │           │  │   │  │ (anchor)  │  │           │  │               │ │
│  │  │  └─────┬─────┘  └─────┬─────┘  │   │  └─────┬─────┘  └─────┬─────┘  │               │ │
│  │  │        │              │        │   │        │              │        │               │ │
│  │  │        ▼              ▼        │   │        ▼              ▼        │               │ │
│  │  │  ┌───────────┐  ┌───────────┐  │   │  ┌───────────┐  ┌───────────┐  │               │ │
│  │  │  │ CouchDB   │  │ CouchDB   │  │   │  │ CouchDB   │  │ CouchDB   │  │               │ │
│  │  │  │  :5984    │  │  :5984    │  │   │  │  :5984    │  │  :5984    │  │               │ │
│  │  │  │  10Gi     │  │  10Gi     │  │   │  │  10Gi     │  │  10Gi     │  │               │ │
│  │  │  └───────────┘  └───────────┘  │   │  └───────────┘  └───────────┘  │               │ │
│  │  │                                 │   │                                 │               │ │
│  │  └─────────────────────────────────┘   └─────────────────────────────────┘               │ │
│  │                                                                                           │ │
│  │  Gossip Protocol: Cross-org discovery enabled                                            │ │
│  │  Endorsement Policy: AND('Org1MSP.peer', 'Org2MSP.peer')                                │ │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                 │
│  ┌───────────────────────────── CERTIFICATE AUTHORITIES ─────────────────────────────────────┐ │
│  │                                                                                           │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                        │ │
│  │  │ ca-root  │ │ ca-tls   │ │ca-orderer│ │ ca-org1  │ │ ca-org2  │                        │ │
│  │  │  :9051   │ │  :9052   │ │  :9053   │ │  :9054   │ │  :9055   │                        │ │
│  │  │          │ │          │ │          │ │          │ │          │                        │ │
│  │  │  Root CA │ │  TLS CA  │ │ Orderer  │ │   Org1   │ │   Org2   │                        │ │
│  │  │          │ │          │ │  Org CA  │ │ Ident CA │ │ Ident CA │                        │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘                        │ │
│  │                                                                                           │ │
│  │  Backend: PostgreSQL (ca_root, ca_tls, ca_orderer, ca_org1, ca_org2 databases)           │ │
│  │  Image: hyperledger/fabric-ca:1.5.7                                                      │ │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                 │
│  ┌───────────────────────────── CHAINCODE (Smart Contracts) ─────────────────────────────────┐ │
│  │                                                                                           │ │
│  │  ┌─────────────────────────────────────────────────────────────────────────────────────┐ │ │
│  │  │                          gxtv3 CHAINCODE (v1.230)                                   │ │ │
│  │  │                          ─────────────────────────                                  │ │ │
│  │  │                                                                                     │ │ │
│  │  │  Language: Go | External Chaincode-as-a-Service (CCaaS) | Port: 7052               │ │ │
│  │  │                                                                                     │ │ │
│  │  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                   │ │ │
│  │  │  │AdminContract│ │IdentityCont.│ │TokenomicsCo.│ │Organization │                   │ │ │
│  │  │  │ (14 funcs)  │ │ (11 funcs)  │ │ (15 funcs)  │ │  (9 funcs)  │                   │ │ │
│  │  │  │             │ │             │ │             │ │             │                   │ │ │
│  │  │  │• Bootstrap  │ │• CreateUser │ │• Genesis    │ │• CreateOrg  │                   │ │ │
│  │  │  │• Pause/     │ │• GetProfile │ │• Transfer   │ │• AddMember  │                   │ │ │
│  │  │  │  Resume     │ │• UpdateTrust│ │• Balance    │ │• Approve    │                   │ │ │
│  │  │  │• Countries  │ │• Recovery   │ │• History    │ │• Endorse    │                   │ │ │
│  │  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘                   │ │ │
│  │  │                                                                                     │ │ │
│  │  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                                   │ │ │
│  │  │  │LoanPoolCont.│ │GovernanceCo.│ │TaxAndFeeCon.│                                   │ │ │
│  │  │  │  (5 funcs)  │ │  (6 funcs)  │ │  (3 funcs)  │                                   │ │ │
│  │  │  │             │ │             │ │             │                                   │ │ │
│  │  │  │• ApplyLoan  │ │• CreateProp │ │• CalcFee    │                                   │ │ │
│  │  │  │• Repay      │ │• Vote       │ │• HoardingTax│                                   │ │ │
│  │  │  │• Liquidate  │ │• Execute    │ │• Distribute │                                   │ │ │
│  │  │  └─────────────┘ └─────────────┘ └─────────────┘                                   │ │ │
│  │  │                                                                                     │ │ │
│  │  │  Total: 7 Contracts | 63 Functions | Max Supply: 1.25 Trillion GXC                 │ │ │
│  │  └─────────────────────────────────────────────────────────────────────────────────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────────────────────────┘ │
│                                                                                                 │
│  Total Fabric Pods: 18 (5 orderers + 4 peers + 4 CouchDB + 5 CAs + 1 chaincode)               │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Namespace & Environment Isolation

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                               KUBERNETES NAMESPACES                                             │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────────────────────────┐│
│  │                    VPS4 (217.196.51.190) - DevNet/TestNet Only                             ││
│  │                    Taint: environment=nonprod:NoSchedule                                   ││
│  │                                                                                            ││
│  │  ┌───────────────────────────┐    ┌───────────────────────────┐                           ││
│  │  │     backend-devnet        │    │     backend-testnet       │                           ││
│  │  │     ──────────────        │    │     ───────────────       │                           ││
│  │  │  • gx-wallet-frontend     │    │  • gx-wallet-frontend     │                           ││
│  │  │  • gx-admin-frontend      │    │  • svc-* (all services)   │                           ││
│  │  │  • svc-* (all services)   │    │  • postgres, redis        │                           ││
│  │  │  • postgres, redis        │    │                           │                           ││
│  │  └───────────────────────────┘    └───────────────────────────┘                           ││
│  │                                                                                            ││
│  │  ┌───────────────────────────┐    ┌───────────────────────────┐                           ││
│  │  │     fabric-devnet         │    │     fabric-testnet        │                           ││
│  │  │     ─────────────         │    │     ──────────────        │                           ││
│  │  │  • 3 orderers             │    │  • 3 orderers             │                           ││
│  │  │  • 4 peers (2 per org)    │    │  • 2 peers (1 per org)    │                           ││
│  │  │  • 4 CouchDB instances    │    │  • 2 CouchDB instances    │                           ││
│  │  │  • 5 Fabric CAs           │    │  • 5 Fabric CAs           │                           ││
│  │  │  • gxtv3 chaincode        │    │  • gxtv3 chaincode        │                           ││
│  │  │  • PostgreSQL (CA backend)│    │  • PostgreSQL (CA backend)│                           ││
│  │  └───────────────────────────┘    └───────────────────────────┘                           ││
│  └────────────────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────────────────────────┐│
│  │        VPS1/VPS2/VPS3 (72.60.210.201, 72.61.116.210, 72.61.81.3) - MainNet Only           ││
│  │                                                                                            ││
│  │  ┌───────────────────────────┐    ┌───────────────────────────┐                           ││
│  │  │     backend-mainnet       │    │     fabric-mainnet        │                           ││
│  │  │     ───────────────       │    │     ──────────────        │                           ││
│  │  │  • svc-* (all services)   │    │  • 5 orderers (Raft)      │                           ││
│  │  │  • postgres (3 replicas)  │    │  • 2 peers (1 per org)    │                           ││
│  │  │  • redis (3 replicas)     │    │  • 2 CouchDB instances    │                           ││
│  │  │  • outbox-submitter (2)   │    │  • 5 Fabric CAs           │                           ││
│  │  │  • projector (1)          │    │  • gxtv3 chaincode        │                           ││
│  │  │  • HPA enabled            │    │  • PostgreSQL (CA backend)│                           ││
│  │  └───────────────────────────┘    └───────────────────────────┘                           ││
│  └────────────────────────────────────────────────────────────────────────────────────────────┘│
│                                                                                                 │
│  ┌────────────────────────────────────────────────────────────────────────────────────────────┐│
│  │                        monitoring (Cross-environment)                                      ││
│  │                        ───────────────────────────────                                     ││
│  │  • Prometheus (50Gi storage)       • Grafana (NodePort 30300)                             ││
│  │  • Loki (log aggregation)          • AlertManager                                         ││
│  │  • Promtail (DaemonSet)            • Node Exporter (DaemonSet)                            ││
│  │  • CouchDB Exporter                • PostgreSQL Exporter                                  ││
│  │  • Kube-state-metrics                                                                     ││
│  └────────────────────────────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                              CQRS DATA FLOW                                                     │
├─────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                 │
│                           WRITE PATH (Commands)                                                 │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                                         │   │
│  │  [User]  ───►  [Frontend]  ───►  [API Service]  ───►  [PostgreSQL]  ───►  [Outbox]     │   │
│  │                                       │                outbox_commands                  │   │
│  │                                       │                     │                           │   │
│  │                                       │                     ▼                           │   │
│  │                                       │            [Outbox Submitter]                   │   │
│  │                                       │                     │                           │   │
│  │                                       │                     ▼                           │   │
│  │                                       │         [Hyperledger Fabric]                    │   │
│  │                                       │          (gxtv3 chaincode)                      │   │
│  │                                       │                     │                           │   │
│  │                                       │            Returns: 202 Accepted                │   │
│  │                                       ◄──────────────────────                           │   │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                 │
│                           READ PATH (Queries + Event Projection)                                │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │                                                                                         │   │
│  │  [Hyperledger Fabric]  ───►  [Chaincode Events]  ───►  [Projector]  ───►  [PostgreSQL] │   │
│  │   (block committed)              (block listener)                          read_models  │   │
│  │                                                                                │        │   │
│  │                                                                                ▼        │   │
│  │  [User]  ◄───  [Frontend]  ◄───  [API Service]  ◄─────────────────────────────┘        │   │
│  │                                   (queries read models)                                 │   │
│  │                                                                                         │   │
│  │  Latency: <100ms from blockchain commit to read model update                           │   │
│  └─────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                 │
└─────────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## Resource Summary

### Kubernetes Resources by Type

| Resource Type | DevNet | TestNet | MainNet | Total |
|---------------|--------|---------|---------|-------|
| Namespaces | 2 | 2 | 2 | 6 |
| Deployments | 12 | 12 | 12 | 36 |
| StatefulSets | 15 | 10 | 10 | 35 |
| Services | 30 | 25 | 25 | 80 |
| ConfigMaps | 15 | 15 | 15 | 45 |
| Secrets | 20 | 20 | 20 | 60 |
| PVCs | 15 | 10 | 10 | 35 |
| NetworkPolicies | 5 | 5 | 5 | 15 |
| Ingresses | 3 | 2 | 2 | 7 |

### Pod Count Summary

| Layer | Component | Replicas | Total Pods |
|-------|-----------|----------|------------|
| **Frontend** | gx-wallet-frontend | 1 | 1 |
| | gx-admin-frontend | 1 | 1 |
| **Backend API** | svc-identity | 3 | 3 |
| | svc-tokenomics | 3 | 3 |
| | svc-organization | 3 | 3 |
| | svc-loanpool | 3 | 3 |
| | svc-governance | 3 | 3 |
| | svc-admin | 3 | 3 |
| | svc-tax | 3 | 3 |
| | svc-government | 3 | 3 |
| | svc-messaging | 1 | 1 |
| **Backend Workers** | outbox-submitter | 2 | 2 |
| | projector | 1 | 1 |
| **Backend Data** | PostgreSQL | 3 | 3 |
| | Redis | 3 | 3 |
| | ClamAV | 1 | 1 |
| **Fabric Orderers** | orderer0-4 | 1 each | 5 |
| **Fabric Peers** | peer0/1-org1/2 | 1 each | 4 |
| **Fabric CouchDB** | couchdb-peer* | 1 each | 4 |
| **Fabric CAs** | ca-root/tls/orderer/org1/org2 | 1 each | 5 |
| **Fabric Chaincode** | gxtv3 | 1 | 1 |
| **Fabric DB** | PostgreSQL (CA backend) | 1 | 1 |
| **Monitoring** | Prometheus | 1 | 1 |
| | Grafana | 1 | 1 |
| | AlertManager | 1 | 1 |
| | Loki | 1 | 1 |
| | Promtail | 4 (DaemonSet) | 4 |
| | Node Exporter | 4 (DaemonSet) | 4 |
| | kube-state-metrics | 1 | 1 |
| | **TOTAL** | | **~70 pods** |

---

## Storage Allocation

| Component | Size | Storage Class | Purpose |
|-----------|------|---------------|---------|
| PostgreSQL (backend) | 100Gi | local-path | User data, read models |
| Redis | 20Gi | local-path | Cache, sessions |
| Prometheus | 50Gi | local-path | Metrics storage |
| Grafana | 10Gi | local-path | Dashboard state |
| Peer ledgers (×4) | 10Gi each | local-path | Blockchain state |
| CouchDB (×4) | 10Gi each | local-path | World state |
| Orderer ledgers (×5) | 5Gi each | local-path | Block storage |
| Fabric CAs (×5) | 5Gi each | local-path | Certificate store |
| PostgreSQL (Fabric CA) | 50Gi | local-path | CA database |
| ClamAV | 5Gi | local-path | Virus definitions |
| **TOTAL** | **~380Gi** | | |

---

## Network Topology

### External Access Points

| Domain | IP (via Cloudflare) | Service | Environment |
|--------|---------------------|---------|-------------|
| admin.gxcoin.money | 217.196.51.190 | Admin Portal | DevNet |
| devnet.gxcoin.money | 217.196.51.190 | Wallet Frontend | DevNet |
| devnet-api.gxcoin.money | 217.196.51.190 | Backend API | DevNet |
| api.gxcoin.money | 72.60.210.201 | Backend API | MainNet |
| wallet.gxcoin.money | TBD | Wallet Frontend | MainNet |

### Internal Service DNS

```
# Backend Services
svc-identity.backend-devnet.svc.cluster.local:3001
svc-tokenomics.backend-devnet.svc.cluster.local:3003
postgres.backend-devnet.svc.cluster.local:5432
redis-master.backend-devnet.svc.cluster.local:6379

# Fabric Network
peer0-org1.fabric-devnet.svc.cluster.local:7051
orderer0.fabric-devnet.svc.cluster.local:7050
ca-org1.fabric-devnet.svc.cluster.local:9054
couchdb-peer0-org1.fabric-devnet.svc.cluster.local:5984
```

---

## Security Implementation

### Network Policies (Zero-Trust Model)

1. **default-deny-all**: Block all traffic by default
2. **allow-intra-namespace**: Allow within same namespace
3. **allow-dns**: CoreDNS access (UDP 53)
4. **allow-monitoring-ingress**: Prometheus scraping
5. **allow-external-access**: Ingress controller only

### RBAC Configuration

| Service Account | Namespace | Purpose |
|-----------------|-----------|---------|
| backend-api | backend-* | API service pods |
| backend-worker | backend-* | Worker processes |
| fabric-peer | fabric-* | Peer nodes |
| fabric-orderer | fabric-* | Orderer nodes |
| fabric-ca | fabric-* | Certificate authorities |
| prometheus | monitoring | Metrics collection |

### TLS Configuration

| Component | TLS Enabled | Certificate Source |
|-----------|-------------|-------------------|
| Ingress (MainNet) | Yes | Let's Encrypt |
| Ingress (DevNet) | No | N/A |
| Fabric Orderers | Yes | Fabric CA |
| Fabric Peers | Yes | Fabric CA |
| Fabric CAs | Yes | Self-signed |
| Internal Services | No | N/A (ClusterIP) |

---

## Monitoring & Observability

### Prometheus Targets

- Kubernetes API server
- All nodes (via node-exporter)
- All pods with `prometheus.io/scrape: "true"` annotation
- PostgreSQL (via postgres-exporter)
- CouchDB (via couchdb-exporter)
- Redis (via redis-exporter)

### Grafana Access

```
URL: http://72.60.210.201:30300
Username: admin
Password: gxcoin2025 (CHANGE IN PRODUCTION!)
```

### Alert Rules

- Infrastructure: CPU > 80%, Memory > 85%, Disk > 90%
- Backend: API latency > 500ms, Error rate > 1%
- Fabric: Peer disconnected, Block time > 10s
- Database: Connection pool exhausted, Replication lag

---

## Key File Locations

### Frontend
- `/home/sugxcoin/prod-blockchain/gx-wallet-frontend/k8s/deployment-devnet.yaml`
- `/home/sugxcoin/prod-blockchain/gx-admin-frontend/Dockerfile`

### Backend
- `/home/sugxcoin/prod-blockchain/gx-protocol-backend/k8s/backend/deployments/`
- `/home/sugxcoin/prod-blockchain/gx-protocol-backend/k8s/infrastructure/database/`
- `/home/sugxcoin/prod-blockchain/gx-protocol-backend/k8s/ingress/`

### Fabric
- `/home/sugxcoin/prod-blockchain/gx-coin-fabric/k8s/base/peers/`
- `/home/sugxcoin/prod-blockchain/gx-coin-fabric/k8s/base/orderers/`
- `/home/sugxcoin/prod-blockchain/gx-coin-fabric/k8s/base/cas/`
- `/home/sugxcoin/prod-blockchain/gx-coin-fabric/k8s/base/chaincode/`

### Infrastructure
- `/home/sugxcoin/prod-blockchain/gx-infra-arch/k8s/devnet/`
- `/home/sugxcoin/prod-blockchain/gx-coin-fabric/k8s/monitoring/`

---

## Summary

The GX Protocol Kubernetes infrastructure is a **production-grade, enterprise-ready** deployment featuring:

- **High Availability**: 3-node PostgreSQL/Redis clusters, 5-node Raft orderer consensus
- **Security**: Zero-trust network policies, RBAC, TLS on all external connections
- **Scalability**: HPA configured for API services, StatefulSets for databases
- **Observability**: Full Prometheus/Grafana/Loki monitoring stack
- **Environment Isolation**: Complete separation between DevNet/TestNet (VPS4) and MainNet (VPS1-3)
- **CQRS Architecture**: Reliable blockchain integration with outbox pattern and event projection

**Total Infrastructure**: ~70 pods across 6 namespaces on 4 globally-distributed servers.

---

*Last Updated: 2026-01-06*
