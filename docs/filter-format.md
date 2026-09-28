# Filter format (v1)

A filter is a TOML file that tells cinder which ledger records you want and
what to keep from each one. `where` picks the **records** from your
validator's PQS (Participant Query Store), and `select` turns each record
into a JSON **document**.

## Quickstart

Point cinder at your PQS in `./cinder.toml`:

```toml
pqs_url = "postgres://user:pass@host:5432/pqs"
```

Then find a template, look at its records, and run a filter:

```sh
# 1. find a template name
cinder types --templates --package splice-amulet
# 2. look at its records
cinder preview --source lifecycle \
  --where '.template_fqn == "splice-amulet:Splice.ValidatorLicense:ValidatorLicense"'
# 3. write documents, one JSON line each
cinder run examples/validator-licenses.toml
```

This is the filter the last command runs, `examples/validator-licenses.toml`,
shown here without its header comment:

```toml
name = "validator_licenses"
source = "lifecycle"
version = 1
where = '.template_fqn == "splice-amulet:Splice.ValidatorLicense:ValidatorLicense"'
[select]
validator = 'party(.payload.validator)'
sponsor = 'party(.payload.sponsor)'
dso = 'party(.payload.dso)'
version = '.payload.metadata.version'
contact_point = '.payload.metadata.contactPoint'
last_active_at = 'time(.payload.lastActiveAt)'
metadata_updated_at = 'time(.payload.metadata.lastUpdatedAt)'
```

| If you want to                       | Use                                                              |
|--------------------------------------|------------------------------------------------------------------|
| turn a preview into a filter         | Add `--save my.toml` to the preview, then fill in `[select]`. The file loads once `[select]` has at least one column. |
| see more than 10 records in a preview | `--limit N`. `0` removes the limit.                             |
| preview a smaller range              | `--range S:E`, `--range S` or `--range :E`. By default a preview reads every offset PQS holds. |
| start a run at another offset        | `--from OFFSET`. By default a run starts at the oldest offset PQS holds. An offset before that one is an error. |
| run further                          | `--to OFFSET` or `--to +N`. By default a run stops 1000 offsets after its start. A `--to` past the latest offset is fine: the run stops at the latest one. |
| resume a run                         | `--cursor TOKEN`. A run that completes logs the cursor (`resume from this cursor`). The run then reads to the latest offset, or to `--to OFFSET`. `--to +N` does not work with a cursor. |
| use another config file              | `--config PATH`. Without it cinder reads `./cinder.toml`, and if that does not exist `$XDG_CONFIG_HOME/cinder/config.toml` (`~/.config/cinder/config.toml` when `XDG_CONFIG_HOME` is unset or empty). |
| run an example on your own validator | Replace the DevNet ids in it: the party `infradao-devnet-1::1220…` with one your validator hosts, `PAR::…` with your participant id, and `DSO::1220…` with the DSO party of your network (the `dso` column of the documents that step 3 writes shows it). |

Tips:

- If a preview shows records but a run writes nothing, compare the offsets.
  A preview reads every offset, while a run stops 1000 offsets after its
  start unless you pass `--to`. The run logs the range it reads.
- Logs go to stderr and documents go to stdout.
- Write each expression in single quotes (`'…'`). That way the double quotes
  inside jq need no escaping.
- A party id is a name, then `::`, then a fingerprint that begins with `1220`.
- Every file in `examples/` starts with a comment that says what it indexes.

## Keys

| key            | required | value                                                  | default  |
|----------------|----------|--------------------------------------------------------|----------|
| `name`         | yes      | `[a-z][a-z0-9_]*`, at most 63 bytes                    |          |
| `source`       | yes      | `"lifecycle"` or `"exercises"`                         |          |
| `version`      | yes      | `1`                                                    |          |
| `where`        | no       | An expression in the `where` grammar. Read the warning there before you leave it out of a `lifecycle` filter. | `'true'` |
| `[select]`     | yes      | `column = 'expression'`, at least one, and no column name starts with `_` | |
| `on_error`     | no       | `"halt"` or `"skip"`                                   | `"halt"` |
| `start_offset` | no       | The offset where the indexer service starts a filter that has not committed anything yet. The service is not part of the published `cinder`, and `cinder run` does not read this key: use `--from`. | the oldest |

