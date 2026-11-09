# DD API V4 Write - Endpoints

## Overzicht

De Write module voegt nieuwe endpoints toe aan DD API V4 voor batch-verwerking van schrijfoperaties. 
Elke entiteit heeft zijn eigen endpoint voor batch-operaties **inclusief job tracking**. 

**Eén endpoint voor status en resultaten**: De job status endpoint retourneert **alleen basisinformatie** tijdens verwerking, en **status + gedetailleerde resultaten** wanneer de job voltooid is.

## Endpoint Overzicht

| Endpoint                        | Methode | Beschrijving                                   | Entiteit    |
|---------------------------------|---------|------------------------------------------------|-------------|
| `/v4/write/observation`         | POST    | Batch insert/update/deprecate van observaties  | Observation |
| `/v4/write/observation/{jobId}` | GET     | Opvragen status en resultaten van een job      | Observation |
| `/v4/write/foi`                 | POST    | Batch insert/update/deprecate van meetobjecten | FoI         |
| `/v4/write/foi/{jobId}`         | GET     | Opvragen status en resultaten van een job      | FoI         |
| `/v4/write/parameter`           | POST    | Batch insert/update van parameters             | Parameter   |
| `/v4/write/parameter/{jobId}`   | GET     | Opvragen status en resultaten van een job      | Parameter   |

## Endpoint Structuur

```
/v4/write/observation/      ← Observation namespace
├── POST                    ← Batch submit
└── GET {jobId}             ← Job status + resultaten (als klaar)

/v4/write/foi/              ← FoI namespace
├── POST                    ← Batch submit
└── GET {jobId}             ← Job status + resultaten (als klaar)

/v4/write/parameter/        ← Parameter namespace
├── POST                    ← Batch submit
└── GET {jobId}             ← Job status + resultaten (als klaar)
```

**Totaal: 6 endpoints** (3 voor batch submit, 3 voor job status met variabele response)

## 1. Observation Endpoints

### 1.1 POST /v4/write/observation

#### Beschrijving

Verstuur een batch van **insert**, **update**, en **deprecate** operaties voor **observaties**. De operaties worden asynchroon verwerkt.

#### Request

**Content-Type**: `application/json`

**Body**:
```jsonc
{
  "operations": [
    {
      "operation": "insert",
      "id": "op1",
      "data": {
        "observation": {
          "sourceId": "my-observation-001",    // Client's sourceId (VEREIST voor INSERT)
          "type": "measure",
          "resultTime": "2024-01-15T10:00:00Z",
          "phenomenonTime": {
            "begin": "2024-01-15T10:00:00Z",
            "end": "2024-01-15T10:00:00Z"
          },
          "parameter": {
            "parameter/waarnemingssoort": "WATHTE"
          },
          "foi": {
            "id": "550e8400-e29b-41d4-a716-446655440002"  // Server ID (string)
          },
          "result": {
            "id": "my-result-001",
            "measure": {
              "uom": "m",
              "value": 1.5
            }
          },
          "modifiedOn": "2024-01-15T10:00:00Z"
        }
      }
    },
    {
      "operation": "update",
      "id": "op2",
      "data": {
        "observation": {
          "id": "550e8400-e29b-41d4-a716-446655440001",  // Server ID (string) VEREIST
          "type": "measure",
          "resultTime": "2024-01-15T11:00:00Z",
          "phenomenonTime": {
            "begin": "2024-01-15T11:00:00Z",
            "end": "2024-01-15T11:00:00Z"
          },
          "parameter": {
            "parameter/waarnemingssoort": "WATHTE"
          },
          "foi": {
            "id": "550e8400-e29b-41d4-a716-446655440002"  // Server ID (string)
          },
          "result": {
            "id": "my-result-001",
            "measure": {
              "uom": "m",
              "value": 1.7
            }
          },
          "modifiedOn": "2024-01-15T11:00:00Z"
        }
      }
    },
    {
      "operation": "deprecate",
      "id": "op3",
      "data": {
        "id": "550e8400-e29b-41d4-a716-446655440003",  // Server ID (string) VEREIST
        "reason": "Foutieve meting"
      }
    }
  ],
  "options": {
    "continueOnError": true,
    "validateOnly": false
  }
}
```

