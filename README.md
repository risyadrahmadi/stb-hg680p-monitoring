# STB HG680P — Monitoring Stack

This repository documents the implementation of a monitoring stack for an **HG680P Linux home server** running Armbian.

The monitoring environment is built using:

* **Prometheus** — metrics collection and storage
* **Grafana** — metrics visualization and dashboards
* **Node Exporter** — host system metrics
* **cAdvisor** — Docker container metrics

All monitoring components are deployed using **Docker Compose**.

The purpose of this project is to provide real-time visibility into the condition of the HG680P home server, including system resources and Docker container activity.

---

## Project Overview

The HG680P was previously configured as a Linux-based home server using Armbian, Docker, CasaOS, and Portainer.

The next step is to add a monitoring system so that system resources and Docker containers can be observed through a centralized dashboard.

The monitoring architecture consists of four main components:

```text
                    ┌─────────────────┐
                    │     Grafana     │
                    │   Visualization │
                    └────────┬────────┘
                             │
                             ▼
                    ┌─────────────────┐
                    │    Prometheus   │
                    │ Metrics Storage │
                    └───────┬─┬───────┘
                            │ │
              ┌─────────────┘ └─────────────┐
              │                             │
              ▼                             ▼
     ┌─────────────────┐           ┌─────────────────┐
     │  Node Exporter  │           │    cAdvisor     │
     │ Host Metrics    │           │ Docker Metrics  │
     └─────────────────┘           └─────────────────┘
```

Node Exporter collects operating-system metrics such as CPU, memory, disk, and network usage.

cAdvisor collects metrics from Docker containers.

Prometheus scrapes and stores these metrics.

Grafana connects to Prometheus and displays the metrics through dashboards.

---

# 1. Monitoring Components

## Prometheus

Prometheus is responsible for collecting and storing time-series metrics.

In this project, Prometheus collects metrics from:

* Prometheus itself
* Node Exporter
* cAdvisor

The default scrape interval is configured to 15 seconds.

---

## Grafana

Grafana is used as the visualization layer.

It connects to Prometheus as its datasource and provides dashboards for:

* CPU usage
* Memory usage
* Disk usage
* Network traffic
* Filesystem usage
* System load
* System uptime
* Docker container resources

---

## Node Exporter

Node Exporter collects metrics from the Linux host.

The collected metrics include information about:

* CPU
* Memory
* Disk
* Filesystem
* Network
* Load average
* System uptime

---

## cAdvisor

cAdvisor collects Docker container metrics.

The collected information includes:

* Container CPU usage
* Container memory usage
* Network RX/TX
* Filesystem usage
* Running containers
* Container restart information

---

# 2. Monitoring Directory

All monitoring configuration and persistent data are stored under:

```text
/opt/monitoring
```

Create the main directory:

```bash
sudo mkdir -p /opt/monitoring
```

Create the configuration directory:

```bash
sudo mkdir -p /opt/monitoring/monitoring-data/config
```

Create the Prometheus data directory:

```bash
sudo mkdir -p /opt/monitoring/monitoring-data/prometheus_data
```

Create the Grafana data directory:

```bash
sudo mkdir -p /opt/monitoring/monitoring-data/grafana_data
```

The resulting layout is:

```text
/opt/monitoring/
├── docker-compose.yml
└── monitoring-data/
    ├── config/
    │   ├── web-config.yml
    │   ├── prometheus.yml
    │   └── datasources.yml
    ├── prometheus_data/
    └── grafana_data/
```

The directories are separated so that configuration and persistent application data are not stored directly inside the containers.

---

# 3. Configure Prometheus Authentication

Prometheus can be configured with Basic Authentication to prevent unrestricted access to the metrics endpoint.

Create the authentication configuration:

```bash
sudo touch /opt/monitoring/monitoring-data/config/web-config.yml
```

The configuration will contain the Prometheus administrator username and a bcrypt password hash.

---

## 3.1 Install Python bcrypt

Install the required Python package:

```bash
sudo apt update
```

```bash
sudo apt install python3-bcrypt -y
```

---

## 3.2 Generate a bcrypt Password Hash

Create a temporary Python script:

```bash
nano generate_pass.py
```

Use:

```python
import bcrypt

password = "YOUR_PASSWORD"

hashed_password = bcrypt.hashpw(
    password.encode("utf-8"),
    bcrypt.gensalt()
)

print(hashed_password.decode())
```

