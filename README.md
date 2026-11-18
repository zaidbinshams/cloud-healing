# Self-Healing Cloud Infrastructure

Online reinforcement learning for fault remediation on a live Kubernetes cluster.

An agent watches a running microservice application every 20 seconds and picks one of twelve repair actions: do nothing, restart a service, scale it, or restore it to its known-good template. The application is Google's Online Boutique on K3s. The agent is trained and evaluated on the real cluster, not in a simulator. Faults are injected with Chaos Mesh, through the Kubernetes API and with load surges. The controller runs on a second machine connected over Wi-Fi, so the control loop also has to cope with an unreliable network link.

The agent is a masked discrete Soft Actor-Critic with prioritized experience replay (PER-SAC). It is warm-started from runbook demonstrations using behaviour cloning and a conservative (CQL) critic, then trained online. It is compared with three baselines, under an evaluation protocol that was pre-registered before training:
- default Kubernetes;
- Kubernetes with a Horizontal Pod Autoscaler (HPA);
- a scripted SRE runbook.

## Contributions

To our knowledge, no earlier work combines all of the following:
1. a closed-loop policy over **discrete repair actions** (restart, scale, restore);
2. **learned online on the live system** it repairs, with no simulator or digital twin;
3. driven by **telemetry alone**, with no fault diagnosis given as input;
4. with **hard guards** on every action (masks, a one-action-in-flight lock, replica bounds);
5. compared with native Kubernetes, HPA and a runbook under a **pre-registered** protocol.

We do **not** claim to be the first to apply RL to a real cluster, the first to use RL for remediation, a new warm-start method or a new guarded action interface.

## Testbed

| Part | What was used |
|---|---|
| Machine A (cluster) | One laptop: 16 CPUs, about 15 GB RAM, Ubuntu, swap off. Single-node K3s v1.36.4+k3s1 (Traefik disabled), Chaos Mesh, Prometheus (cAdvisor, 5 s scrape), metrics-server |
| Machine B (controller) | Windows with WSL2, Python 3.11, CPU-only PyTorch. Runs the environment, the agent, the Locust load generator, the injector and the evaluation. Timestamp authority for every measurement |
| Link | A Wi-Fi hotspot. A–B clock offset stayed within about 6–16 ms (gate: 200 ms) |
| Application | Online Boutique v0.10.7: 11 deployments after removing its load generator. The agent manages 4: `frontend`, `cartservice`, `currencyservice`, `productcatalogservice` |
| Load | Locust, 30 users (about 34 requests/s), 0.5–1.5 s think time, one TCP connection per request |
| SLA | Server-side p95 latency of the frontend ≤ 70 ms and failed requests ≤ 1 % |

Every deviation from the original design was approved before the data it affects: the wireless link, the relaxed preflight thresholds, the CPU limits, the measurement point and the load model. Each one is explained where it is set: `[override]` constants in `scripts/`, `CONTRACT-CHANGE` comments in `config/contract.yaml`.

### Faults

| Fault | Injection | Correct repair |
|---|---|---|
| F1 bad deploy | `EXTRA_LATENCY` of 300 ms, 600 ms or 1 s added to `productcatalogservice` | RESTORE `productcatalogservice` |
| F2 hot pod | Chaos Mesh CPU stress (1 or 2 workers) on `cartservice` or `currencyservice` | RESTART the target |
| F3 scaled to zero | `cartservice` or `currencyservice` set to 0 replicas | RESTORE (or SCALE_UP) the target |
| F4 traffic surge | Load × 2.5 or × 3.0 | SCALE_UP `frontend` |
| NULL | Nothing (fault-free control) | NOOP |

The agent never sees which fault was injected, its target, its severity or the injection time.

### Agent, actions and guards

- **Observation:** 36 features. Five are global (latency, failure ratio, request rate and their one-tick changes). Seven are per managed service (CPU, throttling, memory, readiness, replicas, restarts, rollout recency). Three describe the controller (lock held, ticks since the last action, stale telemetry).
- **Actions (12):**
  - NOOP;
  - RESTART × 4 (one per managed service);
  - SCALE_UP `frontend`, `cartservice` or `currencyservice`, and SCALE_DOWN `frontend`;
  - RESTORE `productcatalogservice`, `cartservice` or `currencyservice` to the golden template and base replicas.
