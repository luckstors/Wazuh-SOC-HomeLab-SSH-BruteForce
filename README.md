# Wazuh-SOC-HomeLab-SSH-BruteForce
A self-built SOC home lab simulating SSH brute-force attacks against a Windows endpoint, detecting them via Wazuh SIEM, and executing incident response.
# 🛡️ SOC Home Lab: Detecting SSH Brute Force Attacks on Windows with Wazuh SIEM

[![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%204.x-blue)]()
[![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-T1110-red)]()
[![Status](https://img.shields.io/badge/status-completed-success)]()

A self-built SOC home lab simulating an SSH brute-force attack against a Windows endpoint, detecting it via Wazuh SIEM, and executing incident response. Built to practice end-to-end detection engineering: attack simulation → log correlation → alert triage → hardening.

---

## 📌 TL;DR

| Item | Detail |
|---|---|
| **Attack simulated** | SSH Brute Force (Valid Accounts via Brute Force) |
| **MITRE ATT&CK** | [T1110 - Brute Force](https://attack.mitre.org/techniques/T1110/) / [T1078 - Valid Accounts](https://attack.mitre.org/techniques/T1078/) |
| **SIEM** | Wazuh 4.x |
| **Detection rules triggered** | 60106 (Windows Logon Success), 67028 (Special Privileges Assigned) |
| **Outcome** | Detected successful compromise after repeated failed logons; executed host hardening |

---

## 🎯 Objective

Most companies can't defend against what they can't see. This lab simulates that visibility gap closing in real time: an attacker brute-forcing SSH credentials on a Windows host, and a SOC analyst (me) using SIEM telemetry to detect, correlate, and respond to it.

**What I did:**
1. Deployed a **Wazuh Manager** (SIEM) to centralize log collection and alerting.
2. Hardened-then-deliberately-exposed an **OpenSSH service on Windows 11** as the attack surface.
3. Simulated a **brute-force attack** from Kali Linux using **Hydra**.
4. Correlated Windows Security Event Logs on the **Wazuh Dashboard** to confirm compromise.
5. Performed **incident response**: revoked access, closed the attack surface, restored baseline.

---

## 💻 Lab Architecture

```
┌─────────────────┐        SSH (port 22)        ┌──────────────────────┐
│   Kali Linux     │ ───────────────────────────▶│   Windows 11 Target   │
│  (Attacker Node)  │         Hydra brute force    │  (OpenSSH + Wazuh     │
│  IP: 10.x.x.x*    │                              │   Agent)               │
└─────────────────┘                              │  IP: 10.x.x.x*         │
                                                   └──────────┬───────────┘
                                                              │ Event forwarding
                                                              ▼
                                                   ┌──────────────────────┐
                                                   │   Wazuh Manager        │
                                                   │   (Ubuntu/Docker)      │
                                                   │   IP: 10.x.x.x*        │
                                                   └──────────────────────┘
```
*\*IPs redacted — lab used private/isolated network ranges only (RFC1918). All testing performed on an isolated home lab network with no internet-facing exposure.*

| Role | Component | Notes |
|---|---|---|
| SIEM Manager | Wazuh Server v4.x | Ubuntu (WSL2 / Docker) |
| Target Endpoint | Windows 11 + Wazuh Agent | OpenSSH Server enabled for this test only |
| Attacker Node | Kali Linux | Hydra for credential brute-forcing |

> ⚠️ **Safety note:** This lab was run entirely on an isolated, private virtual network. No attack traffic touched the public internet or any production system. Do not run brute-force tooling against systems you don't own or have explicit authorization to test.

---

## 🚀 Implementation Walkthrough

### Phase 1 — Expose the Attack Surface (Windows 11 Target)

Install and enable OpenSSH Server:
```powershell
winget install Microsoft.OpenSSH.Preview --source winget
```

Restrict the firewall rule to the private network profile only (never expose SSH to public profiles):
```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow -Profile Private
```

Enable password authentication in `C:\ProgramData\ssh\sshd_config`:
```
PasswordAuthentication yes
```

Create a disposable local test account (lab-only, deleted post-test — see Hardening section):
```powershell
net user labtest <REDACTED_STRONG_PASSWORD> /add
net localgroup administrators labtest /add
```
> 🔒 **Sensor note:** Never publish a real password here — even for a throwaway lab account. Use a placeholder like `<REDACTED_STRONG_PASSWORD>`. Reusing visible passwords (even "fake" ones) is a habit recruiters in security roles will flag.

Start the service:
```powershell
Start-Service sshd
```

📸 *Screenshot: PowerShell confirming `sshd` service running*

---

### Phase 2 — Simulate the Attack (Kali Linux)

Generate a wordlist of incorrect passwords to simulate a real brute-force attempt:
```bash
for i in {1..50}; do echo "WrongPassword$i" >> ~/wordlist.txt; done
```

Launch the attack with Hydra:
```bash
hydra -l labtest -P ~/wordlist.txt -t 1 ssh://<TARGET_IP_REDACTED>
```

📸 *Screenshot: Kali terminal showing Hydra executing against the target*

---

### Phase 3 — Detection & Analysis (Wazuh Dashboard)

**1. Volumetric anomaly detection**
The Wazuh threat-hunting dashboard showed a sharp spike in authentication-failure events on the timestamp histogram — the first visual indicator of an active brute-force attempt.

📸 *Screenshot: histogram spike during the attack window*

**2. Alert correlation**

| Rule ID | Description | Significance |
|---|---|---|
| `60106` | Windows Logon Success | Marks the moment the brute-force succeeded — the pivotal IOC |
| `67028` | Special Privileges Assigned | Confirms the compromised account had local admin rights — escalates severity significantly |

📸 *Screenshot: Wazuh alert detail view*

**3. Why this matters operationally:** A spike in failed logons *followed by* a success event from the same source IP is a textbook brute-force-to-compromise pattern (maps to [Windows Event ID 4625](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/event-4625) → `4624`). In a real SOC, this sequence should trigger automatic account lockout or SOAR-driven IP blocking — a gap I noted for future iterations of this lab (see *Lessons & Next Steps*).

---

## 🔐 Incident Response & Hardening

Post-simulation, I restored the host to a secure baseline:

```powershell
# Remove the test account
net user labtest /delete

# Disable the exposed service and firewall rule
Stop-Service sshd
Remove-NetFirewallRule -Name sshd

# Disable the Wazuh agent (lab cleanup only — disable, don't do this in production)
Set-Service -Name "WazuhSvc" -StartupType Disabled
Stop-Service -Name "WazuhSvc"
```

---

## 📈 Lessons & Next Steps

- **Detection works — response was manual.** A production SOC would pair this detection with automated containment (account lockout policy, SOAR playbook, or active-response script in Wazuh).
- **Next iteration:** add a Wazuh active-response script to auto-block the source IP after N failed attempts (`ossec.conf` active-response block).
- **Next iteration:** extend to detect lateral movement post-compromise (e.g., `Sysmon` + Wazuh ruleset for process creation anomalies).
- **Realistic constraint:** real attackers rarely brute-force this loudly; slow/low-and-slow brute force or password spraying would be a more realistic next lab to build detections for.

## 🎓 Key Takeaways

- Hands-on understanding of **Windows Security Event IDs** (4624/4625) in a real detection context.
- Practical experience reading and correlating **SIEM telemetry** to confirm a security incident, not just generate alerts.
- Applied **incident response fundamentals**: containment, eradication, and restoring baseline.
- Identified the gap between *detection* and *automated response* — informing what I'd build next.

---

## 🛠️ Tech Stack
`Wazuh` `Windows Event Logs` `OpenSSH` `Kali Linux` `Hydra` `PowerShell` `SOC Fundamentals` `Incident Response`

---

*This lab was built independently for skill development. All testing was conducted in an isolated home-lab environment against systems I own, with no production or third-party systems involved.*
