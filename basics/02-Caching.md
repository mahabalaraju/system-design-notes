# Caching

## What is Caching?

**Caching** is a technique used to **store frequently accessed data in a temporary storage layer** so that future requests can be served **faster**.

In simple words:

> Caching helps reduce the time taken to fetch data by storing a copy of commonly used data in a faster storage location.

It improves **performance**, reduces **database load**, and enhances **user experience**.


## Why Do We Need Caching?

### 1) Lightning-Fast Response Time
Instead of hitting a slow database, API, or disk every time. its RAM very faster in microseconds. 

### 2)Reduces server load and costs
Fewer requests reach the origin server, database, or upstream API, so the backend can handle more traffic with the same resources

### 3) Better Scalability
Caching allows your system to handle more concurrent users and requests without needing to vertically scale.

### 4)Saves bandwidth:
Content (like images, JS, CSS, HTML) can be reused from browser/CDN caches instead of downloading again.

---
## Real-World Examples

### Example 1: E-Commerce Website
An e-commerce website may frequently show:

- product details
- best-selling products
- homepage banners
- category pages

These are requested repeatedly by thousands of users.Instead of querying the database every time, this data can be **cached** and served quickly.
This improves performance and reduces backend load.

---

### Example 2: Airline Booking Website
Airline websites often fetch **real-time flight data** from **global service providers** or external APIs.

This may include:

- flight availability
- ticket prices
- schedules

In many cases, these providers may **charge per API call**.

---

### Example 3: Applications Like Google Maps
Applications like **Google Maps** use caching for:

- route data
- nearby places
- frequently searched locations
---

## Simple Flow

```text
Client Request
      |
      v
 [ Cache ]
   |    \
 Hit      Miss
  |         \
  v          v
Return Data  Database
```

---

## Cache Hit vs Cache Miss

### Cache Hit
When the requested data is found in the cache.

**Result:**  
Fast response.

### Cache Miss
When the requested data is **not found** in the cache.

**Result:**  
The application fetches data from the database, returns it to the user, and usually stores it in the cache for future use.

---

## Types of Caching

Caching can happen at **different layers of the system**, depending on what kind of data we want to optimize.

### 1) In-Memory Cache
Data is stored in application memory (RAM).

**Examples:**
- HashMap (small apps)
- Caffeine
- Ehcache

**Pros:**
- Very fast
- No network call required
- Great for frequently accessed local data

**Cons:**
- Data is lost when application restarts
- Not shared across multiple servers unless externalized

> **Note:** If you have multiple app instances, each server maintains its own local cache.  
> So **Server A** does not automatically know what **Server B** has cached.
---

### 2) Distributed Cache
Cache is stored in a separate shared system accessible by multiple application servers.

**Examples:**
- Redis
- Memcached

**Pros:**
- Shared across multiple servers
- Better for scalable systems
- Redis can also support TTL, pub/sub, persistence, etc.

**Cons:**
- Slightly slower than local memory cache
- Requires external infrastructure

---

### 3) Browser Cache
Static assets like:
- CSS
- JavaScript
- images

can be cached in the user’s browser.

**Benefit:**  
- Reduces repeated network calls.
- Improves page load speed
- Saves bandwidth

---

### 4) CDN Cache
Content Delivery Networks cache content closer to the user geographically using **edge servers**.

**Examples:**
- images
- videos
- static files

**Examples of CDN providers:**
- Cloudflare
- AWS CloudFront
- Akamai
- 
**Benefit:**
- Faster delivery to global users
- Reduces latency
- Reduces load on the origin server

 #### Example:
If your server is in **Bengaluru** and a user from **Germany** requests an image, the CDN can serve it from a nearby edge server in **Europe** instead of fetching it again from India.

---

### 5) Database Cache

Most modern databases also use their own internal caching mechanisms.

**Examples:**
- PostgreSQL Buffer Cache
- MySQL Buffer Pool
- Oracle Cache

**Benefit:**
- Frequently accessed rows/pages are kept in RAM
- Reduces repeated disk I/O
- Improves query performance

---

## Quick Comparison

| Type | Location | Speed | Shared Across Servers? | Best Use Case |
|---|---|---|---|---|
| **In-Memory / Local Cache** | App RAM | Extremely Fast | ❌ No | Frequently accessed app data |
| **Distributed Cache** | External RAM (Redis/Memcached) | Very Fast | ✅ Yes | Shared scalable systems |
| **Browser Cache** | User Device | Fastest | N/A | Static frontend assets |
| **CDN Cache** | Edge Servers | Very Fast | N/A | Global static content delivery |
| **Database Cache** | Database RAM | Fast | N/A | Frequently accessed DB pages/rows |

---

## Expert Insight: Multi-Level Caching (L1 and L2 Cache)

In real-world high-scale applications, we often use **multiple layers of cache together**.

### Example:

```text
Request
   ↓
L1 Cache (Local / Caffeine)
   ↓
L2 Cache (Redis)
   ↓
Database
```

### How it works:
- First, check **L1 Local Cache**
- If not found, check **L2 Distributed Cache**
- If still not found, fetch from the **Database**

### Why use this?
Because:

- **L1 Cache** is extremely fast
- **L2 Cache** is shared across servers
- Database is used only when necessary

### Example latency:
- **L1 Local Cache** → ~0.01 ms
- **Redis (L2)** → ~1–5 ms
- **Database** → ~50–100+ ms

