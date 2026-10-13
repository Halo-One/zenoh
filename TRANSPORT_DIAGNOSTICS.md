# Transport lifecycle diagnostics

## Lifecycle diagnostics

The TCP link and unicast transport emit lifecycle-only debug events on the
`zenoh::transport::lifecycle` tracing target. The embedding application must initialize a
tracing subscriber with environment filtering, for example:

```sh
RUST_LOG=info,zenoh::transport::lifecycle=debug
```

Global `info` does not enable these events. Global `debug` does; use
`debug,zenoh::transport::lifecycle=off` to suppress them while retaining ordinary
debug logs.

Events cover TCP creation and shutdown, incoming setup and timeout, receive
lease expiry and errors, teardown callbacks, and actual final socket destruction.
`tcp_link_created` reports requested and effective linger. Pair `tcp_drop_start`
and `tcp_drop_end` using the process and `diag_id`, plus endpoints; file
descriptors can be reused. IDs are assigned even while diagnostics are disabled
so that enabling the filter later still yields unique IDs for existing links.
`close_us` measures explicit destruction of the Tokio stream, including associated
cleanup, rather than an isolated kernel `close()` call or the duration of the
surrounding transport teardown or async shutdown. Start-event emission is outside
that measured interval, but still contributes to overall teardown latency.

On Linux, `outq_bytes` includes unsent and unacknowledged TCP bytes and
`notsent_bytes` includes unsent bytes only. A failed query is `None`, not zero.
An outgoing queue can include a FIN, so a one-byte queue alone does not prove
pending payload. OS thread IDs and descriptors are unavailable on other platforms.

Diagnostics preserve upstream TCP linger behavior. Logging queued teardown alone
does not prove an outage; correlate close duration with independent healthy-path
progress. These events do not log individual messages or alter runtime worker
counts or scheduling policy. The TCP-specific lifecycle events do not cover TLS.

## Performance impact

With the lifecycle target disabled:

- Diagnostic link formatting, allocations, timestamps, and thread lookups are
  skipped during incoming setup and link teardown.
- TCP destruction skips queue and linger queries, thread lookups, and timing.
- Each TCP link still receives an ID using a relaxed atomic increment. There are
  also tracing filter checks and per-link diagnostic state.
- Explicit TCP destruction uses an optional socket, adding a potential
  discriminant check on read/write access as well as socket extraction on drop.
  There are no new per-message log events or diagnostic syscalls on successful I/O.

With the lifecycle target enabled, TCP creation queries effective linger. On
Linux, TCP destruction additionally performs two queue-query ioctls, a thread-ID
syscall, a linger query, clock reads, and start/end log events. Other lifecycle
events incur formatting and subscriber/output costs. A synchronous or congested
logging subscriber can delay runtime workers and teardown, including events
emitted while transport locks are held. Connection churn can amplify these costs
and diagnostics can perturb the behavior being measured.

For long-lived connections the additional lifecycle work is expected to be small,
but no throughput or latency overhead has been quantified. Benchmark steady-state
I/O and connection churn with the intended subscriber before relying on a specific
overhead bound.
