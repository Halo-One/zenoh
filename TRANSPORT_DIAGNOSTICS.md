# Transport diagnostics and socket linger

Lifecycle logging observes transport behavior. The socket linger override changes
TCP socket behavior and works independently of logging.

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

Diagnostics default to upstream TCP linger behavior. Logging queued teardown alone
does not prove an outage; correlate close duration with independent healthy-path
progress. These events do not log individual messages or alter runtime worker
counts or scheduling policy.

## Runtime TCP/TLS linger override

Set `ZENOH_SOCKET_LINGER=off` to disable positive TCP linger on newly created
accepted and outgoing TCP and TLS links (on the underlying TCP socket):

```ini
Environment="RUST_LOG=info,zenoh::transport::lifecycle=debug"
Environment="ZENOH_SOCKET_LINGER=off"
```

Unset, `stock`, and all values other than exactly `off` retain upstream behavior
(normally ten-second positive linger). The override operates independently of
the tracing filter. The environment is read once at the first TCP or TLS link
construction, and the result is shared by both transports. Restart the application
after changing its environment; existing sockets are unaffected.

The default timeout comes from `TCP_LINGER_TIMEOUT` or `TLS_LINGER_TIMEOUT`
(normally ten seconds). Those existing settings are compile-time environment
configuration; `ZENOH_SOCKET_LINGER` is a runtime override.

`off` uses `set_linger(None)`, not `Some(Duration::ZERO)`: it requests ordinary
close without the synchronous bounded wait for queued output, rather than an
abortive close. The kernel can continue processing queued TCP output afterward;
this does not guarantee delivery or peer application processing. For TCP links, check
`tcp_link_created.effective_linger` and `tcp_drop_start.linger` to confirm the
actual socket setting, including whether setting the option succeeded.

This override affects TCP and TLS links, not other transports. TLS retains its
existing asynchronous flush and TCP shutdown steps; disabling linger does not
bound their duration. The TCP lifecycle diagnostic events above do not cover TLS.

## Delivery and teardown tradeoffs

Disabling linger changes final socket close behavior, not delivery on an active
link: TCP still acknowledges and retransmits data, and TLS still encrypts it.
After an ordinary close, the kernel can continue sending queued data and the FIN
while the host and network stack remain running. Close returning does not mean
that queued data has reached the peer, so application teardown must not use it
as a delivery barrier. Positive linger does not prove peer application processing
either; use an application-level acknowledgment when that matters.

On lossy or intermittent radio paths, queued output can outlive the link object
and consume socket buffers and radio airtime during retransmission. Reconnecting
can overlap this old traffic with a new connection; stale data may still arrive
on the old connection if it recovers. This can also happen after positive linger
times out, so a ten-second linger is not a delivery deadline or cancellation
mechanism. Delivery can still fail if an outage exceeds TCP retry limits or the
host loses power or shuts down before the kernel drains its queues.

The benefit is that final close no longer synchronously waits for queued output,
which can help keep teardown from delaying unrelated work during radio outages.
The tradeoff is losing that bounded opportunity to wait before the application
continues, for example before powering down a radio or the host. For critical
final messages, wait for an application acknowledgment before teardown; for
time-sensitive telemetry, use expiry/sequence checks to reject stale data after
reconnection. TLS flush alone is not a peer delivery acknowledgment.

## Performance impact

With the lifecycle target disabled:

- Diagnostic link formatting, allocations, timestamps, and thread lookups are
  skipped during incoming setup and link teardown.
- TCP destruction skips queue and linger queries, thread lookups, and timing.
- Each TCP link still receives an ID using a relaxed atomic increment. There are
  also tracing filter checks and per-link diagnostic state.
- The linger override uses a shared cached value after its first environment
  lookup; it does not read the environment for every connection.
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
overhead bound. Disabling positive linger can reduce final-close stalls caused by
queued output; that behavioral effect is separate from the cost of logging and
does not bound TLS flush/shutdown duration.
