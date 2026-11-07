# Security Engineering

## 1 Threat Modeling

### Các khái niệm

- **Threat modeling:** quá trình mô tả hệ thống, xác định tài sản cần bảo vệ, tìm cách hệ thống có thể bị tấn công hoặc sử dụng sai rồi chọn biện pháp phòng vệ. Hoạt động này nên được thực hiện trước khi triển khai và cập nhật khi kiến trúc thay đổi.
- **Asset:** dữ liệu, chức năng hoặc tài nguyên có giá trị cần được bảo vệ, ví dụ tài khoản người dùng, đơn hàng, mật khẩu, source code và khả năng phục vụ của hệ thống.
- **Threat:** tình huống hoặc hành động có thể gây hại cho asset, chẳng hạn đọc dữ liệu trái phép, sửa đơn hàng hoặc làm dịch vụ ngừng hoạt động.
- **Vulnerability:** điểm yếu trong thiết kế, code hoặc cấu hình mà threat có thể khai thác, ví dụ API không kiểm tra người sở hữu đơn hàng.
- **Attack surface:** toàn bộ điểm mà người dùng hoặc hệ thống bên ngoài có thể tương tác, như API, trang đăng nhập, file upload và cổng mạng. Attack surface càng lớn thì càng có nhiều vị trí cần bảo vệ.
- **Mitigation:** biện pháp làm giảm khả năng xảy ra hoặc tác động của threat, ví dụ kiểm tra quyền, giới hạn request hoặc mã hóa dữ liệu.
- **Risk:** mức độ cần ưu tiên xử lý, thường được đánh giá từ khả năng xảy ra và mức độ ảnh hưởng. Risk có thể vẫn còn sau khi áp dụng mitigation.

### Ví dụ cụ thể

Orders API có endpoint `/orders/1001`. Tài sản cần bảo vệ là thông tin đơn hàng của khách hàng.

Mối đe dọa là khách A đổi URL thành `/orders/1002` để xem đơn của khách B. Điểm yếu là API chỉ kiểm tra người dùng đã đăng nhập mà không kiểm tra người sở hữu đơn.

Biện pháp giảm thiểu là luôn lọc đơn hàng theo cả `order_id` và `user_id`. Nhóm viết thêm test để xác nhận khách A không thể đọc hoặc sửa đơn của khách B.

## 2 Authentication, OAuth 2.0 và OpenID Connect

### Các khái niệm

- **Authentication:** quá trình xác minh một người hoặc hệ thống là ai, thường bằng mật khẩu, MFA hoặc Identity Provider. Kết quả là một danh tính đã được xác thực, chưa có nghĩa danh tính đó được phép làm mọi việc.
- **Authorization:** quá trình kiểm tra danh tính đã xác thực được phép thực hiện hành động nào trên tài nguyên nào. Kiểm tra này cần diễn ra ở mỗi chức năng quan trọng.
- **OAuth 2.0:** framework ủy quyền cho phép ứng dụng nhận quyền truy cập có giới hạn mà không cần lấy mật khẩu của người dùng. OAuth 2.0 chủ yếu giải quyết cấp quyền, không tự định nghĩa đầy đủ việc đăng nhập.
- **OpenID Connect (OIDC):** lớp xác thực danh tính xây dựng trên OAuth 2.0. OIDC bổ sung quy tắc và ID token để ứng dụng biết người dùng đã đăng nhập là ai.
- **Identity Provider:** dịch vụ chịu trách nhiệm đăng nhập, xác minh danh tính và phát hành token, ví dụ Google, Microsoft Entra ID hoặc Amazon Cognito.
- **Access token:** credential có thời hạn được client gửi đến API. API phải xác minh token và quyền đi kèm trước khi xử lý request.
- **ID token:** token mô tả kết quả đăng nhập và danh tính người dùng cho ứng dụng client. ID token không nên được tùy ý dùng thay access token để gọi API.
- **Scope:** phạm vi quyền được cấp cho access token, ví dụ `orders:read` hoặc `orders:write`. Scope cho phép gọi loại chức năng nào nhưng vẫn cần kiểm tra quyền trên đơn hàng cụ thể.

### Ví dụ cụ thể

Người dùng chọn “Đăng nhập với Google”. Google xác thực người dùng rồi trả kết quả đăng nhập cho ứng dụng.

Ứng dụng nhận ID token để biết người dùng là `an@example.com`. Khi gọi Orders API, ứng dụng gửi access token có scope `orders:read`.

Orders API kiểm tra chữ ký, thời hạn, issuer, audience và scope của access token. Token chỉ có `orders:read` được xem đơn hàng nhưng không được gọi endpoint xóa đơn.

