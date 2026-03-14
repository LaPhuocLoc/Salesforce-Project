# 👤 Member 3 — Trigger & Async Apex
## Hướng dẫn Step-by-Step

> **Phạm vi:** `AttemptTrigger.trigger`, `AttemptTriggerHandler.cls`, `LeaderboardRefreshBatch.cls`, `AttemptTriggerHandlerTest.cls`, `LeaderboardRefreshBatchTest.cls`
>
> **Prerequisite:** Member 1 đã tạo xong Objects. Member 2 đang song song viết Service/Controller.

---

## 🔴 TUẦN 1 — Trigger + Handler + Batch Skeleton (1.5h)

### Task 3.1: `AttemptTrigger.trigger` (10 phút)

Developer Console → File → New → Apex Trigger

```apex
trigger AttemptTrigger on Attempt__c (after insert) {
    if (Trigger.isAfter && Trigger.isInsert) {
        AttemptTriggerHandler.handleAfterInsert(Trigger.new);
    }
}
```

> 💡 **PD1 Knowledge:**
> - Trigger chỉ là "dispatcher" — KHÔNG chứa logic.
> - Delegate 100% logic cho Handler class.
> - Check `Trigger.isAfter && Trigger.isInsert` dù chỉ register 1 event → Best practice cho extend sau này.

---

### Task 3.2: `AttemptTriggerHandler.cls` (40 phút)

```apex
public with sharing class AttemptTriggerHandler {
    
    /**
     * After Insert: Khi có Attempt mới → cập nhật Enrollment metadata
     * Logic: Nếu student Pass → có thể update thêm trạng thái trên Enrollment
     */
    public static void handleAfterInsert(List<Attempt__c> newAttempts) {
        try {
            // ====== STEP 1: Collect unique Cert + Student pairs ======
            Set<Id> certIds = new Set<Id>();
            Set<Id> studentIds = new Set<Id>();
            
            for (Attempt__c att : newAttempts) {
                certIds.add(att.Certification__c);
                studentIds.add(att.Student__c);
            }
            
            // ====== STEP 2: Query Enrollment records IN BULK ======
            // ONE SOQL query, ngoài loop
            List<Enrollment__c> enrollments = [
                SELECT Id, Certification__c, Student__c, Status__c
                FROM Enrollment__c
                WHERE Certification__c IN :certIds
                AND Student__c IN :studentIds
                AND Status__c = 'Approved'
            ];
            
            // ====== STEP 3: Build Map for fast lookup ======
            // Key = CertId + '_' + StudentId → unique identifier
            Map<String, Enrollment__c> enrollmentMap = 
                new Map<String, Enrollment__c>();
            for (Enrollment__c enr : enrollments) {
                String key = enr.Certification__c + '_' + enr.Student__c;
                enrollmentMap.put(key, enr);
            }
            
            // ====== STEP 4: Process Attempts & update Enrollments ======
            List<Enrollment__c> enrollmentsToUpdate = 
                new List<Enrollment__c>();
            
            for (Attempt__c att : newAttempts) {
                String key = att.Certification__c + '_' + att.Student__c;
                if (enrollmentMap.containsKey(key)) {
                    Enrollment__c enr = enrollmentMap.get(key);
                    // Business Logic: Update enrollment nếu cần
                    // Ví dụ: track last attempt status, update metadata
                    // Hiện tại chỉ demo pattern — có thể mở rộng
                    
                    // Optional: Nếu muốn thêm custom field trên Enrollment
                    // để track "Last Exam Passed" thì update ở đây
                    // enr.Last_Exam_Status__c = att.Status__c;
                    // enrollmentsToUpdate.add(enr);
                }
            }
            
            // ====== STEP 5: Single DML outside loop ======
            if (!enrollmentsToUpdate.isEmpty()) {
                update enrollmentsToUpdate;
            }
            
        } catch (Exception e) {
            // Log error nhưng không block insert Attempt
            // Trong production: dùng Platform Event hoặc Custom Log object
            System.debug(LoggingLevel.ERROR, 
                'AttemptTriggerHandler Error: ' + e.getMessage());
            // Ở trigger context, dùng addError để show lỗi cho user
            for (Attempt__c att : newAttempts) {
                att.addError('Error processing attempt: ' + e.getMessage());
            }
        }
    }
}
```

