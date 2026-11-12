# DD API V4 Write - Key Exchange Mechanisme

## Overzicht

DD API V4 Write implementeert een **vereenvoudigd key-exchange** mechanisme voor het beheer van entiteit-identificaties. Dit zorgt voor:

- **Eenduidige identificatie**: Server-side keys garanderen unieke identificatie binnen het systeem
- **Security**: Server beheert de autoritatieve identificaties
- **Flexibiliteit**: Clients kunnen hun eigen interne ID's gebruiken
- **Traceerbaarheid**: Beide keys (client en server) worden gelogd en getrackt

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

## Vereenvoudigd Model (INSERT/UPDATE/RETRACT)

### Key Exchange Regels

**INSERT (CREATE) Operatie**:
- Client stuurt `clientId` **op operation niveau** (eigen identifier, kan elk formaat zijn)
- `observation.id` / `foi.id` / `parameter.id` MOET lege string `""` zijn in de request
- Server genereert zijn eigen `id` (string)
- Server retourneert **zowel `clientId` als `id`** in de response

**UPDATE Operatie**:
- Client stuurt `id` (server identifier) in de entiteit
- `clientId` MOET **afwezig** zijn op operation niveau
- Server retourneert alleen `id` in de response (clientId is null)

**RETRACT Operatie**:
- Client stuurt `id` (server identifier) in de data
- `clientId` MOET **afwezig** zijn op operation niveau
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
   │      operation: "insert",      │  ← INSERT
   │      clientId: "client-obs-001", ← clientId op OPERATION niveau (VEREIST)
   │      data: {                    │
   │        observation: {          │
   │          id: "",               ← id MOET lege string "" zijn
   │          type: "measure",      │
   │          ...                   │
   │        }                       │
   │      },
   │      id: "op1"                 │
   │    }]                          │
   │  }                            │
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
   │      operationIndex: 0,       │
   │      operationId: "op1",      │
   │      action: "insert",        │
   │      clientId: "client-obs-001", ← Originele clientId
   │      status: "succeeded",     │
   │      data: {                  │  ← Hele entiteit met server ID
   │        id: "550e8400...",     ← Server gegenereerde ID (string)
   │        type: "measure",       │
   │        ...                    │
   │      }                        │
   │    }]                          │
   │  }                            │
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
   │      data: {                    │
   │        observation: {          │
   │          id: "550e8400..."     ← Server ID (string) VEREIST
   │          type: "measure",      │
   │          ...                   │
   │        }                       │
   │      },
   │      id: "op2"                 │
   │    }]                          │
   │  }                            │
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
   │      operation: "retract",     ← RETRACT
   │      data: {                    │
   │        id: "550e8400..."       ← Server ID (string) VEREIST
   │        reason: "Foutieve meting"
   │        modifiedOn: "2024-01-15T10:00:00Z"
   │      },
   │      id: "op3"                 │
   │    }]                          │
   │  }                            │
   │──────────────────────────────>│
