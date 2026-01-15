# Các file Deployment Service của Kubernetes

Tài liệu này mô tả các file deployment dùng để triển khai các microservice trong Kubernetes cluster.

---

## adservice-deployment.yaml

Triển khai **AdService** trong Kubernetes.

### Deployment – AdService
- Chạy 1 pod AdService sử dụng image từ private registry
- Expose gRPC port `9555`
- Cấu hình resource requests/limits cho CPU và RAM
- Bật readinessProbe & livenessProbe theo chuẩn gRPC
- Áp dụng security hardening:
  - Chạy non-root
  - Read-only root filesystem
  - Drop toàn bộ Linux capabilities

### Service (ClusterIP)
- Cung cấp endpoint nội bộ trong cluster
- Cho phép các service khác gọi tới AdService qua gRPC port `9555`

### ServiceAccount
- Tạo ServiceAccount riêng cho AdService
- Sẵn sàng mở rộng RBAC trong tương lai

---

## cartservice-deployment.yaml

Triển khai **CartService** và **Redis backend** phục vụ chức năng giỏ hàng.

### Deployment – CartService
- Chạy CartService xử lý giỏ hàng qua gRPC port `7070`
- Kết nối Redis thông qua biến môi trường:
  - `REDIS_ADDR=redis-cart:6379`
- Cấu hình resource requests/limits cho workload nhẹ
- Bật readinessProbe & livenessProbe theo chuẩn gRPC
- Áp dụng security hardening:
  - Non-root
  - Read-only filesystem
  - Drop capabilities

### Service (ClusterIP) – CartService
- Expose CartService nội bộ trong cluster
- Cho các service khác gọi qua gRPC

### ServiceAccount – CartService
- ServiceAccount riêng cho CartService
- Sẵn sàng mở rộng RBAC nếu cần

### Deployment – Redis Cart
- Triển khai Redis làm datastore cho CartService
- Sử dụng emptyDir làm storage tạm thời cho dữ liệu giỏ hàng
- Health check bằng TCP probe trên port `6379`
- Giới hạn tài nguyên để tránh ảnh hưởng các service khác

### Service (ClusterIP) – Redis Cart
- Cung cấp endpoint Redis nội bộ:
  - `redis-cart:6379`
- Được CartService sử dụng trực tiếp

---

## checkoutservice-deployment.yaml

Triển khai **CheckoutService** – service trung tâm điều phối luồng thanh toán.

### Deployment – CheckoutService
- Chạy CheckoutService qua gRPC port `5050`
- Kết nối tới nhiều microservice thông qua biến môi trường:
  - Product Catalog
  - Shipping
  - Payment
  - Email
  - Currency
  - Cart
- Đóng vai trò orchestrator cho toàn bộ quy trình checkout
- Cấu hình resource requests/limits phù hợp với service điều phối
- Bật readinessProbe & livenessProbe theo chuẩn gRPC
- Áp dụng security hardening:
  - Non-root
  - Read-only filesystem
  - Drop capabilities

### Service (ClusterIP)
- Expose CheckoutService nội bộ
- Cho frontend hoặc service khác gọi tới

### ServiceAccount
- ServiceAccount riêng cho CheckoutService
- Phục vụ phân quyền và mở rộng RBAC khi cần

---

## currencyservice-deployment.yaml

Triển khai **CurrencyService** xử lý chuyển đổi tiền tệ.

### Deployment – CurrencyService
- Chạy CurrencyService qua gRPC port `7000`
- Cung cấp chức năng quy đổi tiền tệ cho các service khác
- Đặc biệt phục vụ CheckoutService
- Tắt profiler để giảm overhead:
  - `DISABLE_PROFILER=1`
- Cấu hình resource requests/limits cho workload nhẹ
- Bật readinessProbe & livenessProbe theo chuẩn gRPC
- Áp dụng security hardening:
  - Non-root
  - Read-only filesystem
  - Drop capabilities

