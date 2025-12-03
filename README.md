## Routing Protocols Simulator + NetVis

A lightweight network routing simulator (Python) with an interactive web visualizer (NetVis). It includes:

- Learning Switch example
- Distance-Vector router framework and tests
- Built-in topologies and a simple topology file format
- A browser-based UI to visualize nodes, links, and packet flow


## Project layout

```text
simulator/
  simulator.py            # Entry point (boot)
  sim/                    # Core simulator engine and API
  dv/                     # Distance Vector framework and tests
  ls/                     # Learning switch base classes
  topos/                  # Example topologies (.py and .topo)
  examples/               # Small demo modules
  tools/                  # Optional log viewer utilities

netvis/NetVis/            # Web UI (served by the simulator)
  index.html              # Opened automatically by the web server
  *.pde                   # Processing.js visualizer sources
  popupwindow.css         # UI styles
```


## Requirements

- Python 3.8+ (Python 2.7 also works, but Python 3 is recommended)
- A modern browser (Chrome/Edge/Firefox) for the NetVis UI

No third-party Python packages are required; the simulator uses only the standard library. The NetVis UI loads its JS dependencies from CDNs.


## Quick start

These commands assume your shell is at the repository root.

### 1) Learning Switch demo

```bash
python simulator/simulator.py --default-switch-type=learning_switch topos.simple --start
```

- Then open `http://127.0.0.1:65432/` in your browser.

### 2) Distance-Vector router demo

```bash
python simulator/simulator.py --default-switch-type=dv.router topos.simple --start
```

- Then open `http://127.0.0.1:65432/`.

### 3) Run a predefined topology file

Topology files live in `simulator/topos/*.topo`. Use the loader module and pass the filename (without extension):

```bash
python simulator/simulator.py --default-switch-type=dv.router topos.loader=demo --start
```

Examples available: `demo`, `triangle`, `ring`, `star`, and more (see `simulator/topos/`).

### 4) Headless (no UI)

```bash
python simulator/simulator.py --no-interactive --remote-interface=None topos.simple --default-switch-type=dv.router
```


## NetVis (web UI)

When you start the simulator (default: web interface enabled), it also runs a small HTTP server that serves the visualizer:

- URL: `http://127.0.0.1:65432/` (changeable via flags below)
- Shows nodes (hosts are circles, switches are squares), links, and animated packets

Keyboard shortcuts inside NetVis:

- a / b: set the A/B marker (select a node first)
- x: swap A and B
- e: toggle a link between A and B
- p: send a ping from A to B
- d: disconnect the selected node
- o / O: pin or unpin all nodes
- Shift+0..9: user-defined functions (module-dependent)


## Common CLI flags

Pass flags before modules on the command line. Useful options (defaults in parentheses):

- `--default-switch-type=<module[.Class]>` (None): switch type used by topologies and examples
- `--default-host-type=<module[.Class]>` (BasicHost): host type used by topologies
- `--remote-interface=web|tcp|udp|None` (`web`): enable UI/websocket server
- `--remote-interface-port=<int>` (65432): UI/websocket port
- `--remote-interface-address=<addr>` (`0.0.0.0`): bind address
- `--start`: auto-start simulation in interactive mode
- `--no-interactive`: run without the interactive console
- `--very-quiet`: suppress console output

Modules to run are listed after flags, for example:

```bash
python simulator/simulator.py --default-switch-type=dv.router examples.test_simple --start
```

Notes:
- Module names are relative to the `sim` package (`topos.simple`, `dv.router`, `examples.test_simple`, etc.).
- To pass a positional argument to a module’s `launch()`, use `module=arg` (e.g., `topos.loader=demo`).


## Testing (Distance Vector)

Run DV unit tests:

```bash
python simulator/dv/unit_tests.py                # run all stages
python simulator/dv/unit_tests.py 5              # run up to stage 5
python simulator/dv/unit_tests.py -v             # increase verbosity
```


## Troubleshooting

- Port already in use: pick another UI port using `--remote-interface-port=PORT`.
- Browser doesn’t show nodes: confirm you opened the right URL and that your chosen topology/module created entities.
- Nothing animates: if you didn’t pass `--start`, type `start()` in the interactive console, or rerun with `--start`.
- Windows/Python launch: if `python` isn’t recognized, try `py` instead of `python`.


