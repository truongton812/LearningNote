Học lại về hạ tầng pki, rất hay

<img width="1024" height="576" alt="image" src="https://github.com/user-attachments/assets/15448756-d5b1-43dd-89ed-4157f4c36ece" />

---

Container under the hood

Container bản chất là sử dụng linux namespace
- pid namespace: để isolate process. Các process không thể nhìn thấy nhau, tuy nhiên bản chất vẫn là những process chạy trên máy host
- mount namespace
- network namespace
- user namespace

Ngoài namespace, container còn sử dụng cgroup để giới hạn tài nguyên 1 container có thể sử dụng

---

Từ sau bản k8s 1.28, dùng lệnh docker ps sẽ ko thấy container, phải dùng lệnh crictl ps

Lưu ý 
- docker là container runtime + tool để manage container và image
- containerd là container runtime
- podman là tool để manage container và image
- crictl là cli cho cri-compatible container runtime. Ta có thể dùng crictl để tương tác với docker/podman/containerd tùy ý, chỉ cần config runtime endpoint trong /etc/crictl.yaml

## 4. Network policy

Network policy là namespaced resource

- Network policy cho phép tất cả Pod trong default namespace  giao tiếp bình thường
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector: {}
  policyTypes: []
```
- Network policy kiểm soát inbound traffic (ingress) của tất cả các Pod trong namespace default. Tuy nhiên, vì trong trường này không có quy tắc cụ thể nào trong phần ingress: (không định nghĩa rule cho phép hoặc từ chối traffic nào), Kubernetes mặc định sẽ chặn toàn bộ lưu lượng vào các Pod này
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: test-network-policy
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```
- Network policy cô lập hoàn toàn các Pod trong default namespace, chặn cả outbound và inbound traffic. Lưu ý trong thực tế ta sẽ dùng default network policy là deny all như thế này, sau đó tạo các allow ingress và egress để đảm bảo bảo mật. Tuy nhiên deny all sẽ chặn cả DNS traffic (port 53) nên nếu muốn sử dụng service name trong cụm thì cần allow port 53
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
  egress:
  - to:
    ports:
    - port: 53
      protocol: TCP
    - port: 53
      protocol: UDP

```

- Nếu 1 pod được áp nhiều network policy thì sẽ là union của tất cả các network policy áp lên

---

Ví dụ về network policy
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: example
  namespace: default
spec:
  podSelector:
    matchLabels:
      id: frontend
  policyTypes:
  - Egress
  egress:
  - to:
    - namespaceSelector:
        matchLabels:
          id: ns1
    ports:
    - protocol: TCP
      port: 80
  - to:
    - podSelector:
        matchLabels:
          id: backend
```

Ở phần egress có 2 cái block , sẽ là logic OR

Trong 1 block có 2 key là `to` và `ports`. sẽ là logic AND

Chỉ define Egress, tức Ingress ko bị limit

### 4.1 Network policy trong Cloud

Mặc định khi tạo worker node trên cloud thì các pod trên worker node sẽ có quyền truy cập metadata server. Trong đấy có thể chứa các sensitive data

Có thể dùng network policy để restrict pod có quyền truy cập metadata server

Ví dụ dùng network policy để restrict truy cập đến metadata server 169.254.169.254/32
```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: cloud-metadata-deny
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - ipBlock:
        cidr: 0.0.0.0/0
        except:
        - 169.254.169.254/32
```
