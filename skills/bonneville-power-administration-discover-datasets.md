---
name: discover-bpa-datasets
description: Discover what BPA actually publishes — search the OGC API Records catalog, read a dataset record, and follow it to the callable endpoint or a bulk download.
api: bonneville-power-administration:bonneville-power-administration-data-search-api
generated: '2026-09-06'
method: generated
source: openapi/bonneville-power-administration-data-search-api-openapi.yml, json-ld/bonneville-power-administration-dcat-us-catalog.json
operations:
  - OgcRootController_getSiteRoot_api/search/v1
  - CollectionController_getSiteCollections_api/search/v1
  - OgcItemController_getSiteCollectionItems_api/search/v1
  - OgcItemController_getSiteCollectionItemById_api/search/v1
  - QueryableController_getSiteCollectionQueryables_api/search/v1
base: https://data-bpagis.hub.arcgis.com
auth: none
---

# Discover what BPA publishes

Start from the catalog, not the service directory. The ArcGIS tenant exposes 173
FeatureServers, but only 21 datasets are catalogued as published — the rest are story-map
backing services, county cadastral layers and dated project snapshots.

## The fastest path: one file

`GET https://data-bpagis.hub.arcgis.com/data.json`

A DCAT-US 1.1 catalog of all 21 datasets, publisher "Bonneville Power Administration".
Each dataset carries `title`, `description`, `keyword`, `issued`, `modified`, `license`,
`spatial` and a `distribution` array. The distribution titled **"ArcGIS GeoService"** is
the callable API endpoint; CSV, GeoJSON, KML, Shapefile and File Geodatabase entries are
bulk downloads.

If you need the whole picture once, fetch this and stop.

## The queryable path: OGC API Records

`GET /api/search/v1` — landing page with links to collections, conformance and the
OpenAPI definition.

`GET /api/search/v1/collections` — the `dataset` collection, plus its supported filters.

`GET /api/search/v1/collections/dataset/queryables` — the fields you may filter on.
**Read this before constructing a filter**; do not guess field names.

`GET /api/search/v1/collections/dataset/items?q=transmission` — search records.

`GET /api/search/v1/collections/dataset/items/{itemId}` — one record in full.

`GET /api/search/v1/collections/dataset/aggregations` — facet counts, useful for
summarising the catalog by type or keyword without paging every record.

The API declares its own conformance at `/api/search/v1/conformance`: OGC API Common
Parts 1 and 2, Features Part 1 (core, geojson, oas30) and Records Part 1 (core, json).
Its OpenAPI 3.0 definition is at `/api/search/definition/?f=json` — note that the
published document ships an **empty `servers[]`**; the host is
`https://data-bpagis.hub.arcgis.com`, taken from the landing page's own self link.

## Following a record to data

A catalog record's GeoService distribution gives you a FeatureServer layer URL. From
there the query conventions in
`conventions/bonneville-power-administration-conventions.yml` apply: `where`, `outFields`,
`f`, and `resultOffset` / `resultRecordCount` for paging.

## Rules

- No credential anywhere in this flow.
- Catalog endpoints return ordinary HTTP status codes; the underlying FeatureServer
  endpoints do not — those return errors inside HTTP 200. Handle both.
- Prefer `modified` on the catalog record over any date inside the data: BPA publishes no
  changelog, and the record timestamp is the only change signal available.
- Read-only. Nothing in this flow writes.
