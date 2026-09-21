# cinder

Developer CLI for [Canton Network](https://www.canton.network/). It reads a
validator's Participant Query Store (PQS), applies declarative filter
manifests (TOML, with a jq-style `where`/`select`), and writes the matching
ledger records as application-ready documents — the tool for authoring
filters and exploring what a validator holds.

This repository is the public home of the `cinder` binary: releases, issues
and example filters. The source lives in a private repository.

## Install

### Homebrew (macOS, Linux)

```sh
brew install infradao/tap/cinder
```

### Manual download

Grab the binary for your platform from the [Releases page](../../releases):

| Platform      | Asset                    |
|---------------|--------------------------|
| macOS ARM     | `cinder-aarch64-darwin`  |
| macOS Intel   | `cinder-x86_64-darwin`   |
| Linux x86_64  | `cinder-x86_64-linux`    |
| Linux ARM64   | `cinder-aarch64-linux`   |

Linux builds are static (musl). Each release also publishes a `SHA256SUMS`
file — verify before running:

```sh
shasum -a 256 -c SHA256SUMS --ignore-missing
chmod +x cinder-*
```

## Upgrade

```sh
brew update && brew upgrade cinder
```

## Usage

Writing a filter is a loop of three commands:

```sh
# what the store holds: templates, interfaces, choices
cinder types [--package splice-amulet] [--json]

# the records a `where` selects, as a filter sees them
cinder preview --source lifecycle|exercises [--where '<jq>'] [--range S:E|S|:E] [--limit N] [--save my-filter.toml]

# a finished filter over an offset window, as NDJSON documents
cinder run my-filter.toml [--from N] [--to M|+K]
```

`preview` writes each matching record as one NDJSON line, so the paths a
`select` needs are there to read. `--range` takes `S:E` (inclusive), `S` to
read from S on, or `:E` to read up to E; `--limit` defaults to 10 and `0`
removes the cap. `--save` leaves the manifest of that preview — `name` from
the file stem, `source`, `where` — with an empty `[select]` to fill in; an
existing file is never overwritten, and `cinder run` refuses the manifest
until `select` has a column.

## Configuration

`cinder` reads the first of `--config PATH`, `./cinder.toml`,
`$XDG_CONFIG_HOME/cinder/config.toml` (`~/.config/cinder/config.toml` when
`XDG_CONFIG_HOME` is unset). Command-line flags override environment
variables, which override the file:

```toml
pqs_url = "postgres://user:password@localhost:5432/pqs"
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

- the version (`cinder --version`)
- your OS/architecture
- how you installed it (Homebrew or a release binary)
- the filter manifest you ran
- stderr logs

## License

[Apache License 2.0](LICENSE).
