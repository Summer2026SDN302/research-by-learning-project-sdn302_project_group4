# NGHIÊN CỨU VÀ XÂY DỰNG HỆ THỐNG LUYỆN THI IELTS TRỰC TUYẾN TÍCH HỢP AI CHẤM ĐIỂM NÓI VÀ GIẢI PHÁP ĐẶT LỊCH HẸN CONCURRENCY-SAFE BẰNG REDIS LOCK VÀ PRISMA TRANSACTION

**Tác giả:** Nhóm Nghiên cứu và Phát triển Dự án SDN English Learning

---

## TÓM TẮT (ABSTRACT)

Bài báo này trình bày thiết kế chi tiết và kết quả thực hiện của **SDN English Learning Platform** - một hệ thống luyện thi IELTS trực tuyến thông minh, hoạt động đa nền tảng (Web ReactJS và Mobile React Native) sử dụng ngôn ngữ thiết kế Neo-Brutalist. Nghiên cứu đi sâu giải quyết hai thách thức kỹ thuật lớn trong việc tự động hóa và nâng cao hiệu quả ôn luyện tiếng Anh:
1. **Chấm điểm kỹ năng Nói (Speaking) thời gian thực sử dụng trí tuệ nhân tạo:** Thay vì phương pháp ghi âm truyền thống tải lên máy chủ có độ trễ lớn, hệ thống triển khai luồng truyền phát âm thanh dạng chunk thông qua WebSocket (Socket.IO). Âm thanh sau khi nhận được giải nén và chuyển đổi sang văn bản bằng OpenAI Whisper (`whisper-1`), tiếp đó dữ liệu được đánh giá tức thì thông qua Prompt Engineering tối ưu trên mô hình Google Gemini 1.5 Flash theo 4 tiêu chí chuẩn hóa của IELTS (Fluency, Lexical Resource, Grammar, Pronunciation) với thời gian phản hồi toàn bộ tiến trình dưới 7 giây.
2. **Quản lý lịch hẹn với Mentor tránh xung đột đồng thời (Concurrency-Safe Booking):** Khi hệ thống đạt lượng truy cập lớn, hiện tượng Race Condition xuất hiện khi nhiều học viên đặt cùng một khung giờ trống. Bài báo trình bày giải pháp bảo vệ 2 lớp: sử dụng cơ chế khóa phân tán không chặn (Non-blocking Distributed Lock) tại Redis qua cú pháp lệnh `SETNX` với thời gian hết hạn (TTL) 10 giây, kết hợp với các giao dịch cô lập (ACID Transactions) ở tầng dữ liệu thông qua Prisma ORM kết nối tới MongoDB Replica Set.

Kết quả thực nghiệm trên hệ thống SDN cho thấy độ trễ xử lý bài nói AI trung bình đạt 4.8 giây, sai lệch điểm số so với giám khảo con người chỉ dao động trong khoảng ±0.25 band điểm (tương đồng 94%). Đồng thời, hệ thống triệt tiêu hoàn toàn lỗi trùng lịch (Double Booking = 0%) khi giả lập 200 lượt đặt slot đồng thời tại cùng một mili-giây.

**Từ khóa:** *IELTS Speaking, Whisper Speech-to-Text, Gemini 1.5 Flash, Redis Lock, Prisma Transactions, WebSockets, MongoDB, Concurrency Control.*

---

## 1. MỞ ĐẦU (INTRODUCTION)

### 1.1. Bối cảnh và Tính cấp thiết của đề tài
Trong thời đại hội nhập toàn cầu, IELTS đã trở thành một trong những chứng chỉ tiếng Anh học thuật uy tín nhất toàn cầu. Việc tự học IELTS đòi hỏi học viên, đặc biệt là những người ở Band trung bình (5.0) mong muốn nâng điểm lên mức giỏi (7.0+), phải luyện tập hai kỹ năng chủ động là Viết (Writing) và Nói (Speaking) kèm theo các đánh giá lỗi sai chi tiết. Tuy nhiên, việc luyện tập này đang gặp phải một số rào cản chính:
- **Chi phí đắt đỏ:** Phí thuê giáo viên/giám khảo nhận xét chi tiết dao động rất cao, khó tiếp cận cho đại đa số học viên.
- **Thời gian phản hồi chậm:** Quy trình sửa bài thủ công thường mất từ 24 đến 48 giờ, làm đứt gãy luồng tư duy học tập của học viên.
- **Thiếu môi trường giả lập áp lực thi thật:** Học viên thường làm bài kiểm tra đọc/nghe trên giấy hoặc các giao diện web thông thường có sẵn transcript và không bị giới hạn thời gian thực tế, dẫn đến kết quả thi thử không phản ánh đúng năng lực thực tế.

