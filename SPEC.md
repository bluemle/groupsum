# GroupSum — Complete Specification

> A single-file, serverless web app that lets a group of people compute the **sum (and average) of their secret numbers** without revealing individual values to anyone — not even the host.

---

## 1. Core Concept

GroupSum uses **two-round additive secret sharing** over a peer-to-peer mesh. Each participant's number is split into random integer fragments that individually reveal nothing. Only the combined column sums reveal the total.

### Protocol (SCALE = 100 for 2 decimal places)

```
Round 1 — Share distribution
  Each participant i splits value_i into N integer shares:
    share[i][0] + share[i][1] + ... + share[i][N-1] = round(value_i × 100)
  Shares for others are sent via the host relay.
  Own share is kept locally.

Round 2 — Column aggregation
  Each participant j computes:
    colSum_j = Σ_i  share[i][j]     (one share from every participant)
  colSum_j is sent to the host.

Host computes:
  globalScaled = Σ_j colSum_j
  result       = round(globalScaled) / 100   =   Σ_i value_i  ✓
```

**Privacy guarantees:**
- Individual values are never transmitted.
- Shares are random integers — a single share reveals nothing.
- The host only receives column sums, not raw shares.

**Constraints:**
- `SCALE = 100` → up to 2 decimal places.
- Value range: −1,000,000,000 to +1,000,000,000.
- Max scaled value fits safely within `Number.MAX_SAFE_INTEGER` (9e15).

---

## 2. Architecture

- **Single HTML file** — no server, no build step, no dependencies except PeerJS CDN.
- **Star topology** — all peers connect to a designated host peer via PeerJS WebRTC data channels.
- **Host relay** — the host forwards round-1 shares between guests; guests never connect directly to each other.
- **PeerJS server** — `peerjs.92k.de:443` (self-hosted). Only used for WebRTC signalling; no data passes through it after connection.

```
Guest A ──┐
Guest B ──┤──► Host (relay + aggregator)
Guest C ──┘
```

---

## 3. Room & Peer ID Scheme

- **Room ID**: 6 characters from alphabet `ABCDEFGHJKLMNPQRSTUVWXYZ23456789` (no ambiguous chars: 0, 1, I, O).
- **Host peer ID generation 0**: `ROOM_ID` (e.g. `AB3X7K`)
- **Host peer ID generation N**: `ROOM_ID_N` (e.g. `AB3X7K_2`)
- Generations increment on each host handover. Guests probe gen 0→20 to find the active host.
- **Valid room ID regex**: `/^[ABCDEFGHJKLMNPQRSTUVWXYZ23456789]{6}$/`

---

## 4. App State (`S` object)

```js
const S = {
  peer: null,            // PeerJS Peer instance
  myId: null,            // own PeerJS peer ID
  myName: '',            // display name
  isHost: false,
  roomId: null,          // 6-char room ID

  locked: false,         // true after host presses "Lock & Start"
  lockedIds: [],         // FROZEN sorted participant list — set once at lock time

  guestMap: {},          // host only: peerId → { conn }
  members: {},           // display: peerId → { name, submitted }
  hostConn: null,        // guest only: DataConnection to host

  sharesIn: {},          // round 1: { senderId: intShare }   — deduped by key
  colSumsIn: {},         // round 2: { senderId: intColSum }  — deduped (host only)
  resultShown: false,

  heartbeatTimer: null,  // host: setInterval sending pings every 4s
  hostWatchdog: null,    // guest: setTimeout detecting host gone (7s)
  promotionStarted: false,
  guestHeartbeat: null,  // guest: setInterval sending heartbeats to host (3s)
  lastSeen: {},          // host: peerId → timestamp of last heartbeat

  hostPeerId: null,      // actual PeerJS ID of current host
  hostGen: 0,            // current host generation
};
```

---

## 5. Message Protocol

All messages are plain JS objects sent over PeerJS DataConnections.

### Guest → Host

| Type | Fields | Description |
|---|---|---|
| `hello` | `name` | Join / reconnect request |
| `share_for` | `to`, `share` | Route round-1 share to another participant |
| `col_sum` | `sum` | Submit column sum (round 2) |
| `i_submitted` | — | Notify host that this participant submitted |
| `reset` | — | Request new round (host-initiated only) |
| `heartbeat` | — | Keep-alive to host |

### Host → Guest

| Type | Fields | Description |
|---|---|---|
| `welcome` | `myId`, `members`, `locked`, `lockedIds`, `hostGen` | Sent on join/reconnect |
| `joined` | `id`, `name` | Another participant joined |
| `left` | `id` | Participant left |
| `lock` | `lockedIds`, `names` | Room locked, round starts |
| `your_share` | `from`, `share` | Forwarded round-1 share |
| `member_submitted` | `id` | Another participant submitted |
| `result` | `sum` | Final sum |
| `reset` | — | New round |
| `unlock` | — | Host editing participants |
| `rejected` | — | Room locked, join denied |
| `ping` | `gen` | Heartbeat from host (resets guest watchdog) |

