# DD API V4 Write - Batch Structuur

## Overzicht

De Write module gebruikt een specifieke batch-structuur voor het versturen en ontvangen van data.
Deze structuur vervangt de OData-wrapper en is specifiek ontworpen voor batch-operaties.

**Belangrijk**: Het DD API V4 Write module gebruikt **per-entity endpoints** in plaats van een enkele generieke batch endpoint.
Dit betekent dat elk endpoint specifiek is voor één entiteittype (observation, foi, of parameter) **en zijn eigen job tracking endpoint heeft**.

> **🔴 REGEL**: **De client mag NOOIT de server key dicteren**. De server genereert **altijd** zijn eigen identifier (als string) en retourneert deze. De client stuurt alleen zijn eigen `clientId`, en de server bepaalt zelf wat de server-side identifier (`id`) wordt. Dit voorkomt key-conflicten tussen verschillende aanbieders en staat clients toe om elk gewenst ID-formaat te gebruiken.

## Endpoint Specifiek Design

Elke entiteit heeft zijn eigen endpoint met bijbehorende job tracking:

```
/v4/write/observation/      ← Observation batch operaties
├── POST                    ← Batch submit
└── GET {jobId}             ← Job status + resultaten (als klaar)

/v4/write/foi/              ← FoI batch operaties
├── POST                    ← Batch submit
└── GET {jobId}             ← Job status + resultaten (als klaar)

/v4/write/parameter/        ← Parameter batch operaties
├── POST                    ← Batch submit
└── GET {jobId}             ← Job status + resultaten (als klaar)
```

Hierdoor:
- Is het `entityType` veld **niet nodig** in de operatie-definitie
- Is de request structuur simpeler en duidelijker
- Kan validatie entiteit-specifiek worden uitgevoerd
- Is de verwerking per endpoint eenvoudiger
- **Job tracking is consistent**: status en resultaten via één endpoint

## Batch Request Structuur

Een batch-request bevat een verzameling operaties die moeten worden uitgevoerd op de specifieke entiteit.
Elke operatie is van het type **insert**, **update**, of **retract**. 

Retract is alleen geldig voor observations.

### Root Object

```yaml
type: object
required:
  - operations
properties:
  operations:
    type: array
    minItems: 1
    description: |-
      Array van operaties voor deze entiteit.
      **De server MOET een maximum batch grootte specificeren** en deze communiceren via een `X-Max-Batch-Size` header in 413 responses.
    items:
      $ref: '#/components/schemas/WriteOperation'
  options:
    $ref: '#/components/schemas/BatchOptions'
```

### WriteOperation

Elke operatie in de batch heeft de volgende structuur. **Merk op**: `entityType` is niet vereist omdat het endpoint dit al specificeert.

**Vereenvoudigd Key-Exchange Model**:
- **INSERT**: `clientId` in request is **VEREIST**, `id` MOET een lege string zijn
- **UPDATE**: `id` (server) in request is **VEREIST**, `clientId` MOET afwezig/null zijn
- **RETRACT**: `id` (server) in request is **VEREIST**, `clientId` MOET afwezig/null zijn
- In CREATE response: zowel `id` (server) als `clientId` (client) worden geretourneerd

```yaml
type: object
required:
  - operation
  - data
properties:
  operation:
    type: string
    enum: [insert, update, retract]
    description: Het type operatie dat moet worden uitgevoerd
  data:
    type: object
    description: De daadwerkelijke data voor de operatie
    oneOf:
      - $ref: '#/components/schemas/CreateOperationData'
      - $ref: '#/components/schemas/UpdateOperationData'
      - $ref: '#/components/schemas/RetractOperationData'
  clientId:
    type: string
    maxLength: 100
    description: |-
      Optionele unieke identifier voor deze operatie binnen de batch.
      Wordt gebruikt in de response om operaties te correleren.
      Als niet opgegeven, wordt een index-gebaseerde ID gegenereerd.
```

## Operatie Data Structuren

### CreateOperationData (INSERT)

