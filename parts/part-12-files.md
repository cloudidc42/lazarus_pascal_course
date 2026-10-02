# Part 12 - การจัดการไฟล์ (File Management)

## บทนำ

การจัดการไฟล์เป็นทักษะสำคัญในการเขียนโปรแกรม ช่วยให้ข้อมูลยังคงอยู่แม้ปิดโปรแกรมแล้ว Pascal/Lazarus รองรับการจัดการไฟล์หลายรูปแบบ ตั้งแต่ text files ธรรมดาไปจนถึง binary files และ streams

---

## 12.1 ประเภทของไฟล์ใน Pascal

| ประเภท | การประกาศ | ใช้กับ |
|--------|-----------|--------|
| Text File | `TextFile` หรือ `Text` | ไฟล์ข้อความ |
| Typed File | `File of T` | ไฟล์ binary ชนิดเดียวกัน |
| Untyped File | `File` | ไฟล์ binary ทั่วไป |

---

## 12.2 Text Files

### ตัวอย่างที่ 1: เขียน Text File พื้นฐาน

```pascal
program WriteTextFile;

{$mode objfpc}{$H+}

var
  F    : TextFile;
  Line : String;
  i    : Integer;

begin
  // กำหนดชื่อไฟล์
  AssignFile(F, 'output.txt');
  
  // เปิดเพื่อเขียน (สร้างใหม่หรือทับของเดิม)
  Rewrite(F);
  
  // เขียนข้อมูล
  WriteLn(F, 'สวัสดี ไฟล์ข้อความ');
  WriteLn(F, 'บรรทัดที่ 2');
  WriteLn(F, 'บรรทัดที่ 3');
  
  for i := 1 to 5 do
    WriteLn(F, 'รายการที่ ', i, ': ค่า = ', i * i);
  
  // ปิดไฟล์ (สำคัญมาก!)
  CloseFile(F);
  
  WriteLn('เขียนไฟล์เสร็จแล้ว');
  ReadLn;
end.
```

### ตัวอย่างที่ 2: อ่าน Text File

```pascal
program ReadTextFile;

{$mode objfpc}{$H+}

var
  F       : TextFile;
  Line    : String;
  LineNum : Integer;

begin
  AssignFile(F, 'output.txt');
  
  // ตรวจสอบก่อนว่าไฟล์มีอยู่
  if not FileExists('output.txt') then
  begin
    WriteLn('ไม่พบไฟล์ output.txt');
    ReadLn;
    Exit;
  end;
  
  // เปิดเพื่ออ่าน
  Reset(F);
  
  LineNum := 0;
  WriteLn('=== เนื้อหาในไฟล์ ===');
  
  while not EOF(F) do
  begin
    ReadLn(F, Line);
    Inc(LineNum);
    WriteLn(Format('%3d: %s', [LineNum, Line]));
  end;
  
  CloseFile(F);
  WriteLn('อ่านทั้งหมด ', LineNum, ' บรรทัด');
  ReadLn;
end.
```

### ตัวอย่างที่ 3: เพิ่มข้อมูลท้ายไฟล์ (Append)

```pascal
program AppendToFile;

{$mode objfpc}{$H+}

uses SysUtils;

var
  F : TextFile;

procedure LogMessage(Msg: String);
begin
  AssignFile(F, 'app.log');
  
  if FileExists('app.log') then
    Append(F)   // เพิ่มต่อจากเดิม
  else
    Rewrite(F); // สร้างใหม่
  
  WriteLn(F, '[', FormatDateTime('dd/mm/yyyy hh:nn:ss', Now), '] ', Msg);
  CloseFile(F);
end;

begin
  LogMessage('เริ่มต้นโปรแกรม');
  LogMessage('โหลดข้อมูลสำเร็จ');
  LogMessage('ผู้ใช้ล็อกอิน: admin');
  LogMessage('ทำรายการสำเร็จ');
  LogMessage('ปิดโปรแกรม');
  
  WriteLn('บันทึก log เสร็จแล้ว');
  
  // อ่านและแสดง log
  AssignFile(F, 'app.log');
  Reset(F);
  var Line: String;
  WriteLn('=== Log File ===');
  while not EOF(F) do
  begin
    ReadLn(F, Line);
    WriteLn(Line);
  end;
  CloseFile(F);
  
  ReadLn;
end.
```

### ตัวอย่างที่ 4: อ่านทีละอักขระ

```pascal
program ReadCharByChar;

{$mode objfpc}{$H+}

var
  F    : TextFile;
  Ch   : Char;
  Freq : array[Char] of Integer;
  C    : Char;

begin
  // สร้างไฟล์ทดสอบก่อน
  AssignFile(F, 'test.txt');
  Rewrite(F);
  WriteLn(F, 'Hello World! สวัสดีชาวโลก');
  WriteLn(F, 'Pascal Programming is fun!');
  CloseFile(F);
  
  // นับความถี่ตัวอักษร
  for C := 'A' to 'Z' do Freq[C] := 0;
  for C := 'a' to 'z' do Freq[C] := 0;
  
  AssignFile(F, 'test.txt');
  Reset(F);
  
  while not EOF(F) do
  begin
    Read(F, Ch);
    if Ch in ['A'..'Z', 'a'..'z'] then
      Inc(Freq[Ch]);
  end;
  
  CloseFile(F);
  
  WriteLn('=== ความถี่ตัวอักษร ===');
  for C := 'A' to 'Z' do
    if (Freq[C] > 0) or (Freq[LowerCase(C)[1]] > 0) then
      WriteLn(C, ': ', Freq[C], '  ', LowerCase(C), ': ', Freq[LowerCase(C)[1]]);
  
  ReadLn;
end.
```

### ตัวอย่างที่ 5: อ่านตัวเลขจากไฟล์

```pascal
program ReadNumbersFromFile;

{$mode objfpc}{$H+}

var
  F      : TextFile;
  Num    : Real;
  Count  : Integer;
  Sum    : Real;
  Min, Max: Real;

begin
  // สร้างไฟล์ตัวเลข
  AssignFile(F, 'numbers.txt');
  Rewrite(F);
  WriteLn(F, '45.5');
  WriteLn(F, '78.3');
  WriteLn(F, '92.1');
  WriteLn(F, '33.7');
  WriteLn(F, '61.9');
  WriteLn(F, '55.0');
  CloseFile(F);
  
  // อ่านและวิเคราะห์
  AssignFile(F, 'numbers.txt');
  Reset(F);
  
  Count := 0;
  Sum   := 0;
  
  if not EOF(F) then
  begin
    ReadLn(F, Num);
    Min := Num;
    Max := Num;
    Sum := Num;
    Count := 1;
  end;
  
  while not EOF(F) do
  begin
    ReadLn(F, Num);
    Inc(Count);
    Sum := Sum + Num;
    if Num < Min then Min := Num;
    if Num > Max then Max := Num;
  end;
  
  CloseFile(F);
  
  WriteLn('=== สถิติตัวเลขในไฟล์ ===');
  WriteLn('จำนวน : ', Count);
  WriteLn('ต่ำสุด : ', Min:0:2);
  WriteLn('สูงสุด : ', Max:0:2);
  WriteLn('รวม   : ', Sum:0:2);
  WriteLn('เฉลี่ย : ', Sum/Count:0:4);
  
  ReadLn;
end.
```

---

## 12.3 Typed Files (ไฟล์ Binary ชนิดเดียว)

Typed files เก็บข้อมูลชนิดเดียวกันในรูปแบบ binary มีประสิทธิภาพสูงและรองรับ random access

### ตัวอย่างที่ 6: เขียนและอ่าน Typed File

