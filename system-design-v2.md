# SecuraOps AI CyberShield SMB Enhanced Technical \& System Design Specification

**Version: 1.1**
**Date: 27 June 2025**

## 1. Overview

This document outlines the refined technical and system design for the CyberShield SMB platform[^1][^2]. It provides a detailed breakdown of the front-end and back-end architectural layers, including their respective modules, components, and interfaces. These are mapped to the platform's functional and non-functional requirements.

CyberShield SMB is a multi-tenant cybersecurity and compliance Software-as-a-Service (SaaS) platform[^3][^4]. It is designed to deliver real-time threat detection, automated compliance processes, and AI-driven insights for small and medium-sized businesses.

## 2. Front-End Architecture

The front-end is a single-page application (SPA) developed using **Angular v20**, with **Tailwind CSS 4** for styling and **Angular Material (MDC-based)** for user interface widgets. To enhance performance and search engine optimization (SEO), it utilizes server-side rendering (SSR) with incremental hydration.

### 2.1 Component Tree

The front-end is structured into the following modules:

- **App Shell**: Handles the initial bootstrapping, theming, global providers, and lazy-loads feature shells.
- **Core Layout**: Consists of the main interface elements such as the header, side navigation, breadcrumbs, and responsive containers.
- **Auth Module**: Manages user authentication, including login, multi-factor authentication (MFA), password reset, and Single Sign-On (SSO) using OpenID Connect (OIDC).
- **Dashboard Module**: Displays key performance indicators (KPIs) for threats, compliance scorecards, and various informational widgets.
- **Incidents Module**: Features an incident list, a detailed information drawer, a timeline view, and actions for incident response.
- **Compliance Module**: Allows users to select compliance frameworks, view control evidence pages, and access an auditor-specific view.
- **Reporting Module**: Includes an export center, options for scheduling reports, and generation of PDF and CSV files.
- **Admin Module**: Provides settings for tenant management, user roles, billing, and feature flags.


### 2.2 State \& Data Flow

- **State Management**: The application employs a Signals-based reactive state management approach, which reduces boilerplate code and is ready for future zone-less implementation.
- **Data Access**: Data access services utilize generated API clients from OpenAPI specifications. These expose strongly-typed observables (RxJS) that stream data over HTTPS/JSON or gRPC-Web.
- **Real-Time Updates**: A WebSocket Gateway using STOMP over SockJS delivers real-time updates for notifications, incident statuses, and compliance evidence collection.


### 2.3 Non-Functional Requirements (Frontend)

- **Performance**: Aims for a First Contentful Paint (FCP) of less than 1.5 seconds on a 4G connection, achieved through Angular SSR with incremental hydration.
- **Accessibility**: Complies with Web Content Accessibility Guidelines (WCAG) 2.2 AA standards, utilizing the Angular CDK-A11y and Material MDC.
- **Responsiveness**: Implements a mobile-first design with responsive breakpoints managed by Tailwind CSS flex and grid utilities.
- **Internationalization**: Supports multiple languages and right-to-left (RTL) text using ngx-translate.
- **Security**: Ensures security with DOM-sanitized templates and Content Security Policy (CSP) nonces for any inline styles or scripts.


## 3. Back-End Architecture

The back-end is built on a microservices architecture using **Java 21**, **Spring Boot 3.3**, and **Project Reactor WebFlux** for reactive programming[^4][^5]. For AI and machine learning functionalities, services are implemented in **Python 3.12** with **FastAPI** and served via gRPC.

### 3.1 Core Microservices

- **Gateway Service**: Built with Spring Cloud Gateway, it handles OpenID Connect validation and rate limiting.
- **Tenant Service**: Manages the tenant registry, bootstraps schemas, and integrates with billing hooks.
- **Auth Service**: Uses Keycloak Identity Management (IDM) or Spring Authorization Server for authentication and MFA.
- **Ingest Service**: Employs Kafka producers to parse logs and vulnerability feeds.
- **Threat Service**: A reactive service that correlates events to identify and raise incidents[^2].
- **Compliance Service**: Maps security controls, tracks evidence, and generates attestations.
- **Reporting Service**: Uses reactive aggregation pipelines for data processing and PDF exporting.
- **Notification Service**: A WebSocket broker with adapters for email and SMS notifications.
- **AI Insight Service**: A Python-based service using TensorFlow and PyTorch models for anomaly scoring and providing GPT-based advisories[^2].


### 3.2 Data Storage Strategy

