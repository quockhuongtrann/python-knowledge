# Microservices and API Integration

## 1 Service boundaries

### Chia theo năng lực nghiệp vụ

Ranh giới service nên làm rõ một phần nghiệp vụ, dữ liệu thuộc quyền sở hữu của nó và hợp đồng với các bên khác:

| Service | Trách nhiệm | Dữ liệu sở hữu | Hợp đồng chính |
|---|---|---|---|
| Orders | Tiếp nhận đơn và điều phối tiến trình checkout | Đơn, trạng thái tiến trình, kết quả idempotency đầu vào | Tạo đơn, xem trạng thái |
| Inventory | Kiểm tra và giữ hàng | Tồn kho, reservation, thời hạn giữ | Giữ và giải phóng hàng |
| Payment | Thực hiện và đối soát thanh toán | Payment attempt, tham chiếu nhà cung cấp, trạng thái giao dịch | Thu tiền, tra cứu, hoàn tiền |
| Notification | Gửi thông báo theo kết quả đã ghi nhận | Lịch sử gửi, trạng thái giao tin | Nhận yêu cầu gửi thông báo |

Orders sở hữu việc điều phối nhưng không tự cập nhật số dư tồn kho hoặc bảng giao dịch Payment. Inventory có quyền từ chối giữ hàng; Payment có quyền trả kết quả declined hoặc pending.

### Mỗi Microservice quản lý dữ liệu riêng của mình

Mỗi service kiểm soát đường đọc/ghi dữ liệu nội bộ của mình. Có thể dùng hạ tầng database chung lúc đầu, nhưng cần tách quyền sở hữu schema, quyền truy cập và migration. Service khác gọi API hoặc sử dụng dữ liệu đã được công bố qua event thay vì join trực tiếp vào bảng nội bộ.

Ví dụ, Orders lưu giá đã chốt và mã sản phẩm của đơn. Inventory quản lý số lượng có thể giữ. Orders không dùng một transaction ghi thẳng cả bảng đơn lẫn bảng tồn kho để né hợp đồng service.


### Cách các Microservice giao tiếp với nhau: Đồng bộ và Bất đồng bộ

| Cách tích hợp | Phù hợp với ví dụ | Điều phải giải quyết |
|---|---|---|
| HTTP/gRPC đồng bộ | Truy vấn trạng thái hoặc cần kết quả trực tiếp | Deadline, timeout, lỗi từng phần |
| Message/event bất đồng bộ | Gửi thông báo, tiến trình nhiều bước | Giao lại, thứ tự, backlog, eventual consistency |

Đưa việc vào queue không làm mất lỗi; nó chuyển lỗi sang xử lý thông điệp và trạng thái tiến trình. Trong thiết kế minh họa, API tạo đơn trả `202 Accepted` sau khi lưu bền vững yêu cầu; worker tiếp tục xử lý checkout.

**Câu chuyển ý:** “Sau khi biết từng service chịu trách nhiệm gì, chúng ta cần một cách để client tiếp cận hệ thống mà không phải biết toàn bộ topology bên trong.”

## 2 API gateway

API Gateway là máy chủ trung gian, đóng vai trò **điểm truy cập duy nhất** giữa client và các microservice. Thay vì gọi trực tiếp từng service, client gửi yêu cầu đến API Gateway để được kiểm tra, xử lý và chuyển đến service phù hợp.

```text
Client → API Gateway → Microservice → API Gateway → Client
```

### Các chức năng chính

- Định tuyến request đến đúng microservice.
- Xác thực và phân quyền bằng API Key, OAuth 2.0 hoặc JWT.
- Giới hạn số lượng request bằng rate limiting và throttling.
- Chuyển đổi định dạng dữ liệu, chẳng hạn từ XML sang JSON.
- Lọc dữ liệu không cần thiết hoặc nhạy cảm.
- Ghi log và cung cấp dữ liệu phục vụ giám sát.
- Cache response để giảm độ trễ và tải cho backend.
- Có thể cân bằng tải giữa các phiên bản của service.

### Lợi ích

- Client chỉ cần biết một địa chỉ API duy nhất.
- Che giấu cấu trúc và sự phức tạp của hệ thống phía sau.
- Tập trung các chính sách bảo mật và kiểm soát truy cập.
- Giảm mã xử lý lặp lại trong từng microservice.
- Cải thiện hiệu suất và hỗ trợ mở rộng hệ thống.

### Lưu ý khi triển khai

Không nên đưa logic nghiệp vụ phức tạp vào API Gateway. Hệ thống cũng cần cấu hình timeout, logging, monitoring, scaling và cập nhật bảo mật thường xuyên.

**Kết luận:** API Gateway đơn giản hóa giao tiếp giữa client và hệ thống microservices, đồng thời tập trung việc định tuyến, bảo mật và kiểm soát lưu lượng.