### Service (ClusterIP)
- Expose CurrencyService nội bộ
- Giao tiếp qua gRPC port `7000`

### ServiceAccount
- ServiceAccount riêng cho CurrencyService
- Sẵn sàng mở rộng RBAC

---

## emailservice-deployment.yaml

Triển khai **EmailService** chịu trách nhiệm gửi email thông báo.

### Deployment – EmailService
- Chạy EmailService qua gRPC
  - Container port `8080`
- Gửi email xác nhận và thông báo trong luồng checkout
- Tắt profiler để tối ưu hiệu năng:
  - `DISABLE_PROFILER=1`
- Cấu hình resource requests/limits cho workload nhẹ
- Bật readinessProbe & livenessProbe theo chuẩn gRPC
- Áp dụng security hardening:
  - Non-root
  - Read-only filesystem
  - Drop capabilities

### Service (ClusterIP)
- Expose EmailService nội bộ
- gRPC port `5000` (map tới container port `8080`)

### ServiceAccount
- ServiceAccount riêng cho EmailService
- Phục vụ phân quyền và bảo mật khi cần
## frontend-deployment.yaml

Triển khai **Frontend** – giao diện web chính của hệ thống microservices.

### Deployment – Frontend
- Chạy ứng dụng Frontend trên HTTP port `8080`
- Kết nối tới hầu hết các backend microservice thông qua biến môi trường:
  - Product Catalog
  - Cart
  - Checkout
  - Payment
  - Shipping
  - Currency
  - Ad
  - Recommendation
- Đóng vai trò **entry point** cho người dùng cuối
- Cấu hình **HTTP readinessProbe & livenessProbe**:
  - Endpoint: `/_healthz`
  - Tương thích với **Istio sidecar**
- Cấu hình **resource requests/limits** phù hợp cho frontend workload
- Áp dụng **security hardening**:
  - Chạy non-root
  - Read-only root filesystem
  - Drop toàn bộ Linux capabilities

### Service (ClusterIP)
- Expose Frontend nội bộ trong cluster
- Port `80` map tới container port `8080`
- Cho phép các service nội bộ truy cập Frontend nếu cần

### Service (LoadBalancer)
- Expose Frontend ra bên ngoài cluster
- Cho phép người dùng truy cập hệ thống từ Internet

### ServiceAccount
- Tạo ServiceAccount riêng cho Frontend
- Phục vụ phân quyền và bảo mật khi cần mở rộng RBAC
## paymentservice-deployment.yaml

Triển khai **PaymentService** – service xử lý thanh toán trong hệ thống.

### Deployment – PaymentService
- Chạy PaymentService qua gRPC port `50051`
- Thực hiện logic xử lý và mô phỏng thanh toán trong luồng checkout
- Tắt profiler để giảm overhead runtime:
  - `DISABLE_PROFILER=1`
- Cấu hình resource requests/limits cho workload nhẹ
- Bật readinessProbe & livenessProbe theo chuẩn gRPC
- Áp dụng security hardening:
  - Chạy non-root
  - Read-only root filesystem
  - Drop toàn bộ Linux capabilities

### Service (ClusterIP)
- Expose PaymentService nội bộ trong cluster
- Giao tiếp qua gRPC port `50051`

### ServiceAccount
- Tạo ServiceAccount riêng cho PaymentService
- Phục vụ phân quyền và bảo mật khi cần mở rộng RBAC
## productcatalogservice-deployment.yaml

Triển khai **ProductCatalogService** – service quản lý danh mục sản phẩm trong hệ thống.

### Deployment – ProductCatalogService
- Chạy ProductCatalogService qua gRPC port `3550`
- Cung cấp dữ liệu:
  - Danh sách sản phẩm
  - Thông tin chi tiết sản phẩm
  - Giá sản phẩm
- Phục vụ frontend và CheckoutService
- Tắt profiler để tối ưu hiệu năng:
  - `DISABLE_PROFILER=1`
