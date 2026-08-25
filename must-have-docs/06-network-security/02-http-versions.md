# HTTP/1.1 vs HTTP/2 vs HTTP/3 — Head-of-Line Blocking, Multiplex/HPACK và QUIC/UDP 0-RTT, WS vs SSE

> Tags: #http/1.1 #http/2 #http/3 #QUIC #multiplex #HPACK #QPACK #head-of-line-blocking #WebSocket #SSE #EventSource | Nguồn: `docs/06-browser-web-platform.md` câu 114-117 | Mức: P0

## 1. Định nghĩa chính xác

**HTTP/1.1** là giao thức **text**, **1 request/response per TCP connection** (hoặc 6 connections parallel với `keep-alive`), header không nén, chịu **Head-of-Line (HOL) blocking** ở HTTP và TCP.

**HTTP/2 (RFC 7540)** là **binary framing** trên **1 TCP connection** với **multiplexing** (nhiều stream interleave), **header compression HPACK** (static/dynamic table + Huffman, giảm 50-80%), **stream priority** (`Priority: u=0,i`). Giải HOL ở HTTP nhưng vẫn HOL ở TCP (1 packet loss block mọi streams do TCP retransmit).

**HTTP/3 (RFC 9114)** chạy trên **QUIC (UDP)**, mỗi stream là **QUIC stream độc lập**, không HOL blocking ở transport, handshake **0-RTT** (gửi data ngay lần đầu sau khi đã kết nối lần trước), **connection migration** (đổi WiFi→4G không đứt), header nén **QPACK** (tương tự HPACK nhưng thiết kế cho QUIC, khắc phục HOL do HPACK dynamic table).

**WebSocket** là **bidirectional full-duplex** persistent sau `101 Switching Protocols` upgrade, frame 2-14 byte, client↔server đều push. **SSE (Server-Sent Events)** là **unidirectional** server→client trên **HTTP** thuần, `Content-Type: text/event-stream`, `EventSource` tự reconnect + `Last-Event-ID`.

## 2. Cơ chế hoạt động

### 2.1 HTTP/1.1

```
GET /a.js HTTP/1.1
Host: example.com
GET /b.js HTTP/1.1  # phải đợi a.js xong hoặc mở connection mới (max 6/host)
```
- Text, mỗi connection queue FIFO. Để tăng concurrency: **domain sharding** (`a1.cdn.com`, `a2.cdn.com`), **concat** (bundle), **sprite** — đều là workaround cho HOL.
- Header lặp (`Cookie`, `User-Agent`) gửi full mỗi request, không nén.

### 2.2 HTTP/2

- **Binary framing**: 1 TCP connection chia thành **stream** (mỗi request = 1 stream), mỗi stream chia **frame** (`HEADERS`, `DATA`, `PRIORITY`). Frame của nhiều stream **interleave** — không đợi.
  ```
  Connection 1: [Stream1 HEADERS] [Stream3 HEADERS] [Stream1 DATA] [Stream5 DATA] ...
  ```
- **HPACK**: request 1 gửi `user-agent: xxx` full, request 2 chỉ gửi index tham chiếu dynamic table → giảm size.
- **Priority**: client gửi `Priority: u=0, i` (urgency 0 cao nhất) hoặc `fetch(url, { priority: 'high' })`, server ưu tiên CSS/hero trước. Lịch sử dependency tree H2 cũ phức tạp, nay dùng `Priority` header đơn giản.
- **Server Push** (đã deprecated): server đẩy `app.css` khi client request `/`. Bỏ vì cache không hiệu quả, thay bằng `103 Early Hints` + `preload`.

HOL còn lại: HTTP/2 multiplex giải HOL HTTP, nhưng **TCP HOL** vẫn tồn tại — 1 packet loss → TCP retransmit → **mọi streams stall** dù chỉ 1 stream mất packet.

### 2.3 HTTP/3 / QUIC

- QUIC trên **UDP**, implement reliability + congestion control ở user-space. Mỗi HTTP/3 stream = QUIC stream độc lập → **packet loss stream 1 không ảnh hưởng stream 3**.
- **0-RTT**: lần đầu 1-RTT handshake (TLS 1.3 tích hợp), lần sau client gửi **0-RTT** kèm `early data` ngay, giảm 100-300ms cho repeat visit.
- **Migration**: connection ID không gắn IP, đổi mạng (WiFi→4G) vẫn giữ connection.
- **QPACK**: tương tự HPACK nhưng **không block** khi dynamic table chưa sync (HPACK head-of-line do table update phải theo thứ tự).

```
# H2: loss 1 packet → tất cả streams block
TCP Seq: ... | loss | retransmit | all streams wait

# H3/QUIC: loss stream1 → chỉ stream1 retry, stream3 vẫn deliver
QUIC Stream1: loss → retry
QUIC Stream3: deliver immediately
```

