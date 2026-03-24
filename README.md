# 🏥 Patient Management System

A **microservices-based** Patient Management System built with **Spring Boot**, featuring inter-service communication via **gRPC** and **Apache Kafka**, secured with **JWT authentication**, and unified through a **Spring Cloud API Gateway**.

> [!NOTE]
> This project is currently **under active development** and not all features are fully implemented yet.

---

## 📐 Architecture Overview

```
                         ┌─────────────────────┐
                         │    API Gateway       │
                         │    (Port 4004)       │
                         └──────┬───────┬───────┘
                     JWT        │       │
                  Validation    │       │
                         ┌──────┘       └──────┐
                         ▼                     ▼
               ┌──────────────────┐  ┌──────────────────┐
               │  Auth Service    │  │ Patient Service   │
               │  (Port 4005)     │  │ (Port 4000)       │
               │  [PostgreSQL]    │  │ [PostgreSQL]       │
               └──────────────────┘  └──┬──────────┬─────┘
                                        │ gRPC     │ Kafka
                                        ▼          ▼
                              ┌──────────────┐  ┌─────────────────┐
                              │   Billing    │  │  Analytics      │
                              │   Service    │  │  Service        │
                              │ (Port 4001)  │  │  (Port 4002)    │
                              │ (gRPC: 9001) │  │                 │
                              └──────────────┘  └─────────────────┘
```

---

## 🧩 Services

| Service | Port | Description |
|---|---|---|
| **API Gateway** | `4004` | Routes external requests to internal services, validates JWT tokens on protected routes |
| **Auth Service** | `4005` | Handles user authentication — login & JWT token generation/validation |
| **Patient Service** | `4000` | Core CRUD operations for patient records; publishes events to Kafka; calls Billing Service via gRPC |
| **Billing Service** | `4001` (HTTP) / `9001` (gRPC) | Creates billing accounts for patients; exposes a gRPC API |
| **Analytics Service** | `4002` | Consumes patient events from Kafka for analytics processing |

---

## 🛠 Tech Stack

- **Language:** Java 21
- **Framework:** Spring Boot 3.x / 4.x
- **API Gateway:** Spring Cloud Gateway (WebFlux)
- **Security:** Spring Security + JWT (jjwt)
- **Database:** PostgreSQL (patient-service, auth-service)
- **ORM:** Spring Data JPA / Hibernate
- **Inter-Service Communication:**
  - **gRPC** — Patient Service → Billing Service
  - **Apache Kafka** — Patient Service → Analytics Service (Protobuf-serialized events)
- **API Docs:** Springdoc OpenAPI (Swagger UI)
- **Build:** Maven
- **Containerization:** Docker (multi-stage builds)

---

## 📁 Project Structure

```
patient_management/
├── api-gateway/            # Spring Cloud Gateway — routing & JWT filter
├── auth-service/           # Authentication — login, token validation
├── patient-service/        # Patient CRUD, gRPC client, Kafka producer
├── billing-service/        # gRPC billing account service
├── analytics-service/      # Kafka consumer for patient events
├── api-requests/           # HTTP request files for testing REST APIs
│   ├── auth-service/
│   └── patient-service/
└── grpc-requests/          # HTTP request files for testing gRPC APIs
    └── billing-service/
```

---

## 🔌 API Endpoints

### Auth Service

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/login` | Authenticate with email & password, returns JWT |
| `GET` | `/validate` | Validate a JWT token (via `Authorization: Bearer <token>` header) |

**Default test credentials:**
```
Email:    testuser@test.com
Password: password123
```

### Patient Service

| Method | Endpoint | Description |
|---|---|---|
| `GET` | `/patients` | List all patients |
| `POST` | `/patients` | Create a new patient |
| `PUT` | `/patients/{id}` | Update an existing patient |
| `DELETE` | `/patients/{id}` | Delete a patient |

**Example — Create Patient:**
```json
POST /patients
Content-Type: application/json

{
  "name": "John Doe",
  "email": "john.doe@example.com",
  "address": "123 Main Street",
  "dateOfBirth": "1995-09-09",
  "registeredDate": "2024-11-28"
}
```

### Billing Service (gRPC)

| RPC | Method | Description |
|---|---|---|
| `BillingService` | `CreateBillingAccount` | Creates a billing account for a patient |

**gRPC endpoint:** `localhost:9001`

---

## 🚀 Running Locally with Docker

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) installed and running
- Ports `4000–4005`, `5001`, `9001`, `9092`, `9094` available

---

### Step 1 — Create a Docker Network

All services communicate on a shared Docker network called `internal`:

```bash
docker network create internal
```

---

### Step 2 — Start PostgreSQL Containers

The **Patient Service** and **Auth Service** each require their own PostgreSQL instance.

#### PostgreSQL for Patient Service

```bash
docker run -d \
  --name patient-service-db \
  --network internal \
  -e POSTGRES_USER=admin_user \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=db \
  postgres:17
```

> [!NOTE]
> No host port is mapped for `patient-service-db` — it is only accessible from within the `internal` Docker network. If you need to connect from your host (e.g., via a DB client), add `-p 5432:5432`.

#### PostgreSQL for Auth Service

```bash
docker run -d \
  --name auth-service-db \
  --network internal \
  -e POSTGRES_USER=admin_user \
  -e POSTGRES_PASSWORD=password \
  -e POSTGRES_DB=db \
  -v $(pwd)/../auth-service-db:/var/lib/postgresql/data \
  -p 5001:5432 \
  postgres:17