This approach gives both **speed** and **scalability**.

---

## Important Problem: Stale Data / Cache Inconsistency

When using **local cache** in multiple servers, you may face a common issue:

> One server updates the data, but other servers may still have **old cached values**.

### Example:
If **Server A** updates a product price:

- Server A may show the **new price**
- Server B may still show the **old price**

This is called:

- **Cache Inconsistency**
- **Stale Data Problem**

### Common solutions:
- TTL expiry
- cache invalidation
- Redis Pub/Sub
- event-driven cache refresh

> In distributed systems, **keeping cache fresh is often harder than storing it**.

---

## Common Cache Strategies
---

| Strategy | When data is read | When data is written | Pros | Cons |
|---|---|---|---|---|
| **Cache-Aside** | App checks cache → falls back to DB on miss | Write to DB and invalidate/update cache | Simple, flexible, widely used | Manual cache logic; possible cache misses |
| **Write-Through** | Reads from cache (assumed fresh) | Write to cache **and** DB synchronously | Cache stays consistent with DB | Slower writes |
| **Write-Back** | Reads from cache (assumed fresh) | Write to cache first; DB update happens later (async) | Very fast writes | Risk of data loss if cache fails |
| **Read-Through** | App reads only from cache; cache fetches DB on miss | Writes usually handled by DB or cache layer | App code stays simple | Requires a smarter cache layer |

---

### 1) Cache-Aside (Lazy Loading)
Application first checks cache:
- if found → return from cache
- if not found → fetch from database, store in cache, then return

This is the **most common strategy**.

### Flow:
```text
App → Check Cache
        |
      Hit / Miss
        |
Miss → Database → Save to Cache → Return
```

---

### 2) Write-Through
When data is written:
- write to cache
- write to database

Both are updated together.

**Benefit:**  
Cache remains fresh.

**Drawback:**  
Slightly slower writes.

---

### 3) Write-Back (Write-Behind)
When data is written:
- write to cache first
- database update happens later

**Benefit:**  
Fast writes.

**Drawback:**  
Risk of data loss if cache fails before DB update.

---

### 4) Read-Through
Application reads only from cache, and cache itself fetches from DB if data is missing.


---

## Cache Eviction Policies

Cache size is limited, so old data must be removed when space is needed.

### Common eviction policies:

### 1) LRU (Least Recently Used)
Removes the data that has **not been used recently**.

### 2) LFU (Least Frequently Used)
Removes the data that is used **least often**.

### 3) FIFO (First In First Out)
Removes the oldest cached item first.

---

## TTL (Time To Live)

**TTL** means how long a cache entry should stay before it expires.

### Example:
```text
Product Details Cache → TTL = 10 minutes
```

After 10 minutes, cache is removed or refreshed.

**Why TTL matters:**
- prevents stale data
- keeps cache updated
- balances freshness and performance

---

## Cache Invalidation

One of the hardest parts of caching is:

> **When should cached data be updated or removed?**

This is called **Cache Invalidation**.

### Example:
If product price changes in database, the old cached price must be removed or updated.

### Common invalidation methods:
- TTL expiry
- manual eviction
- update cache after DB update
- event-based invalidation

---

## Common Tools Used in Industry

| Tool | Type | Key Feature |
|---|---|---|
| **Redis** | Distributed Cache | Very fast, in-memory, supports TTL |
| **Memcached** | Distributed Cache | Simple and fast key-value cache |
| **Caffeine** | In-Memory Cache | High-performance Java cache |
| **Ehcache** | In-Memory / Disk Cache | Popular Java caching library |
| **Cloudflare** | CDN Cache | Caches static content globally |

---

## Redis and Caching

### What is Redis?
**Redis** is one of the most popular caching tools used in production.

It is:
- in-memory
- very fast
- key-value based
- supports TTL
- often used in distributed systems

### Common use cases:
- session storage
- product cache
- API response cache
- rate limiting
- leaderboard systems

---

## Example Use Cases of Caching

Caching is useful for:

- product details
- user profile data
- API responses
- configuration data
- session data
- search suggestions
- dashboard statistics

---

## Advantages of Caching

- Faster response time
- Reduced database load
- Better scalability
- Improved performance
- Better user experience

---

## Disadvantages of Caching

- Risk of stale data
- Cache invalidation complexity
- Extra infrastructure in distributed systems
- Cache consistency challenges

---

## Interview Questions

### 1) What is Caching?
Caching is the process of storing frequently accessed data in a faster storage layer to improve performance.

### 2) What is Cache Hit and Cache Miss?
- **Cache Hit** → data found in cache
- **Cache Miss** → data not found in cache, so fetched from DB

### 3) What is TTL in caching?
TTL (Time To Live) defines how long data remains in cache before expiring.

### 4) What is Cache Invalidation?
Cache invalidation is the process of updating or removing stale cached data.

### 5) What is the difference between Redis and in-memory cache?
- **In-memory cache** is local to one application instance
- **Redis** is shared across multiple servers

### 6) Which caching strategy is most common?
**Cache-Aside (Lazy Loading)** is the most commonly used caching strategy.

---

## Quick Summary

Caching helps in:

- improving performance
- reducing database calls
- speeding up applications
- handling large traffic efficiently

---

## Conclusion

**Caching** is one of the most important concepts in **System Design** and **Backend Development**.

It helps systems become:

- faster
- more scalable
- more efficient
- more user-friendly
