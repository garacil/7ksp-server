# 7KSP/1 — 7ks Protocol

Protocol version: 1. Document status: draft 0.2.

This document defines everything needed to implement an interoperable 7KSP client or node. It is written so that a complete node implementation fits in a few thousand lines in any language, using only standard components: HTTPS, WebSocket, JSON, Ed25519, X25519, WebRTC.

The key words MUST, MUST NOT, SHOULD and MAY are to be interpreted as described in RFC 2119.

---

## 1. What the protocol does

7KSP answers a single question: *given an email address, how do I open a direct, encrypted, authenticated channel with one of that person's devices?*

To do so it defines four things:

1. **Identity**: how a device proves that it belongs to an email address.
2. **Directory**: where the devices of an email address are published and how they are looked up.
3. **Rendezvous**: how a node delivers a connection request to a device and helps it traverse NAT.
4. **Direct channel**: how the two devices verify that they are talking to each other and not to an intermediary.

It does not define what is transmitted over the channel. Chat, calls, files or any other application are negotiated by name when connecting and run on top.

### 1.1 Actors

| Actor | What it is |
|---|---|
| User | An email address |
| Device | An installation of the client, with its own keys |
| Domain node | Authoritative server for one or more domains, announced in the domain's DNS |
| Shard node | Server of the public directory, responsible for a part of the hash space |
| Federation | Set of shard nodes with a common signed table |
| Verifier | Service that sends a link to an email address and certifies that someone opened it |
| Home | Node a device is connected to right now and through which it is reached |
| Push gateway | Service that wakes up a sleeping mobile device |

A single server MAY play several roles.

### 1.2 Principles

- **The client is the storage unit.** No node ever stores application content. Whatever has not been delivered waits on the sending device.
- **What a node stores about a user fits on one line.** Its device entries, whether it is connected, and who is waiting for it to connect.
- **The protocol is neutral.** No federation or implementation gets special treatment. Domains are sovereign over their addresses.
- **Everything public is verifiable without trusting whoever serves it.** Every directory object carries its signature.

---

## 2. Conventions

### 2.1 Encoding

- All messages and objects are JSON (RFC 8259), UTF-8.
- Bytes (keys, signatures, hashes) are encoded as **base64url without padding** (RFC 4648 §5).
- Times are integers: Unix seconds, UTC.
- Unknown fields MUST be ignored. Fields defined as mandatory MUST be present.

### 2.2 Signatures

Every signed object has a `sig` field. The signature is computed as:

```
sig = base64url( Ed25519_sign( private_key, JCS( object without the "sig" field ) ) )
```

`JCS` is JSON Canonicalization Scheme (RFC 8785). With it, every implementation produces the same bytes to sign without agreeing on a field order.

To verify: remove `sig`, canonicalize, check against the indicated public key.

### 2.3 Cryptography

| Use | Algorithm |
|---|---|
| Signature | Ed25519 (RFC 8032) |
| Key agreement / encryption to a device | X25519 + libsodium *sealed box* (`crypto_box_seal`) |
| Hash | SHA-256 |
| Transport to nodes | TLS 1.3 (ordinary web certificate of the node) |
| Direct channel | WebRTC: DTLS 1.2+ and SRTP |

There is no cryptographic agility in version 1: an implementation does not have to negotiate anything. Changing an algorithm will be version 2.

### 2.4 Identifiers

| Identifier | Definition |
|---|---|
| `uid` | `sha256( normalize(email) )`, 32 bytes, lowercase hex when in a URL and base64url when in JSON |
| `dev_pk` | Ed25519 public key of the device, 32 bytes |
| `dev_dh` | X25519 public key of the device, 32 bytes |
| `nid` | `sha256( node_pk )`, where `node_pk` is the Ed25519 public key of the node |
| `dom_pk` | Ed25519 public key of the domain |
| `fed_pk` | Ed25519 public key of the federation |

`normalize(email)`: Unicode NFC form, lowercase, no leading or trailing whitespace. **No other transformation** (dots and `+` suffixes are not removed). The `uid` has no prefix and no salt: any system MAY compute it knowing only the email address.

Test vectors:

