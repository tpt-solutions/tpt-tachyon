# tpt-tachyon — Project Todo

Owner: TPT Solutions · Licence: MIT OR Apache-2.0 · Source of truth: `spec.txt` (Draft v0.1)

Legend: `[ ]` todo · `[x]` done · `[~]` in progress (edit by hand) · (§N) = spec section

**Policies**
- Dependencies must be MIT, BSD, ISC, Unlicense/CC0, or dual `MIT OR Apache-2.0`. No Apache-2.0-only, no copyleft.
- Stable Rust, Windows + Linux. Core crates `no_std`-friendly.
- Spec-first: no fast-path feature lands without a spec entry and an invariant/test.
- Never claim wireless-RAM or DDR-equivalent latency (§4, §30).

---

## Phase -1 — Project setup

### Repo and licensing
- [ ] `git init`, set default branch, add `.gitignore` (Rust, editor, build output)
- [ ] Create `LICENSE-MIT` (copyright TPT Solutions)
- [ ] Create `LICENSE-APACHE` (Apache-2.0, copyright TPT Solutions)
- [ ] Add licence section to README ("MIT OR Apache-2.0, at your option")
- [ ] Add `license = "MIT OR Apache-2.0"` to every `Cargo.toml`
- [ ] Add SPDX header (`// SPDX-License-Identifier: MIT OR Apache-2.0`) to every source file; add a CI check
- [ ] Add contribution-licensing notice (contributions dual-licensed by default)
- [ ] Add `deny.toml` for cargo-deny: allow MIT/BSD/ISC/Unlicense/CC0/`MIT OR Apache-2.0`; deny Apache-2.0-only and copyleft
- [ ] Audit every dependency's licence expression before adding it

### Workspace
- [ ] Create Cargo workspace root with `crates/`, `tools/`, `formal/`, `docs/`, `configs/`
- [ ] Pin toolchain (`rust-toolchain.toml`, stable) and set MSRV
- [ ] Add `rustfmt.toml`, clippy config, `#![forbid(unsafe_code)]` policy for core crates
- [ ] Scaffold crates: `tachyon-core`, `tachyon-link`, `tachyon-transaction`, `tachyon-sim`
- [ ] Scaffold tools: `tachyonctl`, `tachyon-bench`, `tachyon-trace`
- [ ] Scaffold later crates as empty placeholders: `tachyon-profile-accel`, `tachyon-profile-storage`
- [ ] Mark core crates `no_std` + `alloc` feature flags; add a `no_std` CI build

### Docs scaffolding
- [ ] Convert `spec.txt` → `SPEC.md`; keep a versioned changelog
- [ ] Write `README.md` (tagline, positioning §32, status, build instructions)
- [ ] Write `ROADMAP.md` from §29
- [ ] Create empty `docs/` pages: architecture, latency-budget, reliability-modes, lane-bonding, host-integration

### CI
- [ ] CI matrix: Windows + Linux, stable Rust
- [ ] CI jobs: build, test, fmt, clippy, cargo-deny, `no_std` build, doc build, SPDX check
- [ ] Add TLC (TLA+) model-check job once models exist

---

## Phase 0 — Specification and simulator

### Spec decisions to lock (§33)
- [ ] Decide and document: v0 = fixed-size binary cells, credit-based flow control, virtual channels, simulated lane bonding
- [ ] Define the byte-exact cell header layout (§12.1), endianness, alignment and reserved fields
- [ ] Reserve header flag bits and session fields for security (auth tag, key epoch, nonce/replay counter)
- [ ] Define cell sizes 64/128/256/512 B and per-session negotiation rules
- [ ] Define payload-CRC optional rules per reliability mode
- [ ] Define version numbering and compatibility rules
- [ ] Write the "Fast Path Contract" (§22.3) as a concrete session-setup structure
- [ ] Define the ordering model (§17) precisely, including FENCE domains and posted-write rules
- [ ] Define reset behaviour and error transitions
- [ ] Define the latency-budget accounting model (§15.4, §26)

