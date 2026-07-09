# Insider Threat Playbook

## Description
This playbook outlines the response procedures for investigating internal users (employees, contractors, or compromised accounts) transferring sensitive corporate or customer data outside the organization's trusted boundary. Alerts typically trigger from Data Loss Prevention (DLP) engines, Cloud Access Security Brokers (CASB), or EDR device control modules flagging large uploads or unauthorized USB usage.

## Possible Threat
Insider exfiltration bypasses traditional perimeter defenses because the user already has authorized access. The threats include:
* **Regulatory Violations:** Massive fines under GDPR, PCI-DSS, or financial regulations due to the exposure of customer Personally Identifiable Information (PII) or credit card data.
* **Corporate Espionage:** Theft of proprietary source code, trading algorithms, or strategic financial documents by departing employees.
* **Extortion:** If the "insider" is actually a threat actor who achieved account takeover (ATO) and is exfiltrating data to hold the bank for ransom.

## Log Sources
_**Important Note for Analysts:** Insider threat investigations require a combination of network telemetry, endpoint monitoring, and human context (HR status)._
* **Data Loss Prevention (DLP):** Solutions like Symantec DLP, Microsoft Purview, or Forcepoint for alerts on sensitive data patterns (e.g., Credit Card numbers, SSNs).
* **Endpoint Detection & Response (EDR):** Logs for USB mass storage connections and local file archiving (e.g., creating large .zip or .rar files).
* **Web Proxies / CASB:** Logs showing file uploads to unsanctioned cloud storage (e.g., personal Google Drive, Dropbox, Mega).
* **Email Gateways:** Logs tracking internal users sending large attachments to personal domains (e.g., @gmail.com or @yahoo.com).

## Initial Triage
L1 Analysts must quickly validate if the data transfer is a legitimate business process or a severe security violation:
* **Verify the Payload:** What type of data triggered the alert? (e.g., highly confidential financial reports vs. a benign corporate PowerPoint).
* **Check the Destination:** Did the data go to a sanctioned vendor or an unsanctioned personal cloud/email account?
* **Assess the User's Role:** Is it normal for this user to handle this volume of data? (e.g., a database administrator exporting logs vs. a standard teller downloading a massive customer database).
* **Rule Out ATO:** Confirm the user's recent login locations and MFA logs to ensure their account wasn't compromised by an external attacker.

## Investigation Steps
Because insider threats carry legal and HR implications, the investigation must be handled with strict confidentiality across the SOC tiers:
* **L1 Analyst** - Alert Scoping: Document the exact file names, sizes, destination, and the timeframe of the exfiltration. Determine the specific mechanism used (USB, web upload, email).
* **L2 Analyst** - Behavioral Context & Lineage: Review the user's baseline. Are they accessing databases or SharePoint folders they don't normally touch? Cross-reference the user with the organization's HR departmet to check if they are on a "flight risk" or recent termination list. Look for precursor activities, such as installing unauthorized software (e.g., FTP clients) or archiving tools.
* **L3 Analyst** - Forensics & Legal Coordination: If malicious intent is suspected, freeze the user's endpoint state. Do not alert the user. Coordinate directly with the client's internal Insider Threat, HR, and Legal teams to conduct a covert forensic image of the workstation.

## Indicators of Compromise / Artifacts
Unlike external attacks, insider threats rely on "Indicators of Behavior" and specific data artifacts. Extract these for the incident ticket:
* **Destination URLs / IPs:** Unsanctioned cloud storage domains or FTP IP addresses.
* **External Email Addresses:** The specific personal email addresses used to receive company data.
* **Hardware IDs:** The exact Serial Number and Vendor ID of unauthorized USB drives plugged into the endpoint.
* **File Metadata:** Hashes, exact file names, and the total byte count of the exfiltrated data.

## Containment
Execution of containment for insider threats must be approved by the client's HR/Legal teams unless it is an active, massive data hemorrhage.
* **Network Isolation:** Use the EDR console to isolate the employee's workstation from the internet and internal network, halting any active uploads.
* **Account Suspension:** Suspend the user's Active Directory and Cloud Identity accounts.
* **Proxy/Firewall Blocking:** Block the specific unsanctioned destination URL or IP address on the perimeter proxy to stop the active transfer.
* **Physical Access Revocation:** Work with physical security to disable the employee's badge access to corporate facilities.

## Remediation
To mitigate the impact of the data loss:
* **Data Retrieval/Wipe:** If the data was sent to a known personal device (like a mobile phone with MDM installed), execute a remote corporate wipe.
* **Password Resets:** If the exfiltrated data contained network credentials, API keys, or service account passwords, immediately rotate them.
* **Legal Escalation:** Hand the forensic evidence over to the client's Legal and HR departments for potential termination, litigation, or law enforcement involvement.

## Detection Improvement
Refine controls to catch exfiltration attempts earlier in the kill chain:
* **Strict USB Policies:** Enforce a global "Read-Only" or complete block policy for all USB mass storage devices via EDR or Group Policy, with strict exceptions managed by IT.
* **Tune DLP Regex:** Update DLP dictionaries and regular expressions to better identify proprietary bank documents, account numbers, and SWIFT codes.
* **Implement CASB Controls:** Block uploads (allow only downloads) to unsanctioned personal cloud storage categories on the web proxy.
