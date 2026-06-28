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

