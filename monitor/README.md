# Kiến trúc monitor hệ thống
## kube-state-metrics.yaml

Triển khai **kube-state-metrics** trong Kubernetes Cluster, phục vụ hệ thống **monitoring dựa trên Prometheus và Grafana**.

kube-state-metrics là thành phần chuyên thu thập **metrics về trạng thái (state)** của các Kubernetes object thông qua **Kubernetes API Server**.  
Thành phần này **không thu thập resource usage** (CPU, Memory), mà phản ánh **trạng thái logic** của cluster.

---

### Mục đích
- Cung cấp metrics về **trạng thái Kubernetes object** cho Prometheus
- Hỗ trợ xây dựng dashboard giám sát trên Grafana, bao gồm:
  - Pod, Deployment, DaemonSet, StatefulSet
  - Node và Namespace
  - Job, CronJob
  - PVC, PV, ResourceQuota, LimitRange

---

### Namespace
- kube-state-metrics được triển khai trong namespace:
  - `kube-system`
- Phù hợp với vai trò là **thành phần hệ thống của Kubernetes**

---

### Deployment – kube-state-metrics
- Triển khai dưới dạng **Deployment** với:
  - `1 replica`
- Sử dụng image chính thức:
  - `registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.10.1`
- Container expose **HTTP endpoint port `8080`** để cung cấp metrics
- kube-state-metrics chỉ thực hiện:
  - Read-only (list, watch) từ Kubernetes API Server
- Không can thiệp hoặc thay đổi workload đang chạy trong cluster

---

### Service
- Service type: **NodePort**
- Expose:
  - Service port: `8080`
  - NodePort: `30080`
- Mục đích:
  - Cho phép **Prometheus chạy bên ngoài cluster** truy cập metrics

### Endpoint scrape
- Prometheus scrape metrics tại:
  - `http://<NodeIP>:30080/metrics`

---

### ServiceAccount
- Tạo **ServiceAccount riêng cho kube-state-metrics**
- Deployment sử dụng ServiceAccount này để:
  - Xác thực khi truy cập Kubernetes API Server

---

### ClusterRole
- ClusterRole được cấu hình với quyền **read-only (list, watch)** cho các tài nguyên Kubernetes:

#### Core API
- Node
- Pod
- Service
- Namespace
- PersistentVolume (PV)
- PersistentVolumeClaim (PVC)
- ResourceQuota
- LimitRange

#### Apps API
- Deployment
- DaemonSet
- StatefulSet
- ReplicaSet

#### Batch API
- Job
- CronJob

- Tuân thủ nguyên tắc **least privilege**
- Đảm bảo kube-state-metrics **chỉ có quyền đọc trạng thái hệ thống**

---

### ClusterRoleBinding
- Gán ClusterRole `kube-state-metrics` cho:
  - ServiceAccount `kube-state-metrics`
- Phạm vi áp dụng:
  - Toàn bộ cluster

---

### Vai trò trong hệ thống Monitoring
- Prometheus scrape metrics từ kube-state-metrics thông qua HTTP endpoint
- Grafana sử dụng metrics để xây dựng dashboard:
  - Theo dõi trạng thái workload
  - Phân tích phân bổ tài nguyên theo namespace
  - Phát hiện bất thường trong trạng thái triển khai của cluster

## node-exporter.yaml

Triển khai **Node Exporter** để thu thập **metrics hệ điều hành của từng node** trong Kubernetes Cluster, phục vụ hệ thống monitoring với **Prometheus và Grafana**.

Node Exporter cung cấp **node-level metrics**, bổ sung lớp giám sát hạ tầng cho hệ thống monitoring tổng thể.

---

### Mục đích
- Thu thập metrics hệ điều hành của từng node trong cluster
- Cung cấp dữ liệu cho Prometheus phục vụ giám sát:
  - CPU usage
  - Memory usage
  - Disk và filesystem
  - Network I/O
  - Load average
- Bổ sung **lớp giám sát node-level** cho hệ thống monitoring tổng thể

---

### Namespace
- Tạo namespace:
  - `monitoring`
- Dùng để:
  - Cô lập các thành phần monitoring
  - Phân tách rõ ràng giữa workload ứng dụng và hạ tầng giám sát

---

### ServiceAccount
- Tạo **ServiceAccount `node-exporter`** trong namespace `monitoring`
- Được sử dụng cho DaemonSet node-exporter
- Không yêu cầu quyền RBAC đặc biệt vì:
  - Node Exporter **không truy cập Kubernetes API**

---

### DaemonSet – node-exporter
- Node Exporter được triển khai dưới dạng **DaemonSet** nhằm đảm bảo:
  - Mỗi node trong cluster chạy đúng **1 pod node-exporter**
  - Thu thập metrics trực tiếp từ hệ điều hành của node đó

