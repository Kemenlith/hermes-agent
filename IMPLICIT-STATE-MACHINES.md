# Implicit state machines

This file lists lifecycles whose phase is not one explicit state type. Each entry is a machine read in this tree. The checkout is commit `d97007c32d6340f7ba79ac1dfb5773cc00846448` on `Kemenlith/hermes-agent` `main`. That commit is also on `NousResearch/hermes-agent`. GitHub compare of this fork against upstream `main` reports zero commits ahead. Every finding below is unmodified upstream code. This branch has no local or customized application files to rank above them.

The published docs live in `website/docs`. This inventory sits at the repo root so it is not a Docusaurus page.

## Count

39 implicit machines.

## Ranked list

Risk and complexity first. There is no local machine to place above an upstream one of similar risk.

1. Kanban task card
2. Kanban task run
3. Cron job schedule record
4. Cron run outcome and in-process flight
5. Cron occurrence pending slot
6. Cron execution ledger
7. Cron delivery handoff queue
8. Gateway process lifecycle
9. Gateway delivery obligation ledger
10. Platform reconnect queue
11. Desktop secondary gateway registry
12. Desktop chat turn flags
13. TUI gateway detached session
14. Messaging adapter connection
15. Gateway session turn slot
16. MCP server connection
17. Background process
18. Async delegation unit
19. Credential pool entry
20. Agent turn control
21. Context compression cooldown
22. Session handoff between surfaces
23. Goal loop
24. Local model download job
25. Bot live delivery ticket
26. Messaging adapter busy-session guard
27. Dangerous-command approval
28. DM pairing
29. Hermes update lock
30. Hermes update receipt
31. TUI client handshake
32. TUI composer busy latch
33. Session transcript status
34. Full-text index rebuild
35. Desktop boot screen
36. Browser session health
37. Code kernel cell
38. Session heartbeat
39. In-session todo list

## Findings

### 1. Kanban task card

- Files and symbols. `hermes_cli/kanban_db.py` (`SCHEMA_SQL` table `tasks`, `current_run_id`, `claim_lock`, `worker_pid`). `hermes_cli/kanban_workflow.py` (`Column`, `Workflow`, `DEFAULT_WORKFLOW`, `DEFAULT_STATUSES`, `can_move`).
- Implicit representation. `tasks.status` is `TEXT`. `DEFAULT_STATUSES` is a frozenset of column keys plus `archived`. `Workflow.manual` lists human moves only. The module docstring says kernel semantics and the kernel-driven writes are not modelled. Claim, expiry, worker pid, start fingerprint, heartbeat, failure counter, and `current_run_id` sit beside the column.
- Inferred states. Triage, todo, scheduled, ready, running, blocked, review, done, archived. Running also means "a worker holds a claim." Blocked also means "the circuit breaker or a worker parked the card."
- Inferred transitions. Humans move through `can_move` (archive is always allowed). The dispatcher claims a ready card, spawns a worker, heartbeats, and writes running. Crash, timeout, and reclaim return the card. Parent completion can promote a child. A failure limit blocks the card. Review verbs complete, request changes, or reopen. Restore leaves `archived`.
- Illegal combinations. The column type does not mention the claim. A card can read `ready` while `current_run_id` still points at a live run, or `running` with a null worker and a null run. `done` can keep `claim_lock`. `completed_at` can be set on a non-terminal column. Direct SQL can store a status outside `DEFAULT_STATUSES`. `kanban_workflow.py` documents a quirk: a requested move from review to todo lands in ready.
- Why it matters. The board and the worker disagree. A later reclaim can clobber a live claim. Operators see a ready card that is still executing, or a running card that nobody owns.
- Confidence. High.
- Origin. Upstream.

### 2. Kanban task run

- Files and symbols. `hermes_cli/kanban_db.py` table `task_runs`, `_end_run`, `goal_run_status`. Comments on columns `status` and `outcome`.
- Implicit representation. Two free `TEXT` columns. The schema comment lists status as running, done, blocked, crashed, timed_out, failed, released. It lists outcome as completed, blocked, crashed, timed_out, spawn_failed, gave_up, reclaimed, or null while running. `_end_run` defaults status to the outcome string.
- Inferred states. Open attempt, succeeded, blocked, crashed, timed out, spawn failed, gave up, reclaimed, released.
- Inferred transitions. Claim inserts a running row. Heartbeats refresh `last_heartbeat_at`. Terminal writers set outcome, status, and `ended_at`, then clear `tasks.current_run_id`. A missing open row can be synthesized closed.
- Illegal combinations. Status and outcome are independent. A row can be `status=done` with `outcome=null`, or `status=running` with `ended_at` set. Outcome `completed` can be stored as status `completed`, which is not in the status comment. `worker_pid` can outlive the terminal status on purpose, so "ended" does not mean "process gone."
- Why it matters. Goal loops and the dispatcher read different columns. A closed run can look open, and a still-live worker can look finished. Retries then stack on a card that already has a worker.
- Confidence. High.
- Origin. Upstream.

### 3. Cron job schedule record

- Files and symbols. `cron/jobs.py` (`_normalize_job_record`, `_has_pause_marker`, `is_job_runnable`, `effective_job_state`, `is_terminal_job`, `_is_recoverable_error_job`, `_complete_job_record`, `_activate_job_record`).
- Implicit representation. A JSON object. `state` is a string. `enabled` is a bool. `paused_at` and `paused_reason` are extra pause markers. `next_run_at` is nullable. Display state is derived, not stored as the source of truth.
- Inferred states. Scheduled, paused, completed, error. Error on a recurring schedule is "stuck until next-run recompute," which `is_terminal_job` still treats as terminal.
- Inferred transitions. Create stores scheduled or paused. Pause sets markers and disables. Resume calls `_activate_job_record`. A one-shot completion calls `_complete_job_record`. A schedule that cannot compute a next run sets `state=error`. Self-heal rewrites a half-paused record.
- Illegal combinations. `enabled` true with `state=paused` or `paused_at` set. `enabled` false with `state=scheduled` and no pause marker (`effective_job_state` then displays paused). `state=completed` with `enabled` true. `state=error` with a future `next_run_at`. The comments describe these half-paused records as jobs that displayed paused and still fired, or displayed runnable and did not.
- Why it matters. The ticker and the list can disagree. A job fires after the operator paused it, or a recurring job stays dark after a schedule parse error.
- Confidence. High.
- Origin. Upstream.

### 4. Cron run outcome and in-process flight

- Files and symbols. `cron/jobs.py` (`last_status` writers near the job update that sets `ok`, `delivery_failed`, or `error`). `cron/scheduler.py` (`_running_job_ids`, `_running_since`, `_running_futures`, `_running_worker_pids`, `_running_fire_owners`, `_interrupted_job_ids`, delivery statuses `delivery_queued` and `blocked_config`).
- Implicit representation. Durable `last_status` is a free string. Liveness is several process-local sets keyed by profile home plus job id. An interrupt token is a set membership that must be written before `last_status`.
- Inferred states. Idle, claimed, running, interrupted, ok, error, delivery queued, delivery failed, blocked on config.
- Inferred transitions. The ticker claims, dispatches, and advances `next_run_at` before the run. Success, empty output, delivery failure, config block, inactivity timeout, and shutdown each write a different `last_status`. Interrupt tokens stop a late thread from overwriting `interrupted` with `ok`. Stale `last_status=error` can re-arm a recurring job.
- Illegal combinations. `last_status=ok` while the job id is still in `_running_job_ids` or `_interrupted_job_ids`. `last_status=interrupted` after a later run already succeeded, if the token leaks. `delivery_queued` with no in-flight set entry after a crash. Two profiles can share a bare job id. The host-wide union of ids is the wrong liveness answer for one home. The cron guide documents that mismatch.
- Why it matters. A finished run is reported ok after the user interrupted it. A dead run stays "running" until a force-release. Profile A's running job answers for profile B's idle job of the same id.
- Confidence. High.
- Origin. Upstream.