Run the script:

```bash
python3 generate_pass.py
```

The command produces a bcrypt password hash.

The generated hash is then used in the Prometheus authentication configuration.

Do not commit the real password or generated credentials to a public repository.

---

# 4. Configure Prometheus Authentication File

Open the configuration file:

```bash
sudo nano /opt/monitoring/monitoring-data/config/web-config.yml
```

Use the following structure:

```yaml
basic_auth_users:
  admin: YOUR_BCRYPT_HASH
```

Replace:

```text
YOUR_BCRYPT_HASH
```

with the bcrypt hash generated in the previous step.

Verify the configuration:

```bash
cat /opt/monitoring/monitoring-data/config/web-config.yml
```

The file should contain the `admin` user and the generated bcrypt hash.

---

# 5. Configure Prometheus

Create the main Prometheus configuration:

```bash
sudo touch /opt/monitoring/monitoring-data/config/prometheus.yml
```

Open the file:

```bash
sudo nano /opt/monitoring/monitoring-data/config/prometheus.yml
```

Use:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:

  - job_name: "prometheus"
    static_configs:
      - targets:
          - "localhost:9090"
    basic_auth:
      username: "admin"
      password: "YOUR_PASSWORD"

  - job_name: "cadvisor"
    static_configs:
      - targets:
          - "cadvisor:8080"

  - job_name: "VM-STB"
    static_configs:
      - targets:
          - "node-exporter:9100"
```

Replace:

```text
YOUR_PASSWORD
```

with the actual Prometheus password used by the deployment.

For a public repository, do not commit the real password.

The Prometheus configuration defines three primary monitoring targets:

```text
Prometheus
cAdvisor
Node Exporter
```

Prometheus scrapes the targets every 15 seconds.

---

# 6. Configure Grafana Datasource

Grafana needs a Prometheus datasource before it can visualize the collected metrics.

Create the datasource configuration:

```bash
sudo touch /opt/monitoring/monitoring-data/config/datasources.yml
```

Open it:

```bash
sudo nano /opt/monitoring/monitoring-data/config/datasources.yml
```

Use:

```yaml
apiVersion: 1

datasources:
  - name: prometheus
    type: prometheus
    url: http://prometheus:9090
    access: proxy
    isDefault: true
    basicAuth: true
    basicAuthUser: admin
    secureJsonData:
      basicAuthPassword: "YOUR_PASSWORD"
```

This configuration automatically provisions Prometheus as the default Grafana datasource.

The URL:

```text
http://prometheus:9090
```

uses the Docker Compose service name because the containers communicate through the Docker network.

---

# 7. Docker Compose Monitoring Stack

Create the Docker Compose file:

```bash
sudo touch /opt/monitoring/docker-compose.yml
```

Open it:

```bash
sudo nano /opt/monitoring/docker-compose.yml
```

Use the following configuration:

```yaml
services:

  # Docker container monitoring
  cadvisor:
    container_name: monitoring-cadvisor
    image: "gcr.io/cadvisor/cadvisor:v0.47.1"
    restart: always
    devices:
      - "/dev/kmsg:/dev/kmsg"
    volumes:
      - "/:/rootfs:ro"
      - "/sys:/sys:ro"
      - "/sys/fs/cgroup:/sys/fs/cgroup:ro"
      - "/var/lib/docker:/var/lib/docker:ro"
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "/dev/disk:/dev/disk:ro"
    privileged: true
    ports:
      - "8080:8080"

  # Linux host monitoring
  node-exporter:
    container_name: monitoring-node-exporter
    image: "prom/node-exporter:v1.7.0"
    restart: always
    command:
      - "--path.rootfs=/host"
    volumes:
      - "/:/host:ro"
    ports:
      - "9100:9100"

  # Metrics collection and storage
  prometheus:
    container_name: monitoring-prometheus
    user: root
    image: "prom/prometheus:v3.0.0"
    restart: always
    volumes:
      - "./monitoring-data/config/prometheus.yml:/etc/prometheus/prometheus.yml"
      - "./monitoring-data/config/web-config.yml:/etc/prometheus/web-config.yml"
      - "./monitoring-data/prometheus_data:/prometheus"
    command:
      - "--config.file=/etc/prometheus/prometheus.yml"
      - "--storage.tsdb.path=/prometheus"
      - "--storage.tsdb.retention.time=3d"
      - "--web.config.file=/etc/prometheus/web-config.yml"
    privileged: true
    ports:
      - "9090:9090"
    depends_on:
      - cadvisor
      - node-exporter

  # Metrics visualization
  grafana:
    user: root
    container_name: monitoring-grafana
    image: "grafana/grafana:11.5.0"
    restart: always
    volumes:
      - "./monitoring-data/config/datasources.yml:/etc/grafana/provisioning/datasources/datasources.yml"
      - "./monitoring-data/grafana_data:/var/lib/grafana"
    privileged: true
    ports:
      - "3000:3000"
    depends_on:
      - prometheus
