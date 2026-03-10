# Bài 2: Cài đặt và Thiết lập Môi trường

Chào bạn! Ở Bài 1, chúng ta đã hiểu lý thuyết cơ bản về Kubernetes (K8s) và các thành phần của nó. Hôm nay, ở Bài 2, chúng ta sẽ "xắn tay áo" lên để cài đặt một hệ thống K8s chạy trực tiếp trên máy tính của bạn (hay còn gọi là môi trường local).

Có nhiều cách để chạy K8s:
- **Môi trường Cloud (Production):** AWS EKS, Google GKE, Azure AKS.
- **Môi trường Server vật lý (On-premise):** Kubeadm, Kubespray (cài thủ công lên nhiều máy).
- **Môi trường Local (Học tập & Phát triển):** **Minikube**, **Kind** (Kubernetes in Docker), **Docker Desktop** (có sẵn K8s).

Để phục vụ tốt nhất cho việc học tập, mình đề xuất sử dụng **Minikube** (hoặc kích hoạt K8s trong Docker Desktop nếu bạn đã cài Docker).

---

## 1. Công cụ dòng lệnh: `kubectl`

Trước khi cài đặt Cluster, bạn cần một công cụ để giao tiếp với K8s, đó là `kubectl`. Nhớ lại Bài 1 không? Mọi lệnh bạn gõ sẽ được qua `kubectl`, công cụ này sẽ nói chuyện với "lễ tân" **kube-apiserver**.

### Cách cài đặt `kubectl` trên Windows
1. **Dùng Winget (Trình quản lý gói của Windows):** Mở terminal/PowerShell và gõ:
   ```powershell
   winget install -e --id Kubernetes.kubectl
   ```
2. **Dùng Chocolatey:**
   ```powershell
   choco install kubernetes-cli
   ```
3. **Tải file nhị phân trực tiếp:** 
   Bạn tải file `.exe` từ trang chủ K8s và đưa nó vào thư mục có trong biến môi trường PATH.

*(Tương tự, có các lệnh `brew install kubectl` cho macOS và dùng `apt`/`yum` cho Linux, bạn có thể tham khảo [Tài liệu chính thức](https://kubernetes.io/docs/tasks/tools/)).*

**Kiểm tra cài đặt:**
```bash
kubectl version --client
```
*(Nếu thấy in ra thông tin phiên bản từ client nghĩa là bạn đã cài thành công).*

---

## 2. Cài đặt Local Kubernetes Cluster

Chúng ta cần có 1 Cluster K8s để thực hành. Ở đây mình hướng dẫn 2 cách phổ biến và đơn giản nhất.

### Cách 2.1: Bật Kubernetes trong Docker Desktop (Khuyên dùng nếu đã cài Docker)

1. Mở **Docker Desktop**.
2. Đi tới **Settings** (biểu tượng bánh răng).
3. Chọn thẻ **Kubernetes**.
4. Check vào ô **"Enable Kubernetes"**.
5. Nhấp **Apply & Restart** và chờ một lúc để Docker Desktop tự động tải các container nội bộ của K8s. Tín hiệu nhận biết thành công là biểu tượng K8s nhỏ màu xanh dưới góc trái màn hình UI của Docker Desktop.

### Cách 2.2: Sử dụng Minikube

Minikube là một công cụ giúp tạo một K8s cluster gồm **chỉ 1 Node** chạy ngay trên máy cá nhân. Node này vừa đóng vai trò là Control Plane, vừa là Worker Node.

**Yêu cầu:** Máy bạn phải cài một cái Container Runtime (như Docker) hoặc Hypervisor (như Hyper-V, VirtualBox).

1. Cài đặt Minikube:
   ```powershell
   winget install minikube
   ```
   Hoặc bằng `choco`:
   ```powershell
   choco install minikube
   ```
2. Khởi động Cluster: (Mở cmd hoặc PowerShell với quyền Administrator)
   ```bash
   minikube start 
   # Hoặc nếu bạn muốn chỉ định dùng Docker:
   minikube start --driver=docker
   ```
- Quá trình này sẽ diễn ra: `minikube` tải file ISO ảo -> Tải các thành phần K8s (API Server, Kubelet...) -> Khởi động Cluster.
3. Kiểm tra trạng thái:
   ```bash
   minikube status
   ```

---

## 3. Xác minh Cluster đã hoạt động đúng cách

Sau khi Minikube hoặc Docker Desktop báo cài đặt/start thành công. Chạy lệnh sau để lấy thông tin các thành phần chính của Cluster (ví dụ API server đang chạy ở IP/Port nào):
```bash
kubectl cluster-info
```

Và một lệnh kinh điển để xem ai đang là "công nhân" phục vụ bạn (Nên nhớ K8s local sẽ chỉ có 1 Node):
```bash
kubectl get nodes
```

**Mẫu kết quả:**
```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   10m   v1.28.3
```
- Nếu `STATUS = Ready`, chúc mừng bạn! Kubernetes Cluster của bạn đã sẵn sàng! 

---

## 4. Bàn thêm: Kubeconfig là gì?
Khi bạn chạy `kubectl`, làm sao nó biết Cluster của bạn ở đâu để kết nối tới?
- Nó sẽ tự động đọc cấu hình mặc định nằm trong thư mục người dùng của bạn: `~/.kube/config` (trên Windows là `C:\Users\<Tên_Bạn>\.kube\config`).
- File này chứa thông tin **Cluster, User, và Context**.
- Chạy lệnh sau để xem nội dung file config này:
  ```bash
  kubectl config view
  ```

---

## Tổng kết Bài 2

Chúc mừng bạn đã hoàn thành việc:
1. Cài đặt **`kubectl`** — "chiếc đũa thần" dùng để tương tác với K8s.
2. Cài đặt và kích hoạt thành công một **K8s Local Cluster** qua Minikube (hoặc Docker).
3. Sử dụng `kubectl` để lấy thông tin cluster và kiểm tra danh sách Node đang ở trạng thái `Ready`.

> **Hành động tiếp theo:**
> Hãy đảm bảo rằng trên máy của bạn hiện tại, gõ `kubectl get nodes` ra ít nhất 1 node đang chạy. Vướng mắc gì cứ nói với mình nhé!

Ở bài tới, **Bài 3: Pods và ReplicaSets**, chúng ta sẽ chính thức chạy những ứng dụng đầu tiên lên Cluster! Sẵn sàng chưa? Báo ngay cho mình khi bạn đọc xong và cài đặt thành công nhé!
