# Part 08 - อาร์เรย์ (Arrays)

## สารบัญ
1. [การประกาศ Array](#การประกาศ-array)
2. [Static Arrays](#static-arrays)
3. [Dynamic Arrays](#dynamic-arrays)
4. [Multidimensional Arrays](#multidimensional-arrays)
5. [Array of Arrays (Jagged Arrays)](#array-of-arrays)
6. [Array กับ Loops](#array-กับ-loops)
7. [Array Sorting](#array-sorting)
8. [Array Searching](#array-searching)
9. [Open Array Parameters](#open-array-parameters)
10. [Array Slicing](#array-slicing)
11. [โปรแกรมตัวอย่าง](#โปรแกรมตัวอย่าง)
12. [แบบฝึกหัด 20 ข้อ](#แบบฝึกหัด)

---

## บทนำ

Array (อาร์เรย์) คือโครงสร้างข้อมูลที่เก็บข้อมูลหลายค่าชนิดเดียวกันในตัวแปรเดียว โดยเข้าถึงข้อมูลแต่ละตัวผ่าน index (ดัชนี)

```
Array ชื่อ Scores:
Index:  [1]  [2]  [3]  [4]  [5]
Value:   85   92   78   95   88
```

### ข้อดีของ Array
- เก็บข้อมูลหลายค่าในตัวแปรเดียว
- เข้าถึงข้อมูลได้รวดเร็วผ่าน index
- ทำงานร่วมกับ loop ได้ดี
- ประหยัดหน่วยความจำ

---

## การประกาศ Array

### รูปแบบพื้นฐาน

```pascal
var
  ArrayName: array[IndexStart..IndexEnd] of DataType;
```

### ตัวอย่างที่ 1: ประกาศ array พื้นฐาน

```pascal
program ArrayDeclaration;
var
  // Array ของจำนวนเต็ม index 1-5
  Numbers: array[1..5] of Integer;
  
  // Array ของจำนวนทศนิยม index 0-9
  Prices: array[0..9] of Real;
  
  // Array ของ Boolean
  Flags: array[1..10] of Boolean;
  
  // Array ของอักขระ
  Letters: array['A'..'Z'] of Integer;
  
  I: Integer;
begin
  // กำหนดค่า
  Numbers[1] := 10;
  Numbers[2] := 20;
  Numbers[3] := 30;
  Numbers[4] := 40;
  Numbers[5] := 50;
  
  // แสดงค่า
  for I := 1 to 5 do
    WriteLn('Numbers[', I, '] = ', Numbers[I]);
end.
```

### ตัวอย่างที่ 2: Array ที่ประกาศในส่วน type

```pascal
program TypedArray;
type
  TScoreArray = array[1..30] of Integer;
  TNameArray = array[1..30] of String[50];
  
var
  Scores: TScoreArray;
  Names: TNameArray;
  N: Integer;
  I: Integer;
begin
  Write('จำนวนนักเรียน (1-30): ');
  ReadLn(N);
  
  for I := 1 to N do
  begin
    Write('ชื่อ ', I, ': ');
    ReadLn(Names[I]);
    Write('คะแนน ', I, ': ');
    ReadLn(Scores[I]);
  end;
  
  WriteLn;
  WriteLn('รายชื่อและคะแนน:');
  for I := 1 to N do
    WriteLn(I:3, '. ', Names[I], ': ', Scores[I]);
end.
```

---

## Static Arrays

Static array มีขนาดคงที่ ถูกกำหนดตอน compile time

### ตัวอย่างที่ 3: Static array พื้นฐาน

```pascal
program StaticArrayBasic;
const
  MAX = 10;
var
  Data: array[1..MAX] of Integer;
  I: Integer;
  Sum: Integer;
  Avg: Real;
begin
  WriteLn('ป้อนข้อมูล ', MAX, ' ค่า:');
  for I := 1 to MAX do
  begin
    Write('Data[', I, ']: ');
    ReadLn(Data[I]);
  end;
  
  // หาผลรวมและค่าเฉลี่ย
  Sum := 0;
  for I := 1 to MAX do
    Sum := Sum + Data[I];
  Avg := Sum / MAX;
  
  WriteLn;
  WriteLn('ผลรวม: ', Sum);
  WriteLn('เฉลี่ย: ', Avg:0:2);
  
  // หาค่ามากสุดและน้อยสุด
  var MinVal := Data[1];
  var MaxVal := Data[1];
  for I := 2 to MAX do
  begin
    if Data[I] < MinVal then MinVal := Data[I];
    if Data[I] > MaxVal then MaxVal := Data[I];
  end;
  WriteLn('ค่าน้อยสุด: ', MinVal);
  WriteLn('ค่ามากสุด: ', MaxVal);
end.
```

### ตัวอย่างที่ 4: Array ของ Record

```pascal
program ArrayOfRecord;
type
  TStudent = record
    Name: String[50];
    Score: Integer;
    Grade: Char;
  end;
  
var
  Students: array[1..5] of TStudent;
  I: Integer;
begin
  for I := 1 to 5 do
  begin
    Write('ชื่อนักเรียน ', I, ': ');
    ReadLn(Students[I].Name);
    Write('คะแนน: ');
    ReadLn(Students[I].Score);
    
    // กำหนดเกรด
    case Students[I].Score of
      80..100: Students[I].Grade := 'A';
      70..79:  Students[I].Grade := 'B';
      60..69:  Students[I].Grade := 'C';
      50..59:  Students[I].Grade := 'D';
    else
      Students[I].Grade := 'F';
    end;
  end;
  
  WriteLn;
  WriteLn('=== ผลการเรียน ===');
  for I := 1 to 5 do
    WriteLn(Students[I].Name, ': ', 
            Students[I].Score, ' (', Students[I].Grade, ')');
end.
```

### ตัวอย่างที่ 5: การ initialize array ทั้งหมด

```pascal
program InitArray;
var
  Numbers: array[1..100] of Integer;
  I: Integer;
begin
  // ตั้งค่าทุก element เป็น 0
  for I := 1 to 100 do
    Numbers[I] := 0;
  
  // หรือใช้ FillChar สำหรับ byte arrays
  var ByteArr: array[0..99] of Byte;
  FillChar(ByteArr, SizeOf(ByteArr), 0);
  
  WriteLn('Array initialized!');
  
  // กำหนดค่าเฉพาะบาง index
  Numbers[1] := 100;
  Numbers[50] := 500;
  Numbers[100] := 1000;
  
  WriteLn('Numbers[1] = ', Numbers[1]);
  WriteLn('Numbers[50] = ', Numbers[50]);
  WriteLn('Numbers[100] = ', Numbers[100]);
end.
```

---

## Dynamic Arrays

Dynamic array ปรับขนาดได้ในขณะโปรแกรมทำงาน

### ตัวอย่างที่ 6: Dynamic array พื้นฐาน

```pascal
program DynamicArrayBasic;
var
  Arr: array of Integer; // ประกาศ dynamic array
  N, I: Integer;
begin
  Write('ป้อนขนาด array: ');
  ReadLn(N);
  
  SetLength(Arr, N); // กำหนดขนาด
  
  WriteLn('ขนาดปัจจุบัน: ', Length(Arr));
  WriteLn('Index: 0 ถึง ', High(Arr));
  
  // ป้อนข้อมูล (index เริ่มที่ 0)
  for I := 0 to N - 1 do
  begin
    Write('Arr[', I, ']: ');
    ReadLn(Arr[I]);
  end;
  
  // แสดงข้อมูล
  for I := 0 to High(Arr) do
    WriteLn('Arr[', I, '] = ', Arr[I]);
end.
```

### ตัวอย่างที่ 7: เพิ่มขนาด dynamic array

```pascal
program GrowArray;
var
  Arr: array of Integer;
  N, I, Val: Integer;
  Continue: Boolean;
begin
  SetLength(Arr, 0); // เริ่มต้นว่าง
  
  WriteLn('ป้อนตัวเลข (0 เพื่อหยุด):');
  Continue := True;
  
  while Continue do
  begin
    Write('ตัวเลข: ');
    ReadLn(Val);
    
    if Val = 0 then
      Continue := False
    else
    begin
      // เพิ่มขนาด array ทีละ 1
      N := Length(Arr);
      SetLength(Arr, N + 1);
      Arr[N] := Val;
    end;
  end;
  
  WriteLn('ป้อนข้อมูล ', Length(Arr), ' ค่า:');
  for I := 0 to High(Arr) do
    Write(Arr[I], ' ');
  WriteLn;
end.
```

### ตัวอย่างที่ 8: Copy และ Slice dynamic array

```pascal
program ArrayCopySlice;
var
  Original: array of Integer;
  CopyArr: array of Integer;
  I, N: Integer;
begin
  N := 10;
  SetLength(Original, N);
  
  // กำหนดค่า
  for I := 0 to N - 1 do
    Original[I] := (I + 1) * 10;
  
  // Copy ทั้ง array
  CopyArr := Copy(Original); // ใน Lazarus
  // หรือ: SetLength(CopyArr, N); Move(Original[0], CopyArr[0], N * SizeOf(Integer));
  
  // แสดงผล
  Write('Original: ');
  for I := 0 to High(Original) do
    Write(Original[I], ' ');
  WriteLn;
  
  Write('Copy:     ');
  for I := 0 to High(CopyArr) do
    Write(CopyArr[I], ' ');
  WriteLn;
  
  // เปลี่ยนค่าใน CopyArr ไม่กระทบ Original
  CopyArr[0] := 999;
  WriteLn('หลังแก้ไข CopyArr[0]:');
  WriteLn('Original[0] = ', Original[0]);
  WriteLn('CopyArr[0] = ', CopyArr[0]);
end.
```

### ตัวอย่างที่ 9: Resize และลด dynamic array

```pascal
program ResizeArray;
var
  Arr: array of String;
  I: Integer;
begin
  // สร้าง array ขนาด 5
  SetLength(Arr, 5);
  Arr[0] := 'ข้าว';
  Arr[1] := 'ก๋วยเตี๋ยว';
  Arr[2] := 'ต้มยำ';
  Arr[3] := 'ผัดไทย';
  Arr[4] := 'ส้มตำ';
  
  WriteLn('เริ่มต้น (', Length(Arr), ' รายการ):');
  for I := 0 to High(Arr) do
    WriteLn(I, ': ', Arr[I]);
  
  // เพิ่มขนาด
  SetLength(Arr, 7);
  Arr[5] := 'ข้าวมันไก่';
  Arr[6] := 'ข้าวผัด';
  
  WriteLn('หลังเพิ่ม (', Length(Arr), ' รายการ):');
  for I := 0 to High(Arr) do
    WriteLn(I, ': ', Arr[I]);
  
  // ลดขนาด (ข้อมูลส่วนที่เกินจะหายไป)
  SetLength(Arr, 3);
  WriteLn('หลังลด (', Length(Arr), ' รายการ):');
  for I := 0 to High(Arr) do
    WriteLn(I, ': ', Arr[I]);
end.
```

---

## Multidimensional Arrays

### ตัวอย่างที่ 10: 2D Array (Matrix)

```pascal
program TwoDArray;
const
  ROWS = 3;
  COLS = 4;
var
  Matrix: array[1..ROWS, 1..COLS] of Integer;
  I, J: Integer;
begin
  // กำหนดค่า
  for I := 1 to ROWS do
    for J := 1 to COLS do
      Matrix[I, J] := I * 10 + J;
  
  // แสดงผล
  WriteLn('Matrix (', ROWS, 'x', COLS, '):');
  for I := 1 to ROWS do
  begin
    for J := 1 to COLS do
      Write(Matrix[I, J]:5);
    WriteLn;
  end;
end.
```

### ตัวอย่างที่ 11: Matrix Addition

```pascal
program MatrixAdd;
const
  N = 3;
var
  A, B, C: array[1..N, 1..N] of Integer;
  I, J: Integer;
begin
  WriteLn('Matrix A:');
  for I := 1 to N do
    for J := 1 to N do
    begin
      Write('A[', I, ',', J, ']: ');
      ReadLn(A[I, J]);
    end;
  
  WriteLn('Matrix B:');
  for I := 1 to N do
    for J := 1 to N do
    begin
      Write('B[', I, ',', J, ']: ');
      ReadLn(B[I, J]);
    end;
  
  // บวก
  for I := 1 to N do
    for J := 1 to N do
      C[I, J] := A[I, J] + B[I, J];
  
  WriteLn('A + B:');
  for I := 1 to N do
  begin
    for J := 1 to N do
      Write(C[I, J]:5);
    WriteLn;
  end;
end.
```

### ตัวอย่างที่ 12: Matrix Multiplication

```pascal
program MatrixMultiply;
const
  N = 3;
var
  A: array[1..N, 1..N] of Integer;
  B: array[1..N, 1..N] of Integer;
  C: array[1..N, 1..N] of Integer;
  I, J, K: Integer;
begin
  // สร้าง matrix ตัวอย่าง
  for I := 1 to N do
    for J := 1 to N do
    begin
      A[I, J] := I + J;
      B[I, J] := I * J;
    end;
  
  // คูณ matrix: C = A × B
  for I := 1 to N do
    for J := 1 to N do
    begin
      C[I, J] := 0;
      for K := 1 to N do
        C[I, J] := C[I, J] + A[I, K] * B[K, J];
    end;
  
  // แสดง A
  WriteLn('Matrix A:');
  for I := 1 to N do begin
    for J := 1 to N do Write(A[I,J]:5);
    WriteLn;
  end;
  
  // แสดง B
  WriteLn('Matrix B:');
  for I := 1 to N do begin
    for J := 1 to N do Write(B[I,J]:5);
    WriteLn;
  end;
  
  // แสดง C
  WriteLn('A × B:');
  for I := 1 to N do begin
    for J := 1 to N do Write(C[I,J]:5);
    WriteLn;
  end;
end.
```

### ตัวอย่างที่ 13: 3D Array

```pascal
program ThreeDArray;
var
  Cube: array[1..3, 1..3, 1..3] of Integer;
  X, Y, Z: Integer;
begin
  // กำหนดค่า
  for X := 1 to 3 do
    for Y := 1 to 3 do
      for Z := 1 to 3 do
        Cube[X, Y, Z] := X * 100 + Y * 10 + Z;
  
  // แสดงแต่ละ layer
  for X := 1 to 3 do
  begin
    WriteLn('Layer X = ', X, ':');
    for Y := 1 to 3 do
    begin
      for Z := 1 to 3 do
        Write(Cube[X, Y, Z]:5);
      WriteLn;
    end;
    WriteLn;
  end;
end.
```

---

## Array of Arrays (Jagged Arrays)

Array ที่แต่ละแถวมีขนาดไม่เท่ากัน

### ตัวอย่างที่ 14: Jagged Array

```pascal
program JaggedArray;
var
  Jagged: array of array of Integer;
  I, J: Integer;
  Sizes: array[0..4] of Integer;
begin
  // กำหนดจำนวนแถว
  SetLength(Jagged, 5);
  Sizes[0] := 3;
  Sizes[1] := 1;
  Sizes[2] := 5;
  Sizes[3] := 2;
  Sizes[4] := 4;
  
  // กำหนดขนาดแต่ละแถว
  for I := 0 to 4 do
  begin
    SetLength(Jagged[I], Sizes[I]);
    // กำหนดค่า
    for J := 0 to Sizes[I] - 1 do
      Jagged[I][J] := (I + 1) * 10 + (J + 1);
  end;
  
  // แสดงผล
  WriteLn('Jagged Array:');
  for I := 0 to 4 do
  begin
    Write('Row ', I, ' (', Length(Jagged[I]), ' elements): ');
    for J := 0 to High(Jagged[I]) do
      Write(Jagged[I][J], ' ');
    WriteLn;
  end;
end.
```

---

## Array กับ Loops

### ตัวอย่างที่ 15: Operations ทั่วไปกับ array

```pascal
program ArrayOperations;
var
  Data: array[1..10] of Real;
  I: Integer;
  Sum, Max, Min, Avg: Real;
  MaxIdx, MinIdx: Integer;
begin
  // ป้อนข้อมูล
  WriteLn('ป้อนข้อมูล 10 ค่า:');
  for I := 1 to 10 do
  begin
    Write('ข้อมูล ', I, ': ');
    ReadLn(Data[I]);
  end;
  
  // คำนวณสถิติ
  Sum := 0;
  Max := Data[1];
  Min := Data[1];
  MaxIdx := 1;
  MinIdx := 1;
  
  for I := 1 to 10 do
  begin
    Sum := Sum + Data[I];
    if Data[I] > Max then begin Max := Data[I]; MaxIdx := I; end;
    if Data[I] < Min then begin Min := Data[I]; MinIdx := I; end;
  end;
  
  Avg := Sum / 10;
  
  WriteLn;
  WriteLn('=== สถิติ ===');
  WriteLn('ผลรวม:   ', Sum:0:2);
  WriteLn('ค่าเฉลี่ย: ', Avg:0:2);
  WriteLn('ค่ามากสุด: ', Max:0:2, ' (ตำแหน่ง ', MaxIdx, ')');
  WriteLn('ค่าน้อยสุด: ', Min:0:2, ' (ตำแหน่ง ', MinIdx, ')');
  
  // หาค่ามากกว่าค่าเฉลี่ย
  Write('ค่าที่มากกว่าค่าเฉลี่ย: ');
  for I := 1 to 10 do
    if Data[I] > Avg then
      Write(Data[I]:0:2, ' ');
  WriteLn;
end.
```

---

## Array Sorting

### ตัวอย่างที่ 16: Bubble Sort

```pascal
program BubbleSort;
var
  Arr: array[1..10] of Integer;
  I, J, Temp: Integer;
  N: Integer;
  Swapped: Boolean;
begin
  N := 10;
  
  WriteLn('ป้อนข้อมูล ', N, ' ค่า:');
  for I := 1 to N do
  begin
    Write('Arr[', I, ']: ');
    ReadLn(Arr[I]);
  end;
  
  // Bubble Sort (เรียงจากน้อยไปมาก)
  for I := 1 to N - 1 do
  begin
    Swapped := False;
    for J := 1 to N - I do
    begin
      if Arr[J] > Arr[J + 1] then
      begin
        Temp := Arr[J];
        Arr[J] := Arr[J + 1];
        Arr[J + 1] := Temp;
        Swapped := True;
      end;
    end;
    if not Swapped then Break; // Optimized: หยุดถ้าไม่มีการสลับ
  end;
  
  Write('หลังเรียง: ');
  for I := 1 to N do
    Write(Arr[I], ' ');
  WriteLn;
end.
```

### ตัวอย่างที่ 17: Selection Sort

```pascal
program SelectionSort;
var
  Arr: array of Integer;
  N, I, J, MinIdx, Temp: Integer;
begin
  Write('จำนวนข้อมูล: ');
  ReadLn(N);
  SetLength(Arr, N);
  
  for I := 0 to N - 1 do
  begin
    Write('Arr[', I, ']: ');
    ReadLn(Arr[I]);
  end;
  
  // Selection Sort
  for I := 0 to N - 2 do
  begin
    MinIdx := I;
    for J := I + 1 to N - 1 do
      if Arr[J] < Arr[MinIdx] then
        MinIdx := J;
    
    if MinIdx <> I then
    begin
      Temp := Arr[I];
      Arr[I] := Arr[MinIdx];
      Arr[MinIdx] := Temp;
    end;
  end;
  
  Write('หลังเรียง: ');
  for I := 0 to N - 1 do
    Write(Arr[I], ' ');
  WriteLn;
end.
```

### ตัวอย่างที่ 18: Insertion Sort

```pascal
program InsertionSort;
var
  Arr: array[1..10] of Integer;
  I, J, Key: Integer;
  N: Integer;
begin
  N := 8;
  Arr[1] := 64; Arr[2] := 25; Arr[3] := 12;
  Arr[4] := 22; Arr[5] := 11; Arr[6] := 45;
  Arr[7] := 38; Arr[8] := 9;
  
  Write('ก่อนเรียง: ');
  for I := 1 to N do Write(Arr[I], ' ');
  WriteLn;
  
  // Insertion Sort
  for I := 2 to N do
  begin
    Key := Arr[I];
    J := I - 1;
    
    while (J >= 1) and (Arr[J] > Key) do
    begin
      Arr[J + 1] := Arr[J];
      Dec(J);
    end;
    
    Arr[J + 1] := Key;
  end;
  
  Write('หลังเรียง: ');
  for I := 1 to N do Write(Arr[I], ' ');
  WriteLn;
end.
```

### ตัวอย่างที่ 19: Quick Sort (Recursive)

```pascal
program QuickSort;
var
  Data: array[1..20] of Integer;
  N: Integer;

procedure Swap(var A, B: Integer);
var Temp: Integer;
begin
  Temp := A;
  A := B;
  B := Temp;
end;

function Partition(Lo, Hi: Integer): Integer;
var
  Pivot, I, J: Integer;
begin
  Pivot := Data[Hi];
  I := Lo - 1;
  
  for J := Lo to Hi - 1 do
  begin
    if Data[J] <= Pivot then
    begin
      Inc(I);
      Swap(Data[I], Data[J]);
    end;
  end;
  
  Swap(Data[I + 1], Data[Hi]);
  Result := I + 1;
end;

procedure QSort(Lo, Hi: Integer);
var PI: Integer;
begin
  if Lo < Hi then
  begin
    PI := Partition(Lo, Hi);
    QSort(Lo, PI - 1);
    QSort(PI + 1, Hi);
  end;
end;

var I: Integer;
begin
  N := 10;
  WriteLn('ป้อน ', N, ' ตัวเลข:');
  for I := 1 to N do
  begin
    Write('Data[', I, ']: ');
    ReadLn(Data[I]);
  end;
  
  Write('ก่อนเรียง: ');
  for I := 1 to N do Write(Data[I], ' ');
  WriteLn;
  
  QSort(1, N);
  
  Write('หลัง Quick Sort: ');
  for I := 1 to N do Write(Data[I], ' ');
  WriteLn;
end.
```

### ตัวอย่างที่ 20: Merge Sort

```pascal
program MergeSort;
type
  TIntArray = array of Integer;

procedure Merge(var Arr: TIntArray; Left, Mid, Right: Integer);
var
  N1, N2: Integer;
  L, R: TIntArray;
  I, J, K: Integer;
begin
  N1 := Mid - Left + 1;
  N2 := Right - Mid;
  
  SetLength(L, N1);
  SetLength(R, N2);
  
  for I := 0 to N1 - 1 do L[I] := Arr[Left + I];
  for J := 0 to N2 - 1 do R[J] := Arr[Mid + 1 + J];
  
  I := 0; J := 0; K := Left;
  
  while (I < N1) and (J < N2) do
  begin
    if L[I] <= R[J] then
    begin
      Arr[K] := L[I];
      Inc(I);
    end
    else
    begin
      Arr[K] := R[J];
      Inc(J);
    end;
    Inc(K);
  end;
  
  while I < N1 do begin Arr[K] := L[I]; Inc(I); Inc(K); end;
  while J < N2 do begin Arr[K] := R[J]; Inc(J); Inc(K); end;
end;

procedure DoMergeSort(var Arr: TIntArray; Left, Right: Integer);
var Mid: Integer;
begin
  if Left < Right then
  begin
    Mid := (Left + Right) div 2;
    DoMergeSort(Arr, Left, Mid);
    DoMergeSort(Arr, Mid + 1, Right);
    Merge(Arr, Left, Mid, Right);
  end;
end;

var
  Data: TIntArray;
  I: Integer;
begin
  SetLength(Data, 8);
  Data[0] := 38; Data[1] := 27; Data[2] := 43; Data[3] := 3;
  Data[4] := 9;  Data[5] := 82; Data[6] := 10; Data[7] := 1;
  
  Write('ก่อนเรียง: ');
  for I := 0 to 7 do Write(Data[I], ' ');
  WriteLn;
  
  DoMergeSort(Data, 0, 7);
  
  Write('หลัง Merge Sort: ');
  for I := 0 to 7 do Write(Data[I], ' ');
  WriteLn;
end.
```

---

## Array Searching

### ตัวอย่างที่ 21: Linear Search

```pascal
program LinearSearch;
var
  Arr: array[1..10] of Integer;
  Target, I: Integer;
  Found: Boolean;
  Position: Integer;
begin
  // สร้างข้อมูล
  for I := 1 to 10 do
    Arr[I] := I * 7;
  
  Write('ข้อมูล: ');
  for I := 1 to 10 do Write(Arr[I], ' ');
  WriteLn;
  
  Write('หาตัวเลข: ');
  ReadLn(Target);
  
  Found := False;
  Position := -1;
  
  for I := 1 to 10 do
  begin
    if Arr[I] = Target then
    begin
      Found := True;
      Position := I;
      Break;
    end;
  end;
  
  if Found then
    WriteLn('พบ ', Target, ' ที่ตำแหน่ง ', Position)
  else
    WriteLn('ไม่พบ ', Target);
end.
```

### ตัวอย่างที่ 22: Binary Search (ต้องเรียงก่อน)

```pascal
program BinarySearch;
var
  Arr: array[1..10] of Integer;
  Target, Lo, Hi, Mid: Integer;
  Found: Boolean;
  
begin
  // ข้อมูลที่เรียงแล้ว
  Arr[1] := 2;  Arr[2] := 5;  Arr[3] := 8;
  Arr[4] := 12; Arr[5] := 16; Arr[6] := 23;
  Arr[7] := 38; Arr[8] := 56; Arr[9] := 72;
  Arr[10] := 91;
  
  Write('ข้อมูล: ');
  var I: Integer;
  for I := 1 to 10 do Write(Arr[I], ' ');
  WriteLn;
  
  Write('หาตัวเลข: ');
  ReadLn(Target);
  
  Lo := 1;
  Hi := 10;
  Found := False;
  
  while (Lo <= Hi) and (not Found) do
  begin
    Mid := (Lo + Hi) div 2;
    WriteLn('ตรวจสอบ Arr[', Mid, '] = ', Arr[Mid]);
    
    if Arr[Mid] = Target then
      Found := True
    else if Arr[Mid] < Target then
      Lo := Mid + 1
    else
      Hi := Mid - 1;
  end;
  
  if Found then
    WriteLn('พบ ', Target, ' ที่ตำแหน่ง ', Mid)
  else
    WriteLn('ไม่พบ ', Target);
end.
```

---

## Open Array Parameters

### ตัวอย่างที่ 23: ส่ง array ขนาดใดก็ได้

```pascal
program OpenArrayParam;

function SumArray(const Arr: array of Integer): Integer;
var
  I: Integer;
begin
  Result := 0;
  for I := Low(Arr) to High(Arr) do
    Result := Result + Arr[I];
end;

function MaxArray(const Arr: array of Integer): Integer;
var
  I: Integer;
begin
  Result := Arr[0];
  for I := 1 to High(Arr) do
    if Arr[I] > Result then
      Result := Arr[I];
end;

procedure PrintArray(const Arr: array of Integer);
var
  I: Integer;
begin
  Write('[');
  for I := Low(Arr) to High(Arr) do
  begin
    Write(Arr[I]);
    if I < High(Arr) then Write(', ');
  end;
  WriteLn(']');
end;

var
  Small: array[1..3] of Integer;
  Medium: array[1..6] of Integer;
  Large: array[1..10] of Integer;
  I: Integer;
  
begin
  // กำหนดค่า
  for I := 1 to 3 do Small[I] := I;
  for I := 1 to 6 do Medium[I] := I * 2;
  for I := 1 to 10 do Large[I] := I * I;
  
  // ใช้ฟังก์ชันเดียวกับ array ขนาดต่างกัน
  Write('Small: '); PrintArray(Small);
  WriteLn('Sum = ', SumArray(Small), ', Max = ', MaxArray(Small));
  
  Write('Medium: '); PrintArray(Medium);
  WriteLn('Sum = ', SumArray(Medium), ', Max = ', MaxArray(Medium));
  
  Write('Large: '); PrintArray(Large);
  WriteLn('Sum = ', SumArray(Large), ', Max = ', MaxArray(Large));
  
  // ส่ง literal array
  WriteLn('Literal: Sum = ', SumArray([5, 10, 15, 20]));
end.
```

---

## Array Slicing

### ตัวอย่างที่ 24: การตัด/คัดลอก array

```pascal
program ArraySlicing;
var
  Original: array of Integer;
  Slice: array of Integer;
  I, StartIdx, EndIdx, SliceLen: Integer;
begin
  // สร้าง array ต้นฉบับ
  SetLength(Original, 10);
  for I := 0 to 9 do
    Original[I] := (I + 1) * 5;
  
  Write('Original: ');
  for I := 0 to 9 do Write(Original[I], ' ');
  WriteLn;
  
  // ตัด index 2 ถึง 6
  StartIdx := 2;
  EndIdx := 6;
  SliceLen := EndIdx - StartIdx + 1;
  
  SetLength(Slice, SliceLen);
  for I := 0 to SliceLen - 1 do
    Slice[I] := Original[StartIdx + I];
  
  Write('Slice [', StartIdx, '..', EndIdx, ']: ');
  for I := 0 to High(Slice) do Write(Slice[I], ' ');
  WriteLn;
  
  // ใช้ Copy function
  var SliceCopy := Copy(Original, StartIdx, SliceLen);
  Write('Copy: ');
  for I := 0 to High(SliceCopy) do Write(SliceCopy[I], ' ');
  WriteLn;
end.
```

---

## โปรแกรมตัวอย่าง: การจัดการรายชื่อนักเรียน

```pascal
program StudentManagement;
const
  MAX_STUDENTS = 30;
  
type
  TStudent = record
    ID: Integer;
    Name: String[50];
    Scores: array[1..5] of Real;  // คะแนน 5 วิชา
    Total: Real;
    Average: Real;
    Grade: String[2];
    Rank: Integer;
  end;

var
  Students: array[1..MAX_STUDENTS] of TStudent;
  Count: Integer;
  Choice: Integer;
  I, J: Integer;
  
procedure CalculateGrade(var S: TStudent);
begin
  S.Total := 0;
  for J := 1 to 5 do
    S.Total := S.Total + S.Scores[J];
  S.Average := S.Total / 5;
  
  if S.Average >= 80 then S.Grade := 'A'
  else if S.Average >= 75 then S.Grade := 'B+'
  else if S.Average >= 70 then S.Grade := 'B'
  else if S.Average >= 65 then S.Grade := 'C+'
  else if S.Average >= 60 then S.Grade := 'C'
  else if S.Average >= 55 then S.Grade := 'D+'
  else if S.Average >= 50 then S.Grade := 'D'
  else S.Grade := 'F';
end;

procedure AddStudent;
begin
  if Count >= MAX_STUDENTS then
  begin
    WriteLn('จำนวนนักเรียนเต็มแล้ว!');
    Exit;
  end;
  
  Inc(Count);
  Students[Count].ID := Count;
  
  Write('ชื่อนักเรียน: ');
  ReadLn(Students[Count].Name);
  
  WriteLn('ป้อนคะแนน 5 วิชา:');
  for J := 1 to 5 do
  begin
    Write('วิชา ', J, ': ');
    ReadLn(Students[Count].Scores[J]);
  end;
  
  CalculateGrade(Students[Count]);
  WriteLn('เพิ่มนักเรียน ', Students[Count].Name, ' สำเร็จ');
end;

procedure ShowAllStudents;
begin
  if Count = 0 then
  begin
    WriteLn('ยังไม่มีข้อมูลนักเรียน');
    Exit;
  end;
  
  WriteLn;
  WriteLn('ID':4, 'ชื่อ':15, 'เฉลี่ย':8, 'เกรด':6);
  WriteLn(StringOfChar('-', 35));
  
  for I := 1 to Count do
    WriteLn(Students[I].ID:4, Students[I].Name:15, 
            Students[I].Average:8:2, Students[I].Grade:6);
end;

procedure ShowTopStudents(TopN: Integer);
var
  TempStudents: array[1..MAX_STUDENTS] of TStudent;
  Temp: TStudent;
begin
  // Copy
  for I := 1 to Count do TempStudents[I] := Students[I];
  
  // Sort โดยใช้ Bubble Sort
  for I := 1 to Count - 1 do
    for J := 1 to Count - I do
      if TempStudents[J].Average < TempStudents[J+1].Average then
      begin
        Temp := TempStudents[J];
        TempStudents[J] := TempStudents[J+1];
        TempStudents[J+1] := Temp;
      end;
  
  // กำหนด Rank
  for I := 1 to Count do
    TempStudents[I].Rank := I;
  
  if TopN > Count then TopN := Count;
  
  WriteLn;
  WriteLn('=== Top ', TopN, ' นักเรียน ===');
  WriteLn('อันดับ':6, 'ชื่อ':15, 'เฉลี่ย':8, 'เกรด':6);
  WriteLn(StringOfChar('-', 38));
  
  for I := 1 to TopN do
    WriteLn(I:6, TempStudents[I].Name:15, 
            TempStudents[I].Average:8:2, TempStudents[I].Grade:6);
end;

procedure ShowStatistics;
var
  TotalAvg: Real;
  PassCount, FailCount: Integer;
begin
  if Count = 0 then begin WriteLn('ไม่มีข้อมูล'); Exit; end;
  
  TotalAvg := 0;
  PassCount := 0;
  FailCount := 0;
  
  for I := 1 to Count do
  begin
    TotalAvg := TotalAvg + Students[I].Average;
    if Students[I].Average >= 50 then Inc(PassCount)
    else Inc(FailCount);
  end;
  
  WriteLn;
  WriteLn('=== สถิติห้องเรียน ===');
  WriteLn('จำนวนนักเรียน: ', Count);
  WriteLn('ค่าเฉลี่ยห้อง: ', TotalAvg / Count:0:2);
  WriteLn('ผ่าน: ', PassCount, ' คน');
  WriteLn('ไม่ผ่าน: ', FailCount, ' คน');
  WriteLn('อัตราผ่าน: ', (PassCount / Count * 100):0:1, '%');
end;

begin
  Count := 0;
  
  WriteLn('=== ระบบจัดการนักเรียน ===');
  
  repeat
    WriteLn;
    WriteLn('1. เพิ่มนักเรียน');
    WriteLn('2. แสดงนักเรียนทั้งหมด');
    WriteLn('3. แสดง Top 3');
    WriteLn('4. สถิติห้องเรียน');
    WriteLn('0. ออก');
    Write('เลือก: ');
    ReadLn(Choice);
    
    case Choice of
      1: AddStudent;
      2: ShowAllStudents;
      3: ShowTopStudents(3);
      4: ShowStatistics;
      0: WriteLn('ออกจากระบบ');
    else WriteLn('ตัวเลือกไม่ถูกต้อง');
    end;
  until Choice = 0;
end.
```

---

## โปรแกรมตัวอย่าง: Matrix Operations

```pascal
program MatrixOperations;
const
  MAX_SIZE = 5;

type
  TMatrix = array[1..MAX_SIZE, 1..MAX_SIZE] of Real;

var
  M1, M2, Result: TMatrix;
  R1, C1, R2, C2: Integer;
  Choice: Integer;

procedure ReadMatrix(var M: TMatrix; Rows, Cols: Integer; Name: String);
var I, J: Integer;
begin
  WriteLn('ป้อน Matrix ', Name, ' (', Rows, 'x', Cols, '):');
  for I := 1 to Rows do
    for J := 1 to Cols do
    begin
      Write('M[', I, ',', J, ']: ');
      ReadLn(M[I, J]);
    end;
end;

procedure PrintMatrix(const M: TMatrix; Rows, Cols: Integer; Name: String);
var I, J: Integer;
begin
  WriteLn('Matrix ', Name, ':');
  for I := 1 to Rows do
  begin
    for J := 1 to Cols do
      Write(M[I, J]:8:2);
    WriteLn;
  end;
end;

procedure AddMatrices;
var I, J: Integer;
begin
  Write('ขนาด (NxM): ');
  ReadLn(R1, C1);
  R2 := R1; C2 := C1;
  ReadMatrix(M1, R1, C1, 'A');
  ReadMatrix(M2, R2, C2, 'B');
  
  for I := 1 to R1 do
    for J := 1 to C1 do
      Result[I, J] := M1[I, J] + M2[I, J];
  
  PrintMatrix(Result, R1, C1, 'A+B');
end;

procedure MultiplyMatrices;
var I, J, K: Integer;
begin
  Write('Matrix A (RxC): ');
  ReadLn(R1, C1);
  Write('Matrix B (RxC): ');
  ReadLn(R2, C2);
  
  if C1 <> R2 then
  begin
    WriteLn('ขนาด matrix ไม่ตรงกัน!');
    Exit;
  end;
  
  ReadMatrix(M1, R1, C1, 'A');
  ReadMatrix(M2, R2, C2, 'B');
  
  for I := 1 to R1 do
    for J := 1 to C2 do
    begin
      Result[I, J] := 0;
      for K := 1 to C1 do
        Result[I, J] := Result[I, J] + M1[I, K] * M2[K, J];
    end;
  
  PrintMatrix(Result, R1, C2, 'A×B');
end;

procedure TransposeMatrix;
var I, J: Integer;
begin
  Write('ขนาด (NxM): ');
  ReadLn(R1, C1);
  ReadMatrix(M1, R1, C1, 'A');
  
  // Transpose
  for I := 1 to R1 do
    for J := 1 to C1 do
      Result[J, I] := M1[I, J];
  
  PrintMatrix(M1, R1, C1, 'A');
  PrintMatrix(Result, C1, R1, 'Transpose(A)');
end;

begin
  repeat
    WriteLn;
    WriteLn('=== Matrix Operations ===');
    WriteLn('1. บวก Matrix');
    WriteLn('2. คูณ Matrix');
    WriteLn('3. Transpose');
    WriteLn('0. ออก');
    Write('เลือก: ');
    ReadLn(Choice);
    
    case Choice of
      1: AddMatrices;
      2: MultiplyMatrices;
      3: TransposeMatrix;
    end;
  until Choice = 0;
end.
```

---

## โปรแกรมตัวอย่าง: Statistics Calculator

```pascal
program StatisticsCalculator;
var
  Data: array of Real;
  N: Integer;
  I: Integer;

function Mean(const Arr: array of Real): Real;
var Sum: Real;
begin
  Sum := 0;
  for I := Low(Arr) to High(Arr) do
    Sum := Sum + Arr[I];
  Result := Sum / Length(Arr);
end;

function Median(Arr: array of Real): Real;
var I, J: Integer; Temp: Real;
begin
  // Sort ก่อน
  for I := 0 to High(Arr) - 1 do
    for J := 0 to High(Arr) - I - 1 do
      if Arr[J] > Arr[J+1] then
      begin
        Temp := Arr[J]; Arr[J] := Arr[J+1]; Arr[J+1] := Temp;
      end;
  
  N := Length(Arr);
  if N mod 2 = 1 then
    Result := Arr[N div 2]
  else
    Result := (Arr[N div 2 - 1] + Arr[N div 2]) / 2;
end;

function Variance(const Arr: array of Real): Real;
var M, Sum: Real;
begin
  M := Mean(Arr);
  Sum := 0;
  for I := Low(Arr) to High(Arr) do
    Sum := Sum + (Arr[I] - M) * (Arr[I] - M);
  Result := Sum / Length(Arr);
end;

function StdDev(const Arr: array of Real): Real;
begin
  Result := Sqrt(Variance(Arr));
end;

begin
  Write('จำนวนข้อมูล: ');
  ReadLn(N);
  SetLength(Data, N);
  
  for I := 0 to N - 1 do
  begin
    Write('ข้อมูล ', I+1, ': ');
    ReadLn(Data[I]);
  end;
  
  // หาค่า Min, Max
  var MinVal := Data[0];
  var MaxVal := Data[0];
  for I := 1 to N - 1 do
  begin
    if Data[I] < MinVal then MinVal := Data[I];
    if Data[I] > MaxVal then MaxVal := Data[I];
  end;
  
  WriteLn;
  WriteLn('=== สถิติ ===');
  WriteLn('จำนวนข้อมูล:   ', N);
  WriteLn('ค่าต่ำสุด:     ', MinVal:0:4);
  WriteLn('ค่าสูงสุด:     ', MaxVal:0:4);
  WriteLn('ค่าเฉลี่ย:     ', Mean(Data):0:4);
  WriteLn('มัธยฐาน:       ', Median(Data):0:4);
  WriteLn('ความแปรปรวน:   ', Variance(Data):0:4);
  WriteLn('ส่วนเบี่ยงเบน: ', StdDev(Data):0:4);
  WriteLn('พิสัย:         ', MaxVal - MinVal:0:4);
end.
```

---

## แบบฝึกหัด

### ข้อที่ 1: Rotate Array

```pascal
// หมุน array ไปทางขวา k ตำแหน่ง
program RotateRight;
var
  Arr: array of Integer;
  N, K, I: Integer;
  Temp: Integer;
begin
  SetLength(Arr, 8);
  for I := 0 to 7 do Arr[I] := I + 1;
  
  Write('หมุนกี่ตำแหน่ง: ');
  ReadLn(K);
  K := K mod Length(Arr);
  
  Write('ก่อน: ');
  for I := 0 to High(Arr) do Write(Arr[I], ' ');
  WriteLn;
  
  // หมุนทีละตำแหน่ง K ครั้ง
  for N := 1 to K do
  begin
    Temp := Arr[High(Arr)];
    for I := High(Arr) downto 1 do
      Arr[I] := Arr[I-1];
    Arr[0] := Temp;
  end;
  
  Write('หลังหมุน ', K, ' ตำแหน่ง: ');
  for I := 0 to High(Arr) do Write(Arr[I], ' ');
  WriteLn;
end.
```

### ข้อที่ 2: Two Sum

```pascal
// หาคู่ตัวเลขที่รวมกันได้ค่าเป้าหมาย
program TwoSum;
var
  Arr: array[1..10] of Integer;
  Target, I, J: Integer;
begin
  Arr[1]:=2; Arr[2]:=7; Arr[3]:=11; Arr[4]:=15; Arr[5]:=1;
  Arr[6]:=8; Arr[7]:=3; Arr[8]:=6;  Arr[9]:=9;  Arr[10]:=5;
  
  Write('เป้าหมาย: ');
  ReadLn(Target);
  
  for I := 1 to 9 do
    for J := I + 1 to 10 do
      if Arr[I] + Arr[J] = Target then
        WriteLn('พบ: ', Arr[I], ' + ', Arr[J], ' = ', Target);
end.
```

### ข้อที่ 3: Duplicate Removal

```pascal
program RemoveDuplicates;
var
  Original: array[1..10] of Integer;
  Unique: array[1..10] of Integer;
  I, J, UniqueCount: Integer;
  IsDuplicate: Boolean;
begin
  Original[1]:=3; Original[2]:=1; Original[3]:=4; Original[4]:=1;
  Original[5]:=5; Original[6]:=9; Original[7]:=2; Original[8]:=6;
  Original[9]:=5; Original[10]:=3;
  
  UniqueCount := 0;
  for I := 1 to 10 do
  begin
    IsDuplicate := False;
    for J := 1 to UniqueCount do
      if Unique[J] = Original[I] then
      begin
        IsDuplicate := True;
        Break;
      end;
    if not IsDuplicate then
    begin
      Inc(UniqueCount);
      Unique[UniqueCount] := Original[I];
    end;
  end;
  
  Write('Unique: ');
  for I := 1 to UniqueCount do Write(Unique[I], ' ');
  WriteLn;
end.
```

### ข้อที่ 4-20 (สั้น)

```pascal
// ข้อ 4: นับจำนวนที่มากกว่าค่าเฉลี่ย
program AboveAvgCount;
var A: array[1..10] of Real; I: Integer; Avg, Sum: Real; Count: Integer;
begin
  for I := 1 to 10 do begin Write('A[',I,']: '); ReadLn(A[I]); end;
  Sum := 0;
  for I := 1 to 10 do Sum := Sum + A[I];
  Avg := Sum / 10;
  Count := 0;
  for I := 1 to 10 do if A[I] > Avg then Inc(Count);
  WriteLn('มากกว่าค่าเฉลี่ย: ', Count, ' ค่า');
end.

// ข้อ 5: หาผลรวมแนวทแยง (Diagonal sum of matrix)
program DiagonalSum;
var M: array[1..4,1..4] of Integer; I,J: Integer; Sum: Integer;
begin
  for I := 1 to 4 do
    for J := 1 to 4 do M[I,J] := I*4+J-4;
  Sum := 0;
  for I := 1 to 4 do Sum := Sum + M[I,I];
  WriteLn('Main diagonal sum = ', Sum);
  Sum := 0;
  for I := 1 to 4 do Sum := Sum + M[I, 5-I];
  WriteLn('Anti-diagonal sum = ', Sum);
end.

// ข้อ 6: Flatten 2D array เป็น 1D
program FlattenArray;
var M: array[1..3,1..3] of Integer; Flat: array[1..9] of Integer;
    I,J,K: Integer;
begin
  K := 0;
  for I := 1 to 3 do for J := 1 to 3 do begin M[I,J]:=I*3+J-3; Inc(K); Flat[K]:=M[I,J]; end;
  Write('Flat: ');
  for I := 1 to 9 do Write(Flat[I], ' ');
  WriteLn;
end.

// ข้อ 7: Intersection ของสอง array
program ArrayIntersect;
var A: array[1..5] of Integer; B: array[1..5] of Integer;
    I,J: Integer;
begin
  A[1]:=1;A[2]:=2;A[3]:=3;A[4]:=4;A[5]:=5;
  B[1]:=3;B[2]:=4;B[3]:=5;B[4]:=6;B[5]:=7;
  Write('Intersection: ');
  for I := 1 to 5 do
    for J := 1 to 5 do
      if A[I] = B[J] then Write(A[I], ' ');
  WriteLn;
end.

// ข้อ 8: Count occurrences
program CountOccurrences;
var Arr: array[1..10] of Integer; I, Target, Count: Integer;
begin
  for I := 1 to 10 do Arr[I] := (I mod 3) + 1;
  Write('หาตัวเลข: '); ReadLn(Target);
  Count := 0;
  for I := 1 to 10 do if Arr[I] = Target then Inc(Count);
  WriteLn('พบ ', Target, ' จำนวน ', Count, ' ครั้ง');
end.

// ข้อ 9: Second largest
program SecondLargest;
var A: array[1..8] of Integer; I,Max1,Max2: Integer;
begin
  for I := 1 to 8 do begin Write('A[',I,']: '); ReadLn(A[I]); end;
  Max1 := Low(Integer); Max2 := Low(Integer);
  for I := 1 to 8 do begin
    if A[I] > Max1 then begin Max2 := Max1; Max1 := A[I]; end
    else if A[I] > Max2 then Max2 := A[I];
  end;
  WriteLn('ค่ามากที่สุด: ', Max1);
  WriteLn('ค่ามากที่สอง: ', Max2);
end.

// ข้อ 10: Sum ของ row และ column ใน matrix
program RowColSum;
var M: array[1..3,1..3] of Integer; I,J,Sum: Integer;
begin
  for I := 1 to 3 do for J := 1 to 3 do M[I,J] := I*3+J-3;
  for I := 1 to 3 do begin
    Sum := 0;
    for J := 1 to 3 do Sum := Sum + M[I,J];
    WriteLn('Row ', I, ' sum = ', Sum);
  end;
  for J := 1 to 3 do begin
    Sum := 0;
    for I := 1 to 3 do Sum := Sum + M[I,J];
    WriteLn('Col ', J, ' sum = ', Sum);
  end;
end.

// ข้อ 11-20 (one-liners ด้วย for loop)
program ArrayEx11_20;
var A: array[1..10] of Integer; I: Integer;
begin
  for I := 1 to 10 do A[I] := Random(100);
  
  // ข้อ 11: นับเลขคู่
  var Even := 0;
  for I := 1 to 10 do if A[I] mod 2 = 0 then Inc(Even);
  WriteLn('เลขคู่: ', Even, ' ตัว');
  
  // ข้อ 12: คูณทุกตัวด้วย 2
  for I := 1 to 10 do A[I] := A[I] * 2;
  
  // ข้อ 13: หา index ของค่าน้อยสุด
  var MinIdx := 1;
  for I := 2 to 10 do if A[I] < A[MinIdx] then MinIdx := I;
  WriteLn('น้อยสุดที่ index ', MinIdx);
  
  // ข้อ 14: Reverse in place
  for I := 1 to 5 do begin var T := A[I]; A[I] := A[11-I]; A[11-I] := T; end;
  
  // ข้อ 15: หาผลรวมเฉพาะเลขบวก
  var PosSum := 0;
  for I := 1 to 10 do if A[I] > 0 then PosSum := PosSum + A[I];
  WriteLn('ผลรวมเลขบวก: ', PosSum);
end.
```

---

## สรุป

| ประเภท | การประกาศ | ขนาด | Index |
|--------|-----------|------|-------|
| Static | `array[1..N] of T` | คงที่ | กำหนดเอง |
| Dynamic | `array of T` | ปรับได้ | เริ่มที่ 0 |
| 2D | `array[1..R, 1..C] of T` | คงที่ | กำหนดเอง |
| Open | `array of T` (param) | ใดก็ได้ | 0..High |

---

*จบ Part 08 - อาร์เรย์ (Arrays)*
