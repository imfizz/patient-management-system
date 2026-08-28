# Patient Management System

> A microservices playground I keep coming back to. Every time I open it after a break I've
> forgotten how it all wires together — so this README is my way back in, and honestly it doubles
> as the tour I'd give in an interview.

A distributed **Spring Boot microservices** system for managing patients: JWT-secured, event-driven
over Kafka, service-to-service billing over gRPC, and deployed as infrastructure-as-code to a local
AWS environment. **Java 21, Spring Boot 3.4.1.**

Architecture at a glance: [`Diagram/Patient Management System Flow.drawio.png`](Diagram/Patient%20Management%20System%20Flow.drawio.png)
— top half is the local Docker network, bottom half is the AWS/ECS deployment.

## What this project demonstrates

If I'm walking someone through this, these are the points worth making:

- **Microservice decomposition** — five independent services, each with its own responsibility, port, and (where needed) its own database. No shared schema.
- **API Gateway pattern** — a single public entry point (Spring Cloud Gateway) doing routing + a custom **JWT validation filter**, so downstream services stay focused on business logic.
- **Two communication styles, chosen on purpose:**
  - **gRPC** (synchronous) for patient → billing, with a `.proto` contract — used where the caller needs an immediate answer.
  - **Kafka** (asynchronous) for broadcasting patient events — used where consumers should react independently and scale on their own.
- **Event-driven design** — patient-service produces to a `patients` topic; analytics-service consumes it. Adding a new consumer (e.g. notifications) needs zero changes to the producer.
- **Infrastructure as Code** — the entire AWS stack (ECS/Fargate, RDS, MSK, ALB) is defined in **AWS CDK (Java)** and deployed to **LocalStack**, so the cloud architecture is version-controlled and reproducible without spending a cent.
- **Testing discipline** — a dedicated integration-test module plus a ready-to-run Postman collection.

## The services

| Service | Port | Responsibility |
|---|---|---|
| **api-gateway** | 4004 | Single entry point. Routes `/auth/**`, `/api/patients/**`, `/api-docs/**`; validates JWT before patient calls. |
| **auth-service** | 4005 | Login + JWT issuing/validation. Own database. |
| **patient-service** | 4000 | Core patient CRUD. Own database. Kafka **producer** + gRPC **client** to billing. |
| **billing-service** | 4001 (REST), **9001** (gRPC) | gRPC **server**. Creates a billing account when a patient is created. Contract: `billing_service.proto`. |
| **analytics-service** | 4002 | Kafka **consumer** of the `patients` topic — the read/analytics side. |

> A notification service appears in the diagram as a second Kafka consumer; it's the obvious next
> module and a nice example of how the event-driven design makes that a drop-in addition.

## How it fits together

```
Client → API Gateway → Auth Service        (login → JWT)
                    → Patient Service ──gRPC──→ Billing Service
                                     └─Kafka producer→ "patients" topic → Analytics Service (consumer)
```

- The **gateway** is the only public surface; services talk to each other by name on the Docker network.
- **gRPC** = synchronous billing (contract shared between patient-service and billing-service).
- **Kafka** = async fan-out of patient events, so consumers evolve and scale independently.

## Running it locally

Each service is a standard Maven Spring Boot app — run from its folder:

```bash
./mvnw spring-boot:run
```

Services resolve each other by hostname (`patient-service:4000`, `auth-service:4005`, …), so they're
built to run on a shared **Docker network** — each ships a `Dockerfile`. Patient-service can also run
solo on an in-memory **H2** DB (uncomment the block in
[`patient-service/.../application.properties`](patient-service/src/main/resources/application.properties)).

## Deploying to LocalStack (the AWS simulation)

The part I always have to look up. Full steps live in [`INFO.txt`](INFO.txt); short version:

1. Start LocalStack Pro (`lstk start`).
2. Configure the AWS CLI with dummy creds (`test` / `test`, `us-east-1`, `json`).
3. Deploy the CDK stack:
   ```bash
   cd infrastructure && bash ./localstack-deploy.sh
   ```
4. The script prints the **ALB DNS name** — that's the base URL for Postman
   (e.g. `lb-xxxx.elb.localhost.localstack.cloud`).

The stack itself is code:
[`infrastructure/.../LocalStack.java`](infrastructure/src/main/java/com/pm/stack/LocalStack.java)
provisions 2× **RDS** databases (auth + patient), an **MSK** Kafka cluster, an **ECS/Fargate** cluster
running each service as a task, and an **Application Load Balancer** out front.

## Testing

- **Postman:** import from [`Postman Collection/`](Postman%20Collection) (collection + local environment).
- **api-requests/:** raw HTTP request files for quick manual calls.
- **integration-tests/:** JUnit integration tests as their own Maven module.

## Layout

```
api-gateway/         Spring Cloud Gateway (routing + JWT filter)
auth-service/        Login + JWT
patient-service/     Patient CRUD, Kafka producer, gRPC client
billing-service/     gRPC server (billing)
analytics-service/   Kafka consumer
infrastructure/      AWS CDK stack (deploys everything to LocalStack)
integration-tests/   End-to-end tests
Postman Collection/  API test collection
Diagram/             Architecture diagram (.drawio + .png)
INFO.txt             LocalStack / AWS CLI setup steps
```

---

**Tech:** Java 21 · Spring Boot 3.4.1 · Spring Cloud Gateway · gRPC/Protobuf · Apache Kafka ·
PostgreSQL (RDS) · AWS CDK · ECS Fargate · MSK · LocalStack · Docker · Maven
