# Brue-Force Playbook

## Description
This playbook outlines the response procedures for suspected Brute-Force or Password Spraying attacks against the bank's authentication infrastructure.
* Brute-Force: An attacker attempts hundreds or thousands of passwords against a single high-value account.
* Password Spraying: An attacker attempts a few common passwords (e.g., BankName2024!) across hundreds of user accounts to avoid triggering standard account lockout policies.

Triggers for this playbook typically include SIEM alerts for rapid consecutive failed logins, unexpected account lockouts, or anomalous authentication volumes.

## Possible Threat
If a brute-force or password spray attack is successful, the immediate threats to the banking environment include:
* **Account Takeover (ATO):** Unauthorized access to sensitive financial systems, internal portals, or customer data.
* **Lateral Movement:** Compromise of a low-level account used as a beachhead to access critical infrastructure (e.g., SWIFT systems or payment gateways).
* **Ransomware Deployment:** Escalation of privileges leading to domain-wide compromise and ransomware execution.

## Log Sources
To effectively investigate this alert, analysts must query the following telemetry sources via the SIEM:

<i>Important Note for Analysts: The exact availability of telemetry varies significantly based on the client’s specific enterprise architecture and vendor stack (e.g., whether the bank uses an on-premises Active Directory setup, a hybrid cloud environment, or specific perimeter security vendors). Always cross-reference the client's asset management database or SIEM log source management dashboard to verify active ingestion.</i>
* **Windows Security Logs (Domain Controllers):** Event IDs 4625 (Failed Logon), 4624 (Successful Logon), 4740 (Account Lockout), and 4776 (Credential Validation).
* **Identity Providers (IdP):** Azure AD / Entra ID Sign-in logs, Okta system logs.
* **Perimeter Defenses:** VPN authentication logs (e.g., Cisco AnyConnect, Palo Alto GlobalProtect), Web Application Firewall (WAF) logs, and proxy logs.

## Initial Triage
L1 Analysts must immediately validate the alert and determine its urgency:
* **Identify the Target:** Is the targeted account a standard user, a VIP (e.g., Bank Executive), or a Service Account? (Service accounts are critical; human error rarely causes hundreds of failed service account logins).
* **Check the Source:** Is the authentication attempt originating from the internal network (potential lateral movement) or an external IP?
* **Identify the Source Process (Stale Credentials Check):** If the attempts originate internally, identify which specific process or service is triggering the authentication. High volumes of failed logons are frequently benign, caused by stale passwords cached in background services, mapped network drives, or scheduled tasks on a user's workstation.
* **Verify Success:** Did the SIEM log a successful logon (4624) after the string of failures? If yes, escalate to L2 immediately—the account is likely compromised.

## Investigation Steps

