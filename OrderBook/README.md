# Low-Latency Order Book

| Student Name     | Roll No    |
|------------------|------------|
| Narendra Sirvi   | 24bcs10225 |
| Ojas Maheshwari  | 24bcs10227 |

A C++20 order book for a single instrument. It does not match orders: buy and sell orders stay in the book until they are cancelled. Prices use a tick size of `0.01`; bids range from `50.00` to `99.99` and asks from `100.01` to `150.00`.

We built the order book three ways:

- **Implementation A** keeps bid and ask price levels ordered in a `std::map`.
- **Implementation B** stores price levels in priority queues and uses hash maps for lookup. It uses lazy deletion: empty levels are discarded when the heap is queried.
- **Implementation C** keeps orders in sorted `std::list`s. Each order stores its list iterator, so cancelling is fast, but adding an order means searching the list for its position.

All three implement the `IOrderBook` interface in [order.h](order.h). The interface takes prices as `double`; internally they are rounded to integer ticks. `get_bbo()` returns the best bid and ask `Level`s together with flags saying whether each side is present. The data structures are described in more detail in [implementations.md](implementations.md).

## Project layout

| Path | Contents |
|---|---|
| [order.h](order.h) | `Side`, `Level`, `BBO`, the `IOrderBook` interface and tick conversion helpers |
| [orderbook.h](orderbook.h) | Includes all three implementations; `OrderBook` is an alias for `OrderBookA` |
| [impl_a/](impl_a/), [impl_b/](impl_b/), [impl_c/](impl_c/) | The three implementations, each with a small demo `main` |
| [main.cpp](main.cpp) | Demo program using the default `OrderBook` |
| [verify.cpp](verify.cpp) | Unit tests and a cross-check of A, B and C on random operations |
| [bench.cpp](bench.cpp) | Benchmark harness |
| [benchmark.md](benchmark.md) | Benchmark and `perf` results with discussion |
| [screenshots/](screenshots/) | Screenshots of the benchmark and `perf` runs |

## Build

You need CMake 3.16 or newer and a C++20 compiler.

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j"$(nproc)"
```

This builds `OrderBook`, `OrderBookA`, `OrderBookB`, `OrderBookC`, `verify` and `bench` in `build/`.

## Check correctness

`verify` runs the unit tests and then runs the three order books on the same generated sequence of operations, comparing their results (default 25,000 operations):

```bash
./build/verify --verify --ops 100000
```

The benchmark also has a verification mode:

```bash
./build/bench --verify --ops 100000 --seed 42
```

## Run the benchmark

Run each implementation with the same options:

```bash
./build/bench --impl A --ops 100000 --runs 5 --seed 42
./build/bench --impl B --ops 100000 --runs 5 --seed 42
./build/bench --impl C --ops 100000 --runs 5 --seed 42
```

The seed makes the workload repeatable. It is about 70% adds, 20% cancels and 10% BBO queries, plus a depth query every 100 operations. The program reports median timings, page faults, peak memory use, the final BBO and a checksum.

The assignment asks for one million operations. We used 100,000 for the recorded comparison because Implementation C gets much slower as the book grows. The results are in [benchmark.md](benchmark.md).

## Profile with `perf`

Collect hardware counters for Implementation A:

```bash
perf stat -r 5 \
  -e task-clock,cycles,instructions,branches,branch-misses,cache-misses,L1-dcache-load-misses,dTLB-load-misses,page-faults \
  -- ./build/bench --impl A --ops 100000 --runs 1 --seed 42
```

Repeat with `--impl B` and `--impl C` to compare. Some counters may not be available on every machine.

To see where the time goes, record a profile and open it with `perf report`:

```bash
perf record -g -- ./build/bench --impl A --ops 100000 --runs 1 --seed 42
perf report
```

Repeat for B and C. The results and a short discussion are in [benchmark.md](benchmark.md).
