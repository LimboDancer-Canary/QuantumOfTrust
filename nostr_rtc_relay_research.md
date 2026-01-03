# NostrRTC-Relay-Research

## Abstract

This note records the design exploration that led to the **NostrRTC-Relay** specification: a Nostr-native RTC signaling relay paired with a standard TURN service, with **privacy mode mandatory and locked**. We evaluated whether a Nostr client (in the Nostria ecosystem) can support Zoom-like browser-to-browser conferencing, what infrastructure is required for reliability, and how to align the design with Nostr conventions and existing NIPs.

## 1. Problem statement

We want **audio/video calling** inside the **Nostr ecosystem**.

Key requirements established in discussion:

- **Minimum feature**: real-time **audio + video**.
- Must work within the **Nostr relay model** (clients connect outbound to relays).
- Prefer **maximum privacy and security**.
- First determine **feasibility**, then derive an implementable strategy and plan.

## 2. Background: why WebRTC is feasible in a Nostr client

WebRTC already provides:

- **Media transport** (SRTP) between browsers.
- A requirement for **signaling** (offer/answer + ICE candidates) that can be carried over an existing messaging substrate.

Nostr provides:

- A ubiquitous, open relay-based substrate for message delivery.
- A well-defined client↔relay protocol and conventions (NIP-01), relay metadata (NIP-11), relay authentication (NIP-42), and modern encrypted message patterns (NIP-44, NIP-17).

**Conclusion**: WebRTC is feasible if we map **signaling** onto Nostr message exchange and provide a robust NAT traversal fallback.

## 3. Networking reality: NAT traversal and the need for TURN

We examined why “browser-to-browser” does not mean “no infrastructure”:

- Most clients are **not reliably reachable inbound** (NAT, CGNAT, enterprise firewalls).
- **STUN** enables discovery of public-mapped addresses and can allow direct P2P in many cases.
- Some cases (symmetric NAT / restrictive networks) require a **relay for media**.

This leads to the standard WebRTC architecture split:

- **Signaling relay**: helps peers exchange SDP/ICE.
- **TURN relay**: relays media when direct connectivity fails.

## 4. Design exploration

### 4.1 “STUN-only ephemeral”

We considered an approach where a STUN endpoint is launched per session (or per short window) and torn down after use.

Findings:

- STUN-only can work for some users but will fail for a non-trivial subset.
- A privacy-first product cannot accept “works sometimes” calls.

Outcome: **Rejected as a primary design** (may remain as a fallback optimization, but not sufficient).

### 4.2 “Ephemeral TURN server per call”

We explored the idea of launching a TURN server instance only for the duration of a call.

Findings:

- TURN allocations are already ephemeral; the *server* does not need to be.
- Making the TURN server process itself per-call adds complexity:
  - public IP allocation timing,
  - routing/port exposure,
  - ICE timing and potential ICE restarts.

Outcome: **Not the best tradeoff** for early implementation.

### 4.3 “Ephemeral from a pool” (TURN)

We then explored a pragmatic version: keep a pool of pre-provisioned public endpoints and allocate one per call.

Findings:

- This can reduce setup latency and isolate calls.
- Operationally heavier than necessary if our core privacy goals can be achieved with a long-lived TURN service plus short-lived credentials.

Outcome: viable, but likely **phase-2**.

### 4.4 Reusing the relay’s footprint and public IP

A key architectural question emerged:

> If we already run a Nostr relay on a dedicated host, can the RTC support inhabit the same footprint?

We determined:

- Yes: run TURN on the same host and IP, using **different ports**.
- The major exception is **TURNS on TCP/443**, which conflicts with the relay’s WSS endpoint on 443 unless:
  - TURN has a second IP on the same host, or
  - TURN uses a different port (e.g., 5349), or
  - we accept reduced reachability in very restrictive networks.

Outcome: **Co-locating** Nostr signaling and TURN on the same host is the baseline deployment model.

## 5. Key privacy/security decision: lock privacy mode

We made a deliberately strict product/security choice:

- **Privacy mode is default and locked.**

Implications:

1) **Media path**: clients MUST set `iceTransportPolicy = "relay"`, forcing TURN-only.
   - Prevents peer IP disclosure.
2) **Signaling privacy**: signaling MUST use **NIP-17 gift wrap**, reducing relay-visible metadata.
3) **Payload confidentiality**: inner signaling payloads and TURN credentials MUST be encrypted (NIP-44 inside gift wrap).
4) **Abuse resistance**: RTC actions require **NIP-42 authentication**.

