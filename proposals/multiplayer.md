# Multiplayer

Status: **Draft**
Date: 2026-10-22

## Intended result

Two or more pilots fly the same sky and see each other's aircraft. The first
mode to build is serverless: a casual flight between people who have a link,
with the same direct WebRTC path the phone controller already uses. Voice is
optional. Server mode and hybrid mode use the same aircraft messages later;
they add a server we run, for larger flights and for checks a guest browser
cannot be trusted to make.

Planes, bullets and missiles are already messages. A plane is a pose
snapshot, because each browser runs its own JSBSim and those runs are not a
shared clock. A bullet is one burst, and a missile is one launch. Every
client integrates that event with the same function, so the round they draw
is the round a server can judge. The F-35B still has no stores and no fire
control, so the cockpit sends none of these yet.

Two repositories own the work. [multiplayer](https://github.com/0SFS/multiplayer/blob/main/docs/implementation.md)
owns the messages, the aircraft playout and the integrators.
[anti-cheat](https://github.com/0SFS/anti-cheat/blob/main/docs/implementation.md)
owns the envelope. The server runs that envelope on every message. A pilot
can run the same checks locally, as an opt-in, and be told when someone
else's messages fail. While the option is off, the checker is not called.

Both flight pages stay static files. Serverless production adds no process we
operate. It uses PeerJS Cloud to introduce the browsers, public STUN to find a
direct path, and a direct WebRTC data channel for aircraft motion. That is the
same split as the [phone controller](../../rc/docs/proposals/phone-controller.md):
the broker never sees a join secret or an aircraft snapshot.

## Latency

Three different delays get called "latency." They have different floors.

| What the pilot feels | Where the delay is | Floor |
| --- | --- | --- |
| Own stick, own aircraft | The flight model runs in the pilot's own tab | One flight step, the same as flying alone |
| Another aircraft's motion | One-way network delay, then a pose drawn ahead of the last packet | The direct path between the two browsers |
| Phone controller (`/rc`) | The stick is on the phone and the flight model is on the computer | One-way phone → computer, applied on the next `beforeStep` |

Multiplayer is quicker than `/rc` for the pilot's own aircraft, because the
stick and JSBSim are in the same tab. `/rc` cannot do that: the phone has no
flight model. `createPhoneControlSession` keeps the newest control frame and
`beforeStep` reads it on the next step, which is the right place to apply a
remote stick, and it is still a network wait.

For the other aircraft, a direct WebRTC data channel is the lowest-latency
transport two browsers can open. The browser offers no raw UDP socket and no
peer-to-peer QUIC. `/rc` already uses the fast configuration of that channel:
negotiated, unordered, `maxRetransmits: 0`, signaling unused after setup, STUN
so a direct candidate can win, and a newest-snapshot queue that drops a stale
unsent frame (`peerTransport.ts`). Multiplayer aircraft motion uses that same
channel shape. A lost snapshot costs one sample. A later snapshot carries the
whole pose, so nothing waits for a retransmission.

A 2025 measurement of browser game transports found WebTransport datagrams
ahead of WebRTC data channels, WebSockets, and DTLS-UDP on a client-to-server
path, at 120 ticks per second, including under 0.1% loss
([Evaluating Browser-Based Networking for Real-Time Multiplayer Games](https://aaron.gember-jacobson.com/docs/nsdi2025browser-networking.pdf)).
WebTransport is a connection to a server URL. Server mode, and any hybrid
traffic that travels via the server, uses WebTransport datagrams for aircraft
snapshots. The direct peer path stays on the WebRTC data channel, because that
is the path that skips the server.

Drawing hides part of the remaining one-way delay. Each snapshot carries the
pose, the ground-relative velocity already on `FlightState`, and the body
rates. The receiver advances the newest snapshot by the estimated one-way
delay, clamped to 100 ms, and eases the correction out over a frame or two
when a fresher snapshot arrives. `interpolateFlightState` is the blend. The
receiver does not run a second JSBSim for a remote aircraft: another copy
would diverge, and waiting for the sender's inputs would add delay. Attitude
uses the body rates (`velocities/p-rad_sec`, `q`, `r`) so a steady turn stays
smooth between snapshots.

One-way delay is estimated as half the ICE round trip, which `/rc` already
reads as `currentRoundTripTime`, corrected by a small clock offset measured
with a handshake ping on the reliable channel. Past the 100 ms clamp, the
aircraft is drawn slightly behind the estimate. Guessing a long arc through a
hard pull is worse than being one frame late.

A relay candidate (TURN) adds a hop. Serverless reports that path and keeps
flying on it when ICE selects it. It is a connectivity path. The low-latency
claim belongs to a host or server-reflexive pair. Hybrid prefers the server's
WebTransport relay over a public TURN hop when the direct pair fails, because
a server placed for the session is the shorter client-server path.

Voice has its own budget. It is an audio stream beside the data channels, with
a playback buffer. It never shares the aircraft channel and never gates it.

JSON snapshots are the v1 wire form, as on `/rc`, capped so a snapshot stays
in one packet (under 1100 bytes). A packed binary layout is a later
measurement target if encode time or size shows up in the receive-to-present
trace. The transport, not the text encoding, is the stack cost that the
WebTransport result is about.

## Three modes

One session protocol. The mode chooses who you connect to, and who may decide
that a pose or a future hit counts.

| | Serverless | Server | Hybrid |
| --- | --- | --- | --- |
| Who it is for | Casual flights | Larger, more serious sessions | Competitive sessions, including later duels |
| Setup | PeerJS Cloud, same as `/rc` | Our WebTransport server | Our server admits the session; peers then connect directly |
| Aircraft motion | Direct WebRTC, unreliable, newest snapshot wins | WebTransport datagrams, client → server → others | Direct WebRTC when ICE has a direct pair; that pair's snapshots go through the server when it does not |
| Who simulates an aircraft | The pilot's own browser | The pilot's own browser; the server checks the result | The pilot's own browser; the server checks the result |
| Other aircraft, best case | One-way direct delay | Two legs, via the server | One-way direct delay |
| Pilots in v1 of that mode | 2 to 4 | Up to 16 | 2 to 8 |
| Checks | The join secret. A modified client can lie about its pose | Server envelope on every snapshot. A later version reruns JSBSim from inputs | Same checks as server mode, on a 10 Hz copy. Display uses the direct snapshot |
| Voice | Optional, peer audio | Optional, mixed by the server when a mesh of calls is too many | Optional, peer audio; the server mixes it only for a pair that has no direct path |

Serverless is the one to build. Server and hybrid are specified so the
snapshot, the member list, and the later weapon message do not have to change
shape when they arrive.

Own-aircraft feel stays local in all three modes. A fully server-side flight
model, with the stick waiting on the server the way `/rc` waits on the
computer, is a poor fit for this sim. The server compares or resimulates; it
does not become the place the honest pilot's stick waits.

## Session and channels

Every flight page in a session is a full simulator. The phone controller stays
a controller for the local aircraft. It is not a peer in the flight. A pilot
who flies from the phone publishes the same snapshots as a pilot who flies
from a stick on the computer.

Per peer link, the same two channels as `/rc`:

| Channel | Delivery | Content |
| --- | --- | --- |
| `mp-session-v1` | Reliable, ordered | Hello, join secret, member list, world settings, voice start/stop, later weapon events |
| `mp-pose-v1` | Unordered, `maxRetransmits: 0` | Whole-pose snapshots |

The pose channel is opened negotiated, on an agreed SCTP stream id, after
hello succeeds. The `/rc` rule holds: PeerJS `reliable: false` is ordering
only, in the release the phone controller pins, so the pose channel is a
native data channel on the peer connection.

Snapshot fields:

- Sequence, sender timestamp on the session clock
- Aircraft package id (`f-35b`, `cessna-172`, `sf-50`, …)
- `FlightState`: latitude, longitude, altitude, roll, pitch, heading,
  airspeed, north and east velocity, vertical speed, throttle
- Body rates, radians per second
- Visuals the remote model needs in order to look alive: gear, flaps,
  afterburner, and the F-35B nozzle position when that aircraft sends them

An unknown visual field is ignored. Additive fields work the way they do on
the phone protocol: an older peer keeps flying, and only loses a newer
animation. A snapshot over the size cap, or with a non-finite number, closes
that peer link.

Pose rate is 30 Hz. A hybrid duel may send at 60 Hz, which matches the phone
control rate and the flight step. At a few hundred bytes and 60 Hz, a pilot
sends on the order of 10 KB/s to each other pilot. A mesh of four is a small
upload.

The receiver keeps the newest sequence and drops an older one. Extrapolation
runs from that snapshot until a newer one arrives or until the peer is quiet
for 250 ms (`STALE_MS` on the phone protocol). After that the remote aircraft
holds its last accepted pose and the session reports the link stale. The
local aircraft keeps flying.

## Serverless

### Joining

The host opens Multiplayer and gets an invitation. The page creates a PeerJS
peer, a 256-bit join secret with `crypto.getRandomValues`, and a two-minute
deadline. The link is a flight page, with the secret in the fragment so the
static host does not receive it:

```text
https://0sfs.github.io/fly/#mp=1&v=1&peer=<host-id>&join=<secret>
```

The guest reads the fragment, keeps it in memory, and removes it with
`history.replaceState`, the same as `/rc`. A refresh needs a new invitation
in v1.

The guest loads the simulator, connects one reliable channel to the host, and
sends `hello` with the protocol version and the secret. The secret stays out
of PeerJS metadata, logs, and persistent storage. The host checks version,
deadline, and secret, and consumes the invitation for that guest. Capacity is
four pilots including the host. A fifth, or a reused secret after the
deadline, is rejected. Pending unauthenticated connections are capped at four,
with a five-second hello deadline, same bounds as the phone controller.

ICE uses `resolveIceServers()` from `/rc`: several public STUN servers, with
the build-time and per-device override that already exists. A session that
finds no direct path shows the ICE failure the phone diagnostics already
describe. There is no game-server fallback in this mode.

### Mesh

The host is the member list. When a guest is in, the host sends that guest
the other peers' PeerJS ids. Each pair opens its own session and pose
channels and repeats hello with the session secret. Aircraft motion then
takes one hop.

For two pilots the mesh is a single link, the same shape as a paired phone.
For three or four, a star through the host would make guests see each other
a hop later than they see the host. The mesh avoids that.

The host's departure ends the session in v1. Everyone else's local aircraft
keeps flying, alone. A replacement host can wait.

### Shared sky

On hello, and when the host changes them, the host sends time of day and the
weather settings on the reliable channel. Guests apply that scene. Each
browser still loads its own terrain for where its camera is. The starting
airport is the host's current aircraft position at the moment of the
invitation; a guest who joins later arrives near the host, at the host's
altitude and heading, with the guest's own aircraft id.

A guest may bring a different aircraft. The pose carries the package id. A
receiver that cannot load that package draws a stand-in and still extrapolates
the pose.

Pause is local. A paused peer sends a reliable `held` and stops pose
snapshots. Others draw that aircraft frozen. The paused pilot's own sim
pauses. Nobody else's does.

### What a casual session trusts

The join secret proves the guest has the invitation. It does not prove the
pose is honest. A modified client can publish a teleport. That is acceptable
for a casual flight, and it is the reason server and hybrid exist.

## Voice

Voice is off until a pilot turns it on. Push-to-talk: hold to speak, release
to mute. v1 is a radio. Everyone in the session hears a transmission at the
same level. Spatial volume, quieter with distance, can use the same stream
later; the pose already has the positions.

Voice is a PeerJS media call on its own peer connection, started when the
first pilot pushes to talk, using the same ICE servers as the pose link.
Hanging up, or a microphone denial, closes that call only. The pose channels
stay up. A second connection costs an extra ICE gather at the moment voice
starts, and it keeps a media renegotiation off the flight peer connection.
PeerJS's flight connection is the data-channel negotiator `/rc` already
depends on.

The receiver sets `playoutDelayHint` to 0 on the audio track. Further Opus
frame-size tuning waits on a measurement; the browser default is the v1
encode. Voice diagnostics record the selected candidate pair separately from
the pose link, so a relayed call is visible while the aircraft path is direct.

Server mode, once it exists, mixes voice on the server so a large session
does not open a call per pair. Hybrid keeps peer calls, and uses the server
mix for a pair with no direct path.

## Server mode

A process we run. Browsers open a WebTransport session to it. Datagrams carry
pose snapshots. A reliable WebTransport stream carries hello, membership,
world settings, and later weapon results.

The server fans each snapshot out to the other members. Every client holds
one connection, so the session can grow past a mesh. The first capacity
target is 16 pilots.

Each snapshot is checked against an envelope before it is relayed:

- The step from the last accepted pose stays inside the speed, climb and
  turn bounds `@0sfs/anti-cheat` exports, plus a small slack
- A burst leaves the shooter near the accepted pose, with a muzzle speed
  inside the gun bound. A launch does the same for a missile
- Numbers are finite and inside the globe

Load factor joins the envelope when a pose carries it. The bounds are cheat
limits supplied to the checker, not a second flight model.

A snapshot that fails is dropped. The sender is told, and peers keep the last
accepted pose. Repeated failure removes the pilot from the session.

A later server version runs the same JSBSim build from the pilot's inputs and
publishes that result. Own-ship feel stays on the local model until a
correction arrives; corrections are for disagreement, not for every frame.
That version needs a server-side build of the flight model and is out of
scope until the relay and the envelope are flying.

The server is the member list and the world settings (time of day, weather).
Join is an invitation the server issues, still a fragment secret on a flight
URL, plus the server address.

## Hybrid

The server admits pilots, assigns the session, and runs the same envelope as
server mode. It receives pose snapshots at 10 Hz on WebTransport datagrams,
for the check and for the record.

Display copies come from the direct `mp-pose-v1` channel between each pair.
A pair that never gets a direct candidate receives its display copies through
the server relay instead, and the session shows that pair as relayed. A
public TURN server is configured only as ICE's own last step; the preferred
fallback is our relay, on the WebTransport path the latency section selects.

When the server rejects a snapshot, peers snap that aircraft back to the last
accepted pose on the next display frame. The cheater's local sim can show
whatever it wants; everyone else, and any later hit test, uses the accepted
record.

Competitive size for the first hybrid sessions is a duel, two pilots, one
direct link. The protocol allows up to eight so a larger reserved flight can
exist. Voice and connection count, not the pose format, are what make eight
the cap.

## Rounds

| Entity | On the wire | How everyone else sees it |
| --- | --- | --- |
| Plane | Pose snapshot, unreliable, newest sequence wins | Extrapolate the newest pose, at most 100 ms, then hold |
| Bullets | One `fire` burst: muzzle, velocity, count, rate, time | Integrate the shared ballistic step from that event |
| Missile | One `launch`: muzzle, velocity, target id, navigation gain, turn cap, time | Integrate the shared guidance step against the target's poses |

A client that draws some other trajectory is drawing it alone. The other
pilots never take a per-round position from the shooter, so a forged tracer
stays on the forger's screen. Forging the burst or the launch is visible,
and that message is what the envelope judges.

Hits wait on stores. The server keeps the accepted pose stream and rewinds
it to the firing time. Display extrapolation stays on the clients. Damage
uses the record.

| Mode | Who decides a hit |
| --- | --- |
| Serverless | The shooter's client. Casual, and forgeable |
| Server | The server, from accepted poses |
| Hybrid | The server, from accepted poses. The shooter may draw a tracer immediately; damage appears when the server result arrives |

## Opt-in and download

Multiplayer is off. The flight setting `osfs.multiplayer` defaults off, and
nothing opens a session, a socket, or a peer connection while it is off.

The flight's startup graph does not import `@0sfs/multiplayer` or
`@0sfs/anti-cheat`. The code arrives in two cases:

| When | What happens |
| --- | --- |
| The pilot turns multiplayer on | `import()` runs immediately, including while other transfers are in progress and on save-data |
| The page is idle, the flight is not downloading anything, and the connection is not save-data or 2G | One background fetch. It does not turn the feature on |

`createMultiplayerAvailability` is that gate. A prefetch that finds tiles,
aircraft, or any other transfer still running waits for a later idle. The
client anti-cheat switch stays a second opt-in inside the feature, and its
code arrives with this fetch rather than on its own.

## Anti-cheat

`@0sfs/anti-cheat` is pure checks plus two watchers.

The server watcher runs on every pose, burst and launch before a relay. The
first accepted pose for a pilot is the baseline. A later message that breaks
the envelope is dropped, the pilot is told, and peers keep the last accepted
pose. Three failures in a session remove the pilot. An honest stick still
flies in the local JSBSim. The server compares the published result. It does
not sit in front of the stick.

The client watcher is the same checks, off by default. Turning it on stores
the latest remote message and runs the envelope on a sample interval, then
returns a report the session can show. It does not remove a pilot, does not
change the drawn pose, and does not run while it is off. The flight loop
calls it only when the setting is on, so the cost is limited to pilots who
asked for the reports.

## Landed and still to build

Landed in `@0sfs/multiplayer` and `@0sfs/anti-cheat`: the message codec, pose
playout, ballistic and guidance integrators, a transport-free session world,
the server watcher and the opt-in client watcher. Tests cover sync between
two worlds, extrapolation, round integration, and both watchers.

Still to build, in this order:

1. Two pilots, invite link, one direct link, remote aircraft drawn from the
   session world. Diagnostics show the ICE path, ICE round trip and
   receive-to-present.
2. Mesh to four, shared time of day and weather, mixed aircraft, local pause.
3. The multiplayer tab, including the opt-in client check and its reports.
4. Optional push-to-talk.
5. WebTransport server that relays through the server watcher. Hybrid uses
   the same watcher on a 10 Hz copy.

Hit rewind lands with the F-35B's stores.

## Acceptance for the browser session

The library slice is accepted by its own tests, listed in the two
implementation specs. The browser session, once built, is accepted when:

- Two browsers on one LAN see each other's aircraft move, with the selected
  ICE pair a host or server-reflexive pair, and with pose traffic absent from
  the PeerJS connection after setup.
- A killed peer holds the last pose after 250 ms and the surviving pilot
  keeps flying.
- A third pilot joins from the host's introduction and sees both aircraft on
  one-hop links.
- An oversized, stale, or non-finite snapshot does not move an aircraft.
- The join secret is absent from signaling metadata and from the page URL
  after connect.
- Voice connects and hangs up while pose snapshots continue.
- A pilot on the phone controller still publishes a normal pose.
