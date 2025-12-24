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

```text
manifests/
├─ grafana-pvc.yaml
├─ grafana-deployment.yaml
├─ grafana-service.yaml
├─ grafana-backendconfig.yaml
├─ grafana-ingress.yaml
README.md
```

---
## 前置條件（Prerequisites）

> 本節為 **必要設定**，若未完成，Grafana 將無法查詢 GMP 指標。

1. 確認 GKE 啟用 Workload Identity
2. 確認 GCP Service Account 已正確綁定 Kubernetes ServiceAccount
3. 發生 403 時，請優先回到本節檢查
4. 此篇章建立cluster sa 不同文件內使用Default sa
### 建立SA並綁定 GCP 服務帳戶（Workload Identity） 
參考文件:
[1]https://docs.cloud.google.com/stackdriver/docs/managed-prometheus/query?hl=zh-tw

[2]https://grafana.com/docs/grafana/latest/setup-grafana/installation/kubernetes/

1. gcloud CLI 設定叢集

```text

gcloud container clusters get-credentials CLUSTER_NAME --location LOCATION --project projectid
```

2. 建立GCP服務帳戶 gmp-sa 以及cluster內sa

```text
1.gcloud iam service-accounts create gmp-sa
2. kubectl create sa gmp-sa
```

3.GCP sa 連結到 ns  Kubernetes 服務帳戶，將必要權限授予 Google Cloud 服務帳戶。

```text
1.gcloud config set project projectid

```

4.使用下列指令，將 gmp-sa 連接至 ns 命名空間中的 Kubernetes 服務帳戶：

```text
1.gcloud config set project projectid

2. gcloud iam service-accounts add-iam-policy-binding   --role roles/iam.workloadIdentityUser   --condition=None   --member "serviceAccount:projectid.svc.id.goog[default/default]"  gmp-sa@projectid.iam.gserviceaccount.com 
 #解說 [default/default] 左邊ns name / cluster SA 名稱

3.kubectl annotate serviceaccount   --namespace default   default   iam.gke.io/gcp-service-account=gmp-sa@projectid.iam.gserviceaccount.com
```

5. 授權給服務帳戶
```text
1gcloud projects add-iam-policy-binding projectid  --member=serviceAccount:gmp-sa@xxxlabid.iam.gserviceaccount.com   --role=roles/monitoring.viewer   --condition=None 

2. gcloud projects add-iam-policy-binding projectid   --member=serviceAccount:gmp-sa@projectid.iam.gserviceaccount.com   --role=roles/iam.serviceAccountTokenCreator   --condition=None

```
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

- 對外暴露 Grafana 網址 訪問LB要帶 /login
- 綁定 GCP Managed Certificate（如有設定）
- 使用 GKE 原生 L7 Load Balancer

---
###  創建grafana與 google cloud monitoring
1. 進到 grafana  左側 connections 選擇google cloud monitoring
<img width="2559" height="1116" alt="image" src="https://github.com/user-attachments/assets/985c4aa8-bb77-4668-b082-3c81e5defb88" />
<img width="2558" height="1147" alt="image" src="https://github.com/user-attachments/assets/676b9050-b07d-4997-be1c-8863cddd9db3" />
<img width="2533" height="1167" alt="image" src="https://github.com/user-attachments/assets/a5b00f51-ca1f-43c0-8077-d5be609e3cd5" />
可以使用
```text
sum by (pod) (
  rate(container_cpu_usage_seconds_total[5m])
)
```

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