```
alice@example.com    → ff8d9819fc0e12bf0d24892e45987e249a28dce836a85cad60e28eaaa8c6d976
Alice@Example.COM    → ff8d9819fc0e12bf0d24892e45987e249a28dce836a85cad60e28eaaa8c6d976  (same uid)
garacil@7kas.com     → bfc98f4ae31a81b48d7865e69bf9fa3d657f23a80bb2af3c6aa42609032cec3b
```

The `prefix` of a `uid` is its first two bytes as a 16-bit integer (`0xff8d` = 65421 in the first vector).

---

## 3. Objects

### 3.1 Proof of ownership

Certifies that `dev_pk` belongs to `uid`. There are two forms, depending on who issues it.

Issued by a verifier (public directory):

```json
{
  "type": "attestation",
  "uid": "...", "dev_pk": "...",
  "verifier_pk": "...",
  "iat": 1762560000, "exp": 1765152000,
  "sig": "..."
}
```

Issued by the domain (domain node):

```json
{
  "type": "domainsig",
  "uid": "...", "dev_pk": "...",
  "domain": "7kas.com",
  "iat": 1762560000, "exp": 1765152000,
  "sig": "..."
}
```

Rules: `exp - iat` ≤ 30 days. An expired proof is worth nothing.

### 3.2 Device entry

The unit of the directory. The device itself publishes it.

```json
{
  "type": "device",
  "v": 1,
  "uid": "...",
  "dev_pk": "...",
  "dev_dh": "...",
  "name": "Laptop",
  "homes": ["https://node.7kas.com"],
  "apps": ["7ks-chat/1", "7ks-call/1", "7ks-file/1"],
  "seq": 3,
  "exp": 1765152000,
  "proof": { "type": "domainsig", "...": "..." },
  "sig": "..."
}
```

| Field | Meaning |
|---|---|
| `homes` | HTTPS base URLs of the nodes through which this device is reached, in order of preference |
| `apps` | Application protocols it accepts; names of the form `name/version` |
| `seq` | Strictly increasing per `dev_pk`. A node MUST NOT accept `seq` ≤ the stored one |
| `exp` | MUST NOT exceed `proof.exp` |
| `sig` | Signature with the private key of `dev_pk` |

A node receiving an entry MUST verify: `sig` against `dev_pk`; `proof.sig` against the corresponding verifier or `dom_pk`; that `proof.uid` and `proof.dev_pk` match those of the entry; `seq` and `exp`.

### 3.3 Identity record

The set of current entries of a `uid`. It has no signature of its own: each entry is verified separately.

```json
{ "type": "record", "uid": "...", "devices": [ { "type": "device", "...": "..." } ] }
```

A node MUST omit expired entries when serving it.

### 3.4 Node record

```json
{
  "type": "node",
  "v": 1,
  "nid": "...", "node_pk": "...",
  "url": "https://node.7kas.com",
  "roles": { "domains": ["7kas.com"], "shards": [], "verifier": false },
  "seq": 12, "exp": 1765152000,
  "sig": "..."
}
```

`url` is the base of the node's whole API (§8). A node has a single base URL; redundancy is achieved with DNS and load balancers, outside the protocol.

### 3.5 Shard table

```json
{
  "type": "shards",
  "v": 1,
  "fed_id": "7ks",
  "nodes": { "<nid>": "https://shard3.example", "...": "..." },
  "prefixes": [ ["<nid>", "<nid>", "<nid>"], "... 65536 entries ..." ],
  "verifiers": ["<verifier_pk>", "..."],
  "seq": 40, "exp": 1765152000,
  "prev_sig": "...",
  "sig": "..."
}
```

- `prefixes[i]` is the list of nodes responsible for prefix `i`, from 0 to 65535. Each list SHOULD have at least 3 nodes from different operators.
- `verifiers` are the keys whose attestations the federation accepts.
- `sig` is by `fed_pk`. `prev_sig`, present for 90 days after a key rotation, is the signature with the previous `fed_pk`.
- Indicative size: 65536 × 3 references ≈ 2 MB in JSON. A client downloads it once and refreshes it by `seq`.

### 3.6 Watch

The only thing a node stores persistently about traffic. It lives in the target's node.

```json
{ "watcher_uid": "...", "watcher_home": "https://nodeA.example", "target_uid": "...", "exp": 1763164800 }
```

---

## 4. Domains

A domain joins the network by publishing in its DNS:

```
_7ks.<domain>.   TXT   "v=7ksp1; url=https://node.<domain>; pk=<dom_pk in base64url>"
```

