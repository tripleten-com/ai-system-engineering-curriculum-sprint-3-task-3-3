# Coldline Task 3.3 — Asynchronous recovery

This checkpoint retires Redis Streams as the `JobQueue` transport in favor of a supplied,
conformance-tested LocalStack SQS adapter with a bound dead-letter queue. The transport swap,
its composition wiring, and two failure-exercise scripts all ship complete. What is not yet
correct is the deployed queue's dead-letter policy: the starter's receive-count bound gives a
delivery zero tolerance for a single transient failure. Fixing that policy, and observing what
it does under a forced failure, is this Task's work.

[![Open in GitHub Codespaces](https://github.com/codespaces/badge.svg)](https://codespaces.new/tripleten-com/ai-system-engineering-curriculum-sprint-3-task-3-3/tree/main)

## Start the system

Prerequisites are Python 3.12 and Docker with Compose v2. The supplied bootstrap supports macOS
arm64/x86-64, Windows x86-64, and Linux x86-64/aarch64, and installs pinned uv 0.11.8 under
`.tools/bin`. If your computer cannot run the stack locally, use the Codespaces button above.

On macOS and most Linux distributions the interpreter is `python3`; substitute it wherever these
commands say `python`.

```shell
python infra/scripts/bootstrap.py
./.tools/bin/uv sync --frozen
./.tools/bin/uv run --frozen poe preflight
./.tools/bin/uv run --frozen poe start
./.tools/bin/uv run --frozen poe ready
```

PowerShell and POSIX wrappers are available under `infra/scripts/`. After uv is on `PATH`, the
shorter `uv run --frozen poe <task>` form works.

| Service | Local URL | Purpose |
|---|---|---|
| API | `http://localhost:8000` | Submit exception workflows and retrieval queries; `/version` names the build that answers |
| Grafana | `http://localhost:3000` | Use the focused diagnostics dashboard |
| Prometheus | `http://localhost:9090` | Query bounded metrics |
| Jaeger | `http://localhost:16686` | Inspect local traces |
| LocalStack S3/SQS | `http://localhost:4566` | Inspect the emulated object-storage and queue endpoint |

Each of these ports can be overridden by setting the matching `COLDLINE_API_HOST_PORT`,
`COLDLINE_GRAFANA_HOST_PORT`, `COLDLINE_PROMETHEUS_HOST_PORT`, `COLDLINE_JAEGER_HOST_PORT`, or
`COLDLINE_LOCALSTACK_HOST_PORT` environment variable in your shell environment or a local `.env`
file (copy `.env.example`) if a default collides with something already running on your machine.
Keep the override in place for every `poe` command.

PostgreSQL, Redis, worker metrics, and OTLP remain inside the Compose network. Codespaces uses the
same `compose.yaml` and keeps every forwarded port private. Redis keeps running in this Task only
for an earlier checkpoint's own contract test; no composition root reads it anymore.

## Command path

For this Task, run the supplied commands in this order:

```text
poe start
poe queue-contract
poe verify
```

| Command | Use |
|---|---|
| `poe queue-contract` | Run the automated dead-letter redrive and production-queue policy checks |
| `poe contract` | Check interfaces, boundaries, submissions, and repository structure |
| `poe smoke` | Check the initialized running platform |
| `poe e2e` | Run the external API-to-worker workflow |
| `poe inject-failure` | Submit one reading, then exhaust the queue's redrive budget so it reaches the dead-letter queue |
| `poe redrive` | Resubmit every dead-lettered message and wait for each to reach a terminal state, then deliver the most recent one once more and confirm its completed record is unchanged; any older ones are listed under `also_redriven` |
| `poe verify` | Run the public student verification path |
| `poe student-tests` | Run the supplied tests under `tests/student/`; this Task permits no additions there |
| `poe restart` | Restart the existing API and worker containers **without rebuilding** |
| `poe stop` | Remove containers and the network, keeping named volumes |
| `poe reset` | Remove containers, the network, and local named volumes |

For Task 3.3, `poe verify` starts the stack, ingests the supplied corpus, runs the smoke tests,
the end-to-end exception workflow, the queue-contract checks, and the answer-sheet checks. Run
the two failure exercises by hand, against the live stack, as described in the Task contract —
they are not part of `poe verify`.

## Folder map

```text
repository root/
├── docs/                Student guidance, public contracts, and fidelity notes
│   ├── contracts/       Machine-readable public contracts
│   ├── fidelity/        Local-runtime boundary notes
│   ├── architecture/    Supplied vector engine technical profiles, in prose
│   ├── retrieval/       Supplied retrieval pipeline reference
│   └── student/         This Task's contract
├── config/              Retrieval configuration, settled and supplied from Sprint 2
├── infra/               Local setup and runtime configuration
│   ├── containers/      The API and worker Dockerfiles, with the build identity arguments
│   ├── release/         The supplied Task 3.1 release manifest, unchanged
│   ├── corpus/          Supplied synthetic corpus, query set, and designated investigation
│   ├── judge/           Supplied cached judge evidence and its provenance record
│   ├── profiles/        Supplied engine and emulator profiles, and their provenance record
│   └── postgres/        Database initialization and the migration baseline stamp
├── loadtest/            Supplied traffic profile and provider-latency harness
├── migrations/          Alembic environment, revision template, and revisions
├── src/
│   ├── api/             HTTP application code, the retrieval and document paths, composition
│   ├── worker/          Background application code, fully supplied for this Task
│   ├── domain/          Shared domain code, contracts, the failure taxonomy, service and repository contracts
│   ├── ports/           Application interfaces
│   └── adapters/        Technology-specific implementations, including the supplied SQS/DLQ queue adapter
└── tests/
    ├── unit/            Isolated behavior checks
    ├── benchmark/       Supplied evaluation harness, metrics, and adoption policy
    ├── contract/        Interface, retrieval, and repository checks
    ├── diagnostics/     Supplied stage inspector
    ├── doubles/         Supplied deterministic test doubles
    ├── failure/         Supplied dead-letter failure-exercise scripts — run them, do not edit them
    ├── student/         Supplied student tests; no additions in this Task
    ├── smoke/           Running-platform checks
    └── e2e/             Supplied workflow tools and checks
```

## Overview

Use the Task 3 lesson (Task 3.3 in this repository) to decide what to do. This README covers
local setup and repository orientation.

1. `README.md` — local setup, commands, and permitted changes.
2. [`docs/student/task-3-3-contract.md`](docs/student/task-3-3-contract.md) — the one setting,
   the two exercises, what each check verifies, and the permitted paths.
3. `src/adapters/queue/sqs.py` — the supplied adapter; read its docstrings.
4. `tests/failure/force_dlq_arrival.py` and `tests/failure/redrive_and_verify.py` — the two
   exercise scripts you run against the live stack.

The application source lives in five flat packages:

| Package | Responsibility |
|---|---|
| `api` | HTTP delivery, API use cases, the retrieval workflow, versioned routes, configuration, and composition |
| `worker` | Background processing, retries, configuration, and composition |
| `domain` | Provider-neutral contracts, state rules, identity, redaction, embedding, chunking, fusion, access constraints, failure classification, service and repository contracts |
| `ports` | Exactly five visible application interfaces |
| `adapters` | PostgreSQL, pgvector retrieval, LocalStack SQS/DLQ, S3-compatible object storage, deterministic model, the resilient model-provider wrapper, logs, traces |

`src/api/bootstrap.py` and `src/worker/bootstrap.py` compose each process from its settings and
adapters. Process settings live in `src/api/config.py` and `src/worker/config.py`.

## The five ports

Find the available interfaces in `src/ports/`. A port describes an application capability; an
adapter provides it using a concrete technology.

| Port | General responsibility |
|---|---|
| `ModelProvider` | Call an AI model service |
| `Retriever` | Look up relevant context or documents |
| `ObjectStore` | Store large binary objects or files |
| `JobQueue` | Publish and consume background work |
| `SecretProvider` | Read API keys and credentials |

LocalStack SQS, with a bound dead-letter queue, now carries `JobQueue`. Redis Streams is retired
as its transport; see [JobQueue fidelity](docs/fidelity/JobQueue.md).

## Test levels

| Level | Requires Compose | Main question |
|---|---:|---|
| Unit | No | Does one responsibility behave correctly, including failures? |
| Contract | Some | Do interfaces, schemas, paths, and dependency rules stay compatible? |
| Smoke | Yes | Did the complete local platform initialize and become observable? |
| E2E | Yes | Can an external client complete the supplied workflow? |

Contract checks marked `runtime` need the running stack. `poe contract` skips them; `poe verify`,
`poe runtime-contract`, and `poe queue-contract` run them.

## Submission checks

Run `poe verify` locally before opening your student pull request. Public GitHub CI repeats
the student checks. This Task records `answers: {}`: your deployed configuration and its
automated checks are the evidence, so there is no separate protected answer check. Follow the
Task lesson's instructor-review and progression policy.

## Task boundary

Task 3.3 asks you to raise `queue_max_receive_count` in `compose.yaml` to a value that survives
at least one transient failure before the message reaches the dead-letter queue, then run the
two supplied failure-exercise scripts against the live stack and observe the result.

These paths are student-editable:

- `compose.yaml`
- `submission.yaml`

Keep the SQS/DLQ adapter, every bootstrap and initializer module, the two exercise scripts, and
every test file exactly as supplied; the public checks compare them. Everything else in this
repository is supplied, including the release manifest and the rest of the application source.

### Student walkthrough

See **Task 3: Asynchronous recovery** in your course platform for the full walkthrough. In
outline: read `docs/student/task-3-3-contract.md` and `src/adapters/queue/sqs.py`'s docstrings,
raise `queue_max_receive_count` in `compose.yaml`, run `poe start` and `poe queue-contract` until
it passes, run the two failure exercises (`poe worker-stop`, `poe inject-failure`,
`poe worker-start`, `poe redrive`) against the live stack and confirm the redriven message
completes and its `replay` output shows the second delivery left the record unchanged, run
`poe verify`, and open your pull request.

## Operational limits

This local system does not authenticate users, terminate TLS, or manage production secrets.
The Compose PostgreSQL password and the LocalStack access keys are local-only non-secret
credentials. Never place real credentials, personal data, or production records in this
repository.

LocalStack's SQS emulation is a local reliability primitive, not a managed-service durability,
IAM, availability, or cost claim. See [JobQueue fidelity](docs/fidelity/JobQueue.md) for its exact
boundary.

Named volumes preserve local PostgreSQL, Redis, Prometheus, Grafana, and Jaeger state across
`poe stop`. LocalStack object and queue contents are deliberately not persisted; the initializer
re-uploads the supplied corpus artifacts and re-provisions the queue on every start. The
`poe reset` command deletes the named volumes. This topology makes no backup, replication,
high-availability, disaster-recovery, capacity, latency-SLO, or availability claim.

See [JobQueue fidelity](docs/fidelity/JobQueue.md),
[ModelProvider fidelity](docs/fidelity/ModelProvider.md),
[ObjectStore fidelity](docs/fidelity/ObjectStore.md), and
[Retriever fidelity](docs/fidelity/Retriever.md) for the active adapter boundaries. The
[local runtime evidence](docs/fidelity/local-runtime.md) records the current measurement and its
qualification limits.
