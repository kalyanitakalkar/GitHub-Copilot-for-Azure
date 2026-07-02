# ARM MCP — Azure Resource Graph Query

Run **one** `execute_query` call across **all** subscriptions (pass every subscription ID in
the `subscriptions` array). Never run per-subscription queries. The `summarize` groups by
`recommendationTypeId` across all subscriptions, returning only the minimum fields needed for
enrichment.

## Base query

```kusto
advisorresources
| where type =~ 'microsoft.advisor/recommendations'
| where properties.category =~ '<CATEGORY>'
| where isnull(properties.suppressionIds) or array_length(properties.suppressionIds) == 0
| where properties.recommendationStatus == "New"
| where isempty(properties.tracked) or properties.tracked == false
| summarize resourceCount = dcount(tostring(properties.resourceMetadata.resourceId)),
            subscriptionIds = make_set(subscriptionId, 20),
            subscriptionCount = dcount(subscriptionId)
         by recommendationTypeId = tostring(properties.recommendationTypeId)
```

`<CATEGORY>` defaults to `HighAvailability`. Other values: `Cost`, `Performance`,
`Security`, `OperationalExcellence`.

## Optional filters (insert **before** the `summarize`)

Subcategories:
```kusto
| where tostring(properties.extendedProperties.recommendationSubCategory) in~ ('<subcategory1>', '<subcategory2>')
```

Resource groups:
```kusto
| where tolower(resourceGroup) in ('<rg1>', '<rg2>')
```

Only retirements due within N days:
```kusto
| where todatetime(properties.extendedProperties.retirementDate) between (startofday(now()) .. startofday(now() + <N>d))
```

## Returned fields

| Field | Purpose |
|-------|---------|
| `recommendationTypeId` | Join key with Cosmos metadata (Advisor MCP) |
| `resourceCount` | Distinct affected resources across all subscriptions |
| `subscriptionIds` | Subscriptions affected per recommendation type |
| `subscriptionCount` | How many subscriptions are affected |

Do **not** project extra fields (name, properties, etc.) — Cosmos supplies all display metadata.

## Pagination

If ARM MCP returns a `$skipToken` (or indicates more results), call `execute_query` again with
the token until all pages are collected. Concatenate every page into a single JSON array before
passing it to the Advisor MCP enrichment step.

## Priority labels

The Advisor MCP maps `priorityScore` to a label:

| Score | Label |
|-------|-------|
| ≥ 0.80 | Very High |
| ≥ 0.65 | High |
| ≥ 0.45 | Moderate |
| ≥ 0.25 | Low |
| < 0.25 | Very Low |
