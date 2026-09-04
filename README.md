# SIEM-Active-Defense-Lab
![Wazuh](https://img.shields.io/badge/Wazuh-4.14.0-blue?style=for-the-badge&logo=wazuh&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-29.7.2-blue?style=for-the-badge&logo=docker&logoColor=white)
![Keycloak](https://img.shields.io/badge/Keycloak-26.0.5-blue?style=for-the-badge&logo=keycloak&logoColor=white)
![UFW](https://img.shields.io/badge/UFW-0.36.2-blue?style=for-the-badge&logo=ubuntu&logoColor=white)
![Nginx](https://img.shields.io/badge/NGINX-1.28.3-blue?style=for-the-badge&logo=nginx&logoColor=white)
![Ubuntu](https://img.shields.io/badge/Ubuntu-26.04.1-blue?style=for-the-badge&logo=ubuntu&logoColor=white)
![Status](https://img.shields.io/badge/Status-Active-success?style=for-the-badge)

### *Threat Detection & Automated Defense*

[Tools](#-tools) • [Quick Start](#-quick-start) • [Homelab Architecture](#-homelab-architecture) • [Attack-Simulation](#-attack-simulation)

---

## Project Overview

<div align="center">
  
A complete Security Information and Event Management (SIEM) lab that combines threat detection with automated active defense capabilities. This project demonstrates how to build security monitoring environment using open-source tools and Ubuntu VM.

</div>

---

## Tools
| Tool | Description |
|:-------:|:-----------:|
| Wazuh | Open-source SIEM & XDR for comprehensive security monitoring |
| Docker | Containerization and deployment |
| Keycloak | IAM for Single-Sign-On and authentication |
| UFW | Uncomplicated Firewall for Linux packet-filtering |
| Nginx | Web Server, High performance, reverse proxy, attack surface |

## Homelab Architecture

```mermaid
graph TB
    %% Color definitions
    classDef attacker fill:#FF6B6B,stroke:#FF0000,stroke-width:4px,color:#FFF
    classDef web fill:#4ECDC4,stroke:#00B894,stroke-width:4px,color:#FFF
    classDef siem fill:#45B7D1,stroke:#2196F3,stroke-width:4px,color:#FFF
    classDef db fill:#FFA94D,stroke:#FF6B00,stroke-width:4px,color:#FFF
    classDef auth fill:#A29BFE,stroke:#6C5CE7,stroke-width:4px,color:#FFF
    classDef response fill:#FD79A8,stroke:#E84393,stroke-width:4px,color:#FFF

    A[🖥️ Attacker]:::attacker
    B[🌐 NGINX<br/>Web Server]:::web
    C[🛡️ Wazuh Agent<br/>Endpoint Security]:::siem
    D[📊 Wazuh Server<br/>SIEM Core]:::siem
    E[🔍 Wazuh Indexer<br/>Elasticsearch]:::db
    F[📈 Wazuh Dashboard<br/>Kibana]:::siem
    G[🔐 Keycloak<br/>Identity Management]:::auth
    H[🚨 Active Response<br/>Firewall Drop]:::response

    A -->|HTTP Requests| B
    B -->|Logs| C
    C -->|Security Events| D
    D -->|Store| E
    E -->|Visualize| F
    D -->|Trigger| H
    G -->|Auth| F

## Data Flow
flowchart LR
    A[⚡ Attack Traffic]:::input
    B[📝 Log Collection]:::process
    C[🔍 Threat Detection]:::ai
    D{Attack?}:::decision
    E[📊 Dashboard Alert]:::safe
    F[🚫 Firewall Block]:::danger
    G[📧 Email Alert]:::alert
    
    A --> B --> C --> D
    D -->|No| E
    D -->|Yes| F
    F --> G
    
    classDef input fill:#FF6B6B,stroke:#FF0000,stroke-width:4px,color:#FFF
    classDef process fill:#4ECDC4,stroke:#00B894,stroke-width:4px,color:#FFF
    classDef ai fill:#45B7D1,stroke:#2196F3,stroke-width:4px,color:#FFF
    classDef decision fill:#A29BFE,stroke:#6C5CE7,stroke-width:4px,color:#FFF
    classDef safe fill:#00F5A0,stroke:#00C853,stroke-width:4px,color:#FFF
    classDef danger fill:#FF6B6B,stroke:#FF0000,stroke-width:4px,color:#FFF
    classDef alert fill:#FD79A8,stroke:#E84393,stroke-width:4px,color:#FFF

## Project Structure
SIEM_Active_Defense_Lab/
├── 📄 README.md                     # Comprehensive documentation
├── 🐳 docker-compose.yml            # Main Docker compose file
├── 🔐 generate-indexer-certs.yml    # Certificate generation compose
├── 📁 config/                       # Configuration files
│   ├── 📁 wazuh_indexer/
│   │   ├── internal_users.yml       # Indexer user configuration
│   │   └── config.yml               # Indexer settings
│   └── 📁 wazuh_dashboard/
│       ├── wazuh.yml                # Dashboard configuration
│       └── opensearch_dashboards.yml # Dashboard settings
├── 📁 custom_rules/                 # Detection engineering
│   └── local_rules.xml              # Custom security rules
├── 📁 scripts/                      # Helper scripts
│   └── attack_simulation.sh         # Attack simulation script
└── 📁 screenshots/                  # Evidence for documentation
    └── active_response_alert.png    # Active response screenshot

## Quick Start
Prerequisites
Docker Engine 24.0+
Docker Compose 2.20+
Minimum 8GB RAM
20GB free disk space

## Installation
```bash
# Clone the repository
git clone https://github.com/AljawharaK/SIEM_Active_Defense_Lab.git
cd SIEM_Active_Defense_Lab

# Generate SSL certificates for secure communication
docker compose -f generate-indexer-certs.yml run --rm generator

# Start the complete stack
docker compose up -d

# Verify all services are running
docker compose ps
```

## Access Services
| Service | URL |
|:-------:|:-----------:|
| Wazuh Dashboard | https://localhost:443 |
| Keycloak | http://localhost:8081 |

## Attack-Simulation
| Attack Type | Method |
|:-------:|:-----------:|
| Shellshock | () { :;}; /bin/bash -c 'echo vulnerable' |
| Nmap Scan | nmap -sV -p 1-1000 target |
| Curl Exploit | curl -X GET http://target:80 |