```

---

# 8. Docker Compose Services

The Compose configuration contains four monitoring containers.

| Container                  | Purpose                        |   Port |
| -------------------------- | ------------------------------ | -----: |
| `monitoring-cadvisor`      | Docker container metrics       | `8080` |
| `monitoring-node-exporter` | Host system metrics            | `9100` |
| `monitoring-prometheus`    | Metrics collection and storage | `9090` |
| `monitoring-grafana`       | Metrics visualization          | `3000` |

---

## cAdvisor

cAdvisor requires access to several host resources in order to collect Docker and system information.

The Compose configuration mounts:

```text
/
 /sys
 /sys/fs/cgroup
 /var/lib/docker
 /var/run/docker.sock
 /dev/disk
```

The container also exposes port:

```text
8080
```

---

## Node Exporter

Node Exporter exposes host metrics through port:

```text
9100
```

The host filesystem is mounted at:

```text
/host
```

and the exporter uses:

```text
--path.rootfs=/host
```

to access the host filesystem.

---

## Prometheus

Prometheus uses port:

```text
9090
```

Its data is persisted in:

```text
/opt/monitoring/monitoring-data/prometheus_data
```

The project configures a three-day retention period:

```text
--storage.tsdb.retention.time=3d
```

Prometheus also uses:

```text
web-config.yml
```

for Basic Authentication.

---

## Grafana

Grafana uses port:

```text
3000
```

Persistent Grafana data is stored in:

```text
/opt/monitoring/monitoring-data/grafana_data
```

The Prometheus datasource is automatically provisioned through:

```text
datasources.yml
```

---

# 9. Start the Monitoring Stack

Enter the monitoring directory:

```bash
cd /opt/monitoring
```

Start all services:

```bash
sudo docker compose up -d
```

Docker will download the required images if they are not already available.

The expected containers are:

```text
monitoring-cadvisor
monitoring-node-exporter
monitoring-prometheus
monitoring-grafana
```

---

# 10. Verify Running Containers

Check the running containers:

```bash
sudo docker ps
```

All four monitoring containers should have a running status.

Expected services:

```text
monitoring-grafana
monitoring-prometheus
monitoring-node-exporter
monitoring-cadvisor
```

A status of `Up` indicates that the container is currently running.

---

# 11. Access the Monitoring Services

The monitoring services can be accessed from a computer connected to the same network.

Use the IP address of the STB.

| Service       | URL                     |
| ------------- | ----------------------- |
| Grafana       | `http://SERVER_IP:3000` |
| Prometheus    | `http://SERVER_IP:9090` |
| Node Exporter | `http://SERVER_IP:9100` |
| cAdvisor      | `http://SERVER_IP:8080` |

Replace:

```text
SERVER_IP
```

with the actual IP address of the STB.

---

# 12. Grafana Initial Configuration

Open:

```text
http://SERVER_IP:3000
```

On the first login, Grafana requests administrator credentials.

Use the administrator account configured during the initial Grafana setup.

After successful authentication, the Grafana dashboard interface becomes available.

---

# 13. Prometheus Authentication

Open:

```text
http://SERVER_IP:9090
```

Prometheus is protected by Basic Authentication.

Use:

```text
Username: admin
Password: YOUR_PASSWORD
```

After authentication, the Prometheus interface can be used to inspect collected metrics and execute PromQL queries.

---

# 14. Verify Node Exporter

Open:

```text
http://SERVER_IP:9100
```

Node Exporter does not provide a traditional login page.

A successful installation displays the available host metrics.

The metrics include information about:

* CPU
* Memory
* Disk
* Filesystem
* Network
* Load
* Uptime

---

# 15. Verify cAdvisor

Open:

```text
http://SERVER_IP:8080
```

cAdvisor provides information about Docker containers running on the host.

The interface can be used to inspect container-related metrics.