### `tachyon-core`
- [ ] `version.rs`: protocol version type and compatibility check
- [ ] `opcode.rs`: v0 opcodes (PING, TRAIN, CREDIT, READ_REQ, READ_CPL, WRITE_POSTED, WRITE_NONPOSTED, WRITE_CPL, DMA_DESC, STREAM, FENCE, DOORBELL, INTERRUPT, RESET, ERROR)
- [ ] `opcode.rs`: reserve "later" opcodes (ATOMIC_CAS/ADD, PREFETCH, CACHE_HINT, MEM_PAGE_REQ) as invalid in v0
- [ ] `opcode.rs`: opcode validity table per cell type and profile
- [ ] `error.rs`: error enum and wire error codes
- [ ] `cell.rs`: header struct, zero-copy encode/decode, header CRC
- [ ] `cell.rs`: support all four cell sizes, with payload_len bounds checks
- [ ] `cell.rs`: reject ambiguous or malformed headers (reserved bits, bad lengths, bad version)
- [ ] Session contract types (version, opcodes, cell sizes, max payload, reliability per VC, lane count/rate, retry limit, reorder window, encryption mode, clock mode, profile)
- [ ] Security mode enum (plain / auth / encrypted / isolated); only plain and auth implemented in v0
- [ ] Unit tests and golden byte vectors for every opcode and cell size
- [ ] proptest: encode/decode round-trip; decoder never panics on arbitrary bytes
- [ ] Fuzz target for the cell decoder

### `tachyon-link`
- [ ] `crc.rs`: header CRC32 and payload CRC (choose polynomials, document, test vectors)
- [ ] `credit.rs`: per-VC credit accounting and CREDIT cell handling
- [ ] `scheduler.rs`: superframe model (sync, control, downlink, uplink, retry, train, telemetry)
- [ ] `scheduler.rs`: slot assignment, guard bands, no-contention deterministic mode
- [ ] Virtual channel table (VC 0–5 defaults from §18) with isolation
- [ ] `lane.rs`: lane striping of cells across the active lane set
- [ ] `lane.rs`: bounded reorder window and per-lane skew/jitter/error/reorder-distance tracking
- [ ] `lane.rs`: adaptive active-lane mask and degradation/recovery rules
- [ ] `lane.rs`: control-cell duplication across lanes (R4)
- [ ] `retry.rs`: bounded retry window and deadline expiry → ERROR
- [ ] `retry.rs`: duplicate suppression (retried cell never delivered twice)
- [ ] `ordering.rs`: per-VC in-order delivery, FENCE across selected channels
- [ ] Reliability modes, implemented in this order: R1 → R3 → R0 → R2 → R4 (§30 risk 3)
- [ ] Optional FEC hook (cost modelled only; no real codec in v0)
- [ ] Clock model: master clock, sync cells, drift estimation
- [ ] Link training state machine (TRAIN, PING, link up/down, RESET)
- [ ] Authenticated control plane (auth mode) using a placeholder keyed MAC; real crypto deferred to Phase 2
- [ ] Unit tests and proptests per module

### `tachyon-transaction`
- [ ] `read.rs`: READ_REQ / READ_CPL tracking by transaction_id
- [ ] `write.rs`: posted and non-posted writes, WRITE_CPL
- [ ] `dma.rs`: DMA_DESC handling and chunking into cells
- [ ] `fence.rs`: FENCE semantics
- [ ] `stream.rs`: STREAM payloads
- [ ] DOORBELL and INTERRUPT handling
- [ ] Transaction timeouts and error completion
- [ ] Tests: ordering rules, posted-write vs read behaviour, fence behaviour

### `tachyon-sim`
- [ ] Deterministic simulation core with seeded RNG and virtual time (reproducible runs)
- [ ] `phy.rs`: PHY abstraction trait and `sim` PHY profile
- [ ] `channel.rs`: bandwidth, base latency, jitter, BER, cell loss
- [ ] `channel.rs`: lane skew, beam interruption, clock drift, serialization delay
- [ ] Queue depth, credit starvation, retry cost, FEC cost models
- [ ] YAML config loader (e.g. `desk-dock-60ghz`); choose a maintained MIT/dual-licensed YAML crate and audit its licence
- [ ] `configs/four-lane-dock.yaml` (from §23.1)
- [ ] `workload.rs`: AI tensor transfer
- [ ] `workload.rs`: NVMe-like random read
- [ ] `workload.rs`: GPU command queue
- [ ] `workload.rs`: video stream
- [ ] `workload.rs`: memory page prefetch
- [ ] `workload.rs`: link obstruction, lane failure, retry storm, thermal throttling
- [ ] `metrics.rs`: median/P99 latency, throughput, retries, ordering violations, credit stalls
- [ ] `metrics.rs`: per-component latency breakdown (queue, slot, serialization, propagation, PHY, MAC, retry, reorder)
- [ ] Trace output format (draft; finalised in Phase 1)
- [ ] `tachyon-sim run <config>` CLI entry point
- [ ] Latency dashboard (static report or simple viewer from sim output)

