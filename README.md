# PodFetch: Enterprise Podcast Archival System

**Role:** Lead Software Architect
**Status:** Proprietary / Closed Source

---

## System Overview

PodFetch is a high-throughput podcast archival platform designed to reliably process 10,000+ RSS feeds in continuous 24/7 operation. The system automatically discovers, downloads, validates, and catalogs podcast episodes while maintaining data integrity across network failures, malformed feeds, and resource constraints. It serves as a critical data pipeline for large-scale media preservation and analytics workflows.

---

## Tech Stack

| Category | Technologies |
|----------|--------------|
| **Language** | Python 3.10+ |
| **Concurrency** | ThreadPoolExecutor, threading primitives (Condition, RLock, Event) |
| **Database** | SQLite with WAL mode for concurrent access |
| **Media Processing** | FFmpeg, ffprobe |
| **Networking** | urllib3, connection pooling, SSL/TLS |
| **Monitoring** | Real-time Textual dashboard, IPC-based remote monitoring |
| **Configuration** | INI-based with hot-reload capability |

---

## Key Challenges Solved

### 1. Concurrent Worker Pool with Dynamic Scaling

Designed a unified worker pool management system that supports runtime adjustment of worker limits without thread starvation or resource leaks. Replaced a fragmented 6-lock architecture with a 3-lock hierarchy following a strict ordering protocol to eliminate deadlock potential. The system maintains O(1) worker acquisition using a slot-based allocation strategy.

### 2. Fault-Tolerant Error Classification

Implemented a stratified error handling architecture that distinguishes between transient failures (network timeouts, rate limits), permanent failures (404s, SSL errors), and system-critical failures (disk exhaustion). Each error category triggers appropriate recovery behavior—exponential backoff for transient issues, circuit breaker engagement for chronic failures, and graceful degradation for partial successes.

### 3. Idempotent Processing Pipeline

Engineered an idempotent download-validate-record pipeline that guarantees no orphaned files or duplicate downloads across process restarts, crashes, or partial failures. Every successfully downloaded file is recorded to the deduplication database regardless of downstream processing failures, ensuring data consistency without manual intervention.

### 4. Adaptive Resource Management

Built an adaptive throttling controller that monitors system resources (CPU, memory, network I/O) and dynamically adjusts concurrency levels to prevent resource exhaustion while maximizing throughput. The system gracefully degrades under load and automatically recovers when resources become available.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                         FEED INGESTION                              │
│  CSV/Config Parser → Feed Validation → Priority Queue               │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│                      BATCH ORCHESTRATOR                             │
│  Adaptive Batching → Worker Pool Manager → Progress Checkpointing   │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                    ┌──────────────┼──────────────┐
                    ▼              ▼              ▼
              ┌──────────┐  ┌──────────┐  ┌──────────┐
              │ Worker 1 │  │ Worker 2 │  │ Worker N │
              └────┬─────┘  └────┬─────┘  └────┬─────┘
                   │             │             │
                   └─────────────┼─────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     PROCESSING PIPELINE                             │
│  RSS Parse → Dedup Check → Download → Validate → Transform → Store  │
└─────────────────────────────────────────────────────────────────────┘
                                 │
                   ┌─────────────┼─────────────┐
                   ▼             ▼             ▼
            ┌───────────┐ ┌───────────┐ ┌───────────┐
            │  Circuit  │ │  Episode  │ │   File    │
            │  Breaker  │ │  Database │ │  Storage  │
            └───────────┘ └───────────┘ └───────────┘
```

**Data Flow:**

1. **Ingestion** — Feed URLs are loaded from configuration, validated, and queued for processing with configurable prioritization.

2. **Orchestration** — The batch orchestrator partitions feeds into optimally-sized batches based on system capacity and dispatches work to the worker pool with backpressure management.

3. **Processing** — Each worker executes an atomic pipeline: parse feed → check deduplication → download media → validate integrity → transform metadata → persist to storage.

4. **Resilience** — The circuit breaker tracks feed health across runs, automatically quarantining chronically failing feeds. Error classification informs retry strategy and alerting thresholds.

5. **Observability** — Real-time dashboards expose worker status, throughput metrics, error rates, and resource utilization via IPC for remote monitoring.

---

## Design Principles

- **Fail gracefully, recover automatically** — Individual item failures never halt batch processing
- **Record everything** — All outcomes (success, partial, failure) are persisted for auditability
- **Respect external systems** — Adaptive rate limiting and exponential backoff protect feed providers
- **Crash-safe by default** — Progress checkpointing enables seamless resume after unexpected termination

---

*Note: This repository serves as an architectural overview. The source code is proprietary and not publicly available.*
