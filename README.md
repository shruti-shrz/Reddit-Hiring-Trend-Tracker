# Reddit-Hiring-Trend-Tracker

> **Market Intelligence at Scale:** A real-time dashboard transforming high-volume Reddit discussions into structured hiring signals for top-tier tech firms.

![System Architecture](diagram.png)

## 📖 Overview

The **Reddit-Hiring-Trend-Tracker** is an automated, event-driven platform designed to track the hiring health of 10 targeted tech companies (e.g., Google, Amazon, OpenAI) across 10+ subreddits. It captures real-time signals from posts and comments—such as offer details, interview loops, and hiring freezes—providing a "ground truth" pulse that often precedes official corporate announcements.

### Key Performance Indicators
* **Daily Throughput:** Ingests and filters **30,000+ daily posts/comments**.
* **Query Latency:** Sub-**10ms** dashboard responses via specialized time-series indexing.
* **Resilience:** Decoupled architecture handles API rate limits and traffic spikes without data loss.

---

## 🏗️ System Architecture

The project is built on a **Modular, Event-Driven Architecture** utilizing AWS managed services and TypeScript.

### 1. The Ingestion Layer (Harvester)
* **Trigger:** AWS EventBridge initiates a TypeScript ECS task every 5 minutes.
* **Idempotency:** Utilizes **Redis (Cluster #1)** for atomic deduplication (`SET NX`). Every Reddit ID is checked before being passed downstream to ensure 100% processing efficiency.
* **Security:** Operates in a **Private Subnet**; outbound Reddit API calls are routed through a **NAT Gateway** to protect infrastructure visibility.

### 2. The Buffering Layer (Decoupler)
* **Amazon SQS:** Acts as a shock absorber between ingestion and heavy processing. This allows the system to manage "Megathread" spikes without overwhelming the AI workers.
* **DLQ (Dead Letter Queue):** Captures failed messages for debugging, ensuring no market signal is permanently lost due to transient errors.

### 3. The Intelligence Layer (The Brain)
* **Filter-First Strategy:** To maintain a lean MVP footprint, the worker discards any content that doesn't mention the target Top-10 companies before running heavy analysis.
* **ML Inference:** Processes survived text through an embedded **DistilBERT model** to categorize signals into `OFFER`, `INTERVIEW`, `LAYOFF/FREEZE`, or `MISC`.

### 4. Storage & Performance (The Memory)
* **PostgreSQL + TimescaleDB:** * **Evidence Table:** Standard relational storage for post titles and permalinks (the "Top 5" evidence on the dashboard).
    * **Metrics Hypertable:** Specialized time-series table for counts and sentiment scores, utilizing **Hypertables** to ensure indices remain small and fast.
* **Data Retention:** Automated policy drops data older than 90 days, keeping storage costs flat and performance predictable.

---

## 🛠️ Tech Stack

| Category | Component |
| :--- | :--- |
| **Backend** | TypeScript, Node.js, NestJS |
| **Compute** | AWS ECS Fargate (Serverless Containers) |
| **Messaging** | Amazon SQS, EventBridge |
| **Databases** | Amazon RDS (PostgreSQL/TimescaleDB), ElastiCache (Redis) |
| **Machine Learning** | DistilBERT (Transformers), ONNX Runtime |
| **Frontend** | Next.js, Tailwind CSS, Recharts |
| **DevOps** | Docker, AWS CDK |

---

## 🗄️ Database Strategy

We differentiate between **Evidence** (unstructured links) and **Metrics** (aggregated trends) to optimize for speed.

### `company_evidence` (Standard Table)
| Column | Type | Description |
| :--- | :--- | :--- |
| `reddit_id` | `TEXT (PK)` | Unique Reddit ID (t3_... / t1_...) |
| `company_name` | `TEXT` | Normalized name (e.g., 'Google') |
| `category` | `ENUM` | Classification Result |
| `permalink` | `TEXT` | Direct link for dashboard "Top 5" |

### `hiring_metrics` (TimescaleDB Hypertable)
| Column | Type | Description |
| :--- | :--- | :--- |
| `bucket_time` | `TIMESTAMPTZ` | Partitioning Key (Daily/Hourly) |
| `company_name` | `TEXT` | Normalized name |
| `signal_count` | `INTEGER` | Sum of occurrences |
| `sentiment_avg` | `FLOAT` | Mean DistilBERT score |

---

## 📈 Future Roadmap

* **Dynamic Scaling:** Transitioning the Top-10 list to a dynamic RDS-driven list of 500+ firms.
* **Proactive Alerting:** Integrating AWS SNS to send email/SMS notifications when a company's "Layoff" sentiment spikes by >20% in 24 hours.
* **Comment-Depth Tuning:** Weighting top-level posts differently than deep-thread comments to refine sentiment accuracy.