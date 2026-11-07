# Containers and Kubernetes Fundamentals

## 1 Image và Container

### Các khái niệm

- **Dockerfile:** file chứa các bước dùng để tạo image, ví dụ cài Python, cài thư viện và sao chép source code.
- **Image:** gói chỉ đọc chứa source code, thư viện và môi trường cần thiết để chạy ứng dụng.
- **Image tag:** tên phiên bản dễ đọc của image, ví dụ `orders-api:1.0`.
- **Registry:** nơi lưu và phân phối image, chẳng hạn Docker Hub hoặc Amazon ECR.
- **Container:** một tiến trình đang chạy được tạo từ image. Một image có thể tạo ra nhiều container.
- **Volume:** vùng lưu trữ được gắn vào container. Dữ liệu trong volume có thể tồn tại độc lập với container tùy loại volume.

### Ví dụ cụ thể

Nhóm phát triển tạo image `orders-api:1.0` từ ứng dụng Django rồi đẩy image lên Amazon ECR. Máy chủ tải image này về và chạy ba container giống nhau:

- Container 1 nhận request tạo đơn hàng A.
- Container 2 nhận request xem đơn hàng B.
- Container 3 nhận request cập nhật đơn hàng C.

Nếu Container 2 bị xóa, code trong image không mất và một container mới có thể được tạo lại. Tuy nhiên, file được ghi trực tiếp bên trong Container 2 có thể mất. Vì vậy, thông tin đơn hàng được lưu trong PostgreSQL và ảnh sản phẩm được lưu trong object storage thay vì bên trong container.

## 2 Kubernetes, Cluster, Node và Pod

### Các khái niệm

- **Kubernetes:** nền tảng điều phối container. Kubernetes nhận cấu hình mong muốn rồi quyết định chạy container ở máy nào, theo dõi trạng thái, thay thế Pod bị lỗi và tăng hoặc giảm số lượng Pod. Kubernetes không tạo source code hay image; nó sử dụng image đã được build sẵn.
- **Cluster:** toàn bộ môi trường Kubernetes. Một cluster gồm **control plane** để đưa ra quyết định quản lý và các **node** cung cấp CPU, memory để chạy ứng dụng.
- **Node:** một máy vật lý hoặc máy ảo thuộc cluster. Mỗi node chạy các thành phần Kubernetes cần thiết để nhận Pod được phân công, khởi động container và báo trạng thái về control plane.
- **Pod:** đơn vị triển khai nhỏ nhất trong Kubernetes. Pod bao bọc một hoặc nhiều container cần chạy cùng nhau. Các container trong cùng Pod dùng chung địa chỉ IP, có thể dùng chung volume và thường được tạo hoặc xóa cùng nhau. Pod có tính tạm thời nên không nên giữ dữ liệu nghiệp vụ duy nhất bên trong nó.
- **Replica:** một bản sao của cùng một Pod template. Nhiều replica giúp chia tải và tránh việc toàn bộ ứng dụng dừng khi một Pod gặp lỗi.
- **Desired state:** trạng thái người vận hành muốn hệ thống duy trì, ví dụ image `orders-api:1.0` và luôn có ba Pod hoạt động.
- **Controller:** thành phần liên tục so sánh trạng thái thực tế với desired state. Khi phát hiện thiếu Pod, controller yêu cầu tạo thêm để đưa hệ thống về đúng trạng thái mong muốn.

### Ví dụ cụ thể

Cluster có hai node là `node-a` và `node-b`. Ứng dụng Orders API cần ba Pod:

- `orders-pod-1` chạy trên `node-a`.
- `orders-pod-2` chạy trên `node-a`.
- `orders-pod-3` chạy trên `node-b`.

Mỗi Pod chạy một container Django từ image `orders-api:1.0` và có địa chỉ IP riêng trong cluster. Người vận hành không cần chọn thủ công node cho từng Pod; Kubernetes chọn node còn đủ CPU và memory.

Nếu `orders-pod-2` bị lỗi, container và Pod đó không được sửa tại chỗ. Kubernetes phát hiện thực tế chỉ còn hai Pod trong khi desired state là ba, rồi tạo một Pod thay thế trên node còn đủ tài nguyên.

## 3 Deployment và ReplicaSet

### Các khái niệm

