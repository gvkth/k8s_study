# Bài 12: Cluster Management, Monitoring và Logging

Chúc mừng bạn đã đi đến bài học cuối cùng trong series Kubernetes Cơ bản đến Nâng cao! 

Ở các bài trước, bạn đã biết cách triển khai ứng dụng, mở mạng, và bảo mật. Nhưng khi ứng dụng chạy trên Production, bạn không thể nhắm mắt làm ngơ được. Bạn cần phải biết:
*   **Monitoring (Giám sát):** CPU/RAM của Node có đang quá tải không? Có bao nhiêu request đang bị lỗi 500?
*   **Logging (Ghi nhận lỗi):** Ứng dụng vừa sập, tôi phải tìm log báo lỗi ở đâu khi mà Pod vừa bị xóa mất rồi?
*   **Autoscaling (Tự động mở rộng):** Khi có lượng người truy cập đột biến, làm sao để ứng dụng tự động tăng thêm Pod để chịu tải?

Bài học này sẽ giới thiệu cho bạn 3 trụ cột của vận hành K8s: **Monitoring**, **Logging**, và **Autoscaling**.

---

## 1. Monitoring (Giám sát) với Prometheus & Grafana

**"Cặp bài trùng"** tiêu chuẩn nhất trong thế giới Kubernetes chính là Prometheus và Grafana.

### 1.1. Prometheus (Người thu thập dữ liệu)
*   **Nhiệm vụ:** Prometheus đóng vai trò là một cơ sở dữ liệu chuỗi thời gian (Time-series Database). Nó liên tục **đi kéo (pull/scrape)** các thông số đo lường (metrics) từ các Pod, Node, và bản thân K8s API cứ mỗi 10-15 giây.
*   **Ví dụ metric:** `% CPU Node 1`, `Số lượng HTTP request vào Pod A`, `Số lượng Pod đang chạy`.
*   **PromQL:** Prometheus có một ngôn ngữ truy vấn riêng gọi là PromQL để lọc và tính toán dữ liệu.

### 1.2. Grafana (Người vẽ biểu đồ)
*   Prometheus chứa hàng tỷ con số khô khan. **Grafana** sẽ kết nối vào Prometheus và biến những con số đó thành các biểu đồ (Dashboard) tuyệt đẹp, sinh động.
*   Bạn có thể xem biểu đồ trực quan về sức khoẻ của toàn bộ hệ thống trên một màn hình duy nhất.

> [!TIP]
> **Cách cài đặt nhanh nhất:**
> Cộng đồng K8s cung cấp một Helm Chart khổng lồ tên là `kube-prometheus-stack`. Nó cài đặt sẵn cả Prometheus, Grafana, và hàng chục dashboard dựng sẵn chỉ bằng 1 lệnh duy nhất!
> ```bash
> helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
> helm install my-monitoring prometheus-community/kube-prometheus-stack
> ```

---

## 2. Centralized Logging (Quản lý Log Tập Trung)

Nếu bạn dùng lệnh `kubectl logs <pod-name>`, bạn chỉ xem được log của một Pod đang chạy. Khi Pod chết hoặc bị xóa đi, log đó sẽ biến mất vĩnh viễn. Ở môi trường Production với hàng chục Node và hàng trăm Pod, bạn không thể gõ `kubectl logs` cho từng Pod được.

Bạn cần **Centralized Logging (Log Tập Trung)**. Bộ ba nổi tiếng nhất là **EFK Stack** (Elasticsearch, Fluentd, Kibana).

1.  **Fluentd (hoặc Fluent Bit):**
    *   Được chạy dưới dạng **DaemonSet** (có mặt trên MỌI Node).
    *   Nhiệm vụ: Chặn đường bắt log từ các container trên Node đó, gán thêm thông tin (như tên Pod, tên Namespace) và gửi đi.
2.  **Elasticsearch:**
    *   Kho lưu trữ dữ liệu khổng lồ (Search Engine) chuyên để chứa log.
3.  **Kibana:**
    *   Giao diện Web (tương tự Grafana nhưng dành cho Log). Giúp bạn tìm kiếm log của một Pod cụ thể bằng từ khóa trong vòng vài mili-giây.

> [!NOTE]
> Ngày nay, nhiều công ty cũng chuyển sang dùng **Loki** (của Grafana Labs) vì nó nhẹ hơn Elasticsearch và tích hợp trực tiếp vào Grafana rất mượt mà.

---

## 3. Horizontal Pod Autoscaler (HPA)

Ứng dụng của bạn đang chạy với 2 Replicas. Đột nhiên có một chiến dịch Marketing, lượng người dùng tăng gấp 10 lần. CPU của 2 Pods chạm mức 100%. Nếu không xử lý kịp, ứng dụng sẽ sập!

**HPA (Horizontal Pod Autoscaler)** sinh ra để giải quyết vấn đề này.

### Điều kiện tiên quyết:
Để HPA hoạt động, Cluster của bạn phải được cài đặt **Metrics Server** (một ứng dụng nhỏ gọn giúp K8s biết được mỗi Pod đang ăn bao nhiêu CPU/RAM).

### Cách hoạt động:
1. Bạn khai báo: *"Nếu CPU trung bình của các Pod vượt quá 70%, hãy tạo thêm Pod mới. Số Pod tối thiểu là 2, tối đa là 10"*.
2. HPA sẽ liên tục hỏi Metrics Server xem CPU hiện tại là bao nhiêu.
3. Nếu CPU = 85%, HPA sẽ tự động tăng số Replica lên (ví dụ lên 4 Pods) để chia sẻ tải. Khi tải giảm xuống, HPA sẽ tự động xóa bớt Pod đi (Scale down).

### Ví dụ YAML của HPA:
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: web-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: web-deployment  # Áp dụng cho Deployment nào?
  minReplicas: 2          # Tối thiểu 2 Pods
  maxReplicas: 10         # Tối đa 10 Pods
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # Ngưỡng CPU là 70%
```

---

## 4. Tổng Kết Lộ Trình

Bạn đã đi một chặng đường rất dài! Hãy cùng nhìn lại những gì chúng ta đã chinh phục được:

1.  **Kiến trúc:** Hiểu rõ Master Node, Worker Node, Kubelet.
2.  **Workloads:** Thành thạo Pods, Deployments (chạy ứng dụng stateless), StatefulSets (chạy Database), DaemonSets (chạy Agent).
3.  **Networking:** Biết cách mở port với Services, tạo domain truy cập qua Ingress.
4.  **Configuration:** Quản lý biến môi trường bằng ConfigMaps và Secrets.
5.  **Storage:** Gắn ổ cứng bằng PV, PVC.
6.  **Advanced:** Đóng gói ứng dụng bằng Helm, bảo mật với RBAC, và vận hành hệ thống với Monitoring, Logging, HPA.

Với bộ kỹ năng này, bạn đã hoàn toàn có đủ khả năng để tự mình triển khai, quản lý và vận hành một ứng dụng thực tế trên môi trường Kubernetes.

Chúc bạn thành công trên con đường trở thành một **DevOps / Platform Engineer** thực thụ! Nếu bạn có bất kỳ câu hỏi nào trong quá trình thực hành, đừng ngần ngại quay lại và tìm hiểu thêm.
