# Overview Load Balancing
Load balance HTTP traffic across web or application server groups, with several algorithms and advanced features like slow-start and session persistence.
Load balancing across multiple application instances is a commonly used technique for optimizing resource utilization, maximizing throughput, reducing latency, and ensuring fault‑tolerant configurations[^1].

## Proxying HTTP Traffic to a Group of Servers
To start using NGINX Open Source to load balance HTTP traffic to a group of servers, first you need to define the group with the upstream directive. The directive is placed in the http context.
Servers in the group are configured using the server directive (not to be confused with the server block that defines a virtual server running on NGINX). 
For example, the following configuration defines a group named backend and consists of three server configurations (which may resolve in more than three actual servers):
``` console
upstream api_servers {
    # Load-Balancing Method        
    #list service
    server 192.168.100.251:4010 weight=10;
    server 192.168.100.255:4090;
    server backend1.example.com weight=5;
    server backend2.example.com;
}
```
## Setting and Configuration Nginx Load Balancer
Create a folder to save file configuration, example `sudo mkdir load-balancer`
Move to folder `load-balancer` and create file `docker-compose.yml`
fill file with below command
``` console
version : "2.2"
services:

  nginx:
    image: nginx:latest
    container_name: nginx-loadbalancer
    volumes:
      - ./nginx:/etc/nginx/ ##configuration for nginx load balancer
      - ./log/:/var/log/nginx/ #configuration for nginx exporter
    ports:
      - "4005:4005"
    restart: always
```
in folder `load-balancer`, create a new folder with name `nginx`, and create new file name `nginx.conf`
``` console
# you must set worker processes based on your CPU cores, nginx does not benefit from setting more than that
worker_processes auto; #some last versions calculate it automatically

# number of file descriptors used for nginx
# the limit for the maximum FDs on the server is usually set by the OS.
# if you don't set FD's then OS settings will be used which is by default 2000
worker_rlimit_nofile 100000;

# only log critical errors
error_log /var/log/nginx/error.log crit;

events {
    # determines how much clients will be served per worker
    # max clients = worker_connections * worker_processes
    # max clients is also limited by the number of socket connections available on the system (~64k)
    worker_connections 4000;

    # optimized to serve many clients with each thread, essential for linux
    use epoll;

    # accept as many connections as possible, may flood worker connections if set too low
    multi_accept on;
    }

http {
    # Config for nginxlog-exporter
    #include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # Enable gzip encryption
    gzip on;


    ## Enable Metric Log Formating from martin-helmich/prometheus-nginxlog-exporter
    log_format main '$remote_addr - $remote_user [$time_local] '
                         '"$request" $status $body_bytes_sent '
                         '"$http_referer" "$http_user_agent" "$http_x_forwarded_for" '
                         'rt="$request_time" uct="$upstream_connect_time" uht="$upstream_header_time" urt="$upstream_response_time"';


    # cache informations about FDs, frequently accessed files
    # can boost performance, but you need to test those values
    open_file_cache max=200000 inactive=20s;
    open_file_cache_valid 30s;
    open_file_cache_min_uses 2;
    open_file_cache_errors on;

    # hit rate limiter 
    #limit_req_zone $binary_remote_addr zone=limitbyaddr:10m rate=100r/s;
    #limit_req_status 429;
    #limit_conn_status 429;

    # allow the server to close connection on non responding client, this will free up memory
    reset_timedout_connection on;

    # List of application servers
    upstream api_servers {
        # Load-Balancing Method - default Round Robin        
        #list service
        #server 10.0.28.10:4077 weight=30;
        server 10.0.28.11:4007 weight=9;
        server 10.0.28.12:6644;
        server 10.0.28.10:4066;
        #server 10.0.28.11:4021;
    }

    # Configuration for the server
    server {
        # Nginxlog-exporter read log to parsed
        access_log /var/log/nginx/access.log main;
        error_log /var/log/nginx/error.log;
        # Running port
        listen [::]:4005;
        listen 4005;
        #limit_req zone=limitbyaddr burst=150 nodelay;

        # Proxying the connections
        location / {
            stub_status on;
            proxy_pass http://api_servers;
        }
    }
}
```
# Additional Information
## Choosing a Load-Balancing Method [^2]
NGINX Open Source supports four load‑balancing methods
* Round Robin – Requests are distributed evenly across the servers, with server weights taken into consideration. This method is used by default (there is no directive for enabling it):
  ``` console
  upstream backend {
   # no load balancing method is specified for Round Robin
   server backend1.example.com;
   server backend2.example.com;
  }
  ```
* Least Connections – A request is sent to the server with the least number of active connections, again with server weights taken into consideration:
  ``` console
  upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
  }
  ```
* IP Hash – The server to which a request is sent is determined from the client IP address. In this case, either the first three octets of the IPv4 address or the whole IPv6 address are used to calculate the hash value. The method guarantees that requests from the same address get to the same server unless it is not available.
  ``` console
  upstream backend {
    ip_hash;
    server backend1.example.com;
    server backend2.example.com;
  }
  ```
  If one of the servers needs to be temporarily removed from the load‑balancing rotation, it can be marked with the down parameter in order to preserve the current hashing of client IP addresses. Requests that were to be processed by this server are automatically sent to the next server in the group:
  ``` console
  upstream backend {
    server backend1.example.com;
    server backend2.example.com;
    server backend3.example.com down;
  }
  ```
* Generic Hash – The server to which a request is sent is determined from a user‑defined key which can be a text string, variable, or a combination. For example, the key may be a paired source IP address and port, or a URI as in this example:
  ``` console
  upstream backend {
    hash $request_uri consistent;
    server backend1.example.com;
    server backend2.example.com;
  }
  ```
  The optional consistent parameter to the hash directive enables ketama consistent‑hash load balancing. Requests are evenly distributed across all upstream servers based on the user‑defined hashed key value. If an upstream server is added to or removed from an upstream group, only a few keys are remapped which minimizes cache misses in the case of load‑balancing cache servers or other applications that accumulate state.


Reference:
[^1]: [Nginx Load Balancing](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/)
[^2]: [Choosing a Load-Balancing Method](https://docs.nginx.com/nginx/admin-guide/load-balancer/http-load-balancer/#choosing-a-load-balancing-method)
