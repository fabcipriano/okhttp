# Bug #9237 — HTTP/2 readTimeout / writeTimeout Ineffective in Network Blackhole Scenario

**Issue:** https://github.com/square/okhttp/issues/9237

---

## Summary

When a network node becomes a "blackhole" (packets are silently dropped, e.g. via
`iptables -A OUTPUT -d "$CLIENT_IP" -j DROP` on the server), OkHttp's configured
`readTimeout` / `writeTimeout` fails to interrupt HTTP/2 requests within the configured
period. Instead of failing at 30 s, requests block for the full OS TCP retransmission
timeout (~15 minutes on Linux with 15 retransmissions).

---

## Root Cause Analysis

### HTTP/1.1 vs HTTP/2 timeout behaviour

| Layer | HTTP/1.1 | HTTP/2 |
|-------|----------|--------|
| Timeout unit | Per-socket (`SO_TIMEOUT` + Okio `AsyncTimeout`) | Per-stream (`StreamTimeout extends AsyncTimeout`) |
| Timeout interrupt | Default `AsyncTimeout.timedOut()` closes the socket → all kernel I/O unblocks | Custom `StreamTimeout.timedOut()` calls `closeLater()` + `sendDegradedPingLater()` — **does NOT close the socket** |
| Blocking write interrupted? | **Yes** — socket close unblocks the kernel write | **No** — `notifyAll()` cannot wake a thread blocked in the kernel |

### The two blocking scenarios in HTTP/2

```
CallServerInterceptor.intercept()
  └─ exchange.writeRequestHeaders()          ← Http2ExchangeCodec.writeRequestHeaders()
       └─ http2Connection.newStream()        ← Writes HEADERS frame — no timeout set yet!
       └─ stream.readTimeout().timeout(...)  ← Timeout configured AFTER the write
       └─ stream.writeTimeout().timeout(...) ─┘
  └─ requestBody.writeTo(bufferedBody)       ← Http2Stream.FramingSink.write()
       └─ emitFrame()
            ├─ [Scenario A] withLock { while (flow control) waitForIo() }  ← interruptible ✓
            └─ [Scenario B] connection.writeData() → writer.data()         ← kernel write ✗
  └─ exchange.finishRequest()                ← sink.closed = true
  └─ exchange.readResponseHeaders()          ← stream.takeHeaders()
       └─ [Scenario C] waitForIo()           ← interruptible ✓
```

#### Scenario A — H2 stream/connection flow-control wait (already works)

`FramingSink.emitFrame()` wraps the flow-control wait in `withLock { waitForIo() }`.
When `StreamTimeout.timedOut()` fires:
1. `closeLater(CANCEL)` sets `stream.errorCode = CANCEL` and calls `notifyAll()` on the stream lock.
2. `connection.removeStream(id)` calls `notifyAll()` on the connection lock.
3. Both `waitForIo()` calls wake up, detect `errorCode != null`, and throw `IOException`.
4. `writeTimeout.exitAndThrowIfTimedOut()` promotes this to `SocketTimeoutException("timeout")`.

**This path works before the fix.**

#### Scenario B — Actual TCP socket write (THE BUG)

`FramingSink.emitFrame()` calls `connection.writeData()` which calls `writer.data()`.
This is a **blocking kernel I/O call** — the calling thread is inside the OS kernel:

```kotlin
// emitFrame() — OUTSIDE the lock:
writeTimeout.enter()
try {
    connection.writeData(id, outFinished, sendBuffer, toWrite)  // CAN BLOCK IN KERNEL
} finally {
    writeTimeout.exitAndThrowIfTimedOut()   // NEVER REACHED if writeData() blocks
}
```

When the timeout fires:
- `timedOut()` runs on the Okio watchdog thread.
- `closeLater()` and `notifyAll()` have **no effect** on a thread blocked in a kernel syscall.
- `writeTimeout.exitAndThrowIfTimedOut()` is in the `finally` block — it is **never reached** because `writeData()` never returns.
- The write hangs for up to 15 minutes (OS TCP retransmission timeout).

#### Scenario C — Waiting for response headers (already works)

After `finishRequest()`, `sink.closed = true`, so `doReadTimeout()` returns `true`.
`takeHeaders()` uses `waitForIo()` which is properly interrupted by `notifyAll()`.

**This path also works before the fix.**

### Why HTTP/1.1 works

In HTTP/1.1, Okio wraps the raw socket with its own `AsyncTimeout`. The **default**
`AsyncTimeout.timedOut()` implementation closes the socket's source/sink, which causes
the kernel to return the blocked read or write with an error. OkHttp's HTTP/2
`StreamTimeout` **overrides** `timedOut()` with custom logic that avoids closing the socket
(to protect other multiplexed streams), but this makes it impossible to interrupt a
blocking kernel write.

