# Script thuyết trình tiếng Việt (bản Story Mode)

Script đi kèm file **`Reliability Engineering Session Slides - Story Mode.html`** (47 slide). Mỗi mục bên dưới ứng với đúng một slide; số slide khớp với số ở góc phải dưới của deck (ví dụ `07 / 49`).

## Cách đọc script

- **⏱** thời lượng của slide, và giờ bạn nên rời slide đó (tính từ lúc bắt đầu).
- **[→]** nhấn mũi tên phải (hoặc click) tại đúng chỗ đó trong lời nói. Slide có nhiều bước sẽ hiện dần từng phần.
- **Hỏi:** câu hỏi cho cả phòng. Hỏi xong thì *im lặng đếm thầm đến 5* rồi mới gợi ý; intern cần thời gian để dám trả lời.
- **Nếu bị hỏi:** câu trả lời chuẩn bị sẵn cho các câu hỏi hay gặp.
- **Khái niệm cần biết:** khung trích dẫn ngay dưới tiêu đề slide, giải thích ngắn từng khái niệm *lần đầu* xuất hiện. Đọc khi chuẩn bị, không cần đọc to. Bảng tra cứu ở cuối file.
- Thuật ngữ kỹ thuật (outbox, relay, consumer, commit, ack, timeout…) giữ nguyên tiếng Anh, vì slide viết bằng tiếng Anh và đây cũng là từ các bạn sẽ gặp trong code.
- Không cần đọc nguyên văn. Đọc lướt trước, rồi nói theo cách của bạn. Những câu in *nghiêng* là câu nên nói đúng như vậy.

## Điều hướng trong deck

| Thao tác | Tác dụng |
| --- | --- |
| `→` / `←` | Bước tiếp / lùi lại |
| Click chấm đỏ trên sơ đồ, hoặc thẻ "dive in" | Nhảy vào vấn đề đó |
| Click chấm xanh, hoặc số 1–9 ở góc phải trên | Xem lại một vấn đề bất kỳ |
| `M`, hoặc click nhãn "⌂ map" | Quay về sơ đồ của vấn đề đang nói |
| `N` | Hiện speaker notes |
| `F` | Toàn màn hình |

## Lịch trình 2 tiếng

| Phần | Slide | Phút | Giờ kết thúc |
| --- | --- | --- | --- |
| Mở đầu, bí ẩn 3 giờ sáng, cửa hàng version 0 | 1–5 | 10 | 0:10 |
| Vấn đề 1 · Mất email | 6–10 | 13 | 0:23 |
| Vấn đề 2 · Trừ tiền 2 lần | 11–14 | 8 | 0:31 |
| Vấn đề 3 · Relay giữ lock | 15–18 | 8 | 0:39 |
| Vấn đề 4 · Message giao 2 lần | 19–22 | 9 | 0:48 |
| Vấn đề 5 · Consumer crash, poison message | 23–26 | 8 | 0:56 |
| **Nghỉ giải lao** | — | 5 | 1:01 |
| Vấn đề 6 · Vendor chậm | 27–33 | 18 | 1:19 |
| Vấn đề 7 · Flash sale | 34–36 | 5 | 1:24 |
| Vấn đề 8 · Deploy | 37–39 | 5 | 1:29 |
| Vấn đề 9 · Hỏng mà không ai biết | 40–43 | 12 | 1:41 |
| Tổng kết, checklist, Q&A | 44–47 | 19 | 2:00 |

**Nếu bị trễ giờ**, cắt theo thứ tự: slide 17 (lock time) chỉ nói 1 câu → slide 32 (jitter) nói 1 câu → slide 44 lướt nhanh → rút Q&A.

**Không được cắt:** slide 7–9 (dual write, outbox), 25 (thứ tự ack), 28 (bốn công cụ), 41 (observability và lời giải 3 giờ sáng).

## Trước buổi trình bày

- [ ] Mở deck bằng Chrome, nhấn `F`. Thử bấm chấm đỏ trên slide 5 và phím `M` một lần cho quen.
- [ ] Chuẩn bị bảng trắng hoặc giấy để ghi các câu đoán ở slide 3.
- [ ] Thử kéo thanh trượt Little's law ở slide 30.

---

# PHẦN MỞ ĐẦU (0:00 – 0:10)

## Slide 01 / 47 · Title

⏱ 1 phút · rời slide lúc 0:01

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Reliability (độ tin cậy):** Khả năng hệ thống vẫn làm đúng việc khi có thứ trục trặc: máy chết, mạng chậm, dịch vụ khác lỗi. Không phải "không bao giờ hỏng", mà là hỏng thì không mất dữ liệu, không làm sai, và mình biết ngay.

Chào các bạn. Mình là [tên], làm ở team [team]. Hôm nay mình nói về **reliability**, tức là độ tin cậy của hệ thống. Cụ thể là ba câu hỏi: công việc rời khỏi request thế nào mà không bị mất; service xử lý ra sao khi một hệ thống mình phụ thuộc bị chậm; và làm sao biết chuyện gì đã xảy ra lúc 3 giờ sáng.

Có câu hỏi thì cứ hỏi bất cứ lúc nào nhé, cắt ngang mình cũng được. Hôm nay không có câu hỏi nào ngớ ngẩn cả: nếu bạn thắc mắc, chắc chắn còn người khác cũng đang thắc mắc giống bạn.

## Slide 02 / 47 · Why this session

⏱ 2 phút · rời slide lúc 0:03

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Dependency (hệ thống phụ thuộc):** Bất cứ thứ gì service của mình phải gọi tới mới làm xong việc: database, queue, payment vendor, email vendor.
> - **Vendor:** Dịch vụ của công ty khác mà mình dùng qua API, ví dụ cổng thanh toán, dịch vụ gửi email. Mình không kiểm soát được khi nào nó chậm hay sập.
> - **Production:** Môi trường thật mà khách hàng đang dùng, khác với máy local hay môi trường staging để thử.
> - **Pod:** Đơn vị chạy app trong Kubernetes. Hiểu đơn giản là một bản sao của service đang chạy. 10 pod là 10 bản chạy song song, chia nhau traffic.
> - **Deploy:** Đưa phiên bản code mới lên production. Với Kubernetes, deploy là bật pod mới rồi kill pod cũ, lần lượt từng cái (rolling deploy).

Các bạn đọc câu tiêu đề cùng mình: *phần lớn sự cố không phải do bug trong code, mà do code gặp một hệ thống phụ thuộc bị chậm mà nó chưa từng tính tới.*

Khi viết một tính năng, các bạn test trên laptop. Database trả lời trong 2 mili giây, dịch vụ email lúc nào cũng chạy, không có gì crash cả. Production thì khác: pod bị kill mỗi lần deploy, vendor có lúc chậm, mạng có lúc rớt. Code vẫn "đúng", chỉ là nó chưa bao giờ tính tới những chuyện đó.

Cách mình trình bày hôm nay hơi khác. Mình sẽ đưa ra một hệ thống rất đơn giản cho một cửa hàng online. Sau đó mình **làm hỏng nó chín lần**. Mỗi lần hỏng, mình đào sâu vào nguyên nhân, sửa nó, và thiết kế lớn dần lên. Cuối buổi là toàn bộ hệ thống hoàn chỉnh, checklist khi review code, và hỏi đáp.

## Slide 03 / 47 · 03:07 a.m.

⏱ 3 phút · rời slide lúc 0:06

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **On-call (trực):** Người được phân công nhận cảnh báo và xử lý sự cố, kể cả lúc nửa đêm.
> - **Dashboard:** Màn hình gồm các biểu đồ theo dõi tình trạng hệ thống theo thời gian.
> - **Alert / page:** Luật tự động: "nếu số liệu X vượt ngưỡng thì báo người trực". "Page" là alert đủ nghiêm trọng để gọi, đánh thức người trực.
> - **Queue (hàng đợi):** Nơi một bên bỏ message (thông điệp mô tả một việc cần làm) vào, và bên khác lấy ra xử lý sau. Giống hộp thư: người gửi bỏ thư vào rồi đi làm việc khác, người nhận rảnh thì lấy thư ra. Hai bên không phải chờ nhau.
> - **SQS:** Amazon Simple Queue Service: dịch vụ queue có sẵn của AWS mà team đang dùng.
> - **Độ sâu queue (queue depth):** Số message đang nằm trong queue chờ xử lý. Tăng dần nghĩa là bên xử lý không theo kịp. Gần 0 nghĩa là mọi thứ được xử lý kịp, *hoặc* là không có gì được gửi vào. Khả năng thứ hai chính là mấu chốt của bí ẩn này.
> - **Outbox:** Một bảng trong database chứa "những việc cần gửi đi". Slide 9 giải thích kỹ. Ở đây chỉ cần biết: số dòng outbox chưa gửi tăng lên nghĩa là có việc đang bị kẹt.

*Lúc vào slide: đường màu xanh vẽ ra, mốc deploy 02:10 hiện lên, rồi đường đỏ đi lên.*

Bây giờ là 3 giờ 7 phút sáng. Mọi dashboard đều xanh. Nhưng khách hàng bắt đầu nhắn: "Tôi không nhận được email xác nhận đơn hàng."

Nhìn biểu đồ nhé. Đường xanh là **độ sâu của hàng đợi SQS**, tức là số message đang chờ trong queue. Mình có alert cho nó. Cả đêm nó phẳng, gần bằng 0. Nhìn rất khỏe mạnh.

Lúc 2 giờ 10 có một lần deploy. Còn đường đỏ là số dòng trong một bảng tên `outbox` chưa được gửi đi. Sau lần deploy, nó bắt đầu tăng. Và không có alert nào cho nó.

**Hỏi:** *"Bạn đang trực on-call. Bạn sẽ kiểm tra gì đầu tiên?"* Nhận hai, ba câu trả lời, ghi lên bảng. **Đừng trả lời.** Các câu hay gặp: "email vendor bị sập", "deploy làm hỏng gì đó", "xem log". Câu nào cũng hợp lý.

**[→]** Không có alert nào kêu cả. Độ sâu queue nhìn hoàn hảo. Các bạn giữ câu hỏi này trong đầu nhé. Đến vấn đề số 9, các bạn sẽ tự giải thích được biểu đồ này.

## Slide 04 / 47 · The business

