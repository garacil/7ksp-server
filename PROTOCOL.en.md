# 7KSP/1 — 7ks Protocol

Protocol version: 1. Document status: draft 0.6 (draft, unreleased).

7KSP is an open and free protocol that, starting from an email address, opens a direct, encrypted, authenticated channel between two devices. This document contains everything needed to implement an interoperable client or node without consulting any other source. It is written so that a complete implementation fits in a few thousand lines in any language, using only standard components: HTTPS and WebSocket towards nodes, JSON, Ed25519, X25519, the Noise framework, STUN, UDP and, where a browser is involved, WebRTC.

---

## 0. How to read this document

### 0.1 Key words

MUST, MUST NOT, SHOULD, SHOULD NOT and MAY are to be interpreted as described in RFC 2119. An implementation that breaks a MUST is not conformant.

### 0.2 Conformance levels

Every implementation declares one or more of these roles. The table shows which sections are mandatory for each one.

| Role | What it is | Mandatory sections |
|---|---|---|
| Native client | Desktop or mobile app with UDP sockets | 2–4, 7, 8, 9.3, 10, 11 (`noise` profile; `webrtc` recommended), 12, 13 |
| Web client | App running in a browser | 2–4, 7, 8, 9.3, 10, 11 (`webrtc` profile), 12, 13 |
| Domain node | Authoritative server for one or more domains | 2–5, 8, 9, 11.2 (STUN), 12, 13, 14 |
| Shard node | Server of the public directory | 2–4, 6, 9.1, 9.2, 9.4, 11.2 (STUN), 14 |
| Verifier | Email-link service | 2–4, 8 |
| Relay | TURN server announced on the network | 11.2 |
| Push gateway | Service that wakes mobile devices | 13 |

A single server MAY play several server roles; typically a domain node is also the verifier and relay for its users.

### 0.3 Glossary

| Term | Definition |
|---|---|
| User | An email address. It has no keys of its own. |
| Device | An installation of the client, with its own keys. A user has one or more. |
| `uid` | User identifier: SHA-256 of the normalized email. |
| Device entry | Signed object a device publishes: its keys, how to reach it, what it supports. |
| Identity record | Set of current device entries of a `uid`. |
| Proof of ownership | Object signed by a verifier or by a domain that binds a device key to a `uid`. |
| Node | Server that speaks 7KSP. It has an HTTPS base URL and an Ed25519 key. |
| Home | Node a device is connected to right now and through which signaling reaches it. |
| Domain node | Node authoritative for the addresses of one or more domains, announced in the domain's DNS. |
| Shard node | Node of the public directory, responsible for a part of the `uid` space. |
| Shard, prefix | Partition of the public directory: the first two bytes of the `uid`, from 0 to 65535. |
| Federation | Set of shard nodes that share a table signed by a federation key. |
| Verifier | Service that sends a link to an email and, if someone opens it, issues a proof of ownership. |
| Rendezvous | Delivering a connection request to a device, with the data needed to meet through NAT. |
| Envelope | Signaling message a node transports: target, origin, expiry and an opaque body. |
| Signal | Content of the envelope, encrypted for the target and signed by the origin: offer, answer, candidates, bye. |
| Direct channel | UDP connection between two devices, encrypted and authenticated by their keys, with no node in the path. |
| Transport profile | Concrete form of the direct channel: `noise` or `webrtc`. Same guarantees, different bytes on the wire. |
| Relay | TURN server that forwards encrypted packets when two devices cannot reach each other directly. |
| Watch | Notice of interest: "tell me when this `uid` is online". The only thing a node stores about traffic. |
| App | Application protocol running over the direct channel, identified by `name/version`. |

---

## 1. Overview

### 1.1 What the protocol does

7KSP answers a single question: *given an email address, how do I open a direct, encrypted, authenticated channel with one of that person's devices?* To do so it defines four things:

1. **Identity** (§8): how a device proves that it belongs to an email address. The proof is opening a link sent to that address.
2. **Directory** (§4–§7): where the devices of an email address are published and how they are looked up. A domain with its own node is the authority over its addresses; the rest goes to a public directory spread by the hash of the email.
3. **Rendezvous** (§9–§10): how a node delivers a connection request to a device and how the two devices find each other through NAT.
4. **Direct channel** (§11): a UDP connection between the two devices, encrypted and authenticated by their keys, with no certificate authority and no node in the path. It exists in two profiles, `noise` and `webrtc`, with the same guarantees; which one is used is negotiated per session.

It does not define what travels over the channel. Chat, calls, files or any other application are negotiated by name when connecting and run on top.

### 1.2 The journey of a connection in ten lines

```
1. Device A1 (garacil@7kas.com) wants to call soporte@7kas.com.
2. It resolves the email: DNS of 7kas.com → node N → B's identity record, signed. It pins the keys (TOFU).
3. It prepares network candidates (its public IP via STUN) and an offer: which app, which transports, how to find it.
4. It encrypts the offer for each of B's devices, signs it and puts it in an envelope.
5. It sends the envelope to its home. The home delivers it to B's connected devices; sleeping ones are woken by push.
6. B1 receives the offer, the application decides to ring. The user accepts.
7. B1 answers with its candidates, encrypted and signed, over the same path.
8. A1 and B1 send UDP probes to each other at the same time; their NATs open; they pick an address pair.
9. Noise (or DTLS) handshake: each one checks that the other is exactly the key from the directory.
10. Audio as datagrams, call signaling over a stream. The node no longer intervenes.
```

### 1.3 Actors

| Actor | What it is |
|---|---|
| User | An email address |
| Device | An installation of the client, with its own keys |
| Domain node | Authoritative server for one or more domains, announced in the domain's DNS |
| Shard node | Server of the public directory, responsible for a part of the hash space |
| Federation | Set of shard nodes with a common signed table |
| Verifier | Service that sends a link to an email and certifies that someone opened it |
| Home | Node a device is connected to right now and through which it is reached |
| Relay | Node that forwards encrypted UDP packets between two devices that cannot reach each other directly |
| Push gateway | Service that wakes a sleeping mobile device |

### 1.4 Principles

- **The client is the storage unit.** No node ever stores application content. Whatever has not been delivered waits on the sending device.
- **What a node stores about a user fits on one line.** Its device entries, whether it is connected, and who is waiting for it to connect.
- **There is always NAT.** Both devices are assumed to be behind NAT and unable to accept incoming connections. The whole direct channel starts from there.
- **Trust is in keys, not in certificates.** The directory already tells every device the keys of the other. The direct channel needs no certificate authority and no domain name: it works like SSH, not like HTTPS. This holds for both profiles.
- **One protocol, several transports.** Identity, directory, rendezvous, deferred delivery and the primitives the application sees are the same everywhere. Only the bytes on the wire between two devices change, and the application never notices.
- **The protocol is neutral.** No federation or implementation gets special treatment. Domains are sovereign over their addresses.
- **Everything public is verifiable without trusting whoever serves it.** Every directory object carries its signature.

---

## 2. Encoding and cryptography

### 2.1 Encoding

- All messages and objects exchanged with nodes are JSON (RFC 8259) in UTF-8.
- Bytes (keys, signatures, hashes) are encoded as **base64url without padding** (RFC 4648 §5): alphabet `A–Z a–z 0–9 - _`, no trailing `=`.
- Times are integers: Unix seconds in UTC.
- Unknown fields MUST be ignored. Fields defined as mandatory MUST be present; their absence makes the object invalid.
- Packets of the `noise` profile (§11.4) are binary, with integers in big-endian.

### 2.2 Signatures

Every signed object has a `sig` field. The signature is computed over the canonical serialization of the object without that field:

```
sign(object, private_key):
    o = copy of object without the "sig" field
    bytes = JCS(o)                       // RFC 8785: sorted keys, no whitespace, canonical numbers
    return base64url( Ed25519_sign(private_key, bytes) )

verify(object, public_key):
    o = copy of object without the "sig" field
    return Ed25519_verify(public_key, JCS(o), base64url_decode(object.sig))
```

JCS (JSON Canonicalization Scheme) guarantees that every implementation produces exactly the same bytes to sign without agreeing on a field order or a number format. Implementations exist in all common languages; where there is none, it is a hundred lines.

### 2.3 Algorithms

| Use | Algorithm | Reference |
|---|---|---|
| Signature of objects and signaling | Ed25519 | RFC 8032 |
| Encryption of signaling to a device | libsodium *sealed box*: ephemeral X25519 + XSalsa20-Poly1305 | `crypto_box_seal` |
| Identifier hash | SHA-256 | FIPS 180-4 |
| Transport to nodes | HTTPS and WSS, TLS 1.3, ordinary web certificate of the node | RFC 8446 |
| Direct channel, `noise` profile | `Noise_IK_25519_ChaChaPoly_BLAKE2s` over UDP | Noise rev. 34 |
| Direct channel, `webrtc` profile | DTLS 1.2 or later with a self-signed certificate; SRTP for media | RFC 8827, RFC 8832 |

There is no cryptographic agility in version 1: an implementation negotiates nothing. Changing an algorithm will be version 2 of the protocol.

### 2.4 Sealed box

Encrypts a message for an X25519 public key without the sender needing a long-term key of its own:

```
sealed_box(target_pk, message):
    (epk, esk) = X25519_keypair()                       // ephemeral, discarded afterwards
    nonce = BLAKE2b-24( epk || target_pk )
    ct = XSalsa20-Poly1305( key = X25519(esk, target_pk), nonce, message )
    return epk(32) || ct
```

The receiver recomputes the nonce with its own public key and decrypts with `X25519(target_sk, epk)`. This is exactly libsodium's `crypto_box_seal`, available in every language; an implementation SHOULD use the library rather than reimplement it.

---

## 3. Identifiers

| Identifier | What it is | Size |
|---|---|---|
| `uid` | `sha256( normalize(email) )` | 32 bytes; lowercase hex inside a URL, base64url inside JSON |
| `dev_pk` | Ed25519 public key of the device. Signs its entry and its signaling | 32 bytes |
| `dev_dh` | X25519 public key of the device. Receives sealed signaling and is its identity in the `noise` profile | 32 bytes |
| `nid` | `sha256( node_pk )`, where `node_pk` is the Ed25519 public key of the node | 32 bytes |
| `dom_pk` | Ed25519 public key of the domain, published in its DNS | 32 bytes |
| `fed_pk` | Ed25519 public key of the federation | 32 bytes |

A device's keys are generated on the device and MUST NOT leave it. `dev_pk` and `dev_dh` are distinct keys, generated independently; one is not derived from the other.

### 3.1 Email normalization

```
normalize(email):
    1. Unicode NFC normalization.
    2. Lowercase (full Unicode lowercase mapping, not ASCII only).
    3. Remove leading and trailing whitespace.
    4. Nothing else.
```

Dots are not removed, `+tag` suffixes are not stripped, domains are not rewritten: `juan.perez+7ks@gmail.com` and `juanperez@gmail.com` are two different `uid`s even though Gmail delivers them to the same mailbox. The reason is that the protocol cannot know every provider's rules, and any rule it applied would break the promise that *anyone* computes the same `uid`.

