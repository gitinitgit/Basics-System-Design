# Networking Basics

> **Prerequisites:** none — this is foundational
> **Used in:** every design. Load balancers, CDNs, WebSockets, and gRPC choices all rest on this.

---

## Why interviewers ask this

You propose "L7 load balancer, HTTP/2 to the backend, WebSockets for live updates." Every one of those words is a networking decision. The follow-up is always *why* — and the answer is always in TCP, TLS, or HTTP semantics.

You don't need to recite the OSI model. You need to explain **why a mobile client on flaky wifi behaves differently on HTTP/3**, and **why 5 sequential internal hops cost you 100ms before any work happens.**

---

## The layers that matter

You'll be asked about "L4 vs L7." Here's the minimum map:

| Layer | Name | Unit | What Lives Here | Where You Meet It |
|---|---|---|---|---|
| L7 | Application | Message | HTTP, gRPC, WebSocket, DNS, TLS* | ALB, API gateway, Nginx |
| L4 | Transport | Segment/Datagram | TCP, UDP, QUIC | NLB, firewalls, ports |
| L3 | Network | Packet | IP, ICMP, BGP | Routing, subnets, anycast |
| L2 | Data link | Frame | Ethernet, ARP, MAC | Switches, VLANs |

\* *TLS is technically between L4 and L7 ("L6-ish"). Nobody will fault you for saying "TLS terminates at L7 devices."*

**Practical version:** L4 devices see IP and port. L7 devices can read the HTTP request. That single difference explains everything about load balancer capabilities.

---

## TCP vs UDP

| | TCP | UDP |
|---|---|---|
| **Connection** | Handshake required (3-way) | Connectionless, just send |
| **Reliability** | Retransmits lost segments | None — loss is silent |
| **Ordering** | Guaranteed | None |
| **Flow/congestion control** | Yes (sliding window, slow start) | No — you can flood the network |
| **Head-of-line blocking** | **Yes** | No |
| **Header overhead** | 20-60 bytes | 8 bytes |
| **Broadcast/multicast** | No | Yes |
| **Use for** | Anything where bytes must not be lost | Real-time where late data is useless |

**TCP three-way handshake — 1 RTT before any data:**

```
Client                          Server
  │────── SYN ───────────────────→│
  │←───── SYN-ACK ────────────────│
  │────── ACK + data ────────────→│  ← 1 RTT spent
```

**Add TLS 1.2 and you've spent 3 RTTs before the first byte of your request.** Across a 100ms link that's 300ms of pure protocol. TLS 1.3 cuts it to 2 RTTs total; QUIC to 1; QUIC with 0-RTT resumption to 0. This is the entire motivation for connection reuse and HTTP/3.

**When UDP is right:**

- Voice/video calls — a 200ms-late audio frame is worthless, so don't retransmit it
- Games — send the newest position, don't replay stale ones
- DNS — one small request/response; retry at the app layer is cheaper than a handshake
- Metrics/logs at volume (StatsD) — losing 0.1% of samples is fine
- QUIC/HTTP/3 — reliability rebuilt *on top* of UDP, in user space

**Say this:** "Video frames go over UDP because a retransmitted frame arrives after it was needed — better to drop it and conceal the gap. Signalling and the API go over TCP because losing a 'call ended' message is unacceptable."

### Head-of-line blocking, precisely

```
TCP delivers bytes in order. If segment #3 is lost:

  received: [1][2][ ✗ ][4][5][6]
  delivered to app: 1, 2, ...then nothing until #3 is retransmitted
```

Segments 4-6 sit in the kernel buffer, already arrived, unusable. On a 5% loss mobile link this destroys throughput — and with HTTP/2 it stalls *every* multiplexed stream on that connection, even unrelated ones. QUIC fixes this by tracking loss per-stream.

---

## HTTP versions

| | HTTP/1.1 | HTTP/2 | HTTP/3 |
|---|---|---|---|
| **Transport** | TCP | TCP | **QUIC (UDP)** |
| **Concurrency** | One request at a time per connection | Multiplexed streams | Multiplexed streams |
| **Browser workaround needed** | 6 connections/host | No | No |
| **Header compression** | None (plaintext, repeated) | HPACK | QPACK |
| **HOL blocking** | At HTTP layer | **At TCP layer** | None |
| **Handshake RTTs** | 1 (TCP) + 2 (TLS 1.2) | same | 1, or **0** on resumption |
| **Connection migration** | No | No | **Yes** (survives wifi→cellular) |
| **Server push** | No | Yes (deprecated in practice) | Yes (rarely used) |
| **Format** | Text | Binary framing | Binary framing |

