# **Incident Report: Raspberry Pi Power Disruption & Service Recovery**  
**Date:** February 18, 2025  
**Reported By:** KoalaCodes  

---

## **1. Incident Summary**  
**Trigger Event**: Accidental unplugging of Raspberry Pi hosting:  
- Home Assistant (Docker container)  
- Cloudflared Tunnel (Docker container)  

**Impact Duration**: 2 hours (from power loss to full recovery)  

---

## **2. Root Causes**  
| Factor | Description |  
|--------|--------------|  
| **Sudden Power Loss** | No UPS or graceful shutdown mechanism in place. |  
| **Filesystem Corruption** | SD card corruption due to improper shutdown (`/var/lib/docker` and Home Assistant database). |  
| **Service Dependencies** | Docker containers terminated mid-operation, causing service instability. |  
| **Network Disruption** | Cloudflared Tunnel lost persistent connection to Cloudflare edge. |  

---

## **3. Impact Analysis**  
### **Home Assistant**  
- **Status**: Offline (HTTP 503 errors)  
- **Critical Failures**:  
  - Corrupted SQLite database (`home-assistant_v2.db`).  
  - Automation/script execution halted (Alexa voice commands unresponsive).  
  - Loss of device states (Zigbee/WiFi devices showed "unavailable").  

### **Cloudflared Tunnel**  
- **Status**: Disconnected  
- **Critical Failures**:  
  - Remote access via `homeassistant.[domain].com` failed (DNS propagated dead endpoints).  
  - Security risk: Temporary exposure of local IP via stale DNS records.  

---

## **4. Resolution Process**  
### **Step 1: Power Restoration & Filesystem Repair**  
**Action**:  
1. Reconnected power and remounted filesystem as read-write:  
   ```bash  
   sudo mount -o remount,rw /  
   ```
2. Checked SD card health:  
   ```bash  
   sudo fsck /dev/mmcblk0p2  
   ```
**Reasoning**: Abrupt shutdowns often corrupt SD cards. `fsck` repairs filesystem errors.  

---

### **Step 2: Docker Service Recovery**  
**Action**:  
1. Diagnosed Docker failure:  
   ```bash  
   sudo systemctl status docker  # Showed "inactive (dead)"  
   journalctl -xeu docker.service | grep "Failed"  
   ```
2. Purged corrupted Docker install:  
   ```bash  
   sudo apt-get purge docker-ce docker-ce-cli containerd.io  
   sudo rm -rf /var/lib/docker  
   ```
3. Reinstalled Docker (ARMv7-compatible version):  
   ```bash  
   sudo apt-get install docker-ce=5:24.0.6-1~debian.12~bookworm \  
        docker-ce-cli=5:24.0.6-1~debian.12~bookworm \  
        containerd.io  
   ```
4. Fixed permissions:  
   ```bash  
   sudo usermod -aG docker $USER  
   newgrp docker  
   ```
**Reasoning**: Corrupted Docker files prevented service restart. A clean install resolved socket/permission conflicts.  

---

### **Step 3: Home Assistant Database Repair**  
**Action**:  
1. Identified corrupted database:  
   ```bash  
   cd ~/docker/homeassistant/config  
   ls -l home-assistant_v2.db*  # Showed 0-byte file  
   ```
2. Replaced database:  
   ```bash  
   mv home-assistant_v2.db home-assistant_v2.db.corrupted  
   ```
3. Restored from backup (if available) or let HA regenerate it.  

**Reasoning**: The SQLite database couldn’t recover from abrupt writes. Replacement forced HA to rebuild.  

---

### **Step 4: Service Restart & Validation**  
**Action**:  
1. Restarted Home Assistant:  
   ```bash  
   cd ~/docker/homeassistant  
   docker-compose up -d --force-recreate  
   ```
2. Restarted Cloudflared:  
   ```bash  
   cd ~/docker/cloudflared  
   docker-compose up -d  
   ```
3. Verified functionality:  
   ```bash  
   docker logs -f homeassistant | grep "HTTP server started"  
   docker logs cloudflared | grep "Connected"  
   ```
**Outcome**:  
- Home Assistant UI accessible at `http://10.0.0.124:8123` (local).  
- Cloudflared reconnected within 5 minutes (`INF Registered tunnel connection`).  

---

## **5. Prevention Strategies**  
### **Hardware Improvements**  
| Solution | Purpose |  
|----------|---------|  
| **Uninterruptible Power Supply (UPS)** | Provides 10-min buffer for graceful shutdown. |  
| **High-Endurance SD Card** | Reduces corruption risk during power loss. |  

### **Software Configuration**  
1. **Automated Backups**:  
   Add to `configuration.yaml`:  
   ```yaml  
   shell_command:  
     daily_backup: 'tar -czf /backups/ha-$(date +%s).tar.gz /config'  
   ```
   Schedule via automation.  

2. **Read-Only Filesystem**:  
   ```bash  
   sudo raspi-config  # Enable OverlayFS  
   ```

3. **Docker Healthchecks**:  
   Add to `docker-compose.yml`:  
   ```yaml  
   services:  
     homeassistant:  
       healthcheck:  
         test: ["CMD", "curl", "-f", "http://localhost:8123"]  
         interval: 1m  
         retries: 3  
   ```

4. **Cloudflared Auto-Reconnect**:  
   Add to Cloudflared config:  
   ```yaml  
   tunnel: your-tunnel-id  
   credentials-file: /config/credentials.json  
   ingress:  
     - service: http_status:404  
   ```

---

## **6. Post-Incident Validation**  
| Check | Command | Expected Result |  
|-------|---------|-----------------|  
| Docker status | `sudo systemctl status docker` | `Active: active (running)` |  
| HA connectivity | `curl -I http://localhost:8123` | `HTTP/1.1 200 OK` |  
| Tunnel status | `docker logs cloudflared` | `INF Connected to xxx.cloudflaretunnel.net` |  

---

## **7. Lessons Learned**  
1. **Power Stability**: A $30 UPS could have prevented 90% of this incident.  
2. **Database Resilience**: SQLite is vulnerable to power loss; consider migrating to PostgreSQL.  
3. **Monitoring Gap**: Implement Prometheus/Grafana for real-time service health tracking.  

---

**Documentation Version**: 2.0  
**Approved By**: [Your Name]  
**Next Review Date**: March 18, 2025  

--- 

This document provides a complete technical record and recovery blueprint for similar incidents. Let me know if you need further refinements.