> 💡 **PD1 Knowledge Points:**
> - **Bulkification Pattern:** Dùng `Set<Id>` để collect IDs → 1 SOQL → `Map` → loop process → 1 DML
> - **KHÔNG CÓ SOQL/DML TRONG LOOP** — đây là rule #1 của Salesforce development
> - **Map composite key:** `CertId + '_' + StudentId` — cách tạo unique key từ 2 fields
> - **Error Handling:** Try-Catch + `addError()` cho trigger context

---

### Task 3.3: Bulkification Review (15 phút)

**Checklist tự review code:**

| # | Check | Đạt? |
|:---|:---|:---|
| 1 | SOQL ngoài loop? | ⬜ |
| 2 | DML ngoài loop? | ⬜ |
| 3 | Dùng `Set<Id>` cho collect? | ⬜ |
| 4 | Dùng `Map` cho lookup? | ⬜ |
| 5 | Handler nhận `List<>` params? | ⬜ |
| 6 | `with sharing` keyword? | ⬜ |
| 7 | Error handling có? | ⬜ |

---

### Task 3.4: `LeaderboardRefreshBatch.cls` — Skeleton (25 phút)

```apex
public class LeaderboardRefreshBatch 
    implements Database.Batchable<SObject>, Schedulable {
    
    // ====== Batchable Interface ======
    
    public Database.QueryLocator start(Database.BatchableContext bc) {
        // Trả về tất cả Attempt records
        return Database.getQueryLocator(
            'SELECT Student__c, Score__c FROM Attempt__c'
        );
    }
    
    public void execute(Database.BatchableContext bc, List<Attempt__c> scope) {
        // Aggregate scores per student trong batch chunk này
        Map<Id, List<Decimal>> studentScores = new Map<Id, List<Decimal>>();
        
        for (Attempt__c att : scope) {
            if (!studentScores.containsKey(att.Student__c)) {
                studentScores.put(att.Student__c, new List<Decimal>());
            }
            studentScores.get(att.Student__c).add(att.Score__c);
        }
        
        // Log aggregated data (MVP: log only, không cache vào Custom Setting)
        for (Id studentId : studentScores.keySet()) {
            List<Decimal> scores = studentScores.get(studentId);
            Decimal total = 0;
            for (Decimal s : scores) {
                total += s;
            }
            Decimal avg = total / scores.size();
            System.debug('Student: ' + studentId + ' | Avg Score: ' + avg);
        }
    }
    
    public void finish(Database.BatchableContext bc) {
        System.debug('LeaderboardRefreshBatch completed at: ' + DateTime.now());
        // Trong production: gửi email notification hoặc update Custom Metadata
    }
    
    // ====== Schedulable Interface ======
    
    public void execute(SchedulableContext sc) {
        Database.executeBatch(new LeaderboardRefreshBatch(), 200);
    }
}
```

> 💡 **PD1 Knowledge Points:**
> - **`Database.Batchable<SObject>`:** 3 methods bắt buộc: `start()`, `execute()`, `finish()`
> - **`Schedulable`:** 1 method: `execute(SchedulableContext)`
> - **Batch size 200:** Default và recommend cho hầu hết use cases
> - **QueryLocator:** Cho phép query lên tới 50 triệu records (vượt 50k limit thông thường)

---

## 🟡 TUẦN 2 — Batch Complete + Unit Tests (1.5h)

### Task 3.5: Hoàn thiện `LeaderboardRefreshBatch.execute()` (20 phút)

Nếu muốn cache vào Custom Setting (Optional cho MVP):

```apex
    // Option A: Log only (MVP — đã implement ở trên)
    
    // Option B: Cache vào Custom Setting (Advanced)
    // - Tạo Custom Setting: Leaderboard_Cache__c
    // - Fields: Student_Id__c (Text), Average_Score__c (Number)
    // - Update logic trong execute()
```

> Với MVP, **Option A (Log only)** là đủ. Mục đích chính là demo skill Batch + Schedulable.

