# ตอนที่ 99: Career Development สำหรับ Pascal Developer

## บทนำ: เส้นทางอาชีพนักพัฒนา Pascal

Pascal developers มีตลาดงานเฉพาะทาง โดยเฉพาะในองค์กรที่ใช้ Delphi legacy systems

## 1. Job Market Overview

```
ตลาดงาน Pascal/Delphi ในไทยและสากล:

สายงานที่ต้องการ Pascal/Delphi:
├── Healthcare IT (โรงพยาบาล, คลินิก)
│   ├── HIS (Hospital Information System)
│   ├── LIS (Laboratory Information System)
│   └── Billing Systems
├── Manufacturing/ERP
│   ├── Production Management
│   ├── Inventory Systems
│   └── Quality Control
├── Financial/Banking
│   ├── Core Banking Systems
│   ├── Point of Sale
│   └── Back-office Systems
├── Government Systems
│   ├── Tax Systems (สรรพากร)
│   ├── Customs (ศุลกากร)
│   └── Social Security (ประกันสังคม)
└── Embedded/Industrial
    ├── SCADA Systems
    ├── Automation
    └── Machine Control

ช่วงเงินเดือน (ไทย, ปี 2024):
├── Junior (0-2 ปี):    25,000 - 40,000 บาท
├── Mid (3-5 ปี):       45,000 - 70,000 บาท
├── Senior (5-10 ปี):   75,000 - 120,000 บาท
└── Lead/Architect:    120,000 - 200,000+ บาท

Remote/International:
├── Delphi Developer (US): $80,000 - $150,000/year
├── Pascal Embedded (EU): €60,000 - €100,000/year
└── Consultant Rate: $80 - $150/hour
```

## 2. Technical Skills Portfolio

```pascal
// portfolio/src/PortfolioProject.pas
// ตัวอย่าง Portfolio Project ที่น่าประทับใจ

// Skills ที่ควรมีในปี 2024:

(*
CORE PASCAL/DELPHI:
  ✓ Object Pascal syntax, generics
  ✓ Lazarus IDE, FPC compiler
  ✓ Component development (VCL/LCL)
  ✓ Multi-threading, synchronization
  ✓ Memory management
  ✓ Exception handling

DATABASE:
  ✓ SQL (PostgreSQL, MySQL, SQLite, MSSQL)
  ✓ ORM patterns (Repository, Unit of Work)
  ✓ Connection pooling
  ✓ Database migration

ARCHITECTURE:
  ✓ SOLID principles
  ✓ Design patterns (Gang of Four)
  ✓ Clean Architecture
  ✓ Microservices basics
  ✓ REST API design

TESTING:
  ✓ Unit testing (FPCUnit, DUnit)
  ✓ Integration testing
  ✓ TDD/BDD concepts

TOOLING:
  ✓ Git, GitHub/GitLab
  ✓ Docker basics
  ✓ CI/CD (GitHub Actions)
  ✓ Linux command line

BONUS SKILLS:
  ✓ Second language (Go, Python, C#)
  ✓ Cloud basics (AWS/GCP/Azure)
  ✓ IoT/Embedded (ARM, AVR)
*)

// Portfolio Projects คะแนนสูง:
(*
1. Open Source Component ที่มี users จริง
2. REST API Backend ที่ production-ready
3. Migration tool (Delphi → Lazarus)
4. Performance benchmark (Pascal vs Go vs Python)
5. Real-time Dashboard (WebSocket, Pascal backend)
*)
```

## 3. Resume/CV Template

```
=====================================
SOMCHAI JAIDEE
Pascal/Delphi Developer
=====================================
📧 somchai@example.com
📱 +66 81-234-5678
💼 linkedin.com/in/somchai-jaidee
🐙 github.com/somchai-dev
📍 Bangkok, Thailand

SUMMARY
Senior Pascal/Lazarus developer with 8 years experience
building enterprise healthcare and financial systems.
Expertise in high-performance, multi-threaded applications.
Open source contributor to Lazarus IDE and castle-engine.

EXPERIENCE

Senior Developer | MedTech Thailand | 2021-Present
• Led migration of 200,000-line Delphi HIS to Lazarus
• Reduced startup time 60% through lazy-loading architecture
• Built REST API serving 1,000 concurrent users (fphttpserver)
• Mentored team of 5 junior developers
• Tech: Lazarus 3.x, PostgreSQL 16, Docker, Kubernetes

Developer | ABC Software | 2018-2021
• Developed Point-of-Sale system for 500+ retail locations
• Implemented offline-first sync with conflict resolution
• Integrated with fiscal printers (ESC/POS protocol)
• Tech: Delphi 11, Firebird, SQLite, REST

SKILLS
Languages:    Pascal/Delphi ●●●●●, Go ●●●●○,
              Python ●●●●○, SQL ●●●●●
Frameworks:   Lazarus LCL, VCL, mORMot2, fphttpserver
Databases:    PostgreSQL, MySQL, SQLite, Firebird, MSSQL
Patterns:     Clean Arch, CQRS, Event Sourcing, DDD
Tools:        Git, Docker, GitHub Actions, Kubernetes
OS:           Windows, Linux (Ubuntu/Debian)

EDUCATION
B.Sc. Computer Science | Kasetsart University | 2014-2018

CERTIFICATIONS
• Delphi Developer Certification (Embarcadero)
• AWS Solutions Architect Associate

OPEN SOURCE
• TStarRating - Lazarus component (500+ GitHub stars)
• pascal-json-schema - JSON Schema validator
• Contributor: Lazarus IDE, fpweb
```

