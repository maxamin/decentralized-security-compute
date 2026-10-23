# Decentralised Compute Markets

Several networks sell compute from independent providers with blockchain-based payments. They differ in trust model, workload format and maturity.

| Network | Model | Workload | Notes for security use |
|---|---|---|---|
| Akash | reverse auction for container leases | Docker / SDL manifests | runs ordinary containers; good fit for fuzz workers |
| Golem | requestor/provider task market | VM images, task API | task-oriented; suits batch reproduction jobs |
| Fluence | decentralised serverless/compute | functions, VMs | newer compute offering; check current status |
| io.net, Render and other GPU markets | GPU rental | containers | for model inference / AI-assisted analysis |
| Bacalhau | compute-over-data jobs | Docker / WASM | data-locality model; good for scanning large datasets |

Treat this table as a starting point. Capabilities, pricing and even business models change quickly, so check current docs before building on any of them.

## How the markets actually work

Despite different branding, most of these networks share a lifecycle:

1. **Describe** the workload: a container image, resource requirements (CPU, memory, storage, GPU) and a price ceiling.
2. **Match**: providers bid (Akash's reverse auction), or a matching service assigns work (Golem's market negotiation).
3. **Lease**: payment flows from an escrow while the workload runs. On Akash, a deployment holds a deposit and is billed per block.
4. **Run**: the provider pulls the image and starts it, usually in Kubernetes or a VM.
5. **Close**: the requester closes the lease, or it ends when escrow runs out.

Notice what the protocol does *not* do: it does not verify that your container did useful work. It pays for a lease being up. Everything about "did the fuzzer actually run" is your problem, which is why later chapters exist.

## Fit assessment for security workloads

**Good fits:**
- Fuzzing open-source targets (no confidentiality, outputs mostly self-verifying).
- Crash reproduction and regression across many configurations.
- Large-scale *public* data analysis: scanning all verified contracts with Slither, or clustering bytecode.
- Inference for public-code analysis with open-weight models.

**Poor fits:**
- Private audit code (unless TEEs, Chapter 5).
- Anything that holds keys or signs transactions.
- Latency-sensitive coordination (live MEV defence).

## Threats from providers

- **Lazy providers:** take payment, do little.
- **Leaky providers:** copy crashes and sell or exploit them before disclosure.
- **Malicious providers:** return poisoned corpora (e.g. inputs crafted to slow other workers) or fake crashes (detected by replay).
- **Sybils:** one operator posing as many providers to defeat redundancy.

There is also a threat in the other direction. **Malicious requesters** can submit a "fuzz job" that is really a crypto-miner, a scanner aimed at third parties or a container escape attempt. Providers defend with resource limits and egress filtering. If you run a provider, you are hosting strangers' code.

## Mitigations

- Run only open-source targets on untrusted providers. Embargo nothing there.
- **Replay every crash** on a trusted verifier before reward.
- **Validate corpora** by recomputing coverage on a trusted node before merging (and cap per-input size and execution time).
- Use **redundancy with randomisation**: assign the same seed slice to two random providers occasionally and compare coverage.
- **Deny outbound network** except to the corpus store. A fuzz worker has no reason to reach anything else.
- **Pin images by digest** (`image@sha256:...`), not by tag, so a provider cannot swap the binary silently and you can prove what was meant to run.

## Worked example

> **Execution status:** Not executed in this book's CI; verify locally. Outputs described as "expected" are what the tools should show, not observed results.

Package an AFL++ worker as a container, test it locally with the same constraints you will ask of a provider, then validate its output on a trusted node.

**Step 1: a worker image.** The official `aflplusplus/aflplusplus` image contains the toolchain.

```dockerfile
FROM aflplusplus/aflplusplus
COPY target.c /src/target.c
RUN afl-clang-fast -o /src/target /src/target.c
COPY run.sh /run.sh
ENTRYPOINT ["/bin/bash", "/run.sh"]
```

`run.sh` fuzzes for a fixed time and then packs its queue:

```bash
#!/bin/bash
mkdir -p /seeds && echo seed > /seeds/a
timeout 3600 afl-fuzz -i /seeds -o /out -S "w$RANDOM" -- /src/target @@
tar czf /out/result.tgz -C /out .
```

**Step 2: run it locally under provider-like limits.**

```bash
docker build -t fuzzworker .
docker run --rm --cpus=2 --memory=2g --network=none -v "$PWD/out:/out" fuzzworker
```

`--network=none` proves the worker does not need the internet.

**Step 3: the deployment manifest.** On Akash, the same image becomes an SDL file. Push the image to a registry first and reference it by digest.

```yaml
version: "2.0"
services:
  fuzz:
    image: registry.example/fuzzworker@sha256:<digest>
profiles:
  compute:
    fuzz:
      resources:
        cpu: { units: 2 }
        memory: { size: 2Gi }
        storage: { size: 5Gi }
  placement:
    anywhere:
      pricing:
        fuzz: { denom: uakt, amount: 1000 }
deployment:
  fuzz:
    anywhere: { profile: fuzz, count: 1 }
```

Deploy to Akash's sandbox or testnet first, following the current CLI instructions in the Akash docs; field names and denominations evolve.

**Step 4: validate returned corpora on a trusted node.** Never merge raw submissions. Use `afl-cmin` with tight limits so slow or oversized inputs are dropped:

```bash
mkdir submitted && tar xzf result.tgz -C submitted
afl-cmin -i submitted/w*/queue -o accepted -t 200 -m 512 -- ./target @@
ls accepted | wc -l
```

Only inputs that execute within 200 ms and contribute distinct coverage survive. The count of accepted files is a rough measure of what the provider contributed.

## Common pitfalls

- **Referencing images by mutable tag.** `:latest` lets anyone with registry access change what runs.
- **Writing results only to the provider's disk.** When the lease ends, so does the data. Push to your store continuously.
- **Budgeting by list price only.** Bid prices change, and failed deployments still cost escrow fees and your time.
- **Trusting provider-reported statistics.** `fuzzer_stats` and logs are written by the party being paid.
- **Assuming providers are diverse.** Several "independent" providers may be one data centre or one operator.

## Exercises

1. Write an Akash SDL manifest for an AFL++ worker that syncs with the corpus store from Chapter 2.
2. Design a corpus-poisoning attack and the validation that stops it.
3. Estimate the cost difference between a cloud provider and one decentralised market for 1,000 core-hours (use current public prices).
4. Run the worker with `--network=none` and confirm it still produces a corpus. What breaks if you also make the root filesystem read-only?

## Further reading

- Akash docs: <https://akash.network/docs/>
- Golem docs: <https://docs.golem.network>
- Bacalhau: <https://docs.bacalhau.org>
- Fluence: <https://fluence.network>
- AFL++ Docker image: <https://hub.docker.com/r/aflplusplus/aflplusplus>
