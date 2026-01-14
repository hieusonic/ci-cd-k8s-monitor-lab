# Triển khai private registry với TLS theo mô hình CA-signed. Nginx dùng server certificate được CA ký, Jenkins và Kubernetes nodes trust CA.
## Tạo chứng chỉ tự ký
### STEP 1 – Tạo Certificate Authority (CA) (Thực hiện 1 lần)

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
ca.crt: Phân phối cho Jenkins, Kubernetes nodes, Docker client để thiết lập trust

ca.key: Private key của CA – bắt buộc giữ kín, không copy, không commit
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
subjectAltName = DNS:hieutv.registry.com,IP:10.10.92.43



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

