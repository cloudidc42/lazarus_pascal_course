# Part 21 - OOP พื้นฐาน (Object-Oriented Programming Basics)

## บทนำ

Object-Oriented Programming (OOP) หรือการเขียนโปรแกรมเชิงวัตถุ คือกระบวนทัศน์การเขียนโปรแกรม (Programming Paradigm) ที่จัดระเบียบโค้ดรอบๆ "วัตถุ" (Objects) แทนที่จะเป็นฟังก์ชันและ logic เป็นหลัก OOP ช่วยให้โค้ดมีโครงสร้างที่ชัดเจน ดูแลรักษาง่าย และนำกลับมาใช้ใหม่ได้

ในบทนี้เราจะเรียนรู้หลักการ OOP พื้นฐานในภาษา Pascal/Free Pascal และ Lazarus ตั้งแต่ต้นจนสามารถสร้าง class ได้เองอย่างมั่นใจ

---

## 21.1 ประวัติความเป็นมาของ OOP ใน Pascal

### Pascal ดั้งเดิม (1970s-1980s)
Pascal ถูกออกแบบโดย Niklaus Wirth ในปี 1970 เป็นภาษา procedural ล้วนๆ ไม่มี OOP

### Object Pascal (1986)
Apple และ Niklaus Wirth ร่วมกันพัฒนา Object Pascal เพิ่ม class, inheritance, polymorphism เข้ามา
ใช้ใน MacApp framework บน Macintosh

### Turbo Pascal (1989)
Borland เพิ่ม OOP เข้า Turbo Pascal 5.5 ในรูปแบบ `object` keyword
ต่างจาก class ในภายหลังตรงที่ไม่มี reference semantics

### Delphi (1995)
Borland เปิดตัว Delphi พร้อม Object Pascal เต็มรูปแบบ
มี TObject เป็น base class, interface, exception handling
นี่คือต้นแบบของ Free Pascal ในปัจจุบัน

### Free Pascal และ Lazarus (2000s-ปัจจุบัน)
Free Pascal (FPC) เป็น compiler ที่ compatible กับ Delphi
Lazarus เป็น IDE ที่ใช้ FPC พร้อม visual component library (LCL)
รองรับ Object Pascal เต็มรูปแบบ

---

## 21.2 แนวคิดพื้นฐาน OOP

### Class คืออะไร?
Class คือ "แม่แบบ" หรือ "blueprint" สำหรับสร้างวัตถุ เหมือนแบบบ้านที่ใช้สร้างบ้านจริง

```pascal
// Class คือแม่แบบ
type
  TCar = class
    FBrand: string;
    FModel: string;
    FYear: Integer;
  end;
```

### Object คืออะไร?
Object คือ instance ที่สร้างจาก class เหมือนบ้านจริงที่สร้างจากแบบบ้าน

```pascal
var
  MyCar: TCar;
begin
  MyCar := TCar.Create;  // สร้าง object จาก class
  // MyCar คือ object
  MyCar.Free;
end;
```

### ความแตกต่างระหว่าง Class และ Object

| Class | Object |
|-------|--------|
| แม่แบบ/blueprint | instance จริง |
| ไม่ใช้ memory (มากนัก) | ใช้ memory ใน heap |
| กำหนด structure และ behavior | มี state จริงๆ |
| TCar | MyCar, YourCar, HerCar |

---

## 21.3 หลักการ 4 ข้อของ OOP

### 1. Encapsulation (การห่อหุ้ม)
ซ่อน implementation details ไว้ภายใน class เปิดเผยเฉพาะ interface ที่จำเป็น

```pascal
type
  TBankAccount = class
  private
    FBalance: Double;      // ซ่อนไว้ ไม่ให้เข้าถึงตรงๆ
    FAccountNumber: string;
  public
    procedure Deposit(Amount: Double);   // เปิดเผย interface
    procedure Withdraw(Amount: Double);  // เปิดเผย interface
    function GetBalance: Double;         // เปิดเผย interface
  end;

implementation

procedure TBankAccount.Deposit(Amount: Double);
begin
  if Amount > 0 then
    FBalance := FBalance + Amount;  // ควบคุมการเปลี่ยนแปลงภายใน
end;
```

### 2. Inheritance (การสืบทอด)
Class ลูกสามารถรับ properties และ methods จาก class แม่ได้

```pascal
type
  TAnimal = class
  public
    procedure Eat;
    procedure Sleep;
    procedure MakeSound; virtual;
  end;

  TDog = class(TAnimal)   // สืบทอดจาก TAnimal
  public
    procedure Fetch;       // เพิ่ม behavior ใหม่
    procedure MakeSound; override;  // override พฤติกรรมเดิม
  end;
```

### 3. Polymorphism (พหุสัณฐาน)
Object ชนิดเดียวกันสามารถแสดงพฤติกรรมที่แตกต่างกันได้ขึ้นอยู่กับ type จริง

```pascal
var
  Animals: array[0..2] of TAnimal;
begin
  Animals[0] := TDog.Create;
  Animals[1] := TCat.Create;
  Animals[2] := TBird.Create;

  for var Animal in Animals do
    Animal.MakeSound;  // แต่ละตัวส่งเสียงต่างกัน!
end;
```

### 4. Abstraction (การนามธรรม)
ซ่อนความซับซ้อนและแสดงเฉพาะสิ่งที่จำเป็น

```pascal
type
  TShape = class  // Abstract concept
  public
    function Area: Double; virtual; abstract;
    function Perimeter: Double; virtual; abstract;
    procedure Draw; virtual; abstract;
  end;
```

---

## 21.4 การสร้าง Class ใน Free Pascal

### โครงสร้าง Class พื้นฐาน

```pascal
type
  TMyClass = class
  private
    // fields ที่ซ่อนไว้
    FField1: Integer;
    FField2: string;
  protected
    // สิ่งที่ class ลูกเข้าถึงได้
    procedure HelperMethod;
  public
    // interface สาธารณะ
    constructor Create;
    destructor Destroy; override;
    procedure DoSomething;
    property Field1: Integer read FField1 write FField1;
  end;
```

### ตัวอย่างสมบูรณ์: Class แรกของเรา