### 5. Cron execution ledger

- Files and symbols. `cron/executions.py` (`_initialize_schema`, `_TERMINAL_STATES`, claim and finish updates).
- Implicit representation. `status TEXT` with a SQL `CHECK` of claimed, running, completed, failed, unknown. `handoff_pending` is a separate integer. `started_at` and `finished_at` are nullable. The module says the ledger is an audit, not a retry queue, and terminal rows are immutable in the writers.
- Inferred states. Claimed, running, handed off, completed, failed, unknown (owner proved gone).
- Inferred transitions. Insert claimed. Promote to running when the worker starts, with or without `handoff_pending`. Finish sets completed or failed only from claimed or running. A dead owner with a matching fingerprint becomes unknown.
- Illegal combinations. The check rejects other status strings. It still allows `handoff_pending=1` on a terminal row, `running` with `finished_at` set, and `claimed` with `started_at` set. A fingerprint mismatch must not become unknown. The code treats a bad start-time read as "not proof of death."
- Why it matters. A second fire of the same scheduled instant, or a live worker marked unknown, drops or duplicates an occurrence.
- Confidence. High.
- Origin. Upstream.

### 6. Gateway process lifecycle

- Files and symbols. `gateway/run.py` `GatewayRunner._init_lifecycle_state` (`_running`, `_draining`, `_external_drain_active`, `_exit_cleanly`, `_exit_with_failure`, `_restart_requested`, `_signal_initiated_shutdown`, `_executor_closing`, and the restart-task flags). `gateway/status.py` `retained_gateway_state`, `gateway_state.json`. `gateway/run_startup.py` and `gateway/run_shutdown.py` write `starting`, `running`, `degraded`, `startup_failed`, `draining`, `stopped`.
- Implicit representation. Many booleans on the runner, plus a JSON `gateway_state` string, `desired_state`, `exit_reason`, and a pid record. `retained_gateway_state` reinterprets a dead process. `_DRAINABLE_GATEWAY_STATES` is only `running` and `degraded`.
- Inferred states. Starting, running, degraded, draining, external drain (still up, new turns refused), stopping, stopped, startup failed, restart requested, signal-killed (must not persist stopped).
- Inferred transitions. Boot writes starting, then running or degraded if a platform parked. Drain sets `_draining` and may write `draining`. An external marker sets `_external_drain_active` and clears when the marker goes away. Stop, restart, SIGTERM, and the watchdog take different branches. A signal with no planned-stop marker must leave the file as running so container boot starts again.
- Illegal combinations. `_exit_cleanly` and `_exit_with_failure` can both be true. `_running` and `_draining` and `_external_drain_active` combine freely. Disk can say `running` while the pid is gone. Disk can say `stopped` while the process still accepts turns. `desired_state=stopped` hides a leftover `startup_failed` only in the derived reader.
- Why it matters. The dashboard, `hermes gateway status`, and a supervisor disagree. A drain that persists `stopped` prevents the next boot. A dead process that still looks running blocks update and restart.
- Confidence. High.
- Origin. Upstream.

### 7. Messaging adapter connection

- Files and symbols. `gateway/platforms/base.py` (`_mark_connected`, `_mark_degraded`, `is_connected`, `send_path_degraded`, `DEGRADED_STATUS_MESSAGE`, fatal-error fields).
- Implicit representation. `_running` is the connection bool. `is_connected` returns that bool. A separate runtime status string is `connected` or `retrying`. Fatal error code, message, and `retryable` are nullable fields. `send_path_degraded` defaults to false and adapters override it.
- Inferred states. Down, connected, connected but unproven (degraded / retrying), fatal and retryable, fatal and not retryable.
- Inferred transitions. `connect` calls `_mark_connected`, which clears fatal fields and writes connected, or retrying when the send path is degraded. Disconnect and fatal handlers clear `_running` and publish an error. Reconnect loops live in each adapter.
- Illegal combinations. `_running` true with a fatal message still set, if a path skips `_mark_connected`. `is_connected` true while the published platform state is `retrying`. Fatal retryable true with `_running` false and no reconnect task.
- Why it matters. The Channels page shows a bot as up while polling has not completed a round trip, or as down while a retry is in flight. Inbound messages queue against an adapter that will not deliver.
- Confidence. High.
- Origin. Upstream.

### 8. Gateway session turn slot

- Files and symbols. `gateway/session_state.py` (`TurnState`, `SessionState`). `gateway/run.py` `_AGENT_PENDING_SENTINEL` (defined on the runner, compared from slash handlers). `_running_agents` is a view over `TurnState.agent`.
- Implicit representation. Occupancy is one field with three meanings: `None` (idle), the pending sentinel, or an `AIAgent`. `started_ts == 0` means not running. `event`, `ctx`, `lease`, and `busy_ack_ts` are extra presence flags. `run_generation` on `PersistentState` is a counter that must never reset.
- Inferred states. Idle, pending (admitted, agent not built), running, busy-acked, leased.
- Inferred transitions. Admit stores the sentinel and a timestamp. The turn installs the agent, event, and context. A busy redirect re-anchors event and context. `TurnState.clear` drops them except lease tokens, which release once per generation. Shutdown walks `_running_agents`.
- Illegal combinations. An agent with `started_ts == 0`. A lease with `agent is None`. The sentinel with a live `ctx` from a previous turn. Two generations in `lease_tokens` is briefly legal. A stale generation that clears the new lease is the bug the dict exists to prevent.
- Why it matters. Slash status, `/stop`, and the drain count disagree about whether a chat is busy. A cleared slot can drop a successor's lease.
- Confidence. High.
- Origin. Upstream.

### 9. MCP server connection

- Files and symbols. `tools/mcp_tool.py` `MCPServerTask` (`_ready`, `_shutdown_event`, `_reconnect_event`, `_reconnecting`, `_reconnect_retries`, `_session_proven`, `_ever_connected`, `_error`, `_park_reason`, `_was_parked`, `_auth_park`, `_teardown_race`, `_permanent_grace_used`). `tools/mcp_tool_health.py` `_mark_session_proven`.
- Implicit representation. The class comment calls this a run state machine. The phase is events, bools, a retry counter, an optional error, and an optional park reason. No enum.
- Inferred states. Never connected, connecting, ready and unproven, ready and proven, reconnecting, parked (auth or transport), shutting down.
- Inferred transitions. The long-lived task brings up stdio or HTTP. A proven keepalive or a successful tool call clears the retry budget and park reason. Drops charge `_reconnect_retries` until park. OAuth failure arms `_auth_park`. Shutdown sets `_shutdown_event`. A teardown that fails in-flight calls marks the next connect as race recovery. One permanent failure on a proven session gets a single grace reconnect.
- Illegal combinations. `_ready` set with `session is None`. `_session_proven` with `_park_reason` set. `_reconnecting` and `_shutdown_event` both set. `_error` set on a ready task. `_was_parked` after a successful prove (the prove path clears it, the fields still allow it).
- Why it matters. Tool calls hang on a parked server, or a flapping server hot-restarts forever. Cron preflight cannot tell a network blip from a permanent auth park if `_park_reason` is stale.
- Confidence. High.
- Origin. Upstream.