Voor **insert** (CREATE) operaties:
- `clientId` is **VEREIST** - de client's identifier
- `id` MOET **een lege string** zijn
- Server genereert zijn eigen `id` (string) en retourneert zowel `id` als `clientId`

**Vereenvoudigde Key-Exchange Regels**:
- **INSERT**: alleen `clientId` in request, server retourneert `id` + `clientId`
- **UPDATE/RETRACT**: alleen `id` (server) in request, `clientId` MOET afwezig zijn

**Endpoint specifieke data**:

Voor **POST /v4/write/observation** (INSERT):
```yaml
CreateObservationData:
  type: object
  required:
    - observation
  properties:
    observation:
      allOf:
        - $ref: '#/components/schemas/DigitaleDelta.Observation'
        - type: object
          required:
            - clientId
            - type
            - resultTime
            - phenomenonTime
            - parameter
            - modifiedOn
          properties:
            clientId:
              type: string
              description: Client's identifier (VEREIST voor INSERT)
            id:
              type: string
              nullable: true
              description: MOET null of afwezig zijn voor INSERT
```

Voor **POST /v4/write/foi** (INSERT):
```yaml
CreateFoiData:
  type: object
  required:
    - foi
  properties:
    foi:
      allOf:
        - $ref: '#/components/schemas/DigitaleDelta.Foi'
        - type: object
          required:
            - clientId
            - type
            - name
            - location
            - geometry
          properties:
            clientId:
              type: string
              description: Client's identifier (VEREIST voor INSERT)
            id:
              type: string
              nullable: true
              description: MOET null of afwezig zijn voor INSERT
```

Voor **POST /v4/write/parameter** (INSERT):
```yaml
CreateParameterData:
  type: object
  required:
    - parameter
  properties:
    parameter:
      allOf:
        - $ref: '#/components/schemas/DigitaleDelta.Parameter'
        - type: object
          required:
            - clientId
            - name
            - definition
          properties:
            clientId:
              type: string
              description: Client's identifier (VEREIST voor INSERT)
            id:
              type: string
              nullable: true
              description: MOET null of afwezig zijn voor INSERT
```

### UpdateOperationData (UPDATE)

Voor **update** operaties:
- `id` (server identifier) is **VEREIST**
- `clientId` MOET **afwezig of null** zijn
- Alle entiteit-specifieke velden moeten aanwezig zijn

Voor **POST /v4/write/observation** (UPDATE):
```yaml
UpdateObservationData:
  type: object
  required:
    - observation
  properties:
    observation:
      allOf:
        - $ref: '#/components/schemas/DigitaleDelta.Observation'
        - type: object
          required:
            - id
            - type
            - resultTime
            - phenomenonTime
            - parameter
            - modifiedOn
          properties:
            id:
              type: string
              description: Server identifier (VEREIST voor UPDATE/DEPRECATE)
            clientId:
              type: string
              nullable: true
              description: MOET null of afwezig zijn voor UPDATE
```

Voor **POST /v4/write/foi** (UPDATE):
```yaml
UpdateFoiData:
  type: object
  required:
    - foi
  properties:
    foi:
      allOf:
        - $ref: '#/components/schemas/DigitaleDelta.Foi'
        - type: object
          required:
            - id
            - type
            - name
            - location
            - geometry
          properties:
            id:
              type: string
              description: Server identifier (VEREIST voor UPDATE/DEPRECATE)
            clientId:
              type: string
              nullable: true
              description: MOET null of afwezig zijn voor UPDATE
```

Voor **POST /v4/write/parameter** (UPDATE):
```yaml
UpdateParameterData:
  type: object
  required:
    - parameter
  properties:
    parameter:
      allOf:
        - $ref: '#/components/schemas/DigitaleDelta.Parameter'
        - type: object
          required:
            - id
            - name
            - definition
          properties:
            id:
              type: string
              description: Server identifier (VEREIST voor UPDATE)
            clientId:
              type: string
              nullable: true
              description: MOET null of afwezig zijn voor UPDATE
```

