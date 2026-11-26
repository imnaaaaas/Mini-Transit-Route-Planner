# Mini Transit Route Planner

A lightweight Python application that finds fast routes through a public-transit network. It models stations and services as a weighted, directed graph, then uses shortest-path algorithms to calculate routes based on travel time.

The project is designed as an algorithm-focused transit routing system. It includes a command-line interface, JSON-based network data, runtime service disruptions, automated tests, and tools for comparing routing performance.

## Features

- **Fastest-route calculation with Dijkstra's algorithm** — finds a minimum-travel-time path through the network.
- **A* route search** — uses geographic distance between stops to guide the search toward the destination.
- **Transfer-limited routing** — finds the fastest route while respecting a maximum number of line changes.
- **Service disruption support** — stops and individual directed connections can be disabled or re-enabled without rebuilding the graph.
- **Custom priority queue** — uses a binary min-heap with a position lookup to support efficient `decrease_key` operations.
- **JSON network loading** — transit stops, coordinates, line identifiers, and travel times are read from JSON files.
- **Command-line interface** — routes can be requested directly from a terminal.
- **Tests and benchmarks** — includes tests for graph behavior, the priority queue, routing algorithms, and comparisons between Dijkstra and A*.

## How It Works

The network is represented as a weighted, directed graph:

- A **stop** is a graph vertex. It has an ID, a name, and geographic coordinates (latitude and longitude).
- An **edge** is a connection from one stop to another. It stores the travel time in minutes and the transit line ID.
- A **route** is an ordered list of stop IDs from the requested origin to the destination.

Edges are directional. A connection from stop A to stop B does not automatically create a connection from B to A; both directions must be included in the JSON data when required. Multiple connections between the same pair of stops are supported, for example when different transit lines serve them.

### Routing Algorithms

**Dijkstra's algorithm**

Calculates a minimum-travel-time route using the edge travel times as weights. It uses the custom min-heap priority queue to select the next stop with the lowest known cost.

**A* search**

Also finds a minimum-travel-time route, but uses a geographic heuristic to guide the search. The heuristic is based on the Haversine distance from the current stop to the destination, divided by the maximum observed network edge speed. This estimates a lower bound on the remaining journey time and helps A* avoid exploring as many stops on the tested networks.

The maximum edge speed is calculated when the `Router` is initialized. Geographic distances are calculated in kilometers; edge travel times are stored in minutes.

**Transfer-limited routing**

The transfer-aware search tracks more than the current stop. Its search state contains:

```python
(stop_id, current_line, transfers_used)
```

A transfer is counted when the route changes from one line to another. States that exceed the requested transfer limit are skipped. This allows the planner to compare routes that reach the same stop using different lines or different numbers of transfers.

### Service Disruptions

The graph supports disabling and re-enabling stops and connections at runtime:

- Disabling a stop prevents routes from passing through it.
- Disabling an edge removes that directed connection from route searches.
- Re-enabling a stop or edge makes it available again.

The routing algorithms use the graph's filtered neighbor lookup, so they do not need separate outage-handling logic.

## Project Structure

```text
Mini-Transit-Route-Planner/
├── network_jsons/
│   ├── example_network.json
│   ├── example_network2.json
│   ├── example_network_100.json
│   ├── example_network_1000.json
│   └── example_network_10000.json
├── tests/
│   ├── test_astar_router.py
│   ├── test_graph.py
│   ├── test_network.py
│   ├── test_priority_queue.py
│   ├── test_shortest_path.py
│   └── test_shortest_path_with_transfers.py
├── transit/
│   ├── __init__.py
│   ├── __main__.py
│   ├── graph.py
│   ├── network.py
│   ├── priority_queue.py
│   └── router.py
├── utils/
│   ├── __init__.py
│   ├── benchmark_decorator.py
│   ├── generate_network.py
│   └── logger.py
├── benchmark.py
├── DESIGN.md
├── requirements.txt
└── transit_router.md
```

### Main Components

| File | Responsibility |
|---|---|
| `transit/graph.py` | Defines stops, edges, the adjacency-list graph, outage handling, and JSON loading. |
| `transit/priority_queue.py` | Implements the binary min-heap priority queue and `decrease_key`. |
| `transit/router.py` | Implements Dijkstra, A*, transfer-limited routing, and Haversine distance calculations. |
| `transit/network.py` | Loads a network and exposes a simpler interface to the routing methods and disruption controls. |
| `transit/__main__.py` | Parses command-line arguments and runs a route query using A*. |
| `utils/generate_network.py` | Generates synthetic transit networks for experiments. |
| `utils/benchmark_decorator.py` | Measures route-query execution time and the number of visited nodes. |
| `benchmark.py` | Runs Dijkstra and A* benchmarks against the larger sample networks. |
| `tests/` | Contains automated tests for graph behavior, routing, and the priority queue. |