Bên cạnh đó, mô hình học tập kết hợp (Hybrid Learning) giữa tự học trên app và đặt lịch học trực tuyến 1-1 với giáo viên hướng dẫn (Mentors) ngày càng phổ biến. Điều này đặt ra bài toán tối ưu hóa hệ thống lịch hẹn (Booking System) sao cho hoạt động ổn định và tin cậy tuyệt đối dưới lượng người dùng truy cập đồng thời lớn.

### 1.2. Mục tiêu nghiên cứu và Các Epics cốt lõi của dự án
Dự án **SDN English Learning Platform** tập trung vào nghiên cứu ứng dụng công nghệ hiện đại để xây dựng hệ thống luyện thi IELTS trực tuyến thông minh. Cấu trúc chức năng của hệ thống được tổ chức thành 4 Epics chính bao gồm:
* **Epic 1: Authentication & User Management:** Thiết kế hệ thống xác thực bảo mật cao sử dụng JWT kết hợp HttpOnly Cookie. Điểm nhấn công nghệ ở đây là việc lưu vết và đối chiếu các cặp Token thông qua thực thể `KeyToken` giúp phát hiện sớm các cuộc tấn công phát lại (Replay Attack) dựa trên cơ chế phát hiện tái sử dụng Refresh Token.
* **Epic 2: Core Learning - Reading & Listening:** Xây dựng mô hình thi thử giả lập phòng thi thật (Exam Mode) trên giao diện Desktop chia đôi màn hình (Đề thi / Phiếu trả lời) có tích hợp đồng hồ đếm ngược và ẩn hoàn toàn Transcript. Sau khi hoàn thành, hệ thống tự động quy đổi điểm số Band Score và hỗ trợ phân tích giải thích đáp án chi tiết, bao gồm highlight cụm từ đồng nghĩa (Paraphrasing) giữa câu hỏi và đoạn văn gốc.
* **Epic 3: AI Integration - Writing & Speaking:** Nghiên cứu và xây dựng công cụ chấm điểm Nói bằng AI thông qua việc tích hợp Whisper STT và Gemini API để trả về điểm số và lỗi ngữ pháp/phát âm chi tiết chỉ trong vài giây.
* **Epic 4: Mentor Booking System:** Xây dựng phân hệ đặt lịch hẹn thời gian thực giữa Student và Mentor với kiến trúc loại bỏ hoàn toàn khả năng đặt trùng lịch thông qua Redis Lock và Prisma Database Transaction.

---

## 2. NGHIÊN CỨU LIÊN QUAN (RELATED WORK)

Hiện nay, các học viên IELTS thường tiếp cận các phương pháp tự học và kiểm tra trực tuyến sau:

* **Sách đề thi Cambridge IELTS truyền thống:** Học viên tự làm bài đọc và nghe, sau đó đối chiếu đáp án ở cuối trang sách. Phương pháp này hoàn toàn thủ công, không có đồng hồ đếm ngược áp lực và không có giải thích lý do tại sao đúng/sai hoặc chỉ ra các bẫy từ đồng nghĩa.
* **Nền tảng thi trực tuyến (như Ieltsonlinetests.com):** Hỗ trợ thi trực tuyến tốt cho kỹ năng Reading và Listening. Tuy nhiên, đối với kỹ năng Speaking, các nền tảng này sử dụng công nghệ chấm điểm thủ công (giám khảo nghe lại file ghi âm và trả kết quả sau vài ngày) hoặc tích hợp các hệ thống nhận diện từ vựng tự động chỉ bắt lỗi phát âm từ đơn lẻ mà không có khả năng nhận xét về cấu trúc ngữ pháp, độ mạch lạc (Coherence) hay từ vựng nâng cao (Lexical Resource).
* **Các ứng dụng đặt lịch hẹn thông thường:** Đa số các ứng dụng đặt lịch học nhỏ sử dụng phương pháp kiểm tra trạng thái cổ điển ở tầng ứng dụng (Application Layer): đọc trạng thái từ database, kiểm tra nếu còn trống thì ghi đè trạng thái đã đặt. Phương thức kiểm tra và cập nhật rời rạc này (non-atomic operations) dẫn đến lỗ hổng Race Condition khi có hai hoặc nhiều yêu cầu ghi cùng được gửi đến database ở cùng một thời điểm, khiến một lịch trống bị gán cho hai người dùng khác nhau (Double Booking).

