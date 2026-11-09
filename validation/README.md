# Core archive validation

The JSON reports identify the exact candidate binary and reference client.
`manifest.json` pins the validated Go source commit, both upstream bases, the
canonical RLP input hash, archive settings, and validation limits. Reference RPC
addresses and credentials are not recorded.

- Imported mainnet blocks 1–512 (532 transactions) into a fresh path archive and
  restarted it before comparison. Normal block import validates every state
  root, receipt root, gas total and bloom.
- `rpc-comparison.json`: 102 comparisons at blocks 1, 2, 128, 256 and 512,
  including historical ValidatorSet contract calls, gas estimation, state reads,
  receipts, logs, native tracers and a JavaScript tracer. Only the additional
  `blockTimestamp` field is ignored; seven candidate log timestamps were also
  checked against their block timestamps.
- `modern-replay.json`: five transactions at blocks 19,537,201; 23,000,000;
  30,000,000; 40,000,011; and 40,308,868. Each passed `callTracer` and
  `prestateTracer` comparisons, and each replay matched the reference node's
  original transaction trace. The report records hashes of the captured input
  and normalized output.

Modern replays use touched prestate and block overrides. They do not validate
complete modern block state roots, consensus or state outside the captured
accounts. A full mainnet sync and disk benchmark are still required. Historical
Merkle proofs and validator operation are outside this qualification.

See [reproduction instructions](../README.sentio.md). Local build and regression
checks use Go 1.24 with `GOMAXPROCS=2`, `go test -p=1 -parallel=2`, and linter
concurrency 2.

Local checks passed: geth build, compilation of all test packages, the complete
Core blockchain test suite, VM/state/rawdb/system-contract/Satoshi-consensus
tests, native/JS/logger tracer tests, generated-code and module-tidy checks, and
golangci-lint 2.4.0 (zero issues). Exact commands are in the manifest.
