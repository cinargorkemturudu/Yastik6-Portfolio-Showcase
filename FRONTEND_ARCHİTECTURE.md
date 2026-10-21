# Yastık6 Frontend – Architectural Blueprint & Component Specification

**Status:** Architectural Specification & Mobile Client Design  
**Scope:** Flutter 3 Local-First Mobile Application (`Yastık6.Mobile`)

---

## 1. Executive Summary

The Yastık6 client is engineered as an isolated, privacy-first **Local-First Fortress**. Operating without user accounts or central databases, the Flutter application combines reactive state orchestration (Riverpod), exact decimal arithmetic (`Decimal`), and direct SQLite persistence. By offloading volatile data transformations to a stateless backend BFF, the client keeps the UI responsive while all money math stays exact and on the device.

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

*   **Exact Decimal Arithmetic:** All portfolio cost bases, asset valuations, and profit-and-loss amounts use the `Decimal` package, so there is no floating-point drift. Divisions such as percentages are computed as rational numbers and rounded to 10 decimal places.
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
*   **Transactional Schema Migrations:** Version upgrades (currently up to version 3) rebuild tables explicitly (`CREATE TABLE … new` → `INSERT … SELECT` → `DROP` → `RENAME`) inside sqflite's upgrade transaction, rather than relying on auto-migrations, so user financial history survives updates.
*   **Performance Indexing:** Explicit indexes on the foreign keys (`idx_transactions_asset_id`, `idx_allocations_sell_tx_id`, `idx_allocations_buy_tx_id`) keep lookups fast as the transaction ledger grows.

---

## 7. Algorithmic Data Smoothing (`chart_data_smoother.dart`)

Raw financial APIs often return jagged, highly volatile time-series data that renders poorly on mobile charts.

*   **Decimal-Space Moving Average:** The `ChartDataSmoother` utility intercepts raw `PricePoint` payloads, filters out zero-value anomalies, and computes a sliding-window moving average entirely in `Decimal` space.
*   **Render-Ready Dataset:** Generates clean `ChartPoint` arrays ready for seamless rendering on mobile UI charts without sacrificing underlying data precision.

---

## 8. Live Data Synchronization & Polling Bridge (`live_price_provider.dart`)

> **The Live Price Bridge & State Reactor**  
> The client consumes the RAM cache produced by the backend's `PriceSnapshotRefresher` through a polling loop (`live_price_provider.dart`): an `AsyncNotifierProvider` that refreshes every 10 seconds. Instead of burdening the UI layer with raw network requests, components watch its `MarketState`. When a request fails, it falls back to the last market data cached on the device, so the app keeps working offline. Background price updates seamlessly feed into delta calculations triggered by the `portfolioProvider`, allowing users to experience live price synchronization without manual pull-to-refresh actions, preserving complete local database performance.

---

## 9. Encrypted Cloud Backup (`crypto_service.dart` & `cloud_backup_service.dart`)

Backups are opt-in, and the cloud provider only ever stores an encrypted blob.

*   **Passphrase-Derived Key:** The user chooses a backup passphrase. The key is derived from it with PBKDF2-HMAC-SHA256 (210,000 iterations, random 16-byte salt) on a background isolate, and is never stored. The same passphrase restores the backup on a new phone.
*   **Authenticated Encryption:** The database file is encrypted with AES-256-GCM. A versioned header (format, iteration count, salt, nonce) is authenticated together with the data, so a wrong passphrase or a modified file is rejected instead of decrypting into garbage.
*   **Safe Restore:** The backup is downloaded and decrypted into temporary files and verified to be a SQLite database. Only then is the connection closed and the file swapped in. Any failure leaves the current portfolio untouched.
*   **Provider Strategy:** Uploads go through a `CloudBackupStrategy` interface. Google Drive (app-private folder) is implemented; iCloud is stubbed but not yet configured.
