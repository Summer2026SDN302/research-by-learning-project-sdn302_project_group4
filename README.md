# IELTS Learning App

Dự án học tiếng Anh IELTS tích hợp AI.

## Mục tiêu và Hàm lượng nghiên cứu

Dự án này tập trung vào việc nghiên cứu và ứng dụng công nghệ hiện đại, đặc biệt là Trí tuệ Nhân tạo (AI), nhằm giải quyết các bài toán trong giáo dục và luyện thi IELTS:

1. **Nghiên cứu ứng dụng AI trong đánh giá ngôn ngữ (AI-Assisted Grading)**
   - **Writing**: Khảo sát và tối ưu hóa Prompt Engineering cho LLM (Large Language Models - ví dụ OpenAI GPT) để tự động chấm điểm, chữa lỗi và nhận xét bài viết dựa trên 4 tiêu chí chuẩn của IELTS (Task Achievement, Coherence & Cohesion, Lexical Resource, Grammatical Range & Accuracy).
   - **Speaking**: Tích hợp mô hình nhận dạng giọng nói (Speech-to-Text như Whisper API) kết hợp LLM để đánh giá phát âm, độ trôi chảy và cung cấp feedback cá nhân hóa theo thời gian thực.

2. **Nghiên cứu Tối ưu hóa Kiến trúc Hệ thống (System Architecture)**
   - Xây dựng hệ thống Backend xử lý luồng dữ liệu đa phương tiện (Media Streaming/Upload) hiệu quả cho phần thi Listening & Speaking.
   - Xử lý bài toán đồng bộ dữ liệu và xung đột lịch đặt phòng (Double Booking) ở mức Database level bằng Transaction/Locking trong MongoDB.

3. **Cá nhân hóa trải nghiệm học tập**
   - Áp dụng các thuật toán theo dõi tiến độ và đánh giá điểm mạnh, điểm yếu qua từng bài thi để tối ưu hóa lộ trình tự học của học viên.
