# Deploy Nginx Log Exporter

This deployment process is an integral part of [Deploy Grafana Prometheus](./deploy-grafana-prometheus.md)

In folder `v4-centralized-monitoring` create new file called `exporter.yml`
``` console
listen:
  port: 9114
  address: "0.0.0.0"
  metrics_endpoint: "/metrics"

consul:
  enable: false

namespaces:
  - name: cluster_soetta #you can change this
    format: "$remote_addr - $remote_user [$time_local] \"$request\" $status $body_bytes_sent \"$http_referer\" \"$http_user_agent\" \"$http_x_forwarded_for\" rt=\"$request_time\" uct=\"$upstream_connect_time\" uht=\"$upstream_header_time\" urt=\"$upstream_response_time\" "
    source:
      files:
        - /var/log/nginx/access.log
    labels:
      app: "cluster_soetta" #you can change this
      environment: "imigrasi" #you can change this
```
change `app` and `environment` according to the project name or server name.

Edit file `docker-compose.yml` at folder `v4-centralized-monitoring`, add command below:
``` console
  nginxlog-exporter:
    image: quay.io/martinhelmich/prometheus-nginxlog-exporter
    command: -config-file /etc/exporter.yml
    container_name: nginxlog-exporter
    ports:
      - 9114:9114
    volumes:
      - ./exporter.yml:/etc/exporter.yml
      - /etc/fremisn/nginx/log/:/var/log/nginx/ #Change this according to the directory where the load balancer is installed 
    restart: unless-stopped
```
> [!WARNING]
> Don't forget to change this section `#Change this according to the directory where the load balancer is installed`
> 
> Example:
> 
> My nginx folder is in `/opt/load-balancer/`, so my volume is `- /opt/load-balancer/log/:/var/log/nginx/`
