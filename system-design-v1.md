# SecuraOps AI CyberShield SMB  
## Enhanced Technical & System Design Specification  
**Version 1.1 – 27 June 2025**

---

## 1. Overview  
This document refines the original Technical & System Design Document by segregating the front-end and back-end architecture layers, detailing their modules, components, interfaces, and mapping them to functional and non-functional requirements. The product (CyberShield SMB) is a multi-tenant cybersecurity and compliance SaaS platform delivering real-time threat detection, compliance automation, and AI-assisted insights for small and medium-sized businesses.

---

## 2. Front-End Architecture  
Built with **Angular v20**, **Tailwind CSS 4**, and **Angular Material (MDC-based)** for UI widgets. The presentation layer is a full SPA with server-side rendering (SSR) and incremental hydration for SEO and performance.

### 2.1 Component Tree  
- **App Shell**: Bootstrap, theming, global providers, feature shell lazy-loads  
- **Core Layout**: Header, side-nav, breadcrumb, responsive containers  
- **Auth Module**: Login, MFA, password reset, SSO (OIDC)  
- **Dashboard Module**: Threat KPIs, compliance scorecards, widgets  
- **Incidents Module**: Incident list, detail drawer, timeline, response actions  
- **Compliance Module**: Framework selector, control evidence pages, auditor view  
- **Reporting Module**: Export center, scheduled reports, PDF/CSV  
- **Admin Module**: Tenant settings, user roles, billing, feature flags  

### 2.2 State & Data Flow  
- **Signals-based reactive state management** replaces NgRx, minimizing boilerplate while remaining zone-less ready  
- **Data Access Services**: Generated API clients (OpenAPI) expose strongly-typed observables (RxJS) that stream data via HTTPS/JSON or gRPC-Web  
- **WebSocket Gateway**: STOMP over SockJS delivers real-time updates to notifications, incident status, and compliance evidence collection  

### 2.3 Non-Functional Requirements (Frontend)  
- **Performance**: FCP < 1.5 s on 4G; angular-ssr with incremental hydration  
- **Accessibility**: WCAG 2.2 AA using Angular CDK-A11y and Material MDC  
- **Responsiveness**: Tailwind flex/grid utilities; mobile-first breakpoints  
- **Internationalization**: ngx-translate, RTL support  
- **Security**: DOM-sanitized templates, CSP nonce for inline styles/scripts  

---

## 3. Back-End Architecture  
Microservices built with **Java 21**, **Spring Boot 3.3**, and **Project Reactor WebFlux**. Services communicate over asynchronous Kafka topics or synchronous gRPC where low-latency is critical. A **PostgreSQL** cluster (schema-per-tenant) persists transactional data, while **TimescaleDB** stores high-volume telemetry events. AI/ML services are implemented in **Python 3.12** using FastAPI and served via gRPC.

### 3.1 Core Microservices  
- **Gateway Service**: Spring Cloud Gateway, OpenID Connect validation, rate limiting  
- **Tenant Service**: Tenant registry, schema bootstrap, billing hooks  
- **Auth Service**: Keycloak IDM or Spring Authorization Server, MFA  
- **Ingest Service**: Kafka producers; parses logs, vulnerability feeds  
- **Threat Service**: WebFlux + Reactor; correlates events, raises incidents  
- **Compliance Service**: Maps controls, tracks evidence, generates attestations  
- **Reporting Service**: Reactive aggregation pipelines, PDF export  
- **Notification Service**: WebSocket broker (Spring Messaging) + Email/SMS adapters  
- **AI Insight Service (Python)**: TensorFlow/PyTorch models for anomaly scores, GPT-based advisory  

### 3.2 Data Storage Strategy  
- **PostgreSQL – Schema-per-tenant**: Liquibase migrations per schema; RLS policies enforced by `current_setting(app.current_tenant)`  
- **TimescaleDB** – High-volume time-series security telemetry  
- **MinIO** – Object storage for evidence artifacts (encrypted at rest)  

### 3.3 Service Communication  
- **Command/Query**: gRPC unary (protobuf) for low-latency reads (Gateway → Threat Service)  
- **Event Propagation**: Kafka topics (Avro/Schema Registry) for audit logs, incident events, compliance status  
- **Saga Coordination**: Kafka-based choreography with transactional outbox pattern  

### 3.4 Real-Time Updates  
- **Spring WebSocket (STOMP)** bridged to Redis pub/sub for horizontal scaling  
- Fallback to **Server-Sent Events** for firewall-restricted networks  

### 3.5 Non-Functional Requirements (Backend)  
- **Scalability**: Horizontal via Kubernetes; HPA based on CPU and custom Kafka lag metrics  
- **Resilience**: Circuit Breaker (Resilience4j), retry, bulkhead, graceful degradation  
- **Observability**: OpenTelemetry traces (gRPC + HTTP), Prometheus metrics, Loki logs  
- **Security**: Zero-Trust, mTLS among services (SPIFFE/SPIRE), Vault-managed secrets  
- **Performance**: P99 API latency ≤ 200 ms, Kafka end-to-end ≤ 1 s  
- **Compliance**: Services log to immutable store (WORM S3) within 5 s  

---

## 4. Technology Stack Matrix  
| Layer / Concern | Technology |
|-----------------|------------|
| **Frontend** | Angular v20, Tailwind CSS 4, Angular Material MDC, RxJS 8 |
| **API Gateway** | Spring Cloud Gateway, OIDC, RateLimiter |
| **Service Framework** | Spring Boot 3.3, WebFlux, Reactor |
| **Async Messaging** | Kafka 3.7, Schema Registry (Confluent) Avro |
| **Sync RPC** | gRPC + protobuf, `grpc-spring-boot-starter` |
| **Realtime** | Spring WebSocket/STOMP, Redis pub/sub |
| **Data** | PostgreSQL 16 (schema-per-tenant), TimescaleDB 2.14 |
| **AI/ML** | Python 3.12, FastAPI, PyTorch 2.3, scikit-learn 1.5 |
| **Containerization** | Docker, Helm, Kubernetes 1.30 |
| **CI/CD** | GitHub Actions, ArgoCD, Trivy vulnerability scans |

---

## 5. Functional Traceability Matrix  
| User Story | Service / Component Path |
|------------|-------------------------|
| *As a Security Analyst, I can see real-time incident alerts* | WebSocket Gateway → Dashboard Module → Notification Service |
| *As a Compliance Manager, I can export SOC 2 evidence package* | Compliance Module → Compliance Service → Reporting Service |

---

## 6. Deployment Topology  
Kubernetes (AWS EKS) with three node pools:  
1. **web-tier** (frontend, gateway)  
2. **core-services** (Java microservices)  
3. **ai-batch** (GPU-enabled for Python models)  

Supporting infrastructure:  
- **RDS** for PostgreSQL with read replicas  
- **MSK** for Kafka  
- **Elasticache Redis** for WebSocket fan-out  

---

## 7. Appendix: Key Design Decisions  
1. **Schema-per-tenant** adopted to balance isolation and operational complexity (per AWS SaaS Lens guidance).  
2. **Angular Material 3** components chosen for accessibility and enterprise look-and-feel; fallback to Headless UI for custom widgets.  
3. **Kafka** provides loose coupling and audit trail; **gRPC** reserved for low-latency read paths.  
4. **Signals API** + zoneless change detection planned once stable to remove Zone.js entirely.  
