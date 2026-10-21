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

---

## 5. Algorithmic Integrity & Concurrency

### The LOCF (Last Observation Carried Forward) Algorithm
Located within the `PriceService`, the charting engine handles sparse time-series data using a strict mathematical fill-forward approach. To prevent "time-travel" data corruption, the algorithm requires an absolute historical anchor. If a price point is unknown in the past, the timeline remains mathematically sparse until a true data point is intercepted, strictly preventing the projection of future data into historical gaps.

### Concurrency Protection (Thundering Herd Prevention)
If a chart is missing from the cache and many clients request it at the same moment, each request would run the same heavy SQLite query. The `PriceService` prevents this with a `SemaphoreSlim(1, 1)` lock. 
*   **Execution:** Only the very first request is allowed to hit the database. All concurrent requests wait at the lock. Once the first request completes and populates the RAM cache, the lock releases and the waiting requests are served from that cache entry instead of querying again, which prevents a thundering herd.
