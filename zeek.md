# Installing Zeek on Raspberry Pi 4 (Docker) + Integration with Elastic (Fleet)
> A simplified guide with common issue solutions — tested in November 2025

## The Concept
- Mirror network traffic via **Port Mirroring** on the switch (SPAN) to the Raspberry Pi port.
- Run **Zeek** inside Docker and have it write **JSON** directly.
- Add **Zeek Integration** in Fleet (Elastic Agent) and point paths to the new JSON files.

## Prerequisites
- Managed switch that supports Port Mirroring.
- Raspberry Pi 4 with 64-bit OS and Docker installed.
- Elastic Agent enrolled in Fleet (or Filebeat as an alternative).
- Logs directory on host: `/home/pi/zeek/logs`.

## Setting Up Port Mirroring (Brief)
- **Destination Port** = The port connected to the Raspberry Pi cable.
- **Source Ports** = STC modem/router port + **WAN** port for Google/Nest Wifi.
- Enable **RX & TX** on source ports.

## Running Zeek in JSON Format
Create a script that enforces JSON and specifies the logs directory:
```bash
mkdir -p /home/pi/zeek
cat > /home/pi/zeek/json.zeek <<'EOF'
@load tuning/json-logs
redef LogAscii::json_timestamps = JSON::TS_ISO8601;
redef Log::default_logdir = "/logs";
EOF
```

Run the container:
```bash
docker rm -f zeek 2>/dev/null

docker run --name zeek --net=host   --cap-add=NET_ADMIN --cap-add=NET_RAW   --security-opt seccomp=unconfined --security-opt apparmor=unconfined   --restart unless-stopped   -v /home/pi/zeek/logs:/logs   -v /home/pi/zeek/json.zeek:/json.zeek:ro   -d zeek/zeek:lts   zeek -C -i eth0 /json.zeek
```

### Verify It's Writing JSON
```bash
head -n1 /home/pi/zeek/logs/conn.log
head -n1 /home/pi/zeek/logs/dns.log
# Lines must start with { ... }
```

> **Note:** Running `zeek -e 'print LogAscii::use_json'` from exec starts a new session that doesn't load your script — it's normal to get F. What matters is the actual files.

### (Optional) Disable Offloading on eth0
Improves analysis with mirrored ports:
```bash
sudo apt-get update && sudo apt-get install -y ethtool
sudo ethtool -K eth0 gro off lro off gso off tso off
```

## Adding Zeek Integration in Fleet
In Kibana → Fleet → Policies → Integration: Zeek  
Configure paths **one file per type** (without wildcards):
- connection → `/home/pi/zeek/logs/conn.log`
- dns → `/home/pi/zeek/logs/dns.log`
- files → `/home/pi/zeek/logs/files.log`
- http → `/home/pi/zeek/logs/http.log`
- ssl → `/home/pi/zeek/logs/ssl.log`
- dhcp → `/home/pi/zeek/logs/dhcp.log`
- notice → `/home/pi/zeek/logs/notice.log`
- weird → `/home/pi/zeek/logs/weird.log`
- x509 → `/home/pi/zeek/logs/x509.log`
- quic → `/home/pi/zeek/logs/quic.log`
- ocsp → `/home/pi/zeek/logs/ocsp.log`
- ntp → `/home/pi/zeek/logs/ntp.log`
- software → `/home/pi/zeek/logs/software.log`
- tunnel → `/home/pi/zeek/logs/tunnel.log`

Restart the agent after saving:
```bash
sudo systemctl restart elastic-agent
```

## Key Issues and Solutions
**1) JSON error in pipeline (Unexpected character '#' / Unrecognized token …)**  
Cause: Zeek is writing **TSV** not JSON.  
Solution: Enable JSON as shown above + ensure paths point to JSON files only.  
> Don't use `*.log` as it may read old TSV files.

**2) Option -l requires an argument**  
Cause: Using `-l` without a path (or broken command line).  
Solution: Either `-l /logs` or use `Log::default_logdir="/logs"` inside the script.

**3) Container restart loop**  
Add capabilities: `--cap-add=NET_ADMIN --cap-add=NET_RAW --security-opt seccomp=unconfined --security-opt apparmor=unconfined`

**4) zeek: not found inside exec**  
Use the full path: `/usr/local/zeek/bin/zeek`

**5) No log files generated**  
- Check `docker logs` to see if it prints `listening on eth0`.
- Generate traffic (YouTube/downloads).
- Try `-i any` or multiple interfaces with VLAN/WiFi.
- Disable offloading on eth0.
- Review Port Mirroring settings.

## Verifying in Elasticsearch
```bash
curl -s 'http://ES_VM_IP:9200/_cat/data_stream?v' | grep zeek
curl -s 'http://ES_VM_IP:9200/logs-zeek.connection-*/_count'
curl -s 'http://ES_VM_IP:9200/logs-zeek.dns-*/_count'
```

## Alternative: Filebeat Module
```yaml
filebeat.modules:
- module: zeek
  connection:
    enabled: true
    var.paths: ["/home/pi/zeek/logs/conn.log"]
  dns:
    enabled: true
    var.paths: ["/home/pi/zeek/logs/dns.log"]
  dhcp:
    enabled: true
    var.paths: ["/home/pi/zeek/logs/dhcp.log"]

output.elasticsearch:
  hosts: ["http://ES_VM_IP:9200"]
  username: "elastic"
  password: "****"
```

---
Note: Paths and commands were tested on RPi4 with `zeek/zeek:lts` (currently 8.0.x). Adjust the capture interface `-i eth0` according to your network.
