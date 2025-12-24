# Grafana on GKE（實驗記錄）

這個 Repository 用來記錄我在 GKE 上部署 Grafana 的過程。

## 架構說明

- 使用 Google Kubernetes Deployment 部署 Grafana
- 透過 Service + Ingress 對外暴露 Grafana
- Grafana 資料來源為 Google Managed Prometheus（GMP）
- 使用 PromQL 查詢 Pod CPU 使用量
- 使用 Workload Identity 讓 Grafana 存取 GCP Monitoring API

## 目前完成項目

- Grafana Deployment
- Ingress 對外存取
- PromQL 查詢 Pod CPU（container_cpu_usage_seconds_total）
- 排查並解決 Grafana 無法存取 GMP 的權限問題

## 備註

此 Repository 為個人學習與實驗用途，目的是讓整個 Grafana + GKE + GMP 架構可以被重現。