⏱ 2 phút · rời slide lúc 0:08

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **API / endpoint:** API là cửa để app gọi vào backend. Endpoint là một cửa cụ thể, ví dụ `POST /orders` là tạo order mới.
> - **HTTP status code:** Con số trong response cho biết kết quả. 2xx là thành công (201 Created = đã tạo). 4xx là lỗi do request (400 = dữ liệu gửi lên sai). 5xx là lỗi phía server (503 = đang quá tải, không phục vụ được).
> - **Kubernetes:** Hệ thống quản lý việc chạy container: bật pod, kill pod, restart pod hỏng, chia traffic cho các pod.
> - **Latency (độ trễ):** Thời gian từ lúc gửi request tới lúc nhận được trả lời. "200 ms" là một con số độ trễ.
> - **p99:** 99% request nhanh hơn con số này, chỉ 1% chậm hơn. Dùng để nói về những lần chậm nhất, vì con số trung bình hay che mất chúng.
> - **Timeout:** Thời gian tối đa chịu chờ một lời gọi. Quá thời gian đó thì bỏ cuộc và coi như lỗi.
> - **Retry:** Thử gửi lại một request bị lỗi hoặc bị timeout.
> - **Connection pool (pool kết nối DB):** Một nhóm kết nối tới database được mở sẵn và dùng chung. Mỗi truy vấn mượn một kết nối rồi trả lại. Pool 20 nghĩa là mỗi pod chạy được tối đa 20 truy vấn cùng lúc; hết kết nối thì phải xếp hàng chờ.

Đây là doanh nghiệp của chúng ta hôm nay: một cửa hàng online nhỏ. Chỉ có một nút quan trọng: **"Place order"**, đặt hàng.

Bên trái là phiên bản 0 của endpoint đặt hàng. Nó làm bốn việc: gọi **payment vendor** để trừ tiền thẻ, lưu order vào database, gửi email xác nhận qua **email vendor**, rồi trả về 201. Code này chạy tốt trên mọi laptop, qua mọi test, demo thì hoàn hảo.

Bên phải là vài con số về cửa hàng. Các bạn nhớ sơ sơ, vì con số nào cũng sẽ quay lại sau:

- 50 order mỗi giây lúc cao điểm, ngày flash sale lên tới 250.
- 10 pod chạy trên Kubernetes.
- 10 lần deploy mỗi ngày, và mỗi lần deploy là kill hết các pod cũ.
- App mobile chờ 10 giây rồi sẽ tự gửi lại.
- Payment vendor thường trả lời trong 200 mili giây, ngày xấu thì 3 giây.
- Mỗi pod có pool 20 kết nối tới database.

**[→]** *Chúng ta sẽ làm hỏng nó chín lần. Mỗi lần hỏng, thiết kế thêm một mảnh.*

**Nếu bị hỏi** *"Trừ tiền trước rồi mới lưu order, lỡ lưu thất bại thì sao?"*: Câu hỏi rất hay. Đó cũng chính là vấn đề dual write mà mình sắp nói ở vấn đề 1: hai hệ thống, không có transaction chung. Trong hệ thống thật, mình sẽ lưu order trạng thái "pending" trước, gọi payment kèm idempotency key, rồi cập nhật trạng thái. Hôm nay mình tập trung vào phần email cho dễ theo dõi.

## Slide 05 / 47 · Version 0 (sơ đồ)

⏱ 2 phút · rời slide lúc 0:10

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Database / Postgres:** Nơi lưu dữ liệu bền vững: tắt máy không mất. Postgres là loại database team dùng. Dữ liệu nằm trong các bảng (table), mỗi bản ghi là một dòng (row).

Đây là sơ đồ của version 0. Client gọi API. API gọi payment vendor, lưu order vào Postgres, rồi gọi email vendor sau khi commit. Một database, hai vendor. Rất đơn giản.

Sơ đồ này sẽ đi cùng mình suốt buổi. Mỗi khi có vấn đề, một **chấm đỏ** sẽ hiện lên đúng chỗ bị hỏng. Sửa xong, chấm đó chuyển sang màu xanh, và phần mình thêm vào sẽ sáng lên màu xanh lá.

**[→]** Chấm đỏ số 1 xuất hiện, ngay trên đường gọi email. Vấn đề đầu tiên: *pod chết ngay sau khi commit.*

**[→]** (hoặc click vào chấm đỏ)

---

# VẤN ĐỀ 1 · MẤT EMAIL (0:10 – 0:23)

## Slide 06 / 47 · Problem 1 · The pod dies after the commit

⏱ 2 phút · rời slide lúc 0:12

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Transaction:** Một nhóm thao tác database được coi như một khối: hoặc tất cả cùng thành công, hoặc như chưa có gì xảy ra.
> - **Commit / rollback:** Commit là "chốt" transaction; sau commit, dữ liệu chắc chắn đã được lưu. Rollback là hủy, quay về như lúc chưa làm.
> - **OOM kill:** Container dùng quá giới hạn bộ nhớ (Out Of Memory) nên bị kill ngay lập tức, không kịp chạy code dọn dẹp nào.
> - **Node drain:** Node là máy chạy các pod. Drain là Kubernetes dời hết pod khỏi máy đó (để nâng cấp hoặc thu nhỏ cụm), tức là kill các pod đang chạy trên đó.

Sơ đồ nhỏ bên trái cho thấy lỗi nằm ở đâu: đường gọi email sau khi commit.

**[→]** Khách hàng thấy gì: đã thanh toán, app báo đặt hàng thành công, nhưng không có email. Bộ phận hỗ trợ tìm không ra lỗi nào.

**Hỏi:** *"Các bạn đoán chuyện gì đã xảy ra?"*

**[→]** Chuyện thực tế: một lần deploy kill pod đúng vào khoảng giữa `COMMIT` và `email.send()`. Order đã được lưu. Còn email chỉ là một dòng code chưa kịp chạy.

**[→]** Vì sao dễ bị bỏ sót: trên laptop, tiến trình không bao giờ chết. Trên production: 10 pod nhân 10 lần deploy một ngày là 100 lần kill mỗi ngày, chưa kể bị kill vì hết bộ nhớ (OOM) hay node bị drain.

Mình đi sâu vào ba slide.

## Slide 07 / 47 · Two systems, one request, no shared transaction

⏱ 3 phút · rời slide lúc 0:15

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Coroutine / `launch { }`:** Cách chạy một đoạn code "ngầm" song song trong Kotlin. Nó chỉ sống trong bộ nhớ của tiến trình; tiến trình chết là nó mất theo, và không ai biết.
> - **Worker:** Một vòng lặp chạy nền, liên tục lấy việc ra làm. Relay và consumer ở các slide sau đều là worker.

Đây là đoạn code mà gần như ai cũng viết lần đầu. Trong transaction, insert order. Sau khi transaction xong thì gửi email.

**Hỏi:** *"Đoạn code này có thể hỏng ở đâu?"* Chỉ vào chú thích `// ← here` nếu chưa ai trả lời.

Đúng rồi. Pod có thể chết giữa lúc commit và lúc gửi. Order đã lưu, email không bao giờ đi, và không có gì báo cho mình biết: không lỗi, không log.

Có bạn sẽ nghĩ: vậy đưa lệnh gửi email vào trong transaction. Nhưng khi đó email gửi thành công rồi commit lại thất bại. Thế là mình gửi email cho một order không tồn tại.

Gốc vấn đề là: Postgres và email vendor là **hai hệ thống khác nhau**. Không có transaction nào bao trùm được cả hai.

Vậy một request được phép làm gì?

**[→]** Một: ghi vào database. Một transaction, gồm mọi bảng nó đụng tới, *kể cả một dòng nói rằng "còn một việc khác cần làm"*. Các bạn nhớ ý này, đây là ý chính của cả phần đầu buổi.

**[→]** Hai: trả lời. Client nhận 201 ngay khi commit xong, không phải khi email đã đi.

**[→]** Ba: không làm gì khác. Không `launch { }` để gửi email chạy ngầm, không "gửi thử sau commit". Một coroutine chạy ngầm chỉ sống trong bộ nhớ; pod chết thì công việc chết theo.

*Việc gì quan trọng thì phải là một dòng trong database. Một worker sẽ làm nó.*

## Slide 08 / 47 · Four ways a dual write can end

⏱ 3 phút · rời slide lúc 0:18

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Dual write:** Ghi vào hai hệ thống (ví dụ database và email/queue) trong cùng một request mà không có transaction chung. Luôn có một khoảnh khắc mà một bên đã ghi, bên kia thì chưa.
> - **Event (sự kiện):** Thông báo rằng "điều X đã xảy ra", ví dụ `OrderCreated`. Bên khác nghe event để làm phần việc của họ, ví dụ gửi email.
> - **Happy path:** Kịch bản mà mọi thứ đều chạy đúng.
> - **Source of truth:** Nơi được coi là đúng nhất. Khi hai nơi lệch nhau thì tin nơi này.
> - **Distributed transaction / two-phase commit (2PC):** Cơ chế để nhiều hệ thống cùng commit hoặc cùng hủy. Hiếm nơi hỗ trợ, chậm và dễ bị kẹt, nên mình tránh dùng.

*Lúc vào slide: dòng thời gian BEGIN → INSERT → COMMIT hiện ra.*

Việc ghi vào hai hệ thống như vậy gọi là **dual write**. Đây là dòng thời gian: begin, insert order, commit. Order đã an toàn trong database.

**[→]** Rồi pod bị kill: do deploy, do hết bộ nhớ, do node bị drain. Lệnh gửi email không bao giờ chạy.

Có hai thứ có thể thành công hoặc thất bại: commit và gửi. Vậy có bốn khả năng.

**[→]** Hai trường hợp là trung thực. Cả hai thành công: happy path, demo nào cũng trông như vậy. Cả hai thất bại: không có gì xảy ra, client nhận lỗi và thử lại. Phiền nhưng trung thực.

**[→]** Hai trường hợp còn lại là bug. Commit thành công, gửi thất bại: **lost event**, có order mà không có email. Commit thất bại, gửi thành công: **ghost event**, có email cho một order không tồn tại.

**[→]** Và đây là điểm quan trọng: *đổi thứ tự hai lệnh chỉ đổi bug này lấy bug kia.* Phải có một hệ thống làm nguồn sự thật duy nhất, và đó là database. Pattern này gọi là **transactional outbox**.

**Hỏi:** *"Các bạn đã bao giờ gặp một trong bốn trường hợp này khi dùng app nào chưa?"*

