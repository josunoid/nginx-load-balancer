## Overview Load Balancing
Load balance HTTP traffic across web or application server groups, with several algorithms and advanced features like slow-start and session persistence.
Load balancing across multiple application instances is a commonly used technique for optimizing resource utilization, maximizing throughput, reducing latency, and ensuring fault‑tolerant configurations

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