## 3 Authorization trong Django

### Các khái niệm

- **Permission:** quyền thực hiện một hành động cụ thể, ví dụ `order.view`, `order.create` hoặc `order.delete`. Permission nên được kiểm tra ở backend thay vì chỉ ẩn nút trên giao diện.
- **Role:** tập hợp nhiều permission theo trách nhiệm công việc, ví dụ customer, support hoặc admin. Gán role giúp quản lý quyền dễ hơn nhưng role quá rộng có thể vi phạm least privilege.
- **Object-level authorization:** kiểm tra quyền trên từng tài nguyên cụ thể sau khi biết loại hành động được phép, ví dụ user có được xem chính đơn hàng số 1001 hay không.
- **Least privilege:** chỉ cấp quyền tối thiểu cần thiết, trong đúng phạm vi và thời gian cần dùng. Khi nhiệm vụ thay đổi, quyền không còn cần thiết nên được thu hồi.
- **Default deny:** mặc định từ chối nếu không có quy tắc cho phép rõ ràng. Cách này tránh việc chức năng mới vô tình được mở quyền.
- **Tenant:** một khách hàng hoặc tổ chức có vùng dữ liệu riêng trong hệ thống dùng chung. Mọi truy vấn phải giữ đúng ranh giới tenant để tránh lộ dữ liệu chéo.


### Ví dụ cụ thể

Khách A gửi request `GET /orders/1001`. Django không chỉ kiểm tra `request.user.is_authenticated` mà còn tìm đơn bằng cả ID và chủ sở hữu:

```python
order = get_object_or_404(
    Order,
    id=1001,
    owner=request.user,
)
```

Nếu đơn 1001 thuộc khách A, API trả dữ liệu. Nếu đơn thuộc khách B, truy vấn không tìm thấy và API trả 404.

Nhân viên support có thể được cấp quyền xem đơn để hỗ trợ khách hàng, nhưng không được xóa đơn. Mọi lần nhân viên support truy cập dữ liệu nhạy cảm cần được ghi audit log.

## 4 IAM và Least Privilege

### Các khái niệm

- **IAM (Identity and Access Management):** hệ thống quản lý ai hoặc workload nào được đăng nhập và được thực hiện hành động gì trên tài nguyên cloud.
- **Policy:** tài liệu gồm các quy tắc allow hoặc deny, chỉ rõ action, resource và đôi khi có thêm condition. Có một dòng allow không có nghĩa request chắc chắn được phép nếu còn policy deny khác.
- **Role:** danh tính có một tập quyền nhưng không chứa credential dài hạn cố định. User hoặc workload có thể assume role để nhận session tạm thời.
- **Temporary credential:** access key, secret và session token có thời hạn ngắn. Khi hết hạn, credential tự mất hiệu lực nên giảm rủi ro nếu bị lộ.
- **MFA:** bước xác minh bổ sung ngoài mật khẩu, ví dụ mã từ ứng dụng xác thực hoặc security key. MFA đặc biệt quan trọng với tài khoản có quyền cao.
- **Least privilege:** cấp đúng action trên đúng resource và chỉ trong thời gian cần thiết, thay vì dùng quyền `*` trên toàn bộ account.

### Ví dụ cụ thể

Orders API cần đọc Secret chứa chuỗi kết nối database và đọc hóa đơn trong một S3 bucket. IAM role của ứng dụng chỉ được cấp hai quyền này.

Role không được phép xóa S3 bucket, tạo IAM user hoặc thay đổi quyền của chính nó. Nếu Orders API bị tấn công, kẻ tấn công cũng bị giới hạn bởi các quyền hẹp của role.

Pipeline triển khai dùng credential tạm thời trong 30 phút thay vì lưu AWS access key lâu dài trong GitHub Secrets.

## 5 Encryption và Hashing

### Các khái niệm