Any other top-level key in the file makes the load fail. This page describes version 1 as
cinder 0.8.0 reads it. While cinder is on 0.x, a release may change what
version 1 accepts.

## Records

Inside every expression, `.` is the record: one PQS row as JSON, with one
**field** per PQS column, plus `.event_type`, `.offset` and `.node_id`.
`.offset` and `.node_id` together are the **position** of the record.

| `source`      | You get one record for                                                  |
|---------------|-------------------------------------------------------------------------|
| `"lifecycle"` | Each contract create and each contract archive. Both carry the payload. If the contract implements interfaces, each interface view PQS holds is a record of its own, at the same position. |
| `"exercises"` | Each choice exercise PQS holds.                                         |

The fields of each source:

```text
# lifecycle
.contract_id  .payload  .contract_key  .template_fqn
.payload_type ("template"|"interface")  .divulged_only
.package_name .package_version .package_id .creation_package_id
.created_at_offset  .created_effective_at  .create_event_id  .created_at_ix
.archived_at_offset .archived_effective_at .archive_event_id .archived_at_ix
.signatories  .observers  .witnesses  .metadata  .contract_key_hash
.create_event_pk .archive_event_pk .life_ix .redaction_id
# exercises
.contract_id  .choice  .choice_fqn  .consuming  .argument  .result
.template_fqn .package_name .package_version .package_id
.exercised_at_offset  .exercised_effective_at  .exercise_event_id
.exercised_at_ix  .exercise_event_pk  .last_descendant_node_id
.signatories  .observers  .controllers  .witnesses  .redaction_id
# both
.event_type  .offset  .node_id
```

| field                                  | What it holds                                                |
|----------------------------------------|--------------------------------------------------------------|
| `.event_type`                          | `"created"`, `"archived"` or `"exercised"`                   |
| `.offset`, `.node_id`                  | The position of the record: the offset of its transaction and the node inside it. |
| `.template_fqn`                        | The name of the template or interface.                       |
| `.payload`                             | The arguments of the contract. When `.payload_type == "interface"` it is the interface view. |
| `.argument`, `.result`                 | The argument and the return value of the choice.             |
| `.divulged_only`                       | `true` when the contract was never active for your validator's parties. Either they only witnessed it, or it was archived in the same transaction that created it. |
| `*_event_id`                           | `{offset, node_id}`                                          |
| `*_effective_at`                       | A UTC string that ends with `+00:00`, with trailing zeros of the fraction dropped: `"2026-09-02T12:58:45.798+00:00"`, `"2026-09-10T11:30:56+00:00"` |
| a Daml `Time` inside `.payload`, `.argument` or `.result` | A string that ends with `Z`, with up to six fractional digits. |
| a Daml `Decimal` or `Int64`            | A JSON string. Use `decimal` and `int` to convert it.        |
| a field with no value                  | `null`                                                       |

Good to know:

- PQS only holds what your validator's parties can see.
- PQS only holds exercises when its scribe ingests transaction trees
  (`TransactionTreeStream`).
- Offsets are local to your participant and have gaps. They are not block
  heights.
- A run or a preview only sees records whose offset is in its range.
- Once PQS holds the archive of a contract, the create record carries it as
  well: every `.archive*` field is set and `.life_ix` is closed, no matter how
  far the run reads. So `.archived_at_offset == null` in a `where` drops the
  create of every contract PQS holds an archive for.

## Matching a template or choice

