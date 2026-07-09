# Lateral Movement Playbook

## Description
This playbook outlines the response procedures for investigating threat actors moving internally from one compromised host to another. Attackers use lateral movement to escalate privileges, map the network, and access critical infrastructure. Alerts typically trigger on suspicious Remote Desktop Protocol (RDP) sessions, unauthorized use of administrative tools (e.g., PsExec, WMI), or Pass-the-Hash authentication anomalies.

## Possible Threat
If lateral movement is not contained immediately, a localized endpoint infection becomes a severe enterprise-wide breach. The threats include:
* **Domain Compromise:** Attackers reaching Active Directory Domain Controllers to dump the central credential database (NTDS.dit).
* **Enterprise Ransomware:** Distributing ransomware payloads simultaneously across thousands of bank servers and workstations.
* **Crown Jewel Access:** Pivoting into highly segmented financial networks (e.g., payment gateways, SWIFT, or customer databases) to exfiltrate data or commit fraud.

## Log Sources
_Important Note for Analysts: Detecting lateral movement relies heavily on internal host-to-host telemetry. Perimeter firewalls will not see this traffic; you must rely on endpoint logs and internal network segmentation firewalls._
* **Windows Security Logs:**
  * Event ID 4624 (Successful Logon - specifically looking for Logon Type 3 for Network, or Type 10 for RDP).
  * Event ID 4688 (Process Creation).
  * Event ID 5140 (A network share object was accessed).
* **Endpoint Detection & Response (EDR):** Process execution logs (CrowdStrike, Defender for Endpoint) capturing PowerShell remoting, WMI execution, or SMB file transfers.
* **Internal Firewalls / Core Switches:** Traffic logs between internal VLANs (e.g., User Subnet to Data Center Subnet).

## Initial Triage
L1 Analysts must quickly determine if the activity is a legitimate IT administrator performing maintenance or an attacker pivoting:
* **Identify the Source and Destination:** Is the traffic originating from a standard user laptop but targeting a critical database?
* **Analyze the Identity:** Is the account performing the action a standard user, or a Domain Admin/Service Account?
* **Rule Out Authorized IT Activity:** Cross-reference the activity with approved change control windows. Are Helpdesk or System Administrators using tools like PsExec or RDP for scheduled patching?
* **Verify the Tooling:** Look at the process initiating the connection. An RDP session launched by mstsc.exe might be normal; an SMB connection initiated by powershell.exe or an unknown binary is highly suspicious.

## Investigation Steps
The investigation workflow is divided across SOC tiers to track the attacker's path and establish the "blast radius":
* **L1 Analyst** - Connection Scoping: Query the SIEM for all network connections originating from the suspect internal IP over the last 24 hours. Document every destination IP, port (e.g., 445 for SMB, 3389 for RDP, 5985 for WinRM), and the usernames used for those connections.
* **L2 Analyst** - Process & Execution Analysis: Pivot to the EDR console for the destination hosts. If a connection was successful, did it spawn a new process? Look for suspicious parent-child process relationships (e.g., WmiPrvSE.exe spawning cmd.exe, or services.exe launching an unknown executable).
* **L3 Analyst** - Advanced Credential Hunting: Hunt for signs of credential dumping on the originating host (e.g., LSASS memory access, Mimikatz usage) which the attacker used to harvest the credentials required to move laterally. Check for Pass-the-Hash or Overpass-the-Hash attacks in the Kerberos authentication logs.

## Indicators of Compromise (IoCs)
Extract these artifacts from the SIEM/EDR and document them for containment and enterprise-wide sweeping:
* **Compromised Host IPs:** The specific internal IP addresses used as the "jump box" or beachhead.
* **Compromised Identities:** The specific Active Directory usernames, especially Service Accounts, being abused to authenticate across the network.
* **Malicious Processes & Tools:** Hashes or file names of lateral movement tooling (e.g., PsExec.exe, wmiexec.py, BloodHound, or Cobalt Strike SMB beacons).
* **Suspicious Named Pipes:** Identifiers in SMB traffic often associated with remote execution frameworks.

## Containment
Execute these steps immediately to trap the attacker and halt their movement through the bank's network:
* **Host Isolation (EDR):** Immediately isolate the originating compromised workstation and any successfully accessed destination hosts using the EDR console.
* **Account Suspension:** Disable the compromised user accounts or service accounts in Active Directory to instantly kill their ability to authenticate to new servers.
* **Internal Network Blocking:** If EDR isolation is unavailable, work with the network team to block the compromised IP at the core switch or internal firewall level.

## Remediation
To ensure the attacker is completely removed from the internal network:
* **Enterprise Credential Reset:** Force a password reset for all compromised accounts. If a Domain Admin account was compromised, initiate a wide-scale privileged account password reset.
* **Service Account Rotation:** If a service account was abused, carefully rotate its password, ensuring dependent financial services are updated to prevent downtime.
* **Remove Persistence Mechanisms:** Check isolated hosts for newly created scheduled tasks, malicious services, or altered registry run keys before bringing them back online. (Re-imaging the host is highly recommended).

## Detection Improvement
Harden the internal network to make lateral movement exponentially more difficult:
* **Implement LAPS:** Deploy Microsoft Local Administrator Password Solution (LAPS) to randomize local administrator passwords, preventing Pass-the-Hash attacks from succeeding across multiple workstations.
* **Micro-Segmentation:** Implement strict internal firewall rules preventing workstation-to-workstation communication (users should only communicate with servers, not each other's laptops).
* **Tune EDR & SIEM Rules:** Alert on any usage of PsExec or WMI remote execution originating from outside the dedicated IT/Helpdesk subnets.
