# From-Scratch Systems Ladder
### HTTP Server → WebSockets → Reverse Proxy → Load Balancer
**Stack:** TypeScript (raw `net`/`tls` modules — not `http`), Docker/Docker Compose
**Rule:** No LLM-generated code. Docs, RFCs, and reading real source (for comparison, not copying) are fair game.

---

## Phase 0 — Foundations (before you write a line of code)

**Concepts to nail down first:**
- TCP fundamentals: three-way handshake, what a socket actually is, ports vs addresses
- HTTP/1.1 message format: request line, headers, status line, `Content-Length`, persistent connections (keep-alive), chunked transfer encoding
- Node core APIs you'll actually use: `net` (raw TCP), `Buffer`, `Stream`, `EventEmitter` — *not* `http`
- Docker basics: image vs container, `Dockerfile`, `docker-compose`, container networking (bridge network + DNS-by-service-name)

**Where to read:**
- RFC 9112 (HTTP/1.1 message syntax) — skim, don't study cover to cover
- Node.js docs for `net`, `Buffer`, `stream`
- MDN's HTTP overview pages for conceptual grounding

**✅ Checkpoint 0:** You can explain, without notes, everything that happens between typing a URL and pixels on screen — TCP handshake, request framing, response, connection close/keep-alive.

---

## Phase 1 — Raw TCP echo server

**Build:** A TCP server using `net.createServer` that echoes back whatever bytes a client sends.

**Learn:** socket lifecycle (`data`, `end`, `close`, `error` events), backpressure, handling multiple concurrent connections.

**Dockerize:** one container, exposed via `docker run -p`.

**✅ Checkpoint 1:** Server runs in a container; you can connect with `nc` or `telnet` from your host and see bytes echoed; two simultaneous clients both work.

---

## Phase 2 — HTTP/1.1 server, parsed by hand

**Build:** On top of Phase 1, parse the raw byte stream into a request line + headers + body yourself. No `http` module. Support GET/POST, `Content-Length` bodies, and basic routing (method + path).

**Learn:** message framing, malformed-request handling, why `Content-Length` matters, correct status line/header formatting.

**Stretch (do this before moving on if you can):** keep-alive (persistent connections) and chunked transfer-encoding on responses — both matter a lot once you build the proxy.

**Where to get stuck productively:** RFC 9112 §2–3 for message syntax; read (don't copy) how a minimal parser like `node:http`'s internals or a small existing parser handles edge cases once yours breaks on something unexpected.

**✅ Checkpoint 2:** `curl` against your server returns correct responses for GET and POST-with-body; a basic load test (`ab` or `wrk`) doesn't crash it or hang connections.

---

## Phase 3 — WebSockets (the Upgrade mechanism)

**Build:** Handle an HTTP `Upgrade` request by hand — compute `Sec-WebSocket-Accept` from the client's key, then implement basic WebSocket frame parsing (opcode, masking, payload length variants) for a simple echo app.

**Learn:** RFC 6455 — specifically the handshake and the framing format. This is a natural extension of Phase 2 since the handshake *is* an HTTP request.

**✅ Checkpoint 3:** A real browser WebSocket client connects to your server and exchanges messages — verified in devtools' Network tab, not just a test script.

---

## Phase 4 — Reverse proxy (single backend)

**Build:** A proxy that accepts client connections and forwards them to one backend (can be your Phase 2 server). Stream request and response through without buffering the whole body. Rewrite/add headers (`X-Forwarded-For`, `Host`).

**Docker:** two containers on the same Compose network — `proxy` and `backend` — proxy is the only one exposed to the host; it reaches the backend by service name.

**✅ Checkpoint 4:** Client only ever talks to the proxy's port; requests are transparently forwarded to the backend container; confirm via `curl` + backend logs showing the forwarded request.

---

## Phase 5 — Load balancer (multiple backends)

**Build:** Extend the proxy to hold a pool of backend addresses. Implement round-robin, then least-connections. Add periodic health checks that mark a backend down/up.

**Docker Compose:** 3+ backend replicas + 1 load balancer service.

**Learn:** connection pooling, timeouts, basic retry/circuit-breaking logic — this is where "senior" instincts (what happens when a dependency is slow vs. down) start forming.

**✅ Checkpoint 5:** Kill one backend container mid-traffic. The load balancer detects it and routes around it — no client-visible errors beyond whatever was in-flight at that instant.

---

## Phase 6 — Optional stretch (only if 0–5 felt solid)
- TLS termination at the proxy
- Graceful shutdown (drain connections before exit)
- Basic logging/metrics (request counts, latency per backend)

This is genuinely optional — phases 1–5 already cover the core "senior-level" ground (protocol internals, proxying, load balancing under failure).

---

## Pacing notes (given a full-time job + a track record of stalling on hard self-directed work)
- Treat each phase as **1–3 weeks**, not a weekend. There is no prize for speed here.
- The checkpoints above are deliberately narrow and binary (it works or it doesn't) — use them as your only definition of "done." Resist the urge to gold-plate a phase before moving on.
- If a phase gets boring rather than hard, that's a signal to move to the next one, not to abandon the whole ladder — boredom usually means the checkpoint is basically met.
- "No LLM" applies to *writing your implementation*. Reading RFCs, Node docs, or asking someone (including me) to explain a concept or help you understand an error message is not cheating — it's how you'd work as a senior engineer too.
