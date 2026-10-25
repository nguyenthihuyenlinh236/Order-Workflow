# Resilient Order Workflow — SETUP-01

A new Java 17 / Spring Boot 3.5.7 / Maven project skeleton. Each service exposes
Actuator health and connects to local PostgreSQL and RabbitMQ. There are no
business endpoints, tables, order processing, inventory reservations, payment
processing, Saga, Outbox, message listeners, queues, exchanges, or messaging
workflows. `frontend/` is documentation only.

| Component         | Port         | Database                         | Health               |
| ----------------- | ------------ | -------------------------------- | -------------------- |
| order-service     | 8081         | order_db                         | `/actuator/health`   |
| inventory-service | 8082         | inventory_db                     | `/actuator/health`   |
| payment-service   | 8083         | payment_db                       | `/actuator/health`   |
| PostgreSQL 16     | 5433         | Three databases in one container | Compose health check |
| RabbitMQ 4        | 5672 / 15672 | AMQP / management UI             | Compose health check |

## Requirements

- JDK **17**, selected for both Maven and Java runtime.
- Maven **3.6.3+** on PATH (validated with 3.9.9).
- Docker Desktop running Linux containers, with Docker Compose v2.
- Windows PowerShell 5.1+ or PowerShell 7.

```powershell
java -version
mvn -version
docker version
docker compose version
```

`mvn -version` must report Java 17. If needed, set `$env:JAVA_HOME` to your JDK 17
installation and prepend `"$env:JAVA_HOME\bin;"` to `$env:Path`.

## Start infrastructure (Windows PowerShell)

Run from the repository root. Create `.env` once; do not overwrite an existing
local configuration:

```powershell
if (!(Test-Path .env)) { Copy-Item .env.example .env }
docker compose up -d --wait
docker compose ps
```

The default values are **local development examples**, not production credentials.
Docker publishes infrastructure ports on loopback only. PostgreSQL initializes
`order_db`, `inventory_db`, and `payment_db` on the first start of its data volume.
Each database is owned by the local `POSTGRES_USER`; separate database users and
production security are outside this skeleton's scope.

## Build and test

From the root:

```powershell
mvn clean verify
```

This builds three executable jars and runs one HTTP health smoke test per service.
Tests use random HTTP ports and explicitly disable the PostgreSQL and RabbitMQ
health contributors, so they do not require Docker. They verify application
startup and the Actuator HTTP response; live infrastructure readiness must also
be checked below. There are no domain tests because there is no business logic.

## Run the services

Open three PowerShell terminals at the repository root. **In each terminal**,
load the same `.env` before starting Maven. Compose reads `.env` automatically;
Spring Boot does not. This loader supports the unquoted `KEY=value` format in
`.env.example` (no multiline values or inline comments):

```powershell
Get-Content .env | ForEach-Object {
    $line = $_.Trim()
    if ($line -and !$line.StartsWith('#')) {
        $parts = $line -split '=', 2
        if ($parts.Count -eq 2) {
            [Environment]::SetEnvironmentVariable($parts[0].Trim(), $parts[1], 'Process')
        }
    }
}
```

Then run one command per terminal:

```powershell
mvn -pl order-service spring-boot:run
```

```powershell
mvn -pl inventory-service spring-boot:run
```

```powershell
mvn -pl payment-service spring-boot:run
```

Alternatively, after building, run `java -jar <service>/target/<service>-0.0.1-SNAPSHOT.jar`
with the same environment variables. Services run on the host, outside Docker.

## Verify live readiness

With infrastructure healthy and all three services running, use a fourth terminal:

```powershell
8081, 8082, 8083 | ForEach-Object {
    $health = Invoke-RestMethod "http://localhost:$_/actuator/health"
    if ($health.status -ne 'UP') { throw "Service on port $_ is not healthy" }
    "Port $_ : $($health.status)"
}
```

Expected: `UP` from each. Default application configuration retains both the
PostgreSQL and RabbitMQ health contributors; a dependency failure makes overall
health `DOWN` (HTTP 503). Health details are not exposed. If you customize service
ports, adjust the checks accordingly.

RabbitMQ management: `http://localhost:15672`; use `RABBITMQ_USER` and
`RABBITMQ_PASSWORD` from your local `.env`. Merely connecting for a health check
does not install any messaging workflow.

## Stop and troubleshoot

- Stop Java services with Ctrl+C in their terminals.
- Run `docker compose down` to stop infrastructure; named volumes retain data.
- Initialization SQL only runs on **new** PostgreSQL volumes. Editing `.env`
  credentials also does not update existing database roles or RabbitMQ users.
- Only for disposable local data, `docker compose down -v` removes **all** local
  PostgreSQL and RabbitMQ data so initialization can run again. Never use it on
  data you need to keep.
- For occupied ports, adjust `POSTGRES_PORT`, `RABBITMQ_PORT`,
  `RABBITMQ_MANAGEMENT_PORT`, or the `*_SERVICE_PORT` variables in `.env`, reload
  them into every service terminal, and recreate infrastructure as needed.
- Diagnose dependencies with `docker compose logs postgres rabbitmq` and service
  console output. Check that both containers are healthy before checking HTTP.
- `POSTGRES_HOST` and `RABBITMQ_HOST` default to `localhost` for host processes.
  Containerized application processes would require Compose service hostnames;
  application containerization is not included here.
- Keep `.env` out of version control. Use appropriate authentication, TLS, secret
  management and restricted management access before any deployment.

## Layout

```text
pom.xml                    Maven parent and three-module reactor
order-service/             Application entrypoint, configuration, health smoke test
inventory-service/         Application entrypoint, configuration, health smoke test
payment-service/           Application entrypoint, configuration, health smoke test
infra/postgres/            First-start database initialization SQL
docker-compose.yml         PostgreSQL and RabbitMQ only
.env.example               Shared local settings for Compose and host services
frontend/                  Placeholder documentation only
```

Work is prepared on `feature/setup-01-skeleton`; merge into `main` is a separate
review decision.
