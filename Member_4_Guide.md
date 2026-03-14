# 👤 Member 4 — LWC UI Components
## Hướng dẫn Step-by-Step

> **Phạm vi:** 4 LWC Components: `mockExamFlow`, `dashboardView`, `adminQuestionManage`, `newQuestionModal`
>
> **Prerequisite:** Member 1 đã tạo Objects. Member 2 đã viết `BootcampController` methods.

---

## 🔴 TUẦN 1 — mockExamFlow + dashboardView Skeleton (1.5h)

### Task 4.1: `mockExamFlow` — HTML Structure (30 phút)

**Bước 1:** Tạo LWC component: VS Code → SFDX: Create Lightning Web Component → `mockExamFlow`

**`mockExamFlow.html`:**

```html
<template>
    <!-- ====== CERTIFICATION SELECTION ====== -->
    <template if:false={isExamStarted}>
        <lightning-card title="Mock Exam" icon-name="standard:education">
            <div class="slds-p-around_medium">
                <lightning-combobox
                    name="certification"
                    label="Select Certification"
                    placeholder="-- Choose Certification --"
                    options={certificationOptions}
                    value={selectedCertId}
                    onchange={handleCertChange}
                ></lightning-combobox>
                
                <div class="slds-m-top_medium">
                    <lightning-button 
                        variant="brand" 
                        label="Start Exam" 
                        onclick={startExam}
                        disabled={isStartDisabled}
                    ></lightning-button>
                </div>
            </div>
        </lightning-card>
    </template>

    <!-- ====== EXAM IN PROGRESS ====== -->
    <template if:true={isExamStarted}>
        <template if:false={isSubmitted}>
            <lightning-card title={examTitle} icon-name="standard:question_feed">
                <!-- Progress Indicator -->
                <div class="slds-p-around_medium">
                    <lightning-progress-bar 
                        value={progressPercent} 
                        size="large"
                    ></lightning-progress-bar>
                    <p class="slds-m-top_x-small slds-text-align_center">
                        Question {currentQuestionNumber} of {totalQuestions}
                    </p>
                </div>

                <!-- Question Display -->
                <div class="slds-p-around_medium">
                    <h2 class="slds-text-heading_small slds-m-bottom_small">
                        {currentQuestion.Question_Text__c}
                    </h2>
                    
                    <lightning-radio-group
                        name="answer"
                        label="Select your answer"
                        options={answerOptions}
                        value={selectedAnswer}
                        onchange={handleAnswerChange}
                        type="radio"
                    ></lightning-radio-group>
                </div>

                <!-- Navigation Buttons -->
                <div class="slds-p-around_medium slds-grid slds-grid_align-spread">
                    <lightning-button 
                        label="Previous" 
                        onclick={handlePrevious}
                        disabled={isPrevDisabled}
                    ></lightning-button>
                    
                    <template if:false={isLastQuestion}>
                        <lightning-button 
                            variant="brand"
                            label="Next" 
                            onclick={handleNext}
                        ></lightning-button>
                    </template>
                    
                    <template if:true={isLastQuestion}>
                        <lightning-button 
                            variant="brand"
                            label="Submit Exam" 
                            onclick={handleSubmit}
                        ></lightning-button>
                    </template>
                </div>
            </lightning-card>
        </template>

        <!-- ====== RESULT MODAL ====== -->
        <template if:true={isSubmitted}>
            <section role="dialog" class="slds-modal slds-fade-in-open">
                <div class="slds-modal__container">
                    <header class="slds-modal__header">
                        <h2 class="slds-modal__title">Exam Results</h2>
                    </header>
                    <div class="slds-modal__content slds-p-around_medium slds-text-align_center">
                        <div class="slds-m-bottom_medium">
                            <span class={resultBadgeClass}>
                                {examResult.status}
                            </span>
                        </div>
                        <p class="slds-text-heading_large">{examResult.score}%</p>
                        <p class="slds-m-top_small">
                            {examResult.correctAnswers} / {examResult.totalQuestions} correct
                        </p>
                    </div>
                    <footer class="slds-modal__footer">
                        <lightning-button 
                            variant="brand" 
                            label="Back to Home" 
                            onclick={handleBackToHome}
                        ></lightning-button>
                    </footer>
                </div>
            </section>
            <div class="slds-backdrop slds-backdrop_open"></div>
        </template>
    </template>
</template>
```

