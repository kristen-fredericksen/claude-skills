---
name: alma-api
description: Reference for the Ex Libris Alma REST API in the CUNY consortium environment. Use when writing or debugging Python scripts that query or update Alma bibs, holdings, items, users, acquisitions, configuration, sets, jobs, or Analytics. Covers CUNY institution codes, NZ vs. IZ scoping, barcode handling, error patterns, rate limits, and project conventions such as dry-run flags, multi-IZ key management, and CLI entry points.
---

# Alma API Skill — CUNY Environment

## Overview
Ex Libris Alma is a library services platform used across the CUNY (City University of New York)
consortium. Its REST API allows programmatic access to bibliographic records, holdings, items,
users, acquisitions, fulfillment, configuration, and partner resources. All API calls require an
API key.

---

## Base URL
```
https://api-na.hosted.exlibrisgroup.com/almaws/v1/
```
Regional variants:
```
api-eu    — Europe
api-ap    — Asia-Pacific
api-aps   — Asia-Pacific South
api-ca    — Canada
api-cn    — China (note: uses .com.cn TLD)
```
CUNY uses the North America region (`api-na`).

---

## Authentication
- API key required: `?apikey=YOUR_KEY` or header `Authorization: apikey YOUR_KEY`
- Keys are scoped by area (Bibs, Users, Acquisitions, Configuration, etc.), environment
  (Production, Sandbox, Guest Sandbox), and permission (Read-only, Read/write)
- Generate and manage keys at the Ex Libris Developer Network:
  https://developers.exlibrisgroup.com/

### Key Expiration
**API keys expire after one year of inactivity.** This is a common failure point for scripts
that run infrequently (e.g., annual retention reviews). If a script returns a 401 Unauthorized
error after a period of dormancy, an expired key is the first thing to check. Handle 401 errors
explicitly in scripts with a clear message like:

```
401 Unauthorized — your Alma API key may have expired.
Renew it at the Ex Libris Developer Network: https://developers.exlibrisgroup.com/
```

---

## Content Types
- Default response: XML
- Request JSON: add `Accept: application/json` header
- Sending data: `Content-Type: application/json` or `application/xml`

---

## Core Resource Areas & Common Endpoints

### Bibliographic Records
```
GET    /bibs/{mms_id}
POST   /bibs
PUT    /bibs/{mms_id}
DELETE /bibs/{mms_id}
GET    /bibs?issn=...&title=...&limit=...&offset=...
```

### Holdings & Items
```
GET    /bibs/{mms_id}/holdings
GET    /bibs/{mms_id}/holdings/{holding_id}
GET    /bibs/{mms_id}/holdings/{holding_id}/items
GET    /bibs/{mms_id}/holdings/{holding_id}/items/{item_pid}
PUT    /bibs/{mms_id}/holdings/{holding_id}/items/{item_pid}
POST   /bibs/{mms_id}/holdings/{holding_id}/items/{item_pid}/loans
```

### Item Lookup by Barcode
```
GET    /items?item_barcode={barcode}
```
Returns item details including MMS ID, holding ID, and item PID without needing to know the
bib record first.

### Users
```
GET    /users/{user_id}
GET    /users?q=last_name~Smith&limit=10
PUT    /users/{user_id}
GET    /users/{user_id}/loans
GET    /users/{user_id}/requests
GET    /users/{user_id}/fees
POST   /users/{user_id}/fees/{fee_id}?op=waive
```

### Acquisitions
```
GET    /acq/po-lines/{po_line_id}
GET    /acq/funds
GET    /acq/vendors/{vendor_code}
GET    /acq/invoices/{invoice_id}
```

### Configuration
```
GET    /conf/letters
GET    /conf/letters/{letter_code}
PUT    /conf/letters/{letter_code}
GET    /conf/code-tables/{table_name}
PUT    /conf/code-tables/{table_name}
GET    /conf/integration-profiles
GET    /conf/integration-profiles/{profile_id}
POST   /conf/integration-profiles
```

**Integration profiles — default limit of 10:** `GET /conf/integration-profiles` returns only
10 results by default, even if more exist. An institution may have many integration profiles
across types (USER, UPLOAD_E_HOLDINGS, NEW_ORDER_API, PROXY_DEFINITION, etc.), so the default
will silently omit records. Always filter by type and set an explicit limit:
```
GET /conf/integration-profiles?type=UPLOAD_E_HOLDINGS&limit=100
GET /conf/integration-profiles?type=NEW_ORDER_API&limit=100
```
Confirmed by Ex Libris support (April 2026). Fetching all types in a single call with
`limit=200` also works but filtering by type is the recommended approach.

