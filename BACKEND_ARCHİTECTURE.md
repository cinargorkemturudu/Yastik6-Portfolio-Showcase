# Yastık6 Backend – Component Architecture & Execution Pipeline (C4 Level 3)

**Status:** Architectural Specification & Internal Component Design
**Scope:** .NET 10 Stateless BFF Pipeline (`Yastık6.Api`)

---

## 1. Executive Summary

The Yastık6 backend is engineered not as a traditional CRUD REST API, but as a highly resilient, stateless **Backend-For-Frontend (BFF) normalization pipeline**. Its primary directive is to insulate the client-side local fortress from the extreme volatility of external financial markets. By utilizing preemptive RAM caching, deterministic algorithms (LOCF), and active anomaly blocking, the backend serves every market request straight from memory while remaining entirely unburdened by client state or user identity.

## 2. API Component Architecture (C4 Level 3)


<img src="docs/Backend Architecture Diagram.png" alt="Yastık6 Backend Architecture">

---

## 3. The Engine Room: Background Workers

To ensure zero-latency client responses, external data fetching is strictly decoupled from client HTTP requests.

*   **`PriceSnapshotRefresher` (The Traffic Police):** A background service that runs one polling loop per primary provider (Harem, Binance, FreeCurrency, Yahoo BIST), each on its own refresh interval. It acts as the system's active heartbeat.
    *   **Smart Fallback Switch:** If a primary provider still fails after its retries (Polly, exponential backoff), the Refresher immediately fetches from a backup instead (`FreeCurrencyClient` -> `YahooForexClient`, `BinanceClient` -> `YahooCryptoClient`) and sends a Telegram alert.
    *   **Crash Backoff & Recovery:** The crashed loop then waits 5 minutes (`CrashBackoff`) before trying the primary again, so a failing vendor isn't hammered into banning us. When the primary succeeds again, the Refresher logs how long the outage lasted and sends a "Provider Recovered" alert.
*   **`HistoryCacheWarmer` (The Ghost Worker):** A background daemon that pre-computes dense chart datasets (Day, Week, Month, Year). It stages these massive queries into RAM before the user ever opens the application, ensuring heavy SQLite aggregations never block the main thread.

---

## 4. Normalization & Security Pipeline

All external data is treated as hostile and untrusted. Before updating the immutable `PriceRecord` entities, data must survive a multi-stage validation pipeline.

*   **BFF `PriceProcessor`:** Centralizes cross-currency logic. Crypto assets are dynamically converted to TRY utilizing the pre-cached `USDTRY=X` rate before hitting the database, offloading heavy multi-currency processing from the constrained mobile client.
*   **`PriceShieldValidator`:** The ultimate gatekeeper against upstream data corruption.
    *   **Shield 1 (Zero-Price):** Rejects payload instantly if Ask/Bid is `<= 0`.
    *   **Shield 2 (Flash Crash):** Rejects payload if the instant deviation exceeds `40%`.
    *   **Re-anchoring:** A real move larger than 40% must not freeze a symbol forever. When three consecutive readings agree within 5% on the new level, the shield accepts it and sends a "re-anchored" alert.
    *   **Smart Muting:** Uses an `AlertTracker` cache key to prevent Telegram alert spam, muting identical errors for 1 hour while the system silently falls back to the last known valid price.

### Public Demo Tier (`/api/public/*`)

A keyless, read-only slice of the API for anyone evaluating the project, isolated from the mobile app's traffic:

*   **Same data, no copy:** `PublicPricesController` reads the same in-memory snapshot and history service as the app. History is capped at Day/Week/Month (pre-warmed in memory), and unknown product codes are rejected from the snapshot before any database work. That matters because a history cache miss takes a lock shared with the app's own requests.
*   **Two rate limits stacked, sized to a budget:** 30 req/min per visitor, plus a 120 req/min ceiling shared by *all* visitors. The ceiling is sized from bandwidth: at ~380 KB per uncompressed market response, a sustained flood tops out around 2 TB/month of egress. It runs as the global limiter, so it covers the Swagger page's static files too. IPv6 visitors are bucketed per /64 so address rotation doesn't help. Rejections return `429` with `Retry-After`. Mobile app traffic has its own policy and is never counted.
*   **Quiet under attack:** rejected public requests are logged at Debug, so a flood can't fill Seq or the disk; hitting the shared ceiling sends one Telegram alert per hour instead. A config kill switch (`PublicApi__Enabled=false`) removes the whole tier without a code change.
*   **Real client IP behind Cloudflare:** Production runs Cloudflare → nginx → Kestrel, so the TCP peer is a Cloudflare edge IP shared by many visitors. The limiter partitions on `CF-Connecting-IP` instead.
*   **Separate OpenAPI document:** Swagger generates a second document containing only the public routes, served at `/api/public/docs` (nginx forwards only `/api/*`). The private document and the key header never appear there.
*   **Tested end-to-end:** `WebApplicationFactory` boots the real middleware pipeline in memory and asserts the key gate, both limits, IPv6 bucketing, the kill switch, `Retry-After`, and that the public document leaks no private routes.

---

## 5. Algorithmic Integrity & Concurrency

### The LOCF (Last Observation Carried Forward) Algorithm
Located within the `PriceService`, the charting engine handles sparse time-series data using a strict mathematical fill-forward approach. To prevent "time-travel" data corruption, the algorithm requires an absolute historical anchor. If a price point is unknown in the past, the timeline remains mathematically sparse until a true data point is intercepted, strictly preventing the projection of future data into historical gaps.

### Concurrency Protection (Thundering Herd Prevention)
If a chart is missing from the cache and many clients request it at the same moment, each request would run the same heavy SQLite query. The `PriceService` prevents this with a `SemaphoreSlim(1, 1)` lock. 
*   **Execution:** Only the very first request is allowed to hit the database. All concurrent requests wait at the lock. Once the first request completes and populates the RAM cache, the lock releases and the waiting requests are served from that cache entry instead of querying again, which prevents a thundering herd.
