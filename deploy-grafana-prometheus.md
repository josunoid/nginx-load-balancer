## Overview
Grafana is a multi-platform open source analytics and interactive visualization web application. It provides charts, graphs, and alerts for the web when connected to supported data sources.

First, we must clone docker compose from ths repository
``` console
git clone https://github.com/tribuwana23/v4-centralized-monitoring.git
```
move to folder docker compose
``` console
cd v4-centralized-monitoring
```
> [!NOTE]
> There are 4 services in the docker-compose.yml file, namely grafana, prometheus, and pushprox-master. pushprox-master is only used for centralized monitoring.
## Grafana and Prometheus
next, move to folder prometheus and edit file `prometheus.yml`, change line `job_name` and `static_configs` adjust with your environment.
``` console
global:
  scrape_interval:     15s
  evaluation_interval: 15s

  external_labels:
      monitor: 'docker-host-alpha'
      
scrape_configs:
  - job_name: 'Server Master'
    scrape_interval: 10s
    static_configs:
      - targets: ['192.168.100.251:9090', '192.168.100.251:4004', '192.168.100.251:4010', '192.168.100.251:4006', '192.168.100.251:4007', '192.168.100.251:9835', '192.168.100.251:9100', '192.168.100.251:8085', '192.168.100.251:9114']
```
next, run container with docker-compose command
``` console
docker-compose up -d
```

## Exporter
move to `client` folder
``` console
cd client
```
> [!NOTE]
> There are 4 services in the docker-compose.yml file, namely node-exporter, cadvisor, nvidia-exporter, and pushprox-client. pushprox-client is only used for centralized monitoring.

next, run container with docker-compose command
``` console
docker-compose up -d
```
> [!TIP]
> don't forget to delete `proxy_url: http://localhost:80/` in the `prometheus/prometheus.yml` file if running on your own local server :+1:

**Reference dashboard template:**

Nvidia GPU Metrics:
https://grafana.com/grafana/dashboards/14574-nvidia-gpu-metrics

Node Exporter Full:
https://grafana.com/grafana/dashboards/12486-node-exporter-full

[FremisN - Nginx Exporter Dashboard](/fremisn-nginx-exporter.json)

[Log Metrics FRemis-N Dashboard](/log-metric-fremisn.json)
