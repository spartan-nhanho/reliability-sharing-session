# Chuẩn bị trước buổi: Reliability (queues, outbox, observability)

Chào các bạn! Buổi chia sẻ dài khoảng 2 tiếng, nói về cách xây hệ thống **không mất dữ liệu, không làm sai, và biết ngay khi có sự cố**.

Cả buổi xoay quanh một ví dụ: **một cửa hàng online nhỏ**. Khách bấm "Đặt hàng", hệ thống trừ tiền, lưu đơn hàng và gửi email xác nhận. Mình sẽ làm hỏng hệ thống đó 9 lần, rồi cùng nhau sửa từng chỗ.

Tài liệu này liệt kê các **thuật ngữ nền tảng** mà buổi học coi như các bạn đã biết. Đọc mất khoảng 15–20 phút. Thuật ngữ nào bạn biết rồi thì lướt qua. Không cần thuộc lòng, chỉ cần hiểu ý chính là theo kịp bài.

- **★** là thuật ngữ quan trọng nhất, nên đọc kỹ.
- Thuật ngữ không có ★ thì chỉ cần biết sơ qua.

Những khái niệm "chính" của buổi (outbox, circuit breaker, DLQ…) **sẽ được giảng trong buổi**, nên các bạn không cần đọc trước. Danh sách ở cuối tài liệu.

---

## 1. Web và API

| Thuật ngữ | Giải thích ngắn |
| --- | --- |
| **★ Client / server** | Client là bên gửi yêu cầu (app mobile, trình duyệt). Server là bên nhận và xử lý (backend của mình). |
| **★ API / endpoint** | API là "cửa" để client gọi vào backend. Endpoint là một cửa cụ thể, ví dụ `POST /orders` là tạo đơn hàng mới. |
| **★ Request / response** | Request là lời gọi client gửi đi. Response là câu trả lời server gửi về. Có khi server làm xong rồi nhưng response bị mất giữa đường, và client tưởng là chưa có gì xảy ra. |
| **HTTP method** | Loại hành động: `GET` để đọc dữ liệu, `POST` để tạo mới. |
| **★ HTTP status code** | Con số trong response cho biết kết quả. **2xx** là thành công (`201 Created` = đã tạo, `202 Accepted` = đã nhận, sẽ xử lý sau). **4xx** là lỗi do request (`400` = dữ liệu sai, `429` = gửi nhiều quá). **5xx** là lỗi phía server (`503` = server đang quá tải). |
| **Header** | Thông tin phụ đi kèm request, ngoài nội dung chính, có dạng `Tên: giá trị`. Ví dụ `Retry-After: 5` nghĩa là "5 giây nữa thử lại". |
| **JSON** | Định dạng dữ liệu dạng text mà API và log hay dùng, ví dụ `{"order_id": 1042}`. |
| **★ Latency (độ trễ)** | Thời gian từ lúc gửi request tới lúc nhận được trả lời. Ví dụ 200 ms (mili giây). |
| **★ Timeout** | Thời gian tối đa chịu chờ một lời gọi. Quá thời gian đó thì bỏ cuộc và coi như lỗi. Ví dụ: app chờ 10 giây, không có trả lời thì báo lỗi. |
| **★ Retry** | Thử gửi lại một request bị lỗi hoặc bị timeout. |

## 2. Database

| Thuật ngữ | Giải thích ngắn |
| --- | --- |
| **★ Database / Postgres** | Nơi lưu dữ liệu bền vững: tắt máy cũng không mất. Postgres là loại database mình dùng. Dữ liệu nằm trong các **bảng** (table); mỗi bản ghi là một **dòng** (row). |
| **★ Transaction** | Một nhóm thao tác database được coi như **một khối**: hoặc tất cả cùng thành công, hoặc như chưa có gì xảy ra. Ví dụ chuyển tiền: trừ tài khoản A và cộng tài khoản B phải đi cùng nhau. |
| **★ Commit / rollback** | Commit là "chốt" transaction; sau commit, dữ liệu chắc chắn đã được lưu. Rollback là hủy, quay về như lúc chưa làm. |
| **Unique constraint** | Luật của database: không cho phép hai dòng trùng giá trị ở một cột (hoặc một nhóm cột). Insert trùng sẽ bị từ chối. |
| **Lock (khóa)** | Khi một transaction đang sửa một dòng, database khóa dòng đó lại. Transaction khác muốn sửa dòng đó phải chờ. |
| **★ Connection pool** | Một nhóm kết nối tới database được mở sẵn và dùng chung. Mỗi truy vấn mượn một kết nối rồi trả lại. Pool có 20 kết nối nghĩa là tối đa 20 truy vấn chạy cùng lúc; hết kết nối thì phải xếp hàng chờ. |