**Nếu bị hỏi** *"Sao không dùng distributed transaction (two-phase commit)?"*: SQS và các email vendor không hỗ trợ. Ở chỗ có hỗ trợ thì nó chậm và dễ bị kẹt. Để database làm nguồn sự thật duy nhất thì đơn giản hơn nhiều.

## Slide 09 / 47 · Transactional outbox

⏱ 4 phút · rời slide lúc 0:22

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Transactional outbox:** Thay vì gửi event trực tiếp, ghi event thành một dòng trong bảng `outbox`, trong *cùng transaction* với dữ liệu chính. Sau đó một worker đọc dòng đó và gửi đi.
> - **Relay:** Worker đọc các dòng outbox chưa gửi, publish lên SQS, rồi đánh dấu là đã gửi.
> - **Publish:** Gửi một message vào queue.
> - **Consumer / producer:** Consumer là worker nhận message từ queue và làm việc (ở đây là gửi email). Bên gửi message vào queue gọi là producer.
> - **At-least-once (ít nhất một lần):** Mỗi message chắc chắn được giao, nhưng có thể bị giao nhiều hơn một lần. Đó là cái giá để không bao giờ bị mất.
> - **Dedupe (loại trùng):** Phát hiện một việc đã làm rồi và bỏ qua, thường bằng cách lưu lại id của những thứ đã xử lý.
> - **Event id:** Id duy nhất của mỗi event, để nhận ra hai lần giao là của cùng một event.
> - **Payload:** Phần nội dung chính của message, ví dụ order id.
> - **Org id:** Id của tổ chức (doanh nghiệp khách hàng). Hệ thống phục vụ nhiều tổ chức cùng lúc (multi-tenant), nên dữ liệu luôn gắn org id để không lẫn của nhau.

Đây là cách sửa. Ba phần.

**Phần 1, trong request.** Cùng một transaction: insert order, và insert thêm một dòng vào bảng `outbox`, ghi loại sự kiện, payload và org id. Hai dòng cùng được lưu, hoặc cùng không. Giờ chỉ còn một lần commit, nên vấn đề dual write biến mất.

**Phần 2, relay.** Một vòng lặp chạy trên mọi pod. Nó lấy một lô dòng chưa gửi, commit, publish lên SQS, rồi đánh dấu đã gửi. Nếu pod chết sau khi publish mà trước khi đánh dấu, lô đó sẽ được publish lại. Nghĩa là một message có thể được gửi hai lần. Đó là ý nghĩa của **at-least-once**: ít nhất một lần.

**Phần 3, consumer.** Nhận message và dedupe, tức là loại trùng, theo event id. Nếu id đã xử lý rồi thì bỏ qua.

*Ba phần, ba trách nhiệm.* Request làm cho công việc **bền vững** (không mất). Relay làm cho nó **được giao đi**. Consumer làm cho nó **chỉ xảy ra một lần**.

Ghi chú cuối slide: nhớ xóa hoặc lưu trữ các dòng đã gửi theo lịch, không thì bảng outbox sẽ thành bảng lớn nhất database.

Relay và dedupe mình sẽ đào sâu ở vấn đề 3 và 4.

**Hỏi:** *"Tại sao relay gửi trùng một dòng hai lần lại không sao?"* Đáp: vì consumer dedupe theo event id.

**Ví dụ dễ hiểu (nếu cần):** Nhà hàng. Người phục vụ không tự nấu; họ ghi order ra phiếu và ghim lên bảng của bếp. Đó là outbox. Bếp lấy phiếu xuống làm. Nếu một đầu bếp nghỉ giữa chừng, phiếu vẫn còn trên bảng.

## Slide 10 / 47 · Version 1 (sơ đồ)

⏱ 1 phút · rời slide lúc 0:23

Version 1. Các phần màu xanh lá là phần mới: bảng `outbox`, relay, SQS, consumer. Đường gọi email trực tiếp bị gạch đỏ: mình đã bỏ nó. Giờ email được gửi bởi consumer. Chấm số 1 đã xanh.

**[→]** Vấn đề số 2 xuất hiện, ngay giữa client và API: *app gửi lại request.*

**[→]**

---

# VẤN ĐỀ 2 · TRỪ TIỀN HAI LẦN (0:23 – 0:31)

## Slide 11 / 47 · Problem 2 · The app retries

⏱ 2 phút · rời slide lúc 0:25

**[→]** Khách hàng thấy gì: hai order, hai lần trừ tiền, cho một lần bấm "Place order".

**Hỏi:** *"Vì sao lại có hai order?"*

**[→]** Chuyện thực tế: request `POST` đầu tiên đã commit, nhưng mất 11 giây. App chỉ chờ 10 giây rồi gửi lại. API không có cách nào biết đó vẫn là cùng một lần bấm.

**[→]** Vì sao dễ bị bỏ sót: outbox ghi lại trung thực cả hai order. Consumer dedupe theo event id, mà đây là hai event khác nhau.

## Slide 12 / 47 · Idempotency at the edge

⏱ 3 phút · rời slide lúc 0:28

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Idempotent:** Làm nhiều lần cho kết quả giống như làm một lần.
> - **Idempotency key:** Một id ngẫu nhiên do client tạo cho mỗi ý định của người dùng, gửi kèm trong mọi lần retry. Server dựa vào nó để nhận ra request trùng.
> - **Header:** Thông tin phụ đi kèm HTTP request, ngoài phần body, có dạng `Tên: giá trị`.
> - **Unique constraint:** Luật của database: không được có hai dòng trùng giá trị ở những cột này. Lệnh insert trùng bị database từ chối, kể cả khi hai request đến cùng một lúc.

Một từ mới: **idempotent**. Một thao tác là idempotent nếu làm hai lần có kết quả giống như làm một lần. Bấm nút thang máy năm lần thì vẫn chỉ gọi một thang máy.

**[→]** Có hai loại trùng lặp khác nhau. Queue giao cùng một message hai lần: cái đó consumer xử lý. Còn *một hành động của người dùng* đến hai lần: cái đó phải xử lý ngay tại API.

**[→]** Cách sửa: client gửi header `Idempotency-Key`. Đó là một id ngẫu nhiên, được tạo *một lần cho mỗi ý định của người dùng*. "Người dùng bấm Place order" là một ý định; mọi lần gửi lại của lần bấm đó dùng chung key. Nếu client tạo key mới cho mỗi lần gửi lại thì key vô dụng.

Chỉ vào code: trong cùng transaction, tìm key. Đã thấy rồi thì trả lại response đã lưu. Chưa thấy thì tạo order, dòng outbox và key cùng lúc. Có unique constraint trên `(org_id, key)`, nên nếu hai request trùng đến cùng lúc, một cái commit, cái kia đụng constraint và trả về response đã lưu.

**[→]** Key được giữ đủ lâu để bao hết thời gian app có thể gửi lại, thường là 24 giờ, rồi job dọn dẹp sẽ xóa.

**Nếu bị hỏi** *"Payment vendor được gọi trước khi kiểm tra key, vậy vẫn trừ tiền hai lần à?"*: Đúng, nếu để như code version 0 thì vẫn bị. Trong thực tế phải kiểm tra key **trước** khi gọi payment, và truyền luôn key đó cho payment vendor; hầu hết payment API đều nhận idempotency key để tự loại trùng ở phía họ.

## Slide 13 / 47 · The retry, step by step

⏱ 2 phút · rời slide lúc 0:30

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Request / response:** Request là lời gọi client gửi đi. Response là câu trả lời server gửi về. Mất response nghĩa là server đã làm xong, nhưng client không biết.

Ba làn: client, API, Postgres.

**[→]** Client gửi `POST /orders` với key `9f1c`. API insert key, order và dòng outbox trong một transaction. Commit. Order 1042 tồn tại.

**[→]** API trả 201, nhưng client đã bỏ cuộc ở giây thứ 10. Response bị mất. Với client, coi như chưa có gì xảy ra.

**[→]** Client gửi lại, với *cùng* key `9f1c`. API tìm thấy: order 1042.

**[→]** API trả lại response đã lưu: 201, order 1042. Không có order thứ hai, không có dòng outbox thứ hai. *Một ý định, một order, một event, một lần trừ tiền.*

**Hỏi:** *"Nếu client tạo key mới cho lần gửi lại thì sao?"* Đáp: trông như ý định mới, lại thành hai order.

## Slide 14 / 47 · Version 2 (sơ đồ)

⏱ 1 phút · rời slide lúc 0:31

Version 2: thêm bảng `idempotency_keys`, và header `Idempotency-Key` trên đường từ client tới API.

**[→]** Vấn đề số 3, nằm ở relay: *SQS bị chậm một phút.*

**[→]**

---

# VẤN ĐỀ 3 · RELAY GIỮ LOCK (0:31 – 0:39)

## Slide 15 / 47 · Problem 3 · SQS is slow for a minute

⏱ 2 phút · rời slide lúc 0:33

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Row lock:** Khi một transaction đang sửa một dòng, database khóa dòng đó lại; transaction khác muốn sửa phải chờ tới khi transaction đầu commit hoặc rollback. `SELECT … FOR UPDATE` nghĩa là đọc và khóa luôn.

**[→]** Khách hàng thấy gì: trang thanh toán quay mãi rồi timeout, dù chỉ có SQS đang chậm.

**Hỏi:** *"SQS chậm thì liên quan gì tới API nhỉ?"*

**[→]** Chuyện thực tế: relay giữ row lock (`FOR UPDATE`) và một kết nối database trong suốt lúc chờ SQS. Pod nào cũng vậy. Pool kết nối cạn sạch, request của API phải xếp hàng chờ kết nối.

**[→]** Vì sao dễ bị bỏ sót: ngày thường publish chỉ mất 20 mili giây, nên lock gần như vô hình. Nó chỉ gây hại vào đúng ngày xấu.

## Slide 16 / 47 · The relay

⏱ 3 phút · rời slide lúc 0:36

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Claim / lease:** Đánh dấu "tôi đang xử lý những dòng này tới thời điểm T" bằng một cột (`claimed_until`), thay vì giữ lock. Quá T mà chưa xong (ví dụ vì pod chết) thì worker khác được lấy.
> - **SKIP LOCKED:** Tùy chọn của Postgres: dòng nào đang bị khóa thì bỏ qua và lấy dòng khác, thay vì đứng chờ.
> - **Batch (lô):** Xử lý nhiều mục trong một lần, ví dụ 100 dòng, cho hiệu quả.
> - **FIFO queue:** Loại queue giữ đúng thứ tự: vào trước thì ra trước. Đổi lại thì xử lý được ít hơn. Queue SQS thường (standard) không đảm bảo thứ tự.
> - **Polling:** Cứ vài giây lại hỏi "có gì mới không?".
> - **CDC (change data capture) / Debezium:** Đọc thẳng nhật ký thay đổi của database để biết có dòng nào mới, không cần hỏi liên tục. Debezium là một công cụ làm việc này.