---

## The Fix

**File:** `okhttp/src/commonJvmAndroid/kotlin/okhttp3/internal/http2/Http2Stream.kt`

**Change:** In `StreamTimeout.timedOut()`, add a call to `connection.socket.cancel()` after
the existing `closeLater()` and `sendDegradedPingLater()` calls.

```kotlin
// BEFORE (broken for TCP-level blocking writes):
internal inner class StreamTimeout : AsyncTimeout() {
    override fun timedOut() {
        closeLater(ErrorCode.CANCEL)
        connection.sendDegradedPingLater()
        // ← nothing here to interrupt a blocking kernel write!
    }
}

// AFTER (fix):
internal inner class StreamTimeout : AsyncTimeout() {
    override fun timedOut() {
        closeLater(ErrorCode.CANCEL)
        connection.sendDegradedPingLater()
        // Cancel the underlying socket to interrupt any blocking kernel I/O operations.
        // Unlike HTTP/1.1 where Okio's default timedOut() closes the socket, HTTP/2 uses
        // per-stream timeouts that cannot interrupt a TCP-level write via notifyAll() alone.
        // Closing the socket here matches HTTP/1.1 behavior and is required to unblock a
        // write that is stuck in the kernel (e.g. due to a network blackhole dropping ACKs).
        try {
            connection.socket.cancel()
        } catch (_: Exception) {
            // Ignore: socket may already be closed or cancelled.
        }
    }
}
```

### Why cancelling the socket is correct

1. **Matches HTTP/1.1 semantics.** When an HTTP/1.1 timeout fires, the entire connection
   is torn down. HTTP/2 multiplexes streams on one connection, but when a stream's write
   times out the connection is broken anyway — other streams would also be stuck waiting
   for TCP ACKs.

2. **Idempotent.** Calling `cancel()` on an already-cancelled `BufferedSocket` is safe.

3. **Triggers `SocketTimeoutException`.** After `socket.cancel()` interrupts the blocking
   `writer.data()` call with a `SocketException("Socket closed")`, the `finally` block in
   `emitFrame()` runs `writeTimeout.exitAndThrowIfTimedOut()`. Because the timeout did
   fire (`exit()` returns `true`), this promotes the exception to
   `SocketTimeoutException("timeout")`, which is the expected public API contract.

4. **Affects all streams on the connection.** This is intentional — if one stream cannot
   write because TCP is blocked, no other stream can write either (they share the same
   socket). Tearing down the connection and retrying on a new one is the correct response.

---

## Fix: Sequence Diagram (after the fix)

```
Writer thread                    Okio Watchdog thread
─────────────                    ────────────────────
emitFrame()
  writeTimeout.enter()  ──────→  (timer started: 500 ms)
  connection.writeData()
    writer.data()
      [KERNEL WRITE — BLOCKED]
                                 [500 ms elapsed]
                                 StreamTimeout.timedOut()
                                   closeLater(CANCEL)
                                     stream.errorCode = CANCEL
                                     notifyAll() [stream lock]
                                     connection.removeStream(id)
                                       notifyAll() [conn lock]
                                   sendDegradedPingLater()
                                   connection.socket.cancel()   ← NEW
                                     socket closed
      [KERNEL WRITE — RETURNS WITH SocketException]
    SocketException propagates
  finally:
    writeTimeout.exitAndThrowIfTimedOut()
      exit() → true  (timeout fired)
      throw SocketTimeoutException("timeout")   ← caller sees this
```

---

## Tests Added

### Unit test — `Http2ConnectionTest.kt`

#### `writeTimeoutIsEffectiveWhenPeerStopsReading`

Directly tests the TCP-level blocking bug using `MockHttp2Peer`.

**Setup:**
- Peer sends `SETTINGS { INITIAL_WINDOW_SIZE = 16 MB }` — large stream flow control window.
- Peer sends `WINDOW_UPDATE(0, 16 MB)` — large connection flow control window.
- After a PING/PONG sync (to ensure window updates are applied), peer accepts `HEADERS` then **stops reading** — its socket buffer will fill up.

**Why this demonstrates the bug:**
With 16 MB H2 flow control windows, the H2 flow-control `waitForIo()` path (which was
already working) is NOT the bottleneck. Instead, `writer.data()` fills the TCP receive
buffer and blocks in the kernel — exactly the scenario that was unfixable before.