### L1 SOC Analyst (Triage & Initial Validation)
The primary goal of the L1 analyst is to validate the alert, confirm the technical details, and rule out immediate false positives (e.g., an employee's mobile device or background application repeatedly attempting to authenticate using outdated or cached credentials, generating failure events, before the user manually enters their newly updated password to successfully log in).
* **Validate the Alert Sequence:** Review the SIEM raw logs to confirm that the failed attempts (e.g., Windows Event ID 4625) and the successful login (Event ID 4624) genuinely occurred within the 5-minute window from the exact same Source IP.
* **Analyze the Source IP:** Query Threat Intelligence feeds (e.g.: VirusTotal, AbuseIPDB) to determine if the IP address is a known malicious actor, part of a botnet, a Tor exit node, or a commercial VPN often used by attackers.
* **Identify the Target Asset & User:** Document the specific client service accessed (e.g., VPN gateway, Microsoft 365, external web portal) and the specific username(s) that were successfully breached.
* **Geographic Context:** Check the geolocation of the Source IP. If it originates from a high-risk country or a location where the client has no operations, treat it as highly suspicious.
* **Escalation:** If the alert is deemed a True Positive (a confirmed login from a suspicious or unrecognized IP), immediately escalate the incident to L2 as a confirmed Account Takeover.

### L2 SOC Analyst (In-Depth Analysis & Threat Hunting)
The L2 analyst takes over to determine the scope of the compromise, assess the risk to the organization, and hunt for any malicious actions the attacker took after logging in.
* **Assess Account Privileges:** Cross-reference the compromised username with the client's Active Directory/IAM team. Determine if the account belongs to a standard user, a privileged IT Administrator, a VIP (C-Suite), or a critical systems operator (e.g., SWIFT operator).
* **Investigate MFA Status:** Review authentication logs (e.g., Azure AD or Duo logs) to determine how the attacker bypassed Multi-Factor Authentication. Look for evidence of "MFA Fatigue" (prompt bombing), session hijacking (Pass-the-Cookie), or legacy protocol exploitation where MFA is not enforced.
* **Hunt for Post-Compromise Activity:** Analyze the SIEM logs for the 30–60 minutes immediately following the successful login. Hunt for:
  * **Lateral Movement:** Attempts to access other internal servers or databases.
  * **Data Exfiltration:** Unusual file downloads or access to sensitive file shares.
  * **Email Manipulation:** Creation of malicious inbox forwarding rules (common in O365 compromises) to hide future attacker communications.
* **Escalation:** Upgrade the incident severity to Critical/High based on the account privilege and escalate to L3 for immediate technical containment.

### L3 SOC Analyst / Incident Responder (Containment & Coordination)
The L3 analyst acts as the Incident Commander. Their objective is to immediately sever the attacker's access, secure the environment, and coordinate directly with the client's internal security teams.
* **Account Containment (Immediate):** Request or execute the immediate suspension (disabling) of the compromised Active Directory/IAM account to prevent further access.
* **Session Revocation:** Force a global logout for the compromised user, terminating all active session tokens across Microsoft 365, VPNs, and internal web applications.
* **Network Containment:** Block the malicious Source IP(s) on the perimeter firewalls, Web Application Firewalls (WAF), and Intrusion Prevention Systems (IPS).
* **Endpoint Isolation:** If the attacker successfully logged into a corporate workstation (e.g., via VDI or VPN), use the Endpoint Detection and Response (EDR) tool to isolate the host from the network.
* **Client Coordination:** Notify the internal IAM, Anti-Fraud, and CISO teams as dictated by the Escalation Matrix. If financial systems were accessed, an immediate emergency bridge must be opened.
* **Root Cause Analysis (RCA) & Closure:** Draft the final RCA detailing exactly how the password was compromised, how MFA was bypassed, what systems were accessed, and the mandatory remediation steps (e.g., enforcing stronger MFA policies) before securely closing the incident.

## Indicators of Compromise (IoCs)
Proper documentation of IoCs is critical for immediate containment and identifying if the threat actor is targeting other clients or systems. Extract the following artifacts from the SIEM (e.g., QRadar) and Identity Provider logs, and document them thoroughly in the incident ticket:

### Network & Infrastructure Artifacts
* **Attacker Source IPs:** The specific IP addresses generating the failed/successful authentication attempts.
* **Autonomous System Numbers (ASNs):** Identify if the attack is originating from a specific hosting provider (e.g., DigitalOcean, AWS), a commercial VPN service, or an anonymization network (e.g., Tor exit nodes).
* **Suspicious User-Agent Strings:** Note the User-Agent used during the authentication request. Attackers often use outdated browser versions, automated scripting tools (e.g., python-requests, curl), or custom user-agents that do not match the bank’s standard operating environment.

### Identity & Authentication Artifacts:
* **Compromised Accounts:** The exact User Principal Names (UPNs) or Active Directory usernames that experienced a successful logon (Event ID 4624) after a brute-force sequence.
* **Targeted Account Lists (For Password Sprays):** If a spray attack occurred, export the list of all targeted usernames. Compare this list against public data breaches to determine if the attacker is using a specific compromised credential dump.
* **Rogue Device IDs / Authenticator Registrations:** If the attacker successfully bypassed authentication, check Azure AD / Entra ID logs to see if a new, unauthorized MFA device or mobile phone was registered to the account.

### Post-Compromise Artifacts (If Account Takeover Occurred):
* **Malicious Inbox Rules:** Specific names of any mailbox rules created immediately following the login (e.g., forwarding emails containing words like "invoice," "password," or "security" to the RSS Feeds folder or an external address).
* **Session Tokens/IDs:** The specific session IDs associated with the malicious login, which will be required for global session revocation.