Mình đọc câu SQL từ trên xuống.

Bước 1, **claim** (giành quyền xử lý), trong một transaction ngắn riêng. Tìm các dòng chưa gửi, và chưa ai claim, hoặc claim đã hết hạn. Lấy tối đa 100 dòng.

`FOR UPDATE SKIP LOCKED`: khóa các dòng mình chọn, và nếu pod khác đã khóa dòng nào thì *bỏ qua* dòng đó thay vì đứng chờ. Nhờ vậy mười pod chạy relay cùng lúc mà không giành nhau.

Rồi set `claimed_until` bằng bây giờ cộng 30 giây. Đó giống như một hợp đồng thuê: "tôi đang xử lý mấy dòng này trong 30 giây tới". Rồi commit. Lock được nhả ra.

Bước 2, publish lên SQS, không giữ lock nào. Bước 3, đánh dấu đã gửi, nhưng chỉ những id thật sự publish thành công.

**[→]** Tại sao không làm tất cả trong một transaction? Vì giữ lock trong lúc chờ mạng rất nguy hiểm, như chúng ta vừa thấy. Cột claim thay thế cho lock dài. Nếu pod crash, claim tự hết hạn và pod khác nhận lô đó.

*Không bao giờ giữ row lock qua một lần gọi mạng.* Quy tắc này đúng ở mọi nơi, không chỉ ở relay.

**[→]** Thứ tự không được bảo toàn. Mười pod publish mười lô song song. Nếu thứ tự quan trọng, đưa số thứ tự vào payload, hoặc dùng FIFO queue.

**[→]** Polling là một lựa chọn. Cách khác là CDC, đọc log thay đổi của database bằng công cụ như Debezium: nhanh hơn nhưng nhiều thứ phải vận hành hơn. Mình chọn polling vì chỉ cần một bảng và một vòng lặp.

## Slide 17 / 47 · Lock time, before and after

⏱ 2 phút · rời slide lúc 0:38 *(cắt đầu tiên nếu trễ giờ)*

**[→]** Trước: một transaction. Select nhanh, rồi publish lên SQS, hôm nay chậm, mất 1 giây, rồi update nhanh.

**[→]** Thanh đỏ là thời gian giữ row lock: 1,08 giây. Suốt cả lần gọi chậm.

**[→]** Sau: claim, commit, publish, đánh dấu. Hai khoảng lock rất ngắn, 40 và 15 mili giây. Lần gọi chậm không giữ gì cả.

**[→]** Nếu pod crash ngay sau khi claim? Không mất gì. Các dòng nằm đó 30 giây, rồi pod khác nhận. Consumer dedupe nên gửi lại cũng vô hại.

Con số chỉ để minh họa; cái quan trọng là hình dạng.

## Slide 18 / 47 · Version 3 (sơ đồ)

⏱ 1 phút · rời slide lúc 0:39

Version 3: relay giờ ghi "claims, no locks", tức là claim chứ không giữ lock.

**[→]** Vấn đề số 4, giữa SQS và consumer: *cùng một message, hai lần.*

**[→]**

---

# VẤN ĐỀ 4 · MESSAGE GIAO HAI LẦN (0:39 – 0:48)

## Slide 19 / 47 · Problem 4 · The same message, twice

⏱ 2 phút · rời slide lúc 0:41

**[→]** Khách hàng thấy gì: hai email xác nhận giống hệt nhau, có khi cách nhau vài phút.

**[→]** Chuyện thực tế: SQS giao ít nhất một lần. Có thể relay crash nên publish lại, hoặc consumer xử lý lâu hơn visibility timeout nên một consumer khác nhận luôn message đó.

**[→]** Vì sao dễ bị bỏ sót: hiếm khi gặp lúc test, nhưng bình thường ở production, nhất là trong lúc deploy.

## Slide 20 / 47 · The queue

⏱ 3 phút · rời slide lúc 0:44

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Ack (acknowledge):** Báo cho queue: "tôi xử lý xong rồi, xóa message đi". Với SQS là gọi `DeleteMessage`.
> - **Visibility timeout:** Khi consumer nhận một message, SQS giấu message đó khỏi các consumer khác trong khoảng thời gian này. Không ack kịp thì message hiện lại và được giao lần nữa.
> - **maxReceiveCount:** Số lần một message được nhận mà không được ack, trước khi SQS chuyển nó sang DLQ.
> - **DLQ (dead-letter queue):** Queue riêng chứa các message đã xử lý thất bại quá nhiều lần, để người vào kiểm tra.
> - **Poison message:** Message mà lần nào xử lý cũng lỗi, ví dụ vì dữ liệu hỏng hoặc gặp bug.
> - **Redrive:** Chuyển message từ DLQ về lại queue chính, sau khi đã sửa lỗi.
> - **Long polling:** Lệnh nhận message chờ tối đa 20 giây cho tới khi có message, thay vì trả về rỗng ngay.
> - **Type / version:** Loại event và phiên bản cấu trúc của nó. Nhờ vậy sau này đổi format được mà consumer cũ không bị vỡ.
> - **traceparent:** Chuỗi chứa trace id, gắn vào message để theo dõi được một request đi xuyên qua nhiều hệ thống (xem slide 41).

*SQS giao ít nhất một lần. Mọi thứ khác đều suy ra từ đó.* Gần như luôn là một lần. Thỉnh thoảng hai lần. Code của mình phải ổn với hai lần.

Nhìn message: có `id` để dedupe; `type` và `version` để consumer phát triển mà không vỡ; `org_id` để consumer giới hạn truy vấn trong một khách hàng; `traceparent` để trace đi tiếp, vấn đề 9 sẽ thấy; và payload chỉ chứa *id, không chứa cả object*. Consumer đọc dữ liệu mới nhất từ database. Một bản sao order trong message sẽ cũ đi khi được đọc.

Giải thích cách SQS hoạt động (vẽ lên bảng nếu cần): khi consumer nhận message, SQS không xóa nó mà chỉ *giấu* nó đi một lúc. Consumer xóa kịp thì đó là **ack**, message biến mất. Không xóa kịp thì message hiện lại và có người khác nhận.

**[→]** "Một lúc" đó là **visibility timeout**. Nó phải dài hơn thời gian xử lý, nếu không consumer thứ hai sẽ nhận message khi consumer đầu còn đang làm.

**[→]** **`maxReceiveCount`**: sau bấy nhiêu lần thất bại, message thôi lặp và được chuyển sang dead-letter queue.

**[→]** **Dead-letter queue (DLQ)**: nơi chứa các "poison message", message lần nào xử lý cũng lỗi. Có alert khi DLQ không rỗng. Sửa bug xong thì "redrive", đẩy message về lại queue chính.

**[→]** **Long polling**, `WaitTimeSeconds 20`: lệnh nhận chờ tới 20 giây cho có message thay vì trả về rỗng ngay. Ít lần gọi rỗng, rẻ hơn, nhận nhanh hơn.

## Slide 21 / 47 · Visibility timeout and the DLQ

⏱ 3 phút · rời slide lúc 0:47

**[→]** Consumer A nhận message. Message bị giấu 30 giây. Nhưng hôm nay xử lý mất 45 giây, có thể vì email vendor chậm.

**[→]** Ở giây 30, message hiện lại. Consumer B nhận nó. A làm xong, gửi email. B cũng gửi email. Không có dedupe: *hai email*.

**Hỏi:** *"Sửa thế nào?"* Mong đợi: "tăng timeout" và "dedupe". Cả hai đều đúng.

**[→]** Đặt visibility timeout lớn hơn thời gian xử lý p99, *và vẫn dedupe*. Trùng lặp vẫn xảy ra lúc deploy hoặc lúc tiến trình bị khựng. (p99 nghĩa là 99% message được xử lý nhanh hơn con số này.)

**[→]** Còn message lần nào cũng lỗi, ví dụ dữ liệu hỏng: nhận lần 1 lỗi, 2, 3, 4, 5 vẫn lỗi. `maxReceiveCount` bằng 5, nên nó chuyển sang DLQ và alarm kêu. Không có DLQ thì nó lặp mãi.

## Slide 22 / 47 · Version 4 (sơ đồ)

⏱ 1 phút · rời slide lúc 0:48

Version 4: thêm bảng `processed` để dedupe.

**[→]** Vấn đề số 5, ở consumer: *consumer crash, hoặc gặp message hỏng.*

**[→]**

---

# VẤN ĐỀ 5 · CONSUMER CRASH VÀ POISON MESSAGE (0:48 – 0:56)

## Slide 23 / 47 · Problem 5 · The consumer crashes, or a message is bad

⏱ 2 phút · rời slide lúc 0:50

**[→]** Khách hàng thấy gì: vài order không bao giờ có email. Vài order khác thì email đến trễ hàng giờ.

**[→]** Chuyện thực tế: consumer ack trước khi công việc commit, rồi crash: message mất luôn. Riêng một message hỏng thì throw ra khỏi vòng lặp cả lô, và cứ quay lại mãi.

**[→]** Vì sao dễ bị bỏ sót: ack trước trông gọn gàng. Bug chỉ xuất hiện khi crash đúng dòng sai, và production sớm muộn gì cũng làm vậy.

## Slide 24 / 47 · The consumer

⏱ 3 phút · rời slide lúc 0:53

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Thread pool / executor:** Một nhóm thread (luồng xử lý) cố định để chạy công việc. Mỗi việc chiếm một thread; hết thread thì việc mới phải chờ.
> - **Exception / try-catch:** Exception là lỗi được "ném" ra khi có sự cố. try/catch bắt lỗi đó lại để xử lý, thay vì để nó làm dừng cả vòng lặp.
> - **Stack trace:** Danh sách các hàm đang chạy tại lúc lỗi xảy ra, giúp tìm ra dòng code gây lỗi.

