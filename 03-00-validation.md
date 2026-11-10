# DD API V4 Write - Validatie

## Overzicht

Validatie is een cruciaal onderdeel van de Write module. Alle batch-requests moeten worden gevalideerd voordat ze worden verwerkt. Validatie vindt plaats op verschillende niveaus:

1. **Structuurvalidatie**: Controle of het request voldoet aan het OpenAPI schema
2. **Semantische validatie**: Controle of de data voldoet aan de semantische regels
3. **Contextuele validatie**: Controle of de data consistent is met bestaande data
4. **Key-exchange validatie**: Controle van clientId en id velden

**Belangrijk**: De Write module gebruikt **per-entity endpoints**, dus validatie is specifiek voor elke entiteit. De voorbeelden in dit document gebruiken de nieuwe structuur met **INSERT**, **UPDATE**, en **DEPRECATE** operaties.

## Validatie Niveaus

### 1. Structuurvalidatie

De eerste validatie controleert of het JSON-request voldoet aan het OpenAPI schema.

**Foutcodes**:
- `INVALID_SCHEMA`: Algemene schema validatie fout
- `MISSING_REQUIRED_FIELD`: Verplicht veld ontbreekt
- `INVALID_TYPE`: Ongeldig datatype
- `INVALID_FORMAT`: Ongeldig formaat (bv. datum, UUID)
- `INVALID_ENUM_VALUE`: Ongeldige enum waarde
- `INVALID_OPERATION`: Ongeldige operatie (moet insert, update, of retract zijn)

### 2. Semantische Validatie

Controleert of de data voldoet aan de semantische regels van DD API V4.

#### Observation Validatie

| Regel                                                                          | Foutcode                 |
|--------------------------------------------------------------------------------|--------------------------|
| `type` moet een geldige waarde zijn: count, measure, truth, category, coverage | INVALID_OBSERVATION_TYPE |
| `resultTime` moet een geldige ISO8601 date-time zijn                           | INVALID_RESULT_TIME      |
| `phenomenonTime` moet een geldige periode zijn                                 | INVALID_PHENOMENON_TIME  |
| `phenomenonTime.begin` <= `resultTime`                                         | INVALID_TIME_RELATION    |
| `parameter` moet aanwezig zijn                                                 | MISSING_PARAMETER        |
| `modifiedOn` moet een geldige ISO8601 date-time zijn                           | INVALID_MODIFIED_ON      |
| `foi` of `geometry` moet aanwezig zijn                                         | MISSING_GEOMETRY         |
| `result` moet aanwezig zijn                                                    | MISSING_RESULT           |
| `result` type moet matchen met `observation.type`                              | RESULT_TYPE_MISMATCH     |

#### Result Validatie

| Regel                     | Type                          | Foutcode                    |
|---------------------------|-------------------------------|-----------------------------|
| `measure` verplicht       | observation.type = "measure"  | MISSING_MEASURE_PROPERTIES  |
| `count` moet integer zijn | observation.type = "count"    | INVALID_COUNT_VALUE         |
| `category` verplicht      | observation.type = "category" | MISSING_CATEGORY_PROPERTIES |
| `truth` moet boolean zijn | observation.type = "truth"    | INVALID_TRUTH_VALUE         |

#### FoI Validatie

| Regel                                             | Foutcode                |
|---------------------------------------------------|-------------------------|
| `type` verplicht                                  | MISSING_FOI_TYPE        |
| `name` verplicht                                  | MISSING_FOI_NAME        |
| `code` verplicht                                  | MISSING_FOI_CODE        |
| `description` verplicht                           | MISSING_FOI_DESCRIPTION |
| `location` verplicht                              | MISSING_FOI_LOCATION    |
| `geometry` verplicht                              | MISSING_FOI_GEOMETRY    |
| `geometry.type` moet een geldig GeoJSON type zijn | INVALID_GEOMETRY_TYPE   |
| `coordinates` moeten geldige coördinaten bevatten | INVALID_COORDINATES     |

#### Parameter Validatie

| Regel            | Foutcode               |
|------------------|------------------------|
| `name` verplicht | MISSING_PARAMETER_NAME |
| `definition` verplicht | MISSING_PARAMETER_DEFINITION |

### 3. Contextuele Validatie

| Regel                                                                    | Foutcode                    |
|--------------------------------------------------------------------------|-----------------------------|
| Entiteit met dezelfde clientId mag niet al bestaan (voor INSERT)         | DUPLICATE_SOURCEID          |
| FoI referentie moet bestaan of in dezelfde batch worden toegevoegd       | INVALID_FOI_REFERENCE       |
| Parameter referentie moet bestaan of in dezelfde batch worden toegevoegd | INVALID_PARAMETER_REFERENCE |
| `modifiedOn` moet na de vorige `modifiedOn` zijn                         | INVALID_TIME_SEQUENCE       |
| Entiteit die gedeprecateerd moet worden, moet bestaan                    | ENTITY_NOT_FOUND            |

