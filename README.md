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

## Demo

`cinder types` lists what the store holds; the interface name is picked from
its output:

![cinder types](docs/demo/types.gif)

`cinder preview` selects records with that name in a `where`, narrows the
range, and writes them as JSON:

![cinder preview](docs/demo/preview-where.gif)

An owner from that output narrows the `where` further, and `--save` writes the
manifest:

![cinder preview --save](docs/demo/preview-save.gif)

Fill in `select`, then `cinder run` emits the documents:

![cinder run](docs/demo/run.gif)

## Configuration

`cinder` reads the first of `--config PATH`, `./cinder.toml`,
`$XDG_CONFIG_HOME/cinder/config.toml` (`~/.config/cinder/config.toml` when
`XDG_CONFIG_HOME` is unset). Command-line flags override environment
variables, which override the file:

```toml
pqs_url = "postgres://user:password@localhost:5432/pqs"
```

## Examples

[`docs/filter-format.md`](docs/filter-format.md) describes the manifest: its
keys, the records a filter reads, the `where` grammar, the `select`
expressions and the documents written.

[`examples/`](examples/) has filter manifests that track the latest release.
Each names a DevNet party in its `where`; its header says what to replace.

| File | What it emits |
|------|---------------|
| [`ans-entries.toml`](examples/ans-entries.toml) | The ANS entries whose user is the named party, at each entry's create and at its archive. |
| [`ans-operations.toml`](examples/ans-operations.toml) | The exercises an ANS entry goes through: the named party's requests and payments, and every expiry PQS holds. |
| [`holding-lifecycle.toml`](examples/holding-lifecycle.toml) | The token-standard V2 (CIP-112) holdings in the named party's default account, at each holding's create and at its archive. |
| [`my-balance.toml`](examples/my-balance.toml) | The unlocked token-standard V2 (CIP-112) holdings in the named party's default account, for a balance per instrument. |
| [`my-traffic-balance.toml`](examples/my-traffic-balance.toml) | The `MemberTraffic` contracts of the named participant: synchronizer traffic purchased and what it cost, as running totals. |
| [`report-active-calls.toml`](examples/report-active-calls.toml) | Each `ValidatorLicense_ReportActive` exercise, the validator liveness heartbeat. |
| [`reward-collections.toml`](examples/reward-collections.toml) | Each `AmuletRules_Transfer` the named party sent that credited a reward, with the amount credited per reward type. |
| [`token-standard-transfers.toml`](examples/token-standard-transfers.toml) | Each token-standard V2 (CIP-112) `EventLog_HoldingsChange` the DSO reports on the named party's default account: holdings consumed and created, and the transfer legs. |
| [`token-transfers.toml`](examples/token-transfers.toml) | Each `AmuletRules_Transfer` exercise whose argument the party sees: sender, receivers, amounts and holding fees. |
| [`traffic-purchases.toml`](examples/traffic-purchases.toml) | Each `AmuletRules_BuyMemberTraffic` exercise whose argument the party sees: member, traffic bought, Amulet paid. |
| [`validator-licenses.toml`](examples/validator-licenses.toml) | Every `ValidatorLicense` contract at its create and at its archive, for folding to one row per validator. |
| [`validator-reward-claims.toml`](examples/validator-reward-claims.toml) | The settlements of validator reward coupons: each claim by the named party, and every expiry PQS holds. |
| [`validator-rewards.toml`](examples/validator-rewards.toml) | The validator reward coupons whose user is the named party, at each coupon's create and at its archive. |

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