Alternative for those who cannot comfortably touch DNS: `https://<domain>/.well-known/7ks` returning

```json
{ "v": "7ksp1", "url": "https://node.example.com", "pk": "<dom_pk>" }
```

If both exist, the TXT wins. A client SHOULD cache the result respecting the DNS TTL and in no case for more than 24 hours.

A domain with this entry is **authoritative**:

- Only its node publishes entries for its users, always with `domainsig`.
- A shard node MUST NOT serve entries of a domain with a current DNS entry.
- A client MUST NOT accept, for that domain, entries with `attestation`.
- If the domain's node answers `404` for a `uid`, that user does not exist. The client MUST NOT query the public directory.

A node MAY serve several domains, each with its own `dom_pk`.

---

## 5. Federation

A federation is: a `fed_id`, a `fed_pk`, a signed shard table and a bootstrap URL from which to download it.

The client is configured with one or more federations. The protocol does not distinguish between them; Annex A describes the 7ks public federation, which is the default in the 7ks client.

Obligations of a shard node:

- Keep the table up to date: download it from the bootstrap URL or from any other shard node and accept it only if `sig` verifies against `fed_pk` and `seq` is greater.
- Replicate its prefixes: at most every 5 minutes, ask the other nodes of the same prefix for entries with a new `seq` (§8.4) and accept those that verify.
- Serve public reads (§8.1) without authentication.

Obligations of a federation operator (outside the protocol, but necessary): publish the admission criteria for nodes, maintain coverage of all 65536 prefixes and rotate `fed_pk` with `prev_sig`.

---

## 6. Resolution

Input: an email address. Output: its verified identity record, or certainty that it does not exist.

```
1. If there is a locally cached record with current entries → use it.
2. domain = part after '@'. Query _7ks.<domain> TXT, or /.well-known/7ks.
   2a. It exists:
       GET <url>/7ks/v1/record/<uid hex>
       200 → verify that every entry carries a domainsig valid against pk. Return.
       404 → does not exist. END.
3. No domain entry:
   prefix = uid[0..2]
   Pick a nid from prefixes[prefix]; GET <nodes[nid]>/7ks/v1/record/<uid hex>
   200 → verify that every entry carries an attestation from a verifier_pk in verifiers. Return.
   404 → try another nid of the prefix; if all say 404 → does not exist. END.
   307 → follow Location once.
4. Pinning (TOFU):
   - First time: store the set of dev_pk and, if there is a domain, dom_pk.
   - New device with a valid proof: add it and notify the application ("new device").
   - dom_pk different from the stored one: warn as possible impersonation; do not use without confirmation.
```

A client MAY delegate steps 2 and 3 to its home with `resolve` (§8.3), which does the same on its behalf. The result MUST still be verified on the client.

---

## 7. Enrollment and device linking

A single procedure signs up a new user, adds a device to an existing user and recovers the account after losing every device. The proof is always the same: opening a link sent to the email address.

### 7.1 Flow

```
Device                                   Verifier                       User's mailbox
    |-- POST /7ks/v1/verify {email, dev_pk, dev_dh} -->|
    |<-- 202 {req_id} ------------------------------|
    |                                               |-- link ----------->|
    |-- GET /7ks/v1/verify/<req_id>  (long-poll) -->|                    |
    |                                               |<-- user opens ---- |
    |<-- 200 {proof} -------------------------------|
    |-- build and sign the device entry
    |-- PUT /7ks/v1/record/<uid>/<dev_pk> {entry} --> home / domain node / prefix nodes
```

- The verifier is the domain node if the domain has a DNS entry; otherwise any node with `roles.verifier` whose key is in the federation's `verifiers`.
- The email contains two links with the same token: `https://<verifier>/7ks/v1/verify/<token>/confirm` to open in any browser and `7ks:verify?token=<token>` to open directly in the application.
- The `token` MUST be random with at least 128 bits and expire after 15 minutes. `req_id` and `token` MUST be different values: `req_id` is known to the device; `token` only to whoever reads the email.
- `GET /verify/<req_id>` answers `200 {proof}` once the token has been used, `202` while waiting (the server MAY hold the response for up to 30 s), `410` if it expired.

### 7.2 Verifier obligations

- Send the link only to the indicated address.
- Issue the proof only after the token has been opened.
- Rate-limit: SHOULD reject more than 5 requests per address per hour, and apply limits per network origin.
- Not keep the email after issuing the proof; keep only `uid`.