### 4. Key-Exchange Validatie (Vereenvoudigd Model)

**INSERT operatie**:
- `clientId` **VEREIST**
- `id` MOET **afwezig of null** zijn

**UPDATE/DEPRECATE operatie**:
- `id` (server) **VEREIST**
- `clientId` MOET **afwezig of null** zijn

**Foutcodes**:
- `SOURCEID_REQUIRED`: clientId is vereist voor INSERT
- `ID_NOT_ALLOWED`: id mag niet aanwezig zijn voor INSERT
- `SOURCEID_NOT_ALLOWED`: clientId mag niet aanwezig zijn voor UPDATE/DEPRECATE
- `ID_REQUIRED`: id is vereist voor UPDATE/DEPRECATE
- `INVALID_SERVER_ID`: Ongeldige server ID
- `ENTITY_NOT_FOUND`: Entiteit niet gevonden
- `SOURCEID_IN_UPDATE`: clientId aanwezig in UPDATE/DEPRECATE

## Validatie in de Batch Context

### Validatie Modus

De `validateOnly` optie in BatchOptions bepaalt of alleen validatie wordt uitgevoerd:

- **validateOnly: true**: Alleen validatie, geen daadwerkelijke wijzigingen
- **validateOnly: false** (standaard): Validatie en uitvoering

### Continue On Error

De `continueOnError` optie bepaalt het gedrag bij validatie fouten:

- **continueOnError: true** (standaard): Validatie gaat door, alle fouten worden gerapporteerd
- **continueOnError: false**: Validatie stopt bij de eerste fout

### Validatie Fouten in Response

Validatie fouten worden geretourneerd in:

1. **Direct response**: Als validatie faalt voordat de job wordt aangemaakt
2. **Operation messages**: Validatie fouten voor individuele operaties
3. **Batch errors**: Algemene validatie fouten

## Voorbeelden van Validatie Fouten

### Voorbeeld 1: clientId vereist voor INSERT (POST /v4/write/observation)

**Request**:
```json
{
  "operations": [{
    "operation": "insert",
    "id": "op1",
    "data": {
      "observation": {
        // clientId ontbreekt!
        "type": "measure",
        "resultTime": "2024-01-15T10:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T10:00:00Z",
          "end": "2024-01-15T10:00:00Z"
        },
        "parameter": {"parameter/waarnemingssoort": "WATHTE"},
        "foi": {"id": "550e8400-e29b-41d4-a716-446655440002"},
        "result": {
          "id": "result-001",
          "measure": {"uom": "m", "value": 1.5}
        },
        "modifiedOn": "2024-01-15T10:00:00Z"
      }
    }
  }]
}
```

**Fout**:
```json
{
  "code": "SOURCEID_REQUIRED",
  "message": "clientId is required for INSERT operation",
  "path": "/operations/0/data/observation"
}
```

### Voorbeeld 2: id niet toegestaan bij INSERT (POST /v4/write/observation)

**Request**:
```json
{
  "operations": [{
    "operation": "insert",
    "id": "op1",
    "data": {
      "observation": {
        "clientId": "my-observation-001",
        "id": "should-not-be-here",  // ← id mag niet aanwezig zijn bij INSERT!
        "type": "measure",
        "resultTime": "2024-01-15T10:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T10:00:00Z",
          "end": "2024-01-15T10:00:00Z"
        },
        "parameter": {"parameter/waarnemingssoort": "WATHTE"},
        "foi": {"id": "550e8400-e29b-41d4-a716-446655440002"},
        "result": {
          "id": "result-001",
          "measure": {"uom": "m", "value": 1.5}
        },
        "modifiedOn": "2024-01-15T10:00:00Z"
      }
    }
  }]
}
```

**Fout**:
```json
{
  "code": "ID_NOT_ALLOWED",
  "message": "id must be absent or null for INSERT operation",
  "path": "/operations/0/data/observation/id"
}
```

### Voorbeeld 3: clientId in UPDATE (POST /v4/write/observation)

**Request**:
```json
{
  "operations": [{
    "operation": "update",
    "id": "op1",
    "data": {
      "observation": {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "clientId": "my-observation-001",  // ← clientId mag niet aanwezig zijn bij UPDATE!
        "type": "measure",
        "resultTime": "2024-01-15T11:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T11:00:00Z",
          "end": "2024-01-15T11:00:00Z"
        },
        "parameter": {"parameter/waarnemingssoort": "WATHTE"},
        "foi": {"id": "550e8400-e29b-41d4-a716-446655440002"},
        "result": {
          "id": "result-001",
          "measure": {"uom": "m", "value": 1.7}
        },
        "modifiedOn": "2024-01-15T11:00:00Z"
      }
    }
  }]
}
```

**Fout**:
```json
{
  "code": "SOURCEID_NOT_ALLOWED",
  "message": "clientId must be absent for UPDATE operation",
  "path": "/operations/0/data/observation/clientId"
}
```

