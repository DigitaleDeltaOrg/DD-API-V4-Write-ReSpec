# DD API V4 Write Module

De **DD API V4 Write Module** is een uitbreiding op de DD API V4 specificatie die batch-verwerking mogelijk maakt voor het **inserten, updaten en deprecaten** van data. Deze module biedt asynchrone batch-verwerking met een specifieke wrapper die de OData-wrapper vervangt, terwijl de datamodellen van DD API V4 behouden blijven.

## Kernconcepten

### 1. Per-Entity Endpoints met Job Tracking

- **Afzonderlijke endpoints per entiteit**: Elke entiteitstype heeft zijn eigen namespace
- **Consistente job tracking**: Job status endpoints zitten onder dezelfde path als de batch submit
- **Meerdere operaties in één request**: Clients kunnen meerdere insert, update en deprecate operaties voor één entiteit in één batch versturen
- **Asynchrone uitvoering**: Jobs worden op de achtergrond verwerkt

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

> **Let op**: Er is **geen apart /results endpoint**. Het `GET /{entity}/{jobId}` endpoint retourneert **alleen basisinformatie** tijdens verwerking, en **status + gedetailleerde resultaten** wanneer de job voltooid is.

### 2. Key Exchange Mechanisme

- **Create**: Client stuurt data met eigen ID (client ID)
- **🔴 CRITISCH**: **De server genereert altijd zelf een identifier (als string)** - de client mag NOOIT de server ID dicteren
- **Reden**: Voorkomt key-conflicten tussen verschillende aanbieders en staat clients toe om elk gewenst ID-formaat te gebruiken
- **Response**: Server retourneert zowel client ID als **server ID (string)**
- **Update/Deprecate**: Client MOET de server ID (string) gebruiken
- **Fout**: Gebruik van client ID in update/deprecate resulteert in `CLIENT_ID_IN_UPDATE`

Zie [Key Exchange Mechanisme](06-00-key-exchange.md) voor details.

### 3. Asynchrone Verwerking

- **Direct response**: 202 Accepted met job ID en location header
- **Location header**: wijst naar het specifieke job status endpoint (bijv. `/v4/write/observation/{jobId}`)
- **Status opvragen**: Via het endpoint in de location header
- **Resultaten**: Wanneer de job voltooid is, retourneert hetzelfde endpoint de gedetailleerde resultaten

### 4. Operatie Types

- **insert**: Toevoegen van observaties, FoIs en parameters
- **update**: Bijwerken van bestaande observaties, FoIs en parameters
- **deprecate**: Deprecaten (vervallen laten gaan) van observaties en FoIs (parameters kunnen niet worden gedeprecateerd)

## Documentatie

| Document | Beschrijving |
|----------|--------------|
| [Abstract](00-00-write-abstract.md) | Overzicht en kernprincipes |
| [Inleiding](01-00-write-introduction.md) | Achtergrond en scope |
| [Batch Structuur](02-00-batch-structure.md) | Request en response structuur |
| [Validatie](03-00-validation.md) | Validatie regels en foutcodes |
| [Endpoints](04-00-endpoints.md) | API endpoints en voorbeelden |
| [Key Exchange](06-00-key-exchange.md) | Key exchange mechanisme |

## OpenAPI Specificatie

De complete OpenAPI 3.1 specificatie is beschikbaar in:
- [`definitions/v4.0/oas3.1-write.yaml`](../../definitions/v4.0/oas3.1-write.yaml)

## Conformiteit

Een implementatie die voldoet aan de DD API V4 Write Module:

- ✅ MOET de per-entity batch operatie endpoints implementeren:
    - `POST /v4/write/observation`
    - `POST /v4/write/foi`
    - `POST /v4/write/parameter`
- ✅ MOET de per-entity job status endpoints implementeren:
    - `GET /v4/write/observation/{jobId}`
    - `GET /v4/write/foi/{jobId}`
    - `GET /v4/write/parameter/{jobId}`
- ✅ MOET **altijd zelf identifiers (als string) genereren** voor server-side keys (client mag NOOIT server ID dicteren)
- ✅ MOET het key-exchange mechanisme implementeren
- ✅ MOET asynchrone verwerking ondersteunen
- ✅ MOET validatie uitvoeren volgens de gespecificeerde regels
- ✅ MOET de gespecificeerde datamodellen gebruiken
- ✅ MOET de `Location` header retourneren in 202 responses met de job status URL
- ✅ MOET voor het `GET /{entity}/{jobId}` endpoint **alleen basisinformatie** retourneren wanneer status = `queued` of `processing` is, en **basis + gedetailleerde resultaten** wanneer status = `completed` of `failed` is
- ✅ MOET `callbackUrl` ondersteunen en een mislukte callback als fout in de job behandelen
- ✅ MOET een maximum batch grootte specificeren en deze communiceren via `X-Max-Batch-Size` header in 413 responses
- ❌ MAG additionele endpoints implementeren
- ❌ MAG additionele operatie-types ondersteunen

## Relatie met DD API V4

De Write Module:
- Gebruikt dezelfde datamodellen als DD API V4 (Observation, FoI, Parameter)
- Vervangt de OData-wrapper door een batch-specifieke wrapper voor **elke entiteit**
- Voegt operatie-types toe (insert, update, deprecate)
- Voegt asynchrone verwerkingsstatus toe
- Blijft conform OAS 3.1 standaard (geen AsyncAPI)
- **Voegt server-controlled string identifiers toe** - de server genereert altijd zelf de identifiers

## Voorbeelden

Zie [Batch Structuur](02-00-batch-structure.md) en [Endpoints](04-00-endpoints.md) voor voorbeeld requests en responses.

## Endpoint Overzicht

| Endpoint | Methode | Beschrijving | Entiteit |
|----------|---------|--------------|----------|
| `/v4/write/observation` | POST | Batch insert/update/deprecate van observaties | Observation |
| `/v4/write/observation/{jobId}` | GET | Opvragen status van een observation job | Observation |
| `/v4/write/foi` | POST | Batch insert/update/deprecate van meetobjecten | FoI |
| `/v4/write/foi/{jobId}` | GET | Opvragen status van een FoI job | FoI |
| `/v4/write/parameter` | POST | Batch insert/update van parameters | Parameter |
| `/v4/write/parameter/{jobId}` | GET | Opvragen status van een parameter job | Parameter |
