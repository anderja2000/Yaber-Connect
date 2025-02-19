Here's a comprehensive guide to ensure your Docker containers (Home Assistant + Cloudflared Tunnel) survive reboots/power outages, with deep conceptual explanations:

---

### **1. Docker Restart Policies (First Layer Defense)**  
**What it does**: Automatically restarts containers when Docker service starts.  
**Why it matters**: Ensures services come back online after power restoration.  

#### **Implementation**:
```yaml
# ~/docker/homeassistant/docker-compose.yml
version: '3.8'
services:
  homeassistant:
    image: "ghcr.io/home-assistant/home-assistant:stable"
    restart: unless-stopped  # Critical line
    # ... other configs
```

```yaml
# ~/docker/cloudflared/docker-compose.yml
version: '3.8'
services:
  cloudflared:
    image: cloudflare/cloudflared
    restart: unless-stopped  # Critical line
    # ... other configs
```

#### **Restart Policy Options**:
| Policy | Behavior | Use Case |
|--------|----------|----------|
| `no` | Never restart | Not recommended |
| `always` | Always restart, even if manually stopped | Overkill |
| `on-failure` | Restart only on error exit codes | Good for crash recovery |
| `unless-stopped` | Restart unless explicitly stopped | **Best for your setup** |

**Verification**:
```bash
docker inspect -f '{{.HostConfig.RestartPolicy.Name}}' homeassistant
# Should return "unless-stopped"
```

---

### **2. Systemd Service Persistence (Second Layer)**  
**What it does**: Ensures Docker itself starts on system boot.  
**Why it matters**: Containers can't auto-restart if Docker isn't running.  

#### **Implementation**:
```bash
sudo systemctl enable docker  # Enable autostart
sudo systemctl is-enabled docker  # Verify: should return "enabled"
```

#### **Debugging Boot Issues**:
```bash
# Check Docker boot logs
journalctl -u docker --boot
```

---

### **3. Filesystem Protection (SD Card Preservation)**  
**Problem**: Raspberry Pi SD cards are prone to corruption during power loss.  
**Solution**: Use overlay filesystem to reduce writes.  

#### **Implementation**:
```bash
sudo raspi-config
# -> Performance Options -> Overlay File System -> Yes
```

**How it works**:
- Mounts root filesystem as read-only
- Uses RAM overlay for temporary writes
- **Tradeoff**: Requires manual remount for updates:
  ```bash
  sudo mount -o remount,rw /  # Before making changes
  sudo mount -o remount,ro /  # After changes
  ```

---

### **4. Automated Backups (Third Layer)**  
**Strategy**: Daily backups of critical data.  

#### **Home Assistant Backup Script**:
```bash
# ~/docker/homeassistant/backup.sh
#!/bin/bash
tar -czf /path/to/backups/ha-$(date +%Y%m%d).tar.gz ~/docker/homeassistant/config
```

**Schedule with Cron**:
```bash
(crontab -l ; echo "0 3 * * * /bin/bash ~/docker/homeassistant/backup.sh") | crontab -
```

#### **Cloudflared Tunnel Backup**:
```bash
# Backup credentials.json and config.yml
cp ~/docker/cloudflared/config/* /path/to/backups/
```

---

### **5. UPS Integration (Hardware Layer)**  
**Recommendation**: Use a USB UPS like APC Back-UPS.  

#### **Software Configuration**:
```bash
sudo apt install nut
sudo nano /etc/nut/ups.conf
```
```ini
[apcups]
    driver = usbhid-ups
    port = auto
```

**Automatic Shutdown Script**:
```bash
#!/bin/bash
upslog -l | grep "On battery" && \
docker-compose -f ~/docker/homeassistant/docker-compose.yml down && \
shutdown -h now
```

---

### **6. Docker Healthchecks (Resilience Layer)**  
**Purpose**: Detect and recover from "zombie" containers.  

#### **Enhanced docker-compose.yml**:
```yaml
services:
  homeassistant:
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8123"]
      interval: 1m
      timeout: 10s
      retries: 3
    restart: unless-stopped

  cloudflared:
    healthcheck:
      test: ["CMD", "cloudflared", "tunnel", "list"]
      interval: 2m
      timeout: 20s
      retries: 2
    restart: unless-stopped
```

---

### **7. Network Configuration (Stability Layer)**  
**Problem**: DHCP IP changes break Cloudflared & HA integrations.  

#### **Static IP Setup**:
```bash
sudo nano /etc/dhcpcd.conf
```
```ini
interface eth0
static ip_address=10.0.0.124/24
static routers=10.0.0.1
static domain_name_servers=1.1.1.1 8.8.8.8
```

---

### **Full Recovery Workflow**  
**When power is restored**:
1. UPS provides clean shutdown (if available)
2. Docker auto-restarts (`systemctl enable docker`)
3. Containers restart with `unless-stopped` policy
4. Healthchecks verify service availability
5. HA uses last backup if DB corrupted

---

### **Key Takeaways**  
1. **Defense in Depth**: Multiple layers (Docker + Systemd + Hardware)  
2. **Write Reduction**: OverlayFS extends SD card lifespan  
3. **Automated Recovery**: Restart policies + healthchecks  
4. **Backup Discipline**: Daily backups prevent data loss  

Implement all layers for enterprise-grade reliability. Let me know if you need help with any specific component!