---

# 16. Verify Prometheus Targets

Prometheus provides a Targets page that can be used to verify whether the configured exporters are reachable.

The configured targets include:

```text
Prometheus
cAdvisor
Node Exporter
```

All expected targets should report an `UP` state.

An `UP` state indicates that Prometheus can successfully scrape the target.

---

# 17. Import Grafana Dashboards

After verifying the monitoring services, dashboards can be imported into Grafana.

Open:

```text
http://SERVER_IP:3000
```

Go to:

```text
Dashboards
→ New
→ Import
```

Enter the desired Grafana Dashboard ID and load the dashboard.

Select:

```text
Prometheus
```

as the datasource.

Then select:

```text
Import
```

to complete the process.

---

# 18. Node Exporter Full Dashboard

The first dashboard used in this project is:

```text
Dashboard ID: 1860
Name: Node Exporter Full
```

This dashboard provides detailed host-level monitoring.

It can display:

* CPU usage
* Memory usage
* Disk usage
* Network traffic
* Filesystem usage
* Load average
* System uptime

This dashboard is primarily used to monitor the Linux host itself.

---

# 19. cAdvisor Docker Dashboard

A second dashboard is used for Docker container monitoring:

```text
Dashboard ID: 19908
```

The cAdvisor dashboard can display:

* Container CPU usage
* Container memory usage
* Network RX/TX
* Filesystem usage
* Running containers
* Container restart information

Using separate dashboards allows system-level metrics and container-level metrics to be monitored independently.

---

# 20. Dashboard `No Data` Troubleshooting

During the initial dashboard test, some panels may display:

```text
No Data
```

even though the monitoring containers appear to be running.

The first step is to verify the Docker containers:

```bash
docker ps
```

All four monitoring containers should be running.

Next, verify the Prometheus Targets page.

If the targets are:

```text
UP
```

then Prometheus is successfully scraping the configured exporters.

This indicates that the problem is not necessarily caused by the Prometheus scrape configuration.

---

# 21. Check cAdvisor Logs

Inspect the latest cAdvisor logs:

```bash
sudo docker logs monitoring-cadvisor | tail -5
```

The logs can help determine whether cAdvisor is running correctly or encountering runtime problems.

---

# 22. Compare Different Environments

The monitoring stack was also tested in an Ubuntu virtual machine.

The same dashboard issue appeared in both environments.

The comparison helped identify that the problem was not specific to Armbian or the HG680P hardware.

The investigation then focused on the Docker Engine version and compatibility with cAdvisor.

---

# 23. Check Docker Engine Version

Check the current Docker version:

```bash
docker --version
```

During the original implementation, the environment was using:

```text
Docker version 29.6.2
```

The monitoring configuration itself was working, but cAdvisor dashboard data was not being displayed as expected.

Further testing indicated a compatibility issue involving the Docker Engine version and the cAdvisor environment.

---

# 24. Downgrade Docker Engine

To test the compatibility issue, Docker Engine was downgraded from version 29.6.2 to version 28.5.2.

Before changing the version, inspect available package versions:

```bash
apt-cache madison docker-ce
```

Check the Docker CLI versions:

```bash
apt-cache madison docker-ce-cli
```

Check containerd versions:

```bash
apt-cache madison containerd.io
```

Stop Docker:

```bash
sudo systemctl stop docker
```

Install the tested Docker Engine version:

```bash
sudo apt install --allow-downgrades \
docker-ce=5:28.5.2-1~ubuntu.24.04~noble \
docker-ce-cli=5:28.5.2-1~ubuntu.24.04~noble
```

The exact package version may no longer be available from the repository in future environments.

For that reason, always verify available versions with:

```bash
apt-cache madison docker-ce
```

before attempting the downgrade.

---

# 25. Verify the Docker Version

After the downgrade:

```bash
docker --version
```

The expected tested version is:

```text
Docker version 28.5.2
```

The purpose of this change is to test whether the cAdvisor dashboard issue is related to the Docker Engine version.

---

# 26. Start Monitoring Again

Changing the Docker Engine version causes the Docker service and existing containers to stop.

Enter the monitoring directory:

```bash
cd /opt/monitoring
```

Check running containers:

```bash
sudo docker ps
```

The monitoring containers may not be running immediately after the Docker Engine change.

Start them again:

```bash
sudo docker compose up -d
```

Verify:

```bash
sudo docker ps
```

