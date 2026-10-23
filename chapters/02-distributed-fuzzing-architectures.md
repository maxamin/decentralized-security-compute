# Distributed Fuzzing Architectures

Before decentralising, understand how fuzzing distributes in a trusted setting. Every pattern below assumes workers are honest. Later chapters remove that assumption one piece at a time.

## Patterns

**1. Independent instances with periodic sync (AFL++ `-M`/`-S`).** Each instance writes its queue to a shared directory and imports others' new entries. Simple, robust, tolerant of slow sync (minutes). The main instance (`-M`) performs deterministic stages; secondaries (`-S`) do randomised mutation. Recent AFL++ versions skip deterministic stages by default, but the main/secondary split still controls who does extra work such as corpus import ordering.

**2. Broker-based (LibAFL LLMP).** Instances exchange new testcases and stats through a low-latency message broker. LLMP ("low level message passing") uses shared memory on one host. Multi-machine setups connect brokers over TCP. Efficient on a LAN, sensitive to WAN latency.

**3. Central scheduler (ClusterFuzz).** Bots pull tasks (fuzz target X for N hours, minimise crash Y, run regression) from a central queue, then upload corpora and crashes to blob storage. Corpus pruning runs as a separate task. Crash deduplication uses stack-trace signatures, and fixed bugs are verified automatically by re-running the crash against new builds.

| Pattern | Sync latency | Failure tolerance | WAN suitability | Operational cost |
|---|---|---|---|---|
| Shared directory sync | minutes | high (instances independent) | good with rsync/object store | low |
| LLMP broker | milliseconds | broker is a single point | poor across continents | medium |
| Central scheduler | task-sized (hours) | high if queue is replicated | good | high (needs a service) |

## What scales over a WAN

Corpus sync every few minutes is enough for most targets, as the AFL++ multi-system docs note. That is encouraging for decentralisation: you do not need a tight broker, just a **content-addressed corpus store**.

```
          ┌──────────────── corpus store (content-addressed) ───────────┐
          │  /target/<id>/corpus/<sha256>   /target/<id>/crashes/<sha256> │
          └───────────▲───────────────────────────────▲────────────────┘
                      │ pull new / push new (every 5 min)│
   ┌──────────────────┴──┐                       ┌──────┴──────────────┐
   │ worker A (AFL++ -S) │   ...                 │ worker N (LibAFL)    │
   └─────────────────────┘                       └─────────────────────┘
```

Content addressing (sha256 names, or IPFS CIDs) deduplicates for free and allows any worker to verify what it fetched.

Bandwidth is rarely the bottleneck. Corpora for typical parsers are a few thousand files of a few kilobytes each. The real cost of sync is the *import*: every new input must be executed locally to see whether it adds coverage. A worker that imports 10,000 inputs every five minutes from a noisy peer spends its time re-executing rather than fuzzing. Rate-limit imports and prefer inputs that a trusted verifier has already marked as coverage-increasing.

## Task types to distribute

| Task | Parallel? | Verifiable? |
|---|---|---|
| Fuzzing (exploration) | very | weakly (see Chapter 4) |
| Crash reproduction | yes | strongly (deterministic replay) |
| Corpus minimisation | partly | strongly (coverage recomputation) |
| Regression testing new commits | yes | strongly |
| Symbolic execution of path prefixes | yes (by prefix) | result-dependent |

## Diversity beats raw cores

Running different fuzzers, sanitizers and configurations on different nodes ("ensemble fuzzing", e.g. EnFuzz and later collaborative fuzzing work) usually beats running N copies of one setup. A decentralised network naturally has heterogeneous workers. Turn that into a feature by assigning configurations deliberately.

Useful axes of diversity:

- **Sanitizer:** ASan, UBSan and MSan builds catch different bug classes. MSan needs all code instrumented, so it is the hardest to build.
- **Instrumentation:** CmpLog (`-c`) helps with magic values; laf-intel style comparison splitting helps with multi-byte checks.
- **Power schedule:** AFL++ supports schedules such as `fast`, `explore`, `exploit` and `rare` through `-p`.
- **Dictionary:** a token dictionary (`-x`) for structured formats changes what the fuzzer can reach.

## Worked example

> **Execution status:** Not executed in this book's CI; verify locally. Outputs described as "expected" are what the tools should show, not observed results.

Run a two-machine AFL++ campaign that syncs through a plain directory. You can simulate two machines with two local directories if you only have one host.

```bash
# Build once with CmpLog for one secondary
afl-clang-fast -o target target.c
AFL_LLVM_CMPLOG=1 afl-clang-fast -o target.cmplog target.c

# Machine A: main + one secondary
afl-fuzz -i seeds -o syncA -M main -- ./target @@ &
afl-fuzz -i seeds -o syncA -S s1 -p explore -- ./target @@ &

# Machine B: two secondaries, one using CmpLog
afl-fuzz -i seeds -o syncB -S s2 -c ./target.cmplog -- ./target @@ &
afl-fuzz -i seeds -o syncB -S s3 -p rare -- ./target @@ &
```

Then exchange queues every five minutes. AFL++ imports from any sibling directory under the `-o` path, so copy the other machine's instance directories in:

```bash
while true; do
  rsync -a --include='*/' --include='*/queue/***' --exclude='*' syncB/ syncA/
  rsync -a --include='*/' --include='*/queue/***' --exclude='*' syncA/ syncB/
  sleep 300
done
```

Check progress with `afl-whatsup syncA`. It summarises each instance's executions per second, corpus count and crashes. Compare total edges found against four instances on one machine for the same wall-clock time. You should see roughly comparable coverage, which confirms that minute-level sync is enough.

## Common pitfalls

- **Two main instances in one sync directory.** Each `-M` name must be unique, and you normally want only one.
- **Syncing whole output directories.** Copying `crashes/` and `hangs/` between machines double-counts findings and confuses deduplication.
- **Identical configurations everywhere.** Eight copies of the same schedule waste most of the extra cores.
- **Unbounded imports from untrusted peers.** Import cost dominates; one noisy peer can stall everyone.
- **Ignoring build reproducibility.** If machine B builds with a different compiler, coverage maps differ and imported inputs behave differently.

## Exercises

1. Run AFL++ with one main and three secondaries across two machines using a shared directory synced with `rsync` every 5 minutes. Compare with all four on one machine.
2. Build a minimal content-addressed corpus store with an HTTP API.
3. Design an ensemble assignment: given 20 heterogeneous workers, which configurations go where?
4. Measure import cost: log how many executions per second each instance loses during a sync burst.

## Further reading

- AFL++ parallel fuzzing docs: <https://github.com/AFLplusplus/AFLplusplus/blob/stable/docs/fuzzing_in_depth.md>
- LibAFL book (LLMP, multi-machine): <https://aflplus.plus/libafl-book/>
- Chen et al., "EnFuzz: Ensemble Fuzzing", USENIX Security 2019
- ClusterFuzz architecture: <https://google.github.io/clusterfuzz/architecture/>
- Fioraldi et al., "LibAFL: A Framework to Build Modular and Reusable Fuzzers", ACM CCS 2022