### 10. Background process

- Files and symbols. `tools/process_registry.py` `ProcessSession` (`exited`, `exit_code`, `pid`, `process`, `completion_reason`, `termination_source`, `_finish_claimed`, `_scope_stop_pending`, `detached`, `persist_on_release`).
- Implicit representation. `exited` is the finished bit. `completion_reason` defaults to `exited` even while the process is running. The comment lists exited, killed, lost, failed_start, already_exited. Return payloads use another string set: running, exited, killed, timeout, interrupted, error, not_found, handed_off. `pid is None` covers both "not started" and "backend has no host pid."
- Inferred states. Starting, running, watcher attached, finished, killed, lost, failed to start, detached after crash recovery, scope-stop still owed.
- Inferred transitions. Spawn fills pid and pipes. The reader ingests output until exit. `_move_to_finished` claims the finish once and publishes one completion. Kill sets `completion_reason=killed`. A checkpoint reload sets `detached` with no pipe. `persist_on_release` skips agent-close kill. Handoff changes `owner_task_id`.
- Illegal combinations. `exited` false with `exit_code` set. `completion_reason=killed` while `exited` is false. `exited` true while the systemd scope is still alive (`_scope_stop_pending`). `detached` true with a live `process` handle. Two finishers, if `_finish_claimed` is ignored.
- Why it matters. A completion notice fires twice, or never. A kill receipt misses the last output. A recycled pid is signalled when the start-time fingerprint was not stored. Session end kills a process the user asked to keep, or leaves an orphan the registry calls exited.
- Confidence. High.
- Origin. Upstream.

### 11. Async delegation unit

- Files and symbols. `tools/async_delegation.py` (`_ACTIVE_STATES`, `_records`, `_sweep_stale_locked`, `push_task_failure_notice`).
- Implicit representation. Each record is a dict. `status` is a string. Active means `running` or `stalling`. Stall metadata is `_interrupted_at` and `_stall_quiet_seconds`. Progress and completion are queue events, not a status type.
- Inferred states. Queued in the pool, running, stalling, stalled past grace, completed, failed, interrupted, unknown after a lost durable write.
- Inferred transitions. Dispatch inserts running. The monitor moves a quiet running unit to stalling, then to expired after grace. Completion persists a result and enqueues an event. A child failure notice keeps the batch record active and sends an event whose own `status` is `running`. A restart can load a record the completion write missed.
- Illegal combinations. `stalling` without `_interrupted_at`. A terminal status still listed in `_ACTIVE_STATES` if a writer typos the string. A failure-notice event with `status=running` while that child already failed. Durable row missing while the memory record is completed.
- Why it matters. The parent waits on a dead child. A stall interrupt hits a unit that already finished. After restart the same unit is reported unknown and the work is not in the parent's transcript.
- Confidence. High.
- Origin. Upstream.

### 12. Credential pool entry

- Files and symbols. `agent/credential_pool.py` (`STATUS_OK`, `STATUS_EXHAUSTED`, `STATUS_DEAD`, `_TERMINAL_AUTH_REASONS`, `EXHAUSTED_TTL_*`). Readers in `hermes_cli/auth.py` and `hermes_cli/inventory.py`.
- Implicit representation. `last_status` is a string constant, not an enum. Exhausted entries also carry `last_status_at`, an error code, and a reason. Dead is terminal until a token rewrite. Cooldown length depends on the HTTP code and on whether the entry is the only live key. Those rules are branches, not states.
- Inferred states. Usable, exhausted until a deadline, dead, suppressed by a separate auth flag.
- Inferred transitions. A 429, 402, or transient 401 sets exhausted and a TTL. Provider `reset_at` overrides the TTL. Terminal OAuth reasons set dead. A fresh device-code save clears dead. Manual dead entries prune after 24 hours. A sole key uses a shorter cooldown.
- Illegal combinations. `last_status=exhausted` with no timestamp, so the TTL math has nothing to read. `last_status=dead` while the token file was already replaced. `last_status` set to some other word, which both the ok-path and the dead-path skip. Exhausted and dead both set across memory and disk. `hermes_cli/auth.py` has a merge path because those copies drift.
- Why it matters. Rotation skips the only working key, or keeps using a revoked token. The model picker shows a benched key as live.
- Confidence. High.
- Origin. Upstream.

### 13. Agent turn control

- Files and symbols. `agent/conversation_loop.py` (the loop `agent/AGENTS.md` describes). `agent/turn_retry_state.py` `TurnRetryState`. `agent/turn_facade_lease.py` (`_interrupt_requested`, `_hard_interrupt_requested`). `agent/turn_truncation.py` (`_budget_grace_call`). `agent/turn_recovery.py` (`abort_turn_on_interrupt`, `interruptible_backoff_sleep`).
- Implicit representation. The phase is the call stack across `turn_*.py` siblings. Recovery is a dataclass of one-shot bools plus four `restart_with_*` signals. Interrupt is a bool and an event. The iteration budget has a one-shot grace flag. Loop locals hold `retry_count`. The result dict uses `completed` and `interrupted` bools.
- Inferred states. Preflight, requesting, waiting on the provider, backing off, in a tool round, compacting, restarting the same iteration, grace call after length limit, aborting, finished.
- Inferred transitions. Each sibling returns into the loop. A retry guard fires once and may set a restart signal. Interrupt during backoff either preserves a redirect or returns `interrupted=True`. A length cut can set `_budget_grace_call` for one extra request. Hard cancel clears the interrupt after the abort result is built.
- Illegal combinations. More than one `restart_with_*` flag true. `_interrupt_requested` true together with a restart that assumes the user still wants the same call. `completed` and `interrupted` both true. Grace armed after an interrupt already won. A second auth retry after the matching `*_attempted` bool was left false by an early return.
- Why it matters. The user stops a turn and the loop issues another provider call. A fallback rebuilds messages and then also compresses them. The transcript stores a success flag on an aborted turn.
- Confidence. Medium. The sequencer is real control flow. The scattered flags are the implicit part.
- Origin. Upstream.

### 14. Context compression cooldown