---

### Task 4.2: `mockExamFlow` — JS Controller (30 phút)

**`mockExamFlow.js`:**

```javascript
import { LightningElement, track } from 'lwc';
import getCertifications from '@salesforce/apex/BootcampController.getCertifications';
import getQuestions from '@salesforce/apex/BootcampController.getQuestions';
import submitExam from '@salesforce/apex/BootcampController.submitExam';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

export default class MockExamFlow extends LightningElement {
    // State
    certificationOptions = [];
    selectedCertId = '';
    @track questions = [];
    currentIndex = 0;
    @track answers = {}; // { questionId: selectedOption }
    isExamStarted = false;
    isSubmitted = false;
    examResult = {};

    // ====== LIFECYCLE ======
    connectedCallback() {
        this.loadCertifications();
    }

    async loadCertifications() {
        try {
            const certs = await getCertifications();
            this.certificationOptions = certs.map(cert => ({
                label: cert.Name,
                value: cert.Id
            }));
        } catch (error) {
            this.showToast('Error', error.body.message, 'error');
        }
    }

    // ====== HANDLERS ======
    handleCertChange(event) {
        this.selectedCertId = event.detail.value;
    }

    async startExam() {
        try {
            const questions = await getQuestions({ 
                certificationId: this.selectedCertId 
            });
            if (questions.length === 0) {
                this.showToast('Warning', 
                    'No active questions available for this certification', 
                    'warning');
                return;
            }
            this.questions = questions;
            this.currentIndex = 0;
            this.answers = {};
            this.isExamStarted = true;
            this.isSubmitted = false;
        } catch (error) {
            this.showToast('Error', error.body.message, 'error');
        }
    }

    handleAnswerChange(event) {
        const questionId = this.questions[this.currentIndex].Id;
        this.answers[questionId] = event.detail.value;
    }

    handlePrevious() {
        if (this.currentIndex > 0) {
            this.currentIndex--;
        }
    }

    handleNext() {
        if (this.currentIndex < this.questions.length - 1) {
            this.currentIndex++;
        }
    }

    async handleSubmit() {
        try {
            // Build answer array for Apex
            const answerList = Object.keys(this.answers).map(qId => ({
                questionId: qId,
                selectedOption: this.answers[qId]
            }));

            const result = await submitExam({
                certificationId: this.selectedCertId,
                answersJson: JSON.stringify(answerList)
            });

            this.examResult = result;
            this.isSubmitted = true;
        } catch (error) {
            this.showToast('Error', error.body?.message || 'Submission failed', 'error');
        }
    }

    handleBackToHome() {
        this.isExamStarted = false;
        this.isSubmitted = false;
        this.selectedCertId = '';
    }

    // ====== GETTERS ======
    get isStartDisabled() {
        return !this.selectedCertId;
    }

    get currentQuestion() {
        return this.questions[this.currentIndex] || {};
    }

    get currentQuestionNumber() {
        return this.currentIndex + 1;
    }

    get totalQuestions() {
        return this.questions.length;
    }

    get progressPercent() {
        return this.questions.length > 0 
            ? Math.round(((this.currentIndex + 1) / this.questions.length) * 100)
            : 0;
    }

    get answerOptions() {
        const q = this.currentQuestion;
        return [
            { label: `A. ${q.Option_A__c || ''}`, value: 'A' },
            { label: `B. ${q.Option_B__c || ''}`, value: 'B' },
            { label: `C. ${q.Option_C__c || ''}`, value: 'C' },
            { label: `D. ${q.Option_D__c || ''}`, value: 'D' }
        ];
    }

    get selectedAnswer() {
        const qId = this.currentQuestion.Id;
        return this.answers[qId] || '';
    }

    get isPrevDisabled() {
        return this.currentIndex === 0;
    }

    get isLastQuestion() {
        return this.currentIndex === this.questions.length - 1;
    }

    get examTitle() {
        return `Mock Exam - Question ${this.currentQuestionNumber}/${this.totalQuestions}`;
    }

    get resultBadgeClass() {
        return this.examResult.status === 'Pass'
            ? 'slds-badge slds-theme_success'
            : 'slds-badge slds-theme_error';
    }

    // ====== UTILITIES ======
    showToast(title, message, variant) {
        this.dispatchEvent(new ShowToastEvent({ title, message, variant }));
    }
}
```

