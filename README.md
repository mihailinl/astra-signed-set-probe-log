# astra-signed-set-probe-log

The public, **append-only** log of the Astra plugin registry's signed-set
prober.

Every 15 minutes, and every minute for 20 minutes after each new commit to
[`mihailinl/astra-registry`](https://github.com/mihailinl/astra-registry)'s
`signed` branch, a prober running on a machine outside GitHub's address ranges
and outside the plugin catalogue host reads the four signed-set documents —
`index.json`, `revocations.json`, `trust.json` and `root.json` — from every
place Astra clients read them, verifies each one against root keys it holds
itself rather than takes from the registry, and writes one line per run,
target and document. Once an hour it pushes those lines here.

Nothing here is decided by this repository: it is a record. It exists so that
anyone — the registry, the plugin service, Astra's release desk, or a stranger —
can check what was actually served, when, and whether it verified, without
having to trust the party that served it.

## What is probed

| `target` | `host` | what it is |
|---|---|---|
| `signed` | `raw.githubusercontent.com` | the head of the registry's `signed` branch, read at the exact commit `git ls-remote` returned |
| `pages` | `mihailinl.github.io` | `https://mihailinl.github.io/astra-registry/registry/v1/`, the address released Astra builds read — until it is retired |
| `host` | `registry.minice.ai` | `https://registry.minice.ai/registry/v1/`, the catalogue host, through its CDN — from the day that name is live |

`host` is always the hostname a client would write, never the machine that ran
the probe.

## Layout

```
log/YYYY-MM-DD.jsonl     one file per UTC day of `run_at`; one JSON object per line
host/YYYY-MM-DD.jsonl    one line per hourly push: the probe machine's own uptime that day
```

- **Append-only.** Lines are only ever added. No file is rewritten, truncated or
  deleted, and a correction is a new line, never an edit. A ruleset on this
  repository refuses force-pushes and branch deletion, for every account
  including the owner's.
- **One commit per hourly push**, by `astra-signed-set-probe`.
- **To read a window, take the day files covering it at a named commit.** The
  history does not move under a citation.

## A log line (`astra.registry.probe-log/1`)

| member | meaning |
|---|---|
| `schema` | `astra.registry.probe-log/1` |
| `run_at` | when the run started, `YYYY-MM-DDTHH:MM:SSZ` |
| `mode` | `base` for the 15-minute runs (96 a day, one per UTC quarter-hour); `burst` for the per-minute runs after a new `signed` commit |
| `target`, `host` | the table above |
| `document` | `index.json`, `revocations.json`, `trust.json` or `root.json` |
| `url` | the URL read |
| `reachable` | a `200` with a body a client could use |
| `http_status` | the status, or `null` when no answer came |
| `verified` | the signature verifies against the prober's own root keys, through a verified `trust.json` (for `root.json`: its keys are those roots). Freshness is not part of it |
| `verify_error` | why not, or `null` |
| `serial`, `issued_at`, `expires_at` | what the document says about itself |
| `equal_to_signed_head` | the bytes, after decoding, equal the `signed` head's |
| `signed_head`, `signed_committed_at` | the head it was compared with, and when that head was committed |
| `first_read_lag_seconds` | on the first read that equals a new head, for a document that head changed: seconds from the commit to that read. Otherwise `null` |
| `pages_list_state` | `pre_arming` or `armed`: whether Pages' withdrawal list is meant to be the signed one yet |
| `probe_commit` | the commit of the prober's code |

**An unreachable read** (`reachable: false`) has `verified`, `verify_error`,
`serial`, `issued_at` and `expires_at` all `null` — never `false` and never
carried over from an earlier run — so "the prober could not read it" is never
mistaken for "it failed verification".

**Before arming**, Pages serves an unsigned withdrawal list on purpose; its
lines say `verified: false`, `verify_error: "no_signatures"` and
`pages_list_state: "pre_arming"`, and that is the expected state, not a fault.

## An uptime line (`astra.registry.probe-host/1`)

`pushed_at`, `day`, `boot_at` (when the probe machine last started),
`minutes_elapsed` and `ticks` (minutes of that day the prober actually ran),
`base_slots_elapsed` and `base_slots_with_line`, `newest_line_run_at`, and
`probe_commit`. A quarter-hour with no `base` line is a slot the prober missed;
these lines say whether the machine was up, so a reader can tell "the host
failed" from "the prober was not running".

## Freshness

Pushed hourly, a couple of minutes after a base run. **A reader should treat
the log as stale when its newest line is more than 75 minutes old.**

## What this is not

It is not signed. Its integrity rests on GitHub's history, the append-only
ruleset, and a deploy key scoped to this repository alone, held on the probe
machine in a file only the prober's user can read. Every line can be
re-checked by anyone: fetch the same URL and verify it with the root keys the
registry publishes in `registry/v1/root.json` (and compiles into Astra).

## Licence

GPL-3.0-or-later, like the registry it records (`LICENSE`). Copyright Minice.