```

**Belangrijk verschil met INSERT/UPDATE**: 
Voor RETRACT is de data structuur **direct** (`data: { id, reason, modifiedOn }`) en niet gewrapt in een `observation` object. Dit is specifiek voor retract operaties.

**Belangrijk**: 
- De client MOET de `id` (server identifier) gebruiken voor alle UPDATE/RETRACT operaties
- `clientId` MOET **afwezig** zijn op operation niveau voor UPDATE/RETRACT
- Gebruik van `clientId` in UPDATE/RETRACT operaties zal resulteren in een fout
- **De client mag NOOIT proberen de server-ID te dicteren** - de server bepaalt deze altijd zelf

## Request Structuur

### Insert Operatie

In de request naar een per-entity endpoint voor INSERT:
- `clientId` **VEREIST** op operation niveau (naast operation en id)
- `id` in de entiteit MOET lege string `""` zijn
- Server genereert zijn eigen `id` (string)

```http
POST /v4/write/observation
```
```jsonc
{
  "operations": [
    {
      "operation": "insert",
      "clientId": "my-observation-001",    // clientId op OPERATION niveau (VEREIST voor INSERT)
      "data": {
        "observation": {
          "id": "",                       // MOET lege string zijn
          "type": "measure",
          "resultTime": "2024-01-15T10:00:00Z",
          "phenomenonTime": {
            "begin": "2024-01-15T10:00:00Z",
            "end": "2024-01-15T10:00:00Z"
          },
          "parameter": {
            "quantity": "WATHTE",
            "compartment": "OW"
          },
          "foi": {
            "id": "550e8400-e29b-41d4-a716-446655440002",  // Server-ID van FoI
            "code": "my-location-code",
            "geometry": {}
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
      },
      "id": "op1"
    }
  ]
}
```

### Update Operatie

In de request voor UPDATE:
- `id` (server identifier) is **VEREIST** in de entiteit
- `clientId` MOET **afwezig** zijn op operation niveau

```http
POST /v4/write/observation
```
```jsonc
{
  "operations": [
    {
      "operation": "update",
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
            "quantity": "WATHTE",
            "compartment": "OW"
          },
          "foi": {
            "id": "550e8400-e29b-41d4-a716-446655440002",  // Server ID van FoI
            "code": "my-location-code",
            "geometry": {}
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
      },
      "id": "op2"
    }
  ]
}
```

### Retract Operatie

In de request voor RETRACT:
- `id` (server identifier) is **VEREIST** in de data (direct, geen observation wrapper)
- `clientId` MOET **afwezig** zijn op operation niveau
- Data structuur: `data: { id, reason, modifiedOn }`

```http
POST /v4/write/observation
```
```jsonc
{
  "operations": [
    {
      "operation": "retract",
      "data": {
        "id": "550e8400-e29b-41d4-a716-446655440001",  // Server ID (string) VEREIST
        "reason": "Foutieve meting",
        "modifiedOn": "2024-01-15T10:00:00Z"
      },
      "id": "op3"
    }
  ]
}
```

### Regels voor Identifiers

1. **INSERT**: `clientId` **VEREIST** op operation niveau, `id` in entiteit MOET lege string `""` zijn
2. **UPDATE/RETRACT**: `id` (server) **VEREIST** in entiteit/data, `clientId` MOET **afwezig** zijn op operation niveau
3. **FoI en Parameter referenties**: MOETEN **server-ID's (string)** zijn in alle operaties

## Response Structuur

### Direct Response (202 Accepted)

POST /v4/write/observation
```jsonc
{
  "operations": [...]
}
````

Response:
```jsonc
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

In de response van een job worden `action`, `clientId` (alleen bij INSERT), en het gehele data-blok (entiteit) geretourneerd:

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
      "action": "insert",
      "clientId": "my-observation-001",  // Originele clientId (alleen bij INSERT)
      "status": "succeeded",
      "data": {
        "id": "550e8400-e29b-41d4-a716-446655440001",  // Server gegenereerde ID (string)
        "type": "measure",
        "resultTime": "2024-01-15T10:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T10:00:00Z",
          "end": "2024-01-15T10:00:00Z"
        },
       "parameter": {
         "quantity": "WATHTE",
         "compartment": "OW"
       },
       "foi": {
         "id": "550e8400-e29b-41d4-a716-446655440002",  // Server ID van FoI
         "code": "my-location-code",
         "geometry": {}
       },
        "result": {
          "id": "my-result-001",
          "measure": {
            "uom": "m",
            "value": 1.5
          }
        },
        "modifiedOn": "2024-01-15T10:00:00Z"
      },
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
| `action`         | string  | insert, update, retract (de requested actie)                              |
| `status`         | string  | succeeded, failed, skipped                                                |
| `clientId`       | string  | Originele clientId (alleen bij INSERT, null voor UPDATE/RETRACT)          |
| `data`           | object  | **Hele entiteit** met server IDs (string)                                 |
| `messages`       | array   | Foutmeldingen of waarschuwingen                                           |

## Server Logic

### Key Generatie

De server genereert unieke ID's volgens deze regels:

1. **Server-controlled**: De server genereert **altijd zelf** de identifier
2. **String**: De identifier is altijd een string (conform DD API V4 specificatie)
3. **Uniek**: Garandeert uniekheid binnen het systeem

### Validatie

Bij een **INSERT** operatie:

1. Controleer of `clientId` aanwezig is op operation niveau
2. Controleer dat `id` in de entiteit lege string `""` is
3. **Geen validatie op formaat van clientId** - de client mag elk formaat gebruiken

Bij een **UPDATE/RETRACT** operatie:

1. Controleer of `id` (server identifier) aanwezig is in de entiteit
2. Controleer dat de `id` een **geldige server-ID** is (d.w.z. een string die door de server is gegenereerd)
3. Controleer of `clientId` afwezig is op operation niveau
4. Controleer of de entiteit met die server-ID bestaat
5. **Fout**: Als `clientId` aanwezig is of als `id` geen server-ID is

### Key Mapping

De server onderhoudt een interne mapping tussen client en server-ID's:

```
Client clientId (elk formaat)  → Server-ID (string)
client-obs-001                   → 550e8400-e29b-41d4-a716-446655440001
my-location-001                   → 550e8400-e29b-41d4-a716-446655440002
my-parameter-001                 → 550e8400-e29b-41d4-a716-446655440003
```

Deze mapping wordt gebruikt voor:
- Validatie van client client-Ids bij INSERT
- Retourneren van server-ID's in responses

## Foutcodes

### Identifier Gerelateerde Fouten

| Code                   | Beschrijving                         | Wanneer                                                    |
|------------------------|--------------------------------------|------------------------------------------------------------|
| `CLIENTID_REQUIRED`    | clientId is vereist voor INSERT      | clientId is leeg of afwezig bij INSERT op operation niveau |
| `ID_NOT_ALLOWED`       | id mag niet aanwezig zijn bij INSERT | id is aanwezig en niet "" bij INSERT operatie              |
| `CLIENTID_NOT_ALLOWED` | clientId mag niet aanwezig zijn      | clientId is aanwezig bij UPDATE/RETRACT                    |
| `ID_REQUIRED`          | id is vereist                        | id is leeg of afwezig bij UPDATE/RETRACT                   |
| `INVALID_SERVER_ID`    | Ongeldige server ID                  | Server-ID is leeg of niet gevonden in het systeem          |
| `ENTITY_NOT_FOUND`     | Entiteit niet gevonden               | Server-ID (string) bestaat niet in het systeem             |

Zie [Validatie](03-00-validation.md) voor complete lijst van validatie foutcodes.

## Voorbeelden

### Voorbeeld 1: Complete Workflow

1. Client wil een nieuwe FoI toevoegen

Request naar /v4/write/foi:
```jsonc
{
  "operations": [{
    "operation": "insert",
    "clientId": "my-location-001",      // clientId op OPERATION niveau (VEREIST)
    "data": {
      "foi": {
        "id": "",                      // MOET lege string zijn
        "type": "https://definities.pdok.nl/def/cbsgb/1.0/id/datasetdefinitie/Top10NL",
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
    },
    "id": "create-foi"
  }]
}
````

Response (202 Accepted):
```jsonc
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "location": "/v4/write/foi/550e8400-e29b-41d4-a716-446655440000"
}
````

2. Client haalt de job status op via /v4/write/foi/{jobId}:

GET /v4/write/foi/550e8400-e29b-41d4-a716-446655440000

Response (job voltooid):
```jsonc
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
    "action": "insert",
    "clientId": "my-location-001",
    "status": "succeeded",
    "data": {
      "id": "550e8400-e29b-41d4-a716-446655440002",  // Server gegenereerde ID (string)
      "type": "https://definities.pdok.nl/def/cbsgb/1.0/id/datasetdefinitie/Top10NL",
      "name": "Meetlocatie Amsterdam",
      "organisation": "RWS",
      "code": "LOC001",
      "description": "Meetlocatie Amsterdam",
      "location": "https://pdok.nl",
      "geometry": {
        "type": "Point",
        "coordinates": [4.9, 52.37]
      }
    },
    "messages": []
  }],
  "errors": []
}
````

