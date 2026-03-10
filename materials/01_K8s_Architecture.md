# Bài 1: Tổng quan và Kiến trúc Kubernetes

Chào mừng bạn đến với bài học đầu tiên trong lộ trình chinh phục Kubernetes (K8s). Bài học này sẽ giúp bạn hiểu rõ K8s là gì, tại sao nó lại trở thành tiêu chuẩn trong việc quản lý container, và kiến trúc bên dưới của nó hoạt động như thế nào.

---

## 1. Kubernetes là gì? Tại sao cần dùng K8s?

### 1.1 Khái niệm cơ bản
- **Kubernetes** (hay còn gọi là **K8s** – vì có 8 chữ cái giữa K và s) là một nền tảng mã nguồn mở do Google thiết kế ban đầu, hiện được quản lý bởi Cloud Native Computing Foundation (CNCF).
- Nó giúp tự động hóa việc triển khai (deployment), scale (mở rộng/thu hẹp) và quản lý các ứng dụng được container hóa.

### 1.2 Tại sao cần K8s?
Hãy tưởng tượng bạn có một ứng dụng web, bạn đóng gói nó vào Docker container và chạy bằng `docker run`. Mọi thứ rât trơn tru. Nhưng khi ứng dụng của bạn lớn lên, bạn cần:
- Chạy 10, 50, hay 100 containers trên nhiều máy chủ (nodes) khác nhau.
- Làm sao biết container nào bị chết để khởi động lại?
- Làm sao phân tải (load balance) lượt truy cập vào các containers này?
- Nếu lượng truy cập tăng vọt, làm sao bật thêm container tự động?

**Kubernetes giải quyết tất cả các vấn đề trên:**
- **Service discovery & Load balancing:** K8s có thể cung cấp IP nội bộ và một tên miền DNS duy nhất cho một tập hợp các container, và phân tải traffic giữa chúng.
- **Storage orchestration:** Tự động mount hệ thống lưu trữ bạn chọn (local storage, cloud provider storage như AWS EBS, GCP PD, v.v.).
- **Automated rollouts and rollbacks:** Bạn có thể khai báo trạng thái mong muốn của ứng dụng, K8s sẽ dần dần thay đổi trạng thái hiện tại sang trạng thái mới. Nếu có lỗi, bạn có thể rollback dễ dàng.
- **Self-healing:** Tự động restart các container bị lỗi, thay thế container, xóa container không phản hồi kiểm tra sức khỏe (health checks).
- **Secret and configuration management:** Lưu trữ và quản lý thông tin nhạy cảm (như password, OAuth tokens, SSH keys) mà không cần build lại container image.

---

## 2. Kiến trúc của Kubernetes

Một hệ thống Kubernetes đang chạy được gọi là một **Cluster**. 
Một Cluster sẽ có hai thành phần chính:
1. **Control Plane (Node Quản lý)**: Đưa ra các quyết định chung về cluster (ví dụ: lên lịch chạy ứng dụng ở đâu), phát hiện và phản hồi với các sự kiện trong cluster.
2. **Worker Nodes (Node Máy trạm)**: Máy tính (có thể là máy vật lý hoặc máy ảo) chạy các ứng dụng (dưới dạng Pods).

### 2.1 Thành phần của Control Plane
Control Plane thường chạy trên một hoặc nhiều máy chủ riêng biệt (trong môi trường production) để đảm bảo độ tin cậy.

- **kube-apiserver**: Là trung tâm giao tiếp của K8s. Mọi lệnh bạn gõ (như `kubectl get pods`), hay nội bộ K8s giao tiếp với nhau đều phải đi qua API Server. Nó giống như "lễ tân" của K8s.
- **etcd**: Là kho lưu trữ dữ liệu dạng key-value, nhất quán và có tính sẵn sàng cao. K8s dùng etcd để lưu toàn bộ dữ liệu cấu hình, trạng thái của cluster.
- **kube-scheduler**: Thành phần này theo dõi các Pod mới được tạo mà chưa được phân bổ vào Node nào, sau đó chọn một Node phù hợp để Pod đó chạy (dựa trên tài nguyên yêu cầu, chính sách...).
- **kube-controller-manager**: Chạy các tiến trình điều khiển (controllers). Nó theo dõi trạng thái cluster qua API Server và cố gắng đưa trạng thái hiện tại về trạng thái mong muốn. (Ví dụ: Node Controller theo dõi khi một node bị chết, ReplicaSet Controller đảm bảo số lượng Pod mong muốn luôn chạy).

### 2.2 Thành phần của Worker Nodes
Worker node là nơi thực sự làm việc, chứa các container của bạn.

- **kubelet**: Một agent (tiến trình) chạy trên mỗi node. Nó đảm bảo các container đang thực sự chạy trong một Pod. Kubelet nhận các mô tả Pod (từ API Server) và đảm bảo các container được mô tả đó hoạt động ổn định.
- **kube-proxy**: Thành phần mạng chạy trên mỗi node. Kube-proxy duy trì các luật về mạng (network rules) trên các node, cho phép giao tiếp mạng tới các Pods từ bên trong hoặc bên ngoài cluster.
- **Container Runtime**: Phần mềm thực sự chịu trách nhiệm chạy các container. K8s hỗ trợ nhiều runtime như `containerd`, `CRI-O`, và bất kỳ chương trình nào tuân theo chuẩn Kubernetes CRI (Container Runtime Interface). (Lưu ý: Docker Engine từng là một runtime, nhưng hiện tại K8s dùng `containerd` hoặc `CRI-O` là chủ yếu).

---

## 3. Đối tượng tối thượng: Pod

- Chắc hẳn bạn thắc mắc: "Tại sao nãy giờ nhắc K8s quản lý container, máy thì chạy container runtime, mà K8s lại toàn nói về quản lý Pod?".
- **Pod** là đơn vị triển khai nhỏ nhất, cơ bản nhất trong Kubernetes.
- Thay vì quản lý trực tiếp một container độc lập, K8s tạo ra và quản lý một **Pod**.
- Một Pod chứa **một hoặc nhiều container** (thường là 1) chia sẻ chung:
  - Cùng một địa chỉ IP nội bộ (IP namespace).
  - Cùng không gian lưu trữ (Volume namespace).
  - Cùng network namespace (do đó các container trong cùng 1 Pod có thể gọi nhau qua `localhost`).

Hãy nghĩ Pod như một "cái kén" (hoặc "vỏ bọc") cho container của bạn. K8s giao tiếp với Pod, và dời Pod đi nơi khác nếu Node hiện tại chết, kéo theo container(s) bên trong múa theo.

---

## Tổng kết Bài 1

- Bạn đã biết Kubernetes là "nhạc trưởng" điều phối các container.
- Bạn hiểu K8s Cluster gồm **Control Plane** (chỉ huy) và **Worker Nodes** (công nhân làm việc).
- Các thành phần cốt lõi: API Server, etcd, Scheduler, Controller Manager (bên Control Plane) & kubelet, kube-proxy, container runtime (bên Worker).
- Khái niệm **Pod** là hạt nhân cơ bản nhất.

> **Bài tập nhỏ:**
> Hãy trả lời câu hỏi: Nếu mình muốn gõ lệnh tạo một ứng dụng, lệnh này đầu tiên sẽ chạy tới thành phần nào của Control Plane?
> *(Đáp án: `kube-apiserver`)*

Nếu bạn đã nắm rõ kiến trúc cơ bản này, chúng ta tiếp tục sang **Bài 2: Cài đặt và Thiết lập Môi trường** để thực sự làm quen với các công cụ nhé! Báo cho mình biết nhé!