This is a strong stance: reliability increases (TURN-only avoids P2P corner cases), and privacy improves (no peer IP disclosure), at the cost of TURN bandwidth costs and infrastructure dependence.

## 6. “Do we need to modify coturn?”

We examined the integration boundary with coturn and concluded:

- **No coturn modifications are required**.

Rationale:

- TURN is a specialized protocol and coturn is a mature, hardened implementation.
- The NostrRTC-Relay’s job is:
  - signaling (offer/answer/ICE/hangup),
  - authentication + policy,
  - TURN endpoint advertisement,
  - and TURN credential delivery over Nostr.

Coturn remains a standard TURN service in the same deployment footprint.

## 7. Resulting architecture

The final architecture (captured in the spec) is:

- **RTC Relay** (Nostr relay behavior for RTC signaling kinds)
  - NIP-01 protocol
  - NIP-11 metadata
  - NIP-42 auth required
  - no durable storage for RTC traffic
  - recipient-scoped forwarding
  - strict rate limits

- **TURN service (coturn)**
  - co-located on the relay host or operated by the same entity
  - TURN/TURNS URIs advertised by the relay
  - credentials delivered to participants via encrypted Nostr signaling events

- **Client behavior (privacy locked)**
  - NIP-17 gift wrap for all RTC signaling
  - NIP-44 encryption for inner payloads
  - TURN-only ICE policy

## 8. Compatibility and standardization plan

Because no NIP currently standardizes RTC signaling kinds/tags, the spec defines an **experimental kind range** and tagging scheme, with the intent to:

- prove interoperability among early clients,
- validate privacy and abuse controls,
- and later propose standardization as a NIP once the envelope stabilizes.

## 9. Why this design is “Nostr-native”

This design aligns with Nostr’s philosophy:

- **Relays are infrastructure** that users opt into.
- **Authentication** and **policy** are enforced at the relay edge.
- **Payloads are end-to-end encrypted**; relays minimize state and retention.

The RTC Relay extends this pattern to real-time communication while keeping the media protocol (TURN/WebRTC) in its appropriate domain.

---

## References (normative touchpoints)

| Area | NIP / Component | Role in the design |
|---|---|---|
| Relay protocol | NIP-01 | Base client↔relay message model and event semantics |
| Relay metadata | NIP-11 | Capability advertisement (RTC extensions included) |
| Relay authentication | NIP-42 | Mandatory auth to prevent spam/abuse |
| Payload encryption | NIP-44 | Encrypt signaling and TURN credentials (inner payloads) |
| Metadata resistance | NIP-17 | Gift-wrapped signaling (default + locked) |
| Expiration | NIP-40 | Short TTL guidance for transient RTC messages |
| TURN implementation | coturn | Standard TURN/TURNS service (no modifications required) |

### Reference URLs

| Subsystem | What to read first | URL |
|---|---|---|
| STUN | IETF RFC 8489 | `https://datatracker.ietf.org/doc/html/rfc8489` |
| TURN | IETF RFC 8656 | `https://datatracker.ietf.org/doc/rfc8656/` |
| ICE | IETF RFC 8445 | `https://datatracker.ietf.org/doc/html/rfc8445` |
| SRTP | IETF RFC 3711 | `https://datatracker.ietf.org/doc/html/rfc3711` |
| WebRTC API | W3C WebRTC 1.0 | `https://www.w3.org/TR/webrtc/` |
| WebRTC connectivity | MDN WebRTC Connectivity | `https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Connectivity` |
| Nostr NIPs index | nostr-protocol/nips | `https://github.com/nostr-protocol/nips` |
| NIP-01 | Basic protocol | `https://github.com/nostr-protocol/nips/blob/master/01.md` |
| NIP-11 | Relay information | `https://github.com/nostr-protocol/nips/blob/master/11.md` |
| NIP-17 | Gift wrap | `https://nips.nostr.com/17` |
| NIP-40 | Expiration | `https://nips.nostr.com/40` |
| NIP-42 | Relay auth | `https://nips.nostr.com/42` |
| NIP-44 | Encrypted payloads | `https://nips.nostr.com/44` |
| coturn | TURN server | `https://github.com/coturn/coturn` |

---

## Appendix: Decision log (short)

- **STUN-only**: rejected (unreliable).
- **Per-call ephemeral TURN server**: rejected (operational complexity).
- **TURN from a pool**: viable later; not required to meet privacy goals.
- **Co-locate TURN with relay**: accepted (same footprint, simplest).
- **Privacy mode locked**: accepted (TURN-only + NIP-17 for signaling).
- **No coturn modifications**: accepted (stable integration boundary).

