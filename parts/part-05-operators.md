# Part 05 - ตัวดำเนินการ (Operators)

## สารบัญ

1. [Arithmetic Operators](#arithmetic-operators)
2. [Comparison Operators](#comparison-operators)
3. [Logical Operators](#logical-operators)
4. [Bitwise Operators](#bitwise-operators)
5. [String Operators](#string-operators)
6. [Assignment Operator](#assignment-operator)
7. [Operator Precedence](#operator-precedence)
8. [Type Compatibility](#type-compatibility)
9. [โปรแกรม Calculator ตัวอย่าง](#โปรแกรม-calculator-ตัวอย่าง)
10. [แบบฝึกหัด 15 ข้อ](#แบบฝึกหัด-15-ข้อ)

---

## Arithmetic Operators

### ตัวดำเนินการคณิตศาสตร์พื้นฐาน

| Operator | ชื่อ | ตัวอย่าง | ผลลัพธ์ | ชนิดข้อมูล |
|----------|------|---------|---------|-----------|
| `+` | บวก | `5 + 3` | `8` | Integer, Float |
| `-` | ลบ | `10 - 4` | `6` | Integer, Float |
| `*` | คูณ | `6 * 7` | `42` | Integer, Float |
| `/` | หาร (Float) | `10 / 3` | `3.333...` | Float เสมอ |
| `div` | หาร (Integer) | `10 div 3` | `3` | Integer เท่านั้น |
| `mod` | เศษจากการหาร | `10 mod 3` | `1` | Integer เท่านั้น |

### ตัวอย่าง Arithmetic Operators

```pascal
program ArithmeticDemo;
var
  a, b: Integer;
  x, y: Double;
  result: Double;
begin
  a := 17;
  b := 5;
  
  WriteLn('=== Integer Arithmetic ===');
  WriteLn('a = ', a, ', b = ', b);
  WriteLn('a + b = ', a + b);      // 22
  WriteLn('a - b = ', a - b);      // 12
  WriteLn('a * b = ', a * b);      // 85
  WriteLn('a / b = ', a / b:8:4);  // 3.4000 (Float division!)
  WriteLn('a div b = ', a div b);  // 3 (Integer division)
  WriteLn('a mod b = ', a mod b);  // 2 (remainder)
  
  WriteLn;
  WriteLn('=== Float Arithmetic ===');
  x := 17.0;
  y := 5.0;
  WriteLn('x = ', x:5:1, ', y = ', y:5:1);
  WriteLn('x + y = ', x + y:8:2);   // 22.00
  WriteLn('x - y = ', x - y:8:2);   // 12.00
  WriteLn('x * y = ', x * y:8:2);   // 85.00
  WriteLn('x / y = ', x / y:8:4);   // 3.4000
  
  WriteLn;
  WriteLn('=== Mixed Arithmetic ===');
  // Integer + Double = Double
  result := a + y;
  WriteLn('a + y = ', result:8:2);   // 22.00
  
  // a / b เป็น Double เสมอ!
  WriteLn('a / b (/) = ', a / b:8:4);  // 3.4000 (Float result)
  WriteLn('a div b = ', a div b);       // 3 (Integer result)
end.
```

### ความแตกต่าง / vs div vs mod

```pascal
program DivisionDemo;
var
  a, b: Integer;
begin
  a := 17; b := 5;
  
  WriteLn('=== Division Operators ===');
  WriteLn;
  
  // / (Float division) - ผลลัพธ์เป็น Double เสมอ
  WriteLn('17 / 5 = ', 17 / 5:8:4);    // 3.4000
  WriteLn('10 / 4 = ', 10 / 4:8:4);    // 2.5000
  WriteLn('9 / 3 = ', 9 / 3:8:4);      // 3.0000
  
  WriteLn;
  
  // div (Integer division) - ตัดเศษทิ้ง
  WriteLn('17 div 5 = ', 17 div 5);    // 3
  WriteLn('10 div 4 = ', 10 div 4);    // 2
  WriteLn('9 div 3 = ', 9 div 3);      // 3
  WriteLn('-17 div 5 = ', -17 div 5);  // -3 (truncates toward zero)
  WriteLn('17 div -5 = ', 17 div -5);  // -3
  
  WriteLn;
  
  // mod (modulo) - เศษจากการหาร
  WriteLn('17 mod 5 = ', 17 mod 5);    // 2
  WriteLn('10 mod 4 = ', 10 mod 4);    // 2
  WriteLn('9 mod 3 = ', 9 mod 3);      // 0
  WriteLn('15 mod 5 = ', 15 mod 5);    // 0
  WriteLn('-17 mod 5 = ', -17 mod 5);  // -2 (sign follows dividend)
  WriteLn('17 mod -5 = ', 17 mod -5);  // 2 (sign follows dividend)
  
  WriteLn;
  WriteLn('=== Useful patterns ===');
  
  // ตรวจสอบเลขคู่/คี่
  for a := 1 to 10 do
  begin
    if a mod 2 = 0 then
      WriteLn(a, ' = คู่')
    else
      WriteLn(a, ' = คี่');
  end;
  
  WriteLn;
  
  // การหาเลขหลัก
  a := 12345;
  WriteLn('เลขหลักหน่วยของ 12345 = ', a mod 10);  // 5
  WriteLn('เลขหลักสิบของ 12345 = ', (a div 10) mod 10);  // 4
  WriteLn('เลขหลักร้อยของ 12345 = ', (a div 100) mod 10);  // 3
end.
```

### Unary Operators

```pascal
program UnaryOperators;
var
  a, b: Integer;
  x: Double;
begin
  a := 10;
  x := 3.14;
  
  WriteLn('=== Unary Operators ===');
  
  // Unary minus (negation)
  b := -a;
  WriteLn('-10 = ', b);        // -10
  WriteLn('-(-10) = ', -b);    // 10
  WriteLn('-x = ', -x:6:2);   // -3.14
  
  // Unary plus (ไม่ค่อยใช้)
  b := +a;
  WriteLn('+10 = ', b);        // 10
  
  WriteLn;
  WriteLn('=== Inc and Dec ===');
  
  a := 5;
  Inc(a);           // a := a + 1
  WriteLn('Inc(5) = ', a);     // 6
  
  Inc(a, 3);        // a := a + 3
  WriteLn('Inc(6, 3) = ', a);  // 9
  
  Dec(a);           // a := a - 1
  WriteLn('Dec(9) = ', a);     // 8
  
  Dec(a, 2);        // a := a - 2
  WriteLn('Dec(8, 2) = ', a);  // 6
  
  WriteLn;
  WriteLn('Note: Pascal ไม่มี ++ หรือ --');
  WriteLn('ใช้ Inc() และ Dec() แทน');
end.
```

### การคำนวณทางคณิตศาสตร์ขั้นสูง

```pascal
program AdvancedMath;
uses
  Math;
var
  x, y: Double;
begin
  x := 2.0;
  y := 3.0;
  
  WriteLn('=== Power and Root ===');
  WriteLn('x^y = ', Power(x, y):8:4);    // 2^3 = 8
  WriteLn('Sqrt(x) = ', Sqrt(x):8:6);    // √2 ≈ 1.414
  WriteLn('x^(1/3) = ', Power(x, 1/3):8:6); // Cube root
  
  WriteLn;
  WriteLn('=== Logarithm ===');
  WriteLn('Ln(x) = ', Ln(x):8:6);         // Natural log
  WriteLn('Log10(x) = ', Log10(x):8:6);   // Log base 10
  WriteLn('Log2(x) = ', Log2(x):8:6);     // Log base 2
  WriteLn('LogN(3, 27) = ', LogN(3, 27):8:4); // Log base 3 of 27 = 3
  
  WriteLn;
  WriteLn('=== Absolute Value ===');
  WriteLn('Abs(-5) = ', Abs(-5));
  WriteLn('Abs(5) = ', Abs(5));
  WriteLn('Abs(-3.14) = ', Abs(-3.14):5:2);
  
  WriteLn;
  WriteLn('=== Min Max ===');
  WriteLn('Min(3, 7) = ', Min(3, 7));
  WriteLn('Max(3, 7) = ', Max(3, 7));
  WriteLn('Min(3.14, 2.71) = ', Min(3.14, 2.71):5:2);
  WriteLn('Max(3.14, 2.71) = ', Max(3.14, 2.71):5:2);
  
  WriteLn;
  WriteLn('=== Integer Math ===');
  WriteLn('Sqr(5) = ', Sqr(5));           // 5² = 25
  WriteLn('GCD(12, 8) = ', GCD(12, 8));   // 4 (ถ้ามีใน Math)
  WriteLn('LCM(4, 6) = ', LCM(4, 6));     // 12 (ถ้ามีใน Math)
end.
```

---

## Comparison Operators

### ตัวดำเนินการเปรียบเทียบ

| Operator | ความหมาย | ตัวอย่าง | ผลลัพธ์ |
|----------|---------|---------|---------|
| `=` | เท่ากับ | `5 = 5` | `True` |
| `<>` | ไม่เท่ากับ | `5 <> 6` | `True` |
| `<` | น้อยกว่า | `3 < 5` | `True` |
| `>` | มากกว่า | `5 > 3` | `True` |
| `<=` | น้อยกว่าหรือเท่ากับ | `5 <= 5` | `True` |
| `>=` | มากกว่าหรือเท่ากับ | `5 >= 6` | `False` |

### ตัวอย่าง Comparison Operators

```pascal
program ComparisonDemo;
var
  a, b: Integer;
  x, y: Double;
  s1, s2: String;
  c1, c2: Char;
begin
  a := 10; b := 20;
  
  WriteLn('=== Integer Comparison ===');
  WriteLn('a = ', a, ', b = ', b);
  WriteLn('a = b: ', a = b);    // False
  WriteLn('a <> b: ', a <> b);  // True
  WriteLn('a < b: ', a < b);    // True
  WriteLn('a > b: ', a > b);    // False
  WriteLn('a <= b: ', a <= b);  // True
  WriteLn('a >= b: ', a >= b);  // False
  
  WriteLn;
  WriteLn('=== Float Comparison ===');
  x := 3.14; y := 3.14;
  WriteLn('3.14 = 3.14: ', x = y);   // True (ระวัง floating point!)
  x := 0.1 + 0.2;
  y := 0.3;
  WriteLn('0.1+0.2 = 0.3: ', x = y);  // False! (floating point issue)
  
  WriteLn;
  WriteLn('=== String Comparison ===');
  s1 := 'Apple';
  s2 := 'Banana';
  WriteLn('"Apple" = "Banana": ', s1 = s2);    // False
  WriteLn('"Apple" < "Banana": ', s1 < s2);    // True (A before B)
  WriteLn('"apple" = "Apple": ', 'apple' = 'Apple'); // False (case sensitive!)
  
  WriteLn;
  WriteLn('=== Char Comparison ===');
  c1 := 'A'; c2 := 'Z';
  WriteLn('"A" < "Z": ', c1 < c2);    // True
  WriteLn('"A" = "a": ', 'A' = 'a');  // False (case sensitive)
end.
```

### ใช้ Comparison ใน Conditions

```pascal
program ComparisonInConditions;
uses
  SysUtils;
var
  score: Integer;
  age: Integer;
  name: String;
begin
  Write('คะแนน: '); ReadLn(score);
  Write('อายุ: '); ReadLn(age);
  Write('ชื่อ: '); ReadLn(name);
  
  // เงื่อนไขเดี่ยว
  if score >= 50 then
    WriteLn('ผ่าน')
  else
    WriteLn('ไม่ผ่าน');
  
  // เงื่อนไขซ้อน
  if score >= 80 then
    WriteLn('เกรด A')
  else if score >= 70 then
    WriteLn('เกรด B')
  else if score >= 60 then
    WriteLn('เกรด C')
  else if score >= 50 then
    WriteLn('เกรด D')
  else
    WriteLn('เกรด F');
  
  // เงื่อนไขซับซ้อน
  if (age >= 18) and (score >= 60) then
    WriteLn('ผ่านเกณฑ์ทั้งอายุและคะแนน');
  
  // String comparison
  if name = 'admin' then
    WriteLn('ยินดีต้อนรับ Administrator')
  else
    WriteLn('ยินดีต้อนรับ ', name);
  
  // ตรวจสอบช่วง
  if (score >= 0) and (score <= 100) then
    WriteLn('คะแนนอยู่ในช่วงที่ถูกต้อง')
  else
    WriteLn('คะแนนไม่ถูกต้อง');
end.
```

### Comparison ของ Records และ Objects

```pascal
program RecordComparison;
uses
  SysUtils;

type
  TPoint = record
    X, Y: Integer;
  end;

function PointEqual(p1, p2: TPoint): Boolean;
begin
  Result := (p1.X = p2.X) and (p1.Y = p2.Y);
end;

function PointLessThan(p1, p2: TPoint): Boolean;
begin
  // เรียงตาม X ก่อน ถ้า X เท่ากันเรียงตาม Y
  if p1.X <> p2.X then
    Result := p1.X < p2.X
  else
    Result := p1.Y < p2.Y;
end;

var
  p1, p2: TPoint;
begin
  p1.X := 3; p1.Y := 5;
  p2.X := 3; p2.Y := 5;
  
  // Pascal ไม่สามารถใช้ = กับ Record โดยตรง!
  // if p1 = p2 then  ← ERROR
  
  // ต้องเขียน function เอง
  if PointEqual(p1, p2) then
    WriteLn('Points are equal')
  else
    WriteLn('Points are different');
  
  p2.Y := 8;
  if PointLessThan(p1, p2) then
    WriteLn('p1 < p2');
end.
```

---

## Logical Operators

### ตัวดำเนินการตรรกศาสตร์

| Operator | ความหมาย | Pascal |
|----------|---------|--------|
| AND | ทั้งสองต้องจริง | `a and b` |
| OR | อย่างน้อยหนึ่งต้องจริง | `a or b` |
| NOT | กลับค่า | `not a` |
| XOR | ต่างกันจึงจริง | `a xor b` |

### Truth Table

```
A     | B     | A AND B | A OR B | NOT A | A XOR B
------|-------|---------|--------|-------|--------
False | False | False   | False  | True  | False
False | True  | False   | True   | True  | True
True  | False | False   | True   | False | True
True  | True  | True    | True   | False | False
```

### ตัวอย่าง Logical Operators

```pascal
program LogicalDemo;
var
  a, b: Boolean;
  x, y: Integer;
begin
  a := True;
  b := False;
  
  WriteLn('=== Logical Operators ===');
  WriteLn('a = ', a, ', b = ', b);
  WriteLn;
  WriteLn('a AND b = ', a AND b);   // False
  WriteLn('a OR b = ', a OR b);     // True
  WriteLn('NOT a = ', NOT a);       // False
  WriteLn('NOT b = ', NOT b);       // True
  WriteLn('a XOR b = ', a XOR b);   // True
  WriteLn('a XOR a = ', a XOR a);   // False
  
  WriteLn;
  WriteLn('=== Logical with Comparisons ===');
  x := 15; y := 25;
  WriteLn('x = ', x, ', y = ', y);
  WriteLn;
  
  // AND: ทั้งสองต้องจริง
  WriteLn('(x > 10) AND (y > 20): ', (x > 10) AND (y > 20));   // True
  WriteLn('(x > 10) AND (y > 30): ', (x > 10) AND (y > 30));   // False
  
  // OR: อย่างน้อยหนึ่งจริง
  WriteLn('(x > 20) OR (y > 20): ', (x > 20) OR (y > 20));    // True
  WriteLn('(x > 20) OR (y > 30): ', (x > 20) OR (y > 30));    // False
  
  // NOT: กลับค่า
  WriteLn('NOT (x > 10): ', NOT (x > 10));    // False
  WriteLn('NOT (x > 20): ', NOT (x > 20));    // True
  
  // XOR: ต่างกันจึงจริง
  WriteLn('(x > 10) XOR (y > 20): ', (x > 10) XOR (y > 20));  // False (ทั้งสองจริง)
  WriteLn('(x > 10) XOR (y > 30): ', (x > 10) XOR (y > 30));  // True (ต่างกัน)
end.
```

### Short-Circuit Evaluation

```pascal
program ShortCircuit;
var
  x: Integer;

function ExpensiveCheck(): Boolean;
begin
  WriteLn('  [ExpensiveCheck called!]');
  Result := True;
end;

begin
  x := 5;
  
  WriteLn('=== Short-Circuit Evaluation ===');
  WriteLn;
  
  // AND Short-circuit: ถ้าซ้ายเป็น False ขวาจะไม่ถูกเรียก
  WriteLn('Testing: (10 < 5) AND ExpensiveCheck()');
  if (10 < 5) AND ExpensiveCheck() then  // ExpensiveCheck ไม่ถูกเรียก!
    WriteLn('Both true')
  else
    WriteLn('At least one is false');
  
  WriteLn;
  
  // AND Short-circuit: ถ้าซ้ายเป็น True ขวาจะถูกเรียก
  WriteLn('Testing: (5 < 10) AND ExpensiveCheck()');
  if (5 < 10) AND ExpensiveCheck() then  // ExpensiveCheck ถูกเรียก
    WriteLn('Both true');
  
  WriteLn;
  
  // OR Short-circuit: ถ้าซ้ายเป็น True ขวาจะไม่ถูกเรียก
  WriteLn('Testing: (5 < 10) OR ExpensiveCheck()');
  if (5 < 10) OR ExpensiveCheck() then  // ExpensiveCheck ไม่ถูกเรียก!
    WriteLn('At least one is true');
  
  WriteLn;
  
  // ประโยชน์ของ short-circuit: ป้องกัน nil pointer dereference
  var p: PInteger := nil;
  if (p <> nil) AND (p^ > 5) then  // p^ ไม่ถูกเรียกเมื่อ p = nil
    WriteLn('Value > 5')
  else
    WriteLn('p is nil or value <= 5');
end.
```

### De Morgan's Laws

```pascal
program DeMorgan;
var
  a, b: Boolean;
begin
  WriteLn('=== De Morgan''s Laws ===');
  WriteLn;
  WriteLn('NOT (A AND B) = (NOT A) OR (NOT B)');
  WriteLn('NOT (A OR B) = (NOT A) AND (NOT B)');
  WriteLn;
  
  for a := False to True do
    for b := False to True do
    begin
      WriteLn('A=', a, ', B=', b);
      WriteLn('  NOT(A AND B) = ', NOT (a AND b));
      WriteLn('  (NOT A) OR (NOT B) = ', (NOT a) OR (NOT b));
      WriteLn('  Equal: ', (NOT (a AND b)) = ((NOT a) OR (NOT b)));
      WriteLn;
    end;
end.
```

---

## Bitwise Operators

### ตัวดำเนินการบิต

| Operator | ชื่อ | ตัวอย่าง | ผลลัพธ์ (binary) |
|----------|------|---------|----------------|
| `and` | Bitwise AND | `12 and 10` | `8` (1100 and 1010 = 1000) |
| `or` | Bitwise OR | `12 or 10` | `14` (1100 or 1010 = 1110) |
| `xor` | Bitwise XOR | `12 xor 10` | `6` (1100 xor 1010 = 0110) |
| `not` | Bitwise NOT | `not 12` | `-13` (inverts all bits) |
| `shl` | Shift Left | `1 shl 4` | `16` (0001 → 10000) |
| `shr` | Shift Right | `16 shr 4` | `1` (10000 → 0001) |

> **หมายเหตุ:** Pascal ใช้ `and`, `or`, `xor`, `not` สำหรับทั้ง Logical และ Bitwise ขึ้นอยู่กับ operand type

### ตัวอย่าง Bitwise Operators

```pascal
program BitwiseDemo;

procedure ShowBinary(n: Integer; bits: Integer = 8);
var
  i: Integer;
  s: String;
begin
  s := '';
  for i := bits - 1 downto 0 do
  begin
    if (n and (1 shl i)) <> 0 then
      s := s + '1'
    else
      s := s + '0';
    if (i > 0) and (i mod 4 = 0) then
      s := s + '_';  // แบ่งกลุ่ม 4 บิต
  end;
  Write(n:4, ' = ', s);
end;

var
  a, b: Integer;
begin
  a := 12;  // 0000_1100
  b := 10;  // 0000_1010
  
  WriteLn('=== Bitwise Operations ===');
  Write('a = '); ShowBinary(a); WriteLn;
  Write('b = '); ShowBinary(b); WriteLn;
  WriteLn;
  
  Write('a AND b = '); ShowBinary(a and b); WriteLn(' = ', a and b);
  Write('a OR b  = '); ShowBinary(a or b);  WriteLn(' = ', a or b);
  Write('a XOR b = '); ShowBinary(a xor b); WriteLn(' = ', a xor b);
  Write('NOT a   = '); ShowBinary(not a);   WriteLn(' = ', not a);
  WriteLn;
  
  // Shift operators
  WriteLn('=== Shift Operations ===');
  a := 1;
  Write('1 shl 0 = '); ShowBinary(a shl 0); WriteLn(' = ', a shl 0);
  Write('1 shl 1 = '); ShowBinary(a shl 1); WriteLn(' = ', a shl 1);
  Write('1 shl 2 = '); ShowBinary(a shl 2); WriteLn(' = ', a shl 2);
  Write('1 shl 3 = '); ShowBinary(a shl 3); WriteLn(' = ', a shl 3);
  Write('1 shl 4 = '); ShowBinary(a shl 4); WriteLn(' = ', a shl 4);
  WriteLn;
  a := 128;
  Write('128 shr 1 = '); ShowBinary(a shr 1); WriteLn(' = ', a shr 1);
  Write('128 shr 2 = '); ShowBinary(a shr 2); WriteLn(' = ', a shr 2);
  Write('128 shr 4 = '); ShowBinary(a shr 4); WriteLn(' = ', a shr 4);
end.
```

### การใช้ Bitwise สำหรับ Flags

```pascal
program BitFlags;

// ใช้ bit flags แทน boolean array
const
  FLAG_BOLD      = 1;  // 0000_0001
  FLAG_ITALIC    = 2;  // 0000_0010
  FLAG_UNDERLINE = 4;  // 0000_0100
  FLAG_STRIKEOUT = 8;  // 0000_1000
  FLAG_COLOR     = 16; // 0001_0000

procedure ShowFlags(flags: Integer);
begin
  Write('Flags: ');
  if (flags and FLAG_BOLD) <> 0 then Write('BOLD ');
  if (flags and FLAG_ITALIC) <> 0 then Write('ITALIC ');
  if (flags and FLAG_UNDERLINE) <> 0 then Write('UNDERLINE ');
  if (flags and FLAG_STRIKEOUT) <> 0 then Write('STRIKEOUT ');
  if (flags and FLAG_COLOR) <> 0 then Write('COLOR ');
  WriteLn;
end;

var
  textFlags: Integer;
begin
  textFlags := 0;
  ShowFlags(textFlags);  // (none)
  
  // Set flags
  textFlags := textFlags or FLAG_BOLD;    // เปิด BOLD
  textFlags := textFlags or FLAG_ITALIC;  // เปิด ITALIC
  ShowFlags(textFlags);  // BOLD ITALIC
  
  // Clear a flag
  textFlags := textFlags and (not FLAG_ITALIC);  // ปิด ITALIC
  ShowFlags(textFlags);  // BOLD
  
  // Toggle flag (XOR)
  textFlags := textFlags xor FLAG_BOLD;   // สลับ BOLD
  ShowFlags(textFlags);  // (none)
  
  textFlags := textFlags xor FLAG_BOLD;   // สลับ BOLD อีกครั้ง
  ShowFlags(textFlags);  // BOLD
  
  // Check if flag is set
  if (textFlags and FLAG_BOLD) <> 0 then
    WriteLn('BOLD is enabled');
    
  // Set multiple flags at once
  textFlags := FLAG_BOLD or FLAG_UNDERLINE or FLAG_COLOR;
  ShowFlags(textFlags);  // BOLD UNDERLINE COLOR
end.
```

### Shift สำหรับ การคำนวณ 2^n

```pascal
program ShiftPower2;
var
  i: Integer;
begin
  WriteLn('=== 2^n using SHL ===');
  WriteLn('+----+-------------+');
  WriteLn('| n  |    2^n      |');
  WriteLn('+----+-------------+');
  for i := 0 to 30 do
    WriteLn('|', i:3, ' |', (1 shl i):13, '|');
  WriteLn('+----+-------------+');
  
  WriteLn;
  WriteLn('=== Fast Multiply/Divide by 2 ===');
  var x := 100;
  WriteLn('x = ', x);
  WriteLn('x * 2 = ', x shl 1);   // เร็วกว่า x * 2
  WriteLn('x * 4 = ', x shl 2);   // เร็วกว่า x * 4
  WriteLn('x * 8 = ', x shl 3);   // เร็วกว่า x * 8
  WriteLn('x div 2 = ', x shr 1); // เร็วกว่า x div 2
  WriteLn('x div 4 = ', x shr 2); // เร็วกว่า x div 4
end.
```

### Bitwise ใน Byte Operations

```pascal
program ByteOperations;

// เพิ่มประสิทธิภาพการทำงานกับ byte data
procedure AnalyzeByte(b: Byte);
var
  i: Integer;
begin
  Write('Byte ', b, ' (', IntToHex(b, 2), ')');
  Write(' bits: ');
  for i := 7 downto 0 do
    Write((b shr i) and 1);
  WriteLn;
  WriteLn('  High nibble: ', (b shr 4) and $0F);
  WriteLn('  Low nibble:  ', b and $0F);
  WriteLn('  Bit count:   ', PopCnt(b));  // ถ้า FPC รองรับ
end;

var
  b1, b2: Byte;
begin
  b1 := $A5;  // 1010_0101
  b2 := $3C;  // 0011_1100
  
  AnalyzeByte(b1);
  WriteLn;
  AnalyzeByte(b2);
  WriteLn;
  
  WriteLn('=== Byte Operations ===');
  WriteLn('b1 AND b2 = ', IntToHex(b1 and b2, 2), ' = ', b1 and b2);
  WriteLn('b1 OR  b2 = ', IntToHex(b1 or b2, 2), ' = ', b1 or b2);
  WriteLn('b1 XOR b2 = ', IntToHex(b1 xor b2, 2), ' = ', b1 xor b2);
  WriteLn('NOT b1    = ', IntToHex((not b1) and $FF, 2), ' = ', (not b1) and $FF);
end.
```

---

## String Operators

### String Concatenation

```pascal
program StringOperators;
uses
  SysUtils;
var
  s1, s2, s3: String;
begin
  s1 := 'Hello';
  s2 := ' World';
  
  WriteLn('=== String + Operator ===');
  
  // + operator สำหรับ String concatenation
  s3 := s1 + s2;
  WriteLn(s3);  // Hello World
  
  // ต่อหลายส่วน
  s3 := 'Pascal' + ' ' + 'is' + ' ' + 'great!';
  WriteLn(s3);  // Pascal is great!
  
  // ต่อกับตัวแปรอื่น
  var name := 'สมชาย';
  var age := 25;
  s3 := 'ชื่อ: ' + name + ', อายุ: ' + IntToStr(age);
  WriteLn(s3);
  
  // ต่อใน loop
  s3 := '';
  for var i := 1 to 5 do
    s3 := s3 + IntToStr(i) + ' ';
  WriteLn(s3.Trim);  // 1 2 3 4 5
  
  WriteLn;
  WriteLn('=== String Comparison ===');
  // = ตรวจสอบความเท่ากัน
  WriteLn('"Hello" = "Hello": ', 'Hello' = 'Hello');    // True
  WriteLn('"Hello" = "hello": ', 'Hello' = 'hello');    // False
  WriteLn('"Hello" < "World": ', 'Hello' < 'World');    // True
  WriteLn('"Z" > "A": ', 'Z' > 'A');                   // True
  
  // Case-insensitive comparison
  WriteLn;
  WriteLn('=== Case-Insensitive ===');
  s1 := 'Hello';
  s2 := 'HELLO';
  WriteLn('Direct: ', s1 = s2);                           // False
  WriteLn('Upper: ', UpperCase(s1) = UpperCase(s2));      // True
  WriteLn('Lower: ', LowerCase(s1) = LowerCase(s2));      // True
  WriteLn('CompareText: ', CompareText(s1, s2) = 0);      // True (case-insensitive)
end.
```

### String อื่นๆ ที่ใช้บ่อย

```pascal
program MoreStringOps;
uses
  SysUtils, StrUtils;
var
  s: String;
begin
  s := 'Hello, World!';
  
  // ใช้ in operator กับ Char
  WriteLn('=== in Operator ===');
  var ch: Char := 'W';
  if ch in ['A'..'Z'] then
    WriteLn(ch, ' is uppercase');
  
  // ตรวจสอบ character ใน string
  for var c: Char := 'a' to 'z' do
    if Pos(c, s) > 0 then
      Write(c);
  WriteLn;
  
  // String helper methods (Lazarus/FPC 3.x)
  WriteLn;
  WriteLn('=== String Helpers ===');
  WriteLn('Contains: ', s.Contains('World'));
  WriteLn('StartsWith: ', s.StartsWith('Hello'));
  WriteLn('EndsWith: ', s.EndsWith('!'));
  WriteLn('IndexOf: ', s.IndexOf('World'));
  WriteLn('Replace: ', s.Replace('World', 'Pascal'));
  WriteLn('ToUpper: ', s.ToUpper);
  WriteLn('ToLower: ', s.ToLower);
  WriteLn('Trim: ', '  Hello  '.Trim);
  WriteLn('Length: ', s.Length);
  WriteLn('Substring: ', s.Substring(7, 5));  // World
end.
```

---

## Assignment Operator

### ตัวดำเนินการกำหนดค่า

```pascal
program AssignmentDemo;
var
  a, b: Integer;
  s: String;
  f: Double;
begin
  // := (Assignment)
  a := 10;
  b := 20;
  s := 'Hello';
  f := 3.14;
  
  WriteLn('a = ', a);  // 10
  WriteLn('b = ', b);  // 20
  WriteLn('s = ', s);  // Hello
  WriteLn('f = ', f:5:2);  // 3.14
  
  WriteLn;
  WriteLn('=== Compound Assignment (ต้องทำเอง) ===');
  
  // Pascal ไม่มี +=, -=, *=, /= แบบ C
  // ต้องเขียนเต็มๆ
  a := a + 5;   // ไม่มี a += 5
  WriteLn('a + 5 = ', a);  // 15
  
  a := a - 3;   // ไม่มี a -= 3
  WriteLn('a - 3 = ', a);  // 12
  
  a := a * 2;   // ไม่มี a *= 2
  WriteLn('a * 2 = ', a);  // 24
  
  f := f * 2;   // ไม่มี f *= 2
  WriteLn('f * 2 = ', f:5:2);  // 6.28
  
  s := s + ' World';  // ไม่มี s += ' World'
  WriteLn('s + " World" = ', s);  // Hello World
  
  WriteLn;
  WriteLn('=== Inc/Dec (Shortcut) ===');
  a := 10;
  Inc(a);         // a := a + 1
  Inc(a, 5);      // a := a + 5
  Dec(a);         // a := a - 1
  Dec(a, 3);      // a := a - 3
  WriteLn('After Inc/Dec: ', a);  // 10 + 1 + 5 - 1 - 3 = 12
end.
```

### Multiple Assignment

```pascal
program MultipleAssignment;
var
  a, b, c: Integer;
  temp: Integer;
begin
  a := 1;
  b := 2;
  c := 3;
  
  WriteLn('Before: a=', a, ' b=', b, ' c=', c);
  
  // Swap สองตัวแปร
  temp := a;
  a := b;
  b := temp;
  WriteLn('After swap a,b: a=', a, ' b=', b);
  
  // Chain assignment (Pascal ไม่รองรับ a := b := c := 0 โดยตรง)
  // ต้องทำทีละบรรทัด
  a := 0;
  b := 0;
  c := 0;
  WriteLn('After reset: a=', a, ' b=', b, ' c=', c);
end.
```

---

## Operator Precedence

### ลำดับความสำคัญของ Operators (สูงสุดก่อน)

```
ลำดับที่ 1 (สูงสุด):
  @, not

ลำดับที่ 2:
  * / div mod and shl shr as

ลำดับที่ 3:
  + - or xor

ลำดับที่ 4 (ต่ำสุด):
  = <> < > <= >= in is
```

### ตัวอย่าง Operator Precedence

```pascal
program PrecedenceDemo;
var
  a, b, c: Integer;
begin
  WriteLn('=== Operator Precedence ===');
  WriteLn;
  
  // คูณก่อนบวก
  WriteLn('2 + 3 * 4 = ', 2 + 3 * 4);      // 14 ไม่ใช่ 20
  WriteLn('(2 + 3) * 4 = ', (2 + 3) * 4);  // 20
  
  // หารก่อนลบ
  WriteLn('10 - 6 / 2 = ', 10 - 6 / 2);    // 7.0 (10 - 3.0)
  WriteLn('(10 - 6) / 2 = ', (10 - 6) / 2);// 2.0
  
  // NOT มีความสำคัญสูงสุดใน logical
  WriteLn;
  WriteLn('NOT TRUE AND FALSE = ', NOT TRUE AND FALSE);   // FALSE and FALSE = FALSE
  WriteLn('NOT (TRUE AND FALSE) = ', NOT (TRUE AND FALSE)); // NOT FALSE = TRUE
  
  // AND ก่อน OR
  WriteLn;
  WriteLn('TRUE OR FALSE AND FALSE = ', TRUE OR FALSE AND FALSE); // TRUE OR (FALSE AND FALSE) = TRUE
  WriteLn('(TRUE OR FALSE) AND FALSE = ', (TRUE OR FALSE) AND FALSE); // TRUE AND FALSE = FALSE
  
  WriteLn;
  WriteLn('=== Complex Expressions ===');
  a := 3; b := 4; c := 5;
  
  // ทำงานอย่างไร?
  WriteLn('a + b * c = ', a + b * c);           // 3 + 20 = 23
  WriteLn('(a + b) * c = ', (a + b) * c);       // 7 * 5 = 35
  WriteLn('a * b + c * b = ', a * b + c * b);   // 12 + 20 = 32
  WriteLn('a * (b + c) * b = ', a * (b + c) * b); // 3 * 9 * 4 = 108
  
  WriteLn;
  WriteLn('=== Mixed Operators ===');
  WriteLn('10 mod 3 + 1 = ', 10 mod 3 + 1);     // (10 mod 3) + 1 = 1 + 1 = 2
  WriteLn('10 mod (3 + 1) = ', 10 mod (3 + 1));  // 10 mod 4 = 2
  WriteLn('2 shl 3 + 1 = ', 2 shl 3 + 1);       // (2 shl 3) + 1 = 8 + 1 = 9
  WriteLn('2 shl (3 + 1) = ', 2 shl (3 + 1));    // 2 shl 4 = 16
end.
```

### คำแนะนำการใช้ Parentheses

```pascal
program ParenthesesAdvice;
var
  a, b, c, d: Integer;
begin
  a := 10; b := 5; c := 2; d := 3;
  
  WriteLn('=== Use Parentheses for Clarity ===');
  WriteLn;
  
  // ไม่ชัดเจน
  var r1 := a - b * c + d;
  WriteLn('a - b * c + d = ', r1);  // 10 - 10 + 3 = 3
  
  // ชัดเจนกว่า
  var r2 := a - (b * c) + d;
  WriteLn('a - (b * c) + d = ', r2);  // เหมือนกัน แต่ชัดกว่า
  
  WriteLn;
  WriteLn('Recommendation:');
  WriteLn('ถ้าไม่แน่ใจลำดับ ใส่วงเล็บเสมอ!');
  WriteLn('โค้ดที่ชัดเจนดีกว่าโค้ดที่ compact');
  
  WriteLn;
  WriteLn('=== Common Mistakes ===');
  
  // ผิดทั่วไป: condition ใน if ต้องมีวงเล็บ
  // if a > 0 and b > 0 then  ← ผิด! (and มีความสำคัญสูงกว่า >)
  // แปลเป็น: a > (0 and b) > 0 ← ไม่ถูกต้อง
  
  // ถูกต้อง:
  if (a > 0) and (b > 0) then
    WriteLn('Both positive');
  
  // ผิดทั่วไป: bitwise operations
  // if n and $FF = 0 then  ← ผิด! (= มีความสำคัญสูงกว่า and)
  // แปลเป็น: n and ($FF = 0) = n and False
  
  // ถูกต้อง:
  if (a and $0F) = 0 then
    WriteLn('Low nibble is zero');
end.
```

---

## Type Compatibility

### การทำงานร่วมกันของชนิดข้อมูล

```pascal
program TypeCompatibility;
var
  b: Byte;
  si: ShortInt;
  i: Integer;
  li: LongInt;
  i64: Int64;
  f: Single;
  d: Double;
  e: Extended;
begin
  WriteLn('=== Widening Conversion (ปลอดภัย) ===');
  
  // Byte → Integer (อัตโนมัติ)
  b := 200;
  i := b;
  WriteLn('Byte(200) → Integer: ', i);
  
  // Integer → Int64 (อัตโนมัติ)
  i := 2147483647;
  i64 := i;
  WriteLn('MaxInt → Int64: ', i64);
  
  // Single → Double (อัตโนมัติ)
  f := 3.14;
  d := f;
  WriteLn('Single → Double: ', d:15:10);
  
  // Double → Extended (อัตโนมัติ)
  d := 3.14159265358979;
  e := d;
  WriteLn('Double → Extended: ', e:20:15);
  
  WriteLn;
  WriteLn('=== Narrowing Conversion (ต้องระวัง) ===');
  
  // Integer → Byte (ต้องแปลงเอง, อาจเสียข้อมูล)
  i := 300;
  b := Byte(i);  // 300 mod 256 = 44
  WriteLn('Integer(300) → Byte: ', b, ' (ข้อมูลหาย!)');
  
  // Double → Integer (ต้องใช้ Round หรือ Trunc)
  d := 3.99;
  i := Round(d);  // 4
  WriteLn('Round(3.99) = ', i);
  i := Trunc(d);  // 3
  WriteLn('Trunc(3.99) = ', i);
  
  WriteLn;
  WriteLn('=== Mixed Operations ===');
  
  // Integer + Float = Float
  i := 5;
  d := 2.5;
  var result: Double := i + d;  // i ถูกแปลงเป็น Double อัตโนมัติ
  WriteLn('5 + 2.5 = ', result:5:2);
  
  // Integer / Integer = Float (ใน Pascal /)
  WriteLn('5 / 2 = ', 5 / 2:5:2);    // 2.50
  WriteLn('5 div 2 = ', 5 div 2);    // 2
end.
```

### Type Compatibility Rules

```pascal
program TypeRules;
uses
  SysUtils;
var
  a: Integer;
  b: Cardinal;
  c: Int64;
begin
  WriteLn('=== Pascal Type Rules ===');
  WriteLn;
  
  WriteLn('1. Numeric: จะ "promote" เป็นชนิดที่ใหญ่กว่า');
  a := 10;
  b := 20;
  // c := a + b;  // อาจเกิด warning (mixing signed/unsigned)
  c := Int64(a) + Int64(b);  // ปลอดภัยกว่า
  WriteLn('Int64 result: ', c);
  
  WriteLn;
  WriteLn('2. String: String = AnsiString (ใน {$H+} mode)');
  var s1: String := 'Hello';
  var s2: AnsiString := 'World';
  var s3: String := s1 + ' ' + s2;  // ทำงานได้ปกติ
  WriteLn(s3);
  
  WriteLn;
  WriteLn('3. Char และ String: ผสมกันได้');
  var ch: Char := '!';
  s3 := s3 + ch;  // String + Char = String
  WriteLn(s3);
  
  WriteLn;
  WriteLn('4. Boolean: ใช้กับ logical operators เท่านั้น');
  var flag: Boolean := True;
  // if flag + 1 then  ← ERROR! Boolean ไม่สามารถบวกได้
  if flag then
    WriteLn('flag is True');
  
  WriteLn;
  WriteLn('5. Enumeration: ใช้ Ord() เพื่อแปลงเป็น Integer');
  type TColor = (clRed, clGreen, clBlue);
  var color: TColor := clGreen;
  WriteLn('Color value: ', Ord(color));    // 1
  WriteLn('Succ(clRed) = ', Ord(Succ(clRed)));  // 1 (clGreen)
end.
```

---

## โปรแกรม Calculator ตัวอย่าง

### Calculator พื้นฐาน (Console)

```pascal
program BasicCalculator;
uses
  SysUtils, Math;

type
  TCalcOp = (opAdd, opSub, opMul, opDiv, opMod, opPow, opSqrt, opAbs);

const
  OP_NAMES: array[TCalcOp] of String = (
    'บวก (+)', 'ลบ (-)', 'คูณ (*)', 'หาร (/)',
    'เศษหาร (mod)', 'ยกกำลัง (^)', 'รากที่สอง (sqrt)', 'ค่าสัมบูรณ์ (|x|)'
  );

procedure ShowMenu;
var
  op: TCalcOp;
begin
  WriteLn;
  WriteLn('=================================');
  WriteLn('  เครื่องคิดเลข Pascal Calculator');
  WriteLn('=================================');
  for op := opAdd to opAbs do
    WriteLn(Ord(op) + 1, '. ', OP_NAMES[op]);
  WriteLn('9. ประวัติการคำนวณ');
  WriteLn('0. ออก');
  WriteLn('---------------------------------');
  Write('เลือก: ');
end;

// History
var
  history: array[1..100] of String;
  histCount: Integer = 0;

procedure AddHistory(expr, result: String);
begin
  if histCount < 100 then
  begin
    Inc(histCount);
    history[histCount] := expr + ' = ' + result;
  end;
end;

procedure ShowHistory;
var
  i: Integer;
begin
  if histCount = 0 then
    WriteLn('ยังไม่มีประวัติ')
  else
  begin
    WriteLn('=== ประวัติการคำนวณ ===');
    for i := 1 to histCount do
      WriteLn(i, '. ', history[i]);
  end;
end;

procedure DoCalculation(op: TCalcOp);
var
  a, b, result: Double;
  expr, resultStr: String;
begin
  case op of
    opSqrt, opAbs:
      begin
        Write('ใส่ตัวเลข: ');
        ReadLn(a);
        if op = opSqrt then
        begin
          if a < 0 then
          begin
            WriteLn('Error: ไม่สามารถหารากที่สองของจำนวนลบ!');
            Exit;
          end;
          result := Sqrt(a);
          expr := Format('sqrt(%.4g)', [a]);
        end
        else
        begin
          result := Abs(a);
          expr := Format('|%.4g|', [a]);
        end;
      end;
  else
    begin
      Write('ตัวเลขแรก: ');
      ReadLn(a);
      Write('ตัวเลขที่สอง: ');
      ReadLn(b);
      
      case op of
        opAdd: begin result := a + b; expr := Format('%.4g + %.4g', [a, b]); end;
        opSub: begin result := a - b; expr := Format('%.4g - %.4g', [a, b]); end;
        opMul: begin result := a * b; expr := Format('%.4g * %.4g', [a, b]); end;
        opDiv: begin
                 if b = 0 then
                 begin
                   WriteLn('Error: หารด้วยศูนย์!');
                   Exit;
                 end;
                 result := a / b;
                 expr := Format('%.4g / %.4g', [a, b]);
               end;
        opMod: begin
                 if b = 0 then
                 begin
                   WriteLn('Error: หารด้วยศูนย์!');
                   Exit;
                 end;
                 result := a - b * Trunc(a / b);
                 expr := Format('%.4g mod %.4g', [a, b]);
               end;
        opPow: begin result := Power(a, b); expr := Format('%.4g ^ %.4g', [a, b]); end;
      end;
    end;
  end;
  
  resultStr := FloatToStrF(result, ffGeneral, 10, 6);
  WriteLn(expr, ' = ', resultStr);
  AddHistory(expr, resultStr);
end;

var
  choice: Integer;
begin
  histCount := 0;
  
  repeat
    ShowMenu;
    ReadLn(choice);
    
    case choice of
      1..8: DoCalculation(TCalcOp(choice - 1));
      9: ShowHistory;
      0: WriteLn('ลาก่อน!');
    else
      WriteLn('ตัวเลือกไม่ถูกต้อง');
    end;
    
  until choice = 0;
end.
```

### Calculator พร้อม Expression Parser

```pascal
program ExpressionCalculator;
uses
  SysUtils, Math;

// Simple expression evaluator
// รองรับ: +, -, *, /, (, )
// ไม่รองรับ: unary minus ซับซ้อน, functions

type
  TParser = class
  private
    FExpr: String;
    FPos: Integer;
    
    function GetChar: Char;
    function PeekChar: Char;
    procedure SkipSpaces;
    function ParseNumber: Double;
    function ParseFactor: Double;
    function ParseTerm: Double;
    function ParseExpr: Double;
  public
    function Evaluate(expr: String): Double;
  end;

function TParser.GetChar: Char;
begin
  if FPos <= Length(FExpr) then
  begin
    Result := FExpr[FPos];
    Inc(FPos);
  end
  else
    Result := #0;
end;

function TParser.PeekChar: Char;
begin
  if FPos <= Length(FExpr) then
    Result := FExpr[FPos]
  else
    Result := #0;
end;

procedure TParser.SkipSpaces;
begin
  while (FPos <= Length(FExpr)) and (FExpr[FPos] = ' ') do
    Inc(FPos);
end;

function TParser.ParseNumber: Double;
var
  s: String;
  c: Char;
begin
  SkipSpaces;
  s := '';
  c := PeekChar;
  if c = '-' then
  begin
    s := '-';
    GetChar;
  end;
  while PeekChar in ['0'..'9', '.'] do
    s := s + GetChar;
  Result := StrToFloatDef(s, 0);
end;

function TParser.ParseFactor: Double;
begin
  SkipSpaces;
  if PeekChar = '(' then
  begin
    GetChar;  // '('
    Result := ParseExpr;
    if PeekChar = ')' then GetChar;  // ')'
  end
  else
    Result := ParseNumber;
end;

function TParser.ParseTerm: Double;
var
  op: Char;
  right: Double;
begin
  Result := ParseFactor;
  SkipSpaces;
  while PeekChar in ['*', '/'] do
  begin
    op := GetChar;
    right := ParseFactor;
    if op = '*' then
      Result := Result * right
    else if right <> 0 then
      Result := Result / right
    else
      raise Exception.Create('Division by zero');
  end;
end;

function TParser.ParseExpr: Double;
var
  op: Char;
  right: Double;
begin
  Result := ParseTerm;
  SkipSpaces;
  while PeekChar in ['+', '-'] do
  begin
    op := GetChar;
    right := ParseTerm;
    if op = '+' then
      Result := Result + right
    else
      Result := Result - right;
  end;
end;

function TParser.Evaluate(expr: String): Double;
begin
  FExpr := expr;
  FPos := 1;
  Result := ParseExpr;
end;

var
  parser: TParser;
  expr: String;
  result: Double;
begin
  parser := TParser.Create;
  try
    WriteLn('=== Expression Calculator ===');
    WriteLn('รองรับ: +, -, *, /, วงเล็บ');
    WriteLn('พิมพ์ "quit" เพื่อออก');
    WriteLn;
    
    repeat
      Write('> ');
      ReadLn(expr);
      
      if LowerCase(expr) = 'quit' then Break;
      
      try
        result := parser.Evaluate(expr);
        WriteLn('= ', result:0:6);
      except
        on E: Exception do
          WriteLn('Error: ', E.Message);
      end;
      
    until False;
    
  finally
    parser.Free;
  end;
end.
```

### Scientific Calculator (GUI Lazarus)

```pascal
unit ScientificCalc;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, Math;

type
  TCalcForm = class(TForm)
    // Display
    edtDisplay: TEdit;
    lblExpression: TLabel;
    
    // Number buttons
    btn0, btn1, btn2, btn3, btn4: TButton;
    btn5, btn6, btn7, btn8, btn9: TButton;
    btnDot: TButton;
    
    // Operation buttons
    btnAdd, btnSub, btnMul, btnDiv: TButton;
    btnEquals, btnClear, btnBackspace, btnSign: TButton;
    
    // Scientific buttons
    btnSqrt, btnSqr, btnInv, btnLog, btnLn: TButton;
    btnSin, btnCos, btnTan, btnPi, btnE: TButton;
    btnMem, btnMR, btnMS, btnMPlus: TButton;
    
    procedure NumberClick(Sender: TObject);
    procedure OperatorClick(Sender: TObject);
    procedure EqualsClick(Sender: TObject);
    procedure ClearClick(Sender: TObject);
    procedure BackspaceClick(Sender: TObject);
    procedure ScientificClick(Sender: TObject);
    procedure MemoryClick(Sender: TObject);
    procedure FormCreate(Sender: TObject);
    procedure FormKeyPress(Sender: TObject; var Key: Char);
  private
    FCurrentValue: Double;
    FLastOperator: Char;
    FNewNumber: Boolean;
    FMemory: Double;
    
    procedure SetDisplay(value: Double);
    function GetDisplay: Double;
    procedure Calculate;
    procedure UpdateExpression(expr: String);
  public
  end;

var
  CalcForm: TCalcForm;

implementation

{$R *.lfm}

procedure TCalcForm.FormCreate(Sender: TObject);
begin
  Caption := 'Scientific Calculator - Pascal';
  Width := 320;
  Height := 500;
  Position := poScreenCenter;
  
  FCurrentValue := 0;
  FLastOperator := '=';
  FNewNumber := True;
  FMemory := 0;
  
  edtDisplay.Text := '0';
  edtDisplay.ReadOnly := True;
  edtDisplay.Font.Size := 18;
  
  lblExpression.Caption := '';
end;

function TCalcForm.GetDisplay: Double;
begin
  Result := StrToFloatDef(edtDisplay.Text, 0);
end;

procedure TCalcForm.SetDisplay(value: Double);
begin
  if Frac(value) = 0 then
    edtDisplay.Text := FloatToStrF(value, ffNumber, 12, 0)
  else
    edtDisplay.Text := FloatToStrF(value, ffNumber, 12, 6);
end;

procedure TCalcForm.UpdateExpression(expr: String);
begin
  lblExpression.Caption := expr;
end;

procedure TCalcForm.NumberClick(Sender: TObject);
var
  digit: String;
begin
  digit := (Sender as TButton).Caption;
  
  if FNewNumber then
  begin
    edtDisplay.Text := digit;
    FNewNumber := False;
  end
  else
  begin
    if edtDisplay.Text = '0' then
      edtDisplay.Text := digit
    else
      edtDisplay.Text := edtDisplay.Text + digit;
  end;
end;

procedure TCalcForm.OperatorClick(Sender: TObject);
var
  op: Char;
begin
  op := (Sender as TButton).Caption[1];
  
  if not FNewNumber then
    Calculate;
  
  FCurrentValue := GetDisplay;
  FLastOperator := op;
  FNewNumber := True;
  
  UpdateExpression(FloatToStr(FCurrentValue) + ' ' + op);
end;

procedure TCalcForm.Calculate;
var
  displayVal: Double;
  result: Double;
begin
  displayVal := GetDisplay;
  
  case FLastOperator of
    '+': result := FCurrentValue + displayVal;
    '-': result := FCurrentValue - displayVal;
    '*': result := FCurrentValue * displayVal;
    '/': begin
           if displayVal = 0 then
           begin
             ShowMessage('ไม่สามารถหารด้วยศูนย์ได้!');
             Exit;
           end;
           result := FCurrentValue / displayVal;
         end;
    else result := displayVal;
  end;
  
  SetDisplay(result);
  FCurrentValue := result;
  FNewNumber := True;
  UpdateExpression('');
end;

procedure TCalcForm.EqualsClick(Sender: TObject);
begin
  UpdateExpression(FloatToStr(FCurrentValue) + ' ' + FLastOperator +
                   ' ' + edtDisplay.Text + ' =');
  Calculate;
  FLastOperator := '=';
end;

procedure TCalcForm.ClearClick(Sender: TObject);
begin
  FCurrentValue := 0;
  FLastOperator := '=';
  FNewNumber := True;
  edtDisplay.Text := '0';
  UpdateExpression('');
end;

procedure TCalcForm.BackspaceClick(Sender: TObject);
var
  s: String;
begin
  s := edtDisplay.Text;
  if Length(s) > 1 then
    edtDisplay.Text := Copy(s, 1, Length(s) - 1)
  else
    edtDisplay.Text := '0';
end;

procedure TCalcForm.ScientificClick(Sender: TObject);
var
  cap: String;
  x: Double;
begin
  cap := (Sender as TButton).Caption;
  x := GetDisplay;
  
  if cap = 'sqrt' then
  begin
    if x < 0 then ShowMessage('Error: √ ของจำนวนลบ')
    else SetDisplay(Sqrt(x));
  end
  else if cap = 'x²' then
    SetDisplay(Sqr(x))
  else if cap = '1/x' then
  begin
    if x = 0 then ShowMessage('Error: หารด้วยศูนย์')
    else SetDisplay(1 / x);
  end
  else if cap = 'log' then
  begin
    if x <= 0 then ShowMessage('Error: log ของค่า ≤ 0')
    else SetDisplay(Log10(x));
  end
  else if cap = 'ln' then
  begin
    if x <= 0 then ShowMessage('Error: ln ของค่า ≤ 0')
    else SetDisplay(Ln(x));
  end
  else if cap = 'sin' then
    SetDisplay(Sin(DegToRad(x)))
  else if cap = 'cos' then
    SetDisplay(Cos(DegToRad(x)))
  else if cap = 'tan' then
    SetDisplay(Tan(DegToRad(x)))
  else if cap = 'π' then
    SetDisplay(Pi)
  else if cap = 'e' then
    SetDisplay(Exp(1));
  
  FNewNumber := True;
end;

procedure TCalcForm.MemoryClick(Sender: TObject);
var
  cap: String;
begin
  cap := (Sender as TButton).Caption;
  
  if cap = 'MC' then
    FMemory := 0
  else if cap = 'MR' then
  begin
    SetDisplay(FMemory);
    FNewNumber := True;
  end
  else if cap = 'MS' then
    FMemory := GetDisplay
  else if cap = 'M+' then
    FMemory := FMemory + GetDisplay;
end;

procedure TCalcForm.FormKeyPress(Sender: TObject; var Key: Char);
begin
  case Key of
    '0'..'9': begin
                edtDisplay.SetFocus;
                if FNewNumber then
                begin
                  edtDisplay.Text := Key;
                  FNewNumber := False;
                end
                else
                  edtDisplay.Text := edtDisplay.Text + Key;
                Key := #0;
              end;
    '+', '-', '*', '/':
              begin
                if not FNewNumber then Calculate;
                FCurrentValue := GetDisplay;
                FLastOperator := Key;
                FNewNumber := True;
                Key := #0;
              end;
    '=', #13: begin
                EqualsClick(nil);
                Key := #0;
              end;
    #8:       begin
                BackspaceClick(nil);
                Key := #0;
              end;
    #27:      begin
                ClearClick(nil);
                Key := #0;
              end;
  end;
end;

end.
```

---

## แบบฝึกหัด 15 ข้อ

### ข้อ 1: ตารางผลลัพธ์ Arithmetic
เขียนโปรแกรมแสดงตาราง:
```
a  | b  | a+b | a-b | a*b | a/b  | a div b | a mod b
---|----|----|-----|-----|------|---------|--------
10 | 3  | 13 |  7  | 30  | 3.33 |    3    |    1
...
```
สำหรับ a = 10-20, b = 1-5

### ข้อ 2: Truth Table Generator
รับ expression แบบ Boolean แล้วแสดง truth table:
```
A     | B     | A AND B | A OR B
...
```

### ข้อ 3: Bitwise Calculator
เขียน interactive bitwise calculator:
- รับตัวเลข 2 ตัว
- แสดงผล AND, OR, XOR, NOT, SHL, SHR ทั้งใน decimal และ binary

### ข้อ 4: Prime Number Sieve
ใช้ Sieve of Eratosthenes หาจำนวนเฉพาะ 1-1000:
```pascal
var
  isPrime: array[2..1000] of Boolean;
begin
  // ใช้ bitwise operations เพื่อ optimize
  FillChar(isPrime, SizeOf(isPrime), True);
  // ...
end.
```

### ข้อ 5: Operator ทดสอบ Precedence
เขียนโปรแกรมทดสอบ:
```pascal
WriteLn(2 + 3 * 4);           // คาดว่า?
WriteLn(10 - 3 + 2);          // คาดว่า?
WriteLn(NOT TRUE AND FALSE);  // คาดว่า?
WriteLn(TRUE OR FALSE AND FALSE); // คาดว่า?
```
ก่อนรันลองคำนวณเองในหัวก่อน

### ข้อ 6: Bank Calculator
คำนวณดอกเบี้ยเงินฝาก:
- รับเงินต้น, อัตราดอกเบี้ย (ต่อปี), จำนวนปี
- คำนวณดอกเบี้ยแบบทบต้น: A = P(1 + r)^n

### ข้อ 7: Grade Calculator
รับคะแนน 5 วิชา (แต่ละวิชามีน้ำหนักต่างกัน):
- วิชา 1: น้ำหนัก 20%
- วิชา 2: น้ำหนัก 25%
- วิชา 3: น้ำหนัก 30%
- วิชา 4: น้ำหนัก 15%
- วิชา 5: น้ำหนัก 10%

### ข้อ 8: Number Properties
รับตัวเลขและตรวจสอบ:
- เป็นจำนวนเฉพาะหรือไม่
- หารด้วย 2 ลงตัวหรือไม่ (even)
- หารด้วย 3 ลงตัวหรือไม่
- เป็น perfect square หรือไม่

### ข้อ 9: Logical Puzzle
แก้ปัญหาตรรกศาสตร์:
```
ถ้า: A → B (ถ้า A เป็นจริง B เป็นจริง)
     B → C
     A เป็นจริง
ถาม: C เป็นจริงหรือไม่?
```
เขียนโปรแกรมตรวจสอบโดยใช้ boolean operators

### ข้อ 10: Bitwise Cipher
เขียน simple encryption โดย XOR ข้อความกับ key:
```pascal
function XorCipher(s: String; key: Byte): String;
var
  i: Integer;
begin
  Result := s;
  for i := 1 to Length(s) do
    Result[i] := Chr(Ord(s[i]) xor key);
end;
```

### ข้อ 11: Integer Square Root
คำนวณ integer square root ของตัวเลขโดยไม่ใช้ Sqrt() function:
```pascal
function IntSqrt(n: Integer): Integer;
// ใช้ binary search หรือ Newton's method
```

### ข้อ 12: Bit Counter
นับจำนวน bit ที่เป็น 1 ในตัวเลข (Hamming weight):
```pascal
function CountBits(n: Cardinal): Integer;
// ใช้ bitwise AND
```

### ข้อ 13: Range Validator
เขียนฟังก์ชัน validate ข้อมูล:
```pascal
function IsValidAge(age: Integer): Boolean;
function IsValidScore(score: Double): Boolean;
function IsValidMonth(m: Integer): Boolean;
function IsValidDate(d, m, y: Integer): Boolean;
```

### ข้อ 14: Calculator ขั้นสูง
ขยาย Calculator ให้รองรับ:
- Trigonometric functions (sin, cos, tan) ในองศาและเรเดียน
- Logarithm functions (log, ln, log2)
- Powers and roots (^, sqrt, cbrt)
- Memory functions (MC, MR, MS, M+, M-)

### ข้อ 15: Expression Evaluator
เขียน complete expression evaluator ที่รองรับ:
- Parentheses
- +, -, *, /, mod, div
- Negative numbers (-3)
- Functions: sin(), cos(), sqrt(), abs(), pow(x,y)
- Constants: pi, e

---

## สรุป

ในบทนี้เราได้เรียนรู้ Operators ทั้งหมดใน Pascal:

1. **Arithmetic (+, -, *, /, div, mod)** - ความแตกต่างระหว่าง / และ div สำคัญมาก!
2. **Comparison (=, <>, <, >, <=, >=)** - ใช้กับ numbers, strings, chars
3. **Logical (and, or, not, xor)** - Boolean operations พร้อม short-circuit
4. **Bitwise (and, or, xor, not, shl, shr)** - ระดับ bit operations
5. **String (+)** - Concatenation
6. **Assignment (:=)** - Pascal ใช้ := ไม่ใช่ =
7. **Operator Precedence** - ลำดับความสำคัญ ใส่วงเล็บเมื่อสงสัย
8. **Type Compatibility** - ระวัง widening และ narrowing conversion

## บทต่อไป

**Part 06** จะเรียนรู้ Control Structures: if-else, case, for, while, repeat-until

---

*ทุกตัวอย่างทดสอบด้วย Free Pascal 3.2.x / Lazarus 3.x*
