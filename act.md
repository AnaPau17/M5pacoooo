# M4 – Component-Based Thinking: Component Identification & Architecture Partitioning

**Team 13** | Software Systems Planning (Gpo 101) | Guadalajara Campus  
**Project:** Cloud-Native Project Management Tool  
**Date:** April 7, 2026

---

## 1. Component Identification Process & Rationale

### Methodology: Workflow Approach

We chose the **Workflow Approach** to identify our system components. This method models components around the real end-to-end workflows that users and systems perform, rather than starting from abstract technical layers or pure domain events.

**Why this methodology fits our project:**

Our system — a Cloud-Native Project Management Tool — is fundamentally workflow-driven. Users don't just interact with isolated data entities; they follow sequences of actions: they log in, create a project, create and assign tasks, track progress on a Kanban board, and receive updates via a Telegram bot. These end-to-end sequences are well-defined and documented in our Sprint 0 requirements and test plan.

The Workflow Approach allowed us to:

1. Identify the **key roles** (Developer, Project Manager, System Administrator, Telegram Bot User) who initiate and participate in workflows.
2. Map the **kinds of workflows** each role engages in (authentication, task management, project tracking, bot interaction).
3. **Build components** around the cohesive activities within each workflow — ensuring high cohesion within each component and loose coupling between them.

This approach is also well-suited to our architecture, which is cloud-native and microservices-oriented, since the workflows translate naturally into service boundaries.

---

## 2. Workflow Flow Diagram

The following diagram traces the main workflows of the system, identifying actors, the actions they perform, and the components they interact with.

```mermaid
flowchart TD
    subgraph Actors
        DEV["Developer"]
        PM["Project Manager"]
        ADMIN["System Administrator"]
        BOT_USER["Telegram Bot User"]
    end

    subgraph AUTH_FLOW["Authentication Workflow"]
        A1["Register / Sign In"]
        A2["2FA Verification"]
        A3["JWT Token Issued"]
        A4["RBAC Role Assigned"]
    end

    subgraph TASK_FLOW["Task Management Workflow"]
        T1["Create Task"]
        T2["Assign Task"]
        T3["Update Task Status"]
        T4["View Kanban Board"]
        T5["Track Sprint Progress"]
    end

    subgraph BOT_FLOW["Bot Interaction Workflow"]
        B1["Send Telegram Command"]
        B2["Parse Natural Language"]
        B3["Execute System Action"]
        B4["Confirm Result to User"]
    end

    subgraph ADMIN_FLOW["Administration Workflow"]
        AD1["Manage User Roles"]
        AD2["Configure OCI IAM Policies"]
        AD3["Monitor System Health"]
    end

    DEV --> A1
    PM --> A1
    ADMIN --> A1
    BOT_USER --> B1

    A1 --> A2 --> A3 --> A4

    A4 --> T1
    A4 --> T4
    T1 --> T2 --> T3 --> T4
    T4 --> T5

    B1 --> B2 --> B3 --> B4
    B3 --> T3

    ADMIN --> AD1 --> AD2 --> AD3

    A3 --> C_AUTH["Component: User & Auth Microservice"]
    T1 --> C_TASK["Component: Project & Task Microservice"]
    T2 --> C_TASK
    T3 --> C_TASK
    T4 --> C_WEB["Component: Web Application (React SPA)"]
    T5 --> C_WEB
    B2 --> C_BOT["Component: Bot & LLM Microservice"]
    B3 --> C_GW["Component: OCI API Gateway"]
    C_GW --> C_TASK
    C_TASK --> C_DB["Component: Oracle Autonomous Database (ATP)"]
    C_AUTH --> C_DB
```

---

## 3. Component Table

The following table identifies the actors, events/actions, components, and their responsibilities derived from the workflow analysis above.

