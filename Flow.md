# Business Process Flows — Salesforce Certification Bootcamp Manager

---

## Flow 1: Student Enrollment Process

```mermaid
flowchart TD
    A([Start]) --> B[Student opens Lightning App]
    B --> C[Launch Screen Flow: Exam Registration Wizard]
    C --> D[Step 1: Display available Certifications from Certification__c]
    D --> E{Student selects a Certification?}
    E -- No --> D
    E -- Yes --> F[Step 2: Show confirmation screen with Passing Score]
    F --> G{Student confirms?}
    G -- Cancel --> Z([End: Abandoned])
    G -- Confirm --> H[Step 3: Create Enrollment__c record - Status = Pending]
    H --> I[Record-Triggered Flow fires: Set Enrolled_Date__c = NOW]
    I --> J[System submits Enrollment for Approval Process]
    J --> K[Admin receives approval notification]
    K --> L{Admin Decision}
    L -- Approve --> M[Status = Approved - Student can take Mock Exams]
    L -- Reject --> N[Status = Rejected - Student is notified]
    M --> O([End: Approved])
    N --> P([End: Rejected])
```

---

## Flow 2: Mock Exam Process

```mermaid
flowchart TD
    A([Start]) --> B[Student navigates to Mock Exam Page]
    B --> C[Select Certification from dropdown]
    C --> D[Apex call: BootcampController.getQuestions where Is_Active__c = true]
    D --> E{Questions found?}
    E -- No --> F[Show error: No active questions available]
    F --> Z([End])
    E -- Yes --> G[Display Question 1 with Options A B C D]
    G --> H{Student selects an answer?}
    H -- Pending --> H
    H -- Selected --> I[Store answer in JS client array: questionId + selectedOption]
    I --> J{More questions remaining?}
    J -- Next --> K[Display next question]
    K --> H
    J -- Previous --> L[Navigate to previous question]
    L --> H
    J -- Submit --> M[Send answer array to Apex via Imperative Call]
    M --> N[MockExamService.calculateScore: compare each Selected_Option vs Correct_Answer__c]
    N --> O[Compute Score = Correct Answers divided by Total Questions x 100]
    O --> P{Score >= Certification Passing_Score__c?}
    P -- Yes --> Q[Status = Pass]
    P -- No --> R[Status = Fail]
    Q --> S[DML Insert: Attempt__c with Score + Status + Attempt_Date]
    R --> S
    S --> T[DML Insert: List of Attempt_Answer__c with Is_Correct__c Formula]
    T --> U[AttemptTrigger fires: After Insert on Attempt__c]
    U --> V[AttemptTriggerHandler: Update related Enrollment__c if needed]
    V --> W[Return Score + Status to LWC component]
    W --> X[Display Result Modal: Score % and Pass/Fail badge]
    X --> Y([End: Attempt Saved])
```

---

## Flow 3: Dashboard Data Loading

```mermaid
flowchart TD
    A([Start]) --> B[Student opens Home Page]
    B --> C[dashboardView LWC loads]
    C --> D[Wire Adapter calls: BootcampController.getDashboardStats with currentUserId]
    D --> E[SOQL: Query Attempt__c for current user ORDER BY Attempt_Date DESC]
    E --> F{Any Attempts found?}
    F -- No --> G[Next Action = Take First Mock Exam / Readiness = Not Ready]
    F -- Yes --> H[Calculate Average Score from all Score__c]
    H --> I{Average Score >= 80%?}
    I -- Yes --> J[Readiness = Ready]
    I -- No --> K{Average Score >= 65%?}
    K -- Yes --> L[Readiness = At Risk]
    K -- No --> M[Readiness = Not Ready]
    J --> N[Check last Attempt Score vs Passing_Score__c]
    L --> N
    M --> N
    N --> O{Last Score >= Passing Score?}
    O -- Yes --> P[Next Action = Ready for Certification]
    O -- No --> Q[Next Action = Retake Exam]
    G --> R[Return DashboardWrapper to LWC]
    P --> R
    Q --> R
    R --> S[SOQL: Top 5 users GROUP BY Student__c AVG Score DESC LIMIT 5]
    S --> T[Return Leaderboard list to LWC]
    T --> U[Render Dashboard: Stats Cards + Progress Bar + Leaderboard + Recent Attempts]
    U --> V([End: Dashboard Displayed])
```

