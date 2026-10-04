# Rewater


ReWater gives water a digital identity. Every water source — tap, groundwater, rain, wastewater — can be tested, profiled, tracked through treatment, verified, and matched to a safe reuse pathway. The platform connects a web dashboard to the **AquaSense** ESP32 sensor prototype, turning real hardware readings into Water Passports, Water Credits, and measurable impact.

> **AquaSense is future hardware.** Sensor readings in the prototype are simulated unless a real ESP32 device is connected. All recommendations clearly label their uncertainty.

---

## Table of Contents

1. [Tech Stack](#tech-stack)
2. [Getting Started](#getting-started)
3. [Website Features](#website-features)
4. [Water Passport System](#water-passport-system)
5. [AquaSense Hardware Integration](#aquasense-hardware-integration)
6. [Water Credits System](#water-credits-system)
7. [Database Schema](#database-schema)
8. [ESP32 Firmware Guide](#esp32-firmware-guide)
9. [API Reference](#api-reference)
10. [Security & Safety Notes](#security--safety-notes)

---

## Tech Stack

| Layer | Technology |
|---|---|
| Frontend | React 18, TypeScript, Tailwind CSS, Vite |
| Animations | Framer Motion, GSAP, Three.js, Spline |
| Icons | Lucide React |
| Maps | Leaflet |
| Backend / Database | Supabase (PostgreSQL, Auth, Edge Functions, Row Level Security) |
| Hardware | ESP32 DevKit (Arduino C++), pH/TDS/Turbidity/Temperature sensors |

The Supabase database is **built into this Bolt project** — no separate account is needed. The database URL and anon key are already configured in the project environment.

---

## Getting Started

### Prerequisites

- Node.js 18+
- An Arduino IDE (only if you are building the AquaSense hardware)

### Install & Run

```bash
npm install
npm run dev
```

The dev server starts automatically in Bolt. For local development outside Bolt:

```bash
npm install
npm run dev      # development server
npm run build    # production build
npm run typecheck # TypeScript check
npm run lint     # ESLint
```

### Environment Variables

The `.env` file is pre-configured with:

| Variable | Purpose |
|---|---|
| `VITE_SUPABASE_URL` | Supabase project URL (used by the website and the ESP32) |
| `VITE_SUPABASE_ANON_KEY` | Supabase anon key (safe to embed in frontend code and firmware) |

These same values go into the ESP32 firmware constants so the hardware can send readings to the same database.

---

## Website Features

### Pages

| Page | What the user can do |
|---|---|
| **Home** | Landing page with hero, platform overview, and navigation to all sections |
| **Check** | Test a water sample — enter pH, temperature, turbidity, TDS, and visual observations. Generates a full water quality profile with findings, treatment pathway, recoverability assessment, and reuse recommendations |
| **Water Passport** | View and manage Water Passports. Each passport tracks a water sample through its circular journey from source to verified reuse. See timeline, measurements, treatment history, retest records |
| **Water Journey** | Interactive circular economy simulator. Explore how water moves through collection, treatment, recovery, and reuse stages |
| **Map** | Interactive map of water sources with location, quality data, and recovery potential |
| **Impact** | Impact dashboard showing water recovered, freshwater avoided, SDG contributions, and resource recovery metrics |
| **Water Credits** | View credit balance, transaction history, generate credits from verified passports, donate or assign credits to HydroStations |
| **Hydro Stations** | Manage drinking-water stations for workers. Sponsor stations with Water Credits, track dispensing and worker visits |
| **Future Lab** | Explore future technologies: atmospheric water generation, piezoelectric harvesting, digital twins, AI analysis, circularity modeling |
| **AquaSense** | Connect a real AquaSense ESP32 device, register it to a Water Passport, view live and historical sensor readings |
| **Brands** | Database of bottled water brands with published mineral composition for comparison |
| **Account** | User profile, account settings, subscription management |
| **Business Profile** | Business registration, subscription plans, revenue tracking, feature access management |
| **Auth** | Sign in / sign up with email and password |
| **Tutorial** | Guided walkthrough of the platform (accessible from "How It Works" in the nav) |

### Authentication

- Email and password sign-up/sign-in via Supabase Auth
- Email confirmation is OFF for faster testing
- A demo account is available for exploring without creating an account
- Certain pages (Water Passport, Water Credits, Hydro Stations, Business Profile, Account) require sign-in — a guard modal prompts the user if they try to access these without being logged in

---

## Water Passport System

A **Water Passport** is the digital identity of a water sample. It is created when a user tests water on the Check page and persists through the entire circular journey.

### Passport Journey Stages

```
Collected → Identified → Profiled → Analyzed → Pathway Identified
    → Treated → Retest Required → Verified (or Not Verified) → Reuse Pathway
```

| Stage | Description |
|---|---|
| Collected | Water sample is collected and a passport is created |
| Identified | Source type is classified (tap, groundwater, rain, wastewater, etc.) |
| Profiled | Full water quality profile is generated from measurements |
| Analyzed | Findings, treatment pathway, and recoverability are assessed |
| Pathway Identified | Recommended treatment stages and reuse pathways are determined |
| Treated | Treatment has been applied to the water |
| Retest Required | Post-treatment retesting is needed before verification |
| Verified | Retest confirms treatment was effective |
| Not Verified | Retest shows treatment was insufficient |
| Reuse Pathway | Water is matched to a safe reuse application |

### Measurement Origins

Every measurement records where it came from:

| Origin | Description |
|---|---|
| `aquasense` | Real reading from the AquaSense ESP32 device |
| `aquacard` | Chemical test strip results (nitrate, nitrite, hardness, chlorine) |
| `camera` | Visual observation (colour, cloudiness, particles, foam, oil) |
| `manual` | Manually entered by the user |
| `lab` | Laboratory-verified result |
| `simulated` | Simulated/demo data (clearly flagged) |

### Water Analysis Engine

The analysis engine (`src/lib/waterAnalysis.ts`) evaluates each parameter against reference ranges and assigns a severity level: **normal, watch, elevated, high, or critical**. It then:

1. Identifies findings (individual parameter issues + multi-parameter patterns)
2. Builds a treatment pathway (pre-screening, filtration, adsorption, biological treatment, desalination, disinfection, final testing)
3. Assesses recoverability (high, moderate, low, or not recommended)
4. Recommends reuse pathways (irrigation, toilet flushing, cleaning, industrial, potable)
5. Identifies resource recovery opportunities (nutrients, salts, biogas, heat)
6. Builds a prevention/monitoring plan
7. Lists what needs laboratory verification

---

## AquaSense Hardware Integration

AquaSense is an ESP32-based water quality sensor prototype that sends real readings to the ReWater database.

### How It Works

```
ESP32 + Sensors → Wi-Fi → Supabase REST API → sensor_readings table → Website displays live data
```

1. The ESP32 reads pH, TDS, turbidity, and temperature from connected sensors
2. It sends the readings as JSON to the Supabase `sensor_readings` table via HTTP POST
3. Each reading includes a `passport_id` linking it to a specific Water Passport
4. The website fetches real (non-simulated) readings for the selected passport and displays them

### Device Registration

Before sending readings, the ESP32 registers itself to a Water Passport via the `aquasense-register` edge function:

- **Register**: `POST /functions/v1/aquasense-register` with `{ device_id, passport_id }`
- **Look up**: `GET /functions/v1/aquasense-register?device_id=AQUASENSE-01`

This lets you change which passport a device is linked to without reflashing the firmware — just update the mapping through the edge function.

### Sensor Readings Table

| Column | Type | Description |
|---|---|---|
| `id` | uuid | Auto-generated primary key |
| `device_id` | text | Identifier of the AquaSense device (e.g. `AQUASENSE-01`) |
| `sample_id` | text | Optional link to a water sample |
| `passport_id` | text | Optional link to a Water Passport (e.g. `RW-WP-2026-00001`) |
| `timestamp` | timestamptz | When the reading was taken |
| `ph` | real | pH value (0–14) |
| `tds` | real | Total dissolved solids (ppm) |
| `ec` | real | Electrical conductivity (µS/cm) |
| `turbidity` | real | Turbidity (NTU) |
| `temperature` | real | Temperature (°C) |
| `source` | text | Water source type |
| `status` | text | Reading status (connected, disconnected, error) |
| `calibration_info` | jsonb | Calibration metadata |
| `is_simulated` | boolean | `false` for real device readings, `true` for demo data |

### Device Mappings Table

| Column | Type | Description |
|---|---|---|
| `id` | uuid | Auto-generated primary key |
| `device_id` | text (unique) | One mapping per device |
| `passport_id` | text | The passport this device is linked to |
| `updated_at` | timestamptz | Last update time |
| `created_at` | timestamptz | Creation time |

---

## Water Credits System

Water Credits quantify the environmental value of water recovery. One credit represents 100 litres of credit-eligible water.

### How Credits Are Calculated

```
Credit-eligible litres = recovered litres × 20% (partnership share)
Water Credits = credit-eligible litres ÷ 100
```

**Example**: 1,000 litres of wastewater treated → 1,000 L recovered → 200 L credit-eligible → 2.0 Water Credits

### Revenue Share Model

Every credit generation records the 70/20/10 split:

| Share | Percentage | Recipient |
|---|---|---|
| Company | 70% | The business that recovered the water |
| HydroStation Fund | 20% | Funds drinking-water stations for workers |
| ReWater | 10% | Platform fee |

### Transaction Types

| Type | Description |
|---|---|
| `earned` | Credits generated from a verified Water Passport |
| `donated` | Credits donated to a HydroStation |
| `assigned` | Credits assigned to a specific HydroStation |
| `used` | Credits spent dispensing water at a HydroStation |

### Subscription Plans

| Plan | Price (AED/mo) | Features |
|---|---|---|
| Free | 0 | Water Passports, Water Journey |
| Essential | 299 | + Water Credits, Impact Reporting |
| Professional | 799 | + Analytics, Predictive Mapping, ESG Export |
| Enterprise | 1,999 | + Priority Support, Custom Integrations |

All paid plans include a 7-day free trial.

---

## Database Schema

All tables are in the `public` schema. Row Level Security (RLS) is enabled on every table.

### Tables Overview

| Table | Purpose | RLS Scope |
|---|---|---|
| `water_sources` | Catalog of water sources with location and quality | Public (anon + authenticated) |
| `water_samples` | User-created water sample records with measurements | Public |
| `water_brands` | Bottled water brand database with mineral composition | Public |
| `alerts` | Early-warning alerts triggered by trend changes | Public |
| `scenarios` | Water circularity simulator scenarios | Public |
| `impact_records` | Impact dashboard data | Public |
| `treatment_records` | Before/after treatment comparison data | Public |
| `sensor_readings` | Readings from AquaSense ESP32 devices | Public insert/select, owner-scoped update/delete |
| `aquasense_device_mappings` | Device-to-passport link mappings | Public |
| `water_passports` | Water Passport records with full journey data | Owner-scoped (auth.uid) |
| `water_credit_transactions` | Water Credit transaction ledger | Owner-scoped |
| `hydro_stations` | Drinking-water station management | Owner-scoped |
| `hydro_station_visits` | Worker visit records at stations | Owner-scoped |
| `hydro_station_dispensing` | Water dispensing records with credit info | Owner-scoped |
| `business_subscriptions` | Business subscription plans and features | Owner-scoped |
| `rewater_revenue` | ReWater platform revenue tracking | Owner-scoped |

### Migration History

| # | File | Description |
|---|---|---|
| 1 | `20260912165131_create_rewater_schema.sql` | Core tables: water_sources, water_samples, water_brands, alerts, scenarios, impact_records, treatment_records |
| 2 | `20260913040323_add_camera_fields.sql` | Camera observation fields on water_samples |
| 3 | `20260915084558_add_sensor_readings_and_auth.sql` | sensor_readings table + Supabase Auth |
| 4 | `20260920040147_create_hydro_stations.sql` | hydro_stations + hydro_station_visits tables |
| 5 | `20260921103117_add_credits_dispensing_passport_persistence.sql` | Water Credit transactions, dispensing records, water_passports table |
| 6 | `20260921110856_add_credit_calculation_partners_subscriptions.sql` | Credit calculation fields, business_subscriptions, rewater_revenue |
| 7 | `20260921113244_add_credit_eligible_litres.sql` | credit_eligible_litres column on water_passports |
| 8 | `20260922042457_add_subscription_trial_fields.sql` | Trial fields on business_subscriptions |
| 9 | `20261004091627_add_passport_id_to_sensor_readings.sql` | passport_id column on sensor_readings |
| 10 | `20261004091910_create_aquasense_device_mappings.sql` | aquasense_device_mappings table |

---

## ESP32 Firmware Guide

The complete firmware is in [`firmware/aquasense_firmware.ino`](firmware/aquasense_firmware.ino). Open it in the Arduino IDE.

### Required Hardware

| Component | Purpose |
|---|---|
| ESP32 DevKit (e.g. ESP32-WROOM-32) | Microcontroller with Wi-Fi |
| pH probe + interface board (e.g. Gravity SEN-0161) | pH measurement |
| TDS sensor + probe (e.g. Gravity SEN-0244) | Total dissolved solids |
| Turbidity sensor (e.g. Gravity SEN-0182) | Turbidity in NTU |
| DS18B20 temperature probe | Water temperature |
| 0.96" OLED display (SSD1306, I2C) | On-device readings display |
| Breadboard + jumper wires | Prototyping |

### Wiring

| Sensor | ESP32 Pin |
|---|---|
| pH sensor analog out | GPIO 34 |
| TDS sensor analog out | GPIO 35 |
| Turbidity sensor analog out | GPIO 32 |
| DS18B20 data line | GPIO 4 (with 4.7kΩ pull-up resistor) |
| OLED SDA | GPIO 21 |
| OLED SCL | GPIO 22 |

### Firmware Configuration

Before flashing, update these constants at the top of the firmware file:

```cpp
#define WIFI_SSID       "your-wifi-name"
#define WIFI_PASSWORD   "your-wifi-password"

#define SUPABASE_URL      "https://lkkziixgzkufwzdcjyjl.supabase.co"
#define SUPABASE_ANON_KEY "your-anon-key-here"

#define DEVICE_ID  "AQUASENSE-01"
#define PASSPORT_ID "RW-WP-2026-00001"
```

The `SUPABASE_URL` and `SUPABASE_ANON_KEY` values come from the `.env` file in this project. The `PASSPORT_ID` is the Water Passport ID shown on the website — create a passport first on the Check page, then copy its ID.

### Required Arduino Libraries

Install these via the Arduino Library Manager:

- `WiFi.h` (built into ESP32 core)
- `HTTPClient.h` (built into ESP32 core)
- `Adafruit_SSD1306` (OLED display)
- `Adafruit_GFX` (OLED graphics)
- `OneWire` (DS18B20)
- `DallasTemperature` (DS18B20)

### What the Firmware Does

1. Connects to Wi-Fi
2. Looks up its `passport_id` via the edge function (so you can change the mapping without reflashing)
3. Reads all four sensors every 30 seconds
4. Displays readings on the OLED screen
5. Sends readings to the Supabase `sensor_readings` table with `is_simulated: false`
6. Supports serial calibration commands for pH and TDS

### Serial Calibration Commands

Open the Serial Monitor (115200 baud) and type:

| Command | Action |
|---|---|
| `cal:ph:7.0` | Calibrate pH to 7.0 (put probe in pH 7 buffer) |
| `cal:ph:4.0` | Calibrate pH to 4.0 (put probe in pH 4 buffer) |
| `cal:tds:70` | Calibrate TDS to 70 ppm (put probe in 70 ppm standard) |
| `info` | Print current calibration and device info |
| `test` | Send a single test reading to Supabase |

---

## API Reference

### Supabase REST API

The ESP32 and website both use the Supabase REST API (PostgREST).

#### Insert a sensor reading (from ESP32)

```http
POST https://lkkziixgzkufwzdcjyjl.supabase.co/rest/v1/sensor_readings
apikey: <anon-key>
Content-Type: application/json

{
  "device_id": "AQUASENSE-01",
  "passport_id": "RW-WP-2026-00001",
  "timestamp": "2026-10-04T12:00:00Z",
  "ph": 7.2,
  "tds": 320,
  "ec": 640,
  "turbidity": 0.8,
  "temperature": 24.5,
  "source": "tap",
  "status": "connected",
  "is_simulated": false
}
```

#### Fetch latest real reading for a passport

```http
GET https://lkkziixgzkufwzdcjyjl.supabase.co/rest/v1/sensor_readings?passport_id=eq.RW-WP-2026-00001&is_simulated=eq.false&order=timestamp.desc&limit=1
apikey: <anon-key>
```

### Edge Function: aquasense-register

#### Register a device to a passport

```http
POST https://lkkziixgzkufwzdcjyjl.supabase.co/functions/v1/aquasense-register
Authorization: Bearer <anon-key>
Content-Type: application/json

{
  "device_id": "AQUASENSE-01",
  "passport_id": "RW-WP-2026-00001"
}
```

Response:
```json
{
  "device_id": "AQUASENSE-01",
  "passport_id": "RW-WP-2026-00001",
  "updated_at": "2026-10-04T12:00:00Z",
  "message": "Device registered to passport successfully"
}
```

#### Look up a device's current passport mapping

```http
GET https://lkkziixgzkufwzdcjyjl.supabase.co/functions/v1/aquasense-register?device_id=AQUASENSE-01
Authorization: Bearer <anon-key>
```

Response (registered):
```json
{
  "device_id": "AQUASENSE-01",
  "passport_id": "RW-WP-2026-00001",
  "updated_at": "2026-10-04T12:00:00Z"
}
```

Response (not yet registered):
```json
{
  "device_id": "AQUASENSE-01",
  "passport_id": null,
  "message": "Device not yet registered to a passport"
}
```

### curl Examples

```bash
# Register device
curl -X POST https://lkkziixgzkufwzdcjyjl.supabase.co/functions/v1/aquasense-register \
  -H "Authorization: Bearer <anon-key>" \
  -H "Content-Type: application/json" \
  -d '{"device_id":"AQUASENSE-01","passport_id":"RW-WP-2026-00001"}'

# Look up mapping
curl "https://lkkziixgzkufwzdcjyjl.supabase.co/functions/v1/aquasense-register?device_id=AQUASENSE-01" \
  -H "Authorization: Bearer <anon-key>"

# Insert a reading
curl -X POST https://lkkziixgzkufwzdcjyjl.supabase.co/rest/v1/sensor_readings \
  -H "apikey: <anon-key>" \
  -H "Content-Type: application/json" \
  -d '{"device_id":"AQUASENSE-01","passport_id":"RW-WP-2026-00001","ph":7.2,"tds":320,"ec":640,"turbidity":0.8,"temperature":24.5,"source":"tap","status":"connected","is_simulated":false}'
```

---

## Security & Safety Notes

### Data Safety

- **AquaSense readings are raw sensor measurements**, not certified lab results. They are suitable for screening and monitoring, not for confirming water is safe to drink.
- The platform always recommends laboratory verification for coliform bacteria, heavy metals, and full mineral analysis before any potable reuse.
- Every measurement is labelled with its data origin (measured, estimated, observed, simulated, lab-verified) and the analysis engine reports uncertainty levels.

### Key Safety

- The **anon key** is safe to embed in frontend code and ESP32 firmware. It only allows public CRUD operations permitted by RLS policies — it cannot access admin functions or bypass row-level security.
- The **service role key** is used only by the edge function (server-side) and must **never** be included in firmware or frontend code.
- RLS policies on `sensor_readings` and `aquasense_device_mappings` allow public insert and select (by design, so the ESP32 can send readings without user login). Owner-scoped tables (passports, credits, stations, subscriptions) require authentication and only allow access to the owner's own rows.

### ESP32 Network Security

- The ESP32 connects to Wi-Fi using the credentials you program into it. Use a dedicated network if possible.
- All communication with Supabase uses HTTPS/TLS.
- The firmware does not store any user passwords or service role keys.

---

## Project Structure

```
project/
├── src/
│   ├── pages/              # One file per page (Home, Check, Map, etc.)
│   ├── components/         # Shared UI components
│   │   ├── animations/     # Framer Motion / GSAP animation components
│   │   └── ui/             # Reusable UI primitives (spotlight, card-stack, etc.)
│   ├── lib/                # Business logic and data layer
│   │   ├── supabase.ts     # Supabase client + all TypeScript interfaces
│   │   ├── auth.tsx        # Authentication provider
│   │   ├── waterPassport.tsx # Water Passport context and state
│   │   ├── waterCredits.ts # Water Credit calculations and transactions
│   │   ├── waterAnalysis.ts # Water quality analysis engine
│   │   ├── sensorData.ts   # Sensor reading fetch/save functions
│   │   └── ...
│   └── hooks/              # Custom React hooks
├── supabase/
│   ├── migrations/         # SQL migration files (applied in order)
│   ├── functions/          # Edge functions (Deno/TypeScript)
│   │   └── aquasense-register/
│   └── config.toml         # Supabase project config
├── firmware/
│   └── aquasense_firmware.ino  # ESP32 Arduino firmware
├── .env                    # Supabase URL and anon key
└── package.json
```

---

## License

This is a prototype project for the ReWater water intelligence platform.
