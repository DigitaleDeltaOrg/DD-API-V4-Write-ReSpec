# SBOS: DD API V4 Write Module

**Software Bill of Standards** voor de DD API V4 Write Module specificatie.

---<a name="top"></a>

## 📋 Overzicht

| Eigenschap      | Waarde                                                          |
|-----------------|-----------------------------------------------------------------|
| **Naam**        | DD API V4 Write Module                                          |
| **Versie**      | 4.0.0                                                           |
| **Type**        | API Specificatie                                                |
| **SBOS Versie** | 1.0                                                             |
| **Doel**        | Documentatie van alle standaarden gebruikt in deze specificatie |

---

## 🏗️ Standaarden Hiërarchie

```
DD API V4 Write Module (Deze specificatie)
├── DD API V4 Core (Basis specificatie)
│   ├── Datamodellen (Observation, FoI, Parameter)
│   ├── Semantische property bags
│   └── Vocabularia
├── OpenAPI 3.1 (Specificatie formaat)
├── HTTP/1.1 (Transport protocol)
│   └── HTTPS (Beveiliging)
├── JSON (Data formaat)
│   └── GeoJSON (Geometrie)
└── ISO 8601 (Datum/tijd)
```

---

## 📚 Technische Standaarden

### 1. Specificatie Formaten