- Files and symbols. `agent/conversation_compression.py` (cooldown fields listed around `_COMPRESSOR_COOLDOWN_STATE_FIELDS`, `_rollback_durable_cooldown`). `gateway/session_state.py` `PersistentState.hygiene_failure_streak`.
- Implicit representation. Cooldown deadline, last error, persist-failed flag, consecutive timeout count, consecutive truncation count, and a "verify the compact cleared the threshold" bool. Gateway hygiene keeps a separate streak because each run builds a fresh agent and zeroes the in-agent counter. A DB row mirrors the deadline.
- Inferred states. Idle under threshold, compacting, summary cooling down, timeout ladder climbing, rollback after a failed commit, hygiene escalating across runs.
- Inferred transitions. Overflow starts a compact. Summary failure sets the deadline and the error. A repeated stall falls through to a deterministic summary. Success clears the streak. Rollback restores the previous deadline from a snapshot. Gateway restart drops the in-memory streak and keeps the DB deadline.
- Illegal combinations. A future cooldown with a zero failure count. `_cooldown_persist_failed` with a deadline the DB does not hold. Hygiene streak high while the fresh agent's counter is zero, which is expected, and the inverse after a restart, which is the documented gap. Two compactors writing one session's cooldown row.
- Why it matters. Compression retries on a flat interval and stalls the turn, or stays suppressed after the provider has recovered. A rollback can overwrite a newer cooldown a concurrent claim just wrote. The comment on the lock exists for that race.
- Confidence. Medium.
- Origin. Upstream.

### 15. Goal loop

- Files and symbols. `hermes_cli/goals.py` `GoalState`, `GoalManager` (`status` comment `active | paused | done | cleared`, `last_verdict`, `paused_reason`, failure counters, wait barrier fields).
- Implicit representation. `status` is a plain string. Verdict is another string: done, blocked, continue, wait, skipped. Pause reason, turn counters, and consecutive parse or transport failures are separate. Cleared rows stay in the DB. Loaders treat some cleared rows as absent.
- Inferred states. No goal, active, paused (user or budget or judge transport), waiting on a pid or a process-registry session, done, cleared.
- Inferred transitions. `/goal` stores active. The judge returns a verdict and may pause, continue, or finish. Transport failures auto-pause. A wait barrier parks until a process exits or a watch pattern hits. Clear keeps an audit row. Turn budget can pause with a reason.
- Illegal combinations. `status=active` with `turns_used` past `max_turns`. `status=done` with `paused_reason` set. `last_verdict=continue` on a paused or cleared goal. `status` set to any other word, which the status line prints raw. Wait fields set while status is active, so the loop and the barrier disagree.
- Why it matters. The session keeps re-entering a finished goal, or stops a goal the user still sees as active. A cleared row and a new active row can both exist if a writer updates the wrong one.
- Confidence. High.
- Origin. Upstream.

### 16. Local model download job

- Files and symbols. `hermes_cli/web_routers/local_models.py` (`_job`, `_DOWNLOAD_PHASES`, `_step`, `_job_view`, `_runtime_progress_hook`).
- Implicit representation. An in-memory dict. `status` is running, paused, done, or error. `phase` is a free step name. `pause_requested` and `can_pause` are derived from a threading event plus status plus phase. Byte counters and `error` are nullable.
- Inferred states. Starting, installing runtime, downloading runtime, unpacking, verifying, downloading a model, pause requested, paused, done, error.
- Inferred transitions. `_job` starts at running and starting. `_step` changes phase. A pause event raises `DownloadPaused` and stores paused. Resume continues inside the download phase. Hooks map engine stages to phase strings. Failure sets error.
- Illegal combinations. `status=done` with `phase=downloading`. `status=paused` outside a download phase, so resume has no plan. `status=error` with `error=None`. `pause_requested` while status is already paused. `can_resume` true when the running-entry dict lost its `resume` callback.
- Why it matters. The desktop progress card shows a live download that has failed, or a pause button that cannot resume. A mid-download model can be treated as installed if a caller reads files instead of this job.
- Confidence. High.
- Origin. Upstream.

### 17. Bot live delivery ticket

- Files and symbols. `tools/bot_live_delivery.py` (`_TERMINAL`, `_PENDING`, `_ticket_shape_error`, claim and settle updates). `tools/bot_mode_dm.py` branches on queued, claimed, settled, ambiguous.
- Implicit representation. A JSON file. `status` is a string. Writers know queued, claimed, and the terminal set settled, failed, cancelled, ambiguous. Shape checks reject other strings. `claimed_at` is optional on the record.
- Inferred states. Queued, claimed, settled, failed, cancelled, ambiguous, unreadable.
- Inferred transitions. Enqueue writes queued. A matching owner claims. Settle requires claimed, unless the row is already terminal. Cancel requires queued. A poller rescans the directory. A bad file warns once and stays on disk.
- Illegal combinations. `claimed` without `claimed_at`. `settled` with an empty reply. Terminal status that a later claim still mutates if the status check is skipped. Two files for one delivery id. `ambiguous` is terminal in the set and also a DM branch next to queued and claimed.
- Why it matters. A bot turn is delivered twice, or the reply is dropped and the user sees silence. One corrupt ticket can wedge the profile scan if a reader raises instead of using the shape check.
- Confidence. High.
- Origin. Upstream.

### 18. Messaging adapter busy-session guard

- Files and symbols. `gateway/platforms/base.py` (`_active_sessions`, `_pending_messages`, the handlers that set and pop the guard). `gateway/AGENTS.md` describes the second guard in the runner.
- Implicit representation. Busy is "session key is in `_active_sessions`" (the value is an `asyncio.Event`). A second dict holds at most one pending `MessageEvent`. The runner separately tracks the live agent. Both must agree before `/stop`, `/approve`, and friends run inline.
- Inferred states. Idle, agent running, message queued behind the run, command waiting on the guard, stale task.
- Inferred transitions. A turn inserts the guard. Inbound text while the key is active merges into `_pending_messages`. Turn end pops the pending event and starts it. A stale task drops the guard and the pending message. Shutdown must not leave the key behind.
- Illegal combinations. A pending message with no guard. A guard whose event is already set and whose agent is gone. Two logical messages collapsed by `merge_text=True` when the second was a command. Runner thinks the agent is running while the adapter key is absent, so the command waits for a background task that will never see it.
- Why it matters. An approval reply sits in the queue until the tool times out. A follow-up is dropped on a stale task. Two user messages become one.
- Confidence. Medium. The base adapter and the gateway guide were read. Each platform's override was not.
- Origin. Upstream.

### 19. Dangerous-command approval

- Files and symbols. `tools/approval.py` (return shape `status=pending_approval`, `approval_pending=True`). `gateway/session_state.py` `PersistentState.approvals` and `update_prompt_pending`.
- Implicit representation. The tool result is a dict of flags. The session stores an optional approvals dict (`command`, `pattern_key`) and a bool for an update prompt. Idle versus waiting is "those fields are populated," plus whether a turn is in `TurnState`.
- Inferred states. Not waiting, waiting on a command pattern, waiting on an update prompt, approved, denied, expired because the turn ended.
- Inferred transitions. A dangerous command returns pending and the gateway stores the pattern. `/approve` and `/deny` consume it. Conversation boundaries clear approvals and the update prompt through the security funnel. A CLI panel can resolve the same decision without the pending dict.
- Illegal combinations. `approval_pending` true in a tool result while `persistent.approvals` is null. An approvals dict left after the agent is idle. `update_prompt_pending` true with no turn to answer it. Both an approval and an update prompt set, so one reply answers the wrong waiter.
- Why it matters. The user approves a command that already timed out, or the next message is eaten as the approval. A boundary that fails to clear the dict lets a later chat reuse the pattern.
- Confidence. Medium.
- Origin. Upstream.

