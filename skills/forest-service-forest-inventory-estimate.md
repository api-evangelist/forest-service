---
generated: '2026-09-10'
method: generated
name: Produce a Forest Inventory and Analysis estimate
description: Resolve the FIADB-API's classed parameter values, then request a population estimate with sampling error from the national forest census.
api: https://apps.fs.usda.gov/fiadb-api
operations: [GET /fullreport/parameters/{name}, GET /fullreport, POST /fullreport]
source: >-
  Grounded in the agency's parameter reference at https://apps.fs.usda.gov/fiadb-api/. The call in
  step 3 was executed live on 2026-09-10 and returned HTTP 200 with 34 KB of estimates. No
  OpenAPI exists for this API; nothing below is inferred.
---

# Produce a Forest Inventory and Analysis estimate

FIA is the United States' continuous forest census. The FIADB-API (the API behind EVALIDator)
turns a grouping request into population estimates with standard errors, variances and plot
counts. It is free and anonymous.

## Auth

None. See `authentication/forest-service-authentication.yml`.

## Base URL

`https://apps.fs.usda.gov/fiadb-api`

## Before you start

The docs page carries a standing notice that it is under construction and that not all endpoints
are documented yet. Treat the parameter dictionaries in step 1 as the authoritative source of
legal values, not any list you remember.

## Steps

1. **Resolve every classed parameter from its dictionary.** Each takes its values from a live
   endpoint, and passing an unlisted value returns an error page under HTTP 200:
   - `GET /fullreport/parameters/snum` — the estimate attribute. Accepts the numeric
     `ATTRIBUTE_NBR` or the text `ATTRIBUTE_DESCR`.
   - `GET /fullreport/parameters/wc` — the evaluation group code, from the `EVALID` column.
     Typically the state FIPS code concatenated with a four-digit inventory year: `102020` is the
     Delaware 2020 inventory.
   - `GET /fullreport/parameters/rselected` and `/cselected` — the row and column groupings, from
     the `LABEL_VAR` column. `pselected` adds an optional page grouping.

2. **Choose an output format.** Pass `outputFormat=NJSON` — the flat JSON shape with metadata.
   The default is `HTML`, and the unprefixed `JSON`/`XML` values return the legacy EVALIDator
   page/row/column cross-tabulation, which is much harder to parse. `NHTML` and `NXML` are the
   other flat variants.

3. **Request the estimate.** `GET /fullreport` with `snum`, `wc`, `rselected`, `cselected` and
   `outputFormat=NJSON`. A verified example:

   ```
   GET https://apps.fs.usda.gov/fiadb-api/fullreport
       ?rselected=Land%20Use%20-%20Major
       &cselected=Land%20use
       &snum=79
       &wc=102020
       &outputFormat=NJSON
   ```

   You get `estimates[]` — each row carrying `ESTIMATE`, `GRP1`, `GRP2`, `PLOT_COUNT`, `SE`,
   `SE_PERCENT` and `VARIANCE` — plus `subtotals`, `totals`, a `metadata` object containing the
   generated SQL, and a formatted `citation` string.

   The same request also works as a `POST` with the parameters as a form body. Use POST when the
   query string would be long; it is the same read, so replaying it is safe.

4. **Narrow further if needed.** `strFilter` takes a raw SQL fragment against the FIA tables, with
   or without a leading `and` — for example `COND.OWNCD = 40` or `AND TREE.CCLCD in (1,2,3)`.
   `sdenom` supplies a denominator attribute for ratio estimates. `rtime`/`ctime`/`ptime` set the
   temporal basis for change, growth, removal and mortality estimates (`CURRENT`, `PREVIOUS`,
   `CURRENT IF AVAILABLE ELSE PREVIOUS`, `PREVIOUS IF AVAILABLE ELSE CURRENT`, `ACCOUNTING`).
   `FIAorRPA` switches between the FIA and RPA definitions of forest land.

5. **Always report the sampling error with the estimate.** `SE_PERCENT` on a row can exceed 40%
   when `PLOT_COUNT` is small — the live example above returned an estimate backed by six plots at
   `SE_PERCENT` 42.9 alongside one backed by 126 plots at 4.2. An estimate quoted without its
   error is misleading, and this API hands you the error in the same row.

6. **Cite from the response.** The `citation` field names the program, the retrieval timestamp,
   the application version and the publishing research station. Use it verbatim.

## Urban forests

`GET /urban` is the separate Urban FIADB-API, documented on its own page.

## Errors

An invalid parameter returns **HTTP 200 with a text/html "EVALIDator | Error Page"**, even when
`outputFormat=NJSON` was requested. Branch on the content-type, never on the status. See
`errors/forest-service-error-catalog.yml`.

## Rate limits and reversibility

No published limits and no rate-limit headers (`rate-limits/forest-service-rate-limits.yml`).
Read-only: nothing to undo, no idempotency key to send.

## Support

`SM.FS.FIA.Digital@usda.gov`
