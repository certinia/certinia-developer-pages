---
permalink: /using-the-financial-report-builder-message-channel/
layout: page
---

# Using the Financial Report Builder Message Channel

## Introduction

The FRB External Filters message channel is provided in the Financial Report Builder package. FinancialForce and non-FinancialForce packages can create filter components that can then be injected into Financial Report Builder via the message channel. This enables filters from external packages to filter components in Financial Report Builder and control the data displayed in the package.

For more information about message channels, see the Salesforce Help.

## Publishing To Message Channels

To publish a message on a message channel, include a `lightning:messageChannel` component in your Aura component and use the `publish()` method in your Aura component's controller file. For more information, see the Salesforce Help.

When publishing a message on the FRB External Filters message channel, you must specify the following variables:

- `filterID` (the ID of the filter you are setting)
- `filterJson` (contains the definition of the filter you want to apply)

For example:

```
{
  filterId: "any key text",
  filterJson: "{ ... }" //a string that conforms to the schema
}
```

## Filter Example and Schema

### Example

Below is an example company name filter for the Financial Transactions dataset.

```
{
  fieldApiName: "CompanyName",             // field name
  datasetApiName: "FinancialTransactions", // dataset api name in analytics
  fieldType: "dimension",                  // field type (dimension, measure, date)
  version: 1,
  constraint: {
    operator: "in",                        // filter type (see schema)
    value: ["Merlin Technologies Ltd."]    // filter value (type varies by filter type, see schema)
}
```

### Schema

| Field | Type | Required | Properties |
| ----- | ---- | -------- | ---------- |
| constraint | Object | Yes | if fieldType is "dimension", one of: [ConstraintSchemaInOrNotIn, ConstraintSchemaNullOrNotNull, ConstraintSchemaMatches] |
|  |  |  | if fieldType is "date", one of: [ConstraintSchemaBetween, ConstraintSchemaLessThanOrGreaterThanOrEqualTo, ConstraintSchemaNullOrNotNull] |
|  |  |  | if fieldType is "measure", one of: [ConstraintSchemaEqualOrNotEqual, ConstraintSchemaBetween, ConstraintSchemaLessThanOrGreaterThanOrEqualTo, ConstraintSchemaLessThanOrGreaterThan, ConstraintSchemaNullOrNotNull] |
| datasetApiName | String | Yes | length > 1 |
| fieldType | String | Yes | one of: ["dimension", "date", "measure"] |
| lensApiName | String | Yes | length > 1 |
| Version | Number | Yes | value is 1 |

### ConstraintSchemaInOrNotIn

| Field | Type | Required | Properties |
| ----- | ---- | -------- | ---------- |
| constraint.operator | string | Yes | 	one of: ["in", "not in"] |
| constraint.value | array of strings | Yes |  |

### ConstraintSchemaNullOrNotNull

| Field | Type | Required | Properties |
| ----- | ---- | -------- | ---------- |
| constraint.operator | string | Yes | 	one of: ["isnull", "isnotnull"] |
| constraint.value | null | Yes |  |

### ConstraintSchemaMatches

| Field | Type | Required | Properties |
| ----- | ---- | -------- | ---------- |
| constraint.operator | string | Yes | value is "matches" |
| constraint.value | string | Yes |  |

### ConstraintSchemaBetween

| Field | Type | Required | Properties |
| ----- | ---- | -------- | ---------- |
| constraint.operator | string | Yes | value is ">=<=" |
| constraint.value | object | Yes |  |
| constraint.value.end | number, null | Yes |  |
| constraint.value.start | number, null | Yes |  |

### ConstraintSchemaLessThanOrGreaterThanOrEqualTo

| Field | Type | Required | Properties |
| ----- | ---- | -------- | ---------- |
| constraint.operator | string | Yes | one of: ["<=", ">="] |
| constraint.value | number, null | Yes |  |

### ConstraintSchemaEqualOrNotEqual

| Field | Type | Required | Properties |
| ----- | ---- | -------- | ---------- |
| constraint.operator | string | Yes | one of: ["==", "!="] |
| constraint.value | number, null | Yes |  |

### ConstraintSchemaLessThanOrGreaterThan

| Field | Type | Required | Properties |
| ----- | ---- | -------- | ---------- |
| constraint.operator | string | Yes | 	one of: ["<", ">"] |
| constraint.value | number, null | Yes |  |
