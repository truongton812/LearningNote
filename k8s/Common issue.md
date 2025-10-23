###### không phải lúc nào Helm chart cũng hỗ trợ trực tiếp mọi cấu hình mà người dùng cần thông qua biến values.yaml. Khi bị hạn chế bởi biểu mẫu có sẵn của chart mà không thể tinh chỉnh một trường nào đó, Kustomize sẽ được dùng để "đắp vá" (patch) lại manifest đã được sinh ra từ Helm chart, giúp tùy biến những chỗ chart không hỗ trợ.


Helm chart chỉ cho phép chỉnh sửa các trường đã được người viết chart tạo biến hóa sẵn trong values.yaml.

Nhiều trường hợp chart của bên thứ ba (ví dụ như chart NGINX, Prometheus, Istio...) không có biến để cấu hình chi tiết những gì mình cần – như thêm priorityClassName, chỉnh type của Service v.v.

Fork chart để sửa sẽ gặp rắc rối lớn khi upstream cập nhật phiên bản, mã nguồn bị phân nhánh và khó bảo trì.

Giải pháp: Patch bằng Kustomize

Render manifest từ Helm (bằng lệnh helm template hoặc dùng post-renderer của Helm).

Dùng Kustomize để patch những trường hoặc resource mà chart không hỗ trợ bằng cách overlay và patch trực tiếp vào YAML kết quả.

Lối đi này giúp tận dụng được điểm mạnh của cả Helm (quản lý gói) lẫn Kustomize (patch và tùy biến YAML linh hoạt).

###### Kustomize không hỗ trợ thay biến kiểu ${MY_VAR} ngay tại runtime mà cần phải build manifest trước rồi mới apply

CI/CD pipeline muốn truyền biến image tag vào manifest khi deploy. Helm cho phép truyền biến này dễ dàng, còn Kustomize phải dùng workaround như build trước rồi apply hoặc sử dụng thêm công cụ phụ trợ như envsubst – khá bất tiện và đôi khi gây lỗi.

Muốn scale số lượng replica theo biến môi trường build, Kustomize không có giải pháp runtime thực sự, còn Helm lại đang là lựa chọn phổ biến để giải quyết vấn đề này.

"Truyền biến động ở runtime" trong ngữ cảnh triển khai Kubernetes có nghĩa là truyền hoặc thay thế giá trị biến vào manifest tại thời điểm triển khai, chứ không phải lúc tạo hoặc lưu file manifest trước đó. Tức là, giá trị cụ thể (ví dụ: image tag, tên môi trường, số replica) sẽ được chỉ định và chèn trực tiếp vào YAML ngay khi chạy lệnh deploy, tùy thuộc vào hoàn cảnh thực tế hoặc kết quả CI/CD pipeline.

Ý nghĩa "runtime" ở đây
"Runtime" nghĩa là vào lúc thực thi, khi deployment đang được thực hiện trên môi trường thật (ví dụ: production, staging), chứ không phải chuẩn bị file manifest từ trước.

Ví dụ, khi CI/CD pipeline build xong image v1.2.3, nó sẽ truyền tag này vào manifest thông qua biến runtime rồi triển khai luôn lên cluster. Nếu dùng Helm, chỉ cần chạy helm install --set image.tag=v1.2.3 là biến được truyền vào và manifest sẽ sinh ra đúng giá trị tag đó.

Kustomize thì không hỗ trợ truyền biến ở thời điểm này một cách native
Không thể trực tiếp sửa giá trị trong file patch.yaml của Kustomize để truyền biến động ở runtime theo cách native của Kustomize. File patch.yaml là tĩnh, tức nó chứa giá trị cố định, được áp dụng lúc build manifest trước khi deploy, không hỗ trợ thay biến hoặc nhận biến từ môi trường runtime như Helm. nếu muốn thay đổi phải build lại file trước hoặc dùng công cụ ngoài như dùng script hoặc công cụ bên ngoài như envsubst, sed, hoặc CI/CD pipeline để thay giá trị biến trong patch.yaml trước khi chạy Kustomize build hoặc deploy

Lý do
Kustomize chỉ xử lý các file YAML một cách tĩnh, không có cơ chế native cho phép chèn biến runtime trực tiếp vào patch.yaml khi apply manifest lên cluster.

Biến động muốn truyền phải được chuẩn bị trước bằng cách chỉnh sửa patch.yaml hoặc các file overlay trước khi chạy kubectl apply hoặc kustomize build, tức là lúc build manifest, không phải lúc runtime thực tế deploy.

Minh họa thực tế
Truyền biến động ở runtime hay dùng để:

Gán tag image mới build được từ CI/CD.

Truyền cấu hình môi trường (env, replica, ingress host…) theo môi trường deploy hôm đó.