---

### Task 4.3: `mockExamFlow` — Meta XML (5 phút)

**`mockExamFlow.js-meta.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__AppPage</target>
        <target>lightning__RecordPage</target>
        <target>lightning__HomePage</target>
    </targets>
</LightningComponentBundle>
```

---

### Task 4.4: `dashboardView` — Skeleton (15 phút)

**Tạo component → `dashboardView.html`:**

```html
<template>
    <lightning-card title="My Dashboard" icon-name="standard:dashboard">
        <!-- Stats Cards Row -->
        <div class="slds-grid slds-wrap slds-gutters slds-p-around_medium">
            <!-- Total Certs Enrolled -->
            <div class="slds-col slds-size_1-of-3">
                <div class="slds-box slds-text-align_center">
                    <p class="slds-text-title_caps">Enrolled Certifications</p>
                    <p class="slds-text-heading_large">{stats.totalCertsEnrolled}</p>
                </div>
            </div>
            <!-- Average Score -->
            <div class="slds-col slds-size_1-of-3">
                <div class="slds-box slds-text-align_center">
                    <p class="slds-text-title_caps">Average Score</p>
                    <p class="slds-text-heading_large">{formattedAvgScore}%</p>
                </div>
            </div>
            <!-- Readiness -->
            <div class="slds-col slds-size_1-of-3">
                <div class="slds-box slds-text-align_center">
                    <p class="slds-text-title_caps">Readiness</p>
                    <span class={readinessBadgeClass}>{stats.readiness}</span>
                </div>
            </div>
        </div>

        <!-- Progress Bar -->
        <div class="slds-p-horizontal_medium slds-m-bottom_medium">
            <lightning-progress-bar value={stats.averageScore} size="large">
            </lightning-progress-bar>
        </div>

        <!-- Next Action placeholder - will be completed in Week 3 -->
        <div class="slds-p-around_medium">
            <p class="slds-text-body_regular">
                <strong>Next Action:</strong> {stats.nextAction}
            </p>
        </div>

        <!-- Leaderboard placeholder -->
        <!-- Recent Attempts placeholder -->
    </lightning-card>
</template>
```

**`dashboardView.js`:**

```javascript
import { LightningElement, wire } from 'lwc';
import getDashboardStats from '@salesforce/apex/BootcampController.getDashboardStats';
import getLeaderboard from '@salesforce/apex/BootcampController.getLeaderboard';

export default class DashboardView extends LightningElement {
    stats = {};
    leaderboard = [];
    error;

    @wire(getDashboardStats)
    wiredStats({ error, data }) {
        if (data) {
            this.stats = data;
            this.error = undefined;
        } else if (error) {
            this.error = error;
        }
    }

    @wire(getLeaderboard)
    wiredLeaderboard({ error, data }) {
        if (data) {
            this.leaderboard = data;
        } else if (error) {
            this.error = error;
        }
    }

    get formattedAvgScore() {
        return this.stats.averageScore 
            ? Math.round(this.stats.averageScore) : 0;
    }

    get readinessBadgeClass() {
        const r = this.stats.readiness;
        if (r === 'Ready') return 'slds-badge slds-theme_success';
        if (r === 'At Risk') return 'slds-badge slds-theme_warning';
        return 'slds-badge slds-theme_error';
    }
}
```

**`dashboardView.js-meta.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__AppPage</target>
        <target>lightning__HomePage</target>
    </targets>
</LightningComponentBundle>
```

---

## 🟡 TUẦN 2 — Dashboard Completion + Admin Components (1.5h)

### Task 4.5: `dashboardView` — Stats Cards (20 phút)

Đã viết skeleton ở trên. Tuần 2 verify wire adapter hoạt động với data thật từ Member 2's controller.

**Checklist:**
- [ ] Wire `getDashboardStats` trả data đúng
- [ ] 3 stats cards hiển thị: Enrolled count, Avg Score, Readiness badge
- [ ] Progress bar value = averageScore
- [ ] Next Action text hiển thị đúng logic

---

### Task 4.6: `dashboardView` — Leaderboard Table (15 phút)

