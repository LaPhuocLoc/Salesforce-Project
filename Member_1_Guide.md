# 👤 Member 1 — Data Model, Security & Reports
## Hướng dẫn Step-by-Step

> **Phạm vi:** Tạo toàn bộ Custom Objects, Fields, Relationships, Security (OWD, Profiles, FLS, Permission Set, Sharing Rule), Reports, Dashboard, Home Page và Mock Data.

---

## 🔴 TUẦN 1 — Data Model & Security Foundation (1.5h)

### Task 1.1: Tạo `Certification__c` Object (10 phút)

**Bước 1:** Setup → Object Manager → Create → Custom Object
- **Label:** `Certification`
- **Plural:** `Certifications`
- **API Name:** `Certification__c`
- **Record Name:** `Name` (Text)
- **Deployment Status:** Deployed
- ✅ Allow Reports, Allow Activities

**Bước 2:** Thêm Fields
| Field Label | API Name | Type | Required | Notes |
|:---|:---|:---|:---|:---|
| Passing Score | `Passing_Score__c` | Percent(3,0) | ✅ | Ngưỡng đậu VD: 65 |

> ⚠️ Chưa thêm `Total_Enrolled__c` ở bước này vì cần `Enrollment__c` trước.

---

### Task 1.2: Tạo `Enrollment__c` Object — Junction Object (15 phút)

**Bước 1:** Setup → Object Manager → Create → Custom Object
- **Label:** `Enrollment`
- **API Name:** `Enrollment__c`
- **Record Name:** Auto Number (`ENRL-{0000}`)

**Bước 2:** Thêm Fields theo thứ tự:

| # | Field Label | API Name | Type | Relationship | Notes |
|:---|:---|:---|:---|:---|:---|
| 1 | Certification | `Certification__c` | **Master-Detail** | → `Certification__c` | First Master-Detail |
| 2 | Student | `Student__c` | **Master-Detail** | → `User` | Second Master-Detail (tạo Junction) |
| 3 | Enrolled Date | `Enrolled_Date__c` | DateTime | — | Không bắt buộc (Flow sẽ tự fill) |
| 4 | Status | `Status__c` | Picklist | — | Values: `Pending`, `Approved`, `Rejected`. Default: `Pending` |

> 💡 **PD1 Knowledge:** Khi tạo 2 Master-Detail trên cùng object → đây là **Junction Object** = Many-to-Many relationship. User ↔ Certification qua Enrollment.

---

### Task 1.3: Tạo `Question__c` Object (15 phút)

**Bước 1:** Setup → Object Manager → Create → Custom Object
- **Label:** `Question`
- **API Name:** `Question__c`
- **Record Name:** Auto Number (`Q-{0000}`)

**Bước 2:** Thêm Fields:

| # | Field Label | API Name | Type | Notes |
|:---|:---|:---|:---|:---|
| 1 | Certification | `Certification__c` | **Lookup** → `Certification__c` | ⚠️ Lookup (KHÔNG phải M-D), nới lỏng ownership |
| 2 | Question Text | `Question_Text__c` | Long Text Area(32768) | Required ✅ |
| 3 | Option A | `Option_A__c` | Text(255) | Required ✅ |
| 4 | Option B | `Option_B__c` | Text(255) | Required ✅ |
| 5 | Option C | `Option_C__c` | Text(255) | Required ✅ |
| 6 | Option D | `Option_D__c` | Text(255) | Required ✅ |
| 7 | Correct Answer | `Correct_Answer__c` | Picklist | Values: `A`, `B`, `C`, `D` |
| 8 | Is Active | `Is_Active__c` | Checkbox | Default: ✅ checked |

---

### Task 1.4: Tạo `Attempt__c` Object (10 phút)

**Bước 1:** Setup → Create Custom Object
- **Label:** `Attempt`
- **API Name:** `Attempt__c`
- **Record Name:** Auto Number (`ATT-{0000}`)

**Bước 2:** Thêm Fields:

| # | Field Label | API Name | Type | Notes |
|:---|:---|:---|:---|:---|
| 1 | Student | `Student__c` | **Lookup** → `User` | Lookup to User (KHÔNG phải M-D) |
| 2 | Certification | `Certification__c` | **Lookup** → `Certification__c` | |
| 3 | Score | `Score__c` | Percent(3,0) | |
| 4 | Status | `Status__c` | Picklist | Values: `Pass`, `Fail` |
| 5 | Attempt Date | `Attempt_Date__c` | DateTime | |

> 💡 **PD1 Knowledge:** Dùng Lookup thay vì M-D cho `Student__c` để OWD Private hoạt động — bài ai nấy xem.

