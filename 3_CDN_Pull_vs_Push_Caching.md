# CDN (Content Delivery Network)

> **Prerequisites:** Caching, Load Balancing
> **Used in:** Netflix, Instagram, URL Shortener, any system serving media or static content globally

---

## What is a CDN?

A CDN is a network of servers distributed worldwide (edge servers) that cache and serve content from the location nearest to the user.

**Real-world analogy:** Amazon warehouses. Instead of shipping every order from one central warehouse in Seattle (high latency, expensive), Amazon puts warehouses in every major city (edge locations). Your package comes from 20 miles away, not 2000.

```
Without CDN:
  User in Mumbai → requests image → travels to server in US → 200ms
  User in London → requests image → travels to server in US → 150ms

With CDN:
  User in Mumbai → requests image → CDN edge in Mumbai → 10ms
  User in London → requests image → CDN edge in London → 10ms
```

---

## How a CDN Works

```
First request (cache MISS):
  User → CDN Edge (Mumbai) → "I don't have this" → Origin server (US) → returns image
  CDN Edge caches the image locally
  Returns to user (slow this one time)

Subsequent requests (cache HIT):
  User → CDN Edge (Mumbai) → "I have this!" → returns immediately (fast)
  Origin server not contacted at all
```

```mermaid
flowchart TD
    A[User in Mumbai] --> B{CDN Edge in Mumbai}
    B -->|Cache HIT| C[Return immediately]
    B -->|Cache MISS| D[Origin Server US-East]
    D --> E[Returns content]
    E --> F[CDN Edge caches it]
    F --> G[Future requests served locally]
```

---

## Pull vs Push CDN

### Pull CDN (Lazy — most common)

CDN fetches content from origin only when first requested. If no one requests it, it's never cached at that edge.

```
Flow:
  User requests file → CDN doesn't have it → pulls from origin → caches → serves
  Next user requests same file → CDN has it → serves from cache
```

**Pros:** Only caches what's actually popular. No upfront setup.
**Cons:** First user gets slow response (origin fetch). Edge might not have content during spikes.
**Used by:** Cloudflare, CloudFront (default), most CDNs.

### Push CDN

You explicitly upload content to CDN edge servers. CDN serves it without ever contacting your origin.

```
Flow:
  You upload video to CDN → CDN distributes to all edges
  User requests video → CDN has it immediately → serves
```

**Pros:** First request is always fast. You control exactly what's cached.
**Cons:** Storage cost (paying for all edges). Must manage uploads. Stale content if you forget to update.
**Used by:** Netflix (pre-positions popular content on ISP servers), video platforms.

### Which to Use?

| Content Type | Best Approach |
|---|---|
| Static assets (CSS, JS, images) | Pull CDN |
| User-uploaded media (profile pics) | Pull CDN |
| Popular video content | Push CDN (pre-position) |
| Live streaming | Specialized CDN (not cached) |
| API responses | Usually NOT CDN (dynamic), unless cacheable |

---

## What to Cache on CDN

| Cache | Don't Cache |
|---|---|
| Static files (CSS, JS, fonts) | User-specific data (dashboard, settings) |
| Images, videos, PDFs | Authentication tokens |
| Public API responses (product catalog) | Real-time data (stock prices, chat) |
| HTML pages (for anonymous users) | POST/PUT/DELETE responses |

---

## Cache Invalidation on CDN

When you update content, how do you make the CDN serve the new version?

### Strategy 1: Versioned URLs (Best)

```
v1: /static/app.v1.js
v2: /static/app.v2.js (new URL = CDN treats as new file = no stale cache)
```

CDN caches indefinitely. New deployment = new URL. Old URL still works for users with old HTML.

### Strategy 2: Cache-Control Headers

```http
Cache-Control: public, max-age=86400  (cache for 24 hours)
Cache-Control: no-cache  (always revalidate with origin)
Cache-Control: no-store  (never cache)
```

### Strategy 3: Purge API

Most CDNs offer an API to force-invalidate cached content.

```
POST /purge {"urls": ["https://example.com/image.png"]}
→ CDN removes from all edges → next request fetches fresh from origin
```

**Trade-off:** Purge takes time to propagate (seconds to minutes across all edges).

---

## CDN Performance Numbers

| Metric | Without CDN | With CDN |
|---|---|---|
| Latency (US user, US server) | 50ms | 10ms |
| Latency (India user, US server) | 200ms | 15ms (served from India edge) |
| Origin server load | 100% of requests | 5-10% (90%+ served by CDN) |
| Bandwidth cost | High | Lower (CDN absorbs most traffic) |
| Availability during origin outage | 0% | Partial (cached content still served) |

---

## CDN Providers

| Provider | Best For | Unique Feature |
|---|---|---|
| **Cloudflare** | General web, DDoS protection | Free tier, Workers (edge compute) |
| **AWS CloudFront** | AWS ecosystem | Tight S3 integration, Lambda@Edge |
| **Fastly** | Real-time purging | Instant purge (< 150ms globally) |
| **Akamai** | Enterprise, largest network | 300K+ edge servers |
| **Google Cloud CDN** | GCP ecosystem | Global anycast, load balancing |

---

## Edge Computing (CDN + Logic)

Modern CDNs can run code at the edge (not just cache static files):

```
Traditional: User → CDN (static only) → Origin (all logic)

Edge computing: User → CDN Edge (runs simple logic like auth, redirects, A/B testing
```

**Examples:**

- Cloudflare Workers
- AWS Lambda@Edge / CloudFront Functions
- Fastly Compute@Edge

**Use cases:**

- URL redirects at the edge (no origin hit)
- A/B testing (decide variant at edge)
- Auth token validation (reject unauthenticated requests before hitting origin)
- Image resizing/optimization at edge

---

## CDN in System Design Interviews

**When to mention CDN:**

- System serves static content (images, videos, CSS/JS)
- Users are globally distributed
- Read-heavy workload with cacheable responses
- Need to reduce origin server load

**How to talk about it:**

> "User-uploaded images are stored in S3. We put CloudFront in front of S3 so images are served from the nearest edge location. First request goes to origin, subsequent requests are served from cache with ~10ms latency. We use versioned URLs for cache busting."

---

## Common Interview Questions

**Q: "How does a CDN reduce latency?"**
A: "Content is served from an edge server geographically close to the user (10-20ms) instead of traveling to the origin (100-300ms). Physics — speed of light through fiber has a floor."

**Q: "What if the origin goes down?"**
A: "CDN continues serving cached content. Users see existing content but can't get new content. This is 'stale-while-error' — better than a complete outage."

**Q: "How do you handle personalized content with a CDN?"**
A: "Don't cache personalized content on CDN. Use CDN only for static/public assets. Personalized API responses go directly to origin servers. Or use edge computing to assemble personalized pages from cached fragments."

**Q: "CDN vs Read Replica?"**
A: "CDN caches the final rendered content (bytes). Read replica caches at the database level (queries). CDN eliminates the server entirely for cached requests. Use both: CDN for static content, replicas for dynamic database queries."

---

## Navigation

- ← Back to Fundamentals
- ← Database Sharding
- Next: Distributed Locking →

---

## Related Concepts

- Caching
- Load Balancing
- Scalability

---

*Source: https://systemcraft.in/concepts/cdn/*