The expected containers are:

```text
monitoring-cadvisor
monitoring-node-exporter
monitoring-prometheus
monitoring-grafana
```

All containers should report a running status.

---

# 27. Re-Test Grafana

Open:

```text
http://SERVER_IP:3000
```

Import the Node Exporter Full dashboard again:

```text
Dashboard ID: 1860
```

Configure Prometheus as the datasource.

Then import the cAdvisor dashboard:

```text
Dashboard ID: 19908
```

After the Docker Engine downgrade, the monitoring dashboards can be tested again to verify whether the previously observed `No Data` condition has been resolved.

---

# 28. Monitoring Architecture

The completed monitoring architecture can be summarized as:

```text
                     HG680P
                       │
                 Armbian Linux
                       │
                 Docker Engine
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
    Node Exporter   cAdvisor    Prometheus
          │            │            │
          │            └──────┬─────┘
          │                   │
          └───────────────────┤
                              ▼
                           Grafana
                              │
                              ▼
                     Monitoring Dashboard
```

Node Exporter provides host-level metrics.

cAdvisor provides Docker container metrics.

Prometheus collects and stores the metrics.

Grafana visualizes the metrics.

---

# 29. Result

The HG680P home server now has a complete monitoring stack based on Docker.

The implemented components are:

```text
Prometheus
Grafana
Node Exporter
cAdvisor
```

The system can monitor:

### Host resources

* CPU
* RAM
* Disk
* Filesystem
* Network
* Load average
* Uptime

### Docker resources

* Container CPU
* Container memory
* Network traffic
* Filesystem usage
* Running containers
* Container restarts

The monitoring stack is deployed using Docker Compose and uses persistent directories for Prometheus and Grafana data.

---

# 30. Troubleshooting Summary

| Problem                                | Verification                            |
| -------------------------------------- | --------------------------------------- |
| Container not running                  | `sudo docker ps`                        |
| Prometheus target down                 | Check Prometheus Targets                |
| cAdvisor issue                         | `sudo docker logs monitoring-cadvisor`  |
| Grafana has no data                    | Check datasource and Prometheus targets |
| Prometheus authentication fails        | Check `web-config.yml`                  |
| Docker version compatibility issue     | `docker --version`                      |
| Containers stopped after Docker change | `sudo docker compose up -d`             |

The most important troubleshooting principle is to verify each layer independently:

```text
Docker
   ↓
Exporter
   ↓
Prometheus
   ↓
Datasource
   ↓
Grafana Dashboard
```

This makes it easier to identify which component is responsible when monitoring data is unavailable.

---

# 31. Security Notes

This repository is public.

Do not commit:

* Real Prometheus passwords
* bcrypt hashes generated from real production passwords
* Grafana administrator passwords
* Private IP information
* API tokens
* SSH credentials
* Other sensitive credentials

Use placeholders such as:

```text
YOUR_PASSWORD
YOUR_BCRYPT_HASH
SERVER_IP
USERNAME
```

The configuration shown in this README should be adapted to the actual environment.

---

# 32. Project Result

The HG680P has been successfully extended from a basic Linux home server into a monitored Docker environment.

The final stack consists of:

```text
Armbian
   │
   └── Docker
        │
        ├── Prometheus
        ├── Grafana
        ├── Node Exporter
        └── cAdvisor
```

Prometheus provides the metrics backend, Node Exporter monitors the Linux host, cAdvisor monitors Docker containers, and Grafana provides the visualization layer.

The monitoring environment allows the server administrator to observe system and container resources through centralized dashboards instead of checking individual metrics manually.

---

# 33. Next Stage

The monitoring stage becomes the foundation for the next automation and deployment stages of the project.

The following projects continue with:

* Bash automation
* Ansible automation
* Docker Compose deployment
* Container image versioning
* Kubernetes K3s
* Kubernetes image versioning
* ArgoCD
* GitOps

---

## Related Project

Previous stage:

**STB HG680P — Armbian & RTL8189FS WiFi**

This stage:

**STB HG680P — Docker, CasaOS & Portainer**

Current stage:

**STB HG680P — Monitoring with Prometheus, Grafana, Node Exporter & cAdvisor**

The complete article provides additional practical explanation and implementation context.

---

## Author

**Muhammad Risyad Rahmadi**

Developer Engineer

Building, automating, and deploying applications with Linux, Docker, Kubernetes, CI/CD, and GitOps.
