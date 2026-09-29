# Task 3.3 — Asynchronous recovery contract

Configure the deployed queue's dead-letter policy, then run the two supplied exercise scripts
that force one exception to the dead-letter queue and recover it. You edit one number in
`compose.yaml`. You do not touch the transport, either adapter, or how the worker is wired.

## What is assessed, and by whom

| Assessed | By |
|---|---|
| The pull request changes only `compose.yaml` and `submission.yaml` | Automated, in this repository |
| The deployed queue's redrive policy tolerates at least one transient failure before dead-lettering a message | Automated, against the running `coldline-exception-jobs` queue |
| A message that exceeds the configured `maxReceiveCount` still arrives at the dead-letter queue automatically | Automated, against a fresh exercise queue with the same shape |
| A redriven message still completes exactly once, proving recovery is idempotent, not merely eventual | Shown by `poe redrive`'s `replay` output, which delivers the completed job once more against `WorkerApplication`'s existing terminal-state handling; your instructor reads it in your pull request evidence |
| Your reasoning about why `queue_max_receive_count=1` passes Pydantic's bounds but is still the wrong choice | Your instructor, at the Project Defense |

## What is already supplied

| Supplied | Where | Note |
|---|---|---|
| The SQS/DLQ adapter and its provisioning | `src/adapters/queue/sqs.py` | conformance-tested; do not edit |
| Composition wiring to the queue adapter | `src/api/bootstrap.py`, `src/worker/bootstrap.py`, `src/api/initialize.py` | do not edit |
| The two failure-exercise scripts | `tests/failure/force_dlq_arrival.py`, `tests/failure/redrive_and_verify.py` | run them; do not edit them |
| The recovery contract they exercise | `src/worker/use_cases.py`'s existing `ExceptionRecord` state machine | unchanged from Task 3.2; a delivery for an already-terminal identity is a safe replay |

## The one setting

`compose.yaml` already sets `COLDLINE_QUEUE_VISIBILITY_TIMEOUT_SECONDS` and
`COLDLINE_QUEUE_MAX_RECEIVE_COUNT` — both have no default in `ApiSettings`, so the initializer
would fail to start without them — but the starter's `COLDLINE_QUEUE_MAX_RECEIVE_COUNT` is `1`.
That is inside the published bounds (`queue_visibility_timeout_seconds` in `[5, 300]`,
`queue_max_receive_count` in `[1, 10]`), so the stack starts fine; it is still the wrong choice.

Both bounds are wide on purpose — there is no single correct number. But `maxReceiveCount=1`
gives a delivery zero tolerance for a single transient failure: the very next receive after an
unacknowledged one exceeds the bound and moves the message to the dead-letter queue immediately,
without ever giving the worker's own crash-recovery path a chance to succeed. Raise
`queue_max_receive_count` to a value that allows at least one retry.

## The two exercises

Run these against the live stack, in order, after `poe start`:

```shell
poe worker-stop      # the injector receives without acknowledging; a live worker would race it
poe inject-failure   # submits one reading, then exhausts the queue's own redrive budget
poe worker-start      # re-runs the initializer, then starts the worker; both are required
poe redrive           # resubmits every dead-lettered message, waits for each to finish, then delivers the newest once more
```

`poe inject-failure` submits a fresh reading each run and prints its exception id, the
configured `maxReceiveCount`, and the resulting dead-letter queue depth. `poe redrive` moves
every message in the dead-letter queue back to the main queue, not only the newest one, so a
message left behind by an earlier attempt or by another exercise is recovered too. It prints the
exception id of the most recently sent message, normally the one `poe inject-failure` just
printed, with its final state, which must be `COMPLETED`, and a `replay` object: after completion it
delivers that same job once more and confirms the record's `state`, `updated_at`, and `summary`
did not change. Any older exception ids it redrove are listed under `also_redriven`. It exits `0`
only when every redriven exception reached `COMPLETED` and that second delivery was consumed and
left the record unchanged. A redriven exception the API no longer has a record for, for example
after `poe reset-baseline`, is shown as `MISSING` and listed under `missing`: an older one like
that does not fail the run, but a missing reported exception does. A `dead_letter_queue_depth` above `1` from `poe inject-failure`, or a
non-empty `also_redriven`, means earlier messages were still waiting. If a run goes wrong, run
the two exercises again; each run creates a new exception.

## Commands

```shell
poe queue-contract   # the automated redrive-budget and dead-letter checks
poe verify           # the full public student verification path
```

## What the checks verify

| Check | What it looks at |
|---|---|
| `test_postgres_and_sqs_adapters_preserve_runtime_contracts` | Publish/receive/claim/acknowledge against a fresh exercise queue, an explicit dead-letter redrive once `maxReceiveCount` is exceeded, and — separately — that the *deployed* `coldline-exception-jobs` queue tolerates one unacknowledged receive without dead-lettering it |

## Student-editable paths

- `compose.yaml`
- `submission.yaml`

Keep `src/adapters/queue/sqs.py`, every bootstrap and initializer module, and every test file
exactly as supplied. The transport swap from Redis Streams to SQS is a conformance-tested
adapter, not this Task's assignment; configuring its dead-letter policy correctly, and observing
what it does, is.