Update secret/token hoặc cấu hình nhạy cảm mà phải lấy từ CI/CD hoặc vault



### Pod postgres không khởi động được do mount point
Lỗi xảy ra
```
Error: failed to create containerd task: failed to create shim task: OCI runtime create failed: runc create failed: unable to start container process: error during container init: error mounting "/data/postgres" to rootfs at "/var/lib/postgresql/data/": change mount propagation through procfd: open o_path procfd: open /run/containerd/io.containerd.runtime.v2.task/k8s.io/shopnow-postgresql/rootfs/var/lib/postgresql/data: no such file or directory
```

Nguyên nhân xảy ra lỗi là do liên quan đến cách Kubernetes, container runtime (containerd), và kernel Linux xử lý việc mount volume vào trong container.

 Dưới đây là phân tích chi tiết các nguyên nhân chính:

1. Thư mục đích mount không tồn tại khi runtime khởi tạo container
Khi container runtime (như runc) cố gắng mount thư mục (volume) vào một điểm trong root filesystem của container, điểm mount đích phải có sẵn trong filesystem của container. Nếu thư mục đó không tồn tại hoặc chưa được tạo ra tại thời điểm mount, lỗi kiểu này rất dễ xảy ra.

Ví dụ, thư mục /var/lib/postgresql/data trong image phải tồn tại hoặc được tạo trước khi mount volume. Nếu nó không có, mount sẽ báo lỗi "no such file or directory" kể cả khi thư mục gốc volume tồn tại.

2. Xung đột mount propagation và cơ chế mount qua procfd
Thông báo lỗi liên quan đến “change mount propagation through procfd” cho thấy quá trình mount yêu cầu thay đổi propagation flags của mount point thông qua file descriptor procfs (/proc). Một số phiên bản kernel hoặc container runtime không cho phép hoặc không xử lý tốt thao tác này khi mount vào một thư mục chưa phù hợp hoặc không tồn tại, dẫn đến lỗi.

3. Vấn đề đặc thù với mount trên thư mục gốc đã được khai báo là VOLUME trong image
Image Docker của PostgreSQL đánh dấu /var/lib/postgresql/data là một VOLUME. Khi mount volume bên ngoài vào một VOLUME được khai báo trong image, có thể xảy ra xung đột khi như là copy-on-write hoặc overlayfs thực hiện thao tác phức tạp để duy trì dữ liệu gốc hoặc metadata. Nhiều trường hợp mount phức tạp ở thư mục này bị lỗi do cơ chế kernel hoặc container runtime không xử lý chính xác.

4. Bảo mật ở tầng hệ thống (AppArmor, SELinux)
Chính sách bảo mật dạng mandatory access control có thể ngăn chặn việc mount một số thư mục nhất định hoặc các thao tác mount có propagation flag khiến container không thể mount volume thành công.


Cách xử lý:  Thay vì để container postgres tự mình khởi tạo dữ liệu (initdb) bên trong một volume được mount, chúng ta sẽ dùng một initContainer để làm một việc duy nhất: sao chép (copy) một bản cài đặt PostgreSQL đã được khởi tạo sẵn vào trong volume. Container postgres chính sau đó sẽ khởi động và chỉ việc sử dụng dữ liệu đã có sẵn này.