**The HTTP/2 subtlety worth knowing:** HTTP/2 solved HTTP-level head-of-line blocking by multiplexing — but everything rides one TCP connection, so a *single lost packet* stalls all streams while TCP waits for the retransmit. On a clean datacenter link HTTP/2 is excellent. On a lossy mobile link, HTTP/2 can be *worse* than HTTP/1.1's six connections, because six connections means loss only stalls one sixth of your requests.

**HTTP/3's real wins:** per-stream loss recovery (no cross-stream stalling), fewer handshake round trips, and connection migration — your phone switching from wifi to LTE keeps the same QUIC connection ID instead of re-handshaking everything.

**Say this:** "HTTP/2 between internal services because loss is negligible in a datacenter and I get multiplexing plus header compression cheaply — that's also what gRPC rides on. HTTP/3 at the edge for mobile clients, where packet loss and network switching are the norm and per-stream recovery plus 0-RTT resumption matter."

### Keep-alive and connection reuse

```
Without keep-alive: handshake + TLS + request + response + close  (every time)
With keep-alive:    handshake once, then request/response × N
```

For internal service calls, a fresh connection per request can easily be 3× the cost of the request itself. Always reuse — and be aware of the failure mode: an idle connection the *server* closed but your client still thinks is open produces intermittent "connection reset" errors. Keep client idle timeouts **below** server and load balancer idle timeouts.

---

## Real-time communication options

| | Polling | Long Polling | SSE | WebSocket | Webhook |
|---|---|---|---|---|---|
| **Direction** | Client pulls | Client pulls (held) | **Server → client** | Bidirectional | **Server → server** |
| **Protocol** | HTTP | HTTP | HTTP | Upgraded from HTTP (`ws://`) | HTTP POST |
| **Connection** | New each time | Held until data | One long-lived | One long-lived | Per event |
| **Data** | Any | Any | **Text only** (UTF-8) | Text or **binary** | Any |
| **Auto-reconnect** | n/a | Manual | **Built in** (with `Last-Event-ID`) | Manual | n/a (retries) |
| **Proxy/LB friendliness** | Perfect | Good | Good (watch buffering) | Needs upgrade support + long idle timeouts | Perfect |
| **Server cost** | Wasted requests | Held connections | Held connections | Held connections | None between events |
| **Best for** | Slow-changing data, simplest thing that works | Legacy real-time | Notifications, live feeds, progress bars, token streaming | Chat, collaborative editing, games, trading | Third-party integrations |

**Say this:** "If updates only flow server→client, I use SSE — it's plain HTTP, it auto-reconnects with event IDs, and it works through every proxy. I only take WebSockets when the client also sends a high rate of messages, like chat or collaborative editing, because then I'd otherwise be paying for both a stream and a POST path."

⚠️ **Scaling any long-lived connection:** a connection is server state. That means sticky routing or a shared pub/sub (Redis, Kafka) so any server can push to any user's connection. Also budget ~10-50KB kernel + app memory per connection — 100K connections is real memory, and it caps how few boxes you can run.

---

## DNS

```
browser cache → OS cache → resolver (ISP/8.8.8.8) → root → TLD (.com) → authoritative
                                                                              ↓
                                                              A record → 93.18…
```

| Record | Purpose | Gotcha |
|---|---|---|
| **A** | Name → IPv4 | — |
| **AAAA** | Name → IPv6 | — |
| **CNAME** | Alias to another name | **Not allowed at the zone apex** (`example.com`) |
| **ALIAS / ANAME** | Apex-safe CNAME | Provider-specific (Route 53 alias) |
| **MX** | Mail servers | Priority field matters |
| **TXT** | Verification, SPF, DKIM | — |
| **NS** | Delegation to nameservers | — |
| **SRV** | Service + port | Used by service discovery, Kubernetes |

**TTL is the thing interviewers actually probe.** A 24-hour TTL means resolvers keep serving the old IP for up to a day after you change it — and some ignore short TTLs anyway.

**Say this:** "Before a migration I'd lower the TTL to 60 seconds a day *ahead* of the cutover, migrate, then raise it back. Lowering the TTL at cutover time doesn't help, because the resolvers already cached the old record with the old long TTL."

