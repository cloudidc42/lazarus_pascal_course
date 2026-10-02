# Part 13 - Units และ Modules

## บทนำ

Unit คือ module ของ Pascal ที่ช่วยแบ่งโค้ดออกเป็นส่วนๆ ทำให้จัดการโค้ดได้ดีขึ้น นำ code กลับมาใช้ใหม่ได้ และซ่อน implementation details ได้ Unit เป็นรากฐานของการเขียนโปรแกรมแบบ modular ที่ดี

---

## 13.1 โครงสร้างของ Unit

```pascal
unit UnitName;

// ส่วนที่ 1: Interface - ประกาศสิ่งที่จะ export ออกไป
interface

uses
  // units ที่ต้องการใน interface

// ส่วนที่ 2: Implementation - โค้ดจริงของ unit
implementation

uses
  // units ที่ต้องการใน implementation

// ส่วนที่ 3: Initialization - รันเมื่อโหลด unit (optional)
initialization
  // โค้ด

// ส่วนที่ 4: Finalization - รันเมื่อปิดโปรแกรม (optional)
finalization
  // โค้ด

end.
```

---

## 13.2 Interface Section

Interface section ประกาศ:
- Types ที่ต้องการ export
- Constants ที่ต้องการ export
- Variables ที่ต้องการ export
- Function/Procedure signatures (declarations)

### ตัวอย่างที่ 1: Unit พื้นฐาน - MathUtils

**ไฟล์: MathUtils.pas**

```pascal
unit MathUtils;

{$mode objfpc}{$H+}

interface

const
  PI      = 3.14159265358979;
  E       = 2.71828182845905;
  EPSILON = 1e-10;

type
  TAngleMode = (amDegrees, amRadians);

// ฟังก์ชันคณิตศาสตร์พื้นฐาน
function Max(A, B: Real): Real; overload;
function Min(A, B: Real): Real; overload;
function Max(A, B: Integer): Integer; overload;
function Min(A, B: Integer): Integer; overload;
function Clamp(Value, MinVal, MaxVal: Real): Real;
function Lerp(A, B, T: Real): Real;
function RoundTo(Value: Real; Decimals: Integer): Real;

// ฟังก์ชันตรีโกณมิติ
function DegToRad(Degrees: Real): Real;
function RadToDeg(Radians: Real): Real;
function SinDeg(Degrees: Real): Real;
function CosDeg(Degrees: Real): Real;
function TanDeg(Degrees: Real): Real;

// ฟังก์ชันสถิติ
function Average(const Data: array of Real): Real;
function Variance(const Data: array of Real): Real;
function StdDev(const Data: array of Real): Real;
function Median(Data: array of Real): Real;

// ฟังก์ชันจำนวนเต็ม
function GCD(A, B: Integer): Integer;
function LCM(A, B: Integer): Integer;
function IsPrime(N: Integer): Boolean;
function Factorial(N: Integer): Int64;
function Fibonacci(N: Integer): Int64;
function Power(Base: Real; Exp: Integer): Real;

implementation

uses Math;

function Max(A, B: Real): Real; overload;
begin
  if A > B then Result := A else Result := B;
end;

function Min(A, B: Real): Real; overload;
begin
  if A < B then Result := A else Result := B;
end;

function Max(A, B: Integer): Integer; overload;
begin
  if A > B then Result := A else Result := B;
end;

function Min(A, B: Integer): Integer; overload;
begin
  if A < B then Result := A else Result := B;
end;

function Clamp(Value, MinVal, MaxVal: Real): Real;
begin
  if Value < MinVal then Result := MinVal
  else if Value > MaxVal then Result := MaxVal
  else Result := Value;
end;

function Lerp(A, B, T: Real): Real;
begin
  Result := A + T * (B - A);
end;

function RoundTo(Value: Real; Decimals: Integer): Real;
var
  Factor: Real;
begin
  Factor := Power(10, Decimals);
  Result := Round(Value * Factor) / Factor;
end;

function DegToRad(Degrees: Real): Real;
begin
  Result := Degrees * PI / 180;
end;

function RadToDeg(Radians: Real): Real;
begin
  Result := Radians * 180 / PI;
end;

function SinDeg(Degrees: Real): Real;
begin
  Result := Sin(DegToRad(Degrees));
end;

function CosDeg(Degrees: Real): Real;
begin
  Result := Cos(DegToRad(Degrees));
end;

function TanDeg(Degrees: Real): Real;
begin
  Result := Tan(DegToRad(Degrees));
end;

function Average(const Data: array of Real): Real;
var
  i   : Integer;
  Sum : Real;
begin
  if Length(Data) = 0 then begin Result := 0; Exit; end;
  Sum := 0;
  for i := 0 to High(Data) do Sum := Sum + Data[i];
  Result := Sum / Length(Data);
end;

function Variance(const Data: array of Real): Real;
var
  i    : Integer;
  Avg  : Real;
  Sum  : Real;
begin
  if Length(Data) < 2 then begin Result := 0; Exit; end;
  Avg := Average(Data);
  Sum := 0;
  for i := 0 to High(Data) do
    Sum := Sum + Sqr(Data[i] - Avg);
  Result := Sum / Length(Data);
end;

function StdDev(const Data: array of Real): Real;
begin
  Result := Sqrt(Variance(Data));
end;

function Median(Data: array of Real): Real;
var
  i, j : Integer;
  Temp : Real;
  n    : Integer;
begin
  n := Length(Data);
  // Bubble sort
  for i := 0 to n - 2 do
    for j := 0 to n - 2 - i do
      if Data[j] > Data[j + 1] then
      begin
        Temp       := Data[j];
        Data[j]    := Data[j + 1];
        Data[j + 1] := Temp;
      end;
  
  if n mod 2 = 0 then
    Result := (Data[n div 2 - 1] + Data[n div 2]) / 2
  else
    Result := Data[n div 2];
end;

function GCD(A, B: Integer): Integer;
begin
  while B <> 0 do
  begin
    A := A mod B;
    A := B;
    B := A;
  end;
  // ใช้ Euclidean algorithm
  if B = 0 then Result := A
  else Result := GCD(B, A mod B);
end;

function LCM(A, B: Integer): Integer;
begin
  Result := Abs(A * B) div GCD(A, B);
end;

function IsPrime(N: Integer): Boolean;
var
  i : Integer;
begin
  if N < 2 then begin Result := False; Exit; end;
  if N = 2 then begin Result := True; Exit; end;
  if N mod 2 = 0 then begin Result := False; Exit; end;
  i := 3;
  while i * i <= N do
  begin
    if N mod i = 0 then begin Result := False; Exit; end;
    Inc(i, 2);
  end;
  Result := True;
end;

function Factorial(N: Integer): Int64;
begin
  if N <= 1 then Result := 1
  else Result := N * Factorial(N - 1);
end;

function Fibonacci(N: Integer): Int64;
var
  A, B, C : Int64;
  i       : Integer;
begin
  if N <= 0 then begin Result := 0; Exit; end;
  if N = 1  then begin Result := 1; Exit; end;
  A := 0; B := 1;
  for i := 2 to N do
  begin
    C := A + B;
    A := B;
    B := C;
  end;
  Result := B;
end;

function Power(Base: Real; Exp: Integer): Real;
var
  i : Integer;
begin
  Result := 1;
  if Exp >= 0 then
    for i := 1 to Exp do Result := Result * Base
  else
    for i := 1 to -Exp do Result := Result / Base;
end;

end.
```