Cách này tránh được toàn bộ chuỗi lỗi mount propagation vì container chính không còn thực hiện thao tác initdb trên một volume mới nữa.

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shopnow-postgresql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: shopnow-postgresql
  template:
    metadata:
      labels:
        app: shopnow-postgresql
    spec:
      # --- Init Container để chuẩn bị dữ liệu ---
      initContainers:
      - name: init-postgres-data
        image: postgres:13-alpine # Dùng image nhẹ để copy
        # Lệnh này sẽ kiểm tra xem thư mục data có trống không. Nếu trống, nó sẽ sao chép dữ liệu đã init sẵn vào.
        command:
        - /bin/sh
        - -c
        - |
          if [ -z "$(ls -A /var/lib/postgresql/data)" ]; then
            echo "Initializing PostgreSQL data..."
            # Tạo thư mục tạm và sao chép dữ liệu từ image
            mkdir -p /tmp/postgres-data
            cp -R /var/lib/postgresql/data-init/* /tmp/postgres-data/
            # Di chuyển dữ liệu vào volume
            mv /tmp/postgres-data/* /var/lib/postgresql/data/
            echo "Data initialized."
          else
            echo "Data directory already exists. Skipping initialization."
          fi
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      # --- Container chính ---
      containers:
      - name: shopnow-postgresql
        image: postgres:13-alpine
        env:
        - name: POSTGRES_DB
          value: postgres
        - name: POSTGRES_USER
          value: postgres
        - name: POSTGRES_PASSWORD
          value: admin
        # PGDATA vẫn được chỉ định để đảm bảo postgres tìm đúng chỗ
        - name: PGDATA
          value: /var/lib/postgresql/data
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: shopnow-backend-postgresql
```

Diễn giải từng bước:

1. Chạy trước tiên: Khi Pod của bạn được tạo, Kubernetes thấy có một initContainer và khởi động nó trước. Nó mount PersistentVolumeClaim (volume NFS của bạn) vào đường dẫn /var/lib/postgresql/data bên trong initContainer này.

2. Kiểm tra điều kiện if [ -z "$(ls -A ...)" ]:

- Lệnh ls -A /var/lib/postgresql/data sẽ liệt kê tất cả các file và thư mục (kể cả file ẩn) bên trong volume.

- Lệnh [ -z "..." ] kiểm tra xem kết quả của ls -A có phải là một chuỗi rỗng hay không.

- Lần đầu tiên Pod chạy: Volume của bạn hoàn toàn trống. ls -A không trả về gì. Điều kiện if đúng, và khối lệnh bên trong if được thực thi.

- Những lần sau Pod khởi động lại: Volume đã có dữ liệu từ lần chạy trước. ls -A trả về danh sách file. Điều kiện if sai, và khối lệnh else được thực thi, chỉ in ra một thông báo rồi kết thúc ngay lập tức. Điều này đảm bảo dữ liệu không bị ghi đè.

3. Khởi tạo dữ liệu (chỉ xảy ra lần đầu):

- cp -R /var/lib/postgresql/data-init/* ...: Lệnh này giả định rằng image postgres:13-alpine có một thư mục /var/lib/postgresql/data-init chứa một bản sao của cấu trúc thư mục PostgreSQL đã được khởi tạo sẵn. Nó sao chép toàn bộ cấu trúc này vào volume đã được mount.

- Đây chính là điểm mấu chốt: Thay vì để container chính chạy lệnh initdb phức tạp trên một volume mạng (NFS), initContainer chỉ thực hiện một thao tác cp (sao chép file) đơn giản. Thao tác này ít phức tạp hơn nhiều và tránh được các lỗi liên quan đến "mount propagation" và quyền sở hữu mà bạn đã gặp.

4. Kết thúc và chuyển giao:

- Khi initContainer chạy xong và thoát với mã lỗi 0 (thành công), Kubernetes sẽ hủy nó đi.

- Bây giờ, Kubernetes mới bắt đầu khởi động container ứng dụng chính (shopnow-postgresql).

- Container chính này cũng mount cùng một PersistentVolumeClaim vào /var/lib/postgresql/data. Nhưng lúc này, volume đó đã chứa đầy đủ dữ liệu hợp lệ.

- Script khởi động của PostgreSQL bên trong container chính sẽ thấy thư mục dữ liệu đã tồn tại, nó sẽ bỏ qua bước initdb và khởi động thẳng server để phục vụ kết nối.
  
### Lỗi do nfs server down

Log lỗi
```
kubernetes.io/csi: attacher.MountDevice failed to create newCsiDriverClient: driver name nfs.csi.k8s.io not found in the list of registered CSI drivers

rpc error: code = DeadlineExceeded desc = context deadline exceeded
```

Tóm tắt lại quá trình xử lý sự cố:

Lỗi ban đầu: Pod báo lỗi FailedMount với hai thông báo: driver name nfs.csi.k8s.io not found và context deadline exceeded.

Phân tích Lỗi:

Lỗi driver not found -> Kiểm tra xem NFS CSI Driver đã được cài đặt chưa. Chạy lệnh sau để liệt kê tất cả các CSI driver đã được đăng ký trong cluster của bạn:

```
kubectl get csidrivers
```

Nếu các pod driver đang running bình thường thì nghĩa là kubelet trên worker node không thể tìm thấy hoặc giao tiếp được với tiến trình của NFS CSI driver đang chạy trên cùng node đó.

Lỗi context deadline exceeded (timeout) là hệ quả, cho thấy yêu cầu mount không nhận được phản hồi.

Khoanh vùng sự cố: Khi driver chạy nhưng không thể mount, nghi ngờ lớn nhất chuyển sang các yếu tố bên ngoài mà driver phụ thuộc vào: kết nối mạng và tình trạng của NFS server.


### Cách debug nâng cao khi không có shell
Tạo một ephemeral container có đầy đủ shell (alpine, busybox, ubuntu, vv.) vào cùng pod để debug bằng cách:

```
kubectl debug -it <pod> --image=busybox
```

Ephemeral container sẽ chạy cạnh container chính và có thể dùng shell đầy đủ.
