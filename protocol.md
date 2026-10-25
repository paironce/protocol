# PairOnce protocol 2.13.0 (2026-10-23)

PairOnce moves files and text between two browsers over a WebRTC DataChannel. A small signaling service helps the two
browsers find each other; when a direct path fails, a TURN relay forwards their traffic. Neither service can read what is
sent: every file chunk, text and control message is encrypted by the two browsers with keys the services never see.

This document describes what goes over the wire. It is written so that someone can check the privacy claims, not so
that someone can run a PairOnce service. Licensed CC BY 4.0. The reference implementation of everything here is the
`@paironce/core` package (Apache-2.0).

This version describes paironce.com 2.13.0 and `@paironce/core` 2.13.0, in production since 2026-10-23. Corrections are
welcome as issues or pull requests.

## 1. Devices

Each browser makes one **device identity** the first time PairOnce opens:

- an ECDH key pair on P-256, generated with Web Crypto as non-extractable and kept in IndexedDB;
- a **device ID**, 16–128 characters of `[A-Za-z0-9_-]`, random;
- a **device name**, 1–80 characters, shown to the other side (the person can rename it).

The **device descriptor** `{ id, name, publicKey }` is public: `publicKey` is the raw uncompressed P-256 point (65 bytes,
first byte 0x04) in base64url. The private key never leaves the browser.

A browser remembers the descriptors of devices it has paired with. That list, and nothing on a server, is what makes a
later connection skip the code.

## 2. Rooms and invitations

A pairing happens in a **room**: a random ID (20–128 characters, base64url alphabet) that the signaling service uses to
relay messages between two WebSockets. The room ID is not secret.

The host creates an **invitation**:

| Field | Size | Secret? | Purpose |
| --- | --- | --- | --- |
| `room` | 20–128 chars | no | where to meet |
| `nonce` (the invitation secret) | 32 random bytes | see below | HKDF salt for every key in this pairing |
| host device descriptor | — | no | who to expect |

There are two ways to hand it over:

1. **Link or QR code.** `https://paironce.com/?room=<room>#v=2&nonce=<b64url>&deviceId=…&deviceName=…&devicePublicKey=…`.
   Everything after `#` is the URL fragment: browsers do not send it to any server. The joining device learns the host's
   public key from the link itself.
2. **Six-digit code.** The host page publishes the invitation (room, nonce, host descriptor, expiry) to the signaling
   service, which returns a six-digit code valid once, for about ten minutes. The joining device sends the code and gets
   the invitation back. Lookups are rate limited per network. Here the service *does* see the nonce and the host's
   public key; section 8 explains why that is still safe when the safety code is compared.

In protocol v2 the nonce is a rendezvous value, not a key: it salts the key derivations below, and the actual secrets come
from ECDH between the two device keys.

## 3. Signaling messages

Over the room's WebSocket, each side sends JSON messages (`v: 1`):

- `hello` — `role` (`host` | `join`), a fresh 16-byte `sessionNonce` (base64url, 22 chars), the device descriptor, and
  optional feature flags: `capabilities` (a list, bound into the frame key, see 5.2), `safetyCode: 2`,
  `frameProtocol: 2`, `binaryFileChunks: 1`, `largeFiles: 1`, `hugeFiles: 1`, `receiveWindow: 1`, and `pro: 1` when the
  device has PairOnce Pro. A flag is used only when both hellos carry it (`pro` only needs one side), so an older peer
  keeps working without it.
- `offer` / `answer` — the WebRTC session description, with the sender's `sessionNonce`.
- `ice` — ICE candidates.

The signaling service relays these between the two sockets of a room. It does not keep them.

## 4. Pairing (first connection)

Once the DataChannel is open and both frame keys exist (5.2), the two devices run three encrypted control messages:

1. `pair-claim` (join → host): `claimId` (random), the code the joining device used (the six-digit code, or the
   13-digit safety code), its descriptor, and a **possession proof**.
2. `pair-result` (host → join): `accepted`, and if accepted the host descriptor and the host's possession proof. A refusal
   may carry a short reason and `terminal: true`.
3. `pair-confirm` (join → host): the joining device's proof for the final direction.

**Possession proof.** Each proof is an HMAC-SHA-256 under a key derived as

```
shared = ECDH(own private key, peer public key)             # 32 bytes
key    = HKDF-SHA-256(ikm = shared, salt = invitation nonce,
                      info = "PairOnce device possession proof v1")   → HMAC-SHA-256 key
proof  = HMAC(key, JSON["paironce-device-proof-v1", direction, room, claimId,
                        host.id, host.publicKey, join.id, join.publicKey, code])
```

with `direction` one of `join-claim`, `host-approve`, `join-confirm`. A proof can only be made by the holder of the private
key behind the public key it claims, and only for this room, claim and pair of keys.

**Approval.** The host person sees the request and approves or declines it. Nothing is paired until they approve.

**Safety code.** Both screens show a 13-digit code:

```
shared = ECDH(own private key, peer public key)
bits   = HKDF-SHA-256(ikm = shared, salt = invitation nonce,
                      info = JSON["paironce-pairing-sas-v2", room, sortedPublicKeys...])   → 64 bits
code   = (bits as big-endian uint64) mod 10^13, zero-padded to 13 digits
```

If anyone between the two browsers replaced a public key, the two ECDH results differ and so do the codes. Device IDs and
names are left out on purpose: they are chosen by the other side and would let an attacker search for a colliding code
with cheap hashing alone.

