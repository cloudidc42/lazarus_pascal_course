# Part 11 - Records (ระเบียนข้อมูล)

## บทนำ

Record เป็นโครงสร้างข้อมูลพื้นฐานที่สำคัญมากใน Pascal/Lazarus ช่วยให้เราสามารถรวมข้อมูลหลายประเภทที่เกี่ยวข้องกันไว้ในตัวแปรเดียว เปรียบเสมือนแถวข้อมูลในตาราง เช่น ข้อมูลนักเรียนหนึ่งคนประกอบด้วย ชื่อ นามสกุล อายุ คะแนน ซึ่งมีชนิดข้อมูลแตกต่างกันแต่ต้องอยู่ด้วยกัน

---

## 11.1 การประกาศ Record

### รูปแบบพื้นฐาน

```pascal
type
  ชื่อRecord = record
    ชื่อField1 : ชนิดข้อมูล1;
    ชื่อField2 : ชนิดข้อมูล2;
    ชื่อField3 : ชนิดข้อมูล3;
    // ...
  end;
```

### ตัวอย่างที่ 1: Record นักเรียนพื้นฐาน

```pascal
program RecordBasic;

type
  TStudent = record
    ID       : Integer;
    Name     : String[50];
    Age      : Integer;
    Grade    : Char;
    Score    : Real;
  end;

var
  Student : TStudent;

begin
  // กำหนดค่าให้แต่ละ field
  Student.ID    := 1001;
  Student.Name  := 'สมชาย ใจดี';
  Student.Age   := 20;
  Student.Grade := 'A';
  Student.Score := 95.5;

  // แสดงผล
  WriteLn('=== ข้อมูลนักเรียน ===');
  WriteLn('รหัส   : ', Student.ID);
  WriteLn('ชื่อ   : ', Student.Name);
  WriteLn('อายุ   : ', Student.Age, ' ปี');
  WriteLn('เกรด   : ', Student.Grade);
  WriteLn('คะแนน  : ', Student.Score:0:2);
  
  ReadLn;
end.
```

### ตัวอย่างที่ 2: Record บุคคลทั่วไป

```pascal
program PersonRecord;

type
  TGender = (gMale, gFemale, gOther);
  
  TPerson = record
    FirstName  : String[30];
    LastName   : String[30];
    BirthYear  : Integer;
    Gender     : TGender;
    Email      : String[100];
    Phone      : String[15];
  end;

var
  Person : TPerson;

function GetAge(BirthYear: Integer): Integer;
begin
  Result := 2024 - BirthYear;
end;

function GenderToStr(G: TGender): String;
begin
  case G of
    gMale   : Result := 'ชาย';
    gFemale : Result := 'หญิง';
    gOther  : Result := 'อื่นๆ';
  end;
end;

begin
  Person.FirstName := 'สมหญิง';
  Person.LastName  := 'รักเรียน';
  Person.BirthYear := 1998;
  Person.Gender    := gFemale;
  Person.Email     := 'somying@example.com';
  Person.Phone     := '081-234-5678';

  WriteLn('=== ข้อมูลบุคคล ===');
  WriteLn('ชื่อ-นามสกุล : ', Person.FirstName, ' ', Person.LastName);
  WriteLn('อายุ         : ', GetAge(Person.BirthYear), ' ปี');
  WriteLn('เพศ          : ', GenderToStr(Person.Gender));
  WriteLn('อีเมล        : ', Person.Email);
  WriteLn('โทรศัพท์     : ', Person.Phone);
  
  ReadLn;
end.
```

### ตัวอย่างที่ 3: Record สินค้า

```pascal
program ProductRecord;

type
  TProduct = record
    ProductCode : String[10];
    ProductName : String[50];
    Category    : String[30];
    Price       : Real;
    Quantity    : Integer;
    IsActive    : Boolean;
  end;

var
  Product : TProduct;

procedure DisplayProduct(P: TProduct);
begin
  WriteLn('รหัสสินค้า  : ', P.ProductCode);
  WriteLn('ชื่อสินค้า  : ', P.ProductName);
  WriteLn('หมวดหมู่    : ', P.Category);
  WriteLn('ราคา        : ', P.Price:0:2, ' บาท');
  WriteLn('จำนวนคงเหลือ: ', P.Quantity, ' ชิ้น');
  if P.IsActive then
    WriteLn('สถานะ       : เปิดขาย')
  else
    WriteLn('สถานะ       : ปิดขาย');
end;

begin
  Product.ProductCode := 'PRD-001';
  Product.ProductName := 'โน้ตบุ๊ก Lenovo IdeaPad';
  Product.Category    := 'คอมพิวเตอร์';
  Product.Price       := 25990.00;
  Product.Quantity    := 15;
  Product.IsActive    := True;

  WriteLn('=== ข้อมูลสินค้า ===');
  DisplayProduct(Product);
  
  ReadLn;
end.
```

---

## 11.2 การเข้าถึง Fields

การเข้าถึง fields ของ record ใช้จุด (.) เป็น dot notation

### ตัวอย่างที่ 4: การกำหนดและอ่านค่า Fields

```pascal
program AccessFields;

type
  TPoint = record
    X : Real;
    Y : Real;
  end;
  
  TRectangle = record
    TopLeft     : TPoint;
    BottomRight : TPoint;
    Color       : String[20];
  end;

var
  Rect : TRectangle;

function CalcArea(R: TRectangle): Real;
var
  Width, Height : Real;
begin
  Width  := Abs(R.BottomRight.X - R.TopLeft.X);
  Height := Abs(R.BottomRight.Y - R.TopLeft.Y);
  Result := Width * Height;
end;

begin
  Rect.TopLeft.X     := 0;
  Rect.TopLeft.Y     := 0;
  Rect.BottomRight.X := 10;
  Rect.BottomRight.Y := 5;
  Rect.Color         := 'แดง';

  WriteLn('=== ข้อมูลสี่เหลี่ยม ===');
  WriteLn('มุมซ้ายบน   : (', Rect.TopLeft.X:0:1, ', ', Rect.TopLeft.Y:0:1, ')');
  WriteLn('มุมขวาล่าง  : (', Rect.BottomRight.X:0:1, ', ', Rect.BottomRight.Y:0:1, ')');
  WriteLn('สี           : ', Rect.Color);
  WriteLn('พื้นที่       : ', CalcArea(Rect):0:2, ' ตร.หน่วย');
  
  ReadLn;
end.
```

### ตัวอย่างที่ 5: การคัดลอก Records

```pascal
program CopyRecord;

type
  TEmployee = record
    EmpID    : Integer;
    Name     : String[50];
    Salary   : Real;
    Position : String[30];
  end;

var
  Emp1, Emp2 : TEmployee;

begin
  // กำหนดค่าให้ Emp1
  Emp1.EmpID    := 1001;
  Emp1.Name     := 'สมศักดิ์ มานะ';
  Emp1.Salary   := 35000;
  Emp1.Position := 'นักพัฒนาซอฟต์แวร์';

  // คัดลอก record ทั้งหมดจาก Emp1 ไป Emp2
  Emp2 := Emp1;
  
  // แก้ไขเฉพาะบาง field ของ Emp2
  Emp2.EmpID := 1002;
  Emp2.Name  := 'สมใจ รักงาน';
  // Salary และ Position ยังเหมือนเดิม

  WriteLn('=== พนักงานคนที่ 1 ===');
  WriteLn('รหัส     : ', Emp1.EmpID);
  WriteLn('ชื่อ     : ', Emp1.Name);
  WriteLn('เงินเดือน: ', Emp1.Salary:0:2);
  WriteLn('ตำแหน่ง  : ', Emp1.Position);
  
  WriteLn;
  WriteLn('=== พนักงานคนที่ 2 ===');
  WriteLn('รหัส     : ', Emp2.EmpID);
  WriteLn('ชื่อ     : ', Emp2.Name);
  WriteLn('เงินเดือน: ', Emp2.Salary:0:2);
  WriteLn('ตำแหน่ง  : ', Emp2.Position);
  
  ReadLn;
end.
```

---

## 11.3 Nested Records (Record ซ้อน)

Record สามารถมี field ที่เป็น record อีกอันหนึ่งได้

### ตัวอย่างที่ 6: ที่อยู่ซ้อนใน Record บุคคล

