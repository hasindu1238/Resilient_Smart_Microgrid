# Technical Specification: IoT Pipeline & Security Guardrails


## Introduction
The transition from Day 1 (IoT) to Day 2 (AI) requires grounding agentic decisions in physical reality. This specification defines the hardware-agnostic definitions and security guardrails necessary to prevent "open-loop" failures. We move from the fixed rules of traditional IoT to the "Brain-led" optimization required for rural Sri Lankan resilience.


## Virtual IoT Device Definitions
The system simulates the following village-scale hardware, matching the ESP32 logic provided in Source context:
- **Solar PV Array:** Wattage output simulation (0-5000W).
- **Battery SoC Monitor:** 48V system simulation; reports State of Charge (0-100%).
- **4-Channel Load Switch:** Digital relay control for the hierarchy loads.


## Telemetry JSON Schemas & Handshake
The "Brain" must parse raw payloads from the ESP32 (Source Code: `{"temperature":%.1f,"humidity":%d}`) and map them into the following Pydantic-compatible contract:


```json
{
  "device_id": "string",
  "ts": "iso_timestamp",
  "metric": "string (temperature|humidity|wattage|soc)",
  "value": "float",
  "unit": "string",
  "metadata": {
    "esp32_delay": 5000,
    "secure_client": "WiFiClientSecure"
  }
}