### 20. Session transcript status

- Files and symbols. `hermes_state_sessions.py` (`classify_session_status`, `SESSION_STATUS_*`, `_ERROR_FINISH_REASONS`, `session_lifecycle_statuses`). Consumer in `cron/scheduler.py` (a cron run whose last row classifies as interrupted, error, or empty must not look healthy).
- Implicit representation. There is no status column. The label is computed from the last message's role, a bool for tool calls, and `finish_reason`. Unknown shapes return complete.
- Inferred states. Empty, complete, interrupted, error.
- Inferred transitions. A new user row moves a session to interrupted until an assistant row without tool calls lands. An error finish reason forces error. Compression, branch, and delegate rows change which row is last.
- Illegal combinations. The function always returns one of four strings. The illegal part is the mapping. A trailing user row is interrupted even when the human simply has not been answered yet and the turn is live. An assistant row with leftover `tool_calls` is interrupted after the tools succeeded, if a later tool row was not written. A failed turn whose `finish_reason` is outside the frozenset looks complete.
- Why it matters. Session pickers show a live chat as interrupted, or a crashed cron session as complete. The scheduler then delivers a pathological transcript as a successful job. The cron comment says unknown values fail open.
- Confidence. High.
- Origin. Upstream.

### 21. Hermes update receipt

- Files and symbols. `hermes_cli/update_cmd.py` (`record_stage` for `plan`, `snapshot`, `apply`). `hermes_cli/update_custody.py` (commit point means an `apply` stage exists). `hermes_cli/update_finish.py` (restart and verify follow apply). `hermes_cli/AGENTS.md` names the pipeline plan, snapshot, apply, restart, verify, report.
- Implicit representation. The receipt is a list of stage records with string names and string outcomes (`success`, `failed`, `skipped`). There is no stage enum. "How far did this update get" is "which names appear."
- Inferred states. Not started, planned, snapshotted, applied (commit point), restarting the fleet, verifying, finished with owed follow-ups, failed before commit, failed after commit.
- Inferred transitions. Plan runs first. Snapshot can be skipped. Apply is the commit point. Later failures still exit 0 and record follow-ups. A handoff file resumes a partial run in a new process.
- Illegal combinations. `apply=success` with `plan=failed`. Two snapshots with different outcomes. A receipt past the commit point with no snapshot id. Restart bookkeeping that does not cover every runtime the plan listed. The fleet verifier treats that gap as a missed restart.
- Why it matters. A second update starts on a half-applied tree. The app reports success while a gateway is still on the old code. Rollback uses a snapshot the receipt marked skipped.
- Confidence. Medium. Stage names were taken from the writers and the CLI guide. Not every sibling was line-read.
- Origin. Upstream.

### 22. Desktop boot screen

- Files and symbols. `apps/desktop/src/global.d.ts` `DesktopBootProgress` (`phase: string`, `running`, `error`, `retryable`, `updateHold`). `apps/desktop/src/store/boot.ts` (`$desktopBoot`, `applyDesktopBootProgress`, phases `renderer.init`, `renderer.boot.retry`, `renderer.ready`, `renderer.error`).
- Implicit representation. Phase is an open string. Running, error, retryable, cloud-down, and update-hold are separate fields. The store merges a late progress event so a null error does not wipe a terminal failure.
- Inferred states. Starting, retrying, ready, failed, failed and retryable, blocked on an update hold.
- Inferred transitions. The main process emits progress. The renderer stores it. A boot failure sets the error and stops `running`. Retry emits `renderer.boot.retry`. Ready emits `renderer.ready`. An update hold blocks the local backend.
- Illegal combinations. `running` true with a non-null `error`. `phase=renderer.ready` with `running` false. `phase=renderer.error` with `running` true. `retryable` true on a permanent auth failure. `updateHold` set on a boot that also claims ready.
- Why it matters. The window sits on a spinner after failure, or drops the error and looks healthy while the backend never started. A retry loop hits a permanent 401.
- Confidence. Medium.
- Origin. Upstream.

### 23. Browser session health

- Files and symbols. `tools/browser_tool.py` (`_suspect_browser_sessions`, `_cleanup_failures`, `_cleanup_running`, `_BrowserSessionBackend.mark_suspect`, `ensure_healthy`). Cleanup lives in `tools/browser_tool_lifecycle.py`.
- Implicit representation. A session is healthy unless its key is in the suspect map. A janitor failure count is a second map. `_cleanup_running` is a bool beside the cleanup thread. There is no session-phase enum in the tool module.
- Inferred states. Absent, live, suspect after a command timeout, recycling, janitor failing, force-reaped after three failures.
- Inferred transitions. Timeout calls `mark_suspect` without the session lock. The next command pops the flag and tears the session down before reuse. The flag is popped before teardown so close cannot recurse. Repeated janitor errors force a reap. Process exit runs emergency cleanup.
- Illegal combinations. A suspect flag for a key that no longer exists. `_cleanup_running` false while the thread is alive. Failure count left behind after a successful command. A new session created while teardown of the old one is still closing the browser.
- Why it matters. The next browser call hangs on a wedged page, or a recycle deletes a session the user just opened. The janitor leaks browsers after the flag and the thread diverge.
- Confidence. Medium.
- Origin. Upstream.

### 24. Code kernel cell

- Files and symbols. `tools/code_kernel.py` (cell payload `status` of ok, exit, error, kernel-eof, protocol-error). Host wait status timeout and interrupted. `tools/code_kernel_remote.py` adds no-result.
- Implicit representation. Two status strings. The host status is how the wait ended. The payload status is how the cell ended. Callers branch on both.
- Inferred states. Kernel starting, cell running, cell ok, cell raised, user interrupted, wait timed out, kernel died, protocol desync.
- Inferred transitions. A cell request is queued. The kernel replies or the wait hits a timeout or interrupt. EOF and protocol errors fail the cell. Exit status is a `SystemExit` inside the cell.
- Illegal combinations. Host `timeout` with payload `ok` if a late reply is applied anyway. Host `ok` with payload `kernel-eof`. Payload `error` and a zero exit code in the combined result dict. A remote `no-result` treated as success by a caller that only checks the host status.
- Why it matters. The model is told the code ran when the kernel had already died. A timeout leaves the kernel busy, so the next cell runs against a live previous cell.
- Confidence. Medium.
- Origin. Upstream.

### 25. Session heartbeat

- Files and symbols. `hermes_cli/heartbeat.py` `HeartbeatState` (`status` comment `active | paused | cleared`), `load_heartbeat`, `HeartbeatManager`.
- Implicit representation. `status` is a string on a JSON blob in session meta. Due logic also requires a prompt and a positive interval. Cleared is stored and then hidden on load.
- Inferred states. None, active, paused, cleared, unparseable.
- Inferred transitions. Set stores active. Pause and clear rewrite status. The driver calls `due_prompt` only while the session is idle. Fire bumps `fire_count` and `last_fired_at`. A parse error makes `store_has_active_heartbeat` return true so the sweeper does not skip the row.
- Illegal combinations. `active` with an empty prompt or a zero interval (stored, never due). A status other than the three words. `paused` with `last_fired_at` in the future. Cleared in the file and a second active object only in memory.
- Why it matters. An idle session never re-enters, or a paused heartbeat still injects a user message and breaks the turn the human is typing.
- Confidence. High.
- Origin. Upstream.