```pascal
program NestedRecords;

type
  TAddress = record
    HouseNo  : String[10];
    Street   : String[50];
    District : String[30];
    Province : String[30];
    Zipcode  : String[5];
  end;
  
  TContactInfo = record
    Phone    : String[15];
    Mobile   : String[15];
    Email    : String[100];
    Address  : TAddress;  // nested record
  end;
  
  TCustomer = record
    CustomerID : Integer;
    Name       : String[60];
    TaxID      : String[13];
    Contact    : TContactInfo;  // nested record
    JoinDate   : String[10];
  end;

var
  Customer : TCustomer;

procedure DisplayCustomer(C: TCustomer);
begin
  WriteLn('=== ข้อมูลลูกค้า ===');
  WriteLn('รหัสลูกค้า   : ', C.CustomerID);
  WriteLn('ชื่อ          : ', C.Name);
  WriteLn('เลขภาษี      : ', C.TaxID);
  WriteLn('วันที่สมัคร  : ', C.JoinDate);
  WriteLn('');
  WriteLn('--- ข้อมูลติดต่อ ---');
  WriteLn('โทรศัพท์     : ', C.Contact.Phone);
  WriteLn('มือถือ        : ', C.Contact.Mobile);
  WriteLn('อีเมล         : ', C.Contact.Email);
  WriteLn('');
  WriteLn('--- ที่อยู่ ---');
  WriteLn('บ้านเลขที่   : ', C.Contact.Address.HouseNo);
  WriteLn('ถนน           : ', C.Contact.Address.Street);
  WriteLn('อำเภอ         : ', C.Contact.Address.District);
  WriteLn('จังหวัด       : ', C.Contact.Address.Province);
  WriteLn('รหัสไปรษณีย์ : ', C.Contact.Address.Zipcode);
end;

begin
  Customer.CustomerID               := 10001;
  Customer.Name                     := 'บริษัท ตัวอย่าง จำกัด';
  Customer.TaxID                    := '0-1234-56789-01-2';
  Customer.JoinDate                 := '01/01/2024';
  Customer.Contact.Phone            := '02-123-4567';
  Customer.Contact.Mobile           := '081-234-5678';
  Customer.Contact.Email            := 'info@example.co.th';
  Customer.Contact.Address.HouseNo  := '123/45';
  Customer.Contact.Address.Street   := 'ถนนสุขุมวิท';
  Customer.Contact.Address.District := 'คลองเตย';
  Customer.Contact.Address.Province := 'กรุงเทพมหานคร';
  Customer.Contact.Address.Zipcode  := '10110';
  
  DisplayCustomer(Customer);
  ReadLn;
end.
```

### ตัวอย่างที่ 7: Record วันที่และเวลา

```pascal
program DateTimeRecord;

type
  TDate = record
    Day   : Integer;
    Month : Integer;
    Year  : Integer;
  end;
  
  TTime = record
    Hour   : Integer;
    Minute : Integer;
    Second : Integer;
  end;
  
  TDateTime = record
    Date : TDate;
    Time : TTime;
  end;
  
  TEvent = record
    EventName : String[100];
    StartTime : TDateTime;
    EndTime   : TDateTime;
    Location  : String[100];
    Notes     : String[200];
  end;

function DateToStr(D: TDate): String;
begin
  Result := Format('%02d/%02d/%04d', [D.Day, D.Month, D.Year]);
end;

function TimeToStr(T: TTime): String;
begin
  Result := Format('%02d:%02d:%02d', [T.Hour, T.Minute, T.Second]);
end;

var
  Event : TEvent;

begin
  Event.EventName              := 'ประชุมทีมพัฒนา';
  Event.StartTime.Date.Day     := 15;
  Event.StartTime.Date.Month   := 3;
  Event.StartTime.Date.Year    := 2024;
  Event.StartTime.Time.Hour    := 9;
  Event.StartTime.Time.Minute  := 0;
  Event.StartTime.Time.Second  := 0;
  Event.EndTime.Date.Day       := 15;
  Event.EndTime.Date.Month     := 3;
  Event.EndTime.Date.Year      := 2024;
  Event.EndTime.Time.Hour      := 12;
  Event.EndTime.Time.Minute    := 0;
  Event.EndTime.Time.Second    := 0;
  Event.Location               := 'ห้องประชุม A ชั้น 3';
  Event.Notes                  := 'นำเสนอ roadmap ไตรมาส 2';

  WriteLn('=== รายละเอียดกิจกรรม ===');
  WriteLn('ชื่อกิจกรรม : ', Event.EventName);
  WriteLn('เริ่มต้น     : ', DateToStr(Event.StartTime.Date), ' ', TimeToStr(Event.StartTime.Time));
  WriteLn('สิ้นสุด      : ', DateToStr(Event.EndTime.Date), ' ', TimeToStr(Event.EndTime.Time));
  WriteLn('สถานที่      : ', Event.Location);
  WriteLn('หมายเหตุ     : ', Event.Notes);
  
  ReadLn;
end.
```

---

## 11.4 Array of Records

การเก็บข้อมูลหลายๆ record ไว้ใน array

### ตัวอย่างที่ 8: Array of Records นักเรียน

```pascal
program ArrayOfRecords;

const
  MAX_STUDENTS = 5;

type
  TStudent = record
    ID    : Integer;
    Name  : String[50];
    Score : Real;
    Grade : Char;
  end;

var
  Students : array[1..MAX_STUDENTS] of TStudent;
  i        : Integer;

function CalcGrade(Score: Real): Char;
begin
  if Score >= 80 then Result := 'A'
  else if Score >= 70 then Result := 'B'
  else if Score >= 60 then Result := 'C'
  else if Score >= 50 then Result := 'D'
  else Result := 'F';
end;

procedure InitStudents;
begin
  Students[1].ID := 1; Students[1].Name := 'สมชาย'; Students[1].Score := 85;
  Students[2].ID := 2; Students[2].Name := 'สมหญิง'; Students[2].Score := 72;
  Students[3].ID := 3; Students[3].Name := 'อนุชา'; Students[3].Score := 91;
  Students[4].ID := 4; Students[4].Name := 'วิภา'; Students[4].Score := 63;
  Students[5].ID := 5; Students[5].Name := 'ธนพล'; Students[5].Score := 78;
  
  // คำนวณเกรด
  for i := 1 to MAX_STUDENTS do
    Students[i].Grade := CalcGrade(Students[i].Score);
end;

procedure DisplayAll;
var
  Total : Real;
  Avg   : Real;
begin
  WriteLn('=== ตารางคะแนนนักเรียน ===');
  WriteLn(Format('%-4s %-15s %-8s %-5s', ['ID', 'ชื่อ', 'คะแนน', 'เกรด']));
  WriteLn(StringOfChar('-', 40));
  
  Total := 0;
  for i := 1 to MAX_STUDENTS do
  begin
    WriteLn(Format('%-4d %-15s %-8.2f %-5s',
      [Students[i].ID, Students[i].Name, Students[i].Score, Students[i].Grade]));
    Total := Total + Students[i].Score;
  end;
  
  Avg := Total / MAX_STUDENTS;
  WriteLn(StringOfChar('-', 40));
  WriteLn(Format('คะแนนเฉลี่ย: %.2f', [Avg]));
end;

begin
  InitStudents;
  DisplayAll;
  ReadLn;
end.
```

### ตัวอย่างที่ 9: การเรียงลำดับ Array of Records (Bubble Sort)

```pascal
program SortRecords;

const
  MAX = 6;

type
  TEmployee = record
    ID     : Integer;
    Name   : String[30];
    Salary : Real;
    Dept   : String[20];
  end;

var
  Emps : array[1..MAX] of TEmployee;
  i    : Integer;

procedure InitData;
begin
  Emps[1] := TEmployee(1, 'สมชาย', 35000, 'IT');
  // Pascal ไม่รองรับ aggregate constructor แบบนี้ในทุก compiler
  // ใช้วิธีกำหนดทีละ field แทน
  Emps[1].ID := 1; Emps[1].Name := 'สมชาย'; Emps[1].Salary := 35000; Emps[1].Dept := 'IT';
  Emps[2].ID := 2; Emps[2].Name := 'สมหญิง'; Emps[2].Salary := 42000; Emps[2].Dept := 'HR';
  Emps[3].ID := 3; Emps[3].Name := 'อนุชา'; Emps[3].Salary := 28000; Emps[3].Dept := 'Sales';
  Emps[4].ID := 4; Emps[4].Name := 'วิภา'; Emps[4].Salary := 55000; Emps[4].Dept := 'Finance';
  Emps[5].ID := 5; Emps[5].Name := 'ธนพล'; Emps[5].Salary := 38000; Emps[5].Dept := 'IT';
  Emps[6].ID := 6; Emps[6].Name := 'นภาพร'; Emps[6].Salary := 31000; Emps[6].Dept := 'Sales';
end;

// เรียงตามเงินเดือนจากมากไปน้อย
procedure SortBySalaryDesc;
var
  i, j : Integer;
  Temp : TEmployee;
begin
  for i := 1 to MAX - 1 do
    for j := 1 to MAX - i do
      if Emps[j].Salary < Emps[j+1].Salary then
      begin
        Temp     := Emps[j];
        Emps[j]  := Emps[j+1];
        Emps[j+1] := Temp;
      end;
end;

// เรียงตามชื่อ A-Z
procedure SortByName;
var
  i, j : Integer;
  Temp : TEmployee;
begin
  for i := 1 to MAX - 1 do
    for j := 1 to MAX - i do
      if Emps[j].Name > Emps[j+1].Name then
      begin
        Temp     := Emps[j];
        Emps[j]  := Emps[j+1];
        Emps[j+1] := Temp;
      end;
end;

procedure DisplayEmps(Title: String);
begin
  WriteLn('=== ', Title, ' ===');
  WriteLn(Format('%-4s %-15s %-10s %-10s', ['ID', 'ชื่อ', 'เงินเดือน', 'แผนก']));
  WriteLn(StringOfChar('-', 45));
  for i := 1 to MAX do
    WriteLn(Format('%-4d %-15s %-10.2f %-10s',
      [Emps[i].ID, Emps[i].Name, Emps[i].Salary, Emps[i].Dept]));
  WriteLn;
end;

begin
  InitData;
  
  DisplayEmps('ข้อมูลเดิม');
  
  SortBySalaryDesc;
  DisplayEmps('เรียงตามเงินเดือน (มาก -> น้อย)');
  
  InitData;
  SortByName;
  DisplayEmps('เรียงตามชื่อ (A -> Z)');
  
  ReadLn;
end.
```

