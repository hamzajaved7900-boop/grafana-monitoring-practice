# Grafana Monitoring Practice

Monitoring stack built with Docker Compose: Prometheus, Node Exporter,
Grafana, Loki and Alloy.

## What it does
- Node Exporter exposes server metrics (CPU, RAM) on port 9100
- Prometheus scrapes them every 15 seconds
- Alloy collects container logs and sends them to Loki
- Grafana shows dashboards and sends alerts (Slack)

## Run
    docker compose up -d

- Grafana: http://localhost:3000 (default login admin / admin, change it)
- Prometheus: http://localhost:9090

## Files
- docker-compose.yml: all services
- prometheus.yml: scrape targets
- config.alloy: log collection pipeline
- provisioning/datasources/datasources.yml: Grafana data sources as code

## Note
Secrets (Slack webhook URL, passwords) are not stored in this repo.
