# Business Process Flows — Salesforce Certification Bootcamp Manager

---

## Flow 1: Quy trình Đăng ký Chứng chỉ (Student Enrollment)

```mermaid
flowchart TD
    A([Bắt đầu]) --> B[Student mở Lightning App]
    B --> C[Khởi chạy Screen Flow: Exam Registration Wizard]
    C --> D[Bước 1: Hiển thị danh sách Chứng chỉ từ Certification__c]
    D --> E{Student đã chọn Chứng chỉ?}
    E -- Chưa --> D
    E -- Đã chọn --> F[Bước 2: Hiển thị màn hình xác nhận kèm Passing Score]
    F --> G{Student xác nhận?}
    G -- Hủy --> Z([Kết thúc: Bỏ dở])
    G -- Xác nhận --> H[Bước 3 - Tạo bản ghi Enrollment__c - Status = Pending]
    H --> I[Record-Triggered Flow tự kích hoạt - Gán Enrolled_Date__c = NOW]
    I --> J[Hệ thống gửi Enrollment vào Approval Process]
    J --> K[Admin nhận thông báo yêu cầu phê duyệt]
    K --> L{Quyết định của Admin}
    L -- Phê duyệt --> M[Status = Approved - Student được phép làm Mock Exam]
    L -- Từ chối --> N[Status = Rejected - Student nhận thông báo từ chối]
    M --> O([Kết thúc: Được duyệt])
    N --> P([Kết thúc: Bị từ chối])
```

---

## Flow 2: Quy trình Thi thử (Mock Exam)

```mermaid
flowchart TD
    A([Bắt đầu]) --> B[Student vào trang Mock Exam]
    B --> C[Chọn Chứng chỉ từ dropdown]
    C --> D[Gọi Apex: BootcampController.getQuestions với điều kiện Is_Active__c = true]
    D --> E{Tìm thấy câu hỏi?}
    E -- Không --> F[Hiển thị lỗi: Không có câu hỏi nào đang hoạt động]
    F --> Z([Kết thúc])
    E -- Có --> G[Hiển thị Câu hỏi 1 kèm 4 đáp án A B C D]
    G --> H{Student đã chọn đáp án?}
    H -- Chưa --> H
    H -- Đã chọn --> I[Lưu đáp án vào mảng JS phía client: questionId + selectedOption]
    I --> J{Còn câu hỏi nào không?}
    J -- Tiếp theo --> K[Hiển thị câu hỏi kế tiếp]
    K --> H
    J -- Quay lại --> L[Điều hướng về câu hỏi trước]
    L --> H
    J -- Nộp bài --> M[Gửi mảng đáp án lên Apex qua Imperative Call]
    M --> N[MockExamService.calculateScore: so sánh từng Selected_Option với Correct_Answer__c]
    N --> O[Tính điểm = Số đáp án đúng chia Tổng số câu nhân 100]
    O --> P{Điểm >= Passing_Score__c của Chứng chỉ?}
    P -- Đạt --> Q[Status = Pass]
    P -- Không đạt --> R[Status = Fail]
    Q --> S[DML Insert: Attempt__c với Score + Status + Attempt_Date]
    R --> S
    S --> T[DML Insert: Danh sách Attempt_Answer__c kèm Formula Is_Correct__c]
    T --> U[AttemptTrigger tự kích hoạt: After Insert trên Attempt__c]
    U --> V[AttemptTriggerHandler: Cập nhật Enrollment__c liên quan nếu cần]
    V --> W[Trả Score + Status về LWC component]
    W --> X[Hiển thị Modal Kết quả: Điểm % và huy hiệu Pass/Fail]
    X --> Y([Kết thúc: Bài thi đã được lưu])
```

---

## Flow 3: Tải dữ liệu Dashboard (Dashboard Data Loading)

```mermaid
flowchart TD
    A([Bắt đầu]) --> B[Student mở Home Page]
    B --> C[LWC dashboardView được tải]
    C --> D[Wire Adapter gọi: BootcampController.getDashboardStats với User hiện tại]
    D --> E[SOQL: Truy vấn Attempt__c của User hiện tại ORDER BY Attempt_Date DESC]
    E --> F{Có bài thi nào không?}
    F -- Không --> G[Next Action = Take First Mock Exam / Readiness = Not Ready]
    F -- Có --> H[Tính điểm trung bình từ toàn bộ Score__c]
    H --> I{Điểm trung bình >= 80%?}
    I -- Đúng --> J[Readiness = Ready]
    I -- Sai --> K{Điểm trung bình >= 65%?}
    K -- Đúng --> L[Readiness = At Risk]
    K -- Sai --> M[Readiness = Not Ready]
    J --> N[Kiểm tra điểm bài thi cuối so với Passing_Score__c]
    L --> N
    M --> N
    N --> O{Điểm bài cuối >= Passing Score?}
    O -- Đúng --> P[Next Action = Ready for Certification]
    O -- Sai --> Q[Next Action = Retake Exam]
    G --> R[Trả DashboardWrapper về LWC]
    P --> R
    Q --> R
    R --> S[SOQL: Top 5 User GROUP BY Student__c AVG Score DESC LIMIT 5]
    S --> T[Trả danh sách Leaderboard về LWC]
    T --> U[Render Dashboard: Thẻ Thống kê + Progress Bar + Leaderboard + Bảng Bài thi gần nhất]
    U --> V([Kết thúc: Dashboard đã hiển thị])
```

