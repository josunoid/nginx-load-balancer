# Deploy Stream Monitoring

Ini adalah dokumentasi tambahan jika ingin monitoring stream visionaire.

## Membuat folder baru
Buat folder baru bernama `stream-monitor` di dalam folder v4-centralized-monitoring (folder dimana grafana - prometheus berada), isi direktori akan seperti ini:
```
host@hostname:/opt/v4-centralized-monitoring/monitor-cctv$ ls -al
total 28
drwxr-xr-x 2 root root 4096 Nov  4 12:56 .
drwxr-xr-x 9 root root 4096 Nov  4 12:51 ..
-rw-r--r-- 1 root root  453 Nov  4 12:54 docker-compose.yml
-rw-r--r-- 1 root root  445 Nov  4 12:52 Dockerfile
-rw-r--r-- 1 root root   43 Nov  4 12:53 requirements.txt
-rw-r--r-- 1 root root 4332 Nov  4 13:17 stream_monitor.py
```

## Membuat file baru
### requirement.txt
```
prometheus-client==0.20.0
requests==2.31.0
```
### stream_monitor.py
```
import requests
from prometheus_client import CollectorRegistry, Gauge, generate_latest
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer
import threading
import time

API_BASE_URLS = [
    "http://10.0.28.11:4004",
    "http://10.10.2.19:4004",
    "http://10.10.104.29:4004",
    "http://10.5.19.19:4004",
    "http://10.2.62.19:4004",
    "http://10.5.31.19:4004",
    "http://10.5.30.19:4004",
    "http://10.5.35.19:4004",
]

EXPORTER_PORT = 9898
UPDATE_INTERVAL = 60  # detik

latest_registry = CollectorRegistry(auto_describe=False)
registry_lock = threading.Lock()


def fetch_metrics():
    """Ambil data dari semua server API dan perbarui metrics registry."""
    global latest_registry
    local_registry = CollectorRegistry(auto_describe=False)

    STREAM_STATUS = Gauge(
        'camera_stream_status_up',
        'Status aktif stream kamera (1=RUNNING, 0=DOWN/Lainnya)',
        ['stream_id', 'stream_name', 'node_num', 'state_text', 'source_host'],
        registry=local_registry
    )

    for base_url in API_BASE_URLS:
        source_host = base_url.split('//')[1].split(':')[0]
        try:
            streams_response = requests.get(f"{base_url}/streams", timeout=10)
            streams_response.raise_for_status()
            streams_data = streams_response.json()
        except requests.exceptions.RequestException as e:
            print(f"[WARN] {source_host} gagal ambil daftar streams: {e}")
            continue

        for stream in streams_data.get('streams', []):
            stream_id = stream.get('stream_id')
            node_num = stream.get('stream_node_num')
            stream_name = stream.get('stream_name', 'UNKNOWN_STREAM')
            if not stream_id or node_num is None:
                continue

            detail_url = f"{base_url}/streams/{node_num}/{stream_id}"
            labels = {
                'stream_id': stream_id,
                'stream_name': stream_name,
                'node_num': str(node_num),
                'source_host': source_host,
            }

            try:
                detail_response = requests.get(detail_url, timeout=10)
                detail_response.raise_for_status()
                detail_data = detail_response.json()
                state_text = detail_data.get('stream_stats', {}).get('state', 'UNKNOWN')
                if state_text == 'UNKNOWN' and detail_data.get('status') and detail_data['status'].get('pipelines'):
                    state_text = detail_data['status']['pipelines'][0].get('state', 'UNKNOWN')

                is_active = detail_data.get('active', False)
                metric_value = 1 if state_text == 'RUNNING' and is_active else 0
                labels['state_text'] = state_text
                STREAM_STATUS.labels(**labels).set(metric_value)
            except requests.exceptions.RequestException:
                labels['state_text'] = 'API_ERROR'
                STREAM_STATUS.labels(**labels).set(0)

    with registry_lock:
        latest_registry = local_registry
    print(f"[{time.strftime('%H:%M:%S')}] Metrics updated successfully.")


def background_updater():
    """Thread background untuk update metrics tiap interval."""
    while True:
        try:
            fetch_metrics()
        except Exception as e:
            print(f"[CRITICAL] Error update metrics: {e}")
        time.sleep(UPDATE_INTERVAL)


class PrometheusHandler(BaseHTTPRequestHandler):
    """Handler HTTP non-blocking untuk Prometheus scrape."""
    def do_GET(self):
        if self.path == '/metrics':
            with registry_lock:
                output = generate_latest(latest_registry)
            self.send_response(200)
            self.send_header('Content-Type', 'text/plain; version=0.0.4; charset=utf-8')
            self.end_headers()
            try:
                self.wfile.write(output)
            except BrokenPipeError:
                print("[WARN] Client closed connection early.")
        else:
            self.send_response(404)
            self.end_headers()


def run_exporter():
    print(f"Starting stream exporter on port {EXPORTER_PORT}...")
    server = ThreadingHTTPServer(('', EXPORTER_PORT), PrometheusHandler)
    threading.Thread(target=background_updater, daemon=True).start()
    server.serve_forever()


if __name__ == '__main__':
    run_exporter()
```
### Dockerfile
```
# --- Base image minimal ---
FROM python:3.10-slim

# Non-interaktif mode
ENV PYTHONUNBUFFERED=1

# Buat direktori kerja
WORKDIR /app

# Copy requirements dulu untuk caching layer
COPY requirements.txt .

# Install dependencies
RUN pip install --no-cache-dir -r requirements.txt

# Copy source code exporter
COPY stream_monitor.py .

# Expose port untuk Prometheus scrape
EXPOSE 9898

# Jalankan script Python
CMD ["python", "stream_monitor.py"]
```
### docker-compose.yml
```
version: "3.8"

services:
  stream-monitor:
    build: .
    container_name: stream-cctv-monitor
    restart: always
    ports:
      - "9898:9898"
    environment:
      - TZ=Asia/Jakarta
    healthcheck:
      test: ["CMD", "curl", "-f", "http://10.0.28.11:9898/metrics"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 15s
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "5"
```
Setelah semua file sudah ada, jalankan dengan command 
```
docker-compose down
docker-compose build --no-cache
docker-compose up -d
``` 
Anda akan membutuhkan internet untuk proses ini, karena proses build membutuhkan depedencies file yang harus di download.