### 26. In-session todo list

- Files and symbols. `tools/todo_tool.py` (`VALID_STATUSES`, `TodoStore._validate`, `_normalize_order`).
- Implicit representation. Each item's `status` is a string checked against a set. Unknown values become `pending`. The store does not record a previous status. Several items may be `in_progress`. Order is rewritten only for a flat list.
- Inferred states. Pending, in progress, completed, cancelled. The list as a whole has no phase.
- Inferred transitions. The model replaces or merges the whole list. There is no allowed-edge table. Compression re-injects the list from the store.
- Illegal combinations. Two or more `in_progress` items. A cancelled item moved back to pending by a later write with no history. A parent id that pointed at a removed item (the sanitizer drops it, so the write succeeds). An item id of `?` after a non-dict element.
- Why it matters. The checklist UI shows several current steps. A merge resurrects a finished item. This is low severity next to the job and gateway machines. It is still a workflow with no transition type.
- Confidence. High.
- Origin. Upstream.

### 27. Cron occurrence pending slot

- Files and symbols. `cron/occurrences.py` (`pending_slot_stamp`, `unclaimed_pending_slot`). The job field is `pending_slot`.
- Implicit representation. A dict `{scheduled_at, at, by}` on the job, or absent. The due scan stamps it when it advances `next_run_at`. `claim_job_for_fire` clears it. A schedule rewrite drops it.
- Inferred states. No open slot, slot stamped and unclaimed, slot claimed, slot restored once after a dead owner.
- Inferred transitions. Tick stamps, then claims. A crash after the advance and before the claim leaves the slot. A later scan restores that instant once when the owner is gone or the lease has expired. Catch-up policy then decides whether to fire.
- Illegal combinations. A pending slot on a one-shot. A malformed `scheduled_at`. A slot whose `by` is this process while the job is not in the running set (treated as orphaned). A slot restored more than once.
- Why it matters. The tick moves `next_run_at` before dispatch so a crash cannot double-fire. Without the slot, that same crash drops the occurrence. The comment cites a window where the restarted scan sees a future time and nothing ran.
- Confidence. High.
- Origin. Upstream.

### 28. Cron delivery handoff queue

- Files and symbols. `cron/delivery_queue.py` table `deliveries`, `_TERMINAL`, table `delivery_tombstones`.
- Implicit representation. `status TEXT` with a SQL check: pending, delivering, delivered, failed, unknown, suppressed. Owner pid and start time sit beside the status. A tombstone stores the terminal status after the row is pruned.
- Inferred states. Pending, delivering, delivered, failed, unknown, suppressed.
- Inferred transitions. The worker enqueues pending. The gateway claims the next row and sets delivering. Send finishes as delivered, failed, or suppressed. A dead owner becomes unknown. Unknown is not retried.
- Illegal combinations. The check rejects other strings. It still allows `delivering` with `finished_at` set, and a tombstone whose status disagrees with a live row of the same execution id. A second claim of a terminal row must no-op. The writers guard that. The schema does not.
- Why it matters. A restart can send the same cron brief twice, or mark it unknown and never send it.
- Confidence. High.
- Origin. Upstream.

### 29. Gateway delivery obligation ledger

- Files and symbols. `gateway/delivery_ledger.py` (`record_obligation`, `mark_attempting`, `mark_delivered`, `mark_failed`). Column `state` on the ledger table.
- Implicit representation. `state` is a free `TEXT` column. The module docstring is the transition list: pending, attempting, delivered, failed, abandoned. Retry time, attempt count, and owner pid are separate columns. Ledger errors must not block the send.
- Inferred states. Not recorded, pending, attempting, delivered, failed, abandoned.
- Inferred transitions. Record pending before send. Mark attempting immediately before the await. Success becomes delivered. A hard rejection becomes failed. A crash while attempting is ambiguous: the platform may already have the message, so recovery resends with a visible recovered marker. Caps and stale rows become abandoned.
- Illegal combinations. `attempting` with no owner pid after a crash. `delivered` and a later resend of the same id. `pending` after a partial stream already reached the chat. Two reconnect paths claiming one row. The docstring calls that out.
- Why it matters. A crash between the model reply and the platform ack drops the answer, or a recovery posts it twice.
- Confidence. High.
- Origin. Upstream.

### 30. Platform reconnect queue

- Files and symbols. `gateway/run_adapters.py` (`_reconnect_queue_entry`, `_queue_retryable_fatal_platform`, `_failed_platforms`).
- Implicit representation. A dict per platform: `attempts`, `next_retry`, `queued_at`, `credential_claim`, `listener_claim`, `predecessor`. Membership in `_failed_platforms` is the queued bit. `next_retry` at infinity is the pause. The live adapter map is a second set.
- Inferred states. Connected, queued and waiting, queued and retrying, paused, dropped.
- Inferred transitions. A retryable fatal queues the platform before disconnect. A watcher connects with `is_reconnect`. Success installs the new adapter and drops the queue entry. Pause sets a far `next_retry`. Resume resets attempts. A dead watcher is restarted when a platform is already queued.
- Illegal combinations. The same platform in `adapters` and `_failed_platforms`. `attempts` climbing while `next_retry` is still in the past and the watcher is dead. A predecessor held after install, so inbound dedup state is dropped.
- Why it matters. The gateway looks up and receives nothing. A failed reconnect can also leak the previous adapter's file descriptors. Cold boot can drop an offline queue if it is not marked as a reconnect.
- Confidence. High.
- Origin. Upstream.

### 31. Desktop secondary gateway registry

- Files and symbols. `apps/desktop/src/store/gateway.ts` `Secondary` (`wantOpen`, `reconnecting`, `reconnectAttempt`, `reconnectTimer`, `retained`, `relayRetainCount`, `retiredByPool`, `pendingConnectionRedial`, `activationLeaseUntil`, `livenessProbeFailures`, `stalledDials`, `connectPromise`).
- Implicit representation. Each pooled backend is a struct of booleans, counters, timers, and a promise. The socket's own `connectionState` is a separate explicit union. The comments say `retained` must not be honored by the live-work pruner, and `retiredByPool` must block a wake-nudge redial.
- Inferred states. Closed, dialing, open, reconnect backoff, activation lease, pending redial, retired by the pool, relay-retained, probe-failing.
- Inferred transitions. Open sets `wantOpen` and dials. Close or error starts backoff unless pruning cleared `wantOpen`. Foreground and relay retains pin the socket. Pool retirement blocks the nudge. An activation lease stops the pruner from disposing a switch target mid-dial. A liveness streak force-closes.
- Illegal combinations. `wantOpen` false while a reconnect timer is still armed. `retiredByPool` true and a focus nudge dials anyway. Dispose while `activationLeaseUntil` is in the future. `connectionState` open on a socket that no longer answers. `retained` true used as a prune keep, which the comment says pins every warmed socket forever.
- Why it matters. Profile clicks die mid-dial. Bot relay opens and closes a socket every tick. A tile spins because its backend was pruned.
- Confidence. High.
- Origin. Upstream.

### 32. Desktop chat turn flags

