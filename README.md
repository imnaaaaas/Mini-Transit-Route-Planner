# Mini Transit Route Planner

A small Python project that finds the fastest route between stops in a public-transit network. The network is represented as a weighted, directed graph, and routes are calculated using Dijkstra's algorithm, A*, or a transfer-limited search.

The project focuses on graph algorithms and data structures: it includes an adjacency-list graph, a binary min-heap priority queue implemented from scratch, route reconstruction, network outage simulation, tests, and benchmark utilities.

## Features

- **Shortest route with Dijkstra** — finds the minimum-travel-time path using edge travel times.
- **A* route search** — uses stop coordinates and a Haversine-distance heuristic to guide the search toward the destination.
- **Transfer-limited routing** — finds the fastest route that does not exceed a specified number of line changes.
- **Network outage simulation** — temporarily disable or re-enable individual stops and directed connections without rebuilding the graph.
- **JSON network loading** — load stops, coordinates, route-line identifiers, and travel times from a JSON file.
- **Custom priority queue** — binary min-heap with an index map and `decrease_key`, used by the routing algorithms.
- **Synthetic network generation and benchmarks** — generate larger grid networks and compare Dijkstra and A* search.

This is a command-line and Python-library project. It does not provide a graphical map, live vehicle positions, real-time schedules, or departure-time planning.

## How it works

The transit network is stored as an adjacency-list graph:

- A **stop** is a vertex with an ID, name, latitude, and longitude.
- A **connection** is a directed, weighted edge with a travel time (in minutes) and a line ID.
- A **route** is returned as an ordered list of stop IDs.

The graph and routing logic are separate. The `Graph` class stores the network and handles disabled stops/connections. The `Router` class runs the pathfinding algorithms over that graph. The `Network` class loads the JSON file and provides a simpler interface for callers.

### Routing algorithms

| Method | Purpose |
| --- | --- |
| `shortest_path(start, end)` | Dijkstra's algorithm; finds the route with the lowest total travel time. |
| `shortest_path_astar(start, end)` | A* search; uses geographic distance as a heuristic to prioritize promising stops. |
| `shortest_path_with_transfers(start, end, max_transfers)` | Finds the lowest-time route while limiting changes between line IDs. |

For A*, the project calculates straight-line distance with the Haversine formula. It divides that distance by the maximum edge speed found when the router is initialized, producing a lower-bound travel-time estimate for the heuristic.

The transfer-limited search tracks more than the current stop: its search state also includes the current line and the number of transfers used. A transfer is counted when a journey changes from one `line_id` to another; boarding the first line does not count as a transfer.

### Example network

The small sample network contains a north-south red line and an east-west blue line that meet at Central Hub (`C`). A journey from `N1` to `E2` therefore needs one line change. The extended sample, `example_network2.json`, adds a green line that provides an alternative with no transfers.

## Getting started

### Requirements

- Python 3.10 or newer
- pip

The application code uses the Python standard library. The dependency listed in `requirements.txt` is `pytest`, which is used to run the tests.

### Installation

Clone the repository and move into its directory:

```bash
git clone https://github.com/imnaaaaas/Mini-Transit-Route-Planner.git
cd Mini-Transit-Route-Planner
```

(Optional but recommended) Create and activate a virtual environment:

```bash
python -m venv .venv

# macOS / Linux
source .venv/bin/activate

# Windows PowerShell
.venv\Scripts\Activate.ps1
```

Install the test dependency:

```bash
python -m pip install -r requirements.txt
```

## Run the command-line planner

From the repository root, run:

```bash
python -m transit route N1 E2
```

The CLI loads `network_jsons/example_network.json` by default and uses A* to find the route. For the sample network, the result is:

```text
['N1', 'N2', 'N3', 'C', 'E1', 'E2']
```

To use a different network file, pass `--network` before the `route` subcommand:

```bash
python -m transit --network network_jsons/example_network2.json route N1 E2
```

