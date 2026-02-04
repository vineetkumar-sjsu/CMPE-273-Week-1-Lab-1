# Python HTTP Track - Week 1 Lab 1

Lab Description: Simple distributed system with two services communicating over HTTP done using flask and requests module.

**Name:** Vineet Kumar

**Email:** vineet.kumar@sjsu.edu

## How to Run Locally

You'll need two terminal windows for this.

**Terminal 1 - Start Service A (Echo API):**
```bash
cd service-a
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python app.py
```

**Terminal 2 - Start Service B (Client):**
```bash
cd service-b
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
python app.py
```

## Testing

### Success Case (both services running)

```bash
curl "http://127.0.0.1:8081/call-echo?msg=hello"
```

Output:
```json
{"service_a":{"echo":"hello"},"service_b":"ok"}
```

Service A logs:
```
2026-02-04 14:29:46,799 service=A endpoint=/echo status=ok latency_ms=0
```

Service B logs:
```
2026-02-04 14:29:46,801 service=B endpoint=/call-echo status=ok latency_ms=22
```

### Failure Case (Service A is down)

Stop Service A (Ctrl+C in Terminal 1), then run:
```bash
curl "http://127.0.0.1:8081/call-echo?msg=hello"
```

Output:
```json
{"error":"HTTPConnectionPool(host='127.0.0.1', port=8080): Max retries exceeded with url: /echo?msg=hello (Caused by NewConnectionError(\"HTTPConnection(host='127.0.0.1', port=8080): Failed to establish a new connection: [Errno 61] Connection refused\"))","service_a":"unavailable","service_b":"ok"}
```

HTTP Status: **503 Service Unavailable**

Service B logs:
```
2026-02-04 14:30:33,278 service=B endpoint=/call-echo status=error error="HTTPConnectionPool... Connection refused" latency_ms=2
```

## What Makes This Distributed?

This system is distributed because the two services run as separate processes on different ports and communicate over the network using HTTP. Neither service shares memory with the other - they have to explicitly send messages (HTTP requests) to communicate. Service B has no way to know if Service A is up or down except by trying to call it and handling the response or error. This is a fundamental property of distributed systems: each component is independent and can fail without directly crashing the others. When Service A goes down, Service B keeps running and gracefully returns an error to the client instead of dying.

### What happens on timeout?

Service B sets a 1-second timeout when calling Service A (`timeout=1.0` in the requests call). If Service A takes longer than 1 second to respond, the requests library raises a `ReadTimeout` exception. The except block catches this and returns a 503 with the error message. This prevents Service B from hanging forever waiting for a slow or unresponsive Service A.

### What happens if Service A is down?

When Service A is not running, Service B's HTTP request fails immediately with a `ConnectionRefused` error. The try/except block catches this exception, logs it with `status=error`, and returns a 503 response to the client with `service_a: "unavailable"`. The key thing is that Service B doesn't crash - it handles the failure gracefully and keeps accepting new requests.

### What do your logs show, and how would you debug?

The logs show structured info for each request:
- `service=` which service logged it (A or B)
- `endpoint=` which endpoint was hit
- `status=` ok or error
- `latency_ms=` how long the request took
- `error=` (only on errors) what went wrong

To debug an issue, I'd:
1. Check both service logs for any `status=error` entries
2. Look at the latency - if it's exactly 1000ms, it's probably a timeout
3. Check if the error message mentions "Connection refused" (Service A is down) vs "ReadTimeout" (Service A is slow)
4. Compare timestamps between Service A and B logs to trace a request flow
5. If nothing in the logs, check if the services are actually running on the right ports