- **Encryption:** biến plaintext thành ciphertext không thể đọc trực tiếp và có thể giải mã lại bằng key phù hợp. Encryption bảo vệ dữ liệu nhưng không tự quyết định ai có quyền xem dữ liệu sau khi giải mã.
- **Encryption in transit:** bảo vệ dữ liệu khi truyền giữa các hệ thống, thường bằng HTTPS/TLS. Client vẫn phải xác minh certificate và đúng hostname của server.
- **Encryption at rest:** bảo vệ dữ liệu đang lưu trong database, ổ đĩa, backup hoặc object storage. Nó hữu ích khi thiết bị hoặc file lưu trữ bị truy cập trái phép.
- **Encryption key:** giá trị dùng trong quá trình mã hóa hoặc giải mã. Key cần được lưu tách khỏi dữ liệu, giới hạn quyền sử dụng và có kế hoạch rotation.
- **KMS (Key Management Service):** dịch vụ tạo, lưu và kiểm soát việc sử dụng encryption key. KMS thường ghi audit mỗi lần key được dùng nhưng ứng dụng vẫn cần IAM permission phù hợp.
- **Hashing:** tạo giá trị một chiều từ dữ liệu, phù hợp để kiểm tra mật khẩu mà không cần lưu mật khẩu gốc. Hashing khác encryption vì không có key để giải mã ngược.
- **Salt:** giá trị ngẫu nhiên riêng cho mỗi mật khẩu trước khi hashing. Salt làm hai người dùng có cùng mật khẩu vẫn nhận hai hash khác nhau và gây khó khăn cho việc dùng bảng hash có sẵn.

### Ví dụ cụ thể

Browser gọi Orders API qua HTTPS nên thông tin đơn hàng được mã hóa trên đường truyền. Database và S3 bucket cũng bật mã hóa khi lưu dữ liệu.

Mật khẩu người dùng không được lưu bằng dạng text hoặc mã hóa có thể giải ngược. Django lưu password hash kèm salt và dùng password hasher phù hợp để kiểm tra mật khẩu khi đăng nhập.

Mã hóa không thay thế authorization. Nếu Orders API có quyền giải mã nhưng trả nhầm đơn của khách B cho khách A, dữ liệu vẫn bị lộ.

## 6 Secret Management

### Các khái niệm

- **Secret:** dữ liệu nhạy cảm trao quyền truy cập hoặc dùng để ký, ví dụ mật khẩu, API token và private key. Người có secret thường có thể hành động như chủ thể hợp lệ nên secret phải được bảo vệ chặt chẽ.
- **Secret store:** hệ thống chuyên lưu, mã hóa, phân quyền và phân phối secret, ví dụ AWS Secrets Manager. Dùng secret store không loại bỏ nhu cầu giới hạn IAM permission.
- **Secret rotation:** thay secret cũ bằng secret mới mà không làm gián đoạn consumer. Quy trình phải bao gồm cập nhật ứng dụng và xác nhận credential mới hoạt động.
- **Revocation:** làm secret cũ mất hiệu lực tại hệ thống xác thực. Xóa secret khỏi file hoặc secret store chưa chắc đã thu hồi quyền sử dụng của nó.
- **Redaction:** thay toàn bộ hoặc một phần giá trị nhạy cảm bằng ký hiệu che trước khi ghi log hoặc hiển thị, ví dụ `token=***`.

### Ví dụ cụ thể

Orders API lấy `DATABASE_URL` từ AWS Secrets Manager khi khởi động. Giá trị thật không xuất hiện trong source code, Docker image hoặc log.

Khi đổi mật khẩu database, nhóm thực hiện các bước:

1. Tạo mật khẩu mới trong database.
2. Cập nhật giá trị trong secret store.
3. Khởi động lại các instance Orders API để nhận mật khẩu mới.
4. Xác nhận ứng dụng kết nối thành công.
5. Thu hồi mật khẩu cũ.

Nếu một API token bị commit vào Git, chỉ xóa dòng chứa token là chưa đủ. Nhóm phải thu hồi token cũ và tạo token mới.

## 7 Software Supply Chain Security

### Các khái niệm

- **Software supply chain:** toàn bộ đường đi từ source code, dependency, công cụ build và pipeline đến artifact chạy ở production. Một mắt xích bị chiếm quyền có thể làm artifact cuối cùng không còn đáng tin cậy.
- **Dependency:** thư viện hoặc package bên ngoài mà ứng dụng sử dụng. Dependency cần được khóa version, cập nhật và đánh giá vì code của nó chạy cùng quyền với ứng dụng.
- **SAST:** phân tích source code mà không cần chạy ứng dụng để tìm mẫu code có khả năng không an toàn. Kết quả cần được review vì công cụ có thể báo sai hoặc bỏ sót logic nghiệp vụ.
- **SCA:** đối chiếu dependency và version với cơ sở dữ liệu lỗ hổng đã biết. SCA không phát hiện được mọi lỗ hổng chưa công bố.
- **Secret scanning:** tìm pattern giống credential hoặc key trong source code và Git history. Nếu tìm thấy secret thật, vẫn phải thu hồi hoặc rotation secret đó.
- **Image scanning:** kiểm tra package hệ điều hành và runtime nằm trong container image. Kết quả chỉ đúng với image digest đã được quét.
- **SBOM:** danh sách thành phần, dependency và version có trong artifact. SBOM giúp nhanh chóng xác định hệ thống nào bị ảnh hưởng khi có lỗ hổng mới.
- **Artifact signature:** chữ ký số gắn với artifact để xác minh nguồn phát hành và tính toàn vẹn. Hệ thống triển khai phải thực sự kiểm tra chữ ký trước khi tin artifact.

