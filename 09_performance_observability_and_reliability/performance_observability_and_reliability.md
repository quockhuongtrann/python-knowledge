# Performance, Observability, and Reliability

## 1 Load Testing

### Các khái niệm

- **Load testing:** kiểm tra hệ thống dưới một mô hình traffic có kiểm soát để đo tốc độ, tỷ lệ lỗi và khả năng chịu tải. Kết quả chỉ có ý nghĩa khi dữ liệu, hành vi và môi trường test đủ gần với thực tế.
- **Virtual user:** người dùng giả lập thực hiện một chuỗi hành động, ví dụ đăng nhập, xem đơn rồi tạo đơn. Một virtual user không nhất thiết tương ứng với một thread.
- **Request rate:** số request gửi đến hệ thống trong một đơn vị thời gian, thường là request/giây. Đây là tốc độ đầu vào, không bảo đảm tất cả request đều hoàn thành thành công.
- **Concurrent users:** số người dùng hoặc luồng công việc đang hoạt động cùng lúc. Concurrency cao có thể làm tăng số connection và memory cần dùng dù request rate không đổi.
- **Workload model:** mô tả loại hành động, tỷ lệ, thời gian chờ và mức tải cần mô phỏng. Workload chỉ đọc dữ liệu thường cho kết quả khác hệ thống có cả thao tác ghi.
- **Ramp-up:** tăng tải theo từng bước để quan sát thời điểm hệ thống bắt đầu chậm hoặc lỗi, thay vì gửi toàn bộ traffic ngay lập tức.
- **Throughput:** số request hoặc công việc hoàn thành thành công trong một đơn vị thời gian. Throughput có thể ngừng tăng khi hệ thống chạm bottleneck dù traffic đầu vào tiếp tục tăng.
- **Saturation:** trạng thái một tài nguyên gần hoặc đã dùng hết khả năng phục vụ, ví dụ CPU 100%, connection pool đầy hoặc hàng đợi tăng liên tục.

### Ví dụ cụ thể

Orders API dự kiến có 100 người dùng đồng thời. Trong một phút:

- 70% request dùng để xem danh sách đơn hàng.
- 20% request dùng để xem chi tiết một đơn.
- 10% request dùng để tạo đơn mới.

Bài load test tăng từ 10 lên 100 virtual user trong 5 phút, giữ tải trong 15 phút rồi giảm dần. Nhóm theo dõi thời gian phản hồi, tỷ lệ lỗi, CPU và số database connection.

Nếu API nhanh ở 50 user nhưng bắt đầu trả lỗi khi đạt 90 user, kết quả cho thấy giới hạn hiện tại nằm dưới tải dự kiến và cần được điều tra.

## 2 Profiling

### Các khái niệm

- **Profiling:** thu thập số liệu bên trong chương trình để biết thời gian, CPU hoặc memory được dùng ở đâu. Profiling giúp tối ưu đúng vị trí thay vì đoán.
- **CPU profiling:** đo hàm hoặc đoạn code nào tiêu thụ nhiều CPU. Nó hữu ích với tính toán nặng nhưng không giải thích đầy đủ thời gian chờ database hoặc network.
- **Memory profiling:** theo dõi nơi tạo và giữ object trong memory, giúp tìm peak memory hoặc memory leak.
- **I/O wait:** thời gian ứng dụng chờ database, file, network hoặc dịch vụ bên ngoài trả lời. CPU có thể vẫn thấp trong khi request bị chậm vì I/O wait.
- **Hot path:** đường chạy xuất hiện thường xuyên hoặc đóng góp phần lớn chi phí xử lý. Một cải thiện nhỏ trên hot path có thể ảnh hưởng lớn đến toàn hệ thống.
- **Bottleneck:** tài nguyên hoặc bước đang giới hạn throughput và latency. Sau khi sửa một bottleneck, một giới hạn khác có thể trở thành bottleneck mới.
- **N+1 query:** lỗi lấy danh sách bằng một query rồi chạy thêm một query cho từng phần tử. Số query tăng theo số bản ghi nên endpoint chậm dần khi dữ liệu lớn hơn.

### Ví dụ cụ thể

Endpoint `GET /orders` mất 1,2 giây. Profiling cho thấy:

- Code Python xử lý trong 50 ms.
- Database query mất 1.100 ms.
- Phần còn lại mất 50 ms.

Kết quả cho thấy bottleneck nằm ở database, không phải CPU của Django.