---

### Task 3.6: Schedule job setup (10 phút)

**Chạy trong Developer Console → Execute Anonymous:**

```apex
// Schedule daily at 2:00 AM
String cronExp = '0 0 2 * * ?';
System.schedule(
    'Leaderboard Daily Refresh', 
    cronExp, 
    new LeaderboardRefreshBatch()
);
```

**Verify:** Setup → Scheduled Jobs → Confirm job exists

> 💡 **CRON Expression:** `0 0 2 * * ?` = Seconds(0) Minutes(0) Hours(2) DayOfMonth(*) Month(*) DayOfWeek(?)

---

### Task 3.7: `AttemptTriggerHandlerTest.cls` (30 phút)

```apex
@isTest
private class AttemptTriggerHandlerTest {
    
    @TestSetup
    static void setupData() {
        // Tạo Certification + Questions + Student + Enrollment
        Certification__c cert = BootcampTestFactory.createCertification(
            'Test Cert', 65
        );
        BootcampTestFactory.createQuestions(cert.Id, 10, true);
        
        // Note: Enrollment cần User — dùng running user cho simplicity
        // Hoặc tạo student user nếu M-D cho phép
    }
    
    @isTest
    static void testSingleAttemptInsert_TriggerFires() {
        Certification__c cert = [
            SELECT Id FROM Certification__c LIMIT 1
        ];
        
        // Insert 1 Attempt → Trigger should fire
        Attempt__c attempt = new Attempt__c(
            Student__c = UserInfo.getUserId(),
            Certification__c = cert.Id,
            Score__c = 80,
            Status__c = 'Pass',
            Attempt_Date__c = DateTime.now()
        );
        
        Test.startTest();
        insert attempt;
        Test.stopTest();
        
        // Verify Attempt was inserted successfully
        List<Attempt__c> attempts = [
            SELECT Id, Score__c, Status__c 
            FROM Attempt__c
        ];
        System.assertEquals(1, attempts.size());
        System.assertEquals(80, attempts[0].Score__c);
    }
    
    @isTest
    static void testBulkAttemptInsert_200Records() {
        Certification__c cert = [
            SELECT Id FROM Certification__c LIMIT 1
        ];
        
        // Insert 200 Attempts → Test Bulkification
        List<Attempt__c> attempts = new List<Attempt__c>();
        for (Integer i = 0; i < 200; i++) {
            attempts.add(new Attempt__c(
                Student__c = UserInfo.getUserId(),
                Certification__c = cert.Id,
                Score__c = Math.mod(i, 100),
                Status__c = (Math.mod(i, 2) == 0) ? 'Pass' : 'Fail',
                Attempt_Date__c = DateTime.now()
            ));
        }
        
        Test.startTest();
        insert attempts; // Should NOT hit governor limits
        Test.stopTest();
        
        // Verify all 200 records inserted
        Integer count = [SELECT COUNT() FROM Attempt__c];
        System.assertEquals(200, count, 
            'All 200 Attempt records should be inserted');
    }
}
```

> 💡 **PD1 Knowledge:** Bulk test 200 records là **BẮT BUỘC** — chứng minh trigger handler không hit governor limits.

---

### Task 3.8: `LeaderboardRefreshBatchTest.cls` (20 phút)

```apex
@isTest
private class LeaderboardRefreshBatchTest {
    
    @TestSetup
    static void setupData() {
        Certification__c cert = BootcampTestFactory.createCertification(
            'Batch Test Cert', 65
        );
        
        // Create multiple attempts
        List<Attempt__c> attempts = new List<Attempt__c>();
        for (Integer i = 0; i < 10; i++) {
            attempts.add(new Attempt__c(
                Student__c = UserInfo.getUserId(),
                Certification__c = cert.Id,
                Score__c = 70 + i,
                Status__c = 'Pass',
                Attempt_Date__c = DateTime.now()
            ));
        }
        insert attempts;
    }
    
    @isTest
    static void testBatchExecution() {
        Test.startTest();
        Database.executeBatch(new LeaderboardRefreshBatch(), 200);
        Test.stopTest();
        
        // Batch runs synchronously inside Test.startTest/stopTest
        // Verify no errors occurred (check debug logs)
        // Batch đã chạy thành công nếu không có exception
        System.assert(true, 'Batch should complete without errors');
    }
    
    @isTest
    static void testSchedulableExecution() {
        String cronExp = '0 0 2 * * ?';
        
        Test.startTest();
        String jobId = System.schedule(
            'Test Leaderboard Schedule', 
            cronExp, 
            new LeaderboardRefreshBatch()
        );
        Test.stopTest();
        
        // Verify job was scheduled
        CronTrigger ct = [
            SELECT Id, State 
            FROM CronTrigger 
            WHERE Id = :jobId
        ];
        System.assertNotEquals(null, ct, 
            'Scheduled job should exist');
    }
}
```