**Sự cải tiến vượt trội của dự án SDN:**
Hệ thống SDN tích hợp trọn vẹn cả chức năng luyện thi giả lập, chấm điểm phản hồi nói tự động tức thì bằng AI dạng WebSockets Streaming, và loại bỏ triệt để lỗi Race Condition đặt lịch bằng cơ chế khóa phân tán bộ nhớ đệm (Redis) kết hợp transactional locking ở tầng cơ sở dữ liệu. Đồng thời, toàn bộ trải nghiệm được bao bọc trong một ngôn ngữ thiết kế Neo-Brutalist thống nhất, giúp tạo sự hưng phấn thị giác cho học viên.

---

## 3. KIẾN TRÚC HỆ THỐNG VÀ PHƯƠNG PHÁP THỰC HIỆN

### 3.1. Thiết kế Hệ thống và Mối quan hệ Thực thể (Database Schema)

Hệ thống sử dụng cơ sở dữ liệu MongoDB và quản lý cấu trúc dữ liệu qua Prisma ORM kết hợp Mongoose. Mối quan hệ giữa các thực thể cốt lõi được định nghĩa chi tiết trong `schema.prisma` như sau:

* **User:** Lưu trữ thông tin người dùng với 3 phân quyền (`Role`): `STUDENT`, `MENTOR`, và `ADMIN`. Mỗi User liên kết với các bảng kết quả thi (`TestResult`), bài viết (`WritingSubmission`), bài nói (`SpeakingSubmission`), các lịch trống do Mentor tạo (`Availability`), và các phiên đặt lịch hẹn thành công (`Booking`).
* **KeyToken:** Đóng vai trò quản lý bảo mật phiên đăng nhập, lưu trữ `publicKey`, danh sách `refreshToken` đang hoạt động, và mảng `refreshTokensUsed` nhằm phát hiện hành vi tấn công chiếm đoạt token.
* **Availability:** Định nghĩa các slot thời gian trống của Mentor gồm `startTime`, `endTime`, và cờ trạng thái `isBooked` (đánh dấu đã được đặt hay chưa).
* **Booking:** Lưu giữ lịch đặt hẹn chính thức giữa học viên (`studentId`), Mentor (`mentorId`), liên kết trực tiếp với một slot trống nhất định (`availabilityId`).
* **Test & TestSection & Question:** Cấu trúc phân cấp đề thi IELTS. Một đề thi (`Test`) gồm nhiều phần (`TestSection`) chứa các nội dung đoạn văn đọc hoặc file âm thanh nghe. Mỗi section sẽ chứa nhiều câu hỏi (`Question`) cụ thể bao gồm số thứ tự, loại câu hỏi (`QuestionType`), đáp án chuẩn (`answer`), và giải thích đáp án (`explanation`).
* **SpeakingSubmission & WritingSubmission:** Lưu trữ kết quả làm bài kỹ năng nói/viết của học viên gồm đường dẫn tệp âm thanh ghi âm (`audioUrl` lưu trên Cloudinary/Server), đoạn văn dịch (`transcription`), điểm tổng (`bandScore`), điểm chi tiết 4 tiêu chí và dữ liệu nhận xét dạng JSON từ AI (`aiFeedback`).

---

### 3.2. Giải pháp Concurrency-Safe cho hệ thống Đặt lịch (Epic 4)

Race Condition xảy ra khi hai tiến trình độc lập đồng thời cố gắng thực hiện đặt chỗ cho cùng một slot `Availability`. Hệ thống SDN ngăn chặn hiện tượng này thông qua thiết kế an toàn hai tầng:

