# Family Game — Driver & Navigator

A two-device, first-person 3D co-op driving game inspired by **Left! Left! LEFT!**.

The desktop game runs on itch.io (or any static host) and renders a Three.js cockpit, a procedural
1 km mountain road above a chasm, a first-person camera, and the complete lobby/HUD. The smartphone
controller runs on GitHub Pages and sends steering/brake input over PeerJS WebRTC. There is no game
server: only the public PeerJS signaling service is used to introduce the two browsers.

## Files

| File | Host | Purpose |
| --- | --- | --- |
| `index.html` | itch.io | Three.js game, first-person cockpit, track generator, physics, QR lobby, PeerJS host |
| `controller.html` | GitHub Pages | Mobile lobby, driver controls, live navigator map, retry, PeerJS client |
| `README.md` | — | Deployment and PeerJS linking instructions |

Both HTML files are deliberately self-contained. All CSS and JavaScript are inline, so a deploy is a
single-file upload per host. Three.js, PeerJS, and QRCode.js are loaded from public CDNs.

## Features

- **First-person driver POV** from inside the car, with dashboard, steering wheel, gauge cluster,
  windshield, A-pillars, hood, headlights, and camera shake.
- **Dynamic camera:** subtle speed vibration, acceleration/braking pitch, body roll, and
  look-into-the-turn yaw. FOV expands slightly near top speed.
- **Exactly 1,000 m procedural track** generated on every page load. Turns are 45°, 90°, and 180°
  and their directions strictly alternate (right → left → right → left). Candidate layouts are
  validated for self-intersection with a 16 m centerline clearance.
- **Raised road above a chasm**, with cliff walls, support pillars, guard posts, turn signs,
  a start gate, a checkered start/finish line, and a finish gantry.
- **Solo mode:** no dashboard tablet. Use the keyboard:
  - `A` / `←` = turn left
  - `D` / `→` = turn right
  - `S` / `↓` = hold brake
- **Co-op mode, maximum 2 phones:**
  - First phone = Player 1 / host / room master.
  - P1 chooses Driver or Navigator on the phone.
  - Player 2 automatically receives the remaining role.
  - P1 gets the **Mulai Game** (Start Game) button.
  - In co-op, a 3D tablet appears on the passenger side of the 3D dashboard, mirroring the live
    navigation map: white road lines on a dark blue field.
- **Driver controller:** large left/right steering buttons and a hold-to-brake button.
- **Navigator controller:** 2D minimalist route map, live car position, distance to the next turn,
  turn direction/angle, progress, speed, and timer.
- **5-second 3D countdown**, then automatic smooth acceleration from 0 to a 200 km/h cap.
- **Failure rule:** leaving the road drops the car into the chasm. If the car cannot move for five
  consecutive seconds—or is falling—it catches fire and explodes. Both desktop and P1 phone receive
  the **Ulangi** (Retry) screen.
- **Retry** resets the car to the start line and restarts the 5-second countdown on the same route.
- **Win** when the 1 km finish is reached; result is mirrored to both phones.

## Quick local test

You can test both files from one folder. No build step or npm install is needed.

```bash
# from the folder containing index.html and controller.html
python -m http.server 8080
```

Open `http://localhost:8080/index.html` on the desktop. Because the page is on `localhost`, the game
automatically points its QR code at the sibling `controller.html`. Click **Start Solo** to test
driving. To test co-op without a phone, open the displayed controller URL in a second browser tab.

For a **real phone on the same Wi-Fi**, open the game on the desktop using the notebook's LAN address
instead of `localhost`, for example `http://192.168.1.20:8080/index.html`. The game auto-detects an
IPv4 host and puts the same reachable origin in the QR code. The phone and notebook must be on the
same network, and PeerJS itself still needs internet access. If the phone cannot reach that address,
use the **Atur URL controller** field in the desktop lobby and paste a reachable URL (HTTPS is best).
The URL is stored in `localStorage`. Some browsers restrict WebRTC on plain HTTP LAN origins; for
serious phone testing, use HTTPS (for example the GitHub Pages deployment below).

## Production deployment

### 1. Put the controller on GitHub Pages