---

## 6. Room Lifecycle

### 6.1 Creating a Room (Host)
1. Host enters name (optional, defaults to `"Host"`), clicks **Create Room**.
2. `startHost(name, existingRid?)` creates a PeerJS peer with the room ID as its peer ID.
3. On `peer.on('open')`: sets up heartbeat interval (`startHeartbeat`), shows invite link, renders member list.

### 6.2 Joining a Room (Guest)
1. Guest enters name (optional, defaults to `"Guest"`) and Room ID, clicks **Join**.
2. `startGuest(name, rid)` validates Room ID format. If invalid → error shown immediately, no network call.
3. Creates a PeerJS peer with a random ID, then calls `probeAndJoin(rid, name, peer, gen=0)`.
4. `probeAndJoin` tries to connect to `hostPeerIdForGen(rid, gen)`:
   - **Success**: sends `hello`, sets up guest heartbeat (3s) and host watchdog (7s timeout).
   - **2s timeout or error**: tries `gen+1`. After gen 20 fails: if Room ID is valid format → **creates the room** (`startHost(name, rid)`); else → shows "Room not found".

### 6.3 Hash-based Invite Links
- URL format: `https://example.com/#ROOMID`
- On load: if hash present, pre-fills Room ID, dims the "Create" card, focuses name field.

### 6.4 Locking the Room
1. Host clicks **Lock & Start** (requires ≥ 2 participants).
2. `lockedIds` = sorted `Object.keys(members)` — **frozen**.
3. Broadcasts `lock` message to all guests.
4. Host and guests reset their number input UI and show the number entry card.
5. Status badge changes to `🔒 Locked · N participants` (amber).

### 6.5 Submitting a Number
1. `submitNumber()` validates range, splits value into shares via `makeShares()`.
2. Sends `share_for` to host for each other participant.
3. Calls `r1_receive(myId, ownShare)` locally.
4. Sends `i_submitted` to host for UI updates.

### 6.6 Round 1 — Share Receipt (`r1_receive`)
- Deduplicates by sender ID.
- When `sharesIn.length >= lockedIds.length` → compute `colSum`, send `col_sum` to host.

### 6.7 Round 2 — Host Aggregation
- Host deduplicates `col_sum` by sender.
- When all `lockedIds.length` column sums received → `checkSumComplete()` → broadcast `result`.

### 6.8 Result Display
- Animated count-up over 700ms.
- Shows sum and average.
- Status badge → `✓ Sum ready` (green).

### 6.9 New Round
- Host clicks **New Round** → `doReset(true)` → clears `sharesIn`, `colSumsIn`, re-enables input, keeps same `lockedIds`.

### 6.10 Edit Participants
- Host clicks **Edit Participants** → `unlockForEditing()` → broadcasts `unlock`, clears lock state, shows host card again.
- Guests see "Host is editing participants" banner. Their number card is hidden.

---

## 7. Participant Departure Handling

### While unlocked
- `conn.on('close')` → removes from `members`, broadcasts `left`, re-renders.

### While locked, participant **already submitted**
- Remove from `lockedIds`, `members`, `colSumsIn`.
- Broadcast `left`.
- Call `checkSumComplete()` — may complete the round with fewer participants.

### While locked, participant **not yet submitted**
- Their share fragments may already be distributed to others.
- **Math is corrupt** — including partial shares produces wrong sum.
- Trigger `doReset(true)` after 800ms with banner: `"A participant left — please re-enter your numbers"`.

### On guest side (`left` message)
- Remove from `members`, `lockedIds`, `sharesIn`.
- Re-check if `sharesIn` is now complete enough to send `col_sum`.

---

## 8. Host Election & Failover

### 8.1 Guest detects host gone
- Host watchdog fires after 7s without a `ping`.
- `attemptHostPromotion()` — random jitter (0–1500ms) before each guest tries to claim the next gen peer ID.
- Winner: `promoteToHost(gen)` succeeds, becomes new host.
- Loser: gets `unavailable-id` error → `reconnectToNewHost(gen)` → re-joins as guest.

### 8.2 Promoted host setup (`promoteToHost`)
- Destroys old peer, creates new Peer with `ROOM_ID_N`.
- Resets all state, rebuilds member list, calls `startHeartbeat(gen)`.
- Shows "You became the host" banner. Role label: `"Host (promoted)"`.