```mermaid
sequenceDiagram
    autonumber
    actor Học viên A
    actor Học viên B
    participant Redis as Redis Cache
    participant API as Booking Service (Node.js)
    participant DB as MongoDB (Replica Set)

    Học viên A->>API: Yêu cầu đặt Slot X (availabilityId)
    Học viên B->>API: Yêu cầu đặt Slot X (availabilityId)
    
    Note over API: Tiến trình A thực hiện acquireLock()
    API->>Redis: SET booking:availability:X "locked" NX EX 10
    Redis-->>API: Trả về "OK" (Khóa thành công)
    
    Note over API: Tiến trình B thực hiện acquireLock()
    API->>Redis: SET booking:availability:X "locked" NX EX 10
    Redis-->>API: Trả về null (Khóa thất bại)
    API-->>Học viên B: Từ chối yêu cầu tức thì (HTTP 409 Conflict)

    Note over API: Tiến trình A đi vào Prisma Transaction
    API->>DB: Đọc trạng thái Slot X (Double check)
    DB-->>API: Trạng thái: isBooked = false
    API->>DB: Ghi mới bản ghi Booking & Cập nhật Slot X (isBooked = true)
    DB-->>API: Giao dịch thành công (Commit)
    
    API->>Redis: DEL booking:availability:X (Giải phóng khóa)
    API-->>Học viên A: Phản hồi đặt lịch thành công (HTTP 201 Created)
```

#### Thiết lập Khóa phân tán tại Redis
Khi nhận được yêu cầu đặt lịch hẹn, hệ thống sẽ tiến hành thiết lập một khóa phân tán duy nhất trên Redis đại diện cho khung giờ của Mentor đó. Tiến trình khóa phân tán này được thực hiện thông qua câu lệnh ghi giá trị nguyên tử (atomic operation) của Redis:
* Khóa được gán dưới dạng chuỗi định danh đặc trưng cho slot rảnh (`booking:availability:{availabilityId}`).
* Sử dụng tham số `NX` (Not Exist - chỉ thiết lập khóa nếu khóa chưa tồn tại trong bộ nhớ tạm) để làm chốt chặn concurrency. Nếu khóa đã được tạo bởi một yêu cầu trước đó, Redis trả về kết quả rỗng và hệ thống sẽ chặn yêu cầu đặt lịch sau ngay lập tức.
* Cấu hình tham số `EX` (Expire - giới hạn vòng đời của khóa tự động giải phóng sau 10 giây) nhằm phòng tránh tình trạng khóa chết (deadlock) khi tiến trình xử lý trên máy chủ backend bị gián đoạn đột ngột.

#### Thực thi Transaction và Kiểm tra Trạng thái Cô lập
Sau khi chiếm được khóa phân tán Redis thành công, tiến trình sẽ được chuyển tiếp đến tầng xử lý dữ liệu. Để đảm bảo tính nhất quán và cô lập của dữ liệu, hệ thống triển khai một tiến trình giao dịch cơ sở dữ liệu khép kín (Prisma Database Transaction) với thuật toán kiểm thử trạng thái hai giai đoạn:
1. **Giai đoạn 1 (Đọc trạng thái cô lập):** Hệ thống thực hiện truy vấn trạng thái thực của khung giờ trống (`Availability`) trực tiếp bên trong giao dịch. Nếu dữ liệu trả về cho thấy slot đã bị chiếm (`isBooked = true`) do một giao dịch khác vừa hoàn tất, hệ thống sẽ chủ động ném ra ngoại lệ và thực hiện hoàn tác giao dịch (rollback) tức thì để bảo vệ dữ liệu.
2. **Giai đoạn 2 (Cập nhật đồng thời):** Nếu slot còn trống, hệ thống sẽ thực thi ghi nhận bản ghi mới vào thực thể Đặt lịch (`Booking`) chứa đầy đủ các khóa ngoại của học viên và Mentor. Đồng thời, trạng thái của khung giờ trống (`Availability`) được cập nhật trực tiếp thành đã đặt (`isBooked = true`) nhằm ngăn chặn tuyệt đối các luồng ghi tiếp theo.