---

## 13.3 Implementation Section

Implementation section มีโค้ดจริงของฟังก์ชัน และสิ่งที่ประกาศในนี้ (private) จะไม่สามารถใช้ได้จากภายนอก

### ตัวอย่างที่ 2: Unit ที่ใช้ Private Variables

**ไฟล์: Counter.pas**

```pascal
unit Counter;

{$mode objfpc}{$H+}

interface

// Public API
procedure ResetCounter;
procedure IncrementCounter;
procedure DecrementCounter;
function GetCount: Integer;
function GetHistory: String;

implementation

uses SysUtils;

// Private - ไม่ export ออกไป
var
  FCount   : Integer;
  FHistory : String;

procedure ResetCounter;
begin
  FCount   := 0;
  FHistory := FHistory + Format('[Reset at %s] ', [TimeToStr(Now)]);
end;

procedure IncrementCounter;
begin
  Inc(FCount);
  FHistory := FHistory + '+';
end;

procedure DecrementCounter;
begin
  if FCount > 0 then
  begin
    Dec(FCount);
    FHistory := FHistory + '-';
  end;
end;

function GetCount: Integer;
begin
  Result := FCount;
end;

function GetHistory: String;
begin
  Result := FHistory;
end;

initialization
  FCount   := 0;
  FHistory := '';

end.
```

**โปรแกรมหลัก:**

```pascal
program UseCounter;

uses Counter;

begin
  IncrementCounter;
  IncrementCounter;
  IncrementCounter;
  DecrementCounter;
  IncrementCounter;
  
  WriteLn('Count: ', GetCount);
  WriteLn('History: ', GetHistory);
  
  ResetCounter;
  WriteLn('After reset: ', GetCount);
  
  ReadLn;
end.
```

---

## 13.4 Initialization และ Finalization

### ตัวอย่างที่ 3: Unit ที่มี Initialization/Finalization

**ไฟล์: AppConfig.pas**

```pascal
unit AppConfig;

{$mode objfpc}{$H+}

interface

uses Classes;

type
  TAppConfig = record
    AppName    : String;
    Version    : String;
    Author     : String;
    DebugMode  : Boolean;
    MaxRetries : Integer;
    Timeout    : Integer;
  end;

var
  Config : TAppConfig;

procedure SaveConfig(FileName: String);
procedure LoadConfig(FileName: String);
function GetConfigValue(Key: String): String;
procedure SetConfigValue(Key, Value: String);

implementation

uses SysUtils;

var
  FConfigFile   : String;
  FConfigLoaded : Boolean;
  FInternalSL   : TStringList;

procedure SaveConfig(FileName: String);
begin
  FInternalSL.Clear;
  FInternalSL.Values['AppName']    := Config.AppName;
  FInternalSL.Values['Version']    := Config.Version;
  FInternalSL.Values['Author']     := Config.Author;
  FInternalSL.Values['DebugMode']  := BoolToStr(Config.DebugMode, True);
  FInternalSL.Values['MaxRetries'] := IntToStr(Config.MaxRetries);
  FInternalSL.Values['Timeout']    := IntToStr(Config.Timeout);
  FInternalSL.SaveToFile(FileName);
  FConfigFile := FileName;
end;

procedure LoadConfig(FileName: String);
begin
  if not FileExists(FileName) then Exit;
  FInternalSL.LoadFromFile(FileName);
  FConfigFile := FileName;
  
  Config.AppName    := FInternalSL.Values['AppName'];
  Config.Version    := FInternalSL.Values['Version'];
  Config.Author     := FInternalSL.Values['Author'];
  Config.DebugMode  := StrToBoolDef(FInternalSL.Values['DebugMode'], False);
  Config.MaxRetries := StrToIntDef(FInternalSL.Values['MaxRetries'], 3);
  Config.Timeout    := StrToIntDef(FInternalSL.Values['Timeout'], 30);
  FConfigLoaded     := True;
end;

function GetConfigValue(Key: String): String;
begin
  Result := FInternalSL.Values[Key];
end;

procedure SetConfigValue(Key, Value: String);
begin
  FInternalSL.Values[Key] := Value;
end;

initialization
  // รันก่อน program เริ่ม
  FInternalSL   := TStringList.Create;
  FInternalSL.NameValueSeparator := '=';
  FConfigLoaded := False;
  FConfigFile   := '';
  
  // ค่าเริ่มต้น
  Config.AppName    := 'My App';
  Config.Version    := '1.0.0';
  Config.Author     := 'Unknown';
  Config.DebugMode  := False;
  Config.MaxRetries := 3;
  Config.Timeout    := 30;
  
  WriteLn('[AppConfig] Initialized');

finalization
  // รันเมื่อโปรแกรมปิด
  if FConfigFile <> '' then
    SaveConfig(FConfigFile);
  
  FInternalSL.Free;
  WriteLn('[AppConfig] Finalized');

end.
```

---

## 13.5 การสร้างและใช้งาน Units

### ตัวอย่างที่ 4: Unit สำหรับ String Utilities

**ไฟล์: StringUtils.pas**