### DeprecateOperationData

Voor retract-operaties bevat `data` de identificatie van de entiteit die moet worden teruggetrokken.

**Key-Exchange Regel**:
- Het `id` veld MOET het id die de server heeft uitgegeven, bevatten (van een eerdere create response)
- `clientId` MOET afwezig zijn

```yaml
RetractOperationData:
  type: object
  required:
    - id
    - reason
  properties:
    id:
      type: string
      description: |-
        De **server ID (string)** van de entiteit die moet worden teruggetrokken.
        MOET een server ID zijn (string), niet een client ID.
    reason:
      type: string
      minLength: 1
      maxLength: 1000
      description: De reden voor terugtrekking
    effectiveDate:
      type: string
      format: date-time
      description: |-
        Datum waarop de terugtrekking in werking treedt.
        Als niet opgegeven, is dit direct (current timestamp).
```

## Batch Options

Optionele batch-instellingen:

```yaml
BatchOptions:
  type: object
  properties:
    continueOnError:
      type: boolean
      default: true
      description: |-
        Als true, blijven andere operaties in de batch worden verwerkt,
        zelfs als een operatie faalt.
        Als false, stopt de verwerking bij de eerste fout.
    validateOnly:
      type: boolean
      default: false
      description: |-
        Als true, wordt alleen validatie uitgevoerd, geen daadwerkelijke wijzigingen.
        Handig voor het testen van batch-requests.
    callbackUrl:
      type: string
      format: uri
      maxLength: 2048
      description: |-
        **Verplichte** URL waarnaar een notificatie wordt gestuurd wanneer de batch
        is voltooid. De implementatie **MOET** deze functionaliteit ondersteunen.
        Een mislukte callback wordt beschouwd als een fout in de job.
```

## Batch Response Structuur

### Direct Response (202 Accepted)

Wanneer een batch-request wordt geaccepteerd, retourneert de server een 202 Accepted status met:

```yaml
type: object
required:
  - jobId
  - status
  - createdAt
  - totalOperations
properties:
  jobId:
    type: string
    description: Unieke identifier voor deze batch-job
  status:
    type: string
    enum: [queued, processing, completed, failed]
    description: Huidige status van de job
  createdAt:
    type: string
    format: date-time
    description: Tijdstip waarop de job is aangemaakt
  totalOperations:
    type: integer
    description: Totaal aantal operaties in de batch
  location:
    type: string
    format: uri
    description: |-
      URL om de status van deze job op te vragen.
      Bijvoorbeeld: /v4/write/observation/{jobId}
```

### Voorbeeld Direct Response

```json
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "createdAt": "2024-01-15T10:01:00Z",
  "totalOperations": 2,
  "location": "/v4/write/observation/550e8400-e29b-41d4-a716-446655440000"
}
```

### Job Status Response (GET /v4/write/{entity}/{jobId})

De job status kan worden opgevraagd via het **specifieke entiteit endpoint**. De response varieert op basis van de job status:

- **Status `queued` of `processing`**: Alleen basisinformatie
- **Status `completed` of `failed`**: Basisinformatie + gedetailleerde operatie resultaten

```yaml
type: object
required:
  - jobId
  - status
  - createdAt
  - totalOperations
properties:
  jobId:
    type: string
  status:
    type: string
    enum: [queued, processing, completed, failed]
  createdAt:
    type: string
    format: date-time
  startedAt:
    type: string
    format: date-time
    nullable: true
  completedAt:
    type: string
    format: date-time
    nullable: true
  totalOperations:
    type: integer
  successfulOperations:
    type: integer
    nullable: true
    description: Alleen aanwezig als status == completed of failed
  failedOperations:
    type: integer
    nullable: true
    description: Alleen aanwezig als status == completed of failed
  operations:
    type: array
    nullable: true
    description: Alleen aanwezig als status == completed of failed
    items:
      $ref: '#/components/schemas/OperationResult'
  errors:
    type: array
    nullable: true
    description: Alleen aanwezig als status == completed of failed
    items:
      $ref: '#/components/schemas/BatchError'
```

