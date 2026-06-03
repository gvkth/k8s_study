# Bài 11: Security và RBAC trong Kubernetes

Chào mừng bạn đến với Bài 11! Khi đưa ứng dụng lên môi trường Production, bảo mật là yếu tố sống còn. Bạn không thể cho phép bất kỳ ai (hoặc bất kỳ Pod nào) cũng có quyền xóa Database hoặc xem các Secret nhạy cảm.

Trong bài này, chúng ta sẽ tìm hiểu cách Kubernetes kiểm soát quyền truy cập thông qua **RBAC (Role-Based Access Control)**, cách cấp danh tính cho Pod với **ServiceAccount**, và cách cô lập mạng với **Network Policies**.

---

## 1. Authentication (Xác thực) vs Authorization (Phân quyền)

Trước khi đi sâu vào RBAC, hãy phân biệt rõ hai khái niệm này trong Kubernetes API:

1.  **Authentication (Ai đang gọi API?):**
    *   Xác định danh tính của người dùng hoặc máy móc.
    *   K8s hỗ trợ nhiều phương thức: Client Certificates, Bearer Tokens, OIDC (OpenID Connect), v.v.
2.  **Authorization (Người đó được phép làm gì?):**
    *   Sau khi biết bạn là ai, K8s sẽ kiểm tra xem bạn có quyền thực hiện hành động (như `GET`, `CREATE`, `DELETE`) trên tài nguyên (như `Pods`, `Secrets`) hay không.
    *   Phương thức phổ biến nhất trong K8s chính là **RBAC**.

---

## 2. RBAC: Role-Based Access Control

RBAC hoạt động dựa trên nguyên tắc: Bạn tạo ra các **"Vai trò" (Roles)** chứa các quyền hạn, sau đó **"Gắn" (Bind)** vai trò đó cho một **"Đối tượng" (Subject)**.

### Bốn thành phần chính của RBAC:

1.  **Role (Vai trò cục bộ):**
    *   Chỉ có tác dụng trong một **Namespace** cụ thể.
    *   Định nghĩa các quyền: *được phép `get`, `list`, `watch` các `pods`*.
2.  **ClusterRole (Vai trò toàn cụm):**
    *   Giống như Role nhưng có tác dụng trên **toàn bộ Cluster** (tất cả các Namespace) hoặc đối với các tài nguyên không thuộc Namespace nào (như `Nodes`).
3.  **RoleBinding (Gắn quyền cục bộ):**
    *   Gắn một `Role` cho một Subject (User/Group/ServiceAccount) bên trong một Namespace.
4.  **ClusterRoleBinding (Gắn quyền toàn cụm):**
    *   Gắn một `ClusterRole` cho một Subject trên toàn Cluster.

### Subject (Đối tượng được cấp quyền) là ai?
*   **User:** Con người (như admin, developer). K8s không quản lý User trực tiếp mà dựa vào hệ thống bên ngoài (như chứng chỉ SSL hoặc OIDC).
*   **Group:** Nhóm người dùng (ví dụ: `dev-team`).
*   **ServiceAccount:** Danh tính dành cho máy móc (ví dụ: một Pod cần gọi K8s API để đọc Secret).

---

## 3. Thực hành: Cấp quyền đọc Pod cho ServiceAccount

Giả sử bạn viết một ứng dụng giám sát (Monitoring Pod). Ứng dụng này cần quyền gọi K8s API để xem danh sách các Pod đang chạy, nhưng tuyệt đối không được phép xóa Pod.

Chúng ta sẽ sử dụng **ServiceAccount** kết hợp với **RBAC**.

### Bước 1: Tạo ServiceAccount
ServiceAccount là danh tính của Pod.
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: monitor-sa
  namespace: default
```
*(Lưu vào file `sa.yaml` và chạy `kubectl apply -f sa.yaml`)*

### Bước 2: Tạo Role (Chỉ cho phép đọc)
Tạo một Role chỉ cho phép `get`, `list`, `watch` tài nguyên `pods`.
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""] # "" nghĩa là core API group
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
*(Lưu vào file `role.yaml` và chạy `kubectl apply -f role.yaml`)*

### Bước 3: Gắn Role cho ServiceAccount bằng RoleBinding
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: monitor-sa
  namespace: default
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
*(Lưu vào file `rolebinding.yaml` và chạy `kubectl apply -f rolebinding.yaml`)*

### Bước 4: Kiểm tra quyền (Auth Can-I)
Bạn có thể nhờ K8s kiểm tra xem ServiceAccount `monitor-sa` có quyền làm một việc gì đó hay không bằng lệnh `kubectl auth can-i`:

```bash
# Kiểm tra xem có quyền lấy danh sách pod không?
kubectl auth can-i list pods --as=system:serviceaccount:default:monitor-sa
# Kết quả: yes

# Kiểm tra xem có quyền xóa pod không?
kubectl auth can-i delete pods --as=system:serviceaccount:default:monitor-sa
# Kết quả: no
```

---

## 4. Network Policies (Chính sách Mạng)

Mặc định trong Kubernetes, **mọi Pod đều có thể giao tiếp tự do với nhau**, kể cả khi chúng ở khác Namespace. Đây là một rủi ro lớn! (Ví dụ: Pod Front-end bị hack có thể kết nối thẳng vào Pod Database).

**Network Policy** giống như Firewall (Tường lửa) dành riêng cho các Pod.

### Khái niệm Ingress và Egress:
*   **Ingress:** Traffic đi **VÀO** Pod.
*   **Egress:** Traffic đi **RA** từ Pod.

### Ví dụ thực tế: Chặn tất cả, chỉ cho Web gọi Database
Giả sử bạn có Pod Database dán nhãn `app: db`. Bạn muốn cấm tất cả mọi kết nối đến Database này, TRỪ những Pod có nhãn `app: web`.

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-db
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: db        # Áp dụng tường lửa này cho các Pod có nhãn app=db
  policyTypes:
  - Ingress          # Thiết lập luật cho chiều VÀO (Ingress)
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: web   # CHỈ cho phép các Pod có nhãn app=web được kết nối tới
    ports:
    - protocol: TCP
      port: 3306     # Chỉ cho phép truy cập qua port 3306 (MySQL)
```

> **Lưu ý quan trọng:** Để Network Policies hoạt động, Cluster của bạn phải sử dụng CNI (Container Network Interface) có hỗ trợ Network Policy, ví dụ như **Calico**, **Cilium**, hoặc **Weave Net**. Flannel mặc định không hỗ trợ tính năng này.

---

## Tổng kết Bài 11

1.  **RBAC** giúp phân quyền chặt chẽ thông qua việc gán (Bind) các Quyền (Roles) cho Đối tượng (Subjects).
2.  **ServiceAccount** là cách chuẩn mực để cấp danh tính và quyền hạn cho ứng dụng chạy bên trong Pod.
3.  Lệnh `kubectl auth can-i` là công cụ đắc lực để kiểm tra quyền.
4.  **Network Policy** là tường lửa cấp độ Pod, giúp cô lập giao tiếp mạng giữa các thành phần. Mặc định nên áp dụng chính sách "Default Deny" (Cấm tất cả) và chỉ mở những luồng dữ liệu cần thiết (Zero Trust Network).

Chúc mừng bạn đã hoàn thành bài học về Bảo mật! Ở bài cuối cùng, **Bài 12: Cluster Management, Monitoring và Logging**, chúng ta sẽ tìm hiểu cách theo dõi "sức khỏe" của toàn bộ hệ thống bằng Prometheus và Grafana.