**Operaties**:
- `insert`: Nieuwe observatie toevoegen (CREATE)
- `update`: Bestaande observatie bijwerken (UPDATE)
- `deprecate`: Observatie deprecaten

**Key Exchange Regels**:
- **INSERT**: `sourceId` VEREIST, `id` MOET afwezig zijn
- **UPDATE/DEPRECATE**: `id` (server) VEREIST, `sourceId` MOET afwezig zijn

#### Response

**Status Codes**:
- `202 Accepted`: Batch is geaccepteerd
- `400 Bad Request`: Ongeldige request
- `401 Unauthorized`: Authenticatie ontbreekt
- `403 Forbidden`: Geen toestemming
- `413 Payload Too Large`: Request te groot. De response **MOET** een `X-Max-Batch-Size` header bevatten met de maximale toegestane batch grootte.
- `500 Internal Server Error`: Interne fout

**Headers**:
```
Location: /v4/write/observation/{jobId}
```

**Body** (202 Accepted):
```json
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "createdAt": "2024-01-15T10:01:00Z",
  "totalOperations": 3,
  "location": "/v4/write/observation/550e8400-e29b-41d4-a716-446655440000"
}
```

### 1.2 GET /v4/write/observation/{jobId}

#### Beschrijving

Opvragen van de status en (als de job voltooid is) de resultaten van een observation batch-job.

**Response varieert op basis van job status:**
- **Status `queued` of `processing`**: Alleen basisinformatie (klein, efficiënt voor polling)
- **Status `completed` of `failed`**: Basisinformatie + gedetailleerde operatie resultaten

#### Request

**Parameters**:
| Naam | Type | Verplicht | Beschrijving |
|------|------|-----------|--------------|
| jobId | string (uuid) | Ja | Unieke identifier van de observation job |

#### Response

**Status Codes**:
- `200 OK`: Status (en resultaten als klaar) opgevraagd
- `404 Not Found`: Job niet gevonden
- `500 Internal Server Error`: Interne serverfout

**Body (200 OK, status: queued of processing)**:
```json
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "createdAt": "2024-01-15T10:01:00Z",
  "startedAt": "2024-01-15T10:01:05Z",
  "totalOperations": 3
}
```

**Body (200 OK, status: completed of failed)**:
```jsonc
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "createdAt": "2024-01-15T10:01:00Z",
  "startedAt": "2024-01-15T10:01:05Z",
  "completedAt": "2024-01-15T10:01:10Z",
  "totalOperations": 3,
  "successfulOperations": 3,
  "failedOperations": 0,
  "operations": [
    {
      "operationIndex": 0,
      "operationId": "op1",
      "status": "succeeded",
      "sourceId": "my-observation-001",        // Client's sourceId (alleen bij INSERT)
      "id": "550e8400-e29b-41d4-a716-446655440001",  // Server ID (string)
      "messages": []
    },
    {
      "operationIndex": 1,
      "operationId": "op2",
      "status": "succeeded",
      "sourceId": null,                        // null voor UPDATE
      "id": "550e8400-e29b-41d4-a716-446655440002",  // Server ID (string)
      "messages": []
    },
    {
      "operationIndex": 2,
      "operationId": "op3",
      "status": "succeeded",
      "sourceId": null,                        // null voor DEPRECATE
      "id": "550e8400-e29b-41d4-a716-446655440003",  // Server ID (string)
      "messages": []
    }
  ],
  "errors": []
}
```

## 2. FoI Endpoints

### 2.1 POST /v4/write/foi

#### Beschrijving

Verstuur een batch van **insert**, **update**, en **deprecate** operaties voor **Features of Interest (meetobjecten)**.

#### Request

**Content-Type**: `application/json`

