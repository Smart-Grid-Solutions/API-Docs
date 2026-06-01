# SGS Device Portal - Integration Guide

The **SGS Device Portal** delivers status, line current, and real-time fault data from field devices to your operational systems. This document describes the integration recipes available for connecting your SCADA, OMS, GIS, data platforms, and other systems to the portal.

It will help your technical and security teams understand the available options, what each recipe requires from your side, and how to choose the right approach for your environment.

---

## Recipes at a Glance

| Recipe | Protocol | Connectivity | Typical Use Case |
|---|---|---|---|
| A - DNP3 over Public Endpoint | DNP3/TCP + mTLS | Internet | SCADA integration via public network |
| B - DNP3 over VPN | DNP3/TCP + mTLS | Site-to-site VPN | SCADA integration with no public exposure |
| C - MultiSpeak (Direct) | MultiSpeak SOAP/HTTPS | Internet | OMS notification via internet-facing endpoint |
| D - MultiSpeak via DMZ | MultiSpeak SOAP/HTTPS | Internet to DMZ | OMS notification with DMZ separation |
| E - REST API | REST/HTTPS/JSON | Internet | OMS, data lake, or custom system integration |

Each recipe is described in detail below, including architecture, data flow, what you need to provide, and key security considerations.

---

## Recipe A - SCADA via DNP3 (Public Endpoint)

### Architecture

```
                            SGS Cloud
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│   Devices ──► SGS Device Portal ──► DNP3 Outstation         │
│                                           │                 │
│                                    Public NLB (TCP 20000)   │
└─────────────────────────────────────────────────────────────┘
                                           │
                                    mTLS (Internet)
                                           │
                              ┌──────────────────────────────┐
                              │  Customer Network            │
                              │  DNP3 Master (RTAC / RTU /   │
                              │  Gateway)                    │
                              └──────────────────────────────┘
```

### How It Works

1. Field devices report status and events to the SGS Device Portal.
2. The SGS platform translates device data into DNP3 point mappings and exposes a **DNP3 outstation** endpoint.
3. A **public network load balancer** forwards your DNP3 master's connections to the outstation on TCP **port 20000**.
4. Your DNP3 master polls or receives unsolicited responses as configured.

### What You Need to Provide

- **Fixed IP addresses** for your DNP3 master (optional, for IP allow-listing as an additional layer of security).
- **DNP3 master configuration**: outstation address, class polling / integrity scan settings.
- **TLS client certificate**: SGS provides a certificate package during onboarding. Please contact SGS for certificate setup details.

### Key Security Considerations

- **Mutual TLS (mTLS)**: All connections require mTLS. Both the portal and your DNP3 master present and validate certificates during the TLS handshake. Without a valid client certificate, the connection is refused before any DNP3 data is exchanged. TLS 1.3 is the default; TLS 1.2 is available if your DNP3 master requires it.
- **IP allow-listing (optional)**: As an additional layer, SGS can restrict access to the public endpoint to your declared IP addresses.
- **Certificate management**: Certificates should be rotated on a defined schedule. SGS coordinates rotation in advance. Please contact SGS for the full certificate lifecycle.

---

## Recipe B - SCADA via DNP3 over VPN

### Architecture

```
                     Customer Network
        ┌────────────────────────────────────┐
        │  DNP3 Master (RTAC / RTU /         │
        │  Gateway)                          │
        └──────────────────┬─────────────────┘
                           │ DNP3 (TCP 20000)
                    Site-to-Site VPN
                           │
                           ▼
                     SGS Cloud (Private)
        ┌────────────────────────────────────┐
        │  Internal NLB (TCP 20000)          │
        │         │                          │
        │         ▼                          │
        │  DNP3 Outstation                   │
        │         │                          │
        │         ▼                          │
        │  SGS Device Portal                 │
        └────────────────────────────────────┘
```

### How It Works

1. The same device-to-portal data flow as Recipe A.
2. The DNP3 outstation is not publicly exposed - it is only reachable from within the SGS private network or via the VPN tunnel.
3. SGS establishes a **site-to-site VPN** between your network and the SGS cloud environment.
4. Your DNP3 master connects over the VPN to an **internal load balancer** on TCP **20000**. mTLS is still required within the tunnel.

### What You Need to Provide

- **VPN endpoint details**: public IP of your VPN device and BGP ASN (if using dynamic routing).
- **Network CIDRs**: subnets that need to reach the DNP3 endpoint.
- **DNP3 master configuration**: same DNP3 connection settings as Recipe A.

### Key Security Considerations

- **No public exposure**: The DNP3 endpoint is entirely private; no internet-facing surface.
- **mTLS within the tunnel**: The same mutual TLS authentication from Recipe A applies inside the VPN. The VPN adds network-layer isolation; mTLS provides application-layer authentication.
- **VPN hardening**: Use modern ciphers and strong authentication (certificates or strong pre-shared keys). SGS recommends monitoring VPN tunnel health. AWS provides two tunnels for redundancy.
- **Network segmentation**: Routing and firewall rules are scoped so that only your designated subnets can reach the DNP3 endpoint through the tunnel.

