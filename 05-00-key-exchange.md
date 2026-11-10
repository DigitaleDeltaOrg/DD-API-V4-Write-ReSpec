# DD API V4 Write - Key Exchange Mechanisme

## Overzicht

DD API V4 Write implementeert een **vereenvoudigd key-exchange** mechanisme voor het beheer van entiteit-identificaties. Dit zorgt voor:

- **Eenduidige identificatie**: Server-side keys garanderen unieke identificatie binnen het systeem
- **Security**: Server beheert de autoritatieve identificaties
- **Flexibiliteit**: Clients kunnen hun eigen interne IDs gebruiken
- **Traceerbaarheid**: Beide keys (client en server) worden gelogd en getrackt

> **🔴 CRITISCHE REGEL**: **De client mag NOOIT de server key dicteren**. De server genereert **altijd** zijn eigen identifier (als string) en retourneert deze. De client stuurt alleen zijn eigen `clientId`, en de server bepaalt zelf wat de server-side identifier (`id`) wordt. Dit voorkomt key-conflicten tussen verschillende aanbieders en staat clients toe om elk gewenst ID-formaat te gebruiken.

**Belangrijk**: DD API V4 Write gebruikt **per-entity endpoints** met **per-entity job tracking** via een **enkel endpoint per job**:

```
/v4/write/observation/      ← Alle observation operaties
├── POST                    ← Batch submit
└── GET {jobId}             ← Job status + resultaten (als klaar)

/v4/write/foi/              ← Alle FoI operaties
├── POST                    ← Batch submit
└── GET {jobId}             ← Job status + resultaten (als klaar)

/v4/write/parameter/        ← Alle parameter operaties
├── POST                    ← Batch submit
└── GET {jobId}             ← Job status + resultaten (als klaar)
```

> **Let op**: Er is **geen apart /results endpoint**. Het `GET /{entity}/{jobId}` endpoint retourneert **alleen basisinformatie** tijdens verwerking, en **status + gedetailleerde resultaten** wanneer de job voltooid is (of gefaald).

## Vereenvoudigd Model (INSERT/UPDATE/DEPRECATE)

### Key Exchange Regels

**INSERT (CREATE) Operatie**:
- Client stuurt `clientId` (eigen identifier) in de entiteit
- `id` MOET **afwezig of null** zijn in de request
- Server genereert zijn eigen `id` (string)
- Server retourneert **zowel `clientId` als `id`** in de response

**UPDATE Operatie**:
- Client stuurt `id` (server identifier) in de entiteit
- `clientId` MOET **afwezig of null** zijn in de request
- Server retourneert alleen `id` in de response (clientId is null)

**DEPRECATE Operatie**:
- Client stuurt `id` (server identifier) in de data
- `clientId` MOET **afwezig of null** zijn in de request
- Server retourneert alleen `id` in de response (clientId is null)

## Mechanisme

### 1. Insert Operatie (Key Exchange)

Wanneer een client een nieuwe entiteit wil toevoegen via een per-entity endpoint:

```
Client                          Server
   │                               │
   │  POST /v4/write/observation   │
   │  {                            │
   │    operations: [{             │
   │      operation: "insert",      │  ← INSERT (niet upsert)
   │      id: "op1",                │
   │      data: {                   │
   │        observation: {          │
   │          clientId: "client-obs-001",  ← Client's clientId (VEREIST voor INSERT)
   │          // id MOET afwezig zijn
   │          type: "measure",
   │          ...
   │        }
   │      }
   │    }]
   │  }
   │──────────────────────────────>│
   │                               │
   │  202 Accepted                  │
   │  { jobId: "...", location: "..." }│
   │<──────────────────────────────│
   │                               │
   │  GET /v4/write/observation/{jobId}  ← Status opvragen
   │──────────────────────────────>│
   │                               │
   │  {                            │
   │    status: "completed",       │
   │    operations: [{             │
   │      clientId: "client-obs-001",  ← Originele client clientId
   │      id: "550e8400..."         ← **Server gegenereerde ID (string)**
   │    }]
   │  }
   │<──────────────────────────────│
```

### 2. Update Operatie (Server Key Vereist)

Voor update operaties op een bestaande entiteit:

```
Client                          Server
   │                               │
   │  POST /v4/write/observation   │
   │  {                            │
   │    operations: [{             │
   │      operation: "update",      │  ← UPDATE
   │      id: "op2",                │
   │      data: {                   │
   │        observation: {          │
   │          id: "550e8400..."     ← **Server ID (string) VEREIST**
   │          // clientId MOET afwezig zijn
   │          type: "measure",
   │          ...
   │        }
   │      }
   │    }]
   │  }
   │──────────────────────────────>│
```

### 3. Retract Operatie (Server Key Vereist)

Voor retract operaties:

```
Client                          Server
   │                               │
   │  POST /v4/write/observation   │
   │  {                            │
   │    operations: [{             │
   │      operation: "retract",   ← DEPRECATE
   │      id: "op3",                │
   │      data: {                   │
   │        id: "550e8400..."       ← **Server ID (string) VEREIST**
   │        // clientId MOET afwezig zijn
   │        reason: "Foutieve meting"
   │      }
   │    }]
   │  }
   │──────────────────────────────>│
```

**Belangrijk**: 
- De client MOET de `id` (server identifier) gebruiken voor alle UPDATE/DEPRECATE operaties
- `clientId` MOET **afwezig of null** zijn voor UPDATE/DEPRECATE
- Gebruik van `clientId` in UPDATE/DEPRECATE operaties zal resulteren in een fout
- **De client mag NOOIT proberen de server ID te dicteren** - de server bepaalt deze altijd zelf

## Request Structuur

### Insert Operatie

In de request naar een per-entity endpoint voor INSERT:
- `clientId` is **VEREIST** - de client's identifier
- `id` MOET **afwezig of null** zijn
- Server genereert zijn eigen `id` (string)

```http
POST /v4/write/observation
```
```jsonc
{
  "operations": [
    {
      "operation": "insert",
      "id": "op1",
      "data": {
        "observation": {
          "clientId": "my-observation-001",    // Client's clientId (VEREIST voor INSERT)
          // id MOET afwezig of null zijn
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
            "id": "550e8400-e29b-41d4-a716-446655440002"  // Server ID van FoI
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
    }
  ]
}
```

### Update Operatie

In de request voor UPDATE:
- `id` (server identifier) is **VEREIST**
- `clientId` MOET **afwezig of null** zijn

```http
POST /v4/write/observation
```
```jsonc
{
  "operations": [
    {
      "operation": "update",
      "id": "op2",
      "data": {
        "observation": {
          "id": "550e8400-e29b-41d4-a716-446655440001",  // Server ID (string) VEREIST
          // clientId MOET afwezig of null zijn
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
            "id": "550e8400-e29b-41d4-a716-446655440002"  // Server ID van FoI
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
    }
  ]
}
```

### Regels voor Identifiers

1. **INSERT**: `clientId` **VEREIST**, `id` MOET afwezig zijn
2. **UPDATE/DEPRECATE**: `id` (server) **VEREIST**, `clientId` MOET afwezig zijn
3. **FoI en Parameter referenties**: MOETEN server IDs (string) zijn in alle operaties
4. **🔴 CRITISCH**: **De client mag NOOIT de server ID dicteren** - de server genereert deze altijd zelf als string
5. **clientId veld**: Deprecated - gebruik `clientId` in plaats daarvan

## Response Structuur

### Direct Response (202 Accepted)

```jsonc
POST /v4/write/observation
{
  "operations": [...]
}

Response:
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "createdAt": "2024-01-15T10:01:00Z",
  "totalOperations": 1,
  "location": "/v4/write/observation/550e8400-e29b-41d4-a716-446655440000"
}
```

### Job Status Response (GET /v4/write/observation/{jobId})

**Tijdens verwerking** (status = `queued` of `processing`):

```json
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "createdAt": "2024-01-15T10:01:00Z",
  "startedAt": "2024-01-15T10:01:05Z",
  "totalOperations": 1
}
```

**Na voltooiing** (status = `completed` of `failed`):

In de response van een job worden zowel de client `clientId` als server `id` geretourneerd:

```jsonc
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "createdAt": "2024-01-15T10:01:00Z",
  "completedAt": "2024-01-15T10:01:10Z",
  "totalOperations": 1,
  "successfulOperations": 1,
  "failedOperations": 0,
  "operations": [
    {
      "operationIndex": 0,
      "operationId": "op1",
      "status": "succeeded",
      "clientId": "my-observation-001",  // Originele client clientId (alleen bij INSERT)
      "id": "550e8400-e29b-41d4-a716-446655440001",  // **Server gegenereerde ID (string)**
      "messages": []
    }
  ],
  "errors": []
}
```

### Velden in OperationResult

| Veld             | Type    | Beschrijving                                                              |
|------------------|---------|---------------------------------------------------------------------------|
| `operationIndex` | integer | Index in de originele batch                                               |
| `operationId`    | string  | ID uit het request, of gegenereerd                                        |
| `status`         | string  | succeeded, failed, skipped                                                |
| `clientId`       | string  | Originele client clientId (alleen bij INSERT, null voor UPDATE/DEPRECATE) |
| `id`             | string  | **Server gegenereerde identifier (string)**                               |
| `messages`       | array   | Foutmeldingen of waarschuwingen                                           |

## Server Logic

### Key Generatie

De server genereert unieke IDs volgens deze regels:

1. **Server-controlled**: De server genereert **altijd zelf** de identifier
2. **String**: De identifier is altijd een string (conform DD API V4 specificatie)
3. **Uniek**: Garandeert uniekheid binnen het systeem
4. **🔴 CRITISCH**: De client mag **nooit** de server ID dicteren. Dit voorkomt:
   - Key-conflicten tussen verschillende data-aanbieders
   - Problemen wanneer clients verschillende ID-formaten gebruiken

### Key Mapping

De server onderhoudt een interne mapping tussen client en server IDs:

```
Client clientId (elk formaat)  → Server ID (string)
client-obs-001                   → 550e8400-e29b-41d4-a716-446655440001
LOC001                           → 550e8400-e29b-41d4-a716-446655440002
my-parameter-001                 → 550e8400-e29b-41d4-a716-446655440003
```

Deze mapping wordt gebruikt voor:
- Validatie van client clientIds bij INSERT
- Retourneren van server IDs in responses

### Validatie

Bij een **INSERT** operatie:

1. Controleer of `clientId` aanwezig is
2. Controleer dat `id` afwezig of null is
3. **Geen validatie op formaat van clientId** - de client mag elk formaat gebruiken
4. Controleer of de clientId nog niet bestaat in de server (tenzij het een re-INSERT is)

Bij een **UPDATE/DEPRECATE** operatie:

1. Controleer of `id` aanwezig is
2. Controleer dat de `id` een **geldige server ID** is (d.w.z. een string die door de server is gegenereerd)
3. Controleer of `clientId` afwezig of null is
4. Controleer of de entiteit met die server ID bestaat
5. **Fout**: Als `clientId` aanwezig is of als `id` geen server ID is

## Foutcodes

### Identifier Gerelateerde Fouten

| Code                   | Beschrijving                         | Wanneer                                                   |
|------------------------|--------------------------------------|-----------------------------------------------------------|
| `SOURCEID_REQUIRED`    | clientId is vereist voor INSERT      | clientId is leeg of afwezig bij INSERT                    |
| `ID_NOT_ALLOWED`       | id mag niet aanwezig zijn bij INSERT | id is aanwezig bij INSERT operatie                        |
| `SOURCEID_NOT_ALLOWED` | clientId mag niet aanwezig zijn      | clientId is aanwezig bij UPDATE/DEPRECATE                 |
| `ID_REQUIRED`          | id is vereist                        | id is leeg of afwezig bij UPDATE/DEPRECATE                |
| `INVALID_SERVER_ID`    | Ongeldige server ID                  | Server ID is leeg of niet gevonden in het systeem         |
| `ENTITY_NOT_FOUND`     | Entiteit niet gevonden               | Server ID (string) bestaat niet in het systeem            |
| `SOURCEID_IN_UPDATE`   | clientId in UPDATE/DEPRECATE         | Client probeert clientId te gebruiken in UPDATE/DEPRECATE |

### Voorbeeld Foutmeldingen

#### clientId vereist voor INSERT