### 7.3 Unlinking

```
DELETE /7ks/v1/record/<uid>/<dev_pk>
body: { "uid": "...", "dev_pk": "...", "by": "<signing dev_pk>", "iat": ..., "sig": "..." }
```

`sig` MUST be from another current device of the same `uid`, or from the verifier (after a new link), or from `dom_pk`. The node MUST delete the entry and notify `record_changed` (§8.3) to connected clients subscribed to that `uid`.

### 7.4 Domains

A domain node MAY replace the link flow with whatever the domain decides (corporate login, manual enrollment), as long as the result is a `domainsig`.

---

## 8. Node API

The whole API lives under `<url>/7ks/v1/`. Read routes are unauthenticated HTTPS; write routes carry signed objects; the device session is a WebSocket.

### 8.1 Public reads

| Route | Response |
|---|---|
| `GET /record/<uid hex>` | `200` identity record · `404` · `307 Location` if the node is not responsible and knows who is |
| `GET /node` | `200` node record |
| `GET /shards` | `200` shard table (shard nodes only) |
| `GET /shards?since=<seq>` | `200` table · `304` if unchanged |

These routes MUST NOT require authentication or limit by origin beyond what is reasonable against abuse. Every response is verifiable by its signatures.

### 8.2 Writes

| Route | Body | Response |
|---|---|---|
| `PUT /record/<uid hex>/<dev_pk>` | device entry | `204` · `400` invalid entry · `409` seq not greater · `403` node not responsible |
| `DELETE /record/<uid hex>/<dev_pk>` | signed revocation (§7.3) | `204` · `403` |
| `POST /verify` | `{email, dev_pk, dev_dh}` | `202 {req_id}` · `429` |
| `GET /verify/<req_id>` | — | `200 {proof}` · `202` · `410` |
| `GET /verify/<token>/confirm` | — | confirmation page; marks the token as used |

A node is responsible for a `uid` if: it is the domain node of that domain, or it is a shard node of the prefix and the domain has no DNS entry. In any other case it answers `403` with `{ "responsible": ["<url>", "..."] }` if it knows.

### 8.3 Device session (WebSocket)

A device connected to a node opens `wss://<url>/7ks/v1/session`. While the session is open, the device is **online** on that node. Closing it is disconnecting. There is no other presence mechanism.

All messages are JSON objects with `t` (type). Requests carry `id` (a string chosen by the sender) and the response repeats it.

**Authentication.** The node opens with a challenge; the device answers signed.

```
← { "t": "challenge", "nonce": "<32 bytes>" }
→ { "t": "auth", "uid": "...", "dev_pk": "...", "nonce": "...", "host": "node.7kas.com",
    "push": { "gateway": "https://push.7ks.app", "token": "..." },      // optional
    "sig": "<signature of {uid, dev_pk, nonce, host}>" }
← { "t": "ok", "time": 1762560000,
    "ice_servers": [ { "urls": "stun:node.7kas.com:3478" },
                     { "urls": "turn:node.7kas.com:3478", "username": "...", "credential": "..." } ] }
```

The node MUST check that `dev_pk` has a current entry for `uid` (in its own directory if responsible; by resolving if not). `host` prevents a signature from being valid on another node. `ice_servers` follows the WebRTC format; TURN credentials SHOULD expire after 1 hour.

**Messages from device to node.**

| Message | Response | Function |
|---|---|---|
| `{t:"resolve", id, uid}` | `{t:"record", id, record}` · `{t:"error", id, code:"not_found"}` | The node runs §6 on behalf of the device |
| `{t:"connect", id, env}` | `{t:"sent", id, devices:[dev_pk]}` · `{t:"offline", id}` · `{t:"error", id, code:"unknown"}` | Deliver a signaling envelope (§9) to the online devices of the target |
| `{t:"watch", id, uid, ttl}` | `{t:"ok", id}` · `{t:"error", id, code:"limit"}` | Notify when `uid` is online (§11) |
| `{t:"unwatch", id, uid}` | `{t:"ok", id}` | |
| `{t:"subscribe", id, uids:[...]}` | `{t:"ok", id}` | Receive `record_changed` for those `uid` |
| `{t:"block", id, uid}` / `{t:"unblock", id, uid}` | `{t:"ok", id}` | The node discards everything coming from `uid` to this user |
| `{t:"ping"}` | `{t:"pong"}` | Keep the session alive; every 30 s |