```pascal
program TypedFileDemo;

{$mode objfpc}{$H+}

type
  TStudent = record
    ID    : Integer;
    Name  : String[40];
    Score : Real;
    Grade : Char;
  end;

var
  F       : File of TStudent;
  Student : TStudent;

procedure WriteStudents;
const
  Data: array[1..5] of record
    ID: Integer; Name: String[40]; Score: Real; Grade: Char;
  end = (
    (ID: 1; Name: 'สมชาย ใจดี'; Score: 85.5; Grade: 'A'),
    (ID: 2; Name: 'สมหญิง รักเรียน'; Score: 72.0; Grade: 'B'),
    (ID: 3; Name: 'อนุชา สมาร์ท'; Score: 91.5; Grade: 'A'),
    (ID: 4; Name: 'วิภา มีสุข'; Score: 63.0; Grade: 'C'),
    (ID: 5; Name: 'ธนพล เก่งมาก'; Score: 78.5; Grade: 'B')
  );
  i : Integer;
begin
  AssignFile(F, 'students.dat');
  Rewrite(F);
  
  for i := 1 to 5 do
  begin
    Student.ID    := Data[i].ID;
    Student.Name  := Data[i].Name;
    Student.Score := Data[i].Score;
    Student.Grade := Data[i].Grade;
    Write(F, Student);
  end;
  
  CloseFile(F);
  WriteLn('บันทึกข้อมูล 5 คน สำเร็จ');
end;

procedure ReadStudents;
begin
  AssignFile(F, 'students.dat');
  Reset(F);
  
  WriteLn('=== ข้อมูลนักเรียน ===');
  WriteLn(Format('%-4s %-20s %-8s %-5s', ['ID', 'ชื่อ', 'คะแนน', 'เกรด']));
  WriteLn(StringOfChar('-', 42));
  
  while not EOF(F) do
  begin
    Read(F, Student);
    WriteLn(Format('%-4d %-20s %-8.2f %-5s',
      [Student.ID, Student.Name, Student.Score, Student.Grade]));
  end;
  
  WriteLn('จำนวนระเบียน: ', FileSize(F));
  CloseFile(F);
end;

begin
  WriteStudents;
  WriteLn;
  ReadStudents;
  ReadLn;
end.
```

### ตัวอย่างที่ 7: Random Access กับ Typed File

```pascal
program RandomAccessFile;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TEmployee = record
    EmpID    : Integer;
    Name     : String[50];
    Salary   : Real;
    Position : String[30];
    IsActive : Boolean;
  end;

var
  F   : File of TEmployee;
  Emp : TEmployee;

procedure CreateFile;
var
  Emps : array[1..6] of TEmployee;
  i    : Integer;
begin
  Emps[1].EmpID := 1001; Emps[1].Name := 'สมชาย';  Emps[1].Salary := 35000;
  Emps[1].Position := 'Developer'; Emps[1].IsActive := True;
  
  Emps[2].EmpID := 1002; Emps[2].Name := 'สมหญิง'; Emps[2].Salary := 42000;
  Emps[2].Position := 'Manager';   Emps[2].IsActive := True;
  
  Emps[3].EmpID := 1003; Emps[3].Name := 'อนุชา';  Emps[3].Salary := 28000;
  Emps[3].Position := 'Tester';    Emps[3].IsActive := True;
  
  Emps[4].EmpID := 1004; Emps[4].Name := 'วิภา';   Emps[4].Salary := 55000;
  Emps[4].Position := 'Architect';  Emps[4].IsActive := True;
  
  Emps[5].EmpID := 1005; Emps[5].Name := 'ธนพล';  Emps[5].Salary := 38000;
  Emps[5].Position := 'DevOps';    Emps[5].IsActive := True;
  
  Emps[6].EmpID := 1006; Emps[6].Name := 'นภาพร';  Emps[6].Salary := 31000;
  Emps[6].Position := 'Designer';  Emps[6].IsActive := True;
  
  AssignFile(F, 'employees.dat');
  Rewrite(F);
  for i := 1 to 6 do Write(F, Emps[i]);
  CloseFile(F);
end;

// ค้นหาพนักงานด้วย Seek
function FindEmployee(EmpID: Integer; var Result: TEmployee): Boolean;
var
  i    : LongInt;
  Size : LongInt;
begin
  FindEmployee := False;
  AssignFile(F, 'employees.dat');
  Reset(F);
  Size := FileSize(F);
  
  for i := 0 to Size - 1 do
  begin
    Seek(F, i);
    Read(F, Emp);
    if Emp.EmpID = EmpID then
    begin
      Result := Emp;
      FindEmployee := True;
      CloseFile(F);
      Exit;
    end;
  end;
  
  CloseFile(F);
end;

// อัปเดตเงินเดือน
procedure UpdateSalary(EmpID: Integer; NewSalary: Real);
var
  i    : LongInt;
  Size : LongInt;
begin
  AssignFile(F, 'employees.dat');
  Reset(F);
  Size := FileSize(F);
  
  for i := 0 to Size - 1 do
  begin
    Seek(F, i);
    Read(F, Emp);
    if Emp.EmpID = EmpID then
    begin
      Emp.Salary := NewSalary;
      Seek(F, i);
      Write(F, Emp);
      WriteLn('อัปเดตเงินเดือน EmpID ', EmpID, ' เป็น ', NewSalary:0:2, ' สำเร็จ');
      CloseFile(F);
      Exit;
    end;
  end;
  
  WriteLn('ไม่พบพนักงาน EmpID: ', EmpID);
  CloseFile(F);
end;

// เพิ่มพนักงานใหม่
procedure AddEmployee(EmpID: Integer; Name: String; Salary: Real;
                      Position: String);
var
  NewEmp : TEmployee;
  Size   : LongInt;
begin
  NewEmp.EmpID    := EmpID;
  NewEmp.Name     := Name;
  NewEmp.Salary   := Salary;
  NewEmp.Position := Position;
  NewEmp.IsActive := True;
  
  AssignFile(F, 'employees.dat');
  Reset(F);
  Size := FileSize(F);
  Seek(F, Size); // ไปที่ท้ายไฟล์
  Write(F, NewEmp);
  CloseFile(F);
  WriteLn('เพิ่มพนักงาน: ', Name, ' สำเร็จ');
end;

// แสดงทั้งหมด
procedure DisplayAll;
var
  i    : LongInt;
  Size : LongInt;
begin
  AssignFile(F, 'employees.dat');
  Reset(F);
  Size := FileSize(F);
  
  WriteLn('=== พนักงานทั้งหมด (', Size, ' คน) ===');
  WriteLn(Format('%-6s %-15s %-12s %-15s %-6s',
    ['EmpID', 'ชื่อ', 'เงินเดือน', 'ตำแหน่ง', 'สถานะ']));
  WriteLn(StringOfChar('-', 60));
  
  for i := 0 to Size - 1 do
  begin
    Seek(F, i);
    Read(F, Emp);
    Write(Format('%-6d %-15s %-12.2f %-15s ',
      [Emp.EmpID, Emp.Name, Emp.Salary, Emp.Position]));
    if Emp.IsActive then WriteLn('เปิด') else WriteLn('ปิด');
  end;
  
  CloseFile(F);
end;

var
  Found    : TEmployee;

begin
  CreateFile;
  DisplayAll;
  WriteLn;
  
  // ค้นหา
  if FindEmployee(1003, Found) then
  begin
    WriteLn('พบ: ', Found.Name, ' ตำแหน่ง: ', Found.Position,
            ' เงินเดือน: ', Found.Salary:0:2);
  end;
  WriteLn;
  
  // อัปเดต
  UpdateSalary(1003, 32000);
  WriteLn;
  
  // เพิ่มใหม่
  AddEmployee(1007, 'สมศักดิ์', 45000, 'Lead Dev');
  WriteLn;
  
  DisplayAll;
  ReadLn;
end.
```

---

## 12.4 Untyped Files

Untyped files ใช้สำหรับการอ่าน/เขียนข้อมูล binary ดิบในรูปแบบ blocks

### ตัวอย่างที่ 8: คัดลอกไฟล์ด้วย Untyped File

