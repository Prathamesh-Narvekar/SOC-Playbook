# Distributed Denial of Service (DDoS) Playbook

## Description
This playbook outlines the response procedures for investigating Distributed Denial of Service (DDoS) attacks. These attacks aim to overwhelm the organization’s network bandwidth, perimeter defenses, or web applications, rendering them inaccessible to legitimate users. Triggers typically include high-volume alerts from Anti-DDoS appliances, WAFs, or external monitoring tools (e.g., Pingdom, Datadog) reporting that customer-facing banking portals are down or experiencing severe latency.

## Possible Threat
The impact of a DDoS attack on a financial institution is immediate and highly visible. The threats include:
* **Service Disruption:** Customer banking portals, mobile applications, or payment gateways going offline, leading to reputational damage and financial loss.
* **Resource Exhaustion:** Crashing critical perimeter defenses (firewalls/load balancers) by exhausting their connection state tables.
* **Smokescreening:** Attackers using the chaos and alert fatigue of a DDoS attack to mask a secondary, more targeted attack (e.g., data exfiltration or account takeover).

## Log Sources
_Important Note for Analysts: Standard perimeter firewalls often crash under volumetric DDoS attacks before they can log the traffic. Investigation relies heavily on upstream edge defenses and flow data._
* **Anti-DDoS / Edge Protection:** Alerts and traffic graphs from specialized scrubbers like Cloudflare, Akamai, Arbor Networks, or Radware.
* **Web Application Firewalls (WAF):** Logs tracking HTTP/HTTPS request rates, response times, and HTTP 503/504 errors.
* **NetFlow / sFlow:** Core router telemetry showing anomalous spikes in bandwidth utilization (Gbps) or packet rates (pps).

## Initial Triage
L1 Analysts must quickly confirm if the outage is a malicious attack or a legitimate traffic spike:
* **Verify the Outage:** Test the targeted application externally (e.g., attempt to load the bank's homepage). Are you getting timeouts or 5xx errors?
* **Identify the Attack Type:**
  * Volumetric/Network Layer (L3/L4): Massive spikes in bandwidth (UDP floods, ICMP floods, SYN floods).
  * Application Layer (L7): Massive spikes in web requests (HTTP GET/POST floods) overwhelming the backend database, often with relatively low overall bandwidth.
* **Rule Out Benign Causes:** Verify with the client if there is an active marketing campaign, a new app release, or a known infrastructure misconfiguration causing the traffic spike.

## Investigation Steps
The investigation workflow is divided across SOC tiers to analyze the traffic profile and coordinate mitigation:
* **L1 Analyst** - Traffic Scoping: Identify the targeted public IPs or URLs. Document the current attack volume (Gbps or Requests per Second). Check SIEM dashboards to see if the attack is utilizing a specific protocol or targeting a single service.
* **L2 Analyst** - Signature & Pattern Analysis: Analyze the malicious traffic for common denominators. Is the traffic originating from a specific geographic region? Are the application-layer requests sharing a unique, anomalous User-Agent string or HTTP header?
* **L3 Analyst** - Smokescreen Check & Coordination: While the network team handles the DDoS mitigation, L3 must actively hunt for secondary attacks. Check for unusual administrative logins, failed brute-force attempts on VPNs, or database exfiltration happening simultaneously.

## Indicators of Compromise (IoCs)
Unlike traditional attacks, DDoS IoCs are often volumetric patterns rather than specific malicious files. Extract these for WAF/Firewall tuning:
* **Targeted Assets:** The specific VIPs (Virtual IPs) or domain names being flooded.
* **Top Attacking ASNs / Geographies:** The primary networks or countries generating the junk traffic.
* **Traffic Signatures:** Specific packet sizes, repetitive URI paths being requested, or suspicious User-Agents (e.g., default Python/cURL libraries).
* **Protocol Anomalies:** An abnormal ratio of SYN packets to ACK packets, or massive amounts of fragmented UDP traffic.

## Containment
Execute these steps immediately to absorb or drop the malicious traffic and restore service availability:
* **Activate DDoS Mitigation:** Engage the upstream ISP or cloud mitigation provider (e.g., enable Cloudflare's "Under Attack" mode) to route traffic through scrubbing centers.
* **Implement Rate Limiting:** Apply aggressive rate-limiting rules on the WAF or Load Balancer to drop requests exceeding normal baseline thresholds per IP.
* **Geo-Blocking:** If the application serves a regional customer base (e.g., only India and the US), temporarily drop all traffic originating outside those sanctioned countries at the edge.
* **BGP Blackholing (RTBH):** In extreme cases where a single IP is targeted and threatening the entire data center, work with the ISP to route all traffic destined for that IP to a "black hole" to save the rest of the network.

## Remediation
Once the attack volume subsides and services are restored:
* **Resource Scaling:** If the attack targeted application resources, temporarily scale up cloud compute instances or load balancer capacities to handle residual spikes.
* **Stakeholder Communication:** Provide the client's incident commander with an impact summary (downtime duration, peak attack volume) for internal reporting and customer communications.
* **Disable Emergency Controls:** Carefully roll back strict geo-blocks or "Under Attack" modes (like global CAPTCHAs) to restore a frictionless experience for legitimate banking customers.

## Detection Improvement
Update edge controls to automate defense against future volumetric attacks:
* **Tune Mitigation Thresholds:** Adjust the auto-mitigation triggers on the Anti-DDoS appliances so they engage faster before backend servers are overwhelmed.
* **Implement Bot Management:** Deploy advanced bot-mitigation tools on the WAF that challenge headless browsers and automated scripts without disrupting human users.
* **Establish Baselines:** Continuously update the "known good" traffic baselines in the NetFlow monitors to ensure accurate alerting during seasonal traffic spikes (e.g., end-of-month payroll processing).
