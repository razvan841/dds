r# System Architecture

This document describes the current runtime architecture of the distributed e-commerce system and focuses on five questions:

1. How the gateway works
2. How many services we have
3. How services communicate (including queues/streams and sharing)
4. How messages are read and processed (worker model)
5. How many threads/processes run inside each service

## 1) High-Level Topology

The system has three business services:

- Order service
- Stock service
- Payment service

Each business service is sharded into 5 instances (shard IDs 0..4), for a total of 15 app containers.

In addition:

- 1 NGINX gateway container (external entrypoint on port 8000)
- 15 business Redis containers (one Redis per service shard)
- 5 saga Redis containers (one message-broker Redis per shard)

Total in docker-compose: 36 containers.

## 2) Gateway Behavior (NGINX)

Gateway config: gateway_nginx.conf

The gateway is a smart router, not just a reverse proxy.

### 2.1 Request routing strategy

For each service path prefix:

- /orders/
- /stock/
- /payment/

the gateway extracts the resource identifier from the URL using map rules, then uses upstream hash routing to pick a shard.

Examples:

- /stock/find/<item_id> hashes by item_id
- /payment/find_user/<user_id> hashes by user_id
- /orders/find/<order_id> hashes by order_id
- /orders/addItem/<order_id>/... hashes by order_id

This guarantees that requests for the same resource ID consistently reach the same shard.

### 2.2 Hash compatibility with Python

Python services use the same hash formula as NGINX (Cache::Memcached-compatible):

((crc32(key) >> 16) & 0x7fff) % num_shards

This is critical because services must route async commands to the same shard that gateway routing would select for direct REST calls.

### 2.3 Batch initialization broadcast

batch_init endpoints are special-cased:

- Primary request is sent to shard 0
- NGINX mirror sends copies to shards 1..4

So batch initialization is broadcast across all shards.

## 3) Services and Data Isolation

There are 3 logical services and each is independently sharded.

- Order: order-service-{0..4} with order-db-{0..4}
- Stock: stock-service-{0..4} with stock-db-{0..4}
- Payment: payment-service-{0..4} with payment-db-{0..4}

Important isolation rule:

- No shared business database between services
- Each shard talks only to its own business Redis for service data

The saga Redis layer is shared only for inter-service messaging, not for business state.

## 4) Inter-Service Communication

Communication is mixed-mode:

- Synchronous REST via gateway for regular read/update flows (example: order addItem checks stock via /stock/find/...)
- Asynchronous Redis Streams for checkout orchestration (saga and 2PC)

### 4.1 Stream topology and queue count

Stream names are shard-specific:

- stock-commands-{shard}
- payment-commands-{shard}
- saga-replies-{shard}

With 5 shards, this gives:

- 5 stock command streams
- 5 payment command streams
- 5 orchestrator reply streams
- Total: 15 streams across the deployment

Each shard has its own saga Redis instance (saga-redis-{0..4}), and stream suffix determines which saga Redis node stores that stream.

### 4.2 Are queues shared?

Yes and no:

- Shared by service workers inside the same shard through consumer groups
- Not shared across shards (each shard has distinct stream names and a distinct saga Redis instance)

Consumer groups:

- stock-workers
- payment-workers
- orchestrator-workers

## 5) Message Reading and Worker Model

Message consumption logic lives in common/streams.py (consume_loop).

### 5.1 Do we have workers?

Yes, two worker layers exist:

- Gunicorn workers for HTTP serving
- Redis stream consumers running inside each worker process

There are no separate external worker containers. Consumers run in-process.

### 5.2 How messages are read

Each consumer loop:

1. Builds consumer name as hostname-pid
2. Runs startup recovery with XAUTOCLAIM (reclaim stale pending messages idle >= 5s)
3. Reads new messages with XREADGROUP, block=1000ms, count=10
4. Calls handler function per message
5. ACKs with XACK after handling

This pattern is used for:

- Stock command consumption
- Payment command consumption
- Order orchestrator reply consumption

### 5.3 Duplicate and crash behavior

Idempotency keys are stored in business Redis:

- First delivery marks idempotency:key as processing
- Handler stores final reply payload and marks done
- Duplicate delivery replays stored reply instead of reprocessing

This protects against redelivery after crashes/restarts.

## 6) Threads and Processes per Service

### 6.1 Gunicorn runtime per app container

Every order/stock/payment container runs:

- Gunicorn master process
- 2 gevent worker processes (-w 2)
- worker-connections=1000 per worker

So each app container has 2 request-serving worker processes.

### 6.2 Background consumer threads

In each worker process, app startup creates one daemon thread:

- Order worker: orchestrator consumer thread
- Stock worker: stock command consumer thread
- Payment worker: payment command consumer thread

Therefore, per app container:

- 2 worker processes
- 2 background consumer threads total (1 per worker process)

What each consumer thread actually does:

1. Block on the Redis stream with XREADGROUP (up to 1 second)
2. Receive one or more pending messages from that stream
3. For each message: run the service handler logic
4. ACK the message (XACK)
5. Loop and read again

Inside one consumer thread, message handling is sequential (one message handler call at a time).

Because each app container has 2 Gunicorn worker processes and each worker starts its own consumer thread, there are typically 2 consumers in the same consumer group for that shard/stream. Redis consumer groups then distribute messages across those 2 consumers.

Practical implication in this deployment:

- Per shard stream (for example stock-commands-2), you can process up to about 2 messages concurrently (one per consumer thread)
- Per service overall, there are 5 shards x 2 consumers per shard = up to about 10 messages concurrently across all shards

Notes:

- This is an upper bound, not a guarantee. Real throughput depends on message mix, lock contention, Redis latency, and handler execution time.
- If one message is slow, only that consumer thread is blocked; the other consumer thread in the same shard can still process another message.

Across all 15 app containers:

- 30 gunicorn worker processes
- 30 consumer threads

Gevent handles HTTP concurrency cooperatively inside each worker process; these are greenlets, not extra OS threads per request.

## 7) Checkout Protocols on Shared Messaging Infrastructure

Both checkout modes use the same stream infrastructure.

- Saga mode: compensating transactions
- 2PC mode: prepare/commit/abort with lock-based coordination and retries on lock contention

Order service reply handler dispatches replies by command type to saga or 2PC orchestrator logic.

## 8) Current Configuration Notes

Current docker-compose configuration is 5 shards (SHARD_COUNT=5, IDs 0..4).

If shard count changes in deployment config, queue counts and worker totals scale linearly with shard count. The architectural patterns above remain the same.
