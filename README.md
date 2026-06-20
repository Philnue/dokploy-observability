# Dokploy Observability Stack

Central Grafana/Loki/Tempo/Prometheus/Alloy stack for multiple Dokploy
projects on one VPS.

Only Grafana is public through Traefik. Loki, Tempo, Prometheus, and Alloy stay
inside the Docker network.

This stack is configured for a fresh install. It does not keep backwards
compatibility with the old Promtail/Loki v11 setup.

## What Runs Here

- Grafana: UI, dashboards, Explore, alert overview
- Loki: log storage
- Alloy: Docker log collector for all containers on the host
- Prometheus: metrics scraping and alert rules
- Tempo: trace storage via OTLP

## What Changed

The stack is project-agnostic now. It no longer contains a hardcoded scrape job
for one specific application. Prometheus discovers application metrics through
Docker labels on each app container.

The important behavior is:

- Grafana, Loki, Tempo, Prometheus, Alloy, dashboards, and alerts still run as
  before.
- Logs are still collected automatically from Docker through Alloy.
- Traces still go to Tempo through the OTLP endpoint from this stack.
- App metrics now require labels on the app container:
  `prometheus.scrape=true`, `prometheus.port`, and optionally
  `prometheus.path` / `prometheus.scheme`.
- The stack itself still uses `project="observability"` internally.
- Old app-specific metrics environment variables are no longer read. They can
  stay in an existing `.env`, but they do nothing and should be removed when you
  clean up the file.

## Upgrade Existing Install

Use this when the stack is already deployed and you want to update it to this
generic version.

1. Pull or deploy this repository version in the existing Dokploy project.

2. Keep your current `.env`, but compare it with `.env.example` and make sure
   these values are still set for your real domain and services:

   ```env
   GRAFANA_DOMAIN=grafana.example.com
   GRAFANA_ROOT_URL=https://grafana.example.com
   GRAFANA_ADMIN_PASSWORD=use-a-real-password
   LOKI_URL=http://loki:3100
   LOKI_PUSH_URL=http://loki:3100/loki/api/v1/push
   TEMPO_URL=http://tempo:3200
   TEMPO_OTLP_HTTP_URL=http://tempo:4318
   PROMETHEUS_URL=http://prometheus:9090
   ```

3. Remove old app-specific metrics variables from `.env` when you clean it up.
   They are ignored by the new Compose file.

4. Make sure every app that should send logs, metrics, or traces is attached to
   `dokploy-network`.

5. Add the labels from `Connect Another Dokploy Project` to each app service
   that should expose Prometheus metrics.

6. Redeploy the observability stack on the VPS:

   ```bash
   docker compose up -d
   ```

7. Redeploy the app projects too, so Docker recreates the containers with the
   new labels.

8. Verify labels and targets:

   ```bash
   docker inspect <container> | jq '.[0].Config.Labels'
   docker compose logs prometheus
   docker compose logs alloy
   ```

## URLs From `.env`

All public and internal URLs are configured in `.env`. The defaults in
`.env.example` work for containers attached to `dokploy-network`.

Important values:

- `GRAFANA_ROOT_URL`: public Grafana URL
- `GRAFANA_HEALTH_URL`: internal Grafana healthcheck URL
- `LOKI_URL`: internal Loki query URL for Grafana
- `LOKI_PUSH_URL`: internal Loki push URL for Alloy
- `LOKI_READY_URL`: internal Loki healthcheck URL
- `TEMPO_URL`: internal Tempo API URL for Grafana
- `TEMPO_OTLP_HTTP_URL`: internal OTLP HTTP URL for apps
- `PROMETHEUS_URL`: internal Prometheus URL for Grafana
- `PROMETHEUS_HEALTH_URL`: internal Prometheus healthcheck URL
- `*_METRICS_TARGET`: Prometheus scrape targets without protocol

## Setup

Local validation on macOS can be done with Podman:

```bash
podman compose --env-file .env.example config
```

Deploy and operate the stack on the VPS with Docker.