## 4. Interview Preparation

```pascal
// interview/TechnicalQuestions.pas
// คำถาม Technical Interview Pascal Developer

(*
Q1: อธิบาย difference ระหว่าง TObject.Free และ FreeAndNil
*)

// Answer:
procedure Example;
var
  Obj: TMyObject;
begin
  Obj := TMyObject.Create;

  // Free: เรียก destructor และ deallocate memory
  // แต่ pointer ยัง point ไปที่ freed memory (dangling pointer!)
  Obj.Free;
  // Obj ยังเป็น non-nil แต่ memory ถูก free แล้ว
  // การเรียก Obj.SomeMethod หลัง Free = undefined behavior

  // FreeAndNil: Free + Set to nil (ปลอดภัยกว่า)
  Obj := TMyObject.Create;
  FreeAndNil(Obj);
  // Obj = nil แล้ว - การ check Assigned(Obj) จะ return False
  // if Assigned(Obj) then Obj.SomeMethod; // ปลอดภัย
end;

(*
Q2: อธิบาย TStringList.Sorted และ Duplicates
*)

procedure StringListDemo;
var
  SL: TStringList;
begin
  SL := TStringList.Create;
  try
    SL.Sorted := True;          // Sort ทุกครั้งที่ add
    SL.Duplicates := dupIgnore; // ไม่รับ duplicates
    // หรือ dupError (raise exception)
    // หรือ dupAccept (ยอมรับ duplicates)

    SL.Add('Cherry');
    SL.Add('Apple');  // Will be sorted to position 0
    SL.Add('Banana');
    SL.Add('Apple');  // Ignored (dupIgnore)
    // Result: Apple, Banana, Cherry

    // Binary search บน sorted list
    var Idx := SL.IndexOf('Banana'); // Fast O(log n)
  finally
    SL.Free;
  end;
end;

(*
Q3: Multi-threading และ Thread Safety
*)

// BAD: Race condition
var
  GCounter: Integer = 0;

procedure IncrementUnsafe;
begin
  // Read-Modify-Write ไม่ atomic = race condition!
  Inc(GCounter);
end;

// GOOD: Thread-safe counter
var
  GCounterLock: TCriticalSection;
  GCounterAtomic: Integer = 0;

procedure IncrementSafe;
begin
  // Option 1: CriticalSection
  GCounterLock.Acquire;
  try
    Inc(GCounterAtomic);
  finally
    GCounterLock.Release;
  end;

  // Option 2: Atomic (for simple integers)
  InterlockedIncrement(GCounterAtomic);
end;

(*
Q4: Explain Object Pascal Generics
*)

type
  // Generic Stack
  TStack<T> = class
  private
    FItems: TArray<T>;
    FCount: Integer;
  public
    procedure Push(const AItem: T);
    function Pop: T;
    function Peek: T;
    function IsEmpty: Boolean;
    property Count: Integer read FCount;
  end;

procedure TStack<T>.Push(const AItem: T);
begin
  if FCount >= Length(FItems) then
    SetLength(FItems, Max(4, Length(FItems) * 2));
  FItems[FCount] := AItem;
  Inc(FCount);
end;

function TStack<T>.Pop: T;
begin
  if FCount = 0 then
    raise Exception.Create('Stack is empty');
  Dec(FCount);
  Result := FItems[FCount];
end;

(*
Q5: Memory management patterns
*)

// Pattern 1: try..finally (always)
procedure SafeResource;
var
  Obj: TMyObject;
begin
  Obj := TMyObject.Create;
  try
    Obj.DoWork;
  finally
    Obj.Free; // Always runs
  end;
end;

// Pattern 2: Owner-based (component model)
procedure OwnerPattern(AOwner: TComponent);
begin
  // TButton owned by AOwner - freed when owner freed
  var Btn := TButton.Create(AOwner);
  Btn.Caption := 'Click me';
  // No need to free - owner handles it
end;

// Pattern 3: Interface reference counting
procedure InterfacePattern;
var
  Svc: IMyService;
begin
  Svc := TMyService.Create; // Ref count = 1
  Svc.DoWork;
  // Svc goes out of scope → ref count = 0 → freed automatically
end;
```

## 5. Freelancing กับ Pascal