### 2.4 WS vs SSE

|  | SSE | WebSocket |
|--|-----|-----------|
| Hướng | **Unidirectional** server→client | **Bidirectional** full-duplex |
| Protocol | **HTTP** `text/event-stream`, qua HTTP/2 multiplex | **WS** upgrade `ws://`/`wss://` → frame riêng, không qua HTTP/2 |
| Reconnect | **Tự động** (EventSource retry 3s + `Last-Event-ID`) | **Tự code** |
| Format | Text `data: ...\n\n`, `event: custom`, `id: 123` | Binary + text, opcode |
| Firewall | Thân thiện (HTTP) | Có thể bị proxy chặn |
| HTTP/2 | Có lợi (multiplex) | Không (1 TCP riêng) |
| Dùng khi | Notification, feed, ticker, log, progress (1 chiều, cần reconnect) | Chat, collaborative cursor, game, trading (2 chiều, low latency, binary) |

## 3. Ví dụ tối thiểu

```http
# 3.1 HTTP/1.1 — queue, 6 connections
GET /a.js HTTP/1.1
Host: example.com
# phải đợi hoặc mở connection 2..6

# 3.2 HTTP/2 — binary, multiplex, HPACK, priority
:method: GET
:path: /a.js
:authority: example.com
priority: u=0, i
# stream 1, 3, 5 interleave

# 3.3 HTTP/3 — QUIC 0-RTT
# 1st: ClientHello + TLS1.3 → 1-RTT
# 2nd: 0-RTT early data: GET /  (gửi ngay)

# 3.4 WS handshake
GET /chat HTTP/1.1
Host: example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13

HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
# Sau đó frame: opcode + payload (2-14 byte overhead)

# 3.5 SSE
GET /events HTTP/1.1
Accept: text/event-stream

HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive

id: 1
data: {"time": 1714000000}

id: 2
event: custom
data: hello

# 3.6 Early Hints thay Server Push
HTTP/1.1 103 Early Hints
Link: </style.css>; rel=preload; as=style
Link: </hero.jpg>; rel=preload; as=image
```

```js
// 3.7 Check protocol
performance.getEntriesByType('navigation')[0].nextHopProtocol // "h2" | "h3" | "http/1.1"
// DevTools Network → cột Protocol (enable trong header)

// 3.8 Tối ưu theo version
// H1.1: concat, sprite, domain sharding, inline
// H2/H3: không concat, split nhỏ, nhiều file nhỏ ok (multiplex)

// 3.9 SSE Server (Express)
app.get('/events', (req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', Connection: 'keep-alive' });
  const iv = setInterval(() => {
    res.write(`id: ${Date.now()}\ndata: ${JSON.stringify({ time: Date.now() })}\n\n`);
  }, 1000);
  req.on('close', () => clearInterval(iv));
});

// 3.10 SSE Client — tự reconnect + Last-Event-ID
const es = new EventSource('/events');
es.onmessage = e => console.log(JSON.parse(e.data));
es.addEventListener('custom', e => console.log(e.data));
es.onerror = () => console.log('reconnect tự động');

// 3.11 WebSocket — phải tự heartbeat + reconnect
const ws = new WebSocket('wss://example.com/chat');
ws.onopen = () => ws.send(JSON.stringify({ type: 'join', room: 'general' }));
ws.onmessage = e => console.log(JSON.parse(e.data));
ws.onclose = () => setTimeout(connect, 1000 + Math.random()*2000); // backoff
setInterval(() => { if (ws.readyState === 1) ws.send(JSON.stringify({ type: 'ping' })); }, 30000);

// 3.12 Priority hint
fetch('/critical.css', { priority: 'high' });
fetch('/non-critical.js', { priority: 'low' });
// hoặc <link rel="preload" fetchpriority="high" href="/hero.jpg" as="image">
```

```nginx
# 3.13 Server config
# Nginx
listen 443 ssl http2;
listen 443 quic reuseport; # H3
add_header Alt-Svc 'h3=":443"; ma=86400';

# Cloudflare: tự H2/H3
# Node: import { createSecureServer } from 'http2';
```

## 4. So sánh / Phân loại

