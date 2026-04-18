# GroupSum

**Compute the sum of secret numbers — privately, in real time, with no server.**

GroupSum lets a group of people add up their secret numbers without revealing any individual value — not even to the host. Built on cryptographic secret sharing, it runs entirely in the browser with no backend.

➡️ **[Try it live](https://bluemle.github.io/groupsum/)**

---

## How it works

Each participant enters a secret number. The app splits it into random fragments using **additive secret sharing** — fragments that individually reveal nothing. Fragments are exchanged between participants, column sums are computed locally, and the host aggregates them into the true total. No raw number ever leaves your device.

```
Your number  →  split into N random fragments
                 ↓ each fragment sent to one participant
Each participant sums their received fragments → sends column sum to host
Host adds all column sums → true total  ✓
```

---

## Features

- 🔒 **Private** — individual values are never transmitted or stored
- 🌐 **Serverless** — peer-to-peer via WebRTC; no backend required
- 👥 **Multi-participant** — supports any group size (≥ 2 to lock)
- 🔗 **Invite links** — share a URL, others join instantly
- 🔄 **Multiple rounds** — run new rounds with the same group
- 👤 **Host failover** — if the host disconnects, a guest is automatically promoted
- 📱 **Responsive** — works on desktop and mobile
- 🌙 **Dark/light theme** — follows system preference

---

## Usage

### Create a room
1. Enter your name (optional)
2. Click **Create Room**
3. Share the Room ID or invite link with participants

### Join a room
1. Enter your name (optional) and the 6-character Room ID
2. Click **Join**
3. Wait for the host to start the round

### Run a round
1. Host clicks **Lock & Start** once everyone has joined
2. Each participant enters their secret number and clicks **Submit**
3. The sum (and average) appears automatically when all have submitted

### After the result
- **New Round** — same participants, enter new numbers
- **Edit Participants** — unlock the room to add/remove people, then lock again

---

## Privacy model

- Shares are generated using `crypto.getRandomValues()` — cryptographically random
- The host sees only column sums, not individual shares
- No data touches any server (PeerJS is used only for WebRTC signalling/handshake)
- Everything runs in a single static HTML file — inspect it yourself

---

## Host failover

If the host loses connection, guests automatically elect a new host:
- Each guest waits a random jitter (0–1.5s), then races to claim the next host peer ID
- The winner becomes the new host; others reconnect as guests
- If the original host comes back and finds someone else in charge, they quietly rejoin as a guest

---

## Technical summary

| Concern | Approach |
|---|---|
| Secret sharing | 2-round additive integer secret sharing (`SCALE = 100`) |
| P2P transport | PeerJS WebRTC data channels (star topology, host as relay) |
| Random shares | `crypto.getRandomValues()` |
| Room IDs | 6-char, alphabet `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` |
| Host ID | `ROOM_ID` (gen 0), `ROOM_ID_N` (gen N after handover) |
| Deployment | Single `index.html` — no build step, no dependencies beyond PeerJS CDN |

For the full technical specification including message protocol, state machine, and all edge cases, see [`SPEC.md`](./SPEC.md).

---

## Self-hosting

Just serve `index.html` from any static host (GitHub Pages, Netlify, your own server). The PeerJS signalling server (`peerjs.92k.de`) is self-hosted and separate from your deployment.

---

## License

MIT