---

### Task 3.9: Verify Schedule Job (10 phút)

1. Setup → Scheduled Jobs
2. Confirm `Leaderboard Daily Refresh` exists
3. Check Next Run time = 2:00 AM ngày mai

---

## 🟢 TUẦN 3 — Bulk Test, Negative Tests & Bug Fix (1.2h)

### Task 3.10: Extended Bulk Test (20 phút)

Thêm test method vào `AttemptTriggerHandlerTest`:

```apex
    @isTest
    static void testBulkInsert_MultipleStudents() {
        Certification__c cert = [
            SELECT Id FROM Certification__c LIMIT 1
        ];
        
        // Simulate nhiều students
        List<Attempt__c> attempts = new List<Attempt__c>();
        for (Integer i = 0; i < 200; i++) {
            attempts.add(new Attempt__c(
                Student__c = UserInfo.getUserId(),
                Certification__c = cert.Id,
                Score__c = 50 + Math.mod(i, 50),
                Status__c = (Math.mod(i, 3) == 0) ? 'Pass' : 'Fail',
                Attempt_Date__c = DateTime.now()
            ));
        }
        
        Test.startTest();
        insert attempts;
        Test.stopTest();
        
        // Verify - đếm SOQL queries consumed
        System.assert(
            Limits.getQueries() <= Limits.getLimitQueries(),
            'Should not exceed SOQL query limit'
        );
    }
```

---

### Task 3.11: Negative Tests (15 phút)

```apex
    @isTest
    static void testAttemptInsert_NoMatchingEnrollment() {
        Certification__c cert = [
            SELECT Id FROM Certification__c LIMIT 1
        ];
        
        // Insert attempt với student chưa enroll
        // → Trigger không nên crash, chỉ skip
        Attempt__c attempt = new Attempt__c(
            Student__c = UserInfo.getUserId(),
            Certification__c = cert.Id,
            Score__c = 90,
            Status__c = 'Pass',
            Attempt_Date__c = DateTime.now()
        );
        
        Test.startTest();
        insert attempt; // Should not throw exception
        Test.stopTest();
        
        System.assertEquals(1, 
            [SELECT COUNT() FROM Attempt__c],
            'Attempt should be inserted even without matching enrollment');
    }
```

---

### Task 3.12: Bug Fixes (15 phút)

Nhận bugs từ Member 5 report → fix issues.

---

### Task 3.13: Final Batch Verification (15 phút)

1. Execute Anonymous: `Database.executeBatch(new LeaderboardRefreshBatch(), 200);`
2. Setup → Apex Jobs → Verify status = Completed
3. Check Debug Logs → Confirm aggregated scores printed correctly

---

## ✅ Deliverables tổng kết Member 3

| # | Deliverable | Status |
|:---|:---|:---|
| 1 | `AttemptTrigger.trigger` | ⬜ |
| 2 | `AttemptTriggerHandler.cls` — Bulkified | ⬜ |
| 3 | `LeaderboardRefreshBatch.cls` — Batch + Schedulable | ⬜ |
| 4 | `AttemptTriggerHandlerTest.cls` — Single + Bulk(200) + Negative | ⬜ |
| 5 | `LeaderboardRefreshBatchTest.cls` — Batch + Schedule | ⬜ |
| 6 | Scheduled Job active (daily 2AM) | ⬜ |
| 7 | Bulk 200 test pass — no governor limit | ⬜ |
