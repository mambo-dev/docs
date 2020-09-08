---
date: 2020-07-03
title: KDB+ 인스턴스 모니터링
tags:
  - KDB
  - Prometheus
  - Grafana
---

## 들어가며
[KxSystems/prometheus-kdb-exporter](https://github.com/KxSystems/prometheus-kdb-exporter)는 실행중인 KDB+ 인스턴스에 대한 모니터링을 지원하기 위하여 프로메테우스 매트릭을 제공합니다.

## 🛠 Prometheus Exporter for KDB+
현재 구동중인 KDB+ 인스턴스에서 프로젝트에서 제공하는 `exporter.q`를 실행하여 프로메테우스 매트릭을 `/metrics`를 통해 노출합니다.

### Dynamic Load exporter.q
프로젝트에서 제공하는 `exporter.q`와 `extract.q`를 불러올 수 있도록 파일을 복사합니다. 저는 KDB+ 구동 시 실행되는 `q.q` 파일 하위에 `prometheus-exporter`라는 폴더를 만들었습니다.

![](/images/kdb/prometheus-exporter-dir.png)

q 에서는 `\` 또는 `system` 을 통해 시스템 명령어를 수행할 수 있습니다. 예를 들어, q에서 바라보는 현재 디렉토리가 `q/data`라고 한다면 다음과 같이 디렉토리를 이동하고 파일을 불러올 수 있게 됩니다.

```q kdb+/q
system "cd ../prometheus-exporter"
system "l exporter.q"
system "cd ../data"
```

### KDB+ Prometheus Metrics
exporter.q를 실행하였다면 브라우저를 통해 `/metrics` 경로로 접속하여 Prometheus Metrics를 확인할 수 있습니다.

![](/images/kdb/kdb-prometheus-metrics.png)

#### Prometheus Configuration

```json prometheus.yaml
global:
  scrape_interval: 15s
  scrape_timeout: 10s
  evaluation_interval: 15s
alerting:
  alertmanagers:
  - static_configs:
    - targets: []
    scheme: http
    timeout: 10s
    api_version: v1
scrape_configs:
- job_name: prometheus
  honor_timestamps: true
  scrape_interval: 1m
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - localhost:9090

- job_name: kdb
  honor_timestamps: true
  scrape_interval: 15s
  scrape_timeout: 10s
  static_configs:
  - targets:
    - localhost:5000
```

## 🖥 Grafana Dashboard
프로젝트 예제에서 제공하는 그라파나 대시보드 구성 정보를 통해 프로메테우스로 수집된 매트릭을 시각화합니다.

https://github.com/KxSystems/prometheus-kdb-exporter/blob/master/examples/DockerCompose/grafana-config/dashboards/kdb-dashboard.json

![](/images/kdb/import-grafana-dashboard-for-kdb.png)

이제 그라파나를 통해 KDB+ 인스턴스 상태를 파악할 수 있습니다.

![](/images/kdb/grafana-kdb-dashbaord.png)
