# [![Review Assignment Due Date](https://classroom.github.com/assets/deadline-readme-button-22041afd0340ce965d47ae6ef1cefeee28c7c493a6346c4f15d667ab976d596c.svg)](https://classroom.github.com/a/iZHHambl)
# IELTS Learning App

Dự án học tiếng Anh IELTS tích hợp AI.

## Liên kết dự án

- Jira: [Jira board của nhóm](https://knightdragon184.atlassian.net/jira/software/projects/DEV/boards/2)
- SRS: [Tài liệu SRS của nhóm](https://docs.google.com/document/d/1eY0ko-emSLsJUrF0UbSFMaNeRqPMYBg_/edit)

- Overleaf: [Link Overleaf của nhóm](https://www.overleaf.com/read/zfnjhyvpcwwp#999b02)

## Mục tiêu nghiên cứu & phạm vi đề tài

Dự án tập trung vào nghiên cứu và ứng dụng Trí tuệ Nhân tạo (AI) cùng các kỹ thuật hệ thống hiện đại để xây dựng một nền tảng học IELTS thông minh, có khả năng hỗ trợ đánh giá, phản hồi và cá nhân hóa lộ trình học tập cho người dùng. Hàm lượng nghiên cứu của đề tài nằm ở ba hướng chính:

1. **Nghiên cứu ứng dụng AI trong đánh giá ngôn ngữ (AI-Assisted Grading)**
   - **Writing**: Khảo sát và tối ưu hóa Prompt Engineering cho LLM (Large Language Models - ví dụ OpenAI GPT) để tự động chấm điểm, chữa lỗi và nhận xét bài viết dựa trên 4 tiêu chí chuẩn của IELTS (Task Achievement, Coherence & Cohesion, Lexical Resource, Grammatical Range & Accuracy).
   - **Speaking**: Tích hợp mô hình nhận dạng giọng nói (Speech-to-Text như Whisper API) kết hợp LLM để đánh giá phát âm, độ trôi chảy và cung cấp feedback cá nhân hóa theo thời gian thực.

2. **Nghiên cứu Tối ưu hóa Kiến trúc Hệ thống (System Architecture)**
   - Xây dựng hệ thống Backend xử lý luồng dữ liệu đa phương tiện (Media Streaming/Upload) hiệu quả cho phần thi Listening & Speaking.
   - Xử lý bài toán đồng bộ dữ liệu và xung đột lịch đặt phòng (Double Booking) ở mức Database level bằng Transaction/Locking trong MongoDB.

3. **Cá nhân hóa trải nghiệm học tập**
   - Áp dụng các thuật toán theo dõi tiến độ và đánh giá điểm mạnh, điểm yếu qua từng bài thi để tối ưu hóa lộ trình tự học của học viên.

## Cấu trúc thư mục
- `backend/`: NodeJS Express API + Prisma + MongoDB.
- `frontend/`: React Native (Expo) app.
- `docker-compose.yml`: Orchestration cho môi trường dev.

## Hướng dẫn chạy nhanh

### 1. Backend & Database
```bash
# Chạy database (MongoDB Replica Set)
docker-compose up -d mongodb

# Cài đặt backend
cd backend
npm install
cp .env.example .env # Cấu hình DATABASE_URL

# Đồng bộ schema và tạo dữ liệu mẫu (Seed)
npm run migrate

# Chạy server
npm run dev
```

### 2. Frontend
```bash
cd frontend
npm install
npm start
```

## Tài khoản dùng thử (Test Accounts)
Sau khi chạy lệnh `npm run migrate`, bạn có thể sử dụng các tài khoản sau để test:

| Role | Email | Password |
| :--- | :--- | :--- |
| **Admin** | `admin@sdn.com` | `password123` |
| **Mentor** | `mentor@sdn.com` | `password123` |
| **Student** | `student@sdn.com` | `password123` |

> **Swagger UI**: Bạn có thể truy cập tài liệu API tại `http://localhost:5000/api-docs`

> **Mongo Express (Database GUI)**: Truy cập tại `http://localhost:8082`
> - **User**: `admin`
> - **Password**: `password`

## Tính năng (Epics)
- Epic 1: Authentication & User Management
- Epic 2: Core Learning - Reading & Listening
- Epic 3: AI Integration - Writing & Speaking
- Epic 4: Mentor Booking System
