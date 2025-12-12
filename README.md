# Dokploy Observability Stack (Grafana + Loki + Promtail + Tempo)

## Deploy in Dokploy
- Create a Project: `observability`
- Create a Compose Service
- Connect this GitHub repo
- Deploy

## Access
- Grafana is exposed via Dokploy domain routing (recommended)
- Internal service URLs (from any container on the observability network):
  - Loki:  http://loki:3100
  - Tempo: http://tempo:4318/v1/traces  (OTLP HTTP)
  - Tempo API: http://tempo:3200

## Connect other projects
In your app compose add:

networks:
  observability:
    external: true

Then attach services:
  networks:
    - observability

App env:
  LOKI_URL=http://loki:3100
  TEMPO_URL=http://tempo:4318/v1/traces
  OTEL_SERVICE_NAME=your-service-name