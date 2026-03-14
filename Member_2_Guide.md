# 👤 Member 2 — Apex Backend (Controller + Service)
## Hướng dẫn Step-by-Step

> **Phạm vi:** `BootcampController.cls`, `MockExamService.cls`, `BootcampTestFactory.cls`, `MockExamServiceTest.cls`, `BootcampControllerTest.cls`
>
> **Prerequisite:** Member 1 đã tạo xong 5 Custom Objects (Task 1.1–1.5)

---

## 🔴 TUẦN 1 — Apex Service Skeleton (1.5h)

### Task 2.1: `MockExamService.cls` (40 phút)

**Bước 1:** Developer Console → File → New → Apex Class

```apex
public with sharing class MockExamService {
    
    /**
     * Tính điểm và lưu kết quả thi
     * @param certId - ID chứng chỉ
     * @param answers - List<Map<String,String>> với keys: 'questionId', 'selectedOption'
     * @return Map<String,Object> chứa 'score', 'status', 'attemptId'
     */
    public static Map<String, Object> calculateAndSaveExam(
        Id certId, 
        List<Map<String, String>> answers
    ) {
        // 1. Query Certification để lấy Passing Score
        Certification__c cert = [
            SELECT Id, Passing_Score__c 
            FROM Certification__c 
            WHERE Id = :certId 
            LIMIT 1
        ];
        
        // 2. Collect tất cả Question IDs để query 1 lần duy nhất
        Set<Id> questionIds = new Set<Id>();
        for (Map<String, String> answer : answers) {
            questionIds.add(answer.get('questionId'));
        }
        
        // 3. Query Questions với đáp án đúng (SINGLE SOQL - không loop)
        Map<Id, Question__c> questionMap = new Map<Id, Question__c>(
            [SELECT Id, Correct_Answer__c 
             FROM Question__c 
             WHERE Id IN :questionIds]
        );
        
        // 4. Tính điểm
        Integer correctCount = 0;
        for (Map<String, String> answer : answers) {
            Id qId = answer.get('questionId');
            String selected = answer.get('selectedOption');
            if (questionMap.containsKey(qId) && 
                questionMap.get(qId).Correct_Answer__c == selected) {
                correctCount++;
            }
        }
        
        Decimal score = (answers.size() > 0) 
            ? ((Decimal)correctCount / answers.size()) * 100 
            : 0;
        String status = (score >= cert.Passing_Score__c) ? 'Pass' : 'Fail';
        
        // 5. Insert Attempt__c (SINGLE DML)
        Attempt__c attempt = new Attempt__c(
            Student__c = UserInfo.getUserId(),
            Certification__c = certId,
            Score__c = score,
            Status__c = status,
            Attempt_Date__c = DateTime.now()
        );
        insert attempt;
        
        // 6. Build & Insert Attempt_Answer__c records (BULK DML)
        List<Attempt_Answer__c> attemptAnswers = new List<Attempt_Answer__c>();
        for (Map<String, String> answer : answers) {
            attemptAnswers.add(new Attempt_Answer__c(
                Attempt__c = attempt.Id,
                Question__c = answer.get('questionId'),
                Selected_Option__c = answer.get('selectedOption')
            ));
        }
        insert attemptAnswers;
        
        // 7. Return result
        Map<String, Object> result = new Map<String, Object>();
        result.put('score', score);
        result.put('status', status);
        result.put('attemptId', attempt.Id);
        result.put('totalQuestions', answers.size());
        result.put('correctAnswers', correctCount);
        return result;
    }
}
```

> 💡 **PD1 Knowledge Points:**
> - `with sharing`: Respects OWD & Sharing Rules
> - SOQL ngoài loop: 1 query Questions, 1 query Certification
> - DML ngoài loop: 1 insert Attempt, 1 insert List<Attempt_Answer__c>
> - `Map<Id, SObject>` constructor: convert list → map in 1 line

---

### Task 2.2: `BootcampController.cls` — `getQuestions` (15 phút)