## 3. Code backend chạy thế nào

| Thuật ngữ | Giải thích ngắn |
| --- | --- |
| **Process (tiến trình)** | Một chương trình đang chạy. Process chết thì mọi thứ nằm trong bộ nhớ (RAM) của nó mất theo. |
| **★ Thread / thread pool** | Thread là một "luồng" xử lý công việc. Thread pool là một nhóm thread cố định dùng chung; mỗi việc chiếm một thread, hết thread thì việc mới phải chờ. Giống quầy giao dịch ở ngân hàng: có 5 quầy thì phục vụ được 5 người cùng lúc, người thứ 6 phải xếp hàng. |
| **Exception / try-catch** | Exception là lỗi được "ném" ra khi có sự cố. `try/catch` bắt lỗi lại để xử lý, thay vì để chương trình dừng hẳn. |
| **Coroutine / `launch { }`** | Cách chạy một đoạn code "ngầm" song song trong Kotlin. Nó chỉ sống trong bộ nhớ của process. |
| **★ Background worker** | Một vòng lặp chạy nền, liên tục lấy việc ra làm, không gắn với request nào của người dùng. |
| **Log / stack trace** | Log là các dòng ghi lại chuyện đã xảy ra khi code chạy. Stack trace là danh sách các hàm đang chạy tại lúc lỗi xảy ra, giúp tìm ra dòng code gây lỗi. |

## 4. Hạ tầng (nơi code chạy)

| Thuật ngữ | Giải thích ngắn |
| --- | --- |
| **AWS** | Dịch vụ cloud của Amazon: thuê máy chủ, database, queue… thay vì tự mua máy. |
| **Container** | Gói app cùng mọi thứ nó cần để chạy giống nhau ở mọi nơi. |
| **★ Kubernetes / pod / node** | Kubernetes là hệ thống quản lý việc chạy container. **Pod** là một bản sao của service đang chạy; 10 pod là 10 bản chạy song song, chia nhau traffic. **Node** là máy chạy các pod. Kubernetes có thể kill pod bất cứ lúc nào: khi deploy, khi pod dùng quá bộ nhớ, khi dời pod sang máy khác. |
| **★ Deploy** | Đưa phiên bản code mới lên chạy thật. Với Kubernetes: bật pod mới rồi tắt pod cũ, lần lượt từng cái. Nghĩa là **mỗi lần deploy, mọi pod cũ đều bị tắt**. |
| **★ Production** | Môi trường thật mà khách hàng đang dùng. Khác với máy local hay môi trường staging dùng để thử. |
| **Load balancer** | Bộ phân phối request tới các pod. |
| **Health check** | Endpoint (ví dụ `/health`) mà Kubernetes gọi định kỳ để hỏi "pod còn sống không?". Nhiều lần không trả lời thì Kubernetes khởi động lại pod. |
| **Terraform** | Công cụ mô tả hạ tầng (queue, database…) bằng code. |

## 5. Giao tiếp bất đồng bộ

Phần này quan trọng nhất, vì cả buổi học xây trên nó.

| Thuật ngữ | Giải thích ngắn |
| --- | --- |
| **★ Đồng bộ vs bất đồng bộ (sync vs async)** | **Đồng bộ:** gọi rồi đứng chờ kết quả, như gọi điện thoại. **Bất đồng bộ:** gửi yêu cầu rồi đi làm việc khác, kết quả đến sau, như gửi tin nhắn. |
| **★ Queue (hàng đợi) / message** | Queue là nơi một bên bỏ **message** (thông điệp mô tả một việc cần làm) vào, và bên khác lấy ra xử lý sau. Giống hộp thư: người gửi bỏ thư vào rồi đi làm việc khác, người nhận rảnh thì lấy thư ra. Hai bên không phải chờ nhau. |
| **★ Producer / consumer** | Producer là bên gửi message vào queue. Consumer là bên nhận message ra và làm việc. |
| **★ SQS** | Amazon Simple Queue Service: dịch vụ queue của AWS mà mình dùng. |
| **★ Event (sự kiện)** | Một message thông báo "điều X đã xảy ra", ví dụ `OrderCreated` (đơn hàng vừa được tạo). Bên khác nghe event để làm phần việc của họ, ví dụ gửi email. |
| **★ Vendor / dịch vụ bên thứ ba** | Dịch vụ của công ty khác mà mình gọi qua API, ví dụ cổng thanh toán, dịch vụ gửi email. Mình không kiểm soát được khi nào nó chậm hay sập. |

