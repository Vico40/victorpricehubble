 This document provides the complete architecture overview, data flow specifications, and production handoff roadmap for the MaBanque property valuation prototype.

# 1. Live Environments & Production Endpoints

-  **Website URL: **[https://vico40.github.io/victorpricehubble/](https://vico40.github.io/victorpricehubble/)

-  **n8n Webhook Endpoint: **[https://victoralbrieux.app.n8n.cloud/webhook/valuation-mabanque](https://victoralbrieux.app.n8n.cloud/webhook/valuation-mabanque)

-  **Target Valuation API: **PriceHubble International API ([https://docs.pricehubble.com/international/getting_started_api/](https://docs.pricehubble.com/international/getting_started_api/))

-  **Lead Storage (Google sheets):** [https://docs.google.com/spreadsheets/d/18eZFTo93x_blIvUbQCi1dpJZvFmJKdud6MDDU3ESV0c/edit?gid=0#gid=0](https://docs.google.com/spreadsheets/d/18eZFTo93x_blIvUbQCi1dpJZvFmJKdud6MDDU3ESV0c/edit?gid=0#gid=0)


# 2. System Architecture Diagram

![Image: ](https://devoteam-dmo.monday.com/protected_static/30004817/resources/266292613/PriceHubble%20Workflow%20%281%29.png)


# 3. Component Breakdown: Real vs. Mocked / Simplified

|  **Component** |  **Status** |  **Implementation Details** |
| --- | --- | --- |
|  Frontend UI |  Real |  Mobile-first single-page application built with Vanilla JS and Tailwind CSS, hosted on GitHub Pages. |
|  Client Validation |  Real** (only for FR addresses)** |  Strict HTML5/JS input constraints (Living area 5-1000m², Land area 10-1000000m², Construction Year ≥ 1200). |
|  Address Geocoding |  Real |  Dual-layer address parsing using French Government BAN API ([https://api-adresse.data.gouv.fr](https://api-adresse.data.gouv.fr)) with OpenStreetMap Nominatim fallback. |
|  Valuation Engine |  Real** (only for properties to sale with FR addresses)**
 |  End-to-end integration with PriceHubble API (Authentication, Dossier Creation, Valuation Fetching). |
|  Price per m² |  Real |  Computed dynamically on the client side (targetPrice / surface). |
|  Confidence Index |  Real |  Extracted dynamically from PriceHubble (`valuationSale.valuationConfidence`) and mapped to English labels (_Low_, _Medium_, _High_). |
|  Background Action |  Real |  Asynchronous lead logging into Google Sheets running in parallel with the HTTP webhook response. |

# 4. Data Flow & Payload Specifications
## A. Webhook Payload (Frontend ➔ n8n)

 POST [https://victoralbrieux.app.n8n.cloud/webhook/valuation-mabanque](https://victoralbrieux.app.n8n.cloud/webhook/valuation-mabanque)

```json
{
 "address": "890 Route Départementale 817 40300 Orthevielle",
 "street_number": "890",
 "street": "Route Départementale 817",
 "postcode": "40300",
 "city": "Orthevielle",
 "country_code": "FR",
 "coordinates": {
  "latitude": 43.558109,
  "longitude": -1.14763
 },
 "property_type": "house",
 "surface": 227,
 "land_area": 1800,
 "rooms": 6,
 "building_year": 1900,
  "email": "YOUR_EMAIL_ADDRESS"
}

```

## B. PriceHubble Authentication (n8n ➔ PriceHubble API)

 POST [https://api.pricehubble.com/auth/login/credentials](https://api.pricehubble.com/auth/login/credentials)

```json
{
 "username": "YOUR_PRICEHUBBLE_USERNAME",
 "password": "YOUR_PRICEHUBBLE_PASSWORD"
}

```

## C.  PriceHubble Dossier Creation (n8n ➔ PriceHubble API)

 POST [https://api.pricehubble.com/api/v1/dossiers](https://api.pricehubble.com/api/v1/dossiers)

```json
{
  "dealType": "sale",
  "property": {
    "location": {
      "address": {
        "city": "{{ $('Webhook').item.json.body.city }}",
        "houseNumber": "{{ $('Webhook').item.json.body.street_number }}",
        "postCode": {{ JSON.stringify($('Webhook').item.json.body.postcode) }},
        "street": {{ JSON.stringify($('Webhook').item.json.body.street) }}
      },
      "coordinates": {
        "latitude": {{ $('Webhook').item.json.body.coordinates.latitude }},
        "longitude": {{ $('Webhook').item.json.body.coordinates.longitude }}
      }
    },
    "propertyType": {
      "code": {{ JSON.stringify($('Webhook').item.json.body.property_type) }}
    },
    "livingArea": {{ $('Webhook').item.json.body.surface }},
    "buildingYear" : {{ $('Webhook').item.json.body.building_year }},
    "landArea": {{ $('Webhook').item.json.body.land_area }}
  },
  "countryCode": {{ JSON.stringify($('Webhook').item.json.body.country_code) }}
}

```

## D. PriceHubble Dossier Valuation (n8n ➔ PriceHubble API)

 POST https://api.pricehubble.com/api/v1/dossiers/{{ $json.id }}/valuation

## E. Synchronous Response Payload (n8n ➔ Frontend)

```json
{
  "valuation": {
    "target": {{ $json.valuationSale.value }},
    "lower": {{ $json.valuationSale.valueRange.lower }},
    "upper": {{ $json.valuationSale.valueRange.upper }},
    "confidence": "{{ $json.valuationSale.valuationConfidence }}"
  }
}

```

## F. Google Sheets Column Mapping (Async Branch)

-  Email: {{ $('Webhook').item.json.body.email }}

-  Street number: {{ $('Webhook').item.json.body.street_number }}

-  Street name: {{ $('Webhook').item.json.body.street }}

-  City: {{ $('Webhook').item.json.body.city }}

-  Postcode: {{ $('Webhook').item.json.body.postcode }}

-  Country: {{ $('Webhook').item.json.body.country_code }}

-  Living area: {{ $('Webhook').item.json.body.surface }}

-  Number of rooms: {{ $('Webhook').item.json.body.rooms }}

-  Price: {{ $json.valuationSale.value }}

-  Upper: {{ $json.valuationSale.valueRange.upper }}

-  Lower: {{ $json.valuationSale.valueRange.lower }}

-  Confidence: {{ $json.valuationSale.valuationConfidence }}


# 5. Production Engineering Handoff Roadmap

## 1. API Architecture & Backend-for-Frontend

-  **API Gateway : **Replace the public n8n webhook with an authenticated bank API endpoint protected by API keys or OAuth2 client credentials.

-  **Credentials Management:** Move PriceHubble credentials out of the n8n HTTP Request node into secure environment vaults (e.g., AWS Secrets Manager, HashiCorp Vault).

## 2. Strategy for n8n in Production

-  **Core Logic Migration:** Migrate the primary valuation request chain (Auth ➔ Dossier ➔ Valuation) to a native backend service (Node.js / Java / Python) to minimize network latency and guarantee sub-second response times.

-  **Async Workflow Delegation:** Retain n8n in an internal network segment as an asynchronous event consumer to handle Google Sheets synchronization, email notifications etc...

## 3. Caching & Request Hardening

-  **JWT Caching:** Cache the PriceHubble authentication token in Redis with an automated refresh mechanism prior to its expiration, eliminating 1 HTTP round-trip per user evaluation and the risk of triggering the protection system.

-  **Rate Limiting & Security:** Add Cloudflare / AWS WAF rate limiting on the submit endpoint, integrate bot protection (Cloudflare Turnstile), and sanitize all user input strings.

## 4. Asset Pipeline & GDPR Compliance

-  **Frontend Compilation:** Import Tailwind CSS via PostCSS/Vite inside the bank's core repository instead of using the browser CDN script, optimizing mobile bundle sizes.

-  **GDPR Opt-In:** Implement an explicit marketing consent checkbox prior to capturing the user's email address, ensuring compliance with EU data protection directives.