- **Deployment:** tài nguyên cấp cao dùng để quản lý việc chạy và cập nhật ứng dụng. Deployment khai báo image cần dùng, số Pod mong muốn và cách thay Pod cũ bằng Pod mới.
- **ReplicaSet:** tài nguyên nằm dưới Deployment, chịu trách nhiệm duy trì đúng số lượng Pod của một phiên bản. Nếu thiếu Pod, ReplicaSet tạo thêm; nếu thừa Pod, ReplicaSet giảm bớt.
- **Pod template:** bản mẫu nằm trong Deployment, mô tả cách tạo mỗi Pod, chẳng hạn image, port, biến môi trường và resource request.
- **Replica:** một bản sao của Pod được tạo từ cùng một Pod template. `replicas: 3` nghĩa là cần ba Pod giống nhau.
- **Selector:** điều kiện dùng label để xác định Pod nào thuộc ReplicaSet, ví dụ `app: orders`.
- **Manifest:** file YAML ghi lại trạng thái mong muốn của tài nguyên Kubernetes. Ta gửi manifest cho Kubernetes thay vì tự tạo và quản lý từng Pod.
- **Stateless application:** ứng dụng không giữ dữ liệu duy nhất trong một Pod. Pod nào cũng có thể xử lý request và có thể được thay thế mà không làm mất dữ liệu nghiệp vụ.
- **Self-healing:** khả năng tự khôi phục số Pod mong muốn khi một Pod bị lỗi, bị xóa hoặc node gặp sự cố.

Quan hệ giữa ba thành phần:

`Deployment → ReplicaSet → Pod`

- Deployment quản lý quá trình triển khai và các phiên bản.
- ReplicaSet quản lý số lượng Pod của một phiên bản.
- Pod trực tiếp chạy container của ứng dụng.

### Ví dụ cụ thể

Deployment `orders-api` khai báo:

- Image: `orders-api:1.0`.
- Số replica: `3`.
- Port ứng dụng: `8000`.
- Label của Pod: `app: orders`.

Khi Deployment được tạo, nó tạo một ReplicaSet cho phiên bản `1.0`. ReplicaSet này tạo ba Pod từ Pod template và dùng label `app: orders` để theo dõi chúng.

Nếu một Pod bị xóa, ReplicaSet thấy chỉ còn hai Pod nên tạo một Pod mới. Deployment không trực tiếp sửa Pod bị mất; ReplicaSet thực hiện việc duy trì số lượng này.

Khi image được đổi thành `orders-api:1.1`, Deployment tạo một ReplicaSet mới cho phiên bản `1.1`, tăng dần số Pod mới và giảm dần số Pod thuộc ReplicaSet `1.0`. Vì dữ liệu đơn hàng nằm trong PostgreSQL, các Pod mới vẫn đọc được dữ liệu đã có.

## 4 Service và cách truy cập ứng dụng

### Các khái niệm

- **Service:** điểm truy cập mạng ổn định đứng trước một nhóm Pod. Service có tên DNS và địa chỉ riêng, sau đó phân phối request đến các Pod phù hợp. Nhờ vậy, client không cần biết IP thay đổi liên tục của từng Pod.
- **Label:** thông tin dạng khóa–giá trị gắn vào tài nguyên để phân loại, ví dụ `app: orders` hoặc `version: v1`. Label không tự điều hướng traffic.
- **Selector:** điều kiện Service dùng để chọn Pod dựa trên label. Service có selector `app: orders` chỉ gửi traffic đến các Pod mang label tương ứng.
- **Endpoint:** địa chỉ IP và port thực tế của Pod mà Service có thể chuyển request tới. Pod chưa sẵn sàng thường không được dùng làm endpoint nhận traffic thông thường.
- **ClusterIP:** loại Service mặc định, chỉ cung cấp điểm truy cập bên trong cluster. Nó phù hợp khi frontend hoặc một service nội bộ cần gọi Orders API.
- **LoadBalancer:** loại Service yêu cầu hạ tầng cloud tạo điểm truy cập từ bên ngoài cluster. Việc có địa chỉ public không tự cung cấp authentication, HTTPS hoặc authorization cho ứng dụng.
- **Port:** cổng mà client dùng khi gọi Service, ví dụ port `80`.
- **Target port:** cổng của ứng dụng bên trong Pod mà Service chuyển request tới, ví dụ port `8000` của Django.

### Ví dụ cụ thể

Ba Pod Orders API đều có label `app: orders` và lắng nghe tại port `8000`. Service `orders-service` có cấu hình:

- Selector: `app: orders`.
- Port: `80`.
- Target port: `8000`.

Frontend bên trong cluster gọi `http://orders-service/orders/123`. DNS của Kubernetes phân giải tên `orders-service` đến ClusterIP của Service. Service nhận request tại port 80 rồi chọn một Pod Orders API đang sẵn sàng và chuyển request tới port 8000.