```json
{
  "operationIndex": 0,
  "operationId": "op1",
  "status": "failed",
  "clientId": null,
  "id": null,
  "messages": [
    {
      "code": "SOURCEID_REQUIRED",
      "message": "clientId is required for INSERT operation",
      "severity": "error",
      "path": "/operations/0/data/observation"
    }
  ]
}
```

#### id niet toegestaan bij INSERT

```json
{
  "operationIndex": 0,
  "operationId": "op1",
  "status": "failed",
  "clientId": null,
  "id": null,
  "messages": [
    {
      "code": "ID_NOT_ALLOWED",
      "message": "id must be absent or null for INSERT operation",
      "severity": "error",
      "path": "/operations/0/data/observation/id"
    }
  ]
}
```

#### clientId in UPDATE

```json
{
  "operationIndex": 0,
  "operationId": "op2",
  "status": "failed",
  "clientId": null,
  "id": null,
  "messages": [
    {
      "code": "SOURCEID_NOT_ALLOWED",
      "message": "clientId must be absent for UPDATE operation",
      "severity": "error",
      "path": "/operations/0/data/observation/clientId"
    }
  ]
}
```

#### Entity Not Found (met server ID)

```json
{
  "operationIndex": 0,
  "operationId": "op3",
  "status": "failed",
  "clientId": null,
  "id": null,
  "messages": [
    {
      "code": "ENTITY_NOT_FOUND",
      "message": "Observation with server ID 550e8400-e29b-41d4-a716-446655449999 not found",
      "severity": "error",
      "path": "/operations/0/data/id"
    }
  ]
}
```
## Voorbeelden

### Voorbeeld 1: Complete Workflow

```
1. Client wil een nieuwe FoI toevoegen

Request naar /v4/write/foi:
{
  "operations": [{
    "operation": "insert",
    "id": "create-foi",
    "data": {
      "foi": {
        "clientId": "my-location-001",      // Client's clientId (VEREIST)
        // id MOET afwezig zijn
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
  }]
}

Response (202 Accepted):
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "location": "/v4/write/foi/550e8400-e29b-41d4-a716-446655440000"
}

2. Client pollt de job status via /v4/write/foi/{jobId}:

GET /v4/write/foi/550e8400-e29b-41d4-a716-446655440000

Response (job voltooid):
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "createdAt": "2024-01-15T10:01:00Z",
  "startedAt": "2024-01-15T10:01:05Z",
  "completedAt": "2024-01-15T10:01:10Z",
  "totalOperations": 1,
  "successfulOperations": 1,
  "failedOperations": 0,
  "operations": [{
    "operationIndex": 0,
    "operationId": "create-foi",
    "status": "succeeded",
    "clientId": "my-location-001",
    "id": "550e8400-e29b-41d4-a716-446655440002",  // **Server gegenereerde ID (string)**
    "messages": []
  }],
  "errors": []
}

Client slaat op: my-location-001 → 550e8400-e29b-41d4-a716-446655440002

3. Client wil een observatie toevoegen met referentie naar de FoI

Request naar /v4/write/observation:
{
  "operations": [{
    "operation": "insert",
    "id": "create-obs",
    "data": {
      "observation": {
        "clientId": "my-observation-001",    // Client's clientId (VEREIST)
        // id MOET afwezig zijn
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
          "id": "550e8400-e29b-41d4-a716-446655440002"  // ← **Server gegenereerde ID (string) van FoI**
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
  }]
}

Response (202 Accepted):
{
  "jobId": "660e8400-e29b-41d4-a716-446655440001",
  "status": "queued",
  "location": "/v4/write/observation/660e8400-e29b-41d4-a716-446655440001"
}

4. Client pollt observation job:

GET /v4/write/observation/660e8400-e29b-41d4-a716-446655440001

Response:
{
  "jobId": "660e8400-e29b-41d4-a716-446655440001",
  "status": "completed",
  "createdAt": "2024-01-15T10:02:00Z",
  "startedAt": "2024-01-15T10:02:05Z",
  "completedAt": "2024-01-15T10:02:10Z",
  "totalOperations": 1,
  "successfulOperations": 1,
  "failedOperations": 0,
  "operations": [{
    "operationIndex": 0,
    "operationId": "create-obs",
    "status": "succeeded",
    "clientId": "my-observation-001",
    "id": "660e8400-e29b-41d4-a716-446655440001",  // **Server gegenereerde ID (string)**
    "messages": []
  }],
  "errors": []
}

Client slaat op: my-observation-001 → 660e8400-e29b-41d4-a716-446655440001

5. Client wil de observatie updaten

Request naar /v4/write/observation:
{
  "operations": [{
    "operation": "update",
    "id": "update-obs",
    "data": {
      "observation": {
        "id": "660e8400-e29b-41d4-a716-446655440001",  // ← **Server gegenereerde ID (string)**
        // clientId MOET afwezig zijn
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
          "id": "550e8400-e29b-41d4-a716-446655440002"  // ← Server gegenereerde ID (string) van FoI
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
  }]
}

6. Client pollt de job status om resultaat te zien:

GET /v4/write/observation/{new-job-id}

Response (voltooid):
{
  "jobId": "770e8400-e29b-41d4-a716-446655440003",
  "status": "completed",
  "createdAt": "2024-01-15T11:00:00Z",
  "startedAt": "2024-01-15T11:00:05Z",
  "completedAt": "2024-01-15T11:00:10Z",
  "totalOperations": 1,
  "successfulOperations": 1,
  "failedOperations": 0,
  "operations": [{
    "operationIndex": 0,
    "operationId": "update-obs",
    "status": "succeeded",
    "clientId": null,  // null voor UPDATE
    "id": "660e8400-e29b-41d4-a716-446655440001",  // **Server gegenereerde ID (string)**
    "messages": []
  }],
  "errors": []
}
```

