# Logging Pipeline (like ELK) — Cheat Sheet

## Key Numbers
- Ingestion: 500K events/sec (peak 1M), 250 MB/s bandwidth
- Daily volume: ~43B events, ~4.3 TB/day compressed (5:1)
- 30-day hot storage: ~325 TB (data + inverted index)
- Searchable in < 5 seconds after ingestion (refresh interval)
- Search latency: < 5s for last 24h queries; < 30s for 30-day queries
- Hosts: 100,000 producing logs; 10,000 services

## Core Components (one line each)
- **Log Agent (Vector/Filebeat)**: runs on every host, collects logs, buffers locally, forwards to Kafka
- **Kafka**: durable event bus decoupling collection from processing, absorbs ingestion bursts
- **Log Processor (Logstash/Vector)**: parses (grok/JSON), enriches (geo, service meta), redacts PII
- **Elasticsearch**: inverted index store for full-text search, sharded and replicated across data nodes
- **ILM (Index Lifecycle Manager)**: automates index rollover and hot -> warm -> cold -> delete transitions
- **Query Service**: translates search queries, handles pagination, highlighting, and aggregations
- **Alerting Engine**: evaluates log pattern rules and fires notifications on threshold breach
- **S3 Archive**: compressed long-term storage for compliance and reprocessing

## Architecture in One Sentence
Agents collect logs from 100K hosts into Kafka, processors parse and enrich them, Elasticsearch indexes for full-text search with ILM-managed tiered storage (SSD hot -> HDD warm -> S3 cold), and a query layer serves search, live tail, and alerting.

## Top 3 Trade-offs
1. Elasticsearch (full-text search, expensive) vs. ClickHouse (aggregations, cheaper, weaker full-text)
2. Schema-on-write (fast queries, rigid) vs. schema-on-read (flexible, every query pays parsing cost)
3. Short refresh interval (near-real-time search, lower indexing throughput) vs. long interval (higher throughput, delayed visibility)

## Scaling Story
- **10x**: more ES data nodes, increase Kafka partitions, scale processors horizontally, mandatory time range on queries
- **100x**: S3-primary with lightweight bloom filter index, ClickHouse for aggregations, sampling for large-range searches, multi-cluster federation

## Closing Statement
A centralized logging pipeline using Kafka-buffered ingestion, structured parsing with PII redaction, Elasticsearch full-text indexing with tiered ILM storage, and graceful degradation under load via backpressure and load shedding.