```pascal
program CopyFileDemo;

{$mode objfpc}{$H+}

uses SysUtils;

function CopyFile(SrcFile, DstFile: String): Boolean;
const
  BUFFER_SIZE = 4096;
var
  Src, Dst : File;
  Buffer   : array[0..4095] of Byte;
  BytesRead: LongInt;
begin
  Result := False;
  
  if not FileExists(SrcFile) then
  begin
    WriteLn('ไม่พบไฟล์ต้นทาง: ', SrcFile);
    Exit;
  end;
  
  AssignFile(Src, SrcFile);
  AssignFile(Dst, DstFile);
  
  {$I-}
  Reset(Src, 1);   // block size = 1 byte
  Rewrite(Dst, 1);
  {$I+}
  
  if IOResult <> 0 then
  begin
    WriteLn('เปิดไฟล์ไม่ได้');
    Exit;
  end;
  
  repeat
    BlockRead(Src, Buffer, BUFFER_SIZE, BytesRead);
    if BytesRead > 0 then
      BlockWrite(Dst, Buffer, BytesRead);
  until BytesRead < BUFFER_SIZE;
  
  CloseFile(Src);
  CloseFile(Dst);
  
  Result := True;
end;

function GetFileSize(FileName: String): LongInt;
var
  F : File;
begin
  AssignFile(F, FileName);
  {$I-}
  Reset(F, 1);
  {$I+}
  if IOResult = 0 then
  begin
    Result := FileSize(F);
    CloseFile(F);
  end
  else
    Result := -1;
end;

var
  F    : TextFile;
  i    : Integer;

begin
  // สร้างไฟล์ต้นทาง
  AssignFile(F, 'source.txt');
  Rewrite(F);
  for i := 1 to 20 do
    WriteLn(F, 'บรรทัดที่ ', i, ': ข้อมูลทดสอบการคัดลอกไฟล์');
  CloseFile(F);
  
  WriteLn('ขนาดไฟล์ต้นทาง: ', GetFileSize('source.txt'), ' bytes');
  
  if CopyFile('source.txt', 'destination.txt') then
  begin
    WriteLn('คัดลอกไฟล์สำเร็จ!');
    WriteLn('ขนาดไฟล์ปลายทาง: ', GetFileSize('destination.txt'), ' bytes');
  end;
  
  ReadLn;
end.
```

### ตัวอย่างที่ 9: เปรียบเทียบไฟล์สองไฟล์

```pascal
program CompareFiles;

{$mode objfpc}{$H+}

uses SysUtils;

function FilesAreEqual(File1, File2: String): Boolean;
const
  BUF = 1024;
var
  F1, F2     : File;
  Buf1, Buf2 : array[0..1023] of Byte;
  R1, R2     : LongInt;
begin
  Result := False;
  
  if GetFileSize(File1) <> GetFileSize(File2) then Exit; // ขนาดต่างกัน
  
  AssignFile(F1, File1); Reset(F1, 1);
  AssignFile(F2, File2); Reset(F2, 1);
  
  Result := True;
  repeat
    BlockRead(F1, Buf1, BUF, R1);
    BlockRead(F2, Buf2, BUF, R2);
    
    if R1 <> R2 then begin Result := False; Break; end;
    
    if not CompareMem(@Buf1, @Buf2, R1) then
    begin
      Result := False;
      Break;
    end;
  until R1 = 0;
  
  CloseFile(F1);
  CloseFile(F2);
end;

function GetFileSize(FName: String): Int64;
var
  F : File;
begin
  AssignFile(F, FName);
  {$I-} Reset(F, 1); {$I+}
  if IOResult = 0 then
  begin
    Result := FileSize(F);
    CloseFile(F);
  end else Result := -1;
end;

var
  F  : TextFile;

begin
  // สร้างไฟล์เหมือนกัน
  AssignFile(F, 'file_a.txt'); Rewrite(F);
  WriteLn(F, 'ข้อมูลเหมือนกัน'); CloseFile(F);
  
  AssignFile(F, 'file_b.txt'); Rewrite(F);
  WriteLn(F, 'ข้อมูลเหมือนกัน'); CloseFile(F);
  
  // สร้างไฟล์ต่างกัน
  AssignFile(F, 'file_c.txt'); Rewrite(F);
  WriteLn(F, 'ข้อมูลต่างกัน!!!'); CloseFile(F);
  
  if FilesAreEqual('file_a.txt', 'file_b.txt') then
    WriteLn('file_a.txt และ file_b.txt: เหมือนกัน')
  else
    WriteLn('file_a.txt และ file_b.txt: ต่างกัน');
    
  if FilesAreEqual('file_a.txt', 'file_c.txt') then
    WriteLn('file_a.txt และ file_c.txt: เหมือนกัน')
  else
    WriteLn('file_a.txt และ file_c.txt: ต่างกัน');
  
  ReadLn;
end.
```

---

## 12.5 File Seeking (Seek, FilePos, FileSize)

### ตัวอย่างที่ 10: การใช้ Seek

```pascal
program SeekDemo;

{$mode objfpc}{$H+}

type
  TRecord = record
    ID   : Integer;
    Name : String[20];
    Value: Real;
  end;

var
  F : File of TRecord;
  R : TRecord;
  i : Integer;

begin
  // สร้างไฟล์
  AssignFile(F, 'seek_test.dat');
  Rewrite(F);
  
  for i := 1 to 10 do
  begin
    R.ID    := i;
    R.Name  := 'Item ' + IntToStr(i);
    R.Value := i * 1.5;
    Write(F, R);
  end;
  
  WriteLn('สร้างไฟล์ 10 records สำเร็จ');
  WriteLn('ขนาดไฟล์: ', FileSize(F), ' records');
  
  // ไปที่ record ที่ 5 (index เริ่มที่ 0)
  Seek(F, 4);
  Read(F, R);
  WriteLn('Record ที่ 5: ID=', R.ID, ' Name=', R.Name, ' Value=', R.Value:0:2);
  WriteLn('ตำแหน่งปัจจุบัน: ', FilePos(F));
  
  // ไปที่ record สุดท้าย
  Seek(F, FileSize(F) - 1);
  Read(F, R);
  WriteLn('Record สุดท้าย: ID=', R.ID, ' Name=', R.Name);
  
  // ไปที่ต้นไฟล์
  Seek(F, 0);
  Read(F, R);
  WriteLn('Record แรก: ID=', R.ID, ' Name=', R.Name);
  
  // แก้ไข record ที่ 3
  Seek(F, 2);
  Read(F, R);
  R.Name  := 'แก้ไขแล้ว';
  R.Value := 999.99;
  Seek(F, 2);
  Write(F, R);
  WriteLn('แก้ไข record ที่ 3 สำเร็จ');
  
  // อ่านและแสดงทั้งหมด
  Seek(F, 0);
  WriteLn('=== ข้อมูลในไฟล์ ===');
  while not EOF(F) do
  begin
    Read(F, R);
    WriteLn(Format('%-4d %-15s %.2f', [R.ID, R.Name, R.Value]));
  end;
  
  CloseFile(F);
  ReadLn;
end.
```

---

## 12.6 Directory Operations

### ตัวอย่างที่ 11: ค้นหาไฟล์ด้วย FindFirst/FindNext

```pascal
program FindFilesDemo;

{$mode objfpc}{$H+}

uses SysUtils;

procedure ListFiles(Path, Mask: String);
var
  SR       : TSearchRec;
  Count    : Integer;
  TotalSize: Int64;
begin
  Count     := 0;
  TotalSize := 0;
  
  WriteLn('ไฟล์ในโฟลเดอร์: ', Path);
  WriteLn('Mask: ', Mask);
  WriteLn(Format('%-40s %-12s %-20s', ['ชื่อไฟล์', 'ขนาด (bytes)', 'วันที่แก้ไข']));
  WriteLn(StringOfChar('-', 75));
  
  if FindFirst(IncludeTrailingPathDelimiter(Path) + Mask,
               faAnyFile - faDirectory, SR) = 0 then
  begin
    repeat
      Inc(Count);
      TotalSize := TotalSize + SR.Size;
      WriteLn(Format('%-40s %-12d %-20s',
        [SR.Name, SR.Size,
         FormatDateTime('dd/mm/yyyy hh:nn:ss', FileDateToDateTime(SR.Time))]));
    until FindNext(SR) <> 0;
    FindClose(SR);
  end;
  
  WriteLn(StringOfChar('-', 75));
  WriteLn('รวม ', Count, ' ไฟล์, ขนาดรวม: ', TotalSize, ' bytes');
end;

procedure ListDirectories(Path: String);
var
  SR    : TSearchRec;
  Count : Integer;
begin
  Count := 0;
  WriteLn('โฟลเดอร์ย่อยใน: ', Path);
  
  if FindFirst(IncludeTrailingPathDelimiter(Path) + '*',
               faDirectory, SR) = 0 then
  begin
    repeat
      if (SR.Attr and faDirectory) = faDirectory then
        if (SR.Name <> '.') and (SR.Name <> '..') then
        begin
          Inc(Count);
          WriteLn('  [DIR] ', SR.Name);
        end;
    until FindNext(SR) <> 0;
    FindClose(SR);
  end;
  
  WriteLn('รวม ', Count, ' โฟลเดอร์');
end;

var
  F : TextFile;
  i : Integer;

begin
  // สร้างไฟล์ทดสอบ
  for i := 1 to 3 do
  begin
    AssignFile(F, 'test_file_' + IntToStr(i) + '.txt');
    Rewrite(F);
    WriteLn(F, 'ไฟล์ทดสอบที่ ', i);
    CloseFile(F);
  end;
  
  ListFiles('.', '*.txt');
  WriteLn;
  ListDirectories('.');
  
  ReadLn;
end.
```