```pascal
unit StringUtils;

{$mode objfpc}{$H+}

interface

// การแปลง
function TrimAll(S: String): String;
function RemoveSpaces(S: String): String;
function CollapseSpaces(S: String): String;

// การตรวจสอบ
function IsNumeric(S: String): Boolean;
function IsAlpha(S: String): Boolean;
function IsAlphaNumeric(S: String): Boolean;
function IsEmail(S: String): Boolean;
function IsThaiPhone(S: String): Boolean;

// การจัดการ
function Repeat_Str(S: String; N: Integer): String;
function Reverse_Str(S: String): String;
function PadLeft(S: String; Width: Integer; PadChar: Char = ' '): String;
function PadRight(S: String; Width: Integer; PadChar: Char = ' '): String;
function PadCenter(S: String; Width: Integer; PadChar: Char = ' '): String;
function WordCount(S: String): Integer;
function GetWord(S: String; Index: Integer): String;
function ReplaceAll(S, OldStr, NewStr: String): String;

// การแยก
function SplitString(S, Delimiter: String): TStringArray;
function JoinStrings(Arr: array of String; Delimiter: String): String;

// การแปลงตัวอักษร
function ToTitleCase(S: String): String;

implementation

uses SysUtils, StrUtils;

function TrimAll(S: String): String;
begin
  Result := Trim(S);
  while Pos('  ', Result) > 0 do
    Result := StringReplace(Result, '  ', ' ', [rfReplaceAll]);
end;

function RemoveSpaces(S: String): String;
begin
  Result := StringReplace(S, ' ', '', [rfReplaceAll]);
end;

function CollapseSpaces(S: String): String;
begin
  Result := TrimAll(S);
end;

function IsNumeric(S: String): Boolean;
var
  i : Integer;
  HasDot : Boolean;
begin
  Result := False;
  if Length(S) = 0 then Exit;
  HasDot := False;
  for i := 1 to Length(S) do
  begin
    if S[i] = '.' then
    begin
      if HasDot then Exit;
      HasDot := True;
    end
    else if not (S[i] in ['0'..'9']) then
    begin
      if (i = 1) and (S[i] in ['-', '+']) then Continue;
      Exit;
    end;
  end;
  Result := True;
end;

function IsAlpha(S: String): Boolean;
var i: Integer;
begin
  Result := Length(S) > 0;
  for i := 1 to Length(S) do
    if not (S[i] in ['A'..'Z', 'a'..'z']) then
    begin
      Result := False;
      Exit;
    end;
end;

function IsAlphaNumeric(S: String): Boolean;
var i: Integer;
begin
  Result := Length(S) > 0;
  for i := 1 to Length(S) do
    if not (S[i] in ['A'..'Z', 'a'..'z', '0'..'9']) then
    begin
      Result := False;
      Exit;
    end;
end;

function IsEmail(S: String): Boolean;
var
  AtPos  : Integer;
  DotPos : Integer;
begin
  Result := False;
  AtPos  := Pos('@', S);
  if AtPos < 2 then Exit;
  DotPos := LastDelimiter('.', S);
  if DotPos <= AtPos + 1 then Exit;
  if DotPos >= Length(S) - 1 then Exit;
  Result := True;
end;

function IsThaiPhone(S: String): Boolean;
var
  Clean : String;
  i     : Integer;
begin
  Result := False;
  Clean  := '';
  for i := 1 to Length(S) do
    if S[i] in ['0'..'9'] then
      Clean := Clean + S[i];
  Result := (Length(Clean) = 10) and (Clean[1] = '0');
end;

function Repeat_Str(S: String; N: Integer): String;
var
  i : Integer;
begin
  Result := '';
  for i := 1 to N do
    Result := Result + S;
end;

function Reverse_Str(S: String): String;
var
  i : Integer;
begin
  Result := '';
  for i := Length(S) downto 1 do
    Result := Result + S[i];
end;

function PadLeft(S: String; Width: Integer; PadChar: Char = ' '): String;
begin
  if Length(S) >= Width then Result := S
  else Result := Repeat_Str(PadChar, Width - Length(S)) + S;
end;

function PadRight(S: String; Width: Integer; PadChar: Char = ' '): String;
begin
  if Length(S) >= Width then Result := S
  else Result := S + Repeat_Str(PadChar, Width - Length(S));
end;

function PadCenter(S: String; Width: Integer; PadChar: Char = ' '): String;
var
  Pad, PadL, PadR : Integer;
begin
  if Length(S) >= Width then begin Result := S; Exit; end;
  Pad  := Width - Length(S);
  PadL := Pad div 2;
  PadR := Pad - PadL;
  Result := Repeat_Str(PadChar, PadL) + S + Repeat_Str(PadChar, PadR);
end;

function WordCount(S: String): Integer;
var
  i      : Integer;
  InWord : Boolean;
begin
  Result := 0;
  InWord := False;
  for i := 1 to Length(S) do
  begin
    if S[i] = ' ' then InWord := False
    else if not InWord then
    begin
      Inc(Result);
      InWord := True;
    end;
  end;
end;

function GetWord(S: String; Index: Integer): String;
var
  i       : Integer;
  WordNum : Integer;
  Start   : Integer;
begin
  Result  := '';
  WordNum := 0;
  i       := 1;
  
  while i <= Length(S) do
  begin
    while (i <= Length(S)) and (S[i] = ' ') do Inc(i);
    if i > Length(S) then Break;
    
    Inc(WordNum);
    Start := i;
    while (i <= Length(S)) and (S[i] <> ' ') do Inc(i);
    
    if WordNum = Index then
    begin
      Result := Copy(S, Start, i - Start);
      Exit;
    end;
  end;
end;

function ReplaceAll(S, OldStr, NewStr: String): String;
begin
  Result := StringReplace(S, OldStr, NewStr, [rfReplaceAll, rfIgnoreCase]);
end;

function SplitString(S, Delimiter: String): TStringArray;
var
  Parts  : TStringList;
  i      : Integer;
  DelPos : Integer;
begin
  Parts := TStringList.Create;
  try
    while Length(S) > 0 do
    begin
      DelPos := Pos(Delimiter, S);
      if DelPos > 0 then
      begin
        Parts.Add(Copy(S, 1, DelPos - 1));
        S := Copy(S, DelPos + Length(Delimiter), MaxInt);
      end
      else
      begin
        Parts.Add(S);
        S := '';
      end;
    end;
    
    SetLength(Result, Parts.Count);
    for i := 0 to Parts.Count - 1 do
      Result[i] := Parts[i];
  finally
    Parts.Free;
  end;
end;

function JoinStrings(Arr: array of String; Delimiter: String): String;
var
  i : Integer;
begin
  Result := '';
  for i := 0 to High(Arr) do
  begin
    if i > 0 then Result := Result + Delimiter;
    Result := Result + Arr[i];
  end;
end;

function ToTitleCase(S: String): String;
var
  i          : Integer;
  NextUpper  : Boolean;
begin
  Result    := LowerCase(S);
  NextUpper := True;
  
  for i := 1 to Length(Result) do
  begin
    if Result[i] = ' ' then NextUpper := True
    else if NextUpper then
    begin
      Result[i] := UpCase(Result[i]);
      NextUpper  := False;
    end;
  end;
end;

end.
```

### ตัวอย่างที่ 5: โปรแกรมทดสอบ StringUtils