| To follow               | `where`                                                            |
|-------------------------|--------------------------------------------------------------------|
| a template              | `.template_fqn == "splice-amulet:Splice.Amulet:Amulet"`            |
| a choice                | `.choice_fqn == "splice-amulet:Splice.AmuletRules:AmuletRules:AmuletRules_Transfer"` |
| a choice, in two parts  | `.template_fqn == "splice-amulet:Splice.AmuletRules:AmuletRules" and .choice == "AmuletRules_Transfer"` |
| an interface view       | `.template_fqn == "splice-api-token-holding-v2:Splice.Api.Token.HoldingV2:Holding" and .payload_type == "interface"` |

- A template name is `package:Module:Entity` and a choice name is
  `package:Module:Entity:Choice`. `package` is the name of the package, which
  is the same in every version of it.
- `==` compares the full name. In `cinder types --json`, `aliases` lists the
  full name first. The shorter names after it work in PQS's SQL functions but
  match no record in a filter.
- An interface view is a record of its own, and so is an exercise of an
  interface's choice. On those records `.template_fqn` is the name of the
  interface, `.package_name` is the package of the template that implements
  it (`"splice-amulet"` for the `Holding` view of an Amulet), and
  `.contract_id` is the id of the contract.
- `where` is the only filter you write. `source`, the offset range and what
  PQS holds decide what it filters.

## The `where` grammar

