# Non-Intrusive Load Monitoring (NILM)

## 1.1 General Information and Purpose
- **Service Name/Title:** `nilm`
- **Description and purpose:** The NILM service disaggregates aggregate household energy consumption into device-level usage profiles without intrusive hardware or per-device meters. It applies advanced ML models (DAEs, clustering-based architectures, deep sequence models) to identify appliance usage patterns from whole-home smart meter data. Supports real-time energy disaggregation, flexibility estimation, incentive targeting, and user feedback.
- **Owner/Contact Information:** ICCS

---

## 1.2 Functional Requirements
1. Ingest real-time household energy data from smart meters (e.g., Shelly 3EM).  
2. Disaggregate data into device-level consumption (washing machine, fridge, dishwasher, etc.).  
3. Support learning for multiple appliances via clustering strategies.  
4. Provide ON/OFF state classification and power reconstruction.  
5. Handle variable usage patterns and low-duty-cycle devices.  
6. Operate on 15-minute intervals aligned with MTUs.  
7. Deliver real-time disaggregated profiles for flexibility estimation.  
8. Integrate with other services via Kafka or REST APIs.  
9. Allow model updating/refinement with pilot data.  
10. Provide confidence metrics (e.g., F1-score, recall).  

---

## 1.3 Non-Functional Requirements
- Latency < 5s per MTU.  
- Minimum F1-score ≥ 0.8 on high-impact appliances.  
- Maintain privacy: only aggregate signals processed.  
- Availability ≥ 99.5%.  
- Support offline retraining + fallback models.  

---

## 1.4 Service Interfaces

### 1.4.1 API Endpoints

#### Endpoint 1 — Train NILM Model
- **URL:** `/nilm/train`  
- **Method:** `POST`  
- **Description:** Trains/retrains NILM models with aggregate data and appliance profiles.  
- **Headers:** Content-Type, Authorization.  

**Request Example**
```json
{
  "start_time": "2024-01-01T00:00:00Z",
  "end_time": "2024-06-01T00:00:00Z",
  "customer_id": "CUST456",
  "appliance_list": ["dishwasher", "washing_machine"]
}
```

**Response Example**
```json
{
  "message": "Model training initiated successfully",
  "model_id": "NILM_MODEL_001",
  "status": "queued",
  "timestamp": "2025-06-27T10:30:00Z"
}
```

**Error Handling**
```json
400 Bad Request: { "error": "Invalid input parameters" }
401 Unauthorized: { "error": "Unauthorized", "message": "Invalid token" }
500 Internal Server Error: { "error": "Training failed" }
```

---

#### Endpoint 2 — Get Disaggregated Loads
- **URL:** `/nilm/disaggregation`  
- **Method:** `POST`  
- **Description:** Returns estimated device-level consumption from aggregated data.  

**Request Example**
```json
{
  "customer_id": 102,
  "start_time": "2025-06-20T00:00:00Z",
  "end_time": "2025-06-20T01:00:00Z",
  "aggregation_interval": "1min"
}
```

**Response Example**
```json
{
  "customer_id": 102,
  "appliances": {
    "fridge": [0.12, 0.11, 0.13],
    "washing_machine": [0.0, 0.0, 0.4],
    "dishwasher": [0.0, 0.5, 0.0]
  }
}
```

**Error Handling**
```json
400 Bad Request: { "error": "Invalid format" }
401 Unauthorized: { "error": "Unauthorized", "message": "Invalid token" }
```

---

### 1.4.2 UI Mock-ups
*(Not applicable – backend service)*  

---

## 1.5 Data Model

### Inputs
- Aggregate household consumption (1s–1min).  
- Appliance-level consumption (for training/validation).  
- Optional appliance labels and metadata.  

### Outputs
- Time-series device-level disaggregated power (W).  
- ON/OFF state timelines.  

### Entities & Relationships
- **house_metadata:** static house info (address, area, year, energy_class).  
- **household_consumption:** whole-home energy usage per timestamp.  
- **device_metadata:** details per device (name, ip, device_id, relay, is_em).  
- **device_consumption:** actual device-level data for validation/training.  
- **disaggregated_output:** inferred usage from NILM models, linked to house + device.  

---

## 1.6 Integration and Dependencies

### Roles
- **Aggregator:** Integrates metering, disaggregated consumption, forecasts → flexibility bids.  
- **Market Operator (HENEX):** Indirectly depends on forecast-informed bids.  

### Domains
- **Forecasting Zone:** Regional forecasting.  
- **Metering Points:** Provide real-time & historical usage.  
- **Disaggregated Output:** Provides appliance-level usage.  

---

## 1.7 Security and Privacy
- Encrypted channels (TLS/HTTPS, MQTT with certs).  
- Anonymized datasets for training.  
- No PII stored or shared.  

---

### Initial Development
- Baseline DAEs tested on UK-DALE, REDD datasets.  
- Limitations: poor generalization for rare/low-usage devices.  

### Clustering-Enhanced Models
- Introduced **C-DAE framework** (clustering + DAE).  
- Improved classification for dishwashers, washing machines, washer-dryers.  
- Pilot retraining on Greek households.  

### Results
| Appliance | Model         | Recall | Precision | Accuracy | F1 | RMSE (W) |
|-----------|---------------|--------|-----------|----------|----|----------|
| Dishwasher | Baseline DAE | 0.58   | 0.13      | 0.79     | 0.21 | 289 |
|            | C-DAE (k=4)  | 0.99   | 0.95      | 0.95     | 0.97 | 205 |
| Washing Machine | Baseline DAE | 0.87 | 0.14 | 0.57 | 0.25 | 484 |
|                 | C-DAE (k=2) | 0.98 | 0.96 | 0.95 | 0.97 | 264 |
| Washer Dryer | Baseline DAE | 0.09 | 0.97 | 0.11 | 0.16 | 234 |
|              | C-DAE (k=2) | 0.96 | 0.97 | 0.93 | 0.96 | 218 |

**Key Improvements:**  
- F1-score jumps (e.g., dishwasher 0.21 → 0.97).  
- RMSE reduced (washing machine: −45%).  
- Better ON/OFF classification + noise resilience.  

**Impact:**  
- Enables real-time detection of controllable loads.  
- Improves customer profiling + incentive targeting.  
- Provides accurate input for flexibility optimization and bidding.  