```pascal
program TestStringUtils;

{$mode objfpc}{$H+}

uses SysUtils, StringUtils;

var
  Parts : TStringArray;
  i     : Integer;

begin
  WriteLn('=== ทดสอบ StringUtils ===');
  WriteLn;
  
  // การตรวจสอบ
  WriteLn('--- ตรวจสอบ ---');
  WriteLn('IsNumeric("123.45")   : ', IsNumeric('123.45'));
  WriteLn('IsNumeric("abc")      : ', IsNumeric('abc'));
  WriteLn('IsAlpha("Hello")      : ', IsAlpha('Hello'));
  WriteLn('IsAlpha("Hello123")   : ', IsAlpha('Hello123'));
  WriteLn('IsEmail("a@b.com")    : ', IsEmail('a@b.com'));
  WriteLn('IsEmail("invalid")    : ', IsEmail('invalid'));
  WriteLn('IsThaiPhone("0812345678"): ', IsThaiPhone('0812345678'));
  WriteLn('IsThaiPhone("123456")    : ', IsThaiPhone('123456'));
  WriteLn;
  
  // การจัดการ
  WriteLn('--- จัดการ ---');
  WriteLn('Reverse("Pascal")    : ', Reverse_Str('Pascal'));
  WriteLn('Repeat("ab", 3)      : ', Repeat_Str('ab', 3));
  WriteLn('PadLeft("Hi", 10)    : [', PadLeft('Hi', 10), ']');
  WriteLn('PadRight("Hi", 10)   : [', PadRight('Hi', 10), ']');
  WriteLn('PadCenter("Hi", 10)  : [', PadCenter('Hi', 10), ']');
  WriteLn('WordCount("a b c d") : ', WordCount('a b c d'));
  WriteLn('GetWord("hello world", 2): ', GetWord('hello world and', 2));
  WriteLn('ToTitleCase("hello world"): ', ToTitleCase('hello world'));
  WriteLn;
  
  // Split/Join
  WriteLn('--- Split/Join ---');
  Parts := SplitString('a,b,c,d,e', ',');
  Write('Split("a,b,c,d,e", ","): ');
  for i := 0 to High(Parts) do
    Write('[', Parts[i], ']');
  WriteLn;
  WriteLn('Join(parts, "-"): ', JoinStrings(Parts, '-'));
  WriteLn;
  
  // Email validation
  WriteLn('--- ตรวจสอบอีเมล ---');
  var emails := ['user@example.com', 'invalid', 'a@b', 'test@domain.co.th'];
  for i := 0 to High(emails) do
    WriteLn(Format('%-25s: %s', [emails[i],
      BoolToStr(IsEmail(emails[i]), 'Valid', 'Invalid')]));
  
  ReadLn;
end.
```

---

## 13.6 Circular Dependencies

Circular dependency เกิดเมื่อ Unit A ใช้ Unit B และ Unit B ใช้ Unit A วิธีแก้คือย้าย type ที่ใช้ร่วมกันไปยัง unit ที่สาม

### ตัวอย่างที่ 6: การแก้ Circular Dependency

**ปัญหา (ห้ามทำแบบนี้):**
```
Unit A -> uses Unit B
Unit B -> uses Unit A  <- Circular!
```

**วิธีแก้:**

**ไฟล์: Types.pas (shared types)**
```pascal
unit Types;

{$mode objfpc}{$H+}

interface

type
  TPersonID = Integer;
  TDeptID   = Integer;
  
  TPersonInfo = record
    ID   : TPersonID;
    Name : String[50];
    Dept : TDeptID;
  end;
  
  TDeptInfo = record
    ID      : TDeptID;
    Name    : String[30];
    Manager : TPersonID;
  end;

implementation

end.
```

**ไฟล์: PersonUnit.pas**
```pascal
unit PersonUnit;

{$mode objfpc}{$H+}

interface

uses Types;

function CreatePerson(ID: TPersonID; Name: String; DeptID: TDeptID): TPersonInfo;
procedure DisplayPerson(P: TPersonInfo);

implementation

uses SysUtils;

function CreatePerson(ID: TPersonID; Name: String; DeptID: TDeptID): TPersonInfo;
begin
  Result.ID   := ID;
  Result.Name := Name;
  Result.Dept := DeptID;
end;

procedure DisplayPerson(P: TPersonInfo);
begin
  WriteLn('Person: ID=', P.ID, ' Name=', P.Name, ' DeptID=', P.Dept);
end;

end.
```

**ไฟล์: DeptUnit.pas**
```pascal
unit DeptUnit;

{$mode objfpc}{$H+}

interface

uses Types;

function CreateDept(ID: TDeptID; Name: String; ManagerID: TPersonID): TDeptInfo;
procedure DisplayDept(D: TDeptInfo);

implementation

uses SysUtils;

function CreateDept(ID: TDeptID; Name: String; ManagerID: TPersonID): TDeptInfo;
begin
  Result.ID      := ID;
  Result.Name    := Name;
  Result.Manager := ManagerID;
end;

procedure DisplayDept(D: TDeptInfo);
begin
  WriteLn('Dept: ID=', D.ID, ' Name=', D.Name, ' Manager=', D.Manager);
end;

end.
```

---

## 13.7 Standard Units ที่สำคัญ

### System Unit (ใช้อัตโนมัติ)

```pascal
program SystemUnitDemo;

var
  P : Pointer;
  B : Byte;
  I : Integer;

begin
  // Memory allocation
  GetMem(P, 100);
  FreeMem(P);
  
  // Type conversions
  B := 65;
  WriteLn(Chr(B));    // 'A'
  WriteLn(Ord('A'));  // 65
  
  // String
  I := Length('Hello');
  WriteLn(I);
  
  // Halt
  // Halt(0); // ออกจากโปรแกรม
  
  ReadLn;
end.
```

### ตัวอย่างที่ 7: SysUtils Unit

```pascal
program SysUtilsDemo;

{$mode objfpc}{$H+}

uses SysUtils;

begin
  WriteLn('=== SysUtils Demo ===');
  
  // Date/Time
  WriteLn('Now: ', FormatDateTime('dd/mm/yyyy hh:nn:ss', Now));
  WriteLn('Date: ', DateToStr(Date));
  WriteLn('Time: ', TimeToStr(Time));
  
  // String conversion
  WriteLn('IntToStr(42): ', IntToStr(42));
  WriteLn('FloatToStr(3.14): ', FloatToStr(3.14));
  WriteLn('StrToInt("123"): ', StrToInt('123'));
  WriteLn('StrToFloat("3.14"): ', StrToFloat('3.14'));
  WriteLn('StrToIntDef("abc", 0): ', StrToIntDef('abc', 0));
  
  // File
  WriteLn('DirectorySeparator: "', DirectorySeparator, '"');
  WriteLn('PathSeparator: "', PathSeparator, '"');
  
  // Formatting
  WriteLn(Format('%-10s %5d %8.2f', ['Item', 42, 3.14]));
  WriteLn(Format('%08d', [1234]));   // 00001234
  WriteLn(Format('%+.4f', [3.14159])); // +3.1416
  
  // Exception types
  WriteLn;
  WriteLn('Exception types: EConvertError, EDivByZero, EOverflow...');
  
  ReadLn;
end.
```

