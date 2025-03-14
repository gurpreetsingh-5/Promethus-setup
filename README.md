# Prometheus with Node Exporter and Grafana 
# Prometheus & Node Exporter Setup on AWS EC2

This guide explains how to install and configure **Prometheus** and **Node Exporter** on an AWS EC2 instance for system monitoring.

---

## **1. Launch an EC2 Instance**
1. Go to **AWS Console → EC2**.
2. Click **Launch Instance**.
3. Choose **Ubuntu 22.04** or **Amazon Linux 2**.
4. Select an instance type (**t2.micro** is sufficient for testing).
5. Configure security group:
   - **Port 9090 (TCP)** → Prometheus
   - **Port 9100 (TCP)** → Node Exporter
   - **Port 22 (TCP)** → SSH (for access)
6. Launch and connect to the instance via SSH:
   ```bash
   ssh -i your-key.pem ubuntu@your-ec2-public-ip
   ```

---

## **2. Install Prometheus**

### **1️⃣ Create a Prometheus User**
```bash
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir /etc/prometheus /var/lib/prometheus
sudo chown prometheus:prometheus /etc/prometheus /var/lib/prometheus
```

### **2️⃣ Download and Install Prometheus**
```bash
cd /tmp
curl -LO  https://github.com/prometheus/prometheus/releases/download/v2.16.0/prometheus-2.16.0.linux-amd64.tar.gz
tar -xvzf prometheus-2.16.0.linux-amd64.tar.gz
cd prometheus-2.16.0.linux-amd64.tar.gz
sudo cp prometheus promtool /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/prometheus /usr/local/bin/promtool
```

### **3️⃣ Move Configuration Files**
```bash
sudo cp -r consoles console_libraries /etc/prometheus/
sudo cp prometheus.yml /etc/prometheus/
sudo chown -R prometheus:prometheus /etc/prometheus
```

### **4️⃣ Create Prometheus Systemd Service**
```bash
sudo tee /etc/systemd/system/prometheus.service <<EOF
[Unit]
Description=Prometheus Monitoring
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus/ \
    --web.listen-address=0.0.0.0:9090 \
    --storage.tsdb.retention.time=15d

[Install]
WantedBy=multi-user.target
EOF
```

### **5️⃣ Start Prometheus**
```bash
sudo systemctl daemon-reload
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```
Check if Prometheus is running:
```bash
curl http://localhost:9090
```

---


## **3. Install Node Exporter**

### **1️⃣ Create Node Exporter User**
```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

### **2️⃣ Download and Install**
```bash
cd /tmp
curl -LO  https://github.com/prometheus/node_exporter/releases/download/v0.18.1/node_exporter-0.18.1.linux-amd64.tar.gz
tar -xvzf node_exporter-0.18.1.linux-amd64.tar.gz
cd node_exporter-0.18.1.linux-amd64.tar.gz
sudo cp node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
```

### **3️⃣ Create Systemd Service**
```bash
sudo tee /etc/systemd/system/node_exporter.service <<EOF
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
EOF
```

### **4️⃣ Start Node Exporter**
```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```
Verify by running:
```bash
curl http://localhost:9100/metrics
```

---

## **4. Configure Prometheus to Scrape Node Exporter**
Edit the Prometheus configuration file:
```bash
sudo nano /etc/prometheus/prometheus.yml
```
Add the following scrape job:
```yaml
scrape_configs:
  - job_name: 'node_exporter'
    static_configs:
      - targets: ['localhost:9100']
```
Restart Prometheus:
```bash
sudo systemctl restart prometheus
```

---

## **5. Verify Setup**

1. Open Prometheus UI:
   ```
   http://<your-ec2-public-ip>:9090
   ```
2. Go to `Status → Targets` and ensure **node_exporter** is listed.
3. Run the query:
   ```promql
   node_memory_MemTotal_bytes
   ```

---

## **6. Optional: Install Grafana for Visualization**

```bash
sudo apt install -y apt-transport-https software-properties-common
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"
sudo apt update
sudo apt install -y grafana
sudo systemctl start grafana-server
sudo systemctl enable grafana-server
```

Access Grafana at:
```
http://<your-ec2-public-ip>:3000
```
(Default login: `admin` / `admin`)

---

## 🎯 **Final Check**
- **Prometheus UI:** `http://<your-ec2-public-ip>:9090`
- **Node Exporter Metrics:** `http://<your-ec2-public-ip>:9100/metrics`
- **Grafana (if installed):** `http://<your-ec2-public-ip>:3000`

✅ Your monitoring setup is now complete!

