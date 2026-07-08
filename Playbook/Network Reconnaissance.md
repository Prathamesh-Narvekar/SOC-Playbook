# Description
This playbook outlines the response procedures for investigating external reconnaissance activity, specifically high-volume TCP or UDP port scans targeting public-facing infrastructure. These alerts trigger when an external source IP attempts to connect to multiple destination ports sequentially or simultaneously across one or more servers within a short timeframe.

# Possible Threat
While port scanning itself does not cause direct system damage, it represents the critical initial phase of a cyberattack. The potential threats include:
* **Vulnerability Identification:** Attackers discovering open ports running unpatched or vulnerable services (e.g., exposed administrative portals, database protocols, or unmapped web applications).
* **Network Mapping:** Exposure of the network topology, allowing threat actors to map out the external perimeter and plan subsequent targeted exploits.
* **Targeted Exploitation:** A successful scan is typically followed by automated exploit or brute-force attempts against any discovered open ports.

# Log Sources
_**Important Note for Analysts:** The visibility of scanning traffic depends on where the logs are captured. Perimeter devices see all dropped connection attempts, while endpoint or server logs only see traffic that successfully bypassed firewall filters._
* **Perimeter Firewalls:** Traffic logs, dropped/denied packet logs, and built-in reconnaissance/threat logs from edge defenses (e.g., Palo Alto Networks, FortiGate).
* **Web Application Firewalls (WAF):** Connection and access logs from public-facing web infrastructure.
* **SIEM / NetFlow Aggregators:** Centralized network flows showing traffic patterns, connection counts, and byte distributions.

# Initial Triage
L1 analysts must quickly isolate the alert context and rule out authorized testing:
* **Verify Alert Volume:** Confirm that the traffic consists of an anomalous spike in connections or connection drops from a single external IP across multiple ports or hosts.
* **Check the Targeted Assets:** Determine if the targeted IPs belong to active production servers, internal DMZ infrastructure, or unassigned external IP space (acting as a network telescope).
* **Rule Out Authorized Scans:** Cross-reference the source IP against the client's internal vulnerability scanning schedule, automated security posture assessment tools, or scheduled penetration testing windows.

# Investigation Steps
The investigation workflow is divided across the SOC tiers to analyze traffic volume, determine success, and assess post-scan risk:
* **L1 Analyst** - Traffic Scoping: Identify the total count of targeted destination ports. Determine if it is a horizontal scan (scanning one specific port across many IP addresses) or a vertical scan (scanning many ports on a single host IP address). Document the top targeted ports.
* **L2 Analyst** - Connectivity & Payload Analysis: Review the firewall actions in the SIEM logs. Did the firewall block/drop the packets, or did it allow the connection? If any connection was allowed, check the session bytes transferred. A successful 3-way handshake with data transfer indicates an open port that requires immediate attention. Evaluate the reputation of the external source IP using OSINT tools (VirusTotal, AbuseIPDB, GreyNoise) to check if it belongs to a known malicious scanner, a Tor exit node, or a commercial VPN.
* **L3 Analyst** - Deep Host Inspection: If a connection to a port was established, pivot to the target server's local application or operating system logs to inspect what payload or command was sent immediately after the port was found open. Check for subsequent exploit payloads or authentication attempts.

# Indicators of Compromise (IoCs)
Extract these artifacts from the SIEM and document them in the incident ticket for blocklist integration and threat tracking:
* **Attacker Source IPs:** The external IP addresses conducting the scan.
* **Scan Profile:** The specific list of target ports (e.g., focusing on database ports like 3306/1433, or administrative ports like 22/3389).
* **Originating ISP/ASN:** The hosting provider or network hosting the scanning infrastructure (often commercial cloud providers or compromised servers used as proxies).

# Containment
Execute these steps immediately to block the reconnaissance and close perimeter gaps:
* **Perimeter Blocking:** Immediately add the scanning source IP to the emergency blocklist on the perimeter firewalls (Palo Alto/FortiGate) or Web Application Firewall (WAF).
* **Close Exposed Ports:** If the scan revealed an open port running an unauthorized or unnecessary service, modify the network policy or host firewall rules to close or restrict access to that port immediately.
* **Temporary Geo-IP Restrictions:** If the scanning activity is a massive distributed campaign originating from an unexpected country where the client has no operations, apply a temporary geographic block on incoming traffic from that nation if authorized by policy.

# Remediation
Once the immediate scanning behavior is mitigated, harden the exposed assets:
* **Targeted Vulnerability Scan:** Run an immediate vulnerability sweep on any servers that had ports exposed to ensure the running services are fully patched against known remote code execution (RCE) exploits.
* **Harden Host Configurations:** Disable unnecessary public-facing services and ensure that critical administrative protocols (like SSH or RDP) are restricted behind a VPN or multi-factor authentication (MFA) gateway.
* **Rotate Exposed Credentials:** If an administrative portal or service was found open and accessible during the scan, proactively rotate the credentials associated with that service as a precautionary measure.

# Detection Improvement
Update security controls to automate defense against future scanning campaigns:
* **Tune Firewall Thresholds:** Adjust the firewall's native reconnaissance protection policies (e.g., zone protection profiles) to automatically drop traffic from IPs that trigger a scan threshold before it floods the SIEM.
* **Implement Honeypots:** Deploy low-interaction honeypots inside the network perimeter to catch early scanning activity and automatically feed those malicious IPs into a dynamic blocklist.
* **Optimize SIEM Correlation:** Group port scan alerts by Source IP over a longer window (e.g., 24 hours) to catch slow, stealthy scans ("low and slow" scanning) that deliberately bypass short-duration threshold rules.
