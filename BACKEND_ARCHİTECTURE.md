# Yastık6 Backend – Component Architecture (C4 Level 3)

**Status:** Architectural Specification & Internal Component Design
**Scope:** .NET 10 Stateless BFF Pipeline (`Yastık6.Api`)

---

## 1. Backend Executive Overview

While the overarching system prioritizes the local-first client fortress, the backend acts as the critical shock absorber. Structurally organized around a clean Domain-Driven Design (DDD), this API is engineered not as a traditional CRUD server, but as a high-throughput, stateless data normalization pipeline. It insulates the application from volatile external market APIs, providing deterministic, perfectly formatted data streams to the client while requiring zero user-state retention.

## 2. API Component Architecture (C4 Level 3)

```mermaid
flowchart TD
    %% Custom C4 Styles
    classDef boundary fill:none,stroke:#666,stroke-width:2px,stroke-dasharray: 5 5
    classDef controller fill:#d4c5f9,stroke:#5a3ba0,stroke-width:2px,color:#000
    classDef service fill:#b8e0d2,stroke:#2d7a5d,stroke-width:2px,color:#000
    classDef cache fill:#f9d5e5,stroke:#a63d67,stroke-width:2px,color:#000
    classDef worker fill:#fff2cc,stroke:#d6b656,stroke-width:2px,color:#000
    classDef external fill:#e1e1e1,stroke:#666,stroke-width:2px,color:#000

    Client["Yastık6 Mobile Client"] :::external

    subgraph BackendAPI [".NET 10 API Container (Yastık6.Api)"]
        direction TB
        
        API["PriceController<br/>(REST Endpoints)"] :::controller
        Auth["ApiKeyHeadFilter<br/>(Shared-Secret Auth)"] :::service
        
        subgraph ApplicationLayer ["Application (Core Logic)"]
            Service["PriceService<br/>(Query Orchestrator)"] :::service
            Shield["PriceShieldValidator<br/>(Anomaly Detection)"] :::service
            Processor["PriceProcessor<br/>(Data Normalization)"] :::service
        end
        
        subgraph InfrastructureLayer ["Infrastructure (Workers & Data)"]
            Cache["PriceSnapshotStore &<br/>PreviousCloseStore<br/>(In-Memory Cache)"] :::cache
            Workers["BackgroundJobs<br/>(PriceSnapshotRefresher, HistoryCacheWarmer)"] :::worker
            
            subgraph Providers ["Provider Interfaces (IPriceProvider)"]
                Binance["BinanceClient"] :::service
                Harem["HaremClient"] :::service
                Yahoo["YahooFinance (Bist/Crypto/Forex)"] :::service
                FreeCurrency["FreeCurrencyClient"] :::service
            end
            
            SQLite[("AppDbContext<br/>(SQLite Persistance)")] :::cache
            Alerts["TelegramAlertService<br/>(Infrastructure Alerts)"] :::service
        end
    end

    %% Relationships
    Client -->|"Polls Overview & History"| Auth
    Auth --> API
    API --> Service
    Service -->|"Reads O(1)"| Cache
    Service -->|"Fetches stored history"| SQLite
    
    Workers -->|"Triggers refresh"| Providers
    Providers -->|"Raw Data Payload"| Processor
    Processor -->|"Domain Entities"| Shield
    Shield -->|"Updates on Valid"| Cache
    Shield -.->|"Fires on Anomaly"| Alerts
    Workers -->|"Saves Historical Ticks"| SQLite
```

---

## 3. Core System Components

The directory structure explicitly separates concerns, allowing for rapid iteration on the data provider pipelines without risking the integrity of the core domain. 

### The Application Layer (Business Orchestration)
The `Application/Features/Prices` namespace serves as the brain of the backend. It avoids heavy business logic, focusing instead on coordinating the flow of market data.
*   **`PriceService` & `PriceProcessor`:** Act as the central command, taking validated, normalized data from the infrastructure layer and preparing it for the presentation layer. 
*   **`PriceShieldValidator`:** The primary defense mechanism. Before any external data can update the cache, it passes through this validator. If upstream providers return corrupted anomalies, this component rejects the payload to protect the client's PnL math.

### Provider Strategy (Infrastructure Isolation)
Located in `Infrastructure/Providers`, the system implements a strict interface segregation pattern using `IPriceProvider`. 
*   Distinct clients for **Binance**, **Harem**, **FreeCurrency**, and **YahooFinance** (with specialized subsets for BIST, Crypto, and Forex) encapsulate their specific API quirks, rate limits, and payload structures. 
*   Custom mappers (e.g., `BinanceMapper`, `HaremMapper`) instantly convert these disparate payloads into the unified `PriceRecord` and `TrackedSymbol` domain entities, ensuring the core system only ever speaks one language.

---

## 4. The Hyper-Optimized RAM Polling Strategy

To maintain a strict response time contract for the mobile client and prevent resource exhaustion, the backend completely decouples data fetching from client requests through a highly optimized RAM pipeline.

*   **Server-Side Cache Warming:** Rather than fetching data on-demand, background workers (`PriceSnapshotRefresher` and `HistoryCacheWarmer`) run continuously via the `Infrastructure/BackgroundJobs` namespace. They absorb the latency and rate limits of external APIs, pre-computing and staging normalized data directly into in-memory caches.
*   **The O(1) Memory Endpoint:** Because the data is pre-computed and resident in RAM (`PriceSnapshotStore` and `PreviousCloseStore`), the `GetMarketOverview` endpoint executes with zero database queries or external HTTP calls.
*   **Decoupled Client Execution:** The Flutter app simply polls this endpoint at set intervals. This strategy allows a single .NET container to serve thousands of concurrent users instantly from memory without risking upstream API bans.

---

## 5. Operational & Deployment Architecture

The backend is built for zero-maintenance resilience. The configuration relies entirely on the `.NET 10` dependency injection container (`DependencyInjection.cs`), allowing the application to scale dynamically within its hosted environment.

*   **Self-Contained Infrastructure:** By utilizing internal singleton stores for caching and a local `AppDbContext` (SQLite) for historical tick data, the system eliminates the need for external caching dependencies (like Redis), keeping the architectural footprint light, fast, and portable.
*   **Observability:** The `Infrastructure/Alerts/TelegramAlertService` combined with `Serilog.Sinks.Seq` ensures that when an upstream API degrades or a rate limit is hit, silent failures are prevented, and real-time infrastructure alerts are pushed instantly.
*   **Containerized Production:** The root directory houses the `Dockerfile` and `docker-compose.yml`, outlining a streamlined container setup designed for rapid deployment onto isolated cloud server environments.