**Before fix:** Test hangs until JUnit `@Timeout(5s)` kills it.
**After fix:** `SocketTimeoutException("timeout")` thrown within ~500 ms.

#### `readTimeoutIsEffectiveWhenPeerNeverSendsResponse`

Tests that `readTimeout` fires correctly when no response arrives (the simpler read-side
case, which already worked but is documented for completeness).

**Setup:** Peer accepts `HEADERS` with `END_STREAM` but never sends response headers.
`takeHeaders()` blocks; `readTimeout` must fire.

---

### Integration test — `HttpOverHttp2Test.kt`

#### `writeTimeoutIsEffectiveWhenServerStalls`

End-to-end test with `MockWebServer` using `SocketEffect.Stall`.

**Setup:**
- Server enqueues a response with `onRequestStart(Stall)` — freezes after receiving
  request headers, never sending H2 `WINDOW_UPDATE`.
- Client is configured with `writeTimeout = 500 ms`.
- Client sends a `128 KB` POST body (larger than the default 65535-byte H2 flow control
  window).

**Behaviour:** The server's stall prevents any `WINDOW_UPDATE`, so after writing ~65 KB
the H2 stream flow control window is exhausted. The write blocks in `waitForIo()`
(Scenario A). The `writeTimeout` fires, `closeLater()` + `notifyAll()` wakes the thread,
and `SocketTimeoutException("timeout")` is thrown.

This test also exercises the code path that includes the `socket.cancel()` fix in
`timedOut()`, ensuring the fix does not break the already-working H2 flow-control case.

---

## Files Changed

| File | Change |
|------|--------|
| `okhttp/src/commonJvmAndroid/kotlin/okhttp3/internal/http2/Http2Stream.kt` | Add `connection.socket.cancel()` in `StreamTimeout.timedOut()` |
| `okhttp/src/jvmTest/kotlin/okhttp3/internal/http2/Http2ConnectionTest.kt` | Add `writeTimeoutIsEffectiveWhenPeerStopsReading` and `readTimeoutIsEffectiveWhenPeerNeverSendsResponse` |
| `okhttp/src/jvmTest/kotlin/okhttp3/internal/http2/HttpOverHttp2Test.kt` | Add `writeTimeoutIsEffectiveWhenServerStalls` |

---

## Options Considered

### Option A — Socket cancel in `timedOut()` ✅ (chosen)

Add `connection.socket.cancel()` to `StreamTimeout.timedOut()`.

**Pros:** Minimal change; matches HTTP/1.1 semantics; correctly interrupts any blocking I/O.
**Cons:** Cancels the socket for ALL streams on the connection, not just the timed-out stream.
**Verdict:** Acceptable — if one stream is stuck in a kernel write, the connection is
effectively dead for all streams. Closing it is the correct response.

### Option B — Fix `writeRequestHeaders()` missing timeout

In `Http2ExchangeCodec.writeRequestHeaders()`, the `http2Connection.newStream()` call
writes the HTTP/2 `HEADERS` frame **before** stream timeouts are configured:

```kotlin
stream = http2Connection.newStream(requestHeaders, hasRequestBody)  // writes HEADERS — no timeout!
stream!!.readTimeout().timeout(chain.readTimeoutMillis.toLong(), TimeUnit.MILLISECONDS)
stream!!.writeTimeout().timeout(chain.writeTimeoutMillis.toLong(), TimeUnit.MILLISECONDS)
```

For a fresh connection with an empty TCP buffer, this is rarely a problem (the `HEADERS`
frame is small and writes instantly). Option A already fixes the more impactful DATA-frame
blocking case. This secondary issue could be addressed as a follow-up.

### Option C — Use `callTimeout` as a workaround

Users can configure `OkHttpClient.callTimeout()` to bound the entire request duration.
This is a valid user-side workaround but does not fix the underlying OkHttp behaviour.

### Option D — Non-blocking I/O

Rewriting the HTTP/2 writer to use NIO `Selector`-based non-blocking I/O would allow
per-stream timeout without closing the socket. This is the architecturally cleanest
solution but requires a major refactor of the entire HTTP/2 write path.

---

## Reproduction

```bash
# On the server, after a connection is established, drop outgoing packets to the client:
sudo iptables -A OUTPUT -d "$CLIENT_IP" -j DROP

# The client's TCP stack retransmits up to 15 times (~15 minutes on Linux).
# With the fix, OkHttp's writeTimeout terminates the request within the configured period.
```

Without the fix, `readTimeout = 30 s` has no effect during the write phase because the
timeout timer only starts (via `takeHeaders()`) after the write completes — which never
happens in the blackhole scenario.