---

### Task 1.5: Tạo `Attempt_Answer__c` Object (15 phút)

**Bước 1:** Setup → Create Custom Object
- **Label:** `Attempt Answer`
- **API Name:** `Attempt_Answer__c`
- **Record Name:** Auto Number (`ANS-{0000}`)

**Bước 2:** Thêm Fields:

| # | Field Label | API Name | Type | Notes |
|:---|:---|:---|:---|:---|
| 1 | Attempt | `Attempt__c` | **Master-Detail** → `Attempt__c` | ⚠️ Master-Detail: cascade delete |
| 2 | Question | `Question__c` | **Lookup** → `Question__c` | |
| 3 | Selected Option | `Selected_Option__c` | Picklist | Values: `A`, `B`, `C`, `D` |
| 4 | Is Correct | `Is_Correct__c` | **Formula** (Checkbox) | Formula: `Selected_Option__c = Question__r.Correct_Answer__c` |

> 💡 **PD1 Knowledge:**
> - **Cross-Object Formula:** `Question__r.Correct_Answer__c` — truy cập field của object khác qua relationship.
> - **Cascade Delete:** Xóa Attempt cha → tự xóa tất cả Answer con.

---

### Task 1.6: Roll-up Summary Field (5 phút)

**Quay lại `Certification__c`** → Create Field:
- **Label:** `Total Enrolled`
- **API Name:** `Total_Enrolled__c`
- **Type:** Roll-up Summary
- **Summarized Object:** `Enrollment__c`
- **Roll-up Type:** COUNT

> 💡 **PD1 Knowledge:** Roll-up Summary chỉ hoạt động trên Master-Detail relationship. Vì `Enrollment__c` có M-D tới `Certification__c` nên roll-up hoạt động được.

---

### Task 1.7: Validation Rule trên `Question__c` (5 phút)

Object Manager → `Question__c` → Validation Rules → New

- **Rule Name:** `Require_All_Options`
- **Error Condition Formula:**
```
OR(
  ISBLANK(Option_A__c),
  ISBLANK(Option_B__c),
  ISBLANK(Option_C__c),
  ISBLANK(Option_D__c)
)
```
- **Error Message:** `Phải điền đủ cả 4 đáp án (A, B, C, D).`
- **Error Location:** Top of Page

---

### Task 1.8: OWD Setup (10 phút)

Setup → Sharing Settings

| Object | OWD Setting |
|:---|:---|
| `Certification__c` | Public Read Only |
| `Question__c` | Public Read Only |
| `Attempt__c` | **Private** |
| `Attempt_Answer__c` | Controlled by Parent |
| `Enrollment__c` | Controlled by Parent |

---

### Task 1.9: Profile `Bootcamp_Student` (15 phút)

**Bước 1:** Setup → Profiles → Clone "Standard User"
- **Profile Name:** `Bootcamp_Student`

**Bước 2:** Set Object Permissions:

| Object | Read | Create | Edit | Delete |
|:---|:---|:---|:---|:---|
| `Certification__c` | ✅ | ❌ | ❌ | ❌ |
| `Question__c` | ✅ | ❌ | ❌ | ❌ |
| `Attempt__c` | ✅ | ✅ | ❌ | ❌ |
| `Attempt_Answer__c` | ✅ | ✅ | ❌ | ❌ |
| `Enrollment__c` | ✅ | ✅ | ❌ | ❌ |

---

## 🟡 TUẦN 2 — FLS, Permission Set, Reports, Dashboard (1.5h)

### Task 1.10: FLS Setup (15 phút)

**`Bootcamp_Student` Profile → Field-Level Security:**

| Object | Field | Student | Admin |
|:---|:---|:---|:---|
| `Question__c` | `Correct_Answer__c` | ❌ **Hidden** | ✅ Visible |
| `Enrollment__c` | `Status__c` | Read Only | Edit |
| `Attempt__c` | `Score__c` | Read Only | Read Only |

**Cách làm:** Setup → Profiles → `Bootcamp_Student` → Object Settings → từng Object → Field Permissions

---

### Task 1.11: Permission Set `Exam_Manager` (10 phút)

Setup → Permission Sets → New
- **Label:** `Exam_Manager`
- **API Name:** `Exam_Manager`
- **Object Settings:** Grant Edit trên `Question__c` (cho TA/Trợ giảng)

---

### Task 1.12: Sharing Rule cho `Attempt__c` (10 phút)

**Bước 1:** Setup → Public Groups → New Group: `All_Admins` → Add System Administrator users