### 3.2 Test vectors

```
alice@example.com    → ff8d9819fc0e12bf0d24892e45987e249a28dce836a85cad60e28eaaa8c6d976
Alice@Example.COM    → ff8d9819fc0e12bf0d24892e45987e249a28dce836a85cad60e28eaaa8c6d976  (same uid)
garacil@7kas.com     → bfc98f4ae31a81b48d7865e69bf9fa3d657f23a80bb2af3c6aa42609032cec3b
```

The `prefix` of a `uid` is its first two bytes read as a 16-bit integer: `0xff8d` = 65421 in the first vector; `0xbfc9` = 49097 in the third.

---

## 4. Objects

Every object carries `type`. Examples use `<…>` where a real value would go.

### 4.1 Proof of ownership

Certifies that `dev_pk` belongs to `uid`. There are two forms depending on who issues it; both are verified the same way.

Issued by a verifier, for addresses in the public directory:

```json
{
  "type": "attestation",
  "uid": "<uid>",
  "dev_pk": "<dev_pk>",
  "verifier_pk": "<verifier's Ed25519 key>",
  "iat": 1762560000,
  "exp": 1765152000,
  "sig": "<verifier's signature>"
}
```

Issued by a domain, for addresses of a domain with its own node:

```json
{
  "type": "domainsig",
  "uid": "<uid>",
  "dev_pk": "<dev_pk>",
  "domain": "7kas.com",
  "iat": 1762560000,
  "exp": 1765152000,
  "sig": "<signature with dom_pk>"
}
```

Rules:

- `exp - iat` MUST be ≤ 30 days (2 592 000 s).
- An expired proof is worth nothing: the entry carrying it is invalid.
- An `attestation` is only valid if `verifier_pk` is in the `verifiers` list of the shard table of the federation the client uses (§6).
- A `domainsig` is only valid if `sig` verifies against the `dom_pk` published in the DNS of `domain` (§5) and `domain` matches the email's domain.

### 4.2 Device entry

The unit of the directory. The device itself builds, signs and publishes it.

```json
{
  "type": "device",
  "v": 1,
  "uid": "<uid>",
  "dev_pk": "<dev_pk>",
  "dev_dh": "<dev_dh>",
  "name": "Laptop",
  "homes": ["https://node.7kas.com"],
  "transports": ["noise", "webrtc"],
  "endpoints": ["udp://203.0.113.7:7700"],
  "apps": ["7ks-chat/1", "7ks-call/1", "7ks-file/1"],
  "seq": 3,
  "exp": 1765152000,
  "proof": { "type": "domainsig", "uid": "<uid>", "dev_pk": "<dev_pk>", "domain": "7kas.com", "iat": 1762560000, "exp": 1765152000, "sig": "<…>" },
  "sig": "<signature with dev_pk>"
}
```

| Field | Mandatory | Meaning |
|---|---|---|
| `v` | yes | Format version: 1 |
| `uid`, `dev_pk`, `dev_dh` | yes | Identifiers (§3) |
| `name` | no | Human-readable name so the user recognizes their devices. ≤ 64 characters |
| `homes` | yes | HTTPS base URLs of the nodes through which this device is reached, in order of preference. 1 to 4 |
| `transports` | yes | Transport profiles it supports, in order of preference: `noise`, `webrtc`. At least one |
| `endpoints` | no | UDP addresses where it accepts a direct `noise` handshake without NAT traversal (§11.7). 0 to 4 |
| `apps` | yes | Application protocols it accepts, `name/version`. At least one |
| `seq` | yes | Strictly increasing per `dev_pk`. Using the Unix time at publication is recommended |
| `exp` | yes | Expiry. MUST NOT exceed `proof.exp` |
| `proof` | yes | Proof of ownership (§4.1) |
| `sig` | yes | Signature with the private key of `dev_pk` |

Validation that every node receiving an entry and every client using it MUST perform:

```
validate_entry(e):
    1. e.v == 1; mandatory fields present; correct types.
    2. verify(e, e.dev_pk)                                      // device signature
    3. e.proof.uid == e.uid  and  e.proof.dev_pk == e.dev_pk
    4. now < e.proof.exp  and  now < e.exp  and  e.exp <= e.proof.exp
    5. by e.proof.type:
         attestation: e.proof.verifier_pk ∈ federation verifiers; verify(e.proof, verifier_pk)
         domainsig:   e.proof.domain == domain(uid)*; verify(e.proof, dom_pk from DNS)
    6. on a node: e.seq > stored seq for e.dev_pk (if any)
    * a node knows the domain from the write route; a client, from the email it resolved.
```

### 4.3 Identity record

The set of current entries of a `uid`. It has no signature of its own: each entry is verified separately.

```json
{ "type": "record", "uid": "<uid>", "devices": [ { "type": "device", "…": "…" } ] }
```

A node MUST omit expired entries when serving it. A client MUST discard entries that fail validation and use the rest; if none validates, the record is treated as non-existent.

### 4.4 Node record

```json
{
  "type": "node",
  "v": 1,
  "nid": "<nid>",
  "node_pk": "<node's Ed25519 key>",
  "url": "https://node.7kas.com",
  "stun": "node.7kas.com:3478",
  "roles": { "domains": ["7kas.com"], "shards": [], "verifier": true, "relay": true },
  "seq": 12,
  "exp": 1765152000,
  "sig": "<signature with node_pk>"
}
```

| Field | Meaning |
|---|---|
| `url` | Base of the node's whole API (§9). A node has a single base URL; redundancy is achieved with DNS and load balancers, outside the protocol |
| `stun` | The node's STUN server, `host:port`. Every node MUST run one (§11.2) |
| `roles.domains` | Domains it is authoritative for |
| `roles.shards` | Prefixes of the public directory it serves (informative; the shard table is the authority) |
| `roles.verifier` | Whether it accepts verification requests (§8) |
| `roles.relay` | Whether it offers TURN relaying (§11.2) |

### 4.5 Shard table

```json
{
  "type": "shards",
  "v": 1,
  "fed_id": "7ks",
  "nodes": { "<nid>": "https://shard3.example", "<nid>": "https://shard9.example" },
  "prefixes": [ ["<nid>", "<nid>", "<nid>"], ["<nid>", "<nid>", "<nid>"], "… 65536 lists …" ],
  "verifiers": ["<verifier_pk>", "<verifier_pk>"],
  "seq": 40,
  "exp": 1765152000,
  "prev_sig": "<signature with the previous fed_pk, only after a rotation>",
  "sig": "<signature with fed_pk>"
}
```

- `prefixes[i]` is the list of `nid`s responsible for prefix `i`, from 0 to 65535. Each list SHOULD have at least 3 nodes from different operators.
- `nodes` maps each `nid` to its base URL; its `node_pk` is obtained with `GET /node` at that URL (§9.1).
- `verifiers` are the keys whose attestations the federation accepts.
- `sig` is by `fed_pk`. `prev_sig`, present for 90 days after a rotation, is the signature with the previous `fed_pk`, so that a client holding the old key can accept the table announcing the new one.
- Indicative size: 65536 × 3 references ≈ 2 MB in JSON. A client downloads it once and refreshes it by `seq` (§9.1).

### 4.6 Watch

The only thing a node stores persistently about traffic. It lives in the target's node.

```json
{ "watcher_uid": "<uid>", "watcher_domain": "example.com", "watcher_home": "https://nodeA.example", "target_uid": "<uid>", "exp": 1763164800 }
```

- `watcher_domain` is the domain of the watcher's address; the target's node uses it to check that the node registering the watch is a home of the watcher (§9.4).
- `watcher_home` is the URL at which the watcher's node wants to receive `online`: one of the `homes` of the watcher's entry.

---

## 5. Domains

### 5.1 DNS entry

A domain joins the network by publishing a TXT record:

```
_7ks.<domain>.   TXT   "v=7ksp1; url=https://node.<domain>; pk=<dom_pk in base64url>"
```

Format: `key=value` pairs separated by `;` and optional spaces. `v` MUST be `7ksp1`; `url` is the HTTPS base of the domain node; `pk` is `dom_pk`. A record missing any of the three, or with an unknown `v`, is ignored as if it did not exist.

Alternative for those who cannot publish TXT: `https://<domain>/.well-known/7ks`, returning

```json
{ "v": "7ksp1", "url": "https://node.example.com", "pk": "<dom_pk>" }
```

If both exist, the TXT wins. A client SHOULD cache the result respecting the DNS TTL and in no case for more than 24 hours; a negative answer (no entry) is cached for at most 1 hour.

### 5.2 Authority

A domain with a current entry is **authoritative** over its addresses:

- Only its node publishes entries for its users, always with `domainsig`.
- A shard node MUST NOT serve entries of a domain with a current DNS entry.
- A client MUST NOT accept, for that domain, entries with `attestation`.
- If the domain's node answers `404` for a `uid`, that user does not exist. The client MUST NOT query the public directory.

A node MAY serve several domains, each with its own `dom_pk`. A domain MAY change nodes by publishing a new URL; its entries remain valid because `dom_pk` signs them, not the node.

### 5.3 Joining and leaving

A domain whose users were in the public directory becomes authoritative just by publishing the DNS. From then on the public entries of that domain stop being accepted and expire on their own. Contacts who had pinned keys receive the `dom_pk` change notice (from "none" to one). If the domain withdraws the DNS, its users can again publish in the public directory with `attestation`. Joining and leaving is the domain owner's decision.

---

## 6. Federation

A federation is: a `fed_id`, a `fed_pk`, a signed shard table and a bootstrap URL from which to download it.

The client is configured with one or more federations (`fed_id`, `fed_pk`, bootstrap URL). The protocol does not distinguish between them; Annex A describes the 7ks public federation, which is the default in the 7ks client. Anyone MAY operate a different federation with the same software.

Obligations of a shard node:

- Keep the table up to date: download it from the bootstrap URL or from any other shard node, and accept it only if `sig` verifies against `fed_pk` (or `prev_sig` against the previous one during a rotation) and `seq` is greater than the one it holds.
- Replicate its prefixes: at most every 5 minutes, ask the other nodes of each of its prefixes for changed entries (`GET /sync`, §9.4) and accept those that validate.
- Serve public reads (§9.1) without authentication.
- Accept writes (§9.2) only for the prefixes the table assigns to it.

Obligations of a federation operator, outside the protocol but necessary: publish the admission criteria for shard nodes, maintain coverage of all 65536 prefixes with at least 3 nodes each, publish the `fed_pk` rotation procedure and a neutrality commitment (no filtering, prioritizing or discarding of valid entries).

---

## 7. Resolution

Input: an email address. Output: its verified identity record, or certainty that it does not exist.

