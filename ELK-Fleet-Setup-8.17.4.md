# ELK Home on Ubuntu 24 (Docker Compose) — 8.17.4

A practical guide to running **Elasticsearch + Kibana** on Ubuntu 24 inside a VM using Docker Compose, then adding **Fleet Server** and registering **Elastic Agent**. This guide includes common issues we encountered and their solutions.

---

## 0) Prerequisites

- Updated Ubuntu Server 24.04
- Docker and Docker Compose
- Open ports:
  - 9200 for Elasticsearch
  - 5601 for Kibana
  - 8220 for Fleet Server
- 8 GB RAM or more recommended
- Sufficient disk space for ES data

Install basic requirements:

```bash
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release apt-transport-https
# Docker (if not installed)
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker
```

---

## 1) System Configuration (vm.max_map_count)

```bash
sudo sysctl -w vm.max_map_count=262144
echo vm.max_map_count=262144 | sudo tee -a /etc/sysctl.conf
```

---

## 2) Directory Structure

```
~/elk
│── docker-compose.yml
│── .env
├── elasticsearch/          # ES data
└── kibana/                 # Optional: static kibana.yml
```

---

## 3) `.env` File (Example)

> Adjust values according to your system resources.

```bash
ELASTIC_VERSION=8.17.4
ELASTIC_PASSWORD=Elastic123!
ES_HEAP=2g
```

**Important:** Don't run `source .env` in the shell if you're constantly changing values, because **session variables override** the `.env` file when running `docker compose`. If you must `source`, later use `unset VAR_NAME` before running.

---

## 4) `docker-compose.yml` File (Base: Elasticsearch + Kibana)

```yaml
services:
  es01:
    image: docker.elastic.co/elasticsearch/elasticsearch:${ELASTIC_VERSION}
    container_name: es01
    environment:
      - node.name=es01
      - discovery.type=single-node
      - xpack.security.enabled=true
      - xpack.security.http.ssl.enabled=false
      - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
      - ES_JAVA_OPTS=-Xms${ES_HEAP} -Xmx${ES_HEAP}
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - ./elasticsearch:/usr/share/elasticsearch/data
    ports:
      - 9200:9200
    healthcheck:
      test: [ "CMD-SHELL", "curl -fsS -u elastic:${ELASTIC_PASSWORD} http://localhost:9200 > /dev/null" ]
      interval: 20s
      timeout: 5s
      retries: 30

  kibana:
    image: docker.elastic.co/kibana/kibana:${ELASTIC_VERSION}
    container_name: kibana
    depends_on:
      es01:
        condition: service_healthy
    environment:
      - ELASTICSEARCH_HOSTS=http://es01:9200
      # We'll add the service token later:
      # - ELASTICSEARCH_SERVICEACCOUNTTOKEN=${KIBANA_SA_TOKEN}
      # (Optional later to remove warnings)
      # - xpack.encryptedSavedObjects.encryptionKey=long_random_key_1_____________________
      # - xpack.security.encryptionKey=long_random_key_2_________________________________
      # - xpack.reporting.encryptionKey=long_random_key_3________________________________
    ports:
      - 5601:5601
```

---

## 5) First Run and Verification

```bash
cd ~/elk
docker compose up -d
docker compose ps
```

Quick check:
```bash
curl -u elastic:${ELASTIC_PASSWORD} http://127.0.0.1:9200
```

---

## 6) **Kibana** Should Not Use `elastic` User Internally — Use **Service Account Token**

