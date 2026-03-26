---
name: primo-api
description: Reference for the Ex Libris Primo VE REST API in the CUNY consortium environment. Use when writing or debugging search queries, applying facet filters, looking up CUNY view IDs (vid), tabs, and scopes, or troubleshooting authentication and ROUTING_ERROR issues. Covers query syntax, result limits, rate limits, and all CUNY campus vid values.
---

# Primo API Skill — CUNY Environment

## Overview
Primo VE is the discovery layer for the CUNY (City University of New York) consortium, built on
top of Ex Libris Alma. Its REST API allows programmatic searching, configuration retrieval, and
user favorites management. API keys are shared with Alma — a single key can have permissions
across both Alma and Primo functional areas.

---

## Base URL
```
https://api-na.hosted.exlibrisgroup.com/primo/v1/
```
Regional variants: `api-eu`, `api-ap`, `api-aps`, `api-ca`, `api-cn` (China uses `.com.cn` TLD).

CUNY uses the North America region (`api-na`).

---

## Authentication
- API key required: `?apikey=YOUR_KEY` or header `Authorization: apikey YOUR_KEY`
- Keys are managed at the Ex Libris Developer Network: https://developers.exlibrisgroup.com/
- A single API key can have permissions in both Alma (Bibs, Users, Acquisitions, etc.) and
  Primo (Searching, Favorites, etc.)
- Keys expire after one year of inactivity

### API Key Setup for Primo
When generating or editing an API key on the Developer Network, add a permission row with:
- **Area**: Primo (Searching, Favorites, etc.)
- **Environment**: Production or Sandbox
- **Permission**: Read-only or Read/write

### ROUTER_ERROR — Routing Configuration
If you receive:
```
ROUTING_ERROR: No routing configured for your institution. Please contact Alma Support.
```

This is a **server-side configuration issue**, not a code problem. Ex Libris must configure API
routing for your institution. As of early 2026, all CUNY IZs have Primo API routing configured.
If you encounter this error at CUNY, open a support case with Ex Libris (support.proquest.com)
or open a ticket with the CUNY Office of Library Services (cuny-ols.libanswers.com).

