# Ticket E2-13 — Supervisor and per-domain worker processes

**Epic:** 2 — Executor · **Phase:** 1A · **Order:** E2-13

## Goal
Run each domain plugin in its own operating-system process. End state: the binary
is a supervisor plus one worker per domain; a worker crash, hang or resource
exhaustion stops only its own domain; key material never enters a worker.

## SPEC refs
EXECUTOR.md §1.4 (static plugins, one worker binary per plugin), §1.5 (process
isolation), §13 (single-instance lease), §14 (recovery), §16 (per-worker storage),
§17 (supervisor and worker configuration); SPEC §6.2 (every executor runs every
domain).

## Depends on
Tickets E2-06, E2-07, E2-08 (pipeline, batch builder, recovery) — this ticket re-homes them
inside a worker.

## Key work
- Split the binary: a supervisor (frontier follower, lease, key, Settlement client,
  DA serving) and a worker `main` per plugin (`cmd/worker-wasm`) running the
  existing pipeline with one plugin.
- Local IPC between them (for example a Unix-domain socket): momenta in; unsigned
  commitments, bundle references and status out. The supervisor rejects malformed
  messages rather than repairing them.
- Supervisor restarts a failed worker and applies per-worker memory and CPU
  limits; the restarted worker resumes through the §14 recovery state machine.
- Move per-domain storage under the worker (`<storagePath>/<domainId>/`) and the
  lease and submission log under `<storagePath>/supervisor/`.
- Split configuration per §17; `submissionKeySource` is read only by the
  supervisor.

## Code references
- Executor `cmd/` entry point, `internal/` pipeline, batch and recovery packages
  (zeno-executor) [EXISTS].

## Acceptance criteria
- Killing a worker mid-batch leaves the supervisor and other workers running; the
  worker restarts and resumes from its persisted cursor with identical roots and
  `DAHash`.
- A worker that exceeds its memory limit is stopped and restarted without
  affecting other domains.
- No worker process holds or can read the executor key (checked by test).
- Replay determinism and every conformance suite are unchanged by the split.
