# Bài 9: StatefulSets và DaemonSets

Chào mừng bạn đến với Bài 9! Đến đây, bạn đã biết cách triển khai ứng dụng (Deployment), tạo mạng (Service), và định tuyến (Ingress). 

Tuy nhiên, hầu hết các ứng dụng chúng ta triển khai từ trước đến giờ đều là **Stateless** (Không trạng thái). Điều này có nghĩa là các Pod "vô danh", có thể bị xoá và thay thế bất cứ lúc nào mà không để lại "di chứng".

Nhưng đời không như mơ! Có những ứng dụng mà **Identity (Danh tính)** và **State (Dữ liệu)** của nó cực kỳ quan trọng. Ví dụ: Cơ sở dữ liệu (Database). Đó là lý do chúng ta cần **StatefulSet** và **DaemonSet**.

---

## 1. StatefulSet: Khi "Tên tuổi" là tất cả

Hãy tưởng tượng bạn có 3 Pod chạy MySQL. Trong **Deployment**, các Pod sẽ có tên ngẫu nhiên như `mysql-abc12`, `mysql-xyz34`. Nếu `mysql-abc12` chết, Pod mới thay thế sẽ có tên hoàn toàn khác.

Trong **StatefulSet**, các Pod sẽ có số thứ tự cố định: `mysql-0`, `mysql-1`, `mysql-2`.

### Đặc điểm của StatefulSet:
*   **Tên cố định (Stable Network ID):** Nếu `mysql-0` chết, Pod hồi sinh vẫn tên là `mysql-0`.
*   **Lưu trữ riêng biệt (Stable Storage):** Mỗi Pod có một Persistent Volume (PV) riêng gắn chặt với số thứ tự của nó. Nếu `mysql-1` bị xoá và tạo lại, nó sẽ tự động nhận lại đúng cái ổ cứng cũ của nó.
*   **Thứ tự triển khai:** Pod `mysql-0` sẽ được tạo trước, chạy xong mới đến `mysql-1`. Khi xoá thì ngược lại (xoá số lớn trước).

### Ví dụ YAML cho StatefulSet:
Để StatefulSet hoạt động, bạn cần một **Headless Service** (một Service có `clusterIP: None`) để định danh từng Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: nginx
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None # Đây là Headless Service
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web
spec:
  serviceName: "nginx-service"
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: k8s.gcr.io/nginx-slim:0.8
        ports:
        - containerPort: 80
          name: web
```

---

## 2. DaemonSet: "Mỗi nhà một vẻ, mỗi Node một Pod"

Thông thường, K8s sẽ tự tính toán xem nên đặt Pod ở Node nào cho tối ưu. Nhưng đôi khi bạn muốn: **"Tôi muốn chạy chính xác 1 Pod này trên MỌI Node của Cluster"**.

Ví dụ:
*   **Logging:** Chạy `fluentd` trên mỗi node để thu gom log từ node đó.
*   **Monitoring:** Chạy `prometheus-node-exporter` để theo dõi sức khoẻ của từng cái máy chủ vật lý.

Đó chính là nhiệm vụ của **DaemonSet**. Khi bạn thêm một Node mới vào Cluster, DaemonSet sẽ tự động "đẻ" thêm một Pod trên Node đó. Khi xoá Node, Pod cũng tự mất đi.

### Ví dụ YAML cho DaemonSet:
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
```

---

## 3. So sánh nhanh: Deployment vs StatefulSet vs DaemonSet

| Đặc điểm | Deployment | StatefulSet | DaemonSet |
| :--- | :--- | :--- | :--- |
| **Tên Pod** | Ngẫu nhiên (`web-xh6zj`) | Theo thứ tự (`web-0`) | Theo Node (`fluentd-node1`) |
| **Ứng dụng** | Web, API (Stateless) | Database, ZooKeeper, Kafka | Logs, Monitoring |
| **Lưu trữ** | Thường dùng chung ổ cứng | Mỗi Pod một ổ cứng riêng | Thường đọc log trực tiếp từ Node |
| **Scaling** | Nhanh, đồng loạt | Chậm, theo thứ tự | Tự động theo số lượng Node |

---

## 4. Thực hành: Trải nghiệm StatefulSet

Hãy thử chạy file StatefulSet ở trên và quan sát:

1.  **Kiểm tra tên Pod:**
    `kubectl get pods` -> Bạn sẽ thấy `web-0`, `web-1`, `web-2`.
2.  **Thử xoá một Pod:**
    `kubectl delete pod web-0`
    Đợi một chút rồi `kubectl get pods` lại -> Một Pod `web-0` TRÙNG TÊN sẽ xuất hiện trở lại.
3.  **Kiểm tra DNS (Nâng cao):**
    Bên trong Cluster, bạn có thể gọi đích danh Pod bằng: `web-0.nginx-service.default.svc.cluster.local`. Điều này cực kỳ hữu ích cho các Database Cluster cần biết chính xác IP của "thằng hàng xóm".

---

## Tổng kết Bài 9

1.  **StatefulSet** dùng cho ứng dụng cần "định danh" và "dữ liệu" bền vững (như Database).
2.  **Headless Service** là "trợ thủ đắc lực" giúp StatefulSet định danh các Pod.
3.  **DaemonSet** dùng để chạy các "agent" hệ thống trên mọi Node của Cluster.

Bạn đã nắm vững cách quản lý các loại Workload khác nhau rồi đấy! Tiếp theo, chúng ta sẽ bước sang một công cụ "thần thánh" giúp đóng gói mọi thứ này thành một gói cài đặt duy nhất: **Bài 10: Helm Charts**.

Báo mình khi bạn đã thử vọc vạch với StatefulSet xong nhé!