### ตัวอย่างที่ 12: จัดการโฟลเดอร์

```pascal
program DirectoryManagement;

{$mode objfpc}{$H+}

uses SysUtils;

procedure CreateDirectoryTree;
begin
  // สร้างโฟลเดอร์
  if not DirectoryExists('test_dir') then
    MkDir('test_dir');
  
  if not DirectoryExists('test_dir/sub1') then
    MkDir('test_dir/sub1');
    
  if not DirectoryExists('test_dir/sub2') then
    MkDir('test_dir/sub2');
    
  WriteLn('สร้างโฟลเดอร์สำเร็จ');
end;

procedure CreateFilesInDirs;
var
  F : TextFile;
begin
  AssignFile(F, 'test_dir/main.txt');
  Rewrite(F); WriteLn(F, 'ไฟล์หลัก'); CloseFile(F);
  
  AssignFile(F, 'test_dir/sub1/file1.txt');
  Rewrite(F); WriteLn(F, 'ไฟล์ใน sub1'); CloseFile(F);
  
  AssignFile(F, 'test_dir/sub2/file2.txt');
  Rewrite(F); WriteLn(F, 'ไฟล์ใน sub2'); CloseFile(F);
  
  WriteLn('สร้างไฟล์สำเร็จ');
end;

procedure RenameDemo;
begin
  // เปลี่ยนชื่อไฟล์
  if FileExists('test_dir/main.txt') then
  begin
    RenameFile('test_dir/main.txt', 'test_dir/renamed.txt');
    WriteLn('เปลี่ยนชื่อไฟล์: main.txt -> renamed.txt');
  end;
end;

procedure ShowInfo;
begin
  WriteLn('=== ข้อมูลโฟลเดอร์ ===');
  WriteLn('test_dir มีอยู่: ', DirectoryExists('test_dir'));
  WriteLn('test_dir/sub1 มีอยู่: ', DirectoryExists('test_dir/sub1'));
  WriteLn('test_dir/renamed.txt: ', FileExists('test_dir/renamed.txt'));
  
  WriteLn('ขนาดไฟล์ renamed.txt: ', FileSize('test_dir/renamed.txt'),
          ' bytes'); // ใช้ SysUtils
end;

begin
  CreateDirectoryTree;
  CreateFilesInDirs;
  RenameDemo;
  ShowInfo;
  ReadLn;
end.
```

---

## 12.7 SysUtils File Functions

### ตัวอย่างที่ 13: ฟังก์ชันจาก SysUtils

```pascal
program SysUtilsFiles;

{$mode objfpc}{$H+}

uses SysUtils;

procedure DemoSysUtils;
var
  F        : TextFile;
  FileName : String;
  Path     : String;
begin
  FileName := 'demo.txt';
  
  // สร้างไฟล์
  AssignFile(F, FileName);
  Rewrite(F);
  WriteLn(F, 'ทดสอบ SysUtils');
  CloseFile(F);
  
  WriteLn('=== SysUtils File Functions ===');
  
  // ตรวจสอบการมีอยู่ของไฟล์
  WriteLn('FileExists: ', FileExists(FileName));
  
  // ข้อมูลไฟล์
  WriteLn('FileSize: ', FileSize(FileName), ' bytes');
  
  // Path functions
  Path := '/home/user/lazarus_pascal_course/demo.txt';
  WriteLn('ExtractFilePath  : ', ExtractFilePath(Path));
  WriteLn('ExtractFileName  : ', ExtractFileName(Path));
  WriteLn('ExtractFileExt   : ', ExtractFileExt(Path));
  WriteLn('ExtractFileNameOnly: ', ChangeFileExt(ExtractFileName(Path), ''));
  
  // เปลี่ยน extension
  WriteLn('ChangeFileExt: ', ChangeFileExt(FileName, '.bak'));
  
  // ลบไฟล์
  DeleteFile('demo.txt');
  WriteLn('ลบไฟล์แล้ว');
  WriteLn('FileExists หลังลบ: ', FileExists('demo.txt'));
  
  // GetCurrentDir
  WriteLn('Current Directory: ', GetCurrentDir);
end;

begin
  DemoSysUtils;
  ReadLn;
end.
```

---

## 12.8 TStringList กับ Files

`TStringList` เป็น class ที่สะดวกมากสำหรับจัดการไฟล์ข้อความ

### ตัวอย่างที่ 14: TStringList พื้นฐาน

```pascal
program StringListFiles;

{$mode objfpc}{$H+}

uses SysUtils, Classes;

var
  SL : TStringList;
  i  : Integer;

begin
  SL := TStringList.Create;
  try
    // เพิ่มข้อมูล
    SL.Add('บรรทัดที่ 1: ข้อมูลแรก');
    SL.Add('บรรทัดที่ 2: ข้อมูลที่สอง');
    SL.Add('บรรทัดที่ 3: ข้อมูลที่สาม');
    SL.Add('บรรทัดที่ 4: ข้อมูลที่สี่');
    
    // บันทึกลงไฟล์
    SL.SaveToFile('stringlist.txt');
    WriteLn('บันทึกไฟล์สำเร็จ');
    
    // ล้างข้อมูล
    SL.Clear;
    WriteLn('จำนวนหลังล้าง: ', SL.Count);
    
    // โหลดจากไฟล์
    SL.LoadFromFile('stringlist.txt');
    WriteLn('โหลดไฟล์: ', SL.Count, ' บรรทัด');
    
    for i := 0 to SL.Count - 1 do
      WriteLn(Format('%3d: %s', [i + 1, SL[i]]));
      
    // ค้นหา
    WriteLn;
    WriteLn('IndexOf "บรรทัดที่ 2": ', SL.IndexOf('บรรทัดที่ 2: ข้อมูลที่สอง'));
    
    // เรียงลำดับ
    SL.Sort;
    WriteLn;
    WriteLn('หลังเรียง:');
    for i := 0 to SL.Count - 1 do
      WriteLn('  ', SL[i]);
      
  finally
    SL.Free;
  end;
  
  ReadLn;
end.
```

### ตัวอย่างที่ 15: TStringList กับ Key=Value

