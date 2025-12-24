# Grafana on GKE（搭配 Google Managed Prometheus）

## 架構說明

本專案使用 Kubernetes 原生資源在 GKE 上部署 Grafana，並透過 GKE Ingress 對外提供服務，Grafana 的資料來源為 Google Managed Service for Prometheus（GMP）。

整體架構重點如下：

- 使用 Kubernetes Deployment 部署 Grafana
- 使用 PersistentVolumeClaim（PVC）保存 Grafana 設定與 Dashboard
- 透過 Service（ClusterIP）＋ NEG 供 GKE Ingress 使用
- 使用 BackendConfig 自訂 GCLB Health Check 行為
- 透過 GKE Ingress 對外暴露 Grafana（支援憑證）
- Grafana 可串接 Google Cloud Monitoring（GMP）

---

## 專案結構

manifests/
├─ grafana-backendconfig.yaml
├─ grafana-deployment.yaml
├─ grafana-ingress.yaml
├─ grafana-pvc.yaml
├─ grafana-service.yaml
README.md

---

## 操作順序（Deployment 與設定流程）

> 建議實際套用順序：  
> **grafana-pvc → grafana-deployment → grafana-service → grafana-backendconfig → grafana-ingress**

---

### 1. 建立 Grafana Persistent Volume（PVC）

用於保存 Grafana 的設定、資料來源與 Dashboard，確保 Pod 重啟後資料不會遺失。

- manifests/grafana-pvc.yaml

> 備註  
> - PVC 使用 RWO（ReadWriteOnce）  
> - 同一時間只能掛載在一個 Pod 上

---

### 2. 部署 Grafana Deployment

啟動 Grafana Pod，並將 PVC 掛載至 Grafana 的資料目錄。

- manifests/grafana-deployment.yaml

> 注意事項  
> - 若出現 `Multi-Attach error`  
>   代表同一顆 RWO 磁碟仍被舊 Pod 使用中  
>   請先確認舊 Pod 已終止，再讓新 Pod 啟動

---

### 3. 建立 Grafana Service（ClusterIP）

建立 Grafana 對應的 Kubernetes Service，供 Ingress 與 NEG 使用。

- manifests/grafana-service.yaml

Service 設定重點：

- 類型：ClusterIP
- Port：3000
- 啟用 NEG（供 GKE Ingress 使用）
- 綁定 BackendConfig

---

### 4. 設定 BackendConfig（Health Check）

自訂 Google Cloud Load Balancer 對 Grafana 的健康檢查行為。

- manifests/grafana-backendconfig.yaml

設定內容包含：

- Health Check Path：`/api/health`
- Health Check Port：`3000`

> 說明  
> - `/api/health` 是 Grafana 原生健康檢查 API  
> - 不需登入即可回傳 200 狀態  
> - 適合用於 GCLB Health Check

---

### 5. 建立 Grafana Ingress（對外暴露）

透過 GKE Ingress 將 Grafana 對外提供 HTTP / HTTPS 存取。

- manifests/grafana-ingress.yaml

Ingress 功能包含：

- 對外暴露 Grafana 網址
- 綁定 GCP Managed Certificate（如有設定）
- 使用 GKE 原生 L7 Load Balancer

---

## Grafana 與 Google Managed Prometheus（GMP）

### 資料來源說明

Grafana 串接的 Prometheus 資料來源實際上是：

- Google Cloud Monitoring API
- 透過 PromQL 查詢 GMP 指標

Grafana 並非直接連線到 Prometheus Server，而是：

- 由 Grafana 呼叫 Cloud Monitoring API
- 使用 PromQL 相容查詢介面

---

### 常見 PromQL 範例

查詢 Pod CPU 使用量（秒）：

```promql
container_cpu_usage_seconds_total
