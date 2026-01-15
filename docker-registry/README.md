# Triển khai private registry với TLS theo mô hình CA-signed. Nginx dùng server certificate được CA ký, Jenkins và Kubernetes nodes trust CA.
## Tạo chứng chỉ tự ký
### STEP 1 – Tạo Certificate Authority (CA)

CA (Certificate Authority) đóng vai trò là **root of trust**, dùng để ký các chứng chỉ SSL cho các service nội bộ
trong hệ thống (Docker Registry, Kubernetes, Jenkins, …).

#### 1.1. Tạo private key cho CA

```bash
openssl genrsa -out ca.key 4096
```
#### 1.2. Tạo CA certificate (self-signed)

```bash
openssl req -x509 -new -nodes \
  -key ca.key \
  -sha256 -days 3650 \
  -out ca.crt \
  -subj "/C=VN/ST=HN/O=LabRegistry/CN=LabRegistry-CA"
```

##### File sinh ra
ca.crt: Phân phối cho Jenkins Agent, Kubernetes nodes để thiết lập trust

ca.key: Private key của CA – giữ lại tại server Registry
### STEP 2 – Tạo Server Private Key cho Docker Registry

#### Tạo private key cho Docker Registry
openssl genrsa -out registry.key 2048

File sinh ra
registry.key
Sử dụng cho Nginx (Docker Registry)


### STEP 3 – Tạo file SAN (Subject Alternative Name) (Bắt buộc)


#### Tạo file cấu hình SAN
openssl-san.cnf

#### Nội dung file openssl-san.cnf
```bash
[ req ]
default_bits       = 4096
prompt             = no
default_md         = sha256
req_extensions     = req_ext
distinguished_name = dn

[ dn ]
C  = VN
ST = HN
L  = HN
O  = HieuTV
OU = DevOps
CN = hieutv.registry.com

[ req_ext ]
subjectAltName = @alt_names

[ alt_names ]
DNS.1 = hieutv.registry.com
IP.1  = 10.10.92.43
```



### STEP 4 – Tạo Certificate Signing Request (CSR)

```bash
openssl req -new \
  -key registry.key \
  -out registry.csr \
  -subj "/C=VN/ST=HN/O=LabRegistry/CN=hieutv.registry.com"
```
##### File sinh ra
registry.csr: Certificate Signing Request để CA ký


### STEP 5 – Ký Server Certificate bằng CA nội bộ

```bash
openssl x509 -req \
  -in registry.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out registry.crt \
  -days 365 \
  -sha256 \
  -extfile openssl-san.cnf
```
##### File quan trọng sau khi hoàn thành
###### registry.crt  Cấu hình cho Nginx (Docker Registry)
###### registry.key  Cấu hình cho Nginx
###### ca.crt        Import vào Docker client, Kubernetes nodes, Jenkins


## Cấu hình nginx với chứng chỉ tự ký 
### File config trong /etc/nginx/sites-available
```bash

server {
    listen 443 ssl;
    server_name hieutv.registry.com;


    ssl_certificate     /home/lab/certs-new/registry.crt;
    ssl_certificate_key /home/lab/certs-new/registry.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    client_max_body_size 0;

  
    location / {
        proxy_pass http://10.10.92.43:5000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_read_timeout 900;
    }
}
```
## Cấu hình để Jenkins Agent push được lên Registry, Jenkins Agent (Docker) trust CA Registry
### Tạo thư mục trust certificate cho Docker Registry
```bash
sudo mkdir -p /etc/docker/certs.d/hieutv.registry.com
```
### Copy CA certificate từ Registry vào Docker

### Restart Docker để nhận certificate mới
```bash
sudo systemctl restart docker
```
#### File sử dụng /etc/docker/certs.d/hieutv.registry.com/ca.crt cho phép Docker push image từ Jenkins Agent

## Cấu hình để Kubernetes Cluster có thể pull image từ Private Registry (Kubernetes Worker Node – Trust CA Docker Registry)

### Thực hiện trên từng node chạy pod
#### Tạo thư mục trust certificate cho containerd
```bash
sudo mkdir -p /etc/containerd/certs.d/hieutv.registry.com
```
#### Copy ca.crt CA certificate từ Registry vào containerd vào thư mục /etc/containerd/certs.d/hieutv.registry.com
#### Tạo file cấu hình host cho containerd
File: /etc/containerd/certs.d/hieutv.registry.com/hosts.toml
```bash
server = "https://hieutv.registry.com"

[host."https://hieutv.registry.com"]
  capabilities = ["pull", "resolve"]
  ca = "/etc/containerd/certs.d/hieutv.registry.com/ca.crt"
```
#### Sửa file /etc/containerd/config.toml để containerd sẽ tìm certificates của registry private
Mở file và sửa thành đoạn code sau 
```bash
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"
```
#### Sau đó restart containerd
```bash
sudo systemctl restart containerd
```

