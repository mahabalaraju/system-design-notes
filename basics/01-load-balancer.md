# Load Balancer

## What is a Load Balancer?

A **Load Balancer (LB)** is a system that **distributes incoming traffic across multiple backend servers** so that no single server gets overwhelmed.

In simple words:

> Load balancing is the process of distributing each client request across a pool of available servers to improve performance, availability, and reliability.

A load balancer acts like a **traffic controller** between the client and the servers.

## Simple Flow
## Simple Flow

```text
Client Requests
       ↓
 [ Load Balancer ]  (e.g., Nginx, HAProxy, AWS ALB)
   ↙    ↓    ↘
Server 1 Server 2 Server 3
```

**Real‑world example** – E‑commerce during Black Friday: a load balancer distributes millions of requests across hundreds of backend servers, preventing any single server from melting down.

Common Load‑Balancing Algorithms:

| Algorithm            | Behavior                                                                              |
| -------------------- | ------------------------------------------------------------------------------------- |
| Round Robin          | Requests go to each server in order: A, B, C, A, B, C…nginx+1                         |
| Weighted Round Robin | Servers get traffic proportional to their “weight” (CPU/RAM capacity).theserverside+1 |
| Least Connections    | Request goes to the server with the fewest active connections.linode                  |
| IP Hash              | Same client IP always goes to the same server (session stickiness).linode             |

**Why Do We Need It?**

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

## Health Checks: The "Safety Net"

A **Load Balancer** is useless if it keeps sending traffic to a **dead server**.

Health checks help the load balancer determine whether a backend server is **healthy** or **unhealthy**.

### 1) Passive Health Check
In passive health checks, the load balancer notices problems such as:

- connection timeout
- failed response
- unreachable backend server

If a server fails repeatedly, the load balancer **stops sending traffic** to it.

---

### 2) Active Health Check
In active health checks, the load balancer periodically sends a request to a specific health endpoint, such as:

```text
GET /actuator/health
```

If the server responds with:

```text
200 OK
```

then the server is considered **healthy**.

If it does not respond properly, the server is marked **down** and removed from traffic rotation.

---

### Example
A load balancer may check every **5 seconds** whether a backend server is alive.

If one server goes down:
- it is removed from the server pool
- traffic is redirected to healthy servers only

---

> **Note:** Health checks are often monitored using tools like **Prometheus** and **Grafana** in production environments.

---

## Layer 4 vs Layer 7 Load Balancing

### Layer 4 (Transport Layer)
- Faster
- Routes traffic based on **IP address and Port**
- Does **not inspect request content**

**Example:**  
Routes TCP/UDP traffic based on network-level information.

---

### Layer 7 (Application Layer)
- Smarter
- Routes traffic based on:
  - URL path
  - HTTP headers
  - cookies
  - hostname

**Example:**
```text
/api/users  → Server A
/api/orders → Server B
```

---

## Quick Comparison

| Feature | Layer 4 | Layer 7 |
|---|---|---|
| Works on | IP + Port | HTTP/HTTPS content |
| Speed | Faster | Slightly slower |
| Routing logic | Basic | Smarter |
| Best for | TCP/UDP traffic | Web apps / APIs |

---

## Quick Summary

A **Load Balancer** helps in:

- distributing traffic evenly
- improving availability
- increasing scalability
- boosting performance
- handling SSL termination
- avoiding server overload

---