### ตัวอย่างที่ 8: Math Unit

```pascal
program MathUnitDemo;

{$mode objfpc}{$H+}

uses Math;

var
  Data : array of Double;
  i    : Integer;

begin
  WriteLn('=== Math Unit Demo ===');
  
  // Basic math
  WriteLn('Power(2, 10) = ', Power(2, 10):0:0);
  WriteLn('Log2(1024) = ', Log2(1024):0:2);
  WriteLn('Log10(100) = ', Log10(100):0:2);
  WriteLn('LogN(3, 27) = ', LogN(3, 27):0:2);
  
  // Rounding
  WriteLn('Floor(3.7) = ', Floor(3.7));
  WriteLn('Ceil(3.2) = ', Ceil(3.2));
  WriteLn('Round(3.5) = ', Round(3.5));
  WriteLn('Trunc(3.9) = ', Trunc(3.9));
  
  // Trigonometry (radians)
  WriteLn('Sin(Pi/6) = ', Sin(Pi/6):0:4);    // 0.5
  WriteLn('Cos(Pi/3) = ', Cos(Pi/3):0:4);    // 0.5
  WriteLn('ArcTan(1)*4 = ', ArcTan(1)*4:0:4); // Pi
  
  // Statistics with array
  SetLength(Data, 8);
  Data[0] := 4; Data[1] := 7; Data[2] := 13; Data[3] := 2;
  Data[4] := 1; Data[5] := 9; Data[6] := 5;  Data[7] := 8;
  
  WriteLn;
  WriteLn('Data: 4 7 13 2 1 9 5 8');
  WriteLn('MaxValue = ', MaxValue(Data):0:0);
  WriteLn('MinValue = ', MinValue(Data):0:0);
  WriteLn('Sum      = ', Sum(Data):0:0);
  WriteLn('Mean     = ', Mean(Data):0:4);
  WriteLn('StdDev   = ', StdDev(Data):0:4);
  WriteLn('Variance = ', Variance(Data):0:4);
  
  ReadLn;
end.
```

### ตัวอย่างที่ 9: Classes Unit

```pascal
program ClassesUnitDemo;

{$mode objfpc}{$H+}

uses Classes, SysUtils;

begin
  WriteLn('=== Classes Unit Demo ===');
  
  // TStringList
  var SL := TStringList.Create;
  SL.Add('สมชาย');
  SL.Add('สมหญิง');
  SL.Add('อนุชา');
  SL.Sort;
  WriteLn('TStringList sorted:');
  for var i := 0 to SL.Count - 1 do
    WriteLn('  ', SL[i]);
  SL.Free;
  
  WriteLn;
  
  // TStringList with Name=Value
  var Config := TStringList.Create;
  Config.NameValueSeparator := '=';
  Config.Add('Name=My App');
  Config.Add('Version=1.0');
  Config.Add('Debug=true');
  WriteLn('Config values:');
  WriteLn('  Name: ', Config.Values['Name']);
  WriteLn('  Version: ', Config.Values['Version']);
  WriteLn('  Debug: ', Config.Values['Debug']);
  Config.Free;
  
  WriteLn;
  
  // TMemoryStream
  var MS := TMemoryStream.Create;
  var Msg := 'Hello from MemoryStream!';
  MS.Write(PChar(Msg)^, Length(Msg));
  WriteLn('MemoryStream size: ', MS.Size, ' bytes');
  MS.Position := 0;
  var Buffer: array[0..255] of Char;
  var BytesRead := MS.Read(Buffer, MS.Size);
  Buffer[BytesRead] := #0;
  WriteLn('Read back: ', PChar(@Buffer[0]));
  MS.Free;
  
  ReadLn;
end.
```

---

## 13.8 โปรแกรมตัวอย่าง: สร้าง Utility Library

### ตัวอย่างที่ 10: DateTimeUtils Unit

**ไฟล์: DateTimeUtils.pas**

