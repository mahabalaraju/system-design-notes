# Caching

## What is Caching?

**Caching** is a technique used to **store frequently accessed data in a temporary storage layer** so that future requests can be served **faster**.

In simple words:

> Caching helps reduce the time taken to fetch data by storing a copy of commonly used data in a faster storage location.

It improves **performance**, reduces **database load**, and enhances **user experience**.

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

## Why Do We Need Caching?

### 1) Faster Response Time
Cached data can be returned much faster than fetching it again from the database or external service.

### 2) Reduced Database Load
Frequently requested data does not hit the database every time.

### 3) Better Scalability
Caching helps the system handle **more users and more requests**.

### 4) Improved User Experience
Applications feel faster and smoother when data is quickly available.

---

## Real-World Example

Imagine an e-commerce website showing:

- product details
- best-selling products
- homepage banners
- category pages

These are accessed by thousands of users repeatedly.

Instead of querying the database every time, this data can be **cached** and served quickly.

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

## Example Flow

### Cache Hit
```text
User requests product details
        ↓
Data found in cache
        ↓
Response returned quickly
```

### Cache Miss
```text
User requests product details
        ↓
Data not found in cache
        ↓
Fetch from database
        ↓
Store in cache
        ↓
Return response
```

---

## Types of Caching

### 1) In-Memory Cache
Data is stored in application memory (RAM).

**Examples:**
- HashMap (small apps)
- Caffeine
- Ehcache

**Pros:**
- Very fast
- Easy to use

**Cons:**
- Data is lost when application restarts
- Not shared across multiple servers unless externalized

---

### 2) Distributed Cache
Cache is stored in a separate shared system accessible by multiple application servers.

**Examples:**
- Redis
- Memcached

**Pros:**
- Shared across multiple servers
- Better for scalable systems

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
Reduces repeated network calls.

---

### 4) CDN Cache
Content Delivery Networks cache content closer to the user geographically.

**Examples:**
- images
- videos
- static files

**Examples of CDN providers:**
- Cloudflare
- AWS CloudFront
- Akamai

---

## Common Cache Strategies

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

This is less commonly discussed in basic interview answers but good to know.

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
