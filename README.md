# 🛡️ SOC Automation: EDR + SOAR Integrated Incident Response

> **End-to-end Security Operations Center workflow** — from threat detection to automated containment, with analyst approval gates built in.

---

## 📌 Overview

This project demonstrates a production-grade SOC automation pipeline that integrates **Endpoint Detection and Response (EDR)** with **Security Orchestration, Automation, and Response (SOAR)** to handle the full incident lifecycle — detection, alerting, analyst decision, and containment — with minimal manual effort.

The workflow automatically:
1. Detects malicious activity on a Windows endpoint (credential access via LaZagne)
2. Fires alerts to Slack and Email with full forensic context
3. Presents a human approval prompt to the analyst
4. Either **isolates the endpoint** or **flags for investigation** based on the analyst's decision
5. Verifies isolation and confirms via Slack

---

## 🧰 Tech Stack

| Layer | Tool |
|---|---|
| Cloud Infrastructure | AWS EC2 (Windows Server 2022) |
| EDR | LimaCharlie |
| SOAR | Tines |
| Alerting | Slack, Gmail |
| Attack Simulation | LaZagne |

---

## 🏗️ Architecture

```
Windows Endpoint (AWS EC2)
        │
        ▼
 LimaCharlie Sensor
 (Telemetry Collection)
        │
        ▼
 Custom D&R Rule
 "HackTool - Lazagne"
        │
  Detection Fired
        │
        ▼
   Tines Webhook ◄──── LimaCharlie Output
        │
   ┌────┴────┐
   ▼         ▼
 Slack     Email
 Alert     Alert
        │
        ▼
  User Prompt
  "Isolate machine?"
        │
   ┌────┴────┐
  YES        NO
   │          │
   ▼          ▼
LimaCharlie  Slack
  Isolate    "Please
  Endpoint   Investigate"
   │
   ▼
Verify Isolation
   │
   ▼
Slack Confirmation
"Endpoint Isolated"
```

---

## 📸 Screenshots

### Architecture Diagram
![Architecture Diagram](./screenshots/soar%20edr%201.png)

### LimaCharlie — Sensor Online
![Sensor List](./screenshots/soar%20edr%202.png)

### LimaCharlie — Endpoint Details (Pre-Isolation)
![Endpoint Pre-Isolation](./screenshots/soar%20edr%203.png)

### LimaCharlie — Timeline: LaZagne Execution Captured
![Timeline](./screenshots/soar%20edr%204.png)

### LaZagne Running on Windows Endpoint
![LaZagne Execution](./screenshots/soar%20edr%205.png)

### LimaCharlie — Custom D&R Rule (SOC-EDR-Rule)
![D&R Rule](./screenshots/soar%20edr%206.png)

### LimaCharlie — Detection Fired
![Detection](./screenshots/soar%20edr%207.png)

### LimaCharlie — Endpoint Isolated
![Endpoint Isolated](./screenshots/soar%20edr%208.png)

### Tines — User Prompt (Analyst Approval Form)
![User Prompt](./screenshots/soar%20edr%209.png)

### Tines — Full SOAR Workflow
![Tines Workflow](./screenshots/soar%20edr%2010.png)

### Tines — Webhook Events & Detection Payload
![Tines Events](./screenshots/soar%20edr%2011.png)

### Tines — Slack Action Configuration
![Slack Config](./screenshots/soar%20edr%2012.png)

### Slack — Detection Alert Received
![Slack Alert](./screenshots/soar%20edr%2013.png)

### Gmail — Email Alert Received
![Email Alert](./screenshots/soar%20edr%2014.png)

---

## ⚙️ Setup & Implementation

### 1. AWS Endpoint Deployment

- Deployed a **Windows Server 2022** EC2 instance (`t3.small`)
- Enabled RDP access, assigned public IP
- Renamed host to `corp-windows-client01` to simulate a corporate workstation

### 2. LimaCharlie EDR Setup

- Created a new LimaCharlie organization
- Downloaded and installed the **Windows x64 sensor** on the EC2 instance
- Sensor registered successfully — telemetry streaming began immediately (process creation, network activity, CLI events)

### 3. Attack Simulation

Executed **LaZagne** (`LaZagne.exe all`) on the endpoint to simulate a credential access attack. This triggered:
- Process creation events
- Access to Windows credential stores
- System masterkey decryption attempts

### 4. Detection Engineering

Created a custom **D&R (Detection & Response) Rule** in LimaCharlie:

```yaml
# Detect
events:
  - NEW_PROCESS
  - EXISTING_PROCESS
op: and
rules:
  - op: is windows
  - op: or
    rules:
      - case sensitive: false
        op: ends with
        path: event/FILE_PATH
        value: LaZagne.exe
      - case sensitive: false
        op: contains
        path: event/COMMAND_LINE
        value: LaZagne

# Response
action: report
metadata:
  author: Windows-Client
  description: TEST - Detects Lazagne Usage
  falsepositives:
    - ToTheMoon
  level: high
  tags:
    - attack.credential_access
  name: HackTool - Lazagne
```

