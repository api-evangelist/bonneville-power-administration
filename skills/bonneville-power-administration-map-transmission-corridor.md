---
name: map-bpa-transmission-corridor
description: Assemble a BPA transmission corridor — the line centrelines, the towers carrying them, and the right-of-way polygons around them — as GeoJSON, from BPA's public GIS layers.
api: bonneville-power-administration:bonneville-power-administration-transmission-api
generated: '2026-09-06'
method: generated
source: openapi/bonneville-power-administration-transmission-api-openapi.yml, openapi/bonneville-power-administration-right-of-way-api-openapi.yml
operations:
  - queryTransmissionLines
  - queryTransmissionStructures
  - queryRightOfWay
base: https://services3.arcgis.com/Iz3chmSt4P7oOoZy/arcgis/rest
auth: none
---

# Map a BPA transmission corridor

No credential is needed. Every request below is an anonymous GET.

## 1. Find the line

`queryTransmissionLines` — `GET /services/BPA_TransmissionLines_View/FeatureServer/0/query`

Search by operating line name. ArcGIS `where` is SQL-92, and string comparison is
case-sensitive:

```
?where=OperatingLineNm LIKE '%25JOHN DAY%25'&outFields=OperatingLineNm,VoltageMeas,XRefCd&f=geojson
```

Use `where=1=1` first with `outFields=OperatingLineNm` and `returnGeometry=false` to see
the real names before filtering. Voltage lives in `VoltageMeas`.

## 2. Pull the structures on that line

`queryTransmissionStructures` — `GET /services/BPA_TransmissionStructure_View/FeatureServer/0/query`

Join on `OperatingLineNm`, by value, on the client — the server enforces no referential
integrity and offers no relationship query. Take the exact string from step 1:

```
?where=OperatingLineNm='<exact value from step 1>'&outFields=StrcSerialNbr,OperatingLineNm,StrcSymbolCd,MileStrcKey&f=geojson
```

This layer caps at **2000** features per response, unlike the 1000 elsewhere.
`MileStrcKey` orders structures along the line.

## 3. Pull the right-of-way around it

`queryRightOfWay` — `GET /services/BPA_RightofWay_View/FeatureServer/0/query`

The right-of-way layer carries **no line reference** — only geometry. Relate it
spatially, not with a `where` clause:

```
?geometry=<xmin>,<ymin>,<xmax>,<ymax>&geometryType=esriGeometryEnvelope&inSR=4326&spatialRel=esriSpatialRelIntersects&where=1=1&f=geojson
```

Build the envelope from the bounding box of the line geometry you already have.

## Rules that apply to every step

- **Format is `f=geojson`, not the Accept header.** Setting `Accept: application/geo+json`
  and omitting `f` returns ArcGIS JSON instead, silently.
- **Check the body, not the status.** A bad `where` or `outFields` returns **HTTP 200**
  carrying `{"error":{"code":400,...}}`. Test for a top-level `error` key before parsing
  `features`. An unrecognised `f` value is the one exception — it returns a bare
  plain-text `400 Bad Request` that will not parse as JSON.
- **Page when truncated.** If the response sets `exceededTransferLimit: true`, re-request
  with `resultOffset` and `resultRecordCount` until it is false. Do not assume one call
  returned everything.
- **Cache for 30 seconds.** Responses are `cache-control: public, max-age=30` with an
  ETag. Use conditional GET; do not poll faster.
- **No writes.** This surface is read-only, so there is nothing to retry-guard and
  nothing to undo.
