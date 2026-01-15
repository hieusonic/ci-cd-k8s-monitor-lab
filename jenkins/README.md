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


### 1.2 Jenkins Agent
### 1.2 Jenkins Agent

## 2. Jenkinsfile-CD