### 5. Tines SOAR Workflow

Built a **Tines Story** with the following nodes:

| Node | Type | Purpose |
|---|---|---|
| Retrieve Detections | Webhook | Receive LimaCharlie detections |
| Send a message | Slack | Alert analysts in `#detection-alerts` |
| Send Email Action | Email | Notify via Gmail |
| User Prompt | Page | Analyst approval gate |
| Yes (Condition) | Condition | Route to isolation |
| No (Condition) | Condition | Route to investigation |
| Isolate Sensor | HTTP Request | LimaCharlie API — isolate endpoint |
| Get Isolation Status | HTTP Request | Verify isolation |
| Send a message (final) | Slack | Confirm isolation |

### 6. Alert Contents

Every alert (Slack + Email) contains:

```
Title:          HackTool - Lazagne
Time:           <timestamp>
Computer:       corp-windows-client01.ec2.internal
Source IP:      172.31.69.211
Username:       CORP-WINDOWS-CL\Administrator
File Path:      C:\Users\Administrator\Downloads\LaZagne (1).exe
Command Line:   "C:\Users\Administrator\Downloads\LaZagne (1).exe" all
Sensor ID:      89fec01b-b7f1-4569-a973-bb659ad42f12
Detection Link: https://app.limacharlie.io/...
```

---

## 🔁 Workflow Decision Logic

```
Analyst receives Slack + Email alert
          │
          ▼
    Opens User Prompt
    (Tines web page)
          │
   ┌──────┴──────┐
   │             │
  YES            NO
   │             │
   ▼             ▼
Isolate      Slack msg:
Endpoint     "Not isolated.
   │          Please investigate."
   ▼
Verify
Isolation
   │
   ▼
Slack msg:
"Endpoint isolated."
```

---

## ✅ Results

| Objective | Status |
|---|---|
| LaZagne execution detected | ✅ |
| Custom D&R rule triggered | ✅ |
| Slack alert delivered | ✅ |
| Email alert delivered | ✅ |
| Analyst approval prompt functional | ✅ |
| Endpoint isolation via API | ✅ |
| Isolation verification | ✅ |
| Slack confirmation sent | ✅ |

---

## 🧠 Skills Demonstrated

- **Detection Engineering** — Custom YAML D&R rules, process monitoring, command-line analysis
- **EDR** — LimaCharlie sensor deployment, telemetry analysis, API-driven host isolation
- **SOAR** — Tines workflow automation, conditional branching, human-in-the-loop approval gates
- **Incident Response** — Detection → Analysis → Containment → Verification lifecycle
- **Cloud Security** — AWS EC2 deployment, Windows endpoint hardening concepts
- **SOC Operations** — Alert triage, multi-channel notification, analyst workflows

---

## 🔮 Future Enhancements

- [ ] VirusTotal hash reputation lookup
- [ ] AbuseIPDB IP enrichment
- [ ] Jira/ServiceNow ticket auto-creation
- [ ] Microsoft Teams integration
- [ ] Multi-host isolation support
- [ ] Automated IOC extraction
- [ ] SIEM integration (Splunk / Microsoft Sentinel)
- [ ] Malware sandbox detonation
- [ ] Threat intel enrichment (MISP)

---

## 📁 Project Structure

```
SOC-SOAR-EDR-Project/
├── README.md
├── detection-rules/
│   └── hacktool-lazagne.yaml       # LimaCharlie D&R rule
├── tines-workflows/
│   └── soc-edr-story.json          # Exported Tines story
└── screenshots/
    ├── soar_edr_1.png  - Architecture diagram
    ├── soar_edr_2.png  - Sensor list
    ├── soar_edr_3.png  - Endpoint pre-isolation
    ├── soar_edr_4.png  - Timeline (LaZagne captured)
    ├── soar_edr_5.png  - LaZagne execution
    ├── soar_edr_6.png  - D&R rule
    ├── soar_edr_7.png  - Detection fired
    ├── soar_edr_8.png  - Endpoint isolated
    ├── soar_edr_9.png  - Tines user prompt
    ├── soar_edr_10.png - Tines full workflow
    ├── soar_edr_11.png - Tines webhook events
    ├── soar_edr_12.png - Slack action config
    ├── soar_edr_13.png - Slack alert received
    └── soar_edr_14.png - Email alert received
```

---

## 🔗 References

- [LimaCharlie Documentation](https://docs.limacharlie.io/)
- [Tines Documentation](https://www.tines.com/docs)
- [LaZagne Project](https://github.com/AlessandroZ/LaZagne)
- [MITRE ATT&CK — Credential Access](https://attack.mitre.org/tactics/TA0006/)

---

## 👤 Author

**Syed Mujtaba Ahmed**
- SOC Automation | Detection Engineering | Cloud Security

---

*Built as a hands-on demonstration of modern SOC automation practices.*