1. Create the shared Docker network on the VPS:

   ```bash
   docker network create dokploy-network || true
   ```

2. Create a Dokploy project, for example `observability`.

3. Create a Compose service and connect this repository.

4. In Dokploy, add the public domain only to the `grafana` service:

   - Domain: `grafana.example.com`
   - Service: `grafana`
   - Container port: `3000`
   - HTTPS enabled

   Do not attach a public domain to `loki`, `tempo`, `prometheus`, or `alloy`.
   Loki uses port `3100` internally; Grafana uses port `3000`.

5. Copy the environment example and set a real admin password:

   ```bash
   cp .env.example .env
   ```

   Required values:

   ```env
   GRAFANA_DOMAIN=grafana.example.com
   GRAFANA_ROOT_URL=https://grafana.example.com
   GRAFANA_HEALTH_URL=http://localhost:3000/api/health
   GRAFANA_ADMIN_USER=admin
   GRAFANA_ADMIN_PASSWORD=use-a-real-password
   LOKI_URL=http://loki:3100
   LOKI_PUSH_URL=http://loki:3100/loki/api/v1/push
   TEMPO_URL=http://tempo:3200
   TEMPO_OTLP_HTTP_URL=http://tempo:4318
   PROMETHEUS_URL=http://prometheus:9090
   LOKI_RETENTION=2160h
   TEMPO_RETENTION=2160h
   PROMETHEUS_RETENTION=2160h
   ```

6. Deploy the stack.

7. Check the services on the VPS:

   ```bash
   docker compose ps
   docker compose logs alloy
   docker compose logs loki
   docker compose logs prometheus
   ```

8. Open Grafana at the URL configured in `GRAFANA_ROOT_URL`.

9. Verify in Grafana:

   - Datasource `Loki` works.
   - Datasource `Prometheus` works.
   - Datasource `Tempo` works.
   - Folder `Observability` contains dashboards.
   - Alert rules are visible.

## Fresh Reset

If you already deployed the old stack and want to start clean, remove the old
containers and volumes before deploying this version:

```bash
docker compose down -v
docker network create dokploy-network || true
docker compose up -d
```

`docker compose down -v` deletes Loki, Tempo, Prometheus, Grafana, and Alloy
data for this Compose project. Only run it when you really want a fresh setup.

## Retention

The defaults keep data for 90 days:

```env
LOKI_RETENTION=2160h
TEMPO_RETENTION=2160h
PROMETHEUS_RETENTION=2160h
```

Examples:

- 30 days: `720h`
- 90 days: `2160h`
- 180 days: `4320h`

Change the values in `.env` and redeploy the stack. Lowering retention deletes
older data asynchronously. Deleted data cannot be restored unless you have a
backup of the Docker volumes.

## Connect Another Dokploy Project

Each project can stay in its own Compose file. It only needs the shared network
and observability labels. Logs are collected automatically from Docker. Metrics
are scraped from containers with `prometheus.scrape=true`; you do not need to add
a static Prometheus job for each app.

```yaml
networks:
  dokploy-network:
    external: true

services:
  api:
    networks:
      - dokploy-network
    labels:
      observability.project: "example-project"
      observability.app: "example-api"
      observability.environment: "production"
      prometheus.scrape: "true"
      prometheus.port: "3000"
      prometheus.path: "/metrics"
    environment:
      OTEL_SERVICE_NAME: "example-api"
      OTEL_EXPORTER_OTLP_ENDPOINT: "${TEMPO_OTLP_HTTP_URL}"
      OTEL_EXPORTER_OTLP_PROTOCOL: "http/protobuf"
      OTEL_RESOURCE_ATTRIBUTES: "project=example-project,deployment.environment=production"
```

For metrics, the app must expose `GET /metrics` on the configured port inside
the container. `prometheus.path` is optional and defaults to `/metrics`;
`prometheus.scheme` is optional and defaults to `http`.

For app projects in separate Compose files, copy the required endpoint values
from this stack's `.env` into that project's `.env`, especially
`TEMPO_OTLP_HTTP_URL`.

