# Part 04 - ตัวแปรและชนิดข้อมูล

## สารบัญ

1. [การประกาศตัวแปร (var declaration)](#การประกาศตัวแปร-var-declaration)
2. [Integer Types](#integer-types)
3. [Float Types](#float-types)
4. [Boolean Type](#boolean-type)
5. [Character Type (Char)](#character-type-char)
6. [String Types](#string-types)
7. [Subrange Types](#subrange-types)
8. [Type Casting](#type-casting)
9. [Constants (const)](#constants-const)
10. [Typed Constants](#typed-constants)
11. [โปรแกรมตัวอย่าง 5 โปรแกรม](#โปรแกรมตัวอย่าง-5-โปรแกรม)
12. [แบบฝึกหัด 20 ข้อ](#แบบฝึกหัด-20-ข้อ)

---

## การประกาศตัวแปร (var declaration)

### รูปแบบพื้นฐาน

```pascal
program VarDemo;
var
  // รูปแบบ: ชื่อตัวแปร : ชนิดข้อมูล;
  age: Integer;
  name: String;
  height: Double;
  isActive: Boolean;
  initial: Char;
begin
  // กำหนดค่า
  age := 25;
  name := 'สมชาย';
  height := 175.5;
  isActive := True;
  initial := 'S';
  
  // แสดงผล
  WriteLn('ชื่อ: ', name);
  WriteLn('อายุ: ', age);
  WriteLn('ส่วนสูง: ', height:5:1);
end.
```

### การประกาศหลายตัวแปรพร้อมกัน

```pascal
program MultiVar;
var
  // ประกาศหลายตัวพร้อมกันด้วย comma
  x, y, z: Integer;          // 3 ตัวแปร Integer
  a, b: Double;              // 2 ตัวแปร Double
  firstName, lastName: String; // 2 ตัวแปร String
  
  // ประกาศและกำหนดค่าพร้อมกัน (ใน FPC mode)
  counter: Integer = 0;      // กำหนดค่า default
  pi: Double = 3.14159;
  greeting: String = 'Hello';
begin
  x := 10; y := 20; z := 30;
  WriteLn(x, ' ', y, ' ', z);
  
  a := 1.5; b := 2.5;
  WriteLn(a + b:5:2);
end.
```

### กฎการตั้งชื่อตัวแปร (Identifier Rules)

```pascal
// ✓ ชื่อที่ถูกต้อง
var
  age: Integer;           // ตัวอักษรล้วน
  myAge: Integer;         // camelCase
  my_age: Integer;        // snake_case
  _count: Integer;        // ขึ้นต้นด้วย underscore
  numberOfStudents: Integer; // ยาวก็ได้
  x1, y2: Double;         // มีตัวเลขต่อท้าย

// ✗ ชื่อที่ไม่ถูกต้อง
// var
//   2count: Integer;      // ขึ้นต้นด้วยตัวเลข
//   my-age: Integer;      // มี - (hyphen)
//   my age: Integer;      // มีช่องว่าง
//   begin: Integer;       // ใช้ reserved word
//   integer: String;      // ใช้ type name (เตือน แต่ไม่ error)
```

### Scope ของตัวแปร

```pascal
program Scope;

// Global variables - มองเห็นได้ทุกที่
var
  globalCounter: Integer = 0;

procedure IncrementCounter;
// Local variable - มองเห็นแค่ใน procedure นี้
var
  localStep: Integer;
begin
  localStep := 1;
  Inc(globalCounter, localStep);
  WriteLn('Inside procedure, globalCounter = ', globalCounter);
  // WriteLn(localStep2); ← ERROR: ไม่เห็น local var จาก procedure อื่น
end;

procedure AnotherProcedure;
var
  localStep: Integer;  // ← ตัวแปรคนละตัวกับ IncrementCounter.localStep
begin
  localStep := 100;    // ไม่กระทบกัน
  WriteLn('localStep = ', localStep);
end;

begin
  globalCounter := 0;
  IncrementCounter;
  IncrementCounter;
  AnotherProcedure;
  WriteLn('Final globalCounter = ', globalCounter);  // 2
end.
```

### Stack vs Heap Memory

```pascal
program MemoryDemo;
var
  // Stack allocation (อัตโนมัติ)
  stackVar: Integer;        // อยู่ใน stack
  stackArr: array[1..100] of Double;  // อยู่ใน stack

  // Heap allocation (ด้วยตนเอง)
  heapPtr: PInteger;        // pointer ชี้ไปที่ heap
  
begin
  // Stack - ใช้งานง่าย จัดการอัตโนมัติ
  stackVar := 42;
  stackArr[1] := 3.14;
  
  // Heap - ต้องจัดการเอง แต่ยืดหยุ่นกว่า
  New(heapPtr);             // จอง memory ใน heap
  heapPtr^ := 100;          // กำหนดค่าผ่าน pointer
  WriteLn('Heap value: ', heapPtr^);
  Dispose(heapPtr);         // ปลดปล่อย memory คืน
  heapPtr := nil;           // reset pointer
end.
```

---

## Integer Types

### ตารางชนิด Integer ทั้งหมด

| ชนิด | ขนาด (bytes) | ช่วงค่า | หมายเหตุ |
|------|-------------|---------|---------|
| **ShortInt** | 1 | -128 ถึง 127 | Signed 8-bit |
| **Byte** | 1 | 0 ถึง 255 | Unsigned 8-bit |
| **SmallInt** | 2 | -32,768 ถึง 32,767 | Signed 16-bit |
| **Word** | 2 | 0 ถึง 65,535 | Unsigned 16-bit |
| **Integer** | 4 | -2,147,483,648 ถึง 2,147,483,647 | Signed 32-bit (ใช้บ่อยที่สุด) |
| **Cardinal** | 4 | 0 ถึง 4,294,967,295 | Unsigned 32-bit |
| **LongInt** | 4 | -2,147,483,648 ถึง 2,147,483,647 | Signed 32-bit (เหมือน Integer) |
| **LongWord** | 4 | 0 ถึง 4,294,967,295 | Unsigned 32-bit |
| **Int64** | 8 | -9,223,372,036,854,775,808 ถึง 9,223,372,036,854,775,807 | Signed 64-bit |
| **QWord** | 8 | 0 ถึง 18,446,744,073,709,551,615 | Unsigned 64-bit |
| **NativeInt** | 4/8 | ขึ้นกับ CPU | Same size as pointer |
| **NativeUInt** | 4/8 | 0 ถึง max | Same size as pointer |

### ตัวอย่างการใช้งาน Integer Types

```pascal
program IntegerTypes;
var
  // ประกาศตัวแปรแต่ละชนิด
  b: Byte;           // 0-255: สำหรับ ASCII codes, small counts
  sh: ShortInt;      // -128-127: signed byte
  w: Word;           // 0-65535: port numbers, small positive values
  sm: SmallInt;      // -32768-32767
  i: Integer;        // -2B to 2B: ใช้ทั่วไป
  c: Cardinal;       // 0-4B: positive counts
  li: LongInt;       // เหมือน Integer
  i64: Int64;        // จำนวนใหญ่มาก: file sizes, timestamps
  q: QWord;          // unsigned 64-bit
begin
  // Byte ใช้กับ ASCII/small values
  b := 65;
  WriteLn('Byte: ', b, ' = ', Chr(b));  // A
  
  // ShortInt สำหรับ signed byte
  sh := -100;
  WriteLn('ShortInt: ', sh);
  
  // Word สำหรับ port number
  w := 8080;
  WriteLn('Port: ', w);
  
  // Integer ใช้ทั่วไป
  i := 2147483647;  // Max Integer
  WriteLn('Max Integer: ', i);
  
  // Int64 สำหรับจำนวนใหญ่
  i64 := 9223372036854775807;  // Max Int64
  WriteLn('Max Int64: ', i64);
  
  // คำนวณด้วย Int64
  i64 := 1000000;
  i64 := i64 * 1000000;  // 1 trillion
  WriteLn('1 trillion: ', i64);
end.
```

### ขีดจำกัดของ Integer

```pascal
program IntegerLimits;
uses
  SysUtils;

const
  // ค่าสูงสุดและต่ำสุด
  BYTE_MAX = High(Byte);      // 255
  BYTE_MIN = Low(Byte);       // 0
  INT_MAX  = High(Integer);   // 2147483647
  INT_MIN  = Low(Integer);    // -2147483648
  INT64_MAX = High(Int64);    // 9223372036854775807

begin
  WriteLn('=== Integer Limits ===');
  WriteLn('Byte:    ', BYTE_MIN, ' to ', BYTE_MAX);
  WriteLn('ShortInt:', Low(ShortInt), ' to ', High(ShortInt));
  WriteLn('SmallInt:', Low(SmallInt), ' to ', High(SmallInt));
  WriteLn('Word:    ', Low(Word), ' to ', High(Word));
  WriteLn('Integer: ', Low(Integer), ' to ', High(Integer));
  WriteLn('Cardinal:', Low(Cardinal), ' to ', High(Cardinal));
  WriteLn('Int64:   ', Low(Int64), ' to ', High(Int64));
  WriteLn;
  
  // Integer Overflow (ระวัง!)
  var x: Integer := High(Integer);
  WriteLn('High(Integer) + 1 = ', x + 1);  // Overflow! กลายเป็น Min
  
  // ใช้ Int64 เพื่อหลีกเลี่ยง overflow
  var y: Int64 := High(Integer);
  WriteLn('Int64 ปลอดภัย: ', y + 1);  // ถูกต้อง
end.
```

### ฟังก์ชันสำหรับ Integer

```pascal
program IntegerFunctions;
uses
  SysUtils, Math;
var
  a, b: Integer;
begin
  a := 42;
  b := -17;
  
  WriteLn('=== Integer Functions ===');
  WriteLn('Abs(', b, ') = ', Abs(b));           // ค่าสัมบูรณ์
  WriteLn('Sqr(', a, ') = ', Sqr(a));           // a²
  WriteLn('Succ(', a, ') = ', Succ(a));         // a + 1
  WriteLn('Pred(', a, ') = ', Pred(a));         // a - 1
  WriteLn('Inc(a) → ', a);                       // a := a + 1
  Inc(a); WriteLn('After Inc: ', a);
  Inc(a, 5); WriteLn('After Inc(a,5): ', a);    // a := a + 5
  Dec(a, 3); WriteLn('After Dec(a,3): ', a);   // a := a - 3
  
  // Division
  WriteLn('17 div 5 = ', 17 div 5);   // Integer division = 3
  WriteLn('17 mod 5 = ', 17 mod 5);   // Remainder = 2
  
  // Min/Max
  WriteLn('Min(', a, ',', b, ') = ', Min(a, b));
  WriteLn('Max(', a, ',', b, ') = ', Max(a, b));
  
  // Conversion
  WriteLn('IntToStr(', a, ') = ', IntToStr(a));
  WriteLn('IntToHex(255, 4) = ', IntToHex(255, 4));  // 00FF
  WriteLn('StrToInt("123") = ', StrToInt('123'));
end.
```

---

## Float Types

### ตารางชนิด Float ทั้งหมด

| ชนิด | ขนาด | ช่วงค่า | ทศนิยม | หมายเหตุ |
|------|------|---------|---------|---------|
| **Single** | 4 bytes | ±1.5×10⁻⁴⁵ ถึง ±3.4×10³⁸ | ~7-8 ตำแหน่ง | 32-bit IEEE 754 |
| **Double** | 8 bytes | ±5.0×10⁻³²⁴ ถึง ±1.7×10³⁰⁸ | ~15-16 ตำแหน่ง | 64-bit IEEE 754 (ใช้บ่อยที่สุด) |
| **Extended** | 10 bytes | ±1.9×10⁻⁴⁹⁵¹ ถึง ±1.1×10⁴⁹³² | ~18-19 ตำแหน่ง | 80-bit (x87 FPU) |
| **Real** | 8 bytes | เหมือน Double | เหมือน Double | Alias ของ Double ใน FPC |
| **Currency** | 8 bytes | -922,337,203,685,477.5808 ถึง 922,337,203,685,477.5807 | 4 ตำแหน่ง | Fixed-point, เหมาะกับการเงิน |
| **Comp** | 8 bytes | -2⁶³ ถึง 2⁶³-1 | ไม่มีทศนิยม | Integer stored as float |

### การใช้งาน Float Types

```pascal
program FloatTypes;
uses
  SysUtils, Math;
var
  s: Single;
  d: Double;
  e: Extended;
  c: Currency;
begin
  // Single - ความแม่นยำต่ำ แต่ใช้ memory น้อย
  s := 3.14159265358979;
  WriteLn('Single: ', s:20:10);  // แสดงได้ ~7 ตำแหน่ง

  // Double - ความแม่นยำสูง ใช้ทั่วไป
  d := 3.14159265358979;
  WriteLn('Double: ', d:20:15);  // แสดงได้ ~15 ตำแหน่ง

  // Extended - ความแม่นยำสูงสุด
  e := 3.14159265358979323846;
  WriteLn('Extended:', e:20:18); // แสดงได้ ~18 ตำแหน่ง

  // Currency - สำหรับการเงิน (ไม่มีปัญหา floating point)
  c := 1234567.89;
  WriteLn('Currency:', c:15:4);  // 4 ทศนิยมเสมอ
  
  // ปัญหา floating point precision
  WriteLn;
  WriteLn('=== Floating Point Issues ===');
  WriteLn('0.1 + 0.2 = ', 0.1 + 0.2);           // ไม่ใช่ 0.3 พอดี!
  WriteLn('0.1 + 0.2 = ', (0.1 + 0.2):20:17);   // เห็นความไม่แม่น
  WriteLn;
  WriteLn('ใช้ Currency แทน:');
  var ca: Currency := 0.1;
  var cb: Currency := 0.2;
  WriteLn('0.1 + 0.2 = ', ca + cb);             // 0.3 พอดี!
end.
```

### ฟังก์ชันคณิตศาสตร์

```pascal
program MathFunctions;
uses
  Math, SysUtils;
var
  x, y: Double;
begin
  x := 2.0;
  y := 3.0;
  
  WriteLn('=== Basic Math ===');
  WriteLn('x = ', x:3:1, ', y = ', y:3:1);
  WriteLn('Abs(x) = ', Abs(x):5:2);
  WriteLn('Sqr(x) = ', Sqr(x):5:2);       // x² = 4
  WriteLn('Sqrt(x) = ', Sqrt(x):8:6);     // √2 ≈ 1.414
  WriteLn('x^y = ', Power(x, y):5:2);     // 2³ = 8
  WriteLn('Exp(1) = ', Exp(1):8:6);       // e ≈ 2.718
  WriteLn('Ln(x) = ', Ln(x):8:6);         // ln(2) ≈ 0.693
  WriteLn('Log2(x) = ', Log2(x):5:2);     // log₂(2) = 1
  WriteLn('Log10(x) = ', Log10(x):5:2);   // log₁₀(2) ≈ 0.301
  
  WriteLn;
  WriteLn('=== Trigonometry (radians) ===');
  WriteLn('Sin(Pi/6) = ', Sin(Pi/6):8:6); // 0.5
  WriteLn('Cos(Pi/3) = ', Cos(Pi/3):8:6); // 0.5
  WriteLn('Tan(Pi/4) = ', Tan(Pi/4):8:6); // 1.0
  WriteLn('ArcSin(0.5) = ', ArcSin(0.5):8:6); // π/6
  WriteLn('ArcCos(0.5) = ', ArcCos(0.5):8:6); // π/3
  WriteLn('ArcTan(1.0) = ', ArcTan(1.0):8:6); // π/4
  WriteLn('ArcTan2(1,1) = ', ArcTan2(1,1):8:6); // π/4
  
  WriteLn;
  WriteLn('=== Rounding ===');
  x := 3.567;
  WriteLn('x = ', x:5:3);
  WriteLn('Round(x) = ', Round(x));        // 4 (nearest integer)
  WriteLn('Trunc(x) = ', Trunc(x));        // 3 (truncate)
  WriteLn('Floor(x) = ', Floor(x));        // 3 (round down)
  WriteLn('Ceil(x) = ', Ceil(x));          // 4 (round up)
  WriteLn('Frac(x) = ', Frac(x):5:3);     // 0.567 (fractional part)
  WriteLn('Int(x) = ', Int(x):5:1);        // 3.0 (integer part)
  
  WriteLn;
  WriteLn('=== Special Values ===');
  WriteLn('Pi = ', Pi:20:18);
  WriteLn('MaxDouble = ', MaxDouble);
  WriteLn('MinDouble = ', MinDouble);
  WriteLn('IsNaN(0/0) = ', IsNaN(0/0));    // Not a Number check
  WriteLn('IsInfinite(1/0) = ', IsInfinite(1/0)); // Infinity check
end.
```

### การจัดการปัญหา Floating Point

```pascal
program FloatPrecision;
uses
  Math;

// เปรียบเทียบ float อย่างถูกต้อง
function FloatEqual(a, b, epsilon: Double): Boolean;
begin
  Result := Abs(a - b) < epsilon;
end;

var
  a, b: Double;
begin
  a := 0.1 + 0.2;
  b := 0.3;
  
  WriteLn('0.1 + 0.2 = ', a:20:17);
  WriteLn('0.3       = ', b:20:17);
  
  // ผิด! (float comparison)
  if a = b then
    WriteLn('a = b (direct compare - อาจผิด!)')
  else
    WriteLn('a <> b (direct compare)');
  
  // ถูกต้อง (epsilon comparison)
  if FloatEqual(a, b, 1e-10) then
    WriteLn('a ≈ b (epsilon compare - ถูกต้อง!)')
  else
    WriteLn('a ≠ b');
  
  // หรือใช้ SameValue จาก Math
  if SameValue(a, b) then
    WriteLn('SameValue: a ≈ b')
  else
    WriteLn('SameValue: a ≠ b');
end.
```

---

## Boolean Type

### พื้นฐาน Boolean

```pascal
program BooleanDemo;
var
  isTrue: Boolean;
  isFalse: Boolean;
  a, b: Boolean;
begin
  isTrue := True;
  isFalse := False;
  
  WriteLn('isTrue = ', isTrue);    // TRUE
  WriteLn('isFalse = ', isFalse);  // FALSE
  
  // Boolean operators
  a := True;
  b := False;
  
  WriteLn;
  WriteLn('=== Boolean Operators ===');
  WriteLn('True AND False = ', a AND b);    // False
  WriteLn('True OR False = ', a OR b);     // True
  WriteLn('NOT True = ', NOT a);           // False
  WriteLn('True XOR False = ', a XOR b);  // True
  WriteLn('True XOR True = ', a XOR a);   // False
  
  // Short-circuit evaluation
  WriteLn;
  WriteLn('=== Short-circuit ===');
  // ถ้า a เป็น False, b จะไม่ถูกประเมิน (AND)
  if (10 > 20) and (1/0 = 0) then  // (1/0) จะไม่ถูกคำนวณ!
    WriteLn('This won''t print');
  
  // ถ้า a เป็น True, b จะไม่ถูกประเมิน (OR)
  if (10 < 20) or (1/0 = 0) then  // (1/0) จะไม่ถูกคำนวณ!
    WriteLn('OR Short-circuit works');
end.
```

### Boolean ใน Conditions

```pascal
program BooleanConditions;
var
  score: Integer;
  isPassed, isHonors, isStudent: Boolean;
  grade: Char;
begin
  Write('ใส่คะแนน: ');
  ReadLn(score);
  
  // กำหนดค่า boolean จาก condition
  isPassed := score >= 50;
  isHonors := score >= 80;
  isStudent := True;
  
  WriteLn;
  WriteLn('=== ผลการเรียน ===');
  WriteLn('ผ่าน: ', isPassed);
  WriteLn('เกียรตินิยม: ', isHonors);
  
  if isPassed then
  begin
    if isHonors then
      WriteLn('ยอดเยี่ยม! ผ่านด้วยเกียรตินิยม')
    else
      WriteLn('ผ่านแล้ว!');
  end
  else
    WriteLn('ไม่ผ่าน ต้องสอบใหม่');
  
  // Boolean ใน compound conditions
  if isStudent and isPassed and (score >= 60) then
    WriteLn('นักเรียนที่ผ่านด้วยคะแนน >= 60');
end.
```

### Boolean Conversion

```pascal
program BoolConversion;
uses
  SysUtils;
var
  b: Boolean;
  i: Integer;
  s: String;
begin
  // Boolean → Integer
  b := True;
  WriteLn('Ord(True) = ', Ord(True));   // 1
  WriteLn('Ord(False) = ', Ord(False)); // 0
  
  // Boolean → String
  WriteLn('BoolToStr(True) = ', BoolToStr(True));   // "TRUE" หรือ "-1"
  WriteLn('BoolToStr(False) = ', BoolToStr(False));  // "FALSE" หรือ "0"
  WriteLn('BoolToStr(True, True) = ', BoolToStr(True, True));  // "True"
  
  // Integer → Boolean
  i := 1;
  b := Boolean(i);  // True (ค่าไม่ใช่ 0 = True)
  WriteLn('Boolean(1) = ', b);
  
  i := 0;
  b := Boolean(i);  // False
  WriteLn('Boolean(0) = ', b);
  
  // String → Boolean (parse)
  s := 'TRUE';
  b := StrToBool(s);
  WriteLn('StrToBool("TRUE") = ', b);
end.
```

---

## Character Type (Char)

### พื้นฐาน Char

```pascal
program CharDemo;
var
  ch: Char;
  i: Integer;
begin
  // การกำหนดค่า
  ch := 'A';              // ตัวอักษร
  ch := #65;             // ASCII code 65 = 'A'
  ch := Chr(65);         // ฟังก์ชัน Chr()
  
  WriteLn('ch = ', ch);
  WriteLn('Ord(ch) = ', Ord(ch));  // ASCII code
  
  // แสดงตัวอักษร A-Z
  WriteLn;
  Write('A-Z: ');
  for ch := 'A' to 'Z' do
    Write(ch);
  WriteLn;
  
  // แสดง ASCII table (32-127)
  WriteLn;
  WriteLn('=== ASCII Table ===');
  for i := 32 to 127 do
  begin
    Write(Chr(i), '(', i, ')  ');
    if (i - 31) mod 8 = 0 then WriteLn;  // 8 ต่อบรรทัด
  end;
  WriteLn;
end.
```

### Char Operations

```pascal
program CharOperations;
uses
  SysUtils;
var
  ch: Char;
begin
  ch := 'a';
  
  // Case conversion
  WriteLn('UpCase(', ch, ') = ', UpCase(ch));  // A
  WriteLn('LowerCase(', ch, ') = ', LowerCase(ch));  // a
  
  ch := 'G';
  WriteLn('UpCase(', ch, ') = ', UpCase(ch));  // G
  WriteLn('LowerCase(', ch, ') = ', LowerCase(ch));  // g
  
  // ตรวจสอบประเภท (ใช้ sets)
  WriteLn;
  WriteLn('=== Character Classification ===');
  
  procedure CheckChar(c: Char);
  begin
    Write(c, ': ');
    if c in ['A'..'Z'] then Write('Uppercase  ');
    if c in ['a'..'z'] then Write('Lowercase  ');
    if c in ['0'..'9'] then Write('Digit  ');
    if c in ['!','@','#','$','%','^','&','*'] then Write('Special  ');
    if c = ' ' then Write('Space  ');
    WriteLn;
  end;
  
  CheckChar('A');
  CheckChar('z');
  CheckChar('5');
  CheckChar('!');
  CheckChar(' ');
  
  // Arithmetic on Char
  ch := 'A';
  WriteLn;
  WriteLn('A + 1 = ', Chr(Ord(ch) + 1));  // B
  WriteLn('A + 25 = ', Chr(Ord(ch) + 25)); // Z
end.
```

### WideChar (Unicode Character)

```pascal
program WideCharDemo;
var
  wch: WideChar;
  ws: WideString;
begin
  // WideChar - 2 bytes, รองรับ Unicode
  wch := 'A';
  wch := WideChar(#$0E2A);  // ส (Thai character)
  
  ws := 'สวัสดี';
  WriteLn('WideString length: ', Length(ws));
  
  // ใน FPC/Lazarus สมัยใหม่ ใช้ UnicodeString
  var us: UnicodeString := 'Hello สวัสดี';
  WriteLn(us);
end.
```

---

## String Types

### ตาราง String Types

| ชนิด | ความยาวสูงสุด | Encoding | หมายเหตุ |
|------|-------------|---------|---------|
| **ShortString** | 255 chars | Single-byte | Pascal สมัยก่อน |
| **AnsiString** | 2 GB | Single-byte/ANSI | Default String ใน FPC |
| **WideString** | 2 GB | UTF-16 | Windows COM |
| **UnicodeString** | 2 GB | UTF-16 | รองรับ Unicode เต็ม |
| **RawByteString** | 2 GB | Any | Raw bytes, no encoding |
| **UTF8String** | 2 GB | UTF-8 | เหมาะกับ Linux/modern |

### AnsiString (Default String)

```pascal
program AnsiStringDemo;
uses
  SysUtils;
var
  s1, s2, s3: String;  // ← AnsiString ใน FPC {$H+} mode
  len: Integer;
begin
  // การสร้าง String
  s1 := 'Hello';
  s2 := 'World';
  
  // String concatenation
  s3 := s1 + ' ' + s2;
  WriteLn(s3);  // Hello World
  
  // String functions
  WriteLn;
  WriteLn('=== String Functions ===');
  WriteLn('Length: ', Length(s3));       // 11
  WriteLn('UpperCase: ', UpperCase(s3));  // HELLO WORLD
  WriteLn('LowerCase: ', LowerCase(s3));  // hello world
  WriteLn('Trim: "', Trim('  Hello  '), '"');  // "Hello"
  WriteLn('TrimLeft: "', TrimLeft('  Hello'), '"');
  WriteLn('TrimRight: "', TrimRight('Hello  '), '"');
  
  // Substring
  WriteLn;
  WriteLn('=== Substring ===');
  WriteLn('Copy(s3, 1, 5): ', Copy(s3, 1, 5));     // Hello
  WriteLn('Copy(s3, 7, 5): ', Copy(s3, 7, 5));     // World
  WriteLn('LeftStr(s3, 3): ', LeftStr(s3, 3));     // Hel
  WriteLn('RightStr(s3, 3): ', RightStr(s3, 3));   // rld
  WriteLn('MidStr(s3, 4, 3): ', MidStr(s3, 4, 3)); // lo 
  
  // Search
  WriteLn;
  WriteLn('=== Search ===');
  WriteLn('Pos("World", s3): ', Pos('World', s3));  // 7
  WriteLn('Pos("xyz", s3): ', Pos('xyz', s3));      // 0 (ไม่พบ)
  WriteLn('PosEx("l", s3, 1): ', PosEx('l', s3, 1)); // 3
  WriteLn('PosEx("l", s3, 4): ', PosEx('l', s3, 4)); // 4 (ถัดไป)
  
  // Replace
  WriteLn;
  WriteLn('=== Replace ===');
  WriteLn(StringReplace(s3, 'World', 'Pascal', []));           // Hello Pascal
  WriteLn(StringReplace('aababab', 'a', 'X', [rfReplaceAll])); // XXbXbXb
  WriteLn(StringReplace('HELLO', 'hello', 'HI', [rfIgnoreCase])); // HI
end.
```

### ShortString

```pascal
program ShortStringDemo;
var
  ss: ShortString;        // สูงสุด 255 ตัวอักษร
  ss20: String[20];       // สูงสุด 20 ตัวอักษร
begin
  ss := 'Hello Pascal';
  ss20 := 'Short string!';
  
  WriteLn('ShortString: ', ss);
  WriteLn('Length: ', Length(ss));
  WriteLn('Max length of ss20: ', High(ss20));  // 20
  
  // ตัดทิ้งถ้ายาวเกิน
  ss20 := 'This string is longer than 20 characters';
  WriteLn('Truncated: "', ss20, '"');  // เหลือแค่ 20 ตัว
  
  // SizeOf vs Length
  WriteLn('SizeOf(ss20) = ', SizeOf(ss20));  // 21 (20 + length byte)
  WriteLn('Length(ss20) = ', Length(ss20));  // 20
end.
```

### String Manipulation ขั้นสูง

```pascal
program AdvancedStrings;
uses
  SysUtils, StrUtils;
var
  s: String;
  parts: TStringArray;
  i: Integer;
begin
  s := 'สมชาย,25,กรุงเทพ,นักเรียน';
  
  // Split string
  parts := s.Split([',']);
  WriteLn('Split result:');
  for i := 0 to Length(parts) - 1 do
    WriteLn('  [', i, '] = ', parts[i]);
  
  // Join strings
  WriteLn;
  WriteLn('Joined: ', String.Join(' | ', parts));
  
  // String formatting
  WriteLn;
  WriteLn(Format('Name: %-15s Age: %3d', ['สมชาย', 25]));
  WriteLn(Format('Price: %10.2f Baht', [1234.5]));
  
  // Convert number to string
  WriteLn;
  WriteLn('IntToStr: ', IntToStr(42));
  WriteLn('FloatToStr: ', FloatToStr(3.14));
  WriteLn('FloatToStrF: ', FloatToStrF(3.14159, ffFixed, 10, 3));
  
  // Reverse string
  WriteLn;
  WriteLn('Reverse: ', ReverseString('Hello World'));
  
  // Count occurrences
  var countA := 0;
  var pos := 1;
  repeat
    pos := PosEx('a', 'banana', pos);
    if pos > 0 then
    begin
      Inc(countA);
      Inc(pos);
    end;
  until pos = 0;
  WriteLn('Count of "a" in "banana": ', countA);
end.
```

### Unicode และ UTF-8

```pascal
program UnicodeDemo;
uses
  SysUtils, LazUTF8;  // ต้องใช้ LazUTF8 สำหรับ UTF-8 operations

var
  utf8str: UTF8String;
  s: String;
begin
  // UTF-8 String (สำหรับ Lazarus/Linux)
  utf8str := 'สวัสดีชาวโลก';
  s := 'Hello, World!';
  
  WriteLn('UTF8: ', utf8str);
  WriteLn('Byte length: ', Length(utf8str));
  WriteLn('Char length: ', UTF8Length(utf8str));  // จำนวนตัวอักษรจริง
  
  // การแปลง
  WriteLn;
  WriteLn('UTF8 to AnsiString: ', UTF8ToAnsi(utf8str));
  WriteLn('Ansi to UTF8: ', AnsiToUTF8(s));
  
  // UTF8 operations
  WriteLn;
  WriteLn('UTF8Copy: ', UTF8Copy(utf8str, 1, 4));  // สวัส
  WriteLn('UTF8UpperCase: ', UTF8UpperCase('hello สวัสดี'));
  WriteLn('UTF8LowerCase: ', UTF8LowerCase('HELLO สวัสดี'));
end.
```

---

## Subrange Types

### นิยาม Subrange Type

```pascal
program SubrangeDemo;

type
  // Subrange จาก built-in types
  TScore = 0..100;           // คะแนน 0-100
  TMonth = 1..12;            // เดือน 1-12
  TDay = 1..31;              // วัน 1-31
  TSmallDigit = 0..9;        // ตัวเลข 0-9
  
  // Subrange จาก Char
  TUpperCase = 'A'..'Z';     // ตัวพิมพ์ใหญ่
  TLowerCase = 'a'..'z';     // ตัวพิมพ์เล็ก
  TDigitChar = '0'..'9';     // ตัวเลข 0-9

var
  score: TScore;
  month: TMonth;
  ch: TUpperCase;
begin
  score := 85;
  month := 10;
  ch := 'A';
  
  WriteLn('Score: ', score);
  WriteLn('Month: ', month);
  WriteLn('Char: ', ch);
  
  // Range checking (เปิดด้วย {$R+})
  {$R+}
  // score := 150;  // ← ERROR at runtime: Range check error
  {$R-}
end.
```

### ประโยชน์ของ Subrange Types

```pascal
program SubrangeUsage;
uses
  SysUtils;

type
  TAge = 0..150;
  TYear = 1900..2100;
  TGrade = 'A'..'F';

// ฟังก์ชันที่รับแค่ค่าที่ถูกต้อง
procedure PrintGrade(score: Integer);
var
  grade: TGrade;
begin
  if score >= 80 then grade := 'A'
  else if score >= 70 then grade := 'B'
  else if score >= 60 then grade := 'C'
  else if score >= 50 then grade := 'D'
  else grade := 'F';
  WriteLn('Grade: ', grade);
end;

var
  age: TAge;
  year: TYear;
begin
  age := 25;
  year := 2024;
  
  WriteLn('Age: ', age);
  WriteLn('Year: ', year);
  
  PrintGrade(85);  // A
  PrintGrade(62);  // C
  PrintGrade(45);  // F
end.
```

---

## Type Casting

### Explicit Type Casting

```pascal
program TypeCasting;
uses
  SysUtils;
var
  i: Integer;
  f: Double;
  b: Byte;
  ch: Char;
  s: String;
begin
  // Integer ↔ Double
  i := 42;
  f := Double(i);        // Integer → Double
  WriteLn('Int to Double: ', f:5:2);
  
  f := 3.99;
  i := Integer(Round(f));  // Double → Integer (ปัดเศษ)
  WriteLn('Double to Int (rounded): ', i);  // 4
  
  i := Integer(Trunc(f));  // Double → Integer (ตัดทิ้ง)
  WriteLn('Double to Int (truncated): ', i);  // 3
  
  // Integer ↔ Byte
  i := 300;
  b := Byte(i);          // Integer → Byte (ล้น! 300 mod 256 = 44)
  WriteLn('300 to Byte: ', b);  // 44
  
  // Integer ↔ Char
  i := 65;
  ch := Chr(i);          // Integer → Char
  WriteLn('65 to Char: ', ch);  // A
  
  i := Ord(ch);          // Char → Integer
  WriteLn('A to Integer: ', i);  // 65
  
  // Number ↔ String
  i := 42;
  s := IntToStr(i);      // Integer → String
  WriteLn('Int to String: ', s);
  
  i := StrToInt(s);      // String → Integer (อาจ exception ถ้าไม่ใช่เลข)
  WriteLn('String to Int: ', i);
  
  f := StrToFloat('3.14');
  WriteLn('String to Float: ', f:5:2);
  
  s := FloatToStr(f);
  WriteLn('Float to String: ', s);
  
  // Safe conversion
  var n: Integer;
  if TryStrToInt('123', n) then
    WriteLn('Valid integer: ', n)
  else
    WriteLn('Not a valid integer');
  
  if TryStrToInt('abc', n) then
    WriteLn('Valid integer: ', n)
  else
    WriteLn('Not a valid integer');  // ← จะพิมพ์นี้
end.
```

### Implicit Type Conversion

```pascal
program ImplicitConversion;
var
  i: Integer;
  f: Double;
  s: Single;
begin
  // Integer → Double (อัตโนมัติ)
  i := 5;
  f := i;  // Pascal แปลงให้อัตโนมัติ
  WriteLn('i = ', i, ', f = ', f:5:2);
  
  // Single → Double (อัตโนมัติ - widening)
  s := 3.14;
  f := s;  // Single → Double (ปลอดภัย)
  WriteLn('s = ', s:8:5, ', f = ', f:8:5);
  
  // Double → Single (ต้องระวัง - narrowing, อาจเสียความแม่น)
  f := 3.14159265358979;
  s := f;  // Double → Single (เสียความแม่น)
  WriteLn('Double: ', f:20:15);
  WriteLn('Single: ', s:20:10);  // เห็นความต่าง
end.
```

---

## Constants (const)

### การประกาศ Constants

```pascal
program ConstantsDemo;
uses
  Math;

const
  // Simple constants
  APP_NAME = 'My Pascal App';
  VERSION = '1.0.0';
  MAX_SIZE = 100;
  PI_VALUE = 3.14159265358979;
  
  // Computed constants (คำนวณจาก constant อื่น)
  DOUBLE_MAX = MAX_SIZE * 2;    // 200
  AREA = PI_VALUE * 5 * 5;      // pi * r²
  
  // String constants
  SEPARATOR = '================================';
  NEWLINE = #13#10;  // CR+LF
  
  // Boolean constant
  DEBUG_MODE = False;
  
var
  radius: Double;
  area: Double;
begin
  WriteLn(SEPARATOR);
  WriteLn(APP_NAME, ' v', VERSION);
  WriteLn(SEPARATOR);
  WriteLn;
  
  radius := 5.0;
  area := PI_VALUE * radius * radius;
  WriteLn('Area of circle (r=5) = ', area:10:4);
  WriteLn('Pre-calculated = ', AREA:10:4);  // ใช้ค่าคงที่แทน
  
  if DEBUG_MODE then
    WriteLn('DEBUG: area = ', area);
end.
```

### ค่าคงที่ที่สำคัญ (Built-in Constants)

```pascal
program BuiltinConstants;
uses
  Math, SysUtils;
begin
  WriteLn('=== Mathematical Constants ===');
  WriteLn('Pi = ', Pi:20:18);
  WriteLn('Exp(1) = e = ', Exp(1):20:18);
  WriteLn('Sqrt(2) = ', Sqrt(2):20:18);
  
  WriteLn;
  WriteLn('=== Type Limits ===');
  WriteLn('MaxInt = ', MaxInt);           // Max Integer
  WriteLn('MaxLongInt = ', MaxLongInt);   // Max LongInt
  WriteLn('MaxSingle = ', MaxSingle);     // Max Single
  WriteLn('MaxDouble = ', MaxDouble);     // Max Double
  WriteLn('MinSingle = ', MinSingle);     // Smallest positive Single
  WriteLn('MinDouble = ', MinDouble);     // Smallest positive Double
  
  WriteLn;
  WriteLn('=== Boolean Constants ===');
  WriteLn('True = ', True);
  WriteLn('False = ', False);
  
  WriteLn;
  WriteLn('=== Nil ===');
  var p: PInteger := nil;
  WriteLn('nil pointer: ', p = nil);  // True
end.
```

---

## Typed Constants

### การใช้ Typed Constants

```pascal
program TypedConstants;

// Typed constant - ประกาศพร้อม type และค่า
// ใน mode delphi: สามารถเปลี่ยนค่าได้ (เหมือนตัวแปร global)
// ใน mode objfpc: ต้องใช้ {$J+} ก่อนถึงจะเปลี่ยนค่าได้
{$J+}

const
  // Typed constants
  Counter: Integer = 0;
  DefaultName: String = 'Unknown';
  DefaultScore: Double = 0.0;
  Flags: array[1..5] of Boolean = (True, False, True, True, False);
  
  // Record constant
  type
    TPoint = record
      X, Y: Integer;
    end;
  
  Origin: TPoint = (X: 0; Y: 0);
  TopLeft: TPoint = (X: 100; Y: 50);

begin
  WriteLn('Counter = ', Counter);      // 0
  Counter := Counter + 1;              // เปลี่ยนค่าได้ (ต้องใช้ {$J+})
  WriteLn('Counter = ', Counter);      // 1
  
  WriteLn('DefaultName = ', DefaultName);
  DefaultName := 'Pascal User';
  WriteLn('DefaultName = ', DefaultName);  // Pascal User
  
  WriteLn;
  WriteLn('Flags:');
  var i: Integer;
  for i := 1 to 5 do
    WriteLn('  Flags[', i, '] = ', Flags[i]);
  
  WriteLn;
  WriteLn('Origin = (', Origin.X, ', ', Origin.Y, ')');
  WriteLn('TopLeft = (', TopLeft.X, ', ', TopLeft.Y, ')');
end.
```

---

## โปรแกรมตัวอย่าง 5 โปรแกรม

### โปรแกรมที่ 1: ตัวแปรทุกชนิด

```pascal
program AllDataTypes;
uses
  SysUtils;
var
  // Integer types
  byteVal: Byte;
  shortIntVal: ShortInt;
  smallIntVal: SmallInt;
  wordVal: Word;
  intVal: Integer;
  cardinalVal: Cardinal;
  int64Val: Int64;
  
  // Float types
  singleVal: Single;
  doubleVal: Double;
  currencyVal: Currency;
  
  // Boolean
  boolVal: Boolean;
  
  // Char
  charVal: Char;
  
  // String types
  shortStrVal: ShortString;
  ansiStrVal: AnsiString;
  wideStrVal: WideString;
  
begin
  // Integer
  byteVal := 200;
  shortIntVal := -100;
  smallIntVal := -30000;
  wordVal := 60000;
  intVal := -2000000000;
  cardinalVal := 4000000000;
  int64Val := 9000000000000000000;
  
  // Float
  singleVal := 3.14;
  doubleVal := 3.14159265358979;
  currencyVal := 1234567.89;
  
  // Boolean
  boolVal := True;
  
  // Char
  charVal := 'A';
  
  // String
  shortStrVal := 'Short string (max 255)';
  ansiStrVal := 'ANSI String';
  wideStrVal := 'Wide String - Unicode';
  
  WriteLn('=== ชนิดข้อมูลทั้งหมด ===');
  WriteLn;
  WriteLn('--- Integer Types ---');
  WriteLn('Byte (1 byte):     ', byteVal);
  WriteLn('ShortInt (1 byte): ', shortIntVal);
  WriteLn('SmallInt (2 bytes):', smallIntVal);
  WriteLn('Word (2 bytes):    ', wordVal);
  WriteLn('Integer (4 bytes): ', intVal);
  WriteLn('Cardinal (4 bytes):', cardinalVal);
  WriteLn('Int64 (8 bytes):   ', int64Val);
  
  WriteLn;
  WriteLn('--- Float Types ---');
  WriteLn('Single (4 bytes):  ', singleVal:10:4);
  WriteLn('Double (8 bytes):  ', doubleVal:15:10);
  WriteLn('Currency:          ', currencyVal:15:4);
  
  WriteLn;
  WriteLn('--- Other Types ---');
  WriteLn('Boolean:   ', boolVal);
  WriteLn('Char:      ', charVal);
  WriteLn('ShortStr:  ', shortStrVal);
  WriteLn('AnsiStr:   ', ansiStrVal);
  WriteLn('WideStr:   ', wideStrVal);
end.
```

### โปรแกรมที่ 2: Integer Overflow Demo

```pascal
program IntegerOverflowDemo;
var
  b: Byte;
  i: Integer;
  i64: Int64;
begin
  WriteLn('=== Integer Overflow ===');
  WriteLn;
  
  // Byte overflow
  b := 255;  // Max Byte
  WriteLn('Byte(255): ', b);
  Inc(b);    // Overflow! กลับไป 0
  WriteLn('Inc(255) → ', b);  // 0
  
  Inc(b, 10);
  WriteLn('+ 10 → ', b);  // 10
  
  WriteLn;
  
  // Integer overflow
  i := High(Integer);
  WriteLn('Max Integer: ', i);
  Inc(i);    // Overflow! กลายเป็น Min
  WriteLn('Inc(MaxInt) → ', i);  // -2147483648
  
  WriteLn;
  WriteLn('=== Using Int64 to avoid overflow ===');
  i64 := High(Integer);
  WriteLn('Int64(MaxInt): ', i64);
  Inc(i64);  // ปลอดภัย!
  WriteLn('Inc → ', i64);  // 2147483648
  
  // Multiplication overflow
  WriteLn;
  i := 100000;
  WriteLn('100000 * 100000 = ', i * i);  // Overflow!
  i64 := 100000;
  WriteLn('Int64: 100000 * 100000 = ', i64 * i64);  // ถูกต้อง
end.
```

### โปรแกรมที่ 3: String Operations Complete

```pascal
program StringOperationsComplete;
uses
  SysUtils, StrUtils;

procedure Separator(title: String);
begin
  WriteLn;
  WriteLn('=== ', title, ' ===');
end;

var
  s1, s2, s3: String;
  i: Integer;
begin
  s1 := 'Hello, Pascal World!';
  
  Separator('Basic Operations');
  WriteLn('Original: "', s1, '"');
  WriteLn('Length: ', Length(s1));
  WriteLn('UpperCase: ', UpperCase(s1));
  WriteLn('LowerCase: ', LowerCase(s1));
  
  Separator('Substrings');
  WriteLn('Copy(s1, 1, 5): "', Copy(s1, 1, 5), '"');    // Hello
  WriteLn('Copy(s1, 8, 6): "', Copy(s1, 8, 6), '"');    // Pascal
  WriteLn('Copy(s1, 15): "', Copy(s1, 15), '"');         // World!
  WriteLn('LeftStr(s1, 5): "', LeftStr(s1, 5), '"');    // Hello
  WriteLn('RightStr(s1, 6): "', RightStr(s1, 6), '"');  // World!
  
  Separator('Search and Replace');
  WriteLn('Pos("Pascal"): ', Pos('Pascal', s1));         // 8
  WriteLn('Pos("python"): ', Pos('python', s1));         // 0 (ไม่พบ)
  
  s2 := StringReplace(s1, 'World', 'Universe', []);
  WriteLn('Replace: "', s2, '"');
  
  s3 := StringReplace(s1, 'l', 'L', [rfReplaceAll]);
  WriteLn('Replace all l→L: "', s3, '"');
  
  Separator('Insert and Delete');
  s2 := s1;
  Insert(' Beautiful', s2, Pos('World', s2));
  WriteLn('After Insert: "', s2, '"');
  
  s2 := s1;
  Delete(s2, Pos(',', s2), 1);  // ลบ comma
  WriteLn('After Delete comma: "', s2, '"');
  
  Separator('Conversion');
  WriteLn('IntToStr(42): ', IntToStr(42));
  WriteLn('FloatToStr(3.14): ', FloatToStr(3.14));
  WriteLn('BoolToStr(True): ', BoolToStr(True, True));
  WriteLn('StrToInt("123"): ', StrToInt('123'));
  WriteLn('StrToFloat("3.14"): ', StrToFloat('3.14'):5:2);
  
  Separator('Padding and Formatting');
  s1 := 'Hi';
  WriteLn('PadLeft(s1, 10): "', s1.PadLeft(10), '"');
  WriteLn('PadRight(s1, 10): "', s1.PadRight(10), '"');
  
  Separator('String Test');
  WriteLn('Contains("Pascal"): ', s1.Contains('Pascal'));
  WriteLn('StartsWith("He"): ', s1.StartsWith('He'));
  WriteLn('EndsWith("ld!"): ', s1.EndsWith('ld!'));
end.
```

### โปรแกรมที่ 4: Type Conversion Matrix

```pascal
program TypeConversionMatrix;
uses
  SysUtils;

procedure ConvertAll(i: Integer; f: Double; s: String; b: Boolean; c: Char);
begin
  WriteLn('=== Input Values ===');
  WriteLn('Integer: ', i);
  WriteLn('Double:  ', f:10:4);
  WriteLn('String:  ', s);
  WriteLn('Boolean: ', b);
  WriteLn('Char:    ', c);
  
  WriteLn;
  WriteLn('=== Conversions ===');
  
  // From Integer
  WriteLn('Int→Double: ', Double(i):8:2);
  WriteLn('Int→String: ', IntToStr(i));
  WriteLn('Int→Char:   ', Chr(i mod 256));
  WriteLn('Int→Bool:   ', i <> 0);
  
  // From Double
  WriteLn('Dbl→Int(Round): ', Round(f));
  WriteLn('Dbl→Int(Trunc): ', Trunc(f));
  WriteLn('Dbl→String:     ', FloatToStr(f));
  WriteLn('Dbl→Bool:       ', f <> 0.0);
  
  // From String (ระวัง exception!)
  var si, sf: Integer;
  if TryStrToInt(s, si) then
    WriteLn('Str→Int: ', si)
  else
    WriteLn('Str→Int: ไม่สามารถแปลงได้');
  
  // From Boolean
  WriteLn('Bool→Int:    ', Ord(b));
  WriteLn('Bool→String: ', BoolToStr(b, True));
  
  // From Char
  WriteLn('Char→Int:    ', Ord(c));
  WriteLn('Char→String: ', String(c));
  WriteLn('Char→Upper:  ', UpCase(c));
end;

begin
  ConvertAll(65, 3.14, '42', True, 'A');
end.
```

### โปรแกรมที่ 5: ระบบจัดการนักศึกษาพร้อม Types

```pascal
program StudentSystem;
uses
  SysUtils, Math;

type
  // Subrange types
  TScore = 0..100;
  TStudentID = 1..9999;
  
  // Enumeration
  TGrade = (grA, grB, grC, grD, grF);
  TDepartment = (deptCS, deptEng, deptBusiness, deptArt, deptScience);

  // Record
  TStudent = record
    ID: TStudentID;
    Name: String[50];
    Scores: array[1..5] of TScore;  // 5 วิชา
    Department: TDepartment;
    IsActive: Boolean;
  end;

const
  DEPT_NAMES: array[TDepartment] of String = (
    'Computer Science',
    'Engineering',
    'Business',
    'Art',
    'Science'
  );
  
  GRADE_NAMES: array[TGrade] of String = ('A', 'B', 'C', 'D', 'F');

function CalcAverage(scores: array of Integer): Double;
var
  i: Integer;
  total: Integer;
begin
  total := 0;
  for i := Low(scores) to High(scores) do
    Inc(total, scores[i]);
  Result := total / Length(scores);
end;

function GetGrade(avg: Double): TGrade;
begin
  if avg >= 80 then Result := grA
  else if avg >= 70 then Result := grB
  else if avg >= 60 then Result := grC
  else if avg >= 50 then Result := grD
  else Result := grF;
end;

procedure PrintStudent(s: TStudent);
var
  i: Integer;
  avg: Double;
  scoreArray: array[1..5] of Integer;
begin
  for i := 1 to 5 do
    scoreArray[i] := s.Scores[i];
  
  avg := CalcAverage(scoreArray);
  
  WriteLn('┌────────────────────────────────┐');
  WriteLn('│ ID:    ', s.ID:4, '                        │');
  WriteLn('│ Name:  ', s.Name:20, '     │');
  WriteLn('│ Dept:  ', DEPT_NAMES[s.Department]:20, '     │');
  WriteLn('│ Active:', BoolToStr(s.IsActive, True), '                    │');
  Write  ('│ Scores:');
  for i := 1 to 5 do
    Write(s.Scores[i]:4);
  WriteLn('          │');
  WriteLn('│ Average:', avg:6:2,
          '  Grade: ', GRADE_NAMES[GetGrade(avg)], '          │');
  WriteLn('└────────────────────────────────┘');
end;

var
  students: array[1..3] of TStudent;
  i: Integer;
begin
  // กำหนดข้อมูล
  students[1].ID := 1001;
  students[1].Name := 'Somchai Jaidee';
  students[1].Scores[1] := 85;
  students[1].Scores[2] := 72;
  students[1].Scores[3] := 90;
  students[1].Scores[4] := 68;
  students[1].Scores[5] := 78;
  students[1].Department := deptCS;
  students[1].IsActive := True;
  
  students[2].ID := 1002;
  students[2].Name := 'Somying Srirat';
  students[2].Scores[1] := 95;
  students[2].Scores[2] := 88;
  students[2].Scores[3] := 92;
  students[2].Scores[4] := 85;
  students[2].Scores[5] := 91;
  students[2].Department := deptScience;
  students[2].IsActive := True;
  
  students[3].ID := 1003;
  students[3].Name := 'Manee Porn';
  students[3].Scores[1] := 45;
  students[3].Scores[2] := 55;
  students[3].Scores[3] := 40;
  students[3].Scores[4] := 50;
  students[3].Scores[5] := 48;
  students[3].Department := deptBusiness;
  students[3].IsActive := False;
  
  WriteLn('=== ข้อมูลนักศึกษา ===');
  WriteLn;
  for i := 1 to 3 do
    PrintStudent(students[i]);
end.
```

---

## แบบฝึกหัด 20 ข้อ

### ข้อ 1: ขีดจำกัด Integer
เขียนโปรแกรมแสดงค่าสูงสุดและต่ำสุดของทุก Integer type (Byte, ShortInt, SmallInt, Word, Integer, Cardinal, Int64) พร้อมขนาดใน bytes

### ข้อ 2: Integer Overflow Simulation
เขียนโปรแกรมที่แสดงให้เห็น overflow behavior:
- บวก 1 ให้ Max Byte
- บวก 1 ให้ Max Integer
- ลบ 1 จาก Min Integer

### ข้อ 3: Float Precision
เปรียบเทียบความแม่นยำของ Single, Double และ Extended โดยคำนวณ 1/3 และแสดงด้วย 20 ตำแหน่ง

### ข้อ 4: Currency Calculator
เขียนโปรแกรมคิดภาษี VAT 7% โดยใช้ Currency type เพื่อหลีกเลี่ยงปัญหา floating point

### ข้อ 5: ASCII Table Generator
สร้างตาราง ASCII ตั้งแต่ช่วง 32-127 แสดงในรูปแบบตาราง 8 คอลัมน์

### ข้อ 6: Boolean Logic Table
สร้างตาราง truth table สำหรับ AND, OR, NOT, XOR:
```
A     | B     | A AND B | A OR B | NOT A | A XOR B
------|-------|---------|--------|-------|--------
False | False | False   | False  | True  | False
False | True  | False   | True   | True  | True
...
```

### ข้อ 7: String Palindrome
เขียนฟังก์ชัน `IsPalindrome(s: String): Boolean` และทดสอบกับคำต่างๆ

### ข้อ 8: Caesar Cipher
เข้ารหัสและถอดรหัส text ด้วย Caesar Cipher (เลื่อนตัวอักษร n ตำแหน่ง):
```pascal
function Encrypt(s: String; shift: Integer): String;
function Decrypt(s: String; shift: Integer): String;
```

### ข้อ 9: String Statistics
รับ String แล้วนับ:
- จำนวนตัวพิมพ์ใหญ่
- จำนวนตัวพิมพ์เล็ก
- จำนวนตัวเลข
- จำนวน spaces
- จำนวนตัวอักษรพิเศษ

### ข้อ 10: Type Validator
เขียนฟังก์ชันตรวจสอบว่า string เป็น valid:
- Integer (`IsValidInt`)
- Float (`IsValidFloat`)  
- Email (`IsValidEmail` - ตรวจ @ และ .)
- Date (dd/mm/yyyy) (`IsValidDate`)

### ข้อ 11: Subrange ระดับคะแนน
สร้าง type:
```pascal
type
  TScore = 0..100;
  TGrade = (grA, grB, grC, grD, grF);
```
เขียนฟังก์ชัน `ScoreToGrade(score: TScore): TGrade`
และแสดงตาราง score → grade

### ข้อ 12: Record ข้อมูลสินค้า
สร้าง record:
```pascal
type
  TProduct = record
    Code: String[10];
    Name: String[50];
    Price: Currency;
    Quantity: Integer;
    Category: String[20];
  end;
```
เก็บข้อมูล 5 สินค้า แล้วแสดงรายการพร้อมมูลค่ารวม

### ข้อ 13: อาร์เรย์และ Statistics
รับตัวเลข 10 ตัว แล้วคำนวณ:
- ค่าเฉลี่ย (mean)
- ค่ากลาง (median)
- ฐานนิยม (mode)
- ส่วนเบี่ยงเบนมาตรฐาน (standard deviation)

### ข้อ 14: String Split และ Join
เขียนฟังก์ชัน:
```pascal
function SplitString(s, delim: String): TStringArray;
function JoinStrings(arr: TStringArray; delim: String): String;
```

### ข้อ 15: Number Format
เขียนฟังก์ชัน `FormatNumber(n: Double): String` ที่แสดงตัวเลขพร้อม comma:
- 1234567.89 → "1,234,567.89"
- 1000000 → "1,000,000"

### ข้อ 16: Type Range Checker
เขียนฟังก์ชัน `IsInRange(value, min, max: Int64): Boolean` และทดสอบกับทุก Integer type

### ข้อ 17: Hexadecimal Converter
เขียนโปรแกรมแปลง:
- Decimal → Hexadecimal
- Hexadecimal → Decimal
- Binary → Decimal
- Decimal → Binary

### ข้อ 18: String Encryption (XOR)
เขียน XOR encryption:
```pascal
function XOREncrypt(s: String; key: Byte): String;
```
เข้ารหัสและถอดรหัส string โดยใช้ XOR กับ key

### ข้อ 19: Float Comparison Helper
เขียนฟังก์ชัน:
```pascal
function FloatEquals(a, b: Double; precision: Integer): Boolean;
// precision: 1 = ทศนิยม 1 ตำแหน่ง, 2 = 2 ตำแหน่ง, ฯลฯ
```

### ข้อ 20: Data Type Memory Report
เขียนโปรแกรมแสดงขนาด memory ของทุก type:
```pascal
// ใช้ SizeOf() function
WriteLn('SizeOf(Boolean) = ', SizeOf(Boolean));
WriteLn('SizeOf(Char) = ', SizeOf(Char));
...
```
แสดงผลในรูปแบบตารางสวยงาม

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **การประกาศตัวแปร** - var, scope, stack vs heap
2. **Integer Types** - Byte, ShortInt, SmallInt, Word, Integer, Cardinal, Int64, QWord
3. **Float Types** - Single, Double, Extended, Currency - ความแม่นยำต่างกัน
4. **Boolean** - True/False, operators AND/OR/NOT/XOR, short-circuit
5. **Char** - ASCII, Unicode, case conversion, character sets
6. **String Types** - ShortString, AnsiString, WideString, UnicodeString
7. **Subrange Types** - จำกัดช่วงค่าที่ถูกต้อง
8. **Type Casting** - Explicit และ Implicit conversion
9. **Constants** - const, typed constants

## บทต่อไป

**Part 05** จะอธิบาย Operators ทุกประเภท ทั้ง arithmetic, comparison, logical, bitwise และ operator precedence

---

*ทุกตัวอย่างทดสอบด้วย Free Pascal 3.2.x*
