# Jenkinsfile phân tách làm 2 phần riêng biệt, pipeline cho CI và pipeline cho CD
## 1. Jenkinsfile-CI 
### 1.1 Mục tiêu của Pipeline
Pipeline Jenkins được thiết kế triển khai quy trình **Continuous Integration (CI)** cho hệ thống **microservices**, với các mục tiêu:

- Tự động build Docker image cho nhiều service
- Push Docker image lên **Private Docker Registry**
- Thực hiện build **song song** để tối ưu thời gian xử lý

Pipeline hoạt động dựa trên cấu hình **động**, được định nghĩa trong file `services.json`, giúp tách biệt logic CI khỏi cấu hình service.
### 1.2 Jenkins Agent
Pipeline được thực thi trên Jenkins Agent có label lab

Agent có:

Docker để build image từ source code chứa Dockerfile, docker push
Quyền push image lên registry
Source code của project được pull về từ repo Github

### 1.3 Biến môi trường (Environment Variables)
```bash
environment {
    REGISTRY = "hieutv.registry.com"
}
```
| Biến        | Ý nghĩa                                        |
| ----------- | ---------------------------------------------- |
| `REGISTRY`  | Địa chỉ Docker Registry private                |
### 1.4 Get Git Commit Hash
Mục đích:
Lấy git commit hash (short) của source code hiện tại để dùng làm Docker image tag.
```bash
stage('Get Git Commit Hash')
```

Pipeline sử dụng lệnh:
```bash
git rev-parse --short HEAD
```
Giá trị commit hash được gán vào biến: IMAGE_TAG
Ý nghĩa:
Mỗi image gắn chặt với một phiên bản source code
Dễ rollback và truy vết khi deploy hoặc debug
Phù hợp với CI/CD cho microservices


### 1.5 Read Services Config
Mục đích
Đọc danh sách các microservice từ file services.json để pipeline không hard-code service.
```bash
stage('Read Services Config')
```
Sử dụng readJSON để load file cấu hình
Ví dụ file service.json
```bash
[
    {
        "name": "adservice",
        "dockerPath": "src/adservice"
    },
    {
        "name": "paymentservice",
        "dockerPath": "src/paymentservice"
    }
]
```
### 1.5 Build & Push Docker Images
Mục đích
Build Docker image cho từng microservice
Push image lên Registry
Thực thi song song (parallel) để tối ưu thời gian
#### 1.5.1 Tạo các stage động
```bash
def buildStages = [:]
```
buildStages tạo một map dùng để lưu các job build tương ứng với từng service
Mỗi service là 1 nhánh chạy song song vì sử dụng Parallel, số lượng nhánh chạy song song dựa vào cài đặt và tài nguyên của Jenkins Agent
#### 1.5.2 Build & Push cho từng service
```bash
docker build -t ${REGISTRY}/${service.name}:${IMAGE_TAG} ${service.dockerPath}
docker push ${REGISTRY}/${service.name}:${IMAGE_TAG}

```
Với mỗi service:
Build image từ thư mục chứa Dockerfile
Gắn tag theo:
```bash
<registry>/<service-name>:<git-commit-hash>
```
Push image lên registry
#### 1.5.3 Chạy song song (Parallel Execution)
```bash
parallel buildStages
```
Build tất cả microservices cùng lúc dựa trên số lượng luồng build của Agent đã được cấu hình và dựa trên tài nguyên của Jenkins Agent
Giảm thời gian CI khi số lượng service tăng
### 1.6 Post Actions
Khi pipeline thành công
```bash
post {
    success {
        echo " CI pipeline finished successfully for all services!"
    }
}
```
Xác nhận toàn bộ service đã build & push thành công

Khi pipeline thất bại
```bash
failure {
    echo " CI pipeline failed!"
}
```
Khi 1 service lỗi toàn pipeline fail
## 2. Jenkinsfile-CD
### 2.1 Mục tiêu
Triển khai các microservice lên Kubernetes Cluster
Cập nhật phiên bản Docker image cho từng service
Theo dõi trạng thái rollout sau khi deploy
Thực hiện deploy song song để tối ưu thời gian