```

---

### Step 3 — Start Apache Kafka (KRaft Mode)

Kafka is used for async event streaming between Patient Service and Analytics Service. This project uses the official **Apache Kafka** image in **KRaft mode** (no Zookeeper required).

```bash
docker run -d \
  --name kafka \
  --network internal \
  -e KAFKA_NODE_ID=0 \
  -e KAFKA_PROCESS_ROLES=controller,broker \
  -e KAFKA_CLUSTER_ID=test-cluster-1 \
  -e KAFKA_LISTENERS=PLAINTEXT://:9092,CONTROLLER://:9093,EXTERNAL://:9094 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,EXTERNAL://localhost:9094 \
  -e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=CONTROLLER:PLAINTEXT,EXTERNAL:PLAINTEXT,PLAINTEXT:PLAINTEXT \
  -e KAFKA_CONTROLLER_LISTENER_NAMES=CONTROLLER \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=0@kafka:9093 \
  -p 9092:9092 \
  -p 9094:9094 \
  apache/kafka:latest
```

---

### Step 4 — Build & Run the Application Services

Build each service Docker image from the project root:

```bash
# Build all service images
docker build -t billing-service:latest  ./billing-service
docker build -t patient-service:latest  ./patient-service
docker build -t auth-service:latest     ./auth-service
docker build -t analytics-service:latest ./analytics-service
docker build -t api-gateway:latest      ./api-gateway
```

Run each service on the `internal` network, passing the required environment variables:

#### Billing Service

```bash
docker run -d \
  --name billing-service \
  --network internal \
  -p 4001:4001 \
  -p 9001:9001 \
  billing-service:latest
```

#### Patient Service

```bash
docker run -d \
  --name patient-service \
  --network internal \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://patient-service-db:5432/db \
  -e SPRING_DATASOURCE_USERNAME=admin_user \
  -e SPRING_DATASOURCE_PASSWORD=password \
  -e SPRING_JPA_HIBERNATE_DDL_AUTO=update \
  -e SPRING_SQL_INIT_MODE=always \
  -e SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092 \
  -e BILLING_SERVICE_ADDRESS=billing-service \
  -e BILLING_SERVICE_GRPC_PORT=9001 \
  patient-service:latest
```

#### Auth Service

```bash
docker run -d \
  --name auth-service \
  --network internal \
  -e SPRING_DATASOURCE_URL=jdbc:postgresql://auth-service-db:5432/db \
  -e SPRING_DATASOURCE_USERNAME=admin_user \
  -e SPRING_DATASOURCE_PASSWORD=password \
  -e SPRING_JPA_HIBERNATE_DDL_AUTO=update \
  -e SPRING_SQL_INIT_MODE=always \
  -e JWT_SECRET=IImy3y2vJQpr/+lgD5xf7N+pwkoffvpkEvXCfEv0Q0I= \
  auth-service:latest
```

> [!CAUTION]
> The `JWT_SECRET` value above is for **local development only**. Never commit production secrets to version control. Use a secrets manager or environment-specific configuration in production.

#### Analytics Service

```bash
docker run -d \
  --name analytics-service \
  --network internal \
  -e SPRING_KAFKA_BOOTSTRAP_SERVERS=kafka:9092 \
  -p 4002:4002 \
  analytics-service:latest
```

#### API Gateway

```bash
docker run -d \
  --name api-gateway \
  --network internal \
  -e AUTH_SERVICE_URL=http://auth-service:4005 \
  -p 4004:4004 \
  api-gateway:latest
```

---

### Step 5 — Verify Everything is Running

```bash
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

You should see **8 containers** running:

| Container | Role |
|---|---|
| `patient-service-db` | PostgreSQL for Patient Service |
| `auth-service-db` | PostgreSQL for Auth Service |
| `kafka` | Apache Kafka (KRaft mode) |
| `billing-service` | gRPC Billing Service |
| `patient-service` | Patient CRUD Service |
| `auth-service` | Authentication Service |
| `analytics-service` | Kafka Consumer Service |
| `api-gateway` | Spring Cloud API Gateway |

---

## 🧪 Testing the APIs

### Via API Gateway (Recommended)

All requests through the gateway go via `http://localhost:4004`.

**1. Login to get a JWT:**
```bash
curl -X POST http://localhost:4004/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email": "testuser@test.com", "password": "password123"}'
```

**2. Use the token to access Patient Service:**
```bash
curl http://localhost:4004/api/patients \
  -H "Authorization: Bearer <your-jwt-token>"
```

### Swagger UI (Direct Access)

- **Patient Service:** [http://localhost:4000/swagger-ui.html](http://localhost:4000/swagger-ui.html)
- **Auth Service:** [http://localhost:4005/swagger-ui.html](http://localhost:4005/swagger-ui.html)

### IntelliJ HTTP Client

Pre-made `.http` request files are available in:
- `api-requests/auth-service/` — login & token validation
- `api-requests/patient-service/` — CRUD operations
- `grpc-requests/billing-service/` — gRPC billing account creation

---

## 🧹 Cleanup

Stop and remove all containers and the network:

```bash
docker stop api-gateway analytics-service auth-service patient-service billing-service kafka auth-service-db patient-service-db
docker rm api-gateway analytics-service auth-service patient-service billing-service kafka auth-service-db patient-service-db
docker network rm internal
```

---

## 🗺 Roadmap

- [ ] Docker Compose file for one-command setup
- [ ] Service discovery (Eureka / Consul)
- [ ] Centralized configuration (Spring Cloud Config)
- [ ] Distributed tracing (Zipkin / Jaeger)
- [ ] CI/CD pipeline
- [ ] Kubernetes deployment manifests
- [ ] Frontend application

---

## 📄 License

This project is for educational / portfolio purposes.
