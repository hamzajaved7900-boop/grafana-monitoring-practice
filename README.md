# Grafana Monitoring Practice (Prometheus + Loki + Grafana)

A small monitoring lab you can run on your own laptop with one command.
It shows how companies watch their servers and applications:

- **Metrics** (numbers such as CPU % and RAM %) are stored in Prometheus
- **Logs** (text such as "ERROR database failed") are stored in Loki
- **Grafana** shows both on dashboards and sends alerts

No monitoring experience is needed. You only need Docker.

---

## 1. The big idea (in plain words)

Imagine a server running your application. You want to know:

1. Is the CPU or memory getting full?
2. Is the application writing errors?
3. Can someone tell me **before** users complain?

Monitoring answers these questions. This project builds a mini version of it.

| Tool | Simple explanation |
|------|--------------------|
| **Node Exporter** | Reads the server's CPU, RAM and disk numbers and publishes them on a web page |
| **Prometheus** | A database for numbers. It visits Node Exporter every 15 seconds and saves the numbers |
| **Loki** | A database for logs (text lines written by applications) |
| **Alloy** | A collector. It reads the logs of all containers and sends them to Loki |
| **Grafana** | The screen. It draws graphs from Prometheus and Loki and sends alerts |
| **log-generator** | A fake application that writes INFO and ERROR lines, so we always have logs to look at |

## 2. How the data flows

```
METRICS (numbers)
Node Exporter  <-- scraped every 15s --  Prometheus  <-- queried by --  Grafana

LOGS (text)
Containers --> Alloy --(push)--> Loki  <-- queried by --  Grafana
```

- Prometheus **pulls** metrics from Node Exporter.
- Grafana **pulls** (queries) data from Prometheus and Loki.
- Alloy **pushes** logs to Loki.

> Note: Node Exporter reports the machine it runs on. On Windows with WSL2
> that is the WSL2 Linux virtual machine. On a cloud server (for example
> AWS EC2) it reports that server.

## 3. What you need before starting

- **Docker** and the **Docker Compose plugin** (the command `docker compose`)
- About 2 GB of free RAM
- A web browser

Check your setup:

```
docker --version
docker compose version
```

If `docker compose` is not found on Ubuntu/WSL:

```
sudo apt update
sudo apt install -y docker-compose-v2
```

## 4. Files in this repository

| File | What it does |
|------|--------------|
| `docker-compose.yml` | The master plan. Lists all containers and starts them together |
| `prometheus.yml` | Tells Prometheus **where** to collect metrics and **how often** |
| `config.alloy` | Tells Alloy **which** container logs to read and **where** to send them (Loki) |
| `provisioning/datasources/datasources.yml` | Makes Grafana create its two data sources (Prometheus and Loki) automatically |

### prometheus.yml explained

```
global:
  scrape_interval: 15s            # collect data every 15 seconds
scrape_configs:
  - job_name: 'node'              # a name for this job
    static_configs:
      - targets: ['node-exporter:9100']   # where to collect from
```

`node-exporter` is the service name in `docker-compose.yml`. Containers in the
same Compose project can reach each other by service name.

### docker-compose.yml explained

- `image:` which ready-made program to download and run
- `ports: "3000:3000"` means `port on your laptop : port inside the container`
- `volumes:` connects a file or folder from your laptop to the container
- `grafana-data` is a named volume where Grafana keeps dashboards and alerts,
  so they are not lost when the container is recreated

### config.alloy explained

It is a small pipeline with four steps:

1. `discovery.docker`: find all running containers
2. `discovery.relabel`: turn the container name into a label called `container`
3. `loki.source.docker`: read the logs of those containers
4. `loki.write`: send the logs to Loki at `http://loki:3100/loki/api/v1/push`

## 5. Quick start

```
git clone <this-repository-url>
cd <repository-folder>
docker compose up -d
docker compose ps
```

`docker compose ps` should show all containers as **Up**.
The first start downloads images, so it can take a few minutes.

## 6. Open the tools

