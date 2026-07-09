# Ransomware Playbook

## Description
This playbook outlines the response procedures for a suspected ransomware infection. Modern ransomware is typically a "double-extortion" attack: threat actors first exfiltrate sensitive financial data, and then encrypt the local files, demanding payment for the decryption key and a promise not to leak the data. Alerts typically trigger via Endpoint Detection and Response (EDR) detecting mass file encryption, the deletion of Volume Shadow Copies, or the dropping of known ransomware notes.

## Possible Threat
Ransomware is an existential threat to financial operations. The impacts include:
* **Total Operational Paralysis:** Core banking systems, tellers, and customer portals going completely offline.
* **Data Breach & Regulatory Fines:** Threat actors leaking customer Personally Identifiable Information (PII), account balances, and credit data to the dark web.
* **Reputational Destruction:** Complete loss of customer trust and severe financial penalties from regulatory bodies.

## Log Sources
_Important Note for Analysts: Ransomware acts fast. Rely heavily on real-time EDR telemetry, as local Windows logs may be encrypted or destroyed by the attacker._
* **Endpoint Detection & Response (EDR):** Alerts from CrowdStrike, SentinelOne, or Defender for Endpoint regarding suspicious process injection or mass file modifications.
* **File Server Logs / FIM:** File Integrity Monitoring solutions logging thousands of file renames or extension changes in seconds.
* **Perimeter Firewalls:** Egress traffic logs showing large outbound data transfers (exfiltration) immediately preceding the encryption event.

## Initial Triage
L1 Analysts must treat ransomware alerts as a critical emergency. Validate the infection immediately:
* **Look for the Note:** Check the EDR for the creation of files like README.txt, DECRYPT_FILES.html, or HOW_TO_DECRYPT.txt on the endpoint.
* **Check File Extensions:** Are standard documents (like .pdf or .docx) being rapidly renamed to random or known ransomware extensions (e.g., .lockbit, .ryuk, .encrypted)?
* **Identify "Patient Zero":** Determine if this is an isolated incident on a single user's laptop, or if the encryption is executing on a critical file server or Domain Controller.

## Investigation Steps
The investigation workflow must move in parallel with immediate containment to stop the bleeding:
* **L1 Analyst** - Alert Scoping: Confirm the infection. Identify the hostname, IP address, and logged-in user. Identify the parent process that executed the ransomware (e.g., was it launched via a malicious macro in Word, or an RDP session?).
* **L2 Analyst** - Exfiltration & Lateral Movement Check: Ransomware is usually the final step of an attack. Query the firewall and DLP logs for the past 7-14 days to see if the attacker exfiltrated data before encrypting. Check internal network logs to see if the infection is spreading via SMB (Port 445) to other servers.
* **L3 Analyst** - Forensics & Reverse Engineering: Capture the volatile memory (RAM) from the infected host before it reboots. Analyze the dropped malware to identify the specific ransomware family (e.g., LockBit, BlackCat) and extract Command and Control (C2) domains for enterprise-wide blocking.

## Indicators of Compromise (IoCs)
Extract these artifacts immediately to block the attack from spreading to other banking segments:
* **File Artifacts:** The exact malicious file extensions used, the ransom note filename, and the SHA-256 hash of the ransomware executable.
* **Process Anomalies:** Execution of native tools used to destroy backups (e.g., vssadmin.exe delete shadows /all /quiet, wbadmin, bcdedit).
* **Network Indicators:** IP addresses or domains used for Command and Control (C2) or data exfiltration (e.g., connections to MEGA, Rclone, or unsanctioned FTPs).

## Containment
Do NOT power off the infected machine. Powering it off destroys encryption keys stored in RAM and alerts the attacker.
* **Immediate EDR Isolation:** Use the EDR console to instantly isolate the infected host(s) from the network, cutting off its ability to encrypt network file shares or communicate with the C2 server.
* **Sever Network Ties:** If EDR isolation fails, physically disconnect the ethernet cable or disable the switch port. Disable VPN access for the compromised user.
* **Suspend Accounts:** Disable the Active Directory account of the user logged into the infected machine to prevent lateral movement.
* **Declare an Incident:** Escalate immediately to the Incident Commander, CISO, and Legal counsel. Open an emergency bridge.

## Remediation
Ransomware remediation requires rebuilding, not just cleaning:
* **Do Not Pay (Standard Guidance):** Coordinate with Legal and Incident Response (IR) retainers. Paying the ransom does not guarantee data recovery and funds terrorism.
* **Rebuild from Bare Metal:** Never attempt to "clean" a ransomware-infected host. Wipe the machine completely and re-image from a known-good, golden baseline.
* **Restore from Offline Backups:** Restore critical banking data exclusively from immutable, offline, or air-gapped backups.
* **Enterprise Credential Reset:** Assume the attacker dumped all credentials before encrypting. Force a global password reset for all Domain Admins, Service Accounts, and users.

## Detection Improvement
Update controls to detect the precursor activity before the ransomware executes:
* **Deploy Canary Files:** Place hidden "canary" or "honey" files on network shares. Configure the SIEM to trigger a critical alert if these files are ever modified or renamed.
* **Alert on Shadow Copy Deletion:** Create strict EDR/SIEM rules to block and alert on any execution of vssadmin.exe attempting to delete volume shadow copies.
* **Harden RDP:** Ensure Remote Desktop Protocol (RDP) is never exposed directly to the internet, and internal RDP requires strictly enforced Multi-Factor Authentication (MFA).
