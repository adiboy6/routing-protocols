## Routing Protocols

This repository centers on implementing and experimenting with two classic L2/L3 forwarding/routing algorithms:

- Learning Switch (MAC-learning, flood-on-miss)
- Distance Vector Routing (Bellman–Ford style, with split horizon/poison options)

A small Python simulator provides time, links, and message delivery; a simple web UI (NetVis) can visualize flows. This README focuses on the algorithms and how to implement/validate them; simulator details appear briefly at the end.


## Key files (algorithm-focused)

- Learning Switch:
  - `simulator/learning_switch.py` — the learning switch implementation
  - `simulator/ls/learning_switch_base.py` — base class and port utilities

- Distance Vector:
  - `simulator/dv/dv_base.py` — DV framework (packets, timers, flags, `Table`, `TableEntry`)
  - `simulator/dv/router.py` — your DV router implementation
  - `simulator/dv/unit_tests.py` — comprehensive DV tests (staged)

- Shared API primitives used by both:
  - `simulator/sim/api.py` — `Entity`, `Packet`, timers, NetVis integration hooks
  - `simulator/sim/basics.py` — `BasicHost`, `Ping`/`Pong`, `HostDiscoveryPacket`


## Learning Switch

Goal: Forward frames by learning source locations and flooding on unknown destinations. Robustly handle link up/down and entry expiry.

Core callbacks to use (see `LearningSwitchBase`):

- `on_data_packet(packet, in_port)`
  - Learn: map `packet.src` → `in_port`
  - If destination is known and not the input port, send unicast out the learned port; otherwise, flood out all other up ports.

- `on_link_up(port)` / `on_link_down(port)`
  - Maintain table validity: remove or invalidate entries learned on that port when it drops.

- `on_timer()`
  - Age out stale entries using a configurable timeout (e.g., `TIMEOUT = 15` seconds). Any entry older than `TIMEOUT` should be removed.

Reference implementation approach (high-level):

- Store `routing_table: dict[HostName] -> (port_number, last_seen_time)`
- On every packet from a source, update/refresh its entry timestamp.
- On a miss (unknown destination), flood all up ports except `in_port`.
- On `on_link_down`, immediately invalidate any entries whose output port equals `port` (fast convergence).

Testing locally:

```bash
python simulator/simulator.py --default-switch-type=learning_switch topos.simple --start
```

Tip: Some tests disable host discovery (`ENABLE_DISCOVERY = False`) to ensure true learning based on data-plane packets.


## Distance Vector Routing

Goal: Compute next hops and costs to all hosts using distance-vector advertisements. Support periodic and triggered updates, route expiry, and policy toggles (split horizon, poison reverse, etc.).

Important framework components:

- `Table` and `TableEntry` (`dv_base.py`)
  - `Table` validates entries: keys must be `HostEntity`, values are immutable `TableEntry(destination, port, latency, expire_time)`
  - `FOREVER` denotes a static, never-expiring route (for directly-connected hosts)

- `AdvertisementPacket(destination, latency)`
  - Carries an advertised cost to `destination` from a neighbor. Your router adds the local link latency to produce the total path cost through that neighbor.

- Timers and ports
  - `.ports` tracks up ports and their latencies
  - `handle_timer()` (in the base) calls `send_routes(force=True)` periodically

You implement the DV logic in `simulator/dv/router.py` by overriding these hooks from `DVRouterBase`:

- `add_static_route(host, port)`
  - Install a `TableEntry` for a directly-attached host with `latency = link_latency(port)` and `expire_time = FOREVER`.
  - Typically triggers a (possibly incremental) advertisement.

- `on_route_advertisement(destination, latency, port)`
  - Apply Bellman–Ford: candidate distance = `latency (neighbor->dest) + link_latency(port)`
  - Update or insert routes if the candidate strictly improves (or replaces a stale/poisoned entry), and set `expire_time = now + ROUTE_TTL`.
  - Trigger incremental ads if a route changed.