Client slaat op: my-location-001 → 550e8400-e29b-41d4-a716-446655440002

3. Client wil een observatie toevoegen met referentie naar de FoI

Request naar /v4/write/observation:
```jsonc
{
  "operations": [{
    "operation": "insert",
    "clientId": "my-observation-001",    // clientId op OPERATION niveau (VEREIST)
    "data": {
      "observation": {
        "id": "",                       // MOET lege string zijn
        "type": "measure",
        "resultTime": "2024-01-15T10:00:00Z",
        "phenomenonTime": { 
          "begin": "2024-01-15T10:00:00Z", 
          "end": "2024-01-15T10:00:00Z" 
        },
       "parameter": {
         "quantity": "WATHTE",
         "compartment": "OW"
       },
       "foi": {
         "id": "550e8400-e29b-41d4-a716-446655440002",  // Server ID van FoI
         "code": "my-location-code",
         "geometry": {}
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
    },
    "id": "create-obs"
  }]
}
````

Response (202 Accepted):
```jsonc
{
  "jobId": "660e8400-e29b-41d4-a716-446655440001",
  "status": "queued",
  "location": "/v4/write/observation/660e8400-e29b-41d4-a716-446655440001"
}
````

4. Client haalt observation job op:

GET /v4/write/observation/660e8400-e29b-41d4-a716-446655440001

