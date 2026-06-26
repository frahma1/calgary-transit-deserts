# Does Calgary Have Transit Deserts?
### Measuring recreational access by transit for school-aged kids

---

## The Problem

As cities debate regulating social media for young people, there is little conversation about what social alternatives exist. Public transit is the primary way car-free youth get around independently, but stops aren't always where they need to be, and even when they are, the routes don't always go anywhere useful.

This project asks a specific, computable version of that question: **How many Calgary schools sit within walking distance of a transit stop, and from that stop, can a kid reach a recreation centre or library within 20 minutes on weekday afternoons?**

The answer is mapped, ranked, and broken into four tiers of connectivity. This should show how access to social third spaces spans across the city, and room for improvement.

---

## Data

All data is free, open and provided by the city of Calgary.

| Dataset | Source | Notes |
|---------|--------|-------|
| Calgary Transit stops & schedules | Open Calgary (GTFS) | stops.txt, stop_times.txt, trips.txt, calendar.txt |
| Schools | Open Calgary | Coordinates stored as WKT `POINT(lon lat)` strings |
| Libraries | Open Calgary | Coordinates stored as `(lat, lon)` strings |
| Recreation centres | Open Calgary | Coordinates stored as WKT `POINT(lon lat)` strings |

The coordinate formats were inconsistent across datasets and required separate regex parsers for each. GTFS departure times are stored as `HH:MM:SS` strings including values past midnight so all times were converted to total seconds to make comparisons reliable. Additionally, some data was missing and malformed so the data was cleaned. 

---

## Methodology

### Step 1 - Filter to relevant trips

Due to the complexity of the transit system and the large amount of data, I quickly narrowed down the focus of the analysis to target weekday afternoons from 3:30pm to 10:00pm, when kids are most likely traveling independently after school. Trips were filtered to this window and to weekday service IDs from the GTFS calendar. 

### Step 2 - Spatial radius lookup with a KD-Tree

Rather than computing the distance between every school and every stop (an O(n²) operation), a KD-Tree was built on all stop coordinates converted to radians. `query_ball_point()` then returns all stops within 500m of each school and each third space in a single vectorised call. This runs in roughly O(n log n) and handles hundreds of schools and thousands of stops without looping.

The 500m threshold represents a reasonable walking distance. Both ends of the journey were indexed; stops near schools (departure) and stops near libraries and rec centres (arrival). This further cut down the number of AI computation need to be calculated.

### Step 3 - Build a weighted transit graph

The GTFS schedule was converted into a directed weighted graph where each node is a stop and each edge is a direct bus leg between consecutive stops, weighted by ride time in seconds.

Because the same stop pair can appear on multiple trips at different times, only the minimum ride time across all trips was kept per stop pair. This answers "what is the fastest this leg could possibly be?" rather than "when does the next bus leave?".The graph was stored as an adjacency list (`defaultdict(list)`) so that neighbour lookups during pathfinding are O(1) rather than requiring a full table scan.

Dijkstra's algorithm was run from every stop near a school, finding all third-space entry stops reachable within 20 minutes of total ride time. BFS was considered but rejected because BFS only finds shortest paths when all edges are equal weight and requires the whole graph to be scanned. Here legs have different durations, so BFS would return incorrect results.

Transfers between routes are handled automatically. Dijkstra doesn't track which route an edge belongs to, only the cumulative cost. A path that takes route 3 for 8 minutes and route 17 for 9 minutes has a total cost of 17 minutes and is included. Transfer wait time is not modelled, this is a limitation discussed below.

Two optimizations were applied to keep the search efficient:

- The heap is broken out of early once `cost > cutoff`, since the heap is sorted and nothing cheaper can remain
- Once Dijkstra reaches a third-space entry stop it records it but does not push it onto the heap, since there is no need to explore onward from a destination

```python
if v not in end_stops:
    heapq.heappush(heap, (new_cost, v))  # only keep exploring from non-destinations
```

Results were stored as `reachable_from[stop_id]`, a dict of `{third_space_stop: seconds}`, for every school-adjacent stop.

### Step 5 - Score each school

For each school, the reachable third-space stops from all of its nearby stops were unioned together. A reverse lookup dict (`stop_to_rec`, `stop_to_lib`) was pre-built outside the scoring function so that translating a stop ID to a rec centre or library name is an O(1) dict lookup rather than a DataFrame filter per school.


### Step 6 - Four-tier classification

Schools were classified into four tiers:

| Tier | Criteria |
|------|----------|
|  Transit desert | No stop within 500m |
|  Stop access only | Stop nearby, but no third space reachable in 20 min |
|  Connected | 1–2 third spaces reachable |
|  Well connected | 3 or more third spaces reachable |

---

## Modelling Decisions  

An earlier approach stored every departure time for every stop pair and attempted to simulate exact boarding sequences. This added significant complexity, tracking when a kid arrives at a stop, whether they missed a departure, and when the next one comes, without meaningfully improving the question being answered. Since the goal is "can a kid get there within 20 minutes?" rather than "when exactly does the next bus leave?", departure times are out of scope. The schedule was collapsed to a single minimum-cost edge per stop pair across all trips:

```
pythonkey = (from_stop, to_stop)
if key not in edge_min or ride < edge_min[key]:
    edge_min[key] = ride
```
This answers the capability question directly, if the fastest version of a leg fits within 20 minutes, the destination is reachable. It also reduced the graph to its smallest useful form, cutting Dijkstra's search space significantly. The tradeoff is that it assumes best-case headways, which is noted in the limitations.
--- 
## Limitations

**Transfer wait time is not modelled.** The 20-minute cap applies to pure ride time. In practice, a transfer adds waiting time that could push a journey well past 20 minutes. A more realistic model would add a penalty per transfer. Using an algorithm like RAPTOR would provide more realistic travel time estimates, but requires significantly more computation.

**Walking speed is assumed.** The 500m radius assumes a comfortable walking pace. For younger children, or in winter conditions, a tighter radius (300–400m) may be more realistic.

**Best-case headways.** Using the minimum ride time across all trips assumes the fastest possible leg is always available. In reality a kid might wait 15 minutes for a connection. This analysis measures transit capability, not typical experience.

**Scope of third spaces.** This model only considers city-owned recreation centres and libraries. It does not account for private facilities, extracurricular activities, or pre-scheduled commitments that might change where a kid is actually trying to go.

**Boarding stop assumed.** The model assumes a kid boards at whichever nearby stop gives the best connection. It does not account for stop familiarity or whether a particular stop feels safe or accessible.

---

## Results

Schools in and around downtown Calgary are the most connected, reflecting the density of the transit network there. Schools on the suburban areas, particularly in newer neighbourhoods in the south and northeast, show the weakest connectivity. Regardless, the average school has access to three or more recreational facilities, meaning kids should have their pick of third spaces. Not it is up to libraries, YMCAs and parks to provide programming to keep them engaged. 
[![Map preview](/calgay_transit.png)](/calgary_kids_transit_ranked.html)
---

## Skills Demonstrated

- Spatial indexing with `scipy.spatial.cKDTree`
- Graph construction and shortest-path search (Dijkstra) on real GTFS transit data
- GTFS data parsing including edge cases (>24h time strings, inconsistent coordinate formats)
- Pandas data pipeline with `apply`, `defaultdict` lookups, and set operations
- Interactive map with ranked, layered visualisation using Folium and Branca