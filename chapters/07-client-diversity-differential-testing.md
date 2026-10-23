# Client-Diversity Differential Testing

Ethereum deliberately has several independent execution clients (Geth, Nethermind, Besu, Erigon, Reth) and consensus clients (Lighthouse, Prysm, Teku, Nimbus, Lodestar, Grandine). A consensus bug that makes one client disagree can split the chain. This makes **differential testing** extremely valuable, and it suits distributed compute: each test is independent, and the oracle ("do all clients agree?") needs no human.

## Existing tools

- **Hive:** Ethereum's integration test harness that runs clients in containers against test suites.
- **goevmlab** (by Martin Holst Swende): generates EVM state tests, runs them across client EVMs, and compares traces. It has found consensus issues.
- **FuzzyVM** and other EVM fuzzers built around the `statetest` format.
- **Execution spec tests** (`ethereum/execution-spec-tests`) define expected behaviour per fork.

## How differential EVM fuzzing works

```
generator ─► state test JSON (pre-state, tx, fork)
          ─► run on N client EVMs (evm t8n / statetest runners)
          ─► compare per-opcode traces or post-state roots
          ─► divergence? minimise test, report privately to client teams
```

The comparison relies on a shared trace format. EIP-3155 defines a JSON-lines trace with one object per executed opcode (program counter, opcode, gas, stack, depth and so on) plus a summary line with the state root. Most client EVMs can emit it. Comparing traces line by line pinpoints the **first diverging opcode**, which is much more useful to a client developer than "the state roots differ".

Two levels of oracle are common:

| Oracle | Cost | What it catches | Noise |
|---|---|---|---|
| Post-state root only | cheapest | any persistent state difference | low |
| Full per-opcode trace | expensive (large output) | gas or stack differences that do not change state | medium (formatting quirks) |
| Hybrid: roots first, traces on mismatch | cheap on the common path | both | low |

The hybrid is what you want at scale: run every test for roots, and only re-run divergent tests with tracing.

## Distributing it

- Each worker gets a seed range and the client binaries (pinned versions and hashes).
- Results: only divergences are reported, together with the test JSON, which is self-verifying (anyone re-runs it).
- Workers must run **identical client versions**. Mismatched versions create false divergences, so pin by hash and have workers attest the hashes.

Because a divergence is reproducible from its JSON file, this is the ideal workload for untrusted workers: a coordinator re-runs each reported divergence on its own pinned binaries before crediting it. Fake reports cost the coordinator a few seconds each.

## Where AI helps

- **Generating semantically interesting state tests**: prompt a model with an EIP's text and ask for edge cases (gas boundaries, warm/cold access, precompile inputs at limits). New EIPs are where client bugs are most likely.
- **Explaining divergences**: summarise the first diverging opcode and the relevant spec section for the human reporter.

In both cases the model proposes and the tools decide. A generated test counts only if the reference implementation (the execution specs) accepts it as well formed, and a divergence counts only if it reproduces on pinned binaries. Expect a large share of model-generated tests to be malformed or trivially equivalent to existing ones.

## Responsible disclosure

Consensus bugs are critical infrastructure vulnerabilities. Report privately to the client teams and through the Ethereum Foundation bug bounty. Do not publish divergence-inducing tests before fixes are deployed, and do not send divergence-inducing transactions to any public network, including testnets that others depend on.

## Beyond Ethereum

The same approach applies to any multi-client ecosystem: Solana validators (Agave and Firedancer), multiple ZK proof systems verifying the same statement, or several Wasm runtimes.

## Worked example

> **Execution status:** Not executed in this book's CI; verify locally. Outputs described as "expected" are what the tools should show, not observed results.

Run one state test on two EVM implementations and compare traces. You need Geth's `evm` tool (built from `go-ethereum/cmd/evm`) and a second EVM that runs state tests, such as evmone's `evmone-statetest` or a Nethermind/Besu test runner.

**Step 1: get a known-good test.** Take any JSON file from the `ethereum/tests` repository or from a fixtures release of `execution-spec-tests`.

**Step 2: run it on Geth with an EIP-3155 trace.**

```bash
evm --json statetest some_test.json 2> geth.trace
tail -n 1 geth.trace     # summary line with stateRoot
```

The trace goes to stderr as JSON lines; stdout holds a pass/fail summary.

**Step 3: run the second client and diff.** goevmlab automates exactly this, including normalising trace quirks:

```bash
# from a goevmlab checkout; see its README for current flags
go run ./cmd/runtest --geth /path/to/evm --evmone /path/to/evmone-statetest some_test.json
```

Expected output for a correct test: both clients agree, and the tool reports no difference. To see a divergence on purpose, edit a *copy* of the test so one client is given a different fork name than the other; the first trace line where gas costs differ is printed. That is what a real report looks like, minus the forced setup.

**Step 4: fuzz.** goevmlab's fuzzer commands generate random state tests and run them through all configured clients in parallel. Leave it running on a spare machine and check its output directory for saved divergent tests. Every saved test should be re-run manually before you believe it.

## Common pitfalls

- **Comparing mismatched versions or forks.** Most "consensus bugs" found by beginners are configuration differences.
- **Diffing raw traces without normalisation.** Clients format numbers, memory and refunds differently; use a tool that normalises.
- **Tracing everything.** Full traces for millions of tests fill disks fast. Compare roots first.
- **Publishing a test before the fix ships.** A divergence-inducing test is an exploit for chain splits.
- **Ignoring precompiles and new opcodes.** Bugs cluster where code is newest.

## Exercises

1. Run goevmlab's fuzzer locally against two client EVMs for an hour.
2. Use a model to generate 50 state tests targeting a recent EIP's opcodes. Validate each with the reference implementation. How many are valid?
3. Design the worker attestation for pinned client binaries.
4. Write a script that compares only state roots across clients and escalates to full traces on mismatch.

## Further reading

- Hive: <https://github.com/ethereum/hive>
- goevmlab: <https://github.com/holiman/goevmlab>
- Execution spec tests: <https://github.com/ethereum/execution-spec-tests>
- EIP-3155, EVM trace specification: <https://eips.ethereum.org/EIPS/eip-3155>
- evmone: <https://github.com/ethereum/evmone>
- Ethereum bug bounty: <https://ethereum.org/en/bug-bounty/>
