# Part 06 - การควบคุมการทำงาน (Control Flow)

## สารบัญ
1. [if...then Statement](#if-then-statement)
2. [if...then...else Statement](#if-then-else-statement)
3. [Nested if Statements](#nested-if-statements)
4. [case...of Statement](#case-of-statement)
5. [case with Ranges](#case-with-ranges)
6. [Nested case Statements](#nested-case-statements)
7. [goto Statement](#goto-statement)
8. [โปรแกรมตัวอย่าง: เกรดนักเรียน](#โปรแกรมตัวอย่าง-เกรดนักเรียน)
9. [โปรแกรมตัวอย่าง: เครื่องคิดเลข](#โปรแกรมตัวอย่าง-เครื่องคิดเลข)
10. [โปรแกรมตัวอย่าง: ระบบเมนู](#โปรแกรมตัวอย่าง-ระบบเมนู)
11. [โปรแกรมตัวอย่าง: BMI Calculator](#โปรแกรมตัวอย่าง-bmi-calculator)
12. [แบบฝึกหัด 20 ข้อ พร้อมเฉลย](#แบบฝึกหัด)

---

## บทนำ

การควบคุมการทำงาน (Control Flow) คือกลไกที่ช่วยให้โปรแกรมสามารถตัดสินใจได้ว่าจะทำงานในลำดับใด โดยพิจารณาจากเงื่อนไขต่างๆ ใน Pascal/Lazarus มีโครงสร้างการควบคุมการทำงานหลักๆ ดังนี้:

| โครงสร้าง | การใช้งาน |
|-----------|-----------|
| `if...then` | ตรวจสอบเงื่อนไขและทำงานเมื่อเป็นจริง |
| `if...then...else` | ตรวจสอบเงื่อนไขและเลือกทำงานตามผลลัพธ์ |
| `case...of` | เลือกทำงานตามค่าของตัวแปร |
| `goto` | กระโดดไปยังตำแหน่งที่กำหนด (ไม่แนะนำ) |

---

## if...then Statement

### ไวยากรณ์พื้นฐาน

```pascal
if <เงื่อนไข> then
  <คำสั่ง>;
```

หรือสำหรับหลายคำสั่ง:

```pascal
if <เงื่อนไข> then
begin
  <คำสั่งที่ 1>;
  <คำสั่งที่ 2>;
  // ...
end;
```

### ตัวอย่างที่ 1: การตรวจสอบตัวเลขบวก

```pascal
program CheckPositive;
var
  Number: Integer;
begin
  Write('กรุณาป้อนตัวเลข: ');
  ReadLn(Number);
  
  if Number > 0 then
    WriteLn('ตัวเลขนี้เป็นบวก');
    
  WriteLn('สิ้นสุดโปรแกรม');
end.
```

### ตัวอย่างที่ 2: การตรวจสอบอายุ

```pascal
program CheckAge;
var
  Age: Integer;
begin
  Write('กรุณาป้อนอายุ: ');
  ReadLn(Age);
  
  if Age >= 18 then
  begin
    WriteLn('คุณมีอายุครบบรรลุนิติภาวะแล้ว');
    WriteLn('คุณสามารถลงคะแนนเสียงได้');
  end;
  
  WriteLn('อายุของคุณคือ: ', Age, ' ปี');
end.
```

### ตัวอย่างที่ 3: การตรวจสอบเกรด

```pascal
program CheckGrade;
var
  Score: Real;
begin
  Write('กรุณาป้อนคะแนน (0-100): ');
  ReadLn(Score);
  
  if Score >= 80 then
    WriteLn('เกรด A - ยอดเยี่ยม!');
    
  if Score >= 70 then
    WriteLn('คะแนนสูงกว่า 70');
end.
```

### ตัวอย่างที่ 4: การตรวจสอบตัวเลขคู่-คี่

```pascal
program EvenOdd;
var
  Num: Integer;
begin
  Write('ป้อนตัวเลข: ');
  ReadLn(Num);
  
  if (Num mod 2) = 0 then
    WriteLn(Num, ' เป็นเลขคู่');
    
  if (Num mod 2) <> 0 then
    WriteLn(Num, ' เป็นเลขคี่');
end.
```

### ตัวอย่างที่ 5: การตรวจสอบอุณหภูมิ

```pascal
program TemperatureCheck;
var
  Temp: Real;
begin
  Write('ป้อนอุณหภูมิ (องศาเซลเซียส): ');
  ReadLn(Temp);
  
  if Temp > 37.5 then
  begin
    WriteLn('คำเตือน: อุณหภูมิสูงผิดปกติ!');
    WriteLn('ควรพบแพทย์');
  end;
  
  if Temp < 36.0 then
  begin
    WriteLn('คำเตือน: อุณหภูมิต่ำผิดปกติ!');
    WriteLn('ควรพักผ่อนและดูแลตัวเอง');
  end;
end.
```

---

## if...then...else Statement

### ไวยากรณ์

```pascal
if <เงื่อนไข> then
  <คำสั่งเมื่อจริง>
else
  <คำสั่งเมื่อเท็จ>;
```

**หมายเหตุสำคัญ:** ไม่ต้องใส่เครื่องหมาย `;` ก่อน `else`

### ตัวอย่างที่ 6: ตัวเลขบวกหรือลบ

```pascal
program PosNeg;
var
  N: Integer;
begin
  Write('ป้อนตัวเลข: ');
  ReadLn(N);
  
  if N >= 0 then
    WriteLn(N, ' เป็นตัวเลขไม่ติดลบ')
  else
    WriteLn(N, ' เป็นตัวเลขติดลบ');
end.
```

### ตัวอย่างที่ 7: เปรียบเทียบตัวเลขสองตัว

```pascal
program CompareTwo;
var
  A, B: Integer;
begin
  Write('ป้อนตัวเลขที่ 1: ');
  ReadLn(A);
  Write('ป้อนตัวเลขที่ 2: ');
  ReadLn(B);
  
  if A > B then
    WriteLn(A, ' มากกว่า ', B)
  else if A < B then
    WriteLn(A, ' น้อยกว่า ', B)
  else
    WriteLn('ตัวเลขทั้งสองเท่ากัน');
end.
```

### ตัวอย่างที่ 8: ผ่าน/ไม่ผ่าน

```pascal
program PassFail;
var
  Score: Integer;
begin
  Write('กรุณาป้อนคะแนน: ');
  ReadLn(Score);
  
  if Score >= 50 then
  begin
    WriteLn('ผลการสอบ: ผ่าน');
    WriteLn('ยินดีด้วย!');
  end
  else
  begin
    WriteLn('ผลการสอบ: ไม่ผ่าน');
    WriteLn('ต้องสอบใหม่');
  end;
  
  WriteLn('คะแนนของคุณ: ', Score);
end.
```

### ตัวอย่างที่ 9: การตรวจสอบปีอธิกสุรทิน

```pascal
program LeapYear;
var
  Year: Integer;
  IsLeap: Boolean;
begin
  Write('ป้อนปี ค.ศ.: ');
  ReadLn(Year);
  
  IsLeap := ((Year mod 4 = 0) and (Year mod 100 <> 0)) or
            (Year mod 400 = 0);
  
  if IsLeap then
    WriteLn(Year, ' เป็นปีอธิกสุรทิน (Leap Year)')
  else
    WriteLn(Year, ' ไม่เป็นปีอธิกสุรทิน');
end.
```

### ตัวอย่างที่ 10: การแปลงเกรด A-F

```pascal
program GradeConvert;
var
  Score: Integer;
  Grade: Char;
begin
  Write('ป้อนคะแนน (0-100): ');
  ReadLn(Score);
  
  if Score >= 90 then
    Grade := 'A'
  else if Score >= 80 then
    Grade := 'B'
  else if Score >= 70 then
    Grade := 'C'
  else if Score >= 60 then
    Grade := 'D'
  else
    Grade := 'F';
    
  WriteLn('เกรดของคุณ: ', Grade);
end.
```

---

## Nested if Statements

การซ้อน if คือการนำ if มาอยู่ภายใน if อื่น ช่วยให้ตรวจสอบเงื่อนไขซับซ้อนได้

### ตัวอย่างที่ 11: ตรวจสอบหลายเงื่อนไข

```pascal
program NestedIfExample;
var
  Age: Integer;
  HasID: Boolean;
  Answer: Char;
begin
  Write('ป้อนอายุ: ');
  ReadLn(Age);
  
  Write('มีบัตรประชาชนหรือไม่? (y/n): ');
  ReadLn(Answer);
  HasID := (Answer = 'y') or (Answer = 'Y');
  
  if Age >= 18 then
  begin
    if HasID then
      WriteLn('สามารถลงคะแนนเสียงได้')
    else
      WriteLn('ต้องมีบัตรประชาชนก่อน');
  end
  else
    WriteLn('อายุยังไม่ถึง 18 ปี');
end.
```

### ตัวอย่างที่ 12: ตรวจสอบสามเหลี่ยม

```pascal
program TriangleCheck;
var
  A, B, C: Integer;
begin
  Write('ด้านที่ 1: ');
  ReadLn(A);
  Write('ด้านที่ 2: ');
  ReadLn(B);
  Write('ด้านที่ 3: ');
  ReadLn(C);
  
  if (A > 0) and (B > 0) and (C > 0) then
  begin
    if (A + B > C) and (A + C > B) and (B + C > A) then
    begin
      if (A = B) and (B = C) then
        WriteLn('สามเหลี่ยมด้านเท่า')
      else if (A = B) or (B = C) or (A = C) then
        WriteLn('สามเหลี่ยมหน้าจั่ว')
      else
        WriteLn('สามเหลี่ยมด้านไม่เท่า');
    end
    else
      WriteLn('ไม่สามารถสร้างสามเหลี่ยมได้');
  end
  else
    WriteLn('ความยาวด้านต้องเป็นบวก');
end.
```

### ตัวอย่างที่ 13: ระบบลำดับชั้น

```pascal
program HierarchySystem;
var
  Level: Integer;
  Score: Integer;
begin
  Write('ป้อนระดับ (1-3): ');
  ReadLn(Level);
  Write('ป้อนคะแนน: ');
  ReadLn(Score);
  
  if Level = 1 then
  begin
    if Score >= 50 then
      WriteLn('ระดับ 1: ผ่าน')
    else
      WriteLn('ระดับ 1: ไม่ผ่าน');
  end
  else if Level = 2 then
  begin
    if Score >= 60 then
      WriteLn('ระดับ 2: ผ่าน')
    else
      WriteLn('ระดับ 2: ไม่ผ่าน');
  end
  else if Level = 3 then
  begin
    if Score >= 70 then
      WriteLn('ระดับ 3: ผ่าน')
    else
      WriteLn('ระดับ 3: ไม่ผ่าน');
  end
  else
    WriteLn('ระดับไม่ถูกต้อง');
end.
```

### ตัวอย่างที่ 14: ตรวจสอบฤดูกาล

```pascal
program SeasonCheck;
var
  Month: Integer;
begin
  Write('ป้อนเดือน (1-12): ');
  ReadLn(Month);
  
  if (Month >= 1) and (Month <= 12) then
  begin
    if (Month >= 3) and (Month <= 5) then
      WriteLn('ฤดูร้อน (Hot Season)')
    else if (Month >= 6) and (Month <= 10) then
      WriteLn('ฤดูฝน (Rainy Season)')
    else
      WriteLn('ฤดูหนาว (Cool Season)');
  end
  else
    WriteLn('เดือนไม่ถูกต้อง! ต้องอยู่ระหว่าง 1-12');
end.
```

---

## case...of Statement

`case...of` ใช้เมื่อต้องการเปรียบเทียบค่าของตัวแปรกับหลายๆ ค่า มีความชัดเจนและอ่านง่ายกว่าการใช้ if-else ซ้อนกันหลายชั้น

### ไวยากรณ์

```pascal
case <นิพจน์> of
  <ค่า1>: <คำสั่ง1>;
  <ค่า2>: <คำสั่ง2>;
  // ...
  else
    <คำสั่งกรณีอื่นๆ>;
end;
```

### ตัวอย่างที่ 15: วันในสัปดาห์

```pascal
program DayOfWeek;
var
  Day: Integer;
begin
  Write('ป้อนหมายเลขวัน (1-7): ');
  ReadLn(Day);
  
  case Day of
    1: WriteLn('วันจันทร์');
    2: WriteLn('วันอังคาร');
    3: WriteLn('วันพุธ');
    4: WriteLn('วันพฤหัสบดี');
    5: WriteLn('วันศุกร์');
    6: WriteLn('วันเสาร์');
    7: WriteLn('วันอาทิตย์');
  else
    WriteLn('หมายเลขวันไม่ถูกต้อง');
  end;
end.
```

### ตัวอย่างที่ 16: เดือนในปี

```pascal
program MonthName;
var
  Month: Integer;
begin
  Write('ป้อนหมายเลขเดือน (1-12): ');
  ReadLn(Month);
  
  case Month of
    1: WriteLn('มกราคม (January)');
    2: WriteLn('กุมภาพันธ์ (February)');
    3: WriteLn('มีนาคม (March)');
    4: WriteLn('เมษายน (April)');
    5: WriteLn('พฤษภาคม (May)');
    6: WriteLn('มิถุนายน (June)');
    7: WriteLn('กรกฎาคม (July)');
    8: WriteLn('สิงหาคม (August)');
    9: WriteLn('กันยายน (September)');
    10: WriteLn('ตุลาคม (October)');
    11: WriteLn('พฤศจิกายน (November)');
    12: WriteLn('ธันวาคม (December)');
  else
    WriteLn('เดือนไม่ถูกต้อง');
  end;
end.
```

### ตัวอย่างที่ 17: เครื่องคิดเลขอย่างง่าย

```pascal
program SimpleCalc;
var
  A, B: Real;
  Op: Char;
begin
  Write('ป้อนตัวเลขที่ 1: ');
  ReadLn(A);
  Write('ป้อนตัวเลขที่ 2: ');
  ReadLn(B);
  Write('ป้อนตัวดำเนินการ (+, -, *, /): ');
  ReadLn(Op);
  
  case Op of
    '+': WriteLn(A:0:2, ' + ', B:0:2, ' = ', (A + B):0:2);
    '-': WriteLn(A:0:2, ' - ', B:0:2, ' = ', (A - B):0:2);
    '*': WriteLn(A:0:2, ' * ', B:0:2, ' = ', (A * B):0:2);
    '/': 
      begin
        if B <> 0 then
          WriteLn(A:0:2, ' / ', B:0:2, ' = ', (A / B):0:2)
        else
          WriteLn('ไม่สามารถหารด้วยศูนย์ได้!');
      end;
  else
    WriteLn('ตัวดำเนินการไม่ถูกต้อง');
  end;
end.
```

### ตัวอย่างที่ 18: ประเภทของตัวละครในเกม

```pascal
program GameCharacter;
var
  Choice: Integer;
begin
  WriteLn('=== เลือกอาชีพตัวละคร ===');
  WriteLn('1. นักรบ (Warrior)');
  WriteLn('2. นักเวทย์ (Mage)');
  WriteLn('3. นักธนู (Archer)');
  WriteLn('4. นักบวช (Priest)');
  Write('กรุณาเลือก (1-4): ');
  ReadLn(Choice);
  
  case Choice of
    1: begin
         WriteLn('คุณเลือก: นักรบ');
         WriteLn('พลังโจมตีสูง, พลังป้องกันสูง');
         WriteLn('ความเร็วต่ำ');
       end;
    2: begin
         WriteLn('คุณเลือก: นักเวทย์');
         WriteLn('พลังเวทย์สูง');
         WriteLn('พลังป้องกันต่ำ');
       end;
    3: begin
         WriteLn('คุณเลือก: นักธนู');
         WriteLn('ความแม่นยำสูง, ความเร็วสูง');
         WriteLn('พลังป้องกันปานกลาง');
       end;
    4: begin
         WriteLn('คุณเลือก: นักบวช');
         WriteLn('ฟื้นฟูพลังชีวิตได้');
         WriteLn('พลังโจมตีต่ำ');
       end;
  else
    WriteLn('ตัวเลือกไม่ถูกต้อง');
  end;
end.
```

---

## case with Ranges

`case...of` สามารถใช้กับช่วงของค่าได้โดยใช้ `..`

### ตัวอย่างที่ 19: การแปลงคะแนนเป็นเกรด

```pascal
program ScoreToGrade;
var
  Score: Integer;
begin
  Write('ป้อนคะแนน (0-100): ');
  ReadLn(Score);
  
  case Score of
    90..100: WriteLn('เกรด A - ยอดเยี่ยม');
    80..89:  WriteLn('เกรด B - ดีมาก');
    70..79:  WriteLn('เกรด C - ดี');
    60..69:  WriteLn('เกรด D - พอใช้');
    0..59:   WriteLn('เกรด F - ไม่ผ่าน');
  else
    WriteLn('คะแนนไม่ถูกต้อง');
  end;
end.
```

### ตัวอย่างที่ 20: ช่วงอายุ

```pascal
program AgeGroup;
var
  Age: Integer;
begin
  Write('ป้อนอายุ: ');
  ReadLn(Age);
  
  case Age of
    0..2:   WriteLn('ทารก (Infant)');
    3..12:  WriteLn('เด็ก (Child)');
    13..17: WriteLn('วัยรุ่น (Teenager)');
    18..25: WriteLn('วัยหนุ่มสาว (Young Adult)');
    26..59: WriteLn('วัยผู้ใหญ่ (Adult)');
    60..120: WriteLn('วัยสูงอายุ (Senior)');
  else
    WriteLn('อายุไม่ถูกต้อง');
  end;
end.
```

### ตัวอย่างที่ 21: ช่วงอุณหภูมิ

```pascal
program TempRange;
var
  Temp: Integer;
begin
  Write('ป้อนอุณหภูมิ (°C): ');
  ReadLn(Temp);
  
  case Temp of
    Low(Integer)..-1: WriteLn('หนาวมาก! อุณหภูมิต่ำกว่า 0°C');
    0..10: WriteLn('หนาว');
    11..20: WriteLn('เย็นสบาย');
    21..30: WriteLn('อบอุ่น');
    31..40: WriteLn('ร้อน');
    41..High(Integer): WriteLn('ร้อนมาก! ระวังโรคลมแดด');
  end;
end.
```

### ตัวอย่างที่ 22: เกรดตัวอักษรพร้อมคำอธิบาย

```pascal
program GradeDescription;
var
  Grade: Char;
begin
  Write('ป้อนเกรด (A-F): ');
  ReadLn(Grade);
  
  case Grade of
    'A', 'a': begin
                WriteLn('ยอดเยี่ยม - คะแนน 90-100');
                WriteLn('GPA 4.00');
              end;
    'B', 'b': begin
                WriteLn('ดีมาก - คะแนน 80-89');
                WriteLn('GPA 3.00');
              end;
    'C', 'c': begin
                WriteLn('ดี - คะแนน 70-79');
                WriteLn('GPA 2.00');
              end;
    'D', 'd': begin
                WriteLn('พอใช้ - คะแนน 60-69');
                WriteLn('GPA 1.00');
              end;
    'F', 'f': begin
                WriteLn('ไม่ผ่าน - คะแนนต่ำกว่า 60');
                WriteLn('GPA 0.00');
              end;
  else
    WriteLn('เกรดไม่ถูกต้อง');
  end;
end.
```

---

## Nested case Statements

### ตัวอย่างที่ 23: เมนูซ้อนเมนู

```pascal
program NestedMenu;
var
  MainChoice, SubChoice: Integer;
begin
  WriteLn('=== เมนูหลัก ===');
  WriteLn('1. ข้าว');
  WriteLn('2. ก๋วยเตี๋ยว');
  Write('เลือก: ');
  ReadLn(MainChoice);
  
  case MainChoice of
    1: begin
         WriteLn('--- ประเภทข้าว ---');
         WriteLn('1. ข้าวผัด');
         WriteLn('2. ข้าวมันไก่');
         WriteLn('3. ข้าวหมูแดง');
         Write('เลือก: ');
         ReadLn(SubChoice);
         
         case SubChoice of
           1: WriteLn('คุณสั่ง: ข้าวผัด 50 บาท');
           2: WriteLn('คุณสั่ง: ข้าวมันไก่ 45 บาท');
           3: WriteLn('คุณสั่ง: ข้าวหมูแดง 40 บาท');
         else
           WriteLn('ไม่มีเมนูนี้');
         end;
       end;
    2: begin
         WriteLn('--- ประเภทก๋วยเตี๋ยว ---');
         WriteLn('1. ก๋วยเตี๋ยวหมู');
         WriteLn('2. ก๋วยเตี๋ยวเนื้อ');
         WriteLn('3. ก๋วยเตี๋ยวไก่');
         Write('เลือก: ');
         ReadLn(SubChoice);
         
         case SubChoice of
           1: WriteLn('คุณสั่ง: ก๋วยเตี๋ยวหมู 40 บาท');
           2: WriteLn('คุณสั่ง: ก๋วยเตี๋ยวเนื้อ 55 บาท');
           3: WriteLn('คุณสั่ง: ก๋วยเตี๋ยวไก่ 40 บาท');
         else
           WriteLn('ไม่มีเมนูนี้');
         end;
       end;
  else
    WriteLn('ตัวเลือกไม่ถูกต้อง');
  end;
end.
```

---

## goto Statement

### คำเตือน

`goto` เป็นคำสั่งที่ทำให้โปรแกรมกระโดดไปยังตำแหน่ง (label) ที่กำหนด แต่**ไม่แนะนำ**ให้ใช้เพราะ:

1. ทำให้โค้ดอ่านยากและซับซ้อน (Spaghetti Code)
2. ยากต่อการ debug
3. มีโครงสร้างอื่นที่ดีกว่าทดแทนได้
4. ละเมิดหลักการ structured programming

### ไวยากรณ์

```pascal
label
  <ชื่อ label>;
  
goto <ชื่อ label>;

<ชื่อ label>:
  // คำสั่ง
```

### ตัวอย่างที่ 24: การใช้ goto (ตัวอย่างที่ควรหลีกเลี่ยง)

```pascal
program GotoExample;
// ตัวอย่างนี้แสดงให้เห็นว่า goto ทำงานอย่างไร
// แต่ในทางปฏิบัติควรใช้ while loop แทน

label
  StartLoop, EndLoop;
var
  I: Integer;
begin
  I := 1;
  
StartLoop:
  if I > 5 then goto EndLoop;
  
  WriteLn('I = ', I);
  Inc(I);
  goto StartLoop;
  
EndLoop:
  WriteLn('จบการทำงาน');
end.
```

### ตัวอย่างที่ 25: วิธีที่ดีกว่า (ใช้ while แทน goto)

```pascal
program BetterLoop;
// ใช้ while loop แทน goto
var
  I: Integer;
begin
  I := 1;
  
  while I <= 5 do
  begin
    WriteLn('I = ', I);
    Inc(I);
  end;
  
  WriteLn('จบการทำงาน');
end.
```

---

## โปรแกรมตัวอย่าง: เกรดนักเรียน

```pascal
program StudentGradeSystem;
var
  StudentName: String;
  MidtermScore, FinalScore, AssignmentScore: Real;
  TotalScore, WeightedScore: Real;
  Grade: String;
  GPA: Real;
begin
  WriteLn('========================================');
  WriteLn('   ระบบคำนวณเกรดนักเรียน');
  WriteLn('========================================');
  WriteLn;
  
  Write('ชื่อนักเรียน: ');
  ReadLn(StudentName);
  
  WriteLn;
  WriteLn('กรุณาป้อนคะแนน:');
  
  repeat
    Write('คะแนนสอบกลางภาค (0-100): ');
    ReadLn(MidtermScore);
    if (MidtermScore < 0) or (MidtermScore > 100) then
      WriteLn('คะแนนต้องอยู่ระหว่าง 0-100');
  until (MidtermScore >= 0) and (MidtermScore <= 100);
  
  repeat
    Write('คะแนนสอบปลายภาค (0-100): ');
    ReadLn(FinalScore);
    if (FinalScore < 0) or (FinalScore > 100) then
      WriteLn('คะแนนต้องอยู่ระหว่าง 0-100');
  until (FinalScore >= 0) and (FinalScore <= 100);
  
  repeat
    Write('คะแนนการบ้าน/งาน (0-100): ');
    ReadLn(AssignmentScore);
    if (AssignmentScore < 0) or (AssignmentScore > 100) then
      WriteLn('คะแนนต้องอยู่ระหว่าง 0-100');
  until (AssignmentScore >= 0) and (AssignmentScore <= 100);
  
  // คำนวณคะแนนรวม (กลางภาค 30%, ปลายภาค 50%, งาน 20%)
  WeightedScore := (MidtermScore * 0.30) + 
                   (FinalScore * 0.50) + 
                   (AssignmentScore * 0.20);
  
  // กำหนดเกรดและ GPA
  if WeightedScore >= 80 then
  begin
    Grade := 'A';
    GPA := 4.00;
  end
  else if WeightedScore >= 75 then
  begin
    Grade := 'B+';
    GPA := 3.50;
  end
  else if WeightedScore >= 70 then
  begin
    Grade := 'B';
    GPA := 3.00;
  end
  else if WeightedScore >= 65 then
  begin
    Grade := 'C+';
    GPA := 2.50;
  end
  else if WeightedScore >= 60 then
  begin
    Grade := 'C';
    GPA := 2.00;
  end
  else if WeightedScore >= 55 then
  begin
    Grade := 'D+';
    GPA := 1.50;
  end
  else if WeightedScore >= 50 then
  begin
    Grade := 'D';
    GPA := 1.00;
  end
  else
  begin
    Grade := 'F';
    GPA := 0.00;
  end;
  
  // แสดงผล
  WriteLn;
  WriteLn('========================================');
  WriteLn('   ผลการเรียนของ: ', StudentName);
  WriteLn('========================================');
  WriteLn('คะแนนกลางภาค: ', MidtermScore:0:2, ' (น้ำหนัก 30%)');
  WriteLn('คะแนนปลายภาค: ', FinalScore:0:2, ' (น้ำหนัก 50%)');
  WriteLn('คะแนนงาน:     ', AssignmentScore:0:2, ' (น้ำหนัก 20%)');
  WriteLn('----------------------------------------');
  WriteLn('คะแนนรวม:     ', WeightedScore:0:2);
  WriteLn('เกรด:          ', Grade);
  WriteLn('GPA:           ', GPA:0:2);
  WriteLn('----------------------------------------');
  
  if Grade = 'F' then
    WriteLn('สถานะ: ไม่ผ่าน - ต้องเรียนซ้ำ')
  else
    WriteLn('สถานะ: ผ่าน - ยินดีด้วย!');
    
  WriteLn('========================================');
end.
```

---

## โปรแกรมตัวอย่าง: เครื่องคิดเลข

```pascal
program AdvancedCalculator;
uses
  Math;
  
var
  Num1, Num2, Result: Real;
  Choice: Integer;
  Continue: Boolean;
begin
  Continue := True;
  
  while Continue do
  begin
    WriteLn;
    WriteLn('╔════════════════════════╗');
    WriteLn('║      เครื่องคิดเลข     ║');
    WriteLn('╠════════════════════════╣');
    WriteLn('║ 1. บวก (+)             ║');
    WriteLn('║ 2. ลบ (-)              ║');
    WriteLn('║ 3. คูณ (*)             ║');
    WriteLn('║ 4. หาร (/)             ║');
    WriteLn('║ 5. ยกกำลัง (^)         ║');
    WriteLn('║ 6. รากที่สอง (√)       ║');
    WriteLn('║ 7. หารเอาเศษ (mod)     ║');
    WriteLn('║ 0. ออกจากโปรแกรม       ║');
    WriteLn('╚════════════════════════╝');
    Write('เลือกการดำเนินการ: ');
    ReadLn(Choice);
    
    if Choice = 0 then
    begin
      WriteLn('ขอบคุณที่ใช้งาน!');
      Continue := False;
    end
    else
    begin
      if Choice = 6 then
      begin
        Write('ป้อนตัวเลข: ');
        ReadLn(Num1);
        
        if Num1 >= 0 then
        begin
          Result := Sqrt(Num1);
          WriteLn('√', Num1:0:2, ' = ', Result:0:4);
        end
        else
          WriteLn('ไม่สามารถหารากที่สองของจำนวนลบได้!');
      end
      else
      begin
        Write('ป้อนตัวเลขที่ 1: ');
        ReadLn(Num1);
        Write('ป้อนตัวเลขที่ 2: ');
        ReadLn(Num2);
        
        case Choice of
          1: begin
               Result := Num1 + Num2;
               WriteLn(Num1:0:2, ' + ', Num2:0:2, ' = ', Result:0:2);
             end;
          2: begin
               Result := Num1 - Num2;
               WriteLn(Num1:0:2, ' - ', Num2:0:2, ' = ', Result:0:2);
             end;
          3: begin
               Result := Num1 * Num2;
               WriteLn(Num1:0:2, ' × ', Num2:0:2, ' = ', Result:0:2);
             end;
          4: begin
               if Num2 <> 0 then
               begin
                 Result := Num1 / Num2;
                 WriteLn(Num1:0:2, ' ÷ ', Num2:0:2, ' = ', Result:0:4);
               end
               else
                 WriteLn('ข้อผิดพลาด: ไม่สามารถหารด้วยศูนย์ได้!');
             end;
          5: begin
               Result := Power(Num1, Num2);
               WriteLn(Num1:0:2, ' ^ ', Num2:0:2, ' = ', Result:0:4);
             end;
          7: begin
               if (Num2 <> 0) and (Trunc(Num1) = Num1) and (Trunc(Num2) = Num2) then
               begin
                 Result := Trunc(Num1) mod Trunc(Num2);
                 WriteLn(Num1:0:0, ' mod ', Num2:0:0, ' = ', Result:0:0);
               end
               else
                 WriteLn('การหารเอาเศษใช้ได้กับจำนวนเต็มเท่านั้น');
             end;
        else
          WriteLn('ตัวเลือกไม่ถูกต้อง');
        end;
      end;
    end;
  end;
end.
```

---

## โปรแกรมตัวอย่าง: ระบบเมนู

```pascal
program RestaurantMenu;
var
  MainChoice: Integer;
  Quantity: Integer;
  TotalPrice: Real;
  Running: Boolean;
  
  // ราคาอาหาร
const
  PRICE_RICE_FRIED = 60.0;
  PRICE_RICE_CHICKEN = 55.0;
  PRICE_NOODLE_PORK = 50.0;
  PRICE_NOODLE_BEEF = 65.0;
  PRICE_TOM_YUM = 120.0;
  PRICE_PAD_THAI = 80.0;
  PRICE_WATER = 15.0;
  PRICE_SOFT_DRINK = 25.0;
  PRICE_JUICE = 35.0;
  
begin
  Running := True;
  TotalPrice := 0;
  
  WriteLn('╔══════════════════════════════════╗');
  WriteLn('║     ร้านอาหารไทยแสนอร่อย         ║');
  WriteLn('╚══════════════════════════════════╝');
  WriteLn;
  
  while Running do
  begin
    WriteLn('=== เมนูหลัก ===');
    WriteLn('1. อาหารจานข้าว');
    WriteLn('2. ก๋วยเตี๋ยว');
    WriteLn('3. อาหารจานพิเศษ');
    WriteLn('4. เครื่องดื่ม');
    WriteLn('5. ดูยอดรวม');
    WriteLn('0. ชำระเงิน/ออก');
    WriteLn;
    Write('เลือกหมวดหมู่: ');
    ReadLn(MainChoice);
    
    case MainChoice of
      1: begin
           WriteLn;
           WriteLn('--- อาหารจานข้าว ---');
           WriteLn('1. ข้าวผัด          60 บาท');
           WriteLn('2. ข้าวมันไก่        55 บาท');
           Write('เลือก: ');
           var SubChoice: Integer;
           ReadLn(SubChoice);
           Write('จำนวน: ');
           ReadLn(Quantity);
           
           case SubChoice of
             1: begin
                  TotalPrice := TotalPrice + (PRICE_RICE_FRIED * Quantity);
                  WriteLn('เพิ่ม ข้าวผัด x', Quantity, ' = ', PRICE_RICE_FRIED * Quantity:0:2, ' บาท');
                end;
             2: begin
                  TotalPrice := TotalPrice + (PRICE_RICE_CHICKEN * Quantity);
                  WriteLn('เพิ่ม ข้าวมันไก่ x', Quantity, ' = ', PRICE_RICE_CHICKEN * Quantity:0:2, ' บาท');
                end;
           else
             WriteLn('ไม่มีเมนูนี้');
           end;
         end;
         
      2: begin
           WriteLn;
           WriteLn('--- ก๋วยเตี๋ยว ---');
           WriteLn('1. ก๋วยเตี๋ยวหมู     50 บาท');
           WriteLn('2. ก๋วยเตี๋ยวเนื้อ   65 บาท');
           Write('เลือก: ');
           var SubChoice2: Integer;
           ReadLn(SubChoice2);
           Write('จำนวน: ');
           ReadLn(Quantity);
           
           case SubChoice2 of
             1: begin
                  TotalPrice := TotalPrice + (PRICE_NOODLE_PORK * Quantity);
                  WriteLn('เพิ่ม ก๋วยเตี๋ยวหมู x', Quantity);
                end;
             2: begin
                  TotalPrice := TotalPrice + (PRICE_NOODLE_BEEF * Quantity);
                  WriteLn('เพิ่ม ก๋วยเตี๋ยวเนื้อ x', Quantity);
                end;
           else
             WriteLn('ไม่มีเมนูนี้');
           end;
         end;
         
      3: begin
           WriteLn;
           WriteLn('--- อาหารจานพิเศษ ---');
           WriteLn('1. ต้มยำกุ้ง       120 บาท');
           WriteLn('2. ผัดไทย           80 บาท');
           Write('เลือก: ');
           var SubChoice3: Integer;
           ReadLn(SubChoice3);
           Write('จำนวน: ');
           ReadLn(Quantity);
           
           case SubChoice3 of
             1: begin
                  TotalPrice := TotalPrice + (PRICE_TOM_YUM * Quantity);
                  WriteLn('เพิ่ม ต้มยำกุ้ง x', Quantity);
                end;
             2: begin
                  TotalPrice := TotalPrice + (PRICE_PAD_THAI * Quantity);
                  WriteLn('เพิ่ม ผัดไทย x', Quantity);
                end;
           else
             WriteLn('ไม่มีเมนูนี้');
           end;
         end;
         
      4: begin
           WriteLn;
           WriteLn('--- เครื่องดื่ม ---');
           WriteLn('1. น้ำเปล่า         15 บาท');
           WriteLn('2. น้ำอัดลม         25 บาท');
           WriteLn('3. น้ำผลไม้         35 บาท');
           Write('เลือก: ');
           var SubChoice4: Integer;
           ReadLn(SubChoice4);
           Write('จำนวน: ');
           ReadLn(Quantity);
           
           case SubChoice4 of
             1: TotalPrice := TotalPrice + (PRICE_WATER * Quantity);
             2: TotalPrice := TotalPrice + (PRICE_SOFT_DRINK * Quantity);
             3: TotalPrice := TotalPrice + (PRICE_JUICE * Quantity);
           else
             WriteLn('ไม่มีเมนูนี้');
           end;
           WriteLn('เพิ่มเครื่องดื่ม x', Quantity);
         end;
         
      5: begin
           WriteLn;
           WriteLn('ยอดรวมปัจจุบัน: ', TotalPrice:0:2, ' บาท');
         end;
         
      0: begin
           WriteLn;
           WriteLn('=============================');
           WriteLn('   ใบเสร็จรับเงิน');
           WriteLn('=============================');
           WriteLn('ยอดรวม:    ', TotalPrice:0:2, ' บาท');
           WriteLn('ภาษี 7%:   ', (TotalPrice * 0.07):0:2, ' บาท');
           WriteLn('รวมทั้งสิ้น:', (TotalPrice * 1.07):0:2, ' บาท');
           WriteLn('=============================');
           WriteLn('ขอบคุณที่ใช้บริการ!');
           Running := False;
         end;
    else
      WriteLn('ตัวเลือกไม่ถูกต้อง กรุณาลองใหม่');
    end;
    
    WriteLn;
  end;
end.
```

---

## โปรแกรมตัวอย่าง: BMI Calculator

```pascal
program BMICalculator;
var
  Weight, Height, BMI: Real;
  Age: Integer;
  Gender: Char;
  Category: String;
  HealthAdvice: String;
begin
  WriteLn('╔════════════════════════════════╗');
  WriteLn('║     โปรแกรมคำนวณ BMI          ║');
  WriteLn('╚════════════════════════════════╝');
  WriteLn;
  WriteLn('BMI (Body Mass Index) = น้ำหนัก(kg) / ส่วนสูง(m)²');
  WriteLn;
  
  Write('ชื่อ: ');
  var Name: String;
  ReadLn(Name);
  
  Write('อายุ: ');
  ReadLn(Age);
  
  Write('เพศ (M/F): ');
  ReadLn(Gender);
  
  repeat
    Write('น้ำหนัก (กิโลกรัม): ');
    ReadLn(Weight);
    if Weight <= 0 then
      WriteLn('น้ำหนักต้องมากกว่า 0');
  until Weight > 0;
  
  repeat
    Write('ส่วนสูง (เซนติเมตร): ');
    ReadLn(Height);
    if Height <= 0 then
      WriteLn('ส่วนสูงต้องมากกว่า 0');
  until Height > 0;
  
  // แปลงส่วนสูงจาก cm เป็น m
  Height := Height / 100;
  
  // คำนวณ BMI
  BMI := Weight / (Height * Height);
  
  // จำแนกประเภท BMI (เกณฑ์สำหรับคนเอเชีย)
  if BMI < 18.5 then
  begin
    Category := 'น้ำหนักน้อย (Underweight)';
    HealthAdvice := 'ควรเพิ่มน้ำหนักโดยรับประทานอาหารที่มีประโยชน์';
  end
  else if BMI < 23.0 then
  begin
    Category := 'น้ำหนักปกติ (Normal)';
    HealthAdvice := 'ดูแลสุขภาพให้อยู่ในเกณฑ์นี้ต่อไป';
  end
  else if BMI < 25.0 then
  begin
    Category := 'น้ำหนักเกิน (Overweight)';
    HealthAdvice := 'ควรออกกำลังกายและควบคุมอาหาร';
  end
  else if BMI < 30.0 then
  begin
    Category := 'อ้วน (Obese Class I)';
    HealthAdvice := 'ควรพบแพทย์เพื่อปรึกษาโปรแกรมลดน้ำหนัก';
  end
  else
  begin
    Category := 'อ้วนมาก (Obese Class II+)';
    HealthAdvice := 'จำเป็นต้องพบแพทย์เพื่อรักษา';
  end;
  
  // แสดงผล
  WriteLn;
  WriteLn('════════════════════════════════');
  WriteLn('ผลการประเมิน BMI ของ ', Name);
  WriteLn('════════════════════════════════');
  
  if (Gender = 'M') or (Gender = 'm') then
    WriteLn('เพศ: ชาย')
  else
    WriteLn('เพศ: หญิง');
    
  WriteLn('อายุ: ', Age, ' ปี');
  WriteLn('น้ำหนัก: ', Weight:0:1, ' kg');
  WriteLn('ส่วนสูง: ', Height * 100:0:1, ' cm');
  WriteLn('BMI: ', BMI:0:2);
  WriteLn;
  WriteLn('ประเภท: ', Category);
  WriteLn;
  WriteLn('คำแนะนำ: ', HealthAdvice);
  WriteLn;
  
  // แสดงตารางอ้างอิง BMI
  WriteLn('════════════════════════════════');
  WriteLn('ตารางอ้างอิง BMI (เอเชีย):');
  WriteLn('< 18.5    : น้ำหนักน้อยเกินไป');
  WriteLn('18.5-22.9 : น้ำหนักปกติ');
  WriteLn('23.0-24.9 : น้ำหนักเกิน');
  WriteLn('25.0-29.9 : อ้วน');
  WriteLn('>= 30.0   : อ้วนมาก');
  WriteLn('════════════════════════════════');
end.
```

---

## แบบฝึกหัด

### ข้อที่ 1
เขียนโปรแกรมรับตัวเลข 3 ตัว และแสดงตัวเลขที่มากที่สุด

**เฉลย:**
```pascal
program MaxOfThree;
var
  A, B, C, Max: Integer;
begin
  Write('ป้อนตัวเลข 3 ตัว: ');
  ReadLn(A, B, C);
  
  Max := A;
  if B > Max then Max := B;
  if C > Max then Max := C;
  
  WriteLn('ตัวเลขที่มากที่สุดคือ: ', Max);
end.
```

### ข้อที่ 2
เขียนโปรแกรมตรวจสอบว่าตัวเลขที่รับเข้ามาเป็นเลขคู่หรือเลขคี่

**เฉลย:**
```pascal
program EvenOrOdd;
var
  N: Integer;
begin
  Write('ป้อนตัวเลข: ');
  ReadLn(N);
  
  if N mod 2 = 0 then
    WriteLn(N, ' เป็นเลขคู่')
  else
    WriteLn(N, ' เป็นเลขคี่');
end.
```

### ข้อที่ 3
เขียนโปรแกรมรับเดือน (1-12) และแสดงจำนวนวันในเดือนนั้น

**เฉลย:**
```pascal
program DaysInMonth;
var
  Month, Year, Days: Integer;
begin
  Write('ป้อนเดือน (1-12): ');
  ReadLn(Month);
  Write('ป้อนปี: ');
  ReadLn(Year);
  
  case Month of
    1, 3, 5, 7, 8, 10, 12: Days := 31;
    4, 6, 9, 11: Days := 30;
    2: begin
         if ((Year mod 4 = 0) and (Year mod 100 <> 0)) or
            (Year mod 400 = 0) then
           Days := 29
         else
           Days := 28;
       end;
  else
    Days := 0;
  end;
  
  if Days > 0 then
    WriteLn('เดือน ', Month, ' ปี ', Year, ' มี ', Days, ' วัน')
  else
    WriteLn('เดือนไม่ถูกต้อง');
end.
```

### ข้อที่ 4
เขียนโปรแกรมแปลงหน่วยอุณหภูมิ (เซลเซียส, ฟาเรนไฮต์, เคลวิน)

**เฉลย:**
```pascal
program TempConvert;
var
  Temp: Real;
  FromUnit, ToUnit: Integer;
  Result: Real;
begin
  WriteLn('แปลงหน่วยอุณหภูมิ');
  WriteLn('1. เซลเซียส (°C)');
  WriteLn('2. ฟาเรนไฮต์ (°F)');
  WriteLn('3. เคลวิน (K)');
  
  Write('จากหน่วย: ');
  ReadLn(FromUnit);
  Write('ป้อนอุณหภูมิ: ');
  ReadLn(Temp);
  Write('เป็นหน่วย: ');
  ReadLn(ToUnit);
  
  // แปลงเป็นเซลเซียสก่อน
  case FromUnit of
    1: Result := Temp;
    2: Result := (Temp - 32) * 5 / 9;
    3: Result := Temp - 273.15;
  else
    Result := Temp;
  end;
  
  // แปลงจากเซลเซียสเป็นหน่วยที่ต้องการ
  case ToUnit of
    1: WriteLn(Result:0:2, ' °C');
    2: WriteLn((Result * 9/5 + 32):0:2, ' °F');
    3: WriteLn((Result + 273.15):0:2, ' K');
  else
    WriteLn('หน่วยไม่ถูกต้อง');
  end;
end.
```

### ข้อที่ 5
เขียนโปรแกรมตรวจสอบว่าอักขระที่รับเข้ามาเป็น สระ พยัญชนะ ตัวเลข หรืออักขระพิเศษ

**เฉลย:**
```pascal
program CharType;
var
  Ch: Char;
begin
  Write('ป้อนอักขระ: ');
  ReadLn(Ch);
  
  if Ch in ['a', 'e', 'i', 'o', 'u', 'A', 'E', 'I', 'O', 'U'] then
    WriteLn('"', Ch, '" เป็นสระ (Vowel)')
  else if Ch in ['a'..'z', 'A'..'Z'] then
    WriteLn('"', Ch, '" เป็นพยัญชนะ (Consonant)')
  else if Ch in ['0'..'9'] then
    WriteLn('"', Ch, '" เป็นตัวเลข (Digit)')
  else
    WriteLn('"', Ch, '" เป็นอักขระพิเศษ (Special Character)');
end.
```

### ข้อที่ 6
เขียนโปรแกรมคำนวณค่าไฟฟ้าตามอัตราก้าวหน้า

**เฉลย:**
```pascal
program ElectricBill;
var
  Units: Real;
  Bill: Real;
begin
  Write('ป้อนหน่วยไฟฟ้าที่ใช้: ');
  ReadLn(Units);
  
  // อัตราค่าไฟฟ้าแบบง่าย
  if Units <= 150 then
    Bill := Units * 2.50
  else if Units <= 400 then
    Bill := (150 * 2.50) + ((Units - 150) * 3.30)
  else
    Bill := (150 * 2.50) + (250 * 3.30) + ((Units - 400) * 4.20);
    
  Bill := Bill + 40.50; // ค่าบริการ
  
  WriteLn('ค่าไฟฟ้า: ', Bill:0:2, ' บาท');
end.
```

### ข้อที่ 7
เขียนโปรแกรมแสดงชื่อวันที่จากตัวเลขวันที่ เดือน และปี

**เฉลย:**
```pascal
program DayOfWeekCalc;
var
  D, M, Y: Integer;
  DayNum: Integer;
  // ใช้ Zeller's formula แบบง่าย
begin
  Write('ป้อนวัน: ');
  ReadLn(D);
  Write('ป้อนเดือน (1-12): ');
  ReadLn(M);
  Write('ป้อนปี: ');
  ReadLn(Y);
  
  // Tomohiko Sakamoto algorithm
  var T: array[0..11] of Integer = (0, 3, 2, 5, 0, 3, 5, 1, 4, 6, 2, 4);
  if M < 3 then Dec(Y);
  DayNum := (Y + Y div 4 - Y div 100 + Y div 400 + T[M-1] + D) mod 7;
  
  case DayNum of
    0: WriteLn('วันอาทิตย์');
    1: WriteLn('วันจันทร์');
    2: WriteLn('วันอังคาร');
    3: WriteLn('วันพุธ');
    4: WriteLn('วันพฤหัสบดี');
    5: WriteLn('วันศุกร์');
    6: WriteLn('วันเสาร์');
  end;
end.
```

### ข้อที่ 8
เขียนโปรแกรมคิดราคาตั๋วหนัง

**เฉลย:**
```pascal
program MovieTicket;
var
  Age: Integer;
  TicketType: Integer;
  Price: Real;
  Discount: Real;
begin
  WriteLn('=== ราคาตั๋วหนัง ===');
  WriteLn('1. รอบปกติ');
  WriteLn('2. รอบพิเศษ (IMAX/4DX)');
  WriteLn('3. รอบเช้า');
  Write('เลือกประเภท: ');
  ReadLn(TicketType);
  Write('อายุผู้ชม: ');
  ReadLn(Age);
  
  case TicketType of
    1: Price := 200;
    2: Price := 350;
    3: Price := 150;
  else
    Price := 200;
  end;
  
  if Age < 3 then
    Discount := 1.0    // ฟรี
  else if Age <= 12 then
    Discount := 0.5    // ลด 50%
  else if Age >= 60 then
    Discount := 0.3    // ลด 30%
  else
    Discount := 0.0;   // ไม่มีส่วนลด
  
  Price := Price * (1 - Discount);
  
  WriteLn('ราคาตั๋ว: ', Price:0:2, ' บาท');
  if Discount > 0 then
    WriteLn('(ได้รับส่วนลด ', Discount * 100:0:0, '%)');
end.
```

### ข้อที่ 9
เขียนโปรแกรมตรวจสอบรหัสผ่านง่ายๆ

**เฉลย:**
```pascal
program SimplePasswordCheck;
const
  CORRECT_PASSWORD = 'Pascal2024';
var
  Password: String;
  Attempts: Integer;
begin
  Attempts := 0;
  
  repeat
    Inc(Attempts);
    Write('ป้อนรหัสผ่าน: ');
    ReadLn(Password);
    
    if Password = CORRECT_PASSWORD then
      WriteLn('เข้าสู่ระบบสำเร็จ!')
    else
    begin
      WriteLn('รหัสผ่านไม่ถูกต้อง');
      if Attempts >= 3 then
        WriteLn('คุณป้อนรหัสผ่านผิดเกิน 3 ครั้ง!');
    end;
    
  until (Password = CORRECT_PASSWORD) or (Attempts >= 3);
end.
```

### ข้อที่ 10
เขียนโปรแกรมแปลงเลขฐานสิบเป็น binary (ใช้ if)

**เฉลย:**
```pascal
program DecToBin;
var
  N: Integer;
  Binary: String;
begin
  Write('ป้อนตัวเลขฐานสิบ (0-255): ');
  ReadLn(N);
  
  if (N < 0) or (N > 255) then
    WriteLn('ตัวเลขต้องอยู่ระหว่าง 0-255')
  else
  begin
    Binary := '';
    var Temp := N;
    
    if Temp = 0 then
      Binary := '0'
    else
    begin
      while Temp > 0 do
      begin
        if Temp mod 2 = 0 then
          Binary := '0' + Binary
        else
          Binary := '1' + Binary;
        Temp := Temp div 2;
      end;
    end;
    
    WriteLn(N, ' (ฐาน10) = ', Binary, ' (ฐาน2)');
  end;
end.
```

### ข้อที่ 11-20 (ข้อสั้น)

```pascal
// ข้อ 11: แสดงสัญลักษณ์คณิตศาสตร์ตามตัวเลือก
program MathSymbol;
var C: Integer;
begin
  Write('เลือก (1=บวก, 2=ลบ, 3=คูณ, 4=หาร): ');
  ReadLn(C);
  case C of
    1: WriteLn('+');
    2: WriteLn('-');
    3: WriteLn('×');
    4: WriteLn('÷');
  else WriteLn('ไม่พบสัญลักษณ์');
  end;
end.

// ข้อ 12: ตรวจสอบว่าตัวเลขหารด้วย 3 และ 5 ลงตัวหรือไม่
program FizzBuzz;
var N: Integer;
begin
  Write('ป้อนตัวเลข: ');
  ReadLn(N);
  if (N mod 3 = 0) and (N mod 5 = 0) then WriteLn('FizzBuzz')
  else if N mod 3 = 0 then WriteLn('Fizz')
  else if N mod 5 = 0 then WriteLn('Buzz')
  else WriteLn(N);
end.

// ข้อ 13: คำนวณค่าเฉลี่ยและบอกว่าสูงหรือต่ำกว่าค่าเฉลี่ย
program AboveAverage;
var A, B, C: Real; Avg: Real;
begin
  Write('ป้อนคะแนน 3 วิชา: ');
  ReadLn(A, B, C);
  Avg := (A + B + C) / 3;
  WriteLn('ค่าเฉลี่ย: ', Avg:0:2);
  if A > Avg then WriteLn('วิชา 1 สูงกว่าค่าเฉลี่ย');
  if B > Avg then WriteLn('วิชา 2 สูงกว่าค่าเฉลี่ย');
  if C > Avg then WriteLn('วิชา 3 สูงกว่าค่าเฉลี่ย');
end.

// ข้อ 14: แสดงเดือนที่มีวันน้อยกว่า 31 วัน
program ShortMonths;
var M: Integer;
begin
  for M := 1 to 12 do
  begin
    case M of
      4, 6, 9, 11: WriteLn('เดือน ', M, ' มี 30 วัน');
      2: WriteLn('เดือน ', M, ' มี 28/29 วัน');
    end;
  end;
end.

// ข้อ 15: ตรวจสอบสี่เหลี่ยม
program QuadCheck;
var A, B, C, D: Real;
begin
  Write('ป้อนด้าน 4 ด้าน: ');
  ReadLn(A, B, C, D);
  if (A = B) and (B = C) and (C = D) then WriteLn('สี่เหลี่ยมจัตุรัส')
  else if (A = C) and (B = D) then WriteLn('สี่เหลี่ยมผืนผ้า')
  else WriteLn('สี่เหลี่ยมทั่วไป');
end.

// ข้อ 16: ระบบป้อนคะแนน 5 วิชา
program FiveSubjects;
var S1, S2, S3, S4, S5: Real; Total, Avg: Real;
begin
  Write('คะแนน 5 วิชา: ');
  ReadLn(S1, S2, S3, S4, S5);
  Total := S1 + S2 + S3 + S4 + S5;
  Avg := Total / 5;
  WriteLn('รวม: ', Total:0:2);
  WriteLn('เฉลี่ย: ', Avg:0:2);
  if Avg >= 50 then WriteLn('ผ่าน') else WriteLn('ไม่ผ่าน');
end.

// ข้อ 17: แปลงระดับเสียงเป็นชื่อโน้ต
program NoteConvert;
var Freq: Real;
begin
  Write('ป้อนความถี่ (Hz): ');
  ReadLn(Freq);
  if (Freq >= 261) and (Freq < 294) then WriteLn('โน้ต Do')
  else if (Freq >= 294) and (Freq < 330) then WriteLn('โน้ต Re')
  else if (Freq >= 330) and (Freq < 349) then WriteLn('โน้ต Mi')
  else if (Freq >= 349) and (Freq < 392) then WriteLn('โน้ต Fa')
  else if (Freq >= 392) and (Freq < 440) then WriteLn('โน้ต Sol')
  else if (Freq >= 440) and (Freq < 494) then WriteLn('โน้ต La')
  else if (Freq >= 494) and (Freq < 523) then WriteLn('โน้ต Si')
  else WriteLn('ไม่อยู่ในช่วงโน้ตมาตรฐาน');
end.

// ข้อ 18: คำนวณค่าจ้างโอที
program OvertimePay;
var Hours: Real; Rate: Real; Pay: Real;
begin
  Write('ชั่วโมงทำงาน: '); ReadLn(Hours);
  Write('อัตราค่าจ้างต่อชั่วโมง: '); ReadLn(Rate);
  if Hours <= 8 then
    Pay := Hours * Rate
  else
    Pay := (8 * Rate) + ((Hours - 8) * Rate * 1.5);
  WriteLn('ค่าจ้าง: ', Pay:0:2, ' บาท');
end.

// ข้อ 19: ระบบลดราคาตามจำนวนสินค้า
program BulkDiscount;
var Qty: Integer; Price: Real; Discount: Real;
begin
  Write('จำนวนสินค้า: '); ReadLn(Qty);
  Write('ราคาต่อชิ้น: '); ReadLn(Price);
  if Qty >= 100 then Discount := 0.20
  else if Qty >= 50 then Discount := 0.15
  else if Qty >= 20 then Discount := 0.10
  else if Qty >= 10 then Discount := 0.05
  else Discount := 0;
  WriteLn('ส่วนลด: ', Discount * 100:0:0, '%');
  WriteLn('ราคารวม: ', (Qty * Price * (1 - Discount)):0:2, ' บาท');
end.

// ข้อ 20: ตรวจสอบรูปแบบของตัวเลข
program NumberType;
var N: Real;
begin
  Write('ป้อนตัวเลข: '); ReadLn(N);
  if N > 0 then WriteLn('บวก')
  else if N < 0 then WriteLn('ลบ')
  else WriteLn('ศูนย์');
  if Frac(N) = 0 then WriteLn('จำนวนเต็ม')
  else WriteLn('จำนวนทศนิยม');
end.
```

---

## สรุป

| หัวข้อ | เมื่อควรใช้ |
|--------|------------|
| `if...then` | ตรวจสอบเงื่อนไขเดียว |
| `if...then...else` | มีสองทางเลือก |
| `nested if` | เงื่อนไขซับซ้อนหลายชั้น |
| `case...of` | เปรียบเทียบกับค่าหลายค่า/ช่วง |
| `goto` | หลีกเลี่ยง! ใช้ loop แทน |

---

*จบ Part 06 - การควบคุมการทำงาน*