- Files and symbols. `apps/desktop/src/app/types.ts` `ClientSessionState` (`busy`, `awaitingResponse`, `turnLive`, `needsInput`, `interrupted`, `adoptedRunningTurn`, `sawAssistantPayload`, `streamId`, `heartbeatSettledStreamId`, `interimBoundaryPending`, `turnStartedAt`, `reasoningEffortPending`).
- Implicit representation. The turn phase is a cluster of booleans and nullable ids. `turnStartedAt` is a clock seeded at submit. The comment says it is not proof the turn is live. `turnLive` is that proof. A heartbeat can settle a bubble before `message.complete`.
- Inferred states. Idle, optimistic submit, live stream, awaiting the first token, needs input, interrupted and settling, adopted remote turn, heartbeat settled early.
- Inferred transitions. Send sets busy and the clock. `message.start` sets `turnLive`. Approval or clarify sets `needsInput`. Cancel sets `interrupted`. A `running=false` heartbeat may settle the stream before the complete frame. The next start or complete clears `heartbeatSettledStreamId`.
- Illegal combinations. `busy` true and `turnLive` false after the watchdog should have settled. `needsInput` true after the turn ended. `turnStartedAt` set with `turnLive` false treated as a live turn. `heartbeatSettledStreamId` set and a later complete that appends a second bubble.
- Why it matters. The composer accepts a second send. The sidebar dot stays on "analyzing". A reordered heartbeat duplicates the assistant message.
- Confidence. High.
- Origin. Upstream.

### 33. TUI gateway detached session

- Files and symbols. `tui_gateway/server.py` `_detached_ws_transport`. `tui_gateway/session_lifecycle.py` (`_ws_session_is_detached`, `_ws_session_is_orphaned`, `_interrupt_session_turn`). Session dict keys `running`, `transport`, `_finalized`, `_manual_compress_active`, `_turn_cancel_requested`.
- Implicit representation. A session is a dict. Detached means `transport is _detached_ws_transport` and not `_finalized`. Orphaned means detached and not `running`. Cancel, compress, and running are separate flags.
- Inferred states. Idle and attached, turn running, client gone but turn still running, detached and idle, finalized, manual compress in progress.
- Inferred transitions. WebSocket drop parks the session on the drop transport. A grace reaper interrupts an orphan. Resume reattaches under `_session_resume_lock`. `prompt.submit` sets `running`. Manual compress sets `_manual_compress_active` so an interrupt must not kill the compact.
- Illegal combinations. `_finalized` with `running` true. Detached transport while the socket is still the live one. Reap of a session that reattached between the scan and the pop. `_turn_cancel_requested` during manual compress.
- Why it matters. Closing a laptop kills a turn that should keep running, or a resume gets a conflict while the old row is still the live one. `session.list` can show a ghost.
- Confidence. High.
- Origin. Upstream.

### 34. TUI client handshake

- Files and symbols. `ui-tui/src/gatewayClient.ts` (`ready`, `subscribed`, `reconnectAttempts`, `reconnectTimer`, `bufferedRequests`, `disposed`, `drainGeneration`).
- Implicit representation. Ready and subscribed are separate bools. Requests that arrive before subscribe go into `bufferedRequests`. `disposed` blocks reconnect. `drainGeneration` guards a replay after a newer attach.
- Inferred states. Cold, child starting, socket connecting, waiting for `gateway.ready`, subscribed, reconnect scheduled, disposed.
- Inferred transitions. `gateway.ready` sets `ready` and clears the attempt count. Subscribe flips `subscribed` and drains the buffer. Close schedules backoff unless `disposed`. `kill` sets `disposed`.
- Illegal combinations. `subscribed` false after a transport swap that forgot to reset it, so events emit into a dead handler. `ready` true with `disposed` true. A drain from an old `drainGeneration` replayed onto the new socket.
- Why it matters. The TUI stays on "summoning". An approval sent before the handler is registered is dropped or applied to the wrong session.
- Confidence. High.
- Origin. Upstream.

### 35. TUI composer busy latch

- Files and symbols. `ui-tui/src/app/uiStore.ts` `buildUiState` (`busy`, `busyInputMode`, `compacting`, `sid`, `status`, `streaming`).
- Implicit representation. `busy` is a bool. `status` is a free string (`summoning` at boot). `busyInputMode` is `interrupt`, `queue`, or `steer`. `compacting` is a separate bool. `streaming` here is a display preference, not the wire stream.
- Inferred states. No session, ready, busy, queued for the next turn, compacting, waiting on a status string that is not the same as busy.
- Inferred transitions. Submit sets busy before the RPC returns. Session events clear it. Compact sets `compacting`. A second submit follows `busyInputMode`.
- Illegal combinations. `busy` false while the backend is still streaming. `status` still saying queued after busy cleared. `compacting` true with `busy` false. `sid` null with `busy` true.
- Why it matters. Enter submits twice. A queued line waits until the user hits interrupt. The status line and the composer disagree.
- Confidence. Medium. The store shape was read. The submit path that sets the flags was not line-read.
- Origin. Upstream.

### 36. Session handoff between surfaces

- Files and symbols. `hermes_state_gateway.py` (`request_handoff`, `get_handoff_state`, the SQL that sets `running`, `completed`, and `failed`).
- Implicit representation. `sessions.handoff_state` is a string. Null, pending, running, completed, and failed are the words in the SQL. `handoff_platform` and `handoff_error` are companions. Moves are compare-and-swap updates, not an enum.
- Inferred states. None, pending, running, completed, failed.
- Inferred transitions. Request sets pending only from null, completed, or failed. The watcher promotes pending to running. Success sets completed. Give-up sets failed, and the SQL can restrict the source states. Shutdown can fail every running row.
- Illegal combinations. `pending` with a null platform. `running` after the process that claimed it died. `completed` with `handoff_error` still set. A second request while state is `running` returns false. The column still allows a writer to skip the CAS.
- Why it matters. Desktop, TUI, and CLI can open the same session and each believe it owns the handoff. A crashed running row blocks the next move until something fails it.
- Confidence. High.
- Origin. Upstream.

### 37. DM pairing

- Files and symbols. `gateway/pairing.py` `PairingStore` (`generate_code`, `approve_code`, `_pending_path`, `_rate_limits.json`, lockout helpers).
- Implicit representation. Pending codes live in `{platform}-pending.json` as salted hashes. Approved users live in another file. Rate limits and lockout counters live in `_rate_limits.json`. There is no pairing-state enum. Phase is "which file contains this user" plus the counters.
- Inferred states. Unknown, code issued, approved, rate-limited, platform locked out.
- Inferred transitions. An unknown DM generates a code unless the cap, the rate limit, or the lockout says no. Approve matches the hash, writes the allowlist, and clears the failure streak. Wrong codes increment the streak. TTL cleanup drops old pending rows.
- Illegal combinations. More than the pending cap. A user in both pending and approved. Lockout with a fresh code still issued. A pending file and an approved file that name different ids for one person.
- Why it matters. A stranger gets a code during lockout, or a legitimate user stays locked after a successful approve if the streak is not cleared. The code path does clear it. A second writer can put it back.
- Confidence. High.
- Origin. Upstream.

### 38. Hermes update lock