### Voorbeeld 4: id vereist voor UPDATE (POST /v4/write/observation)

**Request**:
```json
{
  "operations": [{
    "operation": "update",
    "id": "op1",
    "data": {
      "observation": {
        // id ontbreekt!
        "clientId": "my-observation-001",
        "type": "measure",
        "resultTime": "2024-01-15T11:00:00Z",
        "phenomenonTime": {
          "begin": "2024-01-15T11:00:00Z",
          "end": "2024-01-15T11:00:00Z"
        },
        "parameter": {"parameter/waarnemingssoort": "WATHTE"},
        "foi": {"id": "550e8400-e29b-41d4-a716-446655440002"},
        "result": {
          "id": "result-001",
          "measure": {"uom": "m", "value": 1.7}
        },
        "modifiedOn": "2024-01-15T11:00:00Z"
      }
    }
  }]
}
```

**Fout**:
```json
{
  "code": "ID_REQUIRED",
  "message": "id is required for UPDATE operation",
  "path": "/operations/0/data/observation"
}
```

### Voorbeeld 5: Type Mismatch (POST /v4/write/observation)

**Request**:
```json
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
          "id": "550e8400-e29b-41d4-a716-446655440002",
          "code": "LOC001",
          "description": "Locatie 1",
          "geometry": {"type": "Point", "coordinates": [5.387, 52.156]}
        },
        "result": {
          "id": "result-001",
          "category": {
            "nameSpace": "Aquo",
            "code": "HOOG"
          }
        },
        "modifiedOn": "2024-01-15T10:00:00Z"
      }
    }
  }]
}
```

**Fout**:
```json
{
  "code": "RESULT_TYPE_MISMATCH",
  "message": "Result type does not match observation type",
  "details": [
    {
      "field": "/operations/0/data/observation/result",
      "message": "Observation type 'measure' requires result.type 'measure', but got 'category'"
    }
  ]
}
```

### Voorbeeld 6: Ongeldige Geometrie (POST /v4/write/foi)

**Request**:
```json
{
  "operations": [{
    "operation": "insert",
    "id": "op1",
    "data": {
      "foi": {
        "clientId": "my-location-001",
        "type": "http://definities.pdok.nl/def/cbsgb/1.0/id/datasetdefinitie/Top10NL",
        "name": "Meetlocatie",
        "organisation": "RWS",
        "code": "LOC001",
        "description": "Locatie 1",
        "location": "https://pdok.nl",
        "geometry": {
          "type": "Point",
          "coordinates": [999, 999]
        }
      }
    }
  }]
}
```

**Fout**:
```json
{
  "code": "INVALID_COORDINATES",
  "message": "Invalid coordinates",
  "details": [
    {
      "field": "/operations/0/data/foi/geometry/coordinates",
      "message": "Coordinates [999, 999] are outside valid range for ETRS89"
    }
  ]
}
```

### Voorbeeld 7: Retract met clientId (POST /v4/write/observation)

**Request**:
```json
{
  "operations": [{
    "operation": "retract",
    "id": "op1",
    "data": {
      "clientId": "my-observation-001",  // ← clientId mag niet aanwezig zijn bij DEPRECATE!
      "id": "550e8400-e29b-41d4-a716-446655440001",
      "reason": "Foutieve meting"
    }
  }]
}
```

**Fout**:
```json
{
  "code": "SOURCEID_NOT_ALLOWED",
  "message": "clientId must be absent for DEPRECATE operation",
  "path": "/operations/0/data/clientId",
  "hint": "Only id (server ID) is required for DEPRECATE"
}
```

### Voorbeeld 8: Retract zonder id (POST /v4/write/observation)

**Request**:
```json
{
  "operations": [{
    "operation": "retract",
    "id": "op1",
    "data": {
      "reason": "Foutieve meting"
      // id ontbreekt!
    }
  }]
}
```

**Fout**:
```json
{
  "code": "ID_REQUIRED",
  "message": "id is required for DEPRECATE operation",
  "path": "/operations/0/data"
}
```

## Validatie van Retract Operaties

Voor retract operaties worden de volgende validaties uitgevoerd:

1. **Entiteit bestaat**: De entiteit met de opgegeven server ID moet bestaan
2. **Server ID**: Het `id` veld moet een geldige server ID (string) zijn
3. **Reden verplicht**: De `reason` moet aanwezig zijn en niet leeg
4. **Effective date**: Als opgegeven, moet dit een geldige toekomstige datum zijn
5. **clientId afwezig**: `clientId` MOET afwezig of null zijn

**Foutcodes**:
- `ENTITY_NOT_FOUND`: Entiteit met opgegeven server ID niet gevonden
- `INVALID_SERVER_ID`: Ongeldige server ID formaat
- `MISSING_REASON`: Reden voor retract is verplicht
- `INVALID_EFFECTIVE_DATE`: Ongeldige effectiveDate
- `SOURCEID_NOT_ALLOWED`: clientId mag niet aanwezig zijn