**Messages from node to device.**

| Message | Function |
|---|---|
| `{t:"deliver", env}` | Signaling envelope received |
| `{t:"online", uid}` | A `uid` this device was waiting for is online |
| `{t:"record_changed", uid}` | The record of a subscribed `uid` changed; resolve again |

`error.code` for `connect`: `unknown` is returned both for non-existent and for blocked users; they are indistinguishable by design.

### 8.4 Between nodes

Nodes talk to each other over HTTPS with the same objects. Each node-to-node request carries:

```
Authorization: 7ks-node <nid> <ts> <sig>
```

with `sig = Ed25519( node_pk, JCS({ "method": ..., "path": ..., "ts": ..., "body_sha256": ... }) )`. The receiver MUST reject `ts` differing by more than 5 minutes and MUST know `node_pk` (from its node record, obtained with `GET /node` at the `url` matching `nid` in the shard table or in the domain's DNS entry).

| Route | Function |
|---|---|
| `POST /relay` `{env}` | Deliver an envelope to a device of this node. Responds like `connect` |
| `POST /watch` `{watch}` | Register a watch on this node (it is the target's node) |
| `POST /online` `{uid, watcher_uid}` | Tell this node that a target of one of its users is online |
| `GET /sync/<prefix>?since=<ts>` | Entries of the prefix modified since `since` (only between shard nodes of the same prefix) |

A node that receives a `connect` or a `watch` for a `uid` that is not its own MUST resolve (§6) and forward exactly once to a home of the target. A received `POST /relay` is never forwarded.

---

## 9. Signaling

### 9.1 Envelope

What the node transports. The node sees the target, the origin and the expiry; the content is opaque.

```json
{
  "id": "<uuid>",
  "to":   { "uid": "...", "dev": "<dev_pk> | *" },
  "from": { "uid": "...", "dev": "<dev_pk>" },
  "exp": 1762560120,
  "body": "<base64url>"
}
```

- `to.dev = "*"`: the node delivers to every online device of the `uid`.
- `exp`: MUST NOT exceed `now + 120 s`. A node that cannot deliver before `exp` discards the envelope. **A node MUST NOT queue envelopes**: if the target is not online at that moment, it answers `offline`.
- `body` ≤ 16 KB.

### 9.2 Body

```
plaintext = JCS({ "signal": <Signal>, "from_dev": "<dev_pk>", "sig": "<sender's Ed25519 signature over signal>" })
body      = sealed_box( target's dev_dh, plaintext )
```

The receiver MUST: open the sealed box with its X25519 key; check that `from_dev` is a current `dev_pk` of `from.uid` (resolving if it does not have it); verify `sig`. If any step fails, discard without answering.

### 9.3 Signals

```json
{ "type": "offer",  "sid": "<128 bits>", "apps": ["7ks-call/1", "sip/2"], "sdp": "<SDP offer>" }
{ "type": "answer", "sid": "...", "app": "7ks-call/1", "sdp": "<SDP answer>" }
{ "type": "reject", "sid": "...", "reason": "declined | busy | no_app | blocked" }
{ "type": "ice",    "sid": "...", "candidate": "<ICE candidate>", "mid": "..." }
{ "type": "bye",    "sid": "..." }
```

- `sid` is generated by the initiator, at random. Every signal of a session carries the same `sid`; those with an unknown or finished `sid` are discarded.
- `apps` is the list of application protocols the initiator proposes, in order of preference. The receiver picks one. The receiver's application decides whether to ring, accept silently or reject, according to the `app`; 7KSP only delivers.
- `sdp` is a WebRTC SDP description. It includes the DTLS fingerprint, which is what later binds the channel to the identity (§10).
- An `offer` sent with `to.dev = "*"` reaches several devices. On the first `answer`, the initiator MUST send `bye` to the others.

### 9.4 Timers

| Event | Value |
|---|---|
| No `answer` after `offer` | 60 s → `bye` |
| No ICE connection after `answer` | 30 s → `bye` |
| `ice` resend interval | per the ICE agent (trickle) |

---

## 10. Direct channel

The channel is a standard **WebRTC PeerConnection**. It is chosen because it solves in a single piece, available in every language and browser, what would otherwise have to be specified: ICE, DTLS, SRTP and data channels.

### 10.1 Establishment

1. The initiator creates the PeerConnection with the `ice_servers` received in its session, generates the offer and sends it in an `offer` signal.
2. The receiver creates its own, applies the offer, generates the answer and sends it in `answer`.
3. Both exchange candidates with `ice`.
4. ICE picks the pair: direct (host or reflexive) and, only if there is no other, TURN.

### 10.2 Binding to identity

The DTLS fingerprint (`a=fingerprint` in the SDP) traveled inside a body signed by `dev_pk` and encrypted for the target. Each side MUST check, when DTLS completes, that the fingerprint of the remote certificate matches the one in the SDP received. With that, nobody can interpose without the device's private key. There is no further exchange of fingerprints or keys.

### 10.3 Use

- The chosen application (`app` in the `answer`) owns the connection. By convention, it opens its data channels with `protocol` equal to the application name (`createDataChannel("main", { protocol: "7ks-chat/1" })`), and audio and video tracks are WebRTC's.
- 7KSP ends here. Nothing that goes over the channel touches a node again.
- If the connection drops, the application MAY try to reconnect with a new `offer` with the same `sid` (ICE restart) for 60 s; after that, a new `sid`.

### 10.4 Alternative transports

A native implementation MAY offer direct QUIC as an alternative to WebRTC, announcing it in `apps` with the suffix `+quic` (`7ks-file/1+quic`). It is optional and does not affect interoperability: every client MUST support WebRTC.

---

## 11. Deferred delivery

Replaces any message queue. The node does not store content; it stores who is waiting for whom.

When device A wants to deliver something to B and B is not online:

```
1. A sends connect → receives offline.
2. A keeps the content in its own store and sends watch {uid: B, ttl}.
   A's home does POST /watch on B's home (resolving B).
3. B opens a session on its home. The home goes through the watches with target_uid = B and, for each one:
   - if the watcher is connected to this same node → sends it online;
   - otherwise → POST /online to watcher_home, which delivers it over the session or turns it into a silent push.
   The watch is deleted when emitted.
4. A receives online, sends connect and delivers over the direct channel.
5. If A did not receive the notice (it was off and the push did not wake it), it retries on startup.
```

Rules:

- A client MUST retry everything pending on startup and when connectivity is recovered.
- A node MUST accept at most 256 active watches per `watcher_uid` and `ttl` ≤ 7 days. A watch is renewed by sending it again.
- `online` reveals nothing the watcher would not discover with a `connect`; even so, a user MAY block another and the node will discard their watches.

---

## 12. Push

A sleeping mobile device has no open session. To reach it, the device registers in `auth` a `push` with the URL of a gateway and an opaque token. The gateway is chosen by the device, not by the node.

When a node needs to wake a registered device without a session:

```
POST <gateway>/7ks/v1/push
{ "token": "...", "kind": "connect" | "online" }
```

- `connect`: high priority (VoIP push on iOS, high priority on FCM). Triggered by an envelope with an `offer` addressed to that device. The node MUST hold the envelope until `exp` (120 s at most) while waiting for the device to open a session; this is the only envelope retention allowed.
- `online`: low priority, silent; may not be delivered. §11.5 covers it.

The gateway receives no `uid`, `sid` or content. Anyone MAY operate a gateway for their own build of the client. UnifiedPush is a valid transport on Android without anybody's gateway: in that case `push.gateway` is the device's UnifiedPush endpoint.

---

## 13. Errors

HTTP responses use standard codes and a body `{ "error": "<code>", "detail": "..." }`. Session messages use `{ "t": "error", "id": ..., "code": "<code>" }`.

| code | Meaning |
|---|---|
| `invalid` | Malformed object or invalid signature |
| `expired` | Expired proof, entry or envelope |
| `not_found` | The `uid` does not exist on this node |
| `not_responsible` | The node is not responsible for that `uid` |
| `seq` | `seq` is not greater than the stored one |
| `unknown` | Unknown or blocked target |
| `offline` | No device of the target has an open session |
| `limit` | Quota exceeded |
| `unauthorized` | Invalid session or node authentication |

---

## 14. Limits

| Parameter | Value |
|---|---|
| Envelope: maximum `exp` / size | 120 s / 16 KB |
| Device entry: maximum `exp` | 30 days; renew at 20 |
| Devices per `uid` | 32 |
| Verification token | 15 minutes |
| Verification requests | 5 per address per hour |
| Watches per watcher / `ttl` | 256 / 7 days |
| TURN credentials | 1 hour |
| TURN per session | operator's quota; recommended 1 Mbit/s and 10 minutes |
| Session ping | 30 s; the node closes after 90 s without traffic |
| Shard table: `exp` | 7 days |

---

## 15. Security

**Identity model.** Controlling the email is controlling the account. This is a deliberate decision: it is what allows logging in from any device with just the email. The consequences are mitigated in the client, not in the protocol: a warning to the user and their contacts when a new device appears, and the ability to revoke it from any other.

**What a node sees.** The origin and target `uid` of every envelope passing through it, the watches, and which devices have an open session. It sees no signaling or application content, nor can it alter them. Whoever does not want a third party to see even that runs a domain node.

**What a malicious node can do.** Deny service to its users, falsely answer `offline` or `unknown`, and hold envelopes. It cannot read, modify or impersonate, because everything is signed with keys that never leave the devices.

**Address impersonation.** In a domain with a DNS entry it requires the domain's private key. In the public directory it requires fooling a verifier accepted by the federation. Client-side pinning turns a change of `dom_pk` into an alert, and a new device into a notice.

**Direct channel.** The DTLS fingerprint is bound to `dev_pk` by the signature of the signal body. An intermediary would have to forge that signature.

**Enumeration.** The `uid` is an unsalted hash, so whoever has a list of email addresses can check which ones are in the public directory. This is accepted: the directory returns no emails, only keys, and domains with their own node do not appear in it.

**Connection spam.** An `offer` only reaches a device with an open session and the application decides whether to bother the user. `block` cuts off the rest. Nodes SHOULD rate-limit `connect` per origin.

---

## 16. Registered identifiers

| Element | Value |
|---|---|
| DNS record | `_7ks.<domain>` TXT |
| Well-known | `/.well-known/7ks` |
| API prefix | `/7ks/v1/` |
| URI scheme | `7ks:` — `7ks:alice@example.com`, `7ks:alice@example.com?app=7ks-call/1`, `7ks:verify?token=…` |
| Application names | `name/version`, lowercase, no spaces; `7ks-` reserved for the 7ks client's applications |

---

## Annex A. 7ks public federation (informative)

| Field | Value |
|---|---|
| `fed_id` | `7ks` |
| `fed_pk` | to be published |
| Bootstrap URL | to be published |
| Initial verifiers | to be published |

The operator of the public federation publishes, outside this document: admission criteria for shard nodes, neutrality commitment, replica restoration period and `fed_pk` rotation procedure.

## Annex B. Complete example: A calls B

A is `garacil@7kas.com`, device A1, connected to the domain node N (`https://node.7kas.com`). B is `soporte@7kas.com`, devices B1 and B2, same node.

```
A1 → N   ws  {t:"resolve", id:"1", uid:<B>}
N  → A1  ws  {t:"record", id:"1", record:{devices:[B1, B2]}}        // A1 verifies domainsig and pins keys
A1       creates PeerConnection, generates SDP offer
A1 → N   ws  {t:"connect", id:"2", env:{to:{uid:<B>, dev:"*"}, from:{uid:<A>, dev:A1}, exp, body:sealed(B1, offer)}}
A1 → N   ws  {t:"connect", id:"3", env:{to:{uid:<B>, dev:B2}, ..., body:sealed(B2, offer)}}
N  → B1  ws  {t:"deliver", env}          // B1 has a session
N  → B2  push {kind:"connect"}           // B2 is asleep; N holds the envelope until exp
B2 → N   ws  auth…  → N → B2 {t:"deliver", env}
B1       opens the body, verifies A1's signature, the 7ks-call/1 application rings
B1 → N   ws  {t:"connect", env:{to:{uid:<A>, dev:A1}, body:sealed(A1, answer{app:"7ks-call/1"})}}
N  → A1  ws  {t:"deliver", env}
A1 → N   ws  {t:"connect", env:{to:{uid:<B>, dev:B2}, body:sealed(B2, bye)}}   // B2 stops ringing
A1 ↔ B1  ice signals through N; ICE connects; DTLS; fingerprints checked against the SDPs
A1 ↔ B1  direct audio and video. N no longer intervenes.
```