Log query cho thấy API lấy 100 đơn hàng bằng một query, sau đó chạy thêm 100 query để lấy thông tin khách hàng. Nhóm dùng `select_related("customer")` để lấy dữ liệu cần thiết trong một query phù hợp. Sau thay đổi, endpoint còn 180 ms.

## 3 Caching

### Các khái niệm

- **Cache:** lớp lưu tạm dữ liệu đã tính hoặc đã đọc để request sau không phải làm lại công việc tốn thời gian. Cache cải thiện tốc độ nhưng tạo thêm bài toán nhất quán dữ liệu.
- **Cache key:** định danh dùng để lưu và tìm một giá trị, ví dụ `tenant:7:order:1001`. Key cần chứa đủ phạm vi để dữ liệu của các user hoặc tenant không bị trộn lẫn.
- **Cache hit:** ứng dụng tìm thấy dữ liệu hợp lệ trong cache và không cần đọc nguồn chính. Hit rate cao thường giúp giảm tải database.
- **Cache miss:** dữ liệu không tồn tại hoặc đã hết hạn, khiến ứng dụng phải lấy từ nguồn chính rồi có thể ghi lại vào cache.
- **TTL (Time To Live):** thời gian tối đa một giá trị được giữ trước khi tự hết hạn. TTL ngắn giảm thời gian dùng dữ liệu cũ nhưng tăng số lần đọc nguồn chính.
- **Cache invalidation:** xóa hoặc cập nhật cache khi dữ liệu gốc thay đổi để tránh trả giá trị cũ. Đây thường là phần khó nhất của thiết kế cache.
- **Stale data:** dữ liệu cache không còn giống nguồn chính. Mức stale có thể chấp nhận phụ thuộc nghiệp vụ, ví dụ mô tả sản phẩm khác với số dư tài khoản.
- **Cache stampede:** nhiều request cùng gặp cache miss và đồng thời gọi nguồn chính, có thể làm database quá tải. Có thể giảm bằng locking hoặc làm mới trước khi hết hạn.

### Ví dụ cụ thể

Orders API lưu kết quả `GET /products/10` trong Redis với key `product:10` và TTL 60 giây.

Request đầu tiên không tìm thấy cache nên đọc sản phẩm từ PostgreSQL rồi ghi kết quả vào Redis. Các request tiếp theo trong 60 giây đọc từ Redis và trả lời nhanh hơn.

Khi giá sản phẩm 10 thay đổi, ứng dụng xóa key `product:10`. Nếu không xóa, người dùng có thể thấy giá cũ cho đến khi TTL hết hạn.

Không nên dùng chung key `orders:list` cho mọi khách hàng. Key cần chứa tenant hoặc user phù hợp, nếu không dữ liệu của khách A có thể được trả cho khách B.

## 4 Metrics

### Các khái niệm

- **Metric:** giá trị số được thu thập lặp lại theo thời gian để quan sát xu hướng và trạng thái tổng thể. Metric phù hợp cho dashboard và alert nhưng thường không chứa chi tiết từng request.
- **Counter:** giá trị tích lũy chỉ tăng cho đến khi process khởi động lại, ví dụ tổng số request hoặc lỗi. Tốc độ tăng thường hữu ích hơn giá trị tuyệt đối.
- **Gauge:** giá trị tại một thời điểm và có thể tăng hoặc giảm, ví dụ số request đang xử lý, queue depth hoặc memory đang dùng.
- **Histogram:** đếm các quan sát vào nhiều khoảng để mô tả phân phối, thường dùng để tính percentile latency trên nhiều instance.
- **Latency:** thời gian từ lúc nhận công việc đến lúc hoàn thành. Nên xem p95 hoặc p99 vì giá trị trung bình có thể che các request rất chậm.
- **Error rate:** tỷ lệ request được xem là thất bại trên tổng request theo một định nghĩa rõ ràng. Cần phân biệt lỗi server với request không hợp lệ từ client.
- **Traffic:** khối lượng công việc hệ thống nhận được, ví dụ request/giây hoặc message/giây. Traffic giúp giải thích thay đổi của latency và tài nguyên.
- **Cardinality:** số tổ hợp label tạo ra các time series. Label như `user_id` có cardinality lớn, làm tăng memory và chi phí lưu trữ metric.

### Ví dụ cụ thể

Orders API thu thập các metric:

- `http_requests_total`: tổng số request theo endpoint và status code.
- `http_request_duration_seconds`: thời gian phản hồi của request.
- `http_requests_in_progress`: số request đang được xử lý.
- `db_connections_in_use`: số database connection đang được dùng.