Starting from 8.x, using the `elastic` user for internal Kibana connection is prohibited. We generate a **stored token** via REST (it remains saved inside `.security` and won't be lost when recreating containers).

### 6.1 Create Stored Token via REST (Without Body)

```bash
# Enable .env for the session (optional for testing)
set -a; source .env; set +a

# Create the token:
RESP=$(curl -s -u "elastic:${ELASTIC_PASSWORD}" \
  -X POST 'http://127.0.0.1:9200/_security/service/elastic/kibana/credential/token')

# Extract token value:
KIBANA_TOKEN=$(printf '%s' "$RESP" | sed -n 's/.*"token"[^{]*{[^}]*"value"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p')
echo "$KIBANA_TOKEN"
```

Test the token against Elasticsearch:
```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer ${KIBANA_TOKEN}" http://127.0.0.1:9200
curl -s -H "Authorization: Bearer ${KIBANA_TOKEN}" http://127.0.0.1:9200/_security/_authenticate?pretty
```

### 6.2 Save Token and Connect Kibana

```bash
# Store it in .env
sed -i '/^KIBANA_SA_TOKEN=/d' .env
echo KIBANA_SA_TOKEN=${KIBANA_TOKEN} >> .env

# Don't leave an old exported copy that overrides
unset KIBANA_SA_TOKEN

# Add the variable to Kibana service in docker-compose.yml under environment:
# - ELASTICSEARCH_SERVICEACCOUNTTOKEN=${KIBANA_SA_TOKEN}

# Apply the change
docker compose up -d --force-recreate kibana

# Verify that the container read the new value
docker inspect kibana | grep -i ELASTICSEARCH_SERVICEACCOUNTTOKEN

# Monitor the logs
docker logs kibana --tail=120
```

Access the interface:
```
http://<VM_IP>:5601
```
Log in with the `elastic` account and password from `.env` (the prohibition is only on internal connection, not on interface login).

---

## 7) Common Issues and Solutions

- **es01 health: starting / unhealthy**  
  Low vm.max_map_count or data directory permissions:
  ```bash
  sudo sysctl -w vm.max_map_count=262144
  sudo chown -R 1000:0 ~/elk/elasticsearch
  docker compose down && docker compose up -d
  ```

- **es01 unhealthy due to healthcheck returning 401**  
  Because security is enabled — authentication is used in healthcheck.

- **Kibana Exited (78)**  
  You were using `elastic` for internal connection. Solution: Service Account Token (or `kibana_system`).

- **security_exception in Kibana**  
  Invalid token or container reading old token. Solution:
  1. Create stored token via REST.
  2. Ensure no duplicate `KIBANA_SA_TOKEN` in `.env`.
  3. Don't run `source .env` before `docker compose` or execute `unset KIBANA_SA_TOKEN`.
  4. **Force recreate** Kibana and verify the variable value inside the container.

- **no such index .security**  
  First run didn't create the security index. Run a quick Security operation (example: create user):
  ```bash
  curl -s -u elastic:${ELASTIC_PASSWORD} -H 'Content-Type: application/json' \
    -X POST http://127.0.0.1:9200/_security/user/kbn_bootstrap \
    -d '{"password":"Temp123!","roles":["kibana_admin"]}'
  ```

- **encryptionKey warnings in Kibana**  
  Not an operational error. Set static keys to cancel warnings (see comments in compose).

- **UFW**  
  Leave it disabled during setup. When enabling:
  ```bash
  sudo ufw allow 22/tcp
  sudo ufw allow 9200,5601,5044,8220/tcp
  sudo ufw enable
  ```

---

## 8) Useful Commands

```bash
# Service status
docker compose ps
docker ps

# Listening ports
ss -lptn '( sport = :9200 or sport = :5601 or sport = :8220 )'

# Logs
docker logs es01 --tail=120
docker logs kibana --tail=120

# Cluster health
curl -u elastic:${ELASTIC_PASSWORD} http://127.0.0.1:9200/_cluster/health?pretty

# Security indices
curl -u elastic:${ELASTIC_PASSWORD} http://127.0.0.1:9200/_cat/indices/.security*?v
```

---

# (Optional) 9) Adding **Fleet Server** and Running **Elastic Agent**

> Your environment now: ES + Kibana running on Docker **without TLS**.  
> Goal: Run **Fleet Server** as a Docker service, configure Enrollment Tokens, then install **Elastic Agent** on your machine.

### 9.1 Additional Variables

Add to `.env` (or make sure they exist):

```bash
ELASTIC_VERSION=8.17.4
FLEET_SERVER_PORT=8220
```

Calculate the server address (the IP that agents will connect to):
```bash
HOST_IP=$(hostname -I | awk '{print $1}')
echo "HOST_IP=$HOST_IP"
# (You can put it manually in compose as an explicit string)
```

Open the port if UFW is enabled:
```bash
sudo ufw allow 8220/tcp || true
```

### 9.2 Create **Service Token** for Fleet Server (stored token via REST)

```bash
set -a; source .env; set +a

RESP=$(curl -s -u "elastic:${ELASTIC_PASSWORD}" \
  -X POST 'http://127.0.0.1:9200/_security/service/elastic/fleet-server/credential/token')

FLEET_SERVER_SERVICE_TOKEN=$(printf '%s' "$RESP" | sed -n 's/.*"value"[[:space:]]*:[[:space:]]*"\([^"]*\)".*/\1/p')
echo "$FLEET_SERVER_SERVICE_TOKEN"

# Store it in .env
sed -i '/^FLEET_SERVER_SERVICE_TOKEN=/d' .env
echo "FLEET_SERVER_SERVICE_TOKEN=${FLEET_SERVER_SERVICE_TOKEN}" >> .env
```

Test it (optional):
```bash
curl -s -o /dev/null -w "%{http_code}\n" -H "Authorization: Bearer ${FLEET_SERVER_SERVICE_TOKEN}" http://127.0.0.1:9200
```

### 9.3 Add **fleet** Service in `docker-compose.yml`

> Add the following service alongside es01 and kibana. Replace `${HOST_IP}` with your actual server address (you can put it as a fixed value instead of variable).

```yaml
  fleet:
    image: docker.elastic.co/beats/elastic-agent:${ELASTIC_VERSION}
    container_name: fleet
    restart: unless-stopped
    environment:
      - FLEET_SERVER_ENABLE=true
      - FLEET_SERVER_ELASTICSEARCH_HOST=http://es01:9200
      - FLEET_SERVER_SERVICE_TOKEN=${FLEET_SERVER_SERVICE_TOKEN}
      # The address that agents will connect to
      - FLEET_URL=http://${HOST_IP}:${FLEET_SERVER_PORT}
    ports:
      - "${FLEET_SERVER_PORT}:8220"
    depends_on:
      es01:
        condition: service_healthy
      kibana:
        condition: service_started
```

Start the service:
```bash
docker compose up -d fleet
docker logs -f fleet | sed -n '1,160p'
```

### 9.4 Configure **Fleet Server Hosts** inside Kibana

From the interface: **Kibana → Management → Fleet → Settings → Fleet server hosts**  
Set:
```
http://<HOST_IP>:8220
```
Save.

> The goal is for **Enrollment Tokens** to distribute the correct URL to agents.

### 9.5 Create **Enrollment Token** for Agents

From the interface: **Kibana → Fleet → Enrollment tokens → Create** (for Default policy or specific policy).  
(Can also be done via Kibana API, but the interface is easier).

### 9.6 Install **Elastic Agent** on Your Machine

#### Option (a): As a Package on Ubuntu

```bash
# Download and install (same version 8.17.4)
curl -L -O https://artifacts.elastic.co/downloads/beats/elastic-agent/elastic-agent-8.17.4-amd64.deb
sudo dpkg -i elastic-agent-8.17.4-amd64.deb

# Enroll (use --insecure if Fleet is without TLS)
sudo elastic-agent install \
  --url="http://<HOST_IP>:8220" \
  --enrollment-token="YOUR_ENROLLMENT_TOKEN" \
  --insecure
```

#### Option (b): As a Container (quick for testing)

```bash
docker run -d --name agent --restart unless-stopped --privileged --net=host \
  -e FLEET_ENROLL=1 \
  -e FLEET_URL="http://<HOST_IP>:8220" \
  -e FLEET_ENROLLMENT_TOKEN="YOUR_ENROLLMENT_TOKEN" \
  -v /:/hostfs:ro -v /var/run/docker.sock:/var/run/docker.sock \
  docker.elastic.co/beats/elastic-agent:8.17.4
```

### 9.7 Verification

- Kibana → Fleet → Agents: You'll see **Fleet Server** with **Healthy** status and agents will appear after moments.
- Fleet Server logs:
  ```bash
  docker logs fleet --tail=200
  ```
- Elastic Agent logs (inside container):
  ```
  /usr/share/elastic-agent/state/data/logs/*
  ```

### 9.8 Common Failures (Fleet/Agent)

- **failed to authenticate service account [elastic/fleet-server]**  
  Create a new Service Token and store it in `.env` then:
  ```bash
  docker compose up -d --force-recreate fleet
  ```

- **Agent doesn't enroll (Not enrolled)**  
  - Verify **Fleet Server hosts** in Settings is `http://<HOST_IP>:8220`.
  - Open port 8220 in firewall.
  - Verify the **Enrollment Token** is correct.
  - If without TLS, use `--insecure`.

---

## 10) Security Considerations (for Production Later)

- Enable TLS for Elasticsearch, Fleet Server, and Kibana (CA + certificates).
- Set encryption keys in Kibana (`xpack.*.encryptionKey`) instead of random generation.
- Use policies with least privilege when possible, and monitor periodic tokens.

---

Done — This is how you run ELK + Fleet on Docker, properly set up tokens, solved the most common authentication and health issues, and prepared Elastic Agent to enroll.
