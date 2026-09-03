# Innowise.D3S1T1 — Infrastructure

Local-development orchestration for the D3S1T1 microservices stack.

## What comes up

| Service | Container | Host URL | Role |
|---|---|---|---|
| `weakapp` | `d3s1t1-weakapp` | http://localhost:8080 | Unstable external API (provided). `GET /meters`, header `X-Api-Key` |
| `rabbitmq` | `d3s1t1-rabbitmq` | http://localhost:15672 (guest/guest) | Message queue + management UI. AMQP on `localhost:5672` |
| `mssql` | `d3s1t1-mssql` | `localhost,1433` (sa) | MS SQL Server 2022 Developer |
| `ingestor` | `d3s1t1-ingestor` | http://localhost:5001 | Polls the weak API every 10s (Quartz), publishes to the `metric-readings` fanout exchange |
| `processor` | `d3s1t1-processor` | http://localhost:5002 | Consumes `metric-readings-processor`, persists to MS SQL, owns the EF Core migrations |

Not implemented yet, so present only as commented-out stubs at the bottom of
`docker-compose.yml`: **GraphQL API gateway** (`:5003`), **notification
service / SignalR** (`:5004`) and the **frontend** (`:3000`). Uncomment each
block as the service lands — the env wiring and `depends_on` edges are already
filled in.

## Prerequisites

* Docker Desktop / Docker Engine with Compose v2.23+ and BuildKit (default).
* **All four repositories cloned side by side** — the compose file builds from
  relative sibling paths:

  ```
  D3S1T1/
    Innowise.D3S1T1.DataIngestor/
    Innowise.D3S1T1.DataProcessor/
    Innowise.D3S1T1.Infrastructure/   <-- this repo, run compose from here
    Innowise.D3S1T1.WeakApp/
  ```

* A **GitHub PAT with `read:packages`**. The `DataProcessor` image restores
  `DataIngestor.Contracts` from the private `nuget.pkg.github.com/sailtor`
  feed; its Dockerfile reads the token as a BuildKit secret named
  `github_token`.

## Run it

```bash
cp .env.example .env
# then put your PAT in .env  ->  GITHUB_TOKEN=ghp_xxxxxxxx

docker compose up -d --build
docker compose logs -f ingestor processor
```

`.env` is git-ignored (`*.env` in `.gitignore`); `.env.example` is the tracked
template. `GITHUB_TOKEN` is only used at **build** time and is never baked into
an image layer or exposed to a running container.

Tear down, keeping the database and broker state:

```bash
docker compose down
```

Tear down and wipe the volumes (fresh database, migrations re-apply):

```bash
docker compose down -v
```

## Verifying the pipeline

1. `curl -H "X-Api-Key: supersecret" http://localhost:8080/meters` — the weak API
   responds (or fails; it is deliberately flaky, which is the point).
2. RabbitMQ UI → *Exchanges* → `metric-readings`, *Queues* →
   `metric-readings-processor` should show message rates within ~10s.
3. Query the database:

   ```bash
   docker compose exec mssql /opt/mssql-tools18/bin/sqlcmd \
     -S localhost -U sa -P "$MSSQL_SA_PASSWORD" -C \
     -Q "SELECT COUNT(*) FROM DataProcessor.dbo.MetricReadings"
   ```

## File layout

* `docker-compose.yml` — the full stack; internal wiring, healthchecks, volumes.
* `docker-compose.override.yml` — loaded automatically; publishes the AMQP and
  SQL ports to the host for SSMS / locally-debugged services, and turns up
  MassTransit + EF Core logging. Skip it with
  `docker compose -f docker-compose.yml up -d`.
* `.env.example` → copy to `.env` — passwords, API key, host ports.

## Notes and gotchas

* **Startup order.** `ingestor` and `processor` wait for RabbitMQ to report
  healthy, and `processor` additionally waits for SQL Server. The weak API is
  *not* gated on health — the ingestor is expected to survive it being down
  (Polly retry + circuit breaker are configured in
  `Integrations:WeakAPI:Resilience`).
* **`ASPNETCORE_ENVIRONMENT=Development`** is deliberate for `processor`:
  `ApplyPendingMigrations()` only runs in Development, and it also creates the
  database. It also enables the OpenAPI endpoint on both services.
* **HTTP only.** Both services call `UseHttpsRedirection()`, which no-ops when
  no HTTPS port is configured — so no dev certificates are needed.
* **Trailing slash** on `Integrations__WeakAPI__URL` matters:
  `HttpClient.BaseAddress` resolves `meters` relatively, and without it the
  last path segment would be dropped.
* **The weak API ships as prebuilt output.** Its Dockerfile copies
  `./publish/`; that folder is committed (its `.gitignore` whitelists it), so
  no SDK build stage is involved.
* **SQL Server password** must satisfy complexity rules and should avoid
  `;` `"` `'` `$` — those break the connection string or Compose interpolation.