Thêm vào `dashboardView.html` (sau Next Action section):

```html
        <!-- Leaderboard -->
        <div class="slds-p-around_medium">
            <h3 class="slds-text-heading_small slds-m-bottom_small">
                🏆 Top 5 Leaderboard
            </h3>
            <table class="slds-table slds-table_bordered">
                <thead>
                    <tr>
                        <th>#</th>
                        <th>Student</th>
                        <th>Avg Score</th>
                    </tr>
                </thead>
                <tbody>
                    <template for:each={leaderboardRows} for:item="row">
                        <tr key={row.rank}>
                            <td>{row.rank}</td>
                            <td>{row.name}</td>
                            <td>{row.score}%</td>
                        </tr>
                    </template>
                </tbody>
            </table>
        </div>
```

Thêm getter trong JS:

```javascript
    get leaderboardRows() {
        return this.leaderboard.map((item, index) => ({
            rank: index + 1,
            name: item.studentName || 'Unknown',
            score: Math.round(item.avgScore || 0)
        }));
    }
```

> 💡 **PD1 Knowledge:** `@wire` adapter tự fetch data khi component mount. `AggregateResult` trả field names dựa trên alias trong SOQL.

---

### Task 4.7: `dashboardView` — Recent Attempts (15 phút)

Thêm vào `dashboardView.html`:

```html
        <!-- Recent Attempts -->
        <div class="slds-p-around_medium">
            <h3 class="slds-text-heading_small slds-m-bottom_small">
                📋 Recent Attempts
            </h3>
            <lightning-datatable
                key-field="Id"
                data={stats.recentAttempts}
                columns={attemptColumns}
                hide-checkbox-column
            ></lightning-datatable>
        </div>
```

Thêm trong JS:

```javascript
    attemptColumns = [
        { label: 'Certification', fieldName: 'certName', type: 'text' },
        { label: 'Score', fieldName: 'Score__c', type: 'percent' },
        { label: 'Status', fieldName: 'Status__c', type: 'text' },
        { label: 'Date', fieldName: 'Attempt_Date__c', type: 'date' }
    ];
```

> ⚠️ Có thể cần transform data vì relationship field `Certification__r.Name` không auto-flatten trong datatable. Map data trước khi assign.

---

### Task 4.8: `adminQuestionManage` — Datatable (20 phút)

**`adminQuestionManage.html`:**

```html
<template>
    <lightning-card title="Question Management" icon-name="standard:question_feed">
        <lightning-button 
            label="New Question" 
            slot="actions" 
            onclick={handleNewQuestion}
            variant="brand"
        ></lightning-button>

        <div class="slds-p-around_medium">
            <lightning-datatable
                key-field="Id"
                data={questions}
                columns={columns}
                onrowaction={handleRowAction}
                hide-checkbox-column
            ></lightning-datatable>
        </div>
    </lightning-card>

    <!-- New Question Modal -->
    <template if:true={isModalOpen}>
        <c-new-question-modal
            record-id={editRecordId}
            onclose={handleModalClose}
            onsave={handleModalSave}
        ></c-new-question-modal>
    </template>
</template>
```

**`adminQuestionManage.js`:**