**Body**:
```jsonc
{
  "operations": [
    {
      "operation": "insert",
      "id": "op1",
      "data": {
        "foi": {
          "sourceId": "my-location-001",      // Client's sourceId (VEREIST voor INSERT)
          "type": "http://definities.pdok.nl/def/cbsgb/1.0/id/datasetdefinitie/Top10NL",
          "name": "Meetlocatie Amsterdam",
          "organisation": "RWS",
          "code": "LOC001",
          "description": "Meetlocatie Amsterdam",
          "location": "https://pdok.nl",
          "geometry": {
            "type": "Point",
            "coordinates": [4.9, 52.37]
          }
        }
      }
    },
    {
      "operation": "update",
      "id": "op2",
      "data": {
        "foi": {
          "id": "550e8400-e29b-41d4-a716-446655440002",  // Server ID (string) VEREIST
          "type": "http://definities.pdok.nl/def/cbsgb/1.0/id/datasetdefinitie/Top10NL",
          "name": "Meetlocatie Amsterdam Centraal",
          "organisation": "RWS",
          "code": "LOC001",
          "description": "Meetlocatie Amsterdam Centraal Station",
          "location": "https://pdok.nl",
          "geometry": {
            "type": "Point",
            "coordinates": [4.9, 52.37]
          }
        }
      }
    },
    {
      "operation": "deprecate",
      "id": "op3",
      "data": {
        "id": "550e8400-e29b-41d4-a716-446655440003",  // Server ID (string) VEREIST
        "reason": "Locatie niet meer in gebruik"
      }
    }
  ]
}
```

#### Response