| Tiêu chí | HTTP/1.1 | HTTP/2 | HTTP/3 |
|----------|----------|--------|--------|
| Framing | Text, 1 req/connection (queue) | Binary, multiplex nhiều stream/1 TCP | Binary, multiplex trên QUIC/UDP, stream độc lập |
| HOL blocking | Có (HTTP + TCP) | Còn TCP HOL | Không (QUIC) |
| Header nén | Không | **HPACK** (static+dynamic+Huffman) | **QPACK** (không HOL) |
| Handshake | TCP + TLS (2-3 RTT) | TCP + TLS 1.2/1.3 | QUIC + TLS1.3, **0-RTT** repeat |
| Migration | Không (đổi IP đứt) | Không | Có (connection ID) |
| Tối ưu | Concat, sharding, sprite | Split nhỏ, không concat | Như H2 + tốt hơn cho lossy/mobile |

|  | Server Push (deprecated) | 103 Early Hints |
|---|--------------------------|-----------------|
| Cơ chế | Server đẩy resource không cần request | Server gửi `Link: rel=preload` trước khi HTML xong, client tự fetch |
| Cache | Kém (đẩy dù client đã có) | Tốt (client check cache trước khi fetch) |
| Trạng thái | Deprecated | Thay thế |

|  | SSE | WebSocket |
|---|-----|-----------|
| Hướng | 1 chiều server→client | 2 chiều full-duplex |
| Protocol | HTTP | WS (upgrade) |
| Reconnect | Tự (3s + Last-Event-ID) | Tự code |
| Binary | Không | Có |
| Overhead | HTTP header mỗi event | 2-14 byte/frame sau handshake |
| Scaling | Stateless HTTP, dễ (như HTTP) | Stateful, cần sticky/Redis pub/sub |

| HPACK (H2) | QPACK (H3) |
|------------|------------|
| Dynamic table update đồng bộ, nếu update mất → block decode (HOL) | Dynamic table async, không block stream khác khi table chưa sync |
| Huffman | Tương tự |

## 5. Trade-off / Khi nào KHÔNG dùng

- **Không dùng concat/sharding tuning cho H1.1 khi đã H2/H3**: H2 multiplex làm concat vô nghĩa, thậm chí hại (1 file lớn mất cache granuality, 1 byte đổi invalidate cả bundle). H2/H3 nên **split nhỏ** + `immutable` per-chunk.
- **Không bật H3 mọi case không cân nhắc UDP**: QUIC trên UDP bị **corp firewall chặn** (chỉ cho TCP 443), và **CPU cao hơn** (user-space). Fallback về H2 phải mượt (Alt-Svc). Đo trước khi enforce H3-only.
- **Không dùng Server Push**: đã deprecated, Chrome bỏ. Dùng `103 Early Hints` + `preload`/`fetchpriority`.
- **Không chọn WebSocket khi chỉ cần 1 chiều**: 90% server push (notification, feed) thì **SSE đơn giản hơn 70%** — tự reconnect, HTTP thân thiện, multiplex H2. WS chỉ khi cần client gửi liên tục/low-latency/binary.
- **Không mở 100+ streams H2 vô tội vạ**: mỗi stream tốn HPACK table + server memory, server có `MAX_CONCURRENT_STREAMS` (thường 100-128). Quá giới hạn → queue.
- **Không dùng `fetch priority: high` cho mọi thứ**: priority chỉ hint, lạm dụng làm mất tác dụng. Chỉ `high` cho LCP resource (hero, CSS critical).
- **Khi nào KHÔNG dùng SSE**: cần binary, cần client push nhiều (chat game) → WS; hoặc cần request/response đơn giản → `fetch` polling/short-polling đủ.
- **Khi nào KHÔNG dùng H2 multiplex**: nếu vẫn phải support client HTTP/1.1 cũ (IE11, corp proxy H1-only) → giữ concat fallback.

## 6. Failure modes & Cách đo / Debug

- **Lỗi 1: H1.1 tuning (concat) làm hại H2**
  - Triệu chứng: bundle 2MB, đổi 1 dòng invalidate cả bundle, cache hit kém.
  - Fix: H2/H3 → code-splitting granualar, `splitChunks`, mỗi chunk hash + `immutable`.
  - Đo: Network → `Transferred` vs `Size`, `Cache-Control`, `nextHopProtocol`.

- **Lỗi 2: TCP HOL với H2 trên lossy network**
  - Triệu chứng: mobile loss 2% → H2 waterfall stall dài, TTFB spike dù bandwidth đủ.
  - Fix: bật H3/QUIC cho lossy, đo improvement.
  - Đo: `chrome://net-export`, WebPageTest loss simulation, `nextHopProtocol` so sánh `h2` vs `h3`.

- **Lỗi 3: WS không heartbeat → proxy/firewall đóng idle**
  - Triệu chứng: WS `close 1006` sau 60s idle dù không lỗi.
  - Fix: `ping/pong` 30s, server `ws.ping()`, client `setInterval(send ping)`.
  - Đo: Network → WS tab → Frames, `close` code, proxy logs.

