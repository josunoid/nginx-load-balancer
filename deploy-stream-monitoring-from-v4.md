# Deploy Stream Monitoring

Ini adalah dokumentasi tambahan jika ingin monitoring stream visionaire.

## Membuat folder baru
Buat folder baru bernama `stream-monitor` di dalam folder v4-centralized-monitoring (folder dimana grafana - prometheus berada), isi direktory akan seperti ini:
```
host@hostname:/opt/v4-centralized-monitoring/stream-monitor$ ls -al
total 28
drwxr-xr-x 2 root root 4096 Nov  3 10:54 .
drwxr-xr-x 8 root root 4096 Nov  3 10:33 ..
-rw-r--r-- 1 root root  516 Nov  3 10:52 docker-compose.yml
-rw-r--r-- 1 root root  513 Nov  3 10:53 Dockerfile
-rw-r--r-- 1 root root   27 Nov  3 10:52 requirements.txt
-rw-r--r-- 1 root root 5738 Nov  3 10:40 stream_monitor.py
```

## Membuat file baru
### requirement.txt
```
requests
prometheus_client
```
### stream_monitor.py
```
import requests
from prometheus_client import CollectorRegistry, Gauge, generate_latest
from http.server import BaseHTTPRequestHandler, HTTPServer
import threading
import json
import time

# --- KONFIGURASI API ---
API_BASE_URLS = [
    "http://10.0.28.11:4004", 
    "http://10.10.2.19:4004",
    "http://10.10.104.29:4004",
    "http://10.5.19.19:4004",
    "http://10.2.62.19:4004",
    "http://10.5.31.19:4004",
    "http://10.5.30.19:4004",
    "http://10.5.35.19:4004" 
]
# Port yang harus sama dengan konfigurasi Prometheus
EXPORTER_PORT = 9898 
# -----------------------

def fetch_metrics():
    """
    Mengambil data status dari semua API yang dikonfigurasi dan memperbarui metrik 
    menggunakan Registry baru untuk memastikan hanya metrik terbaru yang ada.
    """
    REGISTRY_LOCAL = CollectorRegistry(auto_describe=False)
    
    # Definisikan Gauge di dalam fungsi fetch_metrics()
    STREAM_STATUS = Gauge(
        'camera_stream_status_up', 
        'Status aktif stream kamera (1=RUNNING, 0=EXCEPTION/DOWN/Lainnya)',
        ['stream_id', 'stream_name', 'node_num', 'state_text', 'source_host'],
        registry=REGISTRY_LOCAL
    )
    
    print(f"[{time.strftime('%Y-%m-%d %H:%M:%S')}] Fetching new metrics from all configured servers...")
    
    for base_url in API_BASE_URLS:
        # Ekstrak host/port untuk digunakan sebagai label identitas
        source_host = "unknown"
        try:
            source_host_port = base_url.split('//')[1]
            source_host = source_host_port.split(':')[0]
        except IndexError:
            pass
            
        try:
            # A. Hit API pertama: Ambil daftar stream
            streams_response = requests.get(f"{base_url}/streams", timeout=10)
            streams_response.raise_for_status()
            streams_data = streams_response.json()
            
        except requests.exceptions.RequestException as e:
            print(f"ERROR: Gagal mengambil daftar streams dari {base_url}: {e}")
            continue 

        # B. Iterasi dan Hit API detail untuk setiap stream
        for stream in streams_data.get('streams', []):
            stream_id = stream.get('stream_id')
            node_num = stream.get('stream_node_num')
            stream_name = stream.get('stream_name', 'UNKNOWN_STREAM')
            
            if stream_id and node_num is not None:
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
                    
                    # C. Ekstrak Status 
                    state_text = detail_data.get('stream_stats', {}).get('state', 'UNKNOWN')
                    if state_text == 'UNKNOWN' and detail_data.get('status') and detail_data['status'].get('pipelines'):
                         state_text = detail_data['status']['pipelines'][0].get('state', 'UNKNOWN')
                    
                    # Konversi status ke metrik numerik (1 = UP, 0 = DOWN)
                    is_active = detail_data.get('active', False)
                    metric_value = 1 if state_text == 'RUNNING' and is_active is True else 0
                    
                    labels['state_text'] = state_text 
                    
                    # D. Set nilai metrik
                    STREAM_STATUS.labels(**labels).set(metric_value)
                    
                except requests.exceptions.RequestException as e:
                    print(f"WARNING: Gagal mengambil detail stream {stream_name} dari {source_host}: {e}")
                    
                    # Set status ke 0 jika terjadi kesalahan API
                    labels['state_text'] = 'API_ERROR'
                    STREAM_STATUS.labels(**labels).set(0)

    # Kembalikan registry lokal yang berisi metrik terbaru
    return REGISTRY_LOCAL


class PrometheusHandler(BaseHTTPRequestHandler):
    """Handler HTTP untuk melayani endpoint /metrics"""
    
    def do_GET(self):
        if self.path == '/metrics':
            try:
                # Dapatkan registry terbaru
                registry = fetch_metrics()
                
                self.send_response(200)
                self.send_header('Content-Type', 'text/plain; version=0.0.4; charset=utf-8')
                self.end_headers()
                self.wfile.write(generate_latest(registry))
            except Exception as e:
                print(f"CRITICAL ERROR during metric generation/response: {e}")
                self.send_response(500)
                self.send_header('Content-Type', 'text/plain; version=0.0.4; charset=utf-8')
                self.end_headers()
                self.wfile.write(f"Exporter Internal Error: {e}".encode('utf-8')) # Memberikan pesan error 500
        else:
            self.send_response(404)
            self.end_headers()

def run_exporter():
    """Menjalankan server HTTP"""
    server_address = ('', EXPORTER_PORT)
    try:
        httpd = HTTPServer(server_address, PrometheusHandler)
        print(f"Starting stream exporter on port {EXPORTER_PORT}...")
        httpd.serve_forever()
    except OSError as e:
        print(f"FATAL ERROR: Failed to bind to port {EXPORTER_PORT}. Error: {e}")

if __name__ == '__main__':
    run_exporter()
```
### Dockerfile
```
# Gunakan image Python 3.9 yang ringan sebagai basis
FROM python:3.9-slim

# Tetapkan direktori kerja di dalam container
WORKDIR /app

# Salin file requirements.txt untuk menginstal dependensi terlebih dahulu
COPY requirements.txt .

# Instal dependensi Python
RUN pip install --no-cache-dir -r requirements.txt

# Salin skrip aplikasi utama
COPY stream_monitor.py .

# Expose port yang digunakan oleh exporter (9898)
EXPOSE 9899

# Perintah default saat container dijalankan
CMD ["python3", "stream_monitor.py"]
```
### docker-compose.yml
```
version: '3.8'

services:
  # Definisi layanan untuk exporter metrik
  stream-exporter:
    # Perintah untuk membangun image dari Dockerfile di direktori saat ini
    build: 
      context: .
      dockerfile: Dockerfile
    
    # Nama container yang mudah diidentifikasi
    container_name: custom-stream-exporter
    
    # Menggunakan mode host network: Penting untuk mengakses IP 10.x.x.x
    network_mode: host
    
    # Atur agar container selalu restart kecuali dihentikan secara manual
    restart: always
```
Setelah semua file sudah ada, jalankan dengan command `docker-compose -up -d`. Anda akan membutuhkan internet untuk proses ini, karena proses build membutuhkan depedencies file yang harus di download.

