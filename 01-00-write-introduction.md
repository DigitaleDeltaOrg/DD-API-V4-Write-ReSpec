# DD API V4 Write Module - Inleiding

## Overzicht

DD API V4 Write voegt schrijf functionaliteit toe aan de DD API V4 specificatie. Waar DD API V4 primair een read-only API is voor het opvragen van watergerelateerde observaties, biedt de Write module de mogelijkheid om data bij te werken.

## Waarom een Write Module?

In de praktijk blijkt dat observatiedata niet statisch is. Er kunnen correcties nodig zijn, nieuwe metingen toegevoegd moeten worden, of meetobjecten gewijzigd moeten worden. De Write module voorziet in deze behoeften op een gestructureerde, veilige en efficiënte manier.

## Belangrijke Ontwerpkeuzes

### Per-Entity Endpoints met Consistente Job Tracking

DD API V4 Write gebruikt **per-entity endpoints** met een consistente namespace structuur:

```
/v4/write/observation/      ← Alle observation operaties
├── POST                    ← Batch insert/update/deprecate
└── GET {jobId}             ← Job status + resultaten (als klaar)

/v4/write/foi/              ← Alle FoI operaties
├── POST                    ← Batch insert/update/deprecate
└── GET {jobId}             ← Job status + resultaten (als klaar)

/v4/write/parameter/        ← Alle parameter operaties
├── POST                    ← Batch insert/update
└── GET {jobId}             ← Job status + resultaten (als klaar)
```

Dit ontwerp heeft verschillende voordelen:

- **Consistentie**: Alle operaties voor een entiteit zitten onder één path
- **Duidelijkheid**: Direct zichtbaar welke job bij welke entiteit hoort
- **RESTful**: Volgt het principe van resource nesting
- **Eenvoudige validatie**: Server weet direct welk type job het is
- **Betere organisatie**: Scheiding van verantwoordelijkheden per entiteit
- **Efficiëntie**: Minder requests nodig (geen apart /results endpoint)
- **Flexibiliteit**: Clients kunnen elk ID-formaat gebruiken voor hun eigen IDs

### Asynchrone Verwerking

Batch-operaties zijn asynchroon. Dit betekent:

1. Een batch-request wordt direct geaccepteerd en retourneert een job-ID
2. De `Location` header wijst naar het job status endpoint (bijv. `/v4/write/observation/{jobId}`)
3. De daadwerkelijke verwerking gebeurt op de achtergrond
4. De client kan de status opvragen via het endpoint in de `Location` header
5. **Wanneer de job voltooid is**, retourneert hetzelfde endpoint de gedetailleerde resultaten

De keuze voor asynchrone verwerking is gemaakt omdat:

- Validatie complex kan zijn en tijd kost
- Contextuele validatie (bv. tegen bestaande data) nodig is
- Grote batches lang kunnen duren
- De client niet hoeft te wachten op afronding

### Eigen Wrapper in plaats van OData

Hoewel de datamodellen gelijk blijven aan DD API V4, wordt de OData-wrapper vervangen door een specifieke batch-wrapper. Dit om:

- OData is primair bedoeld voor query-operaties, niet voor batch-mutaties
- De batch-structuur vereist metadata over operaties (type, sequentie, etc.)
- OData's `@odata.bind` en andere mechanismen zijn niet geschikt voor batch-operaties
- Een eenvoudigere, specifiekere structuur beter past bij batch-verwerking

## Scope

De Write module ondersteunt:

- ✅ **Insert** van Observations via `POST /v4/write/observation`
- ✅ **Update** van Observations via `POST /v4/write/observation`
- ✅ **Insert** van Features of Interest via `POST /v4/write/foi`
- ✅ **Update** van Features of Interest via `POST /v4/write/foi`
- ✅ **Insert** van Parameters via `POST /v4/write/parameter`
- ✅ **Update** van Parameters via `POST /v4/write/parameter`
- ✅ **Deprecate/Retract** van Observations via `POST /v4/write/observation`
- ✅ **Deprecate/Retract** van Features of Interest via `POST /v4/write/foi`
- ✅ **Job status tracking** via per-entity endpoints (GET /write/{entity}/{jobId})
- ✅ **Variable response**: GET /write/{entity}/{jobId} retourneert basisinformatie tijdens verwerking, en basis + resultaten wanneer klaar
- ✅ **Server-controlled identifiers**: Server genereert altijd zelf identifiers (als string)
- ❌ Geen delete operaties (alleen deprecate)
- ❌ Geen individuele operaties (alleen batch)
- ❌ Parameters kunnen niet worden gedeprecateerd

## Relatie met Andere Modules

De Write module is een optionele uitbreiding op DD API V4. Een implementatie die alleen de read-functionaliteit biedt, is nog steeds DD API V4 conform. Een implementatie die ook write-functionaliteit biedt, moet voldoen aan de Write module specificatie.

De Write module deelt de volgende elementen met de core DD API V4:
- Datamodellen (Observation, FoI, Parameter)
- Semantische property bags
- Vocabularia (Aquo, etc.)
- Tijdsformaten (ISO8601)

## Compliance

Een DD API V4 Write implementatie:

- MOET de per-entity batch-operatie endpoints implementeren:
  - `POST /v4/write/observation`
  - `POST /v4/write/foi`
  - `POST /v4/write/parameter`
- MOET de per-entity job status endpoints implementeren:
  - `GET /v4/write/observation/{jobId}`
  - `GET /v4/write/foi/{jobId}`
  - `GET /v4/write/parameter/{jobId}`
- MOET **altijd zelf identifiers (als string) genereren** voor server-side keys (client mag NOOIT server ID dicteren)
- MOET voor het `GET /{entity}/{jobId}` endpoint **alleen basisinformatie** retourneren wanneer status = `queued` of `processing` is, en **basis + gedetailleerde resultaten** wanneer status = `completed` of `failed` is
- MOET de datamodellen conform DD API V4 gebruiken
- MOET asynchrone verwerking ondersteunen
- MOET validatie uitvoeren
- MOET de `Location` header retourneren in 202 responses
- MOET `callbackUrl` ondersteunen en een mislukte callback als fout in de job behandelen
- MOET een maximum batch grootte specificeren en deze communiceren via `X-Max-Batch-Size` header in 413 responses
- MAG additionele endpoints implementeren (bv. voor validatie van batch-requests)