### Partners (Resource Sharing)
```
GET    /partners/{partner_code}
PUT    /partners/{partner_code}
```
Used for managing ILL partner records, including contact info and addresses.

### Sets & Jobs
```
GET    /conf/sets
GET    /conf/sets/{set_id}/members
POST   /conf/jobs/{job_id}?op=run
GET    /conf/jobs/{job_id}/instances/{instance_id}
```

### Analytics
```
GET    /analytics/reports?path=/shared/...&limit=25&col_names=true
```
Returns up to 25 rows per call; use `token` parameter to paginate.

---

## Key Parameters
| Parameter | Description |
|-----------|-------------|
| `limit`   | Results per page (max 100) |
| `offset`  | Pagination offset |
| `q`       | Search query (field~value syntax) |
| `expand`  | Include additional data (e.g., `expand=p_avail`) |
| `view`    | Response detail level |

---

## Critical Patterns

### Full-Record PUT (No PATCH)
**Alma does not support PATCH.** To update any record, you must:
1. `GET` the full record
2. Modify the target field(s) in the response body
3. `PUT` the entire object back

Sending a partial record will cause errors or data loss.

### Retrieve a Bib + All Items
1. `GET /bibs/{mms_id}` — get bib record
2. `GET /bibs/{mms_id}/holdings` — get list of holdings
3. `GET /bibs/{mms_id}/holdings/{holding_id}/items` — get items per holding

### Run a Job on a Set
1. Identify set ID via `GET /conf/sets`
2. `POST /conf/jobs/{job_id}?op=run` with set ID in body

---

## Error Handling
Alma returns errors in the response body even on HTTP 200 in some cases. Always check for
`<errorsExist>true</errorsExist>` in XML or `"errorsExist": true` in JSON.

Common errors to handle explicitly:
- **401**: Expired or invalid API key (see Key Expiration note above)
- **400**: Malformed request or invalid field value
- **404**: Record not found (e.g., barcode not in system)
- **200 with errorsExist**: Alma-level business rule violation

Recommended error handling pattern:
```python
try:
    response = requests.get(url, headers=headers)
    response.raise_for_status()
except requests.exceptions.HTTPError as e:
    if e.response is not None:
        print(f"HTTP {e.response.status_code}: {e.response.text[:200]}")
    raise
```

---

## Rate Limits
- 25 API calls per second per institution
- Daily limits vary by subscription
- In a multi-institution script, account for rate limits multiplied by the number of
  institutions being queried
- Safe throttle: `time.sleep(0.05)` between requests (~20 req/sec)

---

## CUNY-Specific Notes

### Institution Codes
CUNY has 22+ institutions in the consortium. Institution codes follow the pattern `01CUNY_XX`.
When writing scripts that query multiple campuses, **always require the institution code as an
explicit input parameter** rather than trying to infer it from barcodes or other data.

Known institution codes:
```
01CUNY_BB       — Baruch College
01CUNY_BM       — Borough of Manhattan Community College
01CUNY_BX       — Bronx Community College
01CUNY_BC       — Brooklyn College
01CUNY_CC       — The City College of New York
01CUNY_SI       — College of Staten Island
01CUNY_GJ       — Craig Newmark Graduate School of Journalism at CUNY
01CUNY_AL       — CUNY Central Office
01CUNY_GC       — CUNY Graduate Center
01CUNY_NETWORK  — CUNY Network Zone
01CUNY_CL       — CUNY School of Law
01CUNY_NC       — Guttman Community College
01CUNY_HO       — Hostos Community College
01CUNY_HC       — Hunter College
01CUNY_JJ       — John Jay College of Criminal Justice
01CUNY_KB       — Kingsborough Community College
01CUNY_LG       — LaGuardia Community College
01CUNY_LE       — Lehman College
01CUNY_ME       — Medgar Evers College
01CUNY_NY       — New York City College of Technology
01CUNY_QC       — Queens College
01CUNY_QB       — Queensborough Community College
01CUNY_YC       — York College
```