| Actor | Event / Action | Component | Component Responsibilities |
|---|---|---|---|
| Developer, Project Manager | Register, Sign In, 2FA, JWT issuance | **User & Auth Microservice** | Authenticate users via OCI IAM; validate and issue JWTs; enforce RBAC role assignments (ADMIN / MEMBER); protect all downstream API calls |
| Developer, Project Manager | Create task, assign task, update task status, track sprint KPIs | **Project & Task Microservice** | Manage full task lifecycle (TODO → IN_PROGRESS → BLOCKED → DONE); manage projects and sprints; calculate KPIs (cycle time, velocity, blocker resolution); expose REST API endpoints for task CRUD operations |
| Developer, Project Manager | View Kanban board, drag-and-drop columns, open task modals | **Web Application (React SPA)** | Render the Kanban UI with real-time task state; handle OCI IAM redirect and JWT storage; provide task creation and management forms with inline validation; present sprint and project dashboards |
| Telegram Bot User | Send natural language command ("mark task 123 as done") | **Bot & LLM Microservice** | Receive and validate Telegram webhook events; build system prompts and call Gemini API for NLP; parse intent and dispatch to appropriate backend action; send confirmation messages back to user |
| All external clients | Any HTTP request to the backend | **OCI API Gateway** | Route all HTTPS requests to the correct internal microservice via Kubernetes NodePort; enforce CORS policies; validate JWT presence; redirect HTTP to HTTPS (443); act as the single entry point for the system |
| All microservices | Persistent read/write operations | **Oracle Autonomous Database (ATP)** | Store and retrieve all persistent data: users, projects, sprints, tasks, and timestamps; support KPI analytics queries; enforce data retention policies |
| System Administrator | Manage roles, configure policies, monitor health | **OCI IAM & Infrastructure** | Define and enforce access policies for all OCI resources; manage user identities and roles at the cloud level; provide the identity provider for the Auth microservice |

---

## 4. Technical Partitioning

Technical partitioning organizes components by their **technical layer**: Presentation, API Gateway, Business Logic (Microservices), and Data Persistence. Think of this like the floors of a building — each floor has a specific purpose, and you must go through the floor above to reach the one below.

This partitioning aligns closely with a layered architecture pattern and clearly separates customization code (e.g., UI themes) from core business logic.

```mermaid
graph TD
    subgraph PRESENTATION["🖥️ Presentation Layer"]
        WEB["Web Application\n(React SPA)\n──────────────\n• Login Interface\n• Kanban Board UI\n• Task Management UI"]
    end

    subgraph GATEWAY["🔀 API Gateway Layer"]
        GW["OCI API Gateway\n──────────────\n• Request Router\n• CORS Handler\n• JWT / Auth Filter\n• Security Filters"]
    end

    subgraph BUSINESS["⚙️ Business Logic Layer (Microservices)"]
        AUTH["User & Auth\nMicroservice\n──────────────\n• Authentication Service\n• JWT Validation Module\n• RBAC Logic"]

        TASK["Project & Task\nMicroservice\n──────────────\n• Task API Endpoints\n• Sprint Management Module\n• KPI Calculation Service"]

        BOT["Bot & LLM\nMicroservice\n──────────────\n• Telegram Webhook Handler\n• LLM Processing Module\n• Command Parser"]
    end

    subgraph DATA["🗄️ Data Layer"]
        DB["Oracle Autonomous\nDatabase (ATP)\n──────────────\n• Users & Auth Schema\n• Projects & Tasks Schema\n• Sprints Schema"]
    end

    subgraph IDENTITY["☁️ Identity Provider"]
        IAM["OCI IAM\n──────────────\n• OIDC / OAuth2\n• Role Assignment\n• Policy Enforcement"]
    end

    WEB -->|"HTTPS Requests"| GW
    GW -->|"Route /auth/*"| AUTH
    GW -->|"Route /tasks/*, /projects/*, /sprints/*"| TASK
    GW -->|"Route /bot/*"| BOT

    AUTH -->|"SQL: Users, Sessions"| DB
    TASK -->|"SQL: Tasks, Projects, Sprints, KPIs"| DB
    BOT -->|"SQL: Task Updates"| DB

    AUTH <-->|"OIDC Redirect / Token Validation"| IAM
    WEB <-->|"OCI IAM Login Redirect"| IAM

    BOT -->|"Telegram API (outbound)"| TELE["📱 Telegram API\n(External)"]
    BOT -->|"Gemini API (NLP)"| GEMINI["🤖 Gemini API\n(External)"]
```