The command accepts stop IDs, not stop names. The returned list also contains stop IDs.

## Use it as a Python library

The `Network` class is the main entry point when using the project from Python:

```python
from transit.network import Network

network = Network("network_jsons/example_network.json")

# Dijkstra
print(network.shortest_path("N1", "E2"))

# A*
print(network.shortest_path_astar("N1", "E2"))

# Find the fastest route using at most one transfer
print(
    network.shortest_path_with_transfers(
        "N1",
        "E2",
        max_transfers=1,
    )
)
```

Each route method returns a list of stop IDs. If no valid route is found, it returns an empty list (`[]`). A journey whose start and destination are the same returns a one-element path.

### Simulate a stop or connection outage

Stops and directed connections can be disabled and enabled at runtime:

```python
from transit.network import Network

network = Network("network_jsons/example_network.json")

# Temporarily close Central Hub
network.disable_stop("C")
print(network.shortest_path("N1", "E2"))

# Reopen it
network.enable_stop("C")

# Disable only the blue connection from C to E1
network.disable_edge("C", "E1", "blue")
network.enable_edge("C", "E1", "blue")
```

Disabling a stop removes it from routing in both directions. Disabling an edge affects only the exact directed connection and line ID supplied; the reverse connection is independent and must be disabled separately when required.

## Network JSON format

A network file contains a `stops` array and an `edges` array:

```json
{
  "stops": [
    {
      "id": "A",
      "name": "Central Station",
      "lat": 40.74,
      "lon": -74.00
    },
    {
      "id": "B",
      "name": "West Station",
      "lat": 40.74,
      "lon": -74.02
    }
  ],
  "edges": [
    {
      "from": "A",
      "to": "B",
      "travel_time": 4,
      "line_id": "blue"
    },
    {
      "from": "B",
      "to": "A",
      "travel_time": 4,
      "line_id": "blue"
    }
  ]
}
```

- `id` uniquely identifies a stop and is used by the routing methods.
- `lat` and `lon` are geographic coordinates in decimal degrees.
- `from` and `to` identify the source and destination stop of an edge.
- `travel_time` is the connection's travel time in minutes.
- `line_id` identifies the transit line used by that connection.

**Edges are directed.** Add both directions explicitly when travel is possible both ways. Multiple edges between the same pair of stops are allowed, for example when different lines connect the same stops.

## Project structure

```text
Mini-Transit-Route-Planner/
├── network_jsons/
│   ├── example_network.json
│   ├── example_network2.json
│   └── example_network_*.json
├── tests/
│   ├── test_astar_router.py
│   ├── test_graph.py
│   ├── test_network.py
│   ├── test_priority_queue.py
│   ├── test_shortest_path.py
│   └── test_shortest_path_with_transfers.py
├── transit/
│   ├── __main__.py
│   ├── graph.py
│   ├── network.py
│   ├── priority_queue.py
│   └── router.py
├── utils/
│   ├── benchmark_decorator.py
│   ├── generate_network.py
│   └── logger.py
├── benchmark.py
├── DESIGN.md
├── requirements.txt
└── README.md
```

## Tests

Run the test suite from the repository root:

```bash
python -m pytest
```

The tests cover graph construction and outage behavior, priority-queue operations, Dijkstra and A* route results, the A* heuristic's admissibility, transfer limits, and the `Network` wrapper.

## Benchmarks

The repository includes `benchmark.py` for comparing Dijkstra and A* on generated networks. The benchmark decorator records elapsed time and the number of visited stops. The benchmark observations and design notes are documented in [DESIGN.md](DESIGN.md).

The generator in `utils/generate_network.py` creates square grid networks. Its `size` argument is the number of rows and columns, so the generated network contains `size × size` stops.

## Design notes

See [DESIGN.md](DESIGN.md) for the class responsibilities, complexity discussion, benchmark observations, and explanation of the A* heuristic.