- `on_data_packet(packet, in_port)`
  - Forward using the best known route, unless the selected route’s latency is ≥ `INFINITY` (drop) or would hairpin (see flag below).

- `send_routes(force=False, single_port=None)`
  - Advertise to neighbors. When `force=True`, send a full table; otherwise, send only deltas (incremental updates).
  - Apply split horizon and/or poison reverse:
    - Split horizon: never advertise a route back out the port it came from.
    - Poison reverse: do advertise back to that neighbor, but with `latency = INFINITY` for that destination.
  - If `single_port` is set, send updates only to that port (e.g., right after link-up).

- `expire_routes()`
  - For non-FOREVER entries: if `now > expire_time` then either remove the entry or (if `POISON_EXPIRED` is enabled) set its latency to `INFINITY` and reset its TTL so neighbors learn it’s no longer valid.

- `on_link_up(port, latency)` / `on_link_down(port, latency)`
  - Maintain `.ports` and, depending on flags, immediately send a full or incremental update. On link-down, optionally poison affected routes (`POISON_ON_LINK_DOWN`).


### DV behavior toggles (CLI)

DV options are configured via the `dv` module’s launch arguments and must appear BEFORE your topology module on the command line. Example:

```bash
python simulator/simulator.py --default-switch-type=dv.router dv --ttl=15 --pr --link-down topos.simple --start
```

Supported options (map to `DVRouterBase`):

- Distances/timers:
  - `--inf=<float>`: `INFINITY` (max cost, default 16)
  - `--ttl=<float>`: `ROUTE_TTL` (seconds a learned route stays valid)
  - `--period=<float>`: `PERIODIC_INTERVAL` (seconds between full ads)
  - `--unsync`: `RANDOMIZE_TIMERS` (jitter timer starts)

- Policy:
  - `--sh`: `SPLIT_HORIZON`
  - `--pr`: `POISON_REVERSE` (do not combine with `--sh`)
  - `--p`: `POISON_EXPIRED` (poison timed-out routes)
  - `--link-up`: `SEND_ON_LINK_UP` (advertise on link up)
  - `--link-down`: `POISON_ON_LINK_DOWN` (poison affected routes)
  - `--nohairpin`: `DROP_HAIRPINS` (drop packets that would return out the input port)

Notes:
- If both split horizon and poison reverse are requested, the launcher will error (they are mutually exclusive).
- Options must be set prior to creating any nodes/links; place the `dv` module (with its flags) before the topology module.


### DV testing

Staged unit tests thoroughly exercise the DV behaviors (static routes, forwarding, advertisements, expiry, split horizon/poison, triggered updates):

```bash
python simulator/dv/unit_tests.py         # run all stages
python simulator/dv/unit_tests.py 5       # run up to stage 5
python simulator/dv/unit_tests.py -v      # increase verbosity
```

The tests modify DV flags internally for each stage; you don’t need to set CLI flags when running the test suite.


## Simulator basics (brief)

You mainly need to know how to run a topology with your chosen switch/router:

- Learning Switch (simple line topology):
  ```bash
  python simulator/simulator.py --default-switch-type=learning_switch topos.simple --start
  ```

- Distance Vector with options:
  ```bash
  python simulator/simulator.py --default-switch-type=dv.router dv --ttl=15 --pr topos.simple --start
  ```

If the web UI is enabled (default), open `http://127.0.0.1:65432/` to visualize nodes, links, and packets.


## Tips and debugging

- Use `api.userlog.debug/info/warning(...)` to emit readable logs.
- Set `s_info` (see `DVRouterBase`) to show per-node info when selected in NetVis.
- To test convergence, add small delays before sending pings (e.g., yield 3–5 seconds in an example tasklet).


## Project layout (reference)

```text
simulator/
  simulator.py            # Entry point (boot)
  sim/                    # Core simulator engine and API
  dv/                     # Distance Vector framework, router, tests
  ls/                     # Learning switch base classes
  topos/                  # Example topologies (.py and .topo)
  examples/               # Small demo modules

netvis/NetVis/            # Web UI (served by the simulator)
```

