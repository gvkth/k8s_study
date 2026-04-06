# Bài 8: Ingress Controllers

Chào mừng bạn đến với Bài 8! Ở [Bài 5](05_Services_and_Networking.md), chúng ta đã dùng **NodePort** để truy cập ứng dụng từ bên ngoài. 

Nhưng hãy tưởng tượng nếu bạn có 100 ứng dụng (microservices). Việc quản lý 100 cái cổng NodePort (như `30080`, `30081`...) sẽ trở thành một "cơn ác mộng":
1. Người dùng phải nhớ từng số cổng.
2. Không hỗ trợ tên miền chuyên nghiệp (như `app1.com`, `app2.com`).
3. Khó quản lý SSL/TLS (HTTPS).

Đây là lúc **Ingress** xuất hiện để giải quyết tất cả.

---

## 1. Ingress là gì?

**Ingress** không phải là một loại Service. Nó là một đối tượng K8s đóng vai trò như một **Smart Proxy** (thường là Nginx, HAProxy...) nằm ở cửa ngõ của Cluster.

Nhiệm vụ của Ingress:
* **Điều hướng Traffic:** Dựa vào đường dẫn (Path) hoặc tên miền (Host) để dẫn luồng vào đúng Service bên trong.
* **SSL Termination:** Quản lý chứng chỉ HTTPS tập trung tại một nơi.
* **Cung cấp một điểm truy cập duy nhất:** Thường là cổng 80/443 tiêu chuẩn.

---

## 2. Ingress Controller và Ingress Resource

Để Ingress hoạt động, bạn cần 2 thứ:
1. **Ingress Controller:** Là phần mềm thực thi việc điều phối (ví dụ: Nginx Ingress Controller). Trong Docker Desktop, bạn phải cài đặt nó trước.
2. **Ingress Resource:** Là file YAML bạn viết để định nghĩa các quy tắc (Rule) điều hướng.

---

## 3. Thực hành: Cài đặt và Cấu hình Ingress

### Bước 1: Kích hoạt Ingress Controller (Docker Desktop)
Mở terminal và chạy lệnh sau (đây là bản dành cho Docker Desktop):
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/cloud/deploy.yaml
```

### Bước 2: Tạo Ingress Resource `my-ingress.yaml`
Giả sử bạn muốn truy cập Nginx qua tên miền ảo `myapp.local`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app-ingress
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: myapp.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: nginx-service # Phải khớp với tên Service đã tạo ở Bài 5
            port:
              number: 80
```

### Bước 3: Đánh lừa máy tính của bạn (File hosts)
Vì `myapp.local` không có thật trên Internet, bạn cần sửa file `hosts` trên Windows để nó trỏ về máy mình.
1. Mở Notepad bằng quyền Administrator.
2. Mở file: `C:\Windows\System32\drivers\etc\hosts`
3. Thêm dòng sau vào cuối:
   `127.0.0.1 myapp.local`

### Bước 4: Áp dụng và Tận hưởng
```bash
kubectl apply -f my-ingress.yaml
```

Bây giờ, hãy mở trình duyệt và gõ: `http://myapp.local`. Bạn sẽ thấy ứng dụng của mình hiện ra mà **không cần gõ số cổng 30080** nữa!

---

## 4. Tại sao Ingress lại "xịn" hơn Service?

Hãy so sánh Ingress với một **Lễ tân toà nhà**:
* **Service (NodePort):** Giống như bạn cho mỗi văn phòng trong toà nhà một số điện thoại nội bộ riêng. Khách phải nhớ từng số.
* **Ingress:** Giống như một bàn Lễ tân ở sảnh. Khách chỉ cần đến nói: "Tôi muốn gặp công ty A", lễ tân sẽ tự dẫn họ vào đúng chỗ. 

Toàn bộ chứng chỉ bảo mật (HTTPS) chỉ cần cài ở bàn Lễ tân (Ingress) này là xong!

---

## Tổng kết Bài 8

1. **Ingress** là giải pháp điều hướng lớp 7 (HTTP/HTTPS) chuyên nghiệp của K8s.
2. Nó cần một **Ingress Controller** để thực thi các quy tắc.
3. Ingress giúp bạn dùng **Tên miền** thay vì Số cổng, và quản lý bảo mật tập trung hơn.

> **Lưu ý:** Việc cài đặt Ingress Controller đôi khi mất vài phút để tải image và khởi động. Bạn hãy kiên nhẫn một chút nhé!

Bạn đã thấy "phê" chưa khi ứng dụng của mình bắt đầu có dáng dấp chuyên nghiệp với tên miền riêng? Hãy báo mình khi bạn đã setup xong để ta sang **Bài 9: StatefulSets và DaemonSets** - nơi chúng ta học cách quản trị những "ông kẹ" khó tính như Database! Báo mình nhé!
