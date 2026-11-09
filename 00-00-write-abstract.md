# DD API V4 Write Module - Abstract

<aside class="advisement" title="Work-in-progress">
De specificatie is nog in ontwikkeling. Tot deze finaal is, zal de specificatie nog kunnen veranderen.
De principes staan echter vast.
</aside>

Digitale Delta API V4 Write is een uitbreiding op de DD API V4 specificatie en biedt de mogelijkheid tot het aanpassen van observaties, meetobjecten en parameters via **per-entity endpoints met consistente job tracking**.

## Kernprincipes

1. **Per-entity batch-verwerking**: DD API V4 Write gebruikt afzonderlijke endpoints voor elke entiteitstype met eigen namespace voor batch-operaties en job tracking.

2. **Consistente job tracking**: Elke entiteit heeft zijn eigen job status endpoint onder dezelfde path:
   ```
   /v4/write/observation/    ← Observation namespace
   ├── POST                 ← Batch submit
   └── GET {jobId}          ← Job status + resultaten (als klaar)
   ```

3. **Asynchrone operaties**: Batch operaties zijn asynchroon. Validatie en verwerking kunnen tijd kosten, temeer doordat validatie in context van bestaande data zal gebeuren.

4. **Eigen wrapper**: Het datamodel is gelijk aan het datamodel van DD API V4, maar in plaats van een OData-wrapper gebruikt elk endpoint een eigen, specifieke wrapper voor batch-operaties.

5. **OAS 3.1 Specificatie**: De Write module blijft een OpenAPI Specification (OAS) 3.1, geen AsyncAPI. De asynchrone aard wordt geïmplementeerd via per-entity status-tracking endpoints.

6. **Key Exchange Mechanisme**: 
   - Bij **CREATE**: Client stuurt data met eigen ID (`clientId`)
     - De server genereert **altijd zelf** een identifier (als string)
     - Response bevat zowel `clientId` als `id` (beide strings)
   - Bij **UPDATE/DEPRECATE**: Client MOET de `id` (string), dus de id die de server terug heeft gegeven, gebruiken
     - Gebruik van `clientId` in update/deprecate resulteert in fout `CLIENT_ID_IN_UPDATE`
   - **Reden**: Voorkomt key-conflicten tussen verschillende aanbieders en staat clients toe om elk gewenst id-formaat te gebruiken

7. **Variable response**: Het `GET /{entity}/{jobId}` endpoint retourneert **alleen basisinformatie** tijdens verwerking (status = `queued` of `processing`), en **basis + gedetailleerde resultaten** wanneer de job voltooid is (status = `completed` of `failed`).

## Doel

De Write module maakt het mogelijk om:
- **Insert en update**: Observaties, meetobjecten (FoI) en parameters toe te voegen of bij te werken via entiteit-specifieke endpoints
- **Deprecaten/Vervallen**: Observaties en meetobjecten als vervallen te markeren
- **Batch verwerking**: Meerdere operaties voor één entiteitstype in één request te versturen
- **Asynchrone afhandeling**: Grote batches asynchroon te verwerken met per-entity status-tracking

## Relatie met DD API V4

De Write module:

- Gebruikt dezelfde datamodellen als DD API V4 (Observation, FoI, Parameter)
- Vervangt de OData-wrapper door een batch-specifieke wrapper voor **elke entiteit**
- Voegt operatie-types toe (insert, update, deprecate)
- Voegt asynchrone verwerkingsstatus toe via per-entity endpoints
- Blijft conform OAS 3.1 standaard
- Voegt key-exchange mechanisme toe met **server-controlled string identifiers**
- Gebruikt per-entity endpoints voor betere scheiding van verantwoordelijkheden en consistente job tracking

## Doelgroep

Deze specificatie is bedoeld voor:

- Data-aanbieders die observatiedata willen bijwerken
- Systeembeheerders die batch-operaties willen implementeren
- Ontwikkelaars die toepassingen bouwen die data naar DD API V4 moeten schrijven