## 5. The encrypted channel

### 5.1 Device session secret

For every connection (first pairing or a later one) both sides derive the same 32-byte secret:

```
shared  = ECDH(own private key, peer public key)
secret  = HKDF-SHA-256(ikm = shared, salt = invitation nonce,
                       info = JSON["paironce-device-session-v2", room, [[id, publicKey] for both devices, sorted by id]])
```

### 5.2 Frame keys

From the secret and the two hello nonces, one AES-256-GCM key per sender:

```
info(sender) = "PairOnce DataChannel key v1" || hostNonce || joinNonce
               || UTF-8(JSON["paironce-frame-v2", sender, base64url(capability context) or null])
key(sender)  = HKDF-SHA-256(ikm = secret, salt = "PairOnce DataChannel salt v1", info = info(sender)) → AES-GCM-256
```

`sender` is `host` or `join`; separate keys stop a frame from being reflected back to its sender. The capability context
is both sides' capability lists in role order, so a list changed in transit gives the two ends different keys and the
channel fails closed. Fresh nonces per connection mean a frame recorded from one connection cannot be replayed into
another.

### 5.3 Frames

Every frame is encrypted with AES-GCM, a 128-bit tag and a 12-byte IV = an 8-byte random prefix (one per message or per
file) followed by a 32-bit big-endian chunk index. The additional data binds the frame to its place:

```
AAD = JSON[2, transferId, kind ("text" | "file"), chunkIndex, totalChunks, plainSize, sender, frameType]
```

so a ciphertext moved to another transfer, index, direction or frame type fails to decrypt. A receiver also refuses a
frame it has already accepted (replay).

JSON frames (`v: 1`): `text`; the control frames `pair-claim`, `pair-result`, `pair-confirm`, `file-offer`,
`file-decision`, `file-cancel`, `file-receipt`, `file-progress`, `file-preview` (each `{ transferId, noncePrefix,
ciphertext, plainSize }`); and the file frames `file-meta`, `file-chunk`, `file-complete`, `file-ack`.

**Binary file chunks** (when both hellos carry `binaryFileChunks: 1`), big-endian:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | `0x50` |
| 1 | 1 | format version, 1 |
| 2 | 1 | frame type, 1 = file chunk |
| 3 | 1 | transfer ID length n (16–128) |
| 4 | n | transfer ID (ASCII) |
| 4+n | 4 | chunk index |
| 8+n | rest | AES-GCM ciphertext and tag |

## 6. Sending a file

1. **Offer.** The sender sends an encrypted `file-offer` with the file's name, type and size (and optionally an encrypted
   `file-preview`, a small JPEG). Nothing is transferred until the receiving person accepts.
2. **Decision.** The receiver answers an encrypted `file-decision`: accept, reject, or busy. An offer not answered in 30
   seconds lapses. An accept carries `privateMeta: true` when the receiver takes the name and type from the offer.
3. **Metadata.** The sender sends `file-meta`: size, chunk size (16–64 KiB, 32 KiB today), chunk count, the file's
   nonce prefix and, when continuing, `resumeFrom`. If the decision carried `privateMeta`, the name is `"-"` and the type
   empty; the receiver uses the ones from the offer it accepted. (An older receiver does not ask, and gets the name and type
   repeated here as before.) The receiver checks the rest against the accepted offer and cancels on any mismatch. Because
   `privateMeta` travels inside the encrypted channel, nobody on the way can strip it to force names into the clear.
4. **Chunks.** Encrypted chunks, in order. With `receiveWindow: 1` the receiver reports with `file-progress` how far it has
   written, and the sender stays within a window ahead of that, so a slow disk cannot fill the receiver's memory.
5. **End.** `file-complete`; the receiver checks every chunk arrived and confirms with `file-ack` / `file-receipt`.

**Resume.** If the connection drops, the receiver keeps what arrived (for 30 minutes, at most three files) and, on
reconnecting with the same device, asks for the rest; the sender's `file-meta` then carries `resumeFrom`.

**Limits.** Up to 2 GB per file when the receiving browser writes to disk (Origin Private File System), and up to 10 GB
(`MAX_FILE_BYTES`) when the receiver also announces `hugeFiles` and either device announces `pro`; else 50 MiB per file
and 100 MiB per page. Text up to 128 KiB per message.

## 7. Later connections

Paired devices reconnect without a code. They meet in a room again; each side already holds the other's public key from
the first pairing and refuses a hello whose descriptor does not match it. The session secret (5.1) comes from ECDH of the
two remembered keys, so the signaling service cannot step in.

## 8. What the services see

| Service | Sees | Does not see |
| --- | --- | --- |
| Signaling (Cloudflare Workers, Durable Objects) | room IDs, hello messages (device descriptors, nonces, flags), SDP and ICE candidates (so IP addresses), timing; for six-digit codes also the invitation nonce and host descriptor | the device private keys; any frame key; file contents, names, text |
| TURN relay (Cloudflare Realtime TURN), only when a direct path fails | the two IP addresses, how many bytes pass, when | anything inside the DTLS-encrypted DataChannel |

## 9. Versioning

`v: 1` is the signaling and frame envelope version. Pairing is `safetyCode: 2` and the live channel is
`frameProtocol: 2`; a peer without them is asked to refresh. New behaviour is added as a separate optional hello flag,
used only when both sides send it.

## 10. Change log

- 2026-10-23: `privateMeta` — `file-meta` no longer repeats the file name and type when the receiver asks in its encrypted
  decision.