### ตัวอย่างที่ 10: การค้นหาใน Array of Records

```pascal
program SearchInRecords;

const
  MAX_PRODUCTS = 8;

type
  TProduct = record
    Code     : String[10];
    Name     : String[40];
    Category : String[20];
    Price    : Real;
    Stock    : Integer;
  end;

var
  Products : array[1..MAX_PRODUCTS] of TProduct;

procedure InitProducts;
begin
  Products[1].Code := 'P001'; Products[1].Name := 'แล็ปท็อป';
  Products[1].Category := 'IT'; Products[1].Price := 25000; Products[1].Stock := 10;
  
  Products[2].Code := 'P002'; Products[2].Name := 'เมาส์ไร้สาย';
  Products[2].Category := 'IT'; Products[2].Price := 890; Products[2].Stock := 50;
  
  Products[3].Code := 'P003'; Products[3].Name := 'คีย์บอร์ด';
  Products[3].Category := 'IT'; Products[3].Price := 1290; Products[3].Stock := 30;
  
  Products[4].Code := 'P004'; Products[4].Name := 'โต๊ะทำงาน';
  Products[4].Category := 'เฟอร์นิเจอร์'; Products[4].Price := 8500; Products[4].Stock := 5;
  
  Products[5].Code := 'P005'; Products[5].Name := 'เก้าอี้สำนักงาน';
  Products[5].Category := 'เฟอร์นิเจอร์'; Products[5].Price := 5500; Products[5].Stock := 8;
  
  Products[6].Code := 'P006'; Products[6].Name := 'ปากกา';
  Products[6].Category := 'เครื่องเขียน'; Products[6].Price := 25; Products[6].Stock := 200;
  
  Products[7].Code := 'P007'; Products[7].Name := 'สมุด A4';
  Products[7].Category := 'เครื่องเขียน'; Products[7].Price := 60; Products[7].Stock := 100;
  
  Products[8].Code := 'P008'; Products[8].Name := 'จอมอนิเตอร์';
  Products[8].Category := 'IT'; Products[8].Price := 7500; Products[8].Stock := 15;
end;

// ค้นหาตามรหัสสินค้า
function FindByCode(Code: String): Integer;
var
  i : Integer;
begin
  Result := -1;
  for i := 1 to MAX_PRODUCTS do
    if Products[i].Code = Code then
    begin
      Result := i;
      Exit;
    end;
end;

// ค้นหาตามหมวดหมู่
procedure FindByCategory(Category: String);
var
  i     : Integer;
  Found : Boolean;
begin
  Found := False;
  WriteLn('สินค้าในหมวด: ', Category);
  WriteLn(StringOfChar('-', 50));
  
  for i := 1 to MAX_PRODUCTS do
    if Products[i].Category = Category then
    begin
      WriteLn(Format('%-6s %-20s %10.2f %5d',
        [Products[i].Code, Products[i].Name,
         Products[i].Price, Products[i].Stock]));
      Found := True;
    end;
    
  if not Found then
    WriteLn('ไม่พบสินค้าในหมวดนี้');
end;

var
  SearchCode  : String;
  SearchIndex : Integer;

begin
  InitProducts;
  
  // ค้นหาด้วยรหัส
  Write('กรอกรหัสสินค้าที่ต้องการค้นหา: ');
  ReadLn(SearchCode);
  
  SearchIndex := FindByCode(SearchCode);
  
  if SearchIndex > 0 then
  begin
    WriteLn('=== พบสินค้า ===');
    WriteLn('รหัส        : ', Products[SearchIndex].Code);
    WriteLn('ชื่อสินค้า  : ', Products[SearchIndex].Name);
    WriteLn('หมวดหมู่    : ', Products[SearchIndex].Category);
    WriteLn('ราคา        : ', Products[SearchIndex].Price:0:2, ' บาท');
    WriteLn('คงเหลือ     : ', Products[SearchIndex].Stock, ' ชิ้น');
  end
  else
    WriteLn('ไม่พบสินค้ารหัส: ', SearchCode);
  
  WriteLn;
  WriteLn('สินค้าหมวด IT:');
  FindByCategory('IT');
  
  ReadLn;
end.
```

---

## 11.5 With...Do Statement

`with...do` ช่วยลดการพิมพ์ชื่อ record ซ้ำๆ เมื่อต้องการเข้าถึงหลาย fields

### ตัวอย่างที่ 11: การใช้ with...do พื้นฐาน

```pascal
program WithDoBasic;

type
  TCircle = record
    X      : Real;
    Y      : Real;
    Radius : Real;
    Color  : String[20];
  end;

var
  Circle : TCircle;
  Area   : Real;
  Perimeter : Real;

const
  PI = 3.14159265358979;

begin
  // ไม่ใช้ with...do (verbose)
  Circle.X      := 100;
  Circle.Y      := 200;
  Circle.Radius := 50;
  Circle.Color  := 'น้ำเงิน';
  
  WriteLn('วิธีที่ 1: ไม่ใช้ with');
  WriteLn('ศูนย์กลาง: (', Circle.X:0:1, ', ', Circle.Y:0:1, ')');
  WriteLn('รัศมี: ', Circle.Radius:0:1);
  
  WriteLn;
  
  // ใช้ with...do (สะอาดกว่า)
  with Circle do
  begin
    X      := 150;
    Y      := 250;
    Radius := 75;
    Color  := 'แดง';
    
    Area      := PI * Radius * Radius;
    Perimeter := 2 * PI * Radius;
    
    WriteLn('วิธีที่ 2: ใช้ with');
    WriteLn('ศูนย์กลาง: (', X:0:1, ', ', Y:0:1, ')');
    WriteLn('รัศมี     : ', Radius:0:1);
    WriteLn('สี        : ', Color);
    WriteLn('พื้นที่    : ', Area:0:2);
    WriteLn('เส้นรอบวง : ', Perimeter:0:2);
  end;
  
  ReadLn;
end.
```

### ตัวอย่างที่ 12: with...do กับ Nested Records

```pascal
program WithDoNested;

type
  TPosition = record
    X, Y, Z : Real;
  end;
  
  TVelocity = record
    Vx, Vy, Vz : Real;
  end;
  
  TParticle = record
    Name     : String[20];
    Mass     : Real;
    Pos      : TPosition;
    Vel      : TVelocity;
    Charge   : Real;
  end;

var
  P : TParticle;

procedure UpdatePosition(var Particle: TParticle; dt: Real);
begin
  with Particle do
  begin
    with Pos do
    begin
      X := X + Vel.Vx * dt;
      Y := Y + Vel.Vy * dt;
      Z := Z + Vel.Vz * dt;
    end;
  end;
end;

function KineticEnergy(P: TParticle): Real;
begin
  with P do
    with Vel do
      Result := 0.5 * P.Mass * (Vx*Vx + Vy*Vy + Vz*Vz);
end;

begin
  with P do
  begin
    Name := 'อิเล็กตรอน';
    Mass := 9.11e-31;
    Charge := -1.6e-19;
    
    with Pos do begin X := 0; Y := 0; Z := 0; end;
    with Vel do begin Vx := 1000; Vy := 500; Vz := 0; end;
  end;
  
  WriteLn('=== ข้อมูลอนุภาค ===');
  WriteLn('ชื่อ       : ', P.Name);
  WriteLn('มวล (kg)   : ', P.Mass);
  WriteLn('ตำแหน่ง   : (', P.Pos.X:0:2, ', ', P.Pos.Y:0:2, ', ', P.Pos.Z:0:2, ')');
  WriteLn('พลังงาน KE : ', KineticEnergy(P));
  
  UpdatePosition(P, 0.001);
  
  WriteLn;
  WriteLn('หลังจาก dt = 0.001s:');
  WriteLn('ตำแหน่ง   : (', P.Pos.X:0:4, ', ', P.Pos.Y:0:4, ', ', P.Pos.Z:0:4, ')');
  
  ReadLn;
end.
```