### 8.3 Original host reconnects after takeover
- When all guest connections close, `onGuestClose` schedules a 5s probe.
- After 5s: if `guestMap` still empty → `probeNewHostGen(roomId, hostGen+1, myName)`.
- `probeNewHostGen` creates a throwaway peer, tries to connect to gen+1:
  - **Connection opens**: new host exists → reset all host UI/state → `startGuest()` (demoted).
  - **2.5s timeout**: room is empty → stay host.
- Also triggered by `peer.on('disconnected')` → `reconnect()` → probe after 2s.

---

## 9. Reaper (Silent Disconnect Detection)

- Host sends `ping` every 4s. Guests reply with `heartbeat` every 3s.
- Host records `lastSeen[peerId]` on each heartbeat.
- `runReaper()` (called every 4s inside heartbeat interval):
  - Evicts any guest not seen for > 8s.
  - Applies same departure logic as `onGuestClose`.

---

## 10. UI Structure

### Screens
- **`s-welcome`**: Create room (name input) / Join room (name + room ID inputs). Hash auto-fills room ID.
- **`s-room`**: Main room screen, shared by host and guest with conditional visibility.

### Room Screen Elements

| Element ID | Visible to | Purpose |
|---|---|---|
| `conn-badge` | All | Connection status (green/amber/red) |
| `role-lbl` | All | "Host" / "Guest" / "Host (promoted)" |
| `rid-display` | All | 6-char Room ID |
| `btn-copy` | All | Copy room ID |
| `share-link-box` | Host | Invite link + copy button |
| `host-card` | Host (unlocked) | Lock button + participant count |
| `plist` | All | Participant list with submitted status |
| `num-card` | All (locked) | Number input + submit button |
| `result-wrap` | All | Animated sum + average display |
| `btn-reset` | Host | Start new round |
| `btn-edit-participants` | Host | Unlock and edit participants |
| `guest-waiting-nr` | Guest | "Waiting for host" message after result |
| `phase-banner` | All | Context banner (info/ok/warn) |

### Status Badge States

| State | Text | Colour |
|---|---|---|
| Room created | `Room open · waiting for guests` | 🟢 green |
| Room locked | `🔒 Locked · N participants` | 🟡 amber |
| New round | `🔒 Locked · N participants` | 🟡 amber |
| Sum computed | `✓ Sum ready` | 🟢 green |
| Disconnected | `Reconnecting…` | grey |
| Error | `Host disconnected…` etc. | 🔴 red |

---

## 11. Key Helper Functions

```js
genId()                          // generate random 6-char room ID
isValidRoomId(rid)               // /^[ABCDEFGHJKLMNPQRSTUVWXYZ23456789]{6}$/
hostPeerIdForGen(roomId, gen)    // gen=0 → roomId, gen>0 → roomId_gen
makeShares(value, lockedIds, myId)  // split value into N integer shares
checkSumComplete()               // if all colSums received → broadcast result
runReaper()                      // evict silent guests, trigger departure logic
onGuestClose(pid)                // unified conn.on('close') handler
startHeartbeat(gen)              // start host ping+reaper interval
claimOrFail(rid, name, peer)     // claim empty room or show not-found
showHostActions()                // show/hide host-only buttons
doReset(andBcast)                // new round, keep participants
unlockForEditing()               // host editing mode
probeNewHostGen(roomId, gen, name)  // probe if new host exists, demote if found
```

---

## 12. Tech Stack

| Concern | Solution |
|---|---|
| P2P signalling + data channels | [PeerJS](https://peerjs.com/) v1.x (`peerjs.92k.de`) |
| Cryptographically random shares | `crypto.getRandomValues()` |
| Styling | Hand-written CSS variables, dark/light theme via `data-theme` on `<html>` |
| Fonts | Inter + Space Grotesk (Google Fonts) |
| Build | None — single `index.html`, no bundler |
| Hosting | GitHub Pages (repo: `bluemle/groupsum`) |

---

## 13. Edge Cases & Invariants

1. **`lockedIds` is frozen** at lock time — never mutated except for participant departure handling.
2. **Deduplication** — `sharesIn` and `colSumsIn` are keyed objects; duplicate messages are silently dropped.
3. **Messages from non-locked peers** are dropped after lock.
4. **Reconnecting locked participant** — matched by name; stale entry evicted, new peer ID takes over.
5. **Unlocked joins** — guests with identical names (e.g. default "Guest") do not overwrite each other.
5. **Race: two guests claim same empty room** — loser gets `unavailable-id` → rejoins as guest.
6. **Corrupt math on departure** — if unsubmitted guest leaves mid-round, full reset is triggered.
7. **Host gen scope** — promoted host uses `S.myName` (not closed-over `name`) in async callbacks.