### Voorbeeld 2: Fout - id aanwezig bij INSERT

```
Request naar /v4/write/observation:
{
  "operations": [{
    "operation": "insert",
    "id": "op1",
    "data": {
      "observation": {
        "clientId": "my-observation-001",
        "id": "should-not-be-here",  // ← id MOET afwezig zijn bij INSERT!
        "type": "measure",
        "resultTime": "2024-01-15T10:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T10:00:00Z",
          "end": "2024-01-15T10:00:00Z"
        },
        "parameter": {"parameter/waarnemingssoort": "WATHTE"}
      }
    }
  }]
}

Response (na polling via /v4/write/observation/{jobId}):
{
  "jobId": "770e8400-e29b-41d4-a716-446655440003",
  "status": "completed",
  "createdAt": "2024-01-15T11:05:00Z",
  "startedAt": "2024-01-15T11:05:05Z",
  "completedAt": "2024-01-15T11:05:10Z",
  "totalOperations": 1,
  "successfulOperations": 0,
  "failedOperations": 1,
  "operations": [{
    "operationIndex": 0,
    "operationId": "op1",
    "status": "failed",
    "clientId": null,
    "id": null,
    "messages": [{
      "code": "ID_NOT_ALLOWED",
      "message": "id must be absent or null for INSERT operation",
      "severity": "error",
      "path": "/operations/0/data/observation/id"
    }]
  }],
  "errors": []
}
```

### Voorbeeld 3: Fout - clientId in UPDATE

```
Request naar /v4/write/observation:
{
  "operations": [{
    "operation": "update",
    "id": "op2",
    "data": {
      "observation": {
        "id": "660e8400-e29b-41d4-a716-446655440001",
        "clientId": "my-observation-001",  // ← clientId MOET afwezig zijn bij UPDATE!
        "type": "measure",
        "resultTime": "2024-01-15T11:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T11:00:00Z",
          "end": "2024-01-15T11:00:00Z"
        },
        "parameter": {"parameter/waarnemingssoort": "WATHTE"}
      }
    }
  }]
}

Response (na polling):
{
  "jobId": "880e8400-e29b-41d4-a716-446655440004",
  "status": "completed",
  "createdAt": "2024-01-15T11:10:00Z",
  "startedAt": "2024-01-15T11:10:05Z",
  "completedAt": "2024-01-15T11:10:10Z",
  "totalOperations": 1,
  "successfulOperations": 0,
  "failedOperations": 1,
  "operations": [{
    "operationIndex": 0,
    "operationId": "op2",
    "status": "failed",
    "clientId": null,
    "id": null,
    "messages": [{
      "code": "SOURCEID_NOT_ALLOWED",
      "message": "clientId must be absent for UPDATE operation",
      "severity": "error",
      "path": "/operations/0/data/observation/clientId"
    }]
  }],
  "errors": []
}
```

