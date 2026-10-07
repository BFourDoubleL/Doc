
# Design Document: MFU-Research Management System

## 1. System Architecture Overview
ระบบถูกออกแบบด้วยสถาปัตยกรรมแบบ Modular UI-Centric Architecture เพื่อรองรับการทำงานของ Clickable Prototype และจำลอง Logic การทำงานของระบบ MFU-Research:

* **Presentation Layer (UI Component)**: ส่วนต่อประสานผู้ใช้สำหรับนักวิจัย (Researchers), กรรมการประเมิน (Reviewers), และเจ้าหน้าที่การเงิน (Finance Admins)
* **Business Logic Layer (Form & Workflow Manager)**: ระบบประมวลผลเงื่อนไขทางธุรกิจ เช่น กฎการล็อกแบบฟอร์ม (Form-Level Locking Rules) ตามสถานะการรีวิว และการจัดสรรงาน (Admin Reassign)
* **Data Mocking Layer (In-Memory Data Store)**: โครงสร้างข้อมูลจำลองสำหรับเก็บข้อมูลข้อเสนอโครงการ (Proposals) และคำขอเบิกจ่าย (Finance Claims)

---

## 2. UML Diagrams

### 2.1 Use Case Diagram
แสดงขอบเขตการทำงานของระบบ MFU-Research ที่ครอบคลุม Must-Have Functional Requirements (FR-1 ถึง FR-6)

```mermaid
graph TD
    actorResearcher(("Researcher\n(นักวิจัย/นิสิต)"))
    actorReviewer(("School Reviewer\n(กรรมการประเมิน)"))
    actorFinance(("Finance Admin\n(เจ้าหน้าที่การเงิน)"))

    subgraph "MFU-Research System Boundary"
        UC01("FR-1: ยื่นข้อเสนอโครงการวิจัย")
        UC02("FR-2: ตรวจทานแบบ 2-Track & School Endorsement")
        UC03("FR-3: ควบคุมการล็อกแบบฟอร์ม (Form Locking)")
        UC04("FR-4: ติดตามสถานะโครงการแบบ Real-time")
        UC05("FR-5: ยื่นคำขอเบิกจ่ายงบประมาณ (Finance Claim)")
        UC06("FR-6: จัดสรรรายการเบิกจ่าย (Admin Reassign)")
    end

    actorResearcher --> UC01
    actorResearcher --> UC04
    actorResearcher --> UC05

    actorReviewer --> UC02
    actorReviewer --> UC03

    actorFinance --> UC04
    actorFinance --> UC05
    actorFinance --> UC06

```

---

### 2.2 Sequence Diagrams

#### Sequence Diagram 01: Happy Path (การยื่นแบบฟอร์มและการล็อกฟอร์มระหว่างรีวิว)

กระบวนการยื่นข้อเสนอโครงการวิจัย (FR-1) และการเข้าสู่กระบวนการตรวจทาน 2-Track Review พร้อมสั่งเปิดใช้งาน Form Locking (FR-2, FR-3)

```mermaid
sequenceDiagram
    autonumber
    actor Researcher as Researcher
    participant UI as MFU-Research UI
    participant Logic as Workflow & Locking Engine
    participant Store as Research Data Store

    Researcher->>UI: 1. กรอกข้อมูลแบบฟอร์มข้อเสนอโครงการ และกด Submit
    UI->>Logic: 2. ส่งข้อมูลโครงการเพื่อตรวจสอบความถูกต้อง
    Logic->>Logic: 3. ตรวจสอบข้อมูลแบบฟอร์ม (Validation Passed)
    Logic->>Store: 4. บันทึกข้อมูล proposal_status = 'IN_REVIEW', is_locked = true
    Store-->>Logic: 5. ยืนยันการบันทึกสำเร็จ
    Logic-->>UI: 6. ส่งคืนสถานะการยื่นคำขอสำเร็จ
    UI-->>Researcher: 7. แสดงข้อความสำเร็จ และเปลี่ยนสถานะฟอร์มเป็น Read-Only

```

#### Sequence Diagram 02: Unhappy Path (การพยายามแก้ไขฟอร์มขณะถูกล็อก / ข้อผิดพลาดคำขอเบิกจ่าย)

กระบวนการจัดการข้อผิดพลาดเมื่อผู้ใช้พยายามแก้ไขข้อมูลระหว่างที่ระบบล็อกแบบฟอร์ม (FR-3) หรือกรอกข้อมูลเบิกจ่ายไม่ครบถ้วน (FR-5)