```javascript
import { LightningElement, wire } from 'lwc';
import getAllQuestions from '@salesforce/apex/BootcampController.getAllQuestions';
import { refreshApex } from '@salesforce/apex';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

export default class AdminQuestionManage extends LightningElement {
    questions = [];
    wiredQuestionsResult;
    isModalOpen = false;
    editRecordId = null;

    columns = [
        { label: 'Certification', fieldName: 'certName', type: 'text' },
        { label: 'Question', fieldName: 'Question_Text__c', type: 'text',
          wrapText: true },
        { label: 'Active', fieldName: 'Is_Active__c', type: 'boolean' },
        { label: 'Correct', fieldName: 'Correct_Answer__c', type: 'text' },
        {
            type: 'action',
            typeAttributes: {
                rowActions: [
                    { label: 'Edit', name: 'edit' },
                    { label: 'Toggle Active', name: 'toggle' }
                ]
            }
        }
    ];

    @wire(getAllQuestions)
    wiredQuestions(result) {
        this.wiredQuestionsResult = result;
        if (result.data) {
            this.questions = result.data.map(q => ({
                ...q,
                certName: q.Certification__r 
                    ? q.Certification__r.Name : 'N/A'
            }));
        }
    }

    handleNewQuestion() {
        this.editRecordId = null;
        this.isModalOpen = true;
    }

    handleRowAction(event) {
        const action = event.detail.action;
        const row = event.detail.row;

        switch (action.name) {
            case 'edit':
                this.editRecordId = row.Id;
                this.isModalOpen = true;
                break;
            case 'toggle':
                this.toggleActive(row.Id, !row.Is_Active__c);
                break;
        }
    }

    async toggleActive(recordId, newValue) {
        // Imperative call to update Is_Active__c
        // Member 2 cần thêm method toggleQuestionActive vào Controller
        try {
            // await toggleQuestionActive({questionId: recordId, isActive: newValue});
            await refreshApex(this.wiredQuestionsResult);
            this.showToast('Success', 'Question updated', 'success');
        } catch (error) {
            this.showToast('Error', error.body?.message, 'error');
        }
    }

    handleModalClose() {
        this.isModalOpen = false;
    }

    handleModalSave() {
        this.isModalOpen = false;
        refreshApex(this.wiredQuestionsResult);
    }

    showToast(title, message, variant) {
        this.dispatchEvent(new ShowToastEvent({ title, message, variant }));
    }
}
```

**`adminQuestionManage.js-meta.xml`:**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<LightningComponentBundle xmlns="http://soap.sforce.com/2006/04/metadata">
    <apiVersion>59.0</apiVersion>
    <isExposed>true</isExposed>
    <targets>
        <target>lightning__AppPage</target>
    </targets>
</LightningComponentBundle>
```

---

### Task 4.9: `newQuestionModal` (20 phút)

**`newQuestionModal.html`:**

```html
<template>
    <section role="dialog" class="slds-modal slds-fade-in-open">
        <div class="slds-modal__container">
            <header class="slds-modal__header">
                <button class="slds-button slds-button_icon slds-modal__close" 
                        onclick={handleClose}>
                    <lightning-icon icon-name="utility:close" size="small">
                    </lightning-icon>
                </button>
                <h2 class="slds-modal__title">
                    {modalTitle}
                </h2>
            </header>
            <div class="slds-modal__content slds-p-around_medium">
                <lightning-record-edit-form
                    object-api-name="Question__c"
                    record-id={recordId}
                    onsuccess={handleSuccess}
                    onerror={handleError}
                >
                    <lightning-messages></lightning-messages>
                    
                    <lightning-input-field 
                        field-name="Certification__c">
                    </lightning-input-field>
                    <lightning-input-field 
                        field-name="Question_Text__c">
                    </lightning-input-field>
                    <lightning-input-field 
                        field-name="Option_A__c">
                    </lightning-input-field>
                    <lightning-input-field 
                        field-name="Option_B__c">
                    </lightning-input-field>
                    <lightning-input-field 
                        field-name="Option_C__c">
                    </lightning-input-field>
                    <lightning-input-field 
                        field-name="Option_D__c">
                    </lightning-input-field>
                    <lightning-input-field 
                        field-name="Correct_Answer__c">
                    </lightning-input-field>
                    <lightning-input-field 
                        field-name="Is_Active__c">
                    </lightning-input-field>

                    <div class="slds-m-top_medium">
                        <lightning-button 
                            variant="brand" 
                            type="submit" 
                            label="Save">
                        </lightning-button>
                        <lightning-button 
                            label="Cancel" 
                            onclick={handleClose} 
                            class="slds-m-left_x-small">
                        </lightning-button>
                    </div>
                </lightning-record-edit-form>
            </div>
        </div>
    </section>
    <div class="slds-backdrop slds-backdrop_open"></div>
</template>
```

**`newQuestionModal.js`:**

```javascript
import { LightningElement, api } from 'lwc';
import { ShowToastEvent } from 'lightning/platformShowToastEvent';

export default class NewQuestionModal extends LightningElement {
    @api recordId; // null = new, non-null = edit

    get modalTitle() {
        return this.recordId ? 'Edit Question' : 'New Question';
    }

    handleSuccess() {
        this.showToast('Success', 'Question saved successfully', 'success');
        this.dispatchEvent(new CustomEvent('save'));
    }