### ตัวอย่างที่ 13: with...do กับ Array of Records

```pascal
program WithDoArray;

const
  N = 4;

type
  TBook = record
    ISBN   : String[13];
    Title  : String[60];
    Author : String[40];
    Pages  : Integer;
    Price  : Real;
  end;

var
  Books : array[1..N] of TBook;
  i     : Integer;

begin
  // กำหนดค่าโดยใช้ with
  with Books[1] do begin
    ISBN   := '978-616-123-001-1';
    Title  := 'เรียน Pascal เบื้องต้น';
    Author := 'สมชาย ใจดี';
    Pages  := 320;
    Price  := 280;
  end;
  
  with Books[2] do begin
    ISBN   := '978-616-123-002-2';
    Title  := 'โปรแกรมมิ่งสำหรับผู้เริ่มต้น';
    Author := 'วิภา รักเรียน';
    Pages  := 450;
    Price  := 350;
  end;
  
  with Books[3] do begin
    ISBN   := '978-616-123-003-3';
    Title  := 'อัลกอริทึมและโครงสร้างข้อมูล';
    Author := 'อนุชา สมาร์ท';
    Pages  := 580;
    Price  := 490;
  end;
  
  with Books[4] do begin
    ISBN   := '978-616-123-004-4';
    Title  := 'Lazarus GUI Programming';
    Author := 'ธนพล คอดี';
    Pages  := 420;
    Price  := 395;
  end;
  
  WriteLn('=== รายการหนังสือ ===');
  for i := 1 to N do
    with Books[i] do
    begin
      WriteLn('หนังสือที่ ', i);
      WriteLn('  ISBN   : ', ISBN);
      WriteLn('  ชื่อ   : ', Title);
      WriteLn('  ผู้แต่ง: ', Author);
      WriteLn('  หน้า   : ', Pages);
      WriteLn('  ราคา   : ', Price:0:2, ' บาท');
      WriteLn;
    end;
  
  ReadLn;
end.
```

---

## 11.6 Variant Records

Variant records ใช้ `case` ในการกำหนดว่า field ไหนที่ใช้งาน ณ เวลานั้น ช่วยประหยัด memory

### ตัวอย่างที่ 14: Variant Record พื้นฐาน

```pascal
program VariantRecord;

type
  TShape = (shCircle, shRectangle, shTriangle);
  
  TGeometry = record
    Color : String[20];
    case Shape: TShape of
      shCircle:    (Radius: Real);
      shRectangle: (Width, Height: Real);
      shTriangle:  (Base, TriHeight: Real);
  end;

const
  PI = 3.14159;

function CalcArea(G: TGeometry): Real;
begin
  case G.Shape of
    shCircle:    Result := PI * G.Radius * G.Radius;
    shRectangle: Result := G.Width * G.Height;
    shTriangle:  Result := 0.5 * G.Base * G.TriHeight;
  end;
end;

var
  G1, G2, G3 : TGeometry;

begin
  // วงกลม
  G1.Shape  := shCircle;
  G1.Color  := 'แดง';
  G1.Radius := 5.0;
  
  // สี่เหลี่ยม
  G2.Shape  := shRectangle;
  G2.Color  := 'น้ำเงิน';
  G2.Width  := 8.0;
  G2.Height := 4.0;
  
  // สามเหลี่ยม
  G3.Shape     := shTriangle;
  G3.Color     := 'เขียว';
  G3.Base      := 6.0;
  G3.TriHeight := 9.0;
  
  WriteLn('=== พื้นที่รูปทรงเรขาคณิต ===');
  WriteLn('วงกลม (r=', G1.Radius:0:1, ')     สี=', G1.Color, '  พื้นที่=', CalcArea(G1):0:4);
  WriteLn('สี่เหลี่ยม (', G2.Width:0:1, 'x', G2.Height:0:1, ') สี=', G2.Color, '  พื้นที่=', CalcArea(G2):0:4);
  WriteLn('สามเหลี่ยม (b=', G3.Base:0:1, ', h=', G3.TriHeight:0:1, ') สี=', G3.Color, '  พื้นที่=', CalcArea(G3):0:4);
  
  ReadLn;
end.
```

### ตัวอย่างที่ 15: Variant Record สำหรับข้อมูลหลายประเภท

```pascal
program VariantData;

type
  TDataType = (dtInteger, dtFloat, dtString, dtBoolean);
  
  TValue = record
    Name : String[30];
    case DataType: TDataType of
      dtInteger : (IntVal  : Integer);
      dtFloat   : (FloatVal: Real);
      dtString  : (StrVal  : String[100]);
      dtBoolean : (BoolVal : Boolean);
  end;

procedure DisplayValue(V: TValue);
begin
  Write(V.Name, ' = ');
  case V.DataType of
    dtInteger : WriteLn(V.IntVal, ' (Integer)');
    dtFloat   : WriteLn(V.FloatVal:0:4, ' (Float)');
    dtString  : WriteLn('"', V.StrVal, '" (String)');
    dtBoolean : if V.BoolVal then WriteLn('True (Boolean)')
                else WriteLn('False (Boolean)');
  end;
end;

var
  V1, V2, V3, V4 : TValue;

begin
  V1.Name := 'จำนวนนักเรียน'; V1.DataType := dtInteger; V1.IntVal := 35;
  V2.Name := 'คะแนนเฉลี่ย';  V2.DataType := dtFloat;   V2.FloatVal := 78.5;
  V3.Name := 'ชื่อห้อง';      V3.DataType := dtString;  V3.StrVal := 'ม.4/1';
  V4.Name := 'สอบผ่าน';       V4.DataType := dtBoolean; V4.BoolVal := True;
  
  WriteLn('=== ค่าตัวแปรประเภทต่างๆ ===');
  DisplayValue(V1);
  DisplayValue(V2);
  DisplayValue(V3);
  DisplayValue(V4);
  
  ReadLn;
end.
```

---

## 11.7 Records กับ Procedures/Functions

### ตัวอย่างที่ 16: ส่ง Record เป็น Parameter (by value)

```pascal
program RecordByValue;

type
  TVector = record
    X, Y, Z : Real;
  end;

function VectorLength(V: TVector): Real;
begin
  Result := Sqrt(V.X*V.X + V.Y*V.Y + V.Z*V.Z);
end;

function DotProduct(A, B: TVector): Real;
begin
  Result := A.X*B.X + A.Y*B.Y + A.Z*B.Z;
end;

function AddVectors(A, B: TVector): TVector;
begin
  Result.X := A.X + B.X;
  Result.Y := A.Y + B.Y;
  Result.Z := A.Z + B.Z;
end;

function ScaleVector(V: TVector; Scale: Real): TVector;
begin
  Result.X := V.X * Scale;
  Result.Y := V.Y * Scale;
  Result.Z := V.Z * Scale;
end;

function Normalize(V: TVector): TVector;
var
  Len : Real;
begin
  Len := VectorLength(V);
  if Len > 0 then
  begin
    Result.X := V.X / Len;
    Result.Y := V.Y / Len;
    Result.Z := V.Z / Len;
  end
  else
    Result := V;
end;

var
  V1, V2, Sum, Norm : TVector;

begin
  V1.X := 3; V1.Y := 4; V1.Z := 0;
  V2.X := 1; V2.Y := 2; V2.Z := 3;
  
  Sum  := AddVectors(V1, V2);
  Norm := Normalize(V1);
  
  WriteLn('=== Vector Operations ===');
  WriteLn(Format('V1 = (%.1f, %.1f, %.1f)', [V1.X, V1.Y, V1.Z]));
  WriteLn(Format('V2 = (%.1f, %.1f, %.1f)', [V2.X, V2.Y, V2.Z]));
  WriteLn(Format('|V1| = %.4f', [VectorLength(V1)]));
  WriteLn(Format('V1·V2 = %.4f', [DotProduct(V1, V2)]));
  WriteLn(Format('V1+V2 = (%.1f, %.1f, %.1f)', [Sum.X, Sum.Y, Sum.Z]));
  WriteLn(Format('Normalize(V1) = (%.4f, %.4f, %.4f)', [Norm.X, Norm.Y, Norm.Z]));
  
  ReadLn;
end.
```

### ตัวอย่างที่ 17: ส่ง Record เป็น Parameter (by reference)

