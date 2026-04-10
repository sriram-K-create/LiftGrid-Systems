# LiftGrid Systems — ER Diagram

**Smart Elevator Control Platform · Database Schema Design**

---

## Overview
LiftGrid Systems is an intelligent elevator control platform built for large commercial buildings across India — corporate towers, malls, airports, hospitals, and high-rise residential complexes.
This repository contains the **Entity Relationship Diagram** for the backend database that powers the platform. The schema is designed to support multi-building infrastructure management, real-time ride requests, elevator allocation, status monitoring, and maintenance tracking.

## Diagram

> 📎 See [`LiftGrid_ERD.pdf`]([https://github.com/sriram-K-create/LiftGrid-Systems/blob/main/LiftGrid_ERD%20(2).pdf]) for the full ER diagram.

## Entities

### Infrastructure Layer

| Entity | Purpose |
|---|---|
| `BUILDING` | Top-level anchor. Each building on the platform is registered here. |
| `FLOOR` | Every floor in a building. Floors belong to one building. |
| `ELEVATOR_SHAFT` | Physical shaft structure inside a building. Permanent — separate from the elevator unit installed in it. |
| `ELEVATOR` | The mechanical unit inside a shaft. Holds config data only — no ride history. |
| `ELEVATOR_FLOOR_COVERAGE` | Junction table resolving the many-to-many between elevators and floors. |

### Operations Layer

| Entity | Purpose |
|---|---|
| `FLOOR_REQUEST` | A ride request raised from a floor — direction, status (pending/assigned/completed), timestamp. |
| `RIDE_ALLOCATION` | The assignment decision — which elevator was dispatched for which request. |
| `RIDE_LOG` | The completed trip record — pickup, destination, start/end time, duration. Used for analytics. |
| `ELEVATOR_STATUS` | Append-only status log — idle, moving, maintenance. Never overwrites; creates a new row per state change. |

### Maintenance Layer

| Entity | Purpose |
|---|---|
| `MAINTENANCE_RECORD` | Full maintenance history per elevator — type, technician, schedule, outcome. Stored independently so history is never lost. |

---

## Key Relationships

```
BUILDING         ||──o{   FLOOR                   (1 building has many floors)
BUILDING         ||──o{   ELEVATOR_SHAFT           (1 building has many shafts)
ELEVATOR_SHAFT   ||──||   ELEVATOR                 (1 shaft houses exactly 1 elevator)
ELEVATOR         ||──o{   ELEVATOR_FLOOR_COVERAGE  (1 elevator serves many floors)
FLOOR            ||──o{   ELEVATOR_FLOOR_COVERAGE  (1 floor served by many elevators)
FLOOR            ||──o{   FLOOR_REQUEST            (1 floor generates many requests)
FLOOR_REQUEST    ||──o|   RIDE_ALLOCATION          (1 request → 1 assignment)
ELEVATOR         ||──o{   RIDE_ALLOCATION          (1 elevator handles many allocations)
RIDE_ALLOCATION  ||──o|   RIDE_LOG                 (1 allocation → 1 completed ride log)
ELEVATOR         ||──o{   RIDE_LOG                 (elevator_id FK for analytics queries)
ELEVATOR         ||──o{   ELEVATOR_STATUS          (1 elevator has many status records)
ELEVATOR         ||──o{   MAINTENANCE_RECORD       (1 elevator has many maintenance events)
```

---

## Design Decisions

**Why is `ELEVATOR_SHAFT` separate from `ELEVATOR`?**
A shaft is a permanent physical structure in the building. An elevator is a replaceable unit. Keeping them separate means you can swap out an elevator without losing the shaft's history or floor coverage configuration.

**Why is `ELEVATOR_STATUS` a separate table?**
Status changes frequently and must be tracked over time. An append-only table lets you answer questions like *"when did Elevator 4 go into maintenance?"* or *"how long was it idle yesterday?"* — which would be impossible if status were a single column on the elevator row.

**Why does `ELEVATOR` hold no ride data?**
The `ELEVATOR` entity is config-only (model, capacity, commissioning date). All dynamic data lives in `RIDE_ALLOCATION`, `RIDE_LOG`, and `ELEVATOR_STATUS`. This keeps the core entity stable and makes analytics queries cleaner.

**Why is there a `building_id` FK on `FLOOR_REQUEST`?**
A deliberate denormalization. It lets operations teams query all requests for a building directly, without joining through `FLOOR` every time.

**Why is `MAINTENANCE_RECORD` fully separate?**
Maintenance history must never be overwritten. A dedicated table with `scheduled_at`, `completed_at`, `technician`, and `outcome` gives a complete audit trail per elevator — independent of operational status.

---

## Queries this schema supports

| Question | How |
|---|---|
| How many buildings are on the platform? | `COUNT(*)` on `BUILDING` |
| How many elevators in a building? | Join `ELEVATOR` → `ELEVATOR_SHAFT` → `BUILDING` |
| Which floors does elevator X serve? | Query `ELEVATOR_FLOOR_COVERAGE` by `elevator_id` |
| Can multiple elevators serve the same floor? | Yes — via `ELEVATOR_FLOOR_COVERAGE` |
| What is the current status of an elevator? | Latest row in `ELEVATOR_STATUS` by `recorded_at` |
| How many rides did elevator X complete today? | Filter `RIDE_LOG` by `elevator_id` + `completed_at` date |
| Which elevator handled the most requests? | `GROUP BY elevator_id` on `RIDE_ALLOCATION` |
| Which requests are still pending? | Filter `FLOOR_REQUEST` where `status = 'pending'` |
| Full maintenance history for an elevator? | Query `MAINTENANCE_RECORD` by `elevator_id` |

---
