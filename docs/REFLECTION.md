# Engineering Reflection: Transport-Layer Protocol Visualizer
**Course:** Computer Networks – Transport Layer  
**Assignment 2:** Dual-Panel Activity & Transport-Layer Protocol Visualizer  
**Author:** Protocol Lab Team  

---

## 1. Choice of AI Platform and Model

For this assignment, **Google Antigravity** powered by **Gemini 3.8 Flash (High)** was chosen as the primary agentic development environment. 

Several technical factors motivated this choice:
1. **End-to-End File Manipulation & Tool Integration:** Google Antigravity operates natively with direct filesystem read/write access and shell execution capabilities, enabling seamless multi-file codebase analysis and iterative refactoring without cumbersome copy-pasting.
2. **High Context Window & Fast Reasoning:** Extending the Assignment 1 dashboard required ingesting the full original codebase, understanding the CSS layout architecture, and generating an extended, mathematically rigorous simulation engine spanning hundreds of lines of code. Gemini 3.8 Flash provided the speed and reasoning density necessary to produce complete, non-truncated source code while retaining the sharp visual design system (`border-radius: 0 !important`).
3. **Reproducibility & Verification:** The agentic environment allowed real-time verification of DOM selectors and JavaScript syntax through automated scratch verification scripts before delivering final artifacts.

---

## 2. Synchronization Architecture Between Panels

A core requirement of Assignment 2 is ensuring that the **Left Panel (Activity)**, **Right Panel (Application Layer)**, and **Right Panel (Transport Layer)** stay in absolute synchrony during playback, single-stepping, and timeline scrubbing.

### The Unified Step Model
Rather than running two independent timers or asynchronous loops that could drift over time, the visualizer uses a **single deterministic event timeline**. Every network activity (`buildBrowse`, `buildMail`, `buildStream`) compiles into an ordered array of discrete wire steps (`steps[]`):

```javascript
createStep({
  t: 28,                          // Cumulative timestamp in ms
  proto: 'TCP',                   // Transport protocol ('TCP' | 'UDP' | 'TLS')
  d: 'c',                         // Direction: 'c' (client -> server) or 's' (server -> client)
  src: '192.168.1.105:49152',
  dst: '93.184.216.34:80',
  flags: ['PSH', 'ACK'],          // TCP Control Flags
  seq: 1, rawSeq: 28419201,       // Relative and 32-bit Raw Sequence Numbers
  ack: 1, rawAck: 71029410,       // Relative and 32-bit Raw Acknowledgement Numbers
  win: 64240, len: 145,           // Window size & payload byte length
  cState: 'ESTABLISHED',          // Client TCP State Machine
  sState: 'ESTABLISHED',          // Server TCP State Machine
  cwnd: 4,                        // Congestion Window (in MSS)
  app: {                          // Associated Application-Layer event (if triggered)
    p: 'HTTP', title: 'HTTP request: GET /', ...
  }
});
```

### Tri-Panel State Derivation
All three UI panels are pure functional projections of the single timeline cursor `idx`:
1. **Activity Panel (Left):** The status message (`#stat`), log entries (`#log`), rendered browser viewport (`#pc`), mail queue confirmation (`#rcpt`), and streaming video buffer/scrubber (`#buf`, `#pos`) are computed directly from `steps.slice(0, idx + 1)`. Stepping forward advances the activity; stepping backward immediately restores previous UI states without side-effects.
2. **Application-Layer Column (Right):** Filters and renders all application-level events (`appMessage`) surfaced by wire steps up to index `idx`. If a transport step represents a pure control packet (such as a TCP `SYN` or `ACK`), the App view displays the establishing state while highlighting the current activity.
3. **Transport-Layer Column (Right):** Renders every packet up to index `idx` as a dissected packet card. The active packet card (`idx`) is visually highlighted with a glowing border and auto-scrolled into view in lockstep with the corresponding application card.
4. **State Machine & Metrics Strip:** The TCP state machine indicators (`#c-state`, `#s-state`), congestion window (`#m-cwnd`), and bytes in flight (`#m-flight`) are dynamically updated on every tick, giving students an instantaneous snapshot of transport-layer internals.

---

## 3. What the AI Got Wrong and How It Was Corrected

While modern agentic models excel at boilerplate generation and visual design, deep protocol state machines and byte-stream mathematics require careful scrutiny. During the development process, the AI initially introduced several subtle protocol errors that were identified and corrected:

### A. Sequence Number Consumption by Pure ACKs
- **The Error:** In early drafts, the AI treated pure `ACK` segments (e.g., the final segment of the 3-way handshake or intermediate transport acknowledgments with `Len: 0`) as incrementing the sender's sequence number by 1 (advancing `cSeq` from 1 to 2).
- **The Protocol Rule (RFC 793):** Only segments carrying payload data (`Len > 0`) or control flags that occupy sequence space (`SYN` and `FIN`) consume sequence numbers. A pure `ACK` carries zero payload and consumes zero sequence numbers.
- **The Correction:** The packet generator was refactored so that `cSeq` and `sSeq` increment strictly when `len > 0` or when `flags.includes('SYN')` or `flags.includes('FIN')`. Pure `ACK` packets maintain the exact sequence number of the preceding transmitted byte.