### Formal specification (`formal/`)
- [ ] `formal/cell_schema/`: machine-readable cell schema (single source for docs, code and tests)
- [ ] `formal/state_machine/`: TLA+ spec for link state, retry, credits, reorder window
- [ ] `formal/state_machine/`: TLA+ spec for ordering, FENCE, reset, error transitions
- [ ] `formal/invariants/`: no data cell sent without credits
- [ ] `formal/invariants/`: reliable ordered channel delivers in order or reports failure
- [ ] `formal/invariants/`: retried cell is never delivered twice
- [ ] `formal/invariants/`: fenced transaction cannot pass earlier incomplete transactions
- [ ] `formal/invariants/`: control cells remain valid during data-lane degradation
- [ ] `formal/invariants/`: VC isolation (bulk traffic cannot starve VC 0)
- [ ] Run TLC and record state-space bounds and results
- [ ] Mirror each TLA+ invariant as a Rust property test
- [ ] Trace-conformance test: sim traces replayed against the TLA+ model (stretch)

### Phase 0 exit criteria (§29)
- [ ] Simulate a 4-lane deterministic link
- [ ] Inject loss and jitter
- [ ] Ordering and credit invariants proven in TLC and passing in property tests
- [ ] Compare R0/R1/R2/R3 modes with published results
- [ ] Benchmark traces committed
- [ ] Published latency budget for the simulator

---

## Phase 1 — Software prototype

### Runtime and transport
- [ ] `tachyond` userspace daemon (session setup, link controller, scheduler loop)
- [ ] `eth-bridge` PHY over UDP (std sockets, Windows + Linux)
- [ ] Raw Ethernet transport (pcap / AF_PACKET) as a later sub-task; document licensing of any capture library
- [ ] Session negotiation over the real transport (fast-path contract exchange)
- [ ] Time sync and slot timing on a non-real-time OS; document jitter limits honestly
- [ ] Lane mapping onto multiple UDP flows / sockets
- [ ] Plain and auth security modes end-to-end
- [ ] Graceful shutdown, reset and reconnect

### Profiles
- [ ] `tachyon-stream` profile
- [ ] `tachyon-dma` profile (submit descriptor, read/write remote buffer, signal completion)
- [ ] Simulated device attachment (virtual device endpoint on the remote side)
- [ ] Remote buffer read/write API in a userspace runtime library

### Tools
- [ ] `tachyonctl`: discover, pair (stub), train, link status, link lanes, profile attach
- [ ] `tachyonctl`: trace (`--vc`, `--show-retries`)
- [ ] `tachyon-bench`: latency (`--payload`, `--duration`)
- [ ] `tachyon-bench`: throughput
- [ ] `tachyon-bench`: dma (`--size`, `--reliability`)
- [ ] `tachyon-trace`: capture, filter and inspect traces
- [ ] Final trace file format spec (versioned, binary or compact)
- [ ] Link telemetry: latency, jitter, BER, loss, lane skew, retry rate, thermal state, beam quality
- [ ] Reproducible benchmark harness with pinned configs and recorded results

### MVP demo (§31)
- [ ] `tachyon-sim run configs/four-lane-dock.yaml` works
- [ ] `tachyon-bench dma --size 256MB --reliability R2` prints the expected report
- [ ] `tachyon-bench latency --payload 256B --duration 60s` works
- [ ] `tachyonctl trace --show-retries` works
- [ ] Report fields: workload, payload, lanes, cell size, reliability, median/P99 latency, throughput, retries, ordering violations, credit stalls

### Phase 1 exit criteria (§29)
- [ ] Stable virtual link over Ethernet/UDP
- [ ] Reproducible benchmarks
- [ ] Remote buffer read/write works
- [ ] Simulated device attachment works
- [ ] Tag `v0.1.0`; publish crates only after the licence audit passes

---

## Phase 2 — Hardware-adjacent prototype (milestones)

### Link layer for FPGA
- [ ] Choose HDL approach (Verilog/VHDL/Amaranth/SpinalHDL/Chisel) and check tool licences
- [ ] FPGA-friendly link layer (no allocation, bounded state, fixed timing)
- [ ] Fixed cell parser in HDL with cell-format conformance tests against the Rust model
- [ ] Credit engine
- [ ] Retry engine with bounded window
- [ ] Deterministic scheduler
- [ ] Hardware timestamping and clock sync
- [ ] Cut-through forwarding (§7.4)
- [ ] Link layer runs in FPGA simulation, cross-checked against the Rust reference
- [ ] Bounded latency under load demonstrated
- [ ] Lane-failure recovery demonstrated
- [ ] No host network stack in the fast path

