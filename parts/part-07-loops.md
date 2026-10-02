# Part 07 - การวนซ้ำ (Loops)

## สารบัญ
1. [for...to...do Loop](#for-to-do-loop)
2. [for...downto...do Loop](#for-downto-do-loop)
3. [while...do Loop](#while-do-loop)
4. [repeat...until Loop](#repeat-until-loop)
5. [Nested Loops](#nested-loops)
6. [break Statement](#break-statement)
7. [continue Statement](#continue-statement)
8. [การเปรียบเทียบ Loops](#การเปรียบเทียบ-loops)
9. [โปรแกรมตัวอย่าง: ตารางสูตรคูณ](#โปรแกรมตัวอย่าง-ตารางสูตรคูณ)
10. [โปรแกรมตัวอย่าง: Fibonacci](#โปรแกรมตัวอย่าง-fibonacci)
11. [โปรแกรมตัวอย่าง: Prime Numbers](#โปรแกรมตัวอย่าง-prime-numbers)
12. [โปรแกรมตัวอย่าง: Pattern Printing](#โปรแกรมตัวอย่าง-pattern-printing)
13. [โปรแกรมตัวอย่าง: Number Guessing Game](#โปรแกรมตัวอย่าง-number-guessing-game)
14. [แบบฝึกหัด 20 ข้อ พร้อมเฉลย](#แบบฝึกหัด)

---

## บทนำ

Loop (การวนซ้ำ) คือโครงสร้างที่ทำให้โปรแกรมทำงานซ้ำๆ หลายครั้งโดยไม่ต้องเขียนโค้ดซ้ำ Pascal มี loop หลัก 3 ประเภท:

```
for loop   → รู้จำนวนรอบล่วงหน้า
while loop → ตรวจสอบเงื่อนไขก่อนทำงาน
repeat     → ทำงานก่อนแล้วจึงตรวจสอบเงื่อนไข
```

---

## for...to...do Loop

### ไวยากรณ์

```pascal
for <ตัวแปร> := <ค่าเริ่มต้น> to <ค่าสิ้นสุด> do
begin
  // คำสั่งที่ต้องการวนซ้ำ
end;
```

ตัวแปรนับใน `for` loop จะเพิ่มขึ้นทีละ 1 โดยอัตโนมัติ

### ตัวอย่างที่ 1: นับ 1 ถึง 10

```pascal
program Count1to10;
var
  I: Integer;
begin
  WriteLn('นับ 1 ถึง 10:');
  for I := 1 to 10 do
    Write(I, ' ');
  WriteLn;
end.
```

**ผลลัพธ์:**
```
1 2 3 4 5 6 7 8 9 10
```

### ตัวอย่างที่ 2: หาผลรวม 1 ถึง N

```pascal
program SumToN;
var
  N, I: Integer;
  Sum: Integer;
begin
  Write('ป้อนจำนวน N: ');
  ReadLn(N);
  
  Sum := 0;
  for I := 1 to N do
    Sum := Sum + I;
    
  WriteLn('ผลรวม 1 ถึง ', N, ' = ', Sum);
  // สูตร: N*(N+1)/2
  WriteLn('ตรวจสอบด้วยสูตร: ', N * (N + 1) div 2);
end.
```

### ตัวอย่างที่ 3: ตารางสูตรคูณ (แถวเดียว)

```pascal
program MultiTable;
var
  N, I: Integer;
begin
  Write('ป้อนตัวเลข: ');
  ReadLn(N);
  
  WriteLn('สูตรคูณแม่ ', N, ':');
  for I := 1 to 12 do
    WriteLn(N, ' x ', I, ' = ', N * I);
end.
```

### ตัวอย่างที่ 4: แสดงตัวเลขคู่

```pascal
program EvenNumbers;
var
  I: Integer;
begin
  WriteLn('ตัวเลขคู่ตั้งแต่ 2 ถึง 20:');
  for I := 1 to 10 do
    Write(I * 2, ' ');
  WriteLn;
end.
```

### ตัวอย่างที่ 5: ผลคูณสะสม (Factorial)

```pascal
program Factorial;
var
  N, I: Integer;
  Fact: Int64;
begin
  Write('ป้อนตัวเลข N: ');
  ReadLn(N);
  
  Fact := 1;
  for I := 1 to N do
    Fact := Fact * I;
    
  WriteLn(N, '! = ', Fact);
end.
```

### ตัวอย่างที่ 6: วนซ้ำกับ array

```pascal
program ArrayLoop;
var
  Scores: array[1..5] of Integer;
  I: Integer;
  Total: Integer;
begin
  WriteLn('ป้อนคะแนน 5 ค่า:');
  for I := 1 to 5 do
  begin
    Write('คะแนน ', I, ': ');
    ReadLn(Scores[I]);
  end;
  
  Total := 0;
  for I := 1 to 5 do
    Total := Total + Scores[I];
    
  WriteLn('ผลรวม: ', Total);
  WriteLn('เฉลี่ย: ', Total / 5:0:2);
end.
```

---

## for...downto...do Loop

ใช้เมื่อต้องการนับถอยหลัง

### ตัวอย่างที่ 7: นับถอยหลัง

```pascal
program Countdown;
var
  I: Integer;
begin
  WriteLn('นับถอยหลัง:');
  for I := 10 downto 1 do
  begin
    Write(I, '... ');
    // WriteLn หรือ delay ถ้าต้องการ
  end;
  WriteLn;
  WriteLn('ปล่อย!');
end.
```

### ตัวอย่างที่ 8: พิมพ์ตัวอักษรจาก Z ถึง A

```pascal
program AlphabetReverse;
var
  C: Char;
begin
  Write('ตัวอักษร Z-A: ');
  for C := 'Z' downto 'A' do
    Write(C, ' ');
  WriteLn;
end.
```

### ตัวอย่างที่ 9: reverse array

```pascal
program ReverseArray;
var
  Arr: array[1..5] of Integer;
  I: Integer;
begin
  WriteLn('ป้อนตัวเลข 5 ตัว:');
  for I := 1 to 5 do
  begin
    Write('ตัวที่ ', I, ': ');
    ReadLn(Arr[I]);
  end;
  
  Write('เรียงกลับ: ');
  for I := 5 downto 1 do
    Write(Arr[I], ' ');
  WriteLn;
end.
```

---

## while...do Loop

ตรวจสอบเงื่อนไขก่อน ถ้าเป็นจริงจึงทำงาน อาจไม่ทำงานเลยถ้าเงื่อนไขเป็นเท็จตั้งแต่ต้น

### ไวยากรณ์

```pascal
while <เงื่อนไข> do
begin
  // คำสั่ง
end;
```

### ตัวอย่างที่ 10: นับถึง N ด้วย while

```pascal
program WhileCount;
var
  I, N: Integer;
begin
  Write('ป้อน N: ');
  ReadLn(N);
  
  I := 1;
  while I <= N do
  begin
    Write(I, ' ');
    Inc(I);
  end;
  WriteLn;
end.
```

### ตัวอย่างที่ 11: หาเลขหลักของตัวเลข

```pascal
program CountDigits;
var
  N: Integer;
  Digits: Integer;
begin
  Write('ป้อนตัวเลข: ');
  ReadLn(N);
  
  if N = 0 then
    Digits := 1
  else
  begin
    Digits := 0;
    N := Abs(N);
    while N > 0 do
    begin
      Inc(Digits);
      N := N div 10;
    end;
  end;
  
  WriteLn('จำนวนหลัก: ', Digits);
end.
```

### ตัวอย่างที่ 12: หา GCD (ตัวหารร่วมมาก)

```pascal
program GCDProgram;
var
  A, B: Integer;
begin
  Write('ป้อน A: ');
  ReadLn(A);
  Write('ป้อน B: ');
  ReadLn(B);
  
  // Euclidean algorithm
  while B <> 0 do
  begin
    var Temp := B;
    B := A mod B;
    A := Temp;
  end;
  
  WriteLn('ห.ร.ม. = ', A);
end.
```

### ตัวอย่างที่ 13: รับข้อมูลจนกว่าจะถูกต้อง

```pascal
program ValidInput;
var
  Age: Integer;
  Valid: Boolean;
begin
  Valid := False;
  
  while not Valid do
  begin
    Write('ป้อนอายุ (1-120): ');
    ReadLn(Age);
    
    if (Age >= 1) and (Age <= 120) then
      Valid := True
    else
      WriteLn('อายุไม่ถูกต้อง กรุณาลองใหม่');
  end;
  
  WriteLn('อายุที่ป้อน: ', Age);
end.
```

### ตัวอย่างที่ 14: ผลรวมของตัวเลขที่รับจนกว่าจะเป็น 0

```pascal
program SumUntilZero;
var
  Num: Integer;
  Total: Integer;
  Count: Integer;
begin
  Total := 0;
  Count := 0;
  
  WriteLn('ป้อนตัวเลข (0 เพื่อหยุด):');
  Read(Num);
  
  while Num <> 0 do
  begin
    Total := Total + Num;
    Inc(Count);
    Read(Num);
  end;
  WriteLn;
  
  WriteLn('จำนวนที่ป้อน: ', Count);
  WriteLn('ผลรวม: ', Total);
  if Count > 0 then
    WriteLn('เฉลี่ย: ', Total / Count:0:2);
end.
```

---

## repeat...until Loop

ทำงานอย่างน้อย 1 ครั้งเสมอ เพราะตรวจสอบเงื่อนไขหลังจากทำงานแล้ว

### ไวยากรณ์

```pascal
repeat
  // คำสั่ง
until <เงื่อนไข>;
```

**หมายเหตุ:** loop จะหยุดเมื่อเงื่อนไขเป็น **จริง** (ต่างจาก while ที่ทำงานเมื่อจริง)

### ตัวอย่างที่ 15: เมนูพื้นฐาน

```pascal
program BasicMenu;
var
  Choice: Integer;
begin
  repeat
    WriteLn('=== เมนู ===');
    WriteLn('1. ตัวเลือก 1');
    WriteLn('2. ตัวเลือก 2');
    WriteLn('3. ออก');
    Write('เลือก: ');
    ReadLn(Choice);
    
    case Choice of
      1: WriteLn('คุณเลือกตัวเลือก 1');
      2: WriteLn('คุณเลือกตัวเลือก 2');
      3: WriteLn('กำลังออก...');
    else
      WriteLn('ตัวเลือกไม่ถูกต้อง');
    end;
    WriteLn;
    
  until Choice = 3;
end.
```

### ตัวอย่างที่ 16: ทายตัวเลข (ง่าย)

```pascal
program GuessNumber;
var
  Secret, Guess: Integer;
begin
  Secret := 42; // ตัวเลขลับ
  
  WriteLn('ทายตัวเลข 1-100!');
  
  repeat
    Write('ทาย: ');
    ReadLn(Guess);
    
    if Guess < Secret then
      WriteLn('น้อยเกินไป!')
    else if Guess > Secret then
      WriteLn('มากเกินไป!')
    else
      WriteLn('ถูกต้อง!');
      
  until Guess = Secret;
end.
```

### ตัวอย่างที่ 17: ตรวจสอบ PIN

```pascal
program PINCheck;
const
  CORRECT_PIN = 1234;
var
  PIN: Integer;
  Tries: Integer;
begin
  Tries := 0;
  
  repeat
    Inc(Tries);
    Write('ป้อน PIN (', 3 - Tries + 1, ' ครั้งที่เหลือ): ');
    ReadLn(PIN);
    
    if PIN <> CORRECT_PIN then
      WriteLn('PIN ไม่ถูกต้อง!');
      
  until (PIN = CORRECT_PIN) or (Tries >= 3);
  
  if PIN = CORRECT_PIN then
    WriteLn('เข้าสู่ระบบสำเร็จ!')
  else
    WriteLn('บัญชีถูกระงับ!');
end.
```

---

## Nested Loops

การใช้ loop ซ้อน loop

### ตัวอย่างที่ 18: ตารางคูณ 10x10

```pascal
program MultiTable10x10;
var
  I, J: Integer;
begin
  Write('   ');
  for J := 1 to 10 do
    Write(J:4);
  WriteLn;
  WriteLn('   ', StringOfChar('-', 40));
  
  for I := 1 to 10 do
  begin
    Write(I:3, '|');
    for J := 1 to 10 do
      Write((I * J):4);
    WriteLn;
  end;
end.
```

### ตัวอย่างที่ 19: สามเหลี่ยมดาว

```pascal
program StarTriangle;
var
  I, J: Integer;
  N: Integer;
begin
  Write('ความสูงของสามเหลี่ยม: ');
  ReadLn(N);
  
  for I := 1 to N do
  begin
    for J := 1 to I do
      Write('* ');
    WriteLn;
  end;
end.
```

**ผลลัพธ์สำหรับ N=5:**
```
* 
* * 
* * * 
* * * * 
* * * * * 
```

### ตัวอย่างที่ 20: สี่เหลี่ยมกลวง

```pascal
program HollowSquare;
var
  I, J, N: Integer;
begin
  Write('ขนาด N: ');
  ReadLn(N);
  
  for I := 1 to N do
  begin
    for J := 1 to N do
    begin
      if (I = 1) or (I = N) or (J = 1) or (J = N) then
        Write('* ')
      else
        Write('  ');
    end;
    WriteLn;
  end;
end.
```

### ตัวอย่างที่ 21: ตารางเลขสองเท่า

```pascal
program DoubleTable;
var
  Row, Col: Integer;
begin
  for Row := 1 to 5 do
  begin
    for Col := 1 to 5 do
      Write((Row * Col * 2):5);
    WriteLn;
  end;
end.
```

---

## break Statement

`break` ใช้หยุด loop ทันที

### ตัวอย่างที่ 22: หยุดเมื่อพบค่า

```pascal
program BreakExample;
var
  I: Integer;
  Target: Integer;
begin
  Target := 7;
  
  for I := 1 to 100 do
  begin
    if I = Target then
    begin
      WriteLn('พบค่า ', Target, ' ที่ตำแหน่ง I = ', I);
      Break;
    end;
    Write(I, ' ');
  end;
  WriteLn;
end.
```

### ตัวอย่างที่ 23: ค้นหาใน array

```pascal
program SearchArray;
var
  Data: array[1..10] of Integer;
  I, Target: Integer;
  Found: Boolean;
begin
  // สร้างข้อมูล
  for I := 1 to 10 do
    Data[I] := I * 3;
  
  Write('หาตัวเลข: ');
  ReadLn(Target);
  
  Found := False;
  for I := 1 to 10 do
  begin
    if Data[I] = Target then
    begin
      WriteLn('พบที่ตำแหน่ง ', I);
      Found := True;
      Break;
    end;
  end;
  
  if not Found then
    WriteLn('ไม่พบ ', Target);
end.
```

### ตัวอย่างที่ 24: break กับ while

```pascal
program WhileBreak;
var
  Input: String;
begin
  WriteLn('พิมพ์ "quit" เพื่อออก');
  
  while True do
  begin
    Write('ป้อนข้อความ: ');
    ReadLn(Input);
    
    if Input = 'quit' then
      Break;
      
    WriteLn('คุณป้อน: ', Input);
  end;
  
  WriteLn('ออกจากโปรแกรมแล้ว');
end.
```

---

## continue Statement

`continue` ข้ามรอบปัจจุบันและไปรอบถัดไป

### ตัวอย่างที่ 25: ข้ามเลขคี่

```pascal
program SkipOdd;
var
  I: Integer;
begin
  Write('เลขคู่ 1-20: ');
  for I := 1 to 20 do
  begin
    if I mod 2 <> 0 then
      Continue;
    Write(I, ' ');
  end;
  WriteLn;
end.
```

### ตัวอย่างที่ 26: ข้ามศูนย์

```pascal
program SkipZero;
var
  Data: array[1..8] of Integer;
  I: Integer;
  Total: Real;
  Count: Integer;
begin
  Data[1] := 10;
  Data[2] := 0;
  Data[3] := 20;
  Data[4] := 0;
  Data[5] := 30;
  Data[6] := 0;
  Data[7] := 40;
  Data[8] := 50;
  
  Total := 0;
  Count := 0;
  
  for I := 1 to 8 do
  begin
    if Data[I] = 0 then
      Continue;
    Total := Total + Data[I];
    Inc(Count);
  end;
  
  WriteLn('ผลรวม (ไม่รวมศูนย์): ', Total:0:0);
  WriteLn('เฉลี่ย: ', Total / Count:0:2);
end.
```

### ตัวอย่างที่ 27: continue กับ while

```pascal
program WhileContinue;
var
  I: Integer;
begin
  I := 0;
  while I < 15 do
  begin
    Inc(I);
    if I mod 3 = 0 then
      Continue;
    Write(I, ' ');
  end;
  WriteLn;
  WriteLn('(ข้ามตัวเลขที่หารด้วย 3 ลงตัว)');
end.
```

---

## การเปรียบเทียบ Loops

| คุณสมบัติ | for | while | repeat |
|-----------|-----|-------|--------|
| ตรวจสอบเงื่อนไข | ต้น loop | ต้น loop | ท้าย loop |
| ทำงานขั้นต่ำ | ขึ้นกับเงื่อนไข | 0 ครั้ง | 1 ครั้ง |
| เหมาะสำหรับ | รู้จำนวนรอบล่วงหน้า | ไม่รู้จำนวนรอบ | ต้องทำอย่างน้อย 1 ครั้ง |
| ตัวนับ | อัตโนมัติ | manual | manual |

```pascal
program LoopComparison;
var
  I: Integer;
begin
  WriteLn('=== for loop ===');
  for I := 1 to 5 do
    Write(I, ' ');
  WriteLn;
  
  WriteLn('=== while loop ===');
  I := 1;
  while I <= 5 do
  begin
    Write(I, ' ');
    Inc(I);
  end;
  WriteLn;
  
  WriteLn('=== repeat loop ===');
  I := 1;
  repeat
    Write(I, ' ');
    Inc(I);
  until I > 5;
  WriteLn;
end.
```

---

## โปรแกรมตัวอย่าง: ตารางสูตรคูณ

```pascal
program MultiplicationTable;
var
  I, J: Integer;
  StartNum, EndNum: Integer;
begin
  WriteLn('╔════════════════════════════╗');
  WriteLn('║    ตารางสูตรคูณ            ║');
  WriteLn('╚════════════════════════════╝');
  WriteLn;
  
  Write('ตั้งแต่แม่ที่: ');
  ReadLn(StartNum);
  Write('ถึงแม่ที่: ');
  ReadLn(EndNum);
  
  if StartNum > EndNum then
  begin
    WriteLn('ค่าเริ่มต้นต้องน้อยกว่าค่าสิ้นสุด!');
    Halt;
  end;
  
  for I := StartNum to EndNum do
  begin
    WriteLn;
    WriteLn('═══ สูตรคูณแม่ ', I, ' ═══');
    for J := 1 to 12 do
    begin
      if J = 1 then
        WriteLn(I:2, ' × ', J:2, ' =', (I * J):4)
      else if J <= 10 then
        WriteLn(I:2, ' × ', J:2, ' =', (I * J):4)
      else
        WriteLn(I:2, ' × ', J:2, ' =', (I * J):4);
    end;
  end;
end.
```

---

## โปรแกรมตัวอย่าง: Fibonacci

```pascal
program FibonacciSeries;
var
  N: Integer;
  A, B, C: Int64;
  I: Integer;
begin
  WriteLn('═══ อนุกรม Fibonacci ═══');
  Write('แสดงกี่จำนวน: ');
  ReadLn(N);
  
  if N <= 0 then
  begin
    WriteLn('กรุณาป้อนจำนวนที่เป็นบวก');
    Halt;
  end;
  
  Write('F(1) = 0');
  
  if N >= 2 then
    Write(', F(2) = 1');
  
  A := 0; // F(1)
  B := 1; // F(2)
  
  for I := 3 to N do
  begin
    C := A + B;
    Write(', F(', I, ') = ', C);
    A := B;
    B := C;
  end;
  
  WriteLn;
  WriteLn;
  
  // แสดงแบบสวยงาม
  WriteLn('═══ แสดงแบบตาราง ═══');
  WriteLn('No.':5, 'Fibonacci':15, 'Ratio':12);
  WriteLn(StringOfChar('-', 32));
  
  A := 0;
  B := 1;
  
  WriteLn(1:5, A:15);
  if N >= 2 then
    WriteLn(2:5, B:15);
  
  for I := 3 to N do
  begin
    C := A + B;
    if A > 0 then
      WriteLn(I:5, C:15, (C / B):12:6)
    else
      WriteLn(I:5, C:15, 'N/A':12);
    A := B;
    B := C;
  end;
  
  WriteLn;
  WriteLn('หมายเหตุ: อัตราส่วนจะเข้าใกล้ Golden Ratio ≈ 1.618034');
end.
```

---

## โปรแกรมตัวอย่าง: Prime Numbers

```pascal
program PrimeNumbers;
var
  N: Integer;
  I, J: Integer;
  IsPrime: Boolean;
  Count: Integer;
  
function CheckPrime(Num: Integer): Boolean;
var
  K: Integer;
begin
  if Num < 2 then
  begin
    Result := False;
    Exit;
  end;
  if Num = 2 then
  begin
    Result := True;
    Exit;
  end;
  if Num mod 2 = 0 then
  begin
    Result := False;
    Exit;
  end;
  
  Result := True;
  K := 3;
  while K * K <= Num do
  begin
    if Num mod K = 0 then
    begin
      Result := False;
      Break;
    end;
    Inc(K, 2);
  end;
end;

begin
  Write('หาจำนวนเฉพาะถึง N: ');
  ReadLn(N);
  
  WriteLn('จำนวนเฉพาะตั้งแต่ 2 ถึง ', N, ':');
  WriteLn(StringOfChar('═', 50));
  
  Count := 0;
  for I := 2 to N do
  begin
    if CheckPrime(I) then
    begin
      Write(I:6);
      Inc(Count);
      if Count mod 10 = 0 then
        WriteLn;
    end;
  end;
  
  WriteLn;
  WriteLn(StringOfChar('═', 50));
  WriteLn('พบจำนวนเฉพาะทั้งหมด ', Count, ' จำนวน');
  
  // ทดสอบ Goldbach's Conjecture
  WriteLn;
  WriteLn('ทดสอบ Goldbach Conjecture (เลขคู่ > 2):');
  WriteLn('(ทุกเลขคู่ที่มากกว่า 2 เป็นผลรวมของจำนวนเฉพาะสองตัว)');
  WriteLn;
  
  for I := 4 to Min(N, 30) do
  begin
    if I mod 2 = 0 then
    begin
      for J := 2 to I div 2 do
      begin
        if CheckPrime(J) and CheckPrime(I - J) then
        begin
          WriteLn(I:4, ' = ', J, ' + ', I - J);
          Break;
        end;
      end;
    end;
  end;
end.
```

---

## โปรแกรมตัวอย่าง: Pattern Printing

```pascal
program PatternPrinting;
var
  N: Integer;
  Choice: Integer;
  I, J: Integer;

procedure PrintTriangle(Size: Integer);
begin
  WriteLn('สามเหลี่ยมดาว:');
  for I := 1 to Size do
  begin
    for J := 1 to I do
      Write('* ');
    WriteLn;
  end;
end;

procedure PrintReverseTriangle(Size: Integer);
begin
  WriteLn('สามเหลี่ยมกลับ:');
  for I := Size downto 1 do
  begin
    for J := 1 to I do
      Write('* ');
    WriteLn;
  end;
end;

procedure PrintPyramid(Size: Integer);
var
  Spaces: Integer;
begin
  WriteLn('พีระมิด:');
  for I := 1 to Size do
  begin
    // พิมพ์ช่องว่าง
    for Spaces := 1 to Size - I do
      Write(' ');
    // พิมพ์ดาว
    for J := 1 to (2 * I - 1) do
      Write('*');
    WriteLn;
  end;
end;

procedure PrintDiamond(Size: Integer);
var
  Spaces: Integer;
begin
  WriteLn('รูปเพชร:');
  // ครึ่งบน
  for I := 1 to Size do
  begin
    for Spaces := 1 to Size - I do
      Write(' ');
    for J := 1 to (2 * I - 1) do
      Write('*');
    WriteLn;
  end;
  // ครึ่งล่าง
  for I := Size - 1 downto 1 do
  begin
    for Spaces := 1 to Size - I do
      Write(' ');
    for J := 1 to (2 * I - 1) do
      Write('*');
    WriteLn;
  end;
end;

procedure PrintNumberSquare(Size: Integer);
begin
  WriteLn('ตารางตัวเลข:');
  for I := 1 to Size do
  begin
    for J := 1 to Size do
      Write((I * 10 + J):4);
    WriteLn;
  end;
end;

procedure PrintChessboard(Size: Integer);
begin
  WriteLn('กระดานหมากรุก:');
  for I := 1 to Size do
  begin
    for J := 1 to Size do
    begin
      if (I + J) mod 2 = 0 then
        Write('█ ')
      else
        Write('░ ');
    end;
    WriteLn;
  end;
end;

begin
  WriteLn('=== Pattern Printing ===');
  WriteLn('1. สามเหลี่ยมดาว');
  WriteLn('2. สามเหลี่ยมกลับ');
  WriteLn('3. พีระมิด');
  WriteLn('4. รูปเพชร');
  WriteLn('5. ตารางตัวเลข');
  WriteLn('6. กระดานหมากรุก');
  Write('เลือก (1-6): ');
  ReadLn(Choice);
  Write('ขนาด: ');
  ReadLn(N);
  WriteLn;
  
  case Choice of
    1: PrintTriangle(N);
    2: PrintReverseTriangle(N);
    3: PrintPyramid(N);
    4: PrintDiamond(N);
    5: PrintNumberSquare(N);
    6: PrintChessboard(N);
  else
    WriteLn('ตัวเลือกไม่ถูกต้อง');
  end;
end.
```

---

## โปรแกรมตัวอย่าง: Number Guessing Game

```pascal
program NumberGuessingGame;
uses
  SysUtils;
  
var
  SecretNumber: Integer;
  Guess: Integer;
  Attempts: Integer;
  MaxAttempts: Integer;
  PlayAgain: Char;
  TotalGames: Integer;
  TotalAttempts: Integer;
  BestScore: Integer;
  
begin
  Randomize; // เริ่มต้น random generator
  TotalGames := 0;
  TotalAttempts := 0;
  BestScore := MaxInt;
  
  WriteLn('╔════════════════════════════════╗');
  WriteLn('║    เกมทายตัวเลข               ║');
  WriteLn('║    ทายตัวเลข 1-100            ║');
  WriteLn('╚════════════════════════════════╝');
  WriteLn;
  
  repeat
    // สร้างตัวเลขลับ
    SecretNumber := Random(100) + 1; // 1-100
    MaxAttempts := 7;
    Attempts := 0;
    Inc(TotalGames);
    
    WriteLn('=== เกมที่ ', TotalGames, ' ===');
    WriteLn('คุณมี ', MaxAttempts, ' ครั้งในการทาย');
    WriteLn;
    
    repeat
      Inc(Attempts);
      Write('ครั้งที่ ', Attempts, '/', MaxAttempts, ' ทาย: ');
      ReadLn(Guess);
      
      if (Guess < 1) or (Guess > 100) then
      begin
        WriteLn('ตัวเลขต้องอยู่ระหว่าง 1-100!');
        Dec(Attempts); // ไม่นับครั้งที่ป้อนผิดรูปแบบ
      end
      else if Guess < SecretNumber then
        WriteLn('น้อยเกินไป! ▲')
      else if Guess > SecretNumber then
        WriteLn('มากเกินไป! ▼')
      else
      begin
        WriteLn('🎉 ถูกต้อง! ตัวเลขคือ ', SecretNumber);
        WriteLn('ใช้ ', Attempts, ' ครั้ง');
        
        // ให้คะแนนตามจำนวนครั้งที่ใช้
        Write('ระดับ: ');
        case Attempts of
          1: WriteLn('★★★★★ ยอดเยี่ยมมาก!');
          2: WriteLn('★★★★☆ ยอดเยี่ยม!');
          3: WriteLn('★★★☆☆ ดีมาก!');
          4: WriteLn('★★☆☆☆ ดี');
          5, 6: WriteLn('★☆☆☆☆ พอใช้');
          7: WriteLn('☆☆☆☆☆ หวุดหวิด');
        end;
      end;
      
    until (Guess = SecretNumber) or (Attempts >= MaxAttempts);
    
    if Guess <> SecretNumber then
    begin
      WriteLn('หมดแล้ว! ตัวเลขที่ถูกต้องคือ ', SecretNumber);
      Attempts := MaxAttempts; // บันทึกว่าใช้เต็มจำนวน
    end;
    
    // อัพเดทสถิติ
    TotalAttempts := TotalAttempts + Attempts;
    if Attempts < BestScore then
      BestScore := Attempts;
    
    WriteLn;
    WriteLn('สถิติ:');
    WriteLn('  เล่นทั้งหมด: ', TotalGames, ' เกม');
    WriteLn('  เฉลี่ยครั้งที่ใช้: ', TotalAttempts / TotalGames:0:1);
    WriteLn('  ทายครั้งน้อยที่สุด: ', BestScore, ' ครั้ง');
    WriteLn;
    
    Write('เล่นอีกครั้ง? (y/n): ');
    ReadLn(PlayAgain);
    WriteLn;
    
  until (PlayAgain = 'n') or (PlayAgain = 'N');
  
  WriteLn('ขอบคุณที่เล่น!');
  WriteLn('สรุปผล: เล่น ', TotalGames, ' เกม, เฉลี่ย ', 
          TotalAttempts / TotalGames:0:1, ' ครั้ง/เกม');
end.
```

---

## แบบฝึกหัด

### ข้อที่ 1: ผลรวมเลขคี่

```pascal
// เขียนโปรแกรมหาผลรวมของเลขคี่ตั้งแต่ 1 ถึง N
program SumOdd;
var
  N, I, Sum: Integer;
begin
  Write('ป้อน N: ');
  ReadLn(N);
  Sum := 0;
  for I := 1 to N do
    if I mod 2 <> 0 then
      Sum := Sum + I;
  WriteLn('ผลรวมเลขคี่ 1 ถึง ', N, ' = ', Sum);
end.
```

### ข้อที่ 2: เลขที่หารด้วย 3 และ 7 ลงตัว

```pascal
program DivBy3And7;
var
  I: Integer;
begin
  WriteLn('เลขที่หารด้วย 3 และ 7 ลงตัว (1-200):');
  for I := 1 to 200 do
    if (I mod 3 = 0) and (I mod 7 = 0) then
      Write(I, ' ');
  WriteLn;
end.
```

### ข้อที่ 3: Pascal Triangle

```pascal
program PascalTriangle;
var
  N, I, J: Integer;
  C: Int64;
begin
  Write('จำนวนแถว: ');
  ReadLn(N);
  
  for I := 0 to N - 1 do
  begin
    C := 1;
    for J := 1 to N - I do Write(' ');
    for J := 0 to I do
    begin
      Write(C:3);
      C := C * (I - J) div (J + 1);
    end;
    WriteLn;
  end;
end.
```

### ข้อที่ 4: Armstrong Numbers

```pascal
program ArmstrongNumbers;
var
  N, I: Integer;
  Sum, Temp, Digits: Integer;
begin
  Write('หา Armstrong numbers ถึง: ');
  ReadLn(N);
  
  WriteLn('Armstrong numbers:');
  for I := 1 to N do
  begin
    // หาจำนวนหลัก
    Digits := 0;
    Temp := I;
    while Temp > 0 do begin Inc(Digits); Temp := Temp div 10; end;
    
    // คำนวณผลรวม
    Sum := 0;
    Temp := I;
    while Temp > 0 do
    begin
      var D := Temp mod 10;
      var P := 1;
      var K: Integer;
      for K := 1 to Digits do P := P * D;
      Sum := Sum + P;
      Temp := Temp div 10;
    end;
    
    if Sum = I then
      Write(I, ' ');
  end;
  WriteLn;
end.
```

### ข้อที่ 5: หาค่า PI ด้วย Leibniz formula

```pascal
program ApproxPI;
var
  Terms: Integer;
  PI: Real;
  I: Integer;
  Sign: Integer;
begin
  Write('จำนวน terms: ');
  ReadLn(Terms);
  
  PI := 0;
  Sign := 1;
  for I := 0 to Terms - 1 do
  begin
    PI := PI + Sign / (2 * I + 1);
    Sign := -Sign;
  end;
  PI := PI * 4;
  
  WriteLn('PI ≈ ', PI:0:10);
  WriteLn('PI จริง = 3.1415926536');
end.
```

### ข้อที่ 6: ตรวจสอบ Palindrome

```pascal
program PalindromeCheck;
var
  N, Reversed, Temp: Integer;
begin
  Write('ป้อนตัวเลข: ');
  ReadLn(N);
  
  Reversed := 0;
  Temp := N;
  while Temp > 0 do
  begin
    Reversed := Reversed * 10 + (Temp mod 10);
    Temp := Temp div 10;
  end;
  
  if N = Reversed then
    WriteLn(N, ' เป็น palindrome')
  else
    WriteLn(N, ' ไม่เป็น palindrome');
end.
```

### ข้อที่ 7: หาตัวประกอบ

```pascal
program Factors;
var
  N, I: Integer;
begin
  Write('ป้อนตัวเลข: ');
  ReadLn(N);
  
  Write('ตัวประกอบของ ', N, ': ');
  for I := 1 to N do
    if N mod I = 0 then
      Write(I, ' ');
  WriteLn;
end.
```

### ข้อที่ 8: ผลรวมหลักตัวเลข

```pascal
program DigitSum;
var
  N, Sum: Integer;
begin
  Write('ป้อนตัวเลข: ');
  ReadLn(N);
  Sum := 0;
  while N > 0 do
  begin
    Sum := Sum + (N mod 10);
    N := N div 10;
  end;
  WriteLn('ผลรวมหลัก = ', Sum);
end.
```

### ข้อที่ 9: แสดงตาราง ASCII

```pascal
program ASCIITable;
var
  I: Integer;
begin
  WriteLn('ตาราง ASCII (32-126):');
  WriteLn('Code':6, 'Char':6);
  WriteLn(StringOfChar('-', 12));
  for I := 32 to 126 do
    WriteLn(I:6, Chr(I):6);
end.
```

### ข้อที่ 10: เกม Collatz Conjecture

```pascal
program Collatz;
var
  N: Integer;
  Steps: Integer;
begin
  Write('ป้อนตัวเลขบวก: ');
  ReadLn(N);
  
  Steps := 0;
  Write(N);
  while N <> 1 do
  begin
    if N mod 2 = 0 then
      N := N div 2
    else
      N := 3 * N + 1;
    Write(' -> ', N);
    Inc(Steps);
  end;
  WriteLn;
  WriteLn('จำนวนขั้นตอน: ', Steps);
end.
```

### ข้อที่ 11-20 (สั้น)

```pascal
// ข้อ 11: หาค่า e (Euler's number)
program EulerNumber;
var N, I: Integer; E, Fact: Real;
begin
  Write('จำนวน terms: '); ReadLn(N);
  E := 1; Fact := 1;
  for I := 1 to N do begin Fact := Fact * I; E := E + 1/Fact; end;
  WriteLn('e ≈ ', E:0:10);
end.

// ข้อ 12: วาดสี่เหลี่ยมผืนผ้าดาว
program Rectangle;
var W, H, I, J: Integer;
begin
  Write('กว้าง: '); ReadLn(W);
  Write('สูง: '); ReadLn(H);
  for I := 1 to H do begin
    for J := 1 to W do Write('*');
    WriteLn;
  end;
end.

// ข้อ 13: แสดงเลขที่มีเลข 7
program Contains7;
var I: Integer;
begin
  for I := 1 to 100 do
    if (I mod 10 = 7) or (I div 10 = 7) then
      Write(I, ' ');
  WriteLn;
end.

// ข้อ 14: หาค่ามากสุดและน้อยสุด
program MinMax;
var N, I, Val, Min, Max: Integer;
begin
  Write('จำนวนข้อมูล: '); ReadLn(N);
  Write('ป้อนตัวแรก: '); ReadLn(Min); Max := Min;
  for I := 2 to N do begin
    Write('ป้อน: '); ReadLn(Val);
    if Val < Min then Min := Val;
    if Val > Max then Max := Val;
  end;
  WriteLn('Min = ', Min, ', Max = ', Max);
end.

// ข้อ 15: นับคำในประโยค
program WordCount;
var S: String; I, Count: Integer; InWord: Boolean;
begin
  Write('ป้อนประโยค: '); ReadLn(S);
  Count := 0; InWord := False;
  for I := 1 to Length(S) do begin
    if S[I] <> ' ' then begin
      if not InWord then begin Inc(Count); InWord := True; end;
    end else InWord := False;
  end;
  WriteLn('จำนวนคำ: ', Count);
end.

// ข้อ 16: ผลรวมของตัวเลขที่หารด้วย N ลงตัว
program SumDivisible;
var N, Limit, I, Sum: Integer;
begin
  Write('หารด้วย: '); ReadLn(N);
  Write('ถึง: '); ReadLn(Limit);
  Sum := 0;
  for I := 1 to Limit do
    if I mod N = 0 then Sum := Sum + I;
  WriteLn('ผลรวม = ', Sum);
end.

// ข้อ 17: แสดงตาราง ASCII ตัวอักษร
program LetterTable;
var C: Char; I: Integer;
begin
  Write('A-Z: ');
  for C := 'A' to 'Z' do Write(C, ' ');
  WriteLn;
  Write('a-z: ');
  for C := 'a' to 'z' do Write(C, ' ');
  WriteLn;
end.

// ข้อ 18: หา Perfect Numbers
program PerfectNumbers;
var N, I, J, Sum: Integer;
begin
  Write('หา Perfect numbers ถึง: '); ReadLn(N);
  for I := 2 to N do begin
    Sum := 1;
    for J := 2 to I div 2 do
      if I mod J = 0 then Sum := Sum + J;
    if Sum = I then Write(I, ' ');
  end;
  WriteLn;
end.

// ข้อ 19: แปลงเลขทศนิยมเป็น binary
program ToBinary;
var N, I: Integer; Bits: array[0..31] of Integer;
begin
  Write('ป้อนตัวเลข: '); ReadLn(N);
  I := 0;
  while N > 0 do begin
    Bits[I] := N mod 2;
    N := N div 2;
    Inc(I);
  end;
  Write('Binary: ');
  for I := I - 1 downto 0 do Write(Bits[I]);
  WriteLn;
end.

// ข้อ 20: หาจำนวนที่เป็น perfect square
program PerfectSquares;
var N, I: Integer;
begin
  Write('ถึง: '); ReadLn(N);
  Write('Perfect squares: ');
  I := 1;
  while I * I <= N do begin
    Write(I * I, ' ');
    Inc(I);
  end;
  WriteLn;
end.
```

---

## สรุป

Loops เป็นโครงสร้างพื้นฐานที่สำคัญมาก ควรเลือกใช้ให้เหมาะสม:

- **for loop**: เมื่อรู้จำนวนรอบที่แน่นอน
- **while loop**: เมื่อไม่รู้จำนวนรอบ ตรวจสอบก่อน
- **repeat...until**: เมื่อต้องทำงานอย่างน้อย 1 ครั้ง
- **break**: หยุด loop กลางคัน
- **continue**: ข้ามรอบปัจจุบัน

---

*จบ Part 07 - การวนซ้ำ (Loops)*
