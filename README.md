## ☁️ Backup & Restore Kubernetes với MinIO + Velero

### ⚠️ Yêu cầu đầu tiên: Phải có cụm K8s hoạt động
> Xem cách cài đặt tại: [https://github.com/vuba2002/Create-K8s-Cluster](https://github.com/vuba2002/Create-K8s-Cluster)

## 🖥️ Thông Tin Tài Nguyên Cụm K8s

| Hostname       | OS           | IP              | RAM    | CPU |
|----------------|--------------|------------------|--------|-----|
| master-node-1  | Ubuntu 22.04 | 192.168.154.130 | 3 GB   | 2   |
| worker-node-1  | Ubuntu 22.04 | 192.168.154.131 | 3 GB   | 1   |
| worker-node-2  | Ubuntu 22.04 | 192.168.154.132 | 3 GB   | 1   |
| minio          | Ubuntu 22.04 | 192.168.154.133 | 3 GB   | 1   |

### 🔧 Cấu hình file /etc/hosts trên tất cả các server

```bash
192.168.154.130 master-node-1
192.168.154.131 worker-node-1
192.168.154.132 worker-node-2
192.168.154.133 minio
```

---

## 🐳 Cài đặt MinIO trên Server MinIO

### 1. Cài đặt Docker và Docker Compose

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"${UBUNTU_CODENAME:-$VERSION_CODENAME}\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update -y
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

sudo usermod -aG docker $USER
newgrp docker
```

### 2. Tạo file docker-compose.yml

```yaml
version: '3'
services:
  minio:
    image: minio/minio
    container_name: minio
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - ./storage:/data
    environment:
      MINIO_ROOT_USER: devopseduvn
      MINIO_ROOT_PASSWORD: devopseduvn
    command: server --console-address ":9001" /data
```

Truy cập `http://192.168.154.133:9001` để tạo bucket mới: **backup-k8s**

---

## ☁️ Cài Đặt Velero Trên `master-node-1`

### 1. Tải và cài đặt Velero

```bash
wget https://github.com/vmware-tanzu/velero/releases/download/v1.15.0/velero-v1.15.0-linux-amd64.tar.gz
tar -xvf velero-v1.15.0-linux-amd64.tar.gz
sudo mv velero-v1.15.0-linux-amd64/velero /usr/local/bin
```

### 2. Export các biến môi trường

```bash
export MINIO_URL="http://192.168.154.133:9000"
export MINIO_ACCESS_KEY_ID="admin"
export MINIO_SECRET_KEY_ID="nguyenbavu2002"
export MINIO_BUCKET="backup-k8s"
```

### 3. Tạo file `credentials-velero`

```bash
cat <<EOF > credentials-velero
[default]
aws_access_key_id=$MINIO_ACCESS_KEY_ID
aws_secret_access_key=$MINIO_SECRET_KEY_ID
EOF
```

### 4. Cài đặt Velero với MinIO

```bash
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.5.0 \
  --bucket $MINIO_BUCKET \
  --secret-file ./credentials-velero \
  --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=$MINIO_URL \
  --use-node-agent \
  --namespace velero
```

---

## 🔄 Thực hiện Backup & Restore

### 1. Tạo namespace và triển khai ứng dụng

```bash
kubectl create namespace mediplus
kubectl create -f Mediplus/
```

### 2. Backup namespace

```bash
velero backup create mediplus-v1 --include-namespace=mediplus
```

### 3. Xóa namespace

```bash
kubectl delete namespace mediplus
```

### 4. Khôi phục namespace

```bash
velero restore create mediplus-v1 --from-backup mediplus-v1 --include-namespace=mediplus
```

---
