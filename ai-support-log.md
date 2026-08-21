# AI Support Log (Phần cá nhân)

> Phần này mỗi thành viên tự điền để ghi lại quá trình hỗ trợ của trí tuệ nhân tạo đối với cá nhân mình trong suốt bài Lab.

- **AI đã giúp tôi ở đâu?**
  AI đã hỗ trợ paraphrase các kịch bản test từ lưới thiết kế thành các câu hỏi tự nhiên mang văn phong học viên. Ngoài ra, AI giúp soạn nháp văn bản báo cáo (Report) dựa trên các gạch đầu dòng nhóm đã chốt, hỗ trợ viết script tự động xử lý file dữ liệu, và tóm tắt pattern từ các case bị lệch của Judge để phân tích nhanh hơn.

- **AI sai, hời hợt hoặc làm mất độ bao phủ ở đâu?**
  Khi paraphrase, AI thỉnh thoảng tự ý bổ sung thêm ngữ cảnh làm cho câu hỏi trở nên quá chi tiết, làm mất đi tính mơ hồ mà nhóm chủ đích kiểm thử. Ngoài ra, khi gọi API Gemini, mô hình thỉnh thoảng báo lỗi 429 hoặc treo do vượt hạn ngạch.

- **Tôi đã tự sửa hoặc quyết định lại điều gì?**
Tôi cũng hoàn toàn tự tay gán nhãn baseline ở Phase 2 cùng nhóm, không sử dụng AI để can thiệp vào golden, và tự quyết định ngưỡng threshold trước khi cho chạy test thật.