```pascal
unit DateTimeUtils;

{$mode objfpc}{$H+}

interface

uses SysUtils;

type
  TWeekDay = (wdSunday, wdMonday, wdTuesday, wdWednesday,
              wdThursday, wdFriday, wdSaturday);

// Thai month/day names
function ThaiMonthName(Month: Integer): String;
function ThaiDayName(Day: TWeekDay): String;
function ThaiDateStr(D: TDateTime): String;

// Date calculations
function DaysBetween(D1, D2: TDateTime): Integer;
function AddDays(D: TDateTime; Days: Integer): TDateTime;
function AddMonths(D: TDateTime; Months: Integer): TDateTime;
function IsLeapYear(Year: Integer): Boolean;
function DaysInMonth(Month, Year: Integer): Integer;
function IsWeekend(D: TDateTime): Boolean;
function IsWorkday(D: TDateTime): Boolean;
function NextWorkday(D: TDateTime): TDateTime;
function WorkdaysBetween(D1, D2: TDateTime): Integer;

// Formatting
function FormatThai(D: TDateTime; Fmt: String): String;
function RelativeTime(D: TDateTime): String;

// Age
function CalcAge(BirthDate: TDateTime): Integer;
function CalcAgeDetailed(BirthDate: TDateTime;
                          out Years, Months, Days: Integer): void;

implementation

function ThaiMonthName(Month: Integer): String;
const
  Months: array[1..12] of String = (
    'มกราคม', 'กุมภาพันธ์', 'มีนาคม', 'เมษายน',
    'พฤษภาคม', 'มิถุนายน', 'กรกฎาคม', 'สิงหาคม',
    'กันยายน', 'ตุลาคม', 'พฤศจิกายน', 'ธันวาคม');
begin
  if (Month >= 1) and (Month <= 12) then
    Result := Months[Month]
  else
    Result := 'ไม่ทราบ';
end;

function ThaiDayName(Day: TWeekDay): String;
const
  Days: array[TWeekDay] of String = (
    'อาทิตย์', 'จันทร์', 'อังคาร', 'พุธ',
    'พฤหัสบดี', 'ศุกร์', 'เสาร์');
begin
  Result := Days[Day];
end;

function ThaiDateStr(D: TDateTime): String;
var
  Y, M, Dy : Word;
begin
  DecodeDate(D, Y, M, Dy);
  Result := Format('%d %s %d', [Dy, ThaiMonthName(M), Y + 543]);
end;

function DaysBetween(D1, D2: TDateTime): Integer;
begin
  Result := Abs(Trunc(D2) - Trunc(D1));
end;

function AddDays(D: TDateTime; Days: Integer): TDateTime;
begin
  Result := D + Days;
end;

function AddMonths(D: TDateTime; Months: Integer): TDateTime;
var
  Y, M, Dy : Word;
  TotalMonths : Integer;
begin
  DecodeDate(D, Y, M, Dy);
  TotalMonths := Y * 12 + (M - 1) + Months;
  Y := TotalMonths div 12;
  M := (TotalMonths mod 12) + 1;
  if Dy > DaysInMonth(M, Y) then
    Dy := DaysInMonth(M, Y);
  Result := EncodeDate(Y, M, Dy) + Frac(D);
end;

function IsLeapYear(Year: Integer): Boolean;
begin
  Result := (Year mod 4 = 0) and
            ((Year mod 100 <> 0) or (Year mod 400 = 0));
end;

function DaysInMonth(Month, Year: Integer): Integer;
const
  DIM: array[1..12] of Integer = (31,28,31,30,31,30,31,31,30,31,30,31);
begin
  if (Month = 2) and IsLeapYear(Year) then
    Result := 29
  else
    Result := DIM[Month];
end;

function IsWeekend(D: TDateTime): Boolean;
var
  DayOfWeek : Integer;
begin
  DayOfWeek := DayOfWeek(D);
  Result    := DayOfWeek in [1, 7]; // 1=Sunday, 7=Saturday
end;

function IsWorkday(D: TDateTime): Boolean;
begin
  Result := not IsWeekend(D);
end;

function NextWorkday(D: TDateTime): TDateTime;
begin
  Result := D + 1;
  while IsWeekend(Result) do
    Result := Result + 1;
end;

function WorkdaysBetween(D1, D2: TDateTime): Integer;
var
  D     : TDateTime;
  Count : Integer;
begin
  Count := 0;
  D     := Trunc(D1);
  while D < Trunc(D2) do
  begin
    if IsWorkday(D) then Inc(Count);
    D := D + 1;
  end;
  Result := Count;
end;

function FormatThai(D: TDateTime; Fmt: String): String;
var
  Y, M, Dy : Word;
  H, Min, S, MS : Word;
begin
  DecodeDate(D, Y, M, Dy);
  DecodeTime(D, H, Min, S, MS);
  
  Result := Fmt;
  Result := StringReplace(Result, 'YYYY', IntToStr(Y + 543), [rfReplaceAll]);
  Result := StringReplace(Result, 'YY', Copy(IntToStr(Y + 543), 3, 2), [rfReplaceAll]);
  Result := StringReplace(Result, 'MMMM', ThaiMonthName(M), [rfReplaceAll]);
  Result := StringReplace(Result, 'MM', Format('%02d', [M]), [rfReplaceAll]);
  Result := StringReplace(Result, 'DD', Format('%02d', [Dy]), [rfReplaceAll]);
  Result := StringReplace(Result, 'HH', Format('%02d', [H]), [rfReplaceAll]);
  Result := StringReplace(Result, 'NN', Format('%02d', [Min]), [rfReplaceAll]);
  Result := StringReplace(Result, 'SS', Format('%02d', [S]), [rfReplaceAll]);
end;

function RelativeTime(D: TDateTime): String;
var
  Diff : Integer;
begin
  Diff := DaysBetween(D, Now);
  
  if Diff = 0 then Result := 'วันนี้'
  else if Diff = 1 then
    if D < Now then Result := 'เมื่อวาน'
    else Result := 'พรุ่งนี้'
  else if Diff < 7 then
    if D < Now then Result := Format('%d วันที่แล้ว', [Diff])
    else Result := Format('อีก %d วัน', [Diff])
  else if Diff < 30 then
    if D < Now then Result := Format('%d สัปดาห์ที่แล้ว', [Diff div 7])
    else Result := Format('อีก %d สัปดาห์', [Diff div 7])
  else if Diff < 365 then
    if D < Now then Result := Format('%d เดือนที่แล้ว', [Diff div 30])
    else Result := Format('อีก %d เดือน', [Diff div 30])
  else
    if D < Now then Result := Format('%d ปีที่แล้ว', [Diff div 365])
    else Result := Format('อีก %d ปี', [Diff div 365]);
end;

function CalcAge(BirthDate: TDateTime): Integer;
var
  BY, BM, BD : Word;
  TY, TM, TD : Word;
begin
  DecodeDate(BirthDate, BY, BM, BD);
  DecodeDate(Now, TY, TM, TD);
  
  Result := TY - BY;
  if (TM < BM) or ((TM = BM) and (TD < BD)) then
    Dec(Result);
end;

procedure CalcAgeDetailed(BirthDate: TDateTime;
                           out Years, Months, Days: Integer);
var
  BY, BM, BD : Word;
  TY, TM, TD : Word;
begin
  DecodeDate(BirthDate, BY, BM, BD);
  DecodeDate(Now, TY, TM, TD);
  
  Years  := TY - BY;
  Months := TM - BM;
  Days   := TD - BD;
  
  if Days < 0 then
  begin
    Dec(Months);
    Days := Days + DaysInMonth(BM, BY);
  end;
  if Months < 0 then
  begin
    Dec(Years);
    Months := Months + 12;
  end;
end;

end.
```

---

## 13.9 โปรแกรมตัวอย่าง: Validation Unit

**ไฟล์: Validation.pas**

