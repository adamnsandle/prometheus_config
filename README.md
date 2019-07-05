## Download and install
#### 1. [Prometheus](https://github.com/prometheus/prometheus):
```
$ mkdir -p $GOPATH/src/github.com/prometheus
$ cd $GOPATH/src/github.com/prometheus
$ git clone https://github.com/prometheus/prometheus.git
$ cd prometheus
$ make build
```
#### 2. [Alertmanager](https://github.com/prometheus/alertmanager):
```
$ mkdir -p $GOPATH/src/github.com/prometheus
$ cd $GOPATH/src/github.com/prometheus
$ git clone https://github.com/prometheus/alertmanager.git
$ cd alertmanager
$ make build
```

#### 3. [Node_exporter](https://github.com/prometheus/node_exporter):
```
$ go get github.com/prometheus/node_exporter
$ cd ${GOPATH-$HOME/go}/src/github.com/prometheus/node_exporter
$ make
```

#### 4. [Nvidia_gpu_prometheus_exporter](https://github.com/mindprince/nvidia_gpu_prometheus_exporter):
```
$ go get github.com/mindprince/nvidia_gpu_prometheus_exporter
$ cd ${GOPATH-$HOME/go}/src/github.com/mindprince/nvidia_gpu_prometheus_exporter
$ make
```

## Yaml files

#### 1. prometheus.yml
prometheus [config](https://prometheus.io/docs/prometheus/latest/configuration/configuration/) file, example that i use

```
global:
  # Set the scrape interval to every 15 seconds
  scrape_interval: 15s # Set the scrape interval to every 15 seconds
  # Evaluate rules every 15 seconds
  evaluation_interval: 15s

# Alertmanager configuration
alerting:
  alertmanagers:
  - scheme: http
    static_configs:
    - targets:
    # Alertmanager port
      - "localhost:9093"

# Rules yaml file (additional metrics + alert rules)
rule_files:
  - "alert_rules.yml"

# A scrape configuration
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  # Where to listen to additional metric exporters
  # GPU metrics (Nvidia_gpu_prometheus_exporter)
  - job_name: 'gpu'
    static_configs:
    - targets: ['localhost:9445']

  # Node metrics (Node_exporter)
  - job_name: 'node'
    static_configs:
    - targets: ['localhost:9100']
```

#### 2. alertmanager.yml

alertmanager [config](https://prometheus.io/docs/alerting/configuration/) file, example
```

global:
  smtp_smarthost: smtp.gmail.com:587
  smtp_from: SOME_EMAIL
  smtp_auth_username: USERNAME
  smtp_auth_password: PASSWORD
  smtp_auth_identity: IDENTITY(EMAIL)

route:
  group_by: [Alertname]
  receiver: email-me
  # How long to wait before sending a notification again if it has already
  # been sent successfully for an alert
  repeat_interval: 2h

receivers:
- name: email-me
  email_configs:
  - to: aveysov@gmail.com, dvoronin322@gmail.com

```

#### 3. alert_rules.yml

metric [rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/) for alertmanager (when to fire alarm), example
```
groups:
- name: box_2_stats
  rules:
  - alert: GPU_HIGH_TEMPERATURE
    # Fire when average gpu temperature for past 2 minutes more than 90 celsius
    expr: avg_over_time(nvidia_gpu_temperature_celsius[2m]) > 90
    for: 30s
    annotations:
        description: "{{ $value }} celsius mean GPU temperature for past 2 minutes!"

  - alert: CPU_HIGH_TEMPERATURE
   # Fire when average cpu temperature for past 2 minutes more than 90 celsius
    expr: avg_over_time(node_hwmon_temp_celsius[2m]) > 75
    for: 30s
    annotations:
        description: "{{ $value }} celsius mean CPU temperature for past 2 minutes!"

  - alert: RAM
   # Fire when available RAM < 500 MB
    expr: round((node_memory_MemAvailable_bytes) / 1024 / 1024) < 500
    for: 1m
    annotations:
        description: "Only {{ $value }} MB RAM available!"

  - alert: NVME_MEM
   # Alert when less than 10 GB available on NVME disk
    expr: round(node_filesystem_avail_bytes{mountpoint="/mnt/nvme"} / 1024 / 1024 / 1024) < 10
    for: 5m
    annotations:
        description: "{{ $value }} space available on /mnt/nvme!"

  - alert: MD
   # Fire when disk is not available
    expr: (node_md_disks_active - node_md_disks) > 0
    for: 10m
    annotations:
        description: "Something wrong with disk"

  - alert: IOWAIT
   # Fire when iowait > 0.85 per second for past 5 minutes
    expr: rate(node_cpu_seconds_total{mode="iowait"}[5m]) > 0.85
    for: 5m
    annotations:
        description: "High iowait value! ({{ $value }} mean for past five minutes)"
        ```

## Run everything
these commands will run prometheus on a local 9092 port (using above .yml files)
```
$ ./node_exporter  # default port 9100
# gpu metrics
$ nvidia-docker run -p 9445:9445 -ti mindprince/nvidia_gpu_prometheus_exporter:0.1
$ ./alertmanager --config.file=alertmanager.yml # default port 9093
$ ./prometheus --config.file=./prometheus.yml --web.listen-address="0.0.0.0:9092"

```