### Security (deferred from P0–1)
- [ ] Choose AEAD (ChaCha20-Poly1305 and/or AES-GCM; verify crate licences)
- [ ] Session key establishment and key epochs
- [ ] Replay protection
- [ ] Inline encryption mode and isolated mode
- [ ] Pairing flows: QR, NFC, USB-C provisioning, physical button, manufacturer certificate, local admin approval
- [ ] Device identity and capability authorization
- [ ] Signed firmware metadata
- [ ] Security review / threat model document

---

## Phase 3 — Wireless PHY prototype (milestones)
- [ ] Select certified 60 GHz module(s) or an optical test link
- [ ] Directional antennas and mounting rig
- [ ] `60ghz` PHY adapter behind the PHY abstraction trait
- [ ] Beam training implementation
- [ ] Per-lane measurement (latency, jitter, BER, skew)
- [ ] Real latency metrics vs the simulator; calibrate sim models
- [ ] Compare against ordinary wireless networking for fixed transactions
- [ ] Stable operation at dock distance (1–50 cm)
- [ ] Document obstruction behaviour
- [ ] Regulatory check for New Zealand (bands, power/EIRP, indoor/outdoor, EMC, safety); record findings (§28)
- [ ] Decide whether experimental PHY work (sub-THz / optical) is justified

---

## Phase 4 — Accelerator dock profile
- [ ] `tachyon-profile-accel`: buffer allocation, command queue, kernel/model upload
- [ ] Doorbells, completion interrupts, wait-completion, fetch-result
- [ ] DMA buffer movement over real transport
- [ ] Runtime integration (explicit offload API first; PCIe bridge later)
- [ ] `tachyon-profile-storage`: block read/write, flush, trim, queues
- [ ] Offload a simple compute workload
- [ ] Transfer tensors or buffers end-to-end
- [ ] Demonstrate practical benefit over normal network RPC
- [ ] Evaluate PCIe endpoint / BAR / interrupt design (§19.2) and GPU feasibility (§30 risk 4)

---

## Phase 5 — Custom interconnect research
- [ ] Sub-THz (100–300 GHz) experiments
- [ ] Optical / infrared link experiments
- [ ] Sub-microsecond link tests
- [ ] Near-field / package-scale exploration
- [ ] CXL-like transaction exploration
- [ ] `tachyon-memext` experimental profile (page fetch/evict, prefetch, consistency fences); no DDR-latency claims
- [ ] Add later opcodes (atomics, PREFETCH, CACHE_HINT, MEM_PAGE_REQ) to the spec with formal models

---

## Cross-cutting workstreams

### Docs and public positioning
- [ ] Public description and tagline in README (§32)
- [ ] Explicit non-goals and "not faster than light" disclaimer on the front page
- [ ] `docs/architecture.md`
- [ ] `docs/latency-budget.md` (published budget; separate sim / prototype / ASIC targets)
- [ ] `docs/reliability-modes.md`
- [ ] `docs/lane-bonding.md`
- [ ] `docs/host-integration.md`
- [ ] Performance-target table (T0–T4) kept in sync with measured results
- [ ] API docs published (docs.rs or equivalent)
- [ ] Docs site (static generator; licence-check the toolchain)
- [ ] Release notes for every tagged version

### Hardware procurement and lab setup (phase-gated)
- [ ] Phase 2: FPGA dev board(s) with enough transceivers and PCIe support
- [ ] Phase 2: HDL toolchain licences reviewed
- [ ] Phase 3: certified 60 GHz modules and evaluation kits
- [ ] Phase 3: directional antennas, mounts, alignment fixtures
- [ ] Phase 3: measurement gear (spectrum analyser, scope or timestamping hardware)
- [ ] Phase 3: RF-safe lab space and safety procedure
- [ ] Phase 5: sub-THz / optical components and test equipment

### Risk watchlist (§30)
- [ ] PHY hardware difficulty: re-assess at each phase gate
- [ ] Unrealistic latency claims: every published number tagged sim / prototype / ASIC
- [ ] Reliability-mode complexity: keep R1 + R3 stable before R2/R4
- [ ] PCIe/GPU integration: FPGA/accelerator profile first
- [ ] Regulatory limits: stay in approved bands; sub-THz is lab-only
