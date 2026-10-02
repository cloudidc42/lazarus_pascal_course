# Part 09 - สตริง (Strings)

## สารบัญ
1. [String Types ใน Pascal](#string-types)
2. [String Operations พื้นฐาน](#string-operations)
3. [String Comparison](#string-comparison)
4. [String Conversion](#string-conversion)
5. [String Formatting](#string-formatting)
6. [String Searching](#string-searching)
7. [Regular Expressions](#regular-expressions)
8. [String Builder Pattern](#string-builder-pattern)
9. [Unicode Strings](#unicode-strings)
10. [TStringList](#tstringlist)
11. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)
12. [แบบฝึกหัด 20 ข้อ](#แบบฝึกหัด)

---

## บทนำ

String (สตริง) คือลำดับของอักขระ เป็นหนึ่งในชนิดข้อมูลที่สำคัญที่สุดในการเขียนโปรแกรม Pascal/Lazarus มีชนิด string หลายประเภทให้เลือกใช้

---

## String Types

| ชนิด | คำอธิบาย | ขนาดสูงสุด |
|------|----------|-----------|
| `String` | String ทั่วไป (Long String) | ~2GB |
| `ShortString` | Short string เก่า | 255 อักขระ |
| `String[N]` | Short string กำหนดขนาด | N อักขระ |
| `AnsiString` | ANSI string (1 byte/char) | ~2GB |
| `WideString` | Unicode string (2 bytes/char) | ~1GB |
| `UnicodeString` | Unicode string | ~2GB |
| `PChar` | Pointer to char (C-style) | - |

### ตัวอย่างที่ 1: การประกาศ string ประเภทต่างๆ

```pascal
program StringTypes;
var
  S1: String;              // Long String (default)
  S2: String[20];          // Short String สูงสุด 20 ตัว
  S3: ShortString;         // Short String สูงสุด 255 ตัว
  S4: AnsiString;          // ANSI String
  S5: WideString;          // Wide String (UTF-16)
  S6: UnicodeString;       // Unicode String
begin
  S1 := 'Hello, World!';
  S2 := 'สวัสดี';           // ไทย (อาจมีปัญหาถ้า > 20 bytes)
  S3 := 'Short String';
  S4 := 'ANSI String';
  S5 := 'Wide String 🌍';  // รองรับ Unicode
  S6 := 'Unicode สวัสดี';
  
  WriteLn('S1 = ', S1);
  WriteLn('S2 = ', S2);
  WriteLn('S3 = ', S3);
  WriteLn('Length(S1) = ', Length(S1));
  WriteLn('SizeOf(S2) = ', SizeOf(S2));  // 21 bytes (20 + length byte)
end.
```

### ตัวอย่างที่ 2: String Character Access

```pascal
program StringCharAccess;
var
  S: String;
  I: Integer;
begin
  S := 'Pascal';
  
  WriteLn('String: ', S);
  WriteLn('ความยาว: ', Length(S));
  WriteLn('อักขระแรก: ', S[1]);  // Pascal strings เริ่ม index ที่ 1
  WriteLn('อักขระสุดท้าย: ', S[Length(S)]);
  
  // วนซ้ำทุกอักขระ
  Write('ทีละตัว: ');
  for I := 1 to Length(S) do
    Write(S[I], ' ');
  WriteLn;
  
  // แก้ไขอักขระ
  S[1] := 'p';
  WriteLn('หลังแก้ไข: ', S);
end.
```

---

## String Operations

### ตัวอย่างที่ 3: Length function

```pascal
program LengthExample;
var
  S: String;
begin
  S := 'Hello, World!';
  WriteLn('Length("', S, '") = ', Length(S));
  
  S := '';
  WriteLn('Length("") = ', Length(S));
  
  S := '   spaces   ';
  WriteLn('Length("', S, '") = ', Length(S));
  
  // หา length แบบต่างๆ
  WriteLn('High() = ', High(S));  // index สูงสุด = Length
  WriteLn('Low() = ', Low(S));    // index ต่ำสุด = 1
end.
```

### ตัวอย่างที่ 4: Copy function

```pascal
program CopyExample;
var
  S, Sub: String;
begin
  S := 'Hello, World!';
  
  // Copy(Source, Start, Count)
  Sub := Copy(S, 1, 5);    // 'Hello'
  WriteLn('Copy(S, 1, 5) = "', Sub, '"');
  
  Sub := Copy(S, 8, 5);    // 'World'
  WriteLn('Copy(S, 8, 5) = "', Sub, '"');
  
  Sub := Copy(S, 8, 100);  // 'World!' (จนสิ้นสุด)
  WriteLn('Copy(S, 8, 100) = "', Sub, '"');
  
  // ตัด prefix
  WriteLn('หลังตัด prefix: ', Copy(S, 8, MaxInt));
  
  // ตัด suffix
  WriteLn('หลังตัด suffix: ', Copy(S, 1, 5));
end.
```

### ตัวอย่างที่ 5: Pos function

```pascal
program PosExample;
var
  S, Sub: String;
  Position: Integer;
begin
  S := 'The quick brown fox jumps over the lazy dog';
  
  // Pos(SubString, Source)
  Position := Pos('fox', S);
  WriteLn('Pos("fox", S) = ', Position);  // 17
  
  Position := Pos('the', S);
  WriteLn('Pos("the", S) = ', Position);  // 32 (case sensitive)
  
  Position := Pos('cat', S);
  WriteLn('Pos("cat", S) = ', Position);  // 0 (ไม่พบ)
  
  // ตรวจสอบว่ามี substring หรือไม่
  if Pos('fox', S) > 0 then
    WriteLn('พบ "fox" ในสตริง')
  else
    WriteLn('ไม่พบ "fox"');
  
  // หา Pos ด้วยจุดเริ่มต้น (ใช้ PosEx)
  // PosEx('the', S, 1) = 1 (ตำแหน่งแรก)
  // PosEx('the', S, 2) = 32 (ตำแหน่งถัดไป)
end.
```

### ตัวอย่างที่ 6: Delete function

```pascal
program DeleteExample;
var
  S: String;
begin
  S := 'Hello, World!';
  WriteLn('ก่อน: "', S, '"');
  
  // Delete(Var String, Start, Count)
  Delete(S, 7, 7);  // ลบ ", World"
  WriteLn('หลัง Delete(S, 7, 7): "', S, '"');
  
  S := 'ABCDEFGH';
  Delete(S, 3, 2);  // ลบ 'CD'
  WriteLn('ABCDEFGH -> Delete(3, 2): "', S, '"');
  
  S := 'Hello World';
  // ลบอักขระตัวแรก
  Delete(S, 1, 1);
  WriteLn('ลบตัวแรก: "', S, '"');
  
  // ลบอักขระตัวสุดท้าย
  Delete(S, Length(S), 1);
  WriteLn('ลบตัวสุดท้าย: "', S, '"');
end.
```

### ตัวอย่างที่ 7: Insert function

```pascal
program InsertExample;
var
  S: String;
begin
  S := 'Hello World!';
  WriteLn('ก่อน: "', S, '"');
  
  // Insert(SubString, Var String, Position)
  Insert(', Beautiful', S, 6);
  WriteLn('หลัง Insert: "', S, '"');
  
  S := 'Pascal';
  Insert('Free', S, 1);  // ใส่ที่ต้น
  WriteLn(S);
  
  S := 'Hello';
  Insert('!', S, Length(S) + 1);  // ใส่ที่ปลาย
  WriteLn(S);
end.
```

### ตัวอย่างที่ 8: Concat และ String Concatenation

```pascal
program ConcatExample;
var
  S1, S2, S3: String;
begin
  S1 := 'Hello';
  S2 := ', ';
  S3 := 'World!';
  
  // วิธีที่ 1: ใช้ + operator
  WriteLn(S1 + S2 + S3);
  
  // วิธีที่ 2: ใช้ Concat function
  WriteLn(Concat(S1, S2, S3));
  
  // วิธีที่ 3: ใช้ := กับ +
  var Result := S1 + S2 + S3;
  WriteLn(Result);
  
  // ต่อกับตัวเลข (ต้องแปลงก่อน)
  var N := 42;
  WriteLn('ตัวเลข: ' + IntToStr(N));
  WriteLn(Format('ตัวเลข: %d', [N]));
end.
```

### ตัวอย่างที่ 9: UpperCase และ LowerCase

```pascal
program CaseConversion;
uses
  SysUtils;
var
  S: String;
begin
  S := 'Hello, World!';
  
  WriteLn('ต้นฉบับ:   ', S);
  WriteLn('UpperCase: ', UpperCase(S));
  WriteLn('LowerCase: ', LowerCase(S));
  
  // ใช้สำหรับ case-insensitive comparison
  var Input: String;
  Write('ป้อนคำ: ');
  ReadLn(Input);
  
  if LowerCase(Input) = 'yes' then
    WriteLn('คุณตอบใช่')
  else if LowerCase(Input) = 'no' then
    WriteLn('คุณตอบไม่')
  else
    WriteLn('ไม่ทราบคำตอบ');
end.
```

### ตัวอย่างที่ 10: Trim functions

```pascal
program TrimExample;
uses
  SysUtils;
var
  S: String;
begin
  S := '  Hello World  ';
  
  WriteLn('"', S, '"');
  WriteLn('Trim:      "', Trim(S), '"');
  WriteLn('TrimLeft:  "', TrimLeft(S), '"');
  WriteLn('TrimRight: "', TrimRight(S), '"');
  
  // ประยุกต์ใช้
  var UserInput: String;
  Write('ป้อนชื่อ: ');
  ReadLn(UserInput);
  UserInput := Trim(UserInput);  // ลบ spaces หน้า-หลัง
  
  if Length(UserInput) = 0 then
    WriteLn('ชื่อว่าง!')
  else
    WriteLn('ชื่อ: "', UserInput, '"');
end.
```

### ตัวอย่างที่ 11: StringReplace

```pascal
program StringReplaceExample;
uses
  SysUtils;
var
  S: String;
begin
  S := 'The cat sat on the mat. The cat is fat.';
  
  // StringReplace(Source, OldStr, NewStr, Flags)
  // rfReplaceAll = แทนทั้งหมด
  // rfIgnoreCase = ไม่สนตัวพิมพ์เล็ก/ใหญ่
  
  WriteLn('ต้นฉบับ: ', S);
  WriteLn('แทน "cat" ด้วย "dog": ');
  WriteLn(StringReplace(S, 'cat', 'dog', [rfReplaceAll]));
  
  WriteLn('แทน "The" (case insensitive): ');
  WriteLn(StringReplace(S, 'the', 'a', [rfReplaceAll, rfIgnoreCase]));
  
  // แทนตัวแรกอย่างเดียว
  WriteLn('แทนตัวแรกอย่างเดียว: ');
  WriteLn(StringReplace(S, 'cat', 'dog', []));
end.
```

### ตัวอย่างที่ 12: String ใน Array

```pascal
program StringArray;
var
  Names: array[1..5] of String;
  I: Integer;
  Search: String;
  Found: Boolean;
begin
  Names[1] := 'Alice';
  Names[2] := 'Bob';
  Names[3] := 'Charlie';
  Names[4] := 'David';
  Names[5] := 'Eve';
  
  // แสดงรายชื่อ
  for I := 1 to 5 do
    WriteLn(I, '. ', Names[I]);
  
  // ค้นหา
  Write('ค้นหาชื่อ: ');
  ReadLn(Search);
  
  Found := False;
  for I := 1 to 5 do
    if LowerCase(Names[I]) = LowerCase(Search) then
    begin
      WriteLn('พบ "', Names[I], '" ที่ตำแหน่ง ', I);
      Found := True;
      Break;
    end;
  
  if not Found then
    WriteLn('ไม่พบ "', Search, '"');
end.
```

---

## String Comparison

### ตัวอย่างที่ 13: การเปรียบเทียบ String

```pascal
program StringCompare;
uses
  SysUtils;
var
  S1, S2: String;
begin
  S1 := 'Apple';
  S2 := 'apple';
  
  // = เปรียบเทียบแบบ case sensitive
  if S1 = S2 then WriteLn('เท่ากัน (case sensitive)')
  else WriteLn('ไม่เท่ากัน (case sensitive)');
  
  // CompareStr: case sensitive
  // Returns: < 0 ถ้า S1 < S2, 0 ถ้าเท่ากัน, > 0 ถ้า S1 > S2
  WriteLn('CompareStr: ', CompareStr(S1, S2));
  
  // CompareText: case insensitive
  WriteLn('CompareText: ', CompareText(S1, S2));
  
  if CompareText(S1, S2) = 0 then
    WriteLn('เท่ากัน (case insensitive)')
  else
    WriteLn('ไม่เท่ากัน (case insensitive)');
  
  // SameText: เปรียบเทียบแบบ case insensitive
  if SameText(S1, S2) then
    WriteLn('SameText: เท่ากัน')
  else
    WriteLn('SameText: ไม่เท่ากัน');
  
  // เรียงตามตัวอักษร
  if S1 < S2 then WriteLn(S1, ' มาก่อน ', S2)
  else if S1 > S2 then WriteLn(S2, ' มาก่อน ', S1)
  else WriteLn('เหมือนกัน');
end.
```

### ตัวอย่างที่ 14: เรียงชื่อตามตัวอักษร

```pascal
program SortNames;
var
  Names: array[1..6] of String;
  I, J: Integer;
  Temp: String;
begin
  Names[1] := 'Charlie';
  Names[2] := 'Alice';
  Names[3] := 'Eve';
  Names[4] := 'Bob';
  Names[5] := 'David';
  Names[6] := 'Frank';
  
  // Bubble Sort
  for I := 1 to 5 do
    for J := 1 to 6 - I do
      if CompareText(Names[J], Names[J+1]) > 0 then
      begin
        Temp := Names[J];
        Names[J] := Names[J+1];
        Names[J+1] := Temp;
      end;
  
  WriteLn('เรียงตามตัวอักษร:');
  for I := 1 to 6 do
    WriteLn(I, '. ', Names[I]);
end.
```

---

## String Conversion

### ตัวอย่างที่ 15: ตัวเลข <-> String

```pascal
program NumberStringConvert;
uses
  SysUtils;
var
  N: Integer;
  F: Real;
  S: String;
begin
  // Integer -> String
  N := 42;
  S := IntToStr(N);
  WriteLn('IntToStr(42) = "', S, '"');
  
  // String -> Integer
  S := '123';
  N := StrToInt(S);
  WriteLn('StrToInt("123") = ', N);
  
  // StrToIntDef: ถ้าแปลงไม่ได้ ใช้ค่า default
  N := StrToIntDef('abc', -1);
  WriteLn('StrToIntDef("abc", -1) = ', N);
  
  // TryStrToInt: ส่งคืน False ถ้าแปลงไม่ได้
  if TryStrToInt('456', N) then
    WriteLn('TryStrToInt OK: ', N)
  else
    WriteLn('TryStrToInt failed');
  
  // Float -> String
  F := 3.14159;
  S := FloatToStr(F);
  WriteLn('FloatToStr(3.14159) = "', S, '"');
  
  // Float กำหนดทศนิยม
  WriteLn('FormatFloat: ', FormatFloat('0.00', F));
  WriteLn('FormatFloat: ', FormatFloat('#,##0.00', 1234567.89));
  
  // String -> Float
  F := StrToFloat('3.14');
  WriteLn('StrToFloat("3.14") = ', F:0:4);
end.
```

### ตัวอย่างที่ 16: Char <-> Integer

```pascal
program CharIntConvert;
var
  C: Char;
  N: Integer;
begin
  C := 'A';
  N := Ord(C);  // Char -> Integer (ASCII)
  WriteLn('Ord(', Chr(39), 'A', Chr(39), ') = ', N);
  
  N := 97;
  C := Chr(N);  // Integer -> Char
  WriteLn('Chr(97) = ', C);
  
  // แปลงอักษรตัวพิมพ์ใหญ่ <-> เล็ก
  C := 'A';
  WriteLn(C, ' -> ', Chr(Ord(C) + 32));  // A -> a
  
  C := 'z';
  WriteLn(C, ' -> ', Chr(Ord(C) - 32));  // z -> Z
  
  // ตรวจสอบประเภทอักขระ
  for C := '!' to '~' do
  begin
    if C in ['0'..'9'] then Write('D')
    else if C in ['A'..'Z', 'a'..'z'] then Write('L')
    else Write('S');
  end;
  WriteLn;
end.
```

---

## String Formatting

### ตัวอย่างที่ 17: Format function

```pascal
program FormatExample;
uses
  SysUtils;
var
  Name: String;
  Score: Real;
  Age: Integer;
begin
  Name := 'Alice';
  Score := 95.5;
  Age := 20;
  
  // %s = String
  // %d = Integer
  // %f = Float
  // %e = Scientific notation
  // %g = ใช้แบบสั้น
  
  WriteLn(Format('ชื่อ: %s', [Name]));
  WriteLn(Format('อายุ: %d ปี', [Age]));
  WriteLn(Format('คะแนน: %.2f', [Score]));
  WriteLn(Format('ชื่อ: %s, อายุ: %d, คะแนน: %.1f', [Name, Age, Score]));
  
  // จัดความกว้าง
  WriteLn(Format('%-20s %5d %8.2f', [Name, Age, Score]));
  WriteLn(Format('%20s %5d %8.2f', [Name, Age, Score]));
  
  // เลขฐาน
  WriteLn(Format('Decimal: %d, Hex: %x, Octal: %o', [255, 255, 255]));
  
  // ตัวอย่างใช้งานจริง: ใบเสร็จ
  WriteLn;
  WriteLn(StringOfChar('=', 40));
  WriteLn(Format('%-25s %8s', ['รายการ', 'จำนวน']));
  WriteLn(StringOfChar('-', 40));
  WriteLn(Format('%-25s %8.2f', ['ข้าวผัด', 60.00]));
  WriteLn(Format('%-25s %8.2f', ['ต้มยำกุ้ง', 120.00]));
  WriteLn(Format('%-25s %8.2f', ['น้ำเปล่า', 15.00]));
  WriteLn(StringOfChar('-', 40));
  WriteLn(Format('%-25s %8.2f', ['รวม', 195.00]));
  WriteLn(StringOfChar('=', 40));
end.
```

### ตัวอย่างที่ 18: FormatDateTime

```pascal
program DateTimeFormat;
uses
  SysUtils, DateUtils;
var
  Now_: TDateTime;
begin
  Now_ := Now;
  
  WriteLn('วันปัจจุบัน:');
  WriteLn(FormatDateTime('dd/mm/yyyy', Now_));
  WriteLn(FormatDateTime('d mmmm yyyy', Now_));
  WriteLn(FormatDateTime('dddd, d mmmm yyyy', Now_));
  WriteLn(FormatDateTime('hh:nn:ss', Now_));
  WriteLn(FormatDateTime('dd/mm/yyyy hh:nn:ss', Now_));
  
  // คำนวณวันที่
  WriteLn;
  WriteLn('7 วันจากนี้: ', FormatDateTime('dd/mm/yyyy', Now_ + 7));
  WriteLn('เดือนที่แล้ว: ', FormatDateTime('dd/mm/yyyy', IncMonth(Now_, -1)));
end.
```

---

## String Searching

### ตัวอย่างที่ 19: ค้นหาขั้นสูง

```pascal
program AdvancedSearch;
uses
  SysUtils;
  
function CountOccurrences(const Text, Pattern: String): Integer;
var
  Start, Count: Integer;
begin
  Count := 0;
  Start := 1;
  while True do
  begin
    Start := PosEx(Pattern, Text, Start);
    if Start = 0 then Break;
    Inc(Count);
    Inc(Start, Length(Pattern));
  end;
  Result := Count;
end;

function FindAllPositions(const Text, Pattern: String): String;
var
  Start, Pos_: Integer;
  Positions: String;
begin
  Positions := '';
  Start := 1;
  while True do
  begin
    Pos_ := PosEx(Pattern, Text, Start);
    if Pos_ = 0 then Break;
    if Positions <> '' then Positions := Positions + ', ';
    Positions := Positions + IntToStr(Pos_);
    Inc(Start, Length(Pattern));
  end;
  Result := Positions;
end;

var
  Text: String;
begin
  Text := 'the cat sat on the mat. the cat is in the hat.';
  
  WriteLn('ข้อความ: "', Text, '"');
  WriteLn;
  WriteLn('นับ "the": ', CountOccurrences(Text, 'the'));
  WriteLn('ตำแหน่ง "the": ', FindAllPositions(Text, 'the'));
  WriteLn('นับ "cat": ', CountOccurrences(Text, 'cat'));
  WriteLn('ตำแหน่ง "cat": ', FindAllPositions(Text, 'cat'));
end.
```

### ตัวอย่างที่ 20: Split String

```pascal
program SplitString;
uses
  SysUtils, Classes;
  
function SplitStr(const S, Delimiter: String): TStringArray;
var
  Parts: TStringList;
  I: Integer;
begin
  Parts := TStringList.Create;
  try
    Parts.Delimiter := Delimiter[1];
    Parts.DelimitedText := S;
    SetLength(Result, Parts.Count);
    for I := 0 to Parts.Count - 1 do
      Result[I] := Parts[I];
  finally
    Parts.Free;
  end;
end;

// Simple manual split
procedure ManualSplit(const S, Sep: String; var Parts: array of String; var Count: Integer);
var
  Start, Pos_: Integer;
begin
  Count := 0;
  Start := 1;
  
  while True do
  begin
    Pos_ := PosEx(Sep, S, Start);
    if Pos_ = 0 then
    begin
      if Count < Length(Parts) then
      begin
        Parts[Count] := Copy(S, Start, Length(S) - Start + 1);
        Inc(Count);
      end;
      Break;
    end;
    
    if Count < Length(Parts) then
    begin
      Parts[Count] := Copy(S, Start, Pos_ - Start);
      Inc(Count);
    end;
    Start := Pos_ + Length(Sep);
  end;
end;

var
  CSV: String;
  Parts: array[0..9] of String;
  Count, I: Integer;
begin
  CSV := 'Alice,25,Engineer,Bangkok';
  
  WriteLn('CSV: "', CSV, '"');
  ManualSplit(CSV, ',', Parts, Count);
  
  WriteLn('ส่วนประกอบ ', Count, ' ส่วน:');
  for I := 0 to Count - 1 do
    WriteLn(I, ': "', Parts[I], '"');
end.
```

---

## Regular Expressions

### ตัวอย่างที่ 21: ใช้ RegEx ใน Lazarus

```pascal
program RegExExample;
uses
  RegExpr;  // ต้องใช้ library RegExpr
  
var
  RE: TRegExpr;
  Text: String;
begin
  RE := TRegExpr.Create;
  try
    Text := 'สมชาย@example.com';
    
    // ตรวจสอบ email
    RE.Expression := '^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$';
    if RE.Exec(Text) then
      WriteLn('"', Text, '" เป็น email ที่ถูกต้อง')
    else
      WriteLn('"', Text, '" ไม่ใช่ email');
    
    // ค้นหาตัวเลขในข้อความ
    Text := 'มีสินค้า 5 ชิ้น ราคา 350 บาท จ่ายแล้ว 200 บาท';
    RE.Expression := '\d+';
    
    if RE.Exec(Text) then
    begin
      Write('ตัวเลขที่พบ: ');
      repeat
        Write(RE.Match[0], ' ');
      until not RE.ExecNext;
      WriteLn;
    end;
    
  finally
    RE.Free;
  end;
end.
```

### ตัวอย่างที่ 22: Pattern Matching แบบ Manual

```pascal
program ManualPattern;
uses SysUtils;

function IsEmail(const S: String): Boolean;
var
  AtPos, DotPos: Integer;
begin
  AtPos := Pos('@', S);
  if AtPos <= 1 then begin Result := False; Exit; end;
  
  DotPos := Pos('.', Copy(S, AtPos + 1, Length(S)));
  if DotPos <= 0 then begin Result := False; Exit; end;
  
  // ตรวจสอบไม่มีช่องว่าง
  Result := Pos(' ', S) = 0;
end;

function IsPhoneNumber(const S: String): Boolean;
var
  Clean: String;
  I: Integer;
begin
  Clean := S;
  // ลบ - และ ()
  Clean := StringReplace(Clean, '-', '', [rfReplaceAll]);
  Clean := StringReplace(Clean, '(', '', [rfReplaceAll]);
  Clean := StringReplace(Clean, ')', '', [rfReplaceAll]);
  Clean := StringReplace(Clean, ' ', '', [rfReplaceAll]);
  
  Result := Length(Clean) in [9, 10];
  if Result then
    for I := 1 to Length(Clean) do
      if not (Clean[I] in ['0'..'9']) then
      begin
        Result := False;
        Break;
      end;
end;

function IsValidPassword(const S: String): Boolean;
var
  HasUpper, HasLower, HasDigit: Boolean;
  I: Integer;
begin
  HasUpper := False;
  HasLower := False;
  HasDigit := False;
  
  if Length(S) < 8 then begin Result := False; Exit; end;
  
  for I := 1 to Length(S) do
  begin
    if S[I] in ['A'..'Z'] then HasUpper := True;
    if S[I] in ['a'..'z'] then HasLower := True;
    if S[I] in ['0'..'9'] then HasDigit := True;
  end;
  
  Result := HasUpper and HasLower and HasDigit;
end;

var
  TestStr: String;
begin
  // ทดสอบ Email
  TestStr := 'user@example.com';
  WriteLn(TestStr, ': ', BoolToStr(IsEmail(TestStr), True));
  
  TestStr := 'invalid.email';
  WriteLn(TestStr, ': ', BoolToStr(IsEmail(TestStr), True));
  
  // ทดสอบเบอร์โทร
  TestStr := '081-234-5678';
  WriteLn(TestStr, ': ', BoolToStr(IsPhoneNumber(TestStr), True));
  
  // ทดสอบรหัสผ่าน
  TestStr := 'Pass123!';
  WriteLn('Password "', TestStr, '": ', BoolToStr(IsValidPassword(TestStr), True));
  
  TestStr := 'weakpass';
  WriteLn('Password "', TestStr, '": ', BoolToStr(IsValidPassword(TestStr), True));
end.
```

---

## String Builder Pattern

### ตัวอย่างที่ 23: String Builder

```pascal
program StringBuilderPattern;
uses
  SysUtils, Classes;

type
  TStringBuilder = class
  private
    FBuffer: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    procedure Append(const S: String);
    procedure AppendLine(const S: String = '');
    procedure AppendFormat(const Fmt: String; const Args: array of const);
    function ToString: String; override;
    procedure Clear;
    function Length: Integer;
  end;

constructor TStringBuilder.Create;
begin
  FBuffer := TStringList.Create;
end;

destructor TStringBuilder.Destroy;
begin
  FBuffer.Free;
  inherited;
end;

procedure TStringBuilder.Append(const S: String);
begin
  if FBuffer.Count = 0 then
    FBuffer.Add(S)
  else
    FBuffer[FBuffer.Count - 1] := FBuffer[FBuffer.Count - 1] + S;
end;

procedure TStringBuilder.AppendLine(const S: String);
begin
  if S = '' then
    FBuffer.Add('')
  else
    FBuffer.Add(S);
end;

procedure TStringBuilder.AppendFormat(const Fmt: String; const Args: array of const);
begin
  Append(Format(Fmt, Args));
end;

function TStringBuilder.ToString: String;
begin
  Result := FBuffer.Text;
end;

procedure TStringBuilder.Clear;
begin
  FBuffer.Clear;
end;

function TStringBuilder.Length: Integer;
begin
  Result := System.Length(ToString);
end;

var
  SB: TStringBuilder;
  I: Integer;
begin
  SB := TStringBuilder.Create;
  try
    SB.AppendLine('=== รายการสินค้า ===');
    SB.AppendLine;
    
    for I := 1 to 5 do
      SB.AppendFormat('สินค้า %d: %.2f บาท%s', [I, I * 29.9, LineEnding]);
    
    SB.AppendLine;
    SB.AppendLine('ขอบคุณที่ซื้อสินค้า');
    
    WriteLn(SB.ToString);
    WriteLn('ความยาวทั้งหมด: ', SB.Length, ' ตัวอักษร');
  finally
    SB.Free;
  end;
end.
```

---

## Unicode Strings

### ตัวอย่างที่ 24: Unicode String

```pascal
program UnicodeExample;
uses
  SysUtils;
  
var
  S: UnicodeString;
  WS: WideString;
  I: Integer;
begin
  S := 'สวัสดีโลก Hello World 🌍';
  
  WriteLn('UnicodeString: ', S);
  WriteLn('ความยาว: ', Length(S));
  
  // เข้าถึงแต่ละ char
  for I := 1 to Length(S) do
    Write(S[I]);
  WriteLn;
  
  // แปลง
  WS := S;
  WriteLn('WideString: ', WS);
  
  // UTF-8 encoding
  var UTF8: String;
  UTF8 := UTF8Encode(S);
  WriteLn('UTF-8 bytes: ', Length(UTF8));
  
  var BackToUnicode: UnicodeString;
  BackToUnicode := UTF8Decode(UTF8);
  WriteLn('Back to Unicode: ', BackToUnicode);
end.
```

### ตัวอย่างที่ 25: String Encoding

```pascal
program EncodingExample;
uses
  SysUtils, LConvEncoding;
  
var
  S: String;
  UTF8S: String;
begin
  S := 'Hello สวัสดี';
  
  // แปลงเป็น UTF-8
  UTF8S := ConvertEncoding(S, 'windows-1255', 'UTF-8');
  
  WriteLn('ต้นฉบับ: ', S);
  WriteLn('UTF-8: ', UTF8S);
  WriteLn('ความยาว: ', Length(S), ' bytes');
end.
```

---

## TStringList

### ตัวอย่างที่ 26: TStringList พื้นฐาน

```pascal
program TStringListBasic;
uses
  Classes;
  
var
  List: TStringList;
  I: Integer;
begin
  List := TStringList.Create;
  try
    // เพิ่มข้อมูล
    List.Add('กล้วย');
    List.Add('แอปเปิ้ล');
    List.Add('ส้ม');
    List.Add('มะม่วง');
    List.Add('สตรอเบอร์รี่');
    
    WriteLn('รายการผลไม้:');
    for I := 0 to List.Count - 1 do
      WriteLn(I + 1, '. ', List[I]);
    
    WriteLn;
    WriteLn('จำนวน: ', List.Count);
    
    // ค้นหา
    var Idx := List.IndexOf('ส้ม');
    if Idx >= 0 then
      WriteLn('พบ "ส้ม" ที่ตำแหน่ง ', Idx)
    else
      WriteLn('ไม่พบ "ส้ม"');
    
    // เรียงลำดับ
    List.Sort;
    WriteLn;
    WriteLn('หลังเรียง:');
    for I := 0 to List.Count - 1 do
      WriteLn(I + 1, '. ', List[I]);
    
    // ลบ
    List.Delete(0);
    WriteLn;
    WriteLn('หลังลบตัวแรก:');
    for I := 0 to List.Count - 1 do
      WriteLn(I + 1, '. ', List[I]);
      
  finally
    List.Free;
  end;
end.
```

### ตัวอย่างที่ 27: TStringList กับ Key=Value

```pascal
program TStringListKeyValue;
uses
  Classes;
  
var
  Config: TStringList;
begin
  Config := TStringList.Create;
  try
    Config.NameValueSeparator := '=';
    
    Config.Add('host=localhost');
    Config.Add('port=3306');
    Config.Add('database=mydb');
    Config.Add('user=root');
    Config.Add('password=secret');
    
    WriteLn('การตั้งค่า:');
    var I: Integer;
    for I := 0 to Config.Count - 1 do
      WriteLn('  ', Config.Names[I], ' = ', Config.ValueFromIndex[I]);
    
    // อ่านค่าตาม key
    WriteLn;
    WriteLn('Host: ', Config.Values['host']);
    WriteLn('Port: ', Config.Values['port']);
    
    // แก้ไขค่า
    Config.Values['port'] := '5432';
    WriteLn('Port ใหม่: ', Config.Values['port']);
    
    // บันทึกและโหลดจากไฟล์
    Config.SaveToFile('config.ini');
    
    var NewConfig := TStringList.Create;
    try
      NewConfig.LoadFromFile('config.ini');
      WriteLn;
      WriteLn('โหลดจากไฟล์:');
      for I := 0 to NewConfig.Count - 1 do
        WriteLn('  ', NewConfig[I]);
    finally
      NewConfig.Free;
    end;
    
  finally
    Config.Free;
  end;
end.
```

### ตัวอย่างที่ 28: TStringList อ่านไฟล์ข้อความ

```pascal
program ReadTextFile;
uses
  Classes, SysUtils;
  
var
  Lines: TStringList;
  I: Integer;
  WordCount, CharCount: Integer;
begin
  Lines := TStringList.Create;
  try
    // สร้างไฟล์ทดสอบ
    Lines.Add('บรรทัดที่หนึ่ง');
    Lines.Add('บรรทัดที่สอง');
    Lines.Add('บรรทัดที่สาม');
    Lines.SaveToFile('test.txt');
    Lines.Clear;
    
    // อ่านไฟล์
    Lines.LoadFromFile('test.txt');
    
    WriteLn('เนื้อหาไฟล์:');
    for I := 0 to Lines.Count - 1 do
      WriteLn(I + 1, ': ', Lines[I]);
    
    // นับคำ
    WordCount := 0;
    CharCount := 0;
    for I := 0 to Lines.Count - 1 do
    begin
      CharCount := CharCount + Length(Lines[I]);
      var Words := Lines[I].Split([' ']);
      WordCount := WordCount + Length(Words);
    end;
    
    WriteLn;
    WriteLn('จำนวนบรรทัด: ', Lines.Count);
    WriteLn('จำนวนตัวอักษร: ', CharCount);
    WriteLn('จำนวนคำ: ', WordCount);
    
  finally
    Lines.Free;
  end;
end.
```

---

## โปรแกรมตัวอย่าง: Text Processing

```pascal
program TextProcessor;
uses
  SysUtils, StrUtils;

function WordCount(const Text: String): Integer;
var
  I: Integer;
  InWord: Boolean;
begin
  Result := 0;
  InWord := False;
  for I := 1 to Length(Text) do
  begin
    if Text[I] = ' ' then
      InWord := False
    else
    begin
      if not InWord then
      begin
        Inc(Result);
        InWord := True;
      end;
    end;
  end;
end;

function ReverseString(const S: String): String;
var
  I: Integer;
begin
  SetLength(Result, Length(S));
  for I := 1 to Length(S) do
    Result[Length(S) - I + 1] := S[I];
end;

function IsPalindrome(const S: String): Boolean;
var
  Clean: String;
  I: Integer;
begin
  // ลบ non-alphabetic และแปลงเป็น lowercase
  Clean := '';
  for I := 1 to Length(S) do
    if S[I] in ['a'..'z', 'A'..'Z', '0'..'9'] then
      Clean := Clean + LowerCase(S[I]);
  
  Result := Clean = ReverseString(Clean);
end;

function CountChar(const S: String; C: Char): Integer;
var
  I: Integer;
begin
  Result := 0;
  for I := 1 to Length(S) do
    if S[I] = C then Inc(Result);
end;

function TitleCase(const S: String): String;
var
  I: Integer;
  AfterSpace: Boolean;
begin
  Result := LowerCase(S);
  AfterSpace := True;
  for I := 1 to Length(Result) do
  begin
    if Result[I] = ' ' then
      AfterSpace := True
    else if AfterSpace then
    begin
      Result[I] := UpCase(Result[I]);
      AfterSpace := False;
    end;
  end;
end;

function RemoveExtraSpaces(const S: String): String;
var
  I: Integer;
  PrevSpace: Boolean;
begin
  Result := Trim(S);
  PrevSpace := False;
  var New := '';
  for I := 1 to Length(Result) do
  begin
    if Result[I] = ' ' then
    begin
      if not PrevSpace then New := New + ' ';
      PrevSpace := True;
    end
    else
    begin
      New := New + Result[I];
      PrevSpace := False;
    end;
  end;
  Result := New;
end;

var
  Text: String;
  Choice: Integer;
  Running: Boolean;
begin
  Running := True;
  
  while Running do
  begin
    WriteLn;
    WriteLn('=== Text Processor ===');
    WriteLn('1. นับคำ');
    WriteLn('2. กลับสตริง');
    WriteLn('3. ตรวจสอบ Palindrome');
    WriteLn('4. นับอักขระ');
    WriteLn('5. Title Case');
    WriteLn('6. ลบช่องว่างซ้ำ');
    WriteLn('7. ค้นหาและแทนที่');
    WriteLn('8. แยกคำ');
    WriteLn('0. ออก');
    Write('เลือก: ');
    ReadLn(Choice);
    
    case Choice of
      1: begin
           Write('ป้อนข้อความ: ');
           ReadLn(Text);
           WriteLn('จำนวนคำ: ', WordCount(Text));
         end;
      2: begin
           Write('ป้อนข้อความ: ');
           ReadLn(Text);
           WriteLn('กลับสตริง: ', ReverseString(Text));
         end;
      3: begin
           Write('ป้อนข้อความ: ');
           ReadLn(Text);
           if IsPalindrome(Text) then
             WriteLn('"', Text, '" เป็น palindrome')
           else
             WriteLn('"', Text, '" ไม่เป็น palindrome');
         end;
      4: begin
           Write('ป้อนข้อความ: ');
           ReadLn(Text);
           var C: Char;
           Write('ป้อนอักขระที่ต้องการนับ: ');
           ReadLn(C);
           WriteLn('พบ "', C, '" จำนวน ', CountChar(Text, C), ' ครั้ง');
         end;
      5: begin
           Write('ป้อนข้อความ: ');
           ReadLn(Text);
           WriteLn('Title Case: ', TitleCase(Text));
         end;
      6: begin
           Write('ป้อนข้อความ: ');
           ReadLn(Text);
           WriteLn('หลังลบช่องว่างซ้ำ: "', RemoveExtraSpaces(Text), '"');
         end;
      7: begin
           Write('ป้อนข้อความ: ');
           ReadLn(Text);
           var OldStr, NewStr: String;
           Write('ค้นหา: ');
           ReadLn(OldStr);
           Write('แทนด้วย: ');
           ReadLn(NewStr);
           WriteLn('ผล: ', StringReplace(Text, OldStr, NewStr, [rfReplaceAll]));
         end;
      8: begin
           Write('ป้อนข้อความ: ');
           ReadLn(Text);
           var Words := SplitString(Text, ' ');
           WriteLn('คำทั้งหมด ', Length(Words), ' คำ:');
           var I: Integer;
           for I := 0 to High(Words) do
             WriteLn(I + 1, ': "', Words[I], '"');
         end;
      0: Running := False;
    else
      WriteLn('ตัวเลือกไม่ถูกต้อง');
    end;
  end;
end.
```

---

## โปรแกรมตัวอย่าง: Password Validator

```pascal
program PasswordValidator;
uses
  SysUtils;

type
  TPasswordStrength = (psWeak, psFair, psGood, psStrong);

function GetPasswordStrength(const Password: String): TPasswordStrength;
var
  Score: Integer;
  HasUpper, HasLower, HasDigit, HasSpecial: Boolean;
  I: Integer;
  SpecialChars: String;
begin
  Score := 0;
  HasUpper := False;
  HasLower := False;
  HasDigit := False;
  HasSpecial := False;
  SpecialChars := '!@#$%^&*()_+-=[]{}|;:,.<>?';
  
  // ความยาว
  if Length(Password) >= 8 then Inc(Score);
  if Length(Password) >= 12 then Inc(Score);
  if Length(Password) >= 16 then Inc(Score);
  
  // ประเภทอักขระ
  for I := 1 to Length(Password) do
  begin
    if Password[I] in ['A'..'Z'] then HasUpper := True;
    if Password[I] in ['a'..'z'] then HasLower := True;
    if Password[I] in ['0'..'9'] then HasDigit := True;
    if Pos(Password[I], SpecialChars) > 0 then HasSpecial := True;
  end;
  
  if HasUpper then Inc(Score);
  if HasLower then Inc(Score);
  if HasDigit then Inc(Score);
  if HasSpecial then Inc(Score);
  
  if Score <= 2 then Result := psWeak
  else if Score <= 4 then Result := psFair
  else if Score <= 6 then Result := psGood
  else Result := psStrong;
end;

function ValidatePassword(const Password: String; var ErrorMsg: String): Boolean;
begin
  Result := True;
  ErrorMsg := '';
  
  if Length(Password) < 8 then
  begin
    ErrorMsg := 'รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร';
    Result := False;
    Exit;
  end;
  
  var HasUpper := False;
  var HasLower := False;
  var HasDigit := False;
  
  var I: Integer;
  for I := 1 to Length(Password) do
  begin
    if Password[I] in ['A'..'Z'] then HasUpper := True;
    if Password[I] in ['a'..'z'] then HasLower := True;
    if Password[I] in ['0'..'9'] then HasDigit := True;
  end;
  
  if not HasUpper then
  begin
    ErrorMsg := 'ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว';
    Result := False;
    Exit;
  end;
  
  if not HasLower then
  begin
    ErrorMsg := 'ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว';
    Result := False;
    Exit;
  end;
  
  if not HasDigit then
  begin
    ErrorMsg := 'ต้องมีตัวเลขอย่างน้อย 1 ตัว';
    Result := False;
    Exit;
  end;
end;

var
  Password: String;
  Strength: TPasswordStrength;
  ErrorMsg: String;
  StrengthNames: array[TPasswordStrength] of String;
  StrengthBars: array[TPasswordStrength] of String;
  
begin
  StrengthNames[psWeak] := 'อ่อนแอ';
  StrengthNames[psFair] := 'พอใช้';
  StrengthNames[psGood] := 'ดี';
  StrengthNames[psStrong] := 'แข็งแกร่งมาก';
  
  StrengthBars[psWeak] := '█░░░';
  StrengthBars[psFair] := '██░░';
  StrengthBars[psGood] := '███░';
  StrengthBars[psStrong] := '████';
  
  WriteLn('=== ตรวจสอบความปลอดภัยรหัสผ่าน ===');
  WriteLn;
  WriteLn('กฎ:');
  WriteLn('  - ความยาวอย่างน้อย 8 ตัวอักษร');
  WriteLn('  - มีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว (A-Z)');
  WriteLn('  - มีตัวพิมพ์เล็กอย่างน้อย 1 ตัว (a-z)');
  WriteLn('  - มีตัวเลขอย่างน้อย 1 ตัว (0-9)');
  WriteLn;
  
  Write('ป้อนรหัสผ่าน: ');
  ReadLn(Password);
  
  WriteLn;
  WriteLn('=== ผลการตรวจสอบ ===');
  WriteLn('รหัสผ่าน: ', Password);
  WriteLn('ความยาว: ', Length(Password), ' ตัวอักษร');
  
  Strength := GetPasswordStrength(Password);
  WriteLn('ความแข็งแกร่ง: ', StrengthBars[Strength], ' ', StrengthNames[Strength]);
  
  if ValidatePassword(Password, ErrorMsg) then
    WriteLn('สถานะ: ✓ ผ่าน')
  else
    WriteLn('สถานะ: ✗ ไม่ผ่าน - ', ErrorMsg);
end.
```

---

## โปรแกรมตัวอย่าง: Email Parser

```pascal
program EmailParser;
uses
  SysUtils;

type
  TEmailInfo = record
    Original: String;
    Username: String;
    Domain: String;
    TLD: String;
    IsValid: Boolean;
    ErrorMsg: String;
  end;

function ParseEmail(const Email: String): TEmailInfo;
var
  AtPos, LastDotPos: Integer;
begin
  Result.Original := Email;
  Result.IsValid := True;
  Result.ErrorMsg := '';
  
  // หา @
  AtPos := Pos('@', Email);
  if AtPos = 0 then
  begin
    Result.IsValid := False;
    Result.ErrorMsg := 'ไม่มีเครื่องหมาย @';
    Exit;
  end;
  
  if AtPos = 1 then
  begin
    Result.IsValid := False;
    Result.ErrorMsg := 'ไม่มี username';
    Exit;
  end;
  
  Result.Username := Copy(Email, 1, AtPos - 1);
  var Rest := Copy(Email, AtPos + 1, Length(Email));
  
  if Length(Rest) = 0 then
  begin
    Result.IsValid := False;
    Result.ErrorMsg := 'ไม่มี domain';
    Exit;
  end;
  
  // หา . สุดท้าย
  LastDotPos := Length(Rest);
  while (LastDotPos > 0) and (Rest[LastDotPos] <> '.') do
    Dec(LastDotPos);
  
  if LastDotPos = 0 then
  begin
    Result.IsValid := False;
    Result.ErrorMsg := 'domain ไม่มี TLD';
    Exit;
  end;
  
  Result.Domain := Copy(Rest, 1, LastDotPos - 1);
  Result.TLD := Copy(Rest, LastDotPos + 1, Length(Rest));
  
  if Length(Result.TLD) < 2 then
  begin
    Result.IsValid := False;
    Result.ErrorMsg := 'TLD สั้นเกินไป';
  end;
  
  // ตรวจสอบ spaces
  if Pos(' ', Email) > 0 then
  begin
    Result.IsValid := False;
    Result.ErrorMsg := 'ไม่ควรมีช่องว่าง';
  end;
end;

var
  Emails: array[1..6] of String;
  Info: TEmailInfo;
  I: Integer;
begin
  Emails[1] := 'user@example.com';
  Emails[2] := 'invalid.email';
  Emails[3] := '@nodomain.com';
  Emails[4] := 'user@domain.x';
  Emails[5] := 'test.user@sub.domain.co.th';
  Emails[6] := 'has space@test.com';
  
  WriteLn('=== Email Parser ===');
  WriteLn;
  
  for I := 1 to 6 do
  begin
    Info := ParseEmail(Emails[I]);
    WriteLn('Email: "', Info.Original, '"');
    if Info.IsValid then
    begin
      WriteLn('  สถานะ:   ✓ ถูกต้อง');
      WriteLn('  Username: ', Info.Username);
      WriteLn('  Domain:   ', Info.Domain);
      WriteLn('  TLD:      ', Info.TLD);
    end
    else
      WriteLn('  สถานะ:   ✗ ไม่ถูกต้อง (', Info.ErrorMsg, ')');
    WriteLn;
  end;
  
  // ทดสอบแบบ interactive
  WriteLn('ทดสอบ Email ของคุณ:');
  var Email: String;
  Write('ป้อน Email: ');
  ReadLn(Email);
  
  Info := ParseEmail(Email);
  if Info.IsValid then
    WriteLn('✓ Email นี้ถูกต้อง (', Info.Username, ' @ ', Info.Domain, '.', Info.TLD, ')')
  else
    WriteLn('✗ Email ไม่ถูกต้อง: ', Info.ErrorMsg);
end.
```

---

## แบบฝึกหัด

### ข้อที่ 1-5

```pascal
// ข้อ 1: นับสระในข้อความภาษาอังกฤษ
program CountVowels;
var S: String; I, Count: Integer;
begin
  Write('ป้อนข้อความ: '); ReadLn(S);
  S := LowerCase(S);
  Count := 0;
  for I := 1 to Length(S) do
    if S[I] in ['a', 'e', 'i', 'o', 'u'] then Inc(Count);
  WriteLn('จำนวนสระ: ', Count);
end.

// ข้อ 2: กลับคำในประโยค (reverse words)
program ReverseWords;
uses SysUtils;
var S: String;
begin
  Write('ประโยค: '); ReadLn(S);
  var Words := SplitString(S, ' ');
  var I: Integer;
  Write('กลับคำ: ');
  for I := High(Words) downto 0 do
  begin
    Write(Words[I]);
    if I > 0 then Write(' ');
  end;
  WriteLn;
end.

// ข้อ 3: ตรวจสอบ Anagram
program AnagramCheck;
uses SysUtils;
function SortString(S: String): String;
var I, J: Integer; Temp: Char;
begin
  S := LowerCase(S);
  for I := 1 to Length(S) - 1 do
    for J := 1 to Length(S) - I do
      if S[J] > S[J+1] then begin Temp := S[J]; S[J] := S[J+1]; S[J+1] := Temp; end;
  Result := S;
end;
var S1, S2: String;
begin
  Write('คำที่ 1: '); ReadLn(S1);
  Write('คำที่ 2: '); ReadLn(S2);
  if SortString(S1) = SortString(S2) then
    WriteLn(S1, ' และ ', S2, ' เป็น Anagram กัน')
  else
    WriteLn('ไม่เป็น Anagram');
end.

// ข้อ 4: แปลงเลขโรมัน
program RomanNumerals;
function ToRoman(N: Integer): String;
const
  Values: array[1..13] of Integer = (1000,900,500,400,100,90,50,40,10,9,5,4,1);
  Symbols: array[1..13] of String = ('M','CM','D','CD','C','XC','L','XL','X','IX','V','IV','I');
var I: Integer;
begin
  Result := '';
  for I := 1 to 13 do
    while N >= Values[I] do begin Result := Result + Symbols[I]; Dec(N, Values[I]); end;
end;
var N: Integer;
begin
  Write('ป้อนเลข (1-3999): '); ReadLn(N);
  WriteLn(N, ' = ', ToRoman(N));
end.

// ข้อ 5: Caesar Cipher
program CaesarCipher;
var S: String; Shift, I: Integer; C: Char;
begin
  Write('ข้อความ: '); ReadLn(S);
  Write('เลื่อน: '); ReadLn(Shift);
  for I := 1 to Length(S) do begin
    C := S[I];
    if C in ['A'..'Z'] then S[I] := Chr((Ord(C) - Ord('A') + Shift) mod 26 + Ord('A'));
    if C in ['a'..'z'] then S[I] := Chr((Ord(C) - Ord('a') + Shift) mod 26 + Ord('a'));
  end;
  WriteLn('เข้ารหัส: ', S);
end.
```

### ข้อที่ 6-10

```pascal
// ข้อ 6: นับความถี่ของตัวอักษร
program LetterFrequency;
var S: String; Freq: array['a'..'z'] of Integer; C: Char; I: Integer;
begin
  Write('ป้อนข้อความ: '); ReadLn(S);
  S := LowerCase(S);
  for C := 'a' to 'z' do Freq[C] := 0;
  for I := 1 to Length(S) do
    if S[I] in ['a'..'z'] then Inc(Freq[S[I]]);
  for C := 'a' to 'z' do
    if Freq[C] > 0 then
      WriteLn(C, ': ', Freq[C]);
end.

// ข้อ 7: แปลง Binary เป็น String
program BinaryToText;
var Binary, Text: String; I, Byte_: Integer;
begin
  Write('Binary: '); ReadLn(Binary);
  Text := '';
  I := 1;
  while I + 7 <= Length(Binary) do begin
    Byte_ := 0;
    var J: Integer;
    for J := 0 to 7 do
      if Binary[I + J] = '1' then Byte_ := Byte_ + (1 shl (7 - J));
    Text := Text + Chr(Byte_);
    Inc(I, 8);
  end;
  WriteLn('Text: ', Text);
end.

// ข้อ 8: Longest Common Substring
program LongestCommon;
function LCS(const A, B: String): String;
var I, J, Len, MaxLen, MaxI: Integer;
begin
  MaxLen := 0; MaxI := 0;
  for I := 1 to Length(A) do
    for J := 1 to Length(B) do begin
      Len := 0;
      while (I + Len <= Length(A)) and (J + Len <= Length(B)) and 
            (A[I + Len] = B[J + Len]) do Inc(Len);
      if Len > MaxLen then begin MaxLen := Len; MaxI := I; end;
    end;
  Result := Copy(A, MaxI, MaxLen);
end;
var S1, S2: String;
begin
  Write('String 1: '); ReadLn(S1);
  Write('String 2: '); ReadLn(S2);
  WriteLn('LCS: "', LCS(S1, S2), '"');
end.

// ข้อ 9: ตรวจสอบ Valid Parentheses
program ValidParentheses;
var S: String; I, Count: Integer; IsValid: Boolean;
begin
  Write('ป้อนข้อความ: '); ReadLn(S);
  Count := 0; IsValid := True;
  for I := 1 to Length(S) do begin
    if S[I] = '(' then Inc(Count)
    else if S[I] = ')' then begin
      if Count = 0 then begin IsValid := False; Break; end;
      Dec(Count);
    end;
  end;
  if Count <> 0 then IsValid := False;
  if IsValid then WriteLn('วงเล็บถูกต้อง') else WriteLn('วงเล็บไม่ถูกต้อง');
end.

// ข้อ 10: Word Frequency
program WordFrequency;
uses SysUtils;
var S: String; Words: TStringArray; I, J: Integer;
    Unique: array[0..99] of String; Counts: array[0..99] of Integer;
    UniqueCount: Integer; Found: Boolean;
begin
  Write('ป้อนประโยค: '); ReadLn(S);
  S := LowerCase(S);
  Words := SplitString(S, ' ');
  UniqueCount := 0;
  for I := 0 to High(Words) do begin
    if Words[I] = '' then Continue;
    Found := False;
    for J := 0 to UniqueCount - 1 do
      if Unique[J] = Words[I] then begin Inc(Counts[J]); Found := True; Break; end;
    if not Found then begin Unique[UniqueCount] := Words[I]; Counts[UniqueCount] := 1; Inc(UniqueCount); end;
  end;
  for I := 0 to UniqueCount - 1 do
    WriteLn(Unique[I], ': ', Counts[I]);
end.
```

### ข้อที่ 11-20 (สั้น)

```pascal
// ข้อ 11: Remove punctuation
program RemovePunct;
var S, Clean: String; I: Integer;
begin
  Write('ข้อความ: '); ReadLn(S);
  Clean := '';
  for I := 1 to Length(S) do
    if S[I] in ['a'..'z','A'..'Z','0'..'9',' '] then Clean := Clean + S[I];
  WriteLn('ไม่มีเครื่องหมาย: ', Clean);
end.

// ข้อ 12: Capitalize first letter of each sentence
program CapitalizeSentences;
var S: String; I: Integer; AfterPeriod: Boolean;
begin
  Write('ข้อความ: '); ReadLn(S);
  AfterPeriod := True;
  for I := 1 to Length(S) do begin
    if AfterPeriod and (S[I] in ['a'..'z']) then begin S[I] := UpCase(S[I]); AfterPeriod := False; end
    else if S[I] = '.' then AfterPeriod := True
    else if S[I] <> ' ' then AfterPeriod := False;
  end;
  WriteLn(S);
end.

// ข้อ 13: Truncate string
program TruncateString;
uses SysUtils;
var S: String; MaxLen: Integer;
begin
  Write('ข้อความ: '); ReadLn(S);
  Write('ความยาวสูงสุด: '); ReadLn(MaxLen);
  if Length(S) > MaxLen then
    WriteLn(Copy(S, 1, MaxLen - 3) + '...')
  else
    WriteLn(S);
end.

// ข้อ 14: Check if string is numeric
program IsNumeric;
var S: String; I: Integer; IsNum: Boolean;
begin
  Write('ข้อความ: '); ReadLn(S);
  IsNum := (Length(S) > 0);
  for I := 1 to Length(S) do
    if not (S[I] in ['0'..'9', '.', '-', '+']) then begin IsNum := False; Break; end;
  if IsNum then WriteLn('เป็นตัวเลข') else WriteLn('ไม่ใช่ตัวเลข');
end.

// ข้อ 15: Pad string
program PadString;
var S: String; Width: Integer;
begin
  Write('ข้อความ: '); ReadLn(S);
  Write('ความกว้าง: '); ReadLn(Width);
  // Pad right
  WriteLn(Format('%-*s|', [Width, S]));
  // Pad left
  WriteLn(Format('%*s|', [Width, S]));
end.

// ข้อ 16-20 ทดสอบ string functions ต่างๆ
program StringFuncTest;
uses SysUtils;
begin
  var S := '  Hello, World!  ';
  WriteLn('Original:   "', S, '"');
  WriteLn('Trim:       "', Trim(S), '"');
  WriteLn('Upper:      "', UpperCase(Trim(S)), '"');
  WriteLn('Lower:      "', LowerCase(Trim(S)), '"');
  WriteLn('Length:     ', Length(Trim(S)));
  WriteLn('Pos World:  ', Pos('World', S));
  WriteLn('Copy 8,5:   "', Copy(Trim(S), 8, 5), '"');
  var T := Trim(S);
  Delete(T, 1, 7);
  WriteLn('Delete 1,7: "', T, '"');
  Insert('Dear ', T, 1);
  WriteLn('Insert:     "', T, '"');
  WriteLn('Replace:    "', StringReplace(Trim(S), 'World', 'Pascal', [rfReplaceAll]), '"');
end.
```

---

## สรุปฟังก์ชัน String สำคัญ

| ฟังก์ชัน | ใช้งาน |
|---------|--------|
| `Length(S)` | ความยาวสตริง |
| `Copy(S, Start, N)` | ตัดสตริง |
| `Pos(Sub, S)` | หาตำแหน่ง |
| `Delete(S, Start, N)` | ลบอักขระ |
| `Insert(Sub, S, Pos)` | แทรกสตริง |
| `UpperCase(S)` | ตัวพิมพ์ใหญ่ |
| `LowerCase(S)` | ตัวพิมพ์เล็ก |
| `Trim(S)` | ลบ spaces |
| `IntToStr(N)` | เลข->สตริง |
| `StrToInt(S)` | สตริง->เลข |
| `Format(Fmt, Args)` | จัดรูปแบบ |
| `StringReplace(...)` | ค้นหาและแทนที่ |

---

*จบ Part 09 - สตริง (Strings)*
