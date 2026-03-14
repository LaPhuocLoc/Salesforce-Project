# 👤 Member 5 — Flows, Integration & QA
## Hướng dẫn Step-by-Step

> **Phạm vi:** Record-Triggered Flow, Screen Flow Wizard, Approval Process, Lightning App Config, E2E Testing, QA
>
> **Prerequisite:** Member 1 đã tạo Objects (Tuần 1). Other members' code deployed for integration testing (Tuần 2-3).

---

## 🔴 TUẦN 1 — Flows Setup (1h)

### Task 5.1: Record-Triggered Flow — `Auto_Set_Enrollment_Date` (15 phút)

**Bước 1:** Setup → Flows → New Flow → **Record-Triggered Flow**

**Bước 2:** Configure Trigger:
- **Object:** `Enrollment__c`
- **Trigger:** A record is created (After Save)
- **Condition:** `{!$Record.Enrolled_Date__c}` **Is Null** = `{!$GlobalConstant.True}`

**Bước 3:** Add Element → **Update Triggering Record**
- Field: `Enrolled_Date__c`
- Value: `{!$Flow.CurrentDateTime}`

**Bước 4:** Save Flow
- **Label:** `Auto Set Enrollment Date`
- **API Name:** `Auto_Set_Enrollment_Date`

**Bước 5:** Activate Flow ✅

```
Flow Structure:
┌──────────────────────┐
│   Start (After Save) │
│   Object: Enrollment │
└──────────┬───────────┘
           │
    ┌──────▼──────┐
    │   Decision   │
    │ Date = NULL? │
    └──────┬───────┘
           │ Yes
    ┌──────▼─────────────┐
    │   Update Record     │
    │ Enrolled_Date = NOW │
    └──────┬──────────────┘
           │
    ┌──────▼──────┐
    │     End      │
    └──────────────┘
```

**Verify:** Tạo 1 Enrollment record bằng tay → Check `Enrolled_Date__c` được auto-fill.

> 💡 **PD1 Knowledge:**
> - **Record-Triggered Flow:** Thay thế Process Builder (deprecated)
> - **After Save:** Cho phép access `$Record.Id` (đã commit)
> - Flow chỉ fire khi `Enrolled_Date__c = NULL` → prevent loop

---

### Task 5.2: Screen Flow — `Exam_Registration_Wizard` Step 1 (15 phút)

**Bước 1:** Setup → Flows → New Flow → **Screen Flow**

**Bước 2:** Tạo Resources:
- **Variable:** `varCertificationId` (Text, Input/Output)
- **Variable:** `varCertificationName` (Text)
- **Variable:** `varPassingScore` (Number)

**Bước 3:** Element 1 — **Get Records**
- Object: `Certification__c`
- Conditions: None (lấy tất cả)
- Store: All records → `colCertifications`
- Fields: `Id, Name, Passing_Score__c`

**Bước 4:** Element 2 — **Screen: "Choose Certification"**
- Add Component: **Radio Buttons** (hoặc **Picklist**)
  - Label: `Select a Certification`
  - Data Source: Record Collection → `colCertifications`
  - Display Field: `Name`
  - Value Field: `Id`
  - Store value in: `varCertificationId`

---

### Task 5.3: Screen Flow — Steps 2 & 3 (20 phút)

**Bước 5:** Element 3 — **Get Records** (Get selected certification details)
- Object: `Certification__c`
- Condition: `Id` Equals `{!varCertificationId}`
- Store: First record → `recSelectedCert`

**Bước 6:** Element 4 — **Assign** (set variables for display)
- `varCertificationName` = `{!recSelectedCert.Name}`
- `varPassingScore` = `{!recSelectedCert.Passing_Score__c}`

**Bước 7:** Element 5 — **Screen: "Confirm Registration"**
- Display Text:
```
You are registering for: {!varCertificationName}
Passing Score: {!varPassingScore}%

⚠️ After registration, your enrollment will be sent for Admin approval.
```
- Add: Previous button enabled

**Bước 8:** Element 6 — **Create Records**
- Object: `Enrollment__c`
- Field Values:
  - `Certification__c` = `{!varCertificationId}`
  - `Student__c` = `{!$User.Id}`
  - `Status__c` = `Pending`

