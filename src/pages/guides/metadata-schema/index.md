---
title: Metadata Schemas developer guide
description: How to structure a metadata schema and use it with the Metadata API
---

## Introduction

Metadata schemas specify the metadata properties that matter to your organization, establishing
their names, types, constraints, and other details. They are intended to serve as the shared contract
for every service and client that reads or writes metadata. A metadata schema is a
[JSON Schema](https://json-schema.org/draft/2020-12) document extended with AEM-specific keywords.

Metadata schemas define metadata for file assets, folders, collections and smart collections, and content fragments.

This guide also covers how a schema shapes what you can read and change through each entity type's metadata API, and the properties that exist independent of any schema. It supplements — rather than replaces — each entity type's own OpenAPI reference, which remains the source of truth for exact request/response contracts and status codes.

The underlying Metadata Schema API is experimental. Details in this guide are subject to change.

Properties in a metadata response are either fixed (`repositoryMetadata`, described in the [Repository metadata](#repository-metadata) appendix) or schema-controlled — determined by the schema described throughout this guide. See [Reading and writing metadata](#reading-and-writing-metadata) for the full response shape and how to read and update it.

## System schemas and custom schemas

AEM ships with a set of **system schemas** covering common properties out of the box — including a default schema (used automatically when no `schemaId` is specified). System schemas cannot be modified or deleted, but they may be extended. Alongside them, AEM ships **definition files** (`x-schemaType: "DEFINITIONS"`) — reusable property libraries that cannot be selected as a `schemaId` directly, but can be `$ref`-referenced from any schema.

A **custom schema** is one your organization creates and manages through the Metadata Schema API — it appears with `source: "CUSTOM"` in the listing response.

The specific system schemas and definition files available, and their identifiers, are not enumerated here — they may be added to or revised over time. `GET /adobe/metadataSchemas` returns the current, authoritative list (with a `title` for each); `GET /adobe/metadataSchemas/default` returns the current default schema document. Fetch an individual schema (`GET /adobe/metadataSchemas/{schemaId}`) to see its `description`.

## Why create a custom schema?

The default schema covers standard AEM and Dublin Core properties and accepts any additional stored property through implicit discovery. Custom schemas become valuable when you need:

- **Your property definitions**: fields particular to your content lifecycle, project management, or rights management workflows
- **Controlled vocabularies**: restrict a field to a defined enumeration of allowed values, or link it to an AEM tag tree
- **Dependent value sets**: make the allowed values for one field depend on the value of another
- **Consistent typing**: guarantee a property is always a specific type across all assets, rather than inferring the type from the value written in each PATCH request
- **Strict property control**: expose only explicitly declared properties; prevent unknown fields from appearing in API responses
- **Tooling integration**: drive AI-assisted metadata validation and editing, metadata import/export, metadata form generation, and future search index configuration from a well-typed schema definition

## Metadata Schema API

Schemas are managed through the Metadata Schema API at base path `/adobe/metadataSchemas` — `GET`/`HEAD` to list, retrieve, or check a schema (including the default schema at `/default`), `POST` to create or `POST /validate` to check a schema without persisting it, `PUT` to update, and `DELETE` to remove a custom schema.

For the full request and response contracts, required headers, and status codes for each operation, see the Metadata Schema API reference.

## Schema document anatomy

A schema document is a JSON object with the following structure:

```json
{
  "$id": "myorg.schema.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "x-schemaType": "SCHEMA",
  "title": "My Organization Schema",
  "type": "object",
  "properties": {
    "assetMetadata": {
      "type": "object",
      "properties": {
        "myorg:status": { "type": "string" }
      }
    }
  }
}
```

### Required fields

| Field | Notes |
|-------|-------|
| `$id` | Unique identifier. By convention, schema files end in `.schema.json` and definition files end in `.defs.json`. Custom IDs must not begin with `aem` or `adobe`, and must not be `default` or `validate`. |
| `x-schemaType` | `"SCHEMA"` for a selectable metadata schema. `"DEFINITIONS"` for a reusable property library (cannot be used as a `schemaId`). |

### Optional fields

| Field | Purpose |
|-------|---------|
| `$schema` | JSON Schema dialect. Always `"https://json-schema.org/draft/2020-12/schema"`. |
| `type` | Always `"object"` for a schema or definitions document. |
| `title` | Human-readable name; returned in listing responses. |
| `description` | Description of the schema's scope and purpose. |
| `properties` | Top-level metadata groups (see below). |
| `allOf` | JSON Schema composition — used to extend other schemas. |
| `$defs` | Local reusable definitions within this document. |
| `x-implicitProperties` | Controls automatic discovery of undeclared stored properties. |

### Property groups

Properties are nested inside named groups. The built-in group names are:

| Group | Entity type |
|-------|------------|
| `assetMetadata` | File assets and content fragments |
| `folderMetadata` | Folders and directories |
| `collectionMetadata` | Collections and smart collections |

The group name appears as a top-level key in the metadata response alongside `repositoryMetadata`.

> **Note**: The group name is a schema-level label only. A property declared under `assetMetadata` and the same property declared under a custom group name are both read from and written to the same storage location. Grouping affects the API surface but not storage.

Schema authors don't choose where a property is stored — its storage location is derived automatically from the property name (and, for certain well-known properties, the entity type). This is transparent to schema authors and API clients and has no effect on how you declare or use a property.

### Schemas vs definition files

A **schema** (`x-schemaType: "SCHEMA"`) is a complete document that clients may select via the `schemaId` parameter. A **definition file** (`x-schemaType: "DEFINITIONS"`) is a reusable library of property specifications that can be `$ref`-referenced from any schema.

Separating property specifications into definition files is strongly recommended when multiple schemas share the same property: the property's type, value set, and characteristics are declared once and shared everywhere. Properties that are referenced from a `.defs.json` file will be consistent across all schemas that use them.

## Defining properties

Properties are declared inside a group object's `properties` map. Each property name maps to its JSON Schema definition.

Only the fields and `x-*` keywords documented in this guide are read by the system. Other JSON Schema keywords (for example `pattern`, `minLength`, or `enum`) may appear in a property definition without causing an error, but they are not enforced or otherwise acted on — they pass through silently ignored.

### Property types and storage

The AEM Assets author repository is built on JCR (Java Content Repository), which has its own, more limited set of storage types. The table below is provided as reference for readers who need to understand how a property's representation in the API relates to how it's actually stored — most schema authors won't need it.

| JSON Schema type | Format | JCR storage type | Notes |
|-----------------|--------|-------------------|-------|
| `string` | — | `STRING` | Direct mapping |
| `string` | `date-time` | `DATE` (Calendar) | Write accepts ISO 8601 datetime or date-only strings; read returns an ISO 8601 datetime |
| `string` | `date` | `DATE` (Calendar) | Write requires strict `YYYY-MM-DD`; datetime strings are rejected. Round-trips without change |
| `integer` | — | `LONG` | |
| `integer` | `integer` | `LONG` | Format accepted; no effect on storage |
| `number` | — | `DOUBLE` | A previously stored `LONG` value is read back as a number but written as `DOUBLE` |
| `number` | `decimal` | `DOUBLE` | Explicit floating-point |
| `number` | `double` | `DOUBLE` | Alias for `decimal`; same storage behavior |
| `number` | `bigdecimal` | `DECIMAL` (BigDecimal) | High-precision; preserves precision that `DOUBLE` storage would lose |
| `boolean` | — | `BOOLEAN` | |
| `array` | — | Multi-value property | Items converted per `items.type` |
| `object` | — | Child node | Objects are stored as JCR child nodes — see "Object properties" below |

To preserve the integer vs. decimal distinction across round-trips, use `type: "integer"` for whole numbers and `type: "number"` with `format: "decimal"` for floating-point values. A bare `type: "number"` preserves the type on read but converts to `DOUBLE` on write.

Only the `format` values listed above are accepted by the validator. Any other value causes schema creation or update to fail with an `UNKNOWN_FORMAT` error. The full list of accepted values: `date-time`, `date`, `decimal`, `double`, `bigdecimal`, `integer`.

### Simple scalar properties

```json
"assetMetadata": {
  "type": "object",
  "properties": {
    "myorg:projectCode": {
      "title": "Project Code",
      "type": "string"
    },
    "myorg:priority": {
      "title": "Priority",
      "type": "integer"
    },
    "myorg:confidence": {
      "title": "Confidence Score",
      "type": "number",
      "format": "decimal"
    },
    "myorg:approved": {
      "title": "Approved",
      "type": "boolean"
    }
  }
}
```

### Date and time properties

Use `type: "string"` with a `format` keyword for temporal values:

```json
"myorg:approvalDate": {
  "title": "Approval Date",
  "type": "string",
  "format": "date-time"
}
```

See "Property types and storage" above for how `date-time` and `date` are validated and represented on read and write.

### Array properties

```json
"myorg:tags": {
  "title": "Tags",
  "type": "array",
  "items": { "type": "string" }
}
```

Array items are individually converted using the same type mappings as scalar properties. See [Reading and writing metadata](#reading-and-writing-metadata) for PATCH semantics, including value-based array operations.

### Object properties

Object properties are stored as a named child entry beneath the entity's metadata. Each sub-property on the object is a separate stored value within that entry.

```json
"myorg:location": {
  "title": "Location",
  "type": "object",
  "properties": {
    "myorg:city":    { "type": "string" },
    "myorg:country": { "type": "string" }
  }
}
```

See [Reading and writing metadata](#reading-and-writing-metadata) for how PATCH targets a whole object versus an individual sub-property.

### Arrays of objects

When `items.type` is `"object"`, each array item is stored as a named child entry. Each item in a GET response includes a key field — `@key` by default — carrying the entry's storage name. PATCH operations address items by key, not by numeric position.

```json
"myorg:reviewers": {
  "title": "Reviewers",
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "myorg:reviewerId": { "type": "string" },
      "myorg:decision":   { "type": "string" }
    }
  }
}
```

GET response:

```json
"myorg:reviewers": [
  { "@key": "item0", "myorg:reviewerId": "user1@example.com", "myorg:decision": "approved" },
  { "@key": "item1", "myorg:reviewerId": "user2@example.com", "myorg:decision": "pending" }
]
```

PATCH operations:

```json
[
  { "op": "add",    "path": "/assetMetadata/myorg:reviewers/-",
    "value": { "@key": "item2", "myorg:reviewerId": "user3@example.com", "myorg:decision": "pending" } },
  { "op": "remove", "path": "/assetMetadata/myorg:reviewers/item0" },
  { "op": "replace","path": "/assetMetadata/myorg:reviewers/item1/myorg:decision", "value": "approved" }
]
```

Use `/prop/-` to append a new item with an auto-generated key, or `/prop/<key>` to upsert by a specific key.

## Extending system schemas

Use `allOf` to extend a system schema, inheriting all its properties. (The schema identifiers used below are examples — confirm the current ones with `GET /adobe/metadataSchemas`.)

```json
{
  "$id": "myorg.schema.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "x-schemaType": "SCHEMA",
  "title": "My Organization Schema",
  "type": "object",
  "allOf": [
    { "$ref": "aem-standard-1.schema.json" },
    {
      "properties": {
        "assetMetadata": {
          "type": "object",
          "properties": {
            "myorg:projectCode": { "type": "string" },
            "myorg:status": { "type": "string" }
          }
        }
      }
    }
  ]
}
```

This inherits all `aem-standard-1.schema.json` properties (Dublin Core, DAM, XMP, and more) and adds `myorg:projectCode` and `myorg:status`.

**Which system schema to extend?**

| Extend | When |
|--------|------|
| The default schema (e.g. `aem-default-1.schema.json`) | You want implicit property discovery in addition to declared properties |
| The standard schema (e.g. `aem-standard-1.schema.json`) | You want only explicitly declared properties with no automatic inclusion of stored but undeclared properties |

### Using definition files for shared properties

Create a definition file to declare property details once, then `$ref` the definition from multiple schemas:

```json
// myorg.defs.json — declare the property details
{
  "$id": "myorg.defs.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "x-schemaType": "DEFINITIONS",
  "$defs": {
    "myorg:status": {
      "title": "Asset Status",
      "type": "string",
      "x-valueSet": [
        { "value": "draft",    "label": { "default": "Draft" } },
        { "value": "review",   "label": { "default": "Under Review" } },
        { "value": "approved", "label": { "default": "Approved" } }
      ]
    }
  }
}
```

```json
// myorg.schema.json — reference the definition
{
  "properties": {
    "assetMetadata": {
      "properties": {
        "myorg:status": { "$ref": "myorg.defs.json#/$defs/myorg:status" }
      }
    }
  }
}
```

Create the definition file first with `POST /adobe/metadataSchemas`, then create the schema that references it.

## Advanced schema features

### Read-only properties

Properties marked `readOnly: true` cannot be modified through the Metadata API. A PATCH targeting a read-only property fails the entire request:

```json
"myorg:internalId": {
  "title": "Internal ID",
  "type": "string",
  "readOnly": true
}
```

### Computed properties

`x-function` marks a property as computed at read time. Computed properties are always read-only:

```json
"assetId": { "type": "string", "x-function": "getAssetId" }
```

Available functions:

| Function | Returns |
|----------|---------|
| `getAssetId` | URN identifier (`urn:aaid:aem:...`) |
| `getEntityName` | Entity name |
| `getEntityPath` | Storage path |
| `getEntityType` | `file`, `directory`, `collection`, or `contentFragment` |
| `getEntityParentPath` | Parent storage path |
| `getAncestorIds` | Array of ancestor asset IDs |
| `getParentId` | Asset ID of the containing folder |
| `getAssetETag` | ETag for caching |
| `getTagInfo` | Enriched AEM tag objects (title, description, path) |

### Implicit properties

`x-implicitProperties` enables automatic discovery and inclusion of stored properties not explicitly declared in the schema:

```json
{
  "x-implicitProperties": {
    "enabled": true,
    "excludePatterns": [".*/(?:cq|jcr|oak|rep|sling):[^/]*$"],
    "defaultObject": "/assetMetadata"
  }
}
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `enabled` | boolean | `false` | Enable implicit discovery |
| `excludePatterns` | string[] | `[]` | Regex patterns matched against the storage path; matching properties are excluded |
| `defaultObject` | string | `"/assetMetadata"` | The group under which discovered properties appear in API responses and where unrecognized PATCH properties are written |

With `enabled: true`, GET responses include all stored metadata properties — not just those declared in the schema. Undeclared PATCH properties are written to the storage location corresponding to `defaultObject`.

Custom schemas that extend the default schema automatically inherit `x-implicitProperties`. To disable it in an extending schema:

```json
{ "x-implicitProperties": { "enabled": false } }
```

A custom schema that declares its own `x-implicitProperties` **replaces** the inherited configuration entirely by default — `excludePatterns` is not merged with what an extended schema declares.

To merge instead of replacing, add a `$ref` inside your `x-implicitProperties` block pointing at the block you want to merge with. The referenced `excludePatterns` and your own are concatenated (not deduplicated), and your own `enabled`/`defaultObject` take precedence over the referenced ones:

```json
"x-implicitProperties": {
  "$ref": "aem-default-1.schema.json#/x-implicitProperties",
  "excludePatterns": ["additional-pattern-here"]
}
```

Setting `enabled: false` always disables implicit discovery immediately, regardless of any `$ref` — disabling short-circuits the merge rather than combining with it.

> **Trade-off**: Implicit discovery is convenient when assets carry properties from varied sources (asset processing, third-party tools, manual editing). The drawback is that the type of an implicitly discovered property on write is inferred from the JSON value in the PATCH body, not from a declaration — which can result in inconsistent stored property types across different assets.

### Object array extensions

Three keywords control how object arrays (arrays with `items.type: "object"`) are stored and surfaced:

> **Every write to an object array is a full replace.** Whichever entries currently match the array (all of them, or the subset matching `x-keyPrefix`) are removed and rewritten from scratch on every PATCH — including a PATCH that only adds or removes a single item. This has a direct consequence for auto-generated keys, described below.

**`x-keyPrefix`** — restricts the array to a named subset of entries:

```json
"dam:colorDistribution": {
  "type": "array",
  "x-keyPrefix": "color",
  "items": { "type": "object", "properties": { ... } }
}
```

On read, only entries named `color0`, `color1`, … are included. New items appended with `/prop/-` are auto-named `color<N>`, where N is the item's *position in the array at write time* — not "one after the current highest number." Because every write rewrites the whole array (see above), a gap left by earlier deletions can result in a new item getting a *lower*-numbered key than existing items: if the array currently holds only `color3` and `color5` and a new item is appended without an explicit key, the array has three items at positions 0, 1, 2 — the new item lands at position 2 and is assigned `color2`, giving `[color3, color5, color2]`, not `[color3, color5, color6]`. To guarantee a predictable key, supply it explicitly: `{ "@key": "color6", ... }`.

**`x-keyAs`** — exposes the storage key (the JCR child node name in this implementation) under a semantic field name instead of the default `@key`:

```json
"xcm:machineKeywords": {
  "type": "array",
  "x-keyAs": "id",
  "items": { "type": "object", "properties": { ... } }
}
```

Each item in GET responses carries `"id": "<key>"`. In PATCH operations, supply that value via `id` (or `@key`, which takes precedence if both are present).

Use `x-keyAs` when the storage key has a meaningful semantic role (for example, a keyword identifier) that deserves a proper field name rather than the generic `@key`.

**`x-orderBy` / `x-orderDirection`** — sort the array on read:

```json
"xcm:machineKeywords": {
  "type": "array",
  "x-orderBy": "confidence",
  "x-orderDirection": "desc",
  "items": { ... }
}
```

`x-orderBy` may be any item property name, or the special value `"nodeName"` to sort by the entry's storage key. `x-orderDirection` is `"asc"` (default) or `"desc"`.

### Renaming stored properties — x-storedAs

By default, each item property name is also the name used in storage. Use `x-storedAs` when the API field name should differ from the stored name — for example, to expose an existing differently-named property under a cleaner API name, or to hide storage-level naming conventions from clients:

```json
"xcm:machineKeywords": {
  "type": "array",
  "items": {
    "type": "object",
    "properties": {
      "value": {
        "type": "string",
        "x-storedAs": "name"
      },
      "confidence": { "type": "number" }
    }
  }
}
```

On read, the stored property `name` on each entry is returned as the API field `value`. On write, a value supplied as `value` in the PATCH body is written to the stored property `name`.

`x-storedAs` applies only to properties inside an object array's `items.properties`, or the `properties` of an object-typed property. It has no effect on top-level or scalar properties.

## Pass-through keywords

he keywords in this section are stored and returned by the API verbatim — the Metadata Schema API and the Metadata API do not interpret or act on them. They exist for API clients, such as a metadata form builder or a metadata validator, to read and use.

### Value sets

`x-valueSet` declares a fixed set of valid values with optional multilingual labels:

```json
"myorg:visibility": {
  "title": "Visibility",
  "type": "string",
  "x-valueSet": [
    { "value": "public",       "label": { "default": "Public",       "fr-FR": "Public" } },
    { "value": "internal",     "label": { "default": "Internal",     "fr-FR": "Interne" } },
    { "value": "confidential", "label": { "default": "Confidential", "fr-FR": "Confidentiel" } }
  ]
}
```

Each label object requires a `default` key and accepts any number of BCP 47 locale codes. Client-side resolution: try exact locale match → language-only match → `default`.

For value sets shared across properties, extract the array to a standalone JSON file and reference it:

```json
"myorg:status": {
  "type": "string",
  "x-valueSet": { "$ref": "workflow-statuses.json" }
}
```

The standalone file contains only the array — no `$id`, `$schema`, or `$defs` wrapper.

### Dependent value sets

`x-condition` makes the allowed values for a property depend on another property's value:

```json
"myorg:country": {
  "title": "Country",
  "type": "string",
  "x-valueSet": [
    { "value": "USA", "label": { "default": "United States" } },
    { "value": "CAN", "label": { "default": "Canada" } }
  ]
},
"myorg:region": {
  "title": "State / Province",
  "type": "string",
  "x-condition": {
    "controlProperty": "assetMetadata/myorg:country",
    "cases": {
      "USA": { "$ref": "myorg-enums.defs.json#/$defs/us-states" },
      "CAN": { "$ref": "myorg-enums.defs.json#/$defs/ca-provinces" }
    }
  }
}
```

`controlProperty` is the full API-level path from the response root. The client reads the control value, selects the matching case, and applies that value set for validation and display. Each case `$ref` points to a standalone value set file (same format as standalone `x-valueSet`).
### Vocabulary (AEM tags)

`x-vocabulary` designates a property's valid values as the tags contained within an AEM tag hierarchy, rather than an explicit list:

```json
"myorg:productCategory": {
  "title": "Product Category",
  "type": "string",
  "x-vocabulary": { "root": "<tag-id>" }
}
```

`root` specifies the identifier of the root of the tag hierarchy of valid values for the property. Labels and the resolved value list come from the AEM tags themselves, at whatever time the consuming process resolves them.

### UI hints

`x-ui` provides rendering hints for metadata editing interfaces:

```json
"myorg:description": {
  "type": "string",
  "x-ui": {
    "widget": "textarea",
    "hidden": false
  }
}
```

When a schema and a definition file both declare `x-ui` for the same property, individual keys are merged — a key set in the schema overrides the same key from the definition, but keys present only in the definition are preserved.

## Reading and writing metadata

### Response envelope

Every metadata response contains three top-level fields, regardless of entity type:

- **An entity identifier** (`assetId`) — the entity's unique identifier, a URN of the form `urn:aaid:aem:{uuid}`. Used for files, folders, collections, and content fragments alike.
- **`repositoryMetadata`** — infrastructure properties computed and maintained by AEM, independent of any schema. Always read-only, always present. See [Repository metadata](#repository-metadata) for the complete property list.
- **A schema-controlled group** — the properties determined by everything covered so far in this guide. Its field name varies by entity type: `assetMetadata` for file assets and content fragments, `folderMetadata` for folders and directories, `collectionMetadata` for collections and smart collections.

```json
{
  "assetId": "urn:aaid:aem:75ebc086-fad4-4682-bc3d-41f3f8a48d63",
  "assetMetadata": {
    "dc:title": "City Skyline at Dusk",
    "dc:description": "Photography for the Q3 campaign",
    "dc:subject": ["photography", "urban"]
  },
  "repositoryMetadata": {
    "repo:assetClass": "file",
    "repo:assetId": "urn:aaid:aem:75ebc086-fad4-4682-bc3d-41f3f8a48d63",
    "repo:createDate": "2025-11-01T09:00:00.000Z",
    "repo:etag": "\"a3f8b2c1\"",
    "repo:modifyDate": "2026-01-15T14:30:00.000Z",
    "repo:name": "city-skyline.jpg",
    "repo:path": "/content/dam/marketing/city-skyline.jpg"
  }
}
```

### PATCH path resolution

Paths mirror the response structure: properties in the schema-controlled group use `/assetMetadata/<property>` (or `/folderMetadata/...`, `/collectionMetadata/...`). `repositoryMetadata` paths are accepted in the syntax but always fail — repository metadata is unconditionally read-only.

The system resolves a path as follows:

1. **Exact match** (`/assetMetadata/dc:title`) — used directly.
2. **Short path** (`/dc:title`) — resolved to the first schema property whose name matches the last segment. Convenient shorthand; use the full path when the property appears in more than one group.
3. **Wrong group prefix** — if the property exists in a different group than specified, the entire request fails and names the correct path.
4. **Unknown property** — if `x-implicitProperties` is enabled in the active schema, the value is written to the default location and appears in the schema-controlled group on subsequent reads. Otherwise the request fails.

### PATCH operations

All standard RFC 6902 operations are supported (`add`, `remove`, `replace`, `test`, `move`, `copy`), plus five AEM extensions. These extensions aren't yet part of the published Asset Metadata API's OpenAPI `op` enum — they're planned for addition there and to the other entity types' metadata APIs as those specs are built out. They work today; each solves a specific pain point standard JSON Patch doesn't:

| `op` | Path form | Effect | Why it's useful |
|------|-----------|--------|-----------------|
| `set` | `/prop` | Create or overwrite; no error if absent | Upsert a property without a prior GET to know whether it already exists — `add` and `replace` each fail if the property is in the wrong state. |
| `removeIfExists` | `/prop` | Remove if present; no-op if absent | Idempotent delete — clearing a property that may or may not be set, without a failed op aborting the whole atomic PATCH. |
| `addUnique` | `/prop` | Append each value not already present in the array | Add tags/keywords by value without fetching the current array first to check for duplicates client-side. |
| `removeFirst` | `/prop` | Remove the first occurrence of each value | Remove a specific value from an array by content, when you don't know (or don't want to track) its index. |
| `removeAll` | `/prop` | Remove all occurrences of each value | Same, but removes every matching occurrence rather than just the first. |

`addUnique`, `removeFirst`, and `removeAll` each accept a single value or a JSON array of values; each element is processed individually:

```json
[
  { "op": "addUnique", "path": "/assetMetadata/dc:subject", "value": "landscape" },
  { "op": "addUnique", "path": "/assetMetadata/dc:subject", "value": ["travel", "nature"] },
  { "op": "removeAll", "path": "/assetMetadata/dc:subject", "value": "draft" }
]
```

### Updating object properties

Properties defined as objects in the schema support two PATCH styles:

- **Whole-object**: supply an object value — only the supplied sub-properties are written; others already stored are left unchanged (a partial update, not a full replacement).
- **Per-field**: target `/prop/field` with a scalar value — only that field is updated.

```json
[
  { "op": "replace", "path": "/assetMetadata/myorg:location",
    "value": { "myorg:city": "San Francisco", "myorg:country": "US" } },

  { "op": "replace", "path": "/assetMetadata/myorg:location/myorg:city",
    "value": "Oakland" }
]
```

### Keyed object arrays

Properties declared as arrays of objects in the schema are addressed by key, not by position. Each item in a GET response includes a key field — `@key` by default, or the field named by `x-keyAs` in the schema. See [Object array extensions](#object-array-extensions) for how the schema shapes this.

| `op` | Path | Effect |
|------|------|--------|
| `add` | `/prop/-` | Append; `@key` in the value names the entry, or auto-generated if absent |
| `add` | `/prop/<key>` | Upsert — replace if the key exists, append if it does not |
| `replace` | `/prop/<key>/<field>` | Update a single field within an item |
| `remove` | `/prop/<key>` | Remove item by key |
| `removeIfExists` | `/prop/<key>` | Remove item by key; no-op if the key is absent |
| `set` | `/prop` | Replace the entire array |

Positional addressing (`/prop/0`, `/prop/1`) is not supported for keyed arrays.

## Complete example

A custom schema that extends the standard system schema and adds organization-specific properties:

```json
{
  "$id": "acme-corp.schema.json",
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "x-schemaType": "SCHEMA",
  "title": "ACME Corp Metadata Schema",
  "description": "Organization metadata schema for ACME Corp assets",
  "type": "object",
  "allOf": [
    { "$ref": "aem-standard-1.schema.json" },
    {
      "properties": {
        "assetMetadata": {
          "type": "object",
          "properties": {
            "acme:projectCode": {
              "title": "Project Code",
              "description": "Internal project identifier",
              "type": "string"
            },
            "acme:status": {
              "title": "Asset Status",
              "type": "string",
              "x-valueSet": [
                { "value": "draft",    "label": { "default": "Draft" } },
                { "value": "review",   "label": { "default": "Under Review" } },
                { "value": "approved", "label": { "default": "Approved" } }
              ]
            },
            "acme:approvalDate": {
              "title": "Approval Date",
              "type": "string",
              "format": "date-time"
            },
            "acme:territory": {
              "title": "Territory",
              "type": "string",
              "x-valueSet": [
                { "value": "AMER", "label": { "default": "Americas" } },
                { "value": "EMEA", "label": { "default": "EMEA" } },
                { "value": "APAC", "label": { "default": "Asia Pacific" } }
              ]
            },
            "acme:region": {
              "title": "Region",
              "type": "string",
              "x-condition": {
                "controlProperty": "assetMetadata/acme:territory",
                "cases": {
                  "AMER": { "$ref": "acme-enums.defs.json#/$defs/amer-regions" },
                  "EMEA": { "$ref": "acme-enums.defs.json#/$defs/emea-regions" }
                }
              }
            },
            "acme:keywords": {
              "title": "Keywords",
              "type": "array",
              "items": { "type": "string" }
            }
          }
        }
      }
    }
  ]
}
```

**Schema creation workflow**

```
# 1. Validate
POST /adobe/metadataSchemas/validate
→ 200 OK, { "status": 200 }

# 2. Create
POST /adobe/metadataSchemas
→ 201 Created, ETag: "a3f8b2c1"
   Location: /adobe/metadataSchemas/acme-corp.schema.json

# 3. Verify the schema is available
GET /adobe/metadataSchemas/acme-corp.schema.json
→ 200 OK

# 4. Read an asset using the new schema
GET /adobe/assets/{assetId}/metadata?schemaId=acme-corp.schema.json
→ 200 OK
```

## Repository metadata

`repositoryMetadata` properties are computed and maintained by AEM — always read-only, always present, unaffected by which schema is in effect. Attempting to modify one through PATCH fails the entire request. No changes are applied.

Each table below describes a category of repository metadata. For this implementation, most of this metadata is populated using JCR source information, either directly or indirectly — a few properties, noted individually, come from outside JCR entirely. This information is included in the **JCR source** column for reference. Future implementations may use different sources.

### Node identity and structure

| Property | JCR source | Notes |
|----------|------------|-------|
| `repo:name` | The entity node's own name | Not a stored property. The entity's name — the filename or folder name. |
| `repo:path` | The entity node's own path | Not a stored property. The full absolute path to the entity (e.g. `/content/dam/marketing/photo.jpg`). |
| `repo:assetId` | Derived from the entity node's JCR UUID | Formatted as a URN: `urn:aaid:aem:{uuid}`. This is the canonical identifier for use in API requests. |
| `repo:assetClass` | Computed from node type, `sling:resourceType`, and a `contentFragment` flag under `jcr:content` | Entity type: `file`, `directory`, `collection`, or `contentFragment`. Resolves to `unknown` for anything outside `/content` or `/content/dam`. |
| `repo:parent` | Derived from the parent node's UUID | ID (URN) of the direct parent folder. Omitted if there is no accessible parent. |
| `repo:ancestors` | Walks parent nodes upward to `/content/dam` or `/content` | Array of IDs (URNs) for all ancestor folders up to `/content/dam` or `/content`. The last item is the ID of the direct parent. |

### Timestamps and authorship

| Property | JCR source | Notes |
|----------|------------|-------|
| `repo:createDate` | `jcr:created` on the entity node | ISO 8601 timestamp when the entity was created. |
| `repo:createdBy` | `jcr:createdBy` on the entity node | User ID of the creator. |
| `repo:modifyDate` | Collections: `jcr:lastModified` on the entity node itself. All other entity types: `jcr:content/jcr:lastModified` | ISO 8601 timestamp of the most recent modification. |
| `repo:modifiedBy` | Collections: `jcr:lastModifiedBy` on the entity node. All other entity types: `jcr:content/jcr:lastModifiedBy` | User ID of the last modifier. |

### Format, size, and content

| Property | JCR source | Notes |
|----------|------------|-------|
| `dc:format` | Files: `jcr:content/metadata/dc:format`. Content fragments and folders/collections: not read from storage | MIME type. For files: stored MIME type (e.g. `image/jpeg`). For content fragments: always `application/vnd.adobe.contentfragment`. For folders and collections: a fixed value derived from entity type. |
| `repo:size` | `jcr:content/metadata/dam:size` | Binary size in bytes. |
| `dam:sha1` | `jcr:content/metadata/dam:sha1` | SHA-1 hash of the original binary. |
| `tiff:imageWidth` | `jcr:content/metadata/tiff:ImageWidth` | Width in pixels. Omitted if not set (non-image assets). |
| `tiff:imageLength` | `jcr:content/metadata/tiff:ImageLength` | Height in pixels. Omitted if not set. |

### Publish state and lifecycle

| Property | JCR source | Notes |
|----------|------------|-------|
| `isPublishedToAemPublish` | Computed from `jcr:content/cq:lastReplicationAction` | `true` when the asset has been activated to AEM Publish (the underlying value is `Activate`); `false` otherwise (not omitted). |
| `isPublishedToDynamicMedia` | Computed from `jcr:content/metadata/dam:scene7FileStatus` | `true` when the asset's Scene7 publish status is `PublishComplete`; `false` otherwise. |
| `aem:published` | `jcr:content/cq:lastReplicated` | ISO 8601 timestamp of the most recent publish to AEM Publish. |
| `lastPublishedToAemPublish` | `jcr:content/cq:lastReplicated` | ISO 8601 timestamp of the most recent publish to AEM Publish. Same underlying property as `aem:published` — the two are always equal. |
| `lastPublishedToDynamicMedia` | `jcr:content/metadata/dam:scene7PublishTimeStamp` | ISO 8601 timestamp of the most recent publish to Dynamic Media. |
| `aem:assetState` | `jcr:content/dam:assetState` | Processing pipeline state: `Processing` or `Processed`. |
| `aem:checkedOutBy` | `jcr:content/cq:drivelock` | User ID of the user who has checked out the asset. Omitted if the asset is not checked out. |
| `repo:state` | Computed: reads `jcr:content/cq:discardState` on the entity; if absent, walks parent folders for a `DISCARDED` state | Discard lifecycle state. Omitted entirely when the value is `ACTIVE` (the default). When a parent folder is in a `DISCARDED` state, this returns `DISCARDED_PARENT`. |

### Repository identity and caching

| Property | JCR source | Notes |
|----------|------------|-------|
| `repo:repositoryId` | Not derived from storage — read from environment/deployment configuration | A stable identifier for the AEM repository instance (e.g. `author-p12345-e123456.adobeaemcloud.com`). Omitted if not configured. |
| `repo:etag` | Not a stored property | Computed as a hash over the schema ID, entity path, and the full serialized metadata response. Changes whenever any of those change. Use with `If-None-Match` (GET) or `If-Match` (PATCH). |

### Dynamic Media (Scene7)

Present only for assets synchronized with Dynamic Media; omitted when not set. All map 1:1 to a property under `jcr:content/metadata/`.

| Property | JCR source | Source |
|----------|------------|--------|
| `repo:scene7Domain` | `jcr:content/metadata/dam:scene7Domain` | Scene7 delivery domain |
| `repo:scene7File` | `jcr:content/metadata/dam:scene7File` | Scene7 file path |
| `repo:scene7FileStatus` | `jcr:content/metadata/dam:scene7FileStatus` | Scene7 synchronization status (e.g. `PublishComplete`) |
| `repo:scene7Folder` | `jcr:content/metadata/dam:scene7Folder` | Scene7 folder path |
| `repo:scene7FontStyle` | `jcr:content/metadata/dam:scene7FontStyle` | Font style (font assets) |
| `repo:scene7FontType` | `jcr:content/metadata/dam:scene7FontType` | Font type (font assets) |
| `repo:scene7LastModified` | `jcr:content/metadata/dam:scene7LastModified` | Timestamp of last Scene7 modification |
| `repo:scene7Name` | `jcr:content/metadata/dam:scene7Name` | Scene7 asset name |
| `repo:scene7RTFName` | `jcr:content/metadata/dam:scene7RTFName` | RTF font name (font assets) |
| `repo:scene7Type` | `jcr:content/metadata/dam:scene7Type` | Scene7 asset type |

### Malware scanning

Present only when the asset has been scanned; omitted when not set. All map 1:1 to a property directly under `jcr:content` — note this is *not* under the `metadata` child node, unlike the Dynamic Media properties above.

| Property | JCR source | Notes |
|----------|------------|-------|
| `dam:avScanStatus` | `jcr:content/dam:avScanStatus` | Scan outcome (e.g. `clean`, `infected`) |
| `dam:avInfectedReason` | `jcr:content/dam:avInfectedReason` | Reason for an infected status |
| `dam:avScanner` | `jcr:content/dam:avScanner` | Name of the antivirus scanner |
| `dam:avScanStartDate` | `jcr:content/dam:avScanStartDate` | ISO 8601 timestamp when the scan began |
| `dam:avScanEndDate` | `jcr:content/dam:avScanEndDate` | ISO 8601 timestamp when the scan completed |
| `dam:avScanDuration` | `jcr:content/dam:avScanDuration` | Duration in milliseconds |
| `dam:avEngineVersion` | `jcr:content/dam:avEngineVersion` | Antivirus engine version string |
| `dam:avOriginalAssetPath` | `jcr:content/dam:avOriginalAssetPath` | Original path of a quarantined asset |
| `dam:originalQuarantinedPath` | `jcr:content/dam:originalQuarantinedPath` | Quarantine path |
