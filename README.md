# Order Fulfillment Automation

> Automate the complete Pick → Pack → Ship fulfillment lifecycle using Postman — no application UI required.

---

## Table of Contents

- [Project Description](#project-description)
- [Assignment Objective](#assignment-objective)
- [Deliverables](#deliverables)
- [Collection Overview](#collection-overview)
- [Data Flow & Request Chaining](#data-flow--request-chaining)
- [Environment Variables](#environment-variables)
- [Setup & Usage](#setup--usage)
- [Edge Cases Handled](#edge-cases-handled)
- [Repository Structure](#repository-structure)

---

## Project Description

This project provides a fully automated Postman collection that executes the complete order fulfillment lifecycle — **Pick**, **Pack**, and **Ship** — through a sequence of API calls, without any interaction with the application UI.

The collection is reusable and configurable via a Postman environment file. It handles real-world edge cases such as orders that already have in-progress shipments, and uses scripting to chain requests and pass data dynamically between API calls.

---

## Assignment Objective

Create a Postman collection that can complete the full fulfillment lifecycle of an order item without using the application UI. The collection should execute the required API calls in sequence and handle the necessary data flow between requests.

---

## Deliverables

| # | Deliverable | Description |
|---|-------------|-------------|
| 1 | **Postman Collection** | `Fulfillment Lifecycle` — automates the complete Pick → Pack → Ship flow using environment/collection variables and scripts to chain requests and manage data across API calls |
| 2 | **Postman Environment** | `Fulfillment Environment` — contains all configurable parameters required to run the collection |

---

## Collection Overview

**Collection name:** `Fulfillment Lifecycle`

The collection contains **4 requests** executed in sequence:

### 1. Find Open Orders
- Queries the Solr search index for orders with `fulfillmentStatus:Created`
- Extracts ship groups from the response and stores them in `shipmentGroups`
- Initialises the `groupIndex` counter to iterate through ship groups

### 2. Pick
- Reads the current ship group from `shipmentGroups[groupIndex]`
- Calls the Pick API to create a new shipment and stores the returned `shipmentId`
- **If an existing in-progress shipment is detected**, skips Pick and jumps directly to Pack via `postman.setNextRequest("Pack")`

### 3. Pack
- Uses the `shipmentId` from Pick to call the Pack API
- Marks the shipment as packed and stores `packedShipmentId`

### 4. Ship
- Uses `packedShipmentId` to call the Ship API and mark the shipment as shipped
- If more ship groups remain, increments `groupIndex` and loops back to Pick

---

## Data Flow & Request Chaining

```
┌─────────────────────┐
│   Find Open Orders  │  ── Queries Solr, extracts ship groups
└────────┬────────────┘     → sets: shipmentGroups, groupIndex
         │
         ▼
┌─────────────────────┐
│        Pick         │  ── Creates shipment for current ship group
└────────┬────────────┘     → sets: shipmentId, orderId, shipGroupSeqId
         │  (skips if existing shipment found → jumps to Pack)
         ▼
┌─────────────────────┐
│        Pack         │  ── Packs the shipment
└────────┬────────────┘     → sets: packedShipmentId
         │
         ▼
┌─────────────────────┐
│        Ship         │  ── Ships the packed shipment
└────────┬────────────┘
         │  (loops back to Pick if more ship groups remain)
         ▼
      [Done]
```

| Technique | Usage |
|-----------|-------|
| `pm.environment.set(key, value)` | Stores response data for downstream requests |
| `pm.environment.get(key)` | Reads stored variables in pre-request and test scripts |
| `postman.setNextRequest("name")` | Controls execution flow — skips steps or loops |
| `JSON.parse(pm.environment.get(...))` | Deserialises stored arrays for iteration |

---

## Environment Variables

**Environment name:** `Fulfillment Environment`

| Variable | Description | Set By |
|----------|-------------|--------|
| `baseSolrUrl` | Base URL for the Solr search API | Manual |
| `basePoortiUrl` | Base URL for the fulfillment API | Manual |
| `facilityId` | Warehouse/facility ID | Manual |
| `productStoreId` | Product store ID | Manual |
| `pickerPartyId` | Party ID of the assigned picker | Manual |
| `picklistId` | Picklist ID used during Pick | Manual |
| `bearer_token` | Bearer token for API authentication | Manual |
| `shipmentGroups` | JSON array of ship groups from Find Open Orders | Runtime |
| `groupIndex` | Current index into `shipmentGroups` | Runtime |
| `shipmentId` | Shipment ID created or found during Pick | Runtime |
| `orderId` | Order ID of the current ship group | Runtime |
| `shipGroupSeqId` | Ship group sequence ID | Runtime |
| `currentOrderItems` | Order items in the current ship group | Runtime |
| `packedShipmentId` | Shipment ID after packing, used in Ship | Runtime |

> Variables marked **Runtime** are set automatically by collection scripts and do not need to be configured before running.

---

## Setup & Usage

### Prerequisites
- [Postman](https://www.postman.com/downloads/) desktop app
- Bearer token for the target fulfillment system
- Network access to the Solr and Poorti API endpoints

### Steps

**1. Clone this repository**
```bash
git clone https://github.com/<your-username>/Order-fulfillment-automation.git
cd Order-fulfillment-automation
```

**2. Import the collection**
- Open Postman → **Import** → select `fulfillment-lifecycle.json`

**3. Import the environment**
- Open Postman → **Import** → select `fulfillment-environment.json`
- Select **Fulfillment Environment** from the environment dropdown (top-right)

**4. Configure environment variables**

Fill in all **Manual** variables in the environment — especially `baseSolrUrl`, `basePoortiUrl`, and `bearer_token`.

**5. Run the collection**
- Right-click `Fulfillment Lifecycle` in the sidebar → **Run collection**
- Confirm request order: Find Open Orders → Pick → Pack → Ship
- Click **Run Fulfillment Lifecycle**

---

## Edge Cases Handled

| Edge Case | How It's Handled |
|-----------|-----------------|
| Order already has an in-progress shipment | Pick detects the existing `shipmentId` and uses `postman.setNextRequest("Pack")` to skip directly to Pack |
| Multiple ship groups per order | After Ship completes, `groupIndex` is incremented and the collection loops back to Pick automatically |
| No open orders found | Collection halts gracefully after Find Open Orders with no further requests executed |

---

## Repository Structure

```
Order-fulfillment-automation/
├── fulfillment-lifecycle.json       # Postman collection export (v2.1)
├── fulfillment-environment.json     # Postman environment export
└── README.md                        # This file
```

---

## Authentication

All requests use Bearer token authentication via the `bearer_token` environment variable:

```
Authorization: Bearer {{bearer_token}}
```

Use Postman's **secret** variable type for `bearer_token` to prevent it from appearing in logs or shared exports.

---

*Built with [Postman](https://www.postman.com/)*