## 6. Vận hành và theo dõi hệ thống

| Thuật ngữ | Giải thích ngắn |
| --- | --- |
| **Metric** | Con số được đo liên tục theo thời gian, ví dụ số request mỗi giây, tỉ lệ lỗi, độ trễ. |
| **★ Dashboard** | Màn hình gồm các biểu đồ metric, để nhìn tình trạng hệ thống. |
| **★ Percentile: p50, p99** | p99 = 200 ms nghĩa là 99% request nhanh hơn 200 ms, chỉ 1% chậm hơn. Mình dùng percentile thay vì trung bình, vì trung bình hay che mất những người dùng phải chờ rất lâu. |
| **★ Alert / on-call** | Alert là luật tự động: "nếu số liệu X vượt ngưỡng thì báo người trực". On-call là người được phân công nhận alert và xử lý sự cố, kể cả nửa đêm. |
| **Incident (sự cố)** | Khi hệ thống chạy sai hoặc chậm, ảnh hưởng tới khách hàng. |
| **Repo / PR / code review** | Repo là kho mã nguồn (git). PR (pull request) là đề xuất đưa một thay đổi vào code chính. Code review là đồng nghiệp đọc và góp ý trước khi merge. |

---

## Những thứ sẽ học trong buổi (không cần đọc trước)

Các bạn sẽ gặp những từ này trên slide. Đừng lo, mỗi từ đều được giải thích kèm hình minh họa:

- **Dual write**, **transactional outbox**, **relay**
- **Idempotent**, **idempotency key**
- **At-least-once delivery**, **dedupe** (loại trùng), **ack**
- **Visibility timeout**, **DLQ** (dead-letter queue), **poison message**
- **Backoff**, **jitter**, **circuit breaker**, **bulkhead**, **Little's law**
- **Backpressure**, **graceful shutdown**
- **Log có cấu trúc**, **trace**, **outbox lag**

---

## Tự kiểm tra nhanh

Thử trả lời 5 câu này trong đầu trước khi vào buổi. Không cần đúng hết; câu nào chưa chắc thì đọc lại phần tương ứng ở trên.

1. Một transaction gồm 2 lệnh insert. Lệnh thứ hai lỗi. Sau rollback, lệnh insert thứ nhất còn trong database không?
2. App mobile có timeout 10 giây. Server mất 11 giây để xử lý xong. Server có làm xong việc không? App nghĩ gì?
3. Gửi một message vào queue khác gọi API trực tiếp ở điểm nào?
4. Pool có 20 kết nối database và đang dùng hết cả 20. Request thứ 21 thì sao?
5. Vì sao mỗi lần deploy lại có thể làm mất công việc đang chạy dở trong một pod?

<details>
<summary>Xem đáp án</summary>

1. **Không.** Transaction là một khối: lệnh nào lỗi thì cả khối bị hủy.
2. **Có, server vẫn làm xong**, nhưng app đã bỏ cuộc ở giây thứ 10 nên tưởng là lỗi. Nếu app gửi lại, server có thể làm việc đó **hai lần**. Đây là một vấn đề sẽ học trong buổi.
3. Gọi API trực tiếp thì phải **đứng chờ** bên kia trả lời. Gửi vào queue thì gửi xong là đi làm việc khác, consumer **xử lý sau**.
4. Request thứ 21 phải **xếp hàng chờ** tới khi có kết nối được trả lại, hoặc bị timeout.
5. Vì deploy **tắt pod cũ**, và mọi thứ chỉ nằm trong bộ nhớ của pod (ví dụ một coroutine đang chạy) sẽ mất theo.

</details>

Có gì chưa rõ thì cứ ghi lại và hỏi trong buổi nhé. *Hỏi bất kỳ ai, bất cứ điều gì, đừng giấu rồi đoán.*
