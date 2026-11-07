# Domain-Driven Design (DDD)

## 1. Domain-Driven Design là gì?

**Domain-Driven Design (DDD)** là phương pháp thiết kế phần mềm tập trung vào nghiệp vụ. Thay vì bắt đầu từ database, framework hoặc công nghệ, hệ thống được xây dựng dựa trên các khái niệm, quy tắc và quy trình thực tế của doanh nghiệp.

## 2. DDD giải quyết vấn đề gì?

Trong hệ thống có nghiệp vụ phức tạp, nếu toàn bộ logic đơn hàng, thanh toán và giao hàng được đặt chung trong một controller hoặc service lớn thì code sẽ ngày càng khó hiểu và khó bảo trì.

DDD chia nghiệp vụ thành những phần có ranh giới rõ ràng. Khi quy trình thanh toán thay đổi, đội phát triển chủ yếu sửa phần Payment thay vì tác động đến toàn bộ hệ thống.

## 3. Các khái niệm chính

### Domain

**Domain** là lĩnh vực nghiệp vụ mà phần mềm cần giải quyết, chẳng hạn thương mại điện tử, ngân hàng hoặc vận chuyển.

### Bounded Context

**Bounded Context** là một phạm vi nghiệp vụ có mô hình, thuật ngữ và quy tắc riêng.

Ví dụ: hệ thống bán hàng có thể được chia thành ba context là Order, Payment và Shipping. Từ “xác nhận” trong Payment có thể mang nghĩa xác nhận thanh toán, còn trong Shipping lại mang nghĩa xác nhận giao hàng.

### Ubiquitous Language

**Ubiquitous Language** là ngôn ngữ chung giữa lập trình viên và chuyên gia nghiệp vụ. Các thuật ngữ như “Đơn hàng”, “Thanh toán” và “Hoàn tiền” được sử dụng nhất quán trong trao đổi, tài liệu và source code.

### Entity

**Entity** là đối tượng có danh tính riêng và tồn tại theo thời gian, ngay cả khi các thuộc tính thay đổi.

Ví dụ: khách hàng có `customer_id` duy nhất. Tên hoặc địa chỉ có thể thay đổi nhưng họ vẫn là cùng một khách hàng.

### Value Object

**Value Object** là đối tượng không cần danh tính riêng và được xác định bởi các giá trị của nó.

Ví dụ: địa chỉ giao hàng hoặc số tiền. Hai Value Object có cùng giá trị có thể được xem là tương đương.

### Aggregate

**Aggregate** là nhóm Entity và Value Object có liên quan chặt chẽ, được quản lý như một đơn vị nhất quán. Một Entity đóng vai trò **Aggregate Root** và là điểm truy cập vào Aggregate.

Ví dụ: `Order` là Aggregate Root quản lý các `OrderItem`. Mọi thay đổi đối với `OrderItem` nên được thực hiện thông qua `Order` để bảo đảm quy tắc của đơn hàng.

### Repository

**Repository** cung cấp cách lưu trữ và truy xuất Aggregate mà domain không cần biết chi tiết database.

Ví dụ: `OrderRepository` cung cấp các phương thức lưu và tìm đơn hàng mà không để business logic phụ thuộc trực tiếp vào SQL.

### Factory

**Factory** chịu trách nhiệm tạo những domain object phức tạp và bảo đảm chúng hợp lệ ngay từ đầu.

Ví dụ: `OrderFactory` tạo một đơn hàng cùng các `OrderItem` và kiểm tra đơn phải có ít nhất một sản phẩm.

## 4. Ví dụ hệ thống bán hàng

Giả sử xây dựng một website bán sách. Khách hàng chọn sách, tạo đơn, thanh toán và nhận hàng. Các khái niệm DDD trong mục 3 có thể được áp dụng như sau.

### Domain

Domain của hệ thống là **bán sách trực tuyến**. Các nghiệp vụ quan trọng gồm:

- Tạo và hủy đơn hàng.
- Tính tổng tiền và giảm giá.
- Thanh toán và hoàn tiền.
- Đóng gói và giao sách.

### Bounded Context

Hệ thống được chia thành ba Bounded Context có trách nhiệm riêng:

| Bounded Context | Trách nhiệm | Đối tượng chính |
|---|---|---|
| Order Management | Tạo đơn, thêm sách, tính tổng tiền và hủy đơn | `Order`, `OrderItem` |
| Payment Processing | Thu tiền, xác nhận thanh toán và hoàn tiền | `Payment`, `Refund` |
| Shipping Management | Tạo vận đơn, đóng gói và cập nhật trạng thái giao hàng | `Shipment`, `DeliveryAddress` |

Mỗi context hiểu từ **trạng thái** theo một cách khác nhau. Order có trạng thái `pending`, `confirmed`, `cancelled`; Payment có `unpaid`, `paid`, `refunded`; Shipping có `waiting`, `shipped`, `delivered`.

### Ubiquitous Language

Nhóm phát triển và chuyên gia nghiệp vụ thống nhất sử dụng các thuật ngữ:

- **Đặt đơn:** khách hàng xác nhận danh sách sách muốn mua.
- **Xác nhận đơn:** hệ thống chấp nhận đơn hợp lệ.
- **Thanh toán:** thu tiền cho một đơn hàng.
- **Hoàn tiền:** trả lại khoản tiền đã thu.
- **Giao hàng:** chuyển kiện hàng cho đơn vị vận chuyển.

Các thuật ngữ này cũng xuất hiện trong code qua những tên như `place_order()`, `confirm()`, `pay()`, `refund()` và `ship()` thay vì các tên chung chung như `process_data()`.

### Entity

`Order` là một Entity vì có mã định danh `order_id`. Thông tin hoặc trạng thái của đơn có thể thay đổi nhưng mã đơn vẫn giữ nguyên.

```text
Order ID: ORD-1001
Customer: CUS-09
Status: pending → confirmed → shipped
```

Sau khi trạng thái thay đổi, đây vẫn là đơn hàng `ORD-1001`.

### Value Object

`Money` là Value Object biểu diễn số tiền và loại tiền tệ:

```text
Money(amount=250000, currency="VND")
```

`DeliveryAddress` cũng là Value Object gồm số nhà, đường, thành phố và mã bưu chính. Hai địa chỉ có toàn bộ giá trị giống nhau được xem là tương đương; chúng không cần ID riêng.

Value Object nên bất biến. Muốn đổi số tiền hoặc địa chỉ, hệ thống tạo một Value Object mới thay vì sửa trực tiếp object cũ.

### Aggregate

`Order` là Aggregate Root, còn `OrderItem` nằm bên trong Order Aggregate:

```text
Order ORD-1001
├── OrderItem: BOOK-DDD, số lượng 1, giá 200.000 VND
└── OrderItem: BOOK-PYTHON, số lượng 2, giá 150.000 VND/cuốn
```

Tổng tiền phải được tính thông qua `Order`:

```text
200.000 + (2 × 150.000) = 500.000 VND
```

Client không được tự sửa trực tiếp số lượng của `OrderItem`. Thay vào đó, client gọi `order.change_quantity("BOOK-PYTHON", 2)` để Order kiểm tra các quy tắc như số lượng phải lớn hơn 0 và đơn đã giao thì không được chỉnh sửa.

### Repository

`OrderRepository` cung cấp cách lưu và tìm Order Aggregate:

```python
class OrderRepository:
    def save(self, order): ...
    def get_by_id(self, order_id): ...
```

Business logic chỉ gọi `save()` hoặc `get_by_id()` và không cần biết dữ liệu được lưu bằng PostgreSQL, MongoDB hay bộ nhớ tạm.

Ví dụ:

```python
order = order_repository.get_by_id("ORD-1001")
order.confirm()
order_repository.save(order)
```

### Factory

`OrderFactory` tạo một đơn hàng hợp lệ ngay từ đầu:

```python
order = OrderFactory.create(
    customer_id="CUS-09",
    items=[
        {"product_id": "BOOK-DDD", "quantity": 1},
        {"product_id": "BOOK-PYTHON", "quantity": 2},
    ],
)
```

Factory có thể kiểm tra danh sách sản phẩm không được rỗng, số lượng phải lớn hơn 0, lấy giá hiện tại và tạo mã đơn. Nhờ đó, hệ thống không tạo ra một `Order` thiếu dữ liệu hoặc vi phạm quy tắc nghiệp vụ.

### Luồng xử lý hoàn chỉnh

