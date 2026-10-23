# The Problem: Security Is Compute-Bound and Centralised

Some of the most effective security techniques are brute force in disguise: fuzzing, symbolic execution, large-scale scanning, model inference and simulation. They scale with compute. Today that compute is concentrated in a few places:

- **Google's ClusterFuzz/OSS-Fuzz** runs continuous fuzzing for over a thousand open-source projects, a public good funded by one company.
- **Commercial scanners and audit firms** run private infrastructure.
- **Individual researchers** get a laptop and maybe a rented server.

Meanwhile, decentralised compute networks rent out spare GPUs and CPUs, and blockchains offer programmable payments and public verification. This book asks: **can security compute be organised as an open, decentralised, verifiable market?** And where should it not be?

## Why compute matters so much

Fuzzing campaigns show diminishing but real returns from more CPU time. Coverage typically climbs quickly, then flattens, and the bugs that remain sit behind rare branches that only long campaigns, better seeds or better harnesses reach. Doubling cores does not double bugs found, but for a project that gets zero fuzzing today, the first few hundred core-hours are worth a great deal.

The same pattern holds for other workloads:

| Workload | What scales with compute | Typical unit of work |
|---|---|---|
| Coverage-guided fuzzing | executions per second, campaign length | core-hours per target |
| Differential testing | number of generated test cases | test cases per client pair |
| Static analysis at scale | number of repositories or contracts scanned | scans per artefact |
| Symbolic execution | number of explored paths | solver time per path prefix |
| AI-assisted review | tokens of code read by a model | GPU-seconds per file |

Each row is embarrassingly parallel at some level, which is what makes distribution attractive.

## Why it is hard

1. **Verification:** how do you know a worker actually ran 1,000 CPU-hours of fuzzing rather than sleeping and claiming payment?
2. **Confidentiality:** pre-disclosure targets and found vulnerabilities are sensitive. Untrusted workers could sell 0-days.
3. **Coordination:** fuzzing benefits from corpus sharing, which needs low-latency synchronisation that decentralised networks lack.
4. **Incentives:** pay per hour invites idling, while pay per bug invites duplicates, hoarding and harness gaming.

These four problems interact. Strong verification usually costs confidentiality (a verifier must see the input), and strong confidentiality usually costs verification (you cannot check what you cannot see). TEEs and zero-knowledge proofs try to break that trade-off, each with its own price.

## Design principles used throughout

- **Prefer self-verifying outputs.** A crashing input is its own proof: anyone can replay it. Design tasks so outputs are checkable.
- **Separate public and confidential work.** Open-source targets can use untrusted workers. Private code needs TEEs or trusted operators.
- **Make work deterministic where possible** so it can be spot-checked by re-execution.
- **Pay for verified results, plus a base for verified effort.**
- **The model proposes, the checker decides.** When AI suggests a bug, a harness or a test, a deterministic tool (a replay, a sanitizer, a test runner) confirms it before anyone pays or reports.

## Worked example

> **Execution status:** Not executed in this book's CI; verify locally. Outputs described as "expected" are what the tools should show, not observed results.

This walkthrough shows the asymmetry the whole book relies on: finding a crash is expensive, checking one is cheap. Run it on your own machine.

**Step 1: build a tiny vulnerable target with AFL++.**

```c
// toy.c - crashes when the input starts with "FUZ!"
#include <stdio.h>
#include <string.h>
int main(int argc, char **argv) {
  char buf[8]; FILE *f = fopen(argv[1], "rb");
  size_t n = fread(buf, 1, sizeof buf, f);
  if (n >= 4 && buf[0]=='F' && buf[1]=='U' && buf[2]=='Z' && buf[3]=='!')
    *(volatile int *)0 = 1;   // deliberate crash
  return 0;
}
```

```bash
AFL_USE_ASAN=1 afl-clang-fast -g -o toy toy.c
mkdir in && echo hello > in/seed
afl-fuzz -i in -o out -- ./toy @@
```

AFL++'s status screen shows `saved crashes` rising after some seconds to minutes. Stop it with Ctrl-C once one appears.

**Step 2: verify the claim as an outsider would.**

```bash
for c in out/default/crashes/id:*; do ./toy "$c"; echo "exit=$?"; done
```

Each replay takes milliseconds and ends with an AddressSanitizer report and a non-zero exit code. That is the whole verification.

**Step 3: put numbers on the asymmetry.** Suppose discovery took 300 seconds on one core and replay took 0.01 seconds. The verification-to-discovery ratio is about 1:30,000. Even if a verifier replays every submission from 1,000 workers, it spends a tiny fraction of the network's compute.

Now contrast a time claim: "I fuzzed for 300 seconds." Nothing in `out/` proves it. `fuzzer_stats` is a text file the worker can edit. That gap is the subject of Chapter 4.

## Common pitfalls

- **Paying for effort you cannot observe.** Hourly payments without attestation or spot checks attract idle workers within days.
- **Treating "decentralised" as a goal.** Many projects add a token before they have a single maintainer who wants the findings.
- **Ignoring disclosure.** A network that publishes crashes the moment they are found becomes a 0-day feed for attackers.
- **Running private code on anonymous hardware.** Without a TEE or contractual trust, assume the provider keeps a copy.
- **Measuring cores instead of outcomes.** Coverage, unique crashes and fixed bugs matter; aggregated CPU-hours are a vanity metric.

## Map of the book

| Chapter | Topic |
|---|---|
| 2 | Distributed fuzzing architectures |
| 3 | Decentralised compute markets |
| 4 | Verifiable security work |
| 5 | TEEs for confidential security compute |
| 6 | Crypto-economic coordination |
| 7 | Client-diversity differential testing |
| 8 | Reference architecture |

Related books: **TEE Attack Engineering** covers what can go wrong with enclaves, and **Consensus Protocol Fuzzing** covers fuzzing distributed systems themselves.

## Exercises

1. Estimate the monthly CPU cost of fuzzing 100 open-source projects at 8 cores each on a cloud provider versus a decentralised market.
2. List three security tasks with self-verifying outputs and three without.
3. Describe a cheating strategy for a pay-per-hour fuzzing worker.
4. Repeat the worked example and record discovery time and replay time. Compute the ratio for your machine.

## Further reading

- OSS-Fuzz: <https://github.com/google/oss-fuzz>; ClusterFuzz: <https://github.com/google/clusterfuzz>
- LibAFL distributed fuzzing docs: <https://aflplus.plus/libafl-book/>
- AFL++ documentation: <https://aflplus.plus/docs/>
- Teutsch & Reitwießner, "A Scalable Verification Solution for Blockchains" (TrueBit), 2017
- Böhme, Cadar & Roychoudhury, "Fuzzing: Challenges and Reflections", IEEE Software, 2021
