# RealC1PHER/epic — testing fork of Epic Cash v4.1.2

> **Status: testing.** This fork exists so the Epic Cash team can see exactly what we
> changed and why. It is **not** an official release and should not be used in production
> without your own review. We will report results on
> [EpicCash/epic PR #161](https://github.com/EpicCash/epic/pull/161).

We run a mainnet solo-mining node on Epic v4.1.2. After the upgrade the node fell behind on
almost every new block ("body_sync" flapping), our miners worked on stale jobs, and about
1 in 4 of our found blocks was orphaned. This fork contains the changes we are testing to fix
that. **No consensus or validation rules are changed**; only how a node chooses peers for
missing blocks, and how newly mined blocks are announced and accepted.

## Branches and tags

Every branch starts from the official tag `v4.1.2` and adds exactly the commits listed.

| Tag | Branch | Adds on top of the previous build |
|---|---|---|
| `v4.1.2-fz1` | `frenziez/v4.1.2-sync-fix` | `f6aaa90`: the upstream fix from PR #161 (keep `no_sync` during in-flight validation; request the compact block during `BodySync`) |
| `v4.1.2-fz2` | `frenziez/v4.1.2-fz2` | `1877807`: body_sync peer selection (see 2 below) |
| `v4.1.2-fz3` | `frenziez/v4.1.2-fz3` (default) | `c5489de`: compact block propagation (see 1 below) |

Full diff against the official release:
[`v4.1.2...frenziez/v4.1.2-fz3`](https://github.com/RealC1PHER/epic/compare/v4.1.2...frenziez/v4.1.2-fz3)

## 1. Pushed compact blocks from the miner were dropped (fz3)

**Root cause.** `servers/src/common/adapters.rs::compact_block_received` gained a
"strict header first" filter in commit `840c8b3` (first released in v4.0.0). It only accepts a
compact block from a peer that previously sent that block's header (`recently_seen_headers`);
otherwise it returns `Ok(true)` without processing or logging. But in `block_accepted`, the node
that **mined** a block (`Options::MINE`) broadcasts the **compact block directly, without a
header**. So every freshly mined block pushed by its finder is silently dropped by all 4.x direct
peers. It only spreads once nodes discover the header via the periodic sync header probe and fetch
the body via body_sync one or two ticks later. On our node that was a consistent **17–19 s** between
header and body for every block, even though the same block had already arrived as a compact block
seconds earlier. v3.6.0 has no such filter.

**Change (`c5489de`).**
- When a node mines a block: `broadcast_header` first, then `broadcast_compact_block`. Unpatched
  4.x peers record the header and then accept the compact block.
- `compact_block_received` accepts a pushed compact block from any peer again (as v3.6.0 did).
  Full header and block validation still applies. A debug line
  `Accepting pushed compact block … without a prior header` replaces the silent drop.

## 2. body_sync asked peers that may not have the block, and disconnected them (fz2)

**Root cause.** v3.6.0 requested missing bodies from `more_work_peers()`: peers advertising more
total difficulty than us, shuffled, re-asking after 6 s, without disconnecting. Since v4.0.0
`body_sync` uses `outgoing_connected_peers()` in fixed list order, regardless of whether a peer
has the block. It waits 10 s, then disconnects the peer and tries the next. In v4.1.x
`more_work_peers` no longer exists. During a one-block gap that meant several 10 s timeouts on
peers that were behind themselves, plus constant peer churn.

**Change (`1877807`, only `servers/src/epic/sync/body_sync.rs`).**
- Per missing block, prefer peers whose advertised height ≥ the block height (shuffled), then the rest.
- Skip a peer that already timed out on that block; retry after 6 s (was 10 s).
- Disconnect a peer only after 3 consecutive timeouts (was 1).
- New tests: `block_request_prefers_peer_that_advertises_the_block`,
  `retry_goes_to_a_peer_that_has_not_timed_out_on_the_block`,
  `single_timeout_keeps_peer_and_marks_it_tried`.

## Results so far (testing period)

Mainnet node in Europe, outbound peers only. "Behind" = the node had a block's header but not yet
its body (stalls ≥ 5 s).

| Build | Window | Stalls/h | Behind/h | Synced |
|---|---|---|---|---|
| v4.0.3 | Nov 8–11 | 3.7 | 0.7 min | 98.8 % |
| v4.1.2 | Nov 12 – 13 17:49 CET | 29 | 8.9 min | 85.1 % |
| v4.1.2 (after a large pool's stratum went down) | Nov 13 evening | 18 | 16.2 min | 72.9 % |
| v4.1.2-fz1 (+ PR #161) | ~1 h | 33 | 29 min | 51 % |
| v4.1.2-fz2 | ~40 min | ~25 | ~8 min | ~87 %, every stall 17–19 s |
| v4.1.2-fz3 | first ~35 min | 3 stalls (≤ 19 s) | — | ~98 %, header→body 0.1–0.7 s |

- Block-download timeouts dropped from 30–120 per hour to almost none.
- The first block we found on fz3 (height 3796795) was broadcast header-first, the network built
  on it within seconds, and it has passed 10 confirmations on the main chain.
- These windows are short and network conditions changed a lot this week. **This is our testing
  period to make sure the fix is real.** We'll update this file and PR #161 with longer-term numbers.

## Building

Same as upstream (see `doc/build.md`); the Rust toolchain is pinned in `rust-toolchain.toml`.

```sh
git clone https://github.com/RealC1PHER/epic.git
cd epic
git checkout v4.1.2-fz3
cargo build --release --locked
cargo test --locked -p epic_servers --lib -- epic::sync common::   # 68 tests
```

The binary still reports `epic 4.1.2` / user agent `MW/Epic 4.1.2`; only the commits above differ.

## License

Unchanged from upstream (Apache-2.0). Copyright of the original code remains with the Epic Cash
and Grin developers.
