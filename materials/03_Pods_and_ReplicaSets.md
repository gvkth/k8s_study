# Bài 3: Pods và ReplicaSets

Chào mừng bạn đến với Bài 3! Ở bài này, chúng ta sẽ bắt đầu thực hành đưa một ứng dụng (container) lên chạy thực tế trên Kubernetes. 

Chúng ta sẽ tìm hiểu về **Pod** (đối tượng nhỏ nhất của K8s) và **ReplicaSet** (người bảo vệ sự sống cho Pod).

---

## 1. Mọi thứ trong K8s bắt đầu từ YAML

Khác với Docker khi bạn thường dùng lệnh (`docker run ...`) dể chạy ứng dụng, ở Kubernetes, cách tốt nhất (best practice) là khai báo mọi thứ bằng **file cấu hình YAML**. K8s theo nguyên lý "Khai báo trạng thái mong muốn" (Declarative). 

Bạn viết ra file YAML: *"K8s ơi, tôi muốn có 3 container chạy web Nginx phiên bản 1.25"*. 
Bạn gửi file đó cho K8s bằng lệnh `kubectl apply -f <file.yaml>`. Phần việc còn lại (chạy ở node nào, tự động restart nếu chết) K8s sẽ tự lo.

---

## 2. Pod - Đơn vị nhỏ nhất trong Kubernetes

Nhắc lại định nghĩa ở Bài 1: **Pod là một "cái vỏ bọc" chứa một hoặc nhiều container.** K8s không trực tiếp quản lý container, nó quản lý Pod.

Hãy tạo một thư mục gọi là `practice` trong máy tính của bạn để chứa file thực hành. (Ví dụ: `materials/practice/`)

### 2.1 Tạo Pod Nginx cơ bản
Tạo một file tên là `my-first-pod.yaml` với nội dung sau:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod
  labels:
    app: web-server
spec:
  containers:
  - name: nginx-container
    image: nginx:1.25
    ports:
    - containerPort: 80
```

**Giải thích:**
- `apiVersion`: Phiên bản API của đối tượng (với Pod là `v1`).
- `kind`: Loại đối tượng bạn muốn tạo (ở đây là `Pod`).
- `metadata.name`: Tên định danh duy nhất của Pod.
- `spec`: Định nghĩa chi tiết bên trong Pod:
  - `containers`: Danh sách các container nằm trong Pod này (ở đây chỉ có 1).
  - `image`: Image Docker cần kéo về (kéo từ Docker Hub).
  - `ports`: Cổng mà container này đang lắng nghe (Nginx mặc định là cổng 80).

### 2.2 Đưa Pod lên Cluster chạy thực tế

Mở terminal tại thư mục chứa file yaml trên, gõ lệnh:
```bash
kubectl apply -f my-first-pod.yaml
```

**Kiểm tra xem Pod đã tạo chưa:**
```bash
kubectl get pods
```
Kết quả mong muốn: `STATUS` chuyển từ `ContainerCreating` rồi sang `Running`.

**Xem thông tin chi tiết từng "cen-ti-mét" của Pod:**
```bash
kubectl describe pod my-nginx-pod
```
(Lệnh này cực kỳ hữu ích để debug khi Pod bị lỗi `CrashLoopBackOff` hoặc `Error`).

### 2.3 Thử nghiệm tương tác với Pod
Pod Nginx đang chạy, làm sao để xem trang web từ máy tính (host) của bạn?
Mặc định, Pod chỉ có địa chỉ IP mạng nội bộ của K8s, không thể nối từ ngoài vào. 
Để test nhanh, K8s cho bạn công cụ "*bắc cầu*":
```bash
kubectl port-forward pod/my-nginx-pod 8080:80
```
Bây giờ, hãy mở trình duyệt và gõ `http://localhost:8080`. Chúc mừng! Bạn đang xem trang chủ Nginx đang chạy trong Pod.

Nhấn `Ctrl+C` trong terminal để tắt chế độ bắc cầu.

### 2.4 Xoá Pod rác
```bash
kubectl delete pod my-nginx-pod
```

---

## 3. ReplicaSet - Người bảo vệ bất tử

**Vấn đề:** Nếu Pod Nginx lúc nãy vô tình bị lỗi hệ thống, bị Node lỗi, hoặc bị ai đó lỡ tay `delete`, Pod đó sẽ chết vĩnh viễn!.
Điều này KHÔNG THỂ chấp nhận trong Production. Ứng dụng của bạn phải luôn sống.