**DNS-based load balancing and its limits:** returning multiple A records or geo-specific answers is cheap global routing, but DNS has no health awareness at the client, caching defeats fast failover, and clients pick unpredictably. That's why CDNs use **anycast** instead: one IP announced from many locations, and BGP routes each user to the nearest — failover is a routing change, invisible and instant, with no cache to expire.

---

## TLS

**What the handshake achieves:** authentication (the cert proves the server is who it claims), key agreement (derive a shared symmetric key), and integrity (nobody can tamper undetected).

```
TLS 1.2:  ClientHello → ServerHello+cert → key exchange → Finished   (2 RTT)
TLS 1.3:  ClientHello+keyshare → ServerHello+cert+Finished           (1 RTT)
TLS 1.3 resumption / QUIC 0-RTT: data in the first packet            (0 RTT)
```

TLS uses asymmetric crypto (slow) only to agree on a symmetric key, then switches to symmetric (fast) for the bulk data. That's the whole design.

| Term | Meaning |
|---|---|
| **TLS termination** | Decrypt at the LB/CDN; backend traffic is plaintext (fast, backend sees plain HTTP, requires a trusted network) |
| **TLS passthrough** | LB forwards encrypted bytes untouched (end-to-end, but the LB can't do L7 routing) |
| **Re-encryption** | Terminate, inspect, re-encrypt to the backend (the usual compliance answer) |
| **mTLS** | Both sides present certs — the standard for service-to-service auth in a mesh |
| **SNI** | Client states the hostname in the handshake so one IP can serve many certs |
| **Certificate pinning** | Client only trusts a specific cert/key — stops MITM via a rogue CA, and bricks your app if you rotate wrong |
| **Forward secrecy** | Ephemeral keys, so stealing the private key later doesn't decrypt past traffic |

⚠️ **The classic outage:** an expired certificate. Automate renewal (ACME/Let's Encrypt, ACM) and alert at 30 days, not 3.

---

## Load balancing at L4 vs L7

| | L4 (Transport) | L7 (Application) |
|---|---|---|
| **Sees** | IP, port | Full HTTP request |
| **Can route on** | Source/dest IP, port | Path, host, header, cookie, method |
| **TLS** | Passthrough (usually) | Terminates |
| **Connections** | One, passed through | Two (client↔LB, LB↔server) |
| **Retries on 5xx** | No (can't see it) | Yes |
| **Throughput** | Millions of pps, μs latency | Lower, ms-level, more CPU |
| **WebSocket** | Works naturally | Needs explicit upgrade handling |
| **AWS** | NLB | ALB |

**Say this:** "L7 at the edge so I can route `/api` and `/static` separately, retry idempotent 5xx, and terminate TLS in one place. L4 when I need raw throughput, non-HTTP protocols, or the client IP preserved without X-Forwarded-For games."

**Health checks:** L4 checks "is the port open" (a hung app with an open socket passes). L7 checks "does `/health` return 200" (much better). Two rules: make the health endpoint cheap, and **don't** have it check downstream dependencies — otherwise one dependency blip marks your whole fleet unhealthy and the LB has nowhere to send traffic.

---

## Costs and constraints that shape architecture

| Constraint | Number | Why It Matters |
|---|---|---|
| **Same-AZ round trip** | ~0.5 ms | Chatty internal calls are affordable |
| **Cross-AZ round trip** | ~1-2 ms + **data transfer charges** | Cross-AZ traffic is billed; chatty cross-AZ is expensive |
| **Cross-region round trip** | 30-200 ms | Never put a sync cross-region call on the user path |
| **Internet egress** | The expensive direction | CDN offload is a cost decision as much as a latency one |
| **MTU / packet size** | 1500 bytes typical | Payloads over ~1400 bytes fragment; keep DNS and initial handshakes small |
| **Ephemeral port range** | ~28,000 per source IP per destination | A NAT gateway or a proxy fanning out to one backend IP can exhaust ports — a real production failure mode |
| **`TIME_WAIT`** | 60s per closed connection | High connection churn accumulates sockets; reuse connections |

**Say this about hops:** "Five sequential internal calls at 1ms each is 5ms of pure network — fine. Five sequential *cross-region* calls is a second, so cross-region work goes async or gets a regional replica."

---

## Interview application

**"How does a request reach your service?"**
"DNS resolves to an anycast CDN IP; the edge terminates TLS 1.3 and serves static assets from cache. Dynamic requests go over an already-warm HTTP/2 connection to the origin ALB, which does L7 routing on path, then HTTP/2 with keep-alive to the service. Internal service-to-service is gRPC over HTTP/2 with mTLS."

**"Why is the mobile app slower than the web app on the same API?"**
"Higher RTT and packet loss on cellular, plus radio wake-up latency. Each new connection costs a handshake, so I'd reuse connections aggressively, move to HTTP/3 for per-stream loss recovery and 0-RTT resumption, batch requests to cut round trips, and put a CDN edge close to users so the TLS handshake terminates nearby even if the origin is far."

**"How do you push updates to 1M connected clients?"**
"SSE if it's server→client only. Connections spread across a fleet behind an L4 LB with long idle timeouts; each server subscribes to the Redis/Kafka channels for the users it holds, so a publish reaches whichever server owns that connection without me needing sticky routing at the publish path. Budget ~30KB per connection, so 1M connections is ~30GB across the fleet — that sets the instance count."

---

## Common Interview Questions

**Q: "TCP or UDP for a video call?"**
A: "UDP (usually via WebRTC/SRTP) for media, because a retransmitted frame arrives too late to display — drop it and conceal. TCP for signalling, where losing a message breaks the call setup."

**Q: "Why is HTTP/2 not always better than HTTP/1.1?"**
A: "HTTP/2 multiplexes over a single TCP connection, so one lost packet stalls every stream via TCP head-of-line blocking. On a lossy network, HTTP/1.1's six parallel connections isolate that loss to one sixth of the requests. HTTP/3 over QUIC fixes it properly with per-stream recovery."

**Q: "What happens when you type a URL?"**
A: "DNS resolution (browser → OS → resolver → authoritative, honoring TTLs), TCP handshake (1 RTT), TLS handshake (1-2 RTTs), then the HTTP request. Response is likely served by a CDN edge; the browser parses HTML and issues subresource requests, reusing the connection. Total pre-first-byte cost is 2-3 RTTs, which is why connection reuse and edge termination matter so much."

**Q: "How do you failover DNS quickly?"**
A: "You don't rely on DNS for fast failover — TTLs are cached and some resolvers ignore short ones. Use anycast with health-aware BGP withdrawal, or a load balancer with a stable DNS name that handles failover behind it."

**Q: "Where do you terminate TLS?"**
A: "At the CDN edge for user latency (the handshake completes close to the user), then re-encrypt to the origin. Inside the cluster, mTLS between services via the mesh so the network isn't a trust boundary."

**Q: "Why do you sometimes get 'connection reset' intermittently?"**
A: "Usually an idle connection the server or LB already closed while the client's pool still thinks it's open. Fix: client idle timeout lower than the server's and the LB's, plus retry on idempotent requests."

**Q: "What's the difference between 502, 503, and 504?"**
A: "502 = the upstream returned something invalid. 503 = the service is unavailable or shedding load — often intentional. 504 = the upstream didn't respond within the gateway's timeout. They point at three different problems: a broken backend, a saturated one, and a slow one."

---

## Decision Framework

| Need | Choose |
|---|---|
| Bytes must not be lost | TCP |
| Late data is worthless | UDP |
| Mobile clients, lossy networks | HTTP/3 / QUIC |
| Internal service-to-service RPC | gRPC over HTTP/2, mTLS |
| Server→client updates only | SSE |
| High-rate bidirectional messages | WebSocket |
| Third-party notifications | Webhook (with signed payloads + retries) |
| Route on path/header, retry 5xx | L7 LB |
| Max throughput, non-HTTP, preserve client IP | L4 LB |
| Global entry point with instant failover | Anycast |
| Global entry point, per-region backends | GeoDNS (accepting cache lag) |

---

## Navigation

- ← Back to Fundamentals
- ← Performance Metrics
- Next: Deployment & Reliability →

---

## Where This Shows Up

This concept is load-bearing in these designs - each link goes straight to the design that leans on it:

- **Design a Chat System** - WebSocket framing, acks and offline sync
- **Design Zomato** - why live order tracking uses a push channel rather than polling
- **Design Google Docs** - a persistent bidirectional channel carrying edit operations

---

*Source: https://systemcraft.in/concepts/networking-basics/*