| Tool | Address | What to check |
|------|---------|---------------|
| Grafana | http://localhost:3000 | Login `admin` / `admin`, then set a new password |
| Prometheus | http://localhost:9090/targets | The `node` target must show **UP** |
| Node Exporter | http://localhost:9100/metrics | Raw numbers (CPU, memory) |
| Loki | http://localhost:3100/ready | Should print `ready` (wait a minute after start) |

In Grafana go to **Connections -> Data sources**. You should already see
`prometheus` and `loki`. Click **Save & test** on each one. Green means connected.

## 7. Try it yourself (first dashboard)

In Grafana create a new dashboard and add panels.

**CPU usage panel** (data source: `prometheus`, mode: Code)

```
100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[1m])) * 100)
```

**RAM usage panel**

```
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)
```

Set Unit to **Percent (0-100)** for both.

**Logs panel** (data source: `loki`, visualization: Logs)

```
{container=~".*log-generator.*"}
```

**Errors per minute panel** (data source: `loki`, visualization: Time series)

```
count_over_time({container=~".*log-generator.*"} |= "ERROR" [1m])
```

Tip: you can also import the ready-made dashboard **ID 1860** (Node Exporter Full)
from *Dashboards -> New -> Import*.

## 8. Try an alert

1. Go to **Alerting -> Alert rules -> New alert rule**
2. Use the CPU query above and set the condition **Is above 80**
3. Create a folder, an evaluation group (every `1m`) and a pending period of `1m`
4. Add a **contact point** (for example a Slack incoming webhook)
5. Create CPU load to test it (change `4` to your number of CPU cores):

```
for i in 1 2 3 4; do yes > /dev/null & done
```

6. Watch the alert change: **Normal -> Pending -> Firing**
7. Stop the load:

```
pkill yes
```

The alert goes back to **Normal** and a "Resolved" message is sent.

## 9. Stop and clean up

```
docker compose stop            # stop, keep everything
docker compose down            # remove containers, keep Grafana data volume
docker compose down -v         # remove containers AND volumes (deletes dashboards!)
```

## 10. Troubleshooting

| Problem | Fix |
|---------|-----|
| `unknown shorthand flag: 'd'` | Install the Compose plugin: `sudo apt install -y docker-compose-v2` |
| Prometheus target `node` is DOWN | Run `docker compose ps` and check that `node-exporter` is Up |
| Loki data source says "Unable to connect" | Wait one minute, then Save & test again. If you see `no such host`, use the container name or IP as the Loki URL (for example `http://<project>-loki-1:3100`) in `provisioning/datasources/datasources.yml`, then run `docker compose restart grafana` |
| Logs panel shows "No data" | Check that `log-generator` and `alloy` are Up, and widen the time range |
| Alert shows "No data" when the log-generator is stopped | Expected. A log query returns an empty result when there are no matching lines. Set "Alert state if no data" to Normal in the rule |
| Port already in use | Another program uses 3000, 3100, 9090 or 9100. Stop it or change the left side of the port mapping |

## 11. Important notes

- **What is saved:** Grafana keeps dashboards and alerts in the `grafana-data`
  volume. Data sources are recreated from `datasources.yml`.
- **What is not saved:** Prometheus metrics and Loki logs have no volume here,
  so they are lost when those containers are recreated. Production setups
  add storage for them.
- **Dashboards and alert rules are created by hand in Grafana** and are not
  stored in this repository (export them as JSON if you want a backup).
- **Security:** the default `admin` / `admin` login is for practice only.
  Never commit Slack webhook URLs, passwords or tokens to Git.

## 12. What I learned

- How Prometheus, Loki and Grafana work together
- The difference between metrics (numbers) and logs (text)
- Writing basic PromQL and LogQL queries
- Building dashboards and alert rules, and sending alerts to Slack
- Configuration as code with provisioning
- Why containers need persistent volumes

## 13. Next steps

- Run the same stack on an AWS EC2 instance
- Add Grafana dashboards and alert rules as code (provisioning or Terraform)
- Add the CloudWatch data source
- Run the stack on Kubernetes with Helm
