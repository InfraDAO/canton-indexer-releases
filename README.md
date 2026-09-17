# canton-indexer

Indexing and query engine for [Canton Network](https://www.canton.network/).
It reads a validator's Participant Query Store (PQS), applies declarative
filter manifests (TOML, with a jq-style `where`/`select`), materializes the
matching ledger records into application-ready documents, and serves them
through a document-query HTTP API.

This repository is the public home of the `canton-indexer` binary: releases,
issues and example filters. The source lives in a private repository.

## Install

### Homebrew (macOS, Linux)

```sh
brew install infradao/tap/canton-indexer
```

### Manual download

Grab the binary for your platform from the [Releases page](../../releases):

| Platform      | Asset                            |
|---------------|-----------------------------------|
| macOS ARM     | `canton-indexer-aarch64-darwin`   |
| macOS Intel   | `canton-indexer-x86_64-darwin`    |
| Linux x86_64  | `canton-indexer-x86_64-linux`     |
| Linux ARM64   | `canton-indexer-aarch64-linux`    |

Linux builds are static (musl). Each release also publishes a `SHA256SUMS`
file — verify before running:

```sh
shasum -a 256 -c SHA256SUMS --ignore-missing
chmod +x canton-indexer-*
```

## Upgrade

```sh
brew update && brew upgrade canton-indexer
```

## Examples

[`examples/`](examples/) has filter manifests that track the latest release:

| File | What it does |
|------|---------------|
| [`holding-lifecycle.toml`](examples/holding-lifecycle.toml) | Every create and archive of a CIP-56 token holding owned by the configured party, as a log of what moved. |
| [`my-balance.toml`](examples/my-balance.toml) | One row per unlocked CIP-56 holding owned by the configured party — current balance per instrument. |
| [`my-traffic-balance.toml`](examples/my-traffic-balance.toml) | Running total of synchronizer traffic purchased by a validator, and what it cost. |
| [`report-active-calls.toml`](examples/report-active-calls.toml) | One row per validator liveness heartbeat (`ValidatorLicense_ReportActive`). |
| [`token-transfers.toml`](examples/token-transfers.toml) | Every observable Amulet transfer, with senders, receivers and fees. |
| [`traffic-purchases.toml`](examples/traffic-purchases.toml) | Every synchronizer traffic purchase: member, bytes bought, Amulet burned. |
| [`validator-licenses.toml`](examples/validator-licenses.toml) | Every `ValidatorLicense` contract in its lifecycle-chain form, for folding down to one row per validator. |

## Issues

Bug reports and feature requests go to [Issues](../../issues). Pull requests
are not accepted here — the source is not public.

Please include:

- the version (`canton-indexer --version`)
- your OS/architecture
- how you installed it (Homebrew or a release binary)
- the filter manifest you ran
- stderr logs

## License

[Apache License 2.0](LICENSE).
