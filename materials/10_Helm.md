# Bài 10: Helm Charts - Trình Quản Lý Gói Cho Kubernetes

Chào mừng bạn đến với Bài 10! Ở các bài học trước, bạn đã làm quen với việc viết hàng tá file YAML cho Pods, Deployments, Services, ConfigMaps, Secrets, Ingress, v.v. 

Nhưng hãy tưởng tượng: Bạn có 3 môi trường: **Dev**, **Staging**, và **Production**. Với mỗi môi trường, bạn phải sửa thủ công số lượng replica, domain Ingress, cấu hình database, hoặc dung lượng ổ cứng trong các file YAML. Việc copy-paste này không chỉ gây mệt mỏi mà còn cực kỳ dễ sai sót.

Đó là lúc **Helm** xuất hiện để giải cứu thế giới! Helm được mệnh danh là **"The Package Manager for Kubernetes"** (tương tự như `npm` cho Node.js, `pip` cho Python hoặc `apt` cho Ubuntu).

---

## 1. Các khái niệm cốt lõi của Helm

Để làm chủ Helm, bạn chỉ cần nắm vững 3 khái niệm cơ bản sau:

1.  **Chart (Gói ứng dụng):** Là một tập hợp các file manifest YAML của Kubernetes được cấu trúc theo một quy chuẩn nhất định, đi kèm với các biến số cấu hình. Bạn có thể coi Chart là một "template" hoặc "bản thiết kế" đóng gói sẵn cho ứng dụng của bạn.
2.  **Repository (Kho lưu trữ):** Là nơi lưu trữ và chia sẻ các Helm Charts. Artifact Hub (https://artifacthub.io) là một thư viện khổng lồ chứa hàng ngàn Helm Charts được cộng đồng đóng gói sẵn (như Nginx, MySQL, Prometheus, Jenkins).
3.  **Release (Bản cài đặt):** Khi bạn cài đặt một Chart vào Kubernetes Cluster, Helm sẽ tạo ra một **Release**. Bạn có thể cài đặt cùng một Chart nhiều lần vào cùng một cluster, mỗi lần cài đặt sẽ tạo ra một Release độc lập với tên gọi riêng (ví dụ: `mysql-dev` và `mysql-prod`).

### Kiến trúc của Helm v3:
Khác với phiên bản Helm v2 (sử dụng một thành phần chạy trong cluster tên là Tiller, dễ gây lỗi bảo mật), **Helm v3** đã hoàn toàn loại bỏ Tiller. Helm v3 chạy trực tiếp dưới quyền của người dùng cấu hình trong file `kubeconfig` và giao tiếp trực tiếp với Kubernetes API.

---

## 2. Cài đặt Helm

Bạn có thể cài đặt Helm rất nhanh chóng tùy theo hệ điều hành:

*   **Trên Windows (sử dụng Winget hoặc Chocolatey):**
    ```powershell
    winget install Helm.Helm
    # hoặc
    choco install kubernetes-helm
    ```
*   **Trên macOS (sử dụng Homebrew):**
    ```bash
    brew install helm
    ```
*   **Trên Linux:**
    ```bash
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    ```

Sau khi cài đặt xong, hãy kiểm tra bằng lệnh:
```bash
helm version
```

---

## 3. Các lệnh Helm cơ bản cần nhớ

Dưới đây là quy trình làm việc điển hình khi sử dụng các Chart có sẵn trên cộng đồng:

### 3.1. Quản lý Repository
Thêm một kho lưu trữ Chart (ví dụ kho Bitnami - đơn vị đóng gói ứng dụng K8s rất nổi tiếng):
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami
```
Cập nhật danh sách Chart mới nhất từ các repo đã thêm:
```bash
helm repo update
```
Tìm kiếm một ứng dụng trong các repo:
```bash
helm search repo nginx
```

### 3.2. Cài đặt và Quản lý Release
Cài đặt một ứng dụng (ví dụ cài đặt Nginx với tên release là `my-web`):
```bash
helm install my-web bitnami/nginx
```
Xem danh sách các ứng dụng đang chạy được quản lý bởi Helm:
```bash
helm list
```
Xem trạng thái chi tiết của một release cụ thể:
```bash
helm status my-web
```

### 3.3. Nâng cấp và Rollback (Quay lui phiên bản)
Nếu bạn muốn thay đổi cấu hình (ví dụ: đổi service từ ClusterIP sang NodePort), bạn có thể dùng tham số `--set` hoặc truyền file cấu hình mới:
```bash
helm upgrade my-web bitnami/nginx --set service.type=NodePort
```
Xem lịch sử các lần nâng cấp của release:
```bash
helm history my-web
```
Nếu bản cập nhật bị lỗi, bạn có thể quay lại phiên bản trước đó (ví dụ quay lại phiên bản revision 1) chỉ bằng một lệnh duy nhất:
```bash
helm rollback my-web 1
```

### 3.4. Gỡ bỏ ứng dụng
Gỡ bỏ hoàn toàn ứng dụng và tất cả các tài nguyên liên quan (Deployment, Service, PVC, v.v.):
```bash
helm uninstall my-web
```

---

## 4. Cấu trúc của một Helm Chart tự viết

Khi bạn muốn đóng gói ứng dụng nội bộ của công ty, bạn sẽ tự tạo ra một Helm Chart. 

Để khởi tạo một Chart mẫu, chạy lệnh:
```bash
helm create my-first-chart
```
Lệnh này sẽ tạo ra một thư mục `my-first-chart/` với cấu trúc như sau:

```text
my-first-chart/
├── Chart.yaml          # Chứa thông tin mô tả về Chart (tên, phiên bản, tác giả...)
├── values.yaml         # File chứa TẤT CẢ các giá trị cấu hình mặc định (biến số)
├── charts/             # Chứa các sub-charts phụ thuộc (nếu có)
└── templates/          # Thư mục chứa các file manifest K8s (dạng template)
    ├── NOTES.txt       # Hướng dẫn hiển thị sau khi cài đặt thành công
    ├── _helpers.tpl    # Định nghĩa các hàm helper, biến dùng chung trong template
    ├── deployment.yaml # Template cho Deployment
    ├── service.yaml    # Template cho Service
    └── ingress.yaml    # Template cho Ingress
```

### Cách hoạt động của Template:
Trong thư mục `templates/`, các file manifest không ghi cứng các giá trị mà sử dụng cú pháp template Go (Go Templates). Ví dụ, file `service.yaml` có thể trông như thế này:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "my-first-chart.fullname" . }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
  selector:
    app: {{ include "my-first-chart.name" . }}
```

Khi chạy lệnh `helm install`, Helm sẽ đọc các giá trị trong file `values.yaml` để điền vào các vị trí có dấu `{{ ... }}`.

Ví dụ trong `values.yaml`:
```yaml
service:
  type: ClusterIP
  port: 80
```
Helm sẽ tự động sinh ra file manifest Kubernetes thực tế với `type: ClusterIP` và `port: 80` để gửi lên cluster.

---

## 5. Thực hành: Tạo và chạy Custom Helm Chart đầu tiên

Hãy cùng tự tay tạo một Chart cực kỳ đơn giản để hiểu cách hoạt động nhé!

### Bước 1: Khởi tạo Chart
Di chuyển vào thư mục học tập của bạn và chạy:
```bash
helm create my-nginx-chart
```

### Bước 2: Tùy biến `values.yaml`
Mở file `my-nginx-chart/values.yaml`. Tìm dòng cấu hình image và chỉnh sửa:
```yaml
replicaCount: 2

image:
  repository: nginx
  pullPolicy: IfNotPresent
  tag: "1.25-alpine" # Sử dụng tag cụ thể thay vì latest
```

Và thay đổi loại Service sang `NodePort` để dễ dàng truy cập từ máy cá nhân:
```yaml
service:
  type: NodePort
  port: 80
```

### Bước 3: Kiểm tra thử template (Dry-run)
Trước khi cài đặt thật vào K8s Cluster, bạn có thể chạy lệnh sau để xem Helm sẽ sinh ra file YAML như thế nào mà không thực sự áp dụng nó:
```bash
helm template my-nginx-chart ./my-nginx-chart
```
Hoặc chạy lệnh giả lập cài đặt (dry-run) để kiểm tra lỗi cú pháp:
```bash
helm install test-release ./my-nginx-chart --dry-run --debug
```

### Bước 4: Cài đặt Chart lên Cluster
Chạy lệnh cài đặt thực tế:
```bash
helm install my-nginx-release ./my-nginx-chart
```

### Bước 5: Kiểm tra kết quả
Xem các tài nguyên được tạo ra:
```bash
kubectl get pods
kubectl get svc
```
Bạn sẽ thấy có 2 Pod Nginx chạy đồng thời và một Service loại `NodePort` được khởi tạo!

### Bước 6: Thay đổi cấu hình trực tiếp (Upgrade)
Hãy thử tăng số lượng bản sao (replicas) lên thành 3 mà không cần sửa file bằng lệnh nâng cấp:
```bash
helm upgrade my-nginx-release ./my-nginx-chart --set replicaCount=3
```
Chạy lại `kubectl get pods` để xem Pod thứ 3 đang được tạo ra.

### Bước 7: Dọn dẹp môi trường
Gỡ cài đặt release để tiết kiệm tài nguyên máy:
```bash
helm uninstall my-nginx-release
```

---

## Tổng kết Bài 10

1.  **Helm** giúp quản lý vòng đời ứng dụng trên K8s dễ dàng hơn thông qua việc đóng gói các file YAML thành **Chart**.
2.  **`values.yaml`** đóng vai trò là file cấu hình duy nhất, tách biệt logic manifest khỏi dữ liệu cấu hình động.
3.  Các lệnh cơ bản: `helm install` (cài), `helm upgrade` (nâng cấp), `helm rollback` (quay lui), `helm uninstall` (xóa).
4.  Cộng đồng có sẵn hàng ngàn ứng dụng chất lượng tại **Artifact Hub**.

Trong bài học tiếp theo, chúng ta sẽ bước sang một chủ đề cực kỳ quan trọng đối với các hệ thống Production thực tế: **Bài 11: Security và RBAC (Role-Based Access Control)**.

Báo mình khi bạn đã sẵn sàng hoặc gặp khó khăn gì trong quá trình chạy thử Helm Chart nhé!
