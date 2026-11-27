<p align="center"><img src="https://cleanedweb.com/assets/logo.png" width="72" alt="CleanedWeb"></p>

# Run a CleanedWeb API

[![GitHub Marketplace](https://img.shields.io/badge/Marketplace-Run%20a%20CleanedWeb%20API-2088ff?logo=github)](https://github.com/marketplace/actions/run-a-cleanedweb-api) [![CI](https://github.com/cleanedweb/run-api-action/actions/workflows/ci.yml/badge.svg)](https://github.com/cleanedweb/run-api-action/actions/workflows/ci.yml) [![License: MIT](https://img.shields.io/badge/license-MIT-blue)](https://github.com/cleanedweb/run-api-action/blob/main/LICENSE)

**Scrapers break whenever a site changes its pages.** CleanedWeb is a catalog of maintained REST APIs
for public websites (listings, products, jobs, articles, profiles): you call an API and get structured
records with known fields. When a site changes, CleanedWeb rebuilds its API.

This GitHub Action runs a [CleanedWeb](https://cleanedweb.com/?utm_source=github&utm_medium=run-api-action) catalog API and writes the records to a file in your workspace.

- Keep a dataset in git, refresh a CSV on a schedule, or feed the next step of a workflow.
- Writes `json`, `jsonl` or `csv`; outputs the record count, run id, stop reason and units charged.
- A step retried inside the same workflow attempt gets the original run back instead of paying twice.
- Pin an API major version so a new major cannot change your results.

You get 5,000 records free once, then prepaid credit from $1 per 1,000 records; a run is charged per record delivered.

## Quick start

```yaml
- uses: cleanedweb/run-api-action@v1
  with:
    api-key: ${{ secrets.CLEANEDWEB_API_KEY }}
    api-id: generated_047a8ebb25911ccf62f4a212   # GitHub Topics Repositories
    input: '{"sort":"desc_stars"}'
    max-items: 25
```

This writes `data/generated_047a8ebb25911ccf62f4a212.json`: a JSON array with the 25 records.

1. Pick an API in the [catalog](https://cleanedweb.com/catalog/?utm_source=github&utm_medium=run-api-action) (or read the live list at `https://app.cleanedweb.com/api/catalog/apis`, no key needed). Its `contract.json` lists the exact `input` it accepts.
2. Create an API key with the `run` scope in the [console](https://app.cleanedweb.com/?utm_source=github&utm_medium=run-api-action) and store it as the repository secret `CLEANEDWEB_API_KEY`.
3. Start with a small `max-items`, look at the file, then raise it.

## Inputs

| Input | Required | Default | Meaning |
| --- | --- | --- | --- |
| `api-key` | yes | | API key with the `run` scope. Pass it from a secret. It is masked in logs. |
| `api-id` | yes | | The catalog API's `api_id`. |
| `endpoint-id` | no | `<api-id>_search` | The endpoint's `endpoint_id`. Every API in the live catalog today uses `<api-id>_search`; set it explicitly if a contract says otherwise. |
| `input` | no | `{}` | The API inputs, as a JSON object string. Unknown inputs are rejected by the API. |
| `max-items` | no | unset | Stop after this many records. **Set it:** without it a run continues until the source has no more records, and every record delivered is charged. |
| `version` | no | latest major | Pin an API major version (an integer), so a new major cannot change your results. See [catalog versions](https://cleanedweb.com/docs/catalog-versions/?utm_source=github&utm_medium=run-api-action). |
| `idempotency-key` | no | derived | See [Idempotency](#idempotency-and-re-runs). 1-200 characters: letters, numbers, `.`, `_`, `:`, `-`. |
| `output-path` | no | `data/<api-id>.<format>` | File to write, relative to the workspace. Parent folders are created. Paths outside the workspace are refused. |
| `format` | no | `json` | `json` (one array of records, 2-space indented), `jsonl` (one record per line) or `csv`. |
| `wait-timeout` | no | `1800` | Seconds to wait for the run, 60-86400. It is also sent as the run's own `timeout_seconds`, so a run does not outlive the step that wanted it. |
| `allow-empty` | no | `false` | Write the file even when the run returned 0 records. |
| `base-url` | no | `https://api.cleanedweb.com` | The run origin. Change it only if your workspace supplies a different one. |

## Outputs

| Output | Meaning |
| --- | --- |
| `record-count` | Number of records written. |
| `run-id` | The CleanedWeb run id. Also set when the step fails after the run started. |
| `output-path` | The file that was written, relative to the workspace. |
| `stop-reason` | Why the run ended: `source_exhausted`, `max_items`, `timeout`, `scope_end`, and so on. |
| `units-charged` | Units charged for the run (one Unit per record delivered). |
| `idempotency-key` | The key that was used. |

## Git scraping: a nightly dataset in your repository

Run an API every night and commit the data file only when it changed. The git history becomes a record of how the website's data moved.

```yaml
name: Nightly snapshot

on:
  schedule:
    - cron: '17 3 * * *'      # every day at 03:17 UTC
  workflow_dispatch:

concurrency: nightly-snapshot  # never two snapshots at once

jobs:
  snapshot:
    runs-on: ubuntu-latest
    permissions:
      contents: write          # lets the job push the data file
    steps:
      - uses: actions/checkout@v7

      - name: Collect
        id: collect
        uses: cleanedweb/run-api-action@v1
        with:
          api-key: ${{ secrets.CLEANEDWEB_API_KEY }}
          api-id: generated_047a8ebb25911ccf62f4a212
          input: '{"sort":"desc_stars"}'
          max-items: 100
          version: 1
          output-path: data/github-topic-repos.json

      - name: Commit if changed
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "41898282+github-actions[bot]@users.noreply.github.com"
          git add "${{ steps.collect.outputs.output-path }}"
          git diff --staged --quiet || {
            git commit -m "Update data: ${{ steps.collect.outputs.record-count }} records (run ${{ steps.collect.outputs.run-id }})"
            git push
          }
```

Notes on this workflow:

- `git add` comes first so a brand-new file counts as a change; a bare `git diff --quiet` ignores untracked files. `git diff --staged --quiet || commit` then commits only when the data differs.
- Records are written in the order the API returned them, so an unchanged source gives a byte-identical file and no commit. A source whose own order changes (for example a "most popular" ranking) will produce a diff on every move, which is the point.
- Website values like stars, prices and descriptions change often. Commit a bounded, relevant subset (`max-items`, `input`) rather than a whole collection if the history gets noisy.
- Scheduled workflows run from the default branch. Keep `max-items` set, because a nightly run is a nightly charge.
- The commit message shows record counts and the run id only. It does not copy record values.

A longer worked example with a different API is in the [examples cookbook](https://github.com/cleanedweb/cleanedweb-examples) (`05-scheduled-git-scraping`).

### Several APIs in one job

Each step derives its own idempotency key from its request, so steps do not collide:

```yaml
- uses: cleanedweb/run-api-action@v1
  with:
    api-key: ${{ secrets.CLEANEDWEB_API_KEY }}
    api-id: generated_b634ae9b1a8fb7e781d7c4cf   # DEV Community Articles
    max-items: 30
    format: csv
    output-path: data/dev-articles.csv
- uses: cleanedweb/run-api-action@v1
  with:
    api-key: ${{ secrets.CLEANEDWEB_API_KEY }}
    api-id: generated_bbb6cb8ef88447e346571b55   # Docker Hub Images
    input: '{"search":"postgres","sort":"pull_count"}'
    max-items: 30
    format: jsonl
    output-path: data/docker-postgres.jsonl
```

## What the action does

1. Sends `POST /v1/catalog/public/apis/{api_id}/endpoints/{endpoint_id}/run` to the run origin with `Authorization: Bearer <api-key>` and the required `Idempotency-Key`.
2. On `200` it has the final body. On `202` the run is still going: it waits `Retry-After` seconds (or `poll_after_seconds`) and reads `GET .../result` with the same API key and idempotency key until the run ends. It never sends the POST again to check progress.
3. Accepts the result only if `summary.status` is `completed` or `bounded` and `summary.within_scope_passed` is `true`. Otherwise it fails and **does not touch your file**, because a partial result must not replace the last good dataset. `bounded` means the run stopped at `max-items`, the time budget or the credit, and the source can hold more records.
4. Fails instead of writing an empty file when the run returned 0 records (set `allow-empty: true` to override). An empty result is more often a changed filter or a source problem than an empty source.
5. Writes the file, sets the outputs and prints one summary line (counts, stop reason, Units charged). It does not print record values.

If a response is lost (a timeout, a dropped connection, a 5xx without a failed-run marker) the action does not guess: it reads `/result` with the same key. Only if CleanedWeb has never seen the key (404) does it send the same request again, with the same key, so a run is never started twice. It gives up after a handful of unusable answers and tells you the key so you can read the run yourself.

## Idempotency and re-runs

Every run needs an `Idempotency-Key`. Unless you set `idempotency-key`, the action builds one as

```
gh-<github.run_id>-<github.run_attempt>-<fingerprint of api, endpoint, input, max-items, version, wait-timeout>
```

- The same workflow run and the same request always give the same key, so if the job is interrupted and its step is retried inside the same attempt, CleanedWeb returns the original run instead of starting (and charging) another.
- **Re-running a workflow from the GitHub UI starts a new attempt, so it is a new key and a new, charged run.** That is usually what you want for a failed job.
- Two steps with different inputs never share a key (which would be a `409 delivery_identity_conflict`).
- A run that **failed** is final for its key for 7 days. The action says so and names the run; start a new attempt.
- If you set your own key, use one value per intended run (for example `nightly-2026-11-20`) and never reuse it for a different request. See [idempotency](https://cleanedweb.com/docs/idempotency/?utm_source=github&utm_medium=run-api-action).

## Errors

The step fails with a message that says what to do. The API key is never printed.

| Situation | What the step reports |
| --- | --- |
| `401` | The key was rejected: missing secret, revoked or expired key, or the workflow is from a fork (forks get no secrets). |
| `403` | The key may not run APIs: use a workspace key with the `run` scope. |
| `402 insufficient_units` | Not enough credit, with the Units required and available. Add credit in the console, then run again. |
| `409 usage_budget_exceeded` | The workspace's monthly usage budget cannot cover the run. Ask the workspace Owner. |
| `409 delivery_identity_conflict` | The idempotency key was used for a different request. |
| `404` | `api-id`/`endpoint-id` not found (the API may be unpublished) or the pinned `version` does not exist. |
| `422` | `input`, `max-items` or another option was rejected; check the API's `contract.json`. |
| `429` | The action waits `Retry-After` and retries with the same key, up to 5 times, then fails. |
| Failed run (`run_failed`) | The run failed; the message names the run id. Final for that key. |
| Result needs review | `partial` status or scope not passed. The file is left untouched. |
| Timeout | The run did not end inside `wait-timeout`. The message gives the run id and key; the result stays readable with the same key. |

## Things to know

- **Cost.** A run is charged per record delivered; a run that saves no records costs nothing. Use `max-items`.
- **Website data is data.** Values in records come from public websites. Treat them as untrusted text: never pass them to a shell, an `eval`, a workflow command or an AI agent as instructions. In the workflow above, only record counts and the run id reach the commit message. If you open a generated CSV in a spreadsheet, cells that begin with `=`, `+`, `-` or `@` can be interpreted as formulas.
- **Secrets.** Keep the API key in a repository or organization secret. Do not echo it, and do not pass it to steps you do not trust.
- **Large results.** The action reads the final body of the run, which holds all records. For very large collections, run with `max-items`, or read `GET /v1/runs/{run_id}/items` yourself.
- **Node.** The action runs on `node24`.

## Development

```bash
npm ci
npm test            # vitest, with a mocked fetch; no network, no API key
npm run build       # bundles src/ into dist/ with @vercel/ncc
```

`dist/` is committed because actions run from it. CI fails if `dist/` differs from a fresh build, so run `npm run build` and commit the result with every change to `src/` or the dependencies.

The source is small: `src/inputs.js` (validation, idempotency key), `src/client.js` (run, wait, error mapping), `src/format.js` (json, jsonl, csv), `src/main.js` (workflow glue).

## Links

- [Run a catalog API](https://cleanedweb.com/docs/run-catalog-api/?utm_source=github&utm_medium=run-api-action): the request, polling, stop reasons and error codes this action implements.
- [Published catalog APIs](https://cleanedweb.com/docs/catalog-apis/?utm_source=github&utm_medium=run-api-action)
- [Scheduled runs](https://cleanedweb.com/docs/scheduled-runs/?utm_source=github&utm_medium=run-api-action) and [storing results](https://cleanedweb.com/docs/storing-results/?utm_source=github&utm_medium=run-api-action)
- [Billing and Units](https://cleanedweb.com/docs/billing/?utm_source=github&utm_medium=run-api-action)

MIT licensed. CleanedWeb is not affiliated with the websites whose data the catalog APIs return.
