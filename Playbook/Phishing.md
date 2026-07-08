# Phishing Playbook

## Description
This playbook outlines the response procedures for investigating suspicious emails designed to steal credentials, commit financial fraud, or deliver malicious payloads. Alerts for this playbook are typically triggered by user reports (e.g., clicking a "Report Phish" button), Secure Email Gateway (SEG) detections, or SIEM alerts flagging anomalous inbox rules.

## Possible Threat
If a phishing campaign successfully bypasses perimeter defenses and a user interacts with it, the threats to the financial environment include:
* **Credential Harvesting:** Stealing employee logins to achieve Account Takeover (ATO) and access banking systems.
* **Malware / Ransomware Infection:** Execution of malicious attachments (e.g., macro-enabled Office documents, ISOs, or ZIPs) leading to system compromise.
* **Business Email Compromise (BEC):** Spoofing executives or vendors to authorize fraudulent wire transfers.

## Log Sources
 _Important Note for Analysts: Email architecture varies by client. Verify if the client uses native cloud security (e.g., Microsoft Defender for Office 365) or a third-party gateway._
* **Secure Email Gateways (SEG):** Logs from Proofpoint, Mimecast, or Cisco Secure Email.
* **Raw Email Evidence:** The .msg or .eml file containing the full SMTP headers and payload.
* **Web Proxies / Firewalls:** Logs showing whether a user successfully clicked and navigated to a malicious URL.
* **Endpoint Detection (EDR):** Telemetry from CrowdStrike or Defender to verify if a downloaded attachment was executed.

## Initial Triage
L1 Analysts must quickly validate the email's intent and establish if it is a true threat or benign spam:
* **Analyze the Sender:** Check the From address, Reply-To address, and the actual sending IP in the email headers. Is the domain newly registered, slightly misspelled (typosquatting), or failing SPF/DKIM/DMARC authentication?
* **Evaluate the Payload:** Identify if the threat relies on a malicious link (URL) or an attachment.
* **Determine Scope:** Query the SEG or SIEM to see if this email was sent to a single VIP (Targeted Spear-Phishing) or hundreds of employees (Mass Campaign).

## Investigation Steps
The investigation workflow is divided across SOC tiers to safely analyze the threat and track the blast radius:
* **L1 Analyst** - Artifact Extraction & Sandboxing: Safely extract URLs and file hashes without executing them on your local machine. Query open-source intelligence (VirusTotal, URLScan.io). If an attachment requires deeper inspection, safely detonate it in an isolated sandbox environment (e.g., Kali Linux in VirtualBox, Any.Run, or Joe Sandbox) to observe its behavior.
* **L2 Analyst** - Blast Radius & Interaction: Query the SIEM and web proxy logs to determine who interacted with the email. Did anyone click the link? Did the firewall block the resulting web traffic? If credentials were submitted, check Identity logs (Event ID 4624 or Entra ID) for successful logins from unusual IPs.
* **L3 Analyst** - Advanced Forensics: For targeted attacks, reverse-engineer the malware dropped by the attachment or analyze the Command and Control (C2) beaconing traffic to identify the specific threat actor group targeting the bank.

## Indicators of Compromise (IoCs)
Extract these artifacts and document them in the ticketing system for immediate blocklist integration:
* **Sender Infrastructure:** The malicious sending IP address and the exact From/Reply-To email addresses.
* **Network Indicators:** The full malicious URLs, domains, and any IP addresses the malware attempts to contact.
* **Host Artifacts:** The SHA-256 file hashes of the malicious attachments and the names of any dropped files on the endpoint.
* **Email Metadata:** The exact Subject Line and the specific Message-ID.

## Containment
Execute these steps immediately to neutralize the email campaign across the client's tenant:
* **Purge the Email:** Execute a "Hard Delete" or "ZAP" (Zero-hour Auto Purge) via Microsoft 365 or the SEG to pull the email out of all employee inboxes globally.
* **Block the Infrastructure:** Add the malicious sender domain, URLs, and associated IP addresses to the organization's Email Gateway, WAF, and Perimeter Firewall blocklists.
* **Isolate Compromised Hosts:** If an employee executed a malicious attachment, immediately isolate their workstation from the network using the EDR console to prevent lateral movement.

## Remediation
To ensure the threat is fully eradicated and the user is secured:
* **Credential Reset:** If the user clicked a link and entered their password, force an immediate password reset and clear their active MFA sessions.
* **Endpoint Sweeps:** Run a full EDR anti-virus sweep on any host that interacted with the email to ensure no persistent malware or registry changes were left behind.
* **User Notification:** Inform the user that the email was malicious, confirm their workstation is secure, and recommend targeted security awareness training if they repeatedly click phishing links.

## Detection Improvement
Update email security controls to automatically block similar campaigns in the future:
* **Tune DMARC/SPF Policies:** If the email successfully spoofed an internal bank domain, work with the messaging team to enforce strict DMARC "Reject" policies.
* **Deploy YARA/SIEM Rules:** Create custom alerts that flag specific combinations of the attacker's language, subject lines, or attachment naming conventions.
* **Gateway Hardening:** Ensure the SEG is configured to automatically sandbox and detonate unknown executable files and block auto-forwarding rules to external domains.