```pascal
program RecordByReference;

type
  TBankAccount = record
    AccountNo : String[15];
    Owner     : String[50];
    Balance   : Real;
    IsActive  : Boolean;
  end;

function Deposit(var Account: TBankAccount; Amount: Real): Boolean;
begin
  Result := False;
  if not Account.IsActive then
  begin
    WriteLn('บัญชีไม่ได้เปิดใช้งาน');
    Exit;
  end;
  if Amount <= 0 then
  begin
    WriteLn('จำนวนเงินต้องมากกว่า 0');
    Exit;
  end;
  Account.Balance := Account.Balance + Amount;
  Result := True;
end;

function Withdraw(var Account: TBankAccount; Amount: Real): Boolean;
begin
  Result := False;
  if not Account.IsActive then
  begin
    WriteLn('บัญชีไม่ได้เปิดใช้งาน');
    Exit;
  end;
  if Amount <= 0 then
  begin
    WriteLn('จำนวนเงินต้องมากกว่า 0');
    Exit;
  end;
  if Amount > Account.Balance then
  begin
    WriteLn('ยอดเงินไม่เพียงพอ');
    Exit;
  end;
  Account.Balance := Account.Balance - Amount;
  Result := True;
end;

procedure PrintStatement(Account: TBankAccount);
begin
  WriteLn('=== ใบแจ้งยอดบัญชี ===');
  WriteLn('เลขที่บัญชี : ', Account.AccountNo);
  WriteLn('เจ้าของ     : ', Account.Owner);
  WriteLn('ยอดคงเหลือ  : ', Account.Balance:0:2, ' บาท');
  if Account.IsActive then
    WriteLn('สถานะ       : เปิดใช้งาน')
  else
    WriteLn('สถานะ       : ปิดบัญชี');
end;

var
  Account : TBankAccount;

begin
  Account.AccountNo := '123-456789-0';
  Account.Owner     := 'สมชาย ใจดี';
  Account.Balance   := 10000;
  Account.IsActive  := True;
  
  PrintStatement(Account);
  WriteLn;
  
  if Deposit(Account, 5000) then
    WriteLn('ฝากเงิน 5,000 บาท สำเร็จ');
  if Withdraw(Account, 3000) then
    WriteLn('ถอนเงิน 3,000 บาท สำเร็จ');
  if not Withdraw(Account, 20000) then
    WriteLn('ถอนเงิน 20,000 บาท ล้มเหลว');
  
  WriteLn;
  PrintStatement(Account);
  
  ReadLn;
end.
```

### ตัวอย่างที่ 18: Function คืนค่า Record

```pascal
program FunctionReturnRecord;

type
  TStatistics = record
    Count  : Integer;
    Min    : Real;
    Max    : Real;
    Sum    : Real;
    Mean   : Real;
    Median : Real;
    StdDev : Real;
  end;

function CalcStats(Data: array of Real): TStatistics;
var
  i    : Integer;
  n    : Integer;
  Temp : Real;
  SumSq: Real;
  Sorted: array of Real;
begin
  n := Length(Data);
  Result.Count := n;
  
  if n = 0 then Exit;
  
  Result.Min := Data[0];
  Result.Max := Data[0];
  Result.Sum := 0;
  
  // คำนวณ min, max, sum
  for i := 0 to n - 1 do
  begin
    if Data[i] < Result.Min then Result.Min := Data[i];
    if Data[i] > Result.Max then Result.Max := Data[i];
    Result.Sum := Result.Sum + Data[i];
  end;
  
  Result.Mean := Result.Sum / n;
  
  // คำนวณ standard deviation
  SumSq := 0;
  for i := 0 to n - 1 do
    SumSq := SumSq + Sqr(Data[i] - Result.Mean);
  Result.StdDev := Sqrt(SumSq / n);
  
  // คำนวณ median (sort ก่อน)
  SetLength(Sorted, n);
  for i := 0 to n - 1 do
    Sorted[i] := Data[i];
  
  // Bubble sort สำหรับ median
  var j: Integer;
  for i := 0 to n - 2 do
    for j := 0 to n - 2 - i do
      if Sorted[j] > Sorted[j+1] then
      begin
        Temp      := Sorted[j];
        Sorted[j] := Sorted[j+1];
        Sorted[j+1] := Temp;
      end;
  
  if n mod 2 = 0 then
    Result.Median := (Sorted[n div 2 - 1] + Sorted[n div 2]) / 2
  else
    Result.Median := Sorted[n div 2];
end;

var
  Data  : array of Real;
  Stats : TStatistics;

begin
  SetLength(Data, 10);
  Data[0] := 85; Data[1] := 72; Data[2] := 91; Data[3] := 68;
  Data[4] := 79; Data[5] := 95; Data[6] := 63; Data[7] := 87;
  Data[8] := 74; Data[9] := 88;
  
  Stats := CalcStats(Data);
  
  WriteLn('=== สถิติคะแนน ===');
  WriteLn('จำนวนข้อมูล : ', Stats.Count);
  WriteLn('ต่ำสุด       : ', Stats.Min:0:2);
  WriteLn('สูงสุด       : ', Stats.Max:0:2);
  WriteLn('ผลรวม        : ', Stats.Sum:0:2);
  WriteLn('ค่าเฉลี่ย    : ', Stats.Mean:0:4);
  WriteLn('มัธยฐาน     : ', Stats.Median:0:4);
  WriteLn('ส่วนเบี่ยงเบน: ', Stats.StdDev:0:4);
  
  ReadLn;
end.
```

---

## 11.8 โปรแกรมตัวอย่าง: ระบบข้อมูลนักเรียน

