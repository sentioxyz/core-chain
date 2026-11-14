# Core RPC/archive with Geth 1.16

This fork integrates Geth **v1.16.9** with Core **v1.0.27**. It retains Core's
Satoshi consensus, network protocol, system transactions, fork schedule, rewards,
RPC interfaces and tracing hooks. Geth's path database, flat-state history and
historical index provide archive reads without retaining every historical trie.

The intended use is an RPC/archive node. Validator operation has not been
qualified. A mainnet full sync has completed; historical query comparisons are
recorded in [the validation reports](validation/README.md). Disk measurements are
observations from that run, not a controlled storage benchmark.

## Build and run

Go 1.24 and a C compiler are required:

```sh
go build -o build/bin/geth ./cmd/geth
build/bin/geth --mainnet \
  --datadir /data/core-path-archive \
  --state.scheme=path --syncmode=full --gcmode=archive \
  --history.state=0 --history.transactions=0 \
  --http --http.addr=127.0.0.1 --http.api=eth,debug,web3,net
```

Use a **fresh data directory**. Full synchronization from genesis is required for
complete historical state. A snap-synced state does not contain earlier state
history, and replacing a hash-archive binary does not convert its database or
reclaim its disk space. Keep the existing node available while the new node syncs.

`--history.state=0` retains all reverse state differences. `--gcmode=archive`
enables historical indexing; `eth_syncing.stateIndexRemaining` reports incomplete
indexing. Historical reads become available after their index is ready.

`--history.transactions=0` retains the complete transaction lookup index. Log
queries retain Core's bloom-bit index and read historical receipts; Geth's
filtermaps index and its `--history.logs` option are not integrated in this port.
This does not limit the retained range of historical logs.

Historical balances, nonces, code, storage, `eth_call`, transaction traces, block
traces and call traces use the historical state reader. Logs include the
additional Geth `blockTimestamp` field. Historical `eth_getProof` needs historical
trie nodes and is not provided by the 1.16 flat-state archive. Old hash-to-path
conversion cannot manufacture the required history and is explicitly rejected.
Core's remote/no-trie verification and persisted diff synchronization modes are
not supported by this RPC build. Their legacy diff-generation tests have been
removed along with the old storage path; normal local block verification remains
enabled. Core's networking and Engine integration stay on its existing protocol
interfaces rather than enabling Ethereum's newer peer-sync protocol.

## Reproduce the official-client comparison

The reference must be an official Core archive node with the `debug` namespace.
The candidate and reference URLs are supplied locally and are never embedded in
the comparison report.

```sh
python3 scripts/verify-core-rpc.py --reference "$REFERENCE_RPC" \
  --export-blocks 512 --output /tmp/core-first-512.rlp

build/bin/geth --mainnet --datadir /tmp/core-path-check \
  --state.scheme=path --syncmode=full --gcmode=archive \
  --history.state=0 --history.transactions=0 \
  import /tmp/core-first-512.rlp

build/bin/geth --mainnet --datadir /tmp/core-path-check \
  --state.scheme=path --syncmode=full --gcmode=archive \
  --history.state=0 --history.transactions=0 \
  --maxpeers=0 --nodiscover --nat=none --port=0 --ipcdisable \
  --http --http.addr=127.0.0.1 --http.port=18546 --http.api=eth,debug,web3,net

python3 scripts/verify-core-rpc.py --reference "$REFERENCE_RPC" \
  --candidate http://127.0.0.1:18546 --blocks 1,2,128,256,512 \
  --output /tmp/core-rpc-comparison.json
```

Import validates every block's state root, receipt root, gas and bloom. Restarting
before the RPC comparison verifies persisted history, including states older
than the live 128-block window. The comparator checks block/transaction data,
logs, receipts, state reads, historical `getValidators()` calls, gas estimation, and `callTracer` /
`prestateTracer` results, plus a bounded JavaScript opcode tracer. It tolerates only the added `blockTimestamp` field and
records input/output SHA-256 digests. RPC errors and null results fail the check.

`--replay-transactions HASH...` captures an official transaction's touched
prestate and original block context. Both clients replay it through
`debug_traceCall`, and both tracer results must also match the official original
transaction. This extends execution coverage across forks without importing tens
of millions of blocks. It does **not** validate those blocks' full state roots;
contracts using `BLOCKHASH` require a full replay if their captured fixture does
not reproduce the original trace.

The initial validation replayed mainnet blocks 1–512 and passed 102 RPC checks
after restart. Five further transactions from blocks 19,537,201; 23,000,000;
30,000,000; 40,000,011; and 40,308,868 passed both tracer comparisons, including
the original-transaction checks. Subsequent mainnet full sync and larger historical
comparisons are recorded in `validation/full-history-audit.json`; RPC and tracing
comparisons remain sampled rather than exhaustive.