- Files and symbols. `hermes_cli/update_lock.py` (marker `.hermes-update-in-progress`, `incarnation_live`, `UPDATE_EXIT_CONCURRENT`). The checkout flock is `hermes-update.lock`.
- Implicit representation. Two artifacts. The marker is pid, start time, and creation time, plus an optional delegate line. The checkout lock is a kernel flock held by the process tree. Liveness is pid plus creation time, not file age. Finding 21 is the receipt stage list. This one is who may enter the pipeline.
- Inferred states. Idle, marker held, checkout locked, delegate child still holding the lock after the parent died, concurrent updater refused.
- Inferred transitions. Update writes the marker and takes the flock. A second update exits 2. A killed parent leaves the flock to the completion child. A v1 marker with no creation time ages out. `incarnation_live` refuses a recycled pid.
- Illegal combinations. Marker present and flock free. Flock held and marker missing. Two live owners on one marker without a delegate line. Desktop and CLI both treating a stale v1 marker as a live update.
- Why it matters. Two updates rewrite one checkout. A dead marker blocks every later update. A recycled pid is treated as the updater and the new update is refused.
- Confidence. High.
- Origin. Upstream.

### 39. Full-text index rebuild

- Files and symbols. `hermes_state_fts.py` (`FTS_STALE_KEY`, rebuild progress comments). `hermes_state_common.py` exports the stale keys.
- Implicit representation. `state_meta` keys mark the index stale and store a high-water mark and progress. Triggers attach only when the marker says the index is current. There is no rebuild-phase enum.
- Inferred states. Healthy, stale, rebuild admitted, rebuilding, deferred after a lock or corruption.
- Inferred transitions. A corruption detector sets the stale key and detaches triggers. An admission claim starts the rebuild. Progress advances the high-water row. Success clears the marker and reattaches triggers. A lock or a busy writer defers.
- Illegal combinations. Triggers attached while the stale key is set. Progress past the high-water mark. Two rebuilders, because the admission row was not claimed. A CJK index and the trigram index disagreeing about which marker is current.
- Why it matters. Search returns nothing, or a trigger writes into a stale external-content index and corrupts it. The file's own comment describes that hazard.
- Confidence. Medium. The marker and the rebuild comment were read. The full admission function was not.
- Origin. Upstream.

## Already explicit

These are single sources of truth. They are not re-listed above.

- Delegated subagent lifecycle. `agent/subagent_lifecycle.py` `SubagentState` (pending through cancelled, plus unknown). Status objects carry that enum.
- Nous sign-in. `hermes_cli/anon_sign_in.py` `SignInState` and subclasses (`Code`, `Waiting`, `Completed`, and the failure types). `run_sign_in` yields them. Renderers do not map reasons themselves.
- Connector setup targets. `tools/connectors/contract.py` `TargetState`, `Actor`, `SettleReason`. The contract module says `operation.py` enforces the moves.
- Plugin install step ids. `hermes_cli/plugin_install_phase.py` `InstallPhase`. This is a label for slow steps, not a transition table.
- Hosted-room tasks. `gateway/hosted_room_driver.py` `TaskStatus` as a `Literal`, checked against `TASK_STATUSES`.
- Skill readiness. `tools/skills_tool_setup.py` `SkillReadinessStatus`. This classifies a skill. It does not run a setup workflow.
- Desktop update stage. `apps/desktop/src/global.d.ts` `DesktopUpdateStage`. The apply store also has an `applying` bool. Writers in `apps/desktop/src/store/updates.ts` set the bool and the stage together.
- Desktop and TUI gateway socket. `apps/shared/src/json-rpc-gateway.ts` `ConnectionState` (`idle`, `connecting`, `open`, `closed`, `error`). A half-open TCP socket can still report `open` until the heartbeat fails. `apps/desktop/src/store/updates.ts` calls `reconnectGateway` after a backend restart for that case. The phase type is explicit. The transport can lie until the deadline.
- Composer send progress. `apps/desktop/src/store/send-diagnostics.ts` phase union `consent`, `uploading`, `done`, `error`.
- Pet sprite rows. `agent/pet/constants.py` `PetState`. This picks an animation row. It is not an agent lifecycle.
- Message and platform kinds. `gateway/platforms/event.py` `MessageType` and `ProcessingOutcome`. `gateway/config.py` `Platform`. These classify payloads. They are not session lifecycles.

Kanban's `DEFAULT_WORKFLOW` is only the human move matrix. Finding 1 covers the kernel, which that module refuses to model.

Cron execution status has a SQL check. Finding 5 stays in the implicit list because `handoff_pending` and the update statements are the rest of the phase.

## What was not searched or not determined

- Other git branches on the fork (`UI`, `architecture-planning`, and the rest) were not checked out. Only `main` at `d97007c` was read.
- Upstream history after that commit was not read. Current `NousResearch/hermes-agent` `main` is thousands of commits ahead of this snapshot.
- A shallow fetch of upstream could not recompute a merge base in this clone. The GitHub commit API confirms `d97007c` exists on `NousResearch/hermes-agent`, and the fork compare reports zero commits ahead.
- Platform adapters under `gateway/platforms/` and `plugins/platforms/` were not each line-read. Finding 7, finding 18, and finding 30 stop at the base adapter and the reconnect queue. Private reconnect loops in Telegram, Slack, and the other adapters may be further machines.
- `plugins/` outside the kanban and connector paths, the skill curator, and image-generation jobs were not opened far enough to number a machine. Image generation in `plugins/image_gen/` looks like a synchronous HTTP call, not a job queue.
- The website catalog UI, `evals/`, and test fixtures were skipped except where a test name pointed at a production symbol.
- Generated files, lockfiles, and vendored dependency trees were skipped.
- Streaming draft seals (`draft_stream_is_message`, interim-send metadata) are a delivery protocol described in `gateway/AGENTS.md`. The adapter implementations were not read line by line, so that protocol is not a numbered finding.
- OAuth device-code polling inside MCP, apart from `_auth_park` on `MCPServerTask`, was not mapped as its own machine. Nous sign-in is the explicit `SignInState` flow. A dashboard MCP OAuth worker (`hermes_cli/web_server_mcp.py`, `tools/mcp_dashboard_oauth.py`) uses a string `status` plus events. It was not given its own entry.
- These were opened far enough to see a lifecycle and then left unnumbered because they sit inside a finding above, or the transition list was not read through. Cron `fire_claim` and `run_claim` (`cron/jobs.py`) sit inside finding 4. The ticker heartbeat files sit beside finding 4. `gateway/turn_lease.py` and `hermes_state_compression.py` lease rows sit beside finding 8 and finding 14. `agent/interrupt_control.py` and `CompressionCommitFence` sit beside finding 13 and finding 14. `hermes_startup_watchdog.py` is a deadline plus a lease phase string. `hermes_cli/onboarding_due.py` is a cache generation. `tools/terminal_tool_lifecycle.py` and the Docker, Modal, and Vercel environment classes are per-task caches with nullable handles. `web/src/lib/pty-reconnect.ts` adds in-flight flags on top of an explicit PTY union. `apps/desktop/src/plugins/hermes-bots/relay.ts` retains sockets across a drain loop. `hermes_state_registry.py` refcounts a SessionDB generation.
