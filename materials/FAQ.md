# Kubernetes FAQ

Tài liệu này tổng hợp các câu hỏi chuyên sâu về lý thuyết và kiến trúc nền tảng của Kubernetes (K8s) được đúc kết trong quá trình học tập.

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
*(Các câu hỏi mới sẽ tiếp tục được cập nhật tại đây...)*