```
host@hostname:/opt/v4-centralized-monitoring/stream-monitor$ docker-compose up -d
Building stream-exporter
[+] Building 14.8s (11/11) FINISHED                                                                                                              docker:default
 => [internal] load build definition from Dockerfile                                                                                                       0.0s
 => => transferring dockerfile: 552B                                                                                                                       0.0s
 => [internal] load .dockerignore                                                                                                                          0.0s
 => => transferring context: 2B                                                                                                                            0.0s
 => [internal] load metadata for docker.io/library/python:3.9-slim                                                                                         6.7s
 => [auth] library/python:pull token for registry-1.docker.io                                                                                              0.0s
 => [internal] load build context                                                                                                                          0.0s
 => => transferring context: 5.86kB                                                                                                                        0.0s
 => [1/5] FROM docker.io/library/python:3.9-slim@sha256:2d97f6910b16bd338d3060f261f53f144965f755599aab1acda1e13cf1731b1b                                   2.6s
 => => resolve docker.io/library/python:3.9-slim@sha256:2d97f6910b16bd338d3060f261f53f144965f755599aab1acda1e13cf1731b1b                                   0.0s
 => => sha256:b3ec39b36ae8c03a3e09854de4ec4aa08381dfed84a9daa075048c2e3df3881d 1.29MB / 1.29MB                                                             0.9s
 => => sha256:fc74430849022d13b0d44b8969a953f842f59c6e9d1a0c2c83d710affa286c08 13.88MB / 13.88MB                                                           0.9s
 => => sha256:ea56f685404adf81680322f152d2cfec62115b30dda481c2c450078315beb508 251B / 251B                                                                 0.5s
 => => sha256:2d97f6910b16bd338d3060f261f53f144965f755599aab1acda1e13cf1731b1b 10.36kB / 10.36kB                                                           0.0s
 => => sha256:dad5b29e3506c35e0fd222736f4d4ef25d21b219acdd73f7bb41d59996ca8e0d 1.74kB / 1.74kB                                                             0.0s
 => => sha256:085da638e1b8a449514c3fda83ff50a3bffae4418b050cfacd87e5722071f497 5.40kB / 5.40kB                                                             0.0s
 => => extracting sha256:b3ec39b36ae8c03a3e09854de4ec4aa08381dfed84a9daa075048c2e3df3881d                                                                  0.2s
 => => extracting sha256:fc74430849022d13b0d44b8969a953f842f59c6e9d1a0c2c83d710affa286c08                                                                  1.5s
 => => extracting sha256:ea56f685404adf81680322f152d2cfec62115b30dda481c2c450078315beb508                                                                  0.0s
 => [2/5] WORKDIR /app                                                                                                                                     0.1s
 => [3/5] COPY requirements.txt .                                                                                                                          0.0s
 => [4/5] RUN pip install --no-cache-dir -r requirements.txt                                                                                               5.1s
 => [5/5] COPY stream_monitor.py .                                                                                                                         0.0s
 => exporting to image                                                                                                                                     0.3s
 => => exporting layers                                                                                                                                    0.3s
 => => writing image sha256:46394b3e6b79003a3f39601485ef851b6d53ef2173e88f7624daf3b48a837ed8                                                               0.0s
 => => naming to docker.io/library/stream-monitor_stream-exporter                                                                                          0.0s
WARNING: Image for service stream-exporter was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating custom-stream-exporter ... done
```
## Konfigurasi Prometheus
Pada file `prometheus.yml` (di server Prometheus), tambahkan job baru.
```
  - job_name: 'custom-stream-exporter'
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