# prometheus-configurations
Prometheus Configurations, alert manager, node exporter, blackbox exporter, etc..

# Prometheus

## VM01

### Prometheus Server

target file　:　/opt/prometheus/prometheus-2.3.2.linux-amd64/prometheus.yml
```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
       - 127.0.0.1:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
   - "alert_rules.yml"
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # Prometheus it's myself
  - job_name: 'prometheus'
    static_configs:
    - targets: ['localhost:9090']

  ### resource monitoring for node_exporter

  # VM01
  - job_name: 'VM01'
    static_configs:
    - targets: ['xxx.xxx.xxx.xxx:9100']

  # VM02
  - job_name: 'VM02'
    static_configs:
    - targets: ['xxx.xxx.xxx.xxy:9100']

  # VM03
  # - job_name: 'VM03'
  #   static_configs:
  #   - targets: ['xxx.xxx.xxx.xxz:9100']

  ### http helth check for blackbox_exporter
  - job_name: 'VM01-nginx-http'
    metrics_path: "/probe"
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - http://xxx.xxx.xxx.xxx
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: localhost:9101
```

### Alert Rule

target file　:　/opt/prometheus/prometheus-2.3.2.linux-amd64/alert_rules.yml
```
groups:
- name: Non Kubernetes VM Alert Rules
  rules:

  # variable exporter down alert
  - alert: exporter_process_down
    expr: up == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: '{{ $labels.job }} process is down'

  # squid process down
  - alert: squid_process_down
    expr: node_systemd_unit_state{name="squid.service", state="active"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: '{{ $labels.name }} process is down'

  # nginx process down
  - alert: nginx_process_down
    expr: node_systemd_unit_state{name="nginx.service", state="active"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: '{{ $labels.name }} process is down'

  # td-agent process down
  - alert: td-agent_process_down
    expr: node_systemd_unit_state{name="td-agent.service", state="active"} == 0
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: '{{ $labels.name }} process is down'

  # nginx http ping failed
  - alert: http_ping_failed
    expr: probe_http_status_code != 200
    annotations:
      summary: Instance {{ $labels.instance }} {{ $labels.job }} http ping is failed

  # cpu usage over limit
  #- alert: cpu_usage_exceeded
  #  expr: (100 * (1 - avg by(instance) (irate(node_cpu_second_total{mode="idle"}[5m])))) > 75
  #  annotations:
  #    summary: Instance {{ $labels.instance }} CPU usage is over 75%

  - alert: cpu_usage_exceeded
    expr: 100 * (1 - avg(rate(node_cpu_seconds_total{mode='idle'}[5m])) BY (instance)) > 80
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Instance {{ $labels.instance }} CPU usage is over 80%
      description: 'CPU使用率が80%を超えて5分経過してます'

  # memory usage over limit
  - alert: memory_usage_exceeded
    expr: (node_memory_MemAvailable_bytes or (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / node_memory_MemTotal_bytes < 0.20
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Instance {{ $labels.instance }} memory usage is over 80%
      description: 'メモリ使用量が80%以上になり5分経過してます'

  # disk usage over limit
  - alert: disk_usage_exceeded
    expr: node_filesystem_avail_bytes{device='/dev/sda1'} / node_filesystem_size_bytes * 100 < 20
    for: 5m
    labels:
      severity: critical
    annotations:
      summary: Instance {{ $labels.instance }} filesystem usage is over 80%
      description: 'ディスクスペースが80%以上になり5分経過してます'
```

### Alert Manager

target file　:　/opt/prometheus/alertmanager-0.15.1.linux-amd64/alertmanager.yml
```
global:
  # resolve_timeout: 5m

  # for SendGrid
  smtp_smarthost: 'smtp.sendgrid.net:587'
  smtp_from: 'hoge@xxx.xxx.xxx'
  #smtp_from: 'hoge@xxx.xxx.xxx'
  smtp_auth_username: 'XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX@azure.com'
  smtp_auth_password: 'XXXXXXX'
  smtp_auth_secret: 'XX.XXXXXXXXXXXXXXXXXX.XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX'

route:
  group_by: ['alertname', 'instance', 'severity']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: "mail"

receivers:
- name: 'mail'
  email_configs:
  - to: hoge@xxx.xxx.xxx
    send_resolved: true
  slack_configs:
    - api_url: 'https://hooks.slack.com/services/XXXXXXXXXXXXX/XXXXXXXXX/XXXXXXXXXXXXXXX'
      channel: '#hoge_notify'
      text: "{{ .CommonAnnotations.summary }}"
      send_resolved: true

```

### Blackbox exporter

target file　:　/opt/prometheus/blackbox_exporter-0.12.0.linux-amd64/blackbox.yml
```
modules:
  http_2xx:
    prober: http
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2"]
      method: GET
```

## Systemd Unit Files

### Prometheus Server

target file　:　/etc/systemd/system/prometheus.service
```
[Unit]
Description=Prometheus

[Service]
Type=simple

ExecStart=/opt/prometheus/prometheus-2.3.2.linux-amd64/prometheus --config.file=/opt/prometheus/prometheus-2.3.2.linux-amd64/prometheus.yml --storage.tsdb.path=/opt/prometheus/prometheus-2.3.2.linux-amd64/data

[Install]
WantedBy=multi-user.target
```

### node exporter

target file　:　/etc/systemd/system/node_exporter.service
```
[Unit]
Description=Node Exporter

[Service]
Type=simple

ExecStart=/opt/prometheus/node_exporter-0.16.0.linux-amd64/node_exporter --collector.cpu --collector.meminfo --collector.diskstats --collector.systemd --collector.systemd.unit-whitelist="^(squid.service|nginx.service)$"

[Install]
WantedBy=multi-user.target

```

### alert manager

/etc/systemd/system/alertmanager.service
```
[Unit]
Description=Alertmanager

[Service]
Type=simple

ExecStart=/opt/prometheus/alertmanager-0.15.1.linux-amd64/alertmanager --config.file=/opt/prometheus/alertmanager-0.15.1.linux-amd64/alertmanager.yml

[Install]
WantedBy=multi-user.target
```

target file　:　/etc/systemd/system/blackbox_exporter.service
```
[Unit]
Description=Blackbox Exporter

[Service]
Type=simple

ExecStart=/opt/prometheus/blackbox_exporter-0.12.0.linux-amd64/blackbox_exporter --config.file=/opt/prometheus/blackbox_exporter-0.12.0.linux-amd64/blackbox.yml --web.listen-address=:9101

[Install]
WantedBy=multi-user.target
```

## VM02

### Systemd Unit File

target file　:　/etc/systemd/system/node_exporter.service
```
[Unit]
Description=Node Exporter

[Service]
Type=simple

ExecStart=/opt/prometheus/node_exporter-0.16.0.linux-amd64/node_exporter --collector.cpu --collector.meminfo --collector.diskstats --collector.systemd --collector.systemd.unit-whitelist="^(td-agent.service)$"

[Install]
WantedBy=multi-user.target
```

## VM03

Scope OUt.