**Bước 2:** Setup → Sharing Settings → `Attempt__c` → Sharing Rules → New
- **Type:** Criteria-Based
- **Criteria:** Owner is not null (share all)
- **Share with:** `All_Admins`
- **Access Level:** Read Only

---

### Task 1.13: Report #1 — Student Exam Results (10 phút)

Reports → New Report
- **Report Type:** `Attempt__c`
- **Format:** Tabular
- **Columns:** Student, Certification, Score, Status, Attempt Date
- **Sort:** Attempt Date Descending
- **Save:** Folder "Bootcamp Reports"

---

### Task 1.14: Report #2 — Certification Enrollment Summary (10 phút)

Reports → New Report
- **Report Type:** `Enrollment__c` with `Certification__c`
- **Format:** Summary
- **Group By:** Certification Name
- **Summary:** Record Count, Group by Status
- **Save:** Folder "Bootcamp Reports"

---

### Task 1.15: Report #3 — Question Bank Overview (10 phút)

Reports → New Report
- **Report Type:** `Question__c`
- **Format:** Summary
- **Group By:** Certification Name
- **Summary:** Record Count, Group by Is_Active
- **Save:** Folder "Bootcamp Reports"

---

### Task 1.16: Dashboard (15 phút)

Dashboards → New Dashboard: **"Bootcamp Overview"**

| # | Title | Type | Source Report | Settings |
|:---|:---|:---|:---|:---|
| 1 | Pass/Fail Distribution | **Donut Chart** | Report #1 | Group by Status |
| 2 | Average Score | **Gauge** | Report #1 | Metric: AVG Score, Range: 0-100 |
| 3 | Total Enrollments | **Metric** | Report #2 | Record Count |

---

### Task 1.17: Insert Mock Data (10 phút)

Tạo data qua UI hoặc Data Import Wizard:

**Certifications (3 records):**
| Name | Passing Score |
|:---|:---|
| Salesforce Administrator | 65% |
| Platform Developer I | 65% |
| App Builder | 68% |

**Questions:** Tạo ít nhất 5 câu/cert (15 câu tổng) — nội dung PD1 knowledge.

---

## 🟢 TUẦN 3 — Home Page, Final Data & Security Review (1h)

### Task 1.18: Lightning Home Page (15 phút)

Setup → Lightning App Builder → New → **Home Page**
- **Label:** `Bootcamp Home`
- **Layout:** 2 columns (66/33)
- Left column: Drag `dashboardView` LWC component (do Member 4 tạo)
- Right column: Drag Dashboard component (nhúng "Bootcamp Overview" dashboard)
- **Activate:** Assign as Org Default / App Default

---

### Task 1.19: Profile-based Home Page — Optional (15 phút)

Nếu đủ thời gian:
- Tạo thêm 1 Home Page cho Admin (nhúng thêm Question stats)
- Setup → Home → Assign Page → By Profile

---

### Task 1.20: Final Mock Data (20 phút)

Đảm bảo org có đủ data để demo đẹp:
- 3 Certifications
- 15+ Questions/Certification
- 5 Student Users (dùng Profile `Bootcamp_Student`)
- 10+ Attempt records (mix Pass/Fail)
- Enrollment records đã Approved

---

### Task 1.21: Final Security Review (10 phút)

**Checklist kiểm tra:**
- [ ] Login bằng Student → Không thấy `Correct_Answer__c` trên Question
- [ ] Student chỉ thấy Attempt của mình (OWD Private)
- [ ] Admin thấy tất cả Attempt (Sharing Rule)
- [ ] Student không thể Edit/Delete Question
- [ ] Permission Set `Exam_Manager` hoạt động đúng

---

## ✅ Deliverables tổng kết Member 1

| # | Deliverable | Status |
|:---|:---|:---|
| 1 | 5 Custom Objects + tất cả Fields | ⬜ |
| 2 | 1 Validation Rule (`Require_All_Options`) | ⬜ |
| 3 | 1 Roll-up Summary (`Total_Enrolled__c`) | ⬜ |
| 4 | OWD configured cho 5 Objects | ⬜ |
| 5 | Profile `Bootcamp_Student` | ⬜ |
| 6 | FLS: Hide `Correct_Answer__c` | ⬜ |
| 7 | Permission Set `Exam_Manager` | ⬜ |
| 8 | Sharing Rule cho `Attempt__c` | ⬜ |
| 9 | 3 Reports | ⬜ |
| 10 | 1 Dashboard (3 components) | ⬜ |
| 11 | Home Page (nhúng Dashboard + LWC) | ⬜ |
| 12 | Mock Data đủ demo | ⬜ |