```pascal
program FirstClass;

{$mode objfpc}{$H+}

uses
  SysUtils;

type
  // กำหนด class Person
  TPerson = class
  private
    FName: string;
    FAge: Integer;
    FEmail: string;
  public
    // Constructor
    constructor Create(const AName: string; AAge: Integer);
    // Destructor
    destructor Destroy; override;
    // Methods
    procedure Introduce;
    function IsAdult: Boolean;
    procedure Birthday;
    // Properties
    property Name: string read FName write FName;
    property Age: Integer read FAge write FAge;
    property Email: string read FEmail write FEmail;
  end;

// Implementation
constructor TPerson.Create(const AName: string; AAge: Integer);
begin
  inherited Create;  // เรียก constructor ของ TObject
  FName := AName;
  FAge := AAge;
  FEmail := '';
  WriteLn('สร้าง TPerson: ', FName);
end;

destructor TPerson.Destroy;
begin
  WriteLn('ลบ TPerson: ', FName);
  inherited Destroy;  // เรียก destructor ของ TObject
end;

procedure TPerson.Introduce;
begin
  WriteLn('สวัสดี ฉันชื่อ ', FName, ' อายุ ', FAge, ' ปี');
  if FEmail <> '' then
    WriteLn('Email: ', FEmail);
end;

function TPerson.IsAdult: Boolean;
begin
  Result := FAge >= 18;
end;

procedure TPerson.Birthday;
begin
  Inc(FAge);
  WriteLn(FName, ' มีอายุครบ ', FAge, ' ปีแล้ว!');
end;

// โปรแกรมหลัก
var
  Person1, Person2: TPerson;
begin
  WriteLn('=== ทดสอบ Class TPerson ===');
  WriteLn;

  // สร้าง objects
  Person1 := TPerson.Create('สมชาย', 25);
  Person2 := TPerson.Create('สมหญิง', 17);

  // ตั้งค่า property
  Person1.Email := 'somchai@example.com';

  // เรียก methods
  Person1.Introduce;
  WriteLn('เป็นผู้ใหญ่: ', Person1.IsAdult);
  WriteLn;

  Person2.Introduce;
  WriteLn('เป็นผู้ใหญ่: ', Person2.IsAdult);
  Person2.Birthday;
  WriteLn('เป็นผู้ใหญ่: ', Person2.IsAdult);
  WriteLn;

  // ต้องลบ objects เมื่อใช้งานเสร็จ
  Person1.Free;
  Person2.Free;

  WriteLn('กด Enter เพื่อออก...');
  ReadLn;
end.
```

---

## 21.5 Instance Variables และ Class Variables

### Instance Variables (Field)
แต่ละ object มี instance ของตัวเอง ค่าของแต่ละ object แยกจากกัน

```pascal
type
  TCounter = class
  private
    FCount: Integer;  // Instance variable - แต่ละ object มีค่าของตัวเอง
  public
    constructor Create;
    procedure Increment;
    property Count: Integer read FCount;
  end;

constructor TCounter.Create;
begin
  FCount := 0;  // เริ่มต้นที่ 0 สำหรับแต่ละ object
end;

procedure TCounter.Increment;
begin
  Inc(FCount);
end;

// ทดสอบ
var
  C1, C2: TCounter;
begin
  C1 := TCounter.Create;
  C2 := TCounter.Create;

  C1.Increment;
  C1.Increment;
  C2.Increment;

  WriteLn('C1 Count: ', C1.Count);  // 2
  WriteLn('C2 Count: ', C2.Count);  // 1
end;
```

### Class Variables
ใช้ร่วมกันทุก instance ของ class นั้น

```pascal
type
  TObjectCounter = class
  private
    class var FTotalCount: Integer;  // Class variable - ใช้ร่วมกัน
    FId: Integer;
  public
    constructor Create;
    destructor Destroy; override;
    class function GetTotalCount: Integer;
    property Id: Integer read FId;
  end;

constructor TObjectCounter.Create;
begin
  inherited Create;
  Inc(FTotalCount);
  FId := FTotalCount;
  WriteLn('สร้าง object #', FId, ' รวมทั้งหมด: ', FTotalCount);
end;

destructor TObjectCounter.Destroy;
begin
  Dec(FTotalCount);
  WriteLn('ลบ object #', FId, ' เหลือ: ', FTotalCount);
  inherited;
end;

class function TObjectCounter.GetTotalCount: Integer;
begin
  Result := FTotalCount;
end;

// ทดสอบ
var
  Obj1, Obj2, Obj3: TObjectCounter;
begin
  WriteLn('จำนวน objects: ', TObjectCounter.GetTotalCount);  // 0

  Obj1 := TObjectCounter.Create;  // สร้าง object #1
  Obj2 := TObjectCounter.Create;  // สร้าง object #2
  Obj3 := TObjectCounter.Create;  // สร้าง object #3

  WriteLn('จำนวน objects: ', TObjectCounter.GetTotalCount);  // 3

  Obj2.Free;  // ลบ object #2
  WriteLn('จำนวน objects: ', TObjectCounter.GetTotalCount);  // 2

  Obj1.Free;
  Obj3.Free;
end;
```

---

## 21.6 Constructor และ Destructor

### Constructor
ฟังก์ชันพิเศษที่เรียกเมื่อสร้าง object จัดสรร memory และกำหนดค่าเริ่มต้น

```pascal
type
  TProduct = class
  private
    FName: string;
    FPrice: Double;
    FQuantity: Integer;
  public
    // Constructor หลายแบบ
    constructor Create; overload;
    constructor Create(const AName: string; APrice: Double); overload;
    constructor Create(const AName: string; APrice: Double; AQty: Integer); overload;
    destructor Destroy; override;

    procedure ShowInfo;
  end;

constructor TProduct.Create;
begin
  inherited Create;
  FName := 'ไม่ระบุ';
  FPrice := 0;
  FQuantity := 0;
end;

constructor TProduct.Create(const AName: string; APrice: Double);
begin
  inherited Create;
  FName := AName;
  FPrice := APrice;
  FQuantity := 0;
end;

constructor TProduct.Create(const AName: string; APrice: Double; AQty: Integer);
begin
  inherited Create;
  FName := AName;
  FPrice := APrice;
  FQuantity := AQty;
end;

destructor TProduct.Destroy;
begin
  WriteLn('กำลังลบสินค้า: ', FName);
  // ทำ cleanup ที่นี่ถ้าจำเป็น
  inherited Destroy;
end;

procedure TProduct.ShowInfo;
begin
  WriteLn('สินค้า: ', FName);
  WriteLn('ราคา: ', FPrice:0:2, ' บาท');
  WriteLn('จำนวน: ', FQuantity, ' ชิ้น');
  WriteLn('มูลค่ารวม: ', (FPrice * FQuantity):0:2, ' บาท');
  WriteLn('---');
end;

// ทดสอบ
var
  P1, P2, P3: TProduct;
begin
  P1 := TProduct.Create;
  P2 := TProduct.Create('แอปเปิ้ล', 15.50);
  P3 := TProduct.Create('มะม่วง', 25.00, 10);

  P1.ShowInfo;
  P2.ShowInfo;
  P3.ShowInfo;

  P1.Free;
  P2.Free;
  P3.Free;
end;
```