Response:
```jsonc
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
    "action": "insert",
    "clientId": "my-observation-001",
    "status": "succeeded",
    "data": {
      "id": "660e8400-e29b-41d4-a716-446655440001",  // Server gegenereerde ID (string)
      "type": "measure",
      "resultTime": "2024-01-15T10:00:00Z",
      "phenomenonTime": { 
        "begin": "2024-01-15T10:00:00Z", 
        "end": "2024-01-15T10:00:00Z" 
      },
       "parameter": {
         "quantity": "WATHTE",
         "compartment": "OW"
       },
       "foi": {
         "id": "550e8400-e29b-41d4-a716-446655440002",  // Server ID van FoI
         "code": "my-location-code",
         "geometry": {}
       },
      "result": { 
        "id": "my-result-001", 
        "measure": { 
          "uom": "m", 
          "value": 1.5 
        } 
      },
      "modifiedOn": "2024-01-15T10:00:00Z"
    },
    "messages": []
  }],
  "errors": []
}
````

Client slaat op: my-observation-001 → 660e8400-e29b-41d4-a716-446655440001

5. Client wil de observatie terugtrekken

Request naar /v4/write/observation:
```jsonc
{
  "operations": [{
    "operation": "retract",
    "data": {
      "id": "660e8400-e29b-41d4-a716-446655440001",  // Server ID (string) VEREIST
      "reason": "Foutieve meting",
      "modifiedOn": "2024-01-15T10:00:00Z"
    },
    "id": "retract-obs"
  }]
}
````

**Belangrijk**: Voor RETRACT is de data structuur **direct** (geen observation wrapper), alleen `id`, `reason` en `modifiedOn`.

6. Client haalt de job status op om resultaat te zien:

GET /v4/write/observation/{new-job-id}

Response (voltooid):
```jsonc
{
  "jobId": "770e8400-e29b-41d4-a716-446655440003",
  "status": "completed",
  "createdAt": "2024-01-15T12:00:00Z",
  "startedAt": "2024-01-15T12:00:05Z",
  "completedAt": "2024-01-15T12:00:10Z",
  "totalOperations": 1,
  "successfulOperations": 1,
  "failedOperations": 0,
  "operations": [{
    "operationIndex": 0,
    "operationId": "retract-obs",
    "action": "retract",
    "clientId": null,  // null voor RETRACT
    "status": "succeeded",
    "data": {
      "id": "660e8400-e29b-41d4-a716-446655440001",  // Server ID (string)
      "reason": "Foutieve meting",
      "modifiedOn": "2024-01-15T12:00:00Z"
    },
    "messages": []
  }],
  "errors": []
}
```

### Voorbeeld 2: Fout - id niet lege string bij INSERT

