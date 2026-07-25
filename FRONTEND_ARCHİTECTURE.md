# Yastık6 Frontend – Architectural Blueprint & Component Specification

**Status:** Architectural Specification & Mobile Client Design  
**Scope:** Flutter 3 Local-First Mobile Application (`Yastık6.Mobile`)

---

## 1. Executive Summary

The Yastık6 client is engineered as an isolated, privacy-first **Local-First Fortress**. Operating without user accounts or central databases, the Flutter application combines reactive state orchestration (Riverpod), infinite-precision arithmetic (`Decimal`), and direct SQLite persistence. By offloading volatile data transformations to a stateless backend BFF, the client maintains zero-latency UI performance while guaranteeing absolute financial accuracy and data sovereignty.

---

## 2. C4 Level 3 Diagram

<img src="docs/Frontend Architecture Diagram.png" alt="Yastık6 Frontend Architecture">

---

## 3. State Orchestration & Concurrency (`portfolio_provider.dart`)

To prevent main-thread jank and eliminate waterfall data fetching, the client replaces traditional multi-provider setups with a centralized, parallelized asynchronous pipeline.

*   **The Master Record (`RawDbData`):** A Dart 3 record type that encapsulates holdings, transactions, allocation maps, and realized summaries into a single immutable payload package.
*   **Parallel DB Execution (`_dbMasterProvider`):** Utilizes `Future.wait` to concurrently fetch holdings, transaction logs, and realized profit reports from SQLite. Dependent data structures (such as allocation maps) are resolved sequentially only after foundational entities are successfully loaded.
*   **Reactive State Boundaries:** The `PortfolioNotifier` manages robust error boundaries and loading states. If SQLite read failures occur, it traps the exception and exposes a clean error model to the UI, avoiding empty loading loops.

---

## 4. The Deterministic Engine (`portfolio_valuator.dart` & `portfolio_service.dart`)

Financial applications cannot tolerate floating-point rounding errors or time-travel logic corruption. 

*   **Infinite-Precision Arithmetic:** All portfolio cost bases, asset valuations, and profit-and-loss calculations are executed using the `Decimal` package, mapping values to rational numbers to eliminate floating-point drift.
*   **The FIFO Sequence Paradox & Barrier Dates:** When users attempt to edit historical transactions, the `PortfolioService` scans related transactions to establish dynamic `barrierDates`. If a backdated transaction compromises the First-In-First-Out (FIFO) allocation chain (e.g., editing a buy date past a linked sell date), the operation throws a deterministic validation exception, protecting local ledger integrity.
*   **Resilient Price Fallbacks:** If live market streams return zero or invalid payloads, the valuator instantly falls back to the last known valid price stored in the local asset register.

---

## 5. The "Dumb UI" Boundary (`universal_portfolio_mapper.dart`)

To enforce a strict separation of concerns, complex domain entities and raw database rows are never passed directly to the widget tree.

*   **Universal Portfolio Mapper:** Intercepts domain models (`AssetPosition`, `Transaction`, `PortfolioRealizedSummary`) and translates them into strictly formatted, immutable UI models (`LotUiModel`, `PortfolioDonutModel`).
*   **Presentation Formatting:** Presentation logic—such as asset symbol cleaning, localized date parsing, and dynamic PnL color mapping—is decoupled entirely from screens and widgets, ensuring maximum reusability across components like the `UniversalAssetPickerSheet` and history screens.

---

## 6. The Local-First Fortress (`local_database.dart`)

Data persistence is structured around raw SQLite execution and strict lifecycle management to ensure zero data loss during schema migrations.

*   **Singleton Enforcement:** Guarantees a single database instance throughout the application lifecycle, preventing race conditions and resource leaks.
*   **Transactional Schema Migrations:** Version upgrades (e.g., migrating to version 3) are handled via explicit SQL DDL transactions (`CREATE TABLE ... AS SELECT`) rather than fragile auto-migrations, safeguarding user financial history during updates.
*   **Performance Indexing:** Explicit composite indexes on foreign keys (`idx_transactions_asset_id`, `idx_allocations_sell_tx_id`) ensure sub-millisecond query execution even across extensive historical ledgers.

---

## 7. Algorithmic Data Smoothing (`chart_data_smoother.dart`)

Raw financial APIs often return jagged, highly volatile time-series data that renders poorly on mobile charts.

*   **Decimal-Space Moving Average:** The `ChartDataSmoother` utility intercepts raw `PricePoint` payloads, filters out zero-value anomalies, and computes a sliding-window moving average entirely in `Decimal` space.
*   **Render-Ready Dataset:** Generates clean `ChartPoint` arrays ready for seamless rendering on mobile UI charts without sacrificing underlying data precision.

---

## 8. Live Data Synchronization & Polling Bridge (`live_price_provider.dart`)

> **The Live Price Bridge & State Reactor**  
> The client side consumes the RAM cache output produced by the backend's `.NET` `PriceSnapshotRefresher` mechanism via an optimized polling loop (`live_price_provider.dart`). Instead of burdening the UI layer with raw network requests, components listen to an updated `MarketState` stream through a reactive `StreamProvider`. Background price updates seamlessly feed into delta calculations triggered by the `portfolioProvider`, allowing users to experience live price synchronization without manual pull-to-refresh actions, preserving complete local database performance.