### OperationResult

Resultaat voor een individuele operatie, inclusief **key-exchange** informatie:

```yaml
OperationResult:
  type: object
  required:
    - operationIndex
    - operationId
    - status
  properties:
    operationIndex:
      type: integer
      description: Index van de operatie in de originele batch
    operationId:
      type: string
      description: De ID die in het request was opgegeven, of gegenereerde index
    status:
      type: string
      enum: [succeeded, failed, skipped]
    clientId:
      type: string
      nullable: true
      description: |-
        De originele client clientId (alleen bij INSERT operaties, voor traceerbaarheid).
        Null voor UPDATE/DEPRECATE operaties.
    id:
      type: string
      nullable: true
      description: De **server gegenereerde identifier (string)** voor de entiteit
    messages:
      type: array
      items:
        type: object
        required:
          - code
          - message
        properties:
          code:
            type: string
            description: Foutcode
          message:
            type: string
            description: Menselijk leesbare foutmelding
          severity:
            type: string
            enum: [info, warning, error]
          path:
            type: string
            description: JSON pointer naar het veld met de fout
```

## Voorbeelden

### Voorbeeld 1: POST /v4/write/foi (Insert FoI - CREATE)

```jsonc
{
  "operations": [
    {
      "operation": "insert",
      "id": "create-location-1",
      "data": {
        "foi": {
          "clientId": "my-location-001",      // Client's clientId (VEREIST voor INSERT)
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
      }
    }
  ],
  "options": {
    "continueOnError": true,
    "validateOnly": false
  }
}
```

**Response**: Server retourneert zowel `clientId` als gegenereerde `id` (server):
```jsonc
{
  "operationIndex": 0,
  "operationId": "create-location-1",
  "status": "succeeded",
  "clientId": "my-location-001",        // Client's clientId
  "id": "550e8400-e29b-41d4-a716-446655440002",  // Server gegenereerde ID (string)
  "messages": []
}
```

### Voorbeeld 2: POST /v4/write/observation (Insert Observation - CREATE)

```jsonc
{
  "operations": [
    {
      "operation": "insert",
      "id": "create-obs-1",
      "data": {
        "observation": {
          "clientId": "my-observation-001",    // Client's clientId (VEREIST voor INSERT)
          "type": "measure",
          "resultTime": "2024-01-15T10:00:00Z",
          "phenomenonTime": {
            "begin": "2024-01-15T10:00:00Z",
            "end": "2024-01-15T10:00:00Z"
          },
          "parameter": {
            "parameter/waarnemingssoort": "WATHTE",
            "parameter/hoedanigheid": "gemiddeld over meetperiode"
          },
          "foi": {
            "id": "550e8400-e29b-41d4-a716-446655440002"  // Server ID (string)
          },
          "result": {
            "id": "my-result-001",
            "measure": {
              "uom": "m",
              "value": 1.5,
              "namespace": "Aquo"
            }
          },
          "modifiedOn": "2024-01-15T10:00:00Z"
        }
      }
    }
  ]
}
```

### Voorbeeld 3: POST /v4/write/observation (Update Observation - UPDATE)

```jsonc
{
  "operations": [
    {
      "operation": "update",
      "id": "update-obs-1",
      "data": {
        "observation": {
          "id": "550e8400-e29b-41d4-a716-446655440001",  // Server ID (string) - VEREIST
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
          // clientId MOET afwezig zijn
        }
      }
    }
  ]
}
```

### Voorbeeld 4: POST /v4/write/observation (Retract Observation)

```jsonc
{
  "operations": [
    {
      "operation": "retract",
      "id": "retract-obs-1",
      "data": {
        "id": "550e8400-e29b-41d4-a716-446655440001",  // Server ID (string) - VEREIST
        "reason": "Foutieve meting, gecorrigeerd door nieuwe observatie"
        // clientId MOET afwezig zijn
      }
    }
  ]
}
```