---

## Recipe C - OMS via MultiSpeak (Direct Internet-Facing)

### Architecture

```
     SGS Cloud                                Customer Network
┌──────────────────────┐     HTTPS/SOAP    ┌──────────────────────────────────┐
│ SGS Device Portal    │ ─────────────────►│ Customer OMS MultiSpeak Endpoint │
│ (MultiSpeak client)  │                   │ (Internet-facing, WAF/proxy)     │
└──────────────────────┘                   └──────────────────────────────────┘
```

### How It Works

1. Field devices report status and events to the SGS Device Portal.
2. On relevant state changes (faults, restorations, and optionally check-ins), the SGS platform builds a **MultiSpeak-compliant SOAP payload** and sends it via HTTPS to your OMS endpoint.
3. Your OMS endpoint (typically behind a reverse proxy or WAF) validates the request and updates its outage model.

### What You Need to Provide

- **MultiSpeak endpoint URL**: a publicly accessible HTTPS URL for your OMS's MultiSpeak listener.
- **Credentials for your OMS endpoint**: a username and password that your OMS expects in the MultiSpeak SOAP header (`MultiSpeakMsgHeader`). SGS encrypts these at rest (AES encryption with keys managed in AWS Secrets Manager) and uses them exclusively for sending notifications to your endpoint.
- **Supported MultiSpeak method**: choose one of the supported fault notification methods (see [Supported MultiSpeak Methods](#supported-multispeak-methods) below).
- **IP allow-listing** (recommended): permit inbound requests from SGS's egress IP ranges only.

### Device Point Mapping

To help your OMS pre-map incoming notifications to the correct assets, SGS exposes a SOAP endpoint that returns all devices in your organization as SCADA points:

- **WSDL**: `GET /soap/wsdl`
- **Operation**: `GetAllSCADAPoints`
- **Authentication**: `MultiSpeakMsgHeader` with your SGS Device Portal `UserID` and `Pwd`

Each entry in the response includes:

| Field | Description |
|---|---|
| `objectID` | Device identifier (IMEI number) used in all fault and check-in notifications |
| `objectName` | Human-readable device name |
| `description` | Device serial number, where available |

Use the `objectID` values to build your OMS point mapping before going live. All subsequent fault and check-in notifications will reference this same identifier.

### Key Security Considerations

- **Credential security**: SGS encrypts the credentials you provide at rest (AES encryption with keys managed in AWS Secrets Manager). They are never shared across customers or exposed in logs.
- **TLS**: Your endpoint must use HTTPS with a valid certificate. Weak cipher suites should be disabled.
- **Customer-side controls**: You may additionally apply WAF rules, rate limiting, and IP allow-listing on your endpoint to restrict inbound traffic to known SGS egress addresses.

---

## Recipe D - OMS via MultiSpeak Through DMZ / Proxy

### Architecture

```
     SGS Cloud                    Customer DMZ                 Customer Internal
┌──────────────────┐  HTTPS/SOAP  ┌──────────────────┐  HTTP(S)  ┌────────────────┐
│ SGS Device Portal│ ────────────►│ Reverse Proxy    │ ─────────►│ OMS / DMS      │
│ (MultiSpeak      │              │ / WAF            │           │ (MultiSpeak    │
│  client)         │              │ (DMZ listener)   │           │  server)       │
└──────────────────┘              └──────────────────┘           └────────────────┘
```

### How It Works

1. Same device-to-portal data flow and MultiSpeak payload construction as Recipe C.
2. Instead of calling your OMS directly, the SGS platform sends SOAP/HTTPS requests to a URL you expose in your **DMZ or proxy tier**.
3. Your DMZ/proxy terminates TLS, applies WAF / rate limiting / authentication, and forwards requests to your internal OMS/DMS.
4. This keeps your core OMS shielded from direct internet traffic.

### What You Need to Provide

- **DMZ endpoint URL**: an HTTPS URL in your DMZ or proxy tier that accepts MultiSpeak SOAP requests.
- **Credentials for your OMS endpoint**: same as Recipe C: a username and password that your OMS expects in the SOAP header.
- **Proxy forwarding rules**: Configure your proxy to forward MultiSpeak requests to your internal OMS/DMS listener.
- **Supported MultiSpeak method**: see [Supported MultiSpeak Methods](#supported-multispeak-methods) below.

### Device Point Mapping

The same `GetAllSCADAPoints` SOAP endpoint described in Recipe C is available for DMZ-based integrations. Use it to retrieve the `objectID` values for all devices in your organization before configuring your internal OMS point mapping. See the Device Point Mapping section under Recipe C for details.

### Key Security Considerations

- **DMZ separation**: Your core OMS is never directly reachable from the internet; all traffic terminates at the DMZ.
- **TLS everywhere**: HTTPS from SGS to your DMZ; TLS (or equivalent internal controls) from DMZ to OMS.
- **Strong authentication**: Consider mutual TLS or signed tokens in addition to IP allow-listing for the DMZ listener.
- **WAF / rate limiting**: Apply WAF rules at the DMZ to guard against replay attacks and request floods.

---

## Recipe E - Direct REST API Integration

### Architecture

```
     Customer Network                               SGS Cloud
┌───────────────────────────┐   HTTPS/JSON   ┌──────────────────────────┐
│ Customer System           │◄──────────────►│ SGS Device Portal        │
│ (OMS / Data Lake / Custom)│   REST API     │ REST API                 │
└───────────────────────────┘                └──────────────────────────┘
```

### How It Works

1. Field devices report data to the SGS Device Portal continuously.
2. Your system authenticates to the SGS REST API and retrieves device status, events, current readings, and other data on demand or on a polling schedule.
3. Optionally, event-driven notifications can be delivered to a webhook endpoint you provide. Contact SGS if your integration requires push notifications rather than polling.

### Authentication

To authenticate, send a POST request to the SGS API with your Device Portal organizational admin credentials:

```
POST https://api.portal-sgs.com/api/auth/login
Content-Type: application/json

{
    "email": "<your_email>",
    "password": "<your_password>"
}
```

The response includes an `accessToken` (Bearer token, valid for 24 hours). Include it in subsequent requests:

```
Authorization: Bearer <accessToken>
```

### Available API Data

Query device data for your organization:

```
GET https://api.portal-sgs.com/api/dnp3/dnp3OrganizationDevices
Authorization: Bearer <accessToken>
```

The response is a JSON array of device objects, each containing:

- **DeviceInfo**: device type, IMEI, model, firmware version, configuration, GPS coordinates, DNP3 address mapping
- **DeviceStatus**: last check-in timestamp, ongoing fault state, battery voltages, power source, signal strength, current readings (1-minute average, 1-hour average, 24-hour peak, hourly history)
- **MostRecentEvent**: event type, timestamp, prefault/fault/recovery currents, per-phase fault details (where applicable)

### What You Need to Provide

- **Target environment**: specify whether you need access to a development, staging, or production environment.
- **Data contract requirements**: let SGS know which data fields and response formats your downstream systems require so that any gaps can be addressed.

SGS will provide organizational admin credentials once onboarding begins. Store these securely in your own secret management system.

### Key Security Considerations

- **Scoped access**: Credentials are limited to your organization and cannot access other customers' data.
- **TLS only**: All API calls must be made over HTTPS; unencrypted access is not available.
- **Token expiry**: Bearer tokens expire after 24 hours. Your integration should handle re-authentication automatically.
- **Credential storage**: Store credentials in a secrets manager, not in source code or configuration files.

---

## Supported MultiSpeak Methods

For MultiSpeak integrations (Recipes C and D), the SGS platform currently supports the following SOAP operation names. You configure one method per integration:

**Fault / status notifications:**
- `SCADAStatusChangedNotificationByPointID` (MultiSpeak 4.1)
- `SCADAStatusChangedNotification` (MultiSpeak 4.1)
- `SCADAStatusChangedNotification` (MultiSpeak 3.0)
- `AssessmentLocationChangedNotification` (MultiSpeak 4.1)
- `ODEventNotification` (MultiSpeak 4.1)
- `SCADAAnalogChangedNotificationByPointID` (MultiSpeak 4.1)

**Check-in / heartbeat notifications:**
- `SCADAAnalogChangedNotificationByPointID` (MultiSpeak 4.1)

Note: `SCADAStatusChangedNotification` is available in both MultiSpeak 3.0 and 4.1 variants. When configuring your integration with SGS, specify which version your OMS expects so the correct namespace and SOAPAction are used.

If your OMS requires a method not listed here, please contact SGS to discuss feasibility. Additional methods can be implemented by extending the platform's MultiSpeak adapter.

---

## Choosing the Right Recipe

| Consideration | Recommendation |
|---|---|
| You have a SCADA system using DNP3 | Recipe A (public + mTLS) or Recipe B (VPN + mTLS) |
| You require no public internet exposure for SCADA | Recipe B (VPN) |
| Your OMS uses MultiSpeak and can accept internet traffic | Recipe C (direct) |
| Your security policy requires DMZ separation | Recipe D (DMZ/proxy) |
| You need flexible data access for OMS, analytics, or custom tools | Recipe E (REST API) |
| You need a combination of real-time SCADA + data access | Recipes A or B combined with Recipe E |

---

## Next Steps

To get started, let your SGS account contact know:

Once you've identified which recipe fits your environment, contact your SGS sales rep to begin onboarding. We'll work through the specific requirements for your chosen integration (certificates, endpoints, credentials, VPN details, etc.) together.
For general questions: info@smartgridsolutions.com