Nếu `orders-pod-1` bị thay bằng một Pod có IP mới, danh sách endpoint của Service được cập nhật. Frontend vẫn gọi cùng tên `orders-service` và không cần biết sự thay đổi đó.

Nếu một Pod vô tình mang label `app: payment` thay vì `app: orders`, selector không chọn Pod đó nên Service không gửi traffic đến nó.

## 5 ConfigMap và Secret

### Các khái niệm

- **ConfigMap:** tài nguyên Kubernetes lưu cấu hình không nhạy cảm dưới dạng key–value hoặc nội dung file. ConfigMap phù hợp với `LOG_LEVEL=INFO`, nhưng không phù hợp để lưu mật khẩu.
- **Secret:** tài nguyên Kubernetes dành cho dữ liệu nhạy cảm như mật khẩu, API token và khóa bí mật. Giá trị Secret thường được biểu diễn bằng base64 trong manifest, nhưng base64 chỉ là encoding, không phải encryption.
- **Environment variable:** biến được đưa vào process khi container khởi động, ví dụ `LOG_LEVEL=INFO`. Nếu ConfigMap hoặc Secret thay đổi sau đó, process đang chạy thường không tự nhận giá trị environment mới.
- **Volume mount:** cách đưa một key hoặc file từ ConfigMap/Secret vào filesystem của container. File được mount có thể được Kubernetes cập nhật, nhưng ứng dụng vẫn phải đọc lại giá trị để thay đổi có hiệu lực.
- **Reference:** Pod chỉ tham chiếu tên ConfigMap hoặc Secret thay vì chép trực tiếp giá trị vào cấu hình container. Điều này giúp cùng một image dùng được ở nhiều môi trường.
- **Secret rotation:** quá trình tạo credential mới, cập nhật ứng dụng sử dụng credential mới rồi thu hồi credential cũ. Chỉ sửa Secret mà không cập nhật các Pod sử dụng nó thì rotation chưa hoàn tất.

### Ví dụ cụ thể

Orders API dùng ConfigMap `orders-config` chứa:

- `APP_ENV=production`.
- `LOG_LEVEL=INFO`.
- `PAYMENT_TIMEOUT_SECONDS=5`.

Ứng dụng dùng Secret `orders-secret` chứa:

- `DATABASE_URL` để kết nối PostgreSQL.
- `DJANGO_SECRET_KEY` để phục vụ cơ chế bảo mật của Django.
- `PAYMENT_API_TOKEN` để gọi dịch vụ thanh toán.

Cùng image `orders-api:1.0` có thể chạy ở staging với `LOG_LEVEL=DEBUG` và ở production với `LOG_LEVEL=INFO`. Deployment chỉ thay tham chiếu hoặc giá trị cấu hình theo từng môi trường, không cần build hai image khác nhau.

Pod nhận `LOG_LEVEL` từ ConfigMap và `DATABASE_URL` từ Secret dưới dạng environment variable lúc khởi động. Mật khẩu thật không được ghi trong Dockerfile hoặc commit vào Git.

Khi đổi mật khẩu database, nhóm tạo mật khẩu mới, cập nhật `orders-secret`, tạo lại các Pod để process nhận `DATABASE_URL` mới, kiểm tra kết nối thành công rồi mới thu hồi mật khẩu cũ.

## 6 Startup, Readiness và Liveness Probe

### Các khái niệm

- **Probe:** phép kiểm tra được kubelet thực hiện định kỳ trên container. Probe có thể gọi một HTTP endpoint, mở kết nối TCP hoặc chạy một command để xác định trạng thái ứng dụng.
- **Startup probe:** xác định ứng dụng đã khởi động hoàn tất chưa. Khi startup probe chưa thành công, Kubernetes chưa chạy readiness và liveness probe, nhờ đó ứng dụng khởi động chậm không bị restart quá sớm.
- **Readiness probe:** xác định Pod có thể nhận traffic hay không. Khi probe thất bại, Pod bị loại khỏi danh sách endpoint của Service nhưng container không bị restart chỉ vì lý do này.
- **Liveness probe:** xác định process còn hoạt động bình thường hay đã mắc kẹt. Khi probe thất bại đủ số lần, Kubernetes restart container để thử phục hồi.
- **Health endpoint:** URL nhẹ do ứng dụng cung cấp để trả lời probe, ví dụ `/health/ready`. Endpoint không nên trả secret hoặc thực hiện công việc tốn nhiều tài nguyên.
- **Period:** khoảng thời gian giữa hai lần kiểm tra, ví dụ cứ 10 giây kiểm tra một lần.
- **Timeout:** thời gian tối đa Kubernetes chờ một lần kiểm tra trả lời trước khi tính là thất bại.
- **Failure threshold:** số lần thất bại liên tiếp cần đạt trước khi Kubernetes thực hiện hành động tương ứng.