#### **OpenAPI Specification 3.1**
- **Versie**: 3.1.0
- **Rol**: Specificatie formaat voor API definities
- **URL**: [https://spec.openapis.org/oas/v3.1.0](https://spec.openapis.org/oas/v3.1.0)
- **Gebruik**: Complete API definitie in `oas3.1-write.yaml`
- **Conformiteit**: Volledig conform OAS 3.1
- **Uitbreidingen**: Geen (geen AsyncAPI, wel asynchrone verwerking via polling)

#### **JSON Schema**
- **Standaard**: RFC 8259 (JSON) + JSON Schema Draft
- **Rol**: Validatie van request/response bodies
- **Gebruik**: Schema validatie in batch structuur

---

### 2. Data Formaten

#### **JSON (JavaScript Object Notation)**
- **Standaard**: RFC 8259
- **Rol**: Primair data formaat
- **Gebruik**: Alle request bodies, response bodies
- **Versie**: ECMAScript 262 (5e editie)
- **Encoding**: UTF-8

#### **GeoJSON**
- **Standaard**: RFC 7946
- **Rol**: Geometrie representatie
- **Gebruik**: FoI (Feature of Interest) geometrie velden
- **Types**: Point, LineString, Polygon, MultiPoint, MultiLineString, MultiPolygon, GeometryCollection
- **Coördinaten**: WGS 84 (EPSG:4326) of ETRS89 (EPSG:4258)

#### **ISO 8601 (Datum en Tijd)**
- **Standaard**: ISO 8601:2004
- **Rol**: Datum en tijd representatie
- **Gebruik**: `resultTime`, `phenomenonTime`, `modifiedOn`, `createdAt`, etc.
- **Formaat**: `yyyy-MM-dd'T'HH:mm:ss.SSSX` (met UTC timezone indicator `Z`)
- **Pourpose**: Tijdsstempels in alle entiteiten

---

### 3. Protocol Standaarden

#### **HTTP/1.1**
- **Standaard**: RFC 9112 (voorheen RFC 7230-7237)
- **Rol**: Transport protocol
- **Gebruik**: Alle API requests en responses
- **Methoden**: POST, GET

#### **HTTPS (HTTP Secure)**
- **Standaard**: RFC 2818
- **Rol**: Beveiligde communicatie
- **Gebruik**: Verplicht voor alle productie implementaties
- **TLS Versie**: Minimaal TLS 1.2 (aanbevolen: TLS 1.3)

---

## 🧩 DD API V4 Specifieke Standaarden

### 1. **DD API V4 Core**
- **Versie**: 4.0
- **Rol**: Basis specificatie waar DD API V4 Write op bouwt
- **URL**: [DD API V4 ReSpec](https://digitaledeltaorg.github.io/DD-API-V4-ReSpec/)
- **Gedeelde Componenten**:
  - Datamodellen: Observation, Feature of Interest (FoI), Parameter
  - Semantische property bags
  - Vocabularia (Aquo)
  - Tijdsformaten (ISO 8601)
- **Relatie**: Write Module is een **uitbreiding** op DD API V4 Core

### 2. **Key Exchange Mechanisme**
- **Standaard**: DD API V4 Write specifiek
- **Rol**: Identifier beheer
- **Principe**: Server-controlled identifiers (altijd string)
- **Regels**:
  - INSERT: Client stuurt `sourceId`, server retourneert `sourceId` + server `id`
  - UPDATE/DEPRECATE: Client moet server `id` (string) gebruiken
  - Client mag **NOOIT** server ID dicteren

---

## 🌐 Semantische Standaarden & Vocabularia

### 1. **Aquo (Aquo Vocabulaire)**
- **Namespace**: `http://definities.pdok.nl/def/imkl/1.0/id/waarnemingssoort/`
- **Rol**: Waarnemingssoort definities
- **Gebruik**: Parameter en Metadata velden
- **Voorbeelden**: WATHTE (waterhoogte), T (temperatuur)
- **Onderhoud**: Informatiehuis Water

### 3. **OM&S (Observations, Measurements & Samples)**
- **Rol**: Basis voor observaties
- **Gebruik**: ID formaat (string) voor alle entiteiten
- **Principe**: Server genereert altijd zelf identifiers (als string)
---

## 🔒 Beveiligingsstandaarden

### 1. **Authenticatie Mechanismen**

#### **Bearer Token (OAuth2)**
- **Standaard**: RFC 6750
- **Header**: `Authorization: Bearer <token>`
- **Rol**: Token-based authenticatie
- **Flow**: Client Credentials Flow (RFC 6749)

#### **mTLS (Mutual TLS)**
- **Standaard**: RFC 5246 (TLS 1.2), RFC 8446 (TLS 1.3)
- **Rol**: Certificate-based authenticatie
- **Gebruik**: Client en server authenticeren elkaar met certificaten

### 2. **Autorisatie**
- **Model**: Server-specifiek (niet gespecificeerd in standaard)
- **Principe**: Client bekend zijn met de entiteit voor UPDATE/DEPRECATE
- **Eigenaar**: Bepaald toegestane actie via authenticatie 

---

## 📊 Batch Verwerkingsstandaarden

### 1. **Job Tracking**
- **Endpoint Structuur**: `/v4/write/{entity}/{jobId}`
- **Status Waarden**: `queued`, `processing`, `completed`, `failed`
- **Variable Response**: Basisinfo tijdens verwerking, compleet met resultaten na voltooiing

### 2. **Asynchrone Verwerking**
- **Pattern**: Polling via status endpoints
- **Aanbevolen Strategie**: Exponentiële backoff (500ms, 1s, 2s, 5s, 10s)
- **Timeout**: Server-specifiek (implementatie keuze)

### 3. **Callback Notificaties**
- **Mechanisme**: HTTP POST naar `callbackUrl`
- **Formaat**: dezelfde response als `GET /{entity}/{jobId}`
- **Vereist**: MOET ondersteund worden
- **Foutafhandeling**: Misukte callback = job fout

---

## 🎯 DD API V4 Write Specifieke Concepten

### 1. **Operatie Types**
- **insert**: Nieuwe entiteit toevoegen (CREATE)
- **update**: Bestaande entiteit bijwerken (UPDATE)
- **deprecate**: Entiteit als vervallen markeren (DEPRECATE)
- **Geen delete**: Alleen deprecate is toegestaan

### 2. **Per-Entity Endpoints**
- **Observation**: `/v4/write/observation`
- **Feature of Interest**: `/v4/write/foi`
- **Parameter**: `/v4/write/parameter`
- **Voordelen**: Consistentie, duidelijke verantwoordelijkheidscheiding

### 3. **Batch Options**
- **continueOnError**: Doorgaan bij fouten in batch (default: true)
- **validateOnly**: Alleen validatie, geen uitvoering (default: false)
- **callbackUrl**: Verplichte notificatie URL

---

## 📝 Conformiteitseisen

### **MOET** (Vereist voor conformiteit)
- [ ] OpenAPI 3.1 specificatie formaat gebruiken
- [ ] DD API V4 Core datamodellen gebruiken
- [ ] Server-controlled identifiers (string) genereren
- [ ] Key Exchange mechanisme implementeren
- [ ] Asynchrone verwerking ondersteunen
- [ ] `callbackUrl` ondersteunen
- [ ] `X-Max-Batch-Size` header in 413 responses
- [ ] Per-entity endpoints implementeren
- [ ] Job tracking via status endpoints

### **MAG** (Optioneel)
- [ ] Additionele endpoints implementeren
- [ ] Additionele authenticatiemethoden ondersteunen
- [ ] Rate limiting implementeren

---

## 🔗 Externe Referenties

| **Standaard** | **Organisatie**    | **URL**                                                              |
|---------------|--------------------|----------------------------------------------------------------------|
| OpenAPI 3.1   | OpenAPI Initiative | [spec.openapis.org/oas/v3.1.0](https://spec.openapis.org/oas/v3.1.0) |
| JSON          | IETF               | [RFC 8259](https://datatracker.ietf.org/doc/html/rfc8259)            |
| GeoJSON       | IETF               | [RFC 7946](https://datatracker.ietf.org/doc/html/rfc7946)            |
| ISO 8601      | ISO                | [ISO 8601:2004](https://www.iso.org/standard/40874.html)             |
| HTTP/1.1      | IETF               | [RFC 9112](https://datatracker.ietf.org/doc/html/rfc9112)            |
| HTTPS         | IETF               | [RFC 2818](https://datatracker.ietf.org/doc/html/rfc2818)            |
| OAuth2        | IETF               | [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749)            |
| Bearer Token  | IETF               | [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750)            |
| Aquo          | PDOK               | [Aquo](https://aquo.nl)                              |
| DD API V4     | Digitale Delta     | [DD API V4](https://digitaledeltaorg.github.io/DD-API-V4-ReSpec/)    |

---

## 📞 Contact & Ondersteuning

| **Eigenschap**   | **Waarde**                                                                                     |
|------------------|------------------------------------------------------------------------------------------------|
| **Organisatie**  | Digitale Delta                                                                                 |
| **Repository**   | [DD-API-V4-Write-ReSpec](https://github.com/DigitaleDeltaOrg/DD-API-V4-Write-ReSpec)           |
| **Documentatie** | [DD API V4 Write Docs](https://digitaledeltaorg.github.io/DD-API-V4-ReSpec/20-00-write-module) |

---

*Dit SBOS document beschrijft alle standaarden, vocabularia en technische specificaties die gebruikt worden in de DD API V4 Write Module. Het is bedoeld voor AI-systemen, ontwikkelaars en compliance tools om de specificatie beter te begrijpen.*
