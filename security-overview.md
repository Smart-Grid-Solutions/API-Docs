# DNP3 Device Portal - Security Architecture and Threat Model

The SGS DNP3 Device Portal is a read-only data relay that delivers line current and fault information from field devices to utility SCADA systems using the DNP3 protocol. This document describes how the system is secured and what is, and is not, at risk.

**Key takeaway:** The portal is a read-only outstation with no control capability. All connections are authenticated with mutual TLS. Even under a worst-case compromise of the entire SGS cloud environment, the bounded impact is limited to incorrect measurement data. No control commands can be issued and no customer systems can be accessed.

For implementation details, certificate setup, and configuration steps, see the companion [Getting Connected](security-implementation-guide.md).

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Connectivity Models](#2-connectivity-models)
3. [Threat Model and Bounded Risk](#3-threat-model-and-bounded-risk)
4. [Transport Security (Mutual TLS)](#4-transport-security-mutual-tls)
5. [Cloud Infrastructure Security](#5-cloud-infrastructure-security)
6. [Certificate Lifecycle](#6-certificate-lifecycle)
7. [Customer-Side Security Recommendations](#7-customer-side-security-recommendations)
8. [Regulatory Framework Alignment](#8-regulatory-framework-alignment)
9. [Contact and Incident Reporting](#9-contact-and-incident-reporting)

---

## 1. System Architecture

### 1.1 Read-Only Outstation Design

The DNP3 Device Portal operates exclusively as a **DNP3 outstation** (message concentrator/responder). It serves measurement and status data to a connecting DNP3 master, typically an RTAC, RTU, or similar gateway device on the customer's SCADA network.

The portal serves only **DNP3 Analog Input (AI) points**: read-only measurement values that include line current readings, fault information, battery status, and device metadata. All discrete status values (fault types, device states) are encoded as integer enumerations on AI points.

**Data flows in one direction:** from the SGS backend, through the portal, to the customer's DNP3 master. The portal never initiates connections to customer systems, never writes to customer devices, and never issues control commands.

### 1.2 Security by Design

The portal is purpose-built for one-way data delivery. Its security boundaries are enforced by the architecture itself, not just by configuration:

- **Read-only by design.** The portal does not implement DNP3 control relay output blocks (CROB), analog outputs, binary outputs, or any operate/select-before-operate functionality. All control requests are rejected by default at the code level.
- **One-way data flow.** The portal cannot write data to the connecting master or to any customer system.
- **No outbound connectivity to customer networks.** The portal accepts incoming TCP connections from the customer's DNP3 master but never initiates connections in the other direction.
- **Isolated per customer.** Each customer's DNP Device Portal instance has its own configuration, certificates, and message queue. There is no shared state or cross-customer data access between portal instances.

### 1.3 Data Flow

```
Field FCIs
        |
        v
SGS Backend Services (API / SQS Message Queue)  <-- AWS Cloud
        |
        v
DNP3 Device Portal (Outstation)  <-- AWS Cloud
        |
        | TCP + mTLS (dedicated port)
        |
        v
Customer DNP3 Master (RTAC / RTU / Gateway)  <-- Customer Network
```

---

## 2. Connectivity Models

The portal supports two connectivity models. Both use mutual TLS (mTLS) for application-layer authentication and encryption. The choice between them depends on the customer's security policy and network requirements.

### 2.1 Option A: Public IP with Mutual TLS

The portal listens on a public IP address and a dedicated port (typically in the 20000-21000 range). The customer's DNP3 master connects directly over the internet using mTLS.

**How it works:**
- The portal's public IP and port are dedicated to this customer
- Both the portal and the master present TLS certificates and validate each other
- Without a valid client certificate and private key, the connection is refused at the TLS handshake before any DNP3 data is exchanged
- TLS 1.3 is the default (TLS 1.2 is available if required by older systems)

### 2.2 Option B: VPN with Mutual TLS

A VPN tunnel is established between the customer's network and the SGS AWS environment. DNP3 traffic with mTLS flows through the encrypted tunnel using private IP addresses.

**How it works:**
- A VPN tunnel is established between the customer's VPN gateway and an AWS Virtual Private Gateway
- The portal's DNP3 port is only reachable through the VPN tunnel and not the public internet
- mTLS authentication still applies within the VPN tunnel
- AWS provides two tunnels for redundancy

The most common configuration is IPSec (IKEv2) with pre-shared key authentication via AWS Site-to-Site VPN, but we can support other VPN types. Contact us for specific VPN requirements.

### 2.3 Choosing a Connectivity Model

| Consideration | Public IP + mTLS | IPSec VPN + mTLS |
|---|---|---|
| **Security layers** | TLS 1.2/1.3 with mutual certificate authentication | IPSec tunnel + TLS mutual certificate authentication |
| **Network visibility** | Portal port is reachable on the public internet (but rejects all unauthenticated connections at the TLS handshake) | Portal port is only reachable within the VPN tunnel; not visible on the public internet |
| **Setup complexity** | *Lower:* firewall rule + certificate exchange | *Higher:* VPN gateway configuration on both sides, plus certificate exchange |
| **Ongoing maintenance** | Certificate management | Certificate management + PSK rotation + VPN tunnel monitoring |
| **Suitable when** | Security policy accepts mTLS as sufficient for read-only, non-control data | Security policy requires private connectivity for all operational technology (OT) traffic, or the DNP3 master cannot reach public endpoints |

**Key point:** mTLS alone provides strong authentication and encryption for the DNP3 session. The VPN option adds network-layer isolation for organizations that require defense-in-depth or whose security policy mandates private connectivity for OT traffic. Both options deliver the same application-layer security. The VPN is additive, not compensatory.

---

## 3. Threat Model and Bounded Risk

This section describes the realistic attack vectors against this system and the bounded impact of each.

### 3.1 Attack Surface Summary

The portal exposes a small number of network endpoints per customer: one or more dedicated TCP ports (one per DNP3 outstation) serving the DNP3 protocol over TLS. There are no web interfaces, APIs, SSH access, or other services exposed to the customer's network.

### 3.2 Attack Vectors

#### Vector 1: Network Interception (Man-in-the-Middle)

**Threat:** An attacker intercepts traffic between the portal and the customer's DNP3 master to read or modify DNP3 data in transit.

**Mitigation:** All traffic is encrypted with TLS 1.3 (or 1.2). With the VPN option, traffic is additionally encrypted by IPSec. The attacker would need to break TLS to read or modify data in transit. mTLS ensures both endpoints are authenticated. An attacker cannot impersonate either side without the corresponding private key.

**Residual risk:** Low, given current TLS cryptographic strength and key sizes (4096-bit RSA).

#### Vector 2: Unauthorized Connection to the Portal

**Threat:** An attacker discovers the portal's IP and port and attempts to connect as a DNP3 master.

**Mitigation:** The portal requires a valid client TLS certificate before completing the handshake. Without the customer's private key material, the connection is rejected before any DNP3 data is exchanged.

With the VPN option, the portal's port is not reachable from the public internet at all—the attacker would also need to breach the IPSec tunnel.

**Residual risk:** An attacker with access to the client's private key could establish a session. This requires compromise of the customer's certificate storage, which is outside the portal's control. See [Customer-Side Security Recommendations](#7-customer-side-security-recommendations).

#### Vector 3: Compromise of the Portal Service

**Threat:** An attacker gains control of the portal's cloud instance (e.g., through a vulnerability in the service or the underlying platform).

**Mitigation:** Even with full control of the portal, the attacker's capabilities are bounded:

- They can serve **spoofed Analog Input values** through the existing mTLS session to the customer's master
- They **cannot** issue control commands as the portal has no control capability in its DNP3 configuration
- They **cannot** access customer networks as the only network path is the single DNP3 port, and the portal is the listener (not the initiator)
- They **cannot** pivot laterally as the customer's firewall restricts traffic to DNP3 on that single port

**Residual risk:** The customer's DNP3 master could receive incorrect measurement data (spoofed current readings, false faults). The severity of this depends on whether the customer has downstream automation that acts on this data without cross-validation. See [Data Validation](#73-data-validation-at-the-dnp3-master) below.

#### Vector 4: Compromise of the SGS Backend

**Threat:** An attacker compromises the upstream backend services (API, SQS message queue) that feed data to the portal.

**Mitigation:** The impact is the same as Vector 3: spoofed measurement data flowing through the portal to the customer's master. The portal has no mechanism to propagate access or control beyond DNP3 Analog Input data.

**Residual risk:** Same as Vector 3.

### 3.3 Worst-Case Summary

**Even under a full compromise of the SGS cloud environment, the worst-case outcome is that a customer's DNP3 master receives incorrect measurement data through one or more dedicated firewall pinholes. No control commands can be issued. No customer systems can be accessed or modified.**

This bounded risk is an inherent property of the system's read-only, outstation-only architecture and not a claim that requires trust in SGS operational security alone.

---

## 4. Transport Security (Mutual TLS)

### 4.1 What Mutual TLS Provides

Standard TLS (as used on websites) authenticates only the server to the client. Mutual TLS (mTLS) adds client authentication: the connecting DNP3 master must also present a valid certificate that the portal verifies before completing the handshake.

This means:
- **The portal verifies the master's identity**: only masters with a recognized certificate can connect
- **The master verifies the portal's identity**: the master can confirm it is connected to the genuine SGS portal
- **All traffic is encrypted**: data in transit cannot be read or modified by a third party
- **No pre-shared secrets in the data path**: authentication is based on asymmetric cryptography (RSA 4096-bit keys)

### 4.2 Certificate Modes

The portal supports two certificate validation modes:

| Mode | How It Works | Best For |
|------|-------------|----------|
| **Self-Signed** | The portal holds the customer's exact certificate. Only that specific certificate is accepted. | Simple deployments, single master per customer |
| **Authority-Based (CA)** | The portal validates the customer's certificate against a Certificate Authority (CA) chain. Any certificate signed by the trusted CA is accepted. | Organizations with their own PKI, or multi-master deployments |

In CA-based mode, an optional subject name check (`TLS_CLIENT_SUBJECT_NAME`) can further restrict connections to certificates with a specific Common Name or Subject Alternative Name. This is recommended to narrow acceptance beyond "any certificate signed by this CA." In self-signed mode, subject name checking is not needed as the portal already performs exact certificate matching, which is stricter.

### 4.3 TLS Version

The portal defaults to **TLS 1.3** as the minimum version. TLS 1.2 is available as a fallback for DNP3 masters that do not yet support TLS 1.3. TLS 1.0 and 1.1 are not supported.

---

## 5. Cloud Infrastructure Security

The portal runs on AWS infrastructure with the following security controls:

### 5.1 Credential Management

- API credentials are stored in **AWS Secrets Manager** and are not placed in configuration files
- The portal retrieves credentials at startup through the AWS SDK using IAM role-based authentication

### 5.2 Access Control

- The portal's AWS execution role uses **minimum-privilege IAM policies** scoped to the specific resources it needs (its SQS queue, S3 paths, and Secrets Manager secret)
- Each customer's portal instance runs with its own IAM permissions scoped to its specific resources

### 5.3 Data at Rest

- TLS certificate files stored in S3 use server-side encryption
- The portal downloads certificates to an ephemeral local directory (`/tmp/dnp3`) at startup—in containerized deployments, this storage is not persisted to disk across restarts

### 5.4 Logging and Monitoring

- Application logs are captured by **AWS CloudWatch** for centralized monitoring and alerting
- TLS handshake failures and data processing errors are logged
- DNP3 protocol-level logging is available for troubleshooting but is disabled in production to avoid logging measurement values

---

## 6. Certificate Lifecycle

Each customer receives a unique certificate pair (4096-bit RSA, configurable validity). Certificates are generated, distributed through secure channels, and rotated on a coordinated schedule. Private keys and certificates are always sent through separate channels, and SHA-256 checksums are provided for integrity verification.

In self-signed mode, revocation is immediate: replacing the trusted client certificate on the portal means the old certificate is rejected at the next TLS handshake. In CA-based mode, note that the portal does not currently perform CRL or OCSP checking, so revocation relies on replacing the trusted certificate chain and restarting the portal.

For VPN deployments, the pre-shared key (PSK) should be rotated at minimum annually or per the customer's security policy.

For detailed procedures covering certificate generation, distribution, rotation, and emergency re-keying, see the [Getting Connected](security-implementation-guide.md).

---

## 7. Customer-Side Security Recommendations

These recommendations help customers minimize risk when connecting to the SGS DNP3 portal, regardless of which connectivity model is used.

### 7.1 Firewall Rules

- **Allow only the assigned DNP3 port** (e.g., TCP 20000) from the portal's IP address (public IP for direct mTLS, or VPN tunnel address for VPN deployments)
- **Deny all other traffic** between the portal and the customer's network
- **Do not open additional ports** for management, monitoring, or any other purpose—the portal does not require or use them

### 7.2 DNP3 Object Filtering

If the DNP3 master supports it, configure it to:
- **Accept only Analog Input responses** (DNP3 Object Groups 30 and 32)
- **Reject any unexpected object groups**—the portal only serves AI points, so any other object group in a response would be anomalous
- **Reject any control-direction messages**—the portal is an outstation and should never send messages that look like master-initiated commands

### 7.3 Data Validation at the DNP3 Master

Because the primary residual risk (even under worst-case compromise) is spoofed measurement data:

- **Implement reasonableness checks** on received analog values—a fault current of 999,999 A or a negative battery voltage should trigger an alarm, not be accepted as valid
- **Cross-validate against local measurements** where possible—if the portal reports a fault but local instrumentation does not confirm it, flag the discrepancy
- **Do not trigger automated control actions** based solely on data from this portal without independent confirmation
- **Alert on sudden, large deviations** from expected baselines

### 7.4 Network Monitoring

- **Deploy an intrusion detection system with DNP3 protocol awareness** (e.g., Claroty, Dragos, or Zeek with a DNP3 parser) to detect anomalous function codes or unexpected DNP3 object types
- **Monitor for unexpected connection patterns**—the portal maintains a persistent session; frequent disconnects and reconnects may indicate an issue
- **Log and review TLS handshake events** to detect unauthorized connection attempts

### 7.5 Certificate Storage

- Store the client private key (`client_cert_key.pem`) in a secure location with restricted access
- If your DNP3 master supports passphrase-protected private keys, consider encrypting the key file at rest. This is especially worth doing if the private key is stored on a shared system, a device with broad administrative access, or any platform where file-level access controls alone may not be sufficient.
- Do not store certificates on network shares or in version control systems
- Use hardware security modules (HSMs) where available for private key storage

---

## 8. Regulatory Framework Alignment

### 8.1 NERC CIP

Many utility customers operate under NERC CIP reliability standards. While this system does not itself carry a NERC CIP compliance certification, its design aligns with several CIP requirements:

- **CIP-005 (Electronic Security Perimeters):** The portal connects through a defined network boundary—a single TCP port with mTLS authentication, optionally wrapped in an IPSec VPN. Customer firewall rules should define the Electronic Security Perimeter (ESP) accordingly. The portal's single-port, single-protocol design makes ESP definition straightforward.

- **CIP-007 (System Security Management):** TLS certificate management, credential storage in AWS Secrets Manager, access controls via IAM, and CloudWatch logging support CIP-007 requirements for system hardening, access management, and security event monitoring.

- **CIP-012 (Communication Between Control Centers):** mTLS encryption of all DNP3 traffic in transit satisfies the intent of protecting real-time assessment and real-time monitoring data. The use of TLS 1.3 with 4096-bit RSA keys provides strong cryptographic protection.

We recommend that your NERC CIP compliance team evaluate which standards apply to this specific interconnection based on your facility's BES Cyber System categorization. SGS will support your compliance team with technical details as needed.

### 8.2 IEC 62351

IEC 62351 defines security mechanisms for power system communication protocols, including DNP3.

- **IEC 62351-3 (TLS for TCP-based protocols):** This system implements TLS-based transport security as described in IEC 62351-3 for DNP3 over TCP. Both mutual authentication and traffic encryption are supported.

- **IEC 62351-5 (DNP3 Secure Authentication):** This system does not implement DNP3 Secure Authentication (SA). SA's primary value is per-message authentication of control operations—since this system has no control capability, TLS-based transport security provides the relevant protections (endpoint authentication and encryption) with better reliability and interoperability. This is the more common approach for DNP3-over-TCP deployments in the industry. If your organization has a specific requirement for DNP3 SA, please contact SGS to discuss.

---

## 9. Contact and Incident Reporting

SGS provides direct support for all security-related matters:

- **Security incidents:** security@smartgridsolutions.com (monitored by the SGS engineering team). If you suspect certificate compromise or unauthorized access, contact us immediately. We will initiate certificate re-keying, coordinate with your security team, and provide relevant logs and timeline information.
- **General questions:** info@smartgridsolutions.com
- **Certificate requests and rotation:** Coordinated through your SGS account contact

During onboarding, SGS and the customer exchange designated security contacts to ensure rapid communication in the event of an incident. SGS is committed to transparency and timely response on all security matters.

For routine connectivity and configuration questions, refer to the [Getting Connected](security-implementation-guide.md).
