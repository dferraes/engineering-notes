# Building an ActiveSync + EWS gateway from scratch in Go

## The problem

The goal: deliver the "Exchange experience" — one account that pushes email, calendar,
and contacts to every device, set up by typing just an address and password — but on
an **open-source mail stack** (Dovecot for mail, SOGo/Radicale for CalDAV/CardDAV).
There is no Microsoft Exchange anywhere in that stack.

The catch: the protocols that deliver that experience — **ActiveSync (EAS)** for
mobile and **Exchange Web Services (EWS)** for desktop — are Microsoft protocols.
EAS in particular isn't really documented; the spec describes the happy path, and
real clients diverge from it constantly. So I built a **gateway**: it speaks EAS and
EWS to the devices, and translates to IMAP, SMTP, CalDAV/CardDAV and ManageSieve on
the backend.

## Architecture

```
Device  ──EAS (mobile) / EWS (desktop)──▶  Gateway  ──┬─ IMAP (QRESYNC/CONDSTORE) ─▶ Dovecot
                                                       ├─ SMTP (STARTTLS) ─────────▶ Postfix
                                                       ├─ CalDAV / CardDAV ────────▶ SOGo/Radicale
                                                       └─ ManageSieve (OOF) ───────▶ Dovecot
```

The gateway holds **no mail of its own** — it's a stateless-ish translator with a
cache for sync state. That was a deliberate choice: the source of truth stays in the
mail server, so the gateway can be restarted, scaled, or rebuilt without data loss.

## The hard parts

### 1. ActiveSync is binary, not XML
EAS goes over the wire as **WBXML** — a tokenized binary encoding, not text. Every
request and response is a byte stream you have to encode/decode by hand against
per-codepage token tables. Writing that decoder safely meant defending against hostile
input from day one: depth limits, integer-overflow checks, a hard cap on string
length, and string-table bounds checking — otherwise a malformed device request is a
denial-of-service.

### 2. Incremental sync that doesn't lie
The whole value of the protocol is *delta* sync: tell the device only what changed.
Get it wrong and users see duplicate calendar events on every poll, or deletions that
never propagate. I leaned on **IMAP QRESYNC/CONDSTORE** to get reliable change/delete
detection for mail, and **etag-based diffing** for CalDAV/CardDAV. Sync state is keyed
per device and survives restarts.

### 3. Reverse-engineering real clients
The spec was not enough. iOS Mail in particular sends and expects bytes the
documentation doesn't mention. The only reliable way forward was to **capture real
traffic** from a working Exchange endpoint with mitmproxy and match my responses
**byte-for-byte** against that reference for a target iOS version. A lot of this
project was "diff my output against what the device actually accepts," not "read the
spec."

### 4. The Autodiscover zoo
"Just type your email and password" hides four different discovery mechanisms — POX
XML, a newer JSON variant, a SOAP `GetUserSettings` call, and Mozilla Autoconfig for
Thunderbird. Each client picks a different one. Supporting zero-config setup meant
implementing all of them.

### 5. Knowing when *not* to use your own protocol
The honest finding: **no Outlook variant reliably works with a third-party EAS/EWS
endpoint.** Microsoft's clients assume Microsoft's servers. Rather than fight that, the
gateway detects Outlook by User-Agent at Autodiscover time and hands it plain
IMAP/SMTP instead. Shipping the limitation explicitly beat shipping a broken
"supported" checkbox.

### 6. Push without Exchange
"Push email" on EAS is a long-lived `Ping`. To drive it from an open-source stack I
wired Dovecot's push-notification plugin into a Valkey (Redis-protocol) PUBSUB
channel; the gateway subscribes and releases the held `Ping` when new mail lands.

## Multi-tenancy

It runs for many domains on shared infrastructure, so every domain resolves its own
backend servers at request time, with per-domain feature gating and scoped admin
access. A central registry of named backend servers means a domain references a server
by ID instead of hard-coding hostnames — which made onboarding a new domain a config
change, not a deploy.

## Security & operations

Because the gateway sits on the public internet and talks to internal services, the
threat model got real attention: SSRF protection on any backend-connection test
(DNS validation + a connect-time dialer that blocks private/metadata IPs and prevents
DNS rebinding), constant-time comparison for secrets, strict TLS minimums, and
fail-closed rate limiting. It ships as a hardened container (read-only rootfs, dropped
capabilities, non-root, internal-only network for the cache and database).

Observability is first-class: Prometheus metrics for every EAS/EWS command and backend
call, Grafana dashboards, and Loki logs. One war story — the **synthetic probes that
check each domain from outside kept timing out against the server's own public IP**,
because Docker's NAT rules skip traffic originating from bridge interfaces (hairpin
NAT). The fix was host-level iptables redirect rules; the lesson was that "monitor it
the way a real client reaches it" has sharp edges in containerized networking.

## What I'd tell someone starting this

- **Capture real traffic before you trust any spec.** For undocumented-in-practice
  protocols, a packet capture from a known-good implementation is worth more than the
  RFC.
- **Treat every decoder as an attack surface.** Binary parsers facing the internet need
  limits before features.
- **Ship limitations honestly.** "Outlook falls back to IMAP" is a better answer than a
  support ticket six months later.
- **Keep the source of truth out of the translator.** A stateless gateway is one you can
  rebuild at 3am.
