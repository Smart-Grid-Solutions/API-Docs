# DNP3 Device Portal - Getting Connected

This document provides the technical details needed to connect to the SGS DNP3 Device Portal. It is intended for network engineers, SCADA engineers, and IT security staff responsible for configuring the customer-side connection.

For the threat model, risk assessment, and security architecture overview, please contact SGS.

## Table of Contents

1. [Connection Overview](#1-connection-overview)
2. [Certificate Setup](#2-certificate-setup)
3. [DNP3 Master Configuration](#3-dnp3-master-configuration)
4. [Firewall Rules](#4-firewall-rules)
5. [VPN Configuration](#5-vpn-configuration)
6. [Monitoring and Verification](#6-monitoring-and-verification)
7. [Certificate Rotation](#7-certificate-rotation)
8. [Incident Reporting](#8-incident-reporting)

---

## 1. Connection Overview

### 1.1 What SGS Provides

During onboarding, SGS will provide:

- A **dedicated IP address and port** for the DNP3 connection (starting at 20000)
- A **certificate package** containing the files needed for mutual TLS authentication
- (If requested) A **DNP3 point map** showing which data points correspond to which field devices
- (Optional VPN) AWS VPN endpoint details for tunnels

### 1.2 What You Need to Provide

- **DNP3 master IP address** (the public IP that will connect to the portal, or the internal IP if using VPN)
- **Preferred connectivity model:** public IP with mTLS, or VPN with mTLS
- **TLS version requirements:** TLS 1.3 is the default; TLS 1.2 is available if your DNP3 master requires it
- **Preferred certificate mode:** self-signed (simpler) or CA-based (if you have your own PKI)
- **Security contact** for incident communication
- (Optional VPN) your VPN gateway's public IP address and internal network CIDR

---

## 2. Certificate Setup

All connections use mutual TLS (mTLS). Both the portal and your DNP3 master must present valid certificates during the TLS handshake. Without a valid certificate and private key, the connection is refused before any DNP3 data is exchanged.

### 2.1 Certificate Package

SGS will provide a certificate package containing:

| File | Purpose |
|------|---------|
| `portal_cert.pem` | The portal's public certificate. Import this so your master can verify the portal's identity. |
| `client_cert.pem` | Your public certificate. Your master presents this to the portal during the TLS handshake. |
| `client_cert_key.pem` | Your private key. **Keep this secure.** Your master uses this to prove its identity. |
| `client_cert.p12` | (If requested) PKCS#12 bundle. Some SCADA systems prefer this format. |
| `*.sha256` | SHA-256 checksums for verifying file integrity after transfer. |

### 2.2 Who Generates the Keys?

By default, SGS generates both key pairs for simplicity. If you prefer to generate your own private key and provide a Certificate Signing Request (CSR) instead, we're happy to work with that. We can discuss specific requirements during onboarding.

For CA-based deployments, certificates are typically issued by your own PKI. SGS will provide the portal's certificate (or CA chain) so your master can verify the portal, and you provide a certificate signed by the CA that the portal is configured to trust.

### 2.3 After Receiving Certificates

1. **Verify checksums:** Compare the SHA-256 checksums against the values provided by SGS to confirm file integrity
2. **Import certificates** into your DNP3 master per its documentation
3. **Store the private key securely.** Restrict file access, avoid network shares and version control.
   - If your master supports passphrase-protected keys, consider encrypting the key file at rest. This is especially worth doing if the private key is stored on a shared system or a device with broad administrative access.
4. **Confirm receipt** with SGS so the connection test can be scheduled

---

## 3. DNP3 Master Configuration

### 3.1 Unsolicited Responses

The portal sends unsolicited responses by default, pushing point updates to your master as soon as new data arrives rather than waiting for polls. Your master should be configured to accept unsolicited responses from this outstation.

If your security policy prohibits unsolicited responses, let SGS know and we can disable them. Your master will then need to poll for updates.

### 3.2 Heartbeat Signal

The portal provides a heartbeat signal on **AI point 0** that counts from 0 to 100, then resets to 0, incrementing once per second. Use this to verify the connection is active and the portal process is running. If the heartbeat stops updating, contact SGS.

### 3.3 Data Points

All data is served as **DNP3 Analog Input (AI) points** (Object Groups 30 and 32). The portal does not serve any other object types. Your master should not expect or request Binary Inputs, Counters, or any control objects.

---

## 4. Firewall Rules

### 4.1 Required Rules

For **public mTLS** deployments:

```
Outbound: TCP to <portal-hostname-or-ip> port <assigned-port>
Inbound:  Stateful return traffic (or explicitly: TCP from <portal-hostname-or-ip> port <assigned-port>)
```

For **VPN** deployments, use the portal's VPN tunnel address instead of the public IP.

### 4.2 Recommendations

- Restrict to the specific portal hostname or IP. Do not allow broad CIDR ranges.
- Allow only the assigned port. Do not open a range.
- The portal will only communicate with your DNP3 master on the assigned port. No additional firewall rules are needed.
- Review and audit these firewall rules periodically.

### 4.3 VPN Firewall Considerations

If using VPN, your VPN gateway also needs to allow:

```
UDP port 500   (IKE)
UDP port 4500  (NAT-T)
IP protocol 50 (ESP)
```

These are between your VPN gateway and the AWS VPN endpoint IPs, not between the DNP3 master and the portal.

---

## 5. VPN Configuration

If your organization requires private connectivity for operational technology (OT) traffic, this section covers the VPN setup. Skip this section if you are using public IP with mTLS.

### 5.1 Architecture

```
Your VPN Gateway  <---IPSec (IKEv2)---> AWS Virtual Private Gateway
        |                                            |
Your DNP3 Master (RTAC / RTU)                  DNP3 Portal
```

AWS Site-to-Site VPN provides two tunnels for redundancy. The most common configuration is IPSec (IKEv2) with pre-shared key (PSK) authentication, but other VPN types can be supported. Contact SGS for specific requirements.

### 5.2 What You Need to Configure

- Your VPN gateway to establish an IPSec tunnel to the AWS VPN endpoint IPs provided by SGS
- IKE and IPSec parameters (SGS will provide the AWS-side configuration details, or you can accept AWS defaults)
- Routing for the portal's subnet through the VPN tunnel

### 5.3 Pre-Shared Key (PSK)

SGS will coordinate PSK exchange through a secure channel. Each of the two tunnels has its own PSK.

**PSK handling:**
- Do not store PSKs in plain text in documentation, wikis, or ticketing systems
- PSKs should be rotated at minimum annually, or per your security policy
- SGS will coordinate PSK rotation in advance. Expect a brief tunnel drop (typically under 5 minutes per tunnel) during rotation.

---

## 6. Monitoring and Verification

### 6.1 Verifying the Connection

After the initial connection is established, verify:

1. TLS handshake succeeds (your master connects without certificate errors)
2. Heartbeat signal is active (AI point 0 counting from 0 to 100)
3. Device data appears on your master with reasonable values
4. If unsolicited responses are enabled: your master receives push updates. To test, trigger a magnet reset on a field device and confirm the update arrives without polling.

### 6.2 Ongoing Monitoring

- **Heartbeat:** If the heartbeat on AI point 0 stops changing, the portal process may have stopped or the connection may be down. Contact SGS.
- **Connection stability:** The portal maintains a persistent session. Frequent disconnects and reconnects may indicate a network issue or certificate problem.
- **Data reasonableness:** Implement checks on received values. A fault current of 999,999 A or a negative battery voltage should trigger an alarm, not be accepted as valid.
- **DNP3 protocol monitoring:** If you have a DNP3-aware intrusion detection system (e.g., Claroty, Dragos, or Zeek), configure it to flag any unexpected object types or function codes. The portal only serves AI points, so any other object type in a response would be anomalous.

---

## 7. Certificate Rotation

Certificates have a configurable validity period (default: 10 years). SGS will coordinate rotation before expiry.

### 7.1 What to Expect

1. SGS generates new certificates and provides a new certificate package
2. A maintenance window is coordinated. The switchover is typically brief (15-30 minutes).
3. You import the new certificates on your DNP3 master
4. SGS restarts the portal with the new certificates
5. Both sides verify the connection is re-established

### 7.2 Emergency Re-Keying

If a certificate is suspected of being compromised:

1. SGS generates new certificates immediately and restarts the portal. The old certificates are instantly invalidated.
2. SGS provides new certificates through a secure channel.
3. The DNP3 session will be down until you install the new certificates.
4. Both sides document the incident and timeline.

If you suspect your private key has been compromised, contact SGS immediately so we can begin re-keying.

### 7.3 VPN PSK Rotation

For VPN deployments, PSK rotation follows a similar coordinated process. SGS will initiate rotation at minimum annually and coordinate timing to minimize downtime.

---

## 8. Incident Reporting

### 8.1 What to Report

Contact SGS immediately if you observe:

- Certificate errors or unexpected TLS handshake failures
- Data that appears incorrect, unexpected, or inconsistent with field conditions
- Connection behavior that seems abnormal (unexpected disconnects, connections from unexpected sources)
- Any suspected compromise of your private key or certificate files

### 8.2 How to Reach Us

- **Security incidents:** security@smartgridsolutions.com (monitored by the SGS engineering team)
- **General questions:** info@smartgridsolutions.com
- **Certificate requests:** Coordinated through your SGS account contact

### 8.3 What SGS Will Do

When a security incident is reported:

1. SGS acknowledges the report within 4 business hours and notifies your designated security contact
2. A shared incident timeline is maintained
3. Both parties share relevant logs and indicators
4. If certificate compromise is suspected, emergency re-keying is initiated immediately
5. Root cause analysis and remediation are documented
6. Post-incident review to improve defenses

---

Once your connection is verified, the portal delivers data continuously with no further configuration needed. For questions at any time, contact your SGS account team.
