# Kubernetes Learning Roadmap (Từ Cơ Bản Đến Nâng Cao)

Chào mừng bạn đến với lộ trình học Kubernetes! Mục tiêu của khóa học/tài liệu này là giúp bạn hiểu rõ về Kubernetes (K8s), có khả năng tự triển khai ứng dụng, và quản lý cluster một cách tự tin.

Dưới đây là lộ trình chi tiết các bài học đã và sẽ được chuẩn bị trong thư mục `materials/`:

## Phần 1: Các Khái Niệm Cơ Bản (Basic)
- [x] **Bài 1: Tổng quan và Kiến trúc Kubernetes** (`01_K8s_Architecture.md`)
  - K8s là gì? Tại sao cần dùng K8s?
  - Các thành phần của K8s Cluster (Control Plane, Worker Nodes, kube-apiserver, kubelet, v.v.).
- [x] **Bài 2: Cài đặt và Thiết lập Môi trường** (`02_Environment_Setup.md`)
  - Cài đặt Docker/containerd.
  - Cài đặt Minikube hoặc Kind (để học tập nội bộ).
  - Cài đặt và cấu hình `kubectl`.
- [x] **Bài 3: Pods và ReplicaSets** (`03_Pods_and_ReplicaSets.md`)
  - Pod là gì? Vòng đời của Pod.
  - Cách tạo và quản lý Pod.
  - Đảm bảo High Availability với ReplicaSets.

## Phần 2: Triển Khai và Cấp Phát Ứng Dụng (Workloads & Networking)
- [x] **Bài 4: Deployments** (`04_Deployments.md`)
  - Quản lý phiên bản ứng dụng với Deployment.
  - Rolling Update và Rollback.
- [x] **Bài 5: Services và Mạng trong K8s** (`05_Services_and_Networking.md`)
  - Giao tiếp giữa các Pods.
  - Các loại Service: ClusterIP, NodePort, LoadBalancer.
- [x] **Bài 6: Quản lý Cấu hình và Mật khẩu** (`06_ConfigMaps_and_Secrets.md`)
  - Tách biệt cấu hình ứng dụng với ConfigMap.
  - Lưu trữ thông tin nhạy cảm với Secret.

## Phần 3: Lưu Trữ và Quản Lý Tải Nâng Cao (Storage & Advanced Networking)
- [x] **Bài 7: Volumes và Persistent Volumes (PVC)** (`07_Storage_and_Volumes.md`)
  - Cách mount data vào Pod.
  - Lưu trữ dữ liệu lâu dài (Persistent Volumes, Storage Classes).
- [x] **Bài 8: Ingress Controllers** (`08_Ingress.md`)
  - Phân tải HTTP/HTTPS vào cluster.
  - Cấu hình Ingress resource.
- [x] **Bài 9: StatefulSets và DaemonSets** (`09_StatefulSets_DaemonSets.md`)
  - Triển khai ứng dụng có trạng thái (Database, Message Queue).
  - Chạy agent trên mỗi node (DaemonSet logs, monitoring).


## Phần 4: Nâng Cao & Quản Lý Cluster (Advanced & Operations)
- [x] **Bài 10: Helm Charts** (`10_Helm.md`)
  - Đóng gói và chia sẻ ứng dụng K8s.
  - Cài đặt ứng dụng bằng Helm.
- [x] **Bài 11: Security và RBAC** (`11_Security_RBAC.md`)
  - Phân quyền người dùng và service accounts.
  - Policies (Network Policies, Pod Security).
- [ ] **Bài 12: Cluster Management, Monitoring và Logging** (`12_Monitoring_Logging.md`)
  - Cài đặt Prometheus và Grafana.
  - Xem log tập trung (EFK stack).
  - Quản trị dung lượng Node (Horizontal Pod Autoscaler - HPA).

Bạn có thể bắt đầu với **Bài 1** và **Bài 2** dể hiểu kiến thức nền tảng và cài đặt môi trường nhé! Lúc nào bạn sẵn sàng, mình sẽ bổ sung tiếp các bài tiếp theo!
