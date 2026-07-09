# Privilege Escalation

## Description
This playbook outlines the response procedures for investigating instances where a standard user (or compromised account) attempts to gain elevated rights (e.g., Local Administrator, Domain Admin, or root). Privilege escalation can occur through exploiting system vulnerabilities, bypassing User Account Control (UAC), or unauthorized modifications to Active Directory (AD) security groups.

## Possible Threat
Without administrative privileges, an attacker's impact is limited to a single user's scope. If they successfully escalate, the threats include:
* **Security Evasion:** The attacker can turn off Endpoint Detection and Response (EDR) sensors, modify firewall rules, and clear event logs to hide their tracks.
* **Credential Dumping:** Gaining the ability to read LSASS memory to steal passwords for other accounts (e.g., via Mimikatz).
* **Enterprise Takeover:** Elevating to Domain Admin grants total control over the bank's Active Directory, allowing the attacker to deploy ransomware globally or access core banking servers.

## Log Sources
_Important Note for Analysts: This attack often blends in with normal IT administration. You must rely heavily on AD auditing and EDR telemetry._
* **Windows Security Logs:**
  * Event ID 4672 (Special privileges assigned to new logon).
  * Event IDs 4728 / 4732 / 4756 (A member was added to a privileged security group).
  * Event ID 4688 (Process Creation with command-line auditing enabled).
* **Endpoint Detection & Response (EDR):** Process execution logs capturing token manipulation, UAC bypass techniques, or memory injection.
* **Linux / Unix Logs:** /var/log/auth.log or /var/log/secure focusing on unauthorized sudo or su usage.

## Initial Triage
L1 Analysts must quickly determine if the action was a rogue attack or approved IT maintenance:
* **Identify the Actor:** Who is the user? Is it a standard employee, a service account, or a known IT administrator?
* **Check for Authorized Changes:** Cross-reference the activity with the ticketing system. Is there an approved Change Request (CR) for this user to be granted temporary admin rights?
* **Analyze the Method:** Did the privilege change happen natively via Active Directory, or did the EDR flag a suspicious exploit running on a local endpoint?

## Investigation Steps
The investigation workflow is divided across SOC tiers to analyze how the escalation occurred and what the attacker did next:
* **L1 Analyst** - Alert Scoping: Document the exact user, the compromised endpoint, the specific privilege gained (e.g., added to the Domain Admins group), and the timestamp.
* **L2 Analyst** - Process & Exploit Analysis: Pivot to the EDR console. Check the process tree leading up to the alert. Look for anomalous child processes, such as a vulnerable service (e.g., spoolsv.exe for PrintNightmare) suddenly spawning a command prompt (cmd.exe or powershell.exe).
* **L3 Analyst** - Post-Escalation Hunting: Hunt for what the attacker did after gaining admin rights. Look for attempts to dump credentials (LSASS access), creation of new backdoor accounts, or lateral movement to Domain Controllers.

## Indicators of Compromise (IoCs)
Extract these artifacts from the SIEM/EDR and document them in the incident ticket:
* **Suspicious Commands:** Command-line execution adding users to local groups (e.g., net localgroup administrators attacker /add).
* **Exploit Tooling:** File names or SHA-256 hashes of known privilege escalation scripts (e.g., WinPEAS, LinPEAS, BloodHound).
* **Anomalous Processes:** Standard users executing binaries typically reserved for system administration (e.g., vssadmin.exe, ntdsutil.exe).

## Containment
Execute these steps immediately to strip the attacker's power and trap them:
* **Revert Privileges:** Immediately remove the compromised user from the privileged Active Directory or local security group.
* **Suspend the Account:** Disable the compromised account entirely and execute a global session revocation to kill any active administrative sessions.
* **Host Isolation (EDR):** Isolate the workstation where the privilege escalation exploit was executed from the network to prevent the attacker from pivoting to other servers.

## Remediation
To ensure the environment is secure and the attacker's administrative foothold is destroyed:
* **Re-image the Host:** Never trust a machine where an attacker achieved "SYSTEM" or "root" level access. Wipe and re-image the machine from a known-good baseline.
* **Audit AD Groups:** Conduct a full audit of all critical AD groups (Domain Admins, Enterprise Admins, Backup Operators) to ensure no stealthy backdoor accounts were created.
* **Credential Rotation:** Force a password reset for the compromised user and any service accounts present on the compromised machine.

## Detection Improvement
Harden the network to prevent standard users from gaining administrative rights:
* **Implement LAPS:** Deploy Microsoft Local Administrator Password Solution (LAPS) to remove static local admin passwords across all workstations.
* **SIEM Group Auditing:** Create high-severity, un-ignorable SIEM alerts for any modification to critical Active Directory groups that occur outside of a dedicated service account or approved IT admin.
* **Restrict Tooling:** Use Application Control (e.g., AppLocker or EDR policies) to block standard users from executing command-line scripting tools (like PowerShell) unless specifically required for their job role.