```pascal
program StudentManagement;

{$mode objfpc}{$H+}

uses SysUtils;

const
  MAX_STUDENTS = 100;

type
  TSubjectScore = record
    SubjectName : String[30];
    Score       : Real;
    Grade       : Char;
  end;
  
  TStudent = record
    ID        : String[10];
    Name      : String[50];
    Surname   : String[50];
    Class     : String[10];
    Subjects  : array[1..5] of TSubjectScore;
    NumSubjects : Integer;
    GPA        : Real;
  end;

var
  Students : array[1..MAX_STUDENTS] of TStudent;
  NumStudents : Integer;

function CalcGrade(Score: Real): Char;
begin
  if Score >= 80 then Result := 'A'
  else if Score >= 70 then Result := 'B'
  else if Score >= 60 then Result := 'C'
  else if Score >= 50 then Result := 'D'
  else Result := 'F';
end;

function GradeToGPA(Grade: Char): Real;
begin
  case Grade of
    'A': Result := 4.0;
    'B': Result := 3.0;
    'C': Result := 2.0;
    'D': Result := 1.0;
    else Result := 0.0;
  end;
end;

procedure CalcStudentGPA(var S: TStudent);
var
  Total : Real;
  i     : Integer;
begin
  Total := 0;
  if S.NumSubjects = 0 then begin S.GPA := 0; Exit; end;
  for i := 1 to S.NumSubjects do
    Total := Total + GradeToGPA(S.Subjects[i].Grade);
  S.GPA := Total / S.NumSubjects;
end;

procedure AddStudent(ID, Name, Surname, Class: String);
begin
  if NumStudents >= MAX_STUDENTS then
  begin
    WriteLn('ระบบเต็ม ไม่สามารถเพิ่มได้');
    Exit;
  end;
  Inc(NumStudents);
  Students[NumStudents].ID          := ID;
  Students[NumStudents].Name        := Name;
  Students[NumStudents].Surname     := Surname;
  Students[NumStudents].Class       := Class;
  Students[NumStudents].NumSubjects := 0;
  Students[NumStudents].GPA         := 0;
end;

procedure AddScore(StudentID, SubjectName: String; Score: Real);
var
  i, j : Integer;
begin
  for i := 1 to NumStudents do
    if Students[i].ID = StudentID then
    begin
      j := Students[i].NumSubjects + 1;
      if j <= 5 then
      begin
        Students[i].Subjects[j].SubjectName := SubjectName;
        Students[i].Subjects[j].Score       := Score;
        Students[i].Subjects[j].Grade       := CalcGrade(Score);
        Students[i].NumSubjects := j;
        CalcStudentGPA(Students[i]);
      end;
      Exit;
    end;
  WriteLn('ไม่พบนักเรียนรหัส: ', StudentID);
end;

function FindStudent(ID: String): Integer;
var
  i : Integer;
begin
  Result := -1;
  for i := 1 to NumStudents do
    if Students[i].ID = ID then
    begin
      Result := i;
      Exit;
    end;
end;

procedure DisplayStudent(Idx: Integer);
var
  i : Integer;
begin
  if (Idx < 1) or (Idx > NumStudents) then Exit;
  
  with Students[Idx] do
  begin
    WriteLn('=================================');
    WriteLn('รหัสนักเรียน : ', ID);
    WriteLn('ชื่อ-นามสกุล : ', Name, ' ', Surname);
    WriteLn('ห้องเรียน    : ', Class);
    WriteLn('---------------------------------');
    WriteLn(Format('%-20s %-8s %-5s', ['วิชา', 'คะแนน', 'เกรด']));
    WriteLn(StringOfChar('-', 36));
    for i := 1 to NumSubjects do
      WriteLn(Format('%-20s %-8.2f %-5s',
        [Subjects[i].SubjectName, Subjects[i].Score, Subjects[i].Grade]));
    WriteLn(StringOfChar('-', 36));
    WriteLn(Format('GPA: %.2f', [GPA]));
    WriteLn('=================================');
  end;
end;

procedure DisplayAllStudents;
var
  i : Integer;
begin
  WriteLn('=== รายชื่อนักเรียนทั้งหมด ===');
  WriteLn(Format('%-10s %-20s %-10s %-8s',
    ['รหัส', 'ชื่อ-นามสกุล', 'ห้อง', 'GPA']));
  WriteLn(StringOfChar('-', 55));
  for i := 1 to NumStudents do
    WriteLn(Format('%-10s %-20s %-10s %-8.2f',
      [Students[i].ID,
       Students[i].Name + ' ' + Students[i].Surname,
       Students[i].Class,
       Students[i].GPA]));
end;

procedure DisplayTopStudents(Top: Integer);
var
  i, j   : Integer;
  Sorted : array[1..MAX_STUDENTS] of Integer;
  Temp   : Integer;
begin
  // สร้าง index array
  for i := 1 to NumStudents do Sorted[i] := i;
  
  // เรียงตาม GPA
  for i := 1 to NumStudents - 1 do
    for j := 1 to NumStudents - i do
      if Students[Sorted[j]].GPA < Students[Sorted[j+1]].GPA then
      begin
        Temp       := Sorted[j];
        Sorted[j]  := Sorted[j+1];
        Sorted[j+1] := Temp;
      end;
  
  if Top > NumStudents then Top := NumStudents;
  
  WriteLn('=== อันดับ ', Top, ' อันดับแรก ===');
  for i := 1 to Top do
    WriteLn(Format('%2d. %-20s GPA = %.2f',
      [i,
       Students[Sorted[i]].Name + ' ' + Students[Sorted[i]].Surname,
       Students[Sorted[i]].GPA]));
end;

var
  Idx : Integer;

begin
  NumStudents := 0;
  
  // เพิ่มนักเรียน
  AddStudent('S001', 'สมชาย', 'ใจดี', 'ม.6/1');
  AddStudent('S002', 'สมหญิง', 'รักเรียน', 'ม.6/1');
  AddStudent('S003', 'อนุชา', 'สมาร์ท', 'ม.6/2');
  AddStudent('S004', 'วิภา', 'มีสุข', 'ม.6/2');
  AddStudent('S005', 'ธนพล', 'เก่งมาก', 'ม.6/1');
  
  // เพิ่มคะแนน
  AddScore('S001', 'คณิตศาสตร์', 85);
  AddScore('S001', 'ภาษาไทย', 78);
  AddScore('S001', 'ภาษาอังกฤษ', 82);
  AddScore('S001', 'วิทยาศาสตร์', 90);
  AddScore('S001', 'สังคมศึกษา', 75);
  
  AddScore('S002', 'คณิตศาสตร์', 92);
  AddScore('S002', 'ภาษาไทย', 88);
  AddScore('S002', 'ภาษาอังกฤษ', 95);
  AddScore('S002', 'วิทยาศาสตร์', 87);
  AddScore('S002', 'สังคมศึกษา', 91);
  
  AddScore('S003', 'คณิตศาสตร์', 72);
  AddScore('S003', 'ภาษาไทย', 65);
  AddScore('S003', 'ภาษาอังกฤษ', 70);
  AddScore('S003', 'วิทยาศาสตร์', 68);
  AddScore('S003', 'สังคมศึกษา', 73);
  
  AddScore('S004', 'คณิตศาสตร์', 58);
  AddScore('S004', 'ภาษาไทย', 62);
  AddScore('S004', 'ภาษาอังกฤษ', 55);
  AddScore('S004', 'วิทยาศาสตร์', 60);
  AddScore('S004', 'สังคมศึกษา', 65);
  
  AddScore('S005', 'คณิตศาสตร์', 98);
  AddScore('S005', 'ภาษาไทย', 94);
  AddScore('S005', 'ภาษาอังกฤษ', 97);
  AddScore('S005', 'วิทยาศาสตร์', 99);
  AddScore('S005', 'สังคมศึกษา', 96);
  
  // แสดงรายชื่อทั้งหมด
  DisplayAllStudents;
  WriteLn;
  
  // แสดงข้อมูลนักเรียนคนเดียว
  Idx := FindStudent('S002');
  if Idx > 0 then DisplayStudent(Idx);
  WriteLn;
  
  // แสดง top 3
  DisplayTopStudents(3);
  
  ReadLn;
end.
```

---

## 11.9 โปรแกรมตัวอย่าง: Phonebook

```pascal
program Phonebook;

{$mode objfpc}{$H+}

uses SysUtils;

const
  MAX_CONTACTS = 200;

type
  TContactType = (ctPersonal, ctWork, ctFamily, ctOther);
  
  TPhone = record
    Number   : String[20];
    PhoneType: String[10]; // mobile, home, work
  end;
  
  TContact = record
    ID         : Integer;
    FirstName  : String[30];
    LastName   : String[30];
    Nickname   : String[20];
    ContactType: TContactType;
    Phones     : array[1..3] of TPhone;
    NumPhones  : Integer;
    Email      : String[80];
    Line       : String[30];
    Facebook   : String[50];
    Notes      : String[200];
    Favorite   : Boolean;
  end;

var
  Contacts    : array[1..MAX_CONTACTS] of TContact;
  NumContacts : Integer;
  NextID      : Integer;

function ContactTypeToStr(CT: TContactType): String;
begin
  case CT of
    ctPersonal: Result := 'ส่วนตัว';
    ctWork    : Result := 'งาน';
    ctFamily  : Result := 'ครอบครัว';
    ctOther   : Result := 'อื่นๆ';
  end;
end;

procedure AddContact(FirstName, LastName, Nickname: String;
                     CT: TContactType; Email: String);
begin
  if NumContacts >= MAX_CONTACTS then
  begin
    WriteLn('สมุดโทรศัพท์เต็มแล้ว');
    Exit;
  end;
  Inc(NumContacts);
  Inc(NextID);
  with Contacts[NumContacts] do
  begin
    ID          := NextID;
    FirstName   := FirstName;
    LastName    := LastName;
    Nickname    := Nickname;
    ContactType := CT;
    Email       := Email;
    NumPhones   := 0;
    Favorite    := False;
    Notes       := '';
    Line        := '';
    Facebook    := '';
  end;
end;

procedure AddPhone(ContactID: Integer; Number, PhoneType: String);
var
  i : Integer;
begin
  for i := 1 to NumContacts do
    if Contacts[i].ID = ContactID then
    begin
      if Contacts[i].NumPhones < 3 then
      begin
        Inc(Contacts[i].NumPhones);
        Contacts[i].Phones[Contacts[i].NumPhones].Number    := Number;
        Contacts[i].Phones[Contacts[i].NumPhones].PhoneType := PhoneType;
      end;
      Exit;
    end;
end;

function FindByName(SearchName: String): Integer;
var
  i : Integer;
begin
  SearchName := LowerCase(SearchName);
  Result     := -1;
  for i := 1 to NumContacts do
    if (Pos(SearchName, LowerCase(Contacts[i].FirstName)) > 0) or
       (Pos(SearchName, LowerCase(Contacts[i].LastName)) > 0) or
       (Pos(SearchName, LowerCase(Contacts[i].Nickname)) > 0) then
    begin
      Result := i;
      Exit;
    end;
end;

procedure SearchContacts(SearchName: String);
var
  i     : Integer;
  Found : Integer;
begin
  SearchName := LowerCase(SearchName);
  Found      := 0;
  
  WriteLn('ผลการค้นหา: "', SearchName, '"');
  WriteLn(StringOfChar('-', 60));
  
  for i := 1 to NumContacts do
    if (Pos(SearchName, LowerCase(Contacts[i].FirstName)) > 0) or
       (Pos(SearchName, LowerCase(Contacts[i].LastName)) > 0) or
       (Pos(SearchName, LowerCase(Contacts[i].Nickname)) > 0) then
    begin
      Inc(Found);
      with Contacts[i] do
      begin
        Write(Format('%-5d %-15s %-15s', [ID, FirstName, LastName]));
        if NumPhones > 0 then
          Write(' ', Phones[1].Number)
        else
          Write(' (ไม่มีเบอร์)');
        WriteLn(' [', ContactTypeToStr(ContactType), ']');
      end;
    end;
  
  if Found = 0 then WriteLn('ไม่พบรายชื่อ')
  else WriteLn('พบทั้งหมด ', Found, ' รายการ');
end;

procedure DisplayContact(Idx: Integer);
var
  i : Integer;
begin
  if (Idx < 1) or (Idx > NumContacts) then Exit;
  with Contacts[Idx] do
  begin
    WriteLn('==========================================');
    WriteLn('ID       : ', ID);
    WriteLn('ชื่อ     : ', FirstName, ' ', LastName);
    if Nickname <> '' then WriteLn('ชื่อเล่น : ', Nickname);
    WriteLn('ประเภท   : ', ContactTypeToStr(ContactType));
    WriteLn('');
    WriteLn('--- เบอร์โทรศัพท์ ---');
    for i := 1 to NumPhones do
      WriteLn('  ', Phones[i].PhoneType, ': ', Phones[i].Number);
    if NumPhones = 0 then WriteLn('  ไม่มีเบอร์');
    WriteLn('');
    if Email <> '' then WriteLn('อีเมล    : ', Email);
    if Line <> '' then WriteLn('LINE     : ', Line);
    if Facebook <> '' then WriteLn('Facebook : ', Facebook);
    if Notes <> '' then WriteLn('หมายเหตุ : ', Notes);
    if Favorite then WriteLn('★ รายการโปรด');
    WriteLn('==========================================');
  end;
end;

procedure DisplayAll;
var
  i : Integer;
begin
  WriteLn('=== สมุดโทรศัพท์ (', NumContacts, ' รายการ) ===');
  WriteLn(Format('%-5s %-15s %-15s %-15s %-10s',
    ['ID', 'ชื่อ', 'นามสกุล', 'เบอร์แรก', 'ประเภท']));
  WriteLn(StringOfChar('-', 65));
  for i := 1 to NumContacts do
    with Contacts[i] do
    begin
      Write(Format('%-5d %-15s %-15s ', [ID, FirstName, LastName]));
      if NumPhones > 0 then
        Write(Format('%-15s ', [Phones[1].Number]))
      else
        Write(Format('%-15s ', ['(ไม่มีเบอร์)']));
      WriteLn(ContactTypeToStr(ContactType));
    end;
end;

begin
  NumContacts := 0;
  NextID      := 0;
  
  // เพิ่มรายชื่อ
  AddContact('สมชาย', 'ใจดี', 'ชาย', ctPersonal, 'somchai@gmail.com');
  AddPhone(1, '081-234-5678', 'mobile');
  AddPhone(1, '02-456-7890', 'home');
  
  AddContact('สมหญิง', 'รักดี', 'หญิง', ctPersonal, 'somying@gmail.com');
  AddPhone(2, '089-876-5432', 'mobile');
  
  AddContact('ผู้จัดการ', 'สมศักดิ์', 'ผจก', ctWork, 'manager@company.com');
  AddPhone(3, '085-111-2222', 'mobile');
  AddPhone(3, '02-333-4444', 'work');
  Contacts[3].Line := '@manager_id';
  
  AddContact('แม่', 'สมใจ', 'แม่', ctFamily, '');
  AddPhone(4, '02-111-9999', 'home');
  AddPhone(4, '082-333-4444', 'mobile');
  Contacts[4].Favorite := True;
  
  AddContact('พ่อ', 'สมพงษ์', 'พ่อ', ctFamily, '');
  AddPhone(5, '083-555-6666', 'mobile');
  Contacts[5].Favorite := True;
  
  // แสดงทั้งหมด
  DisplayAll;
  WriteLn;
  
  // ค้นหา
  SearchContacts('สม');
  WriteLn;
  
  // แสดงรายละเอียด
  DisplayContact(3);
  
  ReadLn;
end.
```