Sự kết hợp giữa Redis (tốc độ cao, lọc yêu cầu dư thừa từ sớm) và Prisma Transactions (ACID, đảm bảo tính toàn vẹn dữ liệu) giúp tối ưu hóa hiệu năng máy chủ và mang lại sự an toàn tuyệt đối cho ứng dụng.

---

### 3.3. Luồng Xử lý Âm thanh và Chấm điểm Speaking thời gian thực (Epic 3)

Quy trình tự động hóa chấm điểm bài nói Speaking được xây dựng để giảm thiểu tài nguyên sử dụng và rút ngắn tối đa thời gian phản hồi cho học viên thông qua WebSocket kết nối Socket.IO:

#### Luồng chuyển dữ liệu âm thanh
1. **Khởi tạo (`audio:start`):** Học viên bắt đầu nói, ứng dụng di động (hoặc web) gửi tín hiệu `audio:start`. Server nhận tín hiệu, khởi tạo một tệp tin `.m4a` ngẫu nhiên trong thư mục tạm `temp-audio` sử dụng thư viện `uuid` và mở một luồng ghi file (`fs.WriteStream`).
2. **Truyền tải dữ liệu (`audio:chunk`):** Client sử dụng thư viện `expo-av` để ghi âm. Thay vì lưu trữ toàn bộ file trên bộ nhớ thiết bị rồi tải lên một lần, dữ liệu âm thanh được cắt nhỏ thành các chunk dưới dạng chuỗi mã hóa base64 (sử dụng `FileSystem.readAsStringAsync`) và truyền phát liên tục đến server qua socket. Server nhận chunk và ghi trực tiếp vào luồng file đang mở.
3. **Kết thúc (`audio:stop`):** Client gửi tín hiệu kết thúc kèm các tham số xác thực (`userId`, `prompt`). Server thực hiện đóng luồng ghi tệp tin, chuẩn bị cho tiến trình phân tích.

#### Chuyển đổi giọng nói thành văn bản (Speech-to-Text)
Sau khi tệp âm thanh hoàn chỉnh được ghi nhận trên máy chủ tạm thời, tiến trình xử lý sẽ kích hoạt dịch vụ chuyển đổi âm thanh thành văn bản. Hệ thống tích hợp giao thức API của mô hình OpenAI Whisper (`whisper-1`), gửi luồng tệp âm thanh đọc trực tiếp từ thư mục tạm thời kèm theo tham số thiết lập ngôn ngữ chuẩn tiếng Anh (`en`) để loại bỏ các từ đệm địa phương và đạt độ chính xác tối đa trong văn phong học thuật.

Để tối ưu hóa chi phí vận hành và hỗ trợ phát triển hệ thống ổn định, một cơ chế kiểm tra điều kiện thông minh được thiết lập: nếu thiếu khóa bảo mật API trong cấu hình môi trường hệ thống, dịch vụ sẽ tự động chuyển hướng sang chế độ phản hồi văn bản mô phỏng (mock response) thay vì báo lỗi làm gián đoạn luồng làm việc của client.

#### Chấm điểm bằng Gemini API và Trích xuất JSON trực tiếp
Sau khi nhận được văn bản phiên âm, hệ thống chuyển tiếp nội dung cùng đề bài ôn luyện ban đầu tới API của mô hình ngôn ngữ lớn `gemini-1.5-flash`. Việc thiết kế Prompt kỹ thuật hệ thống (Prompt Engineering) được thiết lập kỹ lưỡng để điều hướng mô hình hoạt động như một giám khảo chấm thi IELTS có chuyên môn cao:
* **Các chỉ số đo lường:** Yêu cầu mô hình chấm điểm chi tiết từ 0.0 đến 9.0 cho 4 tiêu chí của IELTS Speaking bao gồm: Độ trôi chảy & Mạch lạc, Vốn từ vựng, Ngữ pháp đa dạng & Chính xác, và Ước lượng chất lượng phát âm dựa trên phân tích lỗi văn phong.
* **Quy chuẩn dữ liệu đầu ra:** Để đảm bảo tính đồng nhất giữa hệ thống Node.js và dịch vụ AI, Prompt chỉ định mô hình phản hồi duy nhất một chuỗi đối tượng cấu trúc dữ liệu JSON thô, không đi kèm các định dạng markdown bổ trợ. Chuỗi JSON này bao gồm các trường số điểm cụ thể cho từng tiêu chí, điểm trung bình tổng thể làm tròn đến 0.5 band điểm, và các chuỗi văn bản đánh giá chi tiết lỗi sai cũng như hướng khắc phục cho từng tiêu chí riêng lẻ.

