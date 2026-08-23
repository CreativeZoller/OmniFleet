# OmniFleet 🚗⚡

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Rust](https://img.shields.io/badge/Rust-1.70+-orange.svg)](https://www.rust-lang.org/)
[![Vue.js](https://img.shields.io/badge/Vue%203-TypeScript-blue.svg)](https://vuejs.org/)

**OmniFleet** is an open-source, self-hosted vehicle tracking application designed for families and multi-user setups. It seamlessly supports both Internal Combustion Engine (ICE) vehicles and Electric Vehicles (EVs), offering advanced consumption analytics, multi-currency support, and intelligent receipt management.

## ✨ Key Features

- **Multi-Vehicle & Multi-User:** Family or fleet-oriented with role-based access control (Owner/Member) and independent user accounts.
- **Unified ICE & EV Support:** Dedicated models for refuels (liters) and charges (kWh, SoC, charging time).
- **Advanced Consumption Math:** Goes beyond the standard "Tank-to-Tank" method. Calculates a unique **"Driver vs. Car" efficiency ratio** by comparing real-world usage against the vehicle's onboard range estimations.
- **Multi-Currency:** Perfect for cross-border travel (e.g., EUR and HUF). Stores transactions in native currencies and normalizes them for unified reporting.
- **Receipt Management:** Upload receipts (PDF/Images) with SHA-256 deduplication. (Future: Automated OCR data extraction).
- **Comprehensive Dashboard:** Visualizes consumption trends, unit price history, monthly costs, and top locations without falling into statistical traps (uses weighted averages).

## 🧮 The "Driver vs. Car" Algorithm

Instead of relying solely on full-tank-to-full-tank metrics, OmniFleet captures the vehicle's estimated range before and after a refuel/charge. 

The core ratio used to determine driving efficiency:
```text
Efficiency Ratio = (Range_After_Current_Refuel - Range_Before_Next_Refuel) / Distance_Traveled
```
- `= 1.0` : You drove exactly as the car predicted.
- `< 1.0` : You drove more efficiently than predicted (e.g., 0.83 = 17% better).
- `> 1.0` : You drove less efficiently than predicted (e.g., 1.17 = 17% worse).

This metric works independently of the vehicle's internal l/100km estimation and provides immediate feedback on driving habits, even for partial refuels.

## 🛠 Tech Stack

**Backend (API & Database):**
- Language: [Rust](https://www.rust-lang.org/) (Workspace architecture)
- Web Framework: [Axum](https://github.com/tokio-rs/axum)
- Database ORM: [SQLx](https://github.com/launchbadge/sqlx)
- Database: PostgreSQL (via Docker Compose)
- Auth: Argon2id password hashing, Session cookies

**Frontend:**
- Framework: [Vue 3](https://vuejs.org/) (Composition API) + TypeScript
- UI Library: [PrimeVue](https://primevue.org/)
- State Management: [Pinia](https://pinia.vuejs.org/)
- Charts: Chart.js (via PrimeVue Chart component)

## 🗺 Project Roadmap

- [ ] **Phase 1: Scaffold & Schema** - Rust workspace setup, PostgreSQL Docker Compose, complete DB schema with migrations.
- [ ] **Phase 2: Backend Core** - Environment config, tracing/logging, connection pooling, and error handling.
- [ ] **Phase 3: Authentication** - User models, Argon2id hashing, session management, and role-based middleware.
- [ ] **Phase 4: CRUD APIs** - REST endpoints for vehicles, places, refuels, charges, and attachments.
- [ ] **Phase 5: Frontend Shell** - Vue 3 setup, PrimeVue integration, navigation, and API service layers.
- [ ] **Phase 6: Refuel & Charge Forms** - Real-time calculated fields, validation, and location autocomplete.
- [ ] **Phase 7: Receipt Uploads** - File handling, SHA-256 deduplication, thumbnails, and access protection.
- [ ] **Phase 8: Reporting Dashboard** - Interactive charts, weighted averages, and tabular data exports.
- [ ] **Phase 9: OCR Integration (Optional)** - Tesseract-based auto-extraction for dates, amounts, and locations from receipts.

## 🚀 Getting Started

```bash
# Clone the repository
git clone https://github.com/your-username/omnifleet.git
cd omnifleet

# Instructions for setting up .env files and running docker-compose up -d
# will be finalized during Phase 1.
```

## 📄 License

This project is licensed under the [MIT License](LICENSE) - see the LICENSE file for details. You are free to use, modify, and distribute this software.
