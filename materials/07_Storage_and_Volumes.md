# Bài 7: Volumes và Persistent Volumes (PV/PVC)

Chào mừng bạn đến với Bài 7! Ở bài này, chúng ta sẽ giải quyết một vấn đề cực kỳ hóc búa trong Kubernetes: **Dữ liệu sẽ đi đâu khi Pod bị xoá?**

Theo mặc định, mọi thay đổi dữ liệu bên trong container (như file log, ảnh tải lên, database...) sẽ bị **xoá sạch** khi Pod khởi động lại hoặc bị xoá đi. Để lưu trữ dữ liệu bền vững, chúng ta cần cơ chế **Volume**.

---

## 1. Các khái niệm cơ bản về Lưu trữ

Để dễ hiểu, hãy tưởng tượng một hệ thống gồm 3 thành phần:
1. **Physical Storage (Ổ cứng vật lý):** Là ổ SSD ở máy thật, hoặc ổ cứng mạng trên Cloud (AWS EBS, GCP PD...).
2. **PersistentVolume (PV):** Là "đại diện" của ổ cứng đó trong K8s do Quản trị viên (Admin) tạo ra. Nó là một mảnh tài nguyên lưu trữ có sẵn.
3. **PersistentVolumeClaim (PVC):** Là một "phiếu yêu cầu" do Người dùng (Developer) viết ra. K8s sẽ đi tìm một PV phù hợp với yêu cầu trong phiếu (đủ dung lượng, đúng loại) để "gắn" (bind) chúng lại với nhau.

---

## 2. Các loại Volume phổ biến

Trước khi đi sâu vào PV/PVC phức tạp, hãy làm quen với 2 loại Volume đơn giản nhất:

| Loại Volume | Đặc điểm | Trường hợp sử dụng |
| :--- | :--- | :--- |
| **emptyDir** | Tạo ra một thư mục trống khi Pod sinh ra, xoá sạch khi Pod chết. | Dùng làm bộ nhớ tạm (Cache), thư mục xử lý file trung gian cho nhiều container chung Pod. |
| **hostPath** | Gắn (mount) một thư mục từ **máy chủ vật lý (Worker Node)** vào thẳng bên trong Container. | Dùng khi app cần đọc log của hệ điều hành, hoặc khi chạy K8s local (Minikube/Docker Desktop) để vọc vạch. |

---

## 3. Thực hành: Lưu trữ dữ liệu lâu dài với PVC

Trong môi trường Docker Desktop (của bạn), K8s đã hỗ trợ sẵn tinh năng **Dynamic Provisioning** (Tự động cấp ổ đĩa). Bạn chỉ cần làm "phiếu yêu cầu" (PVC), K8s sẽ tự động tạo "ổ đĩa" (PV) cho bạn.

### Bước 1: Tạo file yêu cầu ổ cứng `my-pvc.yaml`
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-storage-pvc
spec:
  accessModes:
    - ReadWriteOnce # Chỉ 1 Node có quyền đọc/ghi một lúc
  resources:
    requests:
      storage: 1Gi    # Tôi yêu cầu 1GB dung lượng
```

### Bước 2: Gắn PVC vào Deployment
Chúng ta sẽ "mượn" ổ cứng 1GB này để lưu trữ dữ liệu cho Nginx.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-storage-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-with-storage
  template:
    metadata:
      labels:
        app: nginx-with-storage
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.25
        volumeMounts:
        - name: storage-volume
          mountPath: /data # Gắn ổ cứng vào thư mục /data trong container
      volumes:
      - name: storage-volume
        persistentVolumeClaim:
          claimName: my-storage-pvc
```

### Bước 3: Áp dụng và Kiểm chứng

```bash
kubectl apply -f my-pvc.yaml
kubectl apply -f my-deployment-with-storage.yaml
```

**Thử thách "Sống sót":**
1. Bạn hãy vào trong Pod tạo một file nào đó trong thư mục `/data`:
   `kubectl exec -it <tên-pod> -- touch /data/hello-k8s.txt`
2. Sau đó, hãy xoá Pod đó đi bằng lệnh `kubectl delete pod <tên-pod>`.
3. Chờ Deployment tự động đẻ ra một Pod mới.
4. Kiểm tra lại thư mục `/data` của Pod mới đó: 
   `kubectl exec -it <tên-pod-mới> -- ls /data`
👉 **Kết quả:** File `hello-k8s.txt` vẫn còn đó! Ổ cứng (PVC) đã bám theo Pod mới một cách thần kỳ.

---

## 4. Tại sao cần tách PV và PVC?

Đây là một triết lý cực hay của K8s giúp ngăn cách hai thế giới:
* **Admin (Quản trị hạ tầng):** Họ chỉ quan tâm việc mua ổ cứng ở đâu (SSD hay HDD), cấu hình RAID như thế nào... và khai báo chúng qua **PV/StorageClass**.
* **Dev (Lập trình viên):** Họ không cần biết ổ đĩa đó là hãng nào, nằm ở đâu. Họ chỉ cần viết **PVC** nói rằng: "Tôi cần 10GB tốc độ cao". K8s là người môi giới tuyệt vời ở giữa để ghép đôi hai bên.

---

## Tổng kết Bài 7

1. Dữ liệu trong container là **tạm thời** - hãy dùng **Volume** để lưu trữ bền vững.
2. **PersistentVolume (PV)** là tài nguyên ổ cứng có thật.
3. **PersistentVolumeClaim (PVC)** là lời yêu cầu sử dụng tài nguyên của bạn.
4. Cơ chế này giúp dữ liệu của bạn "sống sót" qua mọi biến cố của Pod.

> **Gợi ý:** Để chạy các Database như MySQL, MongoDB... việc nắm vững PVC là điều bắt buộc.

Bài này hơi trừu tượng một chút về mặt hạ tầng, bạn có câu hỏi nào muốn đưa vào **FAQ** không? Nếu bạn đã thực hành xong bài tập "sống sót" ở trên, hãy báo mình để chúng ta sang **Bài 8: Ingress Controllers** - cách đưa ứng dụng của bạn ra thế giới Internet bằng tên miền chuyên nghiệp!