#### Đặc điểm chính
- Image:
  - `quay.io/prometheus/node-exporter:v1.7.0`
- Expose metrics tại port:
  - `9100/TCP`
- Scheduling trên **tất cả các node** trong cluster

#### Tolerations
- DaemonSet được cấu hình tolerations cho:
  - `node-role.kubernetes.io/control-plane`
  - `node-role.kubernetes.io/master`
- Cho phép node-exporter chạy trên cả:
  - Worker node
  - Control-plane / Master node
- Đảm bảo giám sát **đầy đủ toàn bộ cluster**

---

### Host Access & Security Context
Node Exporter cần truy cập trực tiếp vào hệ thống host để thu thập metrics.

- `hostNetwork: true`
  - Pod sử dụng network namespace của host
  - Prometheus có thể scrape trực tiếp qua:
    - `<NodeIP>:9100`
- `hostPID: true`
  - Cho phép truy cập thông tin process của host

---

### Volume Mounts
Các thư mục hệ thống của host được mount vào container ở chế độ **read-only**:

- `/proc` → `/host/proc`
- `/sys` → `/host/sys`
- `/` → `/rootfs`

Các đường dẫn này được truyền vào node-exporter thông qua tham số:
- `--path.procfs`
- `--path.sysfs`
- `--path


## prometheus.yml

File cấu hình **chính của Prometheus**, định nghĩa cách Prometheus thu thập (scrape) metrics từ các hệ thống trong hạ tầng monitoring.

Cấu hình này phục vụ việc giám sát đồng thời:
- Prometheus Server
- Jenkins (CI/CD)
- Jenkins nodes (master và agent)
- Kubernetes Cluster (node-level và state-level metrics)

---

### Global Configuration
- `scrape_interval: 15s`
  - Prometheus thu thập metrics mỗi 15 giây
- `evaluation_interval: 15s`
  - Prometheus đánh giá recording rule và alerting rule mỗi 15 giây
- Cấu hình phù hợp cho môi trường **lab / UAT**
- Cân bằng giữa:
  - Độ chi tiết metrics
  - Chi phí tài nguyên hệ thống

---

### Scrape Configurations

#### Prometheus Self-Monitoring
- Job name: `prometheus`
- Thu thập metrics của **chính Prometheus**
- Phục vụ giám sát:
  - Trạng thái Prometheus server
  - Hiệu năng lưu trữ time-series
  - Số lượng target đang được scrape
- Endpoint scrape:
  - `10.10.92.39:9090`

---

#### Jenkins Metrics
- Job name: `jenkins`
- Thu thập metrics từ Jenkins thông qua **Prometheus plugin**
- Metrics phản ánh:
  - Trạng thái job
  - Executor
  - Pipeline performance
  - Queue length
- Endpoint scrape:
  - `10.10.92.33:8080/prometheus`

---

#### Jenkins Master Node Metrics
- Job name: `jenkins-master-node`
- Thu thập **node-level metrics** từ Node Exporter trên Jenkins master
- Giám sát tài nguyên server Jenkins master
- Endpoint scrape:
  - `10.10.92.33:9100`

---

#### Jenkins Agent Node Metrics
- Job name: `jenkins-agent-node`
- Thu thập **node-level metrics** từ Node Exporter trên Jenkins agent
- Theo dõi hiệu năng các node thực thi job CI/CD
- Endpoint scrape:
  - `10.10.92.32:9100`

---

#### Kubernetes Node Metrics (Node Exporter)
- Job name: `k8s-node-exporter`
- Thu thập **node-level metrics** của các node trong Kubernetes Cluster
- Metrics được cung cấp bởi Node Exporter chạy dưới dạng DaemonSet
- Endpoints scrape:
  - `10.10.92.40:9100`
  - `10.10.92.41:9100`
  - `10.10.92.42:9100`

---

#### Kubernetes State Metrics (kube-state-metrics)
- Job name: `kube-state-metrics`
- Thu thập **state metrics** của Kubernetes Cluster
- Metrics được cung cấp bởi kube-state-metrics service
- Endpoints scrape:
  - `10.10.92.41:30080`
  - `10.10.92.42:30080`
- Metrics phản ánh:
  - Trạng thái Pod, Deployment, StatefulSet, DaemonSet
  - Tình trạng Job, CronJob
  - Phân bổ resource theo namespace

---

### Vai trò trong hệ thống Monitoring
- Prometheus đóng vai trò **trung tâm thu thập và lưu trữ metrics**
- Các job được phân tách rõ ràng theo từng nhóm hệ thống:
  - CI/CD (Jenkins)
  - Infrastructure (Node Exporter)
  - Kubernetes (Node-level và State-level)
- Cấu trúc này giúp:
  - Dễ mở rộng thêm target mới
  - Dễ tích hợp Alertmanager trong tương lai
  - Thuận tiện cho việc xây dựng dashboard trên Grafana