### Ví dụ cụ thể

Orders API cung cấp ba endpoint:

- `/health/startup`: thành công sau khi Django tải xong cấu hình.
- `/health/ready`: thành công khi ứng dụng có thể nhận request.
- `/health/live`: thành công khi process không bị treo.

Ứng dụng cần khoảng 20 giây để khởi động. Startup probe chạy mỗi 5 giây và cho phép tối đa 6 lần thất bại, nên ứng dụng có khoảng 30 giây để hoàn tất. Khi startup thành công, readiness và liveness probe mới bắt đầu hoạt động.

Nếu readiness thành công, Pod được thêm vào endpoint của Service và có thể nhận traffic. Nếu readiness tạm thất bại, request mới không được gửi đến Pod đó cho đến khi probe thành công trở lại.

Nếu process Django bị treo, liveness probe thất bại nhiều lần và Kubernetes restart container. Nếu database chỉ chậm tạm thời, readiness có thể tạm loại Pod khỏi Service; liveness không nên restart process chỉ vì một dependency bên ngoài gặp lỗi.

## 7 Resource Request, Limit và Autoscaling

### Các khái niệm

- **Resource request:** lượng tài nguyên Pod dự kiến cần. Scheduler dùng request để tìm node còn đủ chỗ; request không phải lượng tài nguyên được cấp riêng tuyệt đối cho Pod.
- **CPU request:** lượng CPU dùng trong quyết định scheduling và trong một số phép tính autoscaling. `250m` nghĩa là một phần tư CPU core.
- **Memory request:** lượng bộ nhớ dự kiến cần để Pod chạy bình thường, ví dụ `256Mi`.
- **CPU limit:** mức CPU tối đa container được phép sử dụng. Khi muốn dùng vượt giới hạn trong thời gian dài, container có thể bị CPU throttling và chạy chậm hơn.
- **Memory limit:** mức bộ nhớ tối đa của container. Nếu vượt mức này, container có thể bị kết thúc với trạng thái `OOMKilled`.
- **Metric:** số liệu HPA dùng để đánh giá tải, ví dụ CPU utilization, số request hoặc độ dài hàng đợi.
- **HPA (Horizontal Pod Autoscaler):** controller tính số replica cần thiết từ metric và điều chỉnh Deployment trong giới hạn đã cấu hình. HPA tăng số Pod chứ không tăng CPU hoặc memory của từng Pod.
- **Minimum replicas:** số Pod tối thiểu HPA phải duy trì ngay cả khi tải thấp.
- **Maximum replicas:** giới hạn trên để tránh tạo quá nhiều Pod và làm quá tải database hoặc dịch vụ phụ thuộc.
- **Pending Pod:** Pod đã được tạo nhưng chưa thể chạy, thường vì không có node đáp ứng đủ request hoặc điều kiện scheduling.

### Ví dụ cụ thể

Mỗi Pod Orders API có:

- CPU request: `250m`, tương đương một phần tư CPU core.
- Memory request: `256Mi`.
- Memory limit: `512Mi`.

HPA được cấu hình với tối thiểu hai Pod, tối đa tám Pod và mục tiêu CPU trung bình 60%. Bình thường hệ thống chạy hai Pod. Khi chiến dịch khuyến mãi làm CPU duy trì trên mục tiêu, HPA tăng lên bốn Pod. Khi lượng request giảm và ổn định, HPA giảm số Pod về hai.

Mức 60% được tính so với CPU request. Với request `250m`, 60% tương ứng khoảng `150m` CPU trung bình trên mỗi Pod, không phải 60% của toàn bộ node.

Nếu cluster không còn node đủ CPU hoặc memory, Pod mới có thể ở trạng thái Pending. HPA tạo nhu cầu về Pod nhưng không tự thêm node và cũng không tự giải quyết giới hạn database connection.

## 8 Rollout và Rollback

### Các khái niệm

