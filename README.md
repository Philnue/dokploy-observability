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

## Internal URLs

These URLs work from containers attached to `dokploy-network`:

- Loki: `http://loki:3100`
- Tempo OTLP HTTP: `http://tempo:4318`
- Tempo OTLP gRPC: `http://tempo:4317`
- Tempo API: `http://tempo:3200`
- Prometheus: `http://prometheus:9090`

## Setup

1. Create the shared Docker network on the VPS:

   ```bash
   docker network create dokploy-network || true
   ```

2. Create a Dokploy project, for example `observability`.

3. Create a Compose service and connect this repository.

4. Copy the environment example and set a real admin password:

   ```bash
   cp .env.example .env
   ```

   Required values:

   ```env
   GRAFANA_DOMAIN=grafana.nutline.xyz
   GRAFANA_ROOT_URL=https://grafana.nutline.xyz
   GRAFANA_ADMIN_USER=admin
   GRAFANA_ADMIN_PASSWORD=use-a-real-password
   LOKI_RETENTION=2160h
   TEMPO_RETENTION=2160h
   PROMETHEUS_RETENTION=2160h
   ```

5. Deploy the stack.

6. Check the services on the VPS:

   ```bash
   docker compose ps
   docker compose logs alloy
   docker compose logs loki
   docker compose logs prometheus
   ```

7. Open Grafana at `https://grafana.nutline.xyz`.

8. Verify in Grafana:

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
and observability labels.

```yaml
networks:
  dokploy-network:
    external: true

services:
  api:
    networks:
      - dokploy-network
    labels:
      observability.project: "kundenportal"
      observability.app: "motovers-api"
      observability.environment: "production"
      prometheus.scrape: "true"
      prometheus.port: "3000"
      prometheus.path: "/metrics"
    environment:
      OTEL_SERVICE_NAME: "motovers-api"
      OTEL_EXPORTER_OTLP_ENDPOINT: "http://tempo:4318"
      OTEL_EXPORTER_OTLP_PROTOCOL: "http/protobuf"
      OTEL_RESOURCE_ATTRIBUTES: "project=kundenportal,deployment.environment=production"
```

For metrics, the app must expose `GET /metrics` on the configured port inside
the container.

## Logging Convention

Production logs should be structured JSON written to `stdout` and `stderr`.
Alloy reads Docker logs from the host Docker socket, so apps do not need to know
the Loki URL.

Good stable fields:

```json
{
  "level": "info",
  "msg": "request completed",
  "project": "kundenportal",
  "app": "motovers-api",
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
{project="kundenportal", app="motovers-api"}
```

Errors:

```logql
{environment="production", level=~"error|fatal|50|60"}
```

One request:

```logql
{project="kundenportal", app="motovers-api"} | json | request_id="req_123"
```

One trace ID:

```logql
{project="kundenportal", app="motovers-api"} | json | trace_id="abc"
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
2. Use `OTEL_EXPORTER_OTLP_ENDPOINT=http://tempo:4318`.
3. Use `OTEL_EXPORTER_OTLP_PROTOCOL=http/protobuf`.
4. Set `OTEL_SERVICE_NAME` to the app name you want to see in Tempo.