### Voorbeeld 5: POST /v4/write/parameter (Insert Parameter - CREATE)

```jsonc
{
  "operations": [
    {
      "operation": "insert",
      "id": "create-param-1",
      "data": {
        "parameter": {
          "clientId": "my-parameter-001",    // Client's clientId (VEREIST voor INSERT)
          "name": "Waterhoogte",
          "definition": "https://definities.pdok.nl/def/imkl/1.0/id/waarnemingssoort/WATHTE"
        }
      }
    }
  ]
}
```

### Voorbeeld 6: POST /v4/write/parameter (Update Parameter - UPDATE)

```jsonc
{
  "operations": [
    {
      "operation": "update",
      "id": "update-param-1",
      "data": {
        "parameter": {
          "id": "550e8400-e29b-41d4-a716-446655440004",  // Server ID (string) - VEREIST
          "name": "Waterhoogte (herzien)",
          "definition": "https://definities.pdok.nl/def/imkl/1.0/id/waarnemingssoort/WATHTE"
          // clientId MOET afwezig zijn
        }
      }
    }
  ]
}
```

### Voorbeeld 7: POST /v4/write/foi (Retract FoI)

```jsonc
{
  "operations": [
    {
      "operation": "retract",
      "id": "retract-foi-1",
      "data": {
        "id": "550e8400-e29b-41d4-a716-446655440002",  // Server ID (string) - VEREIST
        "reason": "Locatie niet meer in gebruik"
        // clientId MOET afwezig zijn
      }
    }
  ]
}
```

## Voorbeeld Responses

### Direct Response (202 Accepted)

```json
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "queued",
  "createdAt": "2024-01-15T10:01:00Z",
  "totalOperations": 2,
  "location": "/v4/write/observation/550e8400-e29b-41d4-a716-446655440000"
}
```

### Job Status Response (GET /v4/write/observation/{jobId})

Stel je voor dat de volgende request is verstuurd naar `/v4/write/observation`:
```jsonc
{
  "operations": [
    {
      "operation": "insert",
      "id": "create-obs-1",
      "data": {
        "observation": {
          "clientId": "my-observation-001",    // Client's clientId
          "type": "measure",
          "resultTime": "2024-01-15T10:00:00Z",
          "phenomenonTime": {
            "begin": "2024-01-15T10:00:00Z",
            "end": "2024-01-15T10:00:00Z"
          },
          "parameter": {"parameter/waarnemingssoort": "WATHTE"},
          "foi": {"id": "550e8400-e29b-41d4-a716-446655440002"},  // Server ID (string)
          "result": {
            "measure": {"uom": "m", "value": 2.3}
          },
          "modifiedOn": "2024-01-15T10:00:00Z"
        }
      }
    },
    {
      "operation": "retract",
      "id": "retract-obs-1",
      "data": {
        "id": "550e8400-e29b-41d4-a716-446655449999"  // Server ID (string)
      }
    }
  ]
}
```

**Tijdens verwerking** (GET /v4/write/observation/550e8400-e29b-41d4-a716-446655440000):

```json
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing",
  "createdAt": "2024-01-15T10:01:00Z",
  "startedAt": "2024-01-15T10:01:05Z",
  "totalOperations": 2
}
```

**Na voltooiing** (GET /v4/write/observation/550e8400-e29b-41d4-a716-446655440000):

```jsonc
{
  "jobId": "550e8400-e29b-41d4-a716-446655440000",
  "status": "completed",
  "createdAt": "2024-01-15T10:01:00Z",
  "startedAt": "2024-01-15T10:01:05Z",
  "completedAt": "2024-01-15T10:01:10Z",
  "totalOperations": 2,
  "successfulOperations": 2,
  "failedOperations": 0,
  "operations": [
    {
      "operationIndex": 0,
      "operationId": "create-obs-1",
      "status": "succeeded",
      "clientId": "my-observation-001",         // Client's clientId
      "id": "550e8400-e29b-41d4-a716-446655440001",  // Server gegenereerde ID (string)
      "messages": []
    },
    {
      "operationIndex": 1,
      "operationId": "retract-obs-1",
      "status": "succeeded",
      "clientId": null,
      "id": "550e8400-e29b-41d4-a716-446655449999",  // Server ID (string)
      "messages": []
    }
  ],
  "errors": []
}
```

