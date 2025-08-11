# HA Nginx Reverse Proxy with Keepalived + Automatic Config Sync via lsyncd (Restricted SSH)

This setup will:
- Create an **active–passive Nginx HA cluster** using **Keepalived** and VRRP.
- Monitor the health of Nginx using a custom health check script.
- Automatically sync Nginx configuration files from the **Master** (`lb1`) to the **Backup** (`lb2`) using `lsyncd`.
- Restrict SSH so the Backup server only allows rsync for `/etc/nginx` and `systemctl reload nginx`.

---

## **1. Environment**
- **Master (lb1):** 192.168.55.18
- **Backup (lb2):** 192.168.55.19
- **Virtual IP (VIP):** 192.168.55.17

---

## **2. Install Keepalived on Both Nginx Servers** *(lb1 & lb2)*
```bash
sudo apt update
sudo apt install -y keepalived
```

---

## **3. Create the Health Check Script** *(lb1 & lb2)*
```bash
sudo vim /opt/healthcheck.sh
```
```bash
#!/bin/bash
URL="http://localhost/health"
RESPONSE=$(curl -s -o /dev/null -w "%{http_code}" "$URL")

if [ "$RESPONSE" == "200" ]; then
    logger "Nginx health check: OK"
    exit 0
else
    logger "Nginx health check: DOWN"
    exit 1
fi
```
Make it executable:
```bash
sudo chmod +x /opt/healthcheck.sh
```

---

## **4. Configure Keepalived**

### **Master Server (lb1 - 192.168.55.18)** *(lb1 only)*
```bash
sudo vim /etc/keepalived/keepalived.conf
```
```conf
vrrp_script chk_http_port {
    script "/opt/healthcheck.sh"
    interval 5
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state MASTER
    interface ens160
    virtual_router_id 25
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mypassword
    }
    virtual_ipaddress {
        192.168.55.17
    }
    track_script {
        chk_http_port
    }
}
```

---

### **Backup Server (lb2 - 192.168.55.19)** *(lb2 only)*
```bash
sudo vim /etc/keepalived/keepalived.conf
```
```conf
vrrp_script chk_http_port {
    script "/opt/healthcheck.sh"
    interval 5
    fall 2
    rise 2
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 25
    priority 99
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass mypassword
    }
    virtual_ipaddress {
        192.168.55.17
    }
    track_script {
        chk_http_port
    }
}
```

---

## **5. Enable and Start Keepalived** *(lb1 & lb2)*
```bash
sudo systemctl enable keepalived
sudo systemctl start keepalived
sudo systemctl status keepalived
```
Check MASTER/Backup status:
```bash
journalctl -u keepalived --no-pager | grep -i "state"
```

---

## **6. Create a Restricted Sync User** *(lb2 only)*
```bash
sudo adduser --system --home /var/empty --shell /bin/bash nginxsync
sudo mkdir -p /etc/nginxsync
sudo chown root:root /etc/nginxsync
```

---

## **7. Install rrsync for Restricted rsync** *(lb2 only)*
```bash
sudo apt install -y rsync
sudo cp /usr/share/doc/rsync/scripts/rrsync /usr/local/bin/
sudo chmod +x /usr/local/bin/rrsync
```

---

## **8. Configure Restricted SSH Key** *(lb2 only)*
```bash
sudo mkdir -p /home/nginxsync/.ssh
sudo chown nginxsync:nginxsync /home/nginxsync/.ssh
sudo chmod 700 /home/nginxsync/.ssh
sudo vim /home/nginxsync/.ssh/authorized_keys
```
Add (single line):
```text
command="/usr/local/bin/rrsync /etc/nginx",no-port-forwarding,no-agent-forwarding,no-X11-forwarding ssh-rsa AAAAB3...your-public-key...
```

---

## **9. Allow Nginx Reload via Sudo** *(lb2 only)*
```bash
sudo visudo
```
Add:
```text
nginxsync ALL=(ALL) NOPASSWD: /bin/systemctl reload nginx
```

---

## **10. Install and Configure lsyncd** *(lb1 only)*
```bash
sudo apt install -y lsyncd
sudo vim /etc/lsyncd.conf.lua
```
```lua
settings {
    logfile    = "/var/log/lsyncd/lsyncd.log",
    statusFile = "/var/log/lsyncd/lsyncd.status",
    statusInterval = 20
}

sync {
    default.rsyncssh,
    source = "/etc/nginx/",
    host = "nginxsync@192.168.55.19",
    targetdir = "/etc/nginx/",
    rsync = {
        archive = true,
        compress = true,
        verbose = true
    },
    ssh = {
        port = 22
    },
    delay = 2,
    postcmd = "/usr/bin/ssh nginxsync@192.168.55.19 'sudo systemctl reload nginx'"
}
```

---

## **11. Set Up SSH Key** *(lb1 only)*
```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id nginxsync@192.168.55.19
```

---

## **12. Start lsyncd** *(lb1 only)*
```bash
sudo systemctl enable lsyncd
sudo systemctl start lsyncd
sudo systemctl status lsyncd
```

---

## **13. Test Restricted Sync** *(from lb1 to lb2)*
```bash
rsync -avz /etc/nginx/ nginxsync@192.168.55.19:/etc/nginx/
ssh nginxsync@192.168.55.19 'sudo systemctl reload nginx'
```
Check that normal shell login is **not** possible:
```bash
ssh nginxsync@192.168.55.19
```

---

## **14. How It Works**
1. Keepalived monitors Nginx health every 5 seconds.
2. If Master fails, VIP moves to Backup.
3. lsyncd keeps `/etc/nginx` synced in real time.
4. SSH is restricted to only syncing `/etc/nginx` and reloading Nginx.