**Bước 9:** Element 7 — **Screen: "Success"**
- Display Text:
```
✅ Registration submitted successfully!

Your enrollment is pending admin approval.
You will be able to take mock exams once approved.
```

**Bước 10:** Save & Activate
- **Label:** `Exam Registration Wizard`
- **API Name:** `Exam_Registration_Wizard`
- Activate ✅

```
Flow Structure:
┌────────┐   ┌─────────────┐   ┌──────────┐   ┌─────────┐   ┌─────────┐
│Get Certs│──▶│Screen: Choose│──▶│Get Detail │──▶│Assign   │──▶│Screen:  │
│         │   │Certification │   │Selected   │   │Variables│   │Confirm  │
└────────┘   └──────────────┘   └──────────┘   └─────────┘   └────┬────┘
                                                                   │
                                                            ┌──────▼──────┐
                                                            │Create Record│
                                                            │Enrollment   │
                                                            └──────┬──────┘
                                                                   │
                                                            ┌──────▼──────┐
                                                            │Screen:      │
                                                            │Success      │
                                                            └─────────────┘
```

---

### Task 5.4: Test Flows Manually (10 phút)

**Test 1: Record-Triggered Flow**
1. Tạo Enrollment record (bằng tay hoặc qua Screen Flow)
2. KHÔNG fill `Enrolled_Date__c`
3. Save → Verify `Enrolled_Date__c` auto-filled = NOW

**Test 2: Screen Flow**
1. Từ App Launcher → Search "Exam Registration Wizard" → Run
2. Chọn Certification → Confirm → Verify Enrollment created với Status = Pending
3. Verify `Enrolled_Date__c` cũng được fill (vì Record-Triggered Flow cũng fire)

---

## 🟡 TUẦN 2 — Approval Process + Integration Testing (1h)

### Task 5.5: Approval Process — `Enrollment_Approval` (20 phút)

**Bước 1:** Setup → Approval Processes → `Enrollment__c` → Create New Approval Process → **Use Standard Setup Wizard**

**Bước 2:** Basic Configuration:
- **Process Name:** `Enrollment Approval`
- **API Name:** `Enrollment_Approval`
- **Entry Criteria:** `Status__c` equals `Pending`

**Bước 3:** Specify Approver:
- **Type:** Let the submitter choose the approver manually
- **OR** Automatically assign to: `Admin User` / `Role: System Administrator`

**Bước 4:** Final Approval Actions:
- Add Action → **Field Update:**
  - Object: `Enrollment__c`
  - Field: `Status__c`
  - New Value: `Approved`

**Bước 5:** Final Rejection Actions:
- Add Action → **Field Update:**
  - Object: `Enrollment__c`
  - Field: `Status__c`
  - New Value: `Rejected`

**Bước 6:** Submission Actions (Optional):
- Record Lock: ✅ Lock the record

**Bước 7:** Activate Approval Process ✅

```
Approval Process:
┌──────────────────┐
│ Entry Criteria:   │
│ Status = Pending  │
└────────┬─────────┘
         │
┌────────▼─────────┐
│ Assigned Approver │
│ (Admin/Role)      │
└────────┬─────────┘
         │
    ┌────▼────┐
    │ Decision │
    └─┬─────┬─┘
      │     │
   Approve  Reject
      │     │
┌─────▼──┐ ┌▼────────┐
│Approved│ │Rejected │
│Field   │ │Field    │
│Update  │ │Update   │
└────────┘ └─────────┘
```

> 💡 **PD1 Knowledge:**
> - Approval Process: declarative workflow cho phê duyệt
> - Field Updates: auto-update fields khi approve/reject
> - Record Lock: prevent edits during approval
> - Có thể submit programmatically: `Approval.ProcessSubmitRequest`

---

### Task 5.6: Test Enrollment → Approval Flow (15 phút)