1. `OrderFactory` tạo Order `ORD-1001` cùng các OrderItem.
2. `OrderRepository` lưu Order.
3. Payment Context thu tiền và tạo Payment riêng cho `ORD-1001`.
4. Khi thanh toán thành công, Order chuyển sang `confirmed`.
5. Shipping Context tạo Shipment và giao sách đến `DeliveryAddress`.
6. Nếu đơn bị hủy sau khi đã thanh toán nhưng chưa giao, Payment Context thực hiện hoàn tiền.

Nhờ ranh giới rõ ràng, thay đổi quy tắc hoàn tiền chủ yếu nằm trong Payment Context, còn cách đóng gói hoặc giao sách nằm trong Shipping Context.

## 5. Khi nào nên sử dụng DDD?

DDD phù hợp khi:

- Nghiệp vụ phức tạp và thay đổi thường xuyên.
- Hệ thống lớn, cần phát triển và bảo trì lâu dài.
- Nhiều nhóm cần phối hợp với chuyên gia nghiệp vụ.
- Cần xác định ranh giới rõ ràng cho các module hoặc microservice.

DDD có thể không cần thiết với dự án nhỏ, chủ yếu thực hiện CRUD, ít quy tắc nghiệp vụ hoặc có thời gian phát triển ngắn. Trong các trường hợp này, chi phí mô hình hóa có thể lớn hơn lợi ích nhận được.

## 6. Kết luận

DDD giúp source code phản ánh đúng nghiệp vụ, chia hệ thống thành các phạm vi rõ ràng và cô lập thay đổi. Giá trị quan trọng nhất của DDD không phải là tạo thật nhiều class, mà là xây dựng một mô hình nghiệp vụ chung mà cả chuyên gia nghiệp vụ và đội kỹ thuật đều hiểu.
## 7. Quiz: Domain-Driven Design

### Câu hỏi

1. DDD tập trung trước hết vào điều gì?
   - A. Giao diện đẹp
   - B. Nghiệp vụ và domain
   - C. Cấu hình server
   - D. Tốc độ mạng
2. Ubiquitous Language là gì?
   - A. Ngôn ngữ chung giữa kỹ thuật và chuyên gia nghiệp vụ
   - B. Một ngôn ngữ lập trình
   - C. Một loại database
   - D. Một giao thức mạng
3. Entity được nhận diện chủ yếu bằng gì?
   - A. Màu sắc
   - B. Danh tính ổn định
   - C. Kích thước file
   - D. Địa chỉ IP
4. Value Object thường được so sánh bằng gì?
   - A. Danh tính
   - B. Giá trị các thuộc tính
   - C. Tên bảng
   - D. Thời gian chạy
5. Aggregate có vai trò gì?
   - A. Tạo ranh giới nhất quán cho một nhóm đối tượng
   - B. Thay thế toàn bộ database
   - C. Gửi HTTP request
   - D. Tạo giao diện
6. Đối tượng bên ngoài nên thay đổi Aggregate thông qua đâu?
   - A. Bất kỳ object con nào
   - B. Aggregate Root
   - C. File cấu hình
   - D. Message broker
7. Domain Service phù hợp khi nào?
   - A. Logic nghiệp vụ không thuộc tự nhiên về một Entity/Value Object
   - B. Cần render HTML
   - C. Cần lưu log
   - D. Cần cài thư viện
8. Bounded Context dùng để làm gì?
   - A. Xác định phạm vi mà mô hình và thuật ngữ có ý nghĩa nhất quán
   - B. Giới hạn RAM
   - C. Đóng cổng mạng
   - D. Chia file Python
9. Repository cung cấp abstraction cho việc gì?
   - A. Truy xuất và lưu Aggregate
   - B. Thiết kế CSS
   - C. Mã hóa đường truyền
   - D. Quản lý CPU
10. DDD phù hợp nhất với loại hệ thống nào?
    - A. Có nghiệp vụ phức tạp
    - B. Chỉ có một trang tĩnh
    - C. Không có quy tắc nghiệp vụ
    - D. Chỉ lưu một chuỗi văn bản

### Đáp án

1. **B**; 2. **A**; 3. **B**; 4. **B**; 5. **A**; 6. **B**; 7. **A**; 8. **A**; 9. **A**; 10. **A**.
