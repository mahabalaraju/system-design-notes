# Load Balancer

## What is a Load Balancer?

A **Load Balancer (LB)** is a system that **distributes incoming traffic across multiple backend servers** so that no single server gets overwhelmed.

In simple words:

> Load balancing is the process of distributing each client request across a pool of available servers to improve performance, availability, and reliability.

A load balancer acts like a **traffic controller** between the client and the servers.

## Simple Flow

[ Client Requests ]
              ↓
      [  Load Balancer  ]  (e.g., Nginx, HAProxy, AWS ALB)
      ↙       ↓       ↘
 [Server 1] [Server 2] [Server 3]

```text
Client Requests
       ↓
 [ Load Balancer ]  (e.g., Nginx, HAProxy, AWS ALB)
   ↙    ↓    ↘
Server 1 Server 2 Server 3

**Real‑world example** – E‑commerce during Black Friday: a load balancer distributes millions of requests across hundreds of backend servers, preventing any single server from melting down.

Common Load‑Balancing Algorithms:

| Algorithm            | Behavior                                                                              |
| -------------------- | ------------------------------------------------------------------------------------- |
| Round Robin          | Requests go to each server in order: A, B, C, A, B, C…nginx+1                         |
| Weighted Round Robin | Servers get traffic proportional to their “weight” (CPU/RAM capacity).theserverside+1 |
| Least Connections    | Request goes to the server with the fewest active connections.linode                  |
| IP Hash              | Same client IP always goes to the same server (session stickiness).linode             |

Why Do We Need It?

1) High Availability
If one server crashes, the Load Balancer redirects traffic to the remaining healthy servers.

2) Scalability
You can add more backend servers during peak traffic without changing the client-side URL.

3) Performance
It prevents hotspotting, where one server is overloaded while others remain underutilized.

4) SSL Termination
The load balancer can handle HTTPS decryption, reducing the CPU load on backend application servers.

### Common tools used in production:

| Tool                    | Type               | Key Feature                                          |
| ----------------------- | ------------------ | ---------------------------------------------------- |
| **Nginx**               | Software (L7)      | Lightweight, configurable, great for HTTP/HTTPS      |
| **HAProxy**             | Software (L4 / L7) | High performance, TCP/HTTP support, health checks    |
| **AWS ELB (ALB / NLB)** | Cloud Managed      | Auto-scaling, fully managed, integrates with AWS     |
| **Traefik**             | Software           | Kubernetes-native, automatic service discovery       |
| **Envoy**               | Software           | Service mesh support (Istio), advanced observability |

Health Checks: The "Safety Net": prometeu
A Load Balancer is useless if it sends traffic to a dead server.

Passive: LB notices a connection timeout and stops sending traffic.

Active: LB pings a specific endpoint (e.g., GET /actuator/health in Spring Boot) every 5 seconds. If it doesn't get a 200 OK, the server is "marked down."


What is the difference between Layer 4 and Layer 7 Load Balancing?

Layer 4 (Transport): Faster. Routes based on IP and Port. Doesn't look at the data.
Layer 7 (Application): Smarter. Can route based on the URL (e.g., /api/users goes to Server A, while /api/orders goes to Server B).