Other common causes of this error:
- Passing `vid=01CUNY_XX:CUNY_XX` instead of just `01CUNY_XX:CUNY_XX` as the vid value
  (don't include the parameter name in the value)

---

## Content Types
- Primo APIs return **JSON only** (unlike Alma, which defaults to XML)
- No `Accept` header needed

---

## API Endpoints

### Search (Primary)
```
GET /primo/v1/search
```
The main search endpoint. Returns results, facets, and result count.

#### Required Parameters
| Parameter | Type   | Description |
|-----------|--------|-------------|
| `vid`     | string | View ID (see CUNY-Specific Notes below) |
| `tab`     | string | Tab name (see CUNY-Specific Notes below) |
| `scope`   | string | Search scope (see CUNY-Specific Notes below) |
| `q`       | string | Query string (see Query Syntax below) |

#### Optional Parameters
| Parameter           | Type    | Default | Description |
|---------------------|---------|---------|-------------|
| `offset`            | int     | 0       | Starting position (max 5000) |
| `limit`             | int     | 10      | Results per page (max 50, hard cap 2000 total) |
| `sort`              | string  | rank    | Sort order: `rank`, `title`, `author`, `date`, `date_d` (newest), `date_a` (oldest) |
| `lang`              | string  | eng     | Language code (Primo VE uses 2-letter: `en`, `es`, etc.) |
| `qInclude`          | string  | —       | Include facet filter (AND logic between facets) |
| `qExclude`          | string  | —       | Exclude facet filter |
| `multiFacets`       | string  | —       | Facet filter with OR logic between values, AND between categories |
| `pcAvailability`    | boolean | true    | Include records without full text |
| `conVoc`            | boolean | true    | Enable controlled vocabulary synonyms |
| `skipDelivery`      | boolean | true    | Skip delivery info for faster results |
| `disableSplitFacets`| boolean | true    | Skip facet retrieval for faster results |
| `fromDate`          | string  | —       | Results updated after date (format: `YYYYMMDDHHMMSS`) |
| `newspapers`        | boolean | —       | Enable newspapers search (requires `pcAvailability=true`) |

### Configuration
```
GET /primo/v1/configuration/{vid}
```
Retrieves view configuration for a specific vid.

### Favorites
```
GET  /primo/v1/favorites         — Retrieve user's saved favorites
POST /primo/v1/favorites         — Add or remove favorites
PUT  /primo/v1/favorites         — Update favorite labels
```
Requires user authentication (JWT). Supported in Primo VE with limited functionality.

### Guest JWT
```
GET /primo/v1/jwt/{institution}
```
Generates a guest JSON Web Token for unauthenticated access. **Primo only** (not Primo VE).

### User JWT
```
GET /primo/v1/userjwt
```
Generates an authenticated user JWT.

### Analytics
```
GET /primo/v1/analytics
```
Usage analytics data.

### Translations
```
GET /primo/v1/translations
```
Language/translation data for Primo UI.

### Resource Recommender
```
GET /primo/v1/resourcerecommender
```
Returns resource recommendations.

---

## Query Syntax

### Basic Format
```
q=<field>,<precision>,<value>
```

### Fields
| Field     | Description |
|-----------|-------------|
| `any`     | Any field |
| `title`   | Title |
| `creator` | Author |
| `sub`     | Subject |
| `usertag` | User tag |

### Precision Operators
| Operator      | Description |
|---------------|-------------|
| `exact`       | Exact match |
| `begins_with` | Value at beginning of field |
| `contains`    | Value anywhere in field |

### Logical Operators (for multi-field queries)
| Operator | Description |
|----------|-------------|
| `AND`    | Both conditions must match (default if omitted) |
| `OR`     | Either condition must match |
| `NOT`    | Next field's value must not be found |

### Examples
```
# Simple search — title contains "home"
q=title,contains,home

# Multi-field — title contains "pop music" AND subject contains "korean"
q=title,contains,pop music,AND;sub,contains,korean

# Author search
q=creator,contains,toni morrison

# Exact title match
q=title,exact,beloved
```

**Limitation**: Values cannot contain semicolons (`;`), as semicolons delimit multiple fields.

### Facet Filtering
Use `qInclude` or `qExclude` to filter by facets:
```
qInclude=facet_rtype,exact,books
qInclude=facet_rtype,exact,books|,|facet_lang,exact,eng
```

Valid facet categories: `facet_rtype`, `facet_topic`, `facet_creator`, `facet_tlevel`,
`facet_domain`, `facet_library`, `facet_lang`, `facet_lcc`, `facet_searchcreationdate`

---

## Rate Limits
- **Per second**: Up to 50 API calls per institution
- **Per minute**: Maximum 1,500 API calls per institution
- **Daily**: 5 API calls per FTE (full-time equivalent) user licensed per day
- HTTP 429 returned when thresholds exceeded

---

## Result Limits
- Each request returns a maximum of **2,000 results**
- The `offset` parameter has a hard cap of **5,000**
- Default `limit` is 10; recommended max is 50

---

## Error Handling
Errors return JSON:
```json
{
    "errorCode": 1234,
    "errorMessage": "Error description",
    "details": []
}
```

Common errors:
- **401**: Expired or invalid API key
- **429**: Rate limit exceeded
- **500**: Internal server error (including ROUTING_ERROR — see above)

---

## CUNY-Specific Notes

### View IDs (vid)
The `vid` parameter follows the format `01CUNY_XX:CUNY_XX` where `XX` is the 2-letter campus
code in capital letters:

```
01CUNY_BB:CUNY_BB       — Baruch College
01CUNY_BM:CUNY_BM       — Borough of Manhattan Community College
01CUNY_BX:CUNY_BX       — Bronx Community College
01CUNY_BC:CUNY_BC       — Brooklyn College
01CUNY_CC:CUNY_CC       — The City College of New York
01CUNY_SI:CUNY_SI       — College of Staten Island
01CUNY_GJ:CUNY_GJ       — Craig Newmark Graduate School of Journalism
01CUNY_AL:CUNY_AL       — CUNY Central Office
01CUNY_GC:CUNY_GC       — CUNY Graduate Center
01CUNY_CL:CUNY_CL       — CUNY School of Law
01CUNY_NC:CUNY_NC       — Guttman Community College
01CUNY_HO:CUNY_HO       — Hostos Community College
01CUNY_HC:CUNY_HC       — Hunter College
01CUNY_JJ:CUNY_JJ       — John Jay College of Criminal Justice
01CUNY_KB:CUNY_KB       — Kingsborough Community College
01CUNY_LG:CUNY_LG       — LaGuardia Community College
01CUNY_LE:CUNY_LE       — Lehman College
01CUNY_ME:CUNY_ME       — Medgar Evers College
01CUNY_NY:CUNY_NY       — New York City College of Technology
01CUNY_QC:CUNY_QC       — Queens College
01CUNY_QB:CUNY_QB       — Queensborough Community College
01CUNY_YC:CUNY_YC       — York College
```

### Tabs and Scopes
| Tab              | Scope             | Description |
|------------------|-------------------|-------------|
| `Everything`     | `IZ_CI_AW`        | All resources — local, CDI, and Academic Works (CUNY's institutional repository) |
| `NZPhysical`     | `NZPhysical`      | Physical items across the CUNY Network Zone |
| `IZ_NZ_SUNY`     | `IZ_CI_AW_NZ_SUNY`| CUNY + SUNY resources |
| `CourseReserves`  | `CourseReserves`   | Course reserves (available at some campuses) |

### Example CUNY Search
```
GET https://api-na.hosted.exlibrisgroup.com/primo/v1/search
    ?vid=01CUNY_HC:CUNY_HC
    &tab=Everything
    &scope=IZ_CI_AW
    &q=title,contains,beloved
    &limit=10
    &apikey=YOUR_KEY
```

---

## Useful References
- Primo REST API docs: https://developers.exlibrisgroup.com/primo/apis/
- Search API reference: https://developers.exlibrisgroup.com/primo/apis/search/
- API Console (live testing): https://developers.exlibrisgroup.com/console/
- Primo Search API tutorial:
  https://developers.exlibrisgroup.com/blog/understanding-and-using-the-primo-search-api-as-a-guest-in-the-developer-network/
- Deep links (New UI): https://developers.exlibrisgroup.com/primo/apis/deep-links-new-ui/
