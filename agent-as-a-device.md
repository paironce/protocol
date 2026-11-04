# Agent as a Device

Design note, draft 1 (2026-11-04). Licensed CC BY 4.0. Status of each part is marked **Shipped** or **Proposed**.

An AI agent that runs in a terminal or a cloud sandbox (Claude Code, Cursor, Codex, a CI job) can write files, run
builds and read logs, but it cannot reach the person's phone: a finished build does not land on the phone, a photo of a
whiteboard does not land in the agent's working folder. Today people bridge that gap with cloud drives, chat apps or
email, each of which keeps a copy on someone else's server and none of which asks the person before each transfer.

PairOnce already connects two devices with end-to-end encryption, a safety code and an approval on the receiving side.
This note describes how an agent joins that model as one more device, without accounts and without long-lived tokens.

## 1. Principle

An agent is a device, not a user. It has its own device key, pairs with a person's device the same way two browsers do,
and follows the same rules: a safety code compared on both screens at pairing, and the receiving side accepting every
file. Nothing about the protocol changes for the person's device except what it shows.

## 2. What works today (Shipped)

- **The `paironce` CLI** ([npm](https://www.npmjs.com/package/paironce), [source](https://github.com/paironce/cli),
  Apache-2.0) runs the same connection code as paironce.com. `paironce pair`, `send`, `text`, `receive`, `devices`,
  `forget`. Published from GitHub by trusted publishing, with npm provenance.
- **An MCP server**, `paironce mcp`, with five tools:

  | Tool | What it does |
  | --- | --- |
  | `pair_device` | The person opens paironce.com on their device and reads out the six-digit code; the agent joins with it. The person compares the safety code and approves the agent on their own device. |
  | `send_file`, `send_text` | Send to a paired device. The person accepts each file there. |
  | `receive_files` | Wait while the person sends files from their device; they are saved in the agent's `./inbox`. |
  | `list_devices` | The devices this agent is paired with. |

- **A name the person recognises.** `PAIRONCE_NAME` (or `--name`) sets what the person's device shows, for example
  "Claude Code · Agent". The device key stays the same when the name changes.
- **Connections from sandboxes.** The agent connects directly when the network allows it, otherwise through the TURN
  relay (UDP, TCP or TLS on 443), and where even that is blocked, through the PairOnce tunnel: the same encrypted frames
  over a WebSocket to paironce.com (protocol section 7.1). The CLI uses a web proxy from `HTTPS_PROXY`. Checked on
  2026-11-04 from a Claude Code on the web sandbox that allows only HTTPS through a proxy, with `paironce.com` on its
  allow-list: pairing, a file and a message reached a browser through the tunnel.

## 3. Identity and lifecycle

**Shipped.** The agent's identity is an ECDH P-256 key pair kept in `~/.paironce` (or `$PAIRONCE_HOME`), readable only
by its user; the private key never leaves that machine. A sandbox that is thrown away takes its identity with it if
`PAIRONCE_HOME` points into the sandbox. The person removes the agent from their device at any time, and the agent can
`forget` the person's device.

**Proposed (protocol 2.3).** Two optional hello fields, sent only by an agent:

- `deviceKind: "agent"`. The person's device shows an "Agent" mark next to the name wherever devices are listed and in
  every approval.
- `forgetAfter: <seconds>` (at most 30 days). The person's device removes the saved agent once it has not reconnected for
  that long, so a sandbox that is gone does not stay in the list. `paironce mcp --ephemeral` would set it to one day and
  keep the identity in a temporary folder.

Both fields only ever make a pairing more limited or more visible. An agent that leaves them out or lies about them gains
nothing: it is then shown and kept like any other device, which is today's behaviour. Older pages drop the fields.

## 4. Capabilities

**Shipped:** `pair_device`, `send_file`, `send_text`, `receive_files`, `list_devices` (section 2).

**Proposed:**

- `request_file`, `request_photo`, `request_scan`: the agent asks for something specific ("a photo of the label on the
  back of the router"). The person's device shows the request with the agent's name and its stated purpose; the person
  answers by taking the photo, scanning pages with the existing scanner or choosing a file, or declines. A request is
  never answered automatically.
- `watch_frames`: still pictures from the person's live camera or screen, at most a few per second, taken with the
  existing full-resolution still and only while the person keeps the live picture on. Stills rather than a video stream,
  so the agent side needs no video decoder.

## 5. Consent model

- Pairing: the safety code is compared on both screens and the person approves the agent on their own device. The agent
  cannot approve itself.
- Every file sent to the person's device is accepted there, file by file. A per-device "accept automatically from this
  agent" switch is off by default.
- Every `request_*` needs the person's action on their device, every time; there is no switch to turn that off.
- The person's device keeps a short history of what the agent sent and asked for, on that device only.

## 6. Transport

The agent uses the same three ways as a browser, in the same order: direct, TURN relay, PairOnce tunnel. Frames are
encrypted with the same keys and formats as between two browsers (`@paironce/core`); there is no agent-specific branch in
the protocol. A sandbox usually has no local network, so most agent connections run through the relay or the tunnel.

## 7. Allowance

**Shipped.** Relayed and tunnelled bytes count against each side's monthly relay allowance (200 MB per device, 5 GB with
PairOnce Pro; a network has its own monthly cap), measured by the service for the tunnel. A connection where either device
has Pro counts as Pro.

**Proposed.** A fresh identity for every sandbox must not mean a fresh free allowance every time. Bytes already count
against both devices of a connection, so an agent cannot use more of a person's device's allowance than that device has.
In addition, a device that declares `deviceKind: "agent"` gets the allowance of the person's device it is connected to
rather than its own, so throwaway identities add nothing. Agent-to-agent connections between two fresh identities stay
within the network cap.

## 8. Threat model additions

| Threat | What stops it |
| --- | --- |
| A hijacked or prompt-injected agent tries to send files out, or to put files on the person's device | Every file is accepted on the receiving device; the agent reaches only devices the person paired; requests always need the person. |
| Someone pairs a device that pretends to be the person's agent | The safety code is compared on both screens; the pairing code comes from the person's own device; the "Agent" mark is shown on the person's side. |
| A sandbox is reused by someone else and finds the old identity | `PAIRONCE_HOME` inside the sandbox, `--ephemeral`, `forgetAfter`, and removal from the person's device at any time. |
| Flooding the person with requests | Requests are rate-limited per paired agent and can be switched off per agent on the person's device. |
| The service reads what passes through the tunnel | It cannot: frames are encrypted end to end with keys derived on the two devices (protocol sections 5 and 7.1). |

## 9. Non-goals

- No background daemon that keeps a connection open between tasks.
- No automatic acceptance by default, and none ever for `request_*`.
- No forwarding between agents without a person's approval.
- No storing of an agent's credentials or of files on PairOnce's servers.

## 10. Open questions

- The allowance of a device that declares `deviceKind: "agent"` when its person's device is offline.
- The approval interface when several agents are paired at once.
- The still rate and resolution for `watch_frames`.
- Whether an agent may open a distribution room (today rooms are same-network only and have no relay).

## 11. Roadmap

| Step | Content | Status |
| --- | --- | --- |
| v0 | CLI: pair, send, text, receive | Shipped (CLI 0.1, 2026-11-04) |
| v1 | MCP: send tools, `pair_device`, `receive_files`, device names; relay over TCP/TLS; the PairOnce tunnel and proxies | Shipped (CLI 0.4, paironce.com 2.22.0) |
| v2 | `deviceKind`, `forgetAfter`, `--ephemeral`, the agent allowance rule | Proposed |
| v3 | `request_file`, `request_photo`, `request_scan` | Proposed |
| v4 | `watch_frames` | Proposed |

Comments and corrections are welcome as issues on [paironce/protocol](https://github.com/paironce/protocol).