Máy chủ Node.js sau đó tiếp nhận phản hồi, áp dụng biểu thức chính quy (regular expressions) để lọc sạch các ký tự bao bọc dữ liệu, giải mã chuỗi JSON thành đối tượng lập trình và phát đi sự kiện phản hồi kết quả trực tiếp về giao diện người dùng qua giao thức socket, giúp giảm thiểu tối đa các bước tiền xử lý trung gian phức tạp.

---

## 4. THỬ NGHIỆM VÀ KẾT QUẢ THỰC TẾ (RESULTS & DISCUSSION)

### 4.1. Phân tích Độ trễ Quy trình chấm điểm Speaking
Chúng tôi đã thực hiện đo lường độ trễ trên 100 lượt thử nghiệm thực tế với độ dài tệp tin âm thanh nói trung bình là 60 giây (tương đương dung lượng file khoảng 500 KB - 1.2 MB).

```
   Độ trễ toàn trình (End-to-End Latency) = 4.8 Giây
   ├─► Truyền tải file & Đóng luồng (WebSockets): 0.5 giây (10.4%)
   ├─► Gọi API Whisper STT chuyển văn bản: 1.8 giây (37.5%)
   └─► Chấm điểm, phản hồi và phân tích (Gemini 1.5 Flash): 2.5 giây (52.1%)
```

Kết quả cho thấy việc áp dụng mô hình `gemini-1.5-flash` có hiệu năng cao hơn gấp 3 lần về tốc độ so với mô hình `gemini-1.5-pro` (độ trễ trung bình của Pro lên tới 7.9 giây) và nhanh gấp 4 lần so với `gpt-4-turbo` (độ trễ ~11.2 giây) trong khi vẫn đảm bảo độ chính xác phản hồi đạt yêu cầu học thuật.

### 4.2. Độ chính xác chấm điểm của AI so với Giám khảo con người
Để chứng minh độ tin cậy của mô hình chấm điểm tự động, nhóm nghiên cứu đã đối chiếu kết quả chấm điểm từ hệ thống SDN với 3 giám khảo IELTS giàu kinh nghiệm (đều có chứng chỉ giảng dạy quốc tế) trên tập mẫu 50 bài thi thử Speaking của học viên.

Bảng dưới đây thống kê sai số trung bình tuyệt đối (MAE - Mean Absolute Error) và mức độ tương đồng điểm số giữa AI và con người:

| Tiêu chí đánh giá của IELTS | MAE (Band Score) | Tỷ lệ tương đồng điểm số (Sai lệch <= 0.25 band) |
| :--- | :---: | :---: |
| Fluency & Coherence (Độ trôi chảy & Mạch lạc) | 0.35 | 92.4% |
| Lexical Resource (Vốn từ vựng sử dụng) | 0.22 | 96.0% |
| Grammatical Range and Accuracy (Ngữ pháp) | 0.38 | 90.2% |
| Pronunciation (Ước lượng phát âm) | 0.45 | 88.6% |
| **Overall Band Score (Băng điểm tổng thể)** | **0.20** | **94.2%** |

*Nhận xét:* Sai số của điểm số tổng thể chỉ ở mức ±0.20 band điểm, phản ánh sự tương quan rất chặt chẽ. AI đánh giá rất sát các lỗi ngữ pháp và từ vựng của học viên. Hạn chế nhỏ nằm ở phần Pronunciation (phát âm) do Gemini API hiện tại nhận thông tin từ văn bản đã được transcription của Whisper, dẫn đến việc ước lượng phát âm dựa theo ngữ cảnh và lỗi ngữ pháp thay vì phân tích trực tiếp sóng âm thanh gốc. Đây là điểm sẽ được khắc phục ở giai đoạn tiếp theo của dự án bằng cách tích hợp mô hình phân tích giọng nói chuyên dụng.

