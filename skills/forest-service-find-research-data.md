---
generated: '2026-09-10'
method: generated
name: Find and cite Forest Service research data
description: Search the Forest Service Research Data Archive for a data publication, resolve its legal filter values first, and return a citable DOI record.
api: https://www.fs.usda.gov/rds/archive/webservice
operations: [GET /organizations, GET /efrs, GET /search, GET /product, GET /oaipmh]
source: >-
  Grounded in the agency's own API documentation at
  https://www.fs.usda.gov/rds/archive/webservice/assets/ResearchDataArchiveAPIDocumentation.pdf.
  Every endpoint, parameter and response field named below was exercised live against the
  production service on 2026-09-10 and returned HTTP 200. No OpenAPI exists for this API, so the
  steps name HTTP method + path rather than operationIds; nothing here is inferred.
---

# Find and cite Forest Service research data

The Forest Service Research Data Archive publishes peer-reviewed research datasets, each with a
DOI under the `10.2737` prefix. Its web service is free, anonymous and has been stable since 2014.

## Auth

None. Send no credential — see `authentication/forest-service-authentication.yml`.

## Base URL

`https://www.fs.usda.gov/rds/archive/webservice`

## Format

Append `format=json` to any REST endpoint. **The default is XML and the `Accept` header is
ignored** — see `conventions/forest-service-conventions.yml`.

## Steps

1. **Resolve filter values before you filter.** The archive publishes two lookup endpoints
   precisely so a client never has to guess a filter string.
   - `GET /organizations?format=json` — 405 funders, each with an `acronym`, a full `name`, and a
     Crossref Funder Registry id where one has been matched. The `funder` filter accepts either
     the full name (`USDA Forest Service, Rocky Mountain Research Station`) or the acronym
     (`RMRS`).
   - `GET /efrs?format=json` — the 32 Experimental Forests and Ranges that have products in the
     archive, each with a product `count`. The `efr` filter accepts these names exactly.
   Passing a value you did not read from one of these two endpoints is the most likely way to get
   an empty result set back.

2. **Search.** `GET /search?query=<text>&format=json`, optionally narrowed with `funder=` and
   `efr=`. Paginate with `page` (1-based) and `rows` (default 10); order with `sort=date`
   (default) or `sort=score`. The response echoes `numFound`, `page`, `start` and `rows`, so
   compute the last page as `ceil(numFound / rows)` rather than walking until empty.

3. **Fetch the full record.** Take the `id` from a search hit — the accession format is
   `RDS-YYYY-NNNN` with an optional edition suffix such as `RDS-2011-0003.3` — and call
   `GET /product?id=<id>&format=json`. You get `title`, `edition`, `doi`, `latest_edition`,
   `abstract`, `publication_year`, `authors[]`, `publisher`, `publication_places[]`,
   `keywords[]`, `places[]`, `funders[]` and a bounding box under `coordinates`.

4. **Check you are on the current edition.** `latest_edition` on the product record names the
   successor when the one you fetched is superseded — `RDS-2013-0001` returns
   `latest_edition: RDS-2013-0001-2`. If it differs from `id`, re-fetch the successor before
   citing. This is the only versioning signal in the model and it is easy to miss.

5. **Cite by DOI, not by URL.** Use the `doi` field (`https://doi.org/10.2737/RDS-2013-0001`).
   The web-service URL is not a durable citation.

## Bulk harvesting

If you want the whole catalog rather than a query, do not page `/search`. Harvest over OAI-PMH
2.0 instead: `GET /oaipmh?verb=ListRecords&metadataPrefix=oai_dc` for Dublin Core, or
`metadataPrefix=fgdc` for full FGDC-STD-001.1-1999 geospatial metadata. The repository declares
`earliestDatestamp` 2016-02-22, `deletedRecord: no`, and gzip/deflate compression. Use
`resumptionToken` as the protocol specifies.

## Errors

Failures return HTTP 404 with a **text/html** page, not a machine-readable document, and a
missing required `id` is indistinguishable from an unknown one. Treat any non-XML/non-JSON
content-type as a failure. See `errors/forest-service-error-catalog.yml`.

## Rate limits and reversibility

None published, no rate-limit headers returned — throttle yourself
(`rate-limits/forest-service-rate-limits.yml`). Every operation is a read, so there is nothing to
undo and no idempotency key to send.