```
การ Freelance Pascal Developer:

PLATFORMS:
• Upwork - Delphi/Pascal projects
• Freelancer.com - Legacy Delphi migration
• Toptal - Senior developers (high pay)
• LinkedIn - Enterprise contacts
• Local: Techsauce, Blognone Jobs

COMMON FREELANCE PROJECTS:
1. Delphi → Lazarus Migration
   Rate: $50-100/hour
   Duration: 1-6 months

2. Legacy System Maintenance
   Rate: $40-80/hour
   Good for long-term income

3. New Pascal/Lazarus Development
   Rate: $60-120/hour
   Healthcare, Finance, Industrial

4. Code Review & Consulting
   Rate: $100-200/hour
   Senior expertise

PROPOSAL TEMPLATE:
"I have X years of experience with Pascal/Delphi,
specializing in [healthcare/finance/embedded] systems.
I've successfully:
- Migrated [N] legacy Delphi apps to Lazarus
- Built systems handling [scale] transactions/users
- Contributed to open source: [links]

For this project, I would:
1. [Specific approach]
2. [Timeline with milestones]
3. [Deliverables]

References available upon request."
```

## 6. Certifications และ Learning Path

```
LEARNING ROADMAP:

BEGINNER (0-6 months):
  ✓ Pascal syntax fundamentals
  ✓ Lazarus IDE
  ✓ Basic OOP
  ✓ File I/O, strings
  ✓ Simple forms/UI

INTERMEDIATE (6-18 months):
  ✓ Database (SQLite, PostgreSQL)
  ✓ REST APIs (fphttpserver)
  ✓ Multi-threading
  ✓ Design patterns
  ✓ Git, basic DevOps

ADVANCED (18-36 months):
  ✓ Architecture patterns (Clean, CQRS)
  ✓ Performance optimization
  ✓ Embedded/IoT
  ✓ Open source contribution
  ✓ Team leadership

EXPERT (36+ months):
  ✓ System design at scale
  ✓ Mentoring
  ✓ Speaking at conferences
  ✓ Technical writing/blogging
  ✓ Product development

CERTIFICATIONS (Optional):
• Delphi Developer Certification (Embarcadero)
• AWS/GCP/Azure (Cloud skills)
• Professional Scrum Developer
• Oracle Java SE (cross-skill)

COMMUNITIES:
• Free Pascal mailing list
• Lazarus Forum (forum.lazarus.freepascal.org)
• Pascal Thailand Facebook Group
• Stack Overflow [pascal] tag
• Delphi-PRAXiS (German, great quality)
```

## 7. Salary Negotiation

```
เทคนิค Salary Negotiation:

1. KNOW YOUR VALUE
   ค้นหา benchmark:
   - glassdoor.com → Delphi Developer Thailand
   - levels.fyi (international comparison)
   - ถามใน community/forum
   - LinkedIn Jobs salaries

2. PREPARE YOUR CASE
   Portfolio evidence:
   - "ฉัน migrate legacy system ที่มี X lines of code"
   - "ระบบที่ฉันสร้างรองรับ Y users/day"
   - "ลด bug rate Z% ด้วย testing strategy"

3. NEGOTIATION SCRIPT:
   "จากประสบการณ์ X ปีใน Pascal/Delphi และ
   ผลงาน [concrete examples], ฉันคาดว่า compensation
   ที่เหมาะสมอยู่ที่ [range]. ฉันยืดหยุ่นได้ถ้า
   มี benefits อื่นเช่น [WFH/training/equity]"

4. TOTAL COMPENSATION:
   ไม่ใช่แค่ base salary:
   + Annual bonus (1-4 months)
   + Health insurance
   + Training budget
   + WFH flexibility
   + Equipment allowance
   + Stock options (startups)

5. COUNTER-OFFER:
   เมื่อได้ offer แล้ว:
   - Take time (อย่ารีบตอบ)
   - "ขอบคุณ ขอเวลา 2-3 วันพิจารณา"
   - Counter ด้วย 10-20% สูงกว่า offer
   - เตรียม walk away จริงๆ
```

## 8. สรุปเส้นทางอาชีพ

**Key Takeaways:**
1. Pascal/Delphi มีตลาดเฉพาะทางที่ยังต้องการ
2. Legacy migration เป็น skill ที่มีมูลค่าสูง
3. เพิ่ม value ด้วย cross-skill (Go, Python, Cloud)
4. Open source contribution สร้าง credibility
5. Community participation สร้าง network

**การเติบโตในสาย:**
```
Junior Dev → Mid Dev → Senior Dev → Tech Lead
                                  ↓
                            Staff Engineer
                            Principal Engineer
                            Engineering Manager
                            CTO
```

**Pascal Developer ที่ประสบความสำเร็จ:**
- Focus ที่ quality over quantity
- มี domain expertise (Healthcare, Finance, etc.)
- Keep learning new patterns/architectures
- Contribute back to community
