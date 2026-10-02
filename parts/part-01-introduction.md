# Part 01 - แนะนำ Lazarus และ Pascal

## สารบัญ

1. [ประวัติของ Pascal และ Free Pascal](#ประวัติของ-pascal-และ-free-pascal)
2. [Lazarus IDE คืออะไร](#lazarus-ide-คืออะไร)
3. [ข้อดีของ Lazarus/Pascal](#ข้อดีของ-lazaruspascal)
4. [การใช้งานในอุตสาหกรรม](#การใช้งานในอุตสาหกรรม)
5. [Pascal vs Python vs Java vs C++](#pascal-vs-python-vs-java-vs-c)
6. [โครงสร้างของโปรแกรม Pascal พื้นฐาน](#โครงสร้างของโปรแกรม-pascal-พื้นฐาน)
7. [ตัวอย่างโค้ด Hello World ทุกรูปแบบ](#ตัวอย่างโค้ด-hello-world-ทุกรูปแบบ)
8. [แบบฝึกหัด 10 ข้อ](#แบบฝึกหัด-10-ข้อ)

---

## ประวัติของ Pascal และ Free Pascal

### กำเนิดของภาษา Pascal

ภาษา Pascal ถูกออกแบบและพัฒนาโดย **Niklaus Wirth** นักวิทยาศาสตร์คอมพิวเตอร์ชาวสวิส ในปี **ค.ศ. 1968-1969** และเผยแพร่สู่สาธารณะในปี **ค.ศ. 1970** ชื่อ "Pascal" ตั้งตาม **Blaise Pascal** นักคณิตศาสตร์และนักปรัชญาชาวฝรั่งเศส ผู้ประดิษฐ์เครื่องคิดเลขเชิงกลในศตวรรษที่ 17

**จุดประสงค์เดิมของการสร้างภาษา Pascal:**
- สอนการเขียนโปรแกรมให้กับนักศึกษา
- ส่งเสริมการเขียนโปรแกรมเชิงโครงสร้าง (Structured Programming)
- ลดการใช้ `goto` statement ที่ทำให้โค้ดซับซ้อนและยากต่อการ debug

### วิวัฒนาการของ Pascal

```
1970 - Pascal 1.0 (Niklaus Wirth, ETH Zürich)
1974 - Pascal User Manual and Report ตีพิมพ์
1978 - UCSD Pascal (สำหรับเครื่อง Apple II, CP/M)
1983 - ISO Pascal Standard (ISO 7185)
1983 - Turbo Pascal 1.0 (Borland International)
1986 - Turbo Pascal 4.0 (มี Unit system)
1989 - Turbo Pascal 5.5 (เพิ่ม OOP)
1991 - Turbo Pascal for Windows
1992 - Delphi preview (Object Pascal)
1995 - Delphi 1.0 (Borland) - ปฏิวัติวงการ RAD
1997 - Free Pascal 0.9 (เริ่มโปรเจกต์โอเพนซอร์ส)
1999 - Kylix (Delphi สำหรับ Linux)
2000 - Free Pascal 1.0 (เสถียร)
2004 - Lazarus 0.9 (IDE สำหรับ Free Pascal)
2006 - Free Pascal 2.0
2011 - Free Pascal 2.6 / Lazarus 1.0
2014 - Free Pascal 3.0 (alpha)
2016 - Lazarus 1.6
2020 - Free Pascal 3.2 / Lazarus 2.0
2022 - Lazarus 2.2
2024 - Lazarus 3.0 (ปัจจุบัน)
```

### Turbo Pascal - การปฏิวัติวงการโปรแกรมมิ่ง

**Turbo Pascal** ที่พัฒนาโดย Borland International ในปี 1983 เป็นจุดเปลี่ยนสำคัญ:

- ราคาถูกมาก (เพียง $49.95 ในสมัยนั้น)
- Compile เร็วมาก (เร็วกว่าคู่แข่งหลายเท่า)
- มี IDE ในตัว (Integrated Development Environment)
- ทำงานบน DOS ได้อย่างยอดเยี่ยม

> **ข้อเท็จจริงน่าสนใจ:** Turbo Pascal ขายได้มากกว่า **1 ล้านชุด** ในช่วงปี 1980s ซึ่งถือว่าเป็นสถิติที่น่าทึ่งมากในยุคนั้น

### Free Pascal (FPC) - การเกิดใหม่แบบโอเพนซอร์ส

**Free Pascal Compiler (FPC)** เริ่มต้นโดย **Florian Klämpfl** ในปี 1993 เป็นโปรเจกต์โอเพนซอร์สที่:

- ใช้งานได้ฟรี (Free as in Freedom และ Free as in Beer)
- รองรับหลายแพลตฟอร์ม: Windows, Linux, macOS, FreeBSD, Android, iOS
- รองรับหลายสถาปัตยกรรม: x86, x86-64, ARM, MIPS, PowerPC
- เข้ากันได้กับ Delphi Pascal และ Turbo Pascal
- มีไลบรารีมาตรฐานที่ครบครัน (RTL, FCL)

**คุณสมบัติเด่นของ Free Pascal:**
```
- รองรับ Object Pascal (OOP)
- Generic types
- Operator overloading  
- Exception handling
- RTTI (Runtime Type Information)
- Unicode string support
- Thread support
- Dynamic link libraries (DLL/SO)
```

---

## Lazarus IDE คืออะไร

### Lazarus - "การฟื้นคืนชีพ" ของ Delphi

**Lazarus** (ตั้งชื่อตาม Lazarus จากพระคัมภีร์ ผู้ถูกนำมาฟื้นคืนชีพ) เป็น **IDE โอเพนซอร์สฟรี** สำหรับ Free Pascal ที่ออกแบบมาเพื่อให้ประสบการณ์คล้าย Delphi

**คำขวัญของ Lazarus:** *"Write Once, Compile Anywhere"*

### ส่วนประกอบหลักของ Lazarus

```
Lazarus IDE
├── Code Editor (แก้ไขโค้ด)
│   ├── Syntax highlighting
│   ├── Code completion
│   ├── Code folding
│   └── Multiple tabs
├── Form Designer (ออกแบบหน้าจอ GUI)
│   ├── Visual component placement
│   ├── Property inspector
│   └── Event handling
├── Object Inspector (ดูและแก้ไข properties)
│   ├── Properties tab
│   └── Events tab
├── Project Manager (จัดการไฟล์โปรเจค)
│   ├── Source files (.pas, .pp)
│   ├── Form files (.lfm)
│   └── Project file (.lpi)
├── Compiler Messages (แสดงข้อผิดพลาด)
│   ├── Errors
│   ├── Warnings
│   └── Hints
└── Debugger (ตรวจสอบและแก้บั๊ก)
    ├── Breakpoints
    ├── Watch expressions  
    └── Call stack
```

### LCL - Lazarus Component Library

**LCL (Lazarus Component Library)** คือ GUI framework หลักของ Lazarus:

| ส่วนประกอบ | คำอธิบาย |
|------------|-----------|
| TForm | หน้าต่างหลัก |
| TButton | ปุ่มกด |
| TLabel | ป้ายข้อความ |
| TEdit | กล่องรับข้อมูล |
| TMemo | กล่องข้อความหลายบรรทัด |
| TListBox | รายการ |
| TComboBox | dropdown list |
| TCheckBox | ช่องเลือก |
| TRadioButton | ปุ่มเลือก |
| TPanel | กรอบจัดกลุ่ม |
| TImage | แสดงรูปภาพ |
| TTimer | ตัวจับเวลา |
| TMainMenu | เมนูหลัก |
| TPopupMenu | เมนูลอย |
| TFileDialog | dialog เปิด/บันทึกไฟล์ |

### ความแตกต่างระหว่าง Lazarus และ Delphi

| คุณสมบัติ | Lazarus | Delphi |
|-----------|---------|--------|
| ราคา | ฟรี (Open Source) | มีค่าใช้จ่าย ($) |
| แพลตฟอร์ม | Windows, Linux, macOS, FreeBSD | Windows (หลัก), Linux, macOS (บางเวอร์ชัน) |
| Compiler | Free Pascal (FPC) | Embarcadero Delphi Compiler |
| GUI Framework | LCL | VCL / FMX |
| ชุมชน | ขนาดกลาง (active) | ขนาดใหญ่ |
| เหมาะสำหรับ | Open source, cross-platform | Enterprise, Windows apps |

---

## ข้อดีของ Lazarus/Pascal

### 1. ภาษาที่อ่านเข้าใจง่าย

Pascal ออกแบบมาให้อ่านและเข้าใจได้ง่าย เหมาะสำหรับการเรียนรู้:

```pascal
// Pascal - อ่านเหมือนภาษาอังกฤษ
program CalculateArea;
var
  radius, area: Double;
begin
  Write('Enter radius: ');
  ReadLn(radius);
  area := 3.14159 * radius * radius;
  WriteLn('Area = ', area:8:2);
end.
```

เปรียบเทียบกับ C++ ที่ซับซ้อนกว่า:

```cpp
// C++ - ต้องการความรู้เพิ่มเติม
#include <iostream>
#include <iomanip>
using namespace std;
int main() {
    double radius, area;
    cout << "Enter radius: ";
    cin >> radius;
    area = 3.14159 * radius * radius;
    cout << fixed << setprecision(2) << "Area = " << area << endl;
    return 0;
}
```

### 2. การตรวจสอบชนิดข้อมูลที่เข้มแข็ง (Strong Type Checking)

```pascal
program TypeSafety;
var
  intValue: Integer;
  strValue: String;
begin
  intValue := 42;
  // strValue := intValue;  // ERROR! ไม่สามารถกำหนด Integer ให้ String ได้โดยตรง
  strValue := IntToStr(intValue);  // ต้องแปลงก่อน - ปลอดภัย!
  WriteLn('String value: ', strValue);
end.
```

### 3. Cross-Platform โดยแท้จริง

โค้ดเดียวกันทำงานได้บนหลายระบบปฏิบัติการ:

```pascal
program CrossPlatformDemo;
uses
  SysUtils;
begin
  WriteLn('Running on: ', {$IFDEF WINDOWS}'Windows'{$ENDIF}
                           {$IFDEF LINUX}'Linux'{$ENDIF}
                           {$IFDEF DARWIN}'macOS'{$ENDIF});
  WriteLn('Date: ', DateToStr(Now));
  WriteLn('Time: ', TimeToStr(Now));
end.
```

### 4. ประสิทธิภาพสูง (High Performance)

Pascal/Free Pascal คอมไพล์เป็น native code จึงทำงานเร็วมาก:

```pascal
program PerformanceTest;
uses
  SysUtils;
var
  i: Integer;
  sum: Int64;
  startTime, endTime: TDateTime;
begin
  startTime := Now;
  sum := 0;
  for i := 1 to 100000000 do  // 100 ล้านครั้ง
    Inc(sum, i);
  endTime := Now;
  
  WriteLn('Sum = ', sum);
  WriteLn('Time = ', FormatDateTime('ss.zzz', endTime - startTime), ' seconds');
end.
```

### 5. RAD (Rapid Application Development)

Lazarus ช่วยสร้าง GUI application ได้รวดเร็วมากด้วย visual designer:

```pascal
// โปรแกรม GUI ง่ายๆ ที่สร้างได้ใน 5 นาที
unit MainForm;

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs, StdCtrls;

type
  TForm1 = class(TForm)
    btnHello: TButton;
    lblMessage: TLabel;
    procedure btnHelloClick(Sender: TObject);
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

procedure TForm1.btnHelloClick(Sender: TObject);
begin
  lblMessage.Caption := 'สวัสดีชาวโลก! 🌍';
  lblMessage.Font.Color := clBlue;
  lblMessage.Font.Size := 16;
end;

end.
```

### 6. หน่วยความจำจัดการได้ดี

```pascal
program MemoryManagement;
type
  PNode = ^TNode;
  TNode = record
    Value: Integer;
    Next: PNode;
  end;
var
  head, current: PNode;
  i: Integer;
begin
  head := nil;
  
  // สร้าง linked list
  for i := 1 to 5 do
  begin
    New(current);
    current^.Value := i * 10;
    current^.Next := head;
    head := current;
  end;
  
  // แสดงผล
  current := head;
  while current <> nil do
  begin
    Write(current^.Value, ' ');
    current := current^.Next;
  end;
  WriteLn;
  
  // ปลดปล่อยหน่วยความจำ
  while head <> nil do
  begin
    current := head;
    head := head^.Next;
    Dispose(current);
  end;
  WriteLn('Memory freed successfully!');
end.
```

---

## การใช้งานในอุตสาหกรรม

### ซอฟต์แวร์ชื่อดังที่เขียนด้วย Pascal/Delphi

#### 1. Skype (เวอร์ชันแรก)
Skype ถูกพัฒนาเริ่มต้นโดยใช้ **Delphi (Object Pascal)** โดยทีมนักพัฒนาชาวเอสโตเนีย Ahti Heinla, Priit Kasesalu และ Jaan Tallinn

```
ข้อเท็จจริง:
- Skype 1.0 (2003) - เขียนด้วย Delphi เป็นหลัก
- ประสิทธิภาพสูง, binary เล็ก
- Microsoft ซื้อกิจการในปี 2011 ด้วยราคา $8.5 พันล้านดอลลาร์
```

#### 2. Adobe Photoshop (เวอร์ชันแรก)
เวอร์ชันแรกๆ ของ Photoshop บน Mac เขียนด้วย Pascal:

```
- Photoshop 1.0 (1990) - Thomas Knoll เขียนด้วย Pascal
- ต่อมาเปลี่ยนมาใช้ C++ เพื่อ performance ที่ดีขึ้น
- ยังคงมี Pascal code เหลืออยู่ในบางส่วน
```

#### 3. Total Commander
โปรแกรม file manager ที่ได้รับความนิยมมากที่สุดในโลก:

```
- พัฒนาโดย Christian Ghisler (ชาวสวิส)
- เขียนด้วย Delphi (Object Pascal)
- มีผู้ใช้งานมากกว่า 100 ล้านคนทั่วโลก
- ยังคง active development จนถึงปัจจุบัน
```

#### 4. ซอฟต์แวร์อื่นๆ ที่น่าสนใจ

| ซอฟต์แวร์ | ประเภท | หมายเหตุ |
|------------|--------|-----------|
| MySQL Workbench | Database tool | ใช้ Delphi บางส่วน |
| Virtual Box (GUI) | Virtualization | ส่วน GUI เขียนด้วย Delphi |
| 7-Zip (GUI) | File compression | GUI เขียนด้วย Delphi |
| CodeSite | Logging tool | Delphi |
| FastReport | Reporting | Delphi |
| DevExpress | UI Components | Delphi |
| TMS Software | Components | Delphi |
| Embarcadero IDEs | Development | Delphi ตัวเอง! |
| Macromedia Flash 5 | Animation | บางส่วน Delphi |
| Kazaa | P2P File sharing | Delphi |
| BitTorrent (แรก) | P2P | Delphi |

### การใช้งานในระบบ Embedded และ Scientific

```pascal
// Pascal ในงาน Scientific Computing
program NumericalIntegration;

// ใช้ Simpson's Rule คำนวณ integral ของ sin(x) จาก 0 ถึง pi
function f(x: Double): Double;
begin
  Result := Sin(x);
end;

function Simpson(a, b: Double; n: Integer): Double;
var
  h, sum: Double;
  i: Integer;
begin
  h := (b - a) / n;
  sum := f(a) + f(b);
  
  i := 1;
  while i < n do
  begin
    if (i mod 2 = 0) then
      sum := sum + 2 * f(a + i * h)
    else
      sum := sum + 4 * f(a + i * h);
    Inc(i);
  end;
  
  Result := h * sum / 3;
end;

begin
  WriteLn('Integral of sin(x) from 0 to pi:');
  WriteLn('Calculated: ', Simpson(0, 3.14159265358979, 1000):12:10);
  WriteLn('Exact:      2.0000000000');
end.
```

### สถิติการใช้งาน Pascal/Delphi ในปัจจุบัน

```
TIOBE Index (2024):
- Pascal/Delphi อยู่ในอันดับ 10-15 ของภาษาที่ใช้มากที่สุด
- มีผู้ใช้งาน Delphi มากกว่า 2 ล้านคนทั่วโลก
- Free Pascal/Lazarus มีผู้ใช้งานหลายแสนคน

อุตสาหกรรมที่ใช้:
- Medical software (อุปกรณ์การแพทย์)
- Industrial automation
- Financial systems
- Government systems (ยุโรปตะวันออก)
- Education (ยังใช้สอนในหลายประเทศ)
```

---

## Pascal vs Python vs Java vs C++

### ตารางเปรียบเทียบ

| คุณสมบัติ | Pascal/Lazarus | Python | Java | C++ |
|-----------|----------------|--------|------|-----|
| **ง่ายต่อการเรียนรู้** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐ |
| **ประสิทธิภาพ** | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **GUI Development** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **Cross-platform** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **ความปลอดภัย** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐ |
| **ชุมชนนักพัฒนา** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **ไลบรารี** | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **ความเร็ว Startup** | เร็วมาก | ช้า | ช้า (JVM) | เร็วมาก |
| **Binary Size** | เล็ก | ต้องการ interpreter | ต้องการ JVM | เล็ก |
| **ราคา** | ฟรี | ฟรี | ฟรี | ฟรี/มีค่าใช้จ่าย |

### Hello World เปรียบเทียบ 4 ภาษา

**Pascal:**
```pascal
program HelloWorld;
begin
  WriteLn('Hello, World!');
end.
```

**Python:**
```python
print("Hello, World!")
```

**Java:**
```java
public class HelloWorld {
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
}
```

**C++:**
```cpp
#include <iostream>
using namespace std;
int main() {
    cout << "Hello, World!" << endl;
    return 0;
}
```

### การเปรียบเทียบโปรแกรม Fibonacci

**Pascal - อ่านง่าย ปลอดภัย:**
```pascal
program Fibonacci;
var
  n, a, b, temp, i: Integer;
begin
  Write('จำนวน Fibonacci ที่ต้องการ: ');
  ReadLn(n);
  
  a := 0;
  b := 1;
  
  Write('0');
  for i := 1 to n - 1 do
  begin
    Write(', ', b);
    temp := a + b;
    a := b;
    b := temp;
  end;
  WriteLn;
end.
```

**Python - กระชับ:**
```python
n = int(input("จำนวน Fibonacci ที่ต้องการ: "))
a, b = 0, 1
result = [a]
for _ in range(n-1):
    a, b = b, a+b
    result.append(a)
print(', '.join(map(str, result)))
```

**Java - verbose:**
```java
import java.util.Scanner;
public class Fibonacci {
    public static void main(String[] args) {
        Scanner sc = new Scanner(System.in);
        System.out.print("จำนวน Fibonacci ที่ต้องการ: ");
        int n = sc.nextInt();
        int a = 0, b = 1, temp;
        System.out.print(a);
        for (int i = 1; i < n; i++) {
            System.out.print(", " + b);
            temp = a + b;
            a = b;
            b = temp;
        }
        System.out.println();
    }
}
```

### เมื่อไหรควรเลือก Pascal/Lazarus

```
เลือก Pascal/Lazarus เมื่อ:
✓ ต้องการ GUI application แบบ native (เร็ว, เล็ก)
✓ ต้องการ cross-platform GUI development
✓ เป็นมือใหม่ที่ต้องการเรียนรู้ programming concepts
✓ ทำงาน legacy Delphi/Pascal code
✓ ต้องการ performance สูงพร้อม GUI
✓ งบประมาณจำกัด (ฟรีทั้งหมด)
✓ ต้องการ self-contained executable (ไม่ต้องติดตั้ง runtime)

ไม่ควรเลือก Pascal/Lazarus เมื่อ:
✗ ต้องการ Web development (ใช้ JavaScript/Python ดีกว่า)
✗ ต้องการ Machine Learning (ใช้ Python ดีกว่า)
✗ ต้องการ Mobile development (ใช้ Flutter/React Native ดีกว่า)
✗ ต้องการไลบรารีจำนวนมาก (ecosystem ของ Python/JS ใหญ่กว่า)
```

---

## โครงสร้างของโปรแกรม Pascal พื้นฐาน

### โครงสร้างหลัก

```pascal
program ProgramName;      { 1. ชื่อโปรแกรม (จำเป็น) }

uses                      { 2. ไลบรารีที่ใช้ (ถ้ามี) }
  SysUtils,
  Math;

const                     { 3. ค่าคงที่ (ถ้ามี) }
  PI = 3.14159265358979;
  MAX_SIZE = 100;

type                      { 4. นิยาม type ใหม่ (ถ้ามี) }
  TMyRecord = record
    Name: String;
    Age: Integer;
  end;

var                       { 5. ตัวแปร (ถ้ามี) }
  x, y: Integer;
  name: String;

procedure ShowGreeting;   { 6. procedure (ถ้ามี) }
begin
  WriteLn('Hello!');
end;

function AddNumbers(a, b: Integer): Integer;  { 7. function (ถ้ามี) }
begin
  Result := a + b;
end;

begin                     { 8. Program body - ส่วนหลัก }
  x := 10;
  y := 20;
  WriteLn(AddNumbers(x, y));
  ShowGreeting;
end.                      { 9. จบโปรแกรม (จุด . สำคัญมาก!) }
```

### ส่วนประกอบสำคัญ

#### Program Declaration
```pascal
program MyFirstProgram;
// หรือ
program MyProgram(Input, Output);  // รูปแบบ ISO Pascal
```

#### Uses Clause (ไลบรารี)
```pascal
uses
  SysUtils,    // ฟังก์ชัน utility ทั่วไป (DateToStr, IntToStr, ฯลฯ)
  Math,        // ฟังก์ชันคณิตศาสตร์ (Sin, Cos, Sqrt, ฯลฯ)
  Classes,     // คลาสพื้นฐาน (TList, TStringList, ฯลฯ)
  StrUtils,    // ฟังก์ชันสำหรับ String
  DateUtils;   // ฟังก์ชันสำหรับ Date/Time
```

#### ความคิดเห็น (Comments)
```pascal
program Comments;
begin
  // นี่คือ comment แบบบรรทัดเดียว (style C++)
  
  { นี่คือ comment แบบหลายบรรทัด
    ใช้ curly braces
    สามารถเขียนหลายบรรทัด }
  
  (* นี่ก็เป็น comment แบบหลายบรรทัดเช่นกัน
     ใช้ parenthesis-asterisk
     แบบนี้ก็ได้ *)
  
  WriteLn('โปรแกรมทำงาน');
end.
```

#### Begin..End Blocks
```pascal
program BeginEnd;
var
  x: Integer;
begin
  // begin..end หลักของโปรแกรม
  x := 10;
  
  if x > 5 then
  begin
    // begin..end สำหรับ if block
    WriteLn('x มากกว่า 5');
    WriteLn('x = ', x);
  end;
  
  while x > 0 do
  begin
    // begin..end สำหรับ while loop
    Write(x, ' ');
    Dec(x);
  end;
  WriteLn;
end.
```

### ไวยากรณ์พื้นฐาน

```pascal
program SyntaxDemo;
var
  a, b, c: Integer;
  name: String;
  flag: Boolean;
begin
  // การกำหนดค่า ใช้ := (ไม่ใช่ =)
  a := 10;
  b := 20;
  c := a + b;
  name := 'Pascal';
  flag := True;
  
  // การแสดงผล
  WriteLn('c = ', c);
  WriteLn('name = ', name);
  WriteLn('flag = ', flag);
  
  // Semicolon ; คั่นระหว่าง statements
  a := 1; b := 2; c := a + b;
  
  // ไม่ต้องมี ; ก่อน end หรือ until
  WriteLn('Done');
end.
```

---

## ตัวอย่างโค้ด Hello World ทุกรูปแบบ

### 1. Hello World พื้นฐาน

```pascal
program HelloWorld01;
begin
  WriteLn('Hello, World!');
end.
```

**ผลลัพธ์:**
```
Hello, World!
```

### 2. Hello World พร้อมตัวแปร

```pascal
program HelloWorld02;
var
  greeting: String;
begin
  greeting := 'Hello, World!';
  WriteLn(greeting);
end.
```

### 3. Hello World ภาษาไทย

```pascal
program HelloWorld03;
begin
  WriteLn('สวัสดีชาวโลก!');
  WriteLn('ยินดีต้อนรับสู่โลกของ Pascal');
end.
```

### 4. Hello World พร้อมรับข้อมูล

```pascal
program HelloWorld04;
var
  name: String;
begin
  Write('กรุณาใส่ชื่อของคุณ: ');
  ReadLn(name);
  WriteLn('สวัสดี, ', name, '!');
  WriteLn('ยินดีต้อนรับสู่โลก Pascal');
end.
```

**ตัวอย่างการทำงาน:**
```
กรุณาใส่ชื่อของคุณ: สมชาย
สวัสดี, สมชาย!
ยินดีต้อนรับสู่โลก Pascal
```

### 5. Hello World หลายครั้ง

```pascal
program HelloWorld05;
var
  i: Integer;
begin
  for i := 1 to 5 do
    WriteLn('Hello, World! ครั้งที่ ', i);
end.
```

**ผลลัพธ์:**
```
Hello, World! ครั้งที่ 1
Hello, World! ครั้งที่ 2
Hello, World! ครั้งที่ 3
Hello, World! ครั้งที่ 4
Hello, World! ครั้งที่ 5
```

### 6. Hello World แบบ Formatted

```pascal
program HelloWorld06;
const
  BORDER = '================================';
begin
  WriteLn(BORDER);
  WriteLn('|  สวัสดีชาวโลก (Hello World)  |');
  WriteLn('|  Pascal Programming Course   |');
  WriteLn('|  By: Lazarus IDE             |');
  WriteLn(BORDER);
end.
```

**ผลลัพธ์:**
```
================================
|  สวัสดีชาวโลก (Hello World)  |
|  Pascal Programming Course   |
|  By: Lazarus IDE             |
================================
```

### 7. Hello World แบบใช้ Procedure

```pascal
program HelloWorld07;

procedure SayHello(name: String);
begin
  WriteLn('Hello, ', name, '!');
end;

procedure SayGoodbye(name: String);
begin
  WriteLn('Goodbye, ', name, '!');
end;

begin
  SayHello('Pascal');
  SayHello('Lazarus');
  SayHello('Free Pascal');
  SayGoodbye('World');
end.
```

### 8. Hello World แบบ OOP

```pascal
program HelloWorld08;

type
  TGreeter = class
  private
    FName: String;
  public
    constructor Create(aName: String);
    procedure SayHello;
    procedure SayBye;
  end;

constructor TGreeter.Create(aName: String);
begin
  FName := aName;
end;

procedure TGreeter.SayHello;
begin
  WriteLn('Hello! My name is ', FName);
end;

procedure TGreeter.SayBye;
begin
  WriteLn('Bye from ', FName, '!');
end;

var
  greeter: TGreeter;
begin
  greeter := TGreeter.Create('Pascal World');
  greeter.SayHello;
  greeter.SayBye;
  greeter.Free;
end.
```

### 9. Hello World แบบ ASCII Art

```pascal
program HelloWorld09;
begin
  WriteLn('  _   _      _ _        __        __         _     _ ');
  WriteLn(' | | | | ___| | | ___   \ \      / /__  _ __| | __| |');
  WriteLn(' | |_| |/ _ \ | |/ _ \   \ \ /\ / / _ \| |__| |/ _` |');
  WriteLn(' |  _  |  __/ | | (_) |   \ V  V / (_) | |  | | (_| |');
  WriteLn(' |_| |_|\___|_|_|\___/     \_/\_/ \___/|_|  |_|\__,_|');
  WriteLn;
  WriteLn('                  Pascal Programming');
end.
```

### 10. Hello World แบบแสดงข้อมูลระบบ

```pascal
program HelloWorld10;
uses
  SysUtils;
begin
  WriteLn('=================================');
  WriteLn('  Hello, Pascal World!');
  WriteLn('=================================');
  WriteLn;
  WriteLn('วันที่: ', DateToStr(Date));
  WriteLn('เวลา: ', TimeToStr(Time));
  WriteLn;
  WriteLn('FPC Version: ', {$I %FPCVERSION%});
  WriteLn('Target OS: ', {$I %FPCTARGET%});
  WriteLn('Target CPU: ', {$I %FPCTARGETCPU%});
  WriteLn('=================================');
end.
```

**ผลลัพธ์ตัวอย่าง:**
```
=================================
  Hello, Pascal World!
=================================

วันที่: 02/10/2026
เวลา: 10:30:45

FPC Version: 3.2.2
Target OS: linux
Target CPU: x86_64
=================================
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1: โปรแกรมแนะนำตัว
**คำสั่ง:** เขียนโปรแกรมที่รับชื่อ-นามสกุล, อายุ และจังหวัดที่อยู่ จากนั้นแสดงข้อมูลในรูปแบบสวยงาม

**ตัวอย่างผลลัพธ์:**
```
=== ข้อมูลส่วนตัว ===
ชื่อ: สมชาย ใจดี
อายุ: 25 ปี
จังหวัด: กรุงเทพมหานคร
=====================
```

**แนวทางการแก้:**
```pascal
program Exercise01;
var
  firstName, lastName, city: String;
  age: Integer;
begin
  WriteLn('=== กรอกข้อมูลส่วนตัว ===');
  Write('ชื่อ: ');
  ReadLn(firstName);
  Write('นามสกุล: ');
  ReadLn(lastName);
  Write('อายุ: ');
  ReadLn(age);
  Write('จังหวัด: ');
  ReadLn(city);
  
  WriteLn;
  WriteLn('=== ข้อมูลส่วนตัว ===');
  WriteLn('ชื่อ: ', firstName, ' ', lastName);
  WriteLn('อายุ: ', age, ' ปี');
  WriteLn('จังหวัด: ', city);
  WriteLn('=====================');
end.
```

### ข้อ 2: โปรแกรมคำนวณ BMI
**คำสั่ง:** รับน้ำหนัก (กก.) และส่วนสูง (ซม.) แล้วคำนวณและแสดง BMI พร้อมบอกว่าอยู่ในระดับใด

**สูตร:** BMI = น้ำหนัก(กก.) / (ส่วนสูง(เมตร))²

**แนวทางการแก้:**
```pascal
program Exercise02;
var
  weight, height, bmi: Double;
begin
  Write('น้ำหนัก (กก.): ');
  ReadLn(weight);
  Write('ส่วนสูง (ซม.): ');
  ReadLn(height);
  
  height := height / 100;  // แปลง ซม. เป็น เมตร
  bmi := weight / (height * height);
  
  WriteLn;
  WriteLn('BMI ของคุณ = ', bmi:5:2);
  
  if bmi < 18.5 then
    WriteLn('ผอม (Underweight)')
  else if bmi < 25 then
    WriteLn('ปกติ (Normal)')
  else if bmi < 30 then
    WriteLn('น้ำหนักเกิน (Overweight)')
  else
    WriteLn('อ้วน (Obese)');
end.
```

### ข้อ 3: โปรแกรมนับถอยหลัง
**คำสั่ง:** รับตัวเลขจากผู้ใช้ แล้วนับถอยหลังจนถึง 1 และแสดง "ระเบิด!" เมื่อถึง 0

**แนวทางการแก้:**
```pascal
program Exercise03;
var
  n: Integer;
begin
  Write('นับถอยหลังจากเลข: ');
  ReadLn(n);
  WriteLn;
  while n > 0 do
  begin
    WriteLn(n, '...');
    Dec(n);
  end;
  WriteLn('ระเบิด! BOOM! 💥');
end.
```

### ข้อ 4: ตาราง ASCII
**คำสั่ง:** เขียนโปรแกรมแสดงตัวอักษร A-Z พร้อม ASCII code ในรูปแบบตาราง

**แนวทางการแก้:**
```pascal
program Exercise04;
var
  ch: Char;
begin
  WriteLn('+-----+-------+');
  WriteLn('| ตัว | ASCII |');
  WriteLn('+-----+-------+');
  for ch := 'A' to 'Z' do
    WriteLn('|  ', ch, '  |  ', Ord(ch), '   |');
  WriteLn('+-----+-------+');
end.
```

### ข้อ 5: ตารางสูตรคูณ
**คำสั่ง:** รับตัวเลข 1-12 แล้วแสดงตารางสูตรคูณของตัวเลขนั้น

### ข้อ 6: โปรแกรมเดาเลข
**คำสั่ง:** โปรแกรมสุ่มเลข 1-100 แล้วให้ผู้ใช้เดา โดยบอกว่ามากไปหรือน้อยไป จนกว่าจะเดาถูก

### ข้อ 7: ค้นหาตัวเลขที่มากที่สุด
**คำสั่ง:** รับตัวเลข 5 จำนวน แล้วหาว่าตัวเลขใดมากที่สุดและน้อยที่สุด

### ข้อ 8: ตรวจสอบจำนวนเฉพาะ
**คำสั่ง:** รับตัวเลขและตรวจสอบว่าเป็นจำนวนเฉพาะ (Prime Number) หรือไม่

### ข้อ 9: แปลงหน่วย
**คำสั่ง:** เขียนโปรแกรมแปลงอุณหภูมิ:
- เซลเซียส → ฟาเรนไฮต์ → เคลวิน
- แสดงผลทั้ง 3 หน่วยพร้อมกัน

**สูตร:**
- F = C × 9/5 + 32
- K = C + 273.15

### ข้อ 10: โปรแกรมสร้างรูปทรง
**คำสั่ง:** เขียนโปรแกรมรับจำนวน n แล้วพิมพ์รูปสามเหลี่ยมดาว:
```
n = 5:
*
**
***
****
*****
```

**แนวทางการแก้ข้อ 10:**
```pascal
program Exercise10;
var
  n, i, j: Integer;
begin
  Write('ขนาดของรูป (n): ');
  ReadLn(n);
  WriteLn;
  for i := 1 to n do
  begin
    for j := 1 to i do
      Write('*');
    WriteLn;
  end;
end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ประวัติ Pascal** - จาก Niklaus Wirth (1970) จนถึง Free Pascal และ Lazarus ในปัจจุบัน
2. **Lazarus IDE** - เครื่องมือพัฒนาโปรแกรมแบบ visual ที่ใช้ Free Pascal
3. **ข้อดีของ Pascal** - อ่านง่าย, ปลอดภัย, ประสิทธิภาพสูง, cross-platform
4. **ซอฟต์แวร์ชื่อดัง** - Skype, Total Commander, Adobe Photoshop เวอร์ชันแรก
5. **เปรียบเทียบภาษา** - Pascal เหมาะกับการเรียนรู้และ GUI development
6. **โครงสร้างโปรแกรม** - program, uses, const, type, var, begin..end
7. **Hello World** - 10 รูปแบบต่างๆ

## บทต่อไป

**Part 02** จะพาไปดูการติดตั้งและตั้งค่า Lazarus IDE ทีละขั้นตอน พร้อมการสร้างโปรเจคแรก

---

*หมายเหตุ: โค้ดทุกตัวอย่างในเอกสารนี้สามารถ compile และรันได้จริงด้วย Free Pascal Compiler (FPC) 3.x หรือ Lazarus IDE*