### Voorbeeld 4: Fout - Ongeldige FoI Referentie (niet-server ID)

```
Request naar /v4/write/observation:
{
  "operations": [{
    "operation": "insert",
    "id": "op1",
    "data": {
      "observation": {
        "clientId": "my-observation-001",
        "type": "measure",
        "resultTime": "2024-01-15T10:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T10:00:00Z",
          "end": "2024-01-15T10:00:00Z"
        },
        "parameter": {"parameter/waarnemingssoort": "WATHTE"},
        "foi": {
          "id": "client-foi-001"  // ← Client ID (niet-server ID) in FoI referentie!
        },
        "result": {"id": "result-001", "measure": {"uom": "m", "value": 1.5}},
        "modifiedOn": "2024-01-15T10:00:00Z"
      }
    }
  }]
}

Response (na polling):
{
  "jobId": "880e8400-e29b-41d4-a716-446655440004",
  "status": "completed",
  "createdAt": "2024-01-15T10:10:00Z",
  "startedAt": "2024-01-15T10:10:05Z",
  "completedAt": "2024-01-15T10:10:10Z",
  "totalOperations": 1,
  "successfulOperations": 0,
  "failedOperations": 1,
  "operations": [{
    "operationIndex": 0,
    "operationId": "op1",
    "status": "failed",
    "clientId": null,
    "id": null,
    "messages": [{
      "code": "INVALID_FOI_REFERENCE",
      "message": "FoI reference must use server ID (string)",
      "severity": "error",
      "path": "/operations/0/data/observation/foi/id",
      "details": {
        "providedId": "client-foi-001",
        "expected": "Server ID (string)"
      }
    }]
  }],
  "errors": []
}
```

## Samenvatting

| Scenario              | Endpoint                | Request                                            | Response `clientId` | Response `id`      | Notities                                 |
|-----------------------|-------------------------|----------------------------------------------------|---------------------|--------------------|------------------------------------------|
| Insert Observation    | POST /write/observation | `clientId` **VEREIST**, `id` MOET afwezig          | Client clientId     | Server ID (string) | Server genereert altijd zelf ID (string) |
| Update Observation    | POST /write/observation | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |
| Retract Observation | POST /write/observation | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |
| Insert FoI            | POST /write/foi         | `clientId` **VEREIST**, `id` MOET afwezig          | Client clientId     | Server ID (string) | Server genereert altijd zelf ID (string) |
| Update FoI            | POST /write/foi         | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |
| Retract FoI         | POST /write/foi         | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |
| Insert Parameter      | POST /write/parameter   | `clientId` **VEREIST**, `id` MOET afwezig          | Client clientId     | Server ID (string) | Server genereert altijd zelf ID (string) |
| Update Parameter      | POST /write/parameter   | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |

> **Let op**: Er is **geen apart /results endpoint**. Het `GET /{entity}/{jobId}` endpoint retourneert **alleen basisinformatie** tijdens verwerking, en **status + gedetailleerde resultaten** wanneer de job voltooid is.

**Belangrijkste Regels**:

1. ✅ **INSERT**: alleen `clientId` in request, `id` MOET afwezig zijn
2. ✅ **UPDATE/DEPRECATE**: alleen `id` (server) in request, `clientId` MOET afwezig zijn
3. ❌ **INSERT met id**: **Fout** - id mag niet aanwezig zijn
4. ❌ **UPDATE met clientId**: **Fout** - clientId mag niet aanwezig zijn
5. ✅ **Referenties**: FoI en Parameter referenties MOETEN **server IDs (string)** zijn
6. ✅ **Job Tracking**: Gebruik het **specifieke entiteit endpoint** voor job status
7. ✅ **Mapping**: Client MOET clientId-to-serverId mapping bijhouden
8. ✅ **Validatie**: Server valideert dat identifiers correct zijn
9. ✅ **Single Endpoint**: Gebruik `GET /{entity}/{jobId}` voor zowel status als resultaten

**Vereenvoudigd**: Geen `clientId` veld meer nodig - alleen `clientId` voor INSERT en `id` voor UPDATE/DEPRECATE!

Zie [Endpoints](04-00-endpoints.md) voor complete endpoint documentatie.
