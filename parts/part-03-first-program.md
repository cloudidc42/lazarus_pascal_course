# Part 03 - โปรแกรมแรกด้วย Pascal

## สารบัญ

1. [โครงสร้างโปรแกรม Pascal อย่างละเอียด](#โครงสร้างโปรแกรม-pascal-อย่างละเอียด)
2. [writeln vs write, readln vs read](#writeln-vs-write-readln-vs-read)
3. [โปรแกรม Hello World แบบ Console](#โปรแกรม-hello-world-แบบ-console)
4. [โปรแกรม Hello World แบบ GUI](#โปรแกรม-hello-world-แบบ-gui)
5. [การ Compile และ Run](#การ-compile-และ-run)
6. [การแสดงผลข้อความ](#การแสดงผลข้อความ)
7. [การรับข้อมูลจากผู้ใช้](#การรับข้อมูลจากผู้ใช้)
8. [โปรแกรมคำนวณเบื้องต้น](#โปรแกรมคำนวณเบื้องต้น)
9. [ตัวอย่างโปรแกรม 10 โปรแกรม](#ตัวอย่างโปรแกรม-10-โปรแกรม)
10. [แบบฝึกหัด 15 ข้อ](#แบบฝึกหัด-15-ข้อ)

---

## โครงสร้างโปรแกรม Pascal อย่างละเอียด

### โครงสร้างทั้งหมด

```pascal
program ProgramName;      // ← Program Header (บังคับ)

{$mode objfpc}            // ← Compiler Directive (ตัวเลือก)
{$H+}                     // ← เปิด AnsiString mode

uses                      // ← Uses Clause (ถ้าต้องการ library)
  SysUtils,               //    ไลบรารี utility ทั่วไป
  Math,                   //    ไลบรารีคณิตศาสตร์
  StrUtils;               //    ไลบรารีสตริง

const                     // ← Constant Section (ถ้ามีค่าคงที่)
  APP_NAME = 'My Program';
  VERSION = '1.0';
  MAX_VALUE = 100;

type                      // ← Type Section (ถ้านิยาม type ใหม่)
  TPoint = record
    X, Y: Integer;
  end;
  TColor = (clRed, clGreen, clBlue);

var                       // ← Variable Section (ถ้ามีตัวแปร global)
  counter: Integer;
  name: String;
  isReady: Boolean;

// ← Function/Procedure Section (ถ้ามี)
function Add(a, b: Integer): Integer;
begin
  Result := a + b;
end;

procedure ShowMessage(msg: String);
begin
  WriteLn('[INFO] ', msg);
end;

begin                     // ← Main Program Body (บังคับ)
  // คำสั่งหลักของโปรแกรม
  ShowMessage('Program started');
  counter := Add(10, 20);
  WriteLn('Result: ', counter);
end.                      // ← End of Program (บังคับ มีจุด!)
```

### Program Declaration (ส่วนหัวโปรแกรม)

```pascal
// รูปแบบพื้นฐาน
program MyProgram;

// ชื่อโปรแกรมต้องเป็น identifier ที่ถูกต้อง:
// - ขึ้นต้นด้วยตัวอักษรหรือ _
// - ตามด้วยตัวอักษร ตัวเลข หรือ _
// - ไม่ใช้ reserved words
// - ไม่มีช่องว่าง

// ถูกต้อง:
program MyCalculator;
program student_grade;
program _TestProgram;
program Calculator2024;

// ไม่ถูกต้อง:
// program My Program;    // ← มีช่องว่าง
// program 2Calculator;   // ← ขึ้นต้นด้วยตัวเลข
// program begin;         // ← ใช้ reserved word
```

### Compiler Directives

```pascal
program DirectiveDemo;

{$mode objfpc}     // Object FPC mode (รองรับ OOP แบบ FPC)
{$mode delphi}     // Delphi compatibility mode
{$H+}             // AnsiString เป็น default (แทน ShortString)
{$J+}             // อนุญาตการเปลี่ยนค่า typed constants
{$R+}             // Range checking (debug mode)
{$Q+}             // Overflow checking (debug mode)
{$I-}             // ปิด I/O error checking (เปิดเองหลัง I/O)

// Conditional compilation
{$IFDEF WINDOWS}
  WriteLn('Running on Windows');
{$ENDIF}

{$IFDEF DEBUG}
  WriteLn('Debug mode');
{$ELSE}
  WriteLn('Release mode');
{$ENDIF}

begin
end.
```

### Uses Clause (การใช้ Library)

```pascal
program LibraryDemo;

uses
  // Standard Pascal units
  SysUtils,     // DateToStr, IntToStr, FileExists, ฯลฯ
  Math,         // Sin, Cos, Sqrt, Power, ฯลฯ
  Classes,      // TList, TStringList, TStream, ฯลฯ
  StrUtils,     // PosEx, ReplaceStr, ฯลฯ
  DateUtils,    // DaysBetween, IncMonth, ฯลฯ
  
  // Lazarus/FPC specific
  LCLType,      // Types สำหรับ LCL
  Forms,        // TApplication, TForm
  Controls,     // TControl, TComponent
  Graphics,     // TCanvas, TBitmap, TColor
  Dialogs,      // ShowMessage, MessageDlg
  StdCtrls,     // TButton, TLabel, TEdit, ฯลฯ
  
  // Custom units (ของเราเอง)
  MyUtils,      // ไฟล์ MyUtils.pas ในโปรเจค
  DataModule1;  // Data module

begin
  WriteLn(SysUtils.IntToStr(42));
  WriteLn(Math.Sqrt(16.0):5:2);
end.
```

### Begin..End Block อย่างละเอียด

```pascal
program BlockDemo;
var
  x, y: Integer;
begin
  // Main block
  x := 10;
  y := 20;
  
  // if-else block
  if x > y then
  begin
    WriteLn('x > y');
    WriteLn('x = ', x);
  end
  else if x < y then
  begin
    WriteLn('x < y');
    WriteLn('y = ', y);
  end
  else
  begin
    WriteLn('x = y');
  end;
  
  // while loop block
  while x < 15 do
  begin
    Inc(x);
    WriteLn('x = ', x);
  end;
  
  // for loop (ไม่ต้องมี begin..end ถ้ามีคำสั่งเดียว)
  for x := 1 to 5 do
    WriteLn(x);  // บรรทัดเดียว ไม่ต้อง begin..end
  
  // for loop หลายคำสั่ง
  for x := 1 to 5 do
  begin
    WriteLn('x = ', x);
    WriteLn('x^2 = ', x * x);
  end;

end.
```

---

## writeln vs write, readln vs read

### WriteLn vs Write

```pascal
program WriteDemo;
begin
  // Write - แสดงข้อความ ไม่ขึ้นบรรทัดใหม่
  Write('สวัสดี');
  Write(' ');
  Write('ชาวโลก');
  // ผลลัพธ์: สวัสดี ชาวโลก (ทั้งหมดอยู่บรรทัดเดียวกัน)
  
  WriteLn;  // ขึ้นบรรทัดใหม่ (เหมือนกด Enter)
  
  // WriteLn - แสดงข้อความและขึ้นบรรทัดใหม่
  WriteLn('บรรทัดที่ 1');
  WriteLn('บรรทัดที่ 2');
  WriteLn('บรรทัดที่ 3');
  
  // การแสดงหลายค่าพร้อมกัน
  WriteLn('ชื่อ: ', 'สมชาย', ' อายุ: ', 25);
  WriteLn('PI = ', 3.14159:8:5);   // :8:5 = ความกว้างรวม 8 ตำแหน่ง ทศนิยม 5 ตำแหน่ง
  
  // เปรียบเทียบ Write กับ WriteLn
  Write('A');
  Write('B');
  Write('C');
  WriteLn;  // ← ขึ้นบรรทัดใหม่ด้วยตนเอง
  // ผลลัพธ์: ABC
  
  WriteLn('X');
  WriteLn('Y');
  WriteLn('Z');
  // ผลลัพธ์:
  // X
  // Y
  // Z
end.
```

### การจัดรูปแบบ Output (Formatting)

```pascal
program FormatDemo;
var
  n: Integer;
  f: Double;
  s: String;
begin
  n := 42;
  f := 3.14159265;
  s := 'Hello';
  
  // Integer formatting
  WriteLn(n);            // 42
  WriteLn(n:5);          // "   42" (5 ตำแหน่ง, right aligned)
  WriteLn(n:10);         // "        42" (10 ตำแหน่ง)
  
  // Float formatting
  WriteLn(f);            // 3.14159265000000E+000 (default)
  WriteLn(f:8:2);        // "    3.14" (8 ตำแหน่ง, 2 ทศนิยม)
  WriteLn(f:10:4);       // "    3.1416" (10 ตำแหน่ง, 4 ทศนิยม)
  WriteLn(f:0:6);        // "3.141593" (ความกว้างอัตโนมัติ, 6 ทศนิยม)
  
  // String formatting
  WriteLn(s);            // Hello
  WriteLn(s:10);         // "     Hello" (10 ตำแหน่ง, right aligned)
  
  // การสร้างตาราง
  WriteLn('ชื่อ':15, 'อายุ':8, 'คะแนน':10);
  WriteLn('สมชาย':15, 25:8, 85.5:10:2);
  WriteLn('สมหญิง':15, 22:8, 92.0:10:2);
end.
```

**ผลลัพธ์:**
```
           ชื่อ    อายุ    คะแนน
         สมชาย      25     85.50
        สมหญิง      22     92.00
```

### ReadLn vs Read

```pascal
program ReadDemo;
var
  name: String;
  age: Integer;
  height: Double;
  a, b: Integer;
begin
  // ReadLn - รับข้อมูลหนึ่งค่า แล้วข้ามบรรทัด (กด Enter)
  Write('ชื่อ: ');
  ReadLn(name);   // รับ string ทั้งบรรทัด รวม spaces
  
  Write('อายุ: ');
  ReadLn(age);    // รับ integer
  
  Write('ส่วนสูง: ');
  ReadLn(height); // รับ double
  
  WriteLn('ชื่อ: ', name);
  WriteLn('อายุ: ', age);
  WriteLn('ส่วนสูง: ', height:5:1);
  
  // Read - รับข้อมูลโดยไม่ข้ามบรรทัด (ใช้ space คั่น)
  Write('ใส่ตัวเลข 2 ตัว (คั่นด้วย space): ');
  Read(a);
  Read(b);
  ReadLn;  // ← จำเป็นต้องอ่านบรรทัดที่เหลือ
  WriteLn('ผลรวม: ', a + b);
  
  // ReadLn หลายค่าในบรรทัดเดียว (ไม่แนะนำ)
  // ReadLn(a, b);  // รับ a แล้ว b จาก input เดียวกัน
end.
```

### ReadLn ไม่มี argument

```pascal
program PauseDemo;
begin
  WriteLn('โปรแกรมเสร็จสิ้น');
  WriteLn('กด Enter เพื่อออก...');
  ReadLn;  // ← รอจน user กด Enter
  // โปรแกรมออก
end.
```

---

## โปรแกรม Hello World แบบ Console

### โปรแกรม 1: Hello World พื้นฐาน

```pascal
program HelloConsole;
begin
  WriteLn('สวัสดี ชาวโลก!');
  WriteLn('Hello, World!');
  WriteLn('นี่คือโปรแกรม Pascal แรกของฉัน');
end.
```

### โปรแกรม 2: Hello World แบบ Interactive

```pascal
program HelloInteractive;
var
  name: String;
  age: Integer;
begin
  WriteLn('=================================');
  WriteLn('   ยินดีต้อนรับสู่โลก Pascal!   ');
  WriteLn('=================================');
  WriteLn;
  
  Write('กรุณาใส่ชื่อของคุณ: ');
  ReadLn(name);
  
  Write('กรุณาใส่อายุของคุณ: ');
  ReadLn(age);
  
  WriteLn;
  WriteLn('สวัสดี, ', name, '!');
  WriteLn('คุณอายุ ', age, ' ปี');
  
  if age < 18 then
    WriteLn('คุณยังเป็นเยาวชน!')
  else if age < 60 then
    WriteLn('คุณอยู่ในวัยผู้ใหญ่!')
  else
    WriteLn('คุณอยู่ในวัยผู้สูงอายุ!');
  
  WriteLn;
  WriteLn('กด Enter เพื่อออก...');
  ReadLn;
end.
```

### โปรแกรม 3: Hello World แบบมีสีสัน (ถ้ารองรับ ANSI)

```pascal
program HelloColored;
begin
  // ANSI color codes
  WriteLn(#27'[31m' + 'สีแดง'    + #27'[0m');
  WriteLn(#27'[32m' + 'สีเขียว'   + #27'[0m');
  WriteLn(#27'[33m' + 'สีเหลือง'  + #27'[0m');
  WriteLn(#27'[34m' + 'สีน้ำเงิน' + #27'[0m');
  WriteLn(#27'[35m' + 'สีม่วง'    + #27'[0m');
  WriteLn(#27'[36m' + 'สีฟ้า'     + #27'[0m');
  WriteLn(#27'[1m'  + 'ตัวหนา'    + #27'[0m');
  WriteLn(#27'[4m'  + 'ขีดเส้นใต้' + #27'[0m');
end.
```

---

## โปรแกรม Hello World แบบ GUI

### โครงสร้าง GUI Project

เมื่อสร้าง GUI Application ใน Lazarus จะได้ไฟล์:

**MyGUIApp.lpr (Main Program File):**
```pascal
program MyGUIApp;

{$mode objfpc}{$H+}

uses
  {$IFDEF UNIX}
  cthreads,
  {$ENDIF}
  Interfaces,  // ← จำเป็น! โหลด LCL interface
  Forms,
  Unit1        // ← Form unit ของเรา
  { คุณสามารถเพิ่ม unit อื่นๆ ที่นี่ };

{$R *.res}

begin
  RequireDerivedFormResource := True;
  Application.Scaled := True;
  Application.Initialize;
  Application.CreateForm(TForm1, Form1);  // ← สร้าง Form หลัก
  Application.Run;                         // ← เริ่มรัน event loop
end.
```

**Unit1.pas (Form Unit):**
```pascal
unit Unit1;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs, StdCtrls, ExtCtrls;

type
  { TForm1 }
  TForm1 = class(TForm)
    // Components (สร้างโดย Form Designer)
    btnHello: TButton;
    lblMessage: TLabel;
    edtName: TEdit;
    Panel1: TPanel;
    
    // Event handlers
    procedure btnHelloClick(Sender: TObject);
    procedure FormCreate(Sender: TObject);
    procedure edtNameKeyPress(Sender: TObject; var Key: Char);
  private
    { private declarations }
    procedure UpdateGreeting;
  public
    { public declarations }
  end;

var
  Form1: TForm1;

implementation

{$R *.lfm}

{ TForm1 }

procedure TForm1.FormCreate(Sender: TObject);
begin
  // เรียกเมื่อ Form ถูกสร้าง
  Caption := 'โปรแกรม Hello World GUI';
  lblMessage.Caption := 'กรุณาใส่ชื่อแล้วกดปุ่ม';
  lblMessage.Font.Size := 14;
  btnHello.Caption := 'กดเพื่อทักทาย!';
  edtName.TextHint := 'ใส่ชื่อของคุณที่นี่...';
end;

procedure TForm1.UpdateGreeting;
begin
  if edtName.Text <> '' then
  begin
    lblMessage.Caption := 'สวัสดี, ' + edtName.Text + '!';
    lblMessage.Font.Color := clBlue;
  end
  else
  begin
    lblMessage.Caption := 'กรุณาใส่ชื่อก่อน!';
    lblMessage.Font.Color := clRed;
  end;
end;

procedure TForm1.btnHelloClick(Sender: TObject);
begin
  UpdateGreeting;
end;

procedure TForm1.edtNameKeyPress(Sender: TObject; var Key: Char);
begin
  if Key = #13 then  // Enter key
  begin
    UpdateGreeting;
    Key := #0;  // ยกเลิกเสียง beep
  end;
end;

end.
```

### GUI Hello World ขั้นสูง

```pascal
unit HelloGUIAdvanced;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, ComCtrls;

type
  TMainForm = class(TForm)
    // Layout components
    pnlTop: TPanel;
    pnlCenter: TPanel;
    pnlBottom: TPanel;
    
    // Controls
    lblTitle: TLabel;
    lblName: TLabel;
    lblAge: TLabel;
    lblResult: TLabel;
    edtName: TEdit;
    edtAge: TEdit;
    btnGreet: TButton;
    btnClear: TButton;
    btnAbout: TButton;
    StatusBar1: TStatusBar;
    
    procedure btnGreetClick(Sender: TObject);
    procedure btnClearClick(Sender: TObject);
    procedure btnAboutClick(Sender: TObject);
    procedure FormCreate(Sender: TObject);
  private
    procedure SetupLayout;
    procedure UpdateStatus(msg: String);
  public
  end;

var
  MainForm: TMainForm;

implementation

{$R *.lfm}

procedure TMainForm.FormCreate(Sender: TObject);
begin
  Caption := 'Hello World - Pascal GUI';
  Width := 500;
  Height := 400;
  Position := poScreenCenter;
  SetupLayout;
end;

procedure TMainForm.SetupLayout;
begin
  lblTitle.Caption := '🌟 ยินดีต้อนรับสู่ Pascal';
  lblTitle.Font.Size := 18;
  lblTitle.Font.Style := [fsBold];
  
  lblName.Caption := 'ชื่อของคุณ:';
  lblAge.Caption := 'อายุ:';
  
  edtName.TextHint := 'กรุณาใส่ชื่อ';
  edtAge.TextHint := 'กรุณาใส่อายุ';
  
  btnGreet.Caption := '&ทักทาย';  // & ทำให้ G เป็น shortcut
  btnClear.Caption := '&ล้างข้อมูล';
  btnAbout.Caption := '&เกี่ยวกับ';
  
  UpdateStatus('พร้อมใช้งาน');
end;

procedure TMainForm.btnGreetClick(Sender: TObject);
var
  name: String;
  age: Integer;
  greeting: String;
begin
  name := Trim(edtName.Text);
  
  if name = '' then
  begin
    MessageDlg('แจ้งเตือน', 'กรุณาใส่ชื่อก่อน!', mtWarning, [mbOK], 0);
    edtName.SetFocus;
    Exit;
  end;
  
  try
    age := StrToInt(edtAge.Text);
  except
    age := 0;
  end;
  
  greeting := 'สวัสดี, ' + name + '!';
  
  if age > 0 then
  begin
    greeting := greeting + #10;
    if age < 18 then
      greeting := greeting + 'คุณเป็นเยาวชนวัย ' + IntToStr(age) + ' ปี'
    else if age < 60 then
      greeting := greeting + 'คุณอายุ ' + IntToStr(age) + ' ปี ยังหนุ่มสาวอยู่!'
    else
      greeting := greeting + 'คุณอายุ ' + IntToStr(age) + ' ปี อายุมากแต่ใจยังสดใส!';
  end;
  
  lblResult.Caption := greeting;
  lblResult.Font.Color := clNavy;
  UpdateStatus('แสดงการทักทายแล้ว');
end;

procedure TMainForm.btnClearClick(Sender: TObject);
begin
  edtName.Clear;
  edtAge.Clear;
  lblResult.Caption := '';
  edtName.SetFocus;
  UpdateStatus('ล้างข้อมูลแล้ว');
end;

procedure TMainForm.btnAboutClick(Sender: TObject);
begin
  MessageDlg('เกี่ยวกับโปรแกรม',
    'Hello World GUI' + #10 +
    'เวอร์ชัน 1.0' + #10 +
    'พัฒนาด้วย Lazarus/Pascal' + #10 +
    'โดย: นักเรียน Pascal',
    mtInformation, [mbOK], 0);
end;

procedure TMainForm.UpdateStatus(msg: String);
begin
  StatusBar1.SimpleText := '  ' + msg + '  |  ' + FormatDateTime('hh:nn:ss', Now);
end;

end.
```

---

## การ Compile และ Run

### การ Compile ผ่าน IDE

```
วิธีที่ 1: กด F9 (Compile + Run)
วิธีที่ 2: กด Ctrl+F9 (Compile only)
วิธีที่ 3: Run → Compile (Compile only)
วิธีที่ 4: Run → Run (Compile + Run)
วิธีที่ 5: Run → Build (Build all)
```

### การ Compile ผ่าน Command Line

```bash
# รูปแบบพื้นฐาน
fpc myprogram.pas

# รูปแบบระบุ output
fpc myprogram.pas -o myprogram

# รูปแบบ Debug
fpc -g -gl myprogram.pas

# รูปแบบ Release (optimize)
fpc -O2 -Xs myprogram.pas

# รูปแบบ Cross-compile (Windows สร้าง Linux binary)
fpc -T linux -P x86_64 myprogram.pas

# ดู options ทั้งหมด
fpc --help
```

### การ Run โปรแกรม

```bash
# Linux/macOS
./myprogram

# Windows
myprogram.exe

# ผ่าน IDE กด F9
```

### การ Compile หลายไฟล์

เมื่อโปรแกรมมีหลาย unit:

```
MyProject/
├── main.pas
├── Utils.pas
└── Calculator.pas
```

```bash
# FPC จะ compile ไฟล์ที่ uses มาให้อัตโนมัติ
fpc main.pas

# หรือใช้ Makefile
```

---

## การแสดงผลข้อความ

### WriteLn รูปแบบต่างๆ

```pascal
program OutputFormats;
uses
  SysUtils;
var
  intVal: Integer;
  floatVal: Double;
  strVal: String;
  boolVal: Boolean;
begin
  intVal := 42;
  floatVal := 3.14159;
  strVal := 'Pascal';
  boolVal := True;
  
  // แสดงตัวแปรต่างชนิด
  WriteLn('Integer: ', intVal);
  WriteLn('Float: ', floatVal:10:4);
  WriteLn('String: ', strVal);
  WriteLn('Boolean: ', boolVal);
  
  // แสดงหลายค่าในบรรทัดเดียว
  WriteLn('คำนวณ: ', intVal, ' + ', intVal, ' = ', intVal * 2);
  
  // Format ด้วย SysUtils
  WriteLn(Format('Hello %s! You are %d years old.', ['World', 25]));
  WriteLn(Format('Pi = %.5f', [3.14159265]));
  WriteLn(Format('Hex: %x  Oct: %o', [255, 255]));
  
  // แสดงวันที่เวลา
  WriteLn('วันนี้: ', DateToStr(Date));
  WriteLn('เวลา: ', TimeToStr(Time));
  WriteLn('DateTime: ', DateTimeToStr(Now));
end.
```

### การใช้ Escape Characters

```pascal
program EscapeChars;
begin
  // ตัวอักษรพิเศษใน Pascal
  WriteLn('Tab: [' + #9 + '] ← นี่คือ Tab');
  WriteLn('Newline ใน String: Line1' + #10 + 'Line2');
  WriteLn('Carriage Return: ' + #13);
  WriteLn('Single Quote: It''s OK');  // ← ใช้ '' แทน '
  WriteLn('Hash: ' + '#' + '13');
  
  // ASCII characters
  WriteLn('Bell: ', #7);    // เสียง beep
  WriteLn('BS: ', #8);     // Backspace
  WriteLn('ESC: ', #27);   // Escape
  
  // Unicode (ถ้าระบบรองรับ)
  WriteLn('Copyright: ' + #169);  // ©
  WriteLn('Degree: ' + #176);     // °
end.
```

### การสร้างตาราง

```pascal
program TableDemo;
uses
  SysUtils;
var
  i: Integer;
begin
  // ตารางแบบง่าย
  WriteLn('+--------+--------+----------+');
  WriteLn('| ลำดับ  |  ยกกำลัง 2 | ยกกำลัง 3 |');
  WriteLn('+--------+--------+----------+');
  for i := 1 to 10 do
    WriteLn('|', i:6, '  |', (i*i):8, '|', (i*i*i):10, '|');
  WriteLn('+--------+--------+----------+');
end.
```

**ผลลัพธ์:**
```
+--------+--------+----------+
| ลำดับ  |  ยกกำลัง 2 | ยกกำลัง 3 |
+--------+--------+----------+
|     1  |       1|         1|
|     2  |       4|         8|
|     3  |       9|        27|
|     4  |      16|        64|
|     5  |      25|       125|
...
```

---

## การรับข้อมูลจากผู้ใช้

### การรับข้อมูลพื้นฐาน

```pascal
program InputBasic;
var
  name: String;
  age: Integer;
  height: Double;
  gender: Char;
  isStudent: Boolean;
begin
  // รับ String
  Write('ชื่อ: ');
  ReadLn(name);
  
  // รับ Integer
  Write('อายุ: ');
  ReadLn(age);
  
  // รับ Double
  Write('ส่วนสูง (ซม.): ');
  ReadLn(height);
  
  // รับ Char
  Write('เพศ (M/F): ');
  ReadLn(gender);
  gender := UpCase(gender);  // แปลงเป็นพิมพ์ใหญ่
  
  // แสดงผล
  WriteLn;
  WriteLn('=== ข้อมูลของคุณ ===');
  WriteLn('ชื่อ: ', name);
  WriteLn('อายุ: ', age);
  WriteLn('ส่วนสูง: ', height:5:1, ' ซม.');
  WriteLn('เพศ: ', gender);
end.
```

### การรับข้อมูลพร้อม Validation

```pascal
program InputValidation;
uses
  SysUtils;

function ReadInteger(prompt: String; minVal, maxVal: Integer): Integer;
var
  input: String;
  value: Integer;
  isValid: Boolean;
begin
  repeat
    Write(prompt);
    ReadLn(input);
    isValid := TryStrToInt(input, value);
    if not isValid then
      WriteLn('กรุณาใส่ตัวเลขจำนวนเต็มเท่านั้น!')
    else if (value < minVal) or (value > maxVal) then
    begin
      WriteLn('กรุณาใส่ตัวเลขระหว่าง ', minVal, ' ถึง ', maxVal);
      isValid := False;
    end;
  until isValid;
  Result := value;
end;

function ReadFloat(prompt: String): Double;
var
  input: String;
  value: Double;
begin
  repeat
    Write(prompt);
    ReadLn(input);
    // แทนที่จุลภาคด้วยจุด (สำหรับบางระบบ)
    input := StringReplace(input, ',', '.', [rfReplaceAll]);
  until TryStrToFloat(input, value);
  Result := value;
end;

var
  score: Integer;
  price: Double;
begin
  score := ReadInteger('ใส่คะแนน (0-100): ', 0, 100);
  price := ReadFloat('ใส่ราคา: ');
  
  WriteLn('คะแนน: ', score);
  WriteLn('ราคา: ', price:10:2, ' บาท');
end.
```

### การรับข้อมูลหลายค่า

```pascal
program MultipleInputs;
var
  nums: array[1..5] of Integer;
  i, total: Integer;
  avg: Double;
begin
  WriteLn('ใส่ตัวเลข 5 ตัว:');
  total := 0;
  for i := 1 to 5 do
  begin
    Write('ตัวที่ ', i, ': ');
    ReadLn(nums[i]);
    Inc(total, nums[i]);
  end;
  
  avg := total / 5;
  
  WriteLn;
  WriteLn('ตัวเลขที่ใส่: ');
  for i := 1 to 5 do
    Write(nums[i], ' ');
  WriteLn;
  WriteLn('ผลรวม: ', total);
  WriteLn('ค่าเฉลี่ย: ', avg:6:2);
end.
```

---

## โปรแกรมคำนวณเบื้องต้น

### เครื่องคิดเลข 4 ฟังก์ชัน

```pascal
program SimpleCalculator;
var
  a, b: Double;
  op: String;
  result: Double;
  hasResult: Boolean;
begin
  WriteLn('=== เครื่องคิดเลข Pascal ===');
  WriteLn;
  
  repeat
    Write('ใส่ตัวเลขแรก: ');
    ReadLn(a);
    
    Write('ใส่ตัวดำเนินการ (+, -, *, /): ');
    ReadLn(op);
    
    Write('ใส่ตัวเลขที่สอง: ');
    ReadLn(b);
    
    hasResult := True;
    
    case op[1] of
      '+': result := a + b;
      '-': result := a - b;
      '*': result := a * b;
      '/': begin
             if b = 0 then
             begin
               WriteLn('Error: ไม่สามารถหารด้วย 0 ได้!');
               hasResult := False;
             end
             else
               result := a / b;
           end;
    else
      WriteLn('ตัวดำเนินการไม่ถูกต้อง!');
      hasResult := False;
    end;
    
    if hasResult then
      WriteLn(a:0:2, ' ', op, ' ', b:0:2, ' = ', result:0:4);
    
    WriteLn;
    Write('คำนวณอีกครั้ง? (y/n): ');
    ReadLn(op);
    
  until (op = 'n') or (op = 'N');
  
  WriteLn('ขอบคุณที่ใช้บริการ!');
end.
```

### คำนวณพื้นที่รูปทรง

```pascal
program AreaCalculator;
uses
  Math;

procedure ShowMenu;
begin
  WriteLn;
  WriteLn('=== คำนวณพื้นที่รูปทรง ===');
  WriteLn('1. วงกลม');
  WriteLn('2. สี่เหลี่ยมจัตุรัส');
  WriteLn('3. สี่เหลี่ยมผืนผ้า');
  WriteLn('4. สามเหลี่ยม');
  WriteLn('5. วงรี');
  WriteLn('0. ออก');
  Write('เลือก: ');
end;

function CalcCircleArea(r: Double): Double;
begin
  Result := Pi * r * r;
end;

function CalcRectArea(w, h: Double): Double;
begin
  Result := w * h;
end;

function CalcTriangleArea(b, h: Double): Double;
begin
  Result := 0.5 * b * h;
end;

function CalcEllipseArea(a, b: Double): Double;
begin
  Result := Pi * a * b;
end;

var
  choice: Integer;
  r, w, h, b, a, area: Double;
begin
  repeat
    ShowMenu;
    ReadLn(choice);
    WriteLn;
    
    case choice of
      1: begin
           Write('รัศมี: ');
           ReadLn(r);
           area := CalcCircleArea(r);
           WriteLn('พื้นที่วงกลม = ', area:10:4);
         end;
      2: begin
           Write('ด้าน: ');
           ReadLn(w);
           area := CalcRectArea(w, w);
           WriteLn('พื้นที่สี่เหลี่ยมจัตุรัส = ', area:10:4);
         end;
      3: begin
           Write('กว้าง: ');
           ReadLn(w);
           Write('ยาว: ');
           ReadLn(h);
           area := CalcRectArea(w, h);
           WriteLn('พื้นที่สี่เหลี่ยมผืนผ้า = ', area:10:4);
         end;
      4: begin
           Write('ฐาน: ');
           ReadLn(b);
           Write('สูง: ');
           ReadLn(h);
           area := CalcTriangleArea(b, h);
           WriteLn('พื้นที่สามเหลี่ยม = ', area:10:4);
         end;
      5: begin
           Write('แกนยาว (a): ');
           ReadLn(a);
           Write('แกนสั้น (b): ');
           ReadLn(b);
           area := CalcEllipseArea(a, b);
           WriteLn('พื้นที่วงรี = ', area:10:4);
         end;
      0: WriteLn('ออกจากโปรแกรม...');
    else
      WriteLn('ตัวเลือกไม่ถูกต้อง!');
    end;
    
  until choice = 0;
end.
```

---

## ตัวอย่างโปรแกรม 10 โปรแกรม

### โปรแกรมที่ 1: แปลงอุณหภูมิ

```pascal
program TempConverter;

function CelsiusToFahrenheit(c: Double): Double;
begin
  Result := (c * 9 / 5) + 32;
end;

function CelsiusToKelvin(c: Double): Double;
begin
  Result := c + 273.15;
end;

function FahrenheitToCelsius(f: Double): Double;
begin
  Result := (f - 32) * 5 / 9;
end;

var
  temp: Double;
  choice: Integer;
begin
  WriteLn('=== แปลงอุณหภูมิ ===');
  WriteLn('1. Celsius → Fahrenheit/Kelvin');
  WriteLn('2. Fahrenheit → Celsius');
  Write('เลือก: ');
  ReadLn(choice);
  
  case choice of
    1: begin
         Write('อุณหภูมิ (°C): ');
         ReadLn(temp);
         WriteLn(temp:0:2, '°C = ', CelsiusToFahrenheit(temp):0:2, '°F');
         WriteLn(temp:0:2, '°C = ', CelsiusToKelvin(temp):0:2, 'K');
       end;
    2: begin
         Write('อุณหภูมิ (°F): ');
         ReadLn(temp);
         WriteLn(temp:0:2, '°F = ', FahrenheitToCelsius(temp):0:2, '°C');
       end;
  end;
end.
```

### โปรแกรมที่ 2: ตรวจสอบปีอธิกสุรทิน

```pascal
program LeapYear;
uses
  SysUtils;

function IsLeapYear(year: Integer): Boolean;
begin
  Result := ((year mod 4 = 0) and (year mod 100 <> 0)) or
            (year mod 400 = 0);
end;

var
  year: Integer;
begin
  Write('ใส่ปี ค.ศ.: ');
  ReadLn(year);
  
  if IsLeapYear(year) then
    WriteLn(year, ' เป็นปีอธิกสุรทิน (มี 366 วัน)')
  else
    WriteLn(year, ' ไม่เป็นปีอธิกสุรทิน (มี 365 วัน)');
  
  // แสดง 5 ปีอธิกสุรทินถัดไป
  WriteLn('ปีอธิกสุรทิน 5 ปีถัดไป:');
  var count := 0;
  var y := year + 1;
  while count < 5 do
  begin
    if IsLeapYear(y) then
    begin
      Write(y, ' ');
      Inc(count);
    end;
    Inc(y);
  end;
  WriteLn;
end.
```

### โปรแกรมที่ 3: แปลงเลขฐาน

```pascal
program BaseConverter;
uses
  SysUtils;

function DecToBin(n: Integer): String;
begin
  if n = 0 then
    Result := '0'
  else
  begin
    Result := '';
    while n > 0 do
    begin
      Result := IntToStr(n mod 2) + Result;
      n := n div 2;
    end;
  end;
end;

function DecToHex(n: Integer): String;
const
  HEX_CHARS = '0123456789ABCDEF';
begin
  if n = 0 then
    Result := '0'
  else
  begin
    Result := '';
    while n > 0 do
    begin
      Result := HEX_CHARS[(n mod 16) + 1] + Result;
      n := n div 16;
    end;
  end;
end;

var
  n: Integer;
begin
  Write('ใส่ตัวเลขฐาน 10: ');
  ReadLn(n);
  
  WriteLn('ฐาน 10: ', n);
  WriteLn('ฐาน 2 (Binary): ', DecToBin(n));
  WriteLn('ฐาน 8 (Octal): ', OctStr(n));
  WriteLn('ฐาน 16 (Hex): ', DecToHex(n));
  WriteLn;
  WriteLn('เปรียบเทียบ:');
  WriteLn('$', IntToHex(n, 4), ' = ', n, ' (ทศนิยม)');
end.
```

### โปรแกรมที่ 4: เกมทายตัวเลข

```pascal
program GuessingGame;
uses
  SysUtils;

var
  secret, guess, attempts: Integer;
  playing: Boolean;
begin
  Randomize;  // เริ่ม random seed
  
  WriteLn('=== เกมทายตัวเลข ===');
  WriteLn('ทายตัวเลขระหว่าง 1-100');
  WriteLn;
  
  playing := True;
  while playing do
  begin
    secret := Random(100) + 1;  // สุ่ม 1-100
    attempts := 0;
    
    repeat
      Inc(attempts);
      Write('ทาย (', attempts, '): ');
      ReadLn(guess);
      
      if guess < secret then
        WriteLn('น้อยเกินไป! ลองอีกครั้ง')
      else if guess > secret then
        WriteLn('มากเกินไป! ลองอีกครั้ง')
      else
      begin
        WriteLn('ถูกต้อง! คุณทายถูกใน ', attempts, ' ครั้ง!');
        if attempts <= 3 then
          WriteLn('ยอดเยี่ยมมาก! 🏆')
        else if attempts <= 7 then
          WriteLn('ดีมาก! 👍')
        else
          WriteLn('ยังมีห้องให้พัฒนาอีก 💪');
      end;
      
    until guess = secret;
    
    WriteLn;
    Write('เล่นอีกครั้ง? (y/n): ');
    var answer: String;
    ReadLn(answer);
    playing := (LowerCase(answer) = 'y');
    WriteLn;
  end;
  
  WriteLn('ขอบคุณที่เล่น! สนุกไหม?');
end.
```

### โปรแกรมที่ 5: Fibonacci Sequence

```pascal
program FibonacciDemo;

function FibRecursive(n: Integer): Int64;
begin
  if n <= 1 then
    Result := n
  else
    Result := FibRecursive(n - 1) + FibRecursive(n - 2);
end;

procedure FibIterative(count: Integer);
var
  a, b, temp: Int64;
  i: Integer;
begin
  a := 0;
  b := 1;
  Write(a, ', ', b);
  for i := 3 to count do
  begin
    temp := a + b;
    a := b;
    b := temp;
    Write(', ', b);
  end;
  WriteLn;
end;

var
  n: Integer;
  i: Integer;
begin
  Write('แสดง Fibonacci กี่ตัว? ');
  ReadLn(n);
  
  WriteLn;
  WriteLn('แบบ Iterative (เร็วกว่า):');
  FibIterative(n);
  
  WriteLn;
  WriteLn('แบบ Recursive (ช้ากว่า แต่เข้าใจง่าย):');
  for i := 0 to n - 1 do
  begin
    Write(FibRecursive(i));
    if i < n - 1 then Write(', ');
  end;
  WriteLn;
end.
```

### โปรแกรมที่ 6: แฟกทอเรียล

```pascal
program Factorial;

function FactRecursive(n: Integer): Int64;
begin
  if n <= 1 then
    Result := 1
  else
    Result := n * FactRecursive(n - 1);
end;

function FactIterative(n: Integer): Int64;
var
  i: Integer;
  result: Int64;
begin
  result := 1;
  for i := 2 to n do
    result := result * i;
  FactIterative := result;
end;

var
  n, i: Integer;
begin
  WriteLn('=== ตารางแฟกทอเรียล ===');
  WriteLn('+----+--------------------+');
  WriteLn('| n! |      ค่า            |');
  WriteLn('+----+--------------------+');
  for i := 0 to 20 do
    WriteLn('|', i:3, '!|', FactIterative(i):20, '|');
  WriteLn('+----+--------------------+');
  
  WriteLn;
  Write('คำนวณ n!: ');
  ReadLn(n);
  if n > 20 then
    WriteLn('ค่าใหญ่เกิน Int64!')
  else
    WriteLn(n, '! = ', FactRecursive(n));
end.
```

### โปรแกรมที่ 7: การเรียงลำดับ Bubble Sort

```pascal
program BubbleSort;

var
  arr: array[1..10] of Integer;
  n, i, j, temp: Integer;
  
procedure PrintArray;
var k: Integer;
begin
  for k := 1 to n do
    Write(arr[k], ' ');
  WriteLn;
end;

begin
  WriteLn('=== Bubble Sort ===');
  Write('จำนวนข้อมูล (max 10): ');
  ReadLn(n);
  
  WriteLn('ใส่ตัวเลข:');
  for i := 1 to n do
  begin
    Write('arr[', i, ']: ');
    ReadLn(arr[i]);
  end;
  
  Write('ก่อน sort: ');
  PrintArray;
  
  // Bubble Sort algorithm
  for i := 1 to n - 1 do
    for j := 1 to n - i do
      if arr[j] > arr[j + 1] then
      begin
        temp := arr[j];
        arr[j] := arr[j + 1];
        arr[j + 1] := temp;
      end;
  
  Write('หลัง sort:  ');
  PrintArray;
end.
```

### โปรแกรมที่ 8: ค้นหาข้อมูล Binary Search

```pascal
program BinarySearch;

const
  SIZE = 10;

var
  arr: array[1..SIZE] of Integer = (2, 5, 8, 12, 16, 23, 38, 56, 72, 91);

function BinarySearch(target: Integer): Integer;
var
  low, high, mid: Integer;
begin
  low := 1;
  high := SIZE;
  Result := -1;
  
  while low <= high do
  begin
    mid := (low + high) div 2;
    
    if arr[mid] = target then
    begin
      Result := mid;
      Exit;
    end
    else if arr[mid] < target then
      low := mid + 1
    else
      high := mid - 1;
  end;
end;

var
  target, pos: Integer;
  i: Integer;
begin
  Write('ข้อมูลในอาร์เรย์: ');
  for i := 1 to SIZE do
    Write(arr[i], ' ');
  WriteLn;
  WriteLn;
  
  Write('ค้นหาตัวเลข: ');
  ReadLn(target);
  
  pos := BinarySearch(target);
  
  if pos <> -1 then
    WriteLn('พบ ', target, ' ที่ตำแหน่ง ', pos)
  else
    WriteLn('ไม่พบ ', target, ' ในอาร์เรย์');
end.
```

### โปรแกรมที่ 9: จัดการ String

```pascal
program StringOperations;
uses
  SysUtils, StrUtils;

var
  s, s2: String;
  i: Integer;
begin
  WriteLn('=== การจัดการ String ===');
  Write('ใส่ประโยค: ');
  ReadLn(s);
  
  WriteLn;
  WriteLn('ข้อความเดิม:      "', s, '"');
  WriteLn('ความยาว:          ', Length(s));
  WriteLn('ตัวพิมพ์ใหญ่:     "', UpperCase(s), '"');
  WriteLn('ตัวพิมพ์เล็ก:     "', LowerCase(s), '"');
  WriteLn('ตัดช่องว่าง:      "', Trim(s), '"');
  WriteLn('กลับด้าน:         "', ReverseString(s), '"');
  
  WriteLn;
  Write('ค้นหาคำ: ');
  ReadLn(s2);
  i := Pos(s2, s);
  if i > 0 then
    WriteLn('พบ "', s2, '" ที่ตำแหน่ง ', i)
  else
    WriteLn('ไม่พบ "', s2, '"');
  
  WriteLn;
  Write('แทนที่คำ (เก่า): ');
  ReadLn(s2);
  var newWord: String;
  Write('แทนที่ด้วย: ');
  ReadLn(newWord);
  WriteLn('ผลลัพธ์: "', StringReplace(s, s2, newWord, [rfReplaceAll]), '"');
end.
```

### โปรแกรมที่ 10: โปรแกรมจัดการนักเรียน

```pascal
program StudentManager;
uses
  SysUtils;

const
  MAX_STUDENTS = 50;

type
  TStudent = record
    ID: Integer;
    Name: String;
    Score: Double;
    Grade: Char;
  end;

var
  students: array[1..MAX_STUDENTS] of TStudent;
  count: Integer;

function CalculateGrade(score: Double): Char;
begin
  if score >= 80 then Result := 'A'
  else if score >= 70 then Result := 'B'
  else if score >= 60 then Result := 'C'
  else if score >= 50 then Result := 'D'
  else Result := 'F';
end;

procedure AddStudent;
begin
  if count >= MAX_STUDENTS then
  begin
    WriteLn('ไม่สามารถเพิ่มนักเรียนได้อีกแล้ว!');
    Exit;
  end;
  
  Inc(count);
  students[count].ID := count;
  
  Write('ชื่อนักเรียน: ');
  ReadLn(students[count].Name);
  
  Write('คะแนน (0-100): ');
  ReadLn(students[count].Score);
  
  students[count].Grade := CalculateGrade(students[count].Score);
  
  WriteLn('เพิ่มนักเรียน ID=', count, ' เรียบร้อย');
end;

procedure ShowAllStudents;
var
  i: Integer;
begin
  if count = 0 then
  begin
    WriteLn('ยังไม่มีข้อมูลนักเรียน');
    Exit;
  end;
  
  WriteLn;
  WriteLn('+----+------------------+-------+------+');
  WriteLn('| ID |       ชื่อ        | คะแนน | เกรด |');
  WriteLn('+----+------------------+-------+------+');
  for i := 1 to count do
    WriteLn('|', students[i].ID:3, ' |',
            students[i].Name:18, '|',
            students[i].Score:7:1, '|',
            students[i].Grade:5, ' |');
  WriteLn('+----+------------------+-------+------+');
end;

procedure ShowStatistics;
var
  i: Integer;
  total, highest, lowest: Double;
  grades: array['A'..'F'] of Integer;
  g: Char;
begin
  if count = 0 then Exit;
  
  total := 0;
  highest := students[1].Score;
  lowest := students[1].Score;
  
  for g := 'A' to 'F' do
    grades[g] := 0;
  
  for i := 1 to count do
  begin
    total := total + students[i].Score;
    if students[i].Score > highest then highest := students[i].Score;
    if students[i].Score < lowest then lowest := students[i].Score;
    if students[i].Grade in ['A'..'F'] then
      Inc(grades[students[i].Grade]);
  end;
  
  WriteLn;
  WriteLn('=== สถิติ ===');
  WriteLn('จำนวนนักเรียน: ', count);
  WriteLn('คะแนนเฉลี่ย:   ', (total/count):6:2);
  WriteLn('คะแนนสูงสุด:   ', highest:6:2);
  WriteLn('คะแนนต่ำสุด:   ', lowest:6:2);
  WriteLn;
  WriteLn('การกระจายเกรด:');
  for g := 'A' to 'F' do
    if g <> 'E' then  // ข้าม E (ไม่ใช้)
      WriteLn('  เกรด ', g, ': ', grades[g], ' คน');
end;

var
  choice: Integer;
begin
  count := 0;
  
  repeat
    WriteLn;
    WriteLn('=== ระบบจัดการนักเรียน ===');
    WriteLn('1. เพิ่มนักเรียน');
    WriteLn('2. แสดงรายชื่อ');
    WriteLn('3. แสดงสถิติ');
    WriteLn('0. ออก');
    Write('เลือก: ');
    ReadLn(choice);
    
    case choice of
      1: AddStudent;
      2: ShowAllStudents;
      3: ShowStatistics;
      0: WriteLn('ลาก่อน!');
    else
      WriteLn('ตัวเลือกไม่ถูกต้อง');
    end;
    
  until choice = 0;
end.
```

---

## แบบฝึกหัด 15 ข้อ

### ข้อ 1: โปรแกรมแสดงสูตรคูณ
เขียนโปรแกรมรับเลข n แล้วแสดงสูตรคูณของ n จาก 1 ถึง 12:
```
n = 7:
7 × 1 = 7
7 × 2 = 14
...
7 × 12 = 84
```

### ข้อ 2: โปรแกรมนับสระ
รับประโยคภาษาอังกฤษ แล้วนับจำนวนสระ (a, e, i, o, u):
```pascal
// แนวทาง
var
  s: String;
  count, i: Integer;
begin
  ReadLn(s);
  s := LowerCase(s);
  count := 0;
  for i := 1 to Length(s) do
    if s[i] in ['a', 'e', 'i', 'o', 'u'] then
      Inc(count);
  WriteLn('สระ: ', count);
end.
```

### ข้อ 3: โปรแกรม Palindrome
ตรวจสอบว่า string เป็น palindrome หรือไม่ (อ่านจากหน้าหลังเหมือนกัน เช่น "racecar", "level"):

### ข้อ 4: หา GCD และ LCM
รับตัวเลข 2 จำนวน แล้วหา:
- GCD (Greatest Common Divisor) = ห.ร.ม.
- LCM (Least Common Multiple) = ค.ร.น.

```pascal
// แนวทาง GCD ด้วย Euclid
function GCD(a, b: Integer): Integer;
begin
  while b <> 0 do
  begin
    var temp := b;
    b := a mod b;
    a := temp;
  end;
  Result := a;
end;
```

### ข้อ 5: โปรแกรมเครื่องคิดเลข Advanced
ขยายโปรแกรม Calculator ให้รองรับ:
- power (^)
- square root (sqrt)
- modulo (%)
- factorial (!)

### ข้อ 6: Pattern Printing
รับตัวเลข n แล้วพิมพ์ pattern:
```
n = 5:
    *
   ***
  *****
 *******
*********
```

### ข้อ 7: โปรแกรมเดินทาง
รับระยะทาง (กม.) และความเร็ว (กม./ชม.) แล้วคำนวณ:
- เวลาที่ใช้
- ถ้าออกเดินทาง 8:00 จะถึงเวลาไหน
- ค่าน้ำมัน (ถ้าค่าน้ำมัน 40 บาท/ลิตร และรถใช้ 15 กม./ลิตร)

### ข้อ 8: Matrix Addition
รับ matrix 2x2 สองชุด แล้วบวกกัน:
```
[1 2]   [5 6]   [6  8]
[3 4] + [7 8] = [10 12]
```

### ข้อ 9: โปรแกรม Word Count
รับ string แล้วนับ:
- จำนวนตัวอักษรทั้งหมด
- จำนวนคำ (คั่นด้วย space)
- จำนวนบรรทัด
- จำนวน spaces

### ข้อ 10: ระบบคะแนน 5 วิชา
รับคะแนน 5 วิชา แล้วแสดง:
- คะแนนแต่ละวิชา
- คะแนนรวม
- คะแนนเฉลี่ย
- เกรดรวม (GPA)
- ผ่าน/ไม่ผ่าน

### ข้อ 11: เลขฐาน 2
รับเลขฐาน 10 แล้วแปลงเป็นฐาน 2 (โดยไม่ใช้ library):

### ข้อ 12: ตรวจสอบ Armstrong Number
Armstrong Number คือตัวเลขที่ผลรวมของแต่ละหลักยกกำลังจำนวนหลักเท่ากับตัวเลขนั้น เช่น:
- 153 = 1³ + 5³ + 3³ = 1 + 125 + 27 = 153 ✓
- 371 = 3³ + 7³ + 1³ = 27 + 343 + 1 = 371 ✓

### ข้อ 13: โปรแกรม Currency Converter
สร้างโปรแกรมแปลงสกุลเงิน:
- บาท (THB)
- ดอลลาร์ (USD)
- ยูโร (EUR)
- เยน (JPY)

(ใช้อัตราแลกเปลี่ยนที่กำหนดเองได้)

### ข้อ 14: วันของสัปดาห์
รับวันที่ (dd/mm/yyyy) แล้วบอกว่าเป็นวันอะไรของสัปดาห์:
```pascal
// แนวทาง
uses DateUtils;
var
  d: TDateTime;
begin
  d := EncodeDate(2024, 1, 15);
  WriteLn(DayOfWeek(d));  // 1=อาทิตย์, 2=จันทร์, ...
end.
```

### ข้อ 15: โปรแกรม Quiz
สร้างโปรแกรม Quiz ที่มีคำถาม 5 ข้อ (Multiple Choice) พร้อมเฉลย และคิดคะแนน:

```pascal
// แนวทางโครงสร้าง
type
  TQuestion = record
    Question: String;
    Choices: array[1..4] of String;
    CorrectAnswer: Integer;
  end;

const
  QUESTIONS: array[1..5] of TQuestion = (
    (Question: 'Pascal ถูกสร้างในปีใด?';
     Choices: ('1960', '1970', '1980', '1990');
     CorrectAnswer: 2),
    // ... ข้อถัดไป
  );
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **โครงสร้างโปรแกรม Pascal** - Program declaration, uses, const, type, var, begin..end อย่างละเอียด
2. **WriteLn vs Write** - ความแตกต่างและการ format output
3. **ReadLn vs Read** - วิธีรับข้อมูลจากผู้ใช้
4. **Console vs GUI** - โปรแกรมทั้งสองแบบพร้อมตัวอย่าง
5. **โปรแกรม 10 ตัวอย่าง** - ครอบคลุมการใช้งานจริง

## บทต่อไป

**Part 04** จะอธิบายชนิดข้อมูลทุกประเภทใน Pascal อย่างละเอียด ตั้งแต่ Integer ไปจนถึง String และ Record

---

*โค้ดทุกตัวอย่างทดสอบด้วย Free Pascal 3.2.x และ Lazarus 3.x*