---

## Flow 4: Admin Question Management

```mermaid
flowchart TD
    A([Start]) --> B[Admin navigates to Admin Questions Tab]
    B --> C[adminQuestionManage LWC loads]
    C --> D[Wire: BootcampController.getAllQuestions]
    D --> E[Display lightning-datatable with all Questions]
    E --> F{Admin Action?}
    F -- New --> G[Open newQuestionModal with lightning-record-edit-form]
    G --> H[Admin fills in Certification + Question Text + Options A B C D + Correct Answer]
    H --> I{Validation Rule passes?}
    I -- Fail --> J[Show error: Must fill all 4 options]
    J --> H
    I -- Pass --> K[DML Insert: Question__c with Is_Active__c = true]
    K --> L[Refresh datatable]
    L --> E
    F -- Edit --> M[Open modal pre-populated with existing record]
    M --> N[Admin updates fields]
    N --> O{Validation passes?}
    O -- Fail --> P[Show error message]
    P --> N
    O -- Pass --> Q[DML Update: Question__c]
    Q --> L
    F -- Activate/Deactivate --> R[Toggle Is_Active__c boolean]
    R --> S{Is_Active__c = true?}
    S -- Active --> T[Question appears in exam pool]
    S -- Inactive --> U[Question hidden from exam pool]
    T --> L
    U --> L
    F -- Done --> V([End])
```

---

## Flow 5: Record-Triggered Flow — Auto Set Enrollment Date

```mermaid
flowchart TD
    A([Trigger: Enrollment__c After Insert]) --> B{Enrolled_Date__c is NULL?}
    B -- Yes --> C[Set Enrolled_Date__c = NOW]
    C --> D[Update Enrollment__c record]
    D --> E([End: Date stamped])
    B -- No --> F([End: No action needed])
```

---

## Flow 6: Approval Process — Enrollment Approval

```mermaid
flowchart TD
    A([Student submits Enrollment__c for Approval]) --> B[Approval Process: Enrollment_Approval initiates]
    B --> C[System sends approval request to assigned Admin Approver]
    C --> D{Admin reviews Enrollment record}
    D -- Approve --> E[Final Action: Update Status__c = Approved]
    D -- Reject --> F[Rejection Action: Update Status__c = Rejected]
    E --> G[Student can now take Mock Exams for this Certification]
    F --> H[Student is notified of rejection]
    G --> I([End: Approved])
    H --> J([End: Rejected])
```

---

## Flow 7: Async Apex — LeaderboardRefreshBatch

```mermaid
flowchart TD
    A([Scheduled Job triggers Daily at 02:00 AM]) --> B[LeaderboardRefreshBatch Schedulable.execute]
    B --> C[Database.executeBatch called with Batch size = 200]
    C --> D[start method: QueryLocator returns all Attempt__c records]
    D --> E[execute method: Process each batch chunk]
    E --> F[Aggregate AVG Score__c GROUP BY Student__c]
    F --> G{More chunks?}
    G -- Yes --> E
    G -- No --> H[finish method: Log batch completion]
    H --> I([End: Leaderboard cache refreshed])
```

---

## Flow 8: Apex Trigger — AttemptTriggerHandler

```mermaid
flowchart TD
    A([DML Insert on Attempt__c]) --> B[AttemptTrigger fires: After Insert]
    B --> C[Delegate to AttemptTriggerHandler.handleAfterInsert with Trigger.new list]
    C --> D[Collect unique Certification__c + Student__c pairs using Set - no loop SOQL]
    D --> E[SOQL: Query related Enrollment__c records in bulk - one query]
    E --> F[Build Map: CertId+StudentId to Enrollment__c]
    F --> G[Loop through new Attempts and update Enrollment fields if needed]
    G --> H{Any Enrollment records to update?}
    H -- Yes --> I[Single DML Update: List of Enrollment__c]
    H -- No --> J([End: No updates needed])
    I --> K([End: Enrollments updated])
```