> [!WARNING]
> A `where` must keep at most one record per position. Keeping more is
> undefined behaviour: you may get a document for each of them, or for only
> some, and it can change from one run to the next.
>
> This happens under `source = "lifecycle"`. When a contract is created or
> archived, PQS holds one record for its template and one more for each of
> its interface views. They all have the same position and the same contract
> id, and a lifecycle document is identified by its contract id. An Amulet,
> for example, comes as three records: the `Amulet` template, the `Holding`
> V1 view and the `Holding` V2 view.
>
> So say which one you want with `.template_fqn == "…"`, as in
> [Matching a template or choice](#matching-a-template-or-choice). Leaving
> `where` out keeps them all, and so does a `where` that only tests other
> fields, such as `.payload.owner` or `.contract_id`. `--no-predicate` reads
> every record before `where` filters them, so a lifecycle run with that
> flag is in this case whatever its `where` says.

| Piece      | Form                                                                    | Example                         |
|------------|-------------------------------------------------------------------------|---------------------------------|
| `where`    | `true` on its own, or comparisons joined by `and`, `or`, parentheses and `\| not` | `A and (B or C)`      |
| comparison | `==` or `!=`, between a path and a literal or between two paths         | `.payload.lock == null`         |
| path       | A field from the list of its source under Records, then keys below it. A key that an object does not have is `null`. | `.payload.owner`, `.create_event_id.node_id`, `.payload.meta.values["amulet.splice.lfdecentralizedtrust.org/created-in-round"]` |
| key        | ASCII letters, digits, `_`, `.`, `/` and `-`. It cannot be a 64-bit integer. | `0` and `-1` make the load fail |
| literal    | a string, a number, `true`, `false` or `null`                           | `"alice"`, `42`                 |
| negation   | `C \| not` negates the expression to its left, up to the enclosing parenthesis. | Inside `and` or `or`, write `(C \| not)`. |

Every field can be compared with `null`. Any other literal has to fit the
field:

| field                                                                            | literal                 |
|----------------------------------------------------------------------------------|-------------------------|
| `.payload`, `.contract_key`, `.argument`, `.result`, `*_event_id`                | any, at any path below  |
| `*_offset`, `*_at_ix`, `*_pk`, `.offset`, `.node_id`, `.last_descendant_node_id` | a number                |
| `.consuming`, `.divulged_only`                                                   | `true` or `false`       |
| `.signatories`, `.observers`, `.witnesses`, `.controllers`, `.life_ix`           | `null` only             |
| every other field                                                                | a string                |

These make the load fail:

- A path on its own (`.consuming`), `false` on its own, or `true` inside a
  larger expression.
- A field that is not in the list of its source: `.choice` under `lifecycle`,
  `.payload` under `exercises`, or a misspelt field.
- A key below any field other than `.payload`, `.contract_key`, `.argument`,
  `.result` and `*_event_id`.
- Ordering (`<` `<=` `>` `>=`), `?`, array indexes, `//`, and any function
  call other than `| not`.
- A string with `\(…)` or a NUL (`\u0000`), a format string
  (`@base64 "…"`), and a number outside the range of a 64-bit float
  (`1e400`).

How a `where` runs:

- jq evaluates the `where` on each record. `cinder run` also pushes it into
  the PQS query, unless the `where` is `true` or you pass `--no-predicate`.
- The pushed query and jq disagree on one case: a path that runs into a
  string, a number, a boolean or an array on its way down. The pushed query
  treats it as `null`, and jq raises an error. A record the query rejects is
  dropped without an error. A record that reaches jq fails when jq reads that
  path, and under `--no-predicate` every record reaches jq.

## The `select` expression dictionary

Each entry of `[select]` is a **column** of the document. **Expressions** are
jq as the `jaq` engine (version 3) runs it, and a column holds the result jaq
gives.

| call            | accepts                                                      | emits                                              |
|-----------------|--------------------------------------------------------------|----------------------------------------------------|
| `decimal(expr)` | A decimal string (`"82.2093476348"`, `"1e5"`) or an integral number. A `_` after the first digit and before any exponent is ignored (`"1_000"`). | A string in plain notation: `"82.2093476348"`, `"100000"` |
| `int(expr)`     | An integer string (`"-12"`, `"+5"`) or a number written without a fraction or an exponent, within Int64. So `1.0` and `1e3` raise, and `-0` raises too. | A number: `1200000` |
| `time(expr)`    | A string with a date, a time and an offset or `Z`, in a form that jiff's `Timestamp` parses: RFC 3339, with `T` or a space in the middle, minutes and seconds optional. A `[zone]` suffix is ignored. It has to be before `9999-12-30T22:00:01Z`. | UTC with six fractional digits. A 7th to 9th digit is dropped: `"2026-09-02T12:58:45.798000Z"` |
| `party(expr)`   | A string.                                                    | The same string.                                   |
| `one_of(expr)`  | One argument, a stream `a, b, …`. `false` and `0` count as values. | The first value that is not `null`: `party(one_of(.payload.account.owner, .payload.owner))` |

`decimal`, `int`, `time` and `party` raise on anything else, `null` included.
`one_of` raises when every value in its stream is `null`.

| jq syntax                     | example                                                   |
|-------------------------------|-----------------------------------------------------------|
| identity, field, string key   | `.`, `.payload.owner` (`null` when absent), `.meta.values["a/b"]` |
| optional access               | `.payload.lock?`, `.inputs[]?`                            |
| index, slice, iteration       | `.signatories[0]`, `.contract_id[0:8]`, `[.inputs[]? \| .value // .]` |
| object, string interpolation  | `{who: .payload.owner}`, `"holder:\(.payload.owner)"`     |
| pipe, alternative, grouping   | `.payload \| keys`, `(.payload.missing // "default")`     |
| comparison, logic, arithmetic | `== != < <= > >=`, `and or not`, `+ - * / %`              |
| conditional, error handling   | `if .x then 1 elif .y then 2 else 3 end`, `try (.a \| tonumber) catch -1` |
| recursive descent             | `[..]`                                                    |

Builtins: `split join startswith endswith ltrimstr rtrimstr ascii_downcase
ascii_upcase tostring tojson fromjson tonumber floor ceil round length has in
contains keys keys_unsorted values map map_values select any all add first
last min max min_by max_by sort sort_by unique unique_by reverse flatten range
limit to_entries from_entries with_entries getpath paths recurse type not
empty error`.

Call each builtin with the number of arguments jq gives it: `split` takes
one, a string separator, and `add` and `recurse` take none. An expression can
also use `true`, `false` and `null`, and call `decimal`, `int`, `time`,
`party` and `one_of` with one argument.

Everything else is refused when the filter loads:

| Refused                              | Example                             |
|--------------------------------------|-------------------------------------|
| any other function or argument count | `env`                               |
| a variable                           | `$x`, `$ENV`                        |
| a binding                            | `. as $x`                           |
| `reduce`, `foreach`, `label`/`break` |                                     |
| a format string                      | `@base64`, `@text "…"`              |
| `def`                                |                                     |
| assignment                           | `= \|= += -= *= /= %= //=`          |
| `module`, `import`, `include`        |                                     |

The error names the first thing it refuses:

```text
jq program `env.HOME` uses the function `env/0`, which a filter may not; see the `select` expression dictionary in docs/filter-format.md
```

The engine does not limit time or memory. `[range(1e9)]` runs out of memory,
and `fromjson` over text nested 20,000 deep overflows the stack. Either one
ends the whole `cinder` process, and `on_error` does not see it.

## Documents

`cinder run` writes one JSON line for each record that `where` keeps, unless
an error drops it. `cinder --help` calls a document an "entity".

| In the document        | Value                                                                  |
|------------------------|------------------------------------------------------------------------|
| `_op`                  | What to do with the line when you load it into a table. `"upsert"`, for lifecycle: it replaces the line with the same `_contract_id`. `"append"`, for exercises: it is added. |
| `_archived_at_offset`  | Lifecycle only. On the document of a create it is `null`, even when the record's `.archived_at_offset` is set. On the document of an archive it is the offset of the archive. |
| `_offset`, `_node_id`  | `.offset` and `.node_id` of the record                                 |
| `_contract_id`         | `.contract_id` of the record                                           |
| your columns           | in order                                                               |

`cinder run` only writes the lines. Loading them is up to you, and the SQL in
the headers of the examples reads the table you load them into, which they
call `<table>`.

The columns of an archive are evaluated on the archive record. A column that
reads `.event_type` or `.offset` therefore differs between the two documents
of a contract, as in `examples/holding-lifecycle.toml`.

Here are two of the lines that
`cinder run examples/validator-licenses.toml --from 114984 --to 115218`
writes on our DevNet validator, the create and the archive of one
ValidatorLicense contract (ids shortened):

```json
{"_op":"upsert","_archived_at_offset":null,"_offset":114984,"_node_id":1,"_contract_id":"003e39cd…","validator":"infradao-devnet-1::1220…","sponsor":"infradao-devnet-1::1220…","dso":"DSO::1220…","version":"0.7.4","contact_point":"","last_active_at":"2026-09-02T22:31:52.194324Z","metadata_updated_at":"2026-09-02T12:28:40.242546Z"}
{"_op":"upsert","_archived_at_offset":115218,"_offset":115218,"_node_id":0,"_contract_id":"003e39cd…","validator":"infradao-devnet-1::1220…","sponsor":"infradao-devnet-1::1220…","dso":"DSO::1220…","version":"0.7.4","contact_point":"","last_active_at":"2026-09-02T22:31:52.194324Z","metadata_updated_at":"2026-09-02T12:28:40.242546Z"}
```

## Errors

| What fails | When                                                                     | What happens                     |
|------------|--------------------------------------------------------------------------|----------------------------------|
| the load   | The TOML is invalid. A key of the file is missing, unknown or has the wrong type. A value is not one the keys table allows. The `where` is outside the grammar. A column does not compile or uses something refused. | The error names what failed. |
| a record   | An expression raises, `where` included: a coercion refuses its input, `error` is called, or `.a.b` is read where `.a` is a string. A column yields no value, or more than one. A value is not JSON (`1 / 0`, an object key that is not a string). | `on_error` decides. |
| the run    | PQS fails, or a record has a `contract_id` or `template_fqn` that is missing, empty or not a string. | The run stops, whatever `on_error` says. |

| `on_error` | What happens to a failing record                                         |
|------------|--------------------------------------------------------------------------|
| `"halt"`   | The run stops with that error and logs no cursor. Documents are written in batches of up to 100 records, so stdout holds the documents of the batches before the one with the failing record. |
| `"skip"`   | The record is dropped, with a warning that names the error and the contract. A run that finishes logs how many were dropped. |