**Full flow test:**
1. Login as Student (hoặc dùng `Login As` feature)
2. Run `Exam Registration Wizard` Screen Flow
3. Chọn Certification → Confirm → Enrollment created (Status = Pending)
4. Verify `Enrolled_Date__c` auto-filled (Record-Triggered Flow)
5. Go to Enrollment record → Submit for Approval
6. Login as Admin → Approve
7. Verify `Status__c` changed to `Approved`

**Ghi chú bugs tìm thấy → ghi vào Bug List.**

---

### Task 5.7: Test Mock Exam End-to-End (15 phút)

**Full flow test:**
1. Login as Student (có Enrollment Approved)
2. Navigate to Mock Exam page (LWC)
3. Select Certification → Start Exam
4. Answer all questions → Submit
5. Verify:
   - [ ] Result modal shows Score + Pass/Fail
   - [ ] `Attempt__c` record created
   - [ ] `Attempt_Answer__c` records created
   - [ ] Trigger fired (check Enrollment update)
   - [ ] Dashboard updated with new score

---

### Task 5.8: Bug List Compilation (10 phút)

Tạo bug list chi tiết:

| # | Bug | Severity | Owner | Status |
|:---|:---|:---|:---|:---|
| 1 | _Mô tả bug_ | High/Med/Low | Member X | Open |
| 2 | | | | |

Share bug list với team qua Chatter hoặc Slack.

---

## 🟢 TUẦN 3 — Lightning App, E2E & Final Checklist (1.2h)

### Task 5.9: Lightning App Configuration (15 phút)

**Bước 1:** Setup → App Manager → New Lightning App

**Bước 2:** App Details:
- **App Name:** `Certification Bootcamp Manager`
- **Developer Name:** `Certification_Bootcamp_Manager`
- **Description:** `Quản lý ôn thi chứng chỉ Salesforce`
- **Image:** Upload custom logo (chứng chỉ / mũ tốt nghiệp)

**Bước 3:** App Branding:
- **Primary Color:** `#1B5E20` (Xanh lá đậm)
- **Logo:** Upload logo

**Bước 4:** Navigation Items (Tabs):
1. Home
2. Mock Exam (Lightning Page chứa `mockExamFlow`)
3. Admin Questions (Lightning Page chứa `adminQuestionManage`)
4. Reports

**Bước 5:** Assign Profiles:
- `System Administrator` ✅
- `Bootcamp_Student` ✅

---

### Task 5.10: Tab & Lightning Page Setup (15 phút)

**Tạo Lightning Pages cho mỗi tab:**

**Page 1: Mock Exam Page**
1. Setup → Lightning App Builder → New → App Page
2. Label: `Mock Exam`
3. Template: One column
4. Drag `mockExamFlow` component vào page
5. Save → Activate → Add to `Certification Bootcamp Manager` app

**Page 2: Admin Questions Page**
1. Setup → Lightning App Builder → New → App Page
2. Label: `Admin Questions`
3. Template: One column
4. Drag `adminQuestionManage` component vào page
5. Save → Activate → Add to app

**Lưu ý:** Home Page đã được Member 1 setup ở Task 1.18.

---

### Task 5.11: Full E2E Test — Student Flow (20 phút)

**Login as Student user → Chạy toàn bộ flow:**

| # | Step | Expected Result | ✅ |
|:---|:---|:---|:---|
| 1 | Open Certification Bootcamp Manager app | App loads, Home tab visible | ⬜ |
| 2 | Home tab shows Dashboard | `dashboardView` LWC renders: "Take First Mock Exam" | ⬜ |
| 3 | Run Exam Registration Wizard | Screen Flow: Select cert → Confirm → Success | ⬜ |
| 4 | Enrollment created with Status=Pending | Check record detail page | ⬜ |
| 5 | `Enrolled_Date__c` auto-filled | Record-Triggered Flow verified | ⬜ |
| 6 | Submit Enrollment for Approval | Approval request sent | ⬜ |
| 7 | (Admin approves) | Status = Approved | ⬜ |
| 8 | Navigate to Mock Exam tab | `mockExamFlow` loads, dropdown shows certs | ⬜ |
| 9 | Select cert → Start Exam | Questions displayed | ⬜ |
| 10 | Answer all → Submit | Result modal: Score + Pass/Fail | ⬜ |
| 11 | Return to Home | Dashboard updated: new score, readiness, leaderboard | ⬜ |
| 12 | Student cannot see Correct_Answer | FLS verified | ⬜ |
| 13 | Student only sees own Attempts | OWD Private verified | ⬜ |

