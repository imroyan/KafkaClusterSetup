# Dockerized Kafka Monitoring Stack with Persistent Data

Here's a complete guide to setting up Grafana, Prometheus, and Alertmanager with Docker Compose, ensuring data persistence through volumes.

## Prerequisites

- Docker installed
- Docker Compose installed
- Access to your Kafka brokers

## File Structure

```
kafka-monitoring/
├── docker-compose.yml
├── prometheus/
│   ├── prometheus.yml
│   └── alerts/
│       └── kafka_alerts.yml
├── grafana/
│   └── provisioning/
│       ├── dashboards/
│       │   └── kafka_dashboard.json
│       └── datasources/
│           └── datasource.yml
├── alertmanager/
│   ├── alertmanager.yml
│   └── templates/
│       └── email.tmpl
└── kafka-exporter/
    └── Dockerfile
```

## 1. Docker Compose Setup

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:v2.47.0
    container_name: prometheus
    restart: unless-stopped
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus/alerts/:/etc/prometheus/alerts/
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/usr/share/prometheus/console_libraries'
      - '--web.console.templates=/usr/share/prometheus/consoles'
    depends_on:
      - kafka-exporter

  grafana:
    image: grafana/grafana:10.2.0
    container_name: grafana
    restart: unless-stopped
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning/:/etc/grafana/provisioning/
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
      - GF_USERS_ALLOW_SIGN_UP=false
    ports:
      - "3000:3000"
    depends_on:
      - prometheus

  alertmanager:
    image: prom/alertmanager:v0.26.0
    container_name: alertmanager
    restart: unless-stopped
    volumes:
      - ./alertmanager/alertmanager.yml:/etc/alertmanager/alertmanager.yml
      - ./alertmanager/templates/:/etc/alertmanager/templates/
      - alertmanager_data:/alertmanager
    ports:
      - "9093:9093"
    command:
      - '--config.file=/etc/alertmanager/alertmanager.yml'
      - '--storage.path=/alertmanager'
    depends_on:
      - prometheus

  kafka-exporter:
    build: ./kafka-exporter
    container_name: kafka-exporter
    restart: unless-stopped
    ports:
      - "9308:9308"
    environment:
      - KAFKA_BROKERS=kafka1:9092,kafka2:9092,kafka3:9092
    command:
      - "--kafka.server=kafka1:9092"
      - "--kafka.server=kafka2:9092"
      - "--kafka.server=kafka3:9092"
      - "--web.listen-address=:9308"

volumes:
  prometheus_data:
  grafana_data:
  alertmanager_data:
```

## 2. Prometheus Configuration

Create `prometheus/prometheus.yml`:

```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'kafka-exporter'
    static_configs:
      - targets: ['kafka-exporter:9308']
    metrics_path: /metrics

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['kafka1:9100', 'kafka2:9100', 'kafka3:9100']

  - job_name: 'jmx-exporter'
    static_configs:
      - targets: ['kafka1:7071', 'kafka2:7071', 'kafka3:7071']

rule_files:
  - '/etc/prometheus/alerts/kafka_alerts.yml'

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']
```

Create `prometheus/alerts/kafka_alerts.yml`:

```yaml
groups:
- name: kafka-alerts
  rules:
  - alert: UnderReplicatedPartitions
    expr: sum(kafka_server_ReplicaManager_UnderReplicatedPartitions) by (topic) > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Under-replicated partitions for topic {{ $labels.topic }}"
      description: "Topic {{ $labels.topic }} has {{ $value }} under-replicated partitions"

  - alert: OfflinePartitions
    expr: kafka_controller_KafkaController_OfflinePartitionsCount > 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: "Offline partitions detected"
      description: "There are {{ $value }} offline partitions"

  - alert: HighProduceLatency
    expr: rate(kafka_network_RequestMetrics_TotalTimeMs{request="Produce"}[1m]) / rate(kafka_network_RequestMetrics_RequestsPerSecond{request="Produce"}[1m]) > 100
    for: 10m
    labels:
      severity: warning
    annotations:
      summary: "High produce latency detected"
      description: "Produce latency is {{ $value }}ms"

  - alert: HighDiskUsage
    expr: (node_filesystem_size_bytes{mountpoint="/kafka"} - node_filesystem_free_bytes{mountpoint="/kafka"}) / node_filesystem_size_bytes{mountpoint="/kafka"} > 0.85
    for: 30m
    labels:
      severity: warning
    annotations:
      summary: "High disk usage on {{ $labels.instance }}"
      description: "Disk usage is {{ $value | humanizePercentage }}"
