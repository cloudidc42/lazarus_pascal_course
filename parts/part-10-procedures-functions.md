# Part 10 - Procedures และ Functions

## สารบัญ
1. [ความแตกต่างระหว่าง Procedure และ Function](#ความแตกต่าง)
2. [การประกาศและเรียกใช้ Procedures](#procedures)
3. [การประกาศและเรียกใช้ Functions](#functions)
4. [Parameters: value, var, const, out](#parameters)
5. [Default Parameter Values](#default-parameters)
6. [Overloading](#overloading)
7. [Recursive Functions](#recursive-functions)
8. [Function Pointers และ Procedural Types](#function-pointers)
9. [Anonymous Functions (Closures)](#anonymous-functions)
10. [Inline Functions](#inline-functions)
11. [Forward Declarations](#forward-declarations)
12. [โปรแกรมตัวอย่าง 5 โปรแกรม](#โปรแกรมตัวอย่าง)
13. [แบบฝึกหัด 20 ข้อ](#แบบฝึกหัด)

---

## บทนำ

Procedures และ Functions คือหน่วยย่อยของโค้ดที่ออกแบบมาเพื่อทำงานเฉพาะอย่าง ช่วยให้โค้ดมีโครงสร้าง อ่านง่าย และนำกลับมาใช้ซ้ำได้

### ประโยชน์ของ Procedures/Functions
- **Reusability**: เขียนครั้งเดียว ใช้ได้หลายครั้ง
- **Modularity**: แบ่งโปรแกรมเป็นส่วนย่อยๆ
- **Readability**: โค้ดอ่านง่ายขึ้น
- **Maintainability**: แก้ไขง่าย
- **Testing**: ทดสอบแต่ละส่วนแยกกันได้

---

## ความแตกต่าง

| คุณสมบัติ | Procedure | Function |
|-----------|-----------|---------|
| ส่งค่ากลับ | ไม่มี | มี (ผ่าน Result หรือชื่อ function) |
| การเรียกใช้ | เป็น statement | เป็น expression |
| ใช้ใน expression | ไม่ได้ | ได้ |
| คำสำคัญ | `procedure` | `function` |

```pascal
program DifferenceDemo;

// Procedure: ไม่ส่งค่ากลับ
procedure ShowMessage(const Msg: String);
begin
  WriteLn('ข้อความ: ', Msg);
end;

// Function: ส่งค่ากลับ
function AddTwo(A, B: Integer): Integer;
begin
  Result := A + B;  // ส่งค่ากลับผ่าน Result
end;

begin
  ShowMessage('Hello!');           // เรียก procedure
  var Sum := AddTwo(3, 5);        // เรียก function
  WriteLn('3 + 5 = ', Sum);
  WriteLn('7 + 2 = ', AddTwo(7, 2));  // ใช้ใน expression
end.
```

---

## Procedures

### ไวยากรณ์

```pascal
procedure <ชื่อ>([parameters]);
begin
  // คำสั่ง
end;
```

### ตัวอย่างที่ 1: Procedure ไม่มี Parameter

```pascal
program ProcNoParam;

procedure PrintBanner;
begin
  WriteLn('╔════════════════════╗');
  WriteLn('║   Hello Pascal!    ║');
  WriteLn('╚════════════════════╝');
end;

procedure PrintLine;
begin
  WriteLn(StringOfChar('─', 40));
end;

procedure PrintNewLine;
begin
  WriteLn;
end;

begin
  PrintBanner;
  PrintNewLine;
  PrintLine;
  WriteLn('เนื้อหา');
  PrintLine;
  PrintNewLine;
  PrintBanner;
end.
```

### ตัวอย่างที่ 2: Procedure มี Parameter

```pascal
program ProcWithParam;

procedure PrintBox(const Title: String; Width: Integer);
var
  I: Integer;
begin
  // บนสุด
  Write('┌');
  for I := 1 to Width - 2 do Write('─');
  WriteLn('┐');
  
  // หัวเรื่อง
  var Padding := (Width - 2 - Length(Title)) div 2;
  Write('│');
  for I := 1 to Padding do Write(' ');
  Write(Title);
  for I := 1 to Width - 2 - Length(Title) - Padding do Write(' ');
  WriteLn('│');
  
  // ล่างสุด
  Write('└');
  for I := 1 to Width - 2 do Write('─');
  WriteLn('┘');
end;

procedure PrintHeader(const Title: String);
begin
  WriteLn;
  WriteLn(StringOfChar('=', 50));
  WriteLn('  ', Title);
  WriteLn(StringOfChar('=', 50));
end;

procedure Greet(const Name: String; Times: Integer);
var
  I: Integer;
begin
  for I := 1 to Times do
    WriteLn('สวัสดี, ', Name, '! (', I, '/', Times, ')');
end;

begin
  PrintBox('สวัสดีโลก', 30);
  PrintHeader('โปรแกรมทดสอบ');
  Greet('Alice', 3);
  PrintBox('Pascal', 20);
end.
```

### ตัวอย่างที่ 3: Procedure ที่เรียกกันเอง

```pascal
program ProcCallProc;

procedure DrawLine(Width: Integer; Char_: Char);
var I: Integer;
begin
  for I := 1 to Width do Write(Char_);
  WriteLn;
end;

procedure DrawRect(Width, Height: Integer);
var I: Integer;
begin
  DrawLine(Width, '─');  // เรียก procedure อื่น
  for I := 1 to Height - 2 do
  begin
    Write('│');
    for var J := 1 to Width - 2 do Write(' ');
    WriteLn('│');
  end;
  DrawLine(Width, '─');
end;

procedure PrintReport(const Title: String; Data: array of Integer);
var I: Integer;
begin
  DrawLine(40, '=');
  WriteLn(Title);
  DrawLine(40, '-');
  for I := Low(Data) to High(Data) do
    WriteLn(Format('  รายการ %-3d: %8d', [I + 1, Data[I]]));
  DrawLine(40, '=');
end;

var
  Sales: array[0..4] of Integer;
begin
  Sales[0] := 1500;
  Sales[1] := 2300;
  Sales[2] := 1800;
  Sales[3] := 3200;
  Sales[4] := 2700;
  
  PrintReport('รายงานยอดขาย', Sales);
end.
```

---

## Functions

### ไวยากรณ์

```pascal
function <ชื่อ>([parameters]): <ชนิดข้อมูลที่ส่งกลับ>;
begin
  // คำสั่ง
  Result := <ค่า>;  // กำหนดค่าที่ส่งกลับ
end;
```

### ตัวอย่างที่ 4: Functions พื้นฐาน

```pascal
program BasicFunctions;

function Square(N: Integer): Integer;
begin
  Result := N * N;
end;

function Cube(N: Integer): Integer;
begin
  Result := N * N * N;
end;

function Average(A, B, C: Real): Real;
begin
  Result := (A + B + C) / 3;
end;

function IsEven(N: Integer): Boolean;
begin
  Result := N mod 2 = 0;
end;

function Max2(A, B: Integer): Integer;
begin
  if A > B then Result := A
  else Result := B;
end;

function Min2(A, B: Integer): Integer;
begin
  if A < B then Result := A
  else Result := B;
end;

function Abs2(N: Integer): Integer;
begin
  if N < 0 then Result := -N
  else Result := N;
end;

begin
  WriteLn('Square(5) = ', Square(5));
  WriteLn('Cube(3) = ', Cube(3));
  WriteLn('Average(10, 20, 30) = ', Average(10, 20, 30):0:2);
  WriteLn('IsEven(4) = ', IsEven(4));
  WriteLn('IsEven(7) = ', IsEven(7));
  WriteLn('Max2(15, 8) = ', Max2(15, 8));
  WriteLn('Min2(15, 8) = ', Min2(15, 8));
  WriteLn('Abs2(-42) = ', Abs2(-42));
end.
```

### ตัวอย่างที่ 5: Functions ส่งกลับ String

```pascal
program StringFunctions;
uses SysUtils;

function Repeat_Str(const S: String; Times: Integer): String;
var
  I: Integer;
begin
  Result := '';
  for I := 1 to Times do
    Result := Result + S;
end;

function Capitalize(const S: String): String;
begin
  if Length(S) = 0 then
    Result := ''
  else
    Result := UpperCase(S[1]) + LowerCase(Copy(S, 2, Length(S)));
end;

function PadCenter(const S: String; Width: Integer): String;
var
  TotalPad, LeftPad, RightPad: Integer;
begin
  if Length(S) >= Width then
    Result := S
  else
  begin
    TotalPad := Width - Length(S);
    LeftPad := TotalPad div 2;
    RightPad := TotalPad - LeftPad;
    Result := StringOfChar(' ', LeftPad) + S + StringOfChar(' ', RightPad);
  end;
end;

function FormatMoney(Amount: Real): String;
begin
  Result := Format('฿%,.2f', [Amount]);
end;

begin
  WriteLn(Repeat_Str('-', 30));
  WriteLn(Repeat_Str('AB', 5));
  WriteLn(Capitalize('hello world'));
  WriteLn(PadCenter('Hello', 20));
  WriteLn('|', PadCenter('Pascal', 20), '|');
  WriteLn(FormatMoney(1234567.89));
end.
```

### ตัวอย่างที่ 6: Functions ส่งกลับ Array/Record

```pascal
program FunctionReturnComplex;
uses SysUtils;

type
  TPoint = record
    X, Y: Real;
  end;
  
  TStatistics = record
    Min, Max, Mean, Median: Real;
    Count: Integer;
  end;

function CreatePoint(X, Y: Real): TPoint;
begin
  Result.X := X;
  Result.Y := Y;
end;

function Distance(P1, P2: TPoint): Real;
begin
  Result := Sqrt(Sqr(P2.X - P1.X) + Sqr(P2.Y - P1.Y));
end;

function MidPoint(P1, P2: TPoint): TPoint;
begin
  Result.X := (P1.X + P2.X) / 2;
  Result.Y := (P1.Y + P2.Y) / 2;
end;

function CalcStats(const Data: array of Real): TStatistics;
var
  I: Integer;
  Sum: Real;
  Sorted: array of Real;
begin
  Result.Count := Length(Data);
  if Result.Count = 0 then Exit;
  
  Result.Min := Data[0];
  Result.Max := Data[0];
  Sum := 0;
  
  for I := 0 to High(Data) do
  begin
    if Data[I] < Result.Min then Result.Min := Data[I];
    if Data[I] > Result.Max then Result.Max := Data[I];
    Sum := Sum + Data[I];
  end;
  
  Result.Mean := Sum / Result.Count;
  
  // Sort สำหรับ Median
  SetLength(Sorted, Result.Count);
  for I := 0 to High(Data) do Sorted[I] := Data[I];
  var J: Integer;
  for I := 0 to High(Sorted) - 1 do
    for J := 0 to High(Sorted) - I - 1 do
      if Sorted[J] > Sorted[J+1] then
      begin
        var Temp := Sorted[J]; Sorted[J] := Sorted[J+1]; Sorted[J+1] := Temp;
      end;
  
  if Result.Count mod 2 = 1 then
    Result.Median := Sorted[Result.Count div 2]
  else
    Result.Median := (Sorted[Result.Count div 2 - 1] + Sorted[Result.Count div 2]) / 2;
end;

var
  P1, P2, Mid: TPoint;
  Data: array[0..5] of Real;
  Stats: TStatistics;
begin
  P1 := CreatePoint(0, 0);
  P2 := CreatePoint(3, 4);
  Mid := MidPoint(P1, P2);
  
  WriteLn('P1 = (', P1.X:0:1, ', ', P1.Y:0:1, ')');
  WriteLn('P2 = (', P2.X:0:1, ', ', P2.Y:0:1, ')');
  WriteLn('Distance = ', Distance(P1, P2):0:2);
  WriteLn('MidPoint = (', Mid.X:0:1, ', ', Mid.Y:0:1, ')');
  
  Data[0] := 15; Data[1] := 8; Data[2] := 22;
  Data[3] := 4;  Data[4] := 16; Data[5] := 11;
  
  Stats := CalcStats(Data);
  WriteLn;
  WriteLn('Statistics:');
  WriteLn('Count = ', Stats.Count);
  WriteLn('Min = ', Stats.Min:0:2);
  WriteLn('Max = ', Stats.Max:0:2);
  WriteLn('Mean = ', Stats.Mean:0:2);
  WriteLn('Median = ', Stats.Median:0:2);
end.
```

---

## Parameters

### Value Parameters (ส่งค่าสำเนา)

```pascal
program ValueParam;

procedure DoubleValue(N: Integer);  // Value parameter
begin
  N := N * 2;  // แก้ไขสำเนา ไม่กระทบตัวแปรต้นทาง
  WriteLn('ใน procedure: N = ', N);
end;

begin
  var X := 10;
  WriteLn('ก่อน: X = ', X);
  DoubleValue(X);
  WriteLn('หลัง: X = ', X);  // X ยังเป็น 10
end.
```

### var Parameters (ส่ง reference)

```pascal
program VarParam;

procedure DoubleVar(var N: Integer);  // var parameter
begin
  N := N * 2;  // แก้ไขตัวแปรต้นทางโดยตรง
  WriteLn('ใน procedure: N = ', N);
end;

procedure Swap(var A, B: Integer);
var Temp: Integer;
begin
  Temp := A;
  A := B;
  B := Temp;
end;

procedure GetMinMax(const Data: array of Integer; var MinVal, MaxVal: Integer);
var I: Integer;
begin
  MinVal := Data[0];
  MaxVal := Data[0];
  for I := 1 to High(Data) do
  begin
    if Data[I] < MinVal then MinVal := Data[I];
    if Data[I] > MaxVal then MaxVal := Data[I];
  end;
end;

var
  A, B: Integer;
  MinV, MaxV: Integer;
begin
  A := 10;
  DoubleVar(A);
  WriteLn('หลัง: A = ', A);  // A = 20
  
  A := 5; B := 3;
  WriteLn('ก่อน: A = ', A, ', B = ', B);
  Swap(A, B);
  WriteLn('หลัง: A = ', A, ', B = ', B);
  
  var Data := [10, 5, 25, 3, 18];
  GetMinMax(Data, MinV, MaxV);
  WriteLn('Min = ', MinV, ', Max = ', MaxV);
end.
```

### const Parameters (อ่านอย่างเดียว, มีประสิทธิภาพ)

```pascal
program ConstParam;

// const ป้องกันการแก้ไข, ส่งเป็น reference (ประหยัดหน่วยความจำ)
function SumArray(const Arr: array of Integer): Integer;
var I: Integer;
begin
  Result := 0;
  for I := Low(Arr) to High(Arr) do
    Result := Result + Arr[I];
  // Arr[0] := 0;  // Error! ไม่สามารถแก้ไขได้
end;

procedure PrintStudent(const Name: String; const Score: Real);
begin
  // Name := 'X';  // Error! const ไม่สามารถแก้ไขได้
  WriteLn(Format('%-20s %.2f', [Name, Score]));
end;

begin
  var Scores := [85, 92, 78, 95, 88];
  WriteLn('Sum = ', SumArray(Scores));
  PrintStudent('Alice', 95.5);
  PrintStudent('Bob', 87.3);
end.
```

### out Parameters (ส่งออกอย่างเดียว)

```pascal
program OutParam;
uses SysUtils;

// out parameter: ส่งข้อมูลออก ไม่ส่งเข้า
procedure ParseName(const FullName: String; out FirstName, LastName: String);
var SpacePos: Integer;
begin
  SpacePos := Pos(' ', FullName);
  if SpacePos > 0 then
  begin
    FirstName := Copy(FullName, 1, SpacePos - 1);
    LastName := Copy(FullName, SpacePos + 1, Length(FullName));
  end
  else
  begin
    FirstName := FullName;
    LastName := '';
  end;
end;

procedure Divide(Dividend, Divisor: Integer; out Quotient, Remainder: Integer);
begin
  Quotient := Dividend div Divisor;
  Remainder := Dividend mod Divisor;
end;

var
  First, Last: String;
  Q, R: Integer;
begin
  ParseName('John Smith', First, Last);
  WriteLn('ชื่อ: ', First, ', นามสกุล: ', Last);
  
  Divide(17, 5, Q, R);
  WriteLn('17 ÷ 5 = ', Q, ' เหลือ ', R);
end.
```

### ตัวอย่างที่ 7: การผสมผสาน Parameter Types

```pascal
program MixedParams;

procedure ProcessData(
  const Input: array of Real;  // const: อ่านอย่างเดียว
  Scale: Real;                  // value: สำเนา
  var Sum: Real;               // var: ส่งออก
  out Count: Integer           // out: ส่งออกเท่านั้น
);
var I: Integer;
begin
  Sum := 0;
  Count := Length(Input);
  for I := 0 to High(Input) do
    Sum := Sum + Input[I] * Scale;
end;

var
  Data: array[0..4] of Real;
  Total: Real;
  N: Integer;
begin
  Data[0] := 10; Data[1] := 20; Data[2] := 30;
  Data[3] := 40; Data[4] := 50;
  
  ProcessData(Data, 2.5, Total, N);
  WriteLn('ผลรวม (x2.5) = ', Total:0:2);
  WriteLn('จำนวน = ', N);
end.
```

---

## Default Parameters

### ตัวอย่างที่ 8: Parameter มีค่า Default

```pascal
program DefaultParams;
uses SysUtils;

procedure PrintDivider(Width: Integer = 40; Char_: Char = '=');
var I: Integer;
begin
  for I := 1 to Width do Write(Char_);
  WriteLn;
end;

function FormatNumber(
  Value: Real;
  Decimals: Integer = 2;
  UseComma: Boolean = True
): String;
begin
  if UseComma then
    Result := Format('%,.'+IntToStr(Decimals)+'f', [Value])
  else
    Result := Format('%.'+IntToStr(Decimals)+'f', [Value]);
end;

procedure Greet(
  const Name: String;
  const Greeting: String = 'สวัสดี';
  Times: Integer = 1
);
var I: Integer;
begin
  for I := 1 to Times do
    WriteLn(Greeting, ', ', Name, '!');
end;

begin
  PrintDivider;           // Width=40, Char='='
  PrintDivider(20);       // Width=20, Char='='
  PrintDivider(30, '-');  // Width=30, Char='-'
  
  WriteLn(FormatNumber(1234567.89));           // 2 decimals, comma
  WriteLn(FormatNumber(1234567.89, 0));        // 0 decimals, comma
  WriteLn(FormatNumber(3.14159, 4, False));    // 4 decimals, no comma
  
  Greet('Alice');                   // ใช้ค่า default ทั้งหมด
  Greet('Bob', 'ดีครับ');          // Greeting ต่างกัน
  Greet('Charlie', 'Hello', 3);    // ทุก parameter กำหนดเอง
end.
```

---

## Overloading

ฟังก์ชันชื่อเดียวกัน แต่ parameter ต่างกัน

### ตัวอย่างที่ 9: Function Overloading

```pascal
program FunctionOverloading;

function Add(A, B: Integer): Integer; overload;
begin
  Result := A + B;
end;

function Add(A, B: Real): Real; overload;
begin
  Result := A + B;
end;

function Add(A, B, C: Integer): Integer; overload;
begin
  Result := A + B + C;
end;

function Add(const S1, S2: String): String; overload;
begin
  Result := S1 + S2;
end;

function Max(A, B: Integer): Integer; overload;
begin
  if A > B then Result := A else Result := B;
end;

function Max(A, B: Real): Real; overload;
begin
  if A > B then Result := A else Result := B;
end;

function Max(const S1, S2: String): String; overload;
begin
  if S1 > S2 then Result := S1 else Result := S2;
end;

begin
  WriteLn(Add(3, 5));              // Integer: 8
  WriteLn(Add(3.14, 2.71):0:2);   // Real: 5.85
  WriteLn(Add(1, 2, 3));          // 3 params: 6
  WriteLn(Add('Hello', 'World')); // String: HelloWorld
  
  WriteLn(Max(10, 20));           // Integer
  WriteLn(Max(3.14, 2.71):0:2);  // Real
  WriteLn(Max('Apple', 'Banana')); // String
end.
```

---

## Recursive Functions

ฟังก์ชันที่เรียกตัวเองซ้ำๆ

### ตัวอย่างที่ 10: Factorial

```pascal
program FactorialRecursive;

function Factorial(N: Integer): Int64;
begin
  if N <= 1 then
    Result := 1
  else
    Result := N * Factorial(N - 1);
end;

// แบบ tail recursive
function FactTail(N: Integer; Acc: Int64 = 1): Int64;
begin
  if N <= 1 then
    Result := Acc
  else
    Result := FactTail(N - 1, N * Acc);
end;

begin
  var I: Integer;
  for I := 0 to 15 do
    WriteLn(I, '! = ', Factorial(I));
end.
```

### ตัวอย่างที่ 11: Fibonacci Recursive

```pascal
program FibonacciRecursive;

// แบบ naive recursive (ช้ามาก)
function FibNaive(N: Integer): Int64;
begin
  if N <= 1 then
    Result := N
  else
    Result := FibNaive(N - 1) + FibNaive(N - 2);
end;

// แบบ memoization (เร็วขึ้นมาก)
var Memo: array[0..50] of Int64;

function FibMemo(N: Integer): Int64;
begin
  if N <= 1 then
    Result := N
  else if Memo[N] > 0 then
    Result := Memo[N]
  else
  begin
    Memo[N] := FibMemo(N - 1) + FibMemo(N - 2);
    Result := Memo[N];
  end;
end;

// แบบ iterative (เร็วที่สุด)
function FibIter(N: Integer): Int64;
var
  A, B, C: Int64;
  I: Integer;
begin
  if N <= 1 then begin Result := N; Exit; end;
  A := 0; B := 1;
  for I := 2 to N do
  begin
    C := A + B;
    A := B;
    B := C;
  end;
  Result := B;
end;

begin
  var I: Integer;
  FillChar(Memo, SizeOf(Memo), 0);
  
  WriteLn('N':5, 'Naive':15, 'Memo':15, 'Iter':15);
  WriteLn(StringOfChar('-', 50));
  for I := 0 to 20 do
    WriteLn(I:5, FibNaive(I):15, FibMemo(I):15, FibIter(I):15);
end.
```

### ตัวอย่างที่ 12: Tower of Hanoi

```pascal
program TowerOfHanoi;

var
  MoveCount: Integer;

procedure Hanoi(N: Integer; FromPeg, ToPeg, AuxPeg: Char);
begin
  if N = 1 then
  begin
    Inc(MoveCount);
    WriteLn(Format('เดิน %3d: แผ่น 1 จาก %s ไป %s', [MoveCount, FromPeg, ToPeg]));
  end
  else
  begin
    Hanoi(N - 1, FromPeg, AuxPeg, ToPeg);  // ย้าย N-1 จาน
    Inc(MoveCount);
    WriteLn(Format('เดิน %3d: แผ่น %d จาก %s ไป %s', [MoveCount, N, FromPeg, ToPeg]));
    Hanoi(N - 1, AuxPeg, ToPeg, FromPeg);  // ย้าย N-1 จานกลับ
  end;
end;

var
  N: Integer;
begin
  Write('จำนวนแผ่น (1-10): ');
  ReadLn(N);
  
  MoveCount := 0;
  WriteLn('=== Tower of Hanoi ===');
  WriteLn('ย้าย ', N, ' แผ่น จากหลัก A ไปหลัก C ผ่านหลัก B');
  WriteLn;
  
  Hanoi(N, 'A', 'C', 'B');
  
  WriteLn;
  WriteLn('จำนวนการเคลื่อนย้าย: ', MoveCount);
  WriteLn('สูตร: 2^N - 1 = ', Round(Power(2, N)) - 1);
end.
```

### ตัวอย่างที่ 13: Binary Search Recursive

```pascal
program BinarySearchRecursive;

var
  Data: array[1..20] of Integer;

function BinarySearch(Target, Lo, Hi: Integer): Integer;
var
  Mid: Integer;
begin
  if Lo > Hi then
  begin
    Result := -1;
    Exit;
  end;
  
  Mid := (Lo + Hi) div 2;
  WriteLn('  ตรวจสอบ index ', Mid, ' (', Data[Mid], ')');
  
  if Data[Mid] = Target then
    Result := Mid
  else if Data[Mid] < Target then
    Result := BinarySearch(Target, Mid + 1, Hi)
  else
    Result := BinarySearch(Target, Lo, Mid - 1);
end;

var
  I, Target, Pos_: Integer;
begin
  for I := 1 to 20 do
    Data[I] := I * 3;
  
  Write('ข้อมูล: ');
  for I := 1 to 20 do Write(Data[I], ' ');
  WriteLn;
  
  Write('หาตัวเลข: ');
  ReadLn(Target);
  
  WriteLn('ขั้นตอนการค้นหา:');
  Pos_ := BinarySearch(Target, 1, 20);
  
  if Pos_ > 0 then
    WriteLn('พบ ', Target, ' ที่ index ', Pos_)
  else
    WriteLn('ไม่พบ ', Target);
end.
```

### ตัวอย่างที่ 14: GCD (Greatest Common Divisor) Recursive

```pascal
program GCDRecursive;

function GCD(A, B: Integer): Integer;
begin
  if B = 0 then
    Result := A
  else
    Result := GCD(B, A mod B);
end;

function LCM(A, B: Integer): Integer;
begin
  Result := (A * B) div GCD(A, B);
end;

// ทำงานกับ array
function GCDArray(const Arr: array of Integer): Integer;
var I: Integer;
begin
  Result := Arr[0];
  for I := 1 to High(Arr) do
    Result := GCD(Result, Arr[I]);
end;

begin
  WriteLn('GCD(48, 18) = ', GCD(48, 18));
  WriteLn('GCD(100, 75) = ', GCD(100, 75));
  WriteLn('LCM(4, 6) = ', LCM(4, 6));
  WriteLn('GCD Array [12, 18, 24] = ', GCDArray([12, 18, 24]));
end.
```

---

## Function Pointers และ Procedural Types

### ตัวอย่างที่ 15: Function Pointer

```pascal
program FunctionPointers;

type
  // ประกาศ procedural type
  TMathFunc = function(X: Real): Real;
  TCompareFunc = function(A, B: Integer): Boolean;
  TActionProc = procedure(const Msg: String);

function Double_(X: Real): Real;
begin Result := X * 2; end;

function Triple(X: Real): Real;
begin Result := X * 3; end;

function Square(X: Real): Real;
begin Result := X * X; end;

function IsGreater(A, B: Integer): Boolean;
begin Result := A > B; end;

function IsLess(A, B: Integer): Boolean;
begin Result := A < B; end;

procedure InfoAction(const Msg: String);
begin WriteLn('[INFO] ', Msg); end;

procedure WarnAction(const Msg: String);
begin WriteLn('[WARN] ', Msg); end;

procedure ApplyToArray(var Arr: array of Real; Func: TMathFunc);
var I: Integer;
begin
  for I := Low(Arr) to High(Arr) do
    Arr[I] := Func(Arr[I]);
end;

procedure BubbleSortWith(var Arr: array of Integer; Compare: TCompareFunc);
var I, J, Temp: Integer;
begin
  for I := Low(Arr) to High(Arr) - 1 do
    for J := Low(Arr) to High(Arr) - I - 1 do
      if Compare(Arr[J], Arr[J+1]) then
      begin
        Temp := Arr[J]; Arr[J] := Arr[J+1]; Arr[J+1] := Temp;
      end;
end;

var
  Func: TMathFunc;
  Values: array[0..4] of Real;
  Numbers: array[0..5] of Integer;
  I: Integer;
  Logger: TActionProc;
begin
  Values[0] := 1; Values[1] := 2; Values[2] := 3;
  Values[3] := 4; Values[4] := 5;
  
  // ใช้ function pointer
  Func := @Double_;
  ApplyToArray(Values, Func);
  Write('Double: ');
  for I := 0 to 4 do Write(Values[I]:0:1, ' ');
  WriteLn;
  
  Func := @Square;
  ApplyToArray(Values, Func);
  Write('Square: ');
  for I := 0 to 4 do Write(Values[I]:0:1, ' ');
  WriteLn;
  
  // Sort แบบต่างกัน
  Numbers[0] := 5; Numbers[1] := 2; Numbers[2] := 8;
  Numbers[3] := 1; Numbers[4] := 9; Numbers[5] := 3;
  
  BubbleSortWith(Numbers, @IsGreater);  // เรียงน้อยไปมาก
  Write('น้อย->มาก: ');
  for I := 0 to 5 do Write(Numbers[I], ' ');
  WriteLn;
  
  BubbleSortWith(Numbers, @IsLess);     // เรียงมากไปน้อย
  Write('มาก->น้อย: ');
  for I := 0 to 5 do Write(Numbers[I], ' ');
  WriteLn;
  
  // Logger pattern
  Logger := @InfoAction;
  Logger('ระบบเริ่มต้น');
  Logger := @WarnAction;
  Logger('พบข้อผิดพลาด!');
end.
```

---

## Anonymous Functions (Closures)

### ตัวอย่างที่ 16: Anonymous Function ใน Lazarus/FPC

```pascal
program AnonymousFunctions;
uses
  SysUtils;

type
  TIntFunc = function(X: Integer): Integer;
  TPredicate = function(X: Integer): Boolean;

// Higher-order function ที่รับ anonymous function
function ApplyN(X: Integer; Func: TIntFunc; Times: Integer): Integer;
var I: Integer;
begin
  Result := X;
  for I := 1 to Times do
    Result := Func(Result);
end;

function Filter(const Arr: array of Integer; Pred: TPredicate): TArray<Integer>;
var I: Integer;
begin
  SetLength(Result, 0);
  for I := Low(Arr) to High(Arr) do
    if Pred(Arr[I]) then
    begin
      SetLength(Result, Length(Result) + 1);
      Result[High(Result)] := Arr[I];
    end;
end;

// Closure-like pattern ด้วย object
type
  TAdder = class
  private
    FAddend: Integer;
  public
    constructor Create(Addend: Integer);
    function Add(X: Integer): Integer;
  end;

constructor TAdder.Create(Addend: Integer);
begin
  FAddend := Addend;
end;

function TAdder.Add(X: Integer): Integer;
begin
  Result := X + FAddend;
end;

// ใน FPC 3.2+ รองรับ anonymous procedures
// procedure(X: Integer) begin ... end;
// แต่ syntax อาจต่างกัน ขึ้นกับ version

var
  Add5: TAdder;
  Add10: TAdder;
begin
  Add5 := TAdder.Create(5);
  Add10 := TAdder.Create(10);
  
  try
    WriteLn('Add5(3) = ', Add5.Add(3));   // 8
    WriteLn('Add10(3) = ', Add10.Add(3)); // 13
    WriteLn('Add5(Add10(2)) = ', Add5.Add(Add10.Add(2))); // 17
  finally
    Add5.Free;
    Add10.Free;
  end;
end.
```

---

## Inline Functions

### ตัวอย่างที่ 17: Inline Functions

```pascal
program InlineFunctions;

// inline: คอมไพเลอร์จะแทรกโค้ดลงตรงๆ แทนการ call
// เร็วขึ้น แต่ binary ใหญ่ขึ้น
// เหมาะกับฟังก์ชันเล็กๆ ที่เรียกบ่อยมาก

function IsPositive(N: Integer): Boolean; inline;
begin
  Result := N > 0;
end;

function IsNegative(N: Integer): Boolean; inline;
begin
  Result := N < 0;
end;

function ClampValue(Value, MinVal, MaxVal: Integer): Integer; inline;
begin
  if Value < MinVal then Result := MinVal
  else if Value > MaxVal then Result := MaxVal
  else Result := Value;
end;

function Interpolate(A, B: Real; T: Real): Real; inline;
begin
  Result := A + (B - A) * T;
end;

// Math operations ที่ใช้บ่อย
function Sqr2(X: Integer): Integer; inline;
begin
  Result := X * X;
end;

function RoundUp(Dividend, Divisor: Integer): Integer; inline;
begin
  Result := (Dividend + Divisor - 1) div Divisor;
end;

begin
  WriteLn('IsPositive(5) = ', IsPositive(5));
  WriteLn('IsNegative(-3) = ', IsNegative(-3));
  WriteLn('Clamp(15, 0, 10) = ', ClampValue(15, 0, 10));
  WriteLn('Clamp(-5, 0, 10) = ', ClampValue(-5, 0, 10));
  WriteLn('Clamp(7, 0, 10) = ', ClampValue(7, 0, 10));
  WriteLn('Interpolate(0, 100, 0.5) = ', Interpolate(0, 100, 0.5):0:1);
  WriteLn('RoundUp(10, 3) = ', RoundUp(10, 3));
  WriteLn('RoundUp(9, 3) = ', RoundUp(9, 3));
end.
```

---

## Forward Declarations

### ตัวอย่างที่ 18: Forward Declaration

```pascal
program ForwardDecl;

// Forward declaration: ประกาศก่อน นิยามทีหลัง
// ใช้เมื่อฟังก์ชัน A เรียก B และ B เรียก A (mutual recursion)

procedure ProcB(N: Integer); forward;  // ประกาศล่วงหน้า

procedure ProcA(N: Integer);
begin
  if N <= 0 then Exit;
  WriteLn('A: ', N);
  ProcB(N - 1);  // เรียก B ได้ เพราะ forward declared
end;

procedure ProcB(N: Integer);  // นิยามจริง
begin
  if N <= 0 then Exit;
  WriteLn('B: ', N);
  ProcA(N - 1);
end;

// ตัวอย่าง: IsEven/IsOdd แบบ mutual recursion
function IsOdd(N: Integer): Boolean; forward;

function IsEven2(N: Integer): Boolean;
begin
  if N = 0 then Result := True
  else Result := IsOdd(N - 1);
end;

function IsOdd(N: Integer): Boolean;
begin
  if N = 0 then Result := False
  else Result := IsEven2(N - 1);
end;

begin
  WriteLn('Mutual recursion:');
  ProcA(5);
  
  WriteLn;
  WriteLn('IsEven/IsOdd:');
  var I: Integer;
  for I := 0 to 6 do
    WriteLn(I, ': IsEven=', IsEven2(I), ', IsOdd=', IsOdd(I));
end.
```

---

## โปรแกรมตัวอย่าง 1: Unit Converter

```pascal
program UnitConverter;
uses SysUtils, Math;

// ฟังก์ชันแปลงหน่วย
function CmToInch(Cm: Real): Real; inline;
begin Result := Cm / 2.54; end;

function InchToCm(Inch: Real): Real; inline;
begin Result := Inch * 2.54; end;

function KgToLb(Kg: Real): Real; inline;
begin Result := Kg * 2.20462; end;

function LbToKg(Lb: Real): Real; inline;
begin Result := Lb / 2.20462; end;

function CelsiusToFahrenheit(C: Real): Real; inline;
begin Result := C * 9/5 + 32; end;

function FahrenheitToCelsius(F: Real): Real; inline;
begin Result := (F - 32) * 5/9; end;

function KmToMile(Km: Real): Real; inline;
begin Result := Km * 0.621371; end;

function MileToKm(Mile: Real): Real; inline;
begin Result := Mile / 0.621371; end;

function LiterToGallon(L: Real): Real; inline;
begin Result := L * 0.264172; end;

function GallonToLiter(G: Real): Real; inline;
begin Result := G / 0.264172; end;

procedure ShowConversionMenu;
begin
  WriteLn;
  WriteLn('=== Unit Converter ===');
  WriteLn('1. ความยาว (cm <-> inch, km <-> mile)');
  WriteLn('2. น้ำหนัก (kg <-> lb)');
  WriteLn('3. อุณหภูมิ (C <-> F)');
  WriteLn('4. ปริมาตร (liter <-> gallon)');
  WriteLn('0. ออก');
end;

var
  MainChoice, SubChoice: Integer;
  Value: Real;
begin
  repeat
    ShowConversionMenu;
    Write('เลือก: ');
    ReadLn(MainChoice);
    
    case MainChoice of
      1: begin
           WriteLn('1. cm -> inch');
           WriteLn('2. inch -> cm');
           WriteLn('3. km -> mile');
           WriteLn('4. mile -> km');
           Write('เลือก: ');
           ReadLn(SubChoice);
           Write('ป้อนค่า: ');
           ReadLn(Value);
           
           case SubChoice of
             1: WriteLn(Value:0:4, ' cm = ', CmToInch(Value):0:4, ' inch');
             2: WriteLn(Value:0:4, ' inch = ', InchToCm(Value):0:4, ' cm');
             3: WriteLn(Value:0:4, ' km = ', KmToMile(Value):0:4, ' miles');
             4: WriteLn(Value:0:4, ' miles = ', MileToKm(Value):0:4, ' km');
           end;
         end;
      2: begin
           WriteLn('1. kg -> lb');
           WriteLn('2. lb -> kg');
           Write('เลือก: ');
           ReadLn(SubChoice);
           Write('ป้อนค่า: ');
           ReadLn(Value);
           case SubChoice of
             1: WriteLn(Value:0:4, ' kg = ', KgToLb(Value):0:4, ' lb');
             2: WriteLn(Value:0:4, ' lb = ', LbToKg(Value):0:4, ' kg');
           end;
         end;
      3: begin
           WriteLn('1. Celsius -> Fahrenheit');
           WriteLn('2. Fahrenheit -> Celsius');
           Write('เลือก: ');
           ReadLn(SubChoice);
           Write('ป้อนค่า: ');
           ReadLn(Value);
           case SubChoice of
             1: WriteLn(Value:0:2, '°C = ', CelsiusToFahrenheit(Value):0:2, '°F');
             2: WriteLn(Value:0:2, '°F = ', FahrenheitToCelsius(Value):0:2, '°C');
           end;
         end;
      4: begin
           WriteLn('1. Liter -> Gallon');
           WriteLn('2. Gallon -> Liter');
           Write('เลือก: ');
           ReadLn(SubChoice);
           Write('ป้อนค่า: ');
           ReadLn(Value);
           case SubChoice of
             1: WriteLn(Value:0:4, ' L = ', LiterToGallon(Value):0:4, ' gal');
             2: WriteLn(Value:0:4, ' gal = ', GallonToLiter(Value):0:4, ' L');
           end;
         end;
    end;
  until MainChoice = 0;
end.
```

---

## โปรแกรมตัวอย่าง 2: Library Management

```pascal
program LibrarySystem;
uses SysUtils;

const
  MAX_BOOKS = 100;

type
  TBook = record
    ID: Integer;
    Title: String[100];
    Author: String[50];
    ISBN: String[20];
    Year: Integer;
    Available: Boolean;
  end;

var
  Books: array[1..MAX_BOOKS] of TBook;
  BookCount: Integer;

// ฟังก์ชันสร้าง Book ID
function GenerateID: Integer;
begin
  Result := BookCount + 1;
end;

// เพิ่มหนังสือ
function AddBook(const Title, Author, ISBN: String; Year: Integer): Boolean;
begin
  Result := False;
  if BookCount >= MAX_BOOKS then Exit;
  
  Inc(BookCount);
  Books[BookCount].ID := GenerateID;
  Books[BookCount].Title := Title;
  Books[BookCount].Author := Author;
  Books[BookCount].ISBN := ISBN;
  Books[BookCount].Year := Year;
  Books[BookCount].Available := True;
  Result := True;
end;

// ค้นหาด้วย ID
function FindBookByID(ID: Integer): Integer;
var I: Integer;
begin
  Result := -1;
  for I := 1 to BookCount do
    if Books[I].ID = ID then begin Result := I; Exit; end;
end;

// ค้นหาด้วยชื่อ (ส่วนหนึ่ง)
function SearchByTitle(const Title: String): Integer;
var I: Integer;
begin
  Result := -1;
  for I := 1 to BookCount do
    if Pos(LowerCase(Title), LowerCase(Books[I].Title)) > 0 then
      begin Result := I; Exit; end;
end;

// ยืมหนังสือ
function BorrowBook(ID: Integer): Boolean;
var Idx: Integer;
begin
  Result := False;
  Idx := FindBookByID(ID);
  if Idx = -1 then Exit;
  if not Books[Idx].Available then Exit;
  
  Books[Idx].Available := False;
  Result := True;
end;

// คืนหนังสือ
function ReturnBook(ID: Integer): Boolean;
var Idx: Integer;
begin
  Result := False;
  Idx := FindBookByID(ID);
  if Idx = -1 then Exit;
  if Books[Idx].Available then Exit;
  
  Books[Idx].Available := True;
  Result := True;
end;

// แสดงหนังสือทั้งหมด
procedure PrintAllBooks;
var I: Integer;
begin
  if BookCount = 0 then begin WriteLn('ไม่มีหนังสือ'); Exit; end;
  
  WriteLn(Format('%-5s %-35s %-25s %-5s %s', ['ID', 'ชื่อ', 'ผู้แต่ง', 'ปี', 'สถานะ']));
  WriteLn(StringOfChar('-', 80));
  
  for I := 1 to BookCount do
  begin
    var Status := 'ว่าง';
    if not Books[I].Available then Status := 'ถูกยืม';
    
    WriteLn(Format('%-5d %-35s %-25s %-5d %s',
      [Books[I].ID, Books[I].Title, Books[I].Author, Books[I].Year, Status]));
  end;
end;

// สถิติ
procedure PrintStats;
var I, Available, Borrowed: Integer;
begin
  Available := 0; Borrowed := 0;
  for I := 1 to BookCount do
    if Books[I].Available then Inc(Available) else Inc(Borrowed);
  
  WriteLn('สถิติห้องสมุด:');
  WriteLn('  หนังสือทั้งหมด: ', BookCount);
  WriteLn('  ว่าง: ', Available);
  WriteLn('  ถูกยืม: ', Borrowed);
end;

var Choice: Integer;
begin
  BookCount := 0;
  
  // เพิ่มข้อมูลตัวอย่าง
  AddBook('Pascal Programming', 'Niklaus Wirth', '978-0201504088', 1984);
  AddBook('Object Pascal Handbook', 'Marco Cantu', '978-1785280559', 2020);
  AddBook('Lazarus for Beginners', 'John Smith', '978-0000000001', 2022);
  AddBook('Free Pascal Guide', 'FPC Team', '978-0000000002', 2023);
  
  repeat
    WriteLn;
    WriteLn('=== ระบบห้องสมุด ===');
    WriteLn('1. แสดงหนังสือทั้งหมด');
    WriteLn('2. ค้นหาหนังสือ');
    WriteLn('3. ยืมหนังสือ');
    WriteLn('4. คืนหนังสือ');
    WriteLn('5. สถิติ');
    WriteLn('0. ออก');
    Write('เลือก: ');
    ReadLn(Choice);
    
    case Choice of
      1: PrintAllBooks;
      2: begin
           Write('ค้นหาชื่อ: ');
           var Search: String;
           ReadLn(Search);
           var Idx := SearchByTitle(Search);
           if Idx > 0 then
           begin
             WriteLn('พบ: ', Books[Idx].Title);
             WriteLn('ผู้แต่ง: ', Books[Idx].Author);
             WriteLn('สถานะ: ', BoolToStr(Books[Idx].Available, 'ว่าง', 'ถูกยืม'));
           end
           else WriteLn('ไม่พบ');
         end;
      3: begin
           Write('ID หนังสือ: ');
           var ID: Integer;
           ReadLn(ID);
           if BorrowBook(ID) then
             WriteLn('ยืมสำเร็จ')
           else
             WriteLn('ไม่สามารถยืมได้');
         end;
      4: begin
           Write('ID หนังสือ: ');
           var ID: Integer;
           ReadLn(ID);
           if ReturnBook(ID) then
             WriteLn('คืนสำเร็จ')
           else
             WriteLn('ไม่สามารถคืนได้');
         end;
      5: PrintStats;
    end;
  until Choice = 0;
end.
```

---

## โปรแกรมตัวอย่าง 3: Simple Calculator with History

```pascal
program CalcWithHistory;
uses SysUtils, Math;

const
  MAX_HISTORY = 50;

type
  TOperation = record
    Expression: String;
    Result: Real;
  end;

var
  History: array[1..MAX_HISTORY] of TOperation;
  HistoryCount: Integer;

procedure AddToHistory(const Expr: String; Res: Real);
begin
  if HistoryCount < MAX_HISTORY then
  begin
    Inc(HistoryCount);
    History[HistoryCount].Expression := Expr;
    History[HistoryCount].Result := Res;
  end;
end;

function Calculate(A, B: Real; Op: Char): Real;
begin
  case Op of
    '+': Result := A + B;
    '-': Result := A - B;
    '*': Result := A * B;
    '/': if B <> 0 then Result := A / B else raise Exception.Create('หารด้วยศูนย์');
    '^': Result := Power(A, B);
  else
    raise Exception.Create('ตัวดำเนินการไม่ถูกต้อง');
  end;
end;

procedure ShowHistory;
var I: Integer;
begin
  if HistoryCount = 0 then begin WriteLn('ยังไม่มีประวัติ'); Exit; end;
  
  WriteLn('=== ประวัติการคำนวณ ===');
  for I := 1 to HistoryCount do
    WriteLn(Format('%3d. %-30s = %g', [I, History[I].Expression, History[I].Result]));
end;

procedure ClearHistory;
begin
  HistoryCount := 0;
  WriteLn('ลบประวัติแล้ว');
end;

function GetLastResult: Real;
begin
  if HistoryCount > 0 then
    Result := History[HistoryCount].Result
  else
    Result := 0;
end;

var
  A, B, Res: Real;
  Op: Char;
  Input: String;
  UseLastResult: Boolean;
begin
  HistoryCount := 0;
  
  WriteLn('=== เครื่องคิดเลขพร้อมประวัติ ===');
  WriteLn('พิมพ์ "h" เพื่อดูประวัติ');
  WriteLn('พิมพ์ "c" เพื่อลบประวัติ');
  WriteLn('พิมพ์ "q" เพื่อออก');
  WriteLn('พิมพ์ "l" เพื่อใช้ผลลัพธ์ล่าสุด');
  WriteLn;
  
  repeat
    Write('ตัวเลขที่ 1 (หรือ l=ล่าสุด): ');
    ReadLn(Input);
    
    if LowerCase(Input) = 'q' then Break;
    if LowerCase(Input) = 'h' then begin ShowHistory; Continue; end;
    if LowerCase(Input) = 'c' then begin ClearHistory; Continue; end;
    
    UseLastResult := LowerCase(Input) = 'l';
    if UseLastResult then
      A := GetLastResult
    else
      A := StrToFloatDef(Input, 0);
    
    Write('ตัวดำเนินการ (+,-,*,/,^): ');
    ReadLn(Op);
    
    Write('ตัวเลขที่ 2: ');
    ReadLn(Input);
    B := StrToFloatDef(Input, 0);
    
    try
      Res := Calculate(A, B, Op);
      var Expr := Format('%g %s %g', [A, Op, B]);
      WriteLn(Expr, ' = ', Res:0:4);
      AddToHistory(Expr, Res);
    except
      on E: Exception do
        WriteLn('ข้อผิดพลาด: ', E.Message);
    end;
    
    WriteLn;
  until False;
end.
```

---

## โปรแกรมตัวอย่าง 4: Number Theory Functions

```pascal
program NumberTheory;
uses SysUtils, Math;

function IsPrime(N: Integer): Boolean;
var I: Integer;
begin
  if N < 2 then begin Result := False; Exit; end;
  if N = 2 then begin Result := True; Exit; end;
  if N mod 2 = 0 then begin Result := False; Exit; end;
  
  Result := True;
  I := 3;
  while I * I <= N do
  begin
    if N mod I = 0 then begin Result := False; Exit; end;
    Inc(I, 2);
  end;
end;

function PrimeFactors(N: Integer): String;
var
  Factor: Integer;
  First: Boolean;
begin
  Result := '';
  First := True;
  Factor := 2;
  
  while Factor * Factor <= N do
  begin
    while N mod Factor = 0 do
    begin
      if not First then Result := Result + ' × ';
      Result := Result + IntToStr(Factor);
      First := False;
      N := N div Factor;
    end;
    Inc(Factor);
  end;
  
  if N > 1 then
  begin
    if not First then Result := Result + ' × ';
    Result := Result + IntToStr(N);
  end;
end;

function EulerPhi(N: Integer): Integer;
var I: Integer;
begin
  Result := N;
  I := 2;
  var Temp := N;
  while I * I <= Temp do
  begin
    if Temp mod I = 0 then
    begin
      while Temp mod I = 0 do Temp := Temp div I;
      Result := Result - Result div I;
    end;
    Inc(I);
  end;
  if Temp > 1 then
    Result := Result - Result div Temp;
end;

function SumOfDivisors(N: Integer): Integer;
var I: Integer;
begin
  Result := 0;
  for I := 1 to N do
    if N mod I = 0 then Result := Result + I;
end;

function IsPerfect(N: Integer): Boolean;
begin
  Result := SumOfDivisors(N) = 2 * N;
end;

procedure ShowPrimesUpTo(N: Integer);
var I, Count: Integer;
begin
  Count := 0;
  for I := 2 to N do
    if IsPrime(I) then
    begin
      Write(I:6);
      Inc(Count);
      if Count mod 10 = 0 then WriteLn;
    end;
  WriteLn;
  WriteLn('จำนวนเฉพาะทั้งหมด ', Count, ' จำนวน');
end;

begin
  WriteLn('=== Number Theory ===');
  WriteLn;
  
  // ทดสอบ isPrime
  var I: Integer;
  Write('จำนวนเฉพาะ 2-50: ');
  for I := 2 to 50 do
    if IsPrime(I) then Write(I, ' ');
  WriteLn;
  
  // ทดสอบ PrimeFactors
  WriteLn;
  WriteLn('การแยกตัวประกอบเฉพาะ:');
  for I := 2 to 20 do
    WriteLn(Format('%4d = %s', [I, PrimeFactors(I)]));
  
  // ทดสอบ Perfect Numbers
  WriteLn;
  Write('Perfect Numbers ถึง 10000: ');
  for I := 1 to 10000 do
    if IsPerfect(I) then Write(I, ' ');
  WriteLn;
  
  // Euler Phi
  WriteLn;
  WriteLn('Euler Phi:');
  for I := 1 to 12 do
    WriteLn(Format('φ(%2d) = %d', [I, EulerPhi(I)]));
end.
```

---

## โปรแกรมตัวอย่าง 5: Sorting Algorithms Comparison

```pascal
program SortComparison;
uses SysUtils;

const
  TEST_SIZE = 1000;

type
  TIntArray = array of Integer;
  TSortProc = procedure(var Arr: TIntArray);

procedure GenerateRandom(var Arr: TIntArray; Size: Integer);
var I: Integer;
begin
  SetLength(Arr, Size);
  Randomize;
  for I := 0 to Size - 1 do
    Arr[I] := Random(10000);
end;

procedure CopyArray(const Src: TIntArray; var Dst: TIntArray);
var I: Integer;
begin
  SetLength(Dst, Length(Src));
  for I := 0 to High(Src) do
    Dst[I] := Src[I];
end;

function IsSorted(const Arr: TIntArray): Boolean;
var I: Integer;
begin
  Result := True;
  for I := 0 to High(Arr) - 1 do
    if Arr[I] > Arr[I+1] then begin Result := False; Exit; end;
end;

// Bubble Sort
procedure BubbleSort(var Arr: TIntArray);
var I, J, Temp: Integer; Swapped: Boolean;
begin
  for I := 0 to High(Arr) - 1 do
  begin
    Swapped := False;
    for J := 0 to High(Arr) - I - 1 do
      if Arr[J] > Arr[J+1] then
      begin
        Temp := Arr[J]; Arr[J] := Arr[J+1]; Arr[J+1] := Temp;
        Swapped := True;
      end;
    if not Swapped then Break;
  end;
end;

// Selection Sort
procedure SelectionSort(var Arr: TIntArray);
var I, J, MinIdx, Temp: Integer;
begin
  for I := 0 to High(Arr) - 1 do
  begin
    MinIdx := I;
    for J := I + 1 to High(Arr) do
      if Arr[J] < Arr[MinIdx] then MinIdx := J;
    if MinIdx <> I then
    begin
      Temp := Arr[I]; Arr[I] := Arr[MinIdx]; Arr[MinIdx] := Temp;
    end;
  end;
end;

// Insertion Sort
procedure InsertionSort(var Arr: TIntArray);
var I, J, Key: Integer;
begin
  for I := 1 to High(Arr) do
  begin
    Key := Arr[I];
    J := I - 1;
    while (J >= 0) and (Arr[J] > Key) do
    begin
      Arr[J + 1] := Arr[J];
      Dec(J);
    end;
    Arr[J + 1] := Key;
  end;
end;

// Quick Sort
procedure QuickSort(var Arr: TIntArray; Lo, Hi: Integer);
var I, J, Pivot, Temp: Integer;
begin
  if Lo < Hi then
  begin
    Pivot := Arr[(Lo + Hi) div 2];
    I := Lo; J := Hi;
    while I <= J do
    begin
      while Arr[I] < Pivot do Inc(I);
      while Arr[J] > Pivot do Dec(J);
      if I <= J then
      begin
        Temp := Arr[I]; Arr[I] := Arr[J]; Arr[J] := Temp;
        Inc(I); Dec(J);
      end;
    end;
    QuickSort(Arr, Lo, J);
    QuickSort(Arr, I, Hi);
  end;
end;

procedure DoQuickSort(var Arr: TIntArray);
begin
  if Length(Arr) > 0 then
    QuickSort(Arr, 0, High(Arr));
end;

// Measure time and sort
procedure Benchmark(const Name: String; SortFunc: TSortProc; const Original: TIntArray);
var
  Data: TIntArray;
  StartTime, EndTime: QWord;
begin
  CopyArray(Original, Data);
  StartTime := GetTickCount64;
  SortFunc(Data);
  EndTime := GetTickCount64;
  
  var OK := 'ผ่าน';
  if not IsSorted(Data) then OK := 'ไม่ผ่าน!';
  
  WriteLn(Format('%-20s %6d ms  %s', [Name, EndTime - StartTime, OK]));
end;

var
  Original: TIntArray;
begin
  WriteLn('=== Sorting Algorithm Benchmark ===');
  WriteLn(Format('ขนาดข้อมูล: %d รายการ', [TEST_SIZE]));
  WriteLn;
  WriteLn(Format('%-20s %10s  %s', ['Algorithm', 'เวลา', 'ผลลัพธ์']));
  WriteLn(StringOfChar('-', 45));
  
  GenerateRandom(Original, TEST_SIZE);
  
  Benchmark('Bubble Sort', @BubbleSort, Original);
  Benchmark('Selection Sort', @SelectionSort, Original);
  Benchmark('Insertion Sort', @InsertionSort, Original);
  Benchmark('Quick Sort', @DoQuickSort, Original);
  
  WriteLn;
  WriteLn('หมายเหตุ: Quick Sort เร็วที่สุดสำหรับข้อมูลขนาดใหญ่');
  WriteLn('O(n²) vs O(n log n)');
end.
```

---

## แบบฝึกหัด

### ข้อที่ 1: Temperature Table Generator

```pascal
program TempTable;

function CtoF(C: Real): Real;
begin Result := C * 9/5 + 32; end;

function CtoK(C: Real): Real;
begin Result := C + 273.15; end;

procedure PrintTempTable(StartC, EndC, Step: Real);
var C: Real;
begin
  WriteLn(Format('%-10s %-10s %-10s', ['°C', '°F', 'K']));
  WriteLn(StringOfChar('-', 30));
  C := StartC;
  while C <= EndC do
  begin
    WriteLn(Format('%10.1f %10.1f %10.2f', [C, CtoF(C), CtoK(C)]));
    C := C + Step;
  end;
end;

begin
  PrintTempTable(-40, 100, 10);
end.
```

### ข้อที่ 2-5

```pascal
// ข้อ 2: ฟังก์ชันตรวจสอบตัวเลขนำโชค
function IsLucky(N: Integer): Boolean;
var Sum: Integer;
begin
  Sum := 0;
  while N > 0 do begin Sum := Sum + (N mod 10); N := N div 10; end;
  while Sum > 9 do begin
    var S := Sum; Sum := 0;
    while S > 0 do begin Sum := Sum + (S mod 10); S := S div 10; end;
  end;
  Result := Sum in [1, 3, 7, 9];
end;

// ข้อ 3: ฟังก์ชัน Power ด้วยวิธี fast exponentiation
function FastPow(Base: Int64; Exp: Integer): Int64;
begin
  if Exp = 0 then begin Result := 1; Exit; end;
  if Exp mod 2 = 0 then begin
    var Half := FastPow(Base, Exp div 2);
    Result := Half * Half;
  end else
    Result := Base * FastPow(Base, Exp - 1);
end;

// ข้อ 4: ฟังก์ชันแปลงตัวเลขเป็นคำ (ไทย)
function NumToThaiWord(N: Integer): String;
const
  ONES: array[0..9] of String = ('ศูนย์','หนึ่ง','สอง','สาม','สี่','ห้า','หก','เจ็ด','แปด','เก้า');
  PLACES: array[0..4] of String = ('','สิบ','ร้อย','พัน','หมื่น');
var I, Digit: Integer;
begin
  if N = 0 then begin Result := ONES[0]; Exit; end;
  Result := '';
  I := 0;
  while N > 0 do begin
    Digit := N mod 10;
    if Digit <> 0 then
      Result := ONES[Digit] + PLACES[I] + Result;
    N := N div 10;
    Inc(I);
  end;
end;

// ข้อ 5: ฟังก์ชัน Permutation และ Combination
function Factorial2(N: Integer): Int64;
begin
  if N <= 1 then Result := 1
  else Result := N * Factorial2(N - 1);
end;

function Permutation(N, R: Integer): Int64;
begin
  Result := Factorial2(N) div Factorial2(N - R);
end;

function Combination(N, R: Integer): Int64;
begin
  Result := Factorial2(N) div (Factorial2(R) * Factorial2(N - R));
end;

begin
  var I: Integer;
  
  WriteLn('Lucky numbers 1-50:');
  for I := 1 to 50 do
    if IsLucky(I) then Write(I, ' ');
  WriteLn;
  
  WriteLn;
  WriteLn('Fast Power:');
  WriteLn('2^10 = ', FastPow(2, 10));
  WriteLn('3^5 = ', FastPow(3, 5));
  
  WriteLn;
  WriteLn('ตัวเลขเป็นคำไทย:');
  for I := 0 to 20 do
    WriteLn(I, ' = ', NumToThaiWord(I));
  
  WriteLn;
  WriteLn('P(5,2) = ', Permutation(5, 2));
  WriteLn('C(5,2) = ', Combination(5, 2));
  WriteLn('C(10,3) = ', Combination(10, 3));
end.
```

### ข้อที่ 6-10

```pascal
// ข้อ 6: Matrix transpose ด้วย function
program MatrixTranspose;
type TMatrix = array[1..5, 1..5] of Integer;

procedure Transpose(const M: TMatrix; var T: TMatrix; N: Integer);
var I, J: Integer;
begin
  for I := 1 to N do
    for J := 1 to N do
      T[J, I] := M[I, J];
end;

// ข้อ 7: ฟังก์ชัน Merge สอง array ที่เรียงแล้ว
function MergeSorted(const A, B: array of Integer): TArray<Integer>;
var I, J, K: Integer;
begin
  SetLength(Result, Length(A) + Length(B));
  I := 0; J := 0; K := 0;
  while (I <= High(A)) and (J <= High(B)) do begin
    if A[I] <= B[J] then begin Result[K] := A[I]; Inc(I); end
    else begin Result[K] := B[J]; Inc(J); end;
    Inc(K);
  end;
  while I <= High(A) do begin Result[K] := A[I]; Inc(I); Inc(K); end;
  while J <= High(B) do begin Result[K] := B[J]; Inc(J); Inc(K); end;
end;

// ข้อ 8: ฟังก์ชัน Flatten nested (simulate)
procedure FlattenSimulate;
var Arr1, Arr2, Arr3, Flat: array[0..4] of Integer;
    I: Integer;
begin
  for I := 0 to 4 do begin Arr1[I] := I; Arr2[I] := I+5; Arr3[I] := I+10; end;
  for I := 0 to 4 do Flat[I] := Arr1[I];
  Write('Arr1: '); for I := 0 to 4 do Write(Arr1[I], ' '); WriteLn;
end;

// ข้อ 9: String formatter ที่ยืดหยุ่น
function BuildRow(const Fields: array of String; 
                  const Widths: array of Integer): String;
var I: Integer;
begin
  Result := '│';
  for I := Low(Fields) to High(Fields) do
  begin
    var Field := Fields[I];
    var W := Widths[I];
    if Length(Field) > W then
      Field := Copy(Field, 1, W - 2) + '..';
    Result := Result + ' ' + Format('%-*s', [W, Field]) + ' │';
  end;
end;

// ข้อ 10: ฟังก์ชันวิเคราะห์ข้อมูล
procedure AnalyzeArray(const Data: array of Real);
var I: Integer; Sum, Variance, Mean: Real;
begin
  Sum := 0;
  for I := Low(Data) to High(Data) do Sum := Sum + Data[I];
  Mean := Sum / Length(Data);
  
  Variance := 0;
  for I := Low(Data) to High(Data) do
    Variance := Variance + Sqr(Data[I] - Mean);
  Variance := Variance / Length(Data);
  
  WriteLn('Count:', Length(Data):5);
  WriteLn('Sum:  ', Sum:8:2);
  WriteLn('Mean: ', Mean:8:2);
  WriteLn('Std:  ', Sqrt(Variance):8:2);
end;

begin
  AnalyzeArray([1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0]);
end.
```

### ข้อที่ 11-20 (สั้น)

```pascal
program FinalExercises;
uses SysUtils, Math;

// ข้อ 11-20: ฟังก์ชันต่างๆ
function AbsoluteValue(X: Real): Real; begin if X < 0 then Result := -X else Result := X; end;
function Ceil2(X: Real): Integer; begin Result := Ceil(X); end;
function Floor2(X: Real): Integer; begin Result := Floor(X); end;
function Hypotenuse(A, B: Real): Real; begin Result := Sqrt(A*A + B*B); end;
function PercentOf(Part, Total: Real): Real; begin if Total <> 0 then Result := Part/Total*100 else Result := 0; end;
function IsLeapYear(Y: Integer): Boolean; begin Result := ((Y mod 4 = 0) and (Y mod 100 <> 0)) or (Y mod 400 = 0); end;
function Clamp(V, Lo, Hi: Real): Real; begin if V < Lo then Result := Lo else if V > Hi then Result := Hi else Result := V; end;
function Lerp(A, B, T: Real): Real; begin Result := A + (B-A)*T; end;
function Sign_(X: Real): Integer; begin if X > 0 then Result := 1 else if X < 0 then Result := -1 else Result := 0; end;
function RoundToNearest(X, Step: Real): Real; begin Result := Round(X/Step)*Step; end;

var I: Integer;
begin
  WriteLn('ฟังก์ชันต่างๆ:');
  WriteLn('Abs(-3.7) = ', AbsoluteValue(-3.7):0:1);
  WriteLn('Ceil(3.2) = ', Ceil2(3.2));
  WriteLn('Floor(3.8) = ', Floor2(3.8));
  WriteLn('Hypotenuse(3,4) = ', Hypotenuse(3,4):0:1);
  WriteLn('PercentOf(25,200) = ', PercentOf(25,200):0:1, '%');
  WriteLn('IsLeapYear(2024) = ', IsLeapYear(2024));
  WriteLn('Clamp(15,0,10) = ', Clamp(15,0,10):0:0);
  WriteLn('Lerp(0,100,0.3) = ', Lerp(0,100,0.3):0:1);
  WriteLn('Sign(-5) = ', Sign_(-5));
  WriteLn('RoundToNearest(7.3,0.5) = ', RoundToNearest(7.3,0.5):0:1);
end.
```

---

## สรุป

### การเลือกใช้ Parameter Type

| สถานการณ์ | Parameter Type |
|-----------|----------------|
| ส่งข้อมูลเข้าไป ไม่แก้ไข (ชนิดเล็ก) | value |
| ส่งข้อมูลเข้าไป ไม่แก้ไข (ชนิดใหญ่) | `const` |
| ต้องการแก้ไขตัวแปรต้นทาง | `var` |
| ต้องการส่งข้อมูลออกเท่านั้น | `out` |
| array ขนาดใดก็ได้ | `open array` |

### เมื่อใช้ Recursion

- เมื่อปัญหาสามารถแบ่งเป็นปัญหาย่อยที่คล้ายกัน
- ต้องมี **base case** เสมอ (เงื่อนไขหยุด)
- ระวัง **stack overflow** ถ้า recursion ลึกเกินไป
- พิจารณาใช้ **memoization** เพื่อความเร็ว
- บางครั้ง **iterative** ดีกว่า recursive

---

*จบ Part 10 - Procedures และ Functions*