Dashboard cho thấy latency tăng cùng lúc với `db_connections_in_use` đạt giới hạn 20. Điều này gợi ý request phải chờ connection database.

Không dùng `user_id` hoặc `order_id` làm label metric vì có thể tạo hàng triệu chuỗi thời gian. Các ID cụ thể phù hợp hơn với log hoặc trace.

## 5 Logs

### Các khái niệm

- **Log:** bản ghi rời rạc mô tả một sự kiện đã xảy ra, ví dụ request thất bại hoặc worker xử lý xong message. Log giúp điều tra chi tiết nhưng có thể tốn chi phí nếu ghi quá nhiều.
- **Structured log:** log được chia thành các trường cố định, thường ở dạng JSON, để dễ lọc và thống kê mà không phải phân tích câu text tự do.
- **Log level:** mức độ quan trọng như `DEBUG`, `INFO`, `WARNING` và `ERROR`. Chọn sai level có thể tạo quá nhiều nhiễu hoặc làm bỏ sót lỗi thật.
- **Timestamp:** thời điểm sự kiện xảy ra, nên có timezone thống nhất và đồng bộ clock giữa các máy để dựng timeline chính xác.
- **Correlation ID:** mã được truyền qua các bước của cùng một request hoặc job để nối các log liên quan giữa nhiều process.
- **Context:** metadata giúp hiểu sự kiện, ví dụ service, environment, endpoint, status code hoặc tenant. Context không nên chứa dữ liệu nhạy cảm không cần thiết.
- **Redaction:** che hoặc loại bỏ mật khẩu, token và dữ liệu nhạy cảm trước khi log được ghi hoặc gửi đến hệ thống tập trung.

### Ví dụ cụ thể

Khi request tạo đơn thất bại, Orders API ghi một structured log:

```json
{
  "level": "ERROR",
  "event": "order_create_failed",
  "correlation_id": "req-abc-123",
  "tenant_id": "tenant-7",
  "status_code": 500,
  "duration_ms": 850
}
```

Đội vận hành tìm `correlation_id=req-abc-123` để xem toàn bộ sự kiện của request đó.

Log không được chứa mật khẩu, access token, số thẻ hoặc toàn bộ dữ liệu cá nhân. Một lỗi xuất hiện ở mọi request cũng không nên ghi stack trace lặp lại với tốc độ quá lớn vì có thể làm tăng chi phí và che khuất tín hiệu quan trọng.

## 6 Distributed Tracing

### Các khái niệm

- **Trace:** bản ghi hành trình của một request hoặc job khi đi qua nhiều thành phần. Trace cho biết thứ tự, quan hệ và thời gian của từng bước.
- **Span:** một đơn vị công việc có thời điểm bắt đầu và kết thúc, ví dụ xử lý Django view, truy vấn database hoặc gọi Payment API.
- **Trace ID:** mã chung được gắn cho tất cả span thuộc cùng một hành trình, giúp tìm toàn bộ request từ đầu đến cuối.
- **Span ID:** mã riêng của từng span, dùng cùng parent ID để xây dựng cây quan hệ giữa các bước.
- **Parent span:** span trực tiếp tạo hoặc gọi một span con. Quan hệ parent–child giúp xác định bước nào chờ bước nào.
- **Trace context propagation:** truyền trace ID qua HTTP header hoặc message metadata. Nếu propagation bị thiếu, trace sẽ bị tách thành nhiều phần rời rạc.
- **Sampling:** chỉ thu thập hoặc lưu một phần trace để kiểm soát chi phí. Chính sách sampling nên tránh làm mất toàn bộ trace lỗi hoặc rất chậm.

### Ví dụ cụ thể

Request `POST /orders` mất 1.500 ms. Trace cho thấy:

- Django xử lý request: 100 ms.
- Truy vấn PostgreSQL: 150 ms.
- Gọi Inventory API: 200 ms.
- Gọi Payment API: 1.000 ms.
- Các bước còn lại: 50 ms.

Trace giúp xác định phần chậm nhất là Payment API thay vì Orders API hoặc database.

Orders API truyền trace context khi gọi Payment API. Nhờ vậy, span của hai dịch vụ xuất hiện trong cùng một trace thay vì thành hai hành trình không liên quan.

## 7 SLI, SLO và Error Budget

### Các khái niệm

