# Optimizing PTF-dependent tests on SONiC T2

## Background

SONiC T2 systems contain multiple frontend line cards and route processors. Many tests already use frontend-DUT enumeration fixtures such as `enum_rand_one_per_hwsku_frontend_hostname`, so non-PTF tests can be split across line cards and run in parallel with `--target_hostname`.

PTF-dependent tests are harder to parallelize because a testbed commonly has only one PTF server. Running the same PTF-dependent test module concurrently for multiple line cards would make those workers share packet injection, packet receive queues, PTF adapter state, and sometimes DUT counters.

The primary goal for this investigation is the best risk-adjusted runtime improvement with minimal flakiness.

## Files inspected

- `tests/ip/test_ip_packet.py`
- `tests/drop_packets/test_drop_counters.py`
- `tests/drop_packets/drop_packets.py`
- `tests/conftest.py`
- `tests/common/plugins/ptfadapter/__init__.py`
- `tests/common/plugins/ptfadapter/ptfadapter.py`
- `tests/common/plugins/ptfadapter/templates/ptf_nn_agent.conf.ptf.j2`
- `tests/common/devices/ptf.py`
- `tests/common/fixtures/ptfhost_utils.py`

## Current repo behavior

### Frontend DUT selection

`pytest_generate_tests` in `tests/conftest.py` expands tests that request `enum_rand_one_per_hwsku_frontend_hostname`. For T2, this can select frontend line cards. The repo also has `--target_hostname`, `--parallel_state_file`, `--is_parallel_leader`, `--parallel_followers`, and `--parallel_mode` options. In parallel mode, a worker can be trimmed to one target hostname.

This is useful for non-PTF tests because each worker can focus on a different line card.

### PTF host and adapter model

`ptfhosts` can discover multiple PTF hosts from testbed metadata, but the legacy `ptfhost` fixture returns `ptfhosts[0]`.

The `ptfadapter` fixture is module-scoped. It starts `ptf_nn_agent` on each PTF host, builds `PtfAgent` objects, and creates one `PtfTestAdapter`.

`PtfTestAdapter` initializes global PTF state:

- Updates `ptf.config`.
- Creates `ptf.dataplane_instance = DataPlane(config=ptf.config)`.
- Replaces dataplane `send` and `poll` behavior.
- Attaches helper methods such as `drain` and `clear_masks`.

This means multiple PTF test executions inside the same Python worker are not naturally isolated. Even multiple pytest workers using the same physical PTF host can conflict unless the PTF server side is partitioned.

### PTF agent startup

The current supervisor template defines a single program:

```ini
[program:ptf_nn_agent]
command=/usr/bin/python /opt/ptf_nn_agent.py --device-socket {{ device_num }}@tcp://0.0.0.0:{{ ptf_nn_port }} ...
```

Supporting multiple PTF agents on one PTF server would require unique supervisor program names, unique ports, unique logs, and a disjoint interface map per agent.

## Observations from example tests

### `tests/ip/test_ip_packet.py`

This file uses `enum_rand_one_per_hwsku_frontend_hostname`, `ptfadapter`, and a class-scoped `common_param` fixture. Each test:

1. Selects a frontend DUT.
2. Chooses ingress and egress interfaces from minigraph data.
3. Clears port and RIF counters on the selected DUT.
4. Flushes the PTF dataplane.
5. Sends packets from one PTF port.
6. Counts matching packets on expected PTF output ports.
7. Reads counters and validates packet forwarding or drops.

The test cases are good candidates for controlled multi-LC orchestration because most counter operations are scoped to the selected DUT. The major shared resource is the PTF dataplane.

### `tests/drop_packets/test_drop_counters.py`

This file also uses `enum_rand_one_per_hwsku_frontend_hostname` and `ptfadapter`, but it has broader shared-state behavior:

- `enable_counters` runs on all frontend nodes.
- `base_verification` clears SONiC counters and RIF counters on all frontend nodes.
- Verification often checks that no unrelated L2/L3 drops occurred on all frontend nodes.
- ACL setup and teardown can apply across all frontend nodes.

This file is not safe for naive per-LC parallel execution. Concurrent workers could clear each other's counters or fail "no drops elsewhere" checks due to traffic from another worker.

## Why naive per-LC parallel PTF execution is risky

Running one pytest worker per line card against the same PTF server would introduce several hazards:

- `ptfadapter.dataplane.flush()` can clear packets needed by another worker.
- `count_matched_packets_all_ports` and `verify_no_packet_any` can observe packets from another worker.
- `ptf.config` and `ptf.dataplane_instance` are process-global.
- `ptf_nn_agent` is currently one supervisor program per PTF host.
- Tests that clear counters globally can erase another worker's measurement window.
- Negative assertions, especially "no packet on any sniff port", are highly sensitive to unrelated concurrent traffic.

Because of these hazards, the first implementation should avoid pretending the current PTF model is parallel-safe.

## Optimization options

### Option 1: Single PTF orchestrator with batched multi-LC execution

Keep one `ptfadapter`, but refactor selected high-value tests so one pytest invocation coordinates traffic for multiple line cards.

Possible shape:

1. Build per-LC test parameters.
2. Clear counters on each LC in a controlled window.
3. Use the single PTF adapter to send per-LC traffic in a deterministic sequence.
4. Collect counters from LCs in parallel.
5. Validate results per LC.

Benefits:

- Lowest infrastructure cost.
- Lowest flake risk.
- Works with the current single PTF server model.
- Good first step for `tests/ip/test_ip_packet.py`.

Limitations:

- Packet send and sniff operations are still serialized or carefully scheduled.
- Runtime improvement is bounded by PTF traffic time.
- Tests with global counter clearing, such as `drop_counters`, need more refactoring before this is safe.

### Option 2: Multiple PTF agents or containers on the same PTF server

Partition one PTF server into multiple independent PTF shards. Each shard owns a disjoint set of PTF interfaces and runs a separate `ptf_nn_agent` instance.

Needed changes:

- Add a mapping from line card to PTF port subset.
- Generate multiple supervisor programs, for example `ptf_nn_agent_lc0`, `ptf_nn_agent_lc1`.
- Allocate unique TCP ports and log files.
- Teach `ptfadapter` to create an adapter for a selected PTF shard.
- Ensure one Python process only owns one shard-specific dataplane.
- Add pytest markers or scheduling rules so only shard-safe tests run this way.

Benefits:

- Real per-LC parallel traffic without multiple physical PTF servers.
- Reuses current `--target_hostname` parallel-run model.

Risks:

- More fixture and infrastructure complexity.
- Requires strong interface ownership guarantees.
- Tests that mutate PTF host networking may still need a global lock.
- Tests with global DUT counter behavior remain unsafe until refactored.

### Option 3: Multiple PTF hosts or NIC-level partitioning

Provide each line card with a physically or logically isolated PTF resource.

Examples:

- One PTF host per line card.
- One container or network namespace per line card with dedicated NICs.
- SR-IOV or NIC partitioning, if supported by the server and topology.

Benefits:

- Cleanest isolation model.
- Best long-term scalability.
- Least ambiguity around packet queues and interface ownership.

Risks:

- Highest infrastructure cost.
- Requires testbed metadata updates.
- Many tests still assume the legacy `ptfhost` fixture and would need migration to selected PTF resources.

### Option 4: Remove PTF dependency from selected tests

For tests that only need a counter signal or a control-plane signal, replace PTF traffic with traffic generated from a DUT, fanout, neighbor VM, or another existing endpoint.

Benefits:

- Eliminates the PTF bottleneck for those tests.
- Can make tests easier to parallelize with existing per-LC worker model.

Risks:

- Not suitable for tests that validate exact packet formatting, checksums, or negative dataplane behavior.
- Coverage can change if packet generation path is not equivalent to PTF.

## Recommended direction

Use a phased hybrid approach.

### Phase 1: Classify PTF tests

Add an inventory of PTF-dependent tests and classify them by parallel safety:

- Send-only or counter-only.
- Send plus expected packet receive.
- Negative sniffing.
- PTF-host-mutating.
- Global-counter-mutating.

Only the first two groups should be considered for early parallelization.

### Phase 2: Refactor `test_ip_packet.py` first

`test_ip_packet.py` is a good first candidate because most DUT-side state is selected-DUT scoped. Convert the highest-cost cases to a single orchestrated test path that gathers parameters for multiple frontend LCs, sends traffic through the one PTF adapter in a controlled sequence, then reads DUT counters in parallel.

This should improve runtime without requiring new PTF infrastructure.

### Phase 3: Add a PTF shard model

Introduce a resource model for PTF shards:

```text
target_hostname -> ptf resource -> interface subset -> ptf_nn_agent instance
```

Use this model only for tests marked as PTF-shard-safe. Unsafe tests should remain serial or use a global PTF lock.

### Phase 4: Rework `drop_counters` separately

Before `drop_counters` can run per-LC in parallel, it needs scoped counter behavior:

- Clear counters only for the selected DUT, ASIC, namespace, and interfaces.
- Avoid "no drops on all frontend nodes" checks during concurrent PTF traffic.
- Ensure ACL setup and teardown do not overlap in conflicting ways.

Until then, keep `drop_counters` in the serial or orchestrated bucket.

## Open questions for future work

- How much of the T2 runtime is spent in PTF send or sniff operations versus setup, waits, and counter polling?
- Which PTF-dependent tests dominate T2 wall-clock time?
- Does each line card have a naturally disjoint PTF port range in all target T2 testbeds?
- Are there tests that mutate PTF host network configuration and therefore must always take a global PTF lock?
- Can the scheduler support a "PTF shard safe" marker and route tests accordingly?
- Should `ptfhost` remain backward-compatible while new code uses a selected `ptf_resource` fixture?

## Suggested next steps

1. Collect runtime data for PTF-dependent T2 tests and rank by cost.
2. Build a small classification table for the top 10 to 20 PTF-dependent tests.
3. Prototype Phase 2 on `tests/ip/test_ip_packet.py`.
4. Measure runtime and flake rate before changing the PTF infrastructure.
5. If Phase 2 is not enough, prototype multiple `ptf_nn_agent` instances on one PTF server with disjoint interface maps.

