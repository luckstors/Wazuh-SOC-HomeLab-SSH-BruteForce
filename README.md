# 🛡️ SOC Home Lab: Detecting SSH Brute Force Attacks on Windows with Wazuh SIEM

[![Wazuh](https://img.shields.io/badge/SIEM-Wazuh%204.x-blue)]()
[![MITRE ATT&CK](https://img.shields.io/badge/MITRE%20ATT%26CK-T1110.001-red)]()
[![Status](https://img.shields.io/badge/status-completed-success)]()

A self-built SOC home lab simulating an SSH brute-force attack against a Windows endpoint, detecting it via Wazuh SIEM, and executing incident response. Built to practice end-to-end detection engineering: attack simulation → log correlation → forensic triage → hardening.

📝 **Full write-up (storytelling version):** *[Membangun SOC Home Lab Pertama: Mendeteksi Serangan Brute Force SSH dengan Wazuh SIEM](#)* on Medium

---

## 📌 TL;DR

| Item | Detail |
|---|---|
| **Attack simulated** | SSH Brute Force — Password Guessing |
| **MITRE ATT&CK** | [T1110.001 - Brute Force: Password Guessing](https://attack.mitre.org/techniques/T1110/001/) |
| **SIEM** | Wazuh 4.x |
| **Detection rules triggered** | `60106` Logon Success · `60115` Account Locked Out · `67028` Special Privileges Assigned |
| **Windows Event IDs correlated** | 4624 (Logon Success) · 4625 (Logon Failure) · 4740 (Account Locked Out) |
| **Outcome** | Detected a two-stage attack pattern (successful compromise → triggered lockout policy) and executed host hardening |

---

## 🎯 Objective

Most companies can't defend against what they can't see. This lab simulates that visibility gap closing in real time: an attacker brute-forcing SSH credentials on a Windows host, and a SOC analyst (me) using SIEM telemetry to detect, correlate, and triage it.

**What I did:**
1. Deployed a **Wazuh Manager** (SIEM) to centralize log collection and alerting.
2. Enabled an **OpenSSH service on Windows 11** as a controlled attack surface, restricted to a private/isolated network.
3. Simulated a **brute-force attack** from Kali Linux using **Hydra**, run in two separate attempts.
4. Correlated Windows Security Event Logs on the **Wazuh Dashboard**, drilling into raw JSON event detail to identify the source of the request.
5. Performed **incident response**: removed the test account, closed the attack surface, restored baseline.

---

## 💻 Lab Architecture

```
┌──────────────────┐        SSH (port 22)         ┌───────────────────────┐
│    Kali Linux    │ ───────────────────────────▶│   Windows 11 Target    │
│  (Attacker Node) │         Hydra brute force    │  (OpenSSH + Wazuh     │
│  IP: 192.168.x.x*│                              │   Agent)              │
└──────────────────┘                              │  IP: 192.168.x.x*     │
                                                  └───────────┬───────────┘
                                                              │ Event forwarding
                                                              ▼
                                                      ┌───────────────────────┐
                                                      │    Wazuh Manager      │
                                                      │       (Ubuntu)        │
                                                      └───────────────────────┘
```
*\*IPs redacted — lab used private/isolated network ranges only (RFC1918). All testing was performed on an isolated home-lab network with no internet-facing exposure.*

| Role | Component | Notes |
|---|---|---|
| SIEM Manager | Wazuh Server v4.x | Ubuntu (WSL2 / Docker) |
| Target Endpoint | Windows 11 + Wazuh Agent | OpenSSH Server enabled for this test only |
| Attacker Node | Kali Linux | Hydra for credential brute-forcing |

> ⚠️ **Safety note:** This lab was run entirely on an isolated, private virtual network. No attack traffic touched the public internet or any production system. Do not run brute-force tooling against systems you don't own or lack explicit authorization to test — it's illegal in most jurisdictions without consent.

---

## 🚀 Implementation Walkthrough

### Phase 1 — Enable the Attack Surface (Windows 11 Target)

Install and enable OpenSSH Server:
```powershell
winget install Microsoft.OpenSSH.Preview --source winget
```

Restrict the firewall rule to the private network profile only — never expose SSH to the public profile:
```powershell
New-NetFirewallRule -Name sshd -DisplayName 'OpenSSH Server (sshd)' -Enabled True -Direction Inbound -Protocol TCP -LocalPort 22 -Action Allow -Profile Private
```

Enable password authentication in `C:\ProgramData\ssh\sshd_config`:
```
PasswordAuthentication yes
```

Create a disposable local test account (lab-only, deleted post-test — see Hardening section):
```powershell
net user <REDACTED_USERNAME> <REDACTED_STRONG_PASSWORD> /add
net localgroup administrators <REDACTED_USERNAME> /add
```
> 🔒 **Sensor note:** Never publish a real username/password here, even for a throwaway lab account. Use placeholders. Reusing visible credentials — even "fake" ones tied to a personal naming pattern — is a habit security recruiters will flag.

Start and verify the service:
```bash
sudo service ssh status
```

📸 *Screenshot: terminal confirming `ssh.service` is `active (running)`*

---

### Phase 2 — Simulate the Attack (Kali Linux)

Generate a wordlist of incorrect passwords to simulate a realistic brute-force attempt:
```bash
for i in {1..50}; do echo "WrongPassword$i" >> ~/wordlist.txt; done
```

Launch the attack with Hydra against the test account:
```bash
hydra -l <REDACTED_USERNAME> -P ~/wordlist.txt -t 1 ssh://<TARGET_IP_REDACTED>
```

This was run in **two separate passes** — the first using a wordlist that happened to contain the correct password (resulting in a successful compromise), and a second pass using only incorrect attempts (triggering Windows' account lockout policy). This produced two distinct, comparable detection signatures, covered in Phase 3.

📸 *Screenshot: Kali terminal showing both Hydra runs (credentials and target IP redacted)*

---

### Phase 3 — Detection & Forensic Analysis (Wazuh Dashboard)

**1. Volumetric anomaly detection**
The Wazuh threat-hunting dashboard showed a sharp spike in `authentication_failed` events on the timestamp histogram — the first visual indicator of an active brute-force attempt, alongside a smaller but notable count of `authentication_success` events.

📸 *Screenshot: dashboard summary — total alerts, authentication failure/success counters, and top alert groups*

**2. Event correlation — two-stage attack pattern**

Drilling into the Security event log revealed two distinct stages of the attack:

| Rule ID | Description | Stage |
|---|---|---|
| `60106` | Windows Logon Success | **Stage 1** — the wordlist contained the correct password; the brute force succeeded |
| `67028` | Special Privileges Assigned to New Logon | **Stage 1** — confirms the compromised account held local Administrator rights |
| `60115` | User account locked out (multiple login errors) | **Stage 2** — a second, all-incorrect attempt run triggered Windows' native Account Lockout Policy |

📸 *Screenshot: chronological alert list showing the Stage 1 → Stage 2 sequence*

**3. Forensic drill-down (raw event detail)**

Pivoting into the raw JSON of the underlying Windows Security event (Event ID `4740` — *account locked out*) surfaces the fields a SOC analyst would use to trace the source of an attack in a real investigation: the **Account Name** of the affected/targeted account, and the **Caller Computer Name** / **Source Network Address** identifying where the request originated from.

📸 *Screenshot: raw JSON log detail — account name, domain, and source identifiers redacted*

> 🔒 **Sensor note:** This is the single most important screenshot to redact carefully before publishing. Raw JSON event dumps routinely contain the real account name, real hostname, and internal domain/workgroup name in plaintext. Crop or black out `Account Name`, `Caller Computer Name`, and any `Security ID` (SID) strings before sharing — these directly fingerprint your machine and username.

**4. Why this matters operationally:** Correlating these two stages — a successful compromise followed, in a separate attempt, by a lockout event — mirrors how a real SOC analyst reconstructs an attack timeline from disparate log entries rather than a single alert. In production, Stage 1 alone (successful logon following a failure burst) should trigger automatic response — account lockout, forced password reset, or SOAR-driven IP blocking — rather than relying on Windows' default lockout threshold to catch a *second* attempt. This gap is covered in *Lessons & Next Steps* below.

---

## 🔐 Incident Response & Hardening

Post-simulation, I restored the host to a secure baseline:

```powershell
# Remove the test account
net user <REDACTED_USERNAME> /delete

# Disable the exposed service and firewall rule
Stop-Service sshd
Remove-NetFirewallRule -Name sshd

# Disable the Wazuh agent (lab cleanup only — disable, don't do this in production)
Set-Service -Name "WazuhSvc" -StartupType Disabled
Stop-Service -Name "WazuhSvc"
```

---

## 📈 Lessons & Next Steps

- **Detection worked, but relied on Windows' default lockout threshold rather than a SIEM-driven response.** A production SOC would pair Stage 1's successful-logon alert with *immediate* automated containment, not wait for repeated failures to trip a native OS control.
- **Next iteration:** add a Wazuh active-response script to auto-disable the account or block the source IP the moment a `60106`-class alert follows a failure burst, instead of relying on rule `60115`.
- **Next iteration:** extend detections to lateral movement post-compromise (e.g., Sysmon + Wazuh ruleset for anomalous process creation).
- **Realistic constraint:** real attackers rarely brute-force this loudly. Low-and-slow brute force or password spraying — which evade simple volumetric thresholds — would be a more realistic and challenging detection scenario to build next.

## 🎓 Key Takeaways

- Hands-on understanding of **Windows Security Event IDs** (4624 / 4625 / 4740) in a real detection context, not just textbook definitions.
- Practical experience correlating **SIEM telemetry across multiple alerts** to reconstruct a multi-stage attack timeline, not just reading a single alert in isolation.
- Comfort navigating **raw JSON event data** in Wazuh to extract forensic identifiers (account name, source host) — and recognizing which of those identifiers are sensitive enough to redact before sharing.
- Applied **incident response fundamentals**: containment, eradication, and restoring baseline.

---

## 🛠️ Tech Stack
`Wazuh` `Windows Security Event Log` `OpenSSH` `Kali Linux` `Hydra` `PowerShell` `SOC Fundamentals` `Incident Response`

---

*This lab was built independently for skill development. All testing was conducted in an isolated home-lab environment against systems I own, with no production or third-party systems involved.*