## 3 Contracts: hợp đồng tích hợp

Khi hệ thống được chia thành nhiều microservice, các service có thể sử dụng ngôn ngữ lập trình khác nhau và chạy trên những máy chủ riêng biệt. Chúng không thể chia sẻ trực tiếp `interface` trong source code, vì vậy cần một **API Contract độc lập với ngôn ngữ lập trình** để quy định cách giao tiếp.

### Vì sao cần API Contract?

Nếu API chỉ được mô tả qua tin nhắn hoặc trao đổi miệng, nhiều thông tin quan trọng sẽ không rõ ràng:

- Request cần những trường nào và dữ liệu phải thỏa mãn điều kiện gì?
- Response có cấu trúc và ý nghĩa như thế nào?
- Khi xảy ra lỗi, API trả HTTP status và mã lỗi nào?
- Client có được retry request hay không?
- Một thay đổi trong API có làm hỏng các ứng dụng đang sử dụng nó không?

Ví dụ, nếu Orders Service đang gửi trường `customer_id` nhưng service cung cấp API âm thầm đổi tên thành `user_id`, các client cũ có thể ngừng hoạt động. API Contract giúp provider và consumer thống nhất rõ đầu vào, đầu ra, điều kiện hợp lệ và hành vi khi xảy ra lỗi.

### OpenAPI/Swagger cho REST API

OpenAPI sử dụng file JSON hoặc YAML để mô tả hợp đồng của REST API, bao gồm endpoint, HTTP method, request, response, kiểu dữ liệu, trường bắt buộc và các trường hợp lỗi.

```yaml
openapi: 3.0.0
info:
  title: Payment Service API
  version: 1.0.0
paths:
  /payments:
    post:
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [orderId, amount]
              properties:
                orderId:
                  type: string
                amount:
                  type: number
                  minimum: 1
      responses:
        '200':
          description: Thanh toán thành công
          content:
            application/json:
              schema:
                type: object
                properties:
                  transactionId:
                    type: string
                  status:
                    type: string
                    example: SUCCESS
        '400':
          description: Dữ liệu đầu vào không hợp lệ
```

Từ hợp đồng OpenAPI, nhóm phát triển có thể sinh client SDK, tạo khung code backend, hiển thị tài liệu bằng Swagger UI và tạo mock server để các bên phát triển song song. Tuy nhiên, schema chỉ mô tả hợp đồng; authorization, idempotency và logic nghiệp vụ vẫn phải được triển khai và kiểm thử trong code.

### gRPC và Protocol Buffers

Đối với giao tiếp nội bộ giữa các microservice, gRPC sử dụng file `.proto` để định nghĩa service, phương thức RPC, request, response và kiểu dữ liệu của từng field.

```protobuf
syntax = "proto3";

package payment;

service PaymentService {
  rpc ProcessPayment (PaymentRequest) returns (PaymentResponse);
}

message PaymentRequest {
  string order_id = 1;
  double amount = 2;
}

message PaymentResponse {
  string transaction_id = 1;
  string status = 2;
}
```

Từ cùng một file `.proto`, công cụ gRPC có thể sinh code cho nhiều ngôn ngữ như Java, Go hoặc Python. Protocol Buffers sử dụng định dạng nhị phân nhỏ gọn và hợp đồng kiểu dữ liệu chặt chẽ, phù hợp với giao tiếp service-to-service cần hiệu suất cao.

### Thiết kế hợp đồng trước khi viết code

Nên áp dụng cách tiếp cận **Contract-First Design**:

1. Backend, frontend, mobile và các service liên quan cùng thống nhất OpenAPI hoặc Protobuf.
2. Consumer sử dụng mock server hoặc code được sinh từ hợp đồng để phát triển trước.
3. Provider triển khai logic theo đúng hợp đồng đã thống nhất.
4. CI kiểm tra implementation và các thay đổi có tương thích với consumer hay không.

**Kết luận:** API Contract là giao diện chung giữa các hệ thống phân tán. OpenAPI phù hợp để đặc tả REST API, còn gRPC/Protobuf cung cấp hợp đồng chặt chẽ và hiệu quả cho giao tiếp nội bộ. Giá trị quan trọng nhất là giúp các bên thống nhất đầu vào, đầu ra và hành vi trước khi triển khai.

## 4 Resilience: kiểm soát lỗi và sự cố lan truyền

**Resilience** là khả năng hệ thống vẫn hoạt động ổn định hoặc dừng an toàn khi một service bị chậm hay gặp lỗi. Mục tiêu là không để lỗi của một service làm hỏng toàn bộ hệ thống.

Ví dụ: Payment Service bị lỗi thì Orders Service vẫn ghi nhận đơn ở trạng thái `pending`, thay vì làm sập toàn bộ luồng đặt hàng.