```pascal
unit Validation;

{$mode objfpc}{$H+}

interface

type
  TValidationResult = record
    IsValid  : Boolean;
    Message  : String;
    Field    : String;
  end;

// Validators
function ValidateEmail(Email: String): TValidationResult;
function ValidatePhone(Phone: String): TValidationResult;
function ValidateThaiID(ID: String): TValidationResult;
function ValidatePostalCode(Code: String): TValidationResult;
function ValidateURL(URL: String): TValidationResult;
function ValidatePassword(Password: String;
                           MinLength: Integer = 8): TValidationResult;
function ValidateAge(Age: Integer;
                     MinAge: Integer = 0;
                     MaxAge: Integer = 150): TValidationResult;
function ValidateRange(Value, Min, Max: Real;
                       FieldName: String = 'ค่า'): TValidationResult;

// Helper
function OK(Field: String = ''): TValidationResult;
function Fail(Message: String; Field: String = ''): TValidationResult;

implementation

uses SysUtils;

function OK(Field: String = ''): TValidationResult;
begin
  Result.IsValid := True;
  Result.Message := '';
  Result.Field   := Field;
end;

function Fail(Message: String; Field: String = ''): TValidationResult;
begin
  Result.IsValid := False;
  Result.Message := Message;
  Result.Field   := Field;
end;

function ValidateEmail(Email: String): TValidationResult;
var
  AtPos  : Integer;
  DotPos : Integer;
begin
  if Length(Email) = 0 then begin Result := Fail('กรุณากรอกอีเมล', 'email'); Exit; end;
  
  AtPos := Pos('@', Email);
  if AtPos < 2 then begin Result := Fail('อีเมลไม่ถูกต้อง', 'email'); Exit; end;
  
  DotPos := LastDelimiter('.', Email);
  if (DotPos <= AtPos + 1) or (DotPos >= Length(Email)) then
    begin Result := Fail('อีเมลไม่ถูกต้อง', 'email'); Exit; end;
  
  Result := OK('email');
end;

function ValidatePhone(Phone: String): TValidationResult;
var
  Clean : String;
  i     : Integer;
begin
  Clean := '';
  for i := 1 to Length(Phone) do
    if Phone[i] in ['0'..'9'] then Clean := Clean + Phone[i];
  
  if Length(Clean) <> 10 then
    begin Result := Fail('เบอร์โทรต้องมี 10 หลัก', 'phone'); Exit; end;
  
  if not (Clean[1] in ['0']) then
    begin Result := Fail('เบอร์โทรต้องเริ่มด้วย 0', 'phone'); Exit; end;
  
  Result := OK('phone');
end;

function ValidateThaiID(ID: String): TValidationResult;
var
  Clean  : String;
  i      : Integer;
  Sum    : Integer;
  Check  : Integer;
begin
  // ลบอักขระที่ไม่ใช่ตัวเลข
  Clean := '';
  for i := 1 to Length(ID) do
    if ID[i] in ['0'..'9'] then Clean := Clean + ID[i];
  
  if Length(Clean) <> 13 then
    begin Result := Fail('เลขบัตรประชาชนต้องมี 13 หลัก', 'thaiid'); Exit; end;
  
  // ตรวจสอบ checksum
  Sum := 0;
  for i := 1 to 12 do
    Sum := Sum + StrToInt(Clean[i]) * (13 - i);
  
  Check := (11 - (Sum mod 11)) mod 10;
  
  if Check <> StrToInt(Clean[13]) then
    begin Result := Fail('เลขบัตรประชาชนไม่ถูกต้อง', 'thaiid'); Exit; end;
  
  Result := OK('thaiid');
end;

function ValidatePostalCode(Code: String): TValidationResult;
var
  i : Integer;
begin
  if Length(Code) <> 5 then
    begin Result := Fail('รหัสไปรษณีย์ต้องมี 5 หลัก', 'postal'); Exit; end;
  
  for i := 1 to 5 do
    if not (Code[i] in ['0'..'9']) then
      begin Result := Fail('รหัสไปรษณีย์ต้องเป็นตัวเลข', 'postal'); Exit; end;
  
  Result := OK('postal');
end;

function ValidateURL(URL: String): TValidationResult;
begin
  if Length(URL) = 0 then
    begin Result := Fail('กรุณากรอก URL', 'url'); Exit; end;
  
  if (Pos('http://', LowerCase(URL)) <> 1) and
     (Pos('https://', LowerCase(URL)) <> 1) then
    begin Result := Fail('URL ต้องเริ่มด้วย http:// หรือ https://', 'url'); Exit; end;
  
  if Pos('.', URL) = 0 then
    begin Result := Fail('URL ไม่ถูกต้อง', 'url'); Exit; end;
  
  Result := OK('url');
end;

function ValidatePassword(Password: String; MinLength: Integer = 8): TValidationResult;
var
  HasUpper, HasLower, HasDigit, HasSpecial : Boolean;
  i : Integer;
begin
  if Length(Password) < MinLength then
    begin Result := Fail(Format('รหัสผ่านต้องมีอย่างน้อย %d ตัวอักษร', [MinLength]), 'password'); Exit; end;
  
  HasUpper   := False;
  HasLower   := False;
  HasDigit   := False;
  HasSpecial := False;
  
  for i := 1 to Length(Password) do
  begin
    if Password[i] in ['A'..'Z'] then HasUpper   := True;
    if Password[i] in ['a'..'z'] then HasLower   := True;
    if Password[i] in ['0'..'9'] then HasDigit   := True;
    if Password[i] in ['!','@','#','$','%','^','&','*','(',')'] then HasSpecial := True;
  end;
  
  if not HasUpper   then begin Result := Fail('ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว', 'password'); Exit; end;
  if not HasLower   then begin Result := Fail('ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว', 'password'); Exit; end;
  if not HasDigit   then begin Result := Fail('ต้องมีตัวเลขอย่างน้อย 1 ตัว', 'password'); Exit; end;
  if not HasSpecial then begin Result := Fail('ต้องมีอักขระพิเศษอย่างน้อย 1 ตัว', 'password'); Exit; end;
  
  Result := OK('password');
end;

function ValidateAge(Age: Integer; MinAge: Integer = 0;
                     MaxAge: Integer = 150): TValidationResult;
begin
  if Age < MinAge then
    begin Result := Fail(Format('อายุต้องไม่น้อยกว่า %d ปี', [MinAge]), 'age'); Exit; end;
  if Age > MaxAge then
    begin Result := Fail(Format('อายุต้องไม่เกิน %d ปี', [MaxAge]), 'age'); Exit; end;
  Result := OK('age');
end;

function ValidateRange(Value, Min, Max: Real;
                       FieldName: String = 'ค่า'): TValidationResult;
begin
  if Value < Min then
    begin Result := Fail(Format('%s ต้องไม่น้อยกว่า %.2f', [FieldName, Min]), FieldName); Exit; end;
  if Value > Max then
    begin Result := Fail(Format('%s ต้องไม่เกิน %.2f', [FieldName, Max]), FieldName); Exit; end;
  Result := OK(FieldName);
end;

end.
```

### ตัวอย่างที่ 11: โปรแกรมทดสอบ Validation

```pascal
program TestValidation;

{$mode objfpc}{$H+}

uses SysUtils, Validation;

procedure Check(VR: TValidationResult; ExpectValid: Boolean);
begin
  if VR.IsValid = ExpectValid then
    Write('  PASS ')
  else
    Write('  FAIL ');
  
  if VR.IsValid then
    WriteLn('[Valid]')
  else
    WriteLn('[Invalid] ', VR.Message, ' (field: ', VR.Field, ')');
end;

begin
  WriteLn('=== ทดสอบ Validation Unit ===');
  WriteLn;
  
  WriteLn('--- Email ---');
  Check(ValidateEmail('user@example.com'), True);
  Check(ValidateEmail('invalid'), False);
  Check(ValidateEmail('a@b'), False);
  Check(ValidateEmail('test@domain.co.th'), True);
  
  WriteLn;
  WriteLn('--- Phone ---');
  Check(ValidatePhone('0812345678'), True);
  Check(ValidatePhone('081-234-5678'), True);
  Check(ValidatePhone('12345678'), False);
  Check(ValidatePhone('123456789012'), False);
  
  WriteLn;
  WriteLn('--- Thai ID ---');
  Check(ValidateThaiID('1234567890121'), True);  // อาจผิด checksum
  Check(ValidateThaiID('123'), False);
  
  WriteLn;
  WriteLn('--- Password ---');
  Check(ValidatePassword('Abc123!@#'), True);
  Check(ValidatePassword('abc'), False);
  Check(ValidatePassword('abcdefgh'), False);  // ขาด uppercase, digit, special
  Check(ValidatePassword('Abcdefg!'), False);  // ขาด digit
  
  WriteLn;
  WriteLn('--- Age ---');
  Check(ValidateAge(25, 18, 65), True);
  Check(ValidateAge(15, 18, 65), False);
  Check(ValidateAge(70, 18, 65), False);
  
  WriteLn;
  WriteLn('--- Range ---');
  Check(ValidateRange(50, 0, 100, 'คะแนน'), True);
  Check(ValidateRange(-5, 0, 100, 'คะแนน'), False);
  Check(ValidateRange(150, 0, 100, 'คะแนน'), False);
  
  ReadLn;
end.
```

