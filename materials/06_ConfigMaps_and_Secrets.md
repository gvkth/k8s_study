# Bài 6: Quản lý Cấu hình và Mật khẩu (ConfigMaps & Secrets)

Chào mừng bạn đến với Bài 6! Trong các bài trước, chúng ta đã chạy ứng dụng Nginx với cấu hình mặc định. Nhưng nếu bạn muốn thay đổi file giao diện `index.html`, hay thay đổi cổng, hoặc truyền mật khẩu kết nối Database vào ứng dụng thì sao?

Có hai cách dở:
1. **Sửa code/file rồi build lại Image mới:** Quá tốn thời gian cho một thay đổi nhỏ.
2. **Ghi đè trực tiếp trong file YAML của Deployment:** Khó quản lý và không bảo mật.

Cách đúng chuẩn K8s: Sử dụng **ConfigMap** và **Secret**.

---

## 1. ConfigMap là gì?

**ConfigMap** là nơi lưu trữ các cấu hình dưới dạng key-value hoặc các file văn bản (như file `.conf`, `.html`, `.json`). Nó giúp tách biệt hoàn toàn cấu hình khỏi mã nguồn ứng dụng (Image).

### Thực hành: Thay đổi trang chủ Nginx bằng ConfigMap

#### Bước 1: Tạo file `my-configmap.yaml`
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  index.html: |
    <html>
      <body style="background-color: #2c3e50; color: #ecf0f1; font-family: sans-serif;">
        <h1>Chào mừng tới Kubernetes!</h1>
        <p>Trang web này được phục vụ từ một <b>ConfigMap</b>.</p>
      </body>
    </html>
```

#### Bước 2: Cập nhật Deployment để sử dụng ConfigMap
Chúng ta sẽ "gắn" (mount) cái file `index.html` từ ConfigMap vào thư mục mặc định của Nginx (`/usr/share/nginx/html`).

Sửa file `my-deployment.yaml` (hoặc tạo mới):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.25
        volumeMounts:
        - name: html-volume
          mountPath: /usr/share/nginx/html # Thư mục Nginx đọc file index
      volumes:
      - name: html-volume
        configMap:
          name: nginx-config # Phải khớp với tên metadata.name của ConfigMap
```

#### Bước 3: Áp dụng
```bash
kubectl apply -f my-configmap.yaml
kubectl apply -f my-deployment.yaml
```
Bây giờ, hãy F5 trình duyệt (`localhost:30080`), bạn sẽ thấy giao diện web của mình đã thay đổi mà không cần build lại image!

---

## 2. Secret là gì?

**Secret** tương tự như ConfigMap nhưng được dùng cho các thông tin nhạy cảm: Mật khẩu, API Token, SSH Key.
* **Khác biệt lớn nhất:** Dữ liệu trong Secret được mã hóa dạng **Base64** để tránh việc người khác vô tình đọc được khi xem file cấu hình. (Lưu ý: Base64 không phải là encryption, nó chỉ là một lớp che mắt cơ bản).

### Thực hành: Truyền mật khẩu qua Biến môi trường (Environment Variables)

#### Bước 1: Tạo Secret bằng lệnh (Nhanh và an toàn hơn)
```bash
kubectl create secret generic my-db-secret --from-literal=password=SuperSecret123
```

#### Bước 2: Sử dụng Secret trong Pod
Chúng ta sẽ truyền password này vào ứng dụng dưới dạng biến môi trường.

```yaml
spec:
  containers:
  - name: my-app
    image: my-app-image
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: my-db-secret
          key: password
```

---

## 3. Khi nào dùng cái nào?

| Đặc điểm | ConfigMap | Secret |
| :--- | :--- | :--- |
| **Dữ liệu** | Cấu hình thông thường, file text. | Mật khẩu, Token, Chứng chỉ. |
| **Bảo mật** | Hiển thị rõ ràng (Plan text). | Mã hóa Base64. |
| **Cách dùng** | Mount thành file hoặc Biến môi trường. | Mount thành file hoặc Biến môi trường. |

---

## Tổng kết Bài 6

1. **ConfigMap/Secret** giúp hiện thực hóa triết lý: *"Build một lần, chạy ở mọi nơi"*. Cùng một Image, bạn có thể chạy ở Dev với config A, và chạy ở Prod với config B chỉ bằng cách thay đổi ConfigMap.
2. Bạn đã biết cách **Mount Volume** từ ConfigMap vào container.
3. Bạn đã biết cách bảo mật thông tin với **Secret**.

> **Thử thách:** 
> Bạn hãy thử sửa nội dung `index.html` trong file `my-configmap.yaml`, gõ `apply` lại. Sau đó chờ khoảng 1 phút, F5 trình duyệt. Bạn sẽ thấy K8s tự động cập nhật file bên trong container mà không cần restart Pod (nếu dùng cơ chế Volume Mount)!

Ở bài tiếp theo, chúng ta sẽ bắt đầu chạm tới phần "khó nhằn" nhưng cực kỳ quan trọng: **Bài 7: Lưu trữ dữ liệu với Persistent Volumes (PV/PVC)**. Chuẩn bị tinh thần để xử lý data cho Database nhé! Báo mình khi bạn đã sẵn sàng.