1. Create a GitHub repository, for example `family-game-driver-and-navigator`.
2. Add `controller.html` at the repository root.
3. Commit and push.
4. In the repository: **Settings → Pages → Build and deployment**.
5. Choose **Deploy from a branch**, select `main` (or `master`) and `/ (root)`, then save.
6. Wait for the Pages URL, then verify it. It will look like:

   ```text
   https://YOURUSERNAME.github.io/family-game-driver-and-navigator/controller.html
   ```

The controller URL is intentionally not tied to your itch.io page. Every room code is appended as a
query parameter, so one deployed controller serves every session:

```text
https://YOURUSERNAME.github.io/family-game-driver-and-navigator/controller.html?room=ABCDE
```

### 2. Point the game at the controller URL

Open `index.html` and edit the top-level config:

```js
const CONFIG = Object.freeze({
  CONTROLLER_URL: 'https://YOURUSERNAME.github.io/family-game-driver-and-navigator/controller.html',
  ...
});
```

You do **not** need to hard-code it. You can also paste the URL into the lobby's
**Atur URL controller GitHub Pages** field. That value is saved in `localStorage` and the QR code is
regenerated immediately. The QR button/link then contains the current room code automatically.

For a URL without direct file editing, launch the game as:

```text
https://your-game-host/index.html?controller=https://YOURUSERNAME.github.io/family-game-driver-and-navigator/controller.html
```

The URL can also be set manually by the player; the QR updates. The order of precedence is:

1. `?controller=` query parameter
2. saved URL in `localStorage`
3. automatic sibling `controller.html` when running on localhost / LAN / file
4. `CONFIG.CONTROLLER_URL`

### 3. Upload the game to itch.io

1. Zip only `index.html` (and optionally `README.md`).
2. Create a new itch.io project, set **Kind of project** to **HTML**.
3. Upload the zip and mark `index.html` as the entry point.
4. Set the embed size to at least **1280 × 720** (or **960 × 600**). The menu was tested at both sizes.
5. Make sure external scripts are allowed (the project uses public jsDelivr CDNs).
6. Save and test the QR code with a real phone.

> The game and controller must both be served over HTTPS for the most reliable mobile WebRTC
> experience. itch.io and GitHub Pages both provide HTTPS.

## How the PeerJS link works, step by step

The two files use the same public PeerJS broker only to establish a direct WebRTC data channel.
Gameplay inputs, telemetry, and the room map go peer-to-peer.

### Step 1 — Both files share the same peer-ID prefix

`index.html`:

```js
PEER_PREFIX: 'fgdn-'
```

`controller.html`:

```js
const PEER_PREFIX = 'fgdn-';
```

Keep these strings identical. The host peer ID is the prefix plus the room code, for example
`fgdn-ABCDE`.

### Step 2 — The desktop creates the room and the QR code

On every load, the desktop generates a fresh 5-character room code from an unambiguous alphabet and
creates a PeerJS peer:

```js
this.roomCode = makeRoomCode(5);
this.peer = new Peer(CONFIG.PEER_PREFIX + this.roomCode);
```

Once PeerJS opens, the lobby renders the QR/link using:

```text
<controllerBaseUrl>?room=<roomCode>
```

Refreshing the desktop page destroys the PeerJS object, generates a new room code, shows a new QR,
and disconnects any old phone. The route seed is regenerated at the same time.

Clicking a **Start Solo** game does not require a peer, so the game still works if PeerJS is blocked.

### Step 3 — The phone joins the host

`controller.html` reads `room` from the URL or the manual room input, then creates its own anonymous
peer and dials the deterministic host ID:

```js
const conn = peer.connect('fgdn-' + roomCode, {
  reliable: true,
  serialization: 'json'
});
```

The host listens for `peer.on('connection')` and accepts at most two phones. Additional phones get a
`rejected` message. A phone is accepted only while the host is still in the lobby.

### Step 4 — Handshake / slot assignment

The phone sends its first packet:

```js
conn.send({ type: 'hello', version: 1 });
```

The desktop assigns the first accepted phone to slot **p1** and the second to **p2**, then replies:

```js
{
  type: 'welcome',
  slot: 'p1',           // or 'p2'
  isHost: true,         // true only for p1
  role: 'driver',       // or 'navigator' or null
  phase: 'lobby',
  canStart: false,
  players: { p1: {...}, p2: {...} },
  roles: { p1: null, p2: null }
}
```

### Step 5 — P1 chooses the role; P2 is assigned the remainder

On P1's phone, tapping a role card sends:

```js
conn.send({ type: 'setRole', role: 'driver' }); // or 'navigator'
```

The desktop stores the role and broadcasts a `lobby` packet to both phones:

```js
{
  type: 'lobby',
  hostRole: 'driver',
  roles: { p1: 'driver', p2: 'navigator' },
  players: { ... },
  canStart: true
}
```

P2 cannot change its role; it always receives the opposite of P1. While still in the lobby, P1 can
change roles and both phones update. Roles lock when the game starts.

The host permits starting when P1 is connected and has chosen a role, and either P2 has joined or P1
chose Driver (a single-phone driver launch is allowed). P1 is the only player with the **Mulai Game**
and **Ulangi** buttons.

### Step 6 — Start the match

P1's phone sends:

```js
conn.send({ type: 'start' });
```

The desktop validates host + role availability, sets coop mode, puts the tablet in the cockpit, and
broadcasts:

1. `type: 'track'` — the complete minimap route as compact `[x, z]` pairs plus turn distances.
2. `type: 'state'` — `phase: 'countdown'`, the roles, and the countdown length.

The 5-second countdown is rendered as a floating 3D sprite in the scene. At zero the desktop
broadcasts `phase: 'playing'` and starts the automatic acceleration.

### Step 7 — Telemetry and live navigation

While the vehicle is active, the desktop sends a `telemetry` packet at roughly 10 Hz:

```js
{
  type: 'telemetry',
  phase: 'playing',
  speedKmh: 114,
  progress: 387.4,
  x: 12.3, z: -801.7,
  heading: 1.23,
  elapsed: 22.81,
  failureTimer: 0,
  nextTurn: { dir: 'right', angle: 90, distance: 31, active: false }
}
```

The phone's navigator canvas uses the route from `track` and the live car pose from this packet.
The cockpit tablet uses the same navigation renderer architecture, so both maps present the same
white-on-dark-blue route, with the car arrow and upcoming-turn marker in real time.

### Step 8 — Driver input

The driver phone sends pointer state immediately on press/release and as a 100 ms heartbeat while a
button is held:

```js
conn.send({ type: 'input', steer: -1, brake: false });
```

- `steer`: `-1` left, `0` straight, `1` right.
- `brake`: boolean while the **REM** button is held.

The desktop only accepts this packet from whichever connected slot currently has the `driver` role,
and only during countdown/playing. Input older than 550 ms is treated as neutral, so a lost
`pointerup`, a backgrounded phone, or a network hiccup cannot leave the wheel pinned.

### Step 9 — Failure, win, retry

A stuck timer of five consecutive seconds (or an off-road/chasm fall) triggers the fire and
explosion effects. The desktop broadcasts:

```js
{ type: 'state', phase: 'exploding' }
// shortly after:
{ type: 'state', phase: 'gameover', result: { outcome: 'lose', reason, elapsed, progress } }
```

Finishing the 1 km route broadcasts `phase: 'win'` with the result. Both phones show their end screen.

P1's phone sends:

```js
conn.send({ type: 'retry' });
```

The desktop checks that the sender is P1 and that the round is finished, resets the car to the start
line, and sends a fresh countdown state (and the route again) to both phones. The desktop **Ulangi**
button calls the same local reset path. The same track is retained for a retry; a browser refresh
creates a completely new 1 km track and room.

## Protocol summary

| Direction | Type | Purpose |
| --- | --- | --- |
| phone → desktop | `hello` | Begin slot handshake |
| desktop → phone | `welcome` | Slot, host flag, initial role/phase |
| desktop → phones | `lobby` | Role/player/canStart updates |
| phone → desktop | `setRole` | P1 role selection |
| phone → desktop | `start` | P1 starts co-op |
| desktop → phones | `track` | 2D route and turn metadata |
| desktop → phones | `state` | Countdown / playing / exploding / gameover / win |
| desktop → phones | `telemetry` | Speed, route progress, car pose, next turn |
| desktop → phones | `notice` | Non-fatal informational message |
| desktop → phone | `rejected` | Room full / game already running / invalid handshake |
| phone → desktop | `input` | Steering and brake state |
| phone → desktop | `retry` | P1 restarts after game over or win |