- **Reward:** the negative of three terms: SLA violation, action cost and extra replicas. Range [−1.09, 0]. No MTTR shaping.
- **Guards:**
  - An action mask allows only actions that respect the replica limits (frontend 3, cart 2, currency 2, productcatalog 1) and the current state.
  - A global lock allows only one action in flight until its rollout completes.
  - The executor re-checks the bounds before every call.
  - The agent's credential is limited to one namespace and cannot delete pods. The executor accepts only the four managed deployments.
- **Loop:** a fixed 20 s tick. Telemetry is collected concurrently under a 3 s deadline, and actions are dispatched asynchronously. Missing telemetry is imputed by fixed rules, and a stale next state is never stored for training.

## Results

**Main evaluation.** 50 scheduled episodes per policy: 10 for each of four fault types, plus 10 without a fault. Policies run interleaved on each schedule entry. MTTR is the median over fault episodes; episodes that did not recover count as 360 s.

| Policy | Faults recovered | Median MTTR (s) | Actions in fault-free episodes |
|---|---|---|---|
| Kubernetes (default) | 0 / 40 | 360 | 0 / 10 |
| Kubernetes + HPA | 10 / 40 (all surges) | 360 | 0 / 10 |
| Scripted runbook | 38 / 40 | 79.6 | 0 / 10 |
| **PER-SAC, warm start** | **35 / 40** | **79.1** | **0 / 10** |
| PER-SAC, cold start (100 episodes) | 15 / 40 | 360 | 10 / 10 |

- **Against Kubernetes and HPA:** the warm agent recovers faster on F1–F3 (Holm-corrected p ≤ 0.03), and faster than Kubernetes on F4. **HPA was faster on traffic surges** (median 59.2 vs 59.7 s, p = 0.031), because it scales on CPU before the SLA is breached.
- **Against the runbook:** non-inferior on MTTR. The 95 % CI of the median difference is [−20.0, +0.9] s, inside the pre-registered 20 s margin. It is not superior (p = 0.41), and the SLA penalty is a tie.
- **Safety:**
  - no action in 0 of 10 fault-free evaluation episodes and 0 of 30 fault-free training episodes;
  - 0 mask violations, 0 executor refusals, 0 execution errors;
  - median decision latency 0.33 s (p95 0.89 s).
- **Warm start:** it is decisive. Without it, the agent recovers 15/40 and acts in every fault-free episode, always repeating RESTORE `cartservice`.
- **Second seed:** a second training seed recovers 35/40 and is also non-inferior to the runbook, which ran alongside it as a drift check. The runbook showed no drift between days (37/40 vs 38/40 recovered; medians 79.7 vs 79.6 s, p = 0.70).
- **Degraded link:** with an emulated degraded link (netem: +150 ms delay and 1 % loss on control-plane traffic), the medians barely moved (78.2 vs 78.5 s). With 16 episodes per policy, non-inferiority was **not established** (CI [−20.1, +20.2] s against a 20 s margin).
- **One fault/target pair was never learned:** F2 on `currencyservice` (0/5 in both seeds), even when latency clearly breached the SLA (119–237 ms).