---

## 13.10 Unit Namespace

### ตัวอย่างที่ 12: การจัดการ Namespace

```pascal
program NamespaceDemo;

{$mode objfpc}{$H+}

uses
  SysUtils,
  Math,        // มี Max, Min ที่ต่างจาก System
  Classes;

// เมื่อมีชื่อซ้ำกัน ใช้ unit prefix
var
  A, B : Integer;
  X, Y : Real;

begin
  A := 5; B := 10;
  X := 3.14; Y := 2.71;
  
  // Math.Max vs System.Max
  WriteLn('Math.Max(3.14, 2.71) = ', Math.Max(X, Y):0:4);
  WriteLn('System.Max(5, 10) = ', System.Max(A, B));
  
  // SysUtils functions
  WriteLn('SysUtils.Format: ', SysUtils.Format('%d + %d = %d', [A, B, A+B]));
  
  ReadLn;
end.
```

---

## 13.11 การจัดการ uses clause

### ตัวอย่างที่ 13: uses clause ที่ดี

```pascal
program GoodUsesClause;

{$mode objfpc}{$H+}

// ลำดับ uses ที่แนะนำ:
// 1. Standard RTL units ก่อน
// 2. Third-party units
// 3. Project units ของเราเอง
uses
  // Standard
  SysUtils,
  Classes,
  Math,
  StrUtils,
  // Third-party (ถ้ามี)
  // Indy10Components,
  // Project units
  MathUtils,
  StringUtils,
  Validation;

var
  Email : String;
  VR    : TValidationResult;

begin
  // ใช้ functions จากหลาย units
  Email := 'user@example.com';
  
  VR := ValidateEmail(Email);
  if VR.IsValid then
    WriteLn('Email valid: ', Email)
  else
    WriteLn('Email invalid: ', VR.Message);
  
  // Math
  WriteLn('PI = ', PI:0:10);
  WriteLn('Is 17 prime? ', IsPrime(17));
  WriteLn('Factorial(10) = ', Factorial(10));
  
  // String
  WriteLn('ToTitleCase: ', ToTitleCase('hello world from pascal'));
  WriteLn('Reverse: ', Reverse_Str('Pascal'));
  
  ReadLn;
end.
```

---

## 13.12 แบบฝึกหัด

### แบบฝึกหัดที่ 1
สร้าง unit ชื่อ `ConvertUtils` ที่มีฟังก์ชัน:
- แปลงหน่วยอุณหภูมิ (Celsius, Fahrenheit, Kelvin)
- แปลงหน่วยระยะทาง (km, m, cm, mile, foot)
- แปลงหน่วยน้ำหนัก (kg, g, lb, oz)

### แบบฝึกหัดที่ 2
สร้าง unit `SortUtils` ที่มีอัลกอริทึม sort:
- Bubble Sort
- Selection Sort
- Insertion Sort
- Quick Sort
- Merge Sort
สำหรับ array of Integer และ array of String

### แบบฝึกหัดที่ 3
สร้าง unit `ArrayUtils` ที่มีฟังก์ชัน:
- Reverse array
- Shuffle (สุ่ม)
- Unique (ลบซ้ำ)
- Contains (ตรวจสอบ)
- IndexOf (ค้นหา)
- Sum, Average, Min, Max

### แบบฝึกหัดที่ 4
สร้าง unit `HashUtils` ที่มี:
- Simple hash function (djb2)
- CRC32
- ตรวจสอบ integrity ของ string

### แบบฝึกหัดที่ 5
สร้าง unit `ColorUtils` ที่มี:
- แปลง RGB <-> HEX
- แปลง RGB <-> HSL
- Mix สองสี
- Lighten/Darken สี

### แบบฝึกหัดที่ 6
สร้าง unit `TextTable` ที่ช่วยวาดตารางข้อมูลใน console
รองรับ alignment และ custom borders

### แบบฝึกหัดที่ 7
สร้าง unit `InputUtils` ที่มีฟังก์ชัน:
- `ReadInteger` พร้อม validation
- `ReadReal` พร้อม range check
- `ReadString` พร้อม length validation
- `ReadYesNo` รับ y/n
- `ReadChoice` รับหมายเลขจาก menu

### แบบฝึกหัดที่ 8
สร้าง unit `ThaiUtils` ที่มี:
- แปลงตัวเลขเป็นคำอ่านภาษาไทย
- แปลงวันที่เป็น Thai format
- ตรวจสอบเลขบัตรประชาชน
- format เงิน บาท สตางค์

### แบบฝึกหัดที่ 9
สร้าง unit `CryptoUtils` อย่างง่ายที่มี:
- ROT13 cipher
- XOR cipher
- Caesar cipher
- Base64 encode/decode

### แบบฝึกหัดที่ 10
สร้าง unit `LogUtils` ที่มีระบบ logging:
- Log levels (Debug, Info, Warning, Error)
- Log to file และ console
- Log rotation
- Timestamp และ caller info
ใช้ `initialization` สร้าง log file และ `finalization` ปิดไฟล์

---

## สรุป

| ส่วนของ Unit | หน้าที่ |
|-------------|--------|
| `interface` | ประกาศ public API |
| `implementation` | โค้ด implementation (private) |
| `initialization` | รันครั้งเดียวตอนโหลด unit |
| `finalization` | รันครั้งเดียวตอนปิดโปรแกรม |

**Best Practices สำหรับ Units:**

1. Unit หนึ่งควรมีหน้าที่เดียว (Single Responsibility)
2. ตั้งชื่อ unit ให้สื่อความหมาย
3. ประกาศเฉพาะสิ่งที่จำเป็นใน interface
4. ซ่อน implementation details ไว้ใน implementation section
5. ใช้ `initialization`/`finalization` สำหรับ setup/teardown
6. จัดการ circular dependencies ด้วย shared type unit
7. เรียง uses clause ตามลำดับที่เหมาะสม