- Cấu hình resource requests/limits cho workload nhẹ
- Bật readinessProbe & livenessProbe theo chuẩn gRPC
- Áp dụng security hardening:
  - Chạy non-root
  - Read-only root filesystem
  - Drop toàn bộ Linux capabilities

### Service (ClusterIP)
- Expose ProductCatalogService nội bộ trong cluster
- Giao tiếp qua gRPC port `3550`

### ServiceAccount
- Tạo ServiceAccount riêng cho ProductCatalogService
- Phục vụ phân quyền và bảo mật khi cần mở rộng RBAC
## shippingservice-deployment.yaml

Triển khai **ShippingService** – service xử lý thông tin vận chuyển trong hệ thống.

### Deployment – ShippingService
- Chạy ShippingService qua gRPC port `50051`
- Cung cấp chức năng:
  - Tính toán chi phí vận chuyển
  - Quản lý thông tin vận chuyển trong luồng checkout
- Tắt profiler để giảm overhead runtime:
  - `DISABLE_PROFILER=1`
- Cấu hình resource requests/limits cho workload nhẹ
- Bật readinessProbe & livenessProbe theo chuẩn gRPC
- Áp dụng security hardening:
  - Chạy non-root
  - Read-only root filesystem
  - Drop toàn bộ Linux capabilities

### Service (ClusterIP)
- Expose ShippingService nội bộ trong cluster
- Giao tiếp qua gRPC port `50051`

### ServiceAccount
- Tạo ServiceAccount riêng cho ShippingService
- Phục vụ phân quyền và bảo mật khi cần mở rộng RBAC
## kustomization.yaml

Sử dụng **Kustomize** để quản lý và triển khai toàn bộ hệ thống microservices trong Kubernetes.

### Chức năng chính
- Tập hợp **toàn bộ manifest Kubernetes** của hệ thống microservices, bao gồm:
  - AdService
  - CartService (kèm Redis backend)
  - CheckoutService
  - CurrencyService
  - EmailService
  - Frontend
  - PaymentService
  - ProductCatalogService
  - RecommendationService
  - ShippingService

### Triển khai đồng bộ
- Cho phép deploy **toàn bộ stack trong một lần apply**
- Sử dụng lệnh:
  - `kubectl apply -k .`

### Quản lý cấu hình tập trung
- Dễ dàng mở rộng và tùy chỉnh cấu hình:
  - Image tag
  - Namespace
  - Replica count
- Hỗ trợ **patch theo môi trường**:
  - `dev`
  - `uat`
  - `prod`
## loadgenerator-deployment.yaml

Triển khai **LoadGenerator** – service giả lập traffic người dùng để kiểm thử và demo hệ thống.

### Deployment – LoadGenerator
- Tạo lưu lượng truy cập giả lập **liên tục vào Frontend**
- Sử dụng các biến môi trường để điều chỉnh tải:
  - `USERS`: số lượng người dùng giả lập
  - `RATE`: tần suất request
- Phù hợp cho:
  - Demo hệ thống
  - Stress test nhẹ
  - Quan sát metrics, autoscaling và logging

### InitContainer – Frontend Check
- Kiểm tra **Frontend đã sẵn sàng** trước khi LoadGenerator khởi chạy
- Thực hiện HTTP check và chỉ tiếp tục khi nhận **HTTP 200**
- Tránh sinh load khi hệ thống chưa sẵn sàng
- Giúp môi trường lab ổn định hơn

### Tích hợp Istio
- Cấu hình annotation để **tương thích với Istio sidecar**
- Hỗ trợ triển khai trong môi trường sử dụng service mesh

### Security & Resource
- Áp dụng security hardening:
  - Chạy non-root
  - Read-only root filesystem
  - Drop toàn bộ Linux capabilities
- Cấu hình **resource requests/limits cao hơn**
  - Phù hợp với workload sinh tải liên tục

### ServiceAccount
- Tạo ServiceAccount riêng cho LoadGenerator
- Phục vụ phân quyền và bảo mật khi cần mở rộng RBAC