### Ví dụ cụ thể

Pipeline của Orders API thực hiện lần lượt:

1. Chạy unit test và SAST trên source code.
2. Kiểm tra dependency trong `requirements.txt`.
3. Quét secret bị commit nhầm.
4. Build image `orders-api:1.2`.
5. Quét lỗ hổng trong image.
6. Tạo SBOM và ký image.
7. Chỉ triển khai đúng image digest đã được kiểm tra.

Nếu dependency `example-lib` có lỗ hổng nghiêm trọng, pipeline chặn release cho đến khi dependency được nâng cấp hoặc có ngoại lệ được phê duyệt với thời hạn rõ ràng.

## 8 Security Testing

### Các khái niệm

- **Security test:** kiểm thử xác nhận security control hoạt động đúng khi request hợp lệ và từ chối đúng khi có hành vi không được phép. Test cần bao phủ cả kết quả và việc dữ liệu không bị thay đổi ngoài ý muốn.
- **Unit test:** kiểm tra nhanh một hàm, permission hoặc quy tắc bảo mật riêng lẻ trong phạm vi hẹp, thường không cần hệ thống bên ngoài.
- **Integration test:** kiểm tra nhiều thành phần thật hoạt động cùng nhau, ví dụ authentication middleware, Django view và database filter.
- **DAST:** kiểm tra ứng dụng đang chạy từ góc nhìn bên ngoài bằng cách gửi request, không cần đọc source code. DAST có thể tìm lỗi runtime nhưng khó hiểu đầy đủ logic nghiệp vụ.
- **Penetration testing:** hoạt động mô phỏng tấn công do người có chuyên môn thực hiện trong phạm vi và thời gian được cho phép, nhằm tìm chuỗi khai thác thực tế.
- **Regression test:** test được thêm sau khi sửa lỗi để bảo đảm lỗi cũ không xuất hiện trở lại trong các lần thay đổi sau.
- **Negative test:** chủ động gửi input sai, token hết hạn hoặc hành động vượt quyền để xác nhận hệ thống từ chối an toàn.

### Ví dụ cụ thể

Nhóm tạo hai user là khách A và khách B, mỗi người có một đơn hàng riêng. Các test quan trọng gồm:

- Khách A xem đơn của mình: API trả 200.
- Khách A xem đơn của khách B: API trả 404 hoặc 403.
- Token hết hạn gọi API: API trả 401.
- Token chỉ có quyền đọc gọi API xóa đơn: API trả 403.
- Request tạo đơn thiếu CSRF token khi dùng session: request bị chặn.
- Payload vượt kích thước cho phép: API từ chối có kiểm soát.

Scanner không báo lỗi không có nghĩa hệ thống hoàn toàn an toàn. Logic nghiệp vụ như “khách nào sở hữu đơn nào” vẫn cần test do đội phát triển viết.

## 9 Audit Logging và Security Monitoring

### Các khái niệm

- **Audit log:** bản ghi phục vụ truy vết ai đã làm gì, trên tài nguyên nào, vào lúc nào và kết quả ra sao. Audit log cần khó bị sửa hoặc xóa bởi chính người đang được theo dõi.
- **Security event:** sự kiện có ý nghĩa bảo mật, ví dụ đăng nhập thất bại, thay đổi quyền hoặc truy cập secret. Một event riêng lẻ chưa chắc là incident.
- **Monitoring:** thu thập và phân tích log, metric hoặc event để phát hiện hành vi bất thường theo thời gian.
- **Alert:** thông báo được tạo khi dữ liệu giám sát khớp điều kiện cần điều tra. Alert tốt phải có mức độ, người nhận và hướng xử lý rõ ràng.
- **Redaction:** che mật khẩu, token và dữ liệu nhạy cảm trước khi ghi log; việc che nên diễn ra càng gần nguồn tạo log càng tốt.
- **Correlation ID:** mã duy nhất được truyền qua các thành phần để nối log của cùng một request hoặc workflow, giúp điều tra nhanh hơn.

### Ví dụ cụ thể

Khi nhân viên support xem đơn hàng 1001, hệ thống ghi:

- Người thực hiện: `support-user-7`.
- Hành động: `order.view`.
- Tài nguyên: `order-1001`.
- Thời gian và correlation ID.