### Destructor
ฟังก์ชันพิเศษที่เรียกเมื่อลบ object ทำ cleanup resources

```pascal
type
  TFileProcessor = class
  private
    FFileName: string;
    FFileHandle: TextFile;
    FIsOpen: Boolean;
  public
    constructor Create(const AFileName: string);
    destructor Destroy; override;
    procedure WriteData(const Data: string);
  end;

constructor TFileProcessor.Create(const AFileName: string);
begin
  inherited Create;
  FFileName := AFileName;
  FIsOpen := False;

  try
    AssignFile(FFileHandle, FFileName);
    Rewrite(FFileHandle);
    FIsOpen := True;
    WriteLn('เปิดไฟล์: ', FFileName);
  except
    on E: Exception do
      WriteLn('Error เปิดไฟล์: ', E.Message);
  end;
end;

destructor TFileProcessor.Destroy;
begin
  if FIsOpen then
  begin
    CloseFile(FFileHandle);
    WriteLn('ปิดไฟล์: ', FFileName);
    FIsOpen := False;
  end;
  inherited Destroy;
end;

procedure TFileProcessor.WriteData(const Data: string);
begin
  if FIsOpen then
    WriteLn(FFileHandle, Data);
end;
```

---

## 21.7 Access Modifiers

### public
เข้าถึงได้จากทุกที่

```pascal
type
  TExample = class
  public
    PublicField: Integer;
    procedure PublicMethod;
  end;
```

### private
เข้าถึงได้เฉพาะภายใน class เดียวกัน (ใน unit เดียวกัน)

```pascal
type
  TExample = class
  private
    FPrivateField: Integer;  // เข้าถึงได้เฉพาะใน TExample
    procedure PrivateHelper;
  end;
```

### protected
เข้าถึงได้จาก class เดียวกันและ class ลูก

```pascal
type
  TBase = class
  protected
    FProtectedField: Integer;
    procedure ProtectedHelper;
  end;

  TChild = class(TBase)
  public
    procedure DoSomething;
  end;

procedure TChild.DoSomething;
begin
  FProtectedField := 10;  // OK - เข้าถึงได้จาก class ลูก
  ProtectedHelper;         // OK
end;
```

### published
เหมือน public แต่ข้อมูลจะถูก export ผ่าน RTTI (Run-Time Type Information) ใช้ใน Lazarus components

```pascal
type
  TMyComponent = class(TComponent)
  private
    FCaption: string;
  published
    property Caption: string read FCaption write FCaption;  // ปรากฏใน Object Inspector
  end;
```

### strict private
เข้าถึงได้เฉพาะใน class นั้นเท่านั้น แม้แต่ class ใน unit เดียวกันก็ไม่ได้

```pascal
type
  TSecure = class
  strict private
    FSecret: string;      // เข้าถึงได้เฉพาะ TSecure เท่านั้น
    procedure InternalOp;
  public
    function GetInfo: string;
  end;
```

### strict protected
เข้าถึงได้เฉพาะ class นั้นและ class ลูกโดยตรง ไม่รวม code ใน unit เดียวกัน

```pascal
type
  TBase = class
  strict protected
    procedure StrictProtectedOp;
  end;

  TChild = class(TBase)
  public
    procedure Test;
  end;

procedure TChild.Test;
begin
  StrictProtectedOp;  // OK - class ลูกเข้าถึงได้
end;
```

### ตัวอย่างเปรียบเทียบ Access Modifiers

```pascal
program AccessModifiers;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TBaseClass = class
  strict private
    FStrictPrivate: string;   // เฉพาะ TBaseClass
  private
    FPrivate: string;          // TBaseClass + code ใน unit นี้
  strict protected
    FStrictProtected: string; // TBaseClass + class ลูกโดยตรง
  protected
    FProtected: string;        // TBaseClass + class ลูก + unit นี้
  public
    FPublic: string;           // ทุกที่
  published
    FPublished: string;        // ทุกที่ + RTTI

  public
    constructor Create;
    procedure ShowAll;
    property StrictPrivateVal: string read FStrictPrivate write FStrictPrivate;
    property PrivateVal: string read FPrivate write FPrivate;
  end;

  TChildClass = class(TBaseClass)
  public
    procedure TestAccess;
  end;

constructor TBaseClass.Create;
begin
  inherited;
  FStrictPrivate := 'strict private';
  FPrivate := 'private';
  FStrictProtected := 'strict protected';
  FProtected := 'protected';
  FPublic := 'public';
  FPublished := 'published';
end;

procedure TBaseClass.ShowAll;
begin
  WriteLn('strict private: ', FStrictPrivate);
  WriteLn('private: ', FPrivate);
  WriteLn('strict protected: ', FStrictProtected);
  WriteLn('protected: ', FProtected);
  WriteLn('public: ', FPublic);
  WriteLn('published: ', FPublished);
end;

procedure TChildClass.TestAccess;
begin
  // FStrictPrivate := 'x';  // ERROR! ไม่สามารถเข้าถึง strict private
  // FPrivate := 'x';         // ERROR! ไม่สามารถเข้าถึง private จาก child
  FStrictProtected := 'x';   // OK - class ลูกเข้าถึงได้
  FProtected := 'x';          // OK
  FPublic := 'x';             // OK
  FPublished := 'x';          // OK
  WriteLn('Child สามารถเข้าถึง: strict protected, protected, public, published');
end;

var
  Base: TBaseClass;
  Child: TChildClass;
begin
  Base := TBaseClass.Create;
  Base.ShowAll;
  WriteLn;

  // เข้าถึงจาก unit เดียวกัน (แต่ไม่ใช่ class นั้น)
  // Base.FStrictPrivate  // ERROR!
  Base.FPrivate := 'แก้ไขได้จาก unit เดียวกัน';  // OK ใน unit เดียวกัน
  Base.FPublic := 'public';   // OK

  Child := TChildClass.Create;
  Child.TestAccess;

  Base.Free;
  Child.Free;

  ReadLn;
end.
```

---

## 21.8 Properties

Properties คือ interface สำหรับเข้าถึง fields ของ class สามารถมี getter และ setter เพื่อ validate หรือแปลงค่า

### Property พื้นฐาน

```pascal
type
  TTemperature = class
  private
    FCelsius: Double;
  public
    // Read-write property
    property Celsius: Double read FCelsius write FCelsius;

    // Computed property (read only) - แปลงเป็น Fahrenheit
    property Fahrenheit: Double read GetFahrenheit;

    // Computed property - แปลงเป็น Kelvin
    property Kelvin: Double read GetKelvin;
  end;

function TTemperature.GetFahrenheit: Double;
begin
  Result := FCelsius * 9/5 + 32;
end;

function TTemperature.GetKelvin: Double;
begin
  Result := FCelsius + 273.15;
end;
```

### Property พร้อม Getter/Setter

```pascal
type
  TAge = class
  private
    FValue: Integer;
    function GetValue: Integer;
    procedure SetValue(AValue: Integer);
  public
    property Value: Integer read GetValue write SetValue;
  end;

function TAge.GetValue: Integer;
begin
  Result := FValue;
end;

procedure TAge.SetValue(AValue: Integer);
begin
  if AValue < 0 then
    raise Exception.Create('อายุต้องไม่ติดลบ')
  else if AValue > 150 then
    raise Exception.Create('อายุไม่สมเหตุสมผล')
  else
    FValue := AValue;
end;
```

### ตัวอย่างสมบูรณ์ Properties

```pascal
program PropertiesExample;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TStudent = class
  private
    FName: string;
    FStudentID: string;
    FGrade: Double;
    FIsActive: Boolean;

    function GetName: string;
    procedure SetName(const AValue: string);
    function GetGrade: Double;
    procedure SetGrade(AValue: Double);
    function GetLetterGrade: string;
    function GetIsHonors: Boolean;

  public
    constructor Create(const AName, AID: string);

    property Name: string read GetName write SetName;
    property StudentID: string read FStudentID;  // Read-only หลัง constructor
    property Grade: Double read GetGrade write SetGrade;
    property LetterGrade: string read GetLetterGrade;  // Computed
    property IsHonors: Boolean read GetIsHonors;       // Computed
    property IsActive: Boolean read FIsActive write FIsActive;

    procedure ShowInfo;
  end;

constructor TStudent.Create(const AName, AID: string);
begin
  inherited Create;
  FName := AName;
  FStudentID := AID;
  FGrade := 0;
  FIsActive := True;
end;

function TStudent.GetName: string;
begin
  Result := FName;
end;

procedure TStudent.SetName(const AValue: string);
begin
  if Trim(AValue) = '' then
    raise Exception.Create('ชื่อนักเรียนต้องไม่ว่าง');
  FName := Trim(AValue);
end;

function TStudent.GetGrade: Double;
begin
  Result := FGrade;
end;

procedure TStudent.SetGrade(AValue: Double);
begin
  if (AValue < 0) or (AValue > 4.0) then
    raise Exception.CreateFmt('เกรด %f ไม่ถูกต้อง (ต้อง 0.0-4.0)', [AValue]);
  FGrade := AValue;
end;

function TStudent.GetLetterGrade: string;
begin
  if FGrade >= 3.5 then Result := 'A'
  else if FGrade >= 3.0 then Result := 'B+'
  else if FGrade >= 2.5 then Result := 'B'
  else if FGrade >= 2.0 then Result := 'C+'
  else if FGrade >= 1.5 then Result := 'C'
  else if FGrade >= 1.0 then Result := 'D+'
  else if FGrade >= 0.5 then Result := 'D'
  else Result := 'F';
end;

function TStudent.GetIsHonors: Boolean;
begin
  Result := FGrade >= 3.5;
end;

procedure TStudent.ShowInfo;
begin
  WriteLn('ชื่อ: ', FName);
  WriteLn('รหัส: ', FStudentID);
  WriteLn('เกรด: ', FGrade:0:2, ' (', LetterGrade, ')');
  if IsHonors then WriteLn('*** เกียรตินิยม ***');
  WriteLn('สถานะ: ', IfThen(FIsActive, 'ยังเรียนอยู่', 'จบแล้ว'));
  WriteLn;
end;

var
  S1, S2: TStudent;
begin
  S1 := TStudent.Create('สมชาย ใจดี', '6501001');
  S2 := TStudent.Create('สมหญิง สวยงาม', '6501002');

  S1.Grade := 3.75;
  S2.Grade := 2.80;

  S1.ShowInfo;
  S2.ShowInfo;

  // ทดสอบ validation
  try
    S1.Grade := 5.0;  // จะ raise exception
  except
    on E: Exception do
      WriteLn('Error: ', E.Message);
  end;

  try
    S2.Name := '';  // จะ raise exception
  except
    on E: Exception do
      WriteLn('Error: ', E.Message);
  end;

  S1.Free;
  S2.Free;

  ReadLn;
end.
```

---

## 21.9 Self Reference

`Self` คือการอ้างอิงถึง object ปัจจุบัน เหมือน `this` ในภาษา C++/Java

```pascal
type
  TBuilder = class
  private
    FName: string;
    FAge: Integer;
    FEmail: string;
  public
    function SetName(const AName: string): TBuilder;    // คืน Self
    function SetAge(AAge: Integer): TBuilder;           // คืน Self
    function SetEmail(const AEmail: string): TBuilder;  // คืน Self
    procedure Build;
  end;

function TBuilder.SetName(const AName: string): TBuilder;
begin
  FName := AName;
  Result := Self;  // คืน object ตัวเอง เพื่อทำ method chaining
end;

function TBuilder.SetAge(AAge: Integer): TBuilder;
begin
  FAge := AAge;
  Result := Self;
end;

function TBuilder.SetEmail(const AEmail: string): TBuilder;
begin
  FEmail := AEmail;
  Result := Self;
end;

procedure TBuilder.Build;
begin
  WriteLn('สร้างข้อมูล:');
  WriteLn('ชื่อ: ', FName);
  WriteLn('อายุ: ', FAge);
  WriteLn('Email: ', FEmail);
end;

// Method Chaining ด้วย Self
var
  Builder: TBuilder;
begin
  Builder := TBuilder.Create;
  Builder
    .SetName('สมชาย')
    .SetAge(25)
    .SetEmail('somchai@example.com')
    .Build;
  Builder.Free;
end;
```

---

## 21.10 Class Methods vs Instance Methods

### Instance Methods
เรียกใช้กับ instance ของ class มีการเข้าถึง `Self`

```pascal
type
  TCounter = class
  private
    FCount: Integer;
  public
    procedure Increment;        // Instance method
    function GetCount: Integer; // Instance method
  end;

// เรียกด้วย instance
var
  C: TCounter;
begin
  C := TCounter.Create;
  C.Increment;  // เรียกบน instance
  C.Free;
end;
```

### Class Methods
เรียกได้ทั้งจาก class name และ instance ไม่มี `Self` ที่เป็น instance

```pascal
type
  TMathHelper = class
  public
    class function Max(A, B: Integer): Integer;
    class function Min(A, B: Integer): Integer;
    class function Factorial(N: Integer): Int64;
  end;

class function TMathHelper.Max(A, B: Integer): Integer;
begin
  if A > B then Result := A else Result := B;
end;

class function TMathHelper.Min(A, B: Integer): Integer;
begin
  if A < B then Result := A else Result := B;
end;

class function TMathHelper.Factorial(N: Integer): Int64;
begin
  if N <= 1 then Result := 1
  else Result := N * Factorial(N - 1);
end;

// เรียกจาก class name (ไม่ต้องสร้าง instance)
begin
  WriteLn(TMathHelper.Max(10, 20));    // 20
  WriteLn(TMathHelper.Min(10, 20));    // 10
  WriteLn(TMathHelper.Factorial(10));  // 3628800
end;
```

### ตัวอย่างผสม

```pascal
type
  TSingleton = class
  private
    class var FInstance: TSingleton;
    FData: string;

    constructor Create;
  public
    destructor Destroy; override;

    // Class method - factory pattern
    class function GetInstance: TSingleton;
    class procedure ReleaseInstance;

    // Instance method
    procedure SetData(const AData: string);
    function GetData: string;
  end;

var
  SingletonInstance: TSingleton;  // Global instance

class function TSingleton.GetInstance: TSingleton;
begin
  if FInstance = nil then
    FInstance := TSingleton.Create;
  Result := FInstance;
end;

class procedure TSingleton.ReleaseInstance;
begin
  FreeAndNil(FInstance);
end;

constructor TSingleton.Create;
begin
  inherited;
  FData := '';
  WriteLn('Singleton สร้างครั้งแรก');
end;

destructor TSingleton.Destroy;
begin
  WriteLn('Singleton ถูกลบ');
  inherited;
end;

procedure TSingleton.SetData(const AData: string);
begin
  FData := AData;
end;

function TSingleton.GetData: string;
begin
  Result := FData;
end;

// ทดสอบ
begin
  TSingleton.GetInstance.SetData('ข้อมูลแรก');
  WriteLn(TSingleton.GetInstance.GetData);  // ข้อมูลแรก
  WriteLn(TSingleton.GetInstance.GetData);  // ข้อมูลแรก (instance เดิม)
  TSingleton.ReleaseInstance;
end;
```

---

## 21.11 โปรแกรมตัวอย่าง: Animal Class Hierarchy

```pascal
program AnimalHierarchy;

{$mode objfpc}{$H+}

uses SysUtils;

type
  // Base class
  TAnimal = class
  private
    FName: string;
    FAge: Integer;
    FWeight: Double;
  protected
    FSound: string;
    FLegs: Integer;
  public
    constructor Create(const AName: string; AAge: Integer; AWeight: Double);
    virtual; // ทำให้ constructor เป็น virtual สำหรับ class references

    procedure Eat; virtual;
    procedure Sleep;
    procedure MakeSound; virtual;
    procedure Move; virtual;
    procedure ShowInfo; virtual;

    property Name: string read FName;
    property Age: Integer read FAge write FAge;
    property Weight: Double read FWeight write FWeight;
    property Sound: string read FSound;
    property Legs: Integer read FLegs;
  end;

  // Dog class
  TDog = class(TAnimal)
  private
    FBreed: string;
    FIsVaccinated: Boolean;
  public
    constructor Create(const AName, ABreed: string; AAge: Integer; AWeight: Double);

    procedure MakeSound; override;
    procedure Move; override;
    procedure Fetch;
    procedure ShowInfo; override;

    property Breed: string read FBreed;
    property IsVaccinated: Boolean read FIsVaccinated write FIsVaccinated;
  end;

  // Cat class
  TCat = class(TAnimal)
  private
    FIsIndoor: Boolean;
    FColor: string;
  public
    constructor Create(const AName, AColor: string; AAge: Integer; AWeight: Double; AIsIndoor: Boolean);

    procedure MakeSound; override;
    procedure Move; override;
    procedure Purr;
    procedure ShowInfo; override;

    property IsIndoor: Boolean read FIsIndoor;
    property Color: string read FColor;
  end;

  // Bird class
  TBird = class(TAnimal)
  private
    FCanFly: Boolean;
    FWingSpan: Double;
  public
    constructor Create(const AName: string; AAge: Integer; AWeight, AWingSpan: Double; ACanFly: Boolean);

    procedure MakeSound; override;
    procedure Move; override;
    procedure Fly;
    procedure ShowInfo; override;
  end;

// ============= TAnimal Implementation =============
constructor TAnimal.Create(const AName: string; AAge: Integer; AWeight: Double);
begin
  inherited Create;
  FName := AName;
  FAge := AAge;
  FWeight := AWeight;
  FSound := '...';
  FLegs := 0;
end;

procedure TAnimal.Eat;
begin
  WriteLn(FName, ' กำลังกินอาหาร');
end;

procedure TAnimal.Sleep;
begin
  WriteLn(FName, ' กำลังนอนหลับ');
end;

procedure TAnimal.MakeSound;
begin
  WriteLn(FName, ' ส่งเสียง: ', FSound);
end;

procedure TAnimal.Move;
begin
  WriteLn(FName, ' กำลังเคลื่อนที่');
end;

procedure TAnimal.ShowInfo;
begin
  WriteLn('=== ข้อมูลสัตว์ ===');
  WriteLn('ชื่อ: ', FName);
  WriteLn('อายุ: ', FAge, ' ปี');
  WriteLn('น้ำหนัก: ', FWeight:0:1, ' กก.');
  WriteLn('จำนวนขา: ', FLegs, ' ขา');
  WriteLn('เสียงร้อง: ', FSound);
end;

// ============= TDog Implementation =============
constructor TDog.Create(const AName, ABreed: string; AAge: Integer; AWeight: Double);
begin
  inherited Create(AName, AAge, AWeight);
  FBreed := ABreed;
  FSound := 'โฮ่ง โฮ่ง!';
  FLegs := 4;
  FIsVaccinated := False;
end;

procedure TDog.MakeSound;
begin
  WriteLn(FName, ' เห่า: ', FSound);
end;

procedure TDog.Move;
begin
  WriteLn(FName, ' วิ่งด้วยขา 4 ข้าง');
end;

procedure TDog.Fetch;
begin
  WriteLn(FName, ' วิ่งไปเอาลูกบอลมาให้!');
end;

procedure TDog.ShowInfo;
begin
  inherited ShowInfo;
  WriteLn('สายพันธุ์: ', FBreed);
  WriteLn('ฉีดวัคซีน: ', IfThen(FIsVaccinated, 'แล้ว', 'ยังไม่ได้'));
end;

// ============= TCat Implementation =============
constructor TCat.Create(const AName, AColor: string; AAge: Integer; AWeight: Double; AIsIndoor: Boolean);
begin
  inherited Create(AName, AAge, AWeight);
  FColor := AColor;
  FIsIndoor := AIsIndoor;
  FSound := 'เมี๊ยว~';
  FLegs := 4;
end;

procedure TCat.MakeSound;
begin
  WriteLn(FName, ' ร้องว่า: ', FSound);
end;

procedure TCat.Move;
begin
  WriteLn(FName, ' เดินเบาๆ อย่างนุ่มนวล');
end;

procedure TCat.Purr;
begin
  WriteLn(FName, ' ครอกๆ ดีใจมาก~');
end;

procedure TCat.ShowInfo;
begin
  inherited ShowInfo;
  WriteLn('สี: ', FColor);
  WriteLn('เลี้ยงใน: ', IfThen(FIsIndoor, 'บ้าน', 'นอกบ้าน'));
end;

// ============= TBird Implementation =============
constructor TBird.Create(const AName: string; AAge: Integer; AWeight, AWingSpan: Double; ACanFly: Boolean);
begin
  inherited Create(AName, AAge, AWeight);
  FWingSpan := AWingSpan;
  FCanFly := ACanFly;
  FSound := 'จิ๊บ จิ๊บ';
  FLegs := 2;
end;

procedure TBird.MakeSound;
begin
  WriteLn(FName, ' ร้อง: ', FSound);
end;

procedure TBird.Move;
begin
  if FCanFly then
    WriteLn(FName, ' บินอยู่บนท้องฟ้า')
  else
    WriteLn(FName, ' เดินบนพื้น');
end;

procedure TBird.Fly;
begin
  if FCanFly then
    WriteLn(FName, ' กางปีก ', FWingSpan:0:1, ' ซม. บินขึ้น!')
  else
    WriteLn(FName, ' บินไม่ได้ :(');
end;

procedure TBird.ShowInfo;
begin
  inherited ShowInfo;
  WriteLn('ความกว้างปีก: ', FWingSpan:0:1, ' ซม.');
  WriteLn('บินได้: ', IfThen(FCanFly, 'ใช่', 'ไม่'));
end;

// ============= Main Program =============
var
  Animals: array of TAnimal;
  Dog1: TDog;
  Cat1: TCat;
  Bird1: TBird;
  I: Integer;
begin
  WriteLn('========== Animal Kingdom ==========');
  WriteLn;

  // สร้างสัตว์ต่างๆ
  Dog1 := TDog.Create('บุ๋ม', 'ไทยหลังอาน', 3, 5.5);
  Cat1 := TCat.Create('มีมี่', 'ส้ม', 2, 3.2, True);
  Bird1 := TBird.Create('ทวีต', 5, 0.15, 20, True);

  Dog1.IsVaccinated := True;

  // ใช้ polymorphism ผ่าน base class
  SetLength(Animals, 3);
  Animals[0] := Dog1;
  Animals[1] := Cat1;
  Animals[2] := Bird1;

  // แสดงข้อมูลทุกสัตว์
  for I := 0 to High(Animals) do
  begin
    Animals[I].ShowInfo;
    Animals[I].MakeSound;
    Animals[I].Move;
    WriteLn;
  end;

  // เรียก specific methods
  WriteLn('--- ความสามารถพิเศษ ---');
  Dog1.Fetch;
  Cat1.Purr;
  Bird1.Fly;

  // ล้าง memory
  Dog1.Free;
  Cat1.Free;
  Bird1.Free;

  ReadLn;
end.
```

---

## 21.12 โปรแกรมตัวอย่าง: Bank Account Class

```pascal
program BankAccountSystem;

{$mode objfpc}{$H+}

uses
  SysUtils, DateUtils;

type
  TTransactionType = (ttDeposit, ttWithdraw, ttTransfer);

  TTransaction = class
  private
    FDate: TDateTime;
    FType: TTransactionType;
    FAmount: Double;
    FDescription: string;
    FBalanceAfter: Double;
  public
    constructor Create(AType: TTransactionType; AAmount: Double;
                      const ADesc: string; ABalance: Double);
    procedure ShowInfo;
    property TransDate: TDateTime read FDate;
    property TransType: TTransactionType read FType;
    property Amount: Double read FAmount;
    property Description: string read FDescription;
    property BalanceAfter: Double read FBalanceAfter;
  end;

  TBankAccount = class
  private
    FAccountNumber: string;
    FOwnerName: string;
    FBalance: Double;
    FTransactions: array of TTransaction;
    FTransactionCount: Integer;
    FIsActive: Boolean;

    procedure AddTransaction(AType: TTransactionType; AAmount: Double;
                            const ADesc: string);
    function GenerateAccountNumber: string;
  public
    constructor Create(const AOwnerName: string; InitialDeposit: Double = 0);
    destructor Destroy; override;

    procedure Deposit(Amount: Double; const Description: string = '');
    function Withdraw(Amount: Double; const Description: string = ''): Boolean;
    function Transfer(ToAccount: TBankAccount; Amount: Double): Boolean;
    procedure ShowBalance;
    procedure ShowStatement;
    procedure CloseAccount;

    property AccountNumber: string read FAccountNumber;
    property OwnerName: string read FOwnerName;
    property Balance: Double read FBalance;
    property IsActive: Boolean read FIsActive;
  end;

// ============= TTransaction =============
constructor TTransaction.Create(AType: TTransactionType; AAmount: Double;
                               const ADesc: string; ABalance: Double);
begin
  inherited Create;
  FDate := Now;
  FType := AType;
  FAmount := AAmount;
  FDescription := ADesc;
  FBalanceAfter := ABalance;
end;

procedure TTransaction.ShowInfo;
const
  TypeNames: array[TTransactionType] of string = ('ฝากเงิน', 'ถอนเงิน', 'โอนเงิน');
begin
  WriteLn(FormatDateTime('dd/mm/yyyy hh:nn', FDate), '  ',
          TypeNames[FType], '  ',
          FAmount:10:2, '  ',
          'คงเหลือ: ', FBalanceAfter:10:2, '  ',
          FDescription);
end;

// ============= TBankAccount =============
var
  AccountCounter: Integer = 0;

function TBankAccount.GenerateAccountNumber: string;
begin
  Inc(AccountCounter);
  Result := Format('ACC%08d', [AccountCounter]);
end;

constructor TBankAccount.Create(const AOwnerName: string; InitialDeposit: Double);
begin
  inherited Create;
  FOwnerName := AOwnerName;
  FAccountNumber := GenerateAccountNumber;
  FBalance := 0;
  FIsActive := True;
  FTransactionCount := 0;
  SetLength(FTransactions, 100);

  WriteLn('เปิดบัญชีสำเร็จ: ', FAccountNumber, ' ชื่อ: ', FOwnerName);

  if InitialDeposit > 0 then
    Deposit(InitialDeposit, 'เงินฝากเริ่มต้น');
end;

destructor TBankAccount.Destroy;
var
  I: Integer;
begin
  for I := 0 to FTransactionCount - 1 do
    FTransactions[I].Free;
  inherited;
end;

procedure TBankAccount.AddTransaction(AType: TTransactionType; AAmount: Double;
                                      const ADesc: string);
begin
  if FTransactionCount >= Length(FTransactions) then
    SetLength(FTransactions, Length(FTransactions) + 100);

  FTransactions[FTransactionCount] := TTransaction.Create(AType, AAmount, ADesc, FBalance);
  Inc(FTransactionCount);
end;

procedure TBankAccount.Deposit(Amount: Double; const Description: string);
begin
  if not FIsActive then
  begin
    WriteLn('บัญชีปิดแล้ว ไม่สามารถฝากเงินได้');
    Exit;
  end;

  if Amount <= 0 then
  begin
    WriteLn('จำนวนเงินต้องมากกว่า 0');
    Exit;
  end;

  FBalance := FBalance + Amount;
  AddTransaction(ttDeposit, Amount,
    IfThen(Description = '', 'ฝากเงินสด', Description));
  WriteLn('ฝากเงินสำเร็จ: +', Amount:0:2, ' บาท  คงเหลือ: ', FBalance:0:2, ' บาท');
end;

function TBankAccount.Withdraw(Amount: Double; const Description: string): Boolean;
begin
  Result := False;

  if not FIsActive then
  begin
    WriteLn('บัญชีปิดแล้ว');
    Exit;
  end;

  if Amount <= 0 then
  begin
    WriteLn('จำนวนเงินต้องมากกว่า 0');
    Exit;
  end;

  if Amount > FBalance then
  begin
    WriteLn('ยอดเงินไม่เพียงพอ (ต้องการ: ', Amount:0:2, ' มี: ', FBalance:0:2, ')');
    Exit;
  end;

  FBalance := FBalance - Amount;
  AddTransaction(ttWithdraw, Amount,
    IfThen(Description = '', 'ถอนเงินสด', Description));
  WriteLn('ถอนเงินสำเร็จ: -', Amount:0:2, ' บาท  คงเหลือ: ', FBalance:0:2, ' บาท');
  Result := True;
end;

function TBankAccount.Transfer(ToAccount: TBankAccount; Amount: Double): Boolean;
begin
  Result := False;

  if Withdraw(Amount, 'โอนไปยัง ' + ToAccount.AccountNumber) then
  begin
    ToAccount.Deposit(Amount, 'รับโอนจาก ' + FAccountNumber);
    Result := True;
  end;
end;

procedure TBankAccount.ShowBalance;
begin
  WriteLn('บัญชี: ', FAccountNumber, ' ชื่อ: ', FOwnerName);
  WriteLn('ยอดคงเหลือ: ', FBalance:0:2, ' บาท');
  WriteLn('สถานะ: ', IfThen(FIsActive, 'เปิดใช้งาน', 'ปิดแล้ว'));
end;

procedure TBankAccount.ShowStatement;
var
  I: Integer;
begin
  WriteLn;
  WriteLn('========================================');
  WriteLn('Statement บัญชี: ', FAccountNumber);
  WriteLn('ชื่อผู้ถือบัญชี: ', FOwnerName);
  WriteLn('========================================');
  WriteLn('วันที่           ประเภท    จำนวนเงิน    ยอดคงเหลือ  รายละเอียด');
  WriteLn('----------------------------------------');
  for I := 0 to FTransactionCount - 1 do
    FTransactions[I].ShowInfo;
  WriteLn('========================================');
  WriteLn('ยอดปัจจุบัน: ', FBalance:0:2, ' บาท');
end;

procedure TBankAccount.CloseAccount;
begin
  if not FIsActive then
  begin
    WriteLn('บัญชีปิดแล้ว');
    Exit;
  end;

  if FBalance > 0 then
    WriteLn('คืนเงิน ', FBalance:0:2, ' บาท ให้ผู้ถือบัญชี');

  FBalance := 0;
  FIsActive := False;
  WriteLn('ปิดบัญชี ', FAccountNumber, ' สำเร็จ');
end;

// ============= Main Program =============
var
  Account1, Account2: TBankAccount;
begin
  WriteLn('===== ระบบธนาคาร =====');
  WriteLn;

  Account1 := TBankAccount.Create('นาย สมชาย ใจดี', 5000);
  Account2 := TBankAccount.Create('นางสาว สมหญิง สวยงาม', 1000);
  WriteLn;

  Account1.Deposit(2000, 'เงินเดือน');
  Account1.Withdraw(1500, 'ค่าเช่าบ้าน');
  Account1.Transfer(Account2, 800);
  Account2.Deposit(500, 'ดอกเบี้ย');
  Account2.Withdraw(200, 'ค่าอาหาร');
  WriteLn;

  Account1.ShowStatement;
  Account2.ShowStatement;

  Account1.Free;
  Account2.Free;

  ReadLn;
end.
```

---

## 21.13 TObject - Base Class ของทุก Class

ใน Free Pascal ทุก class สืบทอดจาก `TObject` โดยอัตโนมัติถ้าไม่ระบุ parent

```pascal
// สองบรรทัดนี้เหมือนกัน
TMyClass = class
TMyClass = class(TObject)
```

### Methods ที่สำคัญของ TObject

```pascal
type
  TObject = class
  public
    constructor Create;
    destructor Destroy; virtual;
    procedure Free;                      // ลบ object อย่างปลอดภัย
    procedure FreeInstance; virtual;
    class function NewInstance: TObject; virtual;
    procedure AfterConstruction; virtual;
    procedure BeforeDestruction; virtual;

    // Type checking
    function ClassType: TClass;
    class function ClassName: string;
    class function ClassNameIs(const Name: string): Boolean;
    class function ClassParent: TClass;
    class function InheritsFrom(AClass: TClass): Boolean;
    function IsClass(AClass: TClass): Boolean;   // ตรวจสอบ type
    function GetInterface(const IID: TGUID; out Obj): Boolean;

    // Equality
    function Equals(Obj: TObject): Boolean; virtual;
    function GetHashCode: PtrInt; virtual;
    function ToString: string; virtual;
  end;
```

### ตัวอย่างการใช้งาน TObject methods

```pascal
var
  Dog: TDog;
  Animal: TAnimal;
begin
  Dog := TDog.Create('บุ๋ม', 'Husky', 2, 12.0);

  // ClassName
  WriteLn('ClassName: ', Dog.ClassName);        // TDog
  WriteLn('Parent: ', Dog.ClassParent.ClassName); // TAnimal

  // InheritsFrom
  WriteLn('เป็น TAnimal: ', Dog.InheritsFrom(TAnimal));  // True
  WriteLn('เป็น TObject: ', Dog.InheritsFrom(TObject));  // True
  WriteLn('เป็น TCat: ', Dog.InheritsFrom(TCat));        // False

  // Type casting ด้วย is
  Animal := Dog;
  if Animal is TDog then
    WriteLn('เป็น TDog จริงๆ');

  // Safe cast ด้วย as
  var ActualDog := Animal as TDog;
  ActualDog.Fetch;

  Dog.Free;
end;
```

---

## 21.14 แบบฝึกหัด 20 ข้อ

### ระดับ Basic (ข้อ 1-7)

**ข้อ 1:** สร้าง class `TCircle` ที่มี radius เป็น field ส่วนตัว พร้อม constructor, getter/setter property และ methods สำหรับคำนวณ Area และ Perimeter

**ข้อ 2:** สร้าง class `TRectangle` ที่มี width และ height พร้อม method คำนวณ Area, Perimeter และ Diagonal

**ข้อ 3:** สร้าง class `TStudent` ที่เก็บชื่อ, รหัส, เกรด 3 วิชา พร้อม method คำนวณ GPA

**ข้อ 4:** สร้าง class `TStack` (โครงสร้างข้อมูล stack) สำหรับ Integer พร้อม Push, Pop, Peek และ IsEmpty

**ข้อ 5:** สร้าง class `TQueue` สำหรับ string พร้อม Enqueue, Dequeue, Front และ Size

**ข้อ 6:** สร้าง class `TPassword` ที่ตรวจสอบว่า password มีความยาวอย่างน้อย 8 ตัว มีตัวเลข และมีตัวพิมพ์ใหญ่

**ข้อ 7:** สร้าง class `TDate` ที่เก็บวัน เดือน ปี พร้อม method แสดงวันที่ในรูปแบบต่างๆ

### ระดับ Intermediate (ข้อ 8-14)

**ข้อ 8:** สร้าง class `TMatrix` ขนาด N×M พร้อม method บวก ลบ และ transpose matrix

**ข้อ 9:** สร้าง class `TLinkedList` แบบ singly linked สำหรับ Integer พร้อม Add, Remove, Find และ Print

**ข้อ 10:** สร้าง class `TFraction` สำหรับเศษส่วน พร้อม operator overloading สำหรับ +, -, *, / และแสดงผลเป็น string

**ข้อ 11:** สร้าง class `TTextAnalyzer` ที่รับ string และนับจำนวนคำ ประโยค ย่อหน้า และหาคำที่ซ้ำมากที่สุด

**ข้อ 12:** สร้าง class `TTimer` ที่จับเวลา พร้อม Start, Stop, Pause, Resume และ GetElapsed

**ข้อ 13:** สร้าง class `TConfig` ที่อ่านและเขียน key-value pairs จากไฟล์ .ini อย่างง่าย

**ข้อ 14:** สร้าง class `TGraph` แบบ adjacency matrix พร้อม AddEdge, RemoveEdge และ PrintAdjacencyMatrix

### ระดับ Advanced (ข้อ 15-20)

**ข้อ 15:** สร้าง class `TObservable` และ `TObserver` ตาม Observer pattern เมื่อ observable เปลี่ยนแปลงต้อง notify observers ทั้งหมด

**ข้อ 16:** สร้าง class `TCommandHistory` ที่เก็บประวัติ commands พร้อม Undo/Redo functionality

**ข้อ 17:** สร้าง class `TEventEmitter` ที่ register event handlers และ emit events พร้อม parameter

**ข้อ 18:** ออกแบบ class hierarchy สำหรับ UI Elements (TControl -> TButton, TTextBox, TLabel, TPanel) พร้อม Draw method

**ข้อ 19:** สร้าง class `TExpressionParser` ที่ parse และ evaluate นิพจน์คณิตศาสตร์อย่างง่าย (+, -, *, /)

**ข้อ 20:** สร้าง class `TCache` ที่เก็บ key-value pairs พร้อม TTL (Time-To-Live) expire อัตโนมัติ และ LRU eviction policy

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **OOP Concepts** - Encapsulation, Inheritance, Polymorphism, Abstraction
2. **Class vs Object** - Class คือ blueprint, Object คือ instance
3. **Instance/Class Variables** - Instance vars แยกกันแต่ละ object, Class vars ใช้ร่วมกัน
4. **Constructor/Destructor** - สร้างและทำลาย objects
5. **Access Modifiers** - public, private, protected, published, strict private, strict protected
6. **Properties** - getter, setter พร้อม validation
7. **Self Reference** - อ้างถึง object ปัจจุบัน
8. **Class vs Instance Methods** - Class methods เรียกได้ไม่ต้องมี instance

บทต่อไปจะเรียนเรื่อง Classes และ Objects เชิงลึกมากขึ้น รวมถึง RTTI และ type checking