### Campus Code Formats
Campus codes appear in two formats depending on context:
- **Full institution code**: `01CUNY_XX` (e.g., `01CUNY_QB`) — used in API keys, IZ scoping
- **Short operational code**: `XX001` (e.g., `QB001`) — used in letter notification data

When extracting the 2-letter campus abbreviation, handle both formats:
```python
if raw_code.startswith("01CUNY_"):
    campus = raw_code.split("_")[1]    # 01CUNY_QB → QB
else:
    campus = raw_code[:2]              # QB001 → QB
```

### Network Zone (NZ) vs. Institution Zone (IZ)
- API key scope determines which zone you're operating in
- **Shared bib records in the NZ cannot be edited via an IZ-scoped API key**
- Some endpoints behave differently depending on zone — test carefully
- When building multi-institution scripts, use IZ keys scoped to each institution, not a
  single NZ key, unless the operation explicitly requires NZ access

### Barcode Format — Do Not Assume
CUNY barcodes are **not standardized**. Do not infer owning institution or record validity from
barcode format. Known patterns include:

- Standard: 14-digit numeric — `31407009018250`
- Special collections: alphanumeric prefix — `BAR-1234567`, `T7795`
- Gifted items: barcode may still reflect the *original* library, not the current owner
- Human error: malformed, truncated, or incorrectly entered barcodes exist

Always validate barcodes against the API response rather than parsing the barcode string itself.
Build in graceful handling for 404 responses (barcode not found) and unexpected formats.

### Primo VE — $$Q Marker Technique
When working on Primo VE normalization rules for subject fields, the `$$Q` marker is used to
handle trailing period removal in display. This is a non-obvious Alma/Primo-specific technique
that does not appear in standard documentation. Subject field modifications also require:
- Explicit vocabulary filters (ind2="0" or ind2="7" with specific $2 values:
  aat, bhashe, bidex, homoit, lcdgt, lctgm, pana, tgn)
- 880 (linked alternate script) fields should use the **same explicit vocabulary filters**
  as their associated fields — not the simplified `NOT ind2="2"` shortcut

---

## CUNY Project Conventions

### CLI Pattern
All Alma API scripts use a single Python file with a `main()` entry point registered in
`pyproject.toml`:
```toml
[project.scripts]
alma-letters = "export_letters:main"
alma-partner-sync = "sync_partners:main"
```

### Dry-Run by Default
All scripts that modify data should default to dry-run mode and require an explicit `--apply`
flag to make changes:
```python
parser.add_argument("--apply", action="store_true",
                    help="Actually apply changes (default: dry-run)")
```

### Multi-Environment Support
Scripts that deploy to Alma should support separate sandbox and production environments:
```
ALMA_API_KEY_SANDBOX=...
ALMA_API_KEY_PRODUCTION=...
```

### API Key Storage
- **Single key**: `.env` file with `python-dotenv`
- **Multiple IZ keys**: JSON file (`iz_api_keys.json`) or CSV file (`api_keys.csv`)
- **Never hardcode keys in scripts**
- Provide `.env.example` or `api_keys.csv.example` templates

---

## Finding Available Fields
The most reliable method for finding available API fields is the **XSD files** on the Ex Libris
Developer Network, not Swagger (which has been found to be incomplete).

- Developer Network: https://developers.exlibrisgroup.com/alma/apis/
- API Explorer (live testing): https://developers.exlibrisgroup.com/console/
- XSD files are linked per resource area in the API documentation

---

## Sensitive Data — GitHub / Version Control
When committing Alma API scripts to GitHub, always confirm the following are in `.gitignore`:
```
data/               # school information, barcode files, export CSVs
.env                # API keys and credentials
*.env
iz_api_keys.json    # multi-IZ key files
api_keys.csv        # multi-IZ key files
```
Never hardcode API keys in scripts. Use environment variables loaded from `.env` via
`python-dotenv` or equivalent.

---

## Useful References
- Ex Libris Developer Network: https://developers.exlibrisgroup.com/alma/apis/
- API Explorer: https://developers.exlibrisgroup.com/console/
- CUNY shared print retention script:
  https://github.com/kristen-fredericksen/cuny-shared-print-retention
- CUNY Libraries normalization rules: https://github.com/cuny-libraries/rules
- CUNY Libraries Alma letters: https://github.com/cuny-libraries/alma-letters
- CUNY Libraries access models: https://github.com/cuny-libraries/access-models