Đọc code: nhận tối đa 10 message. Với mỗi message: mở transaction. Thử insert message id vào `processed`. Nếu đã có, việc này làm rồi: bỏ qua. Nếu chưa, làm việc. Commit. *Sau đó* mới xóa message khỏi SQS, tức là ack. Nếu có lỗi, log kèm exception, và *không* xóa. SQS sẽ giao lại sau.

**[→]** try/catch đặt *cho từng message*. Một message hỏng không được làm dừng chín message còn lại.

**[→]** Giữ transaction nhỏ. Không gọi ra ngoài bên trong transaction, cùng quy tắc với relay. Nếu cần gọi vendor, commit trước rồi mới gọi, hoặc ghi thêm một dòng outbox nữa.

**[→]** Mười pod chạy vòng lặp này cùng lúc. Nó đúng nhờ bảng dedupe, không phải nhờ may mắn.

**[→]** Consumer có thread pool riêng và kết nối database riêng. Nếu dùng chung pool với API, lúc queue dồn nhiều, consumer sẽ chiếm hết và API phải chờ.

## Slide 25 / 47 · Where can it crash?

⏱ 2 phút · rời slide lúc 0:55

**[→]** Cách sai: nhận, xóa message, rồi crash trước khi làm việc. Message đã biến khỏi SQS. Công việc chưa bao giờ chạy. *Mất vĩnh viễn.* Không giao lại, không vào DLQ, không có dòng log nào.

**[→]** Cách đúng: nhận, mở transaction, dedupe, làm việc, commit.

**[→]** Giờ crash trước khi ack. SQS giao lại sau 30 giây. Dedupe tìm thấy id, bỏ qua, ack. Công việc làm đúng một lần, message biến mất.

**[→]** *Chỉ có ack-trước mới làm mất việc.* Commit-rồi-ack chỉ có thể làm lặp lại, và dedupe biến lần lặp thành vô hại.

**Hỏi:** *"Nếu crash trước khi commit thì sao?"* Đáp: transaction rollback, coi như chưa có gì, message chạy lại. Ổn.

**Nếu bị hỏi** *"Email không nằm trong database, vậy commit xong mà crash trước khi gửi email thì sao?"*: Rất tinh ý. Lúc giao lại, dedupe sẽ bỏ qua, nên email bị mất. Không thể làm side effect bên ngoài "đúng một lần" chỉ bằng code của mình. Cách tốt nhất: consumer ghi thêm một dòng outbox "gửi email", và một worker riêng gửi nó, kèm idempotency key cho email vendor để họ tự loại trùng. Lại là *"việc quan trọng thì phải là một dòng"*.

## Slide 26 / 47 · Version 5 (sơ đồ)

⏱ 1 phút · rời slide lúc 0:56

Version 5: consumer ghi "commit, then ack", và thêm `orders-dlq` với đường "sau 5 lần nhận".

Năm vấn đề đầu đều về **công việc rời khỏi request mà không mất, không trùng**. Bốn vấn đề sau là về **khi mọi thứ trở nên chậm**.

**[→]** Vấn đề số 6 xuất hiện ở payment vendor. Nhưng trước đó mình nghỉ 5 phút.

---

# NGHỈ GIẢI LAO 5 PHÚT (0:56 – 1:01)

Nói: *"Mình nghỉ 5 phút nhé. Quay lại, chúng ta xem chuyện gì xảy ra khi payment vendor bị chậm."*

Trong lúc nghỉ: xem đồng hồ, quyết định có cần cắt gì ở phần sau không.

Sau khi nghỉ, nhấn **[→]** để vào vấn đề 6.

---

# VẤN ĐỀ 6 · VENDOR CHẬM (1:01 – 1:19)

## Slide 27 / 47 · Problem 6 · The payment vendor gets slow

⏱ 2 phút · rời slide lúc 1:03

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Sập vs chậm (down vs slow):** Sập: gọi tới thì bị từ chối ngay. Chậm: gọi tới thì phải chờ rất lâu mới có trả lời, hoặc chờ mãi không có.
> - **Health check:** Endpoint (ví dụ `/health`) mà Kubernetes gọi định kỳ để hỏi "pod còn sống không?". Nhiều lần không trả lời kịp thì Kubernetes restart pod.

Chào mừng các bạn quay lại. Mình là [tên], phần sau mình dẫn. Năm vấn đề đầu là về công việc không mất, không trùng. Giờ mình nói về chuyện khi mọi thứ trở nên chậm. Câu hỏi khởi động:

**Hỏi:** *"Cái nào tệ hơn: một hệ thống phụ thuộc bị sập, hay bị chậm?"* Cho cả phòng giơ tay. Đa số sẽ chọn "sập".

**[→]** Khách hàng thấy gì: trang nào cũng chậm, kể cả những trang không hề đụng tới thanh toán. Pod bị restart.

**[→]** Chuyện thực tế: mọi thread đều chờ vendor. Health check không lấy được thread, Kubernetes restart cả những pod đang khỏe, và chính các lần retry của mình làm vendor càng chậm hơn.

**[→]** Vì sao dễ bị bỏ sót: *sập thì lỗi nhanh. Chậm thì giữ thread, kết nối và bộ nhớ cho tới khi mọi thứ xếp hàng phía sau nó.* Nên chậm tệ hơn sập.

## Slide 28 / 47 · Four tools, one goal

⏱ 3 phút · rời slide lúc 1:06

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Fail fast:** Lỗi nhanh: trả lỗi ngay thay vì chờ lâu, để giải phóng thread và kết nối cho việc khác.
> - **Backoff:** Mỗi lần retry chờ lâu hơn lần trước, ví dụ 200 ms, 400 ms, 800 ms.
> - **Jitter:** Cộng thêm một khoảng ngẫu nhiên vào thời gian chờ, để các client không retry cùng một lúc.
> - **Circuit breaker:** "Cầu dao": thấy lỗi nhiều thì ngừng gọi một thời gian và trả lỗi ngay; sau đó cho thử lại một request.
> - **Bulkhead:** Chia tài nguyên (thread, kết nối) thành các ngăn riêng cho từng phụ thuộc, để một phụ thuộc hỏng không chiếm hết.
> - **Little's law:** Số request đang xử lý cùng lúc = số request đến mỗi giây × thời gian xử lý mỗi request. Dùng để ước lượng pool cần lớn cỡ nào.
> - **In flight:** Request đã nhận nhưng chưa trả lời xong.

Bốn công cụ, một mục tiêu: *fail fast and alone*, tức là lỗi nhanh, và chỉ lỗi ở phần phụ thuộc vào thứ bị hỏng.

**Timeout ở mọi chặng, chặng trong ngắn hơn chặng ngoài.** Client chờ 10 giây. API cho vendor 3 giây. Kết nối tới vendor 1 giây. Nếu timeout bên trong dài hơn bên ngoài, mình vẫn đang làm việc khi người gọi đã bỏ đi.

**Retry có backoff, jitter và giới hạn.** Ba lần thử. Mỗi lần chờ lâu hơn. Thêm ngẫu nhiên. Chỉ retry lỗi có thể thành công lần sau: timeout, 503. Không bao giờ retry 400: request sai thì gửi lại vẫn sai. Và chỉ retry những lời gọi an toàn khi lặp lại.

**Circuit breaker.** Sau đủ số lần lỗi, ngừng gọi một lúc. Lỗi ngay lập tức. Rồi cho một request đi qua để thử.

**Bulkhead.** Vendor có thread pool nhỏ riêng. Khi vendor treo, chỉ pool đó đầy.

**[→]** Pool nên lớn bao nhiêu? **Little's law**: *số request đang xử lý bằng tốc độ đến nhân thời gian mỗi request.* Tính **cho một pod**. Ngày flash sale có 250 order mỗi giây, chia cho 10 pod là 25 request mỗi giây mỗi pod. Mỗi request gọi vendor 200 mili giây: 25 nhân 0,2 bằng 5 request cùng lúc. Vendor chậm thành 3 giây: 25 nhân 3 bằng 75. Pool 20 không chứa nổi. Đó là lý do vendor cần bulkhead.

Nói phép tính thật chậm, và nhấn mạnh chữ *mỗi pod*. Pool 20 ở slide 4 là của mỗi pod, nên tốc độ đến cũng phải tính cho mỗi pod; lấy 50 order/s của cả cụm nhân với pool của một pod là sai.

Ở đây "pool" là số lời gọi chạy cùng lúc trên một pod (thread, hoặc với coroutine là số slot của semaphore). Lời gọi payment nằm ngoài transaction, nên thứ bị cạn là chỗ này, không phải kết nối DB.

## Slide 29 / 47 · Timeouts and breakers, drawn

⏱ 3 phút · rời slide lúc 1:09

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Fallback:** Phương án dự phòng trả về khi không gọi được phụ thuộc, ví dụ dữ liệu cache hoặc giá trị mặc định.

**[→]** Client 10 giây. API tới vendor 3 giây. Kết nối 1 giây. Cái này nằm gọn trong cái kia, như búp bê Nga.

**[→]** Trường hợp sai: lời gọi vendor có timeout 15 giây. Ở giây 10 client đã bỏ đi. Mình vẫn làm thêm 5 giây nữa, cho không ai cả. Tốn công mà chẳng ai nhận kết quả.

**[→]** Circuit breaker là một máy trạng thái nhỏ. Các bạn xem màu chạy vòng:

- **Closed**, màu xanh: lời gọi đi bình thường. Mình đếm số lỗi. Đủ N lỗi trong một khoảng thời gian thì chuyển sang open.
- **Open**, màu đỏ: không gọi nữa. Trả lỗi hoặc giá trị dự phòng ngay lập tức. Hết thời gian nghỉ thì sang half-open.
- **Half-open**, màu cam: cho đúng một request thử đi qua. Thành công thì về closed. Thất bại thì quay lại open.

Câu cuối slide quan trọng: *khi breaker đang open, trả fallback ngay lập tức.* Breaker mà vẫn ngồi chờ timeout thì chẳng cứu được gì.

**Ví dụ dễ hiểu:** cầu dao điện trong nhà. Chập điện thì cầu dao nhảy để cả nhà không cháy. Lát sau bạn bật lại xem nó có giữ được không.

**Gài sẵn cho slide 31:** *"Để ý nhé: API cho vendor 3 giây, thử 3 lần, cộng backoff là khoảng 9,6 giây, sát mốc 10 giây của client. Lần thử cuối gần như chắc chắn không ai nhận. Hai slide nữa mình sửa chuyện này."*

## Slide 30 / 47 · Bulkheads, visualized