## Logging Convention

Production logs should be structured JSON written to `stdout` and `stderr`.
Alloy reads Docker logs from the host Docker socket, so apps do not need to know
the Loki URL.

Good stable fields:

```json
{
  "level": "info",
  "msg": "request completed",
  "project": "example-project",
  "app": "example-api",
  "environment": "production",
  "module": "http",
  "request_id": "req_123",
  "trace_id": "abc",
  "method": "GET",
  "path": "/api/orders",
  "status": 200,
  "duration_ms": 42
}
```

Use these as Loki labels:

- `project`
- `app`
- `environment`
- `service`
- `container`
- `level`

Do not use these as Loki labels, because they create too many unique streams:

- `request_id`
- `trace_id`
- `user_id`
- `order_id`
- full URLs with IDs
- IP addresses
- error messages

Those values should stay inside the JSON log body or structured metadata, where
you can still search for them in Grafana.

## Node.js Logging With Pino

Use `pino` for production JSON logs. Use `pino-pretty` only locally.

```js
import pino from "pino";

const isProduction = process.env.NODE_ENV === "production";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  base: {
    project: process.env.OBSERVABILITY_PROJECT,
    app: process.env.OTEL_SERVICE_NAME,
    environment: process.env.OBSERVABILITY_ENVIRONMENT ?? process.env.NODE_ENV,
  },
  formatters: {
    level(label) {
      return { level: label };
    },
  },
  redact: [
    "password",
    "token",
    "access_token",
    "refresh_token",
    "authorization",
    "req.headers.authorization",
    "*.secret",
  ],
  ...(!isProduction && process.stdout.isTTY
    ? {
        transport: {
          target: "pino-pretty",
          options: {
            colorize: true,
            translateTime: "SYS:standard",
            ignore: "pid,hostname",
          },
        },
      }
    : {}),
});

logger.info(
  {
    module: "http",
    request_id: "req_123",
    method: "GET",
    path: "/health",
    status: 200,
    duration_ms: 12,
  },
  "request completed",
);
```

Important: Pino's default `level` is numeric. The `formatters.level` block above
turns it into `info`, `warn`, `error`, and so on, which is nicer for Loki
queries.

## Useful LogQL Queries

All logs for one app:

```logql
{project="example-project", app="example-api"}
```

Errors:

```logql
{environment="production", level=~"error|fatal|50|60"}
```

One request:

```logql
{project="example-project", app="example-api"} | json | request_id="req_123"
```

One trace ID:

```logql
{project="example-project", app="example-api"} | json | trace_id="abc"
```

## Troubleshooting

No logs for an app:

1. Check that the app logs to `stdout` or `stderr`.
2. Check that the container is running on the same Docker host as Alloy.
3. Check `docker compose logs alloy` in the observability project.
4. Check that labels are present:

   ```bash
   docker inspect <container> | jq '.[0].Config.Labels'
   ```

No Prometheus metrics:

1. The app must have `prometheus.scrape=true`.
2. The app must be attached to `dokploy-network`.
3. `prometheus.port` must be the internal container port, not the public Traefik
   port.
4. The endpoint from inside the Docker network must work.

No traces:

1. The app must be attached to `dokploy-network`.
2. Use `OTEL_EXPORTER_OTLP_ENDPOINT=${TEMPO_OTLP_HTTP_URL}`.
3. Use `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`.
4. Set `OTEL_SERVICE_NAME` to the app name you want to see in Tempo.

Too many redirects on the Grafana domain:

1. In Dokploy, remove the domain from `loki` if it was added there.
2. Add the domain only to service `grafana` with container port `3000`.
3. Make sure `.env` matches the public URL:

   ```env
   GRAFANA_DOMAIN=grafana.example.com
   GRAFANA_ROOT_URL=https://grafana.example.com
   ```

4. Redeploy the stack.
5. If the domain is behind Cloudflare, use SSL/TLS mode `Full` or `Full strict`,
   not `Flexible`.
6. Clear browser cookies for the domain after fixing the route.
