---
title: POI import
description: API calls to import points of interest (POI) from a preloaded TSV file.
---

# POI import

POI import allows creating many [places](work-with-poi.md) at once from a spreadsheet file. Upload the XLSX, XLS, or CSV file with the [/data/spreadsheet/parse](../../commons/data.md#dataspreadsheetparse) method first: it converts the spreadsheet to a TSV (tab-separated values) file stored on the server and returns its name as `file_id`. Pass that name to [start](#start) as `filename`. The import runs as a background process: start it with [start](#start), track it with [read](#read) or [list](#list), download the rows that failed validation with [download\_failed](#download_failed), and mark the completed process as finished with [finish](#finish).

Address and coordinates complete each other: if only the address is given, the coordinates is obtained by geocoding. If only the coordinates are given, the address is obtained by reverse geocoding.

Tags that don't exist yet are created automatically (within the user's quota for tags).

## Import process object

```json
{
  "id": 7,
  "user_id": 231485,
  "created": "2026-07-02 10:15:32",
  "type": "place",
  "params": {
    "headers": ["label", "address", "lat", "lng", "radius", "description", "tags", "131312"],
    "user_headers": ["Label", "Address", "Latitude", "Longitude", "Radius", "Description", "Tags", "Responsible employee"]
  },
  "filename": "tmp-sheet640571613016981796.tsv",
  "status": "in_progress",
  "status_change_date": "2026-07-02 10:15:40",
  "progress": {
    "imported": 90,
    "failed": 10,
    "percent": 50,
    "processed_lines": 100,
    "warnings": [{"line": 5, "message": "Tags limit exceeded"}],
    "errors": [{"line": 12, "message": "label: Should not be empty"}]
  }
}
```

* `id` - int. Import process ID.
* `user_id` - int. Master user ID.
* `created` - date/time. When the import process was created.
* `type` - string. Import type, always `place`.
* `params` - object. Import parameters.
  * `headers` - string array. List of the file's headers.
  * `user_headers` - optional string array. User labels for headers.
* `filename` - string. Name of the preloaded TSV file.
* `status` - string. Import status: `created` | `in_progress` | `done` | `failed` | `finished`.
* `status_change_date` - date/time. When the status changed last time.
* `progress` - object. Import progress.
  * `imported` - int. Count of successfully imported POIs.
  * `failed` - int. Count of rows that did not pass validation.
  * `percent` - int. Approximate percentage of processed rows.
  * `processed_lines` - int. Count of processed lines.
  * `warnings` - array of objects. First 25 warnings, each with `line` and `message`.
  * `errors` - array of objects. First 25 errors, each with `line` and `message`.

## API actions

API path: `/place/import/`.

### start

Starts the background process of importing POIs.

**required sub-user rights**: `place_update`.

**Parameters**

| name          | description                                                                                                | type         |
| ------------- | ---------------------------------------------------------------------------------------------------------- | ------------ |
| filename      | Name of the server-side TSV file created by the [/data/spreadsheet/parse](../../commons/data.md#dataspreadsheetparse) method (returned as `file_id`). | string       |
| headers       | List of file's headers, see available fields below.                                                       | string array |
| user\_headers | Optional. List of user labels for headers. Must be the same size as `headers`.                             | string array |

Available fields:

* `label`
* `address`
* `lat`
* `lng`
* `radius` (default is 100 if not specified)
* `description`
* `tags`
* custom field ID as a string, e.g. `"131312"` (see [entity/fields](../../commons/entity/fields.md))
* `undefined` (if a meaning of a field is not known)

For custom fields of type `employee` and `multi_employee`, values are matched with the user's employees by full name. For `multi_employee` fields, multiple names can be listed separated by commas or semicolons.

A POI will not be imported if a required custom field is missing or has an invalid value. Invalid values of non-required custom fields are skipped with a warning.

**Response**

```json
{
  "success": true,
  "id": <int>
}
```

* `id` - int. An ID of the created import process.

**Example**

cURL

```sh
curl -X POST "https://api.eu.navixy.com/v2/place/import/start" \
    -H "Content-Type: application/json" \
    --data-binary @- << EOF
{
    "hash": "a6aa75587e5c59c32d347da438505fc3",
    "filename": "tmp-sheet640571613016981796.tsv",
    "headers": ["label", "address", "lat", "lng", "radius", "description", "tags", "131312"],
    "user_headers": ["Label", "Address", "Latitude", "Longitude", "Radius", "Description", "Tags", "Responsible employee"]
}
EOF
```

**Errors**

* 15 - Too many requests (rate limit exceeded) - if too many imports in progress
* 233 - No data file - if the preloaded file is not found
* 234 - Invalid data format - if the file is not a TSV
* 247 - Entity already exists - there is another identical import with the same file

### read

Returns an import process with specified ID.

**Parameters**

| name        | description | type |
| ----------- | ----------- | ---- |
| process\_id | Process ID  | int  |

**Response**

```json
{
  "success": true,
  "value": <import_process>
}
```

* `value` - [Import process object](#import-process-object).

**Example**

cURL

```sh
curl -X POST "https://api.eu.navixy.com/v2/place/import/read" \
    -H "Content-Type: application/json" \
    -d '{"hash": "a6aa75587e5c59c32d347da438505fc3", "process_id": 7}'
```

**Errors**

* 201 – Not found in database (if import is not found)

### list

Returns the list of the user's unfinished POI import processes (with statuses `created`, `in_progress`, `done` or `failed`).

**Response**

```json
{
  "success" : true,
  "list" : [ <import_process>, ... ]
}
```

* `list` - array of [Import process objects](#import-process-object).

**Example**

cURL

```sh
curl -X POST "https://api.eu.navixy.com/v2/place/import/list" \
    -H "Content-Type: application/json" \
    -d '{"hash": "a6aa75587e5c59c32d347da438505fc3"}'
```

### download\_failed

Retrieve a file with lines that contained errors and did not pass validation.

**Parameters**

| name        | description | type |
| ----------- | ----------- | ---- |
| process\_id | Process ID  | int  |

**Response**

File (standard file download).

**Example**

cURL

```sh
curl -X POST "https://api.eu.navixy.com/v2/place/import/download_failed" \
    -H "Content-Type: application/json" \
    -d '{"hash": "a6aa75587e5c59c32d347da438505fc3", "process_id": 7}'
```

**Errors**

* 201 – Not found in database (if import is not found)
* 204 – Entity not found (if file is not found)

### finish

Marks an import process as finished. Finished processes are not returned by [list](#list).

**Parameters**

| name        | description | type |
| ----------- | ----------- | ---- |
| process\_id | Process ID  | int  |

**Response**

```json
{
  "success": true
}
```

**Example**

cURL

```sh
curl -X POST "https://api.eu.navixy.com/v2/place/import/finish" \
    -H "Content-Type: application/json" \
    -d '{"hash": "a6aa75587e5c59c32d347da438505fc3", "process_id": 7}'
```

**Errors**

* 201 – Not found in database (if import is not found)
* 280 – Invalid import request state (if the import process is still in progress)
