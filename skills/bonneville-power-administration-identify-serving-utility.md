---
name: identify-bpa-serving-utility
description: Given a point or place in the Pacific Northwest, identify which BPA customer utility serves it and whether it falls inside the BPA service area.
api: bonneville-power-administration:bonneville-power-administration-customers-api
generated: '2026-09-06'
method: generated
source: openapi/bonneville-power-administration-customers-api-openapi.yml, openapi/bonneville-power-administration-service-area-api-openapi.yml
operations:
  - queryServiceArea
  - queryCustomerPublics
  - queryCustomerIOU
  - queryCustomerTribal
base: https://services3.arcgis.com/Iz3chmSt4P7oOoZy/arcgis/rest
auth: none
---

# Identify the BPA customer utility serving a location

BPA publishes three separate customer-boundary layers. There is no combined layer and no
single lookup — you query all three and take whichever returns a feature.

## 1. Confirm the point is inside the BPA service area

`queryServiceArea` — `GET /services/BPA_ServiceArea/FeatureServer/0/query`

```
?geometry=<lon>,<lat>&geometryType=esriGeometryPoint&inSR=4326&spatialRel=esriSpatialRelIntersects&where=1=1&returnCountOnly=true&f=json
```

A count of 0 means the location is outside BPA's statutory service area and no customer
lookup will match. Stop there rather than reporting "no utility found".

## 2. Query all three customer layers with the same point

Run the same spatial filter against each, changing only the service name:

- `queryCustomerPublics` — `/services/BPA_CustomerPublics/FeatureServer/0/query`,
  `outFields=NAME,FULLNAME,STATE,CAT,BES_NUM`
- `queryCustomerIOU` — `/services/BPA_CustomerIOU/FeatureServer/0/query`,
  `outFields=NAME,BES_NUM`
- `queryCustomerTribal` — `/services/BPA_CustomerTribal/FeatureServer/0/query`,
  `outFields=Name,Website`

**The tribal layer's field is `Name`, not `NAME`.** Field names are case-sensitive in the
`where` clause and in `outFields`; reusing the publics query verbatim against the tribal
layer returns `{"error":{"code":400,"message":"Cannot perform query. Invalid query
parameters.","details":["'outFields' parameter is invalid"]}}` — inside an HTTP 200.

## 3. Report the class, not just the name

Which layer answered is itself the finding:

- publics → a publicly-owned utility (a BPA preference customer)
- IOU → an investor-owned utility
- tribal → a tribal utility

Territories can overlap at boundaries. If more than one layer returns a feature, report
all of them rather than picking one.

## Rules

- No credential. Anonymous GET throughout.
- Check the response body for a top-level `error` key; the HTTP status is 200 either way.
- `f=geojson` if you want RFC 7946 geometry back; `f=json` returns ArcGIS JSON.
- Layers cap at 1000 features. For an area query rather than a point, honour
  `exceededTransferLimit` and page with `resultOffset` / `resultRecordCount`.
- Read-only surface: nothing here changes any BPA record.