```apex
public with sharing class BootcampController {
    
    // ⚠️ KHÔNG dùng cacheable=true vì cần random thứ tự mỗi lần gọi
    @AuraEnabled
    public static List<Question__c> getQuestions(Id certificationId) {
        // Query tất cả câu hỏi active
        List<Question__c> questions = [
            SELECT Id, Question_Text__c, Option_A__c, Option_B__c, 
                   Option_C__c, Option_D__c, Certification__c
            FROM Question__c
            WHERE Certification__c = :certificationId
            AND Is_Active__c = true
        ];
        // NOTE: Không trả về Correct_Answer__c về client (chống gian lận)
        // FLS cũng đã hide field này cho Student
        
        // Shuffle random thứ tự câu hỏi (BDD yêu cầu: "active, random")
        // SOQL không hỗ trợ ORDER BY RANDOM(), nên shuffle trong Apex
        Integer n = questions.size();
        while (n > 1) {
            n--;
            Integer k = (Integer)Math.floor(Math.random() * (n + 1));
            Question__c temp = questions[k];
            questions[k] = questions[n];
            questions[n] = temp;
        }
        return questions;
    }
}
```

> ⚠️ **QUAN TRỌNG:** KHÔNG include `Correct_Answer__c` trong SELECT — chống gian lận client-side.

---

### Task 2.3: `BootcampController.cls` — `submitExam` (15 phút)

Thêm method vào `BootcampController`:

```apex
    @AuraEnabled
    public static Map<String, Object> submitExam(
        Id certificationId, 
        String answersJson
    ) {
        try {
            List<Map<String, String>> answers = 
                (List<Map<String, String>>) JSON.deserialize(
                    answersJson, List<Map<String, String>>.class
                );
            return MockExamService.calculateAndSaveExam(certificationId, answers);
        } catch (Exception e) {
            throw new AuraHandledException(e.getMessage());
        }
    }
```

> 💡 **PD1 Knowledge:**
> - `@AuraEnabled` (không có `cacheable`) cho DML methods
> - `AuraHandledException`: proper error handling cho LWC
> - `JSON.deserialize`: convert JSON string từ LWC → Apex Collection

---

### Task 2.4: `BootcampController.cls` — `getCertifications` (10 phút)

```apex
    @AuraEnabled(cacheable=true)
    public static List<Certification__c> getCertifications() {
        return [
            SELECT Id, Name, Passing_Score__c, Total_Enrolled__c
            FROM Certification__c
            ORDER BY Name
        ];
    }
```

---

### Task 2.5: Error Handling Pattern (10 phút)

**Review tất cả `@AuraEnabled` methods** và đảm bảo pattern:

```apex
    @AuraEnabled
    public static ReturnType methodName(params) {
        try {
            // Logic here
        } catch (Exception e) {
            throw new AuraHandledException(e.getMessage());
        }
    }
```

> ⚠️ `cacheable=true` methods không cần try-catch vì chúng chỉ đọc data. Nhưng non-cacheable (DML) methods **BẮT BUỘC** có try-catch.

---

## 🟡 TUẦN 2 — Controller hoàn thiện + Tests (1.5h)

### Task 2.6: `getDashboardStats` (20 phút)

Thêm vào `BootcampController`:

```apex
    // Wrapper class cho Dashboard data
    public class DashboardStats {
        @AuraEnabled public Decimal averageScore;
        @AuraEnabled public String readiness;
        @AuraEnabled public String nextAction;
        @AuraEnabled public Integer totalCertsEnrolled;
        @AuraEnabled public List<Attempt__c> recentAttempts;
    }
    
    @AuraEnabled(cacheable=true)
    public static DashboardStats getDashboardStats() {
        Id userId = UserInfo.getUserId();
        DashboardStats stats = new DashboardStats();
        
        // 1. Query attempts cho current user
        List<Attempt__c> attempts = [
            SELECT Id, Score__c, Status__c, Attempt_Date__c, 
                   Certification__r.Name, Certification__r.Passing_Score__c
            FROM Attempt__c
            WHERE Student__c = :userId
            ORDER BY Attempt_Date__c DESC
        ];
        
        // 2. Recent Attempts (last 5)
        stats.recentAttempts = new List<Attempt__c>();
        for (Integer i = 0; i < Math.min(5, attempts.size()); i++) {
            stats.recentAttempts.add(attempts[i]);
        }
        
        // 3. Calculate Average Score
        if (attempts.isEmpty()) {
            stats.averageScore = 0;
            stats.readiness = 'Not Ready';
            stats.nextAction = 'Take First Mock Exam';
        } else {
            Decimal totalScore = 0;
            for (Attempt__c a : attempts) {
                totalScore += a.Score__c;
            }
            stats.averageScore = totalScore / attempts.size();
            
            // 4. Readiness Classification
            if (stats.averageScore >= 80) {
                stats.readiness = 'Ready';
            } else if (stats.averageScore >= 65) {
                stats.readiness = 'At Risk';
            } else {
                stats.readiness = 'Not Ready';
            }
            
            // 5. Next Action (based on last attempt)
            Attempt__c lastAttempt = attempts[0];
            if (lastAttempt.Score__c >= lastAttempt.Certification__r.Passing_Score__c) {
                stats.nextAction = 'Ready for Certification';
            } else {
                stats.nextAction = 'Retake Exam';
            }
        }
        
        // 6. Total Certs Enrolled
        stats.totalCertsEnrolled = [
            SELECT COUNT() FROM Enrollment__c 
            WHERE Student__c = :userId 
            AND Status__c = 'Approved'
        ];
        
        return stats;
    }
```

---

### Task 2.7: `getLeaderboard` (15 phút)

```apex
    @AuraEnabled(cacheable=true)
    public static List<AggregateResult> getLeaderboard() {
        return [
            SELECT Student__c, Student__r.Name studentName, 
                   AVG(Score__c) avgScore
            FROM Attempt__c
            GROUP BY Student__c, Student__r.Name
            ORDER BY AVG(Score__c) DESC
            LIMIT 5
        ];
    }
```

> 💡 **PD1 Knowledge:** SOQL Aggregate queries với `GROUP BY`, `AVG()`, `ORDER BY` aggregate function.

---

### Task 2.8: `getAllQuestions` — cho Admin (10 phút)

```apex
    @AuraEnabled(cacheable=true)
    public static List<Question__c> getAllQuestions() {
        return [
            SELECT Id, Question_Text__c, Option_A__c, Option_B__c,
                   Option_C__c, Option_D__c, Correct_Answer__c,
                   Is_Active__c, Certification__r.Name
            FROM Question__c
            ORDER BY Certification__r.Name, CreatedDate DESC
        ];
    }
```

> ⚠️ Method này include `Correct_Answer__c` vì chỉ Admin dùng. FLS sẽ tự block Student.

---

### Task 2.9: `BootcampTestFactory.cls` (20 phút)

```apex
@isTest
public class BootcampTestFactory {
    
    public static Certification__c createCertification(String name, Decimal passingScore) {
        Certification__c cert = new Certification__c(
            Name = name,
            Passing_Score__c = passingScore
        );
        insert cert;
        return cert;
    }
    
    public static List<Question__c> createQuestions(
        Id certId, Integer count, Boolean isActive
    ) {
        List<Question__c> questions = new List<Question__c>();
        for (Integer i = 0; i < count; i++) {
            questions.add(new Question__c(
                Certification__c = certId,
                Question_Text__c = 'Test Question ' + i,
                Option_A__c = 'Option A',
                Option_B__c = 'Option B',
                Option_C__c = 'Option C',
                Option_D__c = 'Option D',
                Correct_Answer__c = 'A',
                Is_Active__c = isActive
            ));
        }
        insert questions;
        return questions;
    }
    
    public static User createStudentUser(String alias) {
        Profile p = [
            SELECT Id FROM Profile 
            WHERE Name = 'Bootcamp_Student' LIMIT 1
        ];
        User u = new User(
            FirstName = 'Test',
            LastName = alias,
            Email = alias + '@test.bootcamp.com',
            Username = alias + '@test.bootcamp.com.' + 
                       DateTime.now().getTime(),
            Alias = alias.left(8),
            TimeZoneSidKey = 'America/Los_Angeles',
            LocaleSidKey = 'en_US',
            EmailEncodingKey = 'UTF-8',
            ProfileId = p.Id,
            LanguageLocaleKey = 'en_US'
        );
        insert u;
        return u;
    }
    
    public static Enrollment__c createEnrollment(
        Id certId, Id userId, String status
    ) {
        Enrollment__c enr = new Enrollment__c(
            Certification__c = certId,
            Student__c = userId,
            Status__c = status,
            Enrolled_Date__c = DateTime.now()
        );
        insert enr;
        return enr;
    }
}
```

