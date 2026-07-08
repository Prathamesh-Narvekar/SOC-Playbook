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
Extract these artifacts from the SIEM and document them in the incident ticket for immediate containment and future threat hunting:
* **Attacker Source IPs:** The specific external addresses or internal host IPs (e.g., 192.168.41.222) generating the authentication traffic.
* **Compromised Accounts:** The exact usernames (e.g., Hp ProOne) or Active Directory User Principal Names (UPNs) that logged a successful authentication after the brute-force sequence.
* **Suspicious User-Agents:** Automated scripting tools (like cURL or Python requests), custom scripts, or outdated browsers identified in the HTTP/HTTPS authentication headers.
* **Malicious Inbox Rules:** Any unauthorized email forwarding rules created immediately after a successful login (often used to hide security alerts or steal invoices).
* **Rogue MFA Devices:** Any new, unrecognized authenticator apps or phone numbers registered to the account post-compromise.


## Containment
Execute these steps immediately to cut off the attacker's access and stop the attack:
* **Disable the Account:** Immediately suspend the targeted user in Active Directory or the Cloud Identity Provider.
* **Revoke Active Sessions:** Force a global logout (e.g., via Azure AD) to kill the attacker's current connection. A password reset alone does not stop an active session.
* **Block the IP:** Add the malicious Source IP to the emergency blocklist on all perimeter firewalls, VPN gateways, and WAFs.
* **Isolate Internal Hosts:** If the brute-force traffic is coming from an internal employee machine, isolate it from the network immediately using your EDR console.

## Remediation
Once the threat is contained, execute these steps to permanently eradicate the attacker's footprint and safely restore business operations:
* **Force a Password Reset:** Require the affected user to set a new, complex password that complies with the organization's security policy. Do not allow them to reuse recent passwords.
* **Clear MFA Registrations:** Delete the user's existing Multi-Factor Authentication (MFA) devices in the Identity Provider and force them to re-register. This ensures the attacker did not add their own device as a backdoor.
* **Post-Compromise Cleanup:** If the account was breached, thoroughly inspect their environment. Delete any malicious inbox forwarding rules, and verify no unauthorized secondary accounts or API keys were created.
* **Restore Access:** Once all security checks are passed and the user is verified, re-enable the account and lift the network isolation on their endpoint (if applicable).

## Detection Improvement
After the incident is closed, update the SIEM and security controls to catch similar attacks earlier and reduce false positives:
* **Alert on Success-After-Failure:** Do not just alert on failed attempts. Create a high-priority SIEM correlation rule that triggers when a successful login (4624) happens immediately after a string of failed logins (4625) from the exact same IP address.
* **Enable Impossible Travel Alerts:** Tune the SIEM or Identity Provider to alert when a user logs in from two geographically distant locations within an impossible timeframe.
* **Disable Legacy Protocols:** Ensure legacy authentication protocols (like IMAP or POP3) are blocked across the tenant, as attackers frequently use them to bypass MFA requirements.
* **Tune Lockout Policies:** Review Active Directory policies to ensure accounts automatically lock out after a reasonable number of failed attempts (e.g., 5 to 10), stopping the brute-force at the system level.
