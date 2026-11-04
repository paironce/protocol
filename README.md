# PairOnce protocol

PairOnce — browser-to-browser file transfer between your devices. No app, no account, files never stored on a server.

[PairOnce](https://paironce.com) is a free web app for sending files, photos, text and links between your own devices
(iPhone, Android, Windows, Mac, Linux) with no app to install and no account. The two browsers connect directly and
end-to-end encrypted (AES-256-GCM, keys from P-256 ECDH, a 13-digit safety code to compare); when a network blocks a direct
connection, an encrypted relay passes on data it cannot read. Devices on the same Wi-Fi find each other on their own, and a
large file resumes after a dropped connection. PairOnce keeps no copy. Overview: [github.com/paironce](https://github.com/paironce)
· guides: [paironce.com/guides](https://paironce.com/guides/).

How [PairOnce](https://paironce.com) pairs two devices and moves files between their browsers, and what the services in
between can and cannot see.

- [protocol.md](protocol.md): devices, invitations, pairing, the safety code, key derivation, frames, sending a file
- [threat-model.md](threat-model.md): what is protected, from whom, and where the protection stops
- [agent-as-a-device.md](agent-as-a-device.md): AI agents (Claude Code, Cursor, CI jobs) as paired devices: what works
  today with the [paironce CLI](https://github.com/paironce/cli) and its MCP server, and what is proposed next

Version 2.27.0 describes paironce.com 2.27.0. The reference implementation is [@paironce/core](https://github.com/paironce/core).
Questions and corrections are welcome as issues. Security problems: see [SECURITY](https://github.com/paironce/security/blob/main/SECURITY.md).

## License

The text is licensed under [CC BY 4.0](LICENSE): share and adapt it with attribution. The PairOnce™ name and logo are not
covered: see [TRADEMARK.md](TRADEMARK.md).