⏱ 4 phút · rời slide lúc 1:13

**[→]** Một pool chung 20 chỗ trên mỗi pod. Vendor treo. Từng thread một bị kẹt chờ nó. Giờ `GET /health`, `GET /me`, `POST /orders` đều phải chờ thread. Health check timeout, và Kubernetes restart một pod *đang khỏe*. Bây giờ ít pod hơn mà vấn đề vẫn y nguyên.

Chữ "bulkhead" lấy từ tàu thủy: thân tàu được chia thành nhiều khoang, một khoang thủng thì tàu không chìm.

**[→]** Mỗi phụ thuộc một pool: 5 cho vendor, 15 cho API. Pool vendor đầy 5 chỗ. Các lời gọi vendor tiếp theo lỗi nhanh với 503. Nhưng health trả lời trong 12 mili giây, mọi thứ khác vẫn phục vụ bình thường.

**[→]** Máy tính Little's law. Kéo thanh trượt trực tiếp:

Thử nhé. Thanh trượt tính cho một pod. 25 request mỗi giây, 200 mili giây: 5, đúng bằng pool vendor 5 vừa rồi. Vạch trắng là pool 20. Giờ vendor chậm: 3000 mili giây. 75. Vượt xa.

**Hỏi:** *"Ai cho mình một tốc độ request và một độ trễ?"* Để một bạn chọn số. Cả phòng tính nhẩm trước, rồi mới kéo thanh trượt.

(Lưu ý: thanh trượt không làm chuyển slide. Click ra ngoài hoặc dùng bàn phím để đi tiếp.)

**Nếu bị hỏi** *"Code mình dùng Kotlin coroutine, vậy bulkhead có phải là thread pool không?"*: Với HTTP client dạng suspend thì thread không bị chặn khi chờ. Thứ cần giới hạn là *số lời gọi vendor chạy cùng lúc*, nên dùng `Semaphore(5)` hoặc `Dispatchers.IO.limitedParallelism(5)` cho vendor. Cùng một ý tưởng, chỉ khác công cụ.

## Slide 31 / 47 · The fine print

⏱ 3 phút · rời slide lúc 1:16

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Retry amplification:** Retry ở nhiều tầng làm số lời gọi bị nhân lên: 3 × 3 × 3 = 27.
> - **Deadline:** Thời điểm cuối cùng mà kết quả còn có ích. Khác timeout cố định ở chỗ nó tính theo phần thời gian *còn lại*.
> - **Cache:** Bản sao dữ liệu lưu tạm để dùng lại cho nhanh. Có thể hơi cũ.
> - **202 Accepted:** Mã HTTP nghĩa là "đã nhận, sẽ xử lý sau".

*Lúc vào slide: cây phân nhánh mọc ra 1 → 3 → 9 → 27.*

**Khuếch đại retry.** Ba tầng: app mobile, API, payments service. Mỗi tầng thử 3 lần (tức 3 lần gọi, không phải 1 lần gọi cộng 3 lần retry). Một lần bấm thành 3, rồi 9, rồi *27 lời gọi* tới một vendor đang quá tải. Chính mình đang tấn công vendor của mình.

Cách sửa: *chỉ retry ở một chỗ phía server*, thường là tầng gần vendor nhất. App vẫn được retry, vì nhờ `Idempotency-Key` ở vấn đề 2 nên retry của app là an toàn; cái cần tránh là API và payments service *cũng* retry chồng lên.

Thêm một ý nếu còn thời gian: **retry budget**. Chỉ cho retry khi số lần retry còn dưới khoảng 10% tổng số lời gọi tới phụ thuộc đó. Vendor sập hẳn thì budget hết, mình thôi retry.

**[→]** **Truyền deadline xuống, không dùng timeout cố định.** Đây là lời giải cho con số 9,6 giây ở slide 29. Client cho mình 10 giây mà 7 giây đã trôi qua, thì lời gọi vendor chỉ được phần còn lại, không phải 3 giây như thường. Truyền deadline xuống, và kiểm tra nó trước mỗi lần retry.

**[→]** **Quyết định "lỗi nhanh" thì trả về gì.** Lỗi là lựa chọn cuối cùng. Tốt hơn là: một kết quả cache, một tính năng thu gọn, hoặc "đã nhận, sẽ xử lý sau": trả 202 và ghi một dòng outbox. Lại là outbox.

## Slide 32 / 47 · Backoff and jitter

⏱ 2 phút · rời slide lúc 1:18 *(rút gọn nếu trễ)*

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Thundering herd:** Nhiều client cùng làm một việc vào đúng một lúc, ví dụ cùng retry, tạo ra những đợt tải dồn dập.

Tưởng tượng 200 client cùng lỗi một lúc, vì vendor vừa chập chờn.

**[→]** Không có jitter: tất cả chờ đúng 200 mili giây rồi retry cùng lúc. Rồi cùng chờ thêm 400 và lại retry cùng lúc. Lần nào cũng 200 request ập vào một lúc, và mỗi đợt lại đánh sập vendor. Hiện tượng này gọi là *thundering herd*, đàn trâu giẫm đạp.

**[→]** Có full jitter: mỗi client chờ một khoảng *ngẫu nhiên* từ 0 tới mức backoff. Cùng số lần retry đó, nhưng trải đều ra.

**[→]** Đường cam là sức chịu của vendor. Cột đỏ là vượt quá. Tổng cộng vẫn 600 lần retry, nhưng đỉnh giảm mạnh. (Đây là mô phỏng nhỏ tính ngay trong trang, không phải dữ liệu thật.)

## Slide 33 / 47 · Version 6 (sơ đồ)

⏱ 1 phút · rời slide lúc 1:19

Version 6: dưới payment vendor có nhãn "timeout · breaker · pool".

**[→]** Vấn đề số 7, ở cửa vào API: *flash sale.*

**[→]**

---

# VẤN ĐỀ 7 · FLASH SALE (1:19 – 1:24)

## Slide 34 / 47 · Problem 7 · A flash sale

⏱ 1 phút · rời slide lúc 1:20

Lần này không phải ai khác chậm, mà chính mình bị quá tải.

**[→]** Khách hàng thấy gì: trang web ổn ở 200 order mỗi giây, nhưng không dùng nổi ở 260.

**[→]** Chuyện thực tế: pool database 20 kết nối bão hòa ở khoảng 240 request mỗi giây. Request dồn vào một hàng đợi không giới hạn cho tới khi pod hết bộ nhớ, và mọi thứ trong hàng đợi mất cùng lúc.

**[→]** Vì sao dễ bị bỏ sót: độ trễ phẳng lì cho tới "điểm gãy", nên dashboard trông khỏe cho tới đúng khoảnh khắc nó không còn khỏe.

## Slide 35 / 47 · Backpressure

⏱ 3 phút · rời slide lúc 1:23

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Backpressure:** Khi quá tải thì đẩy áp lực ngược về bên gọi (từ chối, bảo quay lại sau) thay vì nhận hết mọi request.
> - **Knee (điểm gãy):** Mức tải mà từ đó độ trễ tăng vọt, vì một tài nguyên đã đầy.
> - **Saturation (bão hòa):** Tài nguyên đã dùng gần hết công suất, ví dụ pool đang dùng 20/20 kết nối.
> - **Hàng đợi trong bộ nhớ (unbounded queue):** Khác với SQS! Đây là danh sách việc chờ nằm trong RAM của pod. Không giới hạn thì nó cứ dài ra tới khi hết RAM; pod chết thì mất hết.
> - **429 / 503 / Retry-After:** 429 Too Many Requests: "bạn gửi nhiều quá". 503 Service Unavailable: "server đang quá tải". Retry-After: header báo client sau bao nhiêu giây thì thử lại.
> - **Load test / k6:** Giả lập thật nhiều người dùng cùng lúc để xem hệ thống chịu được tới đâu. k6 là một công cụ làm việc này.
> - **Runbook:** Tài liệu hướng dẫn từng bước xử lý khi một alert cụ thể kêu.

**[→]** Đây là độ trễ p99 khi tải tăng dần. Phẳng, phẳng, phẳng, rồi vọt thẳng lên. Điểm đó gọi là **knee**, điểm gãy.

**[→]** Ở đây là 240 request mỗi giây, khi pool database đầy. *Con số này minh họa thôi; k6 chạy trên service thật sẽ cho các bạn con số thật.* Mọi request sau đó phải chờ trong hàng đợi, và hàng đợi cứ dài ra. Một hàng đợi không giới hạn sẽ dài cho tới khi pod hết bộ nhớ.

**[→]** Có giới hạn: khi đầy, từ chối ngay với 429 hoặc 503, kèm header `Retry-After` báo khi nào quay lại. Request bị từ chối lỗi trong vài mili giây. Mọi người còn lại vẫn nhanh. *Nói "không" nhanh còn hơn nói "có" chậm.*

Làm sao biết điểm gãy của mình? Load test bằng công cụ như k6: tăng dần traffic, xem độ trễ nhảy ở đâu, và tìm giới hạn nào gây ra nó. Con số đó ghi vào runbook.

**Ví dụ dễ hiểu:** một nhà hàng đông khách nói "hết chỗ rồi, 8 giờ quay lại nhé" vẫn tốt hơn nhà hàng nhận hết khách rồi nửa đêm mới dọn món.

**Nếu bị hỏi** *"Sao không autoscale thêm pod?"*: HPA mất vài phút mới bật được pod mới, flash sale thì dồn trong vài giây. Mỗi pod thêm lại mở 20 kết nối, nên sớm muộn chạm `max_connections` của Postgres. Thêm pod chỉ dời điểm gãy ra xa chứ không xóa được nó, nên vẫn cần backpressure.

## Slide 36 / 47 · Version 7 (sơ đồ)

⏱ 1 phút · rời slide lúc 1:24

Version 7: trên đường client tới API có nhãn "429 if full".

**[→]** Vấn đề số 8, ngay trên API: *mỗi ngày mình deploy 10 lần.*

**[→]**

---

# VẤN ĐỀ 8 · DEPLOY (1:24 – 1:29)

## Slide 37 / 47 · Problem 8 · We deploy ten times a day

⏱ 1 phút · rời slide lúc 1:25

**[→]** Khách hàng thấy gì: vài lỗi, vài email đến trễ hoặc bị trùng, mỗi lần mình ship.