### Timeout

**Timeout** là thời gian tối đa một service chờ phản hồi từ service khác. Hết thời gian này, request được dừng để tránh phải chờ vô hạn.

Ví dụ: Orders Service chỉ chờ Payment Service trong 3 giây. Sau 3 giây chưa có phản hồi, Orders dừng chờ và đánh dấu giao dịch là `unknown` để kiểm tra lại sau.

### Retry và backoff

**Retry** là thử gọi lại khi lỗi có thể chỉ xảy ra tạm thời. **Backoff** là chờ một khoảng thời gian trước khi thử lại để service đang lỗi có thời gian phục hồi.

Ví dụ: Orders gọi Payment thất bại, chờ 1 giây rồi thử lại một lần. Không retry lỗi sai mật khẩu, thiếu quyền hoặc dữ liệu không hợp lệ vì thử lại cũng không giải quyết được vấn đề.

Retry phải có giới hạn. Nếu mỗi tầng đều tự retry nhiều lần, số request có thể tăng nhanh và làm service bị quá tải nặng hơn.

### Idempotency

**Idempotency** bảo đảm gửi lại cùng một yêu cầu không tạo ra kết quả trùng lặp. Client gửi kèm một `Idempotency-Key`, server lưu key này để nhận biết request đã được xử lý.

Ví dụ: client gửi yêu cầu thanh toán hai lần với cùng key `PAY-1001`. Payment Service chỉ trừ tiền một lần và trả lại kết quả đã lưu cho lần gọi thứ hai.

- Cùng key và cùng nội dung: trả lại kết quả cũ.
- Cùng key nhưng khác nội dung: từ chối vì dữ liệu xung đột.

### Circuit breaker

**Circuit breaker** tạm ngừng gửi request đến một service đang lỗi liên tục. Cơ chế này giúp hệ thống phản hồi nhanh và tránh tạo thêm tải cho service đang gặp sự cố.

Ví dụ: Payment Service lỗi liên tiếp năm lần, circuit breaker tạm dừng gọi trong 30 giây. Sau đó hệ thống thử một request để kiểm tra service đã phục hồi hay chưa.

### Bulkhead và rate limiting

**Bulkhead** chia riêng tài nguyên cho từng chức năng để lỗi ở một nơi không chiếm hết tài nguyên của hệ thống. **Rate limiting** giới hạn số request được tiếp nhận trong một khoảng thời gian.

Ví dụ: Payment và Notification sử dụng hai connection pool riêng. Notification bị treo sẽ không dùng hết connection dành cho Payment. API cũng có thể giới hạn mỗi client chỉ được gửi 100 request/phút.

### Fallback

**Fallback** là kết quả thay thế khi chức năng chính tạm thời không hoạt động.

Ví dụ: Recommendation Service bị lỗi thì trang sản phẩm vẫn hiển thị, nhưng tạm ẩn phần sản phẩm gợi ý. Không được dùng fallback để giả vờ rằng thanh toán đã thành công.

## 5 Versioning và thay đổi tương thích

**Versioning** là cách quản lý các phiên bản API khi API cần thay đổi. Mục tiêu là cho phép service phát triển tính năng mới mà không làm hỏng các client đang sử dụng phiên bản cũ.

Ví dụ: ứng dụng mobile đang gọi `/v1/orders`. Backend có thể phát hành `/v2/orders` và vẫn giữ `/v1/orders` trong thời gian người dùng chưa cập nhật ứng dụng.


### Cách đặt phiên bản API

| Cách | Ví dụ |
|---|---|
| Đặt trong URL | `/v1/orders`, `/v2/orders` |
| Đặt trong header | `API-Version: 2` |

Đặt version trong URL là cách đơn giản, dễ nhìn và dễ định tuyến. Tuy nhiên, hệ thống phải duy trì nhiều phiên bản API trong giai đoạn chuyển đổi.

### Khi nào cần tạo version mới?

Chỉ nên tạo version mới khi có **breaking change** — thay đổi khiến client cũ không thể tiếp tục hoạt động.

Ví dụ:

```json
// v1
{"total_vnd": 350000}

// v2
{"total": {"amount": 350000, "currency": "VND"}}
```

Cấu trúc dữ liệu đã thay đổi nên API cần phiên bản mới. Backend giữ v1 trong một khoảng thời gian để các client chuyển dần sang v2, sau đó mới thông báo và ngừng hỗ trợ v1.

### Quy trình nâng cấp đơn giản

1. Tạo và công bố contract của phiên bản mới.
2. Giữ phiên bản cũ hoạt động trong thời gian chuyển đổi.
3. Cho các client chuyển dần sang phiên bản mới.
4. Theo dõi lỗi và lượng truy cập của từng phiên bản.
5. Thông báo thời điểm ngừng hỗ trợ phiên bản cũ.