```pascal
program StringListKeyValue;

{$mode objfpc}{$H+}

uses SysUtils, Classes;

var
  Config : TStringList;

procedure LoadConfig(FileName: String);
begin
  Config := TStringList.Create;
  Config.NameValueSeparator := '=';
  
  if FileExists(FileName) then
    Config.LoadFromFile(FileName)
  else
  begin
    // ค่าเริ่มต้น
    Config.Values['AppName']    := 'My Application';
    Config.Values['Version']    := '1.0.0';
    Config.Values['MaxUsers']   := '100';
    Config.Values['Language']   := 'th';
    Config.Values['DebugMode']  := 'false';
    Config.Values['DatabaseURL']:= 'localhost:5432/mydb';
    Config.SaveToFile(FileName);
    WriteLn('สร้างไฟล์ config ใหม่');
  end;
end;

procedure ShowConfig;
var
  i : Integer;
begin
  WriteLn('=== Configuration ===');
  for i := 0 to Config.Count - 1 do
    WriteLn(Config[i]);
end;

procedure UpdateConfig(Key, Value: String);
begin
  Config.Values[Key] := Value;
  WriteLn('อัปเดต: ', Key, ' = ', Value);
end;

begin
  LoadConfig('app.config');
  ShowConfig;
  WriteLn;
  
  // อ่านค่า
  WriteLn('AppName: ', Config.Values['AppName']);
  WriteLn('Version: ', Config.Values['Version']);
  WriteLn('Language: ', Config.Values['Language']);
  WriteLn;
  
  // แก้ไข
  UpdateConfig('Version', '1.0.1');
  UpdateConfig('DebugMode', 'true');
  Config.SaveToFile('app.config');
  
  WriteLn;
  ShowConfig;
  
  Config.Free;
  ReadLn;
end.
```

---

## 12.9 Stream-based I/O (TFileStream)

### ตัวอย่างที่ 16: TFileStream พื้นฐาน

```pascal
program FileStreamDemo;

{$mode objfpc}{$H+}

uses SysUtils, Classes;

procedure WriteWithStream(FileName: String);
var
  Stream : TFileStream;
  S      : String;
  B      : Byte;
  I      : Integer;
  R      : Real;
begin
  Stream := TFileStream.Create(FileName, fmCreate);
  try
    // เขียน string (ต้องเขียนความยาวก่อน)
    S := 'Pascal Stream Demo';
    B := Length(S);
    Stream.Write(B, SizeOf(B));      // เขียนความยาว
    Stream.Write(S[1], B);            // เขียนข้อมูล
    
    // เขียน integer
    I := 12345;
    Stream.Write(I, SizeOf(I));
    
    // เขียน real
    R := 3.14159;
    Stream.Write(R, SizeOf(R));
    
    WriteLn('เขียน stream สำเร็จ');
    WriteLn('ขนาด: ', Stream.Size, ' bytes');
  finally
    Stream.Free;
  end;
end;

procedure ReadWithStream(FileName: String);
var
  Stream : TFileStream;
  S      : String;
  B      : Byte;
  I      : Integer;
  R      : Real;
begin
  Stream := TFileStream.Create(FileName, fmOpenRead);
  try
    // อ่าน string
    Stream.Read(B, SizeOf(B));       // อ่านความยาว
    SetLength(S, B);
    Stream.Read(S[1], B);             // อ่านข้อมูล
    
    // อ่าน integer
    Stream.Read(I, SizeOf(I));
    
    // อ่าน real
    Stream.Read(R, SizeOf(R));
    
    WriteLn('=== อ่านจาก Stream ===');
    WriteLn('String: ', S);
    WriteLn('Integer: ', I);
    WriteLn('Real: ', R:0:5);
  finally
    Stream.Free;
  end;
end;

begin
  WriteWithStream('demo_stream.bin');
  ReadWithStream('demo_stream.bin');
  ReadLn;
end.
```

### ตัวอย่างที่ 17: TMemoryStream - ประมวลผลในหน่วยความจำ

```pascal
program MemoryStreamDemo;

{$mode objfpc}{$H+}

uses SysUtils, Classes;

procedure CompressData(InputFile, OutputFile: String);
var
  MS    : TMemoryStream;
  FS    : TFileStream;
  SL    : TStringList;
  i     : Integer;
  Line  : String;
begin
  // อ่านข้อมูล
  SL := TStringList.Create;
  SL.LoadFromFile(InputFile);
  
  MS := TMemoryStream.Create;
  
  // จำลองการบีบอัด (ในที่นี้แค่ demo การใช้ MemoryStream)
  for i := 0 to SL.Count - 1 do
  begin
    Line := SL[i] + #13#10;
    MS.Write(PChar(Line)^, Length(Line));
  end;
  
  // บันทึกจาก memory ไปยังไฟล์
  MS.Position := 0;
  FS := TFileStream.Create(OutputFile, fmCreate);
  FS.CopyFrom(MS, MS.Size);
  
  WriteLn('ขนาดต้นฉบับ: ', MS.Size, ' bytes');
  WriteLn('บันทึกลงไฟล์: ', OutputFile);
  
  FS.Free;
  MS.Free;
  SL.Free;
end;

begin
  // สร้างไฟล์ต้นทาง
  var SL := TStringList.Create;
  for var i := 1 to 10 do
    SL.Add('บรรทัด ' + IntToStr(i) + ': ข้อมูลทดสอบ MemoryStream');
  SL.SaveToFile('input.txt');
  SL.Free;
  
  CompressData('input.txt', 'output_stream.txt');
  
  // อ่านกลับ
  SL := TStringList.Create;
  SL.LoadFromFile('output_stream.txt');
  WriteLn('อ่านกลับ: ', SL.Count, ' บรรทัด');
  SL.Free;
  
  ReadLn;
end.
```

---

## 12.10 Error Handling กับ Files

### ตัวอย่างที่ 18: การจัดการ Error แบบต่างๆ

```pascal
program FileErrorHandling;

{$mode objfpc}{$H+}

uses SysUtils;

// วิธีที่ 1: ใช้ {$I-} และ IOResult
procedure Method1_IOResult;
var
  F    : TextFile;
  Line : String;
begin
  AssignFile(F, 'nonexistent.txt');
  {$I-}
  Reset(F);
  {$I+}
  
  if IOResult <> 0 then
  begin
    WriteLn('วิธีที่ 1: ไม่พบไฟล์ (IOResult)');
    Exit;
  end;
  
  ReadLn(F, Line);
  CloseFile(F);
end;

// วิธีที่ 2: ตรวจสอบก่อนเปิด
procedure Method2_CheckFirst;
var
  F    : TextFile;
  Line : String;
  FileName : String;
begin
  FileName := 'nonexistent.txt';
  
  if not FileExists(FileName) then
  begin
    WriteLn('วิธีที่ 2: ไฟล์ "', FileName, '" ไม่มีอยู่');
    Exit;
  end;
  
  AssignFile(F, FileName);
  Reset(F);
  ReadLn(F, Line);
  CloseFile(F);
end;

// วิธีที่ 3: try...except
procedure Method3_TryExcept;
var
  F    : TextFile;
  Line : String;
begin
  try
    AssignFile(F, 'nonexistent.txt');
    Reset(F);
    ReadLn(F, Line);
    CloseFile(F);
  except
    on E: EInOutError do
      WriteLn('วิธีที่ 3: IO Error - ', E.Message);
    on E: Exception do
      WriteLn('วิธีที่ 3: Error - ', E.Message);
  end;
end;

// ฟังก์ชันอ่านไฟล์ที่ safe
function SafeReadFile(FileName: String; out Content: TStringList): Boolean;
begin
  Result  := False;
  Content := nil;
  
  if not FileExists(FileName) then
  begin
    WriteLn('ไม่พบไฟล์: ', FileName);
    Exit;
  end;
  
  try
    Content := TStringList.Create;
    Content.LoadFromFile(FileName);
    Result := True;
  except
    on E: Exception do
    begin
      WriteLn('Error อ่านไฟล์: ', E.Message);
      FreeAndNil(Content);
    end;
  end;
end;

// ฟังก์ชันเขียนไฟล์ที่ safe
function SafeWriteFile(FileName, Content: String): Boolean;
var
  F : TextFile;
begin
  Result := False;
  try
    AssignFile(F, FileName);
    Rewrite(F);
    WriteLn(F, Content);
    CloseFile(F);
    Result := True;
  except
    on E: Exception do
    begin
      WriteLn('Error เขียนไฟล์: ', E.Message);
      {$I-} CloseFile(F); {$I+}
    end;
  end;
end;

var
  SL : TStringList;

begin
  WriteLn('=== การจัดการ Error ===');
  Method1_IOResult;
  Method2_CheckFirst;
  Method3_TryExcept;
  WriteLn;
  
  // Safe write
  if SafeWriteFile('safe_test.txt', 'ข้อมูลทดสอบ') then
    WriteLn('เขียนไฟล์สำเร็จ')
  else
    WriteLn('เขียนไฟล์ล้มเหลว');
  
  // Safe read
  if SafeReadFile('safe_test.txt', SL) then
  begin
    WriteLn('อ่านสำเร็จ: ', SL.Count, ' บรรทัด');
    WriteLn('เนื้อหา: ', SL[0]);
    SL.Free;
  end;
  
  ReadLn;
end.
```