### 2.2 Môi trường thực thi (Agent)
```bash
agent { label 'lab' }
```

Pipeline được thực thi trên Jenkins Agent lab, agent này có:

kubectl được cài đặt
Quyền truy cập Kubernetes Cluster
File kubeconfig hợp lệ

### 2.3 Cấu hình biến môi trường

```bash
environment {
    REGISTRY   = "hieutv.registry.com"
    IMAGE_TAG  = "3"
    KUBECONFIG = "/home/lab/.kube/config"
}
```
| Biến         | Mô tả                                                |
| ------------ | ---------------------------------------------------- |
| `REGISTRY`   | Địa chỉ Docker Registry(địa chỉ theo domain)                              |
| `IMAGE_TAG`  | Tag của Docker image được deploy                     |
| `KUBECONFIG` | Đường dẫn kubeconfig dùng để xác thực với Kubernetes |


### 2.4 Read Services Config
#### 2.4.1 Mục đích
Stage đọc danh sách các microservice cần deploy từ file services.json, đảm bảo pipeline CD có thể tái sử dụng cấu hình đã định nghĩa trong CI.
#### 2.4.2 Cách hoạt động
```bash
services = readJSON file: 'services.json'
```
Mỗi service trong file cấu hình bao gồm các thuộc tính liên quan đến triển khai Kubernetes
name: tên deployment
k8sYaml: đường dẫn tới manifest Kubernetes (Deployment)
Pipeline ghi log danh sách các service sẽ được triển khai để phục vụ kiểm soát quá trình deploy.

### 2.5 Deploy to Kubernetes
#### 2.5.1 Mục đích

Áp dụng manifest Kubernetes
Cập nhật phiên bản Docker image
Theo dõi trạng thái rollout
Thực hiện deploy song song cho nhiều service

#### 2.5.2 Tạo các nhánh deploy động
```bash
def deployStages = [:]
```
Pipeline khởi tạo một map để lưu các nhánh deploy cho từng microservice
Mỗi service tương ứng với một nhánh deploy độc lập

#### 2.5.3 Áp dụng manifest Kubernetes
```bash
kubectl apply -f <k8sYaml>
```
Manifest đóng vai trò định nghĩa cấu trúc tài nguyên (Deployment, Service, Replica, …)
Không cố định image tag trong file YAML
Giúp tách biệt cấu hình hạ tầng và phiên bản ứng dụng

#### 2.5.4 Cập nhật Docker image
```bash
kubectl set image deployment/<service-name> \
  server=<registry>/<service-name>:<image-tag>
```
Cập nhật image cho container trong Deployment
Kubernetes tự động tạo ReplicaSet mới
Kích hoạt cơ chế rolling update

#### 2.5.5 Theo dõi trạng thái rollout
```bash
kubectl rollout status deployment/<service-name>
```
Pipeline chờ khi rollout hoàn tất
Nếu rollout thất bại, pipeline đánh dấu lỗi
Đảm bảo deploy thành công khi service sẵn sàng hoạt động

#### 2.5.6 Thực thi song song
```bash
parallel deployStages
```
Các microservice được deploy song song: 
Rút ngắn thời gian triển khai
Phù hợp với hệ thống microservices

### 2.6 Xử lý sau pipeline (Post Actions)
#### 2.6.1 Khi pipeline thành công
```bash
success {
    echo "CD SUCCESS – All services deployed"
}
```
Pipeline được đánh dấu thành công khi toàn bộ microservice hoàn tất rollout thành công.

#### 2.6.1 Khi pipeline thất bại
```bash
failure {
    echo "CD FAILED – Check rollout & image pull"
}
```
Pipeline bị đánh dấu thất bại nếu xảy ra khi:
Lỗi rollout
Image pull thất bại
Manifest Kubernetes không hợp lệ

