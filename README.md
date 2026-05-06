# Hotel Property Management System

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source PMS that unifies reservations, front desk, housekeeping, and revenue management for independent hotels and boutique groups.

A property management system built for the era of agentic guest journeys and AI-driven distribution. It targets independent hotels and small-to-mid groups that find Oracle OPERA over-engineered, Mews and Cloudbeds opaque on pricing, and QloApps under-invested — while embedding AI capabilities (dynamic pricing, predictive housekeeping, automated guest communication) directly into the PMS core rather than bolting them on as third-party add-ons.

---

## Why a New PMS?

- **Revenue management is still bolted on.** Mews and most mid-market PMS platforms rely on third-party RMS integrations for dynamic pricing; an embedded AI pricing engine eliminates that integration cost and complexity.
- **Pricing is opaque across the category.** Cloudbeds, Mews, and OPERA all use custom or hidden pricing; transparent per-room or per-booking pricing is a near-universal gap.
- **Enterprise tools are over-engineered for independents.** Oracle OPERA implementations run into months and hundreds of thousands of dollars, while boutique groups lack a purpose-built multi-property option.
- **The only credible open-source PMS (QloApps) is under-invested.** It lacks enterprise features, native channel management, AI, and modern analytics.
- **AI discovery is reshaping bookings.** US travellers using generative AI for trip planning more than doubled in a year while traditional search dropped from 51% to 36% — hotels with disconnected systems risk disappearing from AI-powered discovery.

---

## Key Features

### Reservations & Front Desk

- Reservation management: create, modify, cancel, group blocks, hold inventory
- Front desk workflows: check-in, check-out, room assignment, folio management
- Guest profiles with stay history, preferences, and communication log
- Group booking and multi-property reservation handling

### Inventory, Distribution & Booking

- Real-time room inventory and availability engine with overbooking prevention
- Channel manager integration for major OTAs (Booking.com, Expedia, Airbnb)
- Direct booking engine: embeddable widget and standalone page
- Rate plan management with seasonal pricing

### Housekeeping & Operations

- Housekeeping module with mobile task assignment and live room status
- Multi-property dashboard with consolidated inventory and revenue view
- Mobile app for housekeeping, front desk, and managers

### Billing, Payments & Compliance

- Billing, invoicing, and payment capture
- PCI-compliant card storage
- Automated PCI DSS and GDPR compliance posture reporting (backlog)

### Analytics & Reporting

- Core operational reporting: occupancy %, ADR, RevPAR, revenue by segment
- Advanced BI with natural-language querying (backlog)

### Platform & Integrations

- REST API with OpenAPI specification and OAuth 2.0 authentication
- Integration marketplace and webhook framework for third-party tools
- MCP server for AI agent access to PMS data and actions (backlog)

---

## AI-Native Advantage

AI sits in the PMS core rather than as a third-party module: a dynamic pricing engine ingests real-time demand signals (events, weather, competitor rates, flight arrivals) to optimise rates at room-type and day-of-week granularity. Agentic guest communication handles pre-arrival, in-stay, and post-stay journeys — answering questions, upselling, and resolving issues without front-desk involvement. Predictive housekeeping scheduling uses check-out patterns and occupancy forecasts to sequence cleaning, while AI-generated content optimisation continuously tunes how the property appears across OTA and AI discovery surfaces. Anonymised guest preference modelling personalises room assignments, amenities, and post-stay offers to drive direct rebooking.

---

## Tech Stack & Deployment

The platform is designed cloud-native with self-hostable deployment as a first-class option, mirroring the data-control benefits of QloApps while delivering enterprise-grade depth. Integration is API-first: a full REST API with OpenAPI specification and OAuth 2.0, modelled on the open-API approach taken by Mews and Apaleo. Distribution interoperability is built around OTA XML standards and HTNG integration patterns, with PCI DSS, GDPR/CCPA, and ISO 27001 considerations addressed in the architecture. An MCP server is on the roadmap to expose PMS data and actions to AI agents.

---

## Market Context

The global hotel PMS software market was estimated at USD 1.62–1.73 billion in 2025–2026 and is projected to reach USD 2.44 billion by 2031 at roughly 7% CAGR (Mordor Intelligence; Market Growth Reports). Pricing ranges from ~€26–€42/month entry-level (Amenitiz, eviivo, Smoobu) through €80–€200/month mid-market (Cloudbeds, Little Hotelier, RMS Cloud) up to custom enterprise contracts (Oracle OPERA, Agilysys) costing hundreds of thousands annually. Primary buyers are independent hotel owners, hotel-group operations and IT directors, revenue managers, and front office managers.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
