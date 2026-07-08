# Web App Attack Playbook

## Description
This playbook outlines the response procedures for investigating attacks targeting the organization's public-facing web applications, such as customer banking portals or API endpoints. These alerts typically involve SQL Injection (SQLi), Cross-Site Scripting (XSS), Path Traversal, or Command Injection. Triggers include Web Application Firewall (WAF) block events, high volumes of HTTP 500 errors, or SIEM alerts detecting malicious syntax in HTTP GET/POST requests.

## Possible Threat
Web applications are directly connected to backend databases holding sensitive financial data. The threats include:
* **Data Exfiltration:** Dumping customer PII, account balances, or credit card numbers via SQLi.
* **Session Hijacking:** Stealing authenticated customer session cookies via XSS to perform unauthorized transactions.
* **Server Compromise:** Achieving Remote Code Execution (RCE) on the underlying web server to pivot into the internal network.

## Log Sources
_Important Note for Analysts: Web traffic analysis requires visibility into both the encrypted payload (decrypted by the WAF) and the backend server response._
* **Web Application Firewalls (WAF):** Logs from Imperva, Cloudflare, F5 BIG-IP, or AWS WAF showing matched attack signatures and payloads.
* **Web Server Logs:** IIS, Apache, or NGINX access and error logs to review HTTP response codes and response sizes.
* **Database & Application Logs:** Backend SQL query logs to verify if an injected payload actually executed against the database.

## Initial Triage
L1 Analysts must determine if the alert represents blind automated scanning or a targeted, successful exploit:
* **Analyze the Payload:** Review the raw HTTP request (URI, Headers, Body) for obvious malicious syntax (_e.g., UNION SELECT, <script>alert(1)</script>, ../../../etc/passwd_).
* **Check the HTTP Response Code:**
  * **403 Forbidden / 406 Not Acceptable:** The WAF or server successfully blocked the attack.
  * **500 Internal Server Error:** The payload caused a backend crash (common during active SQLi attempts).
  * **200 OK:** The payload was accepted by the server. If the payload is malicious and the response is 200, escalate immediately.
* **Rule Out Scanners:** Check if the traffic matches the pattern of a known automated vulnerability scanner (e.g., Qualys, Nessus) or an authorized penetration test.

## Investigation Steps
The investigation workflow is divided across SOC tiers to analyze the exploit, verify the server response, and assess backend impact:
* **L1 Analyst** - Payload & Traffic Analysis: Decode the malicious payload (URL decoding, Base64 decoding) to understand what the attacker is trying to execute. Determine the volume of the attack: Is it a single targeted request, or thousands of requests indicating automated fuzzing?
* **L2 Analyst** - Impact & Exfiltration Assessment: If the server returned a 200 OK, analyze the Response Size (bytes out). A sudden spike in response size during an SQLi attack strongly indicates that the database successfully dumped data back to the attacker. Pivot to the backend database logs to see if the malicious query actually executed.
* **L3 Analyst** - Code-Level Forensics: Investigate the compromised web directory for dropped web shells (e.g., malicious .php, .jsp, or .aspx files). Work with the application development team to identify the specific vulnerable parameter or API endpoint in the source code.

## Indicators of Compromise (IoCs)
Extract these artifacts from the WAF and web server logs and document them for containment:
* **Attacker Infrastructure:** The external Source IPs and hosting providers driving the attack.
* **Malicious Payloads:** The exact SQLi syntax, XSS scripts, or command injection strings used.
* **Targeted Endpoints:** The specific vulnerable URLs, API endpoints, or HTTP parameters being manipulated.
* **Suspicious User-Agents:** Tools like sqlmap, DirBuster, or Nikto explicitly identified in the headers.

## Containment
Execute these steps immediately to protect the web application from further exploitation:
* **WAF IP Blocking:** Add the malicious Source IP to the WAF and perimeter firewall emergency blocklists.
* **Implement Custom WAF Rules:** If the attacker is rotating IPs, create a custom WAF rule to block the specific malicious payload string or User-Agent across all incoming traffic.
* **Endpoint / Parameter Disablement:** If a critical vulnerability is actively being exploited and cannot be immediately blocked by the WAF, work with the application team to temporarily take the specific vulnerable page or API endpoint offline.

## Remediation
To permanently secure the application and eradicate any attacker presence:
* **Code Patching:** The development team must implement secure coding practices, specifically parameterized queries (prepared statements) to fix SQLi, and input validation/output encoding to fix XSS.
* **Remove Web Shells:** If the server was compromised, locate and delete any uploaded web shells or backdoors. In severe cases, rebuild the web server from a known good image.
* **Credential Rotation:** Rotate the backend database service account credentials if there is any suspicion the attacker successfully queried system tables.

## Detection Improvement
Update application security controls to shift from reactive to proactive defense:
* **Enable WAF "Blocking" Mode:** Ensure the WAF is not just in "Monitoring/Alerting" mode for critical OWASP Top 10 signatures.
* **Implement Rate Limiting:** Configure strict rate limiting on sensitive API endpoints (like login portals or transaction submissions) to prevent automated fuzzing.
* **Transition to Positive Security Models:** Instead of trying to block known bad payloads (which attackers can obfuscate), tune the WAF to only accept strictly defined known good input (e.g., enforcing that an account_id parameter can only contain integers).