### B. Sequence Number Tracking Across Conversational SMTP Streams
- **The Error:** In the Mail activity, the AI initially hardcoded static relative sequence numbers (`Seq: 1`, `Ack: 1`) for all SMTP dialogue steps, treating each command as an isolated exchange rather than an incremental conversational stream.
- **The Protocol Rule:** SMTP runs over a **single persistent TCP connection**. As the server greets (`220`), the client introduces itself (`EHLO`), and envelope commands (`MAIL FROM`, `RCPT TO`, `DATA`) are exchanged, the sequence numbers must increment monotonically by the exact byte length (including `\r\n`) of each command and response string.
- **The Correction:** An accumulator loop was built for `buildMail` that tracks both client and server sequence offsets (`cSeq`, `sSeq`). For each transmitted SMTP message, `Seq` is assigned the current offset, and `offset += payload.length`. Acknowledgments (`Ack`) strictly track the opposite peer's sequence number plus length.

### C. Teardown Symmetry and FIN Sequence Space
- **The Error:** The AI omitted the fact that a `FIN` control segment consumes 1 sequence number, causing the final client `ACK` to acknowledge the wrong sequence number. Furthermore, it modeled teardown as a simultaneous disconnect rather than the standard 4-way handshake.
- **The Correction:** Structured the teardown into explicit RFC 793 half-close phases:
  1. Client sends `[FIN, ACK]` (`cSeq`, state `FIN_WAIT_1`). Consumes 1 sequence number (`cSeq + 1`).
  2. Server responds `[ACK]` (`Ack: cSeq + 1`, states: `CLOSE_WAIT` on server, `FIN_WAIT_2` on client).
  3. Server sends `[FIN, ACK]` (`sSeq`, state `LAST_ACK`). Consumes 1 sequence number (`sSeq + 1`).
  4. Client responds `[ACK]` (`Ack: sSeq + 1`, states: `CLOSED` on server, `TIME_WAIT` on client).

---

## 4. Comparative Analysis of Transport-Layer Flows

Observing the three activities in side-by-side Split View reveals stark architectural differences in how Transport-layer protocols support diverse application requirements:

| Dimension | Browsing (HTTP / HTTPS) | Mail (SMTP) | Streaming (TCP vs. UDP) |
|---|---|---|---|
| **Underlying Protocol** | TCP (over Port 80 or 443) | TCP (over Port 25) | TCP (HLS/443) vs. UDP (RTP/5004) |
| **Connection Pattern** | Short-lived, transactional request-response exchange. | Long-lived, interactive conversational dialogue. | Continuous bulk media chunk transfer (TCP) or datagram stream (UDP). |
| **Handshake Overhead** | 3-way handshake + TLS 1.3 key exchange before first HTTP byte. | Single 3-way handshake amortized across the entire email transaction. | TCP has handshake penalty per chunk connection; UDP has **zero setup delay**. |
| **Sequence Number Growth** | Step-wise: small client request (~150 B) followed by larger server payload (~480 B). | Incremental ping-pong: small command packets (~15–35 B) alternating with server status codes. | Monotonic high-volume increase (hundreds of KB per video chunk). |
| **Flow & Congestion Control** | Reaches modest congestion window (`cwnd` ~4–6 MSS) before closing. | Stays in low-bandwidth conversational state (`cwnd` ~2–4 MSS). | **TCP:** Aggressive Slow Start (`cwnd` doubles: 2 → 4 → 8 MSS).<br>**UDP:** Stateless; no congestion control or windowing. |
| **Reliability & Loss Handling** | Guaranteed byte-stream delivery via retransmissions and strict ACKs. | Guaranteed lossless delivery; corruption/loss causes retransmission. | **TCP:** Head-of-line blocking stalls playback if packet drops.<br>**UDP:** Loss-tolerant; dropped frames do not stall video playback. |

### Educational Takeaways
1. **Browsing:** Highlights the latency cost of connection establishment. Before a browser receives the first byte of HTML, two complete round-trips (DNS UDP query + TCP 3-way handshake, plus TLS) must occur.
2. **Mail:** Contrasts transaction-based HTTP with command-driven SMTP. The student visualizes how a stateful application protocol relies on TCP's reliable ordering to ensure commands arrive in strict sequential order.
3. **Streaming:** Provides the clearest contrast between TCP and UDP. Over TCP, video segments benefit from automatic flow control and window scaling, but suffer from connection setup overhead and head-of-line sensitivity. Over UDP, media packets flow instantaneously with minimal 8-byte headers, demonstrating why live voice, gaming, and real-time broadcast systems favor datagram delivery.
