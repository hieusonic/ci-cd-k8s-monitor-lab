# Jenkinsfile phân tách làm 2 phần riêng biệt, pipeline cho CI và pipeline cho CD
## 1. Jenkinsfile-CI 
### 1.1 Mục tiêu của Pipeline
Pipeline Jenkins được thiết kế triển khai quy trình **Continuous Integration (CI)** cho hệ thống **microservices**, với các mục tiêu chính sau:

- Tự động build Docker image cho nhiều service
- Đẩy Docker image lên **Private Docker Registry**
- Thực hiện build **song song** để tối ưu thời gian xử lý
- Cho phép **mở rộng hệ thống** mà không cần chỉnh sửa Jenkinsfile

Pipeline hoạt động dựa trên cấu hình **động**, được định nghĩa trong file `services.json`, giúp tách biệt logic CI khỏi cấu hình service.
### 1.2 Jenkins Agent
Pipeline được thực thi trên Jenkins Agent có label lab

Agent có:

Docker
Quyền push image lên registry
Source code của project được pull về từ repo Github

### 1.3 Biến môi trường (Environment Variables)
```bash
environment {
    REGISTRY = "hieutv.registry.com"
    BUILD_TAG = "${env.BUILD_NUMBER}"
}
```
| Biến        | Ý nghĩa                                        |
| ----------- | ---------------------------------------------- |
| `REGISTRY`  | Địa chỉ Docker Registry private                |
| `BUILD_TAG` | Tag image, lấy theo `BUILD_NUMBER` của Jenkins |


### 1.4 Read Services Config
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
Push image lên Docker Registry
Thực thi song song (parallel) để tối ưu thời gian
#### 1.5.1 Tạo các stage động
```bash
def buildStages = [:]
```
buildStages là một map dùng để lưu các job build tương ứng với từng service
Mỗi service → 1 nhánh chạy song song vì sử dụng Parallel
#### 1.5.2 Build & Push cho từng service
```bash
docker build -t ${REGISTRY}/${service.name}:${BUILD_TAG} ${service.dockerPath}
docker push ${REGISTRY}/${service.name}:${BUILD_TAG}
```
Với mỗi service:
Build image từ thư mục chứa Dockerfile
Gắn tag theo:
```bash
<registry>/<service-name>:<BUILD_NUMBER>
```
Push image lên registry
#### 1.5.3 Chạy song song (Parallel Execution)
```bash
parallel buildStages
```
Build tất cả microservices cùng lúc dựa trên số lượng luồng build của Agnet
Giảm đáng kể thời gian CI khi số lượng service tăng
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
Chỉ cần 1 service lỗi → toàn pipeline fail
## 2. Jenkinsfile-CD