**Giải pháp:** K8s cung cấp **ReplicaSet**.
Nhiệm vụ duy nhất của ReplicaSet là: *Đảm bảo luôn luôn có một số lượng Pod nhất định (Replicas) đang chạy trong hệ thống.* Nếu thiếu, tạo thêm. Nếu thừa, xoá bớt.

### 3.1 Tạo ReplicaSet

Tạo file `my-replicaset.yaml`:
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-web-app
  template:
    # Đoạn này y chang metadata và spec của một chiếc Pod bình thường
    metadata:
      labels:
        app: my-web-app
    spec:
      containers:
      - name: nginx-container
        image: nginx:1.25
```

**Giải thích:**
- `replicas: 3`: Tôi muốn luôn có 3 Pod hoạt động.
- `selector.matchLabels`: Làm sao ReplicaSet biết Pod nào là "con" của nó để quản lý? Nó sẽ đi tìm những Pod có dán nhãn (label) `app: my-web-app`. Bất kể ai tạo ra những Pod đó (có label khớp) nó sẽ nhận làm "con nuôi" và quản lý đủ 3 đứa con. Không đủ thì "đẻ" thêm (dựa vào `template`).
- `template`: Nếu phát hiện báo động số lượng Pod bị hụt (ví dụ chỉ còn 2 có dán nhãn khớp), thì lấy cái bản thiết kế `template` này đi đúc ra một cái Pod mới đắp vào cho đủ.

### 3.2 Khởi chạy và kiểm chứng sự bất tử

**Khởi chạy:**
```bash
kubectl apply -f my-replicaset.yaml
```

**Kiểm tra số lượng Pod:**
```bash
kubectl get pods
```
Bạn sẽ thấy 3 Pod có tên dạng `nginx-rs-xxxx` đang chạy.

**Thử thách xoá lén 1 Pod:**
Sao chép tên của 1 Pod bất kỳ ở trên và xoá nó ngay:
```bash
kubectl delete pod <tên-pod-bất-kỳ>
```
Nhanh tay gõ lại lệnh `kubectl get pods`, bạn sẽ thấy ngay phép màu tự chữa lành (Self-healing) của K8s:
- Pod bạn vừa xoá đang ở trạng thái `Terminating` (Đang bị tiêu diệt).
- Và, chớp mắt, ngay lập tức đã có một đại diện mới toanh được ReplicaSet tạo thế chỗ với trạng thái `ContainerCreating` (Đang sinh ra). 

Bạn vĩnh viễn không thể thủ tiêu được hoàn toàn ứng dụng này nếu không ra lệnh cho chính "Ông Trùm" (ReplicaSet).

### 3.3. Dọn dẹp chiến trường
Khi bạn muốn dỡ bỏ ứng dụng, bạn đưa trát xử bắn "Ông Trùm", tự khắc bọn lính con (Pod) sẽ "bay màu" sạch sẽ:
```bash
kubectl delete rs nginx-rs
```

---

## Tổng kết Bài 3
1. Bạn đã biết cấu trúc cơ bản của một file `YAML` trong K8s.
2. Nắm vững đinh lăng thần chưởng 4 lệnh kinh điển nhất: `kubectl apply`, `get`, `describe`, `delete`.
3. Bạn đã hiểu một nhược điểm chí mạng của **Pod**, đó là cực kỳ mỏng manh và dễ chết.
4. Bạn đã thấy Quyền năng bất tử của **ReplicaSet** thông qua cơ chế Labels và Selector.

> **Một bí mật được hé lộ trước Bài 4:**
> Trong hệ thống thực tế (production), người ta **KHÔNG BAO GIỜ** trần truồng quản lý Pod trực tiếp, và **CŨNG RẤT ÍT KHI** dùng trần tụi `ReplicaSet` để lo liệu.
> Cấp độ thao túng tối thượng và thường thấy nhất cho vòng đời ứng dụng trong K8s là **Deployment**. Chúng ta sẽ "bung lụa" với ông Hoàng này ở **Bài 4** nhé!

Nào, trổ tài hacker của bạn thôi! Hãy tạo thư mục `practice/`, gõ (hoặc dán) file yaml và làm nhiệm vụ xoá thử Pod dưới gầm trời giám sát của gã bảo vệ `ReplicaSet` đi. 

Làm xong hoặc bị lỗi gì không chạy được, dán lên đây để mình xem!
