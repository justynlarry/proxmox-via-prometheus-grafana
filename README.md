# Proxmox Cluster Monitoring with Prometheus and Grafana, using Docker-Compose

![Prometheus](https://img.shields.io/badge/Prometheus-Enabled-orange?logo=prometheus)
![Grafana](https://img.shields.io/badge/Grafana-Dashboard-yellow?logo=grafana)
![Docker](https://img.shields.io/badge/Docker-Containerized-blue?logo=docker)
![Ubuntu](https://img.shields.io/badge/Ubuntu-Server%20Node-E95420?logo=ubuntu&logoColor=white)
![Fedora](https://img.shields.io/badge/Fedora-VM%20Monitoring-294172?logo=fedora&logoColor=white)
![Tailscale](https://img.shields.io/badge/Tailscale-VPN-green?logo=tailscale)
![Bash](https://img.shields.io/badge/Bash-Scripts-4EAA25?logo=gnubash&logoColor=white)

This repository outlines how to build a full-featured monitoring solution for a 4-node Proxmox VE cluster using Prometheus and Grafana. The setup tracks:

- Proxmox metrics (VMs, storage, cluster health)
- Host-level stats (CPU, memory, I/O)
- Custom CPU temperature per node
- Detailed S.M.A.R.T. disk health data

# Table of Contents
- Features
- Architecture Overview
- Prerequisites
- Setup Instructions
  1. Prometheus & Grafana Setup (Docker Compose)
  2. PVE Exporter Setup
  3. Custom CPU Temperature Exporter
  4. Node Exporter Setup
  5. Smartctl Exporter Setup
  6. Prometheus Configuration Update
  7. Grafana Dashboard Setup
- Key Metrics to Monitor
- Troubleshooting
- Contributing


# Features
- Proxmox API Metrics: Monitor VM status, storage pools (ZFS), and cluster health with prometheus-pve-exporter.

- System Metrics: Collect CPU, memory, disk, and network stats via node_exporter.

- Custom CPU Temperature: Track node CPU temps using lm-sensors and a Python exporter.

- Disk Health Monitoring: Surface S.M.A.R.T. data from HDDs/SSDs via smartctl_exporter.

- Centralized Monitoring: Prometheus scrapes all data; Grafana visualizes it.

# Architecture Overview
Component	        Description
Proxmox Nodes	    Run: node_exporter, smartctl_exporter, and cpu_temp_exporter
Prometheus	      Collects metrics from all exporters (Docker)
Grafana	          Dashboards for system metrics (Docker)
pve-exporter	    Gathers Proxmox API data from one cluster node

# Prerequisites
- A running 4-node Proxmox VE cluster
- sudo access on all nodes
- A server (or Proxmox node/VM) with Docker + Docker Compose installed
- Basic familiarity with Linux & Prometheus/Grafana

# My Automation Scripts Repository Contains:
The repository below contains scripts that may be helpful:
https://github.com/jlarry77/automation_scripts
- Docker/Containerd Install Script:  https://github.com/jlarry77/automation_scripts/tree/main/docker-containerd-install
- A Prometheus Monitor Script - Automates the deployment of a Prometheus Node Exporter container using Docker Compose: https://github.com/jlarry77/automation_scripts/tree/main/prometheus-monitor


# Setup Instructions
# 1. Prometheus & Grafana Setup (Docker Compose)
Run these services on a central node or dedicated VM.

bash
```
mkdir ~/monitoring && cd ~/monitoring
nano docker-compose.yml
```
Paste the following:

yaml
```
services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.retention.time=30d'
    restart: unless-stopped
    networks:
      - monitoring_net

  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=your_secure_password
    restart: unless-stopped
    networks:
      - monitoring_net

volumes:
  prometheus_data:
  grafana_data:

networks:
  monitoring_net:
    driver: bridge
```
Then:

bash
```
mkdir -p prometheus
nano prometheus/prometheus.yml
```
Paste:

yaml

```
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']
```
Start everything:

bash
```
docker-compose up -d
```
# 2. PVE Exporter Setup
Install prometheus-pve-exporter on one node. Ensure it scrapes the cluster API and is added to your Prometheus config later.

# 3. Custom CPU Temperature Exporter
Step 1: Install Dependencies:

bash
```
sudo apt update
sudo apt install lm-sensors python3-pip python3-venv
```
Step 2: Set Up Python Virtual Environment:

bash
```
python3 -m venv /root/pve-exporter-env
source /root/pve-exporter-env/bin/activate
pip install prometheus_client
deactivate
```
Step 3: Create Exporter Script:

bash
```
sudo nano /usr/local/bin/cpu_temp_exporter.py
```
Paste:

python
```
#!/usr/bin/env python3
from prometheus_client import Gauge, start_http_server
import subprocess, time

cpu_temp = Gauge('node_cpu_temp_celsius', 'CPU temperature in Celsius')

def read_cpu_temp():
    try:
        output = subprocess.check_output("sensors", shell=True).decode()
        for line in output.splitlines():
            if 'Package id 0:' in line:
                temp_str = line.split()[3].replace('+', '').replace('°C', '')
                return float(temp_str)
    except Exception as e:
        print(f"Error reading temperature: {e}")
    return 0.0

if __name__ == "__main__":
    start_http_server(8001)
    while True:
        cpu_temp.set(read_cpu_temp())
        time.sleep(15)
```
Make the Script Executeable:

bash
```
sudo chmod +x /usr/local/bin/cpu_temp_exporter.py
```
Step 4: Create the Systemd Service:

bash
```
sudo nano /etc/systemd/system/cpu-temp-exporter.service
```
ini
```
[Unit]
Description=Custom CPU Temperature Exporter for Prometheus
After=network.target

[Service]
ExecStart=/root/pve-exporter-env/bin/python /usr/local/bin/cpu_temp_exporter.py
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```
Reload Daemon, enable and start systemd service:

bash
```
sudo systemctl daemon-reload
sudo systemctl enable --now cpu-temp-exporter.service
```
Repeat for each node.

# 4. Node Exporter Setup

Install Node Exporter on each node:

bash
```
cd /tmp
wget https://github.com/prometheus/node_exporter/releases/download/v1.8.1/node_exporter-1.8.1.linux-amd64.tar.gz
tar -xzf node_exporter-1.8.1.linux-amd64.tar.gz
sudo mv node_exporter-*/node_exporter /usr/local/bin/
```

Create systemd unit:

bash
```
sudo nano /etc/systemd/system/node_exporter.service
```
bash
```
[Unit]
Description=Node Exporter
After=network-online.target
Wants=network-online.target

[Service]
User=nobody
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```
bash
```
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
```

# 5. Smartctl Exporter Setup:
   
Install Smartmontools:

bash
```
sudo apt update
sudo apt install smartmontools
```

Add binary and systemd service:

bash
```
cd /tmp
wget https://github.com/prometheus-community/smartctl_exporter/releases/download/v0.14.0/smartctl_exporter-0.14.0.linux-amd64.tar.gz
tar -xzf smartctl_exporter-*.tar.gz
sudo mv smartctl_exporter-*/smartctl_exporter /usr/local/bin/
```

bash
```
sudo setcap cap_sys_rawio+ep /usr/sbin/smartctl
sudo useradd -r -s /sbin/nologin smartctl_exporter
sudo usermod -aG disk smartctl_exporter
```

Create systemd unit:

bash
```
sudo nano /etc/systemd/system/smartctl_exporter.service
```
bash
```
[Unit]
Description=Prometheus Smartctl Exporter
After=network-online.target
Wants=network-online.target

[Service]
User=smartctl_exporter
Group=smartctl_exporter
ExecStart=/usr/local/bin/smartctl_exporter --smartctl.interval=60s --smartctl.device-include="^/dev/(nvme[0-9]+n[0-9]+|sd[a-z])$"
Restart=always

[Install]
WantedBy=multi-user.target
```
Reload Daemon, enable and start systemd process:

bash
```
sudo systemctl daemon-reload
sudo systemctl enable --now smartctl_exporter
```
# 6. Prometheus Configuration Update:
Edit prometheus.yml:

yaml
```
scrape_configs:
  - job_name: 'pve_exporter'
    static_configs:
      - targets: ['node1:9221']

  - job_name: 'cpu_temp_exporter'
    static_configs:
      - targets: ['node1:8001', 'node2:8001', 'node3:8001', 'node4:8001']

  - job_name: 'node_exporter'
    static_configs:
      - targets: ['node1:9100', 'node2:9100', 'node3:9100', 'node4:9100']

  - job_name: 'smartctl_exporter'
    static_configs:
      - targets: ['node1:9633', 'node2:9633', 'node3:9633', 'node4:9633']
```
Restart Prometheus:

bash
```
docker-compose restart prometheus
```
# 7. Grafana Dashboard Setup
1. Access Grafana at http://<host_IP>:3000

2. Add Prometheus as a data source

3. Import dashboards:
- Node Exporter Full (ID: 1860)
- SMART + NVMe status (ID: 16514 or 22381)

Example Queries:
CPU Temp Gauge:
arduino
```
node_cpu_temp_celsius{job="cpu_temp_exporter"}
```
Disk Health:
arduino
```
smartctl_device_smart_status{job="smartctl_exporter"}
```
Drive Temps:
arduino
```
smartctl_device_temperature{job="smartctl_exporter", temperature_type="current"}
```

# Example Dashboards:
Here are a few example panels from the running monitoring stack. These were built using the Node Exporter, CPU Temperature Exporter, and smartctl_exporter.

- CPU Temperature:
<img width="1527" height="422" alt="image" src="https://github.com/user-attachments/assets/7b7a3e90-d29d-47a8-8c3d-b584d84629ca" />
<img width="1524" height="427" alt="image" src="https://github.com/user-attachments/assets/37f8d20c-c819-4fa5-b99f-88006064aeeb" />

- Current CPU Usage
<img width="1526" height="420" alt="image" src="https://github.com/user-attachments/assets/306fea45-cd00-487c-ab21-79c0fec9c452" />
<img width="1527" height="425" alt="image" src="https://github.com/user-attachments/assets/05930c12-78ec-4ec9-b9a9-9f4d132a7b2c" />

- Broad Overview, including Current Memory by VM, Current Allocated Memory, Storage, Drive Temperatures, and Drive Status:
<img width="1908" height="796" alt="image" src="https://github.com/user-attachments/assets/6eaf114c-6b84-4a07-9c2f-00b2ca216a3d" />




Key Metrics to Monitor
- node_cpu_temp_celsius
- node_cpu_seconds_total, node_memory_MemAvailable_bytes
- pve_cluster_status, pve_node_up, pve_vmid_up
- smartctl_device_smart_status, smartctl_device_temperature
- smartctl_device_attribute{attribute_name=...}

Troubleshooting
Exporter not working? Check service logs with journalctl -u <service> -f

SMART permissions?
bash
```
sudo setcap cap_sys_rawio+ep /usr/sbin/smartctl
sudo usermod -a -G disk smartctl_exporter
```
Prometheus target down?
Check Prometheus → Status → Targets

# Contributing
Pull requests and issues welcome! Improve instructions or submit fixes for edge cases you encounter.