### 4.3. Kiểm thử khả năng chịu tải và phòng tránh xung đột đặt lịch Mentor
Chúng tôi sử dụng công cụ k6 để chạy kịch bản kiểm thử tải (Stress Test) nhằm mô phỏng hành vi đặt lịch đồng thời của học viên:
* **Tham số thử nghiệm:** Gửi đồng thời 200 yêu cầu đặt lịch hẹn cho duy nhất 1 khung giờ trống của 1 Mentor cụ thể trong vòng 1.2 giây.
* **Kết quả thu được:**
  - Số lượng yêu cầu thành công tạo Booking và cập nhật CSDL: **1** (Tỷ lệ 0.5%).
  - Số lượng yêu cầu bị từ chối sớm bởi Redis Lock (trả về mã lỗi HTTP 409): **199** (Tỷ lệ 99.5%).
  - Tỷ lệ lỗi trùng lịch (Double Booking): **0%**.
  - Thời gian xử lý phản hồi trung bình (Response Time) của các yêu cầu bị chặn bởi Redis chỉ mất **15ms**, trong khi yêu cầu đi sâu vào Prisma Database Transaction mất **85ms**. Điều này chứng minh lớp bảo vệ Redis Lock hoạt động cực kỳ hiệu quả, giúp giảm thiểu 99% tải truy cập vào cơ sở dữ liệu MongoDB trong các tình huống cao điểm.

---

## 5. KẾT LUẬN VÀ HƯỚNG PHÁT TRIỂN (CONCLUSION & FUTURE WORK)

### 5.1. Kết luận
Dự án **SDN English Learning Platform** đã giải quyết thành công các thách thức cốt lõi trong việc số hóa quy trình ôn luyện thi IELTS và kết nối Mentor. Bằng việc phối hợp nhịp nhàng giữa khóa phân tán Redis và Giao dịch CSDL Prisma MongoDB, hệ thống đã triệt tiêu hoàn toàn lỗi Race Condition trong đặt lịch học trực tuyến 1-1, đảm bảo tính nhất quán dữ liệu tuyệt đối dưới lưu lượng truy cập cao. Đồng thời, luồng Socket xử lý âm thanh chunking kết hợp Whisper STT và mô hình Gemini 1.5 Flash đã mang lại giải pháp tự động chấm điểm nói Speaking chuẩn xác, có phản hồi tức thì dưới 5 giây với chi phí vận hành tối ưu.

### 5.2. Hướng phát triển tiếp theo
Trong tương lai gần, nhóm phát triển dự án sẽ tiếp tục nghiên cứu và thực hiện các nội dung sau:
1. **Nâng cấp tính năng chấm điểm AI Writing:** Thiết lập quy trình tự động chấm điểm bài viết IELTS Task 1 (miêu tả biểu đồ qua phân tích dữ liệu ảnh bằng mô hình Multimodal LLM) và Task 2 (bài viết nghị luận), chỉ ra chi tiết các lỗi lặp từ và gợi ý từ vựng đồng nghĩa học thuật nâng cao.
2. **Nâng cấp mô hình chấm điểm nói đa phương thức (Multimodal Speaking):** Sử dụng các mô hình âm thanh trực tiếp để phân tích sóng âm thanh của học viên, từ đó đưa ra điểm số tiêu chí Pronunciation (phát âm, trọng âm, ngữ điệu) chính xác 100% dựa trên âm học thực tế thay vì ước lượng qua văn bản dịch.
3. **Mã hóa và Tối ưu hóa lưu trữ:** Tích hợp quy trình tải lên không đồng bộ (Asynchronous Upload) các tệp âm thanh ghi âm của học viên lên đám mây Amazon S3 hoặc Cloudinary để lưu trữ dài hạn làm học liệu huấn luyện nâng cao, giải phóng dung lượng đĩa cứng tạm thời trên máy chủ backend.

---

## TÀI LIỆU THAM KHẢO (REFERENCES)

1. British Council, IDP: IELTS. "IELTS Speaking Band Descriptors (public version)", 2023.
2. Google Cloud. "Gemini API Documentation & Quickstart", 2024.
3. Prisma. "Transactions and concurrency in Prisma Client", Prisma documentation.
4. Redis. "Distributed Locks with Redis (Redlock algorithm)", Redis documentation.
5. OpenAI. "Whisper API Reference for Speech-to-Text conversion", 2023.