---

## 12.11 โปรแกรมตัวอย่าง: Log File Writer

```pascal
program LogFileWriter;

{$mode objfpc}{$H+}

uses SysUtils, Classes;

type
  TLogLevel = (llDebug, llInfo, llWarning, llError, llCritical);

  TLogger = record
    LogFile    : String;
    MaxSize    : LongInt;  // bytes
    LogToScreen: Boolean;
    MinLevel   : TLogLevel;
  end;

function LevelToStr(Level: TLogLevel): String;
begin
  case Level of
    llDebug   : Result := 'DEBUG   ';
    llInfo    : Result := 'INFO    ';
    llWarning : Result := 'WARNING ';
    llError   : Result := 'ERROR   ';
    llCritical: Result := 'CRITICAL';
  end;
end;

procedure Log(var Logger: TLogger; Level: TLogLevel; Msg: String);
var
  F       : TextFile;
  LogLine : String;
  Size    : LongInt;
begin
  if Level < Logger.MinLevel then Exit;
  
  LogLine := Format('[%s] [%s] %s',
    [FormatDateTime('dd/mm/yyyy hh:nn:ss', Now), LevelToStr(Level), Msg]);
  
  // แสดงหน้าจอ
  if Logger.LogToScreen then
    WriteLn(LogLine);
  
  // ตรวจสอบขนาดไฟล์
  if FileExists(Logger.LogFile) then
  begin
    AssignFile(F, Logger.LogFile);
    {$I-} Reset(F, 1); {$I+}
    if IOResult = 0 then
    begin
      Size := System.FileSize(F);
      CloseFile(F);
      
      // Rotate ถ้าใหญ่เกินไป
      if Size > Logger.MaxSize then
      begin
        RenameFile(Logger.LogFile, Logger.LogFile + '.bak');
        WriteLn('Rotated log file');
      end;
    end;
  end;
  
  // เขียน log
  AssignFile(F, Logger.LogFile);
  if FileExists(Logger.LogFile) then
    Append(F)
  else
    Rewrite(F);
  
  WriteLn(F, LogLine);
  CloseFile(F);
end;

procedure ShowLogs(FileName: String; LastN: Integer);
var
  SL  : TStringList;
  i   : Integer;
  Idx : Integer;
begin
  if not FileExists(FileName) then
  begin
    WriteLn('ไม่พบ log file');
    Exit;
  end;
  
  SL := TStringList.Create;
  try
    SL.LoadFromFile(FileName);
    
    WriteLn(Format('=== Log สุดท้าย %d บรรทัด ===', [LastN]));
    Idx := SL.Count - LastN;
    if Idx < 0 then Idx := 0;
    
    for i := Idx to SL.Count - 1 do
      WriteLn(SL[i]);
  finally
    SL.Free;
  end;
end;

function CountLogsByLevel(FileName: String; Level: TLogLevel): Integer;
var
  SL      : TStringList;
  i       : Integer;
  LevelStr: String;
begin
  Result   := 0;
  LevelStr := LevelToStr(Level);
  LevelStr := Trim(LevelStr);
  
  if not FileExists(FileName) then Exit;
  
  SL := TStringList.Create;
  try
    SL.LoadFromFile(FileName);
    for i := 0 to SL.Count - 1 do
      if Pos(LevelStr, SL[i]) > 0 then
        Inc(Result);
  finally
    SL.Free;
  end;
end;

var
  Logger : TLogger;

begin
  Logger.LogFile     := 'application.log';
  Logger.MaxSize     := 1024 * 1024; // 1 MB
  Logger.LogToScreen := True;
  Logger.MinLevel    := llDebug;
  
  WriteLn('=== เริ่ม Logging ===');
  WriteLn;
  
  Log(Logger, llInfo, 'เริ่มต้น Application');
  Log(Logger, llDebug, 'โหลด configuration สำเร็จ');
  Log(Logger, llInfo, 'เชื่อมต่อฐานข้อมูลสำเร็จ');
  Log(Logger, llWarning, 'Memory ใช้ไป 75%');
  Log(Logger, llInfo, 'ผู้ใช้ admin เข้าสู่ระบบ');
  Log(Logger, llDebug, 'Query: SELECT * FROM users');
  Log(Logger, llInfo, 'ดึงข้อมูล 150 รายการ');
  Log(Logger, llError, 'Connection timeout ครั้งที่ 1');
  Log(Logger, llWarning, 'Retry connection...');
  Log(Logger, llInfo, 'เชื่อมต่อใหม่สำเร็จ');
  Log(Logger, llInfo, 'ปิด Application');
  
  WriteLn;
  WriteLn('สถิติ log:');
  WriteLn('  INFO   : ', CountLogsByLevel('application.log', llInfo));
  WriteLn('  WARNING: ', CountLogsByLevel('application.log', llWarning));
  WriteLn('  ERROR  : ', CountLogsByLevel('application.log', llError));
  WriteLn('  DEBUG  : ', CountLogsByLevel('application.log', llDebug));
  
  ReadLn;
end.
```

---

## 12.12 โปรแกรมตัวอย่าง: CSV Processor