`python -m eval.analyze` recomputes every number above, with confidence intervals and test statistics, from the raw episode logs (see [Analysis and figures](#analysis-and-figures)).

## Limitations

- **Scale:** one single-node cluster on a laptop, one application, and a wireless link instead of a wired LAN.
- **Seeds and runs:**
  - two training seeds for the warm agent, sharing the same warm-start data;
  - one cold-start run of 100 episodes, half the warm agent's online budget;
  - 10 episodes per fault type.
- **Not tested:**
  - The uniform-replay ablation was not run, so we make no claim about prioritized replay itself.
  - Neither policy used SCALE_UP `cartservice` or `currencyservice`, so whether gRPC connection reuse limits their effect remains open.
- **Resolution:** MTTR is measured in 20 s steps (one tick).
- **Runbook bias:** we wrote the runbook ourselves, knowing the fault classes, which favours it.
- **Shared machine:** Machine A is a personal laptop that also runs other work. Every evaluation session started on AC power, with other workloads stopped and a passing 10-minute steady check.

## Measurement lessons

Making the loop trustworthy on this hardware took as much work as the agent:
- **Where latency is measured decides what the controller can see.** Client-side p99 tracked the Wi-Fi round-trip time (r = 0.95). The SLA is therefore measured at the server, from the frontend's own request logs. It uses p95 because, at about 900 requests per tick, p99 carries about 1.6 times the sampling noise of p95 (CV 0.145 vs 0.09).
- **Keep-alive connections pin load to one pod.** With them, a frontend scale-out left the new pods idle and surges were incurable by any policy. The load generator now opens one connection per request.
- **Limits can make a fault incurable.** At the upstream 200m CPU limit, `productcatalogservice` throttled during surges in a way no catalog action could relieve. Several CPU limits were raised (`config/contract.yaml`).
- **Metric freshness.** With the default kubelet housekeeping interval, the newest cAdvisor sample was a median 12 s (p95 21 s) old when queried, and 8–46 % of live-edge `rate(...[30s])` queries came back empty. The interval is now 5 s (`k8s/k3s/kubelet-housekeeping.sh`).
- **Clock sync across Windows, WSL2 and Linux.** B follows A's NTP server through the Windows time service, with tuned update settings; with the defaults, B overshot A by about 0.6 s.

## Repository layout

| Path | Contents |
|---|---|
| `env/` | Gymnasium environment for the real cluster: tick clock, telemetry and imputation, action catalog and executor, fault injector, reward, reset to the golden state |
| `agents/` | Masked discrete SAC, PER buffer, warm start, scripted runbook, training and policy runners |
| `eval/` | Evaluation runner, metrics, pre-registered statistics, analysis, figures, SLA-measurement analysis |
| `eval/schedules/` | Fixed evaluation schedule (`eval_v1.json`, seed 2026) |
| `config/` | Locked parameters (`contract.yaml`), calibration, golden deployment snapshot, cluster addresses, run settings |
| `k8s/` | K3s install and kubelet tuning, Online Boutique manifests, RBAC, Prometheus, HPA baseline |
| `chaos/` | Chaos Mesh install and a smoke-test manifest |
| `locust/` | Load generator with the custom `/tick` endpoint |
| `scripts/` | Preflight checks, calibration, fault smoke tests, session start, watchdog, experiment chains, latency diagnostics, clock sync |
| `tests/` | Unit tests for the pure logic, with fixtures recorded from the real cluster (no cluster needed) |

Not in the repository (gitignored): `data/` (raw episode logs, checkpoints and diagnostics, kept on Machine B), `config/kube/` (cluster credentials) and `figures/` (generated plots).

## Requirements

- **Machine A (cluster):** Linux, about 12 GB RAM or more, swap off. Runs K3s v1.36.4+k3s1, Online Boutique v0.10.7, Chaos Mesh and Prometheus.
- **Machine B (controller):** Linux or WSL2 with Python 3.11. Runs the environment, the agent, Locust and the evaluation. CPU only; no GPU is used.
- **Link:** the design assumes a wired LAN (RTT p99 < 2 ms). The reported results used a Wi-Fi hotspot; the relaxed thresholds are the `[override]` constants in `scripts/preflight.py`.
- **Clocks:** the machines need synchronised clocks; B is the timestamp authority. A serves NTP (`k8s/k3s/chrony-server.sh`). B follows it with chrony on Linux (`scripts/chrony-client.sh`) or through the Windows time service under WSL2 (`scripts/w32time-follow-a.ps1`).

## Setup

**Machine A**, once:

```bash
A_IP=<static LAN IP> bash k8s/k3s/install-server.sh   # swap off, chrony, pinned K3s
sudo bash k8s/k3s/chrony-server.sh                    # NTP server for Machine B
sudo bash k8s/k3s/kubelet-housekeeping.sh             # 5 s cAdvisor housekeeping
sudo bash k8s/k3s/kubelet-logsize.sh                  # larger container logs (server-side SLA reads)
```

**Machine B**, from the repository root:

```bash
python3.11 -m venv .venv && source .venv/bin/activate
pip install torch --index-url https://download.pytorch.org/whl/cpu
pip install -r requirements.txt
source config/cluster.env          # edit the IPs and paths in this file first
```

Copy the admin kubeconfig from A to `config/kube/admin.kubeconfig`, then build the cluster state:

```bash
# Online Boutique golden manifest (generated from upstream + contract; never hand-edit golden.yaml)
python k8s/boutique/build_golden.py --upstream k8s/boutique/upstream/kubernetes-manifests.yaml \
    --contract config/contract.yaml --out k8s/boutique/golden.yaml
kubectl --kubeconfig "$KUBE_ADMIN" apply -f k8s/boutique/namespace.yaml -f k8s/boutique/golden.yaml

# least-privilege credentials for the agent and the fault injector
kubectl --kubeconfig "$KUBE_ADMIN" apply -f k8s/rbac/agent-rbac.yaml -f k8s/rbac/controller-rbac.yaml
bash k8s/rbac/make-kubeconfig.sh agent-sa "$KUBE_AGENT"
bash k8s/rbac/make-kubeconfig.sh controller-sa "$KUBE_CTRL"

# golden snapshot (the RESTORE / reset source of truth)
python -m scripts.snapshot_golden

# observability, fault injection, load
kubectl --kubeconfig "$KUBE_ADMIN" apply -f k8s/prometheus/
bash chaos/install-chaos-mesh.sh
bash locust/run_locust.sh --users 30 --background

# checks and calibration
python -m scripts.preflight --stage m1
python -m scripts.preflight --stage m2
python -m scripts.calibrate --minutes 30          # writes config/calibration.json
```

`python -m scripts.fault_smoke --fault F1 --severity 600ms` (and the F2–F4 variants listed at the top of that file) checks that each fault is injected, detected and cured by its correct repair.

## Running the experiments

Every session starts with:

```bash
bash scripts/session_start.sh
```

This checks the cluster, the clock skew and steady-state latency against the frozen calibration.

The main steps are below. In the chain scripts each runs under `scripts/watchdog.sh` and can be resumed after an interruption.

```bash
# runbook demonstrations for the warm start (120 episodes, 25 % random valid actions)
python -m agents.run_policy --policy runbook --episodes 120 --epsilon 0.25 --seed 0 --out data/m4/warmstart_runbook

# training (200 episodes on the live cluster)
python -m agents.train --run sac_per_s0 --episodes 200 --seed 0 --warmstart data/transitions/warmstart_runbook
python -m agents.train --run sac_per_cold_s0 --episodes 100 --seed 0 --cold

# evaluation on the fixed schedule, policies interleaved per entry
python -m eval.run_eval --schedule eval/schedules/eval_v1.json --out data/eval/eval_v1_main \
  --policy k8s_default --policy k8s_hpa --policy runbook \
  --policy sac_per=data/m4/sac_per_s0/ckpt/final.pt --policy sac_per_cold=data/m4/sac_per_cold_s0/ckpt/final.pt
```

`scripts/chain_m4.sh` and `scripts/chain_seed1.sh` run the full sequence used for the reported results, including the degraded-link subset (`scripts/edge_stage.sh`, with `scripts/netem.sh` and `scripts/install_netem_sudo.sh`).

One episode takes 6–8 minutes, so a 200-episode training run takes roughly a day of cluster time.

## Analysis and figures

```bash
# pre-registered analysis of the evaluation and training runs
python -m eval.analyze --main data/eval/eval_v1_main --edge data/eval/eval_v1_edge \
  --train sac_per=data/m4/sac_per_s0 --train sac_per_cold=data/m4/sac_per_cold_s0 \
  --seed1 sac_per_s1=data/eval/eval_v1_seed1=data/m4/sac_per_s1 --out data/eval/analysis.json

# evaluation figures (Kaplan–Meier, MTTR by fault, training curves, edge and seed-1 comparisons)
python -m eval.plots --analysis data/eval/analysis.json --main data/eval/eval_v1_main \
  --edge data/eval/eval_v1_edge --train sac_per=data/m4/sac_per_s0 --out figures/evaluation

# SLA-measurement study (client vs server latency, estimator noise, detectability)
python -m eval.paperb_analysis && python -m eval.paperb_figures --out figures/measurement

# training progress report
python -m scripts.training_report --run sac_per_s0 --run sac_per_cold_s0
```

All of these read the raw logs in `data/`, which are not in the repository.

## Tests

```bash
pytest -q
ruff check .
```

The unit tests cover the pure parts of the system:
- normalisation and imputation;
- the reward and the action mask;
- the runbook rules;
- the SAC and PER mathematics;
- the statistics.

They use fixtures recorded from the real cluster and need no cluster access. Tests that touch the cluster are marked `cluster` and are not run by default.

## Notes on reproducing

The code, configuration, evaluation schedule and analysis are complete, but cluster behaviour is not deterministic: the same seed on different hardware will not give identical numbers. Expect results of the same shape, not the same values.

## Citing

TBA (after publication).

## License

Licensed under the [Apache License, Version 2.0](LICENSE). Copyright 2026 Pratyush, Zaid and contributors.

`k8s/boutique/upstream/kubernetes-manifests.yaml` comes from Google's [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) (Copyright Google LLC) and is also licensed under Apache 2.0; `k8s/boutique/golden.yaml` is generated from it.