- **Rollout:** quá trình Deployment đưa thay đổi trong Pod template vào sử dụng, ví dụ đổi image hoặc environment variable. Kubernetes tạo một ReplicaSet mới và chuyển dần số replica sang đó.
- **Revision:** một phiên bản lịch sử của Pod template do Deployment quản lý, có thể được dùng làm mốc khi rollback.
- **Rolling update:** chiến lược thay Pod cũ bằng Pod mới theo từng bước thay vì dừng toàn bộ phiên bản cũ cùng lúc.
- **Max surge:** số Pod được phép tạo vượt quá số replica mong muốn trong lúc cập nhật. Với ba replica và `maxSurge: 1`, hệ thống có thể tạm thời chạy tối đa bốn Pod.
- **Max unavailable:** số Pod tối đa được phép không sẵn sàng trong lúc cập nhật. Với `maxUnavailable: 0`, Kubernetes chờ Pod mới ready trước khi giảm một Pod cũ.
- **Rollback:** yêu cầu Deployment quay Pod template về revision trước. Rollback không mặc định tự xảy ra chỉ vì rollout bị lỗi.
- **Zero-downtime deployment:** mục tiêu cập nhật mà người dùng vẫn tiếp tục sử dụng được ứng dụng. Mục tiêu này còn phụ thuộc readiness probe, tài nguyên cluster và khả năng tương thích của database.

### Ví dụ cụ thể

Hệ thống đang chạy ba Pod `orders-api:1.0`. Nhóm đổi Deployment sang image `orders-api:1.1` với `maxSurge: 1` và `maxUnavailable: 0`:

1. Deployment tạo ReplicaSet mới cho phiên bản `1.1` và Kubernetes tạo thêm một Pod mới.
2. Readiness probe kiểm tra Pod mới đã phục vụ được hay chưa.
3. Khi Pod mới sẵn sàng, Kubernetes xóa một Pod `1.0`.
4. Quá trình lặp lại cho đến khi cả ba Pod chạy phiên bản `1.1`; ReplicaSet cũ được giảm về 0 replica nhưng có thể được giữ lại trong lịch sử.

Nếu Pod `1.1` liên tục lỗi readiness, rollout sẽ không thể tiếp tục bình thường. Nếu tỷ lệ request lỗi tăng cao, nhóm chủ động dừng rollout và rollback về revision dùng image `1.0`.

Rollback chỉ đưa Pod template, như image và một số cấu hình, về phiên bản trước. Nó không tự khôi phục dữ liệu database, secret bên ngoài hoặc thay đổi không tương thích đã được thực hiện bởi migration.
## 9 Quiz: Containers and Kubernetes Fundamentals

### Câu hỏi

1. Container được tạo từ thành phần nào?
   - A. Image
   - B. Service
   - C. Node
   - D. Secret
2. Đơn vị triển khai nhỏ nhất trong Kubernetes là gì?
   - A. Cluster
   - B. Pod
   - C. Node
   - D. Registry
3. Deployment thường quản lý tài nguyên nào để duy trì số Pod?
   - A. ReplicaSet
   - B. ConfigMap
   - C. Volume
   - D. Registry
4. Kubernetes Service cung cấp điều gì?
   - A. Điểm truy cập ổn định đến một nhóm Pod
   - B. Nơi build image
   - C. Nơi lưu source code
   - D. Một database mặc định
5. Dữ liệu nào phù hợp để lưu trong ConfigMap?
   - A. `LOG_LEVEL=INFO`
   - B. Mật khẩu database
   - C. Private key
   - D. API token
6. Readiness probe thất bại thường khiến Pod thế nào?
   - A. Tạm không nhận traffic từ Service
   - B. Bị xóa cluster ngay
   - C. Tự đổi image
   - D. Tạo database mới
7. Memory limit bị vượt có thể dẫn đến trạng thái nào?
   - A. OOMKilled
   - B. Ready
   - C. Running mãi mãi
   - D. CacheHit
8. HPA dùng để làm gì?
   - A. Tự điều chỉnh số replica theo metric
   - B. Mã hóa Secret
   - C. Build Dockerfile
   - D. Tạo domain name
9. Rolling update có mục tiêu gì?
   - A. Thay dần Pod cũ bằng Pod mới
   - B. Xóa toàn bộ cluster
   - C. Sao lưu database
   - D. Đổi tên Service
10. Rollback Deployment dùng để làm gì?
    - A. Quay lại revision trước
    - B. Phục hồi tự động mọi dữ liệu database
    - C. Tạo registry mới
    - D. Tăng CPU node

### Đáp án

1. **A**; 2. **B**; 3. **A**; 4. **A**; 5. **A**; 6. **A**; 7. **A**; 8. **A**; 9. **A**; 10. **A**.