---

### Task 5.12: Full E2E Test — Admin Flow (15 phút)

**Login as Admin → Chạy admin flow:**

| # | Step | Expected Result | ✅ |
|:---|:---|:---|:---|
| 1 | Open app → Admin Questions tab | `adminQuestionManage` loads with all questions | ⬜ |
| 2 | Click New Question | `newQuestionModal` opens | ⬜ |
| 3 | Fill form → Save | Question created, datatable refreshed | ⬜ |
| 4 | Edit question | Modal pre-filled, update saves correctly | ⬜ |
| 5 | Toggle Active/Inactive | `Is_Active__c` toggled, datatable refreshed | ⬜ |
| 6 | Create question with missing option | Validation Rule fires error message | ⬜ |
| 7 | Approve enrollment request | Status changed to Approved | ⬜ |
| 8 | View Reports tab | 3 reports accessible | ⬜ |
| 9 | View Dashboard | 3 components render (Donut, Gauge, Metric) | ⬜ |
| 10 | Admin sees ALL Attempts | Sharing Rule verified | ⬜ |

---

### Task 5.13: BDD Final Checklist (10 phút)

**Chạy 26-point checklist từ BDD Mục 20:**

| # | Hạng mục | Trạng thái |
|:---|:---|:---|
| 1 | Custom Lightning App (logo, brand, tabs) | ⬜ |
| 2 | Custom Objects (≥ 4) = 5 objects | ⬜ |
| 3 | Lookup Relationship | ⬜ |
| 4 | Master-Detail Relationship | ⬜ |
| 5 | Many-to-Many (Junction Object) | ⬜ |
| 6 | Validation Rule (≥ 1) | ⬜ |
| 7 | Roll-up Summary (≥ 1) | ⬜ |
| 8 | 2 Profiles | ⬜ |
| 9 | FLS khác nhau | ⬜ |
| 10 | OWD configured | ⬜ |
| 11 | Permission Set | ⬜ |
| 12 | Record-Triggered Flow | ⬜ |
| 13 | Screen Flow Wizard | ⬜ |
| 14 | Approval Process | ⬜ |
| 15 | Apex Trigger + Handler | ⬜ |
| 16 | Async Apex (Batch/Scheduled) | ⬜ |
| 17 | LWC Component (≥ 1) = 4 | ⬜ |
| 18 | SLDS compliance | ⬜ |
| 19 | Reports (≥ 3) | ⬜ |
| 20 | Dashboard (Chart + Gauge + Metric) | ⬜ |
| 21 | Home Page (Dashboard + LWC) | ⬜ |
| 22 | Test: Positive / Negative / Bulk | ⬜ |
| 23 | Code Coverage ≥ 75% | ⬜ |
| 24 | ERD Diagram | ⬜ |
| 25 | Business Process Flows | ⬜ |
| 26 | Trigger/Flow Specification | ⬜ |

**Nếu có item chưa đạt → ghi bug → assign cho member tương ứng.**

---

## ✅ Deliverables tổng kết Member 5

| # | Deliverable | Status |
|:---|:---|:---|
| 1 | Record-Triggered Flow: `Auto_Set_Enrollment_Date` (Active) | ⬜ |
| 2 | Screen Flow: `Exam_Registration_Wizard` (Active) | ⬜ |
| 3 | Approval Process: `Enrollment_Approval` (Active) | ⬜ |
| 4 | Lightning App: Certification Bootcamp Manager | ⬜ |
| 5 | Lightning Pages: Mock Exam + Admin Questions | ⬜ |
| 6 | E2E Test: Student flow — 13 steps verified | ⬜ |
| 7 | E2E Test: Admin flow — 10 steps verified | ⬜ |
| 8 | BDD Checklist: 26/26 passed | ⬜ |
| 9 | Bug report compiled & shared | ⬜ |