Zie [POST /v4/write/observation](#11-post-v4writeobservation) voor response format (202 Accepted).

**Headers**:
```
Location: /v4/write/foi/{jobId}
```

### 2.2 GET /v4/write/foi/{jobId}

Opvragen van de status en resultaten van een FoI batch-job.

**Response format**: Zie [GET /v4/write/observation/{jobId}](#12-get-v4writeobservationjobid)

## 3. Parameter Endpoints

### 3.1 POST /v4/write/parameter

#### Beschrijving

Verstuur een batch van **insert** en **update** operaties voor **parameters**.

**Belangrijk**: Parameters kunnen niet worden gedeprecateerd.

#### Request

**Content-Type**: `application/json`

**Body**:
```jsonc
{
  "operations": [
    {
      "operation": "insert",
      "id": "op1",
      "data": {
        "parameter": {
          "sourceId": "my-parameter-001",    // Client's sourceId (VEREIST voor INSERT)
          "name": "Waterhoogte",
          "definition": "http://definities.pdok.nl/def/imkl/1.0/id/waarnemingssoort/WATHTE"
        }
      }
    },
    {
      "operation": "update",
      "id": "op2",
      "data": {
        "parameter": {
          "id": "550e8400-e29b-41d4-a716-446655440004",  // Server ID (string) VEREIST
          "name": "Waterhoogte (herzien)",
          "definition": "http://definities.pdok.nl/def/imkl/1.0/id/waarnemingssoort/WATHTE"
        }
      }
    }
  ]
}
```

#### Response

Zie [POST /v4/write/observation](#11-post-v4writeobservation) voor response format (202 Accepted).

**Headers**:
```
Location: /v4/write/parameter/{jobId}
```

### 3.2 GET /v4/write/parameter/{jobId}

Opvragen van de status en resultaten van een parameter batch-job.

**Response format**: Zie [GET /v4/write/observation/{jobId}](#12-get-v4writeobservationjobid)

## Status Waarden

| Status       | Beschrijving                                  | Response inhoud                                                        |
|--------------|-----------------------------------------------|------------------------------------------------------------------------|
| `queued`     | Job wacht op verwerking                       | Basisinfo (jobId, status, timestamps, totalOperations)                 |
| `processing` | Job wordt verwerkt                            | Basisinfo + startedAt                                                  |
| `completed`  | Job is voltooid (met of zonder fouten)        | Basisinfo + successfulOperations, failedOperations, operations, errors |
| `failed`     | Job is mislukt voordat verwerking kon starten | Basisinfo + errors                                                     |

## Polling voor Job Status

Clients MOETEN de status van jobs opvragen via polling op het **specifieke entiteit endpoint**. 

### Aanbevolen Polling Strategie

1. **Exponentiële Backoff**: Begin met korte intervallen en verhoog geleidelijk
   - Eerste polling: 500ms
   - Tweede: 1s
   - Derde: 2s
   - Vierde: 5s
   - Vervolgens: 10s (maximum)

2. **Maximaal Aantal Pogingen**: Stel een maximum in (bv. 100 pogingen)
3. **Timeout**: Geef een globale timeout (bv. 5 minuten)

### Voorbeeld Polling Flow

```
1. POST /v4/write/observation → 202 Accepted, jobId: abc-123
2. GET /v4/write/observation/abc-123 → status: queued
3. GET /v4/write/observation/abc-123 → status: processing
4. GET /v4/write/observation/abc-123 → status: completed, operations: [...], errors: [...]
```

**Belangrijk**: Na stap 4 heeft de client **alle** informatie (status + resultaten) in één response!

## Authenticatie

Alle Write endpoints vereisen authenticatie volgens **de standaard authenticatie methoden die DD API V4 specificeert**.

De implementatie MOET minimaal één van de volgende standaard methoden ondersteunen:
- API Key (via `X-API-Key` header)
- Bearer Token (via `Authorization: Bearer <token>` header)
- OAuth2 (client credentials flow)
- mTLS (Mutual TLS certificaten)

> **Noot**: De exacte authenticatiemethode(n) en configuratie zijn implementatie-specifiek, zolang deze voldoen aan de DD API V4 standaard.

## Autorisatie

Autorisatie is **aan de server**. De server bepaalt welke clients toegang hebben tot welke operaties.

De Write module definieert geen specifieke autorisatieregels; dit is een implementatie detail.

## Foutcodes

### Algemene Fouten

| Code                | Beschrijving                           |
|---------------------|----------------------------------------|
| `INVALID_REQUEST`   | Ongeldige request structuur            |
| `UNAUTHORIZED`      | Ontbrekende of ongeldige authenticatie |
| `FORBIDDEN`         | Geen toestemming voor deze operatie    |
| `NOT_FOUND`         | Resource niet gevonden                 |
| `PAYLOAD_TOO_LARGE` | Request te groot                       |
| `INTERNAL_ERROR`    | Interne serverfout                     |

### Validatie Fouten

Zie [Validatie](03-00-validation.md) voor een complete lijst van validatie foutcodes.

### Key Exchange Fouten

Zie [Key Exchange Mechanisme](06-00-key-exchange.md) voor key-exchange specifieke foutcodes.

## Samenvatting Endpoint Structuur

Elke entiteit heeft zijn eigen **namespace** onder `/v4/write/` met **twee endpoints**:

```
/v4/write/observation/    ← Observation namespace
├── POST                  ← Batch insert/update/deprecate
└── GET {jobId}           ← Job status + resultaten

/v4/write/foi/            ← FoI namespace
├── POST                  ← Batch insert/update/deprecate
└── GET {jobId}           ← Job status + resultaten

/v4/write/parameter/      ← Parameter namespace
├── POST                  ← Batch insert/update
└── GET {jobId}           ← Job status + resultaten
```

**Totaal: 6 endpoints** (3 voor submit, 3 voor status met variabele response)

Deze structuur zorgt voor:
- ✅ **Consistentie**: Alle operaties voor een entiteit onder één path
- ✅ **Eenvoud**: Slechts 2 endpoints per entiteit
- ✅ **Efficiëntie**: Één request voor status + resultaten
- ✅ **RESTful design**: Resources nested op logische manier
- ✅ **Vereenvoudigd Key Exchange**: INSERT met sourceId, UPDATE/DEPRECATE met server id