- **Lỗi 4: WS scale lỗi — sticky missing**
  - Triệu chứng: 2 server, client A gửi nhưng B không nhận broadcast.
  - Fix: Redis pub/sub hoặc `sticky session` (IP hash) + message bus.
  - Đo: logs per-server, `X-Forwarded-For`.

- **Lỗi 5: SSE không set `Cache-Control: no-cache` → proxy cache**
  - Triệu chứng: event không tới, proxy trả cache.
  - Fix: `Cache-Control: no-cache`, `Connection: keep-alive`, `X-Accel-Buffering: no` (Nginx).
  - Đo: `curl -v /events` → `Content-Type: text/event-stream`.

- **Lỗi 6: H3 0-RTT replay attack**
  - Triệu chứng: 0-RTT early data bị replay (không idempotent POST).
  - Fix: chỉ 0-RTT cho **safe/idempotent** (`GET`), server reject 0-RTT cho `POST` hoặc check `Early-Data: 1`.
  - Đo: `Alt-Svc` header, `chrome://net-internals/#quic`.

- **Lỗi 7: Thiếu `Alt-Svc` fallback**
  - Triệu chứng: client H3 fail không downgrade H2.
  - Fix: `Alt-Svc: h3=":443"; ma=86400, h3-29=":443"; ma=86400`.

- **Công cụ**:
  - DevTools **Network** → cột **Protocol** (`h2`, `h3`, `http/1.1`), **Waterfall** (H2 multiplex start cùng lúc, H1 queue Stalled dài), **WS** tab.
  - `performance.getEntriesByType('navigation')[0].nextHopProtocol`
  - `chrome://net-internals/#http2`, `#quic`, `chrome://net-export`
  - `curl --http2 -I https://example.com`, `curl --http3` (build quiche)
  - WebPageTest, Lighthouse (HTTP/2 multiplex audit), Cloudflare Analytics (H3 ratio).

## 7. Câu hỏi tự kiểm tra

1. Phân biệt Head-of-Line blocking ở HTTP/1.1 vs HTTP/2 vs TCP — vì sao H2 giải HOL HTTP nhưng vẫn stall khi loss, còn H3/QUIC thì không?
2. HPACK vs QPACK khác gì và vì sao QPACK sinh ra cho QUIC? `0-RTT` và `connection migration` của QUIC hoạt động thế nào?
3. Khi nào chọn SSE (EventSource, unidirectional, tự reconnect) vs WebSocket (bidirectional, full-duplex) — so sánh overhead, firewall, HTTP/2 multiplex và bài toán phù hợp?

<details>
<summary>Đáp án 30s</summary>

1. **H1.1 HOL**: 1 connection FIFO, request sau đợi trước xong (hoặc mở 6 conn). **H2 HOL HTTP**: đã giải bằng multiplex (nhiều stream interleave trên 1 TCP), nhưng **TCP HOL**: 1 packet loss → TCP retransmit → mọi streams block vì TCP là byte stream có thứ tự. **H3/QUIC**: QUIC trên UDP, mỗi HTTP stream = QUIC stream độc lập, loss stream1 chỉ retry stream1, stream3 vẫn deliver → không HOL transport.
2. **HPACK** (H2) dynamic table update đồng bộ — nếu table entry chưa tới, decoder block mọi streams (HOL). **QPACK** (H3) thiết kế async, stream không block khi table chưa sync. **0-RTT**: sau lần đầu 1-RTT (QUIC+TLS1.3), lần sau client gửi early data (GET) kèm ClientHello ngay, giảm 1 RTT. **Migration**: QUIC connection ID không gắn 4-tuple IP/port, đổi WiFi→4G vẫn giữ connection, TCP thì đứt.
3. **SSE**: HTTP `text/event-stream`, 1 chiều server→client, `EventSource` tự reconnect 3s + `Last-Event-ID`, qua H2 multiplex, firewall thân thiện, text only, 6-connection limit. **WS**: upgrade `101`, 2 chiều full-duplex, binary+text, overhead 2-14 byte/frame, phải tự heartbeat/reconnect, stateful (scale cần Redis pub/sub/sticky). Chọn **SSE** cho notification/feed/ticker/progress (1 chiều, cần reconnect rẻ); **WS** cho chat/collaborative/game/trading (cần client push liên tục, low latency, binary).

</details>

---
*Tham khảo chi tiết: `docs/06-browser-web-platform.md` — Câu 114-117. Spec: [RFC 7540 HTTP/2](https://httpwg.org/specs/rfc7540.html), [RFC 9114 HTTP/3](https://www.rfc-editor.org/rfc/rfc9114), [RFC 6455 WebSocket](https://datatracker.ietf.org/doc/html/rfc6455), [MDN — SSE](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events).*
