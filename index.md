#  SSH Honeypot & SIEM Integration


## 🎯 Project Overview

This project involves deploying a **Cowrie SSH Honeypot** in **Microsoft Azure** to attract and log malicious login attempts. The collected data is forwarded to **Microsoft Sentinel** for real-time analysis, correlation, and visualization using custom KQL queries and workbooks, offering practical experience in **threat detection, log monitoring & attack analysis** within a controlled, simulated environment.

The main objectives are:
- Deploy and configure a **Cowrie SSH honeypot** on an **Azure VM**.
- Forward **Syslog** and **JSON logs** to a **Log Analytics Workspace (LAW)**.
- Integrate with **Microsoft Sentinel** for event monitoring and alert creation.
- Develop **KQL queries** and **interactive dashboards** to analyse attack trends.

> ⚠️ Disclaimer: This is more of a **report** rather then a guide/walkthrough. I haven't documented every single step or command - I encourage readers to research, experiment, and adapt the setup to their own goals. Use this repository as an **idea/starting point**, not a **step-by-step script**.

---

## ⚙️ Key Components

| Component | Description |
|------------|--------------|
| **Azure VM** | Cloud-hosted Linux server for running the honeypot |
| **Cowrie Honeypot** | Simulated SSH environment capturing attacker activity |
| **Log Analytics Workspace** | Collects and queries system & honeypot logs |
| **Microsoft Sentinel** | SIEM platform for correlation, alerting & dashboards |
| **KQL Queries** | Used for data filtering, IP extraction & alert logic |

---

## 📊 Features

- SSH hon