**Key interactions in Technical Partitioning:**
- The Web App communicates **only** through the API Gateway — it never calls microservices directly.
- The API Gateway applies security filters (JWT, CORS) **before** routing to any microservice.
- All three microservices share the same data layer, but each owns a distinct schema namespace.
- The Bot microservice has two external dependencies: the Telegram API (inbound webhooks) and the Gemini API (outbound NLP calls).

---

## 5. Domain Partitioning

Domain partitioning organizes components by **business capability or domain** — grouping everything related to a user-facing function together. Think of it like organizing a department store by product category (Electronics, Clothing, Food) rather than by the type of shelf they sit on.

This partitioning aligns more closely with how the business thinks about the system and makes it easier to evolve domains independently (e.g., swapping the bot without touching authentication).

```mermaid
graph TD
    subgraph AUTH_DOMAIN["🔐 Authentication Domain"]
        AUTH_SVC["User & Auth Microservice\n──────────────\n• Sign-in / Sign-up Forms\n• JWT Validation Module\n• RBAC Logic\n• OCI IAM Integration"]
    end

    subgraph TASK_DOMAIN["📋 Task & Project Domain"]
        TASK_SVC["Project & Task Microservice\n──────────────\n• Task API Endpoints\n• Sprint Management\n• Project Tracking\n• KPI Calculation"]
        KANBAN["Kanban Board UI\n(Web App Module)\n──────────────\n• Column Views (TODO→DONE)\n• Drag-and-Drop\n• Task Cards"]
    end

    subgraph BOT_DOMAIN["🤖 Conversational Bot Domain"]
        BOT_SVC["Bot & LLM Microservice\n──────────────\n• Telegram Webhook Handler\n• Gemini NLP Processing\n• Command Parser\n• Action Dispatcher"]
    end

    subgraph INFRA_DOMAIN["🌐 Infrastructure Domain"]
        GW["OCI API Gateway\n──────────────\n• Routing\n• CORS\n• Security Filters"]
        DB["Oracle ATP Database\n──────────────\n• All persistent schemas"]
        IAM["OCI IAM\n──────────────\n• Identity & Access"]
    end

    USER["👤 User\n(Developer / PM / Admin)"] -->|"Login"| AUTH_SVC
    USER -->|"Web Interactions"| KANBAN

    TELE_USER["📱 Telegram User"] -->|"Natural Language Commands"| BOT_SVC

    AUTH_SVC -->|"Token + Role"| GW
    KANBAN -->|"Task CRUD via Gateway"| GW
    BOT_SVC -->|"Task Updates via Gateway"| GW

    GW -->|"Authenticated Requests"| TASK_SVC
    GW -->|"Auth Validation"| AUTH_SVC

    TASK_SVC <-->|"Read/Write Task Data"| DB
    AUTH_SVC <-->|"Read/Write User Data"| DB
    BOT_SVC <-->|"Read/Write via Task Service"| TASK_SVC

    AUTH_SVC <-->|"OIDC Flows"| IAM

    BOT_SVC -->|"Webhook Events"| TELE_API["📡 Telegram API"]
    BOT_SVC -->|"NLP Requests"| GEMINI_API["🧠 Gemini API"]

    KANBAN -->|"Task State Display"| TASK_SVC
```

**Key interactions in Domain Partitioning:**
- The **Authentication Domain** is the entry point for all human users; it gates access to all other domains.
- The **Task & Project Domain** is the core of the application — it is the primary consumer of the database and the target of updates from both the web UI and the bot.
- The **Conversational Bot Domain** acts as an alternative user interface to the Task domain, translating natural language into structured task operations.
- The **Infrastructure Domain** is shared by all other domains but is treated as a concern separate from business logic — consistent with the Stable Dependencies Principle (stable components that many depend on, but that depend on very little themselves).

---

## Summary

| Partitioning Style | Best For | Our Use |
|---|---|---|
| **Technical** | Clear layer separation, layered monolith patterns, easy to apply CORS/Auth globally | Useful for our API Gateway and infrastructure design |
| **Domain** | Aligns with business capability, easier microservices migration, message flows match real workflows | Preferred view for team collaboration and future scaling |

Both views are complementary. The technical partitioning guides **how we deploy and secure** the system; the domain partitioning guides **how we evolve and own** each business capability independently.