> 💡 **PD1 Knowledge:** Test Data Factory pattern — DRY principle, tái sử dụng cho tất cả Test Classes.

---

### Task 2.10: `MockExamServiceTest.cls` (25 phút)

```apex
@isTest
private class MockExamServiceTest {
    
    @TestSetup
    static void setupData() {
        Certification__c cert = BootcampTestFactory.createCertification(
            'PD1 Test', 65
        );
        BootcampTestFactory.createQuestions(cert.Id, 10, true);
    }
    
    @isTest
    static void testSubmitExam_AllCorrect_ShouldPass() {
        Certification__c cert = [
            SELECT Id FROM Certification__c LIMIT 1
        ];
        List<Question__c> questions = [
            SELECT Id, Correct_Answer__c FROM Question__c
        ];
        
        // Build answer list — tất cả đúng
        List<Map<String, String>> answers = new List<Map<String, String>>();
        for (Question__c q : questions) {
            Map<String, String> ans = new Map<String, String>();
            ans.put('questionId', q.Id);
            ans.put('selectedOption', q.Correct_Answer__c); // Correct!
            answers.add(ans);
        }
        
        Test.startTest();
        Map<String, Object> result = 
            MockExamService.calculateAndSaveExam(cert.Id, answers);
        Test.stopTest();
        
        // Assertions
        System.assertEquals(100, (Decimal)result.get('score'), 
            'Score should be 100% when all answers correct');
        System.assertEquals('Pass', (String)result.get('status'),
            'Status should be Pass');
        
        // Verify DML
        List<Attempt__c> attempts = [
            SELECT Id, Score__c FROM Attempt__c
        ];
        System.assertEquals(1, attempts.size(), 
            'Should create 1 Attempt record');
        
        List<Attempt_Answer__c> attemptAnswers = [
            SELECT Id FROM Attempt_Answer__c
        ];
        System.assertEquals(10, attemptAnswers.size(),
            'Should create 10 Attempt_Answer records');
    }
    
    @isTest
    static void testSubmitExam_AllWrong_ShouldFail() {
        Certification__c cert = [
            SELECT Id FROM Certification__c LIMIT 1
        ];
        List<Question__c> questions = [
            SELECT Id FROM Question__c
        ];
        
        // Build answer list — tất cả sai (chọn D, correct là A)
        List<Map<String, String>> answers = new List<Map<String, String>>();
        for (Question__c q : questions) {
            Map<String, String> ans = new Map<String, String>();
            ans.put('questionId', q.Id);
            ans.put('selectedOption', 'D'); // Wrong (correct is A)
            answers.add(ans);
        }
        
        Test.startTest();
        Map<String, Object> result = 
            MockExamService.calculateAndSaveExam(cert.Id, answers);
        Test.stopTest();
        
        System.assertEquals(0, (Decimal)result.get('score'));
        System.assertEquals('Fail', (String)result.get('status'));
    }
    
    @isTest
    static void testSubmitExam_EmptyAnswers_ShouldHandleGracefully() {
        Certification__c cert = [
            SELECT Id FROM Certification__c LIMIT 1
        ];
        
        List<Map<String, String>> answers = new List<Map<String, String>>();
        
        Test.startTest();
        Map<String, Object> result = 
            MockExamService.calculateAndSaveExam(cert.Id, answers);
        Test.stopTest();
        
        System.assertEquals(0, (Decimal)result.get('score'));
    }
}
```

---

## 🟢 TUẦN 3 — Controller Test + Coverage (1.2h)

### Task 2.11: `BootcampControllerTest.cls` (30 phút)

```apex
@isTest
private class BootcampControllerTest {
    
    @TestSetup
    static void setupData() {
        Certification__c cert = BootcampTestFactory.createCertification(
            'Admin Test', 65
        );
        BootcampTestFactory.createQuestions(cert.Id, 5, true);
        BootcampTestFactory.createQuestions(cert.Id, 2, false); // Inactive
    }
    
    @isTest
    static void testGetCertifications() {
        List<Certification__c> certs = 
            BootcampController.getCertifications();
        System.assertEquals(1, certs.size());
    }
    
    @isTest
    static void testGetQuestions_OnlyActive() {
        Certification__c cert = [
            SELECT Id FROM Certification__c LIMIT 1
        ];
        List<Question__c> questions = 
            BootcampController.getQuestions(cert.Id);
        System.assertEquals(5, questions.size(), 
            'Should return only active questions');
    }
    
    @isTest
    static void testGetAllQuestions_IncludesInactive() {
        List<Question__c> questions = 
            BootcampController.getAllQuestions();
        System.assertEquals(7, questions.size(),
            'Should return ALL questions including inactive');
    }
    
    @isTest
    static void testGetDashboardStats_NoAttempts() {
        BootcampController.DashboardStats stats = 
            BootcampController.getDashboardStats();
        System.assertEquals('Not Ready', stats.readiness);
        System.assertEquals('Take First Mock Exam', stats.nextAction);
    }
    
    @isTest
    static void testGetLeaderboard() {
        List<AggregateResult> lb = 
            BootcampController.getLeaderboard();
        System.assertNotEquals(null, lb);
    }
    
    @isTest
    static void testSubmitExam() {
        Certification__c cert = [
            SELECT Id FROM Certification__c LIMIT 1
        ];
        List<Question__c> questions = [
            SELECT Id FROM Question__c WHERE Is_Active__c = true
        ];
        
        List<Map<String, String>> answers = new List<Map<String, String>>();
        for (Question__c q : questions) {
            answers.add(new Map<String, String>{
                'questionId' => q.Id,
                'selectedOption' => 'A'
            });
        }
        
        Test.startTest();
        Map<String, Object> result = 
            BootcampController.submitExam(cert.Id, JSON.serialize(answers));
        Test.stopTest();
        
        System.assertNotEquals(null, result.get('score'));
    }
}
```

---

### Task 2.12: Security Test (15 phút)

Thêm test method vào `BootcampControllerTest`:

```apex
    @isTest
    static void testStudentCannotSeeCorrectAnswer() {
        User student = BootcampTestFactory.createStudentUser('stud1');
        
        System.runAs(student) {
            List<Question__c> questions = 
                BootcampController.getQuestions(
                    [SELECT Id FROM Certification__c LIMIT 1].Id
                );
            // Verify Correct_Answer__c is not accessible
            // (field not in SELECT, plus FLS blocks it)
            for (Question__c q : questions) {
                System.assertEquals(null, q.get('Correct_Answer__c'),
                    'Student should NOT see Correct Answer');
            }
        }
    }
```

---

### Task 2.13: Bug Fixes (15 phút)

Nhận bug list từ Member 5 → Fix issues trong `BootcampController` hoặc `MockExamService`.

---

### Task 2.14: Coverage Check (10 phút)

1. Developer Console → Test → Run All
2. Hoặc CLI: `sfdx force:apex:test:run --codecoverage --resultformat human`
3. Target: ≥ 75%, mục tiêu ≥ 85%
4. Nếu thiếu → thêm test cases cho uncovered lines

---

## ✅ Deliverables tổng kết Member 2

| # | Deliverable | Status |
|:---|:---|:---|
| 1 | `MockExamService.cls` — complete | ⬜ |
| 2 | `BootcampController.cls` — 6+ methods | ⬜ |
| 3 | `BootcampTestFactory.cls` — 4 factory methods | ⬜ |
| 4 | `MockExamServiceTest.cls` — positive + negative | ⬜ |
| 5 | `BootcampControllerTest.cls` — all methods covered | ⬜ |
| 6 | Security Test `System.runAs` | ⬜ |
| 7 | Code Coverage ≥ 75% | ⬜ |
