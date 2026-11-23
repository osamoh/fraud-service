# Benchmarks

Measured in Claude's cloud workspace (Linux, 2 CPU cores, Python 3.13), with the load
generator running on the same machine as the server. Numbers on another machine will
differ: re-run the commands there and replace these.

## Lab 2b: before warm-up

First three requests to `POST /v1/predict` after a restart (`curl -w "%{time_total}s"`):

| Request | Time |
|---|---|
| 1 | 0.043 s |
| 2 | 0.009 s |
| 3 | 0.007 s |

The first request is about five times slower: it pays the model's first-call cost.

## Lab 2b: after warm-up

Same measurement with the warm-up prediction in `lifespan`:

| Request | Time |
|---|---|
| 1 | 0.012 s |
| 2 | 0.010 s |
| 3 | 0.010 s |

The first-call cost moved to start-up, so the first caller no longer pays it.

## Lab 2b: load test

`python scripts/loadtest.py` (2,000 requests, 25 at a time, `payloads/sample.json`):

| Server | p50 | p99 | Throughput | Non-2xx |
|---|---|---|---|---|
| `uvicorn` with 1 worker | 200.3 ms | 311.0 ms | 124 req/s | 0 |
| `uvicorn --workers 4` | 84.2 ms | 241.2 ms | 274 req/s | 0 |

Throughput moved most (about 2.2x) and p50 fell with it. Scoring is CPU work, and one
Python process uses one core for it, so with a single worker the 25 clients queue behind
each other. Extra worker processes score in parallel, up to the number of cores: this
machine has 2, so 4 workers gave about 2x, not 4x.

## Lab 2b: the async trap (optional)

`scripts/async_trap.py`, 50 requests, 10 at a time, each handler sleeps 0.2 s:

| Endpoint | p50 | p99 | Throughput |
|---|---|---|---|
| `/slow-async` (`async def` with a blocking call) | 1405.7 ms | 7430.2 ms | 5 req/s |
| `/slow-def` (plain `def`) | 204.9 ms | 225.7 ms | 47 req/s |

A blocking call inside `async def` stops the event loop, so requests are served one at a
time. The same call in a plain `def` runs in a worker thread and the others keep moving.

## Lab 3: images

Same cloud workspace as above (Linux, 2 CPU cores, amd64).

| Image | Size | Build time | Runs as | `leaked.env` inside |
|---|---|---|---|---|
| `fraud-service:naive` (`Dockerfile.naive`) | not measured | not measured | not measured | not measured |
| `fraud-service:dev` (multi-stage `Dockerfile`) | 803 MB | 48 s | `appuser` | no |

The naive image was not built in this workspace: Docker Hub rate-limited the pull of its
`python:3.12` base image. Build it on your own machine to fill in that row.

Rebuild of `fraud-service:dev` after a one-line change to `routes.py`: 0.5 s. Every layer
up to the model was `CACHED`; only `COPY src/` ran again.

## Lab 3: the service in a container

Compose stack (`fraud-api` with 2 workers and a 512 MB limit, plus `feature-cache`):

| Measurement | Value |
|---|---|
| Load test p50 | 77.4 ms |
| Load test p99 | 202.8 ms (Lab 2b: 311.0 ms with 1 worker, 241.2 ms with 4) |
| Load test throughput | 295 req/s, 0 non-2xx |
| Memory after the load test (`docker stats`) | 296 MiB of 512 MiB (58%) |
| `docker compose stop fraud-api` | 0.6 s (graceful: uvicorn is PID 1 and receives SIGTERM) |