## Tuning knobs

All in the `CONFIG` object at the top of `index.html`:

| Key | Default | Meaning |
| --- | --- | --- |
| `CONTROLLER_URL` | placeholder | GitHub Pages controller address |
| `PEER_PREFIX` | `fgdn-` | Must match `controller.html` |
| `TRACK_LENGTH` | `1000` | Exact meter length of the route |
| `MAX_KMH` | `200` | Speed cap |
| `COUNTDOWN_SECONDS` | `5` | 3D countdown length |
| `STUCK_SECONDS` | `5` | Consecutive stopped seconds before explosion |
| `ROAD_HALF_WIDTH` | `5.2` | Asphalt half-width in meters |
| `OFFROAD_LIMIT` | `4.05` | Lateral distance at which the car starts falling |

Track generation lives in `TrackGenerator`; the turn type pool, radii, and clearance are inside that
class. Turn signs, the start/finish gantries, scenery, and the chasm are built by `TrackWorld`.

## Troubleshooting

**The QR opens a “controller URL” page that does not load**
Set `CONFIG.CONTROLLER_URL` or use the in-game URL field with the real GitHub Pages URL. Make sure
the controller is actually enabled under GitHub Pages and the path ends in `controller.html`.

**“Room tidak ditemukan” on the phone**
The room code was refreshed (new page load), the game tab was closed, or the QR is stale. Scan the
current QR again. Both devices need internet access to reach the PeerJS signaling service.

**The phone connects but the game does not start**
P1 must choose a role first. If P1 chooses Navigator, Player 2 must join as Driver. The Start button
has a hint under it explaining what is missing.

**Mobile controls do not reach the car**
Use the phone in the foreground. Keep the desktop game tab visible/open. The input protocol has a
550 ms stale guard, and the phone sends a heartbeat while a button is held.

**Public PeerJS is unavailable / restrictive NAT**
The game uses the public PeerJS Cloud broker and the default WebRTC ICE configuration. Corporate
networks and some carrier NATs can block P2P. For production reliability, self-host PeerJS or add a
TURN server to both Peer constructors:

```js
new Peer(idOrNothing, {
  config: {
    iceServers: [
      { urls: 'stun:stun.l.google.com:19302' },
      { urls: 'turn:YOUR_TURN_HOST:3478', username: '...', credential: '...' }
    ]
  }
});
```

**Safari/iOS**
The controller uses pointer events, `100dvh`, safe-area insets, wake-lock when available, and
multi-touch pointers. Motion/pointer capture behavior varies by iOS version, so test on the target
phone model. Keeping the controller page in the foreground is recommended.

## Design notes

- The track centerline is sampled every **1 meter**, giving exactly 1,000 m of driving distance from
  `d = 0` to `d = 1000`.
- Turns store a signed direction and are constructed as exact circular arcs. The sign flips for every
  turn, guaranteeing the requested strict Right/Left alternation.
- Layout candidates compare non-adjacent track samples and reject any centerline pair closer than
  16 m (approximately 5.6 m of cliff gap at the 10.4 m road width). This keeps curves readable and
  avoids self-intersecting roads.
- The road is a custom ribbon mesh. The cliff sides are layered vertex-colored meshes with jagged
  edges, plus instanced rocks, support pillars, and distant mountain spires.
- Physics is an arcade bicycle-style model: automatic engine acceleration, strong braking, speed-
  dependent steering rate, lateral edge detection against the nearest centerline sample, and a
  substep loop so collisions remain stable at low frame rates.
- The camera is parented inside the cockpit. Body roll, camera yaw, pitch, FOV, and high-speed
  vibration are layered in `Game.updateVisuals()`.

## License / attribution

Original code created for this project. No copyrighted assets are used. The game’s inspiration is
the *Left! Left! LEFT!* communication-driven driving concept; all geometry, UI, and map rendering
here are procedural.