- **SLI (Service Level Indicator):** phép đo cụ thể phản ánh trải nghiệm thực tế, ví dụ tỷ lệ request hợp lệ trả thành công. SLI phải nêu rõ nguồn dữ liệu và request nào được tính.
- **SLO (Service Level Objective):** mục tiêu nội bộ cho một SLI trong một khoảng thời gian, ví dụ 99,9% request thành công trong 30 ngày. SLO giúp cân bằng phát triển tính năng với độ ổn định.
- **SLA (Service Level Agreement):** cam kết chính thức với khách hàng, có thể kèm hoàn tiền hoặc điều khoản khác khi không đạt. SLA khác SLO ở tính hợp đồng và hậu quả kinh doanh.
- **Availability:** tỷ lệ thời gian hoặc request mà dịch vụ thực hiện thành công chức năng cần thiết. Đo theo request thường phản ánh trải nghiệm tốt hơn chỉ kiểm tra process còn chạy.
- **Latency SLO:** mục tiêu về phần trăm request phải hoàn thành dưới một ngưỡng, ví dụ 95% dưới 300 ms. Percentile giúp nhìn thấy request chậm bị trung bình che khuất.
- **Error budget:** lượng lỗi hoặc thời gian không đạt được phép xảy ra mà vẫn giữ SLO. Nó tạo giới hạn rõ ràng cho mức rủi ro có thể chấp nhận.
- **Burn rate:** tốc độ tiêu thụ error budget so với tốc độ cho phép. Burn rate cao cho thấy SLO sẽ sớm bị vi phạm nếu tình trạng tiếp tục.

### Ví dụ cụ thể

Orders API đặt SLO trong 30 ngày:

- 99,9% request hợp lệ phải trả response không phải lỗi 5xx.
- 95% request xem đơn phải hoàn thành trong 300 ms.

Với 1.000.000 request mỗi tháng, SLO 99,9% cho phép tối đa khoảng 1.000 request không thành công theo cách đo đã định nghĩa. Đây là error budget của tháng đó.

Nếu hệ thống đã dùng gần hết error budget trong tuần đầu, nhóm ưu tiên sửa độ ổn định và có thể hoãn thay đổi rủi ro cao.

## 8 Alerting

### Các khái niệm

- **Alert:** thông báo tự động cho con người khi tín hiệu cho thấy dịch vụ có thể đang ảnh hưởng người dùng và cần điều tra.
- **Alert rule:** biểu thức kết hợp metric, ngưỡng và thời gian duy trì để quyết định khi nào tạo hoặc đóng alert.
- **Threshold:** giá trị ranh giới, ví dụ error rate 5%. Threshold nên dựa trên mức bình thường và tác động thay vì chọn tùy ý.
- **Actionable alert:** cảnh báo có tác động, người chịu trách nhiệm, mức ưu tiên và hành động rõ ràng. Nếu không có hành động cần làm, tín hiệu có thể chỉ nên nằm trên dashboard.
- **False positive:** alert xuất hiện nhưng không có vấn đề thực sự cần xử lý. Quá nhiều false positive làm giảm niềm tin vào hệ thống cảnh báo.
- **Alert fatigue:** trạng thái người trực nhận quá nhiều cảnh báo lặp lại hoặc ít giá trị, dẫn đến phản ứng chậm hay bỏ sót sự cố nghiêm trọng.
- **Runbook:** tài liệu ngắn hướng dẫn cách xác nhận tác động, kiểm tra dashboard, log, trace và thực hiện biện pháp giảm thiểu an toàn.

### Ví dụ cụ thể

Orders API có rule: tạo alert nghiêm trọng khi tỷ lệ lỗi 5xx vượt 5% trong 10 phút và có ít nhất 100 request.

Alert chứa:

- Tên dịch vụ và môi trường.
- Thời điểm bắt đầu.
- Dashboard liên quan.
- Link đến log và trace.
- Runbook xử lý lỗi 5xx.

Không nên đánh thức người trực chỉ vì CPU vượt 80% trong một phút nếu người dùng vẫn được phục vụ bình thường. Cảnh báo nên ưu tiên tác động thực tế như tỷ lệ lỗi hoặc latency vi phạm SLO.

## 9 Incident Response và Reliability

### Các khái niệm