```pascal
program CSVProcessor;

{$mode objfpc}{$H+}

uses SysUtils, Classes, StrUtils;

type
  TCSVRow = array of String;
  TCSVData = array of TCSVRow;

function SplitCSV(Line: String; Delimiter: Char): TCSVRow;
var
  i        : Integer;
  InQuotes : Boolean;
  CurField : String;
  Fields   : TStringList;
begin
  Fields   := TStringList.Create;
  CurField := '';
  InQuotes := False;
  
  for i := 1 to Length(Line) do
  begin
    if Line[i] = '"' then
      InQuotes := not InQuotes
    else if (Line[i] = Delimiter) and not InQuotes then
    begin
      Fields.Add(Trim(CurField));
      CurField := '';
    end
    else
      CurField := CurField + Line[i];
  end;
  Fields.Add(Trim(CurField));
  
  SetLength(Result, Fields.Count);
  for i := 0 to Fields.Count - 1 do
    Result[i] := Fields[i];
  
  Fields.Free;
end;

procedure CreateSampleCSV(FileName: String);
var
  F : TextFile;
begin
  AssignFile(F, FileName);
  Rewrite(F);
  WriteLn(F, 'ID,Name,Department,Salary,JoinDate');
  WriteLn(F, '1001,สมชาย ใจดี,IT,35000,01/01/2020');
  WriteLn(F, '1002,สมหญิง รักเรียน,HR,42000,15/03/2019');
  WriteLn(F, '1003,อนุชา สมาร์ท,Sales,28000,01/07/2021');
  WriteLn(F, '1004,วิภา มีสุข,Finance,55000,20/02/2018');
  WriteLn(F, '1005,ธนพล เก่งมาก,IT,38000,10/11/2022');
  WriteLn(F, '1006,นภาพร ฉลาด,Marketing,31000,05/06/2021');
  WriteLn(F, '1007,สมศักดิ์ ดีมาก,IT,45000,01/01/2017');
  CloseFile(F);
  WriteLn('สร้าง CSV สำเร็จ: ', FileName);
end;

function LoadCSV(FileName: String; out Headers: TCSVRow; out Data: TCSVData): Boolean;
var
  SL  : TStringList;
  i   : Integer;
begin
  Result := False;
  if not FileExists(FileName) then Exit;
  
  SL := TStringList.Create;
  try
    SL.LoadFromFile(FileName);
    if SL.Count < 2 then Exit;
    
    Headers := SplitCSV(SL[0], ',');
    SetLength(Data, SL.Count - 1);
    
    for i := 1 to SL.Count - 1 do
      Data[i - 1] := SplitCSV(SL[i], ',');
    
    Result := True;
  finally
    SL.Free;
  end;
end;

procedure DisplayCSV(Headers: TCSVRow; Data: TCSVData);
var
  i, j : Integer;
begin
  // หัวตาราง
  for j := 0 to High(Headers) do
    Write(Format('%-15s', [Headers[j]]));
  WriteLn;
  WriteLn(StringOfChar('-', 15 * Length(Headers)));
  
  // ข้อมูล
  for i := 0 to High(Data) do
  begin
    for j := 0 to High(Data[i]) do
      Write(Format('%-15s', [Data[i][j]]));
    WriteLn;
  end;
  WriteLn('รวม ', Length(Data), ' rows');
end;

// กรองตามแผนก
procedure FilterByDept(Headers: TCSVRow; Data: TCSVData; Dept: String);
var
  i        : Integer;
  DeptCol  : Integer;
  NameCol  : Integer;
  SalCol   : Integer;
  Total    : Real;
  Count    : Integer;
begin
  // หาคอลัมน์
  DeptCol := -1;
  NameCol := -1;
  SalCol  := -1;
  
  for i := 0 to High(Headers) do
  begin
    if Headers[i] = 'Department' then DeptCol := i;
    if Headers[i] = 'Name'       then NameCol := i;
    if Headers[i] = 'Salary'     then SalCol  := i;
  end;
  
  if DeptCol < 0 then begin WriteLn('ไม่พบคอลัมน์ Department'); Exit; end;
  
  WriteLn('=== พนักงานแผนก: ', Dept, ' ===');
  Total := 0;
  Count := 0;
  
  for i := 0 to High(Data) do
    if (DeptCol < Length(Data[i])) and (Data[i][DeptCol] = Dept) then
    begin
      Inc(Count);
      if NameCol >= 0 then Write(Data[i][NameCol], '  ');
      if SalCol >= 0 then
      begin
        var Sal := StrToFloatDef(Data[i][SalCol], 0);
        Write('เงินเดือน: ', Sal:0:0);
        Total := Total + Sal;
      end;
      WriteLn;
    end;
  
  WriteLn('จำนวน: ', Count, ' คน  เงินเดือนรวม: ', Total:0:0);
end;

// คำนวณสถิติเงินเดือน
procedure SalaryStats(Headers: TCSVRow; Data: TCSVData);
var
  i      : Integer;
  SalCol : Integer;
  Sal    : Real;
  Total  : Real;
  Min, Max: Real;
  Count  : Integer;
begin
  SalCol := -1;
  for i := 0 to High(Headers) do
    if Headers[i] = 'Salary' then SalCol := i;
  
  if SalCol < 0 then Exit;
  
  Total := 0;
  Count := 0;
  Min   := 1e15;
  Max   := 0;
  
  for i := 0 to High(Data) do
    if SalCol < Length(Data[i]) then
    begin
      Sal := StrToFloatDef(Data[i][SalCol], 0);
      if Sal > 0 then
      begin
        Inc(Count);
        Total := Total + Sal;
        if Sal < Min then Min := Sal;
        if Sal > Max then Max := Sal;
      end;
    end;
  
  WriteLn('=== สถิติเงินเดือน ===');
  WriteLn('จำนวนพนักงาน : ', Count);
  WriteLn('เงินเดือนต่ำสุด: ', Min:0:0);
  WriteLn('เงินเดือนสูงสุด: ', Max:0:0);
  WriteLn('เงินเดือนเฉลี่ย: ', Total/Count:0:2);
  WriteLn('รวมค่าจ้างทั้งหมด: ', Total:0:0);
end;

// บันทึก CSV ใหม่
procedure SaveCSV(FileName: String; Headers: TCSVRow; Data: TCSVData);
var
  SL  : TStringList;
  i, j: Integer;
  Line: String;
begin
  SL := TStringList.Create;
  try
    // header
    Line := '';
    for j := 0 to High(Headers) do
    begin
      if j > 0 then Line := Line + ',';
      Line := Line + Headers[j];
    end;
    SL.Add(Line);
    
    // data
    for i := 0 to High(Data) do
    begin
      Line := '';
      for j := 0 to High(Data[i]) do
      begin
        if j > 0 then Line := Line + ',';
        Line := Line + Data[i][j];
      end;
      SL.Add(Line);
    end;
    
    SL.SaveToFile(FileName);
    WriteLn('บันทึก CSV: ', FileName, ' (', SL.Count - 1, ' rows)');
  finally
    SL.Free;
  end;
end;

var
  Headers : TCSVRow;
  Data    : TCSVData;

begin
  CreateSampleCSV('employees.csv');
  WriteLn;
  
  if LoadCSV('employees.csv', Headers, Data) then
  begin
    WriteLn('=== ข้อมูลพนักงานทั้งหมด ===');
    DisplayCSV(Headers, Data);
    WriteLn;
    
    FilterByDept(Headers, Data, 'IT');
    WriteLn;
    
    SalaryStats(Headers, Data);
    WriteLn;
    
    SaveCSV('employees_copy.csv', Headers, Data);
  end;
  
  ReadLn;
end.
```

---

## 12.13 โปรแกรมตัวอย่าง: File Manager

```pascal
program SimpleFileManager;

{$mode objfpc}{$H+}

uses SysUtils, Classes;

type
  TFileInfo = record
    Name    : String;
    Size    : Int64;
    IsDir   : Boolean;
    ModTime : TDateTime;
    Attr    : Integer;
  end;

var
  CurrentDir : String;
  FileList   : array[1..500] of TFileInfo;
  FileCount  : Integer;

function FormatSize(Size: Int64): String;
begin
  if Size < 1024 then
    Result := Format('%d B', [Size])
  else if Size < 1024 * 1024 then
    Result := Format('%.1f KB', [Size / 1024])
  else if Size < 1024 * 1024 * 1024 then
    Result := Format('%.1f MB', [Size / (1024 * 1024)])
  else
    Result := Format('%.2f GB', [Size / (1024 * 1024 * 1024)]);
end;

procedure ListDirectory(Path: String);
var
  SR : TSearchRec;
  i  : Integer;
begin
  FileCount  := 0;
  CurrentDir := Path;
  
  if FindFirst(IncludeTrailingPathDelimiter(Path) + '*',
               faAnyFile, SR) = 0 then
  begin
    repeat
      if FileCount < 500 then
      begin
        Inc(FileCount);
        FileList[FileCount].Name    := SR.Name;
        FileList[FileCount].Size    := SR.Size;
        FileList[FileCount].IsDir   := (SR.Attr and faDirectory) > 0;
        FileList[FileCount].ModTime := FileDateToDateTime(SR.Time);
        FileList[FileCount].Attr    := SR.Attr;
      end;
    until FindNext(SR) <> 0;
    FindClose(SR);
  end;
end;

procedure DisplayDirectory;
var
  i     : Integer;
  Dirs  : Integer;
  Files : Integer;
  Total : Int64;
begin
  Dirs  := 0;
  Files := 0;
  Total := 0;
  
  WriteLn('=== ', CurrentDir, ' ===');
  WriteLn(Format('%-5s %-30s %-12s %-20s',
    ['', 'ชื่อ', 'ขนาด', 'วันที่แก้ไข']));
  WriteLn(StringOfChar('-', 70));
  
  // แสดง directories ก่อน
  for i := 1 to FileCount do
    if FileList[i].IsDir then
    begin
      Inc(Dirs);
      if (FileList[i].Name <> '.') and (FileList[i].Name <> '..') then
        WriteLn(Format('%-5s %-30s %-12s %-20s',
          ['[DIR]', FileList[i].Name, '',
           FormatDateTime('dd/mm/yyyy hh:nn', FileList[i].ModTime)]));
    end;
  
  // แสดง files
  for i := 1 to FileCount do
    if not FileList[i].IsDir then
    begin
      Inc(Files);
      Total := Total + FileList[i].Size;
      WriteLn(Format('%-5s %-30s %-12s %-20s',
        ['', FileList[i].Name,
         FormatSize(FileList[i].Size),
         FormatDateTime('dd/mm/yyyy hh:nn', FileList[i].ModTime)]));
    end;
  
  WriteLn(StringOfChar('-', 70));
  WriteLn(Format('%d โฟลเดอร์, %d ไฟล์, รวม %s',
    [Dirs, Files, FormatSize(Total)]));
end;

procedure CopyFileWithProgress(Src, Dst: String);
var
  FSrc, FDst : File;
  Buffer     : array[0..65535] of Byte;
  Read       : LongInt;
  Total      : LongInt;
  Written    : LongInt;
  FileSize   : LongInt;
  Percent    : Integer;
begin
  if not FileExists(Src) then
  begin
    WriteLn('ไม่พบไฟล์ต้นทาง: ', Src);
    Exit;
  end;
  
  AssignFile(FSrc, Src);
  AssignFile(FDst, Dst);
  
  Reset(FSrc, 1);
  Rewrite(FDst, 1);
  
  FileSize := System.FileSize(FSrc);
  Total    := 0;
  
  Write('กำลังคัดลอก: ');
  
  repeat
    BlockRead(FSrc, Buffer, SizeOf(Buffer), Read);
    if Read > 0 then
    begin
      BlockWrite(FDst, Buffer, Read, Written);
      Total   := Total + Written;
      Percent := Round(Total * 100 / FileSize);
      Write(Percent, '% ');
    end;
  until Read = 0;
  
  WriteLn;
  WriteLn('คัดลอกสำเร็จ: ', Dst, ' (', FormatSize(Total), ')');
  
  CloseFile(FSrc);
  CloseFile(FDst);
end;

function DiskUsage(Path: String): Int64;
var
  SR    : TSearchRec;
  Total : Int64;
begin
  Total := 0;
  
  if FindFirst(IncludeTrailingPathDelimiter(Path) + '*',
               faAnyFile, SR) = 0 then
  begin
    repeat
      if (SR.Attr and faDirectory) > 0 then
      begin
        if (SR.Name <> '.') and (SR.Name <> '..') then
          Total := Total + DiskUsage(IncludeTrailingPathDelimiter(Path) + SR.Name);
      end
      else
        Total := Total + SR.Size;
    until FindNext(SR) <> 0;
    FindClose(SR);
  end;
  
  Result := Total;
end;

var
  F    : TextFile;
  i    : Integer;

begin
  // สร้างไฟล์ทดสอบ
  for i := 1 to 5 do
  begin
    AssignFile(F, 'file' + IntToStr(i) + '.txt');
    Rewrite(F);
    WriteLn(F, 'ไฟล์ทดสอบที่ ', i);
    CloseFile(F);
  end;
  
  ListDirectory('.');
  DisplayDirectory;
  WriteLn;
  
  // ทดสอบคัดลอก
  CopyFileWithProgress('file1.txt', 'file1_copy.txt');
  WriteLn;
  
  // ขนาดโฟลเดอร์
  WriteLn('ขนาดโฟลเดอร์ปัจจุบัน: ', FormatSize(DiskUsage('.')));
  
  ReadLn;
end.
```