```
host@hostname:/opt/v4-centralized-monitoring/monitor-cctv$ docker-compose build --no-cache
Building stream-monitor
[+] Building 6.8s (11/11) FINISHED                                                                                                               docker:default
 => [internal] load build definition from Dockerfile                                                                                                       0.0s
 => => transferring dockerfile: 484B                                                                                                                       0.0s
 => [internal] load .dockerignore                                                                                                                          0.0s
 => => transferring context: 2B                                                                                                                            0.0s
 => [internal] load metadata for docker.io/library/python:3.10-slim                                                                                        1.9s
 => [auth] library/python:pull token for registry-1.docker.io                                                                                              0.0s
 => [internal] load build context                                                                                                                          0.0s
 => => transferring context: 4.42kB                                                                                                                        0.0s
 => [1/5] FROM docker.io/library/python:3.10-slim@sha256:e0c4fae70d550834a40f6c3e0326e02cfe239c2351d922e1fb1577a3c6ebde02                                  0.0s
 => CACHED [2/5] WORKDIR /app                                                                                                                              0.0s
 => [3/5] COPY requirements.txt .                                                                                                                          0.0s
 => [4/5] RUN pip install --no-cache-dir -r requirements.txt                                                                                               4.5s
 => [5/5] COPY stream_monitor.py .                                                                                                                         0.0s
 => exporting to image                                                                                                                                     0.3s
 => => exporting layers                                                                                                                                    0.3s
 => => writing image sha256:c9e002608c62d251fd89d42fddc238e953d7a1d8382320b7b0ae6467297c3c7f                                                               0.0s
 => => naming to docker.io/library/monitor-cctv_stream-monitor 
```
## Konfigurasi Prometheus
Pada file `prometheus.yml` (di server Prometheus), tambahkan job baru.
```
  - job_name: 'stream-cctv-monitor'
    # Ganti dengan IP server tempat Anda menjalankan stream_monitor.py
    static_configs:
      - targets: ['10.0.28.11:9898']
    scrape_interval: 60s
    scrape_timeout: 45s
```

## Muat Ulang Prometheus dan Verifikasi
Muat ulang konfigurasi Prometheus.
Pergi ke Prometheus UI (/targets) dan pastikan custom-stream-exporter berstatus UP dan di-scrape dengan interval 60 detik.


>[!NOTE]
> For your information

Monitoring ini adalah membaca file jason dari API yang disediakan oleh visionaire
1. http://192.168.103.46:4004/streams : untuk mendapatkan semua stream, dan mem-parsing menjadi bentuk API untuk mendapatkan detail stream (API no. 2)
2. http://192.168.103.46:4004/streams/0/c73a1992f67961d1 : API untuk mendapatkan stream detail, yang mana Response dari API ini yang akan dijadikan acuan monitoring. `"state": "RUNNING"` ini lah yang menjadi acuan bahwa stream UP atau DOWN
   ```
   ...
     "stream_stats": {
    "fps": 15,
    "frame_height": 646,
    "frame_width": 1146,
    "last_error_msg": "Stream is running..",
    "state": "RUNNING"
   ```


Contoh dashboard: [camera-stream-monitoring.json](https://github.com/josunoid/nginx-load-balancer/blob/7c00dfe932b4fe90e04047fb9802b0dd679c1ff1/camera-stream-monitoring.json)