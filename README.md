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
| `gateway` | `d3s1t1-gateway` | http://localhost:5003/graphql | Read-only GraphQL API over MS SQL (HotChocolate); Nitro IDE at the same URL |
| `frontend` | `d3s1t1-frontend` | http://localhost:3000 | Angular SSR dashboard — latest values, readings table, aggregation charts over the gateway |

Not implemented yet, so present only as a commented-out stub at the bottom of
`docker-compose.yml`: **notification service / SignalR** (`:5004`). Uncomment
that block as the service lands — the env wiring and `depends_on` edges are
already filled in.

## Prerequisites

* Docker Desktop / Docker Engine with Compose v2.23+ and BuildKit (default).
* **All sibling repositories cloned side by side** — the compose file builds
  from relative sibling paths:

  ```
  D3S1T1/
    Innowise.D3S1T1.DataIngestor/
    Innowise.D3S1T1.DataProcessor/
    Innowise.D3S1T1.Frontend/
    Innowise.D3S1T1.Gateway/
    Innowise.D3S1T1.Infrastructure/   <-- this repo, run compose from here
    Innowise.D3S1T1.WeakApp/
  ```

* A **GitHub PAT with `read:packages`**. The `DataProcessor` image restores
  `DataIngestor.Contracts` from the private `nuget.pkg.github.com/sailtor`
  feed; its Dockerfile reads the token as a BuildKit secret named
  `github_token`. It is the only image that needs one — the gateway consumes no
  private packages, so its build takes no secret.

## Run it

```bash
cp .env.example .env
# then put your PAT in .env  ->  GITHUB_TOKEN=ghp_xxxxxxxx

docker compose up -d --build
docker compose logs -f ingestor processor gateway frontend
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

4. Read the same rows back through the gateway — open
   <http://localhost:5003/graphql> for the Nitro IDE, or:

   ```bash
   curl -s http://localhost:5003/graphql \
     -H 'Content-Type: application/json' \
     -d '{"query":"{ availableRooms rooms { room totalReadings } }"}'
   ```

   `docker compose ps gateway` should read `healthy`; its `/health` runs the
   metrics-database check, so unhealthy means the database is not queryable
   (or the processor has not migrated it yet).

5. Open the dashboard — <http://localhost:3000> — and confirm the Home page's
   latest-values panel renders real data (not a loading skeleton or error
   banner) within a few seconds of load. `docker compose ps frontend` reporting
   `healthy` only proves the SSR server itself started; the browser's own
   GraphQL calls are a separate code path (see the gotcha below), so this step
   is what actually proves the dashboard works end to end.

## File layout

* `docker-compose.yml` — the full stack; internal wiring, healthchecks, volumes.
* `docker-compose.override.yml` — loaded automatically; publishes the AMQP and
  SQL ports to the host for SSMS / locally-debugged services, and turns up
  MassTransit + EF Core logging. Skip it with
  `docker compose -f docker-compose.yml up -d`.
* `.env.example` → copy to `.env` — passwords, API key, host ports.

## Notes and gotchas

* **Startup order.** `ingestor` and `processor` wait for RabbitMQ to report
  healthy, and `processor` additionally waits for SQL Server. `gateway` waits
  for SQL Server to be healthy and for `processor` to have *started* — not to
  be healthy, because the processor owns the migrations and nothing would
  release the gate if they had not run yet. The weak API is *not* gated on
  health — the ingestor is expected to survive it being down (Polly retry +
  circuit breaker are configured in `Integrations:WeakAPI:Resilience`).
* **The gateway tolerates a missing schema.** It never migrates anything, so
  if it comes up before the processor has created the table it starts anyway
  and reports `/health` as unhealthy rather than crash-looping. `/health/live`
  runs no checks and answers regardless.
* **`ASPNETCORE_ENVIRONMENT=Development`** is deliberate for `processor`:
  `ApplyPendingMigrations()` only runs in Development, and it also creates the
  database. It also enables the OpenAPI endpoint on both services.
* **HTTP only.** `ingestor` and `processor` call `UseHttpsRedirection()`,
  which no-ops when no HTTPS port is configured — so no dev certificates are
  needed. The gateway deliberately never calls it at all.
* **Port collisions.** Every service listens on 8080 *inside* its container;
  only the published host port differs (`INGESTOR_PORT`, `PROCESSOR_PORT`,
  `GRAPHQL_PORT`). Services address each other by compose service name on the
  internal network, never through a published port.
* **Gateway log levels live under `Serilog:*`**, not `Logging:LogLevel:*` — it
  logs through Serilog rather than the default logger, which is why the
  override file sets `Serilog__MinimumLevel__Default` for it and
  `Logging__LogLevel__Default` for the other two. `Serilog__UseJsonConsole` is
  forced on in compose so `docker logs` shows the structured output a collector
  would receive, even though the container runs as Development.
* **Introspection.** HotChocolate 16 disables introspection outside
  Development, which would break the frontend's codegen. The compose gateway
  runs as `Development` (so Nitro is served too); to keep introspection in a
  Production-configured run, set `GraphQL__AllowIntrospection=true` instead.
* **Trailing slash** on `Integrations__WeakAPI__URL` matters:
  `HttpClient.BaseAddress` resolves `meters` relatively, and without it the
  last path segment would be dropped.
* **The weak API ships as prebuilt output.** Its Dockerfile copies
  `./publish/`; that folder is committed (its `.gitignore` whitelists it), so
  no SDK build stage is involved.
* **SQL Server password** must satisfy complexity rules and should avoid
  `;` `"` `'` `$` — those break the connection string or Compose interpolation.
* **Frontend's GraphQL URL is wired two different ways on purpose.** The SSR
  server reads `GATEWAY_GRAPHQL_URL` (container-to-container DNS,
  `http://gateway:8080/graphql`) because its fetches happen inside this
  network. The browser's own fetches happen on the developer's host, so they
  use a build-time-baked absolute URL (`http://localhost:5003/graphql`,
  `src/environments/environment.prod.ts` in that repo) instead of an env var —
  there is no way to hand a browser bundle a runtime environment variable
  without a server-side template step this app doesn't have.
* **`NG_ALLOWED_HOSTS` is not optional for the frontend container.** Angular's
  SSR server rejects any request whose `Host` header isn't on this allowlist
  (hostname only, no port — `@angular/ssr`'s own URL parsing strips it) with a
  400, the container's own healthcheck included. Compose sets
  `NG_ALLOWED_HOSTS: localhost`, which is why `${FRONTEND_PORT}` can change
  without touching it. Omitting this var is the single most likely way to see
  `frontend` stuck `unhealthy` with every request 400ing.
