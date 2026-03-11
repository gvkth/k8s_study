# Bài 4: Deployments

Chào mừng bạn đến với Bài 4! Ở [Bài 3](03_Pods_and_ReplicaSets.md), chúng ta đã biết về sự mỏng manh của Pod và cách ReplicaSet đóng vai trò "người bảo vệ" giúp duy trì số lượng Pod. 

Tuy nhiên, trong thực tế sản xuất (Production), khi bạn có một phiên bản ứng dụng mới (từ `v1` lên `v2`), ReplicaSet gặp giới hạn lớn: Nó không hỗ trợ quá trình cập nhật phiên bản một cách an toàn và trơn tru.

Đó là lúc **Deployment** bước lên sân khấu! Deployment là cấp độ quản lý cao nhất và phổ biến nhất dành cho ứng dụng không có trạng thái (stateless) trên Kubernetes. 

---

## 1. Deployment là gì?

* **Deployment** ("Triển khai") thực chất là một lớp bọc bổ sung xây dựng bên trên ReplicaSet.
* Nếu ReplicaSet quản lý các **Pod**, thì Deployment quản lý các **ReplicaSet**.
* **Lý do cần Deployment:** Nó cung cấp tính năng **Rolling Update** (Cập nhật cuốn chiếu) và **Rollback** (Quay xe/Hạ cấp) cực kỳ thần thánh mà không cần thời gian ngừng hoạt động (Zero-Downtime).

*(Mô hình cấp bậc: Deployment -> ReplicaSet -> Pods)*

---

## 2. Khởi tạo một Deployment cơ bản

Hãy chuyển vào thư mục `practice/` mà chúng ta đã từng dùng ở Bài 3. Tạo một file tên là `my-deployment.yaml`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
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
      - name: nginx-container
        image: nginx:1.24 # Chúng ta bắt đầu với phiên bản 1.24
        ports:
        - containerPort: 80
```
> **Để ý một chút:** Nội dung YAML của Deployment giống hệt ReplicaSet ở Bài 3 đến 99%. Khác biệt duy nhất nằm ở chỗ `kind: Deployment`.

### Tiến hành khởi chạy:
```bash
kubectl apply -f my-deployment.yaml
```

**Sức mạnh lồng ghép của Kubernetes:**
Mặc dù bạn lệnh tạo Deployment, K8s sẽ âm thầm tạo ra một luồng domino:
1. `kubectl get deployments`: Bạn sẽ thấy 1 Deployment được tạo.
2. `kubectl get rs`: Bạn sẽ thấy 1 ReplicaSet được Deployment đó tự động đẻ ra, tên nó có thêm một dãy số băm đằng sau, ví dụ `nginx-deployment-859cf5789f`.
3. `kubectl get pods`: Bạn sẽ thấy 3 Pod có tên dạng `nginx-deployment-859cf5789f-xxxxx` đang được chạy.

---

## 3. Bản chất của Deployment: Cập nhật ứng dụng!

Giả sử hiện tại ứng dụng Nginx của chúng ta đang chạy ổn định với `image: nginx:1.24`. Đội dev vừa lập trình xong và phát hành phiên bản `1.25`, yêu cầu bạn cập nhật toàn bộ hệ thống.

Nếu chỉ dùng ReplicaSet cũ, bạn phải xóa ứng dụng cũ đi (Downtime) và bật bản mới lên. 

Với Deployment, việc này đơn giản như một cái búng tay:
### Bước 1: Yêu cầu K8s thay đổi phiên bản image sang 1.25
(Bạn có thể sửa file yaml rối chạy `apply` lại, nhưng lệnh sau sẽ nhanh hơn):
```bash
kubectl set image deployment/nginx-deployment nginx-container=nginx:1.25 --record
```
*(Tham số `--record` để Kubernetes ghi lại lịch sử thay đổi để sau này biết đường rollback, dù tham số này sắp deprecated, đây vẫn là một thủ thuật thường gặp).*

### Bước 2: Xem quá trình Rolling Update thần thánh
Gõ nhanh lệnh này liên tục:
```bash
kubectl get pods
```

**Chuyển biến mà bạn sẽ thấy:**
1. Deployment tạo ra một **ReplicaSet thứ MỚI**.
2. Nó từ từ tạo 1 Pod mới (v1.25) trong ReplicaSet mới.
3. Chờ Pod mới Sống (Ready), nó mới Xoá đi 1 Pod cũ (v1.24) ở ReplicaSet cũ.
4. Nó lặp lại quá trình này dần dần cho đến khi ReplicaSet Mới chứa đủ 3 Pod Mới, và ReplicaSet Cũ bị thu hẹp (scale down) về mức 0 Pod.
👉 **Kết quả:** Hệ thống không lúc nào bị ngắt kết nối (Zero-Downtime) vì luôn có một lượng Pod cũ/mới nhất định phục vụ lưu lượng trong quá trình chuyển giao!

### Kiểm tra lịch sử cập nhật:
```bash
kubectl rollout history deployment/nginx-deployment
```

---

## 4. Quay xe (Rollback) thì sao?

Trong tình huống ác mộng nhất: Phiên bản bản 1.25 vừa deploy lên bị lỗi (Crash, lỗi logic, v.v.). Người dùng đang phàn nàn chửi bới. 
Bạn không có thời gian sửa code, bạn phải Cứu hệ thống bằng việc đưa ứng dụng về bản 1.24 lập tức.

Với Deployment, chỉ cần 1 dòng thần chú:

```bash
kubectl rollout undo deployment/nginx-deployment
```

**Phép màu xảy ra:**
- K8s sẽ lục lại "Lịch sử" mà bạn đã record, tìm lại cái ReplicaSet Cũ.
- K8s sẽ bắt đầu scale down (giảm) số Pod của bản 1.25 đang gặp lỗi, và từ từ scale up (Tăng) lại bản 1.24 mà ngày xưa đã từng chạy.
- Hệ thống bạn phục hồi lại như chưa hề có cuộc chia ly!

---

## Tổng kết Bài 4

1. Bạn đã biết rằng **Deployment** là lựa chọn số 1 để chạy các ứng dụng Stateless trong K8s. 
2. Nó xây dựng dựa trên cốt lõi của ReplicaSet để đẻ thêm phép màu: **Rolling Update** và **Rollback**.
3. Bài này đánh dấu sự chuyển mình từ "người vọc vạch" sang một "người vận hành thực thụ" của Kubernetes!

> **Nhiệm vụ cho bạn:** 
> Hãy thử chạy cái `my-deployment.yaml` lên, làm động tác update image lên `1.25` và theo dõi quá trình Rollout nhé! 

Làm xong ngẫm nghĩ chút, K8s xoá Pod liên tục, chớp nháy từ cũ qua mới mà IP của cái hệ thống cứ thay đổi loạn xạ... Vậy thì một ai đó đứng bên ngoài làm sao gọi ứng dụng của ta được? Chúng đành "lạc mất nhau" à?
Đó là lúc chúng ta qua **Bài 5: Services (Mạng nội bộ trong Kubernetes)**. Hãy ra tín hiệu cho mình biết khi bạn thực hành bài này xong!
