# Load Balancer

## What is it?
Distributes incoming traffic across multiple servers so no single 
server gets overwhelmed.

## Why do we need it?
Without it — one server handles all requests and crashes under heavy load.
With it — traffic is spread, system stays stable and fast.

## Types
- Round Robin — requests go to each server one by one
- Least Connections — goes to server with fewest active connections
- IP Hash — same user always goes to same server

## Real world example
When you open Swiggy — millions of users hit at the same time. 
Load balancer silently distributes all that traffic across hundreds 
of servers.

## Tools used in industry
Nginx, AWS Elastic Load Balancer, HAProxy

## My notes
Date: 25-Feb-2026