    handleError(event) {
        this.showToast('Error', event.detail.message, 'error');
    }

    handleClose() {
        this.dispatchEvent(new CustomEvent('close'));
    }

    showToast(title, message, variant) {
        this.dispatchEvent(new ShowToastEvent({ title, message, variant }));
    }
}
```

> 💡 **PD1 Knowledge:**
> - `lightning-record-edit-form`: Standard form, tự handle validation rules
> - `@api` decorator: cho phép parent truyền data xuống
> - `CustomEvent`: Child → Parent communication pattern
> - `lightning-messages`: Auto hiển thị server-side errors (bao gồm Validation Rules)

---

## 🟢 TUẦN 3 — Polish & Bug Fixes (1.2h)

### Task 4.10: Activate/Deactivate Button (15 phút)

**Yêu cầu từ Member 2:** Thêm method vào `BootcampController`:

```apex
    @AuraEnabled
    public static void toggleQuestionActive(Id questionId, Boolean isActive) {
        try {
            Question__c q = [
                SELECT Id, Is_Active__c FROM Question__c 
                WHERE Id = :questionId LIMIT 1
            ];
            q.Is_Active__c = isActive;
            update q;
        } catch (Exception e) {
            throw new AuraHandledException(e.getMessage());
        }
    }
```

**Cập nhật `adminQuestionManage.js`** — uncomment và import method:

```javascript
import toggleQuestionActive from '@salesforce/apex/BootcampController.toggleQuestionActive';

// Trong toggleActive():
await toggleQuestionActive({ questionId: recordId, isActive: newValue });
```

---

### Task 4.11: SLDS Polish (20 phút)

**Checklist SLDS compliance:**

| # | Item | Check |
|:---|:---|:---|
| 1 | Tất cả spacing dùng `slds-p-` / `slds-m-` tokens | ⬜ |
| 2 | Cards dùng `lightning-card` | ⬜ |
| 3 | Buttons dùng SLDS variants: `brand`, `neutral`, `destructive` | ⬜ |
| 4 | Icons dùng `standard:*` hoặc `utility:*` | ⬜ |
| 5 | Tables dùng `lightning-datatable` hoặc `slds-table` | ⬜ |
| 6 | Badges dùng `slds-badge slds-theme_*` | ⬜ |
| 7 | Modals follow SLDS modal blueprint | ⬜ |
| 8 | Không custom CSS phá UX | ⬜ |

---

### Task 4.12: Next Action Functional Button (15 phút)

Cập nhật `dashboardView.html` — Next Action section:

```html
        <!-- Next Action with functional button -->
        <div class="slds-p-around_medium slds-box slds-theme_shade">
            <p class="slds-text-body_regular slds-m-bottom_small">
                <lightning-icon icon-name="utility:forward" size="x-small" 
                                class="slds-m-right_x-small"></lightning-icon>
                <strong>Next Action:</strong> {stats.nextAction}
            </p>
            <template if:true={showExamButton}>
                <lightning-button 
                    variant="brand" 
                    label={stats.nextAction}
                    onclick={navigateToExam}
                ></lightning-button>
            </template>
        </div>
```

JS getter:

```javascript
    get showExamButton() {
        return this.stats.nextAction && 
               this.stats.nextAction !== 'Ready for Certification';
    }

    navigateToExam() {
        // Navigate to Mock Exam tab
        // Hoặc dùng NavigationMixin
    }
```

---

### Task 4.13: Bug Fixes (15 phút)

Nhận bugs từ Member 5 → Fix display issues, data binding errors.

---

## ✅ Deliverables tổng kết Member 4

| # | Deliverable | Status |
|:---|:---|:---|
| 1 | `mockExamFlow` — Complete (Select → Exam → Submit → Result Modal) | ⬜ |
| 2 | `dashboardView` — Stats + Leaderboard + Recent Attempts + Next Action | ⬜ |
| 3 | `adminQuestionManage` — Datatable + Row Actions + Toggle Active | ⬜ |
| 4 | `newQuestionModal` — Create/Edit form + Validation | ⬜ |
| 5 | SLDS compliant — no custom CSS | ⬜ |
| 6 | All meta.xml exposed for Lightning App Pages | ⬜ |
