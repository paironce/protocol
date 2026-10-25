# PairOnce threat model 2.11.0 (2026-10-23)

What PairOnce protects, from whom, and where the protection stops. Read with the protocol description (`protocol.md`).

## Assets

1. **What is sent:** file contents, file names and types, text and links.
2. **Device keys:** each browser's P-256 private key, which stands for "this device" in every later connection.
3. **Who talks to whom, and when:** IP addresses, timing, sizes.

## Parties

| Party | Can do | PairOnce's position |
| --- | --- | --- |
| Network observer (Wi-Fi, ISP) | read and change packets | sees only TLS (signaling) and DTLS (WebRTC). |
| Signaling service, honest | relays messages, rate limits | sees what section 8 of the protocol lists. Holds no key that opens content. |
| Signaling service, **malicious** | changes any signaling message: SDP, ICE, hellos, invitations behind six-digit codes | see "Active attack by the service" below. |
| TURN relay | forwards DTLS packets | sees IP addresses and byte counts only. |
| Someone who learns a six-digit code or an invitation link | tries to join first | the host must approve, and the joining device's safety code must match. Codes work once, last about ten minutes, and lookups are rate limited. |
| The other paired device | receives what you send | trusted for what you choose to send it; it must accept each file first. |
| Malicious web page or extension in the same browser | — | out of scope (see Limits). |

## Active attack by the service

The strongest adversary is the signaling service itself (or someone who controls it), because it sits between the two
browsers when they first meet.

- **Link or QR pairing.** The host's public key and the invitation nonce travel in the URL fragment, which browsers never
  send to a server. The joining device therefore knows the real host key before signaling starts. A service that
  substitutes keys in the hello is caught: the session secret and the possession proofs are bound to the keys from the
  link.
- **Six-digit code pairing.** Here the service hands the joining device the host's key and the nonce, so it *could* give
  each side a key of its own and sit in the middle. The 13-digit safety code is what stops this: it comes from ECDH of the
  keys each side actually holds, so in a man-in-the-middle the two screens show different codes. **This protection
  depends on the person comparing the codes.** The codes carry about 43 bits; a mismatch the person notices ends the
  attack, and the attacker cannot search for a matching pair offline because the code depends on ECDH secrets it does not
  hold for the honest keys.
- **Later connections.** Each side pins the other's public key from the first pairing and refuses a different one, so the
  service cannot step into a reconnection.
- **WebRTC transport.** The service could also swap the DTLS fingerprints in SDP and terminate both DTLS sessions. That
  gives it the DataChannel's frames, but every file chunk, text and control message inside is encrypted under frame keys
  derived from device ECDH (protocol 5.2), which it does not have. Until 2026-10-23 the `file-meta` frame repeated the
  file name and type outside that encryption; since then a receiver asks, inside the encrypted channel, for them to be left
  out (`privateMeta`), so this needs both devices on a current version. File sizes remain visible in this position.

## What a passive service operator learns

IP addresses of both devices, when they connect, which rooms they use, device names and IDs in hellos, the size of
transfers through the relay, and how often a device uses the relay (to apply its monthly allowance). Not file contents,
names or text. Daily aggregate counts kept for operations contain no IP address or device ID.

## Limits (not protected)

- **The endpoints.** Malware, a malicious browser extension, or someone with access to an unlocked device sees whatever
  that browser sees. Received files sit in the page's memory or the browser's private storage until saved.
- **A person who skips the safety code** with six-digit pairing (see above).
- **Traffic analysis.** Sizes and timing are visible to the network and the relay.
- **The web app itself.** The page is served by PairOnce each time; a compromised server could serve a malicious page.
  This is true of every web app. Mitigations: the core library is open and versioned, each production release names the
  core version it uses in a signed record, and the page uses a strict content security policy. A future goal is
  reproducible builds that can be checked against the published core.
- **Denial of service.** The service can refuse to relay; the relay has a monthly budget.

## Reporting

See `SECURITY.md`.