Log không chứa access token, mật khẩu hoặc toàn bộ thông tin thẻ. Nếu một tài khoản thử truy cập hàng trăm order ID trong một phút, hệ thống tạo alert để đội vận hành kiểm tra.

## 10 Incident Response

### Các khái niệm

- **Security incident:** sự kiện đã hoặc có khả năng làm ảnh hưởng đến tính bí mật, toàn vẹn hoặc khả dụng của hệ thống và cần phản ứng có phối hợp.
- **Detection:** phát hiện và xác nhận dấu hiệu sự cố qua alert, log, threat intelligence hoặc báo cáo của người dùng. Phát hiện sớm giúp giảm phạm vi ảnh hưởng.
- **Containment:** hành động tạm thời nhằm ngăn sự cố lan rộng, ví dụ khóa credential, cô lập workload hoặc tắt một endpoint bị khai thác.
- **Eradication:** loại bỏ nguyên nhân và khả năng truy cập của kẻ tấn công, ví dụ vá lỗ hổng, xóa persistence và thu hồi credential bị lộ.
- **Recovery:** khôi phục dịch vụ từ trạng thái sạch, kiểm tra hoạt động bình thường và tăng giám sát để phát hiện sự cố quay lại.
- **Postmortem:** phân tích timeline, nguyên nhân gốc, tác động và khoảng trống kiểm soát sau sự cố, sau đó gán người phụ trách và thời hạn cho hành động cải thiện.

### Ví dụ cụ thể

Đội vận hành phát hiện khách A có thể xem đơn của khách B bằng cách đổi ID trên URL:

1. Tạm khóa endpoint bị lỗi hoặc thêm biện pháp chặn có phạm vi nhỏ.
2. Kiểm tra audit log để xác định dữ liệu nào đã bị truy cập.
3. Sửa truy vấn để luôn lọc theo người sở hữu và tenant.
4. Viết regression test với hai khách hàng khác nhau.
5. Triển khai bản sửa và xác nhận truy cập chéo bị từ chối.
6. Thông báo và xử lý ảnh hưởng theo quy trình của tổ chức.
7. Thực hiện postmortem để ngăn lỗi tương tự ở các endpoint khác.
## 11 Quiz: Security Engineering

### Câu hỏi

1. Threat modeling giúp nhóm làm gì?
   - A. Nhận diện và ưu tiên mối đe dọa
   - B. Tăng tốc CPU
   - C. Thiết kế CSS
   - D. Tạo database
2. Authentication trả lời câu hỏi nào?
   - A. Bạn là ai?
   - B. Bạn được phép làm gì?
   - C. Server ở đâu?
   - D. File lớn bao nhiêu?
3. Authorization trả lời câu hỏi nào?
   - A. Chủ thể được phép làm gì?
   - B. Mật khẩu dài bao nhiêu?
   - C. Request mất bao lâu?
   - D. Server dùng CPU nào?
4. Least privilege nghĩa là gì?
   - A. Chỉ cấp quyền tối thiểu cần thiết
   - B. Cấp quyền admin cho mọi người
   - C. Không ghi audit log
   - D. Dùng chung tài khoản
5. Mật khẩu nên được bảo vệ chủ yếu bằng cách nào?
   - A. Hash bằng thuật toán dành cho mật khẩu
   - B. Mã hóa base64
   - C. Lưu plain text
   - D. Ghi vào log
6. Secret không nên được đặt ở đâu?
   - A. Trong source code được commit
   - B. Trong secret manager
   - C. Trong kho bí mật có kiểm soát
   - D. Trong biến được cấp lúc chạy
7. Kiểm tra dependency giúp phát hiện điều gì?
   - A. Thư viện có lỗ hổng đã biết
   - B. Màu giao diện sai
   - C. DNS chậm
   - D. User quên tên
8. SAST phân tích chủ yếu thứ gì?
   - A. Source code hoặc artifact mà không cần chạy ứng dụng
   - B. Chỉ traffic production
   - C. Chỉ database backup
   - D. Chỉ hóa đơn cloud
9. Audit log nên ghi điều gì?
   - A. Ai thực hiện hành động quan trọng nào và khi nào
   - B. Mọi mật khẩu
   - C. Mọi access token
   - D. Private key
10. Khi xảy ra sự cố bảo mật, ưu tiên ban đầu là gì?
    - A. Hạn chế tác động và bảo toàn bằng chứng
    - B. Xóa mọi log ngay
    - C. Công khai mọi secret
    - D. Bỏ qua cảnh báo

### Đáp án

1. **A**; 2. **A**; 3. **A**; 4. **A**; 5. **A**; 6. **A**; 7. **A**; 8. **A**; 9. **A**; 10. **A**.