```mermaid
sequenceDiagram
    autonumber
    actor User as User / Researcher
    participant UI as MFU-Research UI
    participant Logic as Validation & Locking Engine
    participant Store as Research Data Store

    alt Case 1: Form Locking Violation (FR-3)
        User->>UI: 1. พยายามกดแก้ไขข้อมูลในแบบฟอร์มที่อยู่ระหว่าง Review
        UI->>Logic: 2. ตรวจสอบสิทธิ์และสถานะการล็อก (Check is_locked)
        Logic-->>UI: 3. ปฏิเสธคำขอ (Error: Form is locked during 2-Track Review)
        UI-->>User: 4. แสดง Inline Alert 'ไม่สามารถแก้ไขได้ เนื่องจากอยู่ในขั้นตอนการพิจารณา'
    else Case 2: Finance Claim Validation Error (FR-5)
        User->>UI: 1. ส่งคำขอเบิกจ่ายงบประมาณโดยระบุจำนวนเงินเกินวงเงินอนุมัติ
        UI->>Logic: 2. ตรวจสอบเงื่อนไขงบประมาณ (Validate Claim Amount)
        Logic-->>UI: 3. ตรวจพบข้อผิดพลาด (Error: Claim exceeds approved budget)
        UI-->>User: 4. แสดง Error Notification แจ้งเตือนยอดเงินไม่ถูกต้อง
    end

```

---

## 3. Data Model & Database Schema

โครงสร้างจัดเก็บข้อมูลหลักเพื่อรองรับกระบวนการวิจัยและการเงิน

### Table Specifications

#### Table 1: `users`

| Attribute | Data Type | Key | Constraint | Description |
| --- | --- | --- | --- | --- |
| `user_id` | VARCHAR(36) | PK | NOT NULL, UNIQUE | รหัสประจำตัวผู้ใช้ |
| `full_name` | VARCHAR(100) | - | NOT NULL | ชื่อ-นามสกุล |
| `role` | VARCHAR(20) | - | CHECK (role IN ('RESEARCHER', 'REVIEWER', 'FINANCE_ADMIN')) | บทบาทผู้ใช้งาน |
| `school` | VARCHAR(100) | - | NOT NULL | สำนักวิชาที่สังกัด |

#### Table 2: `proposals` (FR-1, FR-2, FR-3, FR-4)

| Attribute | Data Type | Key | Constraint | Description |
| --- | --- | --- | --- | --- |
| `proposal_id` | VARCHAR(36) | PK | NOT NULL, UNIQUE | รหัสข้อเสนอโครงการ |
| `researcher_id` | VARCHAR(36) | FK | REFERENCES users(user_id) | รหัสผู้ยื่นโครงการ |
| `title` | VARCHAR(200) | - | NOT NULL | ชื่อโครงการวิจัย |
| `status` | VARCHAR(30) | - | DEFAULT 'DRAFT' | สถานะ (DRAFT, IN_REVIEW, APPROVED, REJECTED) |
| `is_locked` | BOOLEAN | - | DEFAULT FALSE | สถานะการล็อกแบบฟอร์ม (FR-3) |
| `created_at` | TIMESTAMP | - | DEFAULT CURRENT_TIMESTAMP | เวลาที่สร้างรายการ |

#### Table 3: `finance_claims` (FR-5, FR-6)

| Attribute | Data Type | Key | Constraint | Description |
| --- | --- | --- | --- | --- |
| `claim_id` | VARCHAR(36) | PK | NOT NULL, UNIQUE | รหัสคำขอเบิกจ่าย |
| `proposal_id` | VARCHAR(36) | FK | REFERENCES proposals(proposal_id) | รหัสโครงการวิจัยที่อ้างอิง |
| `amount` | DECIMAL(10,2) | - | NOT NULL | จำนวนเงินที่ขอเบิก |
| `claim_status` | VARCHAR(30) | - | DEFAULT 'SUBMITTED' | สถานะการเบิกจ่าย |
| `assigned_admin_id` | VARCHAR(36) | FK | REFERENCES users(user_id) | เจ้าหน้าที่การเงินที่ได้รับมอบหมาย (FR-6) |

---

## 4. UI Screen Mapping & State Matrix

ตารางเชื่อมโยงการแสดงผลบนหน้าจอ UI เข้ากับ Requirement และแผนการจัดการ Error State

| UI Screen ID | Screen Name | Mapped FR | Target Component / Action | State Handled (Happy / Unhappy) |
| --- | --- | --- | --- | --- |
| `UI-01` | Proposal Submission Form | FR-1, FR-3 | ปุ่ม "Submit Proposal" | **Happy**: บันทึกสำเร็จ -> ล็อกแบบฟอร์ม -> ไปหน้า `UI-03`<br>

<br>**Unhappy**: หากอยู่ในช่วง Review จะปิดการแก้ไข (Disabled Form Fields) |
| `UI-02` | 2-Track Review Dashboard | FR-2 | ปุ่ม "Approve / Reject / Endorse" | **Happy**: อัปเดตสถานะโครงการสำเร็จ<br>

<br>**Unhappy**: แสดงข้อความแจ้งเตือนเมื่อลงความเห็นไม่ครบตาม Track |
| `UI-03` | Project Status Tracker | FR-4 | Timeline Indicator | **Happy**: แสดงสถานะ Real-time พร้อม Progress Bar<br>

<br>**Unhappy**: แสดง Empty State หากไม่พบบันทึกโครงการ |
| `UI-04` | Finance Claim Panel | FR-5, FR-6 | ปุ่ม "Submit Claim" / "Reassign Admin" | **Happy**: สร้างคำขอเบิกจ่ายและแจกจ่ายงานให้ Admin สำเร็จ<br>

<br>**Unhappy**: แสดง Error Alert เมื่อยอดเงินเกินวงเงิน หรือไม่เลือก Admin |

```

```