- **Transactional Data**: A PostgreSQL cluster is used with a schema-per-tenant model for data isolation[^5][^6]. Migrations are managed by Liquibase on a per-schema basis, and Row-Level Security (RLS) policies are enforced.
- **Time-Series Data**: TimescaleDB is used for storing high-volume security telemetry data.
- **Object Storage**: MinIO provides object storage for evidence artifacts, which are encrypted at rest.


### 3.3 Service Communication

- **Synchronous Communication**: For low-latency read operations, gRPC with unary protobuf messages is used, such as between the Gateway and Threat Service.
- **Asynchronous Events**: Kafka topics with Avro schemas are utilized for event propagation, including audit logs, incident events, and compliance status updates.
- **Saga Coordination**: A Kafka-based choreography approach is implemented using the transactional outbox pattern to ensure data consistency across services.


### 3.4 Real-Time Updates

Real-time functionality is enabled by **Spring WebSocket (STOMP)**, which is bridged to a Redis pub/sub system for horizontal scaling. For networks with firewall restrictions, the system can fall back to using Server-Sent Events (SSE).

### 3.5 Non-Functional Requirements (Backend)

- **Scalability**: Services are designed for horizontal scaling using Kubernetes Horizontal Pod Autoscaler (HPA), based on CPU usage and custom Kafka lag metrics.
- **Resilience**: The system incorporates resilience patterns such as Circuit Breaker (Resilience4j), retry mechanisms, and bulkheads for graceful degradation.
- **Observability**: Achieved through OpenTelemetry for tracing, Prometheus for metrics, and Loki for log aggregation[^7].
- **Security**: A Zero-Trust security model is enforced with mutual TLS (mTLS) between services, managed by SPIFFE/SPIRE, and secrets are managed by Vault[^7].
- **Performance**: The system targets a P99 API latency of 200 milliseconds or less and a Kafka end-to-end latency of 1 second or less.
- **Compliance**: All services log to an immutable Write-Once-Read-Many (WORM) S3 bucket within 5 seconds for compliance purposes[^7].


## 4. Technology Stack Matrix

| Category | Technology |
| :-- | :-- |
| **Frontend** | Angular v20, Tailwind CSS 4, Angular Material MDC, RxJS 8 |
| **API Gateway** | Spring Cloud Gateway, OIDC, RateLimiter |
| **Service Framework** | Spring Boot 3.3, WebFlux, Reactor |
| **Async Messaging** | Kafka 3.7, Schema Registry (Confluent), Avro |
| **Sync RPC** | gRPC + protobuf, grpc-spring-boot-starter |
| **Realtime** | Spring WebSocket/STOMP, Redis pub/sub |
| **Data** | PostgreSQL 16 (schema-per-tenant), TimescaleDB 2.14 |
| **AI/ML** | Python 3.12, FastAPI, PyTorch 2.3, scikit-learn 1.5 |
| **Containerization** | Docker, Helm, Kubernetes 1.30 |
| **CI/CD** | GitHub Actions, ArgoCD, Trivy vulnerability scans |

## 5. Functional Traceability Matrix

| User Story | Mapped Components |
| :-- | :-- |
| **As a Security Analyst, I can see real-time incident alerts** | WebSocket Gateway → Dashboard Module → Notification Service |
| **As a Compliance Manager, I can export a SOC 2 evidence package** | Compliance Module → Compliance Service → Reporting Service |

## 6. Deployment Topology

The platform is deployed on Kubernetes using **Amazon EKS**. The cluster is organized into three distinct node pools:

- **web-tier**: Hosts the front-end application and the API gateway.
- **core-services**: Runs the Java-based microservices.
- **ai-batch**: Utilizes GPU-enabled nodes for the Python-based AI/ML models.

The deployment also leverages **Amazon RDS** for PostgreSQL with read replicas, **Amazon MSK** for the Kafka cluster, and **Amazon ElastiCache** for Redis to facilitate WebSocket fan-out.

## 7. Appendix: Key Design Decisions

1. **Multi-Tenancy**: The schema-per-tenant model was adopted to provide a balance between data isolation and operational complexity, following guidance from the AWS SaaS Lens[^4][^5].
2. **UI Components**: Angular Material 3 components were selected for their accessibility features and enterprise-grade look and feel. Headless UI components are used as a fallback for custom widget development.
3. **Communication Protocols**: Kafka was chosen for its capabilities in ensuring loose coupling and providing a comprehensive audit trail. gRPC is reserved for specific low-latency read paths to optimize performance.
4. **Future Enhancements**: There is a plan to adopt the Signals API with zoneless change detection once the technology becomes stable, with the goal of completely removing Zone.js.