```

## 3. Kafka Exporter Setup

Create `kafka-exporter/Dockerfile`:

```dockerfile
FROM alpine:3.18

RUN wget https://github.com/danielqsj/kafka_exporter/releases/download/v1.7.0/kafka_exporter-1.7.0.linux-amd64.tar.gz \
    && tar -xzf kafka_exporter-1.7.0.linux-amd64.tar.gz \
    && mv kafka_exporter-1.7.0.linux-amd64/kafka_exporter /usr/local/bin/ \
    && rm -rf kafka_exporter-1.7.0.linux-amd64*

EXPOSE 9308
ENTRYPOINT ["kafka_exporter"]
```

## 4. Grafana Provisioning

Create `grafana/provisioning/datasources/datasource.yml`:

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

Download a comprehensive Kafka dashboard (e.g., ID 7589 from Grafana.com) and save as `grafana/provisioning/dashboards/kafka_dashboard.json`.

## 5. Alertmanager Configuration

Create `alertmanager/alertmanager.yml`:

```yaml
route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 5m
  repeat_interval: 3h
  receiver: 'email-notifications'

receivers:
- name: 'email-notifications'
  email_configs:
  - to: 'your-email@example.com'
    from: 'alertmanager@example.com'
    smarthost: 'smtp.example.com:587'
    auth_username: 'alertmanager@example.com'
    auth_password: 'password'
    require_tls: true

templates:
- '/etc/alertmanager/templates/*.tmpl'
```

Create `alertmanager/templates/email.tmpl`:

```tmpl
{{ define "email.default.html" }}
{{- if gt (len .Alerts.Firing) 0 -}}
{{ range .Alerts }}
<h2>Alert: {{ .Labels.alertname }}</h2>
<p><strong>Severity:</strong> {{ .Labels.severity }}</p>
<p><strong>Summary:</strong> {{ .Annotations.summary }}</p>
<p><strong>Description:</strong> {{ .Annotations.description }}</p>
<p><strong>Time:</strong> {{ .StartsAt }}</p>
<hr>
{{ end }}
{{- end -}}
{{- end -}}
```

## 6. Running the Stack

1. Start the services:
```bash
docker-compose up -d
```

2. Verify all containers are running:
```bash
docker-compose ps
```

3. Access the services:
   - Grafana: http://localhost:3000 (admin/admin123)
   - Prometheus: http://localhost:9090
   - Alertmanager: http://localhost:9093

## 7. Persistent Data Verification

1. Check Docker volumes:
```bash
docker volume ls
```

2. Inspect volume data:
```bash
# For Grafana
docker run -it --rm -v kafka-monitoring_grafana_data:/var/lib/grafana busybox ls /var/lib/grafana

# For Prometheus
docker run -it --rm -v kafka-monitoring_prometheus_data:/prometheus busybox ls /prometheus
```

## 8. Additional Configuration

### Adding JMX Exporter to Kafka Brokers

For each Kafka broker, add JMX exporter:

1. Create a Dockerfile for your Kafka image with JMX exporter:
```dockerfile
FROM bitnami/kafka:3.6

# Download JMX exporter
USER root
RUN curl -L https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.19.0/jmx_prometheus_javaagent-0.19.0.jar -o /opt/bitnami/kafka/jmx_prometheus.jar

COPY kafka-jmx-config.yml /opt/bitnami/kafka/conf/

# Add JMX exporter to KAFKA_OPTS
ENV KAFKA_OPTS="$KAFKA_OPTS -javaagent:/opt/bitnami/kafka/jmx_prometheus.jar=7071:/opt/bitnami/kafka/conf/kafka-jmx-config.yml"

USER 1001
```

2. Create `kafka-jmx-config.yml` (same as in the non-Docker setup)

### Node Exporter on Kafka Brokers

For system metrics, run Node Exporter on each Kafka broker host (not in Docker):

```bash
docker run -d \
  --name node-exporter \
  --net="host" \
  --pid="host" \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```

## Maintenance

1. **Backup volumes**:
```bash
docker run --rm -v kafka-monitoring_grafana_data:/source -v $(pwd):/backup alpine tar czf /backup/grafana_backup.tar.gz -C /source .
```

2. **Update containers**:
```bash
docker-compose pull
docker-compose up -d
```

This Dockerized setup provides a persistent, production-ready monitoring solution for your Kafka cluster with all data stored in Docker volumes. The configuration can be easily version-controlled and deployed across environments.