**[→]** Chuyện thực tế: Kubernetes gửi `SIGTERM`. App thoát ngay lập tức, hoặc lờ nó đi cho tới khi bị `SIGKILL` ở giây 30. Request đang chạy, lô message đang xử lý, các claim đang giữ: tất cả bị cắt ngang.

**[→]** Vì sao dễ bị bỏ sót: *mỗi lần deploy là một lần shutdown.* Mất một chút mỗi lần, nhân 10 lần một ngày thì thành nhiều.

## Slide 38 / 47 · Graceful shutdown

⏱ 3 phút · rời slide lúc 1:28

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Graceful shutdown:** Tắt "lịch sự": ngừng nhận việc mới, làm nốt việc đang dở, rồi mới thoát.
> - **SIGTERM / SIGKILL:** Tín hiệu hệ điều hành gửi cho tiến trình. SIGTERM là "làm ơn dừng": code còn kịp phản ứng. SIGKILL là "dừng ngay": không chạy thêm dòng code nào nữa.
> - **preStop hook:** Lệnh Kubernetes chạy trước khi gửi SIGTERM, thường là sleep vài giây.
> - **Load balancer:** Bộ phân phối traffic tới các pod. Nó cần vài giây để biết pod đang tắt và ngừng gửi request mới tới pod đó.
> - **terminationGracePeriodSeconds:** Thời gian Kubernetes chờ trước khi gửi SIGKILL, mặc định 30 giây.
> - **Exit 0:** Thoát với mã 0, nghĩa là kết thúc bình thường, không có lỗi.

**[→]** Kubernetes xóa pod. Một `preStop` sleep 5 giây cho load balancer kịp ngừng gửi traffic mới tới mình. Hết 5 giây đó app mới nhận `SIGTERM`. (Trên slide, mốc ở giây 0 là "pod deleted", không phải SIGTERM.)

**[→]** Rồi từng vòng lặp làm nốt việc của mình. HTTP server xử lý xong các request đang dở. Consumer ngừng nhận và làm xong lô hiện tại. Relay ngừng claim và publish nốt những gì đã claim.

**[→]** Đóng pool database, thoát với mã 0. Ở giây 9,5 là xong. Kubernetes sẽ gửi `SIGKILL` ở giây 30. Cái gì còn chạy lúc đó bị cắt.

Nhưng nhờ dedupe và claim, một lần bị kill đột ngột chỉ làm công việc *trễ*, không làm nó *mất*. Graceful shutdown giúp mỗi lần deploy êm thay vì ồn ào.

**Nếu bị hỏi về thứ tự chính xác:** hook `preStop` chạy trước, rồi container mới nhận `SIGTERM`; 30 giây tính từ lúc bắt đầu, gồm cả thời gian preStop.

**Nếu bị hỏi** *"Health check có nên gọi DB hay vendor không?"*: Liveness thì không. Liveness chỉ trả lời "tiến trình còn sống không"; nếu nó phụ thuộc DB hay vendor, một phụ thuộc chậm sẽ làm Kubernetes restart cả loạt pod khỏe (đúng chuyện ở slide 30). Readiness mới là chỗ báo "tạm thời đừng gửi traffic cho tôi".

## Slide 39 / 47 · Version 8 (sơ đồ)

⏱ 1 phút · rời slide lúc 1:29

Version 8: góc dưới bên trái có khung "Every pod, every deploy": SIGTERM, ngừng nhận, làm nốt, thoát.

**[→]** Vấn đề cuối cùng, số 9, ở relay: *có thứ ngừng chạy, và không ai biết.* Quay lại lúc 3 giờ 7 phút sáng.

**[→]**

---

# VẤN ĐỀ 9 · HỎNG MÀ KHÔNG AI BIẾT (1:29 – 1:41)

## Slide 40 / 47 · Problem 9 · Something stops, and nobody notices

⏱ 2 phút · rời slide lúc 1:31

Đây chính là bí ẩn lúc đầu buổi.

**[→]** Khách hàng thấy gì: 3 giờ 7 sáng, khách báo không nhận được email xác nhận.

**[→]** Dashboard cho thấy gì: mọi thứ xanh. Độ sâu queue SQS phẳng gần 0. Số dòng outbox có `sent_at IS NULL` tăng dần từ lần deploy lúc 2 giờ 10, mà không có alert nào cho nó.

**[→]** Đến lượt các bạn: cái gì đã ngừng chạy? Tại sao không alert nào về queue kêu? Một alert nào sẽ báo được trong 5 phút?

**Hỏi:** để các bạn thảo luận nhanh 1 phút. Nhìn lại sơ đồ nhỏ, chấm đỏ đang ở relay. **Chưa xác nhận đáp án**; đáp án ở slide 41.

## Slide 41 / 47 · Observability

⏱ 7 phút (gồm lời giải 3 giờ sáng) · rời slide lúc 1:38

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **Observability:** Khả năng nhìn từ bên ngoài mà hiểu hệ thống đang làm gì, nhờ log, metric và trace.
> - **Structured log (log JSON):** Mỗi dòng log là một object JSON với các trường cố định, nên tìm kiếm và lọc được, ví dụ lọc theo request_id.
> - **Request id:** Id duy nhất cho mỗi request, gắn vào mọi dòng log của request đó để gom chúng lại với nhau.
> - **MDC:** Chỗ lưu thông tin theo từng request (request id, org id…) mà thư viện log tự động gắn vào mọi dòng log.
> - **Metric:** Con số được đo liên tục theo thời gian, ví dụ số request mỗi giây, tỉ lệ lỗi, độ trễ, và vẽ thành biểu đồ.
> - **RED:** Rate (số request mỗi giây), Errors (số hoặc tỉ lệ lỗi), Duration (độ trễ). Ba metric cơ bản của một service.
> - **Trace / span:** Trace là toàn bộ hành trình của một request qua các hệ thống. Span là một đoạn trong hành trình đó (một truy vấn DB, một lời gọi vendor), có thời gian bắt đầu và kết thúc.
> - **OpenTelemetry:** Bộ công cụ chuẩn để thu thập trace, metric và log.
> - **Triệu chứng vs nguyên nhân:** Triệu chứng là thứ người dùng cảm nhận được: lỗi, chậm. Nguyên nhân là lý do bên trong, ví dụ CPU cao. Alert nên dựa trên triệu chứng.
> - **SLO:** Mục tiêu chất lượng của service, ví dụ "99,9% request thành công trong dưới 500 ms".
> - **Outbox lag:** Tuổi của dòng outbox chưa gửi cũ nhất. Con số này cứ lớn dần nghĩa là relay không gửi được.

Để trả lời được, mình cần biết hệ thống đang làm gì. Ba loại tín hiệu, ba câu hỏi.

**Log** trả lời: chuyện gì đã xảy ra với *request này*? JSON có cấu trúc, mỗi sự kiện một dòng. Dòng nào cũng có request id, org id, user id, được tự động gắn vào từ MDC. Các bạn cứ hình dung MDC như một "nhãn dán" theo từng request mà logger dán lên mọi dòng log. Lỗi thì log kèm exception và stack trace. Không log mật khẩu, token, hay dữ liệu cá nhân không cần thiết.

**Metric** trả lời: service có khỏe *ngay lúc này* không? Rate, errors, duration, gọi tắt là RED, cho từng endpoint và từng consumer. Dùng percentile như p99, không dùng trung bình: trung bình có thể che mất 1% người dùng phải chờ 10 giây. Thêm độ bão hòa: pool đầy bao nhiêu, message cũ nhất trong queue bao lâu, DLQ bao nhiêu, và *outbox lag*.

**Trace** trả lời: thời gian đi đâu hết? Một trace id đi theo request qua API, database, vendor và consumer. Làm được vậy là vì mình đặt trace context trong HTTP header *và trong cả message*.

**[→]** Cái gì nên đánh thức người trực lúc nửa đêm? *Triệu chứng, không phải nguyên nhân.* Tỉ lệ lỗi trên 1% trong 5 phút. p99 vượt mục tiêu. DLQ không rỗng. Message cũ nhất trong queue quá 10 phút. Và *dòng outbox chưa gửi cũ nhất quá 5 phút.* Không phải "CPU 80%". CPU cao có thể vẫn ổn. Người dùng bị ảnh hưởng thì không bao giờ ổn.

**Lời giải 3 giờ sáng (nói ngay tại đây, không có slide riêng).** Chỉ vào dòng outbox trên thẻ, rồi hỏi: *"Giờ các bạn đã thấy toàn bộ đường đi. Chuyện gì đã xảy ra lúc 3 giờ sáng?"* Chờ. Câu mong đợi: "Relay ngừng chạy. Không có gì được publish, nên queue rỗng, mà queue rỗng thì trông khỏe."

Chỉ lại các câu đoán đã ghi trên bảng ở slide 3: *"Cách nào trong số này sẽ tìm ra được?"*

Chính xác. Relay ngừng chạy lúc deploy. Queue rỗng vì *không có gì được publish cả*. Một queue rỗng trông giống hệt một queue khỏe, nên mọi alert về queue đều im lặng.

*Vì sao relay ngừng?* Vòng lặp relay chạy trong một `launch { }`. Bản deploy mới gặp một exception không ai bắt, vòng lặp chết. Pod vẫn trả lời `/health` bình thường, vì health check chỉ kiểm tra HTTP server, không kiểm tra vòng lặp worker. Nối lại: `launch { }` ở slide 7, try/catch cho từng message ở slide 24, và mục "silent relay" sắp gặp ở slide 46.

Với alert "dòng outbox chưa gửi cũ nhất quá 5 phút", người trực sẽ bị gọi lúc **2 giờ 15**. Khách phát hiện lúc 3 giờ 7, tức là sớm hơn **52 phút**. Cách sửa chỉ là một câu query và một luật alert. Bài học lớn hơn: *mỗi chặng chạy ngầm cần một tín hiệu riêng cho câu hỏi "việc có đang bị dồn ở đây không?".* Outbox lag cho relay. Tuổi message trong queue cho consumer. Kích thước DLQ cho poison message.

(Nếu bạn dùng một sự cố thật của team ở slide 3, kể kết thúc thật ở đây.)

**[→]** Alert nào cũng phải có runbook và có người chịu trách nhiệm. Alert không ai xử lý sẽ bị tắt tiếng trong vòng một tháng. Ít alert, nhưng cái nào cũng thật.

## Slide 42 / 47 · One trace, end to end

⏱ 2 phút · rời slide lúc 1:40

4,2 giây đã đi đâu?

