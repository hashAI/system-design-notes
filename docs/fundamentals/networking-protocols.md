# Networking protocols — TCP, UDP, HTTP, gRPC (practical guide)

Choosing the right protocol affects latency, throughput, and system design. This guide covers when to use TCP vs UDP, HTTP/1.1 vs HTTP/2 vs gRPC.

---

## 0) The default mental model (when to use which)

- **TCP**: Reliable, ordered delivery. Use for almost all request-response and streaming. Default choice.
- **UDP**: Unreliable, low latency. Use for real-time (video, gaming, VoIP) where latency matters more than reliability.
- **HTTP/1.1**: One request per connection (or pipelining). Simple, universal.
- **HTTP/2**: Multiplexing, header compression. Better for many small requests.
- **gRPC**: HTTP/2 + protobuf. Best for service-to-service, streaming, performance.

---

## 1) TCP vs UDP

### TCP (Transmission Control Protocol)

- **Reliable**: Retransmits lost packets, guarantees delivery.
- **Ordered**: Packets arrive in order.
- **Connection-oriented**: Handshake (SYN, SYN-ACK, ACK) before data.
- **Flow control**: Prevents overwhelming receiver.
- **Congestion control**: Backs off under network congestion.

**Use for**: APIs, DB connections, file transfer, most application traffic.

### UDP (User Datagram Protocol)

- **Unreliable**: No retransmission; packets can be lost.
- **Unordered**: Packets can arrive out of order.
- **Connectionless**: No handshake; send and forget.
- **Low overhead**: No connection state, smaller headers.
- **Low latency**: No retransmit wait; good for real-time.

**Use for**: Video streaming, gaming, VoIP, DNS (simple queries), metrics (loss acceptable).

### When to choose

| Use TCP when | Use UDP when |
|--------------|--------------|
| Correctness matters (APIs, DB) | Latency matters more than loss |
| You need ordered delivery | Real-time (video, gaming, VoIP) |
| You can tolerate connection setup | You send many small, independent messages |
| | Loss of a packet is acceptable (next frame replaces it) |

---

## 2) HTTP/1.1

### Characteristics

- One request per connection (or pipelining, rarely used).
- Text-based (headers, body).
- Stateless.
- Connection overhead: new TCP connection per request (or connection reuse with Keep-Alive).

### Limitations

- **Head-of-line blocking**: One slow request blocks others on same connection.
- **Many connections**: Browsers open 6–8 connections per origin to parallelize.
- **Redundant headers**: Same headers sent repeatedly.

### When to use

- Public APIs (broad compatibility).
- Simple integrations.
- When HTTP/2 or gRPC aren't available.

---

## 3) HTTP/2

### Characteristics

- **Multiplexing**: Many streams over one TCP connection; no head-of-line blocking at HTTP layer.
- **Header compression** (HPACK): Reduces redundant header data.
- **Binary framing**: More efficient than text.
- **Server push**: Server can push resources before client asks.

### Benefits

- Fewer connections, lower latency for many small requests.
- Better utilization of single connection.

### Limitation

- **TCP head-of-line blocking**: If one TCP packet is lost, all streams on that connection stall until retransmit. (HTTP/3/QUIC addresses this with UDP.)

### When to use

- Web APIs with many concurrent requests.
- Service-to-service where both support HTTP/2.
- Foundation for gRPC.

---

## 4) gRPC

### Characteristics

- Built on HTTP/2.
- **Protocol Buffers** (binary serialization): Smaller, faster than JSON.
- **Strongly typed**: Code generation from `.proto` files.
- **Streaming**: Client, server, or bidirectional streams.

### Benefits

- High performance (binary, multiplexing).
- Streaming (good for real-time, large data).
- Code generation, contract-first.
- Widely used in microservices.

### Limitations

- Less universal than REST (browser support requires gRPC-Web).
- Binary format (harder to debug than JSON).
- Steeper learning curve for teams new to protobuf.

### When to use

- Service-to-service communication.
- When you need streaming (e.g., real-time updates, large file transfer).
- When performance and type safety matter.

---

## 5) REST vs gRPC (quick comparison)

| Aspect | REST (HTTP/JSON) | gRPC |
|--------|------------------|------|
| **Format** | JSON (text) | Protobuf (binary) |
| **Streaming** | Limited (SSE, chunked) | Native (unary, client, server, bidirectional) |
| **Performance** | Good | Better (smaller payloads, multiplexing) |
| **Browser support** | Full | Needs gRPC-Web |
| **Contract** | OpenAPI (optional) | .proto (required) |
| **Use case** | Public APIs, web | Internal services, streaming |

---

## 6) WebSockets (when you need long-lived connections)

- Full-duplex over single TCP connection.
- Good for: chat, live updates, collaborative editing.
- Persists connection; low latency for frequent small messages.
- Not HTTP/2 (different protocol); often used alongside REST for different purposes.

---

## 7) Common interview questions (and answers)

**Q: When would you use UDP over TCP?**  
A: When latency matters more than reliability — video streaming, gaming, VoIP. A lost packet can be skipped; retransmitting would add latency and hurt real-time experience.

**Q: What's the main benefit of HTTP/2?**  
A: Multiplexing — many requests over one connection without head-of-line blocking. Plus header compression. Reduces connection overhead and improves latency for many small requests.

**Q: When would you choose gRPC over REST?**  
A: For service-to-service communication when we need performance, streaming, or strong typing. For public APIs, REST is often better for compatibility.

**Q: What is head-of-line blocking?**  
A: When one slow or stuck request blocks others. HTTP/1.1 has it at the request level (one request per connection). HTTP/2 fixes that with multiplexing, but TCP still has it — one lost packet stalls all streams on that connection.

---

## 8) Interview-ready summary

"TCP is reliable and ordered — we use it for almost everything. UDP is for real-time when latency beats reliability. HTTP/2 adds multiplexing and compression over HTTP/1.1. gRPC uses HTTP/2 and protobuf for high-performance service-to-service and streaming. We choose based on latency, reliability, and compatibility needs."
