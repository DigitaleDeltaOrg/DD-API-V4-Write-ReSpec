# SBOM: DD API V4 Write Module

**Software Bill of Materials** voor de DD API V4 Write Module specificatie.

---<a name="top"></a>

## 📋 Overzicht

| Eigenschap         | Waarde                       |
|--------------------|------------------------------|
| **Naam**           | DD API V4 Write Module       |
| **Versie**         | 4.0.0                        |
| **Type**           | API Specificatie             |
| **Licentie**       | CC BY 4.0                    |
| **Laatste update** | 2024                         |
| **SBOM Formaat**   | CycloneDX 1.4 (geïnspireerd) |
| **SBOM Versie**    | 1.0                          |

---

## 📦 Componenten

### 1. Specificatiedocumenten

| **Naam**                    | **Type** | **Versie** | **Beschrijving**                            | **Locatie**                    |
|-----------------------------|----------|------------|---------------------------------------------|--------------------------------|
| README.md                   | Markdown | 4.0        | Hoofddocument met overzicht en conformiteit | `/README.md`                   |
| 00-00-write-abstract.md     | Markdown | 4.0        | Abstract met kernprincipes                  | `/00-00-write-abstract.md`     |
| 01-00-write-introduction.md | Markdown | 4.0        | Inleiding met scope en ontwerpkeuzes        | `/01-00-write-introduction.md` |
| 02-00-batch-structure.md    | Markdown | 4.0        | Batch request/response structuur            | `/02-00-batch-structure.md`    |
| 03-00-validation.md         | Markdown | 4.0        | Validatie regels en foutcodes               | `/03-00-validation.md`         |
| 04-00-endpoints.md          | Markdown | 4.0        | API endpoints en voorbeelden                | `/04-00-endpoints.md`          |
| 05-00-key-exchange.md       | Markdown | 4.0        | Key exchange mechanisme                     | `/05-00-key-exchange.md`       |
| CONTRIBUTING.md             | Markdown | 4.0        | Bijdrage richtlijnen                        | `/CONTRIBUTING.md`             |
| LICENSE                     | Text     | -          | Licentietekst (CC BY 4.0)                   | `/LICENSE`                     |

### 2. OpenAPI Specificatie

| **Naam**          | **Type**    | **Versie** | **Beschrijving**              | **Locatie**                           |
|-------------------|-------------|------------|-------------------------------|---------------------------------------|
| oas3.1-write.yaml | OpenAPI 3.1 | 4.0.0      | Complete OpenAPI specificatie | `/definitions/v4.0/oas3.1-write.yaml` |

### 3. HTML Documentatie

| **Naam**   | **Type** | **Beschrijving** | **Locatie**                    |
|------------|----------|------------------|--------------------------------|
| index.html | HTML     | Hoofdindex       | `/definitions/index.html`      |

---

## 🔗 Afhankelijkheden

### 1. Externe Specificaties

| **Afhankelijkheid** | **Type**         | **Versie** | **Beschrijving**                         | **URL**                                                           |
|---------------------|------------------|------------|------------------------------------------|-------------------------------------------------------------------|
| **DD API V4 Core**  | API Specificatie | 4.0        | Basis DD API V4 specificatie (read-only) | [DD API V4](https://digitaledeltaorg.github.io/DD-API-V4-ReSpec/) |
| **OpenAPI 3.1**     | Specificatie     | 3.1.0      | OpenAPI Specification                    | [OAS 3.1](https://spec.openapis.org/oas/v3.1.0)                   |

### 2. Data Formaten

| **Formaat**  | **Standaard** | **Versie** | **Gebruik**            |
|--------------|---------------|------------|------------------------|
| **JSON**     | RFC 8259      | -          | Request/response body  |
| **GeoJSON**  | RFC 7946      | -          | Geometrie definitie    |
| **ISO 8601** | ISO 8601      | -          | Datum/tijd formatering |

### 3. Protocol Standaarden

| **Protocol** | **Standaard** | **Versie** | **Gebruik**           |
|--------------|---------------|------------|-----------------------|
| **HTTP/1.1** | RFC 9112      | -          | Transport protocol    |
| **HTTPS**    | RFC 2818      | -          | Beveiligde verbinding |

---

## 📜 Vocabularia & Namespaces

### 1. Semantische Vocabularia

| **Vocabulaire** | **Namespace** | **Beschrijving**                    | **Gebruik**         |
|-----------------|---------------|-------------------------------------|---------------------|
| **OM&S**        | -             | Objecten en Services                | ID formaat (string) |
| **Aquo**        | -             | Domeintabellen/waarden en begrippen | ID formaat (string) |

### 2. Eigene Namespaces

| **Namespace**   | **Beschrijving**       | **Base URL** |
|-----------------|------------------------|--------------|
| DD API V4 Write | Write module endpoints | `/v4/write/` |

---

## 🔐 Beveiliging & Authenticatie

### Ondersteunde Mechanismen

| **Mechanisme** | **Standaard** | **Beschrijving**                |
|----------------|---------------|---------------------------------|
| Bearer Token   | RFC 6750      | `Authorization: Bearer <token>` |
| OAuth2         | RFC 6749      | Client Credentials Flow         |
| mTLS           | RFC 5246      | Mutual TLS certificaten         |

---

## 📄 Licenties

| **Component**          | **Licentie** | **Licentie URL**                                                                             |
|------------------------|--------------|----------------------------------------------------------------------------------------------|
| DD API V4 Write Module | CC BY 4.0    | [https://creativecommons.org/licenses/by/4.0/](https://creativecommons.org/licenses/by/4.0/) |
| OpenAPI Specificatie   | Apache 2.0   | [https://www.apache.org/licenses/LICENSE-2.0](https://www.apache.org/licenses/LICENSE-2.0)   |

---

## 🔍 SBOM Metadata

```yaml
sbom:
  name: DD-API-V4-Write-Module
  version: "1.0"
  format: "CycloneDX-inspired"
  created: "2024-01-01T00:00:00Z"
  tools:
    - name: "Mistral Vibe"
      version: "1.0"
      vendor: "Mistral AI"
```

---

## 📞 Contact & Ondersteuning

| **Eigenschap**   | **Waarde**                                                                           |
|------------------|--------------------------------------------------------------------------------------|
| **Organisatie**  | Digitale Delta                                                                       |
| **Repository**   | [DD-API-V4-Write-ReSpec](https://github.com/DigitaleDeltaOrg/DD-API-V4-Write-ReSpec) |
| **Documentatie** | [DD-API-V4-Write-ReSpec](https://github.com/DigitaleDeltaOrg/DD-API-V4-Write-ReSpec) |

---

*Dit SBOM document is gegenereerd voor de DD API V4 Write Module specificatie en beschrijft de componenten, afhankelijkheden en standaarden die gebruikt worden in de specificatie zelf, niet in een implementatie ervan.*