Request naar /v4/write/observation:
```jsonc
{
  "operations": [{
    "operation": "insert",
    "clientId": "my-observation-001",
    "data": {
      "observation": {
        "id": "should-be-empty-string",  // ← id MOET lege string "" zijn bij INSERT!
        "type": "measure",
        "resultTime": "2024-01-15T10:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T10:00:00Z",
          "end": "2024-01-15T10:00:00Z"
        },
          "parameter": {
            "quantity": "WATHTE",
            "compartment": "OW"
          },
          "foi": {
            "id": "550e8400-e29b-41d4-a716-446655440002",  // Server ID van FoI
            "code": "my-location-code",
            "geometry": {}
          },
        "result": {"id": "my-result-001", "measure": {"uom": "m", "value": 1.5}},
        "modifiedOn": "2024-01-15T10:00:00Z"
      }
    },
    "id": "op1"
  }]
}
````

Response (na polling via /v4/write/observation/{jobId}):
```jsonc
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
    "action": "insert",
    "clientId": "my-observation-001",
    "status": "failed",
    "data": null,
    "messages": [{
      "code": "ID_NOT_ALLOWED",
      "message": "id must be empty string for INSERT operation",
      "severity": "error",
      "path": "/operations/0/data/observation/id"
    }]
  }],
  "errors": []
}
```

### Voorbeeld 3: Fout - clientId aanwezig bij UPDATE

Request naar /v4/write/observation:
```jsonc
{
  "operations": [{
    "operation": "update",
    "clientId": "my-observation-001",  // ← clientId MOET afwezig zijn bij UPDATE!
    "data": {
      "observation": {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "type": "measure",
        "resultTime": "2024-01-15T11:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T11:00:00Z",
          "end": "2024-01-15T11:00:00Z"
        },
          "parameter": {
            "quantity": "WATHTE",
            "compartment": "OW"
          },
          "foi": {
            "id": "550e8400-e29b-41d4-a716-446655440002",  // Server ID van FoI
            "code": "my-location-code",
            "geometry": {}
          },
        "result": {"id": "my-result-001", "measure": {"uom": "m", "value": 1.7}},
        "modifiedOn": "2024-01-15T11:00:00Z"
      }
    },
    "id": "op2"
  }]
}
````

Response (na polling via /v4/write/observation/{jobId}):
```jsonc
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
    "action": "update",
    "clientId": null,
    "status": "failed",
    "data": null,
    "messages": [{
      "code": "CLIENTID_NOT_ALLOWED",
      "message": "clientId must be absent for UPDATE operation",
      "severity": "error",
      "path": "/operations/0/clientId"
    }]
  }],
  "errors": []
}
```

## Samenvatting

| Scenario            | Endpoint                | Request                                                       | Response `clientId` | Response `action` | Notities                                 |
|---------------------|-------------------------|---------------------------------------------------------------|---------------------|-------------------|------------------------------------------|
| Insert Observation  | POST /write/observation | `clientId` **VEREIST** op operation niveau, `id` MOET "" zijn | Client clientId     | insert            | Server genereert altijd zelf ID (string) |
| Update Observation  | POST /write/observation | `id` (server) **VEREIST** in data, geen clientId              | null                | update            | MOET server ID (string) zijn             |
| Retract Observation | POST /write/observation | `id` (server) **VEREIST** in data, geen clientId              | null                | retract           | MOET server ID (string) zijn             |
| Insert FoI          | POST /write/foi         | `clientId` **VEREIST** op operation niveau, `id` MOET "" zijn | Client clientId     | insert            | Server genereert altijd zelf ID (string) |
| Update FoI          | POST /write/foi         | `id` (server) **VEREIST** in data, geen clientId              | null                | update            | MOET server ID (string) zijn             |
| Insert Parameter    | POST /write/parameter   | `clientId` **VEREIST** op operation niveau, `id` MOET "" zijn | Client clientId     | insert            | Server genereert altijd zelf ID (string) |
| Update Parameter    | POST /write/parameter   | `id` (server) **VEREIST** in data, geen clientId              | null                | update            | MOET server ID (string) zijn             |

> **Let op**: Er is **geen apart /results endpoint**. Het `GET /{entity}/{jobId}` endpoint retourneert **alleen basisinformatie** tijdens verwerking, en **status + gedetailleerde resultaten** wanneer de job voltooid is.

**Belangrijkste Regels**:

1. ✅ **INSERT**: `clientId` VEREIST op operation niveau, `id` in data MOET "" zijn
2. ✅ **UPDATE/RETRACT**: `id` (server) VEREIST in data, geen `clientId` op operation niveau
3. ❌ **INSERT met id != ""**: **Fout** - id moet lege string zijn
4. ❌ **UPDATE met clientId**: **Fout** - clientId mag niet aanwezig zijn
5. ✅ **Referenties**: FoI en Parameter referenties MOETEN **server-ID's (string)** zijn
6. ✅ **Job Tracking**: Gebruik het **specifieke entiteit endpoint** voor job status
7. ✅ **Mapping**: Client MOET clientId-to-serverId mapping bijhouden
8. ✅ **Validatie**: Server valideert dat identifiers correct zijn
9. ✅ **Single Endpoint**: Gebruik `GET /{entity}/{jobId}` voor zowel status als resultaten

Zie [Endpoints](04-00-endpoints.md) voor complete endpoint documentatie.
