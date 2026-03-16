# Kubernetes FAQ

Tài liệu này tổng hợp các câu hỏi chuyên sâu về lý thuyết và kiến trúc nền tảng của Kubernetes (K8s) được đúc kết trong quá trình học tập.

---

## Mục lục
- [1. Khái niệm cơ bản & Kiến trúc](#1-khái-niệm-cơ-bản--kiến-trúc)
  - [Q1: Pod được hiểu là wrapper của 1 hoặc nhiều container à? Cụ thể hơn nó có thêm gì ngoài các container nó wrap?](#q1-pod-được-hiểu-là-wrapper-của-1-hoặc-nhiều-container-à-cụ-thể-hơn-nó-có-thêm-gì-ngoài-các-container-nó-wrap)
  - [Q2: Tại sao K8s (trong Docker Desktop) đã chạy nhưng tôi mở giao diện Containers lên lại không thấy container nào?](#q2-tại-sao-k8s-trong-docker-desktop-đã-chạy-nhưng-tôi-mở-giao-diện-containers-lên-lại-không-thấy-container-nào)
  - [Q3: Các Pod (như Nginx) cung cấp dịch vụ ở cổng nào? Làm sao để truy cập từ máy thật vào?](#q3-các-pod-như-nginx-cung-cấp-dịch-vụ-ở-cổng-nào-làm-sao-để-truy-cập-từ-máy-thật-vào)

---

## 1. Khái niệm cơ bản & Kiến trúc

### Q1: Pod được hiểu là wrapper của 1 hoặc nhiều container à? Cụ thể hơn nó có thêm gì ngoài các container nó wrap?
**A:** Đúng vậy, Pod là "lớp vỏ" (wrapper) bọc lấy 1 hoặc nhiều container. Tuy nhiên, Pod không chỉ là một cái túi rỗng mà nó cung cấp một môi trường (context) với các giá trị gia tăng cực kỳ quan trọng cho các container sống ở bên trong nó:

1. **Một danh tính mạng duy nhất (Shared Networking):**
   - Tất cả các container trong cùng 1 Pod sẽ dùng chung 1 địa chỉ IP và chung một dải Port hệ thống.
   - Nhờ vậy, các container khác nhau trong cùng 1 Pod có thể "nói chuyện" trực tiếp với nhau thông qua `localhost` một cách cực kỳ nhanh chóng.

2. **Không gian lưu trữ dùng chung (Shared Storage/Volumes):**
   - Pod có thể khai báo các thư mục lưu trữ (Volume) để các container dùng chung. Ví dụ: Container A ghi log ra một file, Container B ngay lập tức đọc được file log đó để đẩy lên server giám sát.

3. **Init Containers (Tiền khởi tạo môi trường):**
   - Ngoài các container ứng dụng chính, Pod cho phép khai báo các *Init Containers*. Đây là những container đặc biệt phải chạy và hoàn thành nhiệm vụ (ví dụ: chờ Database khởi động xong, hoặc download trước 1 file cấu hình lớn) thì các container chính mới được phép khởi động.

4. **Vòng đời vận hành liền khối (Lifecycle):**
   - K8s đối xử với Pod như một thể thống nhất logic. Các container trong Pod sẽ **luôn luôn** được khởi tạo và chạy trên **cùng một Worker Node** (máy chủ vật lý/ảo).
   - Khi Pod "chết" hoặc bị xoá, toàn bộ các container bên trong đều bị buộc kết thúc cùng nhau.

> **Bản chất kỹ thuật (Deep dive):** Ẩn sâu bên dưới Docker/Containerd, mỗi khi bạn tạo 1 Pod, K8s sẽ âm thầm chạy thêm một container vô hình gọi là **Pause Container** (hay Sandbox container). Nhiệm vụ của nó gần như không tiêu tốn CPU/RAM, mà chỉ để "giữ chốt" môi trường ảo (giữ Network Namespace, v.v.) để các container nghiệp vụ của bạn có chỗ dọn vào. Đây mới chính là "cái vỏ bọc" thật sự về mặt logic hệ điều hành!

---

### Q2: Tại sao K8s (trong Docker Desktop) đã chạy nhưng tôi mở giao diện Containers lên lại không thấy container nào?
**A:** Đây là một đặc điểm của việc phân chia "**luồng hiển thị**" và "**bảo vệ hệ thống**" của Docker Desktop:

1. **Docker ngầm ẩn các Container Hệ thống của Kubernetes:** 
   Các thành phần xương sống của K8s như `kube-apiserver`, `etcd`, `kube-scheduler` thực chất **đang chạy dưới dạng container**. Tuy nhiên, mặc định Docker Desktop sẽ ẩn chúng khỏi người dùng. Nó làm vậy nhằm tránh việc người dùng thấy giao diện quá lộn xộn hoặc lỡ tay bấm `Stop` / `Delete` một tiến trình hệ thống nào đó khiến toàn bộ cụm K8s "sập nguồn".
   *(Nếu bạn tò mò muốn thấy chúng trong giao diện Docker Desktop, vào **Settings -> Kubernetes -> Tích chọn "Show system containers"**, một nùi container `k8s_...` sẽ hiện ra).*

2. **Chưa có Ứng dụng/Pod nào được triển khai (Deploy):**
   Màn hình Container của Docker Desktop trống bốc vì cụm K8s vừa khởi tạo là một "vùng đất trống". Bộ sậu "ban quản lý" đang chạy ngầm, nhưng chưa có "cư dân" ứng dụng nào được dọn đến ở.
   Trong K8s, người quản trị thường sẽ chẳng bao giờ ngó ngàng vào giao diện Docker Desktop để xem container. Mọi thao tác kiểm tra trạng thái đều được gõ lệnh qua terminal. Ví dụ: Dùng lệnh `kubectl get pods -A` (`-A` tức *All Namespaces*) để dòm thẳng trực tiếp vào hệ thống (để thấy luôn cả những Pod cốt lõi của "Ban quản lý").

### Q3: Các Pod (như Nginx) cung cấp dịch vụ ở cổng nào? Làm sao để truy cập từ máy thật vào?
**A:** Đây là vấn đề cốt lõi về Mạng (Networking) trong Kubernetes. Cần phân biệt rõ các lớp truy cập:

1. **Bên trong Container:** Nginx mặc định lắng nghe ở cổng **80**. Đây là cấu hình cứng bên trong phần mềm.
2. **Bên trong Pod:** Trong file YAML, ta khai báo `containerPort: 80`. Lưu ý: Dòng này mang tính chất "khai báo" cho K8s biết là container này muốn dùng cổng đó, nó không thực sự mở cổng ra ngoài máy thật.
3. **Bên trong Cluster:** Mỗi Pod sau khi tạo sẽ được cấp 1 địa chỉ IP riêng (chỉ có hiệu lực trong mạng nội bộ K8s). Các Pod khác có thể gọi Pod Nginx qua `IP:80`.
4. **Từ máy thật (Windows) truy cập vào:** Đây là điểm mấu chốt. Mạng của máy thật và mạng của Pod là **tách biệt**. Bạn không thể gõ IP của Pod vào trình duyệt Windows để xem web.

**Để truy cập được, ta có 2 cách phổ biến:**
- **Cách tạm thời (Dùng để debug):** Sử dụng lệnh `kubectl port-forward <tên-pod> 8080:80`. Lệnh này sẽ tạo một "đường ống" nối từ cổng 8080 máy thật vào cổng 80 của Pod. Khi đó bạn truy cập qua `localhost:8080`.
- **Cách chính thống (Production):** Sử dụng đối tượng **Service**. Service sẽ tạo ra một điểm truy cập ổn định (Static IP hoặc Port trên Node) để dẫn luồng traffic từ thế giới bên ngoài vào các Pod đang chạy.

---
*(Các câu hỏi mới sẽ tiếp tục được cập nhật tại đây...)*
