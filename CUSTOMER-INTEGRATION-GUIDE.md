# Device Data Flow Technical Documentation

## Overview
This document maps the complete data flow from when a device record is received via AWS SQS through database storage and finally to Web Portal display. It includes data transformation stages, field mappings, endpoint conventions, and term/label changes throughout the pipeline.

**Note**: This documentation covers the **Wireless API** system. The complete end-to-end flow begins with devices sending raw data to the **Async Socket Server** (a separate Python TCP server), which parses and reformats the data before pushing it to SQS - [Async Socket Server Data Flow](https://github.com/Smart-Grid-Solutions/Async-Socket-Server/blob/main/DATA_FLOW_DOCUMENTATION.md). This document starts from the point where data arrives in the SQS queue.

## System Context

This system processes data from fault indicators that periodically send status updates ("check-ins") containing:
- Device status information (battery levels, GPS coordinates, fault states)
- Electrical current measurements
- Event logs (faults, resets, etc.)

**Complete Data Flow (End-to-End):**
1. **Devices** → Send raw TCP data to **Async Socket Server** (Python TCP server)
2. **Async Socket Server** → Parses raw device data, reformats it, and pushes JSON messages to **AWS SQS Queue**
   - Repository: [Async Socket Server](https://github.com/Smart-Grid-Solutions/Async-Socket-Server)
   - Data Flow Documentation: [Async Socket Server Data Flow](https://github.com/Smart-Grid-Solutions/Async-Socket-Server/blob/main/DATA_FLOW_DOCUMENTATION.md)
3. **Wireless API** (this system) → Receives messages from SQS, processes and enriches data, stores in database
4. **Web Portal** → Queries database and displays formatted data to users

**Key Concepts:**
- **Device Check-in**: A periodic message sent by a device containing its current status and measurements
- **IMEI**: International Mobile Equipment Identity - a unique identifier for each device (similar to a serial number)
- **Historical Preservation**: Device information (name, location) is captured at check-in time so that if these values change later, we can still see what they were when each check-in occurred
- **Async Socket Server**: The Python TCP server that receives raw device data, parses it, and formats it into JSON before sending to SQS. By the time data reaches this system (Wireless API), it has already been parsed and structured.

---

## Glossary

**Acronyms and Terms:**
- **Async Socket Server**: Python TCP server that receives raw device data, parses it, and formats it into JSON before sending to SQS. See [GitHub repository](https://github.com/Smart-Grid-Solutions/Async-Socket-Server) and [Data Flow Documentation](https://github.com/Smart-Grid-Solutions/Async-Socket-Server/blob/main/DATA_FLOW_DOCUMENTATION.md)
- **SQS** (AWS Simple Queue Service): Amazon's message queue service that receives formatted JSON messages from the Async Socket Server
- **IMEI**: International Mobile Equipment Identity - unique device identifier
- **DM1**: A specific type of three-phase fault indicator device
- **DNP3**: Distributed Network Protocol 3 - a communication protocol used in electrical grid systems
- **RSSI**: Received Signal Strength Indicator - measures cellular signal strength
- **SOTF**: Switch Onto Fault - a type of electrical fault event
- **PFault**: Permanent Fault - a persistent electrical fault
- **TFault**: Temporary Fault - a transient electrical fault
- **Ongoing_fault**: A numeric code indicating the current fault state of the device (0 = no fault, higher numbers = different fault types)
- **Enrichment**: The process of adding additional data from the `device_information` table to the incoming device check-in data

**Database Tables:**
- **device_information**: Master table containing device configuration and metadata (name, location, organization, etc.)
- **device_records**: Historical log of all device check-ins (note: despite the name, these are actually device logs, not device records)
- **event_records**: Log of fault events and other significant occurrences from standard devices
- **dm1_event_records**: Log of fault events from DM1 devices (has additional phase-specific fields)
- **device_currents**: Historical electrical current measurements (hourly averages, peak currents)

---

## 1. Data Flow Block Diagram

**Note**: This diagram shows the flow starting from the SQS queue. The complete flow begins with devices sending raw TCP data to the Async Socket Server, which parses and formats the data before pushing to SQS.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Electrical Grid Monitoring Devices                   │
│                    (Fault Indicators)                                    │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Raw TCP Data
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Async Socket Server (Python TCP Server)                    │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Receives raw device data → Parses → Reformats → JSON            │  │
│  │  Repository: https://github.com/Smart-Grid-Solutions/             │  │
│  │              Async-Socket-Server                                  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Formatted JSON Message
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         AWS SQS Queue                                    │
│                  (DEVICE_DATA_SQS_QUEUE_URL)                             │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ JSON Message Body
                             │ (Already parsed and formatted)
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    SQS Service (sqs.service.ts)                         │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  listenForMessages() → receiveMessage() → saveToDatabase()       │  │
│  └──────────────────────────────────────────────────────────────────┘  │
│                                                                          │
│  Transformation Stages:                                                  │
│  1. Parse JSON message body (already formatted by Async Socket Server)  │
│  2. Model replacement (MODEL_REPLACEMENTS map)                          │
│  3. Enrich with DeviceInformation (CurrentName, CurrentLat, CurrentLong) │
│  4. Filter new events (filterNewEvents)                                 │
│  5. Compute ActualOngoingFault                                          │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Transformed Data
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    Database Service (db.service.ts)                     │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  createDeviceRecord() → DeviceRecord                            │  │
│  │  bulkCreateEventRecords() → EventRecord/DM1EventRecord          │  │
│  │  createDeviceCurrents() → DeviceCurrents                        │  │
│  │  computeActualOngoingFault() → ActualOngoingFault calculation  │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Stored Records
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    PostgreSQL Database                                  │
│  ┌──────────────────────┐  ┌──────────────────────┐                  │
│  │  device_records      │  │  event_records        │                  │
│  │  (Table)            │  │  (Table)              │                  │
│  └──────────────────────┘  └──────────────────────┘                  │
│  ┌──────────────────────┐  ┌──────────────────────┐                  │
│  │  dm1_event_records   │  │  device_currents     │                  │
│  │  (Table)            │  │  (Table)              │                  │
│  └──────────────────────┘  └──────────────────────┘                  │
│  ┌──────────────────────┐                                             │
│  │  device_information  │                                             │
│  │  (Table)            │                                             │
│  └──────────────────────┘                                             │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Query Results
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Device Records Service (device-records.service.ts)        │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  Data Formatting & Transformation:                               │  │
│  │  - Timezone conversion (UTC → User Timezone)                     │  │
│  │  - Enum value mapping (Ongoing_fault, State, FaultType)          │  │
│  │  - Field masking (MaskFaultDetails)                               │  │
│  │  - Date formatting (YYYY-MMM-DD hh:mm A)                        │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ Formatted Data
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│              Device Records Controller (device-records.controller.ts)  │
│  ┌──────────────────────────────────────────────────────────────────┐  │
│  │  REST API Endpoints:                                             │  │
│  │  GET /device-records/data-log                                     │  │
│  │  GET /device-records/device-information/data-log/:imeiNumber     │  │
│  │  GET /device-records/device-information/events/:imeiNumber       │  │
│  │  GET /device-records/device-information/load-history/:imeiNumber │  │
│  │  GET /device-records/critical-data                               │  │
│  │  GET /device-records/map-data                                     │  │
│  └──────────────────────────────────────────────────────────────────┘  │
└────────────────────────────┬────────────────────────────────────────────┘
                             │
                             │ JSON Response
                             ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                         Web Portal                                      │
│                    (Frontend Application)                               │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 2. SQS Message Structure

### Incoming Message Format
```json
{
  "device_data": {
    "IMEINumber": "string",
    "model": "string",
    "version": "string",
    "date": "ISO8601 timestamp",
    "rssi": number,
    "current_status": number,
    "master": number,
    "statusPacket": "string",
    "eventPacket": "string",
    "LEDDuration": number,
    "LEDTimer": number,
    "BatteryMCU": number,
    "BatteryLED": number,
    "State": number,
    "FaultType": number,
    "BatteryOrLine": number,
    "FastMode": number,
    "Calibrated": number,
    "AverageCurrent1m": number,
    "PrefaultCurrent": number,
    "AverageCurrent1h": number,
    "PeakCurrent24h": number,
    "Tempfault_Reset_Hours": "string",
    "Permfault_Reset_Hours": number,
    "SOTF": number,
    "Brightness": "string",
    "Current_Reset": "string",
    "Block_Reset": number,
    "Ongoing_fault": number,
    "channel": number,
    "gps_long": number,
    "gps_lat": number,
    "Program": "string",
    "ConnectTime": number,
    "Temperature": number
  },
  "events": [
    {
      "EventID": number,
      "EventTimestamp": "ISO8601 timestamp",
      "EventType": "string",
      "PrefaultCurrent": number,
      "FaultCurrent": number,
      "RecoverCurrent": number,
      "PhaseAEvent": "string",  // DM1 devices only
      "PhaseBEvent": "string",  // DM1 devices only
      "PhaseCEvent": "string"   // DM1 devices only
    }
  ],
  "currents": {
    "PeakCurrent": number,
    "PeakCurrentTimestamp": "ISO8601 timestamp",
    "HourlyAverages": [number]  // Array of 24 hourly averages
  },
  "dm1_device": boolean
}
```

---

## 3. Field Mappings: SQS → Database → API Response

### 3.1 Device Record Fields

**Transformation Stage Legend:**
- **SQS → DB**: Transformation happens during SQS message processing (before database save)
- **DB → API**: Transformation happens during API response formatting (when querying from database)
- **SQS → DB → API**: Transformation happens at both stages

**Table Column Explanation:**
- **SQS Message Field**: Field name and source in the incoming SQS message
- **Database Field**: Field name as stored in the `device_records` table
- **API Response Field**: Field name in the API response sent to the Web Portal
- **Transformation Notes**: Explicitly shows what happens at each stage (SQS → DB and/or DB → API)

| SQS Message Field | Database Field (device_records) | API Response Field | Transformation Notes |
|-------------------|--------------------------------|-------------------|---------------------|
| `device_data.IMEINumber` | `IMEINumber` | `IMEINumber` | **SQS → DB**: Direct mapping |
| `device_data.model` | `model` | `model` | **SQS → DB**: Model replacement applied via `MODEL_REPLACEMENTS` map |
| `device_data.version` | `version` | `version` | **SQS → DB**: Direct mapping |
| `device_data.date` | `date` | `date` | **SQS → DB**: Direct mapping (value: ISO8601 timestamp, stored as UTC)<br>**DB → API**: Timezone conversion (UTC → User Timezone) + format conversion (value: `YYYY-MMM-DD hh:mm A` formatted string) |
| `device_data.rssi` | `rssi` | `rssi` | **SQS → DB**: Direct mapping |
| `device_data.current_status` | `current_status` | `current_status` | **SQS → DB**: Direct mapping |
| `device_data.master` | `master` | `master` | **SQS → DB**: Direct mapping |
| `device_data.statusPacket` | `statusPacket` | `statusPacket` | **SQS → DB**: Direct mapping |
| `device_data.eventPacket` | `eventPacket` | `eventPacket` | **SQS → DB**: Direct mapping |
| `device_data.LEDDuration` | `LEDDuration` | `LEDDuration` | **SQS → DB**: Direct mapping |
| `device_data.LEDTimer` | `LEDTimer` | `LEDTimer` | **SQS → DB**: Direct mapping |
| `device_data.BatteryMCU` | `BatteryMCU` | `BatteryMCU` | **SQS → DB**: Direct mapping |
| `device_data.BatteryLED` | `BatteryLED` | `BatteryLED` | **SQS → DB**: Direct mapping |
| `device_data.State` | `State` | `State` | **SQS → DB**: Direct mapping (value: numeric code)<br>**DB → API**: Enum mapping via `getStateValue()` (value: human-readable string) |
| `device_data.FaultType` | `FaultType` | `FaultType` | **SQS → DB**: Direct mapping (value: numeric code)<br>**DB → API**: Enum mapping via `getFaultTypeValue()` (value: human-readable string) |
| `device_data.BatteryOrLine` | `BatteryOrLine` | `BatteryOrLine` | **SQS → DB**: Direct mapping (0=Line, 1=Battery) |
| `device_data.FastMode` | `FastMode` | `FastMode` | **SQS → DB**: Direct mapping |
| `device_data.Calibrated` | `Calibrated` | `Calibrated` | **SQS → DB**: Direct mapping |
| `device_data.AverageCurrent1m` | `AverageCurrent1m` | `AverageCurrent1m` | **SQS → DB**: Direct mapping |
| `device_data.PrefaultCurrent` | `PrefaultCurrent` | `PrefaultCurrent` | **SQS → DB**: Direct mapping |
| `device_data.AverageCurrent1h` | `AverageCurrent1h` | `AverageCurrent1h` | **SQS → DB**: Direct mapping |
| `device_data.PeakCurrent24h` | `PeakCurrent24h` | `PeakCurrent24h` | **SQS → DB**: Direct mapping |
| `device_data.Tempfault_Reset_Hours` | `Tempfault_Reset_Hours` | `Tempfault_Reset_Hours` | **SQS → DB**: Direct mapping |
| `device_data.Permfault_Reset_Hours` | `Permfault_Reset_Hours` | `Permfault_Reset_Hours` | **SQS → DB**: Direct mapping |
| `device_data.SOTF` | `SOTF` | `SOTF` | **SQS → DB**: Direct mapping |
| `device_data.Brightness` | `Brightness` | `Brightness` | **SQS → DB**: Direct mapping |
| `device_data.Current_Reset` | `Current_Reset` | `Current_Reset` | **SQS → DB**: Direct mapping |
| `device_data.Block_Reset` | `Block_Reset` | `Block_Reset` | **SQS → DB**: Direct mapping |
| `device_data.Ongoing_fault` | `Ongoing_fault` | `Ongoing_fault` | **SQS → DB**: Computed via `computeActualOngoingFault()` (value: numeric code, may differ from device-reported value)<br>**DB → API**: Enum mapping via `getOngoingFaultValue()` (value: human-readable string, or "None"/"Fault" if masked when `MaskFaultDetails = true`) |
| N/A | `ActualOngoingFault` | `ActualOngoingFault` | **SQS → DB**: Computed during save (value: numeric code, may differ from `Ongoing_fault`)<br>**DB → API**: Enum mapping via `getOngoingFaultValue()` (value: human-readable string, or "None"/"Fault" if masked when `MaskFaultDetails = true`) |
| `device_data.channel` | `channel` | `channel` | **SQS → DB**: Direct mapping |
| `device_data.gps_long` | `gps_long` | `gps_long` | **SQS → DB**: Direct mapping |
| `device_data.gps_lat` | `gps_lat` | `gps_lat` | **SQS → DB**: Direct mapping |
| `device_data.Program` | `Program` | `Program` | **SQS → DB**: Direct mapping |
| `device_data.ConnectTime` | `ConnectTime` | `ConnectTime` | **SQS → DB**: Direct mapping |
| `device_data.Temperature` | `Temperature` | `Temperature` | **SQS → DB**: Direct mapping |
| N/A (Enriched) | `CurrentName` | `CurrentName` | **SQS → DB**: Enriched from `DeviceInformation.Real_Name` at check-in time (preserves historical device name) |
| N/A (Enriched) | `CurrentLat` | `CurrentLat` | **SQS → DB**: Enriched from `DeviceInformation.Lat` at check-in time (preserves historical device location) |
| N/A (Enriched) | `CurrentLong` | `CurrentLong` | **SQS → DB**: Enriched from `DeviceInformation.Long` at check-in time (preserves historical device location) |
| N/A | `RecordID` | `RecordID` | **SQS → DB**: Auto-generated primary key |
| N/A | `CreatedDate` | `CreatedDate` | **SQS → DB**: Auto-generated timestamp |

### 3.2 Event Record Fields

#### Standard Events (Non-DM1)

| SQS Message Field | Database Field (event_records) | API Response Field | Transformation Notes |
|-------------------|-------------------------------|-------------------|---------------------|
| `events[].EventID` | `EventID` | `EventID` | **SQS → DB**: Direct mapping |
| `events[].EventTimestamp` | `EventTimestamp` | `EventTimestamp` | **SQS → DB**: Direct mapping (value: ISO8601 timestamp, stored as UTC)<br>**DB → API**: Timezone conversion (UTC → User Timezone) + format conversion (value: `YYYY-MMM-DD hh:mm A` formatted string) |
| `events[].EventType` | `EventType` | `EventType` | **SQS → DB**: Direct mapping |
| `events[].PrefaultCurrent` | `PrefaultCurrent` | `PrefaultCurrent` | **SQS → DB**: Direct mapping (value: number)<br>**DB → API**: Conditional transformation - For "Current Reset" events, `FaultCurrent` value is moved to `PrefaultCurrent` (value: number, may be from different source field) |
| `events[].FaultCurrent` | `FaultCurrent` | `FaultCurrent` | **SQS → DB**: Direct mapping (value: number)<br>**DB → API**: Conditional transformation - Cleared (set to empty/null) for "Current Reset" events |
| `events[].RecoverCurrent` | `RecoverCurrent` | `RecoverCurrent` | **SQS → DB**: Direct mapping (value: number)<br>**DB → API**: Conditional transformation - Cleared (set to empty/null) for "SOTF" events |
| N/A (Added) | `PacketTimestamp` | `PacketTimestamp` | **SQS → DB**: Set from `device_data.date` (value: ISO8601 timestamp, stored as UTC)<br>**DB → API**: Timezone conversion (UTC → User Timezone) + format conversion (value: `YYYY-MMM-DD hh:mm A` formatted string) |
| N/A (Added) | `IMEINumber` | `IMEINumber` | **SQS → DB**: Set from `device_data.IMEINumber` |
| N/A (Added) | `DeviceRecordID` | `DeviceRecordID` | **SQS → DB**: Set from created `DeviceRecord.RecordID` |
| N/A | `Event_Record_ID` | `Event_Record_ID` | **SQS → DB**: Auto-generated primary key |
| N/A | N/A | `PermanentFaultResetCurrentThreshold` | **DB → API**: Computed in Device Records Service (for PFault/Current Reset events) |
| N/A | N/A | `PostEventRef` | **DB → API**: Computed in Device Records Service (for Current Reset, Fault Like, Fault <50 events) |

#### DM1 Events

| SQS Message Field | Database Field (dm1_event_records) | API Response Field | Transformation Notes |
|-------------------|-----------------------------------|-------------------|---------------------|
| `events[].EventID` | `EventID` | `EventID` | **SQS → DB**: Direct mapping |
| `events[].EventTimestamp` | `EventTimestamp` | `EventTimestamp` | **SQS → DB**: Direct mapping (value: ISO8601 timestamp, stored as UTC)<br>**DB → API**: Timezone conversion (UTC → User Timezone) + format conversion (value: `YYYY-MMM-DD hh:mm A` formatted string) |
| `events[].EventType` | `EventType` | `EventType` | **SQS → DB**: Direct mapping |
| `events[].PhaseAEvent` | `PhaseAEvent` | `PhaseAEvent` | **SQS → DB**: Direct mapping |
| `events[].PhaseBEvent` | `PhaseBEvent` | `PhaseBEvent` | **SQS → DB**: Direct mapping |
| `events[].PhaseCEvent` | `PhaseCEvent` | `PhaseCEvent` | **SQS → DB**: Direct mapping |
| N/A (Added) | `PacketTimestamp` | `PacketTimestamp` | **SQS → DB**: Set from `device_data.date` (value: ISO8601 timestamp, stored as UTC)<br>**DB → API**: Timezone conversion (UTC → User Timezone) + format conversion (value: `YYYY-MMM-DD hh:mm A` formatted string) |
| N/A (Added) | `IMEINumber` | `IMEINumber` | **SQS → DB**: Set from `device_data.IMEINumber` |
| N/A (Added) | `DeviceRecordID` | `DeviceRecordID` | **SQS → DB**: Set from created `DeviceRecord.RecordID` |
| N/A | `Event_Record_ID` | `Event_Record_ID` | **SQS → DB**: Auto-generated primary key |

### 3.3 Device Currents Fields

| SQS Message Field | Database Field (device_currents) | API Response Field | Transformation Notes |
|-------------------|----------------------------------|-------------------|---------------------|
| `currents.PeakCurrent` | `PeakCurrent` | `PeakCurrent` | **SQS → DB**: Direct mapping |
| `currents.PeakCurrentTimestamp` | `PeakCurrentTimestamp` | `PeakCurrentTimestamp` | **SQS → DB**: Direct mapping |
| `currents.HourlyAverages[0]` | `AverageCurrentH0` | `AverageCurrentH0` | **SQS → DB**: Array → Column transformation (HourlyAverages array indexed to individual columns) |
| `currents.HourlyAverages[1]` | `AverageCurrentH1` | `AverageCurrentH1` | **SQS → DB**: Array → Column transformation |
| ... | ... | ... | (H0 through H23) |
| `currents.HourlyAverages[23]` | `AverageCurrentH23` | `AverageCurrentH23` | **SQS → DB**: Array → Column transformation |
| N/A (Added) | `PacketTimestamp` | `PacketTimestamp` | **SQS → DB**: Set from `device_data.date` |
| N/A (Added) | `IMEINumber` | `IMEINumber` | **SQS → DB**: Set from `device_data.IMEINumber` |
| N/A (Enriched) | `CurrentName` | `CurrentName` | **SQS → DB**: Enriched from `DeviceInformation.Real_Name` (via `device_data.CurrentName` which was enriched earlier). This preserves the device name at check-in time, even if the device name is later changed in `device_information` table. |

---

## 4. Data Transformation Stages

### Stage 1: SQS Message Reception

**Method**: `saveToDatabase(message)` in SQS Service

1. **JSON Parsing**
   - Input: Raw SQS message body (string)
   - Output: Parsed JSON object

2. **Model Replacement**
   - **Purpose**: Standardize device model names (older model names are mapped to newer standardized names)
   - **Mapping**: `MODEL_REPLACEMENTS` constant
   - **Example**: `'FI-5A 3-C04TRB-X-1(80)EU'` → `'FI-5A E04TRB-X'`
   - **Why**: Devices may report legacy model names, but the system uses standardized names for consistency

3. **Device Information Enrichment**
   - **Query**: `getDeviceInfoByImei(IMEINumber)`
   - **Purpose**: Capture device information values at check-in time to preserve historical data
   - **Added Fields**:
     - `CurrentName` ← `DeviceInformation.Real_Name` (preserves device name at check-in time)
     - `CurrentLat` ← `DeviceInformation.Lat` (preserves device location at check-in time)
     - `CurrentLong` ← `DeviceInformation.Long` (preserves device location at check-in time)
   - **Note**: These enriched fields are saved to both `device_records` and `device_currents` tables. If a user later changes the device name or location in `device_information`, the historical values remain preserved in the check-in records.

4. **Event Filtering**
   - **Method**: `filterNewEvents()`
   - **Purpose**: Remove duplicate events to prevent storing the same event multiple times
   - **How it works**: 
     - Compares incoming events with the most recent event in the database
     - Only keeps events that are truly new (higher EventID) or reset events that occurred more than 5 minutes after the last event
     - This prevents duplicate storage if a device resends the same check-in data

5. **ActualOngoingFault Computation**
   - **Method**: `computeActualOngoingFault()`
   - **Purpose**: Enhance the device-reported fault status by analyzing events and historical data
   - **Why**: Devices may report `Ongoing_fault = 0` (no fault) even when events indicate a fault exists
   - **How it works**: 
     - Starts with the device-reported `Ongoing_fault` value
     - Analyzes new events and previous check-in data
     - May override the device value if events indicate a fault
   - **Special Cases**:
     - `PROBABLE_TEMP_FAULT (11)`: Permanent faults reported by device, but previous current readings were very low (suggests it might actually be temporary)
     - `CURRENT_RESET (10)`: Device reports no fault, but a "Current Reset" event occurred (indicates a fault was reset)
     - `OVER_CURRENT (6)`: Device reports no fault, but an "Overcurrent" event occurred
     - `PERMANENT (4)`: DM1 device reports no fault, but a "FAULT_INITIAL" event occurred

### Stage 2: Database Storage

**Component**: Database Service

1. **Device Record Creation**
   - **Method**: `createDeviceRecord(recordData, latestRecord)`
   - **Actions**:
     - Duplicate check (compares with latest record)
     - Insert into `device_records` table
     - Update `device_information.LastCheckInTime`

2. **Event Record Creation**
   - **Method**: `bulkCreateEventRecords(eventsArray, dm1_device, deviceRecordID)`
   - **Actions**:
     - Route to `event_records` or `dm1_event_records` based on `dm1_device`
     - Add `DeviceRecordID` foreign key
     - Insert events sequentially

3. **Device Currents Creation**
   - **Method**: `createDeviceCurrents(currentsPacket, imeiNumber, packetTimestamp, currentName)`
   - **Actions**:
     - Transform `HourlyAverages` array to `AverageCurrentH0` through `AverageCurrentH23` columns
     - Set `CurrentName` from enriched `device_data.CurrentName` (which was enriched from `DeviceInformation.Real_Name`)
     - Insert into `device_currents` table
   - **Note**: The `CurrentName` parameter comes from the enriched `device_data.CurrentName` field, preserving the device name at check-in time

### Stage 3: API Response Formatting

**Component**: Device Records Service

1. **Timezone Conversion**
   - **Method**: `parseDateByUserTZ(obj, dateFields, user)`
   - **Transformation**: UTC timestamps → User timezone
   - **Format**: `YYYY-MMM-DD hh:mm A` (e.g., "2024-Jan-15 02:30 PM")

2. **Enum Value Mapping**
   - **Ongoing_fault**: `getOngoingFaultValue(value, model)`
     - Maps numeric values to human-readable strings
     - Special handling for DM1 devices
   - **State**: `getStateValue(value)`
   - **FaultType**: `getFaultTypeValue(value)`

3. **Fault Details Masking**
   - **Condition**: `user.MaskFaultDetails === true`
   - **Transformation**: 
     - `Ongoing_fault = 0` → `"None"`
     - `Ongoing_fault > 0` → `"Fault"`

4. **Event-Specific Transformations**
   - **PFault**: Calculate `PermanentFaultResetCurrentThreshold = ceil(PrefaultCurrent / 2)`, min 9
   - **Current Reset**: 
     - Move `FaultCurrent` → `PrefaultCurrent`
     - Clear `FaultCurrent`
     - Set `PostEventRef = "Reset Current (A)"`
   - **SOTF**: Clear `RecoverCurrent`
   - **Fault Like / Fault <50**: Set `PostEventRef = "Load 15s After Event (A)"`

---

## 5. Endpoint Naming Conventions

### 5.1 REST API Endpoints

| Endpoint Pattern | HTTP Method | Controller Method | Service Method | Purpose |
|-----------------|------------|------------------|---------------|---------|
| `/device-records/get-table-columns` | GET | `getTableColumns()` | `getTableColumns()` | Get database column metadata |
| `/device-records/critical-data` | GET | `getCriticalData()` | `getCriticalData()` | Get fault events with device info |
| `/device-records/map-data` | GET | `getMapData()` | `getMapData()` | Get device locations for map view |
| `/device-records/data-log` | GET | `getDataLog()` | `getDataLog()` | Get paginated device records |
| `/device-records/all-devices` | GET | `getAllDevices()` | `getAllDevices()` | Get all devices with filters |
| `/device-records/device-summary` | GET | `getDeviceSummary()` | `getDeviceSummary()` | Get device summary statistics |
| `/device-records/device-information/:imeiNumber` | GET | `getDeviceInformationById()` | `getDeviceInformationById()` | Get single device information |
| `/device-records/device-information/data-log/:imeiNumber` | GET | `getDeviceRecordsById()` | `getDataLogById()` | Get device records for specific IMEI |
| `/device-records/device-information/events/:imeiNumber` | GET | `getDeviceEventsById()` | `getDeviceEventsById()` | Get events for specific IMEI |
| `/device-records/device-information/load-history/:imeiNumber` | GET | `getDeviceLoadHistoryById()` | `getDeviceLoadHistoryById()` | Get load history (currents) for IMEI |

### 5.2 Naming Convention Patterns

1. **Resource Hierarchy**:
   - Base: `/device-records`
   - Nested: `/device-records/device-information/:imeiNumber`
   - Sub-resources: `/device-information/events/:imeiNumber`

2. **HTTP Methods**:
   - All endpoints use `GET` (read-only operations)

3. **Path Parameters**:
   - Use `:imeiNumber` for device identification
   - Use kebab-case for multi-word paths (`device-information`, `data-log`)

4. **Query Parameters**:
   - `page`: Pagination page number (0-indexed)
   - `limit`: Results per page
   - `from`: Start date filter (YYYYMMDD)
   - `to`: End date filter (YYYYMMDD)
   - `sortBy`: Sorting configuration (JSON string)
   - `filterBy`: Filtering configuration (nested object)

### 5.3 Database Table Naming

| Table Name | Model Class | Primary Key | Notes |
|-----------|------------|-------------|-------|
| `device_records` | `DeviceRecord` | `RecordID` | Note: These are device logs, not device records |
| `device_information` | `DeviceInformation` | `DeviceID` | Master device information |
| `event_records` | `EventRecord` | `Event_Record_ID` | Standard device events |
| `dm1_event_records` | `DM1EventRecord` | `Event_Record_ID` | DM1-specific events |
| `device_currents` | `DeviceCurrents` | Composite: `[PacketTimestamp, IMEINumber]` | Historical current data |

---

## 6. Term and Label Changes Through Pipeline

### 6.1 Field Name and Value Changes Through Pipeline

**Note**: This table shows how field names and values change as data flows through the system. The "Stage" column indicates where the transformation occurs.

| Stage | Source Field/Value | Destination Field/Value | Transformation Details |
|-------|------------------|------------------------|----------------------|
| **SQS → DB** | `device_data.date` | `date` | Field name extracted from nested object structure |
| **SQS → DB** | `device_data.IMEINumber` | `IMEINumber` | Field name extracted from nested object structure |
| **SQS → DB** (Enrichment) | `DeviceInformation.Real_Name` | `CurrentName` | Enriched from `device_information` table and added to `device_data` before database save. Also saved to `device_currents.CurrentName`. Preserves historical device name at check-in time. |
| **SQS → DB** (Enrichment) | `DeviceInformation.Lat` | `CurrentLat` | Enriched from `device_information` table and added to `device_data` before database save. Preserves historical device location at check-in time. |
| **SQS → DB** (Enrichment) | `DeviceInformation.Long` | `CurrentLong` | Enriched from `device_information` table and added to `device_data` before database save. Preserves historical device location at check-in time. |
| **DB → API** | `Ongoing_fault` (numeric: 0, 1, 2, etc.) | `Ongoing_fault` (string: "None", "Decision", "Temporary", etc.) | Enum value mapping via `getOngoingFaultValue()`. May be masked to "None" or "Fault" if `MaskFaultDetails = true`. |
| **DB → API** | `date` (UTC ISO timestamp) | `date` (User timezone formatted string) | Timezone conversion from UTC to user's timezone, then formatted as `YYYY-MMM-DD hh:mm A` |
| **DB → API** | `EventTimestamp` (UTC ISO timestamp) | `EventTimestamp` (User timezone formatted string) | Timezone conversion from UTC to user's timezone, then formatted as `YYYY-MMM-DD hh:mm A` |
| **DB → API** | `State` (numeric) | `State` (human-readable string) | Enum value mapping via `getStateValue()` |
| **DB → API** | `FaultType` (numeric) | `FaultType` (human-readable string) | Enum value mapping via `getFaultTypeValue()` |

### 6.2 Value Transformations

**Note**: This table shows specific value transformations. The "Stage" column indicates where each transformation occurs.

| Stage | Field | Input Value | Output Value | Transformation Details |
|-------|-------|------------|-------------|----------------------|
| **SQS → DB** | `model` | `'FI-5A 3-C04TRB-X-1(80)EU'` | `'FI-5A E04TRB-X'` | Model replacement map applied during SQS message processing |
| **DB → API** | `Ongoing_fault` | `0` (numeric) | `"None"` (string) | Enum mapping via `getOngoingFaultValue()` (if not masked) |
| **DB → API** | `Ongoing_fault` | `4` (numeric) | `"Permanent"` (string) | Enum mapping via `getOngoingFaultValue()` (if not masked) |
| **DB → API** | `Ongoing_fault` | `0` (numeric, masked) | `"None"` (string) | Masking applied when `MaskFaultDetails = true` |
| **DB → API** | `Ongoing_fault` | `>0` (numeric, masked) | `"Fault"` (string) | Masking applied when `MaskFaultDetails = true` |
| **DB → API** | `EventTimestamp` | `"2024-01-15T14:30:00.000Z"` (UTC ISO) | `"2024-Jan-15 02:30 PM"` (User TZ formatted) | Timezone conversion (UTC → User timezone) + format conversion |
| **DB → API** | `State` | `1` (numeric) | Human-readable string | Enum mapping via `getStateValue()` |
| **DB → API** | `FaultType` | `2` (numeric) | Human-readable string | Enum mapping via `getFaultTypeValue()` |
| **SQS → DNP3** | `BatteryOrLine` | `0` (numeric) | `"Line"` (string) | Enum mapping in DNP3 JSON output (not in API response) |
| **SQS → DNP3** | `BatteryOrLine` | `1` (numeric) | `"Battery"` (string) | Enum mapping in DNP3 JSON output (not in API response) |

### 6.3 Computed Field Additions

| Field | Source | Added At | Purpose |
|-------|--------|----------|---------|
| `ActualOngoingFault` | Computed from `Ongoing_fault` + events + previous record | **SQS → DB**: During database save | Enhanced fault detection that may differ from device-reported `Ongoing_fault` |
| `CurrentName` | `DeviceInformation.Real_Name` | **SQS → DB**: Before database save (enrichment stage) | Preserves device name at check-in time. Saved to both `device_records` and `device_currents` tables. |
| `CurrentLat` | `DeviceInformation.Lat` | **SQS → DB**: Before database save (enrichment stage) | Preserves device latitude at check-in time |
| `CurrentLong` | `DeviceInformation.Long` | **SQS → DB**: Before database save (enrichment stage) | Preserves device longitude at check-in time |
| `PermanentFaultResetCurrentThreshold` | Calculated from `PrefaultCurrent` or `FaultCurrent` | **DB → API**: During API response formatting | Display threshold value for PFault and Current Reset events |
| `PostEventRef` | Based on `EventType` | **DB → API**: During API response formatting | Display reference label for specific event types (Current Reset, Fault Like, Fault <50) |

### 6.4 Event-Specific Transformations

| Event Type | Field Changes | Implementation |
|-----------|--------------|----------------|
| **PFault** | `PermanentFaultResetCurrentThreshold = ceil(PrefaultCurrent / 2)` (min 9) | Device Records Service |
| **Current Reset** | `PrefaultCurrent = FaultCurrent`, `FaultCurrent = ''`, `PostEventRef = "Reset Current (A)"` | Device Records Service |
| **SOTF** | `RecoverCurrent = ''` | Device Records Service |
| **Fault Like / Fault <50** | `PostEventRef = "Load 15s After Event (A)"` | Device Records Service |
| **Fault <50 followed by SOTF/PFault/TFault** | `EventTimestamp` adjusted back 30 seconds (within 6 min window) | Database Service |

---

## 7. Data Flow Sequence Diagram

**Note**: This diagram shows the complete end-to-end flow. The Wireless API system starts processing from the SQS Queue step.

```
Device → Async Socket Server (Python TCP Server)
         │
         ├─→ Receive raw TCP data
         │   │
         │   ├─→ Parse raw device data
         │   │
         │   ├─→ Reformat to JSON structure
         │   │
         │   └─→ Push to AWS SQS Queue
         │
         └─→ SQS Queue
             │
             ├─→ SqsService.listenForMessages()
         │   │
         │   ├─→ receiveMessage()
         │   │   └─→ ReceiveMessageCommand (max 10 messages)
         │   │
         │   └─→ saveToDatabase(message)
         │       │
         │       ├─→ Parse JSON messageBody
         │       │
         │       ├─→ Apply MODEL_REPLACEMENTS
         │       │
         │       ├─→ dbService.getDeviceInfoByImei()
         │       │   └─→ Query DeviceInformation
         │       │
         │       ├─→ Enrich device_data:
         │       │   ├─→ CurrentName ← Real_Name
         │       │   ├─→ CurrentLat ← Lat
         │       │   └─→ CurrentLong ← Long
         │       │
         │       ├─→ Query previousDeviceRecord
         │       │
         │       ├─→ checkModelChangeAndAlert() (if applicable)
         │       │
         │       ├─→ dbService.filterNewEvents()
         │       │   └─→ Query existing events, filter duplicates
         │       │
         │       ├─→ dbService.computeActualOngoingFault()
         │       │   └─→ Calculate enhanced fault status
         │       │
         │       ├─→ dbService.createDeviceRecord()
         │       │   ├─→ Check for duplicates
         │       │   ├─→ Insert DeviceRecord
         │       │   └─→ Update DeviceInformation.LastCheckInTime
         │       │
         │       ├─→ dbService.bulkCreateEventRecords()
         │       │   └─→ Insert EventRecord or DM1EventRecord
         │       │
         │       ├─→ dbService.createDeviceCurrents()
         │       │   └─→ Transform HourlyAverages → Columns
         │       │
         │       ├─→ buildDNP3Json() (if Dnp3SqsTopicURL exists)
         │       │   └─→ Send to DNP3 SQS Topic
         │       │
         │       ├─→ sendMultispeakCheckinNotifications()
         │       │
         │       └─→ deleteMessage(receiptHandle)

Web Portal → API Request
              │
              ├─→ DeviceRecordsController (JWT Auth)
              │   │
              │   └─→ DeviceRecordsService
              │       │
              │       ├─→ dbService.getDeviceRecordsLog()
              │       │   └─→ Query with joins, filters, pagination
              │       │
              │       ├─→ Format response:
              │       │   ├─→ parseDateByUserTZ()
              │       │   ├─→ getOngoingFaultValue()
              │       │   ├─→ getStateValue()
              │       │   ├─→ getFaultTypeValue()
              │       │   └─→ Apply masking (if enabled)
              │       │
              │       └─→ Return JSON response
```

---

## 8. Key Constants and Enums

### 8.1 Ongoing Fault Values
```typescript
ONGOING_FAULT_VALUE = {
  NONE: 0,
  DECISION: 1,
  TEMPORARY: 2,
  OVER_CURRENT_NO_LOCAL_FAULT: 3,
  PERMANENT: 4,
  PERMANENT_ALT: 5,
  OVER_CURRENT: 6,
  CURRENT_RESET: 10,
  PROBABLE_TEMP_FAULT: 11,
  DM1_PORT_1_FAULT: 21,
  DM1_PORT_2_FAULT: 22,
  DM1_PORT_3_FAULT: 23,
  DISCONNECTED: 400,
  NO_DATA: 401,
  LOW_BATTERY: 402
}
```

### 8.2 Model Replacements
```typescript
MODEL_REPLACEMENTS = {
  'FI-5A 3-C04TRB-X-1(80)EU': 'FI-5A E04TRB-X',
  'FI-5A 3-C04TRB-Y-1(80)EU': 'FI-5A E04TRB-Y',
  'FI-5A 3-CS24TNB-Z-2-3U': 'FI-5A F24TNB-Y',
  // ... additional mappings
}
```

---

## 9. Database Relationships

### 9.1 Core Device Data Relationships

```
device_information (1) ──< (many) device_records
    │                        │
    │                        │ (1) ──< (many) event_records
    │                        │
    │                        │ (1) ──< (many) dm1_event_records
    │                        │
    │                        └── (related via) device_currents
    │                            (IMEINumber + PacketTimestamp/date)
    │
    ├── (1) ──< (many) group_device_records ──> (many) group_records
    │
    ├── (many) ──> (1) organization_records
    │
    └── (many) ──> (1) power_lines
```

### 9.2 Relationship Details

**Primary Relationships (Defined in Objection.js Models):**

1. **device_information** ↔ **device_records**
   - **Type**: One-to-Many (via IMEINumber)
   - **Join**: `device_records.IMEINumber` = `device_information.IMEINumber`
   - **Model Relation**: `DeviceRecord.deviceInformation` (HasOneRelation)
   - **Purpose**: Links device check-in logs to master device information

2. **device_records** ↔ **event_records**
   - **Type**: One-to-Many
   - **Join**: `event_records.DeviceRecordID` = `device_records.RecordID`
   - **Model Relation**: `EventRecord.DeviceRecord` (BelongsToOneRelation)
   - **Additional**: `event_records.IMEINumber` also exists for direct device lookup
   - **Purpose**: Links fault events to the specific device check-in that reported them

3. **device_records** ↔ **dm1_event_records**
   - **Type**: One-to-Many
   - **Join**: `dm1_event_records.DeviceRecordID` = `device_records.RecordID`
   - **Model Relation**: `DM1EventRecord.DeviceRecord` (BelongsToOneRelation)
   - **Additional**: `dm1_event_records.IMEINumber` also exists for direct device lookup
   - **Purpose**: Links DM1-specific fault events to the device check-in

4. **device_information** ↔ **device_currents**
   - **Type**: One-to-Many
   - **Join**: `device_currents.IMEINumber` = `device_information.IMEINumber`
   - **Usage**: Used in queries via explicit joins (see [`db.service.ts`](src/db/db.service.ts))
   - **Purpose**: Links historical current measurements to devices

5. **device_records** ↔ **device_currents**
   - **Type**: One-to-One
   - **Join**: `device_currents.IMEINumber` = `device_records.IMEINumber` AND `device_currents.PacketTimestamp` = `device_records.date`
   - **Usage**: Used in queries via explicit joins (see [`db.service.ts`](src/db/db.service.ts))
   - **Purpose**: Links current measurements to the specific check-in that reported them

6. **device_information** ↔ **group_device_records**
   - **Type**: One-to-Many
   - **Join**: `group_device_records.DeviceID` = `device_information.DeviceID`
   - **Model Relation**: `DeviceInformation.groupsDevices` (HasManyRelation)
   - **Purpose**: Links devices to groups (many-to-many relationship)

7. **group_device_records** ↔ **group_records**
   - **Type**: Many-to-One
   - **Join**: `group_device_records.GroupID` = `group_records.GroupID`
   - **Model Relation**: `GroupDeviceRecord.group` (HasOneRelation)
   - **Purpose**: Links device-group associations to groups

8. **device_information** ↔ **group_records** (Many-to-Many)
   - **Type**: Many-to-Many (via group_device_records)
   - **Join**: Through `group_device_records` table
   - **Model Relation**: `DeviceInformation.groups` (ManyToManyRelation)
   - **Purpose**: Devices can belong to multiple groups, groups can contain multiple devices

9. **device_information** ↔ **organization_records**
   - **Type**: Many-to-One
   - **Join**: `device_information.OrganizationID` = `organization_records.OrganizationID`
   - **Model Relation**: `DeviceInformation.organization` (HasOneRelation)
   - **Purpose**: Links devices to their organization

10. **device_information** ↔ **power_lines**
    - **Type**: Many-to-One
    - **Join**: `device_information.PowerLineID` = `power_lines.Id`
    - **Model Relation**: `DeviceInformation.powerLine` (BelongsToOneRelation)
    - **Purpose**: Links devices to power lines they monitor

**Soft Delete Relationships:**
- `device_information.DeletedBy` → `user_records.UserID` (HasOneRelation)
- `organization_records.DeletedBy` → `user_records.UserID` (HasOneRelation)
- `group_records.DeletedBy` → `user_records.UserID` (HasOneRelation)
- `power_lines.DeletedBy` → `user_records.UserID` (HasOneRelation)

### 9.3 Key Relationship Columns

| Relationship | Foreign Key Column | References Table | References Column | Notes |
|-------------|-------------------|----------------|------------------|-------|
| device_records → device_information | `IMEINumber` | `device_information` | `IMEINumber` | Primary lookup key |
| event_records → device_records | `DeviceRecordID` | `device_records` | `RecordID` | Links event to check-in |
| event_records → device_information | `IMEINumber` | `device_information` | `IMEINumber` | Direct device lookup |
| dm1_event_records → device_records | `DeviceRecordID` | `device_records` | `RecordID` | Links DM1 event to check-in |
| dm1_event_records → device_information | `IMEINumber` | `device_information` | `IMEINumber` | Direct device lookup |
| device_currents → device_information | `IMEINumber` | `device_information` | `IMEINumber` | Used in joins |
| device_currents → device_records | `IMEINumber` + `PacketTimestamp` | `device_records` | `IMEINumber` + `date` | Composite join condition |
| group_device_records → device_information | `DeviceID` | `device_information` | `DeviceID` | Links device to group |
| group_device_records → group_records | `GroupID` | `group_records` | `GroupID` | Links group to device |
| device_information → organization_records | `OrganizationID` | `organization_records` | `OrganizationID` | Device's organization |
| device_information → power_lines | `PowerLineID` | `power_lines` | `Id` | Device's power line |

---

## 10. Error Handling and Edge Cases

### 10.1 Duplicate Detection
- **Method**: `createDeviceRecord()` in Database Service
- **Logic**: Compares new record with latest record
- **Action**: Discards duplicate, returns empty DeviceRecord

### 10.2 Event Filtering Edge Cases
- **Reset Events**: Events with `EventID = 0` or `1` are accepted if >5 minutes after last event
- **Fault <50 Adjustment**: Subsequent faults within 6 minutes are adjusted back 30 seconds

### 10.3 Missing Data Handling
- **No DeviceInformation**: If `DeviceInformation` is not found for the IMEI, the enrichment step is skipped. `CurrentName`, `CurrentLat`, and `CurrentLong` remain undefined/null in both `device_records` and `device_currents` tables.
- **No Events**: Empty events array is handled gracefully - no event records are created, but the device record is still saved.
- **No Currents**: Empty currents object creates an empty `DeviceCurrents` instance (no database record is created). The `CurrentName` field will not be saved to `device_currents` in this case.

### 10.4 Timezone Handling
- **Storage**: All timestamps stored in UTC
- **Display**: Converted to user's timezone in service layer
- **Format**: `YYYY-MMM-DD hh:mm A` (e.g., "2024-Jan-15 02:30 PM")

---

## 11. Performance Considerations

1. **SQS Polling**: 100ms interval between polls
2. **Batch Processing**: Up to 10 messages per SQS receive
3. **Database Queries**: 
   - Previous record query reused to avoid duplicate queries
   - Event filtering uses efficient MAX queries
4. **Pagination**: Default limits applied to prevent large result sets

---

## 12. Security and Access Control

1. **JWT Authentication**: All API endpoints require valid JWT token
2. **IMEI Validation**: `ValidationService.validateImeiNumber()` checks user access
3. **Group-Based Access**: Users can only access devices in their groups
4. **Soft-Delete Filtering**: Deleted devices filtered out at query level
5. **MaskFaultDetails**: Organization-level setting to mask fault details

---

## Appendix A: File Locations

| Component | File Path |
|-----------|-----------|
| SQS Service | [`src/sqs.service.ts`](src/sqs.service.ts) |
| Database Service | [`src/db/db.service.ts`](src/db/db.service.ts) |
| Device Records Service | [`src/device-records/device-records.service.ts`](src/device-records/device-records.service.ts) |
| Device Records Controller | [`src/device-records/device-records.controller.ts`](src/device-records/device-records.controller.ts) |
| Device Record Model | [`src/db/models/deviceRecord.ts`](src/db/models/deviceRecord.ts) |
| Event Record Model | [`src/db/models/eventRecords.ts`](src/db/models/eventRecords.ts) |
| DM1 Event Record Model | [`src/db/models/dm1EventRecords.ts`](src/db/models/dm1EventRecords.ts) |
| Device Currents Model | [`src/db/models/deviceCurrents.ts`](src/db/models/deviceCurrents.ts) |
| Device Information Model | [`src/db/models/deviceInformation.ts`](src/db/models/deviceInformation.ts) |
| Utilities | [`src/common/utilities.ts`](src/common/utilities.ts) |
| Constants | [`src/common/constants.ts`](src/common/constants.ts) |

---

## Appendix B: Key Methods Reference

| Method | Service/Component | Purpose |
|--------|------------------|---------|
| `saveToDatabase()` | SQS Service | Main entry point for processing SQS messages |
| `createDeviceRecord()` | Database Service | Creates device record with duplicate check |
| `filterNewEvents()` | Database Service | Filters duplicate events |
| `computeActualOngoingFault()` | Database Service | Computes enhanced fault status |
| `createDeviceCurrents()` | Database Service | Creates device currents record |
| `getDataLogById()` | Device Records Service | Retrieves device records for API |
| `getDeviceEventsById()` | Device Records Service | Retrieves events for API |
| `getDeviceLoadHistoryById()` | Device Records Service | Retrieves load history for API |
| `parseDateByUserTZ()` | Device Records Service | Converts UTC to user timezone |
| `getOngoingFaultValue()` | Device Records Service | Maps fault enum to string |

---
