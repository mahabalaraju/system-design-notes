# Load Balancer

## What is it?
Distributes incoming traffic across multiple servers so no single 
server gets overwhelmed.

or the load balancing is a process of distributing each client request equally among the pool of available servers . in the order to prevent overtaxing or clashing the servers . 

## Why do we need it?
Without it — one server handles all requests and crashes under heavy load.
With it — traffic is spread, system stays stable and fast.

## Types
- Round Robin — requests go to each server one by one
- weighted round robin — when some server better equipped to handle client requests we will give some weightage .  
- Least Connections — goes to server with fewest active connections
- IP Hash — same user always goes to same server

## Tools used in industry
Nginx, AWS Elastic Load Balancer, HAProxy


# Load Balancer

## What is a Load Balancer?

A **Load Balancer (LB)** is a system that **distributes incoming traffic across multiple backend servers** so that no single server gets overwhelmed.

In simple words:

> Load balancing is the process of distributing each client request across a pool of available servers to improve performance, availability, and reliability.

A load balancer acts like a **traffic controller** between the client and the servers.

### Simple Flow

```text
Client Requests
       ↓
 [ Load Balancer ]
   ↙    ↓    ↘
Server1 Server2 Server3


# Load Balancer – Complete Guide

## What is a Load Balancer?

A **Load Balancer (LB)** distributes incoming network traffic across multiple backend servers so that no single server gets overwhelmed. It acts as a reverse proxy and traffic controller between clients and your server pool.

**Simple definition**  
> Load balancing is the process of distributing each client request equally (or according to a strategy) among available servers to prevent overtaxing or crashing them.

## Why Do We Need a Load Balancer?

| Without Load Balancer                     | With Load Balancer                              |
|-------------------------------------------|--------------------------------------------------|
| One server handles all requests → crashes | Traffic is spread → system stays stable         |
| Single point of failure → whole app down  | High availability – if one fails, others take over |
| Slow response times, poor UX              | Fast, consistent performance                    |
| Hard to scale – manual intervention       | Auto‑scale – add/remove servers on the fly      |

**Real‑world example** – E‑commerce during Black Friday: a load balancer distributes millions of requests across hundreds of backend servers, preventing any single server from melting down.

## How It Works (Flow)

```text
Client (browser/mobile)
        │
        ▼
┌───────────────────┐
│  Load Balancer    │  (e.g., Nginx, HAProxy, AWS ALB)
│  - terminates SSL │
│  - chooses server │
└───────────────────┘
        │
   (algorithm)
        │
        ▼
┌─────────────────────────────────┐
│  Server Pool                    │
│  ┌──────┐ ┌──────┐ ┌──────┐    │
│  │ App1 │ │ App2 │ │ App3 │    │
│  └──────┘ └──────┘ └──────┘    │
└─────────────────────────────────┘