**[→]** Request API: 180 mili giây, hai span database ngắn. Người dùng nhận 201 ở đây. Nhanh.

**[→]** Rồi event chờ khoảng 1 giây cho lần poll tiếp theo của relay; đó là cái giá của polling. Rồi publish 60 mili giây, và 30 mili giây trong queue.

**[→]** Rồi consumer dedupe và ghi một dòng `email_outbox`, rất nhanh. Rồi email sender gọi email vendor: 2,83 giây. *Thời gian đi vào đây.* Consumer không gọi vendor trong transaction; nó chỉ ghi một dòng, đúng quy tắc "việc quan trọng thì phải là một dòng".

**[→]** Và đây là một trace duy nhất, từ lúc `POST` tới lúc gửi email, vì `traceparent` đi theo trong message SQS và trong dòng `email_outbox`. Không có nó, mình sẽ thấy hai trace rời rạc và phải đoán.

## Slide 43 / 47 · Version 9 (sơ đồ)

⏱ 1 phút · rời slide lúc 1:41

Đây là version 9: vẫn cửa hàng đó, nhưng giờ đã chịu được sự cố. Dải "Observability" ở dưới cùng là phần mới.

**Hỏi (nếu còn thời gian):** chỉ vào vài chấm xanh và hỏi: *"Số 4 là vấn đề gì? Mình sửa bằng cái gì?"*

So với version 0, sơ đồ lớn hơn nhiều. Nhưng mỗi mảnh đều có lý do: mỗi mảnh là câu trả lời cho một sự cố cụ thể.

**[→]** Tiếp theo: theo một order đi qua toàn bộ hệ thống.

---

# TỔNG KẾT (1:41 – 2:00)

## Slide 44 / 47 · One order, end to end

⏱ 2 phút · rời slide lúc 1:43

Đi nhanh, vì các bạn đã biết từng mảnh:

**[→]** Request: key, order và dòng outbox trong một transaction. Rồi 201.

**[→]** Relay lấy dòng outbox và publish lên SQS.

**[→]** Consumer dedupe trong `processed`, ghi một dòng `email_outbox`, commit, ack. Một email sender gửi dòng đó kèm idempotency key cho email vendor, nên email không mất mà cũng không bị gửi hai lần.

**[→]** Một poison message: sau 5 lần nhận, chuyển sang DLQ, và alarm kêu.

*Một dòng bền vững trong request, rồi at-least-once suốt phần còn lại.* Mỗi chặng có thể làm lặp lại, nhưng không chặng nào được làm mất.

## Slide 45 / 47 · The numbers to remember

⏱ 1 phút · rời slide lúc 1:44

Mỗi con số một câu, đi nhanh:

- 27 lời gọi: một lần bấm, ba tầng mỗi tầng thử 3 lần. Chỉ retry ở một chỗ phía server.
- 75 request cùng lúc: 25 request mỗi giây *trên một pod* nhân 3 giây. Pool 20 không chứa nổi, nên vendor có pool riêng.
- 10, 3, 1 giây: timeout chặng trong ngắn hơn chặng ngoài.
- 30 giây: thời hạn claim của relay.
- 5 lần nhận: rồi vào DLQ.
- 24 giờ: thời gian giữ idempotency key.
- 1% trong 5 phút: tỉ lệ lỗi thì gọi người trực.
- 10 phút: tuổi queue thì gọi người trực.
- 5 phút: dòng outbox chưa gửi thì gọi người trực.

Đây là các con số ví dụ. Service thật của các bạn có thể khác; cách suy luận thì giống.

## Slide 46 / 47 · Ten reliability mistakes review sees most

⏱ 2 phút · rời slide lúc 1:46

> **Khái niệm cần biết** · để bạn hiểu, không cần đọc to
>
> - **PR (pull request) / code review:** PR là đề xuất đưa một thay đổi code vào nhánh chính. Review là đồng nghiệp đọc và góp ý trước khi merge.

Đây là checklist khi review code. Khi các bạn mở PR có đụng tới queue, gọi vendor, hoặc việc chạy ngầm, hãy đọc danh sách này trước.

Lướt nhanh, vì tất cả đều đã gặp trong 9 vấn đề. Có thể nối từng lỗi với số vấn đề: dual write và `launch { }` là vấn đề 1; ack first là vấn đề 5; no dedupe là vấn đề 4; batch stops là vấn đề 5; no timeout và retry everywhere là vấn đề 6; no DLQ là vấn đề 5; silent relay là vấn đề 9; no context là vấn đề 9.

**Hỏi:** *"Giơ tay nhé: lỗi nào các bạn nghĩ mình dễ viết nhất?"* Thường `launch { }` và "no timeout" nhiều tay nhất. Nói rằng chuyện đó hoàn toàn bình thường.

Các bạn không cần nhớ hết mười lỗi. Lưu slide này lại. Review sẽ bắt giúp các bạn, và mọi người đều học những điều này theo cách đó.

## Slide 47 / 47 · Close

⏱ 14 phút (Q&A là chính) · rời slide lúc 2:00

Bốn câu mang về nhà:

- *If it matters, it's a row.* Việc gì quan trọng thì phải là một dòng trong database.
- *Dedupe, commit, ack.* Loại trùng, commit, rồi mới ack.
- *Fail fast and alone.* Lỗi nhanh, và chỉ lỗi một mình.
- *Page on symptoms.* Gọi người trực theo triệu chứng, không theo nguyên nhân.

Muốn đọc kỹ hơn: cuốn *Release It!* của Michael Nygard, và trang transactional outbox trên microservices.io.

Và dòng cuối slide: *hỏi bất kỳ ai, bất cứ điều gì, đừng giấu rồi đoán.* Câu này đúng cho buổi hôm nay, và đúng cho cả kỳ thực tập của các bạn. Giờ là phần hỏi đáp. Mỗi bạn intern hỏi ít nhất một câu nhé.

**Nếu chưa ai hỏi**, gợi ý:

1. *"Trong mười lỗi đó, lỗi nào sẽ cắn service của mình đầu tiên?"*
2. *"Nếu email vendor sập một tiếng thì hệ thống của mình ra sao?"* Đáp: breaker mở, message lỗi và được retry, một số có thể vào DLQ. Không mất gì, vì công việc vẫn nằm trong queue hoặc DLQ. Vendor lên lại thì redrive.
3. *"Nếu tối nay các bạn trực, các bạn mở dashboard nào đầu tiên?"*

Các câu hỏi hay gặp khác có câu trả lời soạn sẵn trong `03-qa-and-exercises.md`.

**Kết:** Cảm ơn các bạn. Deck ở [link]. Tối nay nếu nghĩ ra câu hỏi, cứ nhắn mình trên Slack; như vậy cũng tính là đã hỏi.

---

# Tra cứu khái niệm

Khái niệm nào được giải thích ở slide nào.

| Khái niệm | Slide |
| --- | --- |
| 202 Accepted | 31 |
| 429 / 503 / Retry-After | 35 |
| Ack (acknowledge) | 20 |
| Alert / page | 03 |
| API / endpoint | 04 |
| At-least-once (ít nhất một lần) | 09 |
| Backoff | 28 |
| Backpressure | 35 |
| Batch (lô) | 16 |
| Bulkhead | 28 |
| Cache | 31 |
| CDC (change data capture) / Debezium | 16 |
| Circuit breaker | 28 |
| Claim / lease | 16 |
| Commit / rollback | 06 |
| Connection pool (pool kết nối DB) | 04 |
| Consumer / producer | 09 |
| Coroutine / `launch { }` | 07 |
| Dashboard | 03 |
| Database / Postgres | 05 |
| Deadline | 31 |
| Dedupe (loại trùng) | 09 |
| Dependency (hệ thống phụ thuộc) | 02 |
| Deploy | 02 |
| Distributed transaction / two-phase commit (2PC) | 08 |
| DLQ (dead-letter queue) | 20 |
| Dual write | 08 |
| Event (sự kiện) | 08 |
| Event id | 09 |
| Exception / try-catch | 24 |
| Exit 0 | 38 |
| Fail fast | 28 |
| Fallback | 29 |
| FIFO queue | 16 |
| Graceful shutdown | 38 |
| Happy path | 08 |
| Header | 12 |
| Health check | 27 |
| HTTP status code | 04 |
| Hàng đợi trong bộ nhớ (unbounded queue) | 35 |
| Idempotency key | 12 |
| Idempotent | 12 |
| In flight | 28 |
| Jitter | 28 |
| Knee (điểm gãy) | 35 |
| Kubernetes | 04 |
| Latency (độ trễ) | 04 |
| Little's law | 28 |
| Load balancer | 38 |
| Load test / k6 | 35 |
| Long polling | 20 |
| maxReceiveCount | 20 |
| MDC | 41 |
| Metric | 41 |
| Node drain | 06 |
| Observability | 41 |
| On-call (trực) | 03 |
| OOM kill | 06 |
| OpenTelemetry | 41 |
| Org id | 09 |
| Outbox | 03 |
| Outbox lag | 41 |
| p99 | 04 |
| Payload | 09 |
| Pod | 02 |
| Poison message | 20 |
| Polling | 16 |
| PR (pull request) / code review | 46 |
| preStop hook | 38 |
| Production | 02 |
| Publish | 09 |
| Queue (hàng đợi) | 03 |
| RED | 41 |
| Redrive | 20 |
| Relay | 09 |
| Reliability (độ tin cậy) | 01 |
| Request / response | 13 |
| Request id | 41 |
| Retry | 04 |
| Retry amplification | 31 |
| Row lock | 15 |
| Runbook | 35 |
| Saturation (bão hòa) | 35 |
| SIGTERM / SIGKILL | 38 |
| SKIP LOCKED | 16 |
| SLO | 41 |
| Source of truth | 08 |
| SQS | 03 |
| Stack trace | 24 |
| Structured log (log JSON) | 41 |
| Sập vs chậm (down vs slow) | 27 |
| terminationGracePeriodSeconds | 38 |
| Thread pool / executor | 24 |
| Thundering herd | 32 |
| Timeout | 04 |
| Trace / span | 41 |
| traceparent | 20 |
| Transaction | 06 |
| Transactional outbox | 09 |
| Triệu chứng vs nguyên nhân | 41 |
| Type / version | 20 |
| Unique constraint | 12 |
| Vendor | 02 |
| Visibility timeout | 20 |
| Worker | 07 |
| Độ sâu queue (queue depth) | 03 |
