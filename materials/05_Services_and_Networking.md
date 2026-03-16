# Bài 5: Services và Mạng trong K8s

Chào mừng bạn đến với Bài 5! Sau khi đã biết cách triển khai ứng dụng bằng [Deployment](04_Deployments.md), bạn sẽ gặp một vấn đề lớn: **Làm sao để người dùng truy cập được vào ứng dụng?**

Như đã thảo luận ở phần FAQ, các Pod có địa chỉ IP nội bộ và chúng có thể bị xoá/thay thế bất cứ lúc nào (khiến IP thay đổi). Chúng ta cần một giải pháp bền vững hơn để "phơi bày" ứng dụng ra ngoài. Đó chính là **Service**.

---

## 1. Service là gì?

**Service** là một đối tượng trong K8s giúp:
1. **Cung cấp một địa chỉ IP tĩnh và duy nhất** (Virtual IP) cho một nhóm các Pod.
2. **Cân bằng tải (Load Balancing):** Tự động phân phối các yêu cầu đến các Pod đang chạy phía sau.
3. **Service Discovery:** Cho phép các ứng dụng tìm thấy nhau thông qua tên (DNS) thay vì IP.

---

## 2. Các loại Service phổ biến

Có 3 loại Service chính mà bạn cần nhớ:

| Loại Service | Đặc điểm | Trường hợp sử dụng |
| :--- | :--- | :--- |
| **ClusterIP** (Mặc định) | Chỉ cấp IP nội bộ bên trong Cluster. | Giao tiếp giữa các service nội bộ (ví dụ: Web gọi Database). |
| **NodePort** | Mở một cổng cố định trên **tất cả các Node**. | Cho phép truy cập từ bên ngoài Cluster qua `NodeIP:NodePort`. |
| **LoadBalancer** | Tích hợp với dịch vụ của Cloud Provider (AWS, GCP). | Cách chính thống để đưa ứng dụng ra Internet trên Cloud. |

---

## 3. Thực hành tạo Service

Chúng ta sẽ tạo một **NodePort** Service để có thể truy cập được Nginx từ trình duyệt Windows của bạn.

### Bước 1: Tạo file `my-service.yaml`
Hãy tạo file này trong thư mục `practice/`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx      # Phải khớp với label 'app: nginx' trong Deployment
  ports:
    - protocol: TCP
      port: 80      # Cổng của Service (nội bộ cluster)
      targetPort: 80 # Cổng mà Pod đang lắng nghe
      nodePort: 30080 # Cổng mở ra trên máy thật (phải nằm trong dải 30000-32767)
```

### Bước 2: Khởi chạy Service
```bash
kubectl apply -f my-service.yaml
```

### Bước 3: Kiểm tra
```bash
kubectl get svc
```
Bạn sẽ thấy `nginx-service` xuất hiện với cổng `80:30080/TCP`.

### Bước 4: Truy cập từ trình duyệt
Bây giờ, hãy mở Chrome/Edge và truy cập: `http://localhost:30080`

**Tại sao lại là localhost?** 
Vì bạn đang dùng Docker Desktop, nó tự động "móc" các cổng NodePort vào localhost của máy Windows. Nếu chạy trên server thật, bạn sẽ dùng `IP_CUA_SERVER:30080`.

---

## 4. Tại sao cần Selector?

Bạn hãy để ý dòng `selector: app: nginx`. 
* Service không quan tâm có bao nhiêu Pod, nó chỉ đi tìm tất cả những Pod nào có dán nhãn (label) `app: nginx`. 
* Nếu bạn scale Deployment lên 10 Pod, Service sẽ tự động nhận biết và chia tải cho cả 10 đứa đó. 
* Nếu một Pod chết và Pod mới sinh ra có cùng label, Service sẽ tự động cập nhật danh sách "đầu mối" (Endpoints) mà không cần bạn can thiệp.

---

## Tổng kết Bài 5

1. **Service** là "bộ mặt" ổn định của ứng dụng, giúp giải quyết vấn đề IP của Pod luôn thay đổi.
2. **NodePort** là cách đơn giản nhất để mở cổng từ cluster ra máy thật trong môi trường lab.
3. **Selector** là sợi dây liên kết linh hồn giữa Service và các Pod phía sau.

> **Thử thách cho bạn:** 
> Hãy thử scale Deployment lên 3 pod (`kubectl scale deployment/nginx-deployment --replicas=3`). Sau đó truy cập `localhost:30080` và nhấn F5 liên tục. Bạn sẽ không thấy khác biệt về giao diện, nhưng thực tế K8s đang âm thầm chuyển hướng bạn tới các Pod khác nhau đấy!

Ở bài sau, chúng ta sẽ học cách quản lý cấu hình linh hoạt mà không cần sửa code qua **Bài 6: ConfigMaps và Secrets**. Sẵn sàng chưa nào? Báo mình nhé!