---

## 11.10 โปรแกรมตัวอย่าง: Library Catalog

```pascal
program LibraryCatalog;

{$mode objfpc}{$H+}

uses SysUtils;

const
  MAX_BOOKS    = 500;
  MAX_MEMBERS  = 200;
  MAX_BORROWED = 50;

type
  TBookStatus = (bsAvailable, bsBorrowed, bsReserved, bsLost);
  
  TBook = record
    ISBN      : String[13];
    Title     : String[80];
    Author    : String[60];
    Publisher : String[50];
    Year      : Integer;
    Category  : String[30];
    Location  : String[10]; // shelf code
    Status    : TBookStatus;
    BorrowedBy: String[10]; // member ID
    DueDate   : String[10];
  end;
  
  TMember = record
    MemberID   : String[10];
    Name       : String[60];
    Phone      : String[15];
    Email      : String[60];
    JoinDate   : String[10];
    NumBorrowed: Integer;
    IsActive   : Boolean;
  end;

var
  Books      : array[1..MAX_BOOKS] of TBook;
  Members    : array[1..MAX_MEMBERS] of TMember;
  NumBooks   : Integer;
  NumMembers : Integer;

function StatusToStr(S: TBookStatus): String;
begin
  case S of
    bsAvailable: Result := 'ว่าง';
    bsBorrowed : Result := 'ถูกยืม';
    bsReserved : Result := 'ถูกจอง';
    bsLost     : Result := 'สูญหาย';
  end;
end;

procedure AddBook(ISBN, Title, Author, Publisher: String;
                  Year: Integer; Category, Location: String);
begin
  if NumBooks >= MAX_BOOKS then Exit;
  Inc(NumBooks);
  with Books[NumBooks] do
  begin
    ISBN       := ISBN;
    Title      := Title;
    Author     := Author;
    Publisher  := Publisher;
    Year       := Year;
    Category   := Category;
    Location   := Location;
    Status     := bsAvailable;
    BorrowedBy := '';
    DueDate    := '';
  end;
end;

procedure AddMember(MemberID, Name, Phone, Email: String);
begin
  if NumMembers >= MAX_MEMBERS then Exit;
  Inc(NumMembers);
  with Members[NumMembers] do
  begin
    MemberID    := MemberID;
    Name        := Name;
    Phone       := Phone;
    Email       := Email;
    JoinDate    := '01/01/2024';
    NumBorrowed := 0;
    IsActive    := True;
  end;
end;

function FindBook(ISBN: String): Integer;
var i: Integer;
begin
  Result := -1;
  for i := 1 to NumBooks do
    if Books[i].ISBN = ISBN then begin Result := i; Exit; end;
end;

function FindMember(MemberID: String): Integer;
var i: Integer;
begin
  Result := -1;
  for i := 1 to NumMembers do
    if Members[i].MemberID = MemberID then begin Result := i; Exit; end;
end;

procedure BorrowBook(ISBN, MemberID, DueDate: String);
var
  BookIdx, MemberIdx : Integer;
begin
  BookIdx   := FindBook(ISBN);
  MemberIdx := FindMember(MemberID);
  
  if BookIdx < 0 then begin WriteLn('ไม่พบหนังสือ ISBN: ', ISBN); Exit; end;
  if MemberIdx < 0 then begin WriteLn('ไม่พบสมาชิก ID: ', MemberID); Exit; end;
  if Books[BookIdx].Status <> bsAvailable then
    begin WriteLn('หนังสือไม่ว่าง สถานะ: ', StatusToStr(Books[BookIdx].Status)); Exit; end;
  if Members[MemberIdx].NumBorrowed >= 5 then
    begin WriteLn('สมาชิกยืมครบจำนวนที่กำหนดแล้ว (5 เล่ม)'); Exit; end;
  
  Books[BookIdx].Status     := bsBorrowed;
  Books[BookIdx].BorrowedBy := MemberID;
  Books[BookIdx].DueDate    := DueDate;
  Inc(Members[MemberIdx].NumBorrowed);
  
  WriteLn('ยืมสำเร็จ: "', Books[BookIdx].Title, '" กำหนดคืน: ', DueDate);
end;

procedure ReturnBook(ISBN: String);
var
  BookIdx, MemberIdx : Integer;
begin
  BookIdx := FindBook(ISBN);
  if BookIdx < 0 then begin WriteLn('ไม่พบหนังสือ'); Exit; end;
  if Books[BookIdx].Status <> bsBorrowed then
    begin WriteLn('หนังสือนี้ไม่ได้ถูกยืม'); Exit; end;
  
  MemberIdx := FindMember(Books[BookIdx].BorrowedBy);
  if MemberIdx > 0 then
    Dec(Members[MemberIdx].NumBorrowed);
  
  Books[BookIdx].Status     := bsAvailable;
  Books[BookIdx].BorrowedBy := '';
  Books[BookIdx].DueDate    := '';
  
  WriteLn('คืนสำเร็จ: "', Books[BookIdx].Title, '"');
end;

procedure SearchBooks(Keyword: String);
var
  i     : Integer;
  Found : Integer;
begin
  Found   := 0;
  Keyword := LowerCase(Keyword);
  
  WriteLn('ผลการค้นหา: "', Keyword, '"');
  WriteLn(Format('%-14s %-30s %-20s %-8s',
    ['ISBN', 'ชื่อหนังสือ', 'ผู้แต่ง', 'สถานะ']));
  WriteLn(StringOfChar('-', 76));
  
  for i := 1 to NumBooks do
    if (Pos(Keyword, LowerCase(Books[i].Title)) > 0) or
       (Pos(Keyword, LowerCase(Books[i].Author)) > 0) or
       (Pos(Keyword, LowerCase(Books[i].Category)) > 0) then
    begin
      Inc(Found);
      WriteLn(Format('%-14s %-30s %-20s %-8s',
        [Books[i].ISBN,
         Copy(Books[i].Title, 1, 28),
         Copy(Books[i].Author, 1, 18),
         StatusToStr(Books[i].Status)]));
    end;
  
  WriteLn(StringOfChar('-', 76));
  WriteLn('พบ ', Found, ' รายการ');
end;

procedure DisplayBooksByCategory(Category: String);
var
  i     : Integer;
  Found : Integer;
begin
  Found := 0;
  WriteLn('=== หนังสือหมวด: ', Category, ' ===');
  
  for i := 1 to NumBooks do
    if Books[i].Category = Category then
    begin
      Inc(Found);
      WriteLn(Found, '. ', Books[i].Title);
      WriteLn('   ผู้แต่ง: ', Books[i].Author, ' | ปี: ', Books[i].Year);
      WriteLn('   สถานที่: ', Books[i].Location, ' | สถานะ: ', StatusToStr(Books[i].Status));
    end;
  
  WriteLn('รวม: ', Found, ' เล่ม');
end;

begin
  NumBooks   := 0;
  NumMembers := 0;
  
  // เพิ่มหนังสือ
  AddBook('9786160000001', 'การเขียนโปรแกรม Pascal', 'สมชาย ใจดี',
    'สำนักพิมพ์ IT', 2023, 'คอมพิวเตอร์', 'A1-01');
  AddBook('9786160000002', 'โครงสร้างข้อมูลและอัลกอริทึม', 'วิภา รักวิชา',
    'สำนักพิมพ์ IT', 2022, 'คอมพิวเตอร์', 'A1-02');
  AddBook('9786160000003', 'ฐานข้อมูลเบื้องต้น', 'อนุชา สมาร์ท',
    'สำนักพิมพ์ IT', 2023, 'คอมพิวเตอร์', 'A1-03');
  AddBook('9786160000004', 'คณิตศาสตร์ ม.ปลาย', 'ธนพล เก่งมาก',
    'สำนักพิมพ์การศึกษา', 2024, 'คณิตศาสตร์', 'B2-01');
  AddBook('9786160000005', 'ฟิสิกส์ทั่วไป', 'นภาพร ฉลาด',
    'สำนักพิมพ์วิทย์', 2023, 'วิทยาศาสตร์', 'C3-01');
  
  // เพิ่มสมาชิก
  AddMember('M001', 'สมชาย รักอ่าน', '081-111-2222', 'somchai@mail.com');
  AddMember('M002', 'สมหญิง ชอบเรียน', '089-333-4444', 'somying@mail.com');
  AddMember('M003', 'อนุชา หัวดี', '085-555-6666', 'anucha@mail.com');
  
  WriteLn('=== ระบบห้องสมุด ===');
  WriteLn;
  
  // ยืมหนังสือ
  WriteLn('--- การยืมหนังสือ ---');
  BorrowBook('9786160000001', 'M001', '15/02/2024');
  BorrowBook('9786160000002', 'M001', '15/02/2024');
  BorrowBook('9786160000003', 'M002', '20/02/2024');
  WriteLn;
  
  // ค้นหา
  SearchBooks('pascal');
  WriteLn;
  
  // แสดงตามหมวด
  DisplayBooksByCategory('คอมพิวเตอร์');
  WriteLn;
  
  // คืนหนังสือ
  WriteLn('--- การคืนหนังสือ ---');
  ReturnBook('9786160000001');
  WriteLn;
  
  // ค้นหาอีกครั้งหลังคืน
  SearchBooks('pascal');
  
  ReadLn;
end.
```

---

## 11.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1
สร้าง record `TRectangle` ที่มี fields: ความกว้าง ความสูง สี
เขียนฟังก์ชันคำนวณพื้นที่และเส้นรอบรูป

### แบบฝึกหัดที่ 2
สร้าง record `TStudent` ที่เก็บข้อมูลนักเรียน 10 คน
เขียนโปรแกรมหาคะแนนสูงสุด ต่ำสุด และค่าเฉลี่ย

### แบบฝึกหัดที่ 3
สร้าง record `TDate` แล้วเขียนฟังก์ชัน:
- `IsValidDate` ตรวจสอบว่าวันที่ถูกต้อง
- `DateToString` แปลงเป็น string รูปแบบ DD/MM/YYYY
- `DaysBetween` คำนวณจำนวนวันระหว่างสองวัน

### แบบฝึกหัดที่ 4
สร้าง Nested record สำหรับระบบพนักงาน:
- `TDepartment` (ชื่อแผนก, หัวหน้า, งบประมาณ)
- `TEmployee` (รหัส, ชื่อ, ตำแหน่ง, เงินเดือน, แผนก)
เขียนโปรแกรมแสดงข้อมูลพนักงานและคำนวณค่าใช้จ่ายรวมแต่ละแผนก

### แบบฝึกหัดที่ 5
สร้าง Array of Records สำหรับ inventory สินค้า 20 รายการ
เขียนฟังก์ชัน:
- เพิ่มสินค้า
- ลดสินค้า
- แสดงสินค้าที่ใกล้หมด (stock < 5)
- มูลค่าสินค้าทั้งหมด

### แบบฝึกหัดที่ 6
เขียนโปรแกรม `with...do` กับ record `TTriangle` ที่มีด้าน 3 ด้าน
คำนวณพื้นที่ด้วยสูตร Heron's formula

### แบบฝึกหัดที่ 7
สร้าง Variant record `TShape` ที่รองรับ:
- วงกลม (รัศมี)
- สี่เหลี่ยมจัตุรัส (ด้าน)
- สามเหลี่ยม (ฐาน, สูง)
- วงรี (แกน a, แกน b)
เขียนฟังก์ชันคำนวณพื้นที่สำหรับแต่ละรูป

### แบบฝึกหัดที่ 8
สร้างระบบสต็อกสินค้าร้านขายยา
- `TMedicine` record (รหัส, ชื่อยา, ประเภท, ราคา, stock, วันหมดอายุ)
- ค้นหายาตามชื่อ
- แสดงยาที่หมดอายุ
- แสดงยาที่ stock เหลือน้อย

### แบบฝึกหัดที่ 9
เขียนโปรแกรม phonebook อย่างง่าย:
- `TContact` record
- เพิ่ม/ลบ/แก้ไขรายชื่อ
- ค้นหาตามชื่อหรือเบอร์โทร
- แสดงรายชื่อทั้งหมดเรียงตาม A-Z

### แบบฝึกหัดที่ 10
สร้าง record `TMatrix2x2` ที่เก็บเมทริกซ์ 2x2
เขียนฟังก์ชัน:
- บวกเมทริกซ์
- คูณเมทริกซ์
- หา Determinant
- หา Inverse

### แบบฝึกหัดที่ 11
สร้างระบบจัดการรายการอาหาร (Menu):
- `TMenuItem` record (รหัส, ชื่อ, ราคา, หมวดหมู่, กำลังขาย)
- `TOrder` record (รหัสออเดอร์, รายการที่สั่ง, จำนวน, ราคารวม)
คำนวณยอดรวมออเดอร์

### แบบฝึกหัดที่ 12
สร้าง record `TRGB` และ `THSV` สำหรับสี
เขียนฟังก์ชันแปลง RGB <-> HSV

### แบบฝึกหัดที่ 13
สร้างระบบ quiz:
- `TQuestion` record (คำถาม, ตัวเลือก 4 ข้อ, เฉลย, คะแนน)
- โปรแกรมถามคำถาม 5 ข้อ
- คำนวณคะแนนและแสดงผล

### แบบฝึกหัดที่ 14
สร้าง record `TComplex` สำหรับจำนวนเชิงซ้อน
เขียนฟังก์ชัน:
- บวก ลบ คูณ หาร
- Modulus (|z|)
- Conjugate

### แบบฝึกหัดที่ 15
สร้างระบบ tournament บาสเกตบอล:
- `TTeam` record (ชื่อทีม, แพ้, ชนะ, เสมอ, คะแนน)
- `TMatch` record (ทีมA, ทีมB, คะแนนA, คะแนนB)
อัปเดตตารางคะแนนจากผลการแข่งขัน

---

## สรุป

| หัวข้อ | คำอธิบาย |
|--------|---------|
| `type TName = record ... end` | ประกาศ record type |
| `Var.Field` | เข้าถึง field ด้วย dot notation |
| Nested records | record ซ้อน record |
| `array of records` | เก็บ records หลายรายการ |
| `with Var do` | ลดการพิมพ์ชื่อ record |
| Variant records | `case` ใน record เพื่อประหยัด memory |
| `var` parameter | ส่ง record by reference เพื่อแก้ไขได้ |

Records เป็นพื้นฐานสำคัญก่อนเรียน Object-Oriented Programming เพราะมีแนวคิดคล้ายกับ `class` แต่ง่ายกว่า