## 6 Distributed tracing

**Distributed Tracing** là kỹ thuật theo dõi toàn bộ hành trình của một request khi nó đi qua nhiều thành phần trong hệ thống phân tán hoặc microservices. Nó giúp biết request đã đi qua đâu, service nào xử lý chậm và lỗi phát sinh tại bước nào.

### Cách hoạt động

Distributed Tracing sử dụng ba khái niệm chính:

- **Trace:** toàn bộ hành trình của một request.
- **Span:** một hoạt động cụ thể trong trace, chẳng hạn truy vấn database hoặc gọi một service.
- **Context propagation:** truyền thông tin trace giữa các service để liên kết các span.

Quy trình cơ bản:

1. Client gửi request đến hệ thống.
2. Hệ thống tạo một trace.
3. Trace context được truyền qua các service.
4. Mỗi service ghi lại các span của mình.
5. Các span được tập hợp để hiển thị toàn bộ luồng xử lý.

Ví dụ:

```text
Client → Orders Service → Inventory Service → Payment Service
```

Nếu Payment Service mất 2 giây trong tổng thời gian 2,5 giây, trace sẽ giúp xác định Payment là vị trí gây chậm.

### Lợi ích

- Hiểu cách các service tương tác với nhau.
- Xác định service hoặc thao tác gây độ trễ.
- Phát hiện lỗi trong một request cụ thể.
- Hỗ trợ tối ưu hiệu suất và độ tin cậy.
- Giảm thời gian điều tra sự cố.

### Distributed Tracing và Logging

| Distributed Tracing | Logging |
|---|---|
| Theo dõi một request qua nhiều service | Ghi lại sự kiện trong từng ứng dụng |
| Cho biết request đi qua đâu và chậm ở bước nào | Cung cấp chi tiết lỗi và dữ liệu debug |
| Phù hợp với hệ thống phân tán | Có thể sử dụng trong mọi hệ thống |

Hai công cụ bổ trợ cho nhau, không thay thế nhau. Trace nên chứa thông tin liên kết đến log để việc điều tra lỗi thuận tiện hơn.

**Kết luận:** Distributed Tracing giúp quan sát luồng xử lý xuyên suốt hệ thống microservices, đặc biệt hữu ích để tìm điểm nghẽn và nguyên nhân lỗi. Có thể bắt đầu thử nghiệm bằng Jaeger, sau đó mở rộng tùy theo nhu cầu giám sát.
## 7 Quiz: Microservices and API Integration

### Câu hỏi

1. Service boundary nên được xác định chủ yếu dựa trên gì?
   - A. Năng lực nghiệp vụ
   - B. Số dòng code
   - C. Màu giao diện
   - D. Tên máy chủ
2. API Gateway thường đứng ở đâu?
   - A. Trước các service phía sau
   - B. Bên trong database
   - C. Trong entity
   - D. Sau log file
3. API contract mô tả điều gì?
   - A. Cách các hệ thống trao đổi request và response
   - B. Cách mua máy chủ
   - C. Cách viết CSS
   - D. Cách đặt tên biến nội bộ
4. Timeout giúp tránh điều gì?
   - A. Chờ một dependency vô thời hạn
   - B. Tạo log
   - C. Xác thực user
   - D. Lưu cache
5. Retry phù hợp nhất với lỗi nào?
   - A. Lỗi tạm thời
   - B. Dữ liệu đầu vào luôn sai
   - C. Không có quyền truy cập
   - D. API đã bị xóa vĩnh viễn
6. Circuit breaker dùng để làm gì?
   - A. Tạm dừng gọi dependency đang lỗi liên tục
   - B. Tăng kích thước database
   - C. Thay đổi API version
   - D. Tạo user
7. Thay đổi backward-compatible là gì?
   - A. Client cũ vẫn hoạt động
   - B. Mọi client cũ đều hỏng
   - C. Không có tài liệu
   - D. Xóa trường bắt buộc
8. Distributed tracing giúp theo dõi điều gì?
   - A. Một request qua nhiều service
   - B. Chỉ dung lượng ổ đĩa
   - C. Chỉ source code
   - D. Chỉ mật khẩu
9. Correlation/trace ID nên được làm gì giữa các service?
   - A. Truyền tiếp
   - B. Xóa ngay
   - C. Dùng làm mật khẩu
   - D. Ghi cứng
10. Mỗi microservice nên sở hữu điều gì?
    - A. Trách nhiệm nghiệp vụ rõ ràng
    - B. Toàn bộ nghiệp vụ của hệ thống
    - C. Mọi database của công ty
    - D. Một bản sao của mọi service khác

### Đáp án

1. **A**; 2. **A**; 3. **A**; 4. **A**; 5. **A**; 6. **A**; 7. **A**; 8. **A**; 9. **A**; 10. **A**.