- **Reliability:** khả năng hệ thống liên tục thực hiện đúng chức năng ở mức chất lượng đã cam kết khi gặp tải, lỗi thành phần và thay đổi thực tế.
- **Incident:** sự kiện ngoài kế hoạch làm giảm chất lượng, sai dữ liệu hoặc gián đoạn dịch vụ và cần phản ứng có phối hợp.
- **Detection:** nhận biết dấu hiệu sự cố qua alert, dashboard, synthetic check hoặc phản hồi người dùng. Detection nhanh giúp giảm thời gian ảnh hưởng.
- **Triage:** nhanh chóng xác định phạm vi, mức độ nghiêm trọng, dịch vụ liên quan và người cần tham gia trước khi điều tra sâu.
- **Mitigation:** hành động ưu tiên giảm tác động ngay, ví dụ rollback hoặc tắt tính năng. Mitigation không nhất thiết là bản sửa nguyên nhân gốc.
- **Recovery:** đưa dịch vụ về trạng thái ổn định, xác nhận các chỉ số phục hồi và theo dõi xem sự cố có tái diễn hay không.
- **Runbook:** hướng dẫn đã chuẩn bị cho một loại sự cố, giúp người trực thực hiện các bước an toàn và nhất quán dưới áp lực.
- **Postmortem:** phân tích timeline, tác động, nguyên nhân và khoảng trống phòng vệ, sau đó tạo hành động cải thiện có người phụ trách.
- **MTTR:** thời gian trung bình từ khi sự cố bắt đầu hoặc được phát hiện đến khi dịch vụ được khôi phục, tùy định nghĩa thống nhất của tổ chức.

### Ví dụ cụ thể

Alert báo latency của Orders API vượt 2 giây và tỷ lệ lỗi đạt 8%. Đội trực thực hiện:

1. Xác nhận khách hàng đang bị ảnh hưởng và công bố trạng thái sự cố.
2. Kiểm tra dashboard, log và trace theo cùng khoảng thời gian.
3. Phát hiện database connection pool đã đầy sau một bản release mới.
4. Rollback release để giảm ảnh hưởng ngay lập tức.
5. Xác nhận latency và tỷ lệ lỗi trở lại bình thường.
6. Sửa nguyên nhân, bổ sung load test và triển khai lại có kiểm soát.
7. Viết postmortem, nêu rõ timeline, nguyên nhân và người phụ trách từng hành động tiếp theo.

Mục tiêu của postmortem là cải thiện hệ thống và quy trình, không phải tìm cá nhân để đổ lỗi.
## 10 Quiz: Performance, Observability, and Reliability

### Câu hỏi

1. Load testing dùng để đánh giá điều gì?
   - A. Hệ thống hoạt động dưới tải dự kiến
   - B. Màu giao diện
   - C. Độ dài mật khẩu
   - D. Tên database
2. Profiling giúp tìm gì?
   - A. Phần code tiêu tốn thời gian hoặc tài nguyên
   - B. User mới
   - C. DNS record
   - D. API key
3. Cache hit nghĩa là gì?
   - A. Tìm thấy dữ liệu hợp lệ trong cache
   - B. Cache bị xóa
   - C. Database bị lỗi
   - D. Request bị từ chối
4. Counter có đặc điểm nào?
   - A. Thường chỉ tăng cho đến khi process khởi động lại
   - B. Luôn giảm
   - C. Chứa source code
   - D. Là một loại log
5. Correlation ID trong log giúp làm gì?
   - A. Liên kết các sự kiện của cùng request
   - B. Mã hóa database
   - C. Thay thế user ID
   - D. Tăng CPU
6. Một trace bao gồm nhiều thành phần nhỏ nào?
   - A. Span
   - B. Queue
   - C. Image
   - D. Secret
7. SLO là gì?
   - A. Mục tiêu cho một chỉ số dịch vụ trong khoảng thời gian
   - B. Một loại database
   - C. Một access token
   - D. Một ngôn ngữ lập trình
8. Error budget biểu thị điều gì?
   - A. Mức lỗi được phép mà vẫn đạt SLO
   - B. Chi phí mua server
   - C. Số developer
   - D. Dung lượng cache
9. Một alert tốt nên có đặc điểm nào?
   - A. Có thể hành động và gắn với tác động thực tế
   - B. Luôn báo mỗi phút
   - C. Không có người phụ trách
   - D. Không có ngữ cảnh
10. Mitigation trong incident response nhằm mục tiêu gì?
    - A. Giảm tác động trước mắt
    - B. Viết lại toàn bộ hệ thống ngay
    - C. Xóa mọi metric
    - D. Bỏ qua người dùng

### Đáp án

1. **A**; 2. **A**; 3. **A**; 4. **A**; 5. **A**; 6. **A**; 7. **A**; 8. **A**; 9. **A**; 10. **A**.
