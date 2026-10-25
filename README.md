# Resilient Order Workflow — personal development skeleton

Initial **SETUP-01** project skeleton, built independently of the shared repository.
This phase deliberately contains **no order processing, inventory reservation, payment logic, Saga, Outbox, or frontend app**.

## Components

| Service | Tech | Database | Health endpoint |
| --- | --- | --- | --- |
| Order | Spring Boot / Maven / Java 17 | `order_db` | http://localhost:8081/actuator/health |
| Inventory | Spring Boot / Maven / Java 17 | `inventory_db` | http://localhost:8082/actuator/health |
| Payment | Spring Boot / Maven / Java 17 | `payment_db` | http://localhost:8083/actuator/health |
| Infra | PostgreSQL 16 + RabbitMQ management | Docker Compose | RabbitMQ UI: http://localhost:15672 |

Each service has its own database inside **one PostgreSQL container** for local development. No tables or domain objects exist yet.

## Requirements

- **JDK 17** selected for Maven and runtime (`java -version`, `mvn -version`).
- **Maven 3.6.3+** installed and on PATH.
- Docker Desktop with Docker Compose v2.

## Start (Windows PowerShell)

From repository root:

```powershell
Copy-Item .env.example .env
docker compose up -d
docker compose ps
```

Next start **three separate PowerShell terminals** at the repository root:

```powershell
cd order-service
mvn spring-boot:run
```

```powershell
cd inventory-service
mvn spring-boot:run
```

```powershell
cd payment-service
mvn spring-boot:run
```

Once each process reports it has started, use a fourth terminal:

```powershell
Invoke-RestMethod http://localhost:8081/actuator/health
Invoke-RestMethod http://localhost:8082/actuator/health
Invoke-RestMethod http://localhost:8083/actuator/health
```

Expected from each: `status: UP` (Spring Actuator JSON `{ "status": "UP" }`). Health includes checks of PostgreSQL and RabbitMQ when enabled by their starters.

RabbitMQ management UI: http://localhost:15672 — login using `RABBITMQ_USER` and `RABBITMQ_PASSWORD` values in `.env`.

## Notes and troubleshooting

- `.env` is deliberately ignored by Git. The `application.yml` fallbacks are **local-only example credentials**; never deploy them as-is.
- If port 5432, 5672, or 15672 is already occupied, stop the conflicting application/container or adjust port mappings and the matching service variables.
- SQL files in `/docker-entrypoint-initdb.d/` execute only when the PostgreSQL volume is first created. Do **not** delete volumes on a real data store. On this empty local skeleton only, `docker compose down -v` resets all data.
- These Spring Boot services run **on the host**, not inside Docker; `localhost` is correct in `application.yml`. When containerizing Java services later, switch to service hostnames `postgres` and `rabbitmq`.
- If Java 24 is installed on your machine, select the Java 17 JDK for `JAVA_HOME` and ensure Maven is using it (see `mvn -version`).
- Service-to-service calls, RabbitMQ queues/exchanges, error handling, and transaction coordination are **out of scope** for SETUP-01.

## Acceptance checklist

- [ ] `docker compose ps` shows PostgreSQL and RabbitMQ healthy.
- [ ] All three Spring Boot services start with JDK 17.
- [ ] `/actuator/health` responds `UP` on ports 8081, 8082, and 8083.
- [ ] No business flow or Saga code exists yet.
- [ ] Changes remain in the **personal repository** until reviewed for integration into the shared project.