### Voorbeeld met Fouten

Response van `GET /v4/write/observation/{jobId}`:

```jsonc
{
  "jobId": "550e8400-e29b-41d4-a716-446655440001",
  "status": "completed",
  "createdAt": "2024-01-15T10:02:00Z",
  "startedAt": "2024-01-15T10:02:05Z",
  "completedAt": "2024-01-15T10:02:10Z",
  "totalOperations": 2,
  "successfulOperations": 1,
  "failedOperations": 1,
  "operations": [
    {
      "operationIndex": 0,
      "operationId": "op1",
      "status": "succeeded",
      "clientId": "my-observation-001",
      "id": "550e8400-e29b-41d4-a716-446655440003",
      "messages": []
    },
    {
      "operationIndex": 1,
      "operationId": "op2",
      "status": "failed",
      "clientId": null,
      "id": null,
      "messages": [
        {
          "code": "INVALID_REQUEST",
          "message": "For UPDATE operation: id is required, clientId must be absent",
          "severity": "error",
          "path": "/operations/1/data/observation"
        }
      ]
    }
  ],
  "errors": []
}
```

## Samenvatting Key-Exchange in Batch Structuur

| Scenario            | Endpoint                | Request veld                                       | Response `clientId` | Response `id`      | Notities                                 |
|---------------------|-------------------------|----------------------------------------------------|---------------------|--------------------|------------------------------------------|
| Insert Observation  | POST /write/observation | `clientId` **VEREIST**, `id` MOET afwezig          | Client clientId     | Server ID (string) | Server genereert altijd zelf ID (string) |
| Update Observation  | POST /write/observation | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |
| Retract Observation | POST /write/observation | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |
| Insert FoI          | POST /write/foi         | `clientId` **VEREIST**, `id` MOET afwezig          | Client clientId     | Server ID (string) | Server genereert altijd zelf ID (string) |
| Update FoI          | POST /write/foi         | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |
| Retract FoI         | POST /write/foi         | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |
| Insert Parameter    | POST /write/parameter   | `clientId` **VEREIST**, `id` MOET afwezig          | Client clientId     | Server ID (string) | Server genereert altijd zelf ID (string) |
| Update Parameter    | POST /write/parameter   | `id` (server) **VEREIST**, `clientId` MOET afwezig | null                | Server ID (string) | MOET server ID (string) zijn             |

> **🔴 REGELS**:

> 1. **INSERT**: alleen `clientId` in request, `id` MOET afwezig zijn
> 2. **UPDATE/DEPRECATE**: alleen `id` (server) in request, `clientId` MOET afwezig zijn
> 3. Clients kunnen **elk ID-formaat** gebruiken voor `clientId`
> 4. **Server-ID's zijn altijd strings** - het formaat wordt door de server bepaald
> 5. Bij `INSERT` response: server retourneert zowel `clientId` als `id`
> 6. Bij `UPDATE/DEPRECATE` response: server retourneert alleen `id` (clientId is null)
> 7. FoI en Parameter referenties **MOETEN server-ID's (string)** zijn

## Job Status Endpoint

Voor het opvragen van status en resultaten, gebruik het **specifieke entiteit endpoint**:
- Observation jobs: `GET /v4/write/observation/{jobId}`
- FoI jobs: `GET /v4/write/foi/{jobId}`
- Parameter jobs: `GET /v4/write/parameter/{jobId}`

De response bevat **alleen basisinformatie** tijdens verwerking, en **status + gedetailleerde resultaten** wanneer de job voltooid is.

Zie [Endpoints](04-00-endpoints.md) voor complete endpoint documentatie.