## Requirements

- Python 3.10 or newer
- pip

The project dependencies are listed in `requirements.txt`. The current requirements file pins pytest for running the test suite.

## Installation

Clone the repository and move into the project directory:

```bash
git clone https://github.com/imnaaaaas/Mini-Transit-Route-Planner.git
cd Mini-Transit-Route-Planner
```

Create and activate a virtual environment:

**Linux / macOS**

```bash
python3 -m venv .venv
source .venv/bin/activate
```

**Windows PowerShell**

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
```

Install the dependencies:

```bash
python -m pip install -r requirements.txt
```

## Running the Application

Run the CLI from the repository root.

Use the default sample network:

```bash
python -m transit route N1 E2
```

The command loads `network_jsons/example_network.json`, runs A* search, and prints the resulting sequence of stop IDs.

Choose a different network with the `--network` option:

```bash
python -m transit --network network_jsons/example_network_100.json route S1 S78
```

The start and destination arguments are stop IDs defined in the selected JSON file.

## Using the Python API

The `Network` class provides access to the routing methods without requiring callers to work directly with the graph and router objects.

```python
from transit.network import Network

network = Network("network_jsons/example_network.json")

# Fastest route using Dijkstra's algorithm
dijkstra_route = network.shortest_path("N1", "E2")

# Fastest route using A*
astar_route = network.shortest_path_astar("N1", "E2")

# Fastest route with at most one transfer
limited_route = network.shortest_path_with_transfers(
    "N1",
    "E2",
    max_transfers=1,
)

print("Dijkstra:", dijkstra_route)
print("A*:", astar_route)
print("Transfer-limited:", limited_route)
```

The routing methods return a list of stop IDs in travel order. If no route can be found under the requested conditions, they return an empty list.

### Disabling a Stop or Connection

The same `Network` object exposes methods for simulating service disruptions:

```python
network.disable_stop("C")

# A route that depends on stop C may now be unavailable.
route_during_disruption = network.shortest_path("N1", "E2")

network.enable_stop("C")

# Stop C is available again.
restored_route = network.shortest_path("N1", "E2")
```

Connections can also be disabled or re-enabled by specifying their source stop, destination stop, and line ID:

```python
network.disable_edge("C", "E1", "blue")
network.enable_edge("C", "E1", "blue")
```

Because edges are directed, disabling `C -> E1` does not automatically disable `E1 -> C`.

## Sample Networks

The repository includes several JSON datasets:

- `example_network.json` — a small sample network with 10 stops and intersecting red and blue lines.
- `example_network2.json` — an extended sample that adds a green line, allowing tests to compare a slower route with no transfers against a faster route that requires a transfer.
- `example_network_100.json`, `example_network_1000.json`, and `example_network_10000.json` — larger generated networks used for performance experiments.

Each network file contains a `stops` array and an `edges` array. A stop contains `id`, `name`, `lat`, and `lon`. An edge contains `from`, `to`, `travel_time`, and `line_id`.

Example edge:

```json
{
  "from": "N1",
  "to": "N2",
  "travel_time": 3,
  "line_id": "red"
}
```

Travel times are expressed in minutes. Coordinates are used by the A* heuristic.

## Running Tests

Run the test suite from the repository root:

```bash
python -m pytest -v
```

The tests cover graph loading and outage behavior, priority queue operations, shortest-path results, transfer constraints, and A* heuristic admissibility.

## Benchmarking

Run the benchmark script from the repository root:

```bash
python benchmark.py
```

It runs a set of route queries on the 100-, 1,000-, and 10,000-stop sample networks. For each algorithm, it reports aggregate execution time and the total number of visited nodes across the configured queries.

The existing benchmark notes in `DESIGN.md` record fewer visited nodes and lower execution time for A* than Dijkstra on the tested datasets. Actual timings depend on the machine and should be measured again when comparing environments.

## Additional Documentation

- [DESIGN.md](DESIGN.md) — implementation decisions, class responsibilities, complexity notes, benchmark observations, and the A* heuristic discussion.
- [transit_router.md](transit_router.md) — the original project brief and functional requirements.

## Scope

This repository is an algorithm-focused route planner that operates on supplied JSON networks. It provides a command-line interface and a Python API; it does not currently provide a graphical map, a web application, live transit feeds, or integration with a real city's transit service.