```
resolve(email):
    uid = sha256(normalize(email)); domain = part after '@'
    1. If there is a cached record with current entries and the cache has not expired → return it.
    2. Look up the domain's DNS entry (§5.1).
       2a. It exists (url, dom_pk):
           GET <url>/7ks/v1/record/<uid hex>
           200 → keep the entries whose proof is a domainsig validating against dom_pk.
                 If any remain → pin(uid, entries, dom_pk); return.
                 If none remain → treat as 404.
           404 → DOES NOT EXIST. End. The public directory is not queried.
           network error → retry with backoff; if it persists, return "unavailable" (not "does not exist").
    3. No DNS entry:
       prefix = uid[0..2]; candidates = table.prefixes[prefix], in random order
       for each nid in candidates:
           GET <table.nodes[nid]>/7ks/v1/record/<uid hex>
           200 → keep the entries whose proof is an attestation from a verifier_pk ∈ table.verifiers and validates.
                 If any remain → pin(uid, entries, no dom_pk); return.
           404 → next candidate.
           307 → follow Location once, then continue.
           error → next candidate.
       If all answered 404 → DOES NOT EXIST. If none answered → "unavailable".
```

**Resolution by a node.** When a node resolves on behalf of a device (`auth` of a device whose directory it does not serve, `resolve`, `connect`, `watch`), it receives `uid` and `domain` from the device: the domain cannot be recovered from the `uid`, and without it step 2 is impossible. The node runs the same algorithm with the pair (`uid`, `domain`) in place of the address, validates the entries exactly as a client would, and MAY cache the validated record until the smallest `exp` of its entries and at most 1 hour. A node MUST NOT query or forward to any URL the algorithm did not produce: the `url` of the domain's DNS entry, the nodes the table (or the naming convention of Annex A) assigns to the prefix, and the `homes` of validated entries, each checked as in §9.4.

### 7.1 Key pinning (TOFU)

```
pin(uid, entries, dom_pk):
    known = local_store[uid]
    if known does not exist:
        store {devices: {dev_pk → dev_dh}, dom_pk}; notify the app: "new contact"
    if it exists:
        if known.dom_pk != dom_pk → notify the app: "possible impersonation: the domain key changed";
                                    do NOT use the new entries until the user confirms
        for each entry with an unknown dev_pk → add; notify the app: "new device of <uid>"
        for each known dev_pk now absent → mark as retired (do not delete: it may return with a higher seq)
```

### 7.2 Cache and refresh

- A client SHOULD cache a record until the smallest `exp` among its entries and at most 24 hours.
- A client SHOULD subscribe (§9.3 `subscribe`) to the `uid`s it talks to frequently, to receive `record_changed` and resolve again without waiting for expiry. A node emits `record_changed` for the `uid`s whose directory it serves; for the others it is not guaranteed, so a client SHOULD resolve a contact homed elsewhere again when its cached record is older than 1 hour at the start of a session with that contact, and after any `unknown` or validation failure.
- A client MAY delegate steps 2 and 3 to its home with `resolve` (§9.3), passing `uid` and `domain`. The home runs the same algorithm. The client MUST still validate the result: the home is a transport, not an authority.

---

## 8. Enrollment and device linking

A single procedure signs up a new user, adds a device to an existing user and recovers the account after losing every device. The proof is always the same: opening a link sent to the email address.

### 8.1 Flow

```
Device                                          Verifier                           User's mailbox
    |                                                |                                   |
    | generates dev_pk, dev_dh                       |                                   |
    |-- GET /7ks/v1/verify/challenge --------------->|                                   |
    |<- 200 {nonce, difficulty, exp} ----------------|                                   |
    | solves the proof of work (§8.9)                |                                   |
    |-- POST /7ks/v1/verify {email,keys,lang,pow} -->|                                   |
    |<- 202 {req_id, code} --------------------------|                                   |
    | shows the number (code)                        |-- email: number, link(token) ---->|
    |-- GET /7ks/v1/verify/<req_id> (waits 30 s) --->|                                   |
    |<- 202 (not yet) -------------------------------|                                   |
    |                                                |<- user opens link, sees number ---|
    |                                                |<- user presses Accept ------------|
    |-- GET /7ks/v1/verify/<req_id> ---------------->|                                   |
    |<- 200 {proof} ---------------------------------|                                   |
    | builds and signs its device entry              |                                   |
    |-- PUT /7ks/v1/record/<uid>/<dev_pk> {entry} --> responsible node (domain or shards)
    |<- 204 -----------------------------------------|
```

1. The device generates `dev_pk` and `dev_dh`.
2. It picks a verifier: the domain node if the domain has a DNS entry; otherwise any node with `roles.verifier` whose key is in the federation's `verifiers`.
3. When the verifier requires it, the device asks for a challenge and solves it (§8.9). Then `POST /verify` with the email in clear (the only operation of the protocol that carries the email), its public keys, its language and the solution. Response `202 {req_id, code}`; the device shows the number `code` (§8.8).
4. The verifier sends an email with two links carrying the same `token`: `https://<verifier>/7ks/v1/verify/<token>/confirm` to open in any browser, and `7ks:verify?token=<token>` to open directly in the application.
5. The user opens the link, sees the number and presses Accept (§8.8). Meanwhile the device waits with `GET /verify/<req_id>`. The server MAY hold the response for up to 30 s (long-poll); it answers `202` if not yet, `200 {proof}` once the user has accepted, `410` if it expired or was cancelled from the link.
6. The device builds its device entry with the `proof`, signs it and publishes it with `PUT /record` (§9.2) on the responsible node: the domain node, or one of the shard nodes of the prefix (which replicates it to the others).
7. The device opens a session on its home (§9.3). It is now online.
8. A client SHOULD then offer to import the user's history, contacts and preferences from another device of the same address, or from an export the user holds, before starting empty. How that import works is the application's business (the 7ks client uses `7ks-sync/1`, a document of its own); the protocol only guarantees that two devices of one address can open a direct channel like any other pair (§10.4).

Rules on identifiers:

- `req_id` and `token` MUST be different random values of at least 128 bits. `req_id` is known to the device; `token` only to whoever reads the email. Thus, whoever intercepts the request cannot confirm it, and whoever reads the email cannot collect the proof without the device.
- `token` expires after 15 minutes. `req_id` stops being queryable 5 minutes after the proof is delivered.

### 8.2 Email content

The verifier SHOULD send an email, in the language of the request, containing: the device name if provided, the number of the request (§8.8), the two links, the validity period and a sentence saying that if the user did not request anything they can ignore it. It MUST NOT include the keys or anything that identifies the request beyond the token.

### 8.3 Verifier obligations

- Send the link only to the indicated address, never to another.
- Issue the proof only after the user accepted on the confirmation page (§8.8); opening the link alone MUST NOT issue it.
- Rate-limit: SHOULD reject more than 5 requests per address per hour (`429` with `Retry-After`), apply limits per network origin, and MAY require a proof of work (§8.9).
- Not keep the email in clear after issuing the proof; keep only `uid`.
- Sign `iat` with its own clock; `exp = iat + 30 days` at most.

### 8.4 Renewal