---

## Flow 4: Admin Quản lý Câu hỏi (Admin Question Management)

```mermaid
flowchart TD
    A([Bắt đầu]) --> B[Admin vào Tab Admin Questions]
    B --> C[LWC adminQuestionManage được tải]
    C --> D[Wire: BootcampController.getAllQuestions]
    D --> E[Hiển thị lightning-datatable với toàn bộ câu hỏi]
    E --> F{Hành động của Admin?}
    F -- Tạo mới --> G[Mở newQuestionModal với lightning-record-edit-form]
    G --> H[Admin điền Certification + Nội dung câu hỏi + 4 đáp án A B C D + Đáp án đúng]
    H --> I{Validation Rule thành công?}
    I -- Thất bại --> J[Hiển thị lỗi: Phải điền đủ cả 4 đáp án]
    J --> H
    I -- Thành công --> K[DML Insert: Question__c với Is_Active__c = true]
    K --> L[Làm mới datatable]
    L --> E
    F -- Chỉnh sửa --> M[Mở modal đã điền sẵn thông tin bản ghi hiện tại]
    M --> N[Admin cập nhật các trường]
    N --> O{Validation thành công?}
    O -- Thất bại --> P[Hiển thị thông báo lỗi]
    P --> N
    O -- Thành công --> Q[DML Update: Question__c]
    Q --> L
    F -- Kích hoạt/Vô hiệu hóa --> R[Đảo trạng thái boolean Is_Active__c]
    R --> S{Is_Active__c = true?}
    S -- Đang hoạt động --> T[Câu hỏi xuất hiện trong pool đề thi]
    S -- Vô hiệu hóa --> U[Câu hỏi bị ẩn khỏi pool đề thi]
    T --> L
    U --> L
    F -- Xong --> V([Kết thúc])
```

---

## Flow 5: Record-Triggered Flow — Tự động gán Ngày đăng ký

```mermaid
flowchart TD
    A([Enrollment__c After Insert]) --> B{Enrolled_Date__c đang trống?}
    B -- Đúng --> C[Gán Enrolled_Date__c = NOW]
    C --> D[Cập nhật bản ghi Enrollment__c]
    D --> E([Kết thúc: Đã đóng dấu ngày])
    B -- Sai --> F([Kết thúc: Không cần xử lý])
```

---

## Flow 6: Approval Process — Phê duyệt Đăng ký

```mermaid
flowchart TD
    A([Student gửi Enrollment__c để phê duyệt]) --> B[Approval Process: Enrollment_Approval khởi động]
    B --> C[Hệ thống gửi yêu cầu phê duyệt tới Admin được chỉ định]
    C --> D{Admin xem xét bản ghi Enrollment}
    D -- Phê duyệt --> E[Final Action: Cập nhật Status__c = Approved]
    D -- Từ chối --> F[Rejection Action: Cập nhật Status__c = Rejected]
    E --> G[Student có thể bắt đầu làm Mock Exam cho Chứng chỉ này]
    F --> H[Student nhận thông báo bị từ chối]
    G --> I([Kết thúc: Được phê duyệt])
    H --> J([Kết thúc: Bị từ chối])
```

---

## Flow 7: Async Apex — LeaderboardRefreshBatch

```mermaid
flowchart TD
    A([Scheduled Job tự kích hoạt mỗi ngày lúc 02:00 AM]) --> B[LeaderboardRefreshBatch Schedulable.execute]
    B --> C[Database.executeBatch được gọi với kích thước Batch = 200]
    C --> D[Phương thức start: QueryLocator trả về toàn bộ bản ghi Attempt__c]
    D --> E[Phương thức execute: Xử lý từng chunk dữ liệu]
    E --> F[Tính tổng hợp AVG Score__c GROUP BY Student__c]
    F --> G{Còn chunk nào không?}
    G -- Còn --> E
    G -- Hết --> H[Phương thức finish: Ghi log hoàn thành batch]
    H --> I([Kết thúc: Bảng xếp hạng đã được làm mới])
```

---

## Flow 8: Apex Trigger — AttemptTriggerHandler

```mermaid
flowchart TD
    A([DML Insert trên Attempt__c]) --> B[AttemptTrigger tự kích hoạt: After Insert]
    B --> C[Delegate toàn bộ sang AttemptTriggerHandler.handleAfterInsert với danh sách Trigger.new]
    C --> D[Gom các cặp Certification__c + Student__c độc nhất bằng Set — không SOQL trong vòng lặp]
    D --> E[SOQL: Truy vấn hàng loạt bản ghi Enrollment__c liên quan — một câu truy vấn duy nhất]
    E --> F[Xây dựng Map: CertId+StudentId ánh xạ tới Enrollment__c]
    F --> G[Duyệt qua danh sách Attempt mới và cập nhật các trường Enrollment nếu cần]
    G --> H{Có bản ghi Enrollment nào cần cập nhật không?}
    H -- Có --> I[DML Update duy nhất: Danh sách Enrollment__c]
    H -- Không --> J([Kết thúc: Không cần cập nhật])
    I --> K([Kết thúc: Enrollments đã được cập nhật])
```