---

## 12.14 แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนโปรแกรมรับข้อมูลนักเรียน 5 คน (ชื่อ, คะแนน) จากผู้ใช้
แล้วบันทึกลง text file และอ่านกลับมาแสดง

### แบบฝึกหัดที่ 2
เขียนโปรแกรม word counter ที่อ่านไฟล์ข้อความแล้วนับ:
- จำนวนบรรทัด
- จำนวนคำ
- จำนวนตัวอักษร
- คำที่ใช้บ่อยที่สุด 5 คำ

### แบบฝึกหัดที่ 3
สร้างระบบ Typed File สำหรับเก็บข้อมูลสินค้า
- เพิ่ม/ลบ/แก้ไข/ค้นหาสินค้า
- บันทึกและโหลดจากไฟล์

### แบบฝึกหัดที่ 4
เขียน Binary file editor ที่แสดง hex dump ของไฟล์
แสดง offset, hex values, และ ASCII ต่อบรรทัด

### แบบฝึกหัดที่ 5
สร้างโปรแกรม config file reader/writer ด้วย TStringList
รองรับ sections เหมือน .ini file เช่น [Database] Host=localhost

### แบบฝึกหัดที่ 6
เขียนโปรแกรม log analyzer ที่:
- อ่าน log file
- กรองตาม log level
- หา error messages
- สร้าง summary report

### แบบฝึกหัดที่ 7
สร้าง CSV import/export ที่:
- อ่าน CSV ข้อมูลนักเรียน
- คำนวณสถิติ
- export CSV ใหม่พร้อมคอลัมน์ grade

### แบบฝึกหัดที่ 8
เขียนโปรแกรม file backup ที่:
- คัดลอกไฟล์ไปยัง backup folder
- เพิ่ม timestamp ในชื่อ
- ลบ backups เก่าเกิน 7 วัน

### แบบฝึกหัดที่ 9
สร้าง TFileStream ที่เขียนและอ่าน Records โดยตรง
ทดสอบกับข้อมูลพนักงาน 10 คน

### แบบฝึกหัดที่ 10
เขียน recursive directory scanner ที่:
- แสดงโครงสร้างโฟลเดอร์เป็น tree
- นับจำนวนและขนาดรวม
- กรองตาม extension

### แบบฝึกหัดที่ 11
สร้าง simple database engine อย่างง่ายด้วย Typed File:
- Insert, Delete, Update, Select
- Primary key
- Index file สำหรับค้นหาเร็ว

### แบบฝึกหัดที่ 12
เขียนโปรแกรม diff tool อย่างง่ายที่เปรียบเทียบสองไฟล์
แสดงบรรทัดที่ต่างกัน

### แบบฝึกหัดที่ 13
สร้าง file encryption อย่างง่าย (XOR cipher)
เข้ารหัสและถอดรหัสไฟล์

### แบบฝึกหัดที่ 14
เขียน batch file processor ที่:
- อ่านรายการคำสั่งจากไฟล์
- ประมวลผลทีละคำสั่ง
- บันทึกผลลัพธ์ลงไฟล์

### แบบฝึกหัดที่ 15
สร้าง contact book ที่บันทึกลง binary file:
- CRUD operations
- Sort โดย field ต่างๆ
- Export เป็น CSV

### แบบฝึกหัดที่ 16
เขียน file monitor ที่ตรวจสอบการเปลี่ยนแปลงใน folder:
- ไฟล์ใหม่
- ไฟล์ถูกลบ
- ไฟล์ถูกแก้ไข

### แบบฝึกหัดที่ 17
สร้าง checksum tool:
- คำนวณ CRC32 หรือ MD5 ของไฟล์
- เปรียบเทียบ checksums

### แบบฝึกหัดที่ 18
เขียนโปรแกรม multi-file merge:
- รวมหลายไฟล์ text เป็นหนึ่ง
- เพิ่ม separator ระหว่างไฟล์
- ตัดบรรทัดซ้ำออก

### แบบฝึกหัดที่ 19
สร้าง structured binary format สำหรับเก็บข้อมูล
เขียน header, data blocks, และ footer

### แบบฝึกหัดที่ 20
เขียน full-featured text editor แบบ CLI:
- เปิด/บันทึกไฟล์
- แสดงเนื้อหา (page by page)
- Find & Replace
- แสดงหมายเลขบรรทัด

---

## สรุป

| ประเภทไฟล์ | คำสั่งหลัก | ใช้กับ |
|-----------|-----------|-------|
| Text | `TextFile`, `WriteLn`, `ReadLn` | ข้อความ, CSV, Config |
| Typed | `File of T`, `Read`, `Write` | Binary records |
| Untyped | `File`, `BlockRead`, `BlockWrite` | Raw binary, copy |
| TStringList | `LoadFromFile`, `SaveToFile` | ข้อความ, Key=Value |
| TFileStream | `Read`, `Write`, `Seek` | Binary, streaming |

**Best Practices:**
- ตรวจสอบ `FileExists` ก่อนเปิดเสมอ
- ใช้ `try...finally` เพื่อให้แน่ใจว่าไฟล์จะถูกปิด
- จัดการ IOError ด้วย `{$I-}...{$I+}` และ `IOResult`
- ปิดไฟล์ด้วย `CloseFile` ทุกครั้ง