Entries expire after 30 days. At 20 days the device SHOULD renew: in a domain, the node MAY renew the `domainsig` without a new link (the domain's decision); in the public directory, the flow of §8.1 is repeated with the same `dev_pk`. A device that does not renew disappears from the directory with no further consequence; when it returns, it repeats enrollment.

### 8.5 Unlinking

```
DELETE /7ks/v1/record/<uid hex>/<dev_pk>
Body:
{ "type": "revoke", "uid": "<uid>", "dev_pk": "<dev_pk to remove>", "by": "<signing key>", "iat": 1762560000, "sig": "<…>" }
```

`sig` MUST be from one of these keys: another current device of the same `uid` (`by` = its `dev_pk`), an accepted verifier after a new link (`by` = `verifier_pk`), or the domain (`by` = `dom_pk`). `iat` MUST be within 5 minutes of the node's clock. The node MUST delete the entry, propagate it to its replicas and notify `record_changed` (§9.3) to connected clients subscribed to that `uid`.

### 8.6 Recovery

Someone who has lost all their devices repeats enrollment on a new one: they obtain a new proof and publish a new entry. The entries of the lost devices remain valid until they expire; the user SHOULD revoke them from the new device (§8.5). Contacts receive "new device" and, on revocation, `record_changed`.

### 8.7 Domains

A domain node MAY replace the link flow with whatever the domain decides (corporate login, manual enrollment, management by an administrator), as long as the result is a `domainsig`. The rest of the protocol does not change.

### 8.8 Number match

The link proves control of the mailbox, not that the person who opens it is the person in front of the device that asked. They are different people when someone requests a link for another person's address, and when a user opens an old or unexpected mail. The number match binds the two.

1. The device MAY send `lang` in `POST /verify`: a lower-case primary language subtag (BCP 47), for example `en` or `es`. The verifier writes the email and the confirmation page in that language when it supports it, otherwise in its default language.
2. The verifier generates `code`, three decimal digits chosen uniformly at random (`000` to `999`), and returns it in the answer: `202 { "req_id": "…", "code": "472" }`. The device MUST show the number while it waits, with the instruction to accept only if the page shows the same number.
3. The email SHOULD carry the number, and the link SHOULD carry `?lang=<lang>`.
4. `GET /verify/<token>/confirm` MUST NOT change any state. It returns a page that shows the number, the device name if one was given, and two buttons, Accept and Cancel, that submit a form `POST /verify/<token>/confirm` with `action=accept` or `action=cancel`. Opening the link, whether by the user, a link scanner or a browser prefetch, issues nothing.
5. `action=accept` issues the proof and uses the token; the page tells the user to go back to the device. `action=cancel` cancels the request: the token is dead and `GET /verify/<req_id>` answers `410 { "error": "expired", "detail": "cancelled" }`, so the device can say that the request was cancelled from the link.
6. A used, cancelled or expired token gets a page saying that the link is no longer valid (`410`), in the language of the request.
7. An application that opens `7ks:verify?token=<token>` MUST show the same number and the same two choices before it sends the same `POST`.

The number is short on purpose: a person compares it, nobody has to guess it, and the token, not the number, is the secret. Its job is to make the user stop when the page shows a number that their device does not.

### 8.9 Proof of work on the request

A verifier sends mail on behalf of whoever asks, so it is a target for mass sign-ups and mail flooding. In addition to the limits of §8.3, a verifier MAY require a proof of work on `POST /verify`. Every client MUST support it: it costs a fraction of a second on a phone and the user never sees it.

1. **Challenge.** `GET /7ks/v1/verify/challenge` answers `200 { "v": 1, "nonce": "<base64url of 16 random bytes>", "difficulty": 18, "exp": 1762650120 }`. `nonce` is single use and valid until `exp`, at most 120 s after issue; `difficulty` is the number of leading zero bits required. A verifier that requires no work answers `404`, and the client sends its request without `pow`.
2. **Puzzle.**

```
preimage = nonce ":" email_normalized ":" counter        (ASCII bytes, no spaces)
valid    = leading_zero_bits( SHA-256(preimage) ) >= difficulty
```

   `nonce` is the 22-character string received. `email_normalized` is the address after §3.1, the same value the client sends in `email`, so a solution is bound to one address. `counter` is a decimal integer written without sign or leading zeros (`0` alone is valid), at most 20 digits. Leading zero bits are counted from the most significant bit of the first byte: `00 00 0f …` has 20.
3. **Solution.** In the body of `POST /verify`: `"pow": { "nonce": "<nonce>", "counter": "<decimal string>" }`. `counter` is a string so that no question of number precision arises.
4. **Verification, in this order.** `nonce` known, unused and not expired, otherwise `429 limit` with detail `challenge`; not answered sooner than 1 s after the challenge was issued, otherwise `429 limit`; `counter` well formed and the digest, recomputed with the verifier's own normalisation of `email`, has at least the `difficulty` of that nonce, otherwise `400 invalid` with detail `pow`. Then the verifier marks the nonce used and applies the limits of §8.3. A client SHOULD NOT send a solution sooner than 1 s after receiving the challenge; after a `429` it waits the seconds of `Retry-After` and asks for a new challenge.
5. **Difficulty.** It SHOULD be 18 and MUST NOT exceed 24, so that every client finishes in seconds; a verifier MAY raise it within that bound under load. A client MAY abandon a challenge above 24.
6. **Honeypot.** A verifier MAY treat a request whose `website` field is not empty as automated and answer it with a `202` that leads nowhere; the client's form renders that field invisible and never fills it. A client MUST NOT send `website` with content.

Test vectors (computed with an independent SHA-256; every client and verifier MUST reproduce them):

| | V3a | V3b |
|---|---|---|
| `nonce` | `AAECAwQFBgcICQoLDA0ODw` (bytes 00…0f) | `eywNj05qHJA19-K41KYOEQ` |
| `email_normalized` | `alice@example.com` | `garacil@7kas.com` |
| `difficulty` | 16 | 20 |
| smallest valid `counter` | `124804` | `1618843` |
| preimage | `AAECAwQFBgcICQoLDA0ODw:alice@example.com:124804` | `eywNj05qHJA19-K41KYOEQ:garacil@7kas.com:1618843` |
| SHA-256 | `0000f1f226808091af52ba1e68804011af223db07de17c80f2a56fa60d9425a5` | `00000b8ffc4158d862b253574d1c6a4708f1846b11326208f94b27fe3b29e4a2` |
| leading zero bits | 16 | 20 |

Negative checks: the counter `0124804` is malformed (leading zero); V3a's counter with V3b's nonce does not validate; `Alice@Example.COM` normalised gives the V3a digest, the unnormalised string does not.

---

## 9. Node API

The whole API lives under `<url>/7ks/v1/`. Read routes are unauthenticated HTTPS; write routes carry signed objects; the device session is a WebSocket. Nodes use HTTPS because they are ordinary servers with names and certificates; devices never use it among themselves.

### 9.1 Public reads

| Route | Response |
|---|---|
| `GET /record/<uid hex>` | `200` identity record · `404` · `307 Location` if the node is not responsible and knows who is |
| `GET /node` | `200` node record |
| `GET /shards` | `200` shard table (shard nodes only) |
| `GET /shards?since=<seq>` | `200` table · `304` if unchanged |

These routes MUST NOT require authentication or limit by origin beyond what is reasonable against abuse. Every response is verifiable by its signatures; a client MUST NOT trust a response because of the node it came from. `Content-Type: application/json`. Responses SHOULD carry `Cache-Control` consistent with the expiries of §7.2.

### 9.2 Writes

| Route | Body | Response |
|---|---|---|
| `PUT /record/<uid hex>/<dev_pk>` | device entry (§4.2) | `204` · `400 invalid` · `409 seq` · `403 not_responsible` |
| `DELETE /record/<uid hex>/<dev_pk>` | signed revocation (§8.5) | `204` · `400 invalid` · `403` |
| `GET /verify/challenge` | — | `200 {v, nonce, difficulty, exp}` (§8.9) · `404` if the verifier requires no work · `429 limit` |
| `POST /verify` | `{ "email": "…", "dev_pk": "…", "dev_dh": "…", "name": "…", "lang": "en", "pow": { "nonce": "…", "counter": "…" } }` | `202 {req_id, code}` (§8.8) · `400 invalid` (detail `pow`) · `429 limit` with `Retry-After` · `403` if the node is not a verifier for that domain |
| `GET /verify/<req_id>` | — | `200 {proof}` · `202` · `410 expired` (detail `cancelled` when cancelled from the link) |
| `GET /verify/<token>/confirm` | — | HTML page with the number and the Accept and Cancel buttons (§8.8); changes nothing · `410` page when the link is no longer valid |
| `POST /verify/<token>/confirm` | form `action=accept` or `action=cancel` | HTML page; `accept` uses the token and issues the proof, `cancel` cancels the request |

A node is responsible for a `uid` if: it is the domain node of that domain, or it is a shard node of the prefix and the domain has no DNS entry. In any other case it answers `403` with `{ "error": "not_responsible", "responsible": ["<url>", "…"] }` if it knows. `PUT` and `DELETE` need no session: the object carries its own authority. A node MUST reject (`invalid`) an entry whose `homes` contain a URL that is not `https` or whose host does not pass the address rule of §9.4, and SHOULD also require a valid node record at each of them (§9.4), because other nodes will forward to those URLs.

### 9.3 Device session (WebSocket)

A device connected to a node opens `wss://<host of url>/7ks/v1/session` with the subprotocol `Sec-WebSocket-Protocol: 7ksp.v1`. While the session is open, the device is **online** on that node. Closing it is disconnecting. There is no other presence mechanism.

All messages are JSON objects with `t` (type). Requests carry `id` (a string chosen by the sender, unique within the session) and the response repeats it. Unsolicited messages from the node carry no `id`.

**Authentication.** The node opens with a challenge; the device answers signed.

```
← { "t": "challenge", "nonce": "<32 random bytes>" }

→ { "t": "auth",
    "uid": "<uid>", "domain": "7kas.com", "dev_pk": "<dev_pk>",
    "nonce": "<the same>", "host": "node.7kas.com",
    "push": { "gateway": "https://push.7ks.app", "token": "<opaque>" },
    "sig": "<Ed25519 signature with dev_pk of JCS({uid, dev_pk, nonce, host})>" }

← { "t": "ok", "time": 1762560000,
    "stun": ["node.7kas.com:3478", "stun2.example:3478"],
    "relay": [ { "addr": "node.7kas.com:3478", "username": "<…>", "credential": "<…>", "exp": 1762563600 } ] }
   or
← { "t": "error", "code": "unauthorized" }   and close
```

- The node MUST check that `dev_pk` has a current entry for `uid`: in its own directory if responsible; by resolving (§7) if not. A node MAY accept as home devices of any domain; it need not be responsible for their directory.
- `domain` is the domain of the device's address. It is not signed: a node that serves that domain's directory ignores it; any other node needs it to resolve (§7), and a wrong value makes the resolution fail and the session `unauthorized`.
- `host` prevents a signature from being valid on another node. `nonce` prevents replay.
- `push` is optional (§13). A desktop device does not send it.
- `stun` lists STUN servers, at least the node's own. `relay` lists TURN servers the device may use, with credentials that SHOULD expire after 1 hour; the device MAY authenticate again to obtain new ones. Both lists MAY include other nodes that announce `roles.relay`. In the `webrtc` profile they are passed to the WebRTC stack as its ICE servers.
- After `ok`, the node MUST go through the watches whose target is this `uid` (§12) and update the `homes` entry if the device requests it.

**Messages from device to node.**

| Message | Response | Function |
|---|---|---|
| `{ "t": "resolve", "id": "…", "uid": "…", "domain": "…" }` | `{ "t": "record", "id", "record" }` · `{ "t": "error", "id", "code": "not_found" }` · `{ "t": "error", "id", "code": "unavailable" }` | The node runs §7 on behalf of the device with (`uid`, `domain`) |
| `{ "t": "connect", "id": "…", "env": {…} }` | `{ "t": "sent", "id", "devices": ["<dev_pk>", …] }` · `{ "t": "offline", "id" }` · `{ "t": "error", "id", "code": "unknown" }` · `{ "t": "error", "id", "code": "unavailable" }` | Deliver a signaling envelope (§10) to the online devices of the target, through its home when that is another node (§9.4) |
| `{ "t": "watch", "id": "…", "uid": "…", "domain": "…", "ttl": 604800 }` | `{ "t": "ok", "id" }` · `{ "t": "error", "id", "code": "limit" }` · `{ "t": "error", "id", "code": "unavailable" }` | Notify when `uid` is online (§12) |
| `{ "t": "unwatch", "id": "…", "uid": "…" }` | `{ "t": "ok", "id" }` | |
| `{ "t": "subscribe", "id": "…", "uids": ["…"] }` | `{ "t": "ok", "id" }` | Receive `record_changed` for those `uid`s during the session |
| `{ "t": "block", "id": "…", "uid": "…" }` / `unblock` | `{ "t": "ok", "id" }` | The node discards everything coming from `uid` to this user |
| `{ "t": "ping" }` | `{ "t": "pong" }` | Keep the session alive; every 30 s |

**Messages from node to device.**

| Message | Function |
|---|---|
| `{ "t": "deliver", "env": {…} }` | Signaling envelope received |
| `{ "t": "online", "uid": "…" }` | A `uid` this device was waiting for is online |
| `{ "t": "record_changed", "uid": "…" }` | The record of a subscribed `uid` changed; resolve again |

Semantics of `connect`:

- `sent` says which devices the envelope was delivered to (those with a session) or which were sent a push with the envelope held (§13). It does not guarantee they have read it.
- `offline`: no device of the target has a session or a registered push. The envelope is discarded.
- `unknown`: the `uid` does not exist or has blocked the origin. They are indistinguishable by design.
- `unavailable`: the target's node could not be reached or did not answer in time (§9.4). Nothing was delivered; the client keeps the content and retries later. It is not `offline`: a `watch` would not reach that node either.
- A node MUST validate that `env.from.uid` and `env.from.dev` match the sending session and that `env.from.domain` is the domain that session authenticated with; otherwise `error invalid`.

### 9.4 Between nodes

Nodes talk to each other over HTTPS with the same objects. A node calls another one only in four situations: it is the home of a device that sends to a user of the other node (`POST /relay`); it is the home of a watcher (`POST /watch`); it is the home of a target someone is watching (`POST /online`); or both are shard nodes of the same prefix (`GET /sync`). Each node-to-node request carries:

```
Authorization: 7ks-node <nid> <url> <ts> <sig>
sig = base64url( Ed25519_sign( node_pk, JCS({ "method": "POST", "path": "/7ks/v1/relay", "host": "bfc9.7ks.app", "ts": 1762560000, "body_sha256": "<base64url>" }) ) )
```

- `url` is the caller's base URL as a node: the `url` of its node record (§4.4) or an alias of it under the naming convention of its federation (Annex A). The caller uses the one of its names that appears in the `homes` of the user it speaks for.
- `host` is the receiver's name as the caller addressed it (the host of the URL it used). The receiver MUST check that it is one of its own names; a request signed for another node is `401`.
- `ts` is the caller's clock, in seconds. The receiver MUST reject a difference of more than 5 minutes (`401`). Nodes keep their clocks synchronised (NTP).
- The receiver SHOULD remember (`nid`, `ts`, `body_sha256`) for 5 minutes and reject a repetition (`401`).

**Binding `nid` to a node.** Before trusting a signature, the receiver MUST fetch `GET <url>/7ks/v1/node`, check that the record validates, that `nid = sha256(node_pk)` and that the record's `url` is `url` or that `url` is an alias of it under the naming convention. It MAY cache the binding until the record's `exp` and at most 24 hours; when a signature fails against a cached binding, it fetches the record once more before rejecting. Binding proves who is calling, not that the caller may: the content decides.

| Route | The caller is legitimate when `url` … |
|---|---|
| `POST /relay` | is one of the `homes` of the entry `env.from.dev` in the validated record of (`env.from.uid`, `env.from.domain`), resolved per §7 |
| `POST /watch` | equals `watcher_home` and is one of the `homes` of the watcher in the validated record of (`watcher_uid`, `watcher_domain`) |
| `POST /online` | is the node this receiver sent the matching watch to: a node keeps the watches it has forwarded (watcher, target, target's node) until they expire and accepts `online` only for one of them |
| `GET /sync` | is a node that the shard table assigns to that prefix |

A request whose caller is not legitimate is `403`. This is what makes limits per caller node meaningful: a node can only speak for the users it is the home of, and a stranger who runs a node cannot flood another node in the name of invented users.

| Route | Body | Function |
|---|---|---|
| `POST /relay` | `{ "env": {…} }` | Deliver an envelope to a device of this node. Responds like `connect`: `{ "status": "sent" \| "offline" \| "unknown", "devices": [...] }` |
| `POST /watch` | `{ "watch": {…} }` | Register a watch (§4.6) on this node, which is the target's node |
| `POST /online` | `{ "uid": "<target>", "watcher_uid": "<uid>" }` | Tell this node that a target one of its users was waiting for is online |
| `GET /sync/<prefix>?since=<ts>` | — | Entries and revocations of the prefix modified since `since` (only between shard nodes of the same prefix) |

**Forwarding.** A node that receives a `connect` or a `watch` for a `uid` it is not the home of MUST resolve (§7) with the `domain` the device gave, take the `homes` of the target entry (`to.dev`), or of every entry when `to.dev = "*"`, and forward exactly once to the first home that answers, in the entry's order. A received `POST /relay` is never forwarded. Before contacting a home the node MUST check that it is a node: scheme `https`; a host name that resolves to a public unicast address, never loopback, private, link-local, carrier-grade NAT, multicast or reserved ranges (the address rule); and a valid node record at `<home>/7ks/v1/node` whose `url` is that home or of which that home is an alias, cached as above. A home that fails any check is skipped. The node does not start a `/relay` when `env.exp` is less than 5 s away, and answers `unavailable` when no home answered. The answer of the home (`sent`, `offline`, `unknown`) is passed to the device unchanged.

**Timeouts and limits.** Connection 5 s, whole request 10 s; at most 4 homes per entry, one attempt per home per envelope. A node SHOULD keep connections to frequent nodes open, SHOULD stop calling a node that has failed repeatedly for 30 s and answer `unavailable` meanwhile, and SHOULD limit requests per caller node and per `from.uid`.

**What a node needs to talk to other nodes.** Its node key and signed node record served at `/node` (§4.4), with `exp` renewed before it lapses; the parameters of its federation (§6) and the current table or naming convention; a DNS resolver with the caches of §5.1; an HTTPS client that verifies certificates, follows at most one `307` and applies the address rule; the caches of §7 (validated records, at most 1 hour) and of this section (node records and bindings, at most 24 hours); the replay memory; the watches it stores (§4.6) and the watches it has forwarded.

---

## 10. Signaling

Signaling is the only thing that passes through nodes. It carries what the two devices need to find each other and to bind the channel to their keys; nothing else.

### 10.1 Envelope

What the node transports. The node sees target, origin and expiry; the content is opaque.

```json
{
  "id": "<uuid v4>",
  "to":   { "uid": "<uid>", "domain": "example.com", "dev": "<dev_pk> | *" },
  "from": { "uid": "<uid>", "domain": "7kas.com", "dev": "<dev_pk>" },
  "exp": 1762560120,
  "body": "<base64url>"
}
```

- `to.dev = "*"`: the node delivers to every online device of the `uid`. Note: the body is sealed for one specific `dev_dh`, so an envelope with `*` can only be opened by the device it was sealed for; `*` serves to avoid knowing in advance which ones are online, not to send one body to several.
- `exp`: MUST NOT exceed `now + 120 s`. A node that cannot deliver before `exp` discards the envelope. **A node MUST NOT queue envelopes**: if the target is not online at that moment, it answers `offline`. The only retention allowed is the push one (§13).
- `body` ≤ 16 KB.
- `id` lets a receiver discard duplicates for 5 minutes.
- `to.domain` and `from.domain` are the domains of the two addresses, in lower case. A `uid` cannot be reversed and resolution (§7) needs the domain: the sender's home uses `to.domain` to find the target's node; the receiver uses `from.domain` to verify `from_dev` (§10.2); the target's node uses it to check who is calling (§9.4). The sending node checks `from.domain` against the session. The address itself travels only inside the sealed body, in the application protocol, and the receiver checks that its hash is `from.uid` before showing it.

### 10.2 Body

```
inner = JCS({ "signal": <Signal>, "from_dev": "<sender's dev_pk>", "sig": "<sender's Ed25519 signature over JCS(signal)>" })
body  = sealed_box( target's dev_dh, inner )
```

The receiver MUST, in this order:

1. Open the sealed box with its X25519 private key. If it fails, discard.
2. Check that `from_dev` is a current `dev_pk` of `from.uid`; if it is not cached, resolve (§7) with `from.domain`. If it is not, discard.
3. Verify `sig` with `from_dev`. If it fails, discard.

A body that fails validation is never answered. Because the body is signed by `dev_pk` and readable only by the target, everything it contains is bound to the sender's identity. This is what lets both profiles authenticate the channel without certificates: the `noise` profile rests the handshake's static key on it; the `webrtc` profile, the DTLS fingerprint.

### 10.3 Signals

```json
{ "type": "offer", "sid": "<16 random bytes, base64url>",
  "apps": ["7ks-call/1", "sip/2"],
  "transports": ["noise", "webrtc"],
  "noise":  { "ice": { "ufrag": "<4–256 chars>", "pwd": "<22–256 chars>", "candidates": ["<cand>", "…"],
                       "prange": { "start": 61236, "count": 16 } } },
  "webrtc": { "sdp": "<full SDP offer>" } }

{ "type": "answer", "sid": "<the same>", "app": "7ks-call/1", "transport": "noise",
  "noise":  { "ice": { "ufrag": "…", "pwd": "…", "candidates": ["…"], "prange": { "…": "…" } } } }

{ "type": "reject", "sid": "…", "reason": "declined | busy | no_app | no_transport | blocked" }

{ "type": "ice", "sid": "…", "candidate": "<cand>", "mid": "<webrtc only>" }

{ "type": "bye", "sid": "…", "reason": "cancel | answered_elsewhere | failed | hangup" }
```

| Field | Rules |
|---|---|
| `sid` | Generated by the initiator, random, 128 bits. Every signal of a session carries the same `sid`; those with an unknown or finished `sid` are discarded |
| `apps` | Application protocols the initiator proposes, in order of preference; they MUST be in the target entry's `apps`, otherwise the offer is answered with `reject no_app`. The receiver picks one. The receiver's application decides whether to ring, accept silently or reject, according to the `app`; 7KSP only delivers |
| `transports` | Profiles the initiator offers for this session, in order of preference; MUST be a subset of the intersection of both entries' `transports`. For each profile offered, the offer carries its parameters under its name. The answer names the chosen `transport` and carries only its parameters. No common profile → `reject no_transport` |
| `noise.ice.candidates` | ICE candidate strings with the `candidate:` syntax of RFC 8839, without the `a=candidate:` prefix. Example: `1 1 udp 2130706431 192.0.2.4 50000 typ host` |
| `noise.ice.prange` | Optional. Range of public ports the sender predicts for itself when behind a symmetric NAT (§11.2). `count` ≤ 20 |
| `webrtc.sdp` | SDP generated by the WebRTC stack. Its `a=fingerprint` line is the DTLS fingerprint that §11.5 binds to the identity |
| `ice` | Candidates discovered after sending the offer or the answer (trickle). `mid` only in the `webrtc` profile |
| `bye.reason` | `cancel` before an answer; `answered_elsewhere` to the other devices when one answers; `failed` if no channel could be established; `hangup` when ending |

### 10.4 Several devices

An `offer` addressed to a user with several devices is sent **once per device**, each envelope sealed for the corresponding `dev_dh` and with the same `sid`. When the first `answer` arrives, the initiator MUST send `bye answered_elsewhere` to the others and discard their later answers. A receiver that gets `bye` stops ringing.

A device MAY open a session with another device of its own address (same `uid`, different `dev`); the node treats it like any other session and MUST NOT distinguish it.

### 10.5 Session state machine

```
Initiator
  IDLE ──offer sent──► OFFERED ──answer──► CONNECTING ──channel ready──► CONNECTED ──bye/drop──► CLOSED
                         │ reject / 60 s without answer → CLOSED (bye cancel)
                                              │ 30 s without channel → CLOSED (bye failed)
Receiver
  IDLE ──offer received──► RINGING ──user accepts (answer)──► CONNECTING ──channel ready──► CONNECTED ──► CLOSED
                             │ reject / bye / 60 s → IDLE
```

In CONNECTED, signaling through the node is only used if the direct channel is lost and must be renegotiated (§11.8); the closing `bye` MAY go over the direct channel or through the node.

### 10.6 Timers

| Event | Value |
|---|---|
| No `answer` after `offer` | 60 s → `bye cancel` |
| No direct channel after `answer` | 30 s → `bye failed` |
| Resending `ice` candidates | as the ICE agent discovers them (trickle); no periodic resend |
| Memory of envelope `id`s for duplicates | 5 minutes |

---

## 11. Direct channel

### 11.1 What it guarantees

Whatever the profile, once established the channel gives the application:

- Confidentiality and integrity of every packet.
- Mutual authentication at device level: the two ends are exactly the devices whose keys the directory published. Nobody without the private key of a device can take its place.
- Forward secrecy: compromising a device's long-term key later does not expose past sessions.
- No node in the path. A relay, when used, forwards packets it cannot read.
- Two primitives: unreliable datagrams and reliable ordered streams (§11.6). Media tracks in the `webrtc` profile.

### 11.2 NAT traversal

Both devices are behind NAT. Each one creates an outbound mapping in its own NAT and then both use those mappings towards each other. The procedure is symmetric: both sides do the same thing at the same time. It is ICE (RFC 8445) restricted to UDP. In the `noise` profile the client runs ICE itself; in the `webrtc` profile the WebRTC stack runs it, configured with the same STUN and relay servers.

How it works, in four steps:

```
1. A sends a packet to a STUN server from its local UDP socket (say port 50000).
   Its NAT creates the mapping 50000 → 203.0.113.5:61234 and the server tells A "I see you at 203.0.113.5:61234".
   B does the same and gets 198.51.100.9:40011.
2. A and B exchange those addresses through signaling (candidates).
3. A, from the SAME socket, sends probes to 198.51.100.9:40011. B, from the SAME socket, sends probes to 203.0.113.5:61234.
   Both at the same time, every 50–200 ms. The first outbound packet of each one authorizes the other's inbound in its NAT.
4. When a probe gets a reply in both directions, the pair is validated. The server no longer intervenes.
```

A client (or its WebRTC stack) MUST:

- Run **full ICE**, not ICE-lite. Gather host candidates (every local address, IPv6 included), server-reflexive candidates (its public address as seen by each STUN server in `stun`) and relay candidates (allocations on servers in `relay`), in that order of preference.
- Use **one UDP socket** for everything: STUN queries, connectivity checks and the channel itself. The public mapping only stays valid for that socket.
- Send connectivity checks (STUN Binding requests with `ufrag`/`pwd`, message integrity included) to every candidate of the peer, from its own socket, every 50–200 ms, for at most 30 s. **Both sides probe at the same time.**
- Accept **peer-reflexive candidates**: a valid probe arriving from an unannounced address becomes a new candidate.
- In the `noise` profile, detect a **symmetric NAT** on its side by comparing the mappings returned by two STUN servers or two sockets. If the ports differ, announce `prange` with the predicted next ports (usually consecutive) and `count` ≤ 20. A device receiving `prange` MUST send probes to those ports too.
- Keep the chosen pair alive every 15–25 s (an empty keepalive packet in `noise`; the WebRTC stack does it by itself in `webrtc`).
- Never use TCP for the direct channel. If the network blocks UDP, the only path is a relay reached over TCP/TLS on port 443, which still carries UDP packets.

The initiator is the ICE *controlling* agent: if several pairs validate, it chooses which one is used and the receiver accepts the choice.

A node MUST run a STUN server (RFC 8489) and announce it in its record. A node SHOULD run a TURN server (RFC 8656) with per-session quotas and announce `roles.relay`; it SHOULD also accept TURN over TCP and TLS on port 443 for networks that block UDP. A relay only sees encrypted packets and cannot read or alter them. Any standard TURN software works.

Expected outcome: in the large majority of sessions the channel is direct; the relay is needed when both sides are behind symmetric NAT and port prediction fails.

### 11.3 Transport profiles and negotiation

| Profile | Stack | Identity binding | Who implements it |
|---|---|---|---|
| `noise` | ICE → UDP → `Noise_IK` → datagrams and streams (§11.4, Annex C) | the handshake's static key is `dev_dh` from the directory | native clients MUST |
| `webrtc` | ICE → UDP → DTLS → data channels and media tracks (§11.5) | the DTLS fingerprint arrives inside the signed, sealed signal | web clients MUST; native clients SHOULD |

Negotiation: each device announces its profiles in `transports` of its entry. The initiator offers the intersection in its order of preference; the receiver picks one in the `answer`. Two native devices normally end up on `noise`; a session with a browser on either side uses `webrtc`. A native client implementing only `noise` is a complete 7KSP client; it simply cannot talk to browsers.

The application does not see the profile. It opens streams and sends datagrams; the profile materializes them (§11.6).

### 11.4 `noise` profile

#### Handshake

Over the UDP pair chosen by ICE, the two devices run `Noise_IK_25519_ChaChaPoly_BLAKE2s` (Noise Protocol Framework, revision 34) with the prologue `"7ksp/1"` (ASCII bytes, no terminator). The static keys are the devices' `dev_dh`. The initiator already knows the receiver's `dev_dh` from the directory; the receiver learns the initiator's static key when opening the first message and checks it against the directory.

```
→ e, es, s, ss     handshake_init   initiator: ephemeral key; encrypts its static key and a timestamp
← e, ee, se        handshake_resp   receiver: ephemeral key; both derive the session keys
```

Rules:

- The handshake initiator is the session initiator (`offer`).
- The receiver MUST verify that the decrypted static key is a `dev_dh` of the `from.uid` of the offer (the one in the sealed signaling, which the directory confirms). If it is not, it MUST discard the message without answering.
- The initiator MUST verify that the receiver's message decrypts with the keys derived from the `dev_dh` it expected. A wrong receiver cannot produce it.
- The payload of `handshake_init` is a 12-byte TAI64N timestamp. The receiver MUST reject a timestamp earlier than the last one accepted from that static key (replay protection) and SHOULD reject one more than 90 s away from its clock.
- Under load, a receiver MAY ignore `handshake_init` beyond a per-source-address limit; the initiator retransmits every 1 s up to 5 times.
- Rekey: a new handshake over the same session every 120 s, or after 2⁶⁰ packets, or whenever either side wants. The old keys remain valid for 10 s after the new ones are established.

No certificate, name or signature takes part in the handshake. Authentication is Diffie-Hellman against keys both sides already trust.

#### Packet format

Every packet starts with one type byte. The values are chosen so that a single socket can carry STUN (first byte 0–3), TURN channel data (64–79) and 7KSP (193–196) at once, following the demultiplexing rule of RFC 7983.

```
0xC1  handshake_init   type(1) reserved(3) sender_idx(4) e(32) enc_s(48) enc_ts(28)          = 116 bytes
0xC2  handshake_resp   type(1) reserved(3) sender_idx(4) receiver_idx(4) e(32) enc_empty(16)  = 60 bytes
0xC3  data             type(1) reserved(3) receiver_idx(4) counter(8) AEAD(payload)           = 16 + payload + 16
0xC4  keepalive        = data with empty payload                                              = 32 bytes
```

- `reserved`: three zero bytes. A receiver MUST ignore their value.
- `sender_idx` / `receiver_idx`: 4 random bytes per session chosen by each side. A receiver finds the session by `receiver_idx` without decrypting; several sessions coexist on one socket.
- `e`: ephemeral X25519 public key. `enc_s`: encrypted static key (32 + 16-byte tag). `enc_ts`: encrypted timestamp (12 + 16). `enc_empty`: encrypted empty payload (tag only).
- `counter`: 64 bits, strictly increasing per session key, used as the AEAD nonce (12 bytes: 4 zero bytes + the 8 of the counter). The receiver MUST discard repeated or out-of-window counters (window of 2048 backwards).
- `AEAD(payload)`: ChaCha20-Poly1305 with the session key and the nonce; the 16-byte tag follows the ciphertext.
- Maximum payload: 1200 bytes, to avoid IP fragmentation on any path (maximum packet 1232 bytes, below the IPv6 minimum MTU). Larger application data is split by the stream layer.

#### Primitives on the wire

The first byte of a decrypted payload selects the primitive:

```
0x00  datagram   rest = application bytes, delivered as one unit, no reliability or ordering
0x01  stream     rest = a stream frame (Annex C), reliable and ordered per stream
```

### 11.5 `webrtc` profile

The channel is a standard WebRTC `PeerConnection`, as implemented by browsers and native libraries (libdatachannel in C++, pion in Go, webrtc-rs in Rust, react-native-webrtc and flutter-webrtc on mobile).

- The initiator creates the `PeerConnection` with its session's `stun` and `relay` servers as ICE servers, generates an SDP offer and sends it in `offer.webrtc.sdp`. The receiver applies it, generates the answer and sends it in `answer.webrtc.sdp`. Later candidates travel in `ice` signals with `mid`.
- **Identity binding.** The SDP's `a=fingerprint` line arrived inside a body signed by the sender's `dev_pk` and readable only by the target. When DTLS completes, each side MUST compare the fingerprint of the remote certificate with the one in the SDP it received and MUST abort if they differ. The certificate is self-signed and is validated against nothing else; no certificate authority is involved. The guarantee is the same as in `noise`: without the device's private key the signaling cannot be forged, and without the certificate's private key the DTLS handshake cannot be completed.
- A WebRTC stack generates its own self-signed certificate; the protocol imposes nothing on it beyond the fingerprint check. A client SHOULD generate a new certificate per session.
- Audio and video use WebRTC tracks over SRTP, with keys derived from the same DTLS handshake.

### 11.6 Primitives

The application asks for the same things in both profiles:

| Primitive | `noise` | `webrtc` |
|---|---|---|
| Stream (reliable, ordered, bidirectional) | stream of Annex C | data channel with `ordered: true`, no retransmit limit |
| Datagram (unreliable, unordered) | datagram payload | data channel with `ordered: false, maxRetransmits: 0` |
| Media track | not available; the application sends media as datagrams | WebRTC track |

The application chosen in the `answer` owns the session. By convention it opens the first stream as its control stream, and in `webrtc` it labels data channels with the application name (`createDataChannel("control", { protocol: "7ks-chat/1" })`). The framing it uses inside streams and datagrams is its own. 7KSP ends here.

### 11.7 Directly reachable devices

A device with a public UDP address (a server, a desktop with a forwarded port) MAY announce it in `endpoints`. An initiator that supports `noise` MUST try the announced `endpoints` first, sending `handshake_init` directly, before starting ICE; if no `handshake_resp` arrives within 3 s, it continues with ICE as usual. The handshake is the same, so the guarantees are the same. `endpoints` applies to the `noise` profile only.

### 11.8 Reconnection

If the pair stops working (no packets for 3 keepalive intervals, or a network change), either side MAY restart ICE with new candidates under the same `sid` for at most 60 s. In `noise` the session resumes with a new handshake; it MAY also migrate to a new address pair without a handshake if a valid `data` packet arrives from it (the counter proves it is the same peer). In `webrtc` this is an ICE restart of the same `PeerConnection`. After 60 s, a new `sid`.

### 11.9 Browsers

A browser has no UDP sockets and cannot run the `noise` profile. A web client implements only the `webrtc` profile and announces `"transports": ["webrtc"]`. It is a complete 7KSP client: same identity, same directory, same signaling, same guarantees.

### 11.10 Mobile devices

Mobile operating systems close an application's sockets in the background (iOS within seconds; Android under battery saving). This constrains the client, not the protocol:

- The direct channel lives while something is active (a call, a transfer). When nothing is, the client SHOULD close it and not keep sending keepalives.
- Presence is the WebSocket session with the home (§9.3), not the channel between devices. If the system does not allow keeping it, the device remains reachable by push (§13).
- When woken by push, the client opens a session, receives the held envelope and rebuilds the channel from scratch. With ICE and the Noise handshake this takes a few hundred milliseconds.
- A client SHOULD register a `push` in `auth`; a desktop device does not need it.

---

## 12. Deferred delivery

Replaces any message queue. The node stores no content; it stores who is waiting for whom.

```
A1 (uid A)              A's home (N)               B's home (M)               B1 (uid B)
  |-- connect {→ B} ------>|                             |                          |
  |                        |-- POST /relay ------------->|  B has no session        |
  |                        |<-- {status: offline} -------|                          |
  |<-- offline ------------|                             |                          |
  | keeps the content in its local store                 |                          |
  |-- watch {B, ttl} ----->|-- POST /watch {A, N, B} --->| stores Watch(A→B)        |
  |<-- ok -----------------|                             |                          |
  …                                                                                 |
  |                        |                             |<-- auth (session) -------|
  |                        |<-- POST /online {B, A} -----| deletes Watch(A→B)       |
  |<-- online {B} ---------|  (or push "online" if A has no session)                |
  |-- connect {→ B} ------>|-- POST /relay ------------->|-- deliver -------------->|
  …  direct channel A1 ↔ B1; A1 delivers what it kept
```

Rules:

1. When `connect` answers `offline`, the client keeps the content in its own store and sends `watch` with `domain` and `ttl` ≤ 7 days. Its home forwards it to the target's home (resolving) as `POST /watch`, with `watcher_domain` and with `watcher_home` set to the one of its own names that appears in the watcher's `homes`, and keeps a copy of what it forwarded in order to accept the `online` (§9.4).
2. When a device of the target opens a session, its home goes through the watches with that `target_uid` and, for each one: if the watcher is connected to this same node, it sends it `online`; otherwise, `POST /online` to `watcher_home`, which delivers it over the session or turns it into a silent push. The watch is deleted when emitted.
3. The watcher, on receiving `online`, does `connect` and delivers over the direct channel.
4. If the watcher did not receive the notice (it was off and the push did not wake it), it retries on startup.
5. A client MUST retry everything pending on startup and when connectivity is recovered, and SHOULD renew pending watches before they expire.
6. A node MUST accept at most 256 active watches per `watcher_uid`; a repeated one (same watcher/target pair) renews the existing one instead of adding another.
7. `online` reveals nothing the watcher would not discover with a `connect`; even so, a user MAY block another and the node will discard their watches and envelopes.

---

## 13. Push

A sleeping mobile device has no open session. To reach it, the device registers in `auth` a `push` with the URL of a gateway and an opaque token. The gateway is chosen by the device, not by the node. The node stores `{dev_pk → gateway, token}` while the device's entry is current.

When a node needs to wake a registered device without a session:

```
POST <gateway>/7ks/v1/push
Content-Type: application/json
{ "token": "<opaque>", "kind": "connect" | "online" }
→ 204, or 410 if the token is no longer valid (the node forgets it)
```

- `connect`: high priority (VoIP push on iOS, high priority on FCM). Triggered by an envelope with an `offer` addressed to that device. The node MUST hold the envelope until its `exp` (120 s at most) while waiting for the device to open a session, and deliver it with `deliver` as soon as it does. This is the only envelope retention allowed.
- `online`: low priority, silent; may not be delivered. §12.4 covers it.

The gateway receives no `uid`, `sid` or content; only a token and one word. Anyone MAY operate a gateway for their own build of the client, because push credentials (FCM/APNs) are tied to the published app. UnifiedPush is a valid transport on Android without anybody's gateway: in that case `push.gateway` is the device's UnifiedPush endpoint and the body is the same.

---

## 14. Errors

HTTP responses use standard codes and a body `{ "error": "<code>", "detail": "<free text>" }`. Session messages use `{ "t": "error", "id": "…", "code": "<code>" }`.

| code | HTTP | Meaning |
|---|---|---|
| `invalid` | 400 | Malformed object, missing mandatory field or invalid signature |
| `expired` | 410 | Expired proof, entry, token or envelope |
| `not_found` | 404 | The `uid` does not exist on this node |
| `not_responsible` | 403 | The node is not responsible for that `uid`; it may say who is |
| `seq` | 409 | `seq` is not greater than the stored one |
| `unknown` | — | Unknown or blocked target (session only) |
| `offline` | — | No device of the target has a session or push (session only) |
| `limit` | 429 | Quota exceeded |
| `unauthorized` | 401 | Invalid session or node authentication |
| `unavailable` | 503 | The node responsible for the target could not be reached or did not answer in time; retry later |

A `429` SHOULD carry the `Retry-After` header (RFC 9110 §10.2.3) with the seconds to wait. `detail` MAY name the cause; the values with a defined meaning are `challenge` and `pow` (§8.9) and `cancelled` (§8.8).

---

## 15. Limits

| Parameter | Value |
|---|---|
| Envelope: maximum `exp` / size | 120 s / 16 KB |
| Device entry: maximum `exp` | 30 days; renew at 20 |
| Devices per `uid` | 32 |
| `homes` / `endpoints` per entry | 4 / 4 |
| Verification token | 15 minutes |
| Verification requests | 5 per address per hour |
| Watches per watcher / `ttl` | 256 / 7 days |
| Relay credentials | 1 hour |
| Relay per session | operator's quota; recommended 1 Mbit/s and 10 minutes |
| WebSocket session ping | 30 s; the node closes after 90 s without traffic |
| Shard table: `exp` | 7 days |
| Direct `endpoints` attempt | 3 s |
| ICE probes: interval / total | 50–200 ms / 30 s |
| `prange.count` | ≤ 20 |
| Channel keepalive | 15–25 s; dead after 3 missed |
| Handshake retransmission (`noise`) | 1 s, 5 attempts |
| Rekey (`noise`) | 120 s or 2⁶⁰ packets |
| Packet payload (`noise`) | ≤ 1200 bytes |
| Envelope duplicate memory | 5 minutes |
| Node-to-node request: connection / total | 5 s / 10 s |
| Node-to-node: homes tried per entry / circuit breaker | 4 / 30 s |
| Node caches: validated records / node records and bindings | 1 hour / 24 hours |
| Node-to-node replay memory | 5 minutes |
| Number of a verification request | 3 decimal digits |
| Proof-of-work challenge: validity / difficulty | 120 s / 18 recommended, 24 maximum |
| Proof of work: minimum time from challenge to solution | 1 s |

---

## 16. Security

**Identity model.** Controlling the email is controlling the account. This is a deliberate decision: it is what allows logging in from any device with just the email. The consequences are mitigated in the client, not in the protocol: a warning to the user and their contacts when a new device appears, and the ability to revoke it from any other.

**No certificate authority between devices.** Both profiles authenticate with keys the directory already vouched for: directly in `noise`, through a fingerprint carried in signed signaling in `webrtc`. The only signatures in the system are on directory objects and on signaling, both verified once and cached. There is no authority to trust, to compromise or to pay.

**What a node sees.** The origin and target `uid` of every envelope passing through it, the watches, and which devices have an open session. When two devices of one address talk to each other (§10.4), the node sees envelopes with the same `uid` on both ends, and nothing else. It sees no signaling or application content, nor can it alter them. A relay sees encrypted packets and their addresses, nothing more. Whoever does not want a third party to see even that runs a domain node.

**What a malicious node can do.** Deny service to its users, falsely answer `offline` or `unknown`, hold envelopes or refuse to relay. It cannot read, modify or impersonate, because everything is signed or encrypted with keys that never leave the devices.

**Address impersonation.** In a domain with a DNS entry it requires the domain's private key. In the public directory it requires fooling a verifier accepted by the federation. Client-side pinning turns a change of `dom_pk` into an alert and a new device into a notice.

**Handshake abuse.** `handshake_init` costs the receiver two Diffie-Hellman operations. A receiver under load MAY rate-limit by source address; since the initiator retransmits, legitimate sessions survive. The timestamp prevents replaying a captured `handshake_init`. DTLS has its own cookie mechanism.

**Enumeration.** The `uid` is an unsalted hash, so whoever has a list of email addresses can check which ones are in the public directory. This is accepted: the directory returns no emails, only keys, and domains with their own node do not appear in it.

**Mass sign-up and mail flooding.** The verifier is the only part of the protocol that acts on behalf of a stranger, by sending mail. The proof of work (§8.9) makes every request cost the requester, the limits per address and per network (§8.3) bound what remains, and no third-party captcha is involved, so no request data leaves the verifier.

**A link opened by the wrong person.** Opening a link signs nobody in (§8.8): a link scanner or a curious recipient changes nothing, and a user who sees a number that their device does not show cancels the request.

**Open forwarding.** A node forwards only to URLs that resolution produced and that prove to be nodes (§9.4), never to a URL a client or an entry names on its own, and it accepts forwarded requests only from nodes that are homes of the users they speak for. Without the first rule a node would be a proxy towards arbitrary hosts, including its own private network; without the second, anyone able to run a node could flood any other node in the name of invented users, and `online` notices would become a push-spam channel.

**Connection spam.** An `offer` only reaches a device with an open session, and the application decides whether to bother the user. `block` cuts off the rest. Nodes SHOULD rate-limit `connect` per origin.

**Network metadata.** ICE candidates reveal IP addresses to the other device (not to the node, since they travel sealed). A user who does not want to reveal their IP to a contact MAY configure the client to offer relay candidates only.

---

## 17. Registered identifiers

| Element | Value |
|---|---|
| DNS record | `_7ks.<domain>` TXT |
| Well-known | `/.well-known/7ks` |
| API prefix | `/7ks/v1/` |
| WebSocket subprotocol | `7ksp.v1` |
| STUN / relay port | 3478 (standard); relay also on 443 TCP/TLS |
| Transport profiles | `noise`, `webrtc` |
| `noise` packet types | 0xC1–0xC4 (first byte) |
| Noise prologue | `7ksp/1` |
| URI scheme | `7ks:` — `7ks:alice@example.com`, `7ks:alice@example.com?app=7ks-call/1`, `7ks:verify?token=…` |
| Application names | `name/version`, lowercase, no spaces; `7ks-` reserved for the 7ks client's applications |
| 7ks client applications | `7ks-chat/1`, `7ks-file/1`, `7ks-call/1`, `7ks-sync/1` (own-device sync, 7ks document); `7ks-archive/1` is the 7ks export container, not a channel application |

---

## Annex A. 7ks public federation (informative)

| Field | Value |
|---|---|
| `fed_id` | `7ks` |
| `fed_pk` | to be published |
| Bootstrap URL | to be published |
| Initial verifiers | to be published |

The operator of the public federation publishes, outside this document: admission criteria for shard nodes, neutrality commitment, replica restoration period and `fed_pk` rotation procedure.

**Naming convention.** The public federation publishes its shard nodes as host names: the prefix `p` of the public directory (`uid[0..2]`) is served at `https://<p as 4 lowercase hex digits>.7ks.app`, from `0000.7ks.app` to `ffff.7ks.app`. A client or node of this federation MAY derive the node of a prefix from the name without downloading the table; the table remains the authority and is published at the bootstrap URL. All the names of one node are aliases of its node record (`url`), and a record obtained at any of them is accepted for the others with the same `nid`. Every device of an address lives at the name of its prefix: the `homes` of public entries, `watcher_home` and the `host` of sessions name it. Moving a prefix to another node is a change of that name's DNS record after replication (§6, `GET /sync`); meanwhile the previous node answers `307` on reads and `403 not_responsible` on writes for that prefix (§9.1, §9.2).

## Annex B. Complete examples

### B.1 Call within the same domain

A is `garacil@7kas.com`, device A1 (native, `noise` and `webrtc`), connected to the domain node N (`https://node.7kas.com`). B is `soporte@7kas.com`, devices B1 (native) and B2 (browser, `webrtc` only), same node.

```
A1 → N   ws  {t:"resolve", id:"1", uid:<B>, domain:"7kas.com"}
N  → A1  ws  {t:"record", id:"1", record:{devices:[B1, B2]}}        // A1 validates domainsig and pins keys
A1       opens its UDP socket, queries STUN, gathers ICE candidates; also prepares a PeerConnection for B2
A1 → N   ws  {t:"connect", id:"2", env:{to:{uid:<B>, dev:B1}, from:{uid:<A>, dev:A1}, exp,
                                         body:sealed(B1, offer{sid, apps:["7ks-call/1"], transports:["noise","webrtc"], noise:{ice}, webrtc:{sdp}})}}
A1 → N   ws  {t:"connect", id:"3", env:{to:{uid:<B>, dev:B2}, …, body:sealed(B2, offer{sid, transports:["webrtc"], webrtc:{sdp}})}}
N  → B1  ws  {t:"deliver", env}          // B1 has a session
N  → B2  ws  {t:"deliver", env}          // B2 has a session in the browser
B1, B2   open the body, verify A1's signature; the 7ks-call/1 application rings on both
B1 → N   ws  {t:"connect", env:{to:{uid:<A>, dev:A1}, body:sealed(A1, answer{sid, app:"7ks-call/1", transport:"noise", noise:{ice}})}}
N  → A1  ws  {t:"deliver", env}
A1 → N   ws  {t:"connect", env:{to:{uid:<B>, dev:B2}, body:sealed(B2, bye{sid, reason:"answered_elsewhere"})}}   // B2 stops ringing
A1 ↔ B1  connectivity probes from both sides at once; a pair validates; A1 picks it
A1 → B1  handshake_init  (Noise_IK, static key = dev_dh_A1)
B1       checks that the static key is a dev_dh of A; replies
B1 → A1  handshake_resp
A1 ↔ B1  audio as datagrams; call signaling over stream 1. N no longer intervenes.
```

Had B2 answered first, the session would use `webrtc`: A1 applies B2's SDP answer, DTLS completes, both compare the remote fingerprint with the one received in signed signaling, and audio goes over a WebRTC track.

### B.2 Chat between different domains

A is `garacil@7kas.com` on N. C is `cliente@otra.com`; `otra.com` has no node of its own and C1 is homed at the public node P.

```
A1 → N   ws  {t:"resolve", id:"1", uid:<C>, domain:"otra.com"}
N        DNS _7ks.otra.com → none; prefix = uid_C[0..2]; GET at a shard node → record with attestation
N  → A1  ws  {t:"record", id:"1", record:{devices:[C1 {homes:["https://p.example"]}]}}   // A1 validates the attestation against verifiers
A1 → N   ws  {t:"connect", id:"2", env:{to:{uid:<C>, domain:"otra.com", dev:C1}, from:{uid:<A>, domain:"7kas.com", dev:A1}, exp, body:sealed(C1, offer{apps:["7ks-chat/1"], …})}}
N        C is not its own; resolves; POST https://p.example/7ks/v1/relay {env} with Authorization 7ks-node
P  → C1  ws  {t:"deliver", env}
C1 → P   ws  {t:"connect", env:{to:{uid:<A>, dev:A1}, body:sealed(A1, answer{…})}}
P        A is not its own; resolves 7kas.com → N; POST https://node.7kas.com/7ks/v1/relay {env}
N  → A1  ws  {t:"deliver", env}
A1 ↔ C1  ICE, handshake, stream 1 with the chat messages.
```

### B.3 Message to someone who is switched off

Continuing B.2, with C1 switched off:

```
A1 → N   connect → N: POST /relay at P → P: C1 has no session but a registered push → push "connect" → no answer within 120 s → P discards the envelope
N  → A1  sent {devices:[C1]} … and A1 gets no answer within 60 s → A1 sends bye cancel and keeps the message
A1 → N   watch {C, ttl:604800} → N: POST /watch at P → P stores Watch(A→C)
(two days later) C1 starts → auth at P → P: POST /online at N {uid:C, watcher_uid:A} → N → A1 online {C}
A1 → N   connect → … → direct channel → A1 delivers the message it kept.
```

## Annex C. Reliable streams (`noise` profile)

Stream frames travel inside `data` packets whose payload starts with `0x01`. All integers are big-endian.

```
frame_type(1) stream_id(4) …

0x01 OPEN    stream_id(4) window(4)                          opens a stream; window = bytes the sender may receive
0x02 DATA    stream_id(4) offset(8) len(2) bytes             data at byte offset
0x03 ACK     stream_id(4) cum(8) window(4) n(1) [from(8) to(8)]×n
                                                             everything below cum received; n extra received ranges
0x04 FIN     stream_id(4) final_offset(8)                    no more data after final_offset
0x05 RESET   stream_id(4) code(4)                            abort, discard everything
```

Rules:

- `stream_id`s are even when opened by the Noise initiator and odd when opened by the receiver; 0 is reserved. An OPEN with an id already in use is a protocol error: RESET with `code` 1.
- Recommended initial `window`: 65536 bytes. A sender MUST NOT have more than `window` unacknowledged bytes in flight per stream. The receiver updates `window` in every ACK.
- A receiver sends an ACK at most every 2 DATA frames or every 25 ms, whichever comes first, and immediately on detecting a gap.
- Retransmission: a DATA range not acknowledged within the retransmission timeout (RTO) is resent. RTO starts at 300 ms and is estimated from measured round trips as in RFC 6298; minimum 100 ms, maximum 10 s. A range that three consecutive ACKs report as missing is resent at once.
- Several frames MAY be packed into one packet as long as the payload does not exceed 1200 bytes.
- A stream is closed when both sides have sent FIN and all data up to `final_offset` is acknowledged, or on RESET.
- Flow control is per stream; the session's congestion control SHOULD be an additive-increase, multiplicative-decrease window over all streams. Implementations MAY use any standard algorithm.

This is deliberately the minimum needed for chat and file transfer; it is not a general-purpose transport and does not try to compete with one.

## Annex D. Conformance checklist

What an implementation MUST satisfy, per role, to call itself 7KSP/1. Each item refers to the section that defines it.

**Every client**
- Computes `uid` exactly as in §3.1 and reproduces the vectors of §3.2.
- Generates `dev_pk` and `dev_dh` on the device and never exports them (§3).
- Validates device entries per §4.2 and discards those that fail (§4.3).
- Resolves per §7, DNS before public directory, never falling back to the public directory when the domain is authoritative.
- Pins keys by TOFU and warns of new devices and `dom_pk` changes (§7.1).
- Enrolls by link (§8) and renews before expiry (§8.4).
- Opens a session with a signed challenge (§9.3) and keeps it alive with `ping`.
- Seals and signs signaling (§10.2) and validates received signaling in the stated order.
- Sends one `offer` per device and `bye answered_elsewhere` to the rest (§10.4).
- Respects the timers of §10.6.
- Runs full ICE, over one socket, with simultaneous probes and peer-reflexive candidates (§11.2).
- Verifies the identity of the other end when establishing the channel (§11.4 or §11.5) and aborts on mismatch.
- Retries pending work on startup and on network recovery, and uses watches (§12).
- Sends `domain` with `auth`, `resolve` and `watch` and in both ends of every envelope (§9.3, §10.1), and never the address itself outside the sealed body.
- Solves the verifier's proof of work when asked (§8.9) and shows the number of the request until the proof arrives (§8.8).

**Native client, additionally**: full `noise` profile (§11.4, Annex C), `endpoints` if any (§11.7), mobile behaviour (§11.10).
**Web client, additionally**: `webrtc` profile (§11.5), announcing only that profile.

**Every node**
- Exposes the read routes without authentication and the write routes with full validation (§9.1, §9.2).
- Authenticates sessions by challenge and checks the device entry (§9.3).
- Never queues envelopes except the push retention (§10.1, §13).
- Delivers `connect` only to devices with a session; answers `offline` and `unknown` per §9.3.
- Forwards to other nodes exactly once, only to verified nodes on public addresses, authenticates node-to-node requests, accepts them only from the homes of the users they speak for, and answers `unavailable` when the other node cannot be reached (§9.4).
- Stores and processes watches per §12.
- Runs STUN (§11.2).
- Applies the limits of §15.

**Domain node, additionally**: issues `domainsig` (§4.1, §8.7), is a verifier or delegates to the domain's procedure (§8).
**Shard node, additionally**: keeps the signed table, replicates its prefixes, accepts writes only for its prefixes and rejects domains with DNS (§5.2, §6).
**Verifier**: complies with §8.3 and §8.8, and with §8.9 when it requires work.
**Relay**: standard TURN with quotas and, where possible, TCP/TLS on 443 (§11.2).
**Push gateway**: accepts `POST /7ks/v1/push` with no data beyond token and kind (§13).

---

## History

- **0.6 (draft, unreleased)** — Contact between nodes completed: `domain` beside every `uid` a node has to resolve (`auth`, `resolve`, `watch`, the two ends of the envelope, the watch object); node-to-node authentication binds the caller to the users it speaks for and the request to the receiver (`url` in `Authorization`, `host` in the signed object, replay memory); forwarding only to verified nodes on public addresses; `homes` checked on write; `unavailable` error; node caches, timeouts and circuit breaker; naming convention of the public federation (Annex A); number match on the login link: `lang` and `code`, a confirmation page that changes nothing on `GET` and accepts or cancels on `POST`, the `cancelled` state (§8.8); optional proof of work on the verification request, with test vectors (§8.9); `Retry-After` on `429` (§14). Own devices: §8.1 step 8 (import from another device or an export), §10.4 (a device may open a session with another device of its own address), §16 (what the node sees of it), §17 (`7ks-sync/1`, `7ks-archive/1`).
- **0.5** — Rewrite for maximum detail and clarity: glossary, conformance levels per role, the journey of a connection, validation and resolution algorithms in pseudocode, sealed box specified, session state machine, several devices, exact semantics of `connect`, renewal and account recovery, step-by-step NAT explanation, mobile section, cross-domain and deferred-delivery examples, conformance checklist. No wire-format changes relative to 0.4 except: `bye` carries `reason`; `revoke` carries `type`; WebSocket subprotocol `7ksp.v1`; AEAD nonce defined explicitly.
- **0.4** — Two transport profiles under one protocol: `noise` (native) and `webrtc` (browsers and natives wanting to reach them), negotiated per session with `transports` in device entries and in `offer`/`answer`; identical guarantees, both without certificate authorities; application primitives mapped to each profile; web clients back in scope as `webrtc`-only clients.
- **0.3** — Direct channel redefined: UDP only; `Noise_IK` handshake with the devices' `dev_dh` keys instead of TLS/DTLS; no certificates or SDP; full ICE requirements; packet format and primitives; reliable streams in Annex C; `endpoints`; `stun` and `roles.relay` in node records.
- **0.2** — Full definition: JSON with JCS signatures, HTTPS/WebSocket node API, enrollment by email link, per-device proofs, watch-based deferred delivery.
- **0.1** — Initial outline.
