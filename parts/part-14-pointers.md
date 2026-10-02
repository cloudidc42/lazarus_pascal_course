# Part 14 - Pointers (ตัวชี้)

## บทนำ

Pointer คือตัวแปรที่เก็บ **ที่อยู่ (address) ในหน่วยความจำ** แทนที่จะเก็บค่าโดยตรง การเข้าใจ pointers เป็นสิ่งสำคัญมากสำหรับ:
- การจัดการหน่วยความจำแบบ dynamic
- การสร้างโครงสร้างข้อมูลซับซ้อน (linked list, tree, graph)
- การทำงานกับ large data อย่างมีประสิทธิภาพ
- การเขียน low-level code

---

## 14.1 Pointer คืออะไร

```
หน่วยความจำ (RAM):
Address  | ค่า
---------|-----
1000     | 42      <- ตัวแปร X : Integer
1004     | 1000    <- ตัวแปร P : ^Integer (pointer ชี้ไปที่ X)

P^ = 42  (เข้าถึงค่าที่ P ชี้ไป)
```

### ตัวอย่างที่ 1: Pointer พื้นฐาน

```pascal
program PointerBasic;

var
  X : Integer;    // ตัวแปรธรรมดา
  P : ^Integer;   // pointer ไปยัง Integer

begin
  X := 42;
  
  // @ = address-of operator
  P := @X;  // P เก็บ address ของ X
  
  WriteLn('ค่าของ X    : ', X);
  WriteLn('Address ของ X: ', HexStr(PtrUInt(@X), 8));
  WriteLn('ค่าของ P    : ', HexStr(PtrUInt(P), 8));
  WriteLn('P^ (dereference): ', P^);  // อ่านค่าผ่าน pointer
  
  // แก้ไขค่าผ่าน pointer
  P^ := 100;
  WriteLn('หลังแก้ผ่าน P^: X = ', X);
  
  // สังเกตว่า X เปลี่ยนไปเพราะ P ชี้ไปที่ X
  WriteLn('X ตอนนี้ = ', X);
  
  ReadLn;
end.
```

### ตัวอย่างที่ 2: ประเภท Pointer ต่างๆ

```pascal
program PointerTypes;

type
  PInteger = ^Integer;
  PReal    = ^Real;
  PChar    = ^Char;
  PBoolean = ^Boolean;
  
  TPoint = record
    X, Y : Real;
  end;
  PPoint = ^TPoint;

var
  I : Integer;
  R : Real;
  C : Char;
  B : Boolean;
  Pt : TPoint;
  
  PI : PInteger;
  PR : PReal;
  PC : PChar;
  PB : PBoolean;
  PP : PPoint;

begin
  I := 100;
  R := 3.14;
  C := 'A';
  B := True;
  Pt.X := 5.0;
  Pt.Y := 3.0;
  
  PI := @I;
  PR := @R;
  PC := @C;
  PB := @B;
  PP := @Pt;
  
  WriteLn('=== Pointer Types ===');
  WriteLn('Integer   : ', PI^);
  WriteLn('Real      : ', PR^:0:4);
  WriteLn('Char      : ', PC^);
  WriteLn('Boolean   : ', PB^);
  WriteLn('Record X  : ', PP^.X:0:2);
  WriteLn('Record Y  : ', PP^.Y:0:2);
  
  // แก้ไขผ่าน pointer
  PP^.X := 10.0;
  PP^.Y := 20.0;
  WriteLn('หลังแก้ผ่าน pointer:');
  WriteLn('Pt.X = ', Pt.X:0:2);
  WriteLn('Pt.Y = ', Pt.Y:0:2);
  
  ReadLn;
end.
```

---

## 14.2 การประกาศและใช้งาน Pointer

### ตัวอย่างที่ 3: การประกาศ Pointer Type

```pascal
program DeclarePointers;

type
  // ประกาศ pointer type พร้อมกับ type
  PStudent = ^TStudent;
  TStudent = record
    ID    : Integer;
    Name  : String[50];
    Score : Real;
  end;
  
  // หรือประกาศ forward reference
  PNode = ^TNode;
  TNode = record
    Data : Integer;
    Next : PNode;  // self-referential pointer
  end;

var
  S  : TStudent;
  PS : PStudent;
  
begin
  S.ID    := 1;
  S.Name  := 'สมชาย';
  S.Score := 85.5;
  
  PS := @S;
  
  WriteLn('ผ่าน pointer:');
  WriteLn('ID    : ', PS^.ID);
  WriteLn('Name  : ', PS^.Name);
  WriteLn('Score : ', PS^.Score:0:2);
  
  // แก้ไขผ่าน pointer
  PS^.Score := 90.0;
  WriteLn;
  WriteLn('หลังแก้ไข:');
  WriteLn('Score : ', S.Score:0:2);  // เปลี่ยนไป
  
  ReadLn;
end.
```

---

## 14.3 New และ Dispose

`New` จัดสรร (allocate) หน่วยความจำใหม่
`Dispose` คืน (deallocate) หน่วยความจำ

### ตัวอย่างที่ 4: New และ Dispose พื้นฐาน

```pascal
program NewDisposeDemo;

type
  PData = ^TData;
  TData = record
    Value : Integer;
    Name  : String[20];
  end;

var
  P : PData;

begin
  // จัดสรรหน่วยความจำ
  New(P);  // สร้าง TData ใหม่ใน heap
  
  // ใช้งาน
  P^.Value := 42;
  P^.Name  := 'ทดสอบ';
  
  WriteLn('Value = ', P^.Value);
  WriteLn('Name  = ', P^.Name);
  WriteLn('Address: ', HexStr(PtrUInt(P), 8));
  
  // คืนหน่วยความจำ (สำคัญมาก!)
  Dispose(P);
  P := nil;  // ป้องกัน dangling pointer
  
  WriteLn('หลัง Dispose, P = nil: ', P = nil);
  
  ReadLn;
end.
```

### ตัวอย่างที่ 5: New หลายๆ ตัว

```pascal
program MultipleNew;

type
  PInteger = ^Integer;

var
  P1, P2, P3 : PInteger;
  Sum : Integer;

begin
  New(P1);
  New(P2);
  New(P3);
  
  P1^ := 10;
  P2^ := 20;
  P3^ := 30;
  
  Sum := P1^ + P2^ + P3^;
  
  WriteLn('P1^ = ', P1^);
  WriteLn('P2^ = ', P2^);
  WriteLn('P3^ = ', P3^);
  WriteLn('Sum = ', Sum);
  
  // addresses ต่างกัน
  WriteLn;
  WriteLn('Address P1: ', HexStr(PtrUInt(P1), 8));
  WriteLn('Address P2: ', HexStr(PtrUInt(P2), 8));
  WriteLn('Address P3: ', HexStr(PtrUInt(P3), 8));
  
  // คืนหน่วยความจำ
  Dispose(P3);
  Dispose(P2);
  Dispose(P1);
  
  ReadLn;
end.
```

---

## 14.4 Nil Pointer

`nil` คือ pointer ที่ไม่ชี้ไปที่ไหนเลย ใช้ตรวจสอบว่า pointer ถูก assign แล้วหรือยัง

### ตัวอย่างที่ 6: การตรวจสอบ nil

```pascal
program NilPointerCheck;

type
  PValue = ^Integer;

function SafeRead(P: PValue): Integer;
begin
  if P = nil then
  begin
    WriteLn('WARNING: nil pointer!');
    Result := 0;
    Exit;
  end;
  Result := P^;
end;

procedure SafeWrite(P: PValue; Value: Integer);
begin
  if P = nil then
  begin
    WriteLn('ERROR: Cannot write to nil pointer!');
    Exit;
  end;
  P^ := Value;
end;

var
  P1, P2 : PValue;
  X      : Integer;

begin
  P1 := nil;  // ตอนแรกเป็น nil
  X  := 42;
  P2 := @X;   // ชี้ไปที่ X
  
  WriteLn('P1 = nil: ', P1 = nil);
  WriteLn('P2 = nil: ', P2 = nil);
  WriteLn;
  
  WriteLn('อ่านจาก P1 (nil): ', SafeRead(P1));
  WriteLn('อ่านจาก P2: ', SafeRead(P2));
  WriteLn;
  
  SafeWrite(P1, 100);  // จะแสดง error
  SafeWrite(P2, 100);  // สำเร็จ
  WriteLn('X หลัง SafeWrite: ', X);
  
  ReadLn;
end.
```

---

## 14.5 Pointer Arithmetic

Pascal รองรับ pointer arithmetic กับ typed pointers

### ตัวอย่างที่ 7: เดิน pointer ใน array

```pascal
program PointerArithmetic;

var
  Arr    : array[0..4] of Integer;
  P      : ^Integer;
  i      : Integer;

begin
  // กำหนดค่า array
  for i := 0 to 4 do
    Arr[i] := (i + 1) * 10;
  
  WriteLn('=== Array ผ่าน Pointer ===');
  
  // ชี้ไปที่ element แรก
  P := @Arr[0];
  
  // อ่านทุก element ด้วย pointer
  for i := 0 to 4 do
  begin
    WriteLn(Format('Arr[%d] = %d  (address: %s)',
      [i, P^, HexStr(PtrUInt(P), 8)]));
    Inc(P);  // เลื่อน pointer ไปหน้า sizeof(Integer) bytes
  end;
  
  WriteLn;
  
  // เดินถอยหลัง
  WriteLn('เดินถอยหลัง:');
  Dec(P);  // กลับมาที่ element สุดท้าย
  for i := 4 downto 0 do
  begin
    WriteLn(Format('Arr[%d] = %d', [i, P^]));
    if i > 0 then Dec(P);
  end;
  
  ReadLn;
end.
```

### ตัวอย่างที่ 8: PChar และการจัดการ String

```pascal
program PCharDemo;

var
  S    : String;
  PC   : PChar;
  i    : Integer;

procedure ReverseString(P: PChar; Len: Integer);
var
  Left, Right : Integer;
  Temp        : Char;
begin
  Left  := 0;
  Right := Len - 1;
  while Left < Right do
  begin
    Temp        := P[Left];
    P[Left]     := P[Right];
    P[Right]    := Temp;
    Inc(Left);
    Dec(Right);
  end;
end;

function CountChar(P: PChar; C: Char): Integer;
begin
  Result := 0;
  while P^ <> #0 do
  begin
    if P^ = C then Inc(Result);
    Inc(P);
  end;
end;

begin
  S  := 'Hello, Pascal!';
  PC := PChar(S);
  
  WriteLn('String: ', S);
  WriteLn('Length: ', Length(S));
  WriteLn;
  
  // อ่านทีละตัว
  WriteLn('ตัวอักษรทีละตัว:');
  i := 0;
  while PC[i] <> #0 do
  begin
    Write(PC[i], ' ');
    Inc(i);
  end;
  WriteLn;
  WriteLn;
  
  // นับตัวอักษร
  WriteLn('จำนวน l: ', CountChar(PC, 'l'));
  WriteLn;
  
  // กลับด้าน string
  var SC : String;
  SC := S;
  var PCS := PChar(SC);
  ReverseString(PCS, Length(SC));
  WriteLn('Reversed: ', SC);
  
  ReadLn;
end.
```

---

## 14.6 Typed vs Untyped Pointers

### ตัวอย่างที่ 9: Untyped Pointer

```pascal
program UntypedPointer;

var
  I : Integer;
  R : Real;
  P : Pointer;  // untyped pointer

begin
  I := 100;
  R := 3.14;
  
  // Untyped pointer สามารถชี้ไปที่อะไรก็ได้
  P := @I;
  WriteLn('ชี้ไปที่ Integer: ', Integer(P^));
  
  P := @R;
  WriteLn('ชี้ไปที่ Real: ', Real(P^):0:4);
  
  // ต้อง cast ก่อนใช้
  P := @I;
  PInteger(P)^ := 999;
  WriteLn('หลัง cast และแก้: I = ', I);
  
  // GetMem/FreeMem กับ untyped pointer
  var Buf : Pointer;
  GetMem(Buf, 100);
  
  // เขียน bytes ลงไป
  var ByteP := PByte(Buf);
  for var j := 0 to 9 do
  begin
    ByteP^ := j * 10;
    Inc(ByteP);
  end;
  
  // อ่านกลับ
  ByteP := PByte(Buf);
  Write('Bytes: ');
  for var j := 0 to 9 do
  begin
    Write(ByteP^, ' ');
    Inc(ByteP);
  end;
  WriteLn;
  
  FreeMem(Buf);
  
  ReadLn;
end.
```

---

## 14.7 Pointer กับ Arrays

### ตัวอย่างที่ 10: Dynamic Array ด้วย Pointer

```pascal
program DynamicArrayPointer;

type
  PIntArray = ^TIntArray;
  TIntArray = array[0..0] of Integer;  // open array trick

procedure FillArray(P: PIntArray; Size, StartVal: Integer);
var i: Integer;
begin
  for i := 0 to Size - 1 do
    P^[i] := StartVal + i;
end;

procedure PrintArray(P: PIntArray; Size: Integer);
var i: Integer;
begin
  Write('[');
  for i := 0 to Size - 1 do
  begin
    if i > 0 then Write(', ');
    Write(P^[i]);
  end;
  WriteLn(']');
end;

function SumArray(P: PIntArray; Size: Integer): Integer;
var i: Integer;
begin
  Result := 0;
  for i := 0 to Size - 1 do
    Result := Result + P^[i];
end;

var
  ArrPtr : PIntArray;
  Size   : Integer;

begin
  Size := 10;
  
  // จัดสรรหน่วยความจำสำหรับ 10 integers
  GetMem(ArrPtr, Size * SizeOf(Integer));
  
  FillArray(ArrPtr, Size, 1);
  
  Write('Array: ');
  PrintArray(ArrPtr, Size);
  
  WriteLn('Sum: ', SumArray(ArrPtr, Size));
  
  // แก้ไขบาง elements
  ArrPtr^[0] := 100;
  ArrPtr^[9] := 200;
  
  Write('หลังแก้ไข: ');
  PrintArray(ArrPtr, Size);
  
  // คืนหน่วยความจำ
  FreeMem(ArrPtr, Size * SizeOf(Integer));
  
  ReadLn;
end.
```

---

## 14.8 Pointer กับ Records

### ตัวอย่างที่ 11: Pointer กับ Records

```pascal
program PointerWithRecords;

type
  TColor = record
    R, G, B : Byte;
    Alpha   : Byte;
  end;
  PColor = ^TColor;
  
  TSprite = record
    X, Y     : Real;
    Width    : Integer;
    Height   : Integer;
    Color    : TColor;
    Visible  : Boolean;
    ZOrder   : Integer;
  end;
  PSprite = ^TSprite;

procedure CreateSprite(var P: PSprite; X, Y: Real; W, H: Integer;
                        R, G, B: Byte);
begin
  New(P);
  P^.X       := X;
  P^.Y       := Y;
  P^.Width   := W;
  P^.Height  := H;
  P^.Color.R := R;
  P^.Color.G := G;
  P^.Color.B := B;
  P^.Color.Alpha := 255;
  P^.Visible := True;
  P^.ZOrder  := 0;
end;

procedure MoveSprite(P: PSprite; DX, DY: Real);
begin
  if P = nil then Exit;
  P^.X := P^.X + DX;
  P^.Y := P^.Y + DY;
end;

procedure DisplaySprite(P: PSprite);
begin
  if P = nil then begin WriteLn('Sprite is nil'); Exit; end;
  WriteLn(Format('Sprite at (%.1f, %.1f) size=%dx%d RGB=(%d,%d,%d) visible=%s',
    [P^.X, P^.Y, P^.Width, P^.Height,
     P^.Color.R, P^.Color.G, P^.Color.B,
     BoolToStr(P^.Visible, 'Yes', 'No')]));
end;

procedure DestroySprite(var P: PSprite);
begin
  if P <> nil then
  begin
    Dispose(P);
    P := nil;
  end;
end;

var
  S1, S2, S3 : PSprite;

begin
  CreateSprite(S1, 10, 20, 64, 64, 255, 0, 0);    // Red
  CreateSprite(S2, 100, 50, 32, 32, 0, 255, 0);   // Green
  CreateSprite(S3, 200, 100, 128, 128, 0, 0, 255); // Blue
  
  WriteLn('=== Sprites ===');
  DisplaySprite(S1);
  DisplaySprite(S2);
  DisplaySprite(S3);
  WriteLn;
  
  MoveSprite(S1, 5, 5);
  MoveSprite(S2, -10, 20);
  
  WriteLn('หลัง Move:');
  DisplaySprite(S1);
  DisplaySprite(S2);
  
  // S2 ซ่อน
  S2^.Visible := False;
  WriteLn;
  WriteLn('หลังซ่อน S2:');
  DisplaySprite(S2);
  
  // คืนหน่วยความจำ
  DestroySprite(S1);
  DestroySprite(S2);
  DestroySprite(S3);
  
  WriteLn;
  WriteLn('หลัง Destroy:');
  WriteLn('S1 = nil: ', S1 = nil);
  WriteLn('S2 = nil: ', S2 = nil);
  WriteLn('S3 = nil: ', S3 = nil);
  
  ReadLn;
end.
```

---

## 14.9 Linked List

Linked List เป็นโครงสร้างข้อมูลพื้นฐานที่ใช้ pointers

### ตัวอย่างที่ 12: Singly Linked List

```pascal
program LinkedListDemo;

{$mode objfpc}{$H+}

type
  PNode = ^TNode;
  TNode = record
    Data : Integer;
    Next : PNode;
  end;
  
  TLinkedList = record
    Head  : PNode;
    Count : Integer;
  end;

procedure InitList(var L: TLinkedList);
begin
  L.Head  := nil;
  L.Count := 0;
end;

// เพิ่มที่ต้น (O(1))
procedure PushFront(var L: TLinkedList; Value: Integer);
var
  NewNode : PNode;
begin
  New(NewNode);
  NewNode^.Data := Value;
  NewNode^.Next := L.Head;
  L.Head        := NewNode;
  Inc(L.Count);
end;

// เพิ่มที่ท้าย (O(n))
procedure PushBack(var L: TLinkedList; Value: Integer);
var
  NewNode, Current : PNode;
begin
  New(NewNode);
  NewNode^.Data := Value;
  NewNode^.Next := nil;
  
  if L.Head = nil then
    L.Head := NewNode
  else
  begin
    Current := L.Head;
    while Current^.Next <> nil do
      Current := Current^.Next;
    Current^.Next := NewNode;
  end;
  Inc(L.Count);
end;

// ลบที่ต้น (O(1))
function PopFront(var L: TLinkedList): Integer;
var
  OldHead : PNode;
begin
  if L.Head = nil then
  begin
    Result := 0;
    WriteLn('List ว่าง!');
    Exit;
  end;
  OldHead := L.Head;
  Result  := OldHead^.Data;
  L.Head  := OldHead^.Next;
  Dispose(OldHead);
  Dec(L.Count);
end;

// ค้นหา
function Contains(L: TLinkedList; Value: Integer): Boolean;
var
  Current : PNode;
begin
  Current := L.Head;
  while Current <> nil do
  begin
    if Current^.Data = Value then begin Result := True; Exit; end;
    Current := Current^.Next;
  end;
  Result := False;
end;

// ลบตัวที่ระบุ
function Remove(var L: TLinkedList; Value: Integer): Boolean;
var
  Current, Prev : PNode;
begin
  Result  := False;
  Current := L.Head;
  Prev    := nil;
  
  while Current <> nil do
  begin
    if Current^.Data = Value then
    begin
      if Prev = nil then
        L.Head := Current^.Next
      else
        Prev^.Next := Current^.Next;
      Dispose(Current);
      Dec(L.Count);
      Result := True;
      Exit;
    end;
    Prev    := Current;
    Current := Current^.Next;
  end;
end;

procedure PrintList(L: TLinkedList);
var
  Current : PNode;
begin
  Write('List[', L.Count, ']: ');
  Current := L.Head;
  while Current <> nil do
  begin
    Write(Current^.Data);
    if Current^.Next <> nil then Write(' -> ');
    Current := Current^.Next;
  end;
  WriteLn(' -> nil');
end;

procedure FreeList(var L: TLinkedList);
var
  Current, Next : PNode;
begin
  Current := L.Head;
  while Current <> nil do
  begin
    Next := Current^.Next;
    Dispose(Current);
    Current := Next;
  end;
  L.Head  := nil;
  L.Count := 0;
end;

// แปลงเป็น array
procedure ListToArray(L: TLinkedList; var Arr: array of Integer);
var
  Current : PNode;
  i       : Integer;
begin
  Current := L.Head;
  i       := 0;
  while (Current <> nil) and (i <= High(Arr)) do
  begin
    Arr[i]  := Current^.Data;
    Current := Current^.Next;
    Inc(i);
  end;
end;

var
  List : TLinkedList;
  Arr  : array[0..9] of Integer;

begin
  InitList(List);
  
  // เพิ่มข้อมูล
  PushBack(List, 10);
  PushBack(List, 20);
  PushBack(List, 30);
  PushFront(List, 5);
  PushFront(List, 1);
  
  WriteLn('=== Linked List Demo ===');
  PrintList(List);
  
  WriteLn;
  WriteLn('Contains 20: ', Contains(List, 20));
  WriteLn('Contains 99: ', Contains(List, 99));
  
  WriteLn;
  WriteLn('PopFront: ', PopFront(List));
  PrintList(List);
  
  WriteLn;
  Remove(List, 20);
  WriteLn('หลังลบ 20:');
  PrintList(List);
  
  WriteLn;
  // แปลงเป็น array
  SetLength(Arr, 0);
  var DynArr : array of Integer;
  SetLength(DynArr, List.Count);
  ListToArray(List, DynArr);
  Write('Array: ');
  for var i := 0 to High(DynArr) do
    Write(DynArr[i], ' ');
  WriteLn;
  
  FreeList(List);
  WriteLn;
  WriteLn('หลัง Free:');
  PrintList(List);
  
  ReadLn;
end.
```

---

## 14.10 Stack Using Pointers

### ตัวอย่างที่ 13: Stack Implementation

```pascal
program StackPointer;

{$mode objfpc}{$H+}

type
  PStackNode = ^TStackNode;
  TStackNode = record
    Data : Integer;
    Next : PStackNode;
  end;
  
  TStack = record
    Top   : PStackNode;
    Size  : Integer;
  end;

procedure InitStack(var S: TStack);
begin
  S.Top  := nil;
  S.Size := 0;
end;

function IsEmpty(S: TStack): Boolean;
begin
  Result := S.Top = nil;
end;

procedure Push(var S: TStack; Value: Integer);
var
  Node : PStackNode;
begin
  New(Node);
  Node^.Data := Value;
  Node^.Next := S.Top;
  S.Top      := Node;
  Inc(S.Size);
end;

function Pop(var S: TStack): Integer;
var
  OldTop : PStackNode;
begin
  if IsEmpty(S) then
  begin
    WriteLn('Stack underflow!');
    Result := 0;
    Exit;
  end;
  OldTop := S.Top;
  Result := OldTop^.Data;
  S.Top  := OldTop^.Next;
  Dispose(OldTop);
  Dec(S.Size);
end;

function Peek(S: TStack): Integer;
begin
  if IsEmpty(S) then
  begin
    WriteLn('Stack empty!');
    Result := 0;
    Exit;
  end;
  Result := S.Top^.Data;
end;

procedure PrintStack(S: TStack);
var
  Current : PStackNode;
begin
  Write('Stack[', S.Size, ']: ');
  if IsEmpty(S) then begin WriteLn('(empty)'); Exit; end;
  
  Current := S.Top;
  Write('[TOP] ');
  while Current <> nil do
  begin
    Write(Current^.Data);
    if Current^.Next <> nil then Write(' | ');
    Current := Current^.Next;
  end;
  WriteLn(' [BOTTOM]');
end;

procedure FreeStack(var S: TStack);
begin
  while not IsEmpty(S) do Pop(S);
end;

// ตัวอย่างใช้งาน: ตรวจสอบ balanced brackets
function IsBalanced(S: String): Boolean;
var
  Stack    : TStack;
  i        : Integer;
  Ch, Top  : Char;
  
  function CharToInt(C: Char): Integer;
  begin Result := Ord(C); end;
  
  function IntToChar(I: Integer): Char;
  begin Result := Chr(I); end;
  
begin
  InitStack(Stack);
  Result := True;
  
  for i := 1 to Length(S) do
  begin
    Ch := S[i];
    case Ch of
      '(','[','{': Push(Stack, CharToInt(Ch));
      ')':
        begin
          if IsEmpty(Stack) or (IntToChar(Pop(Stack)) <> '(') then
            begin Result := False; Break; end;
        end;
      ']':
        begin
          if IsEmpty(Stack) or (IntToChar(Pop(Stack)) <> '[') then
            begin Result := False; Break; end;
        end;
      '}':
        begin
          if IsEmpty(Stack) or (IntToChar(Pop(Stack)) <> '{') then
            begin Result := False; Break; end;
        end;
    end;
  end;
  
  if not IsEmpty(Stack) then Result := False;
  FreeStack(Stack);
end;

// ตัวอย่าง: แปลงเลขฐาน 10 เป็น ฐาน 2
function ToBinary(N: Integer): String;
var
  S : TStack;
begin
  InitStack(S);
  Result := '';
  
  if N = 0 then begin Result := '0'; Exit; end;
  
  while N > 0 do
  begin
    Push(S, N mod 2);
    N := N div 2;
  end;
  
  while not IsEmpty(S) do
    Result := Result + IntToStr(Pop(S));
    
  FreeStack(S);
end;

var
  S : TStack;

begin
  InitStack(S);
  
  WriteLn('=== Stack Demo ===');
  
  Push(S, 10);
  Push(S, 20);
  Push(S, 30);
  Push(S, 40);
  Push(S, 50);
  
  PrintStack(S);
  WriteLn('Peek: ', Peek(S));
  WriteLn('Pop: ', Pop(S));
  WriteLn('Pop: ', Pop(S));
  PrintStack(S);
  
  WriteLn;
  WriteLn('=== Balanced Brackets ===');
  var Exprs := ['(a+b)', '(a+[b*c])', '((a+b)', '{a+[b*(c+d)]}'];
  for var i := 0 to High(Exprs) do
    WriteLn(Format('%-20s: %s', [Exprs[i],
      BoolToStr(IsBalanced(Exprs[i]), 'Balanced', 'Unbalanced')]));
  
  WriteLn;
  WriteLn('=== แปลงเลขฐาน 2 ===');
  for var n := 0 to 10 do
    WriteLn(Format('%3d -> %s', [n, ToBinary(n)]));
  
  FreeStack(S);
  ReadLn;
end.
```

---

## 14.11 Queue Using Pointers

### ตัวอย่างที่ 14: Queue Implementation

```pascal
program QueuePointer;

{$mode objfpc}{$H+}

type
  PQNode = ^TQNode;
  TQNode = record
    Data : String[50];
    Next : PQNode;
  end;
  
  TQueue = record
    Front : PQNode;
    Rear  : PQNode;
    Size  : Integer;
  end;

procedure InitQueue(var Q: TQueue);
begin
  Q.Front := nil;
  Q.Rear  := nil;
  Q.Size  := 0;
end;

function IsEmptyQ(Q: TQueue): Boolean;
begin
  Result := Q.Front = nil;
end;

procedure Enqueue(var Q: TQueue; Value: String);
var
  Node : PQNode;
begin
  New(Node);
  Node^.Data := Value;
  Node^.Next := nil;
  
  if Q.Rear = nil then
    Q.Front := Node
  else
    Q.Rear^.Next := Node;
  Q.Rear := Node;
  Inc(Q.Size);
end;

function Dequeue(var Q: TQueue): String;
var
  OldFront : PQNode;
begin
  if IsEmptyQ(Q) then
  begin
    Result := '';
    WriteLn('Queue empty!');
    Exit;
  end;
  OldFront := Q.Front;
  Result   := OldFront^.Data;
  Q.Front  := OldFront^.Next;
  if Q.Front = nil then Q.Rear := nil;
  Dispose(OldFront);
  Dec(Q.Size);
end;

function PeekQ(Q: TQueue): String;
begin
  if IsEmptyQ(Q) then Result := '(empty)'
  else Result := Q.Front^.Data;
end;

procedure PrintQueue(Q: TQueue);
var
  Current : PQNode;
begin
  Write(Format('Queue[%d]: ', [Q.Size]));
  if IsEmptyQ(Q) then begin WriteLn('(empty)'); Exit; end;
  
  Write('[FRONT] ');
  Current := Q.Front;
  while Current <> nil do
  begin
    Write('"', Current^.Data, '"');
    if Current^.Next <> nil then Write(' -> ');
    Current := Current^.Next;
  end;
  WriteLn(' [REAR]');
end;

procedure FreeQueue(var Q: TQueue);
begin
  while not IsEmptyQ(Q) do Dequeue(Q);
end;

// ตัวอย่างใช้งาน: จำลองคิวร้านอาหาร
procedure RestaurantQueueDemo;
var
  Q     : TQueue;
  Order : String;

begin
  InitQueue(Q);
  
  WriteLn('=== จำลองคิวร้านอาหาร ===');
  WriteLn;
  
  // ลูกค้าเข้าคิว
  WriteLn('ลูกค้าเข้าคิว:');
  Enqueue(Q, 'โต๊ะ 1: ผัดไทย 2');
  WriteLn('  + โต๊ะ 1 (ผัดไทย)');
  Enqueue(Q, 'โต๊ะ 3: ข้าวผัด 3');
  WriteLn('  + โต๊ะ 3 (ข้าวผัด)');
  Enqueue(Q, 'Delivery: ต้มยำกุ้ง 1');
  WriteLn('  + Delivery (ต้มยำกุ้ง)');
  Enqueue(Q, 'โต๊ะ 2: ก๋วยเตี๋ยว 2');
  WriteLn('  + โต๊ะ 2 (ก๋วยเตี๋ยว)');
  WriteLn;
  
  PrintQueue(Q);
  WriteLn;
  
  // ครัวทำอาหาร
  WriteLn('ครัวกำลังทำ:');
  while not IsEmptyQ(Q) do
  begin
    Order := Dequeue(Q);
    WriteLn('  -> ส่ง: ', Order);
    WriteLn('     คิวที่เหลือ: ', Q.Size, ' รายการ');
  end;
  
  FreeQueue(Q);
end;

// Priority Queue อย่างง่าย (ใช้ insertion sort)
type
  PPQNode = ^TPQNode;
  TPQNode = record
    Data     : String[50];
    Priority : Integer;  // ต่ำ = priority สูง
    Next     : PPQNode;
  end;
  
  TPriorityQueue = record
    Head : PPQNode;
    Size : Integer;
  end;

procedure InitPQ(var PQ: TPriorityQueue);
begin
  PQ.Head := nil;
  PQ.Size := 0;
end;

procedure EnqueuePQ(var PQ: TPriorityQueue; Data: String; Priority: Integer);
var
  Node, Current, Prev : PPQNode;
begin
  New(Node);
  Node^.Data     := Data;
  Node^.Priority := Priority;
  Node^.Next     := nil;
  
  // แทรกตาม priority
  if (PQ.Head = nil) or (Priority < PQ.Head^.Priority) then
  begin
    Node^.Next := PQ.Head;
    PQ.Head    := Node;
  end
  else
  begin
    Current := PQ.Head;
    while (Current^.Next <> nil) and
          (Current^.Next^.Priority <= Priority) do
      Current := Current^.Next;
    Node^.Next    := Current^.Next;
    Current^.Next := Node;
  end;
  Inc(PQ.Size);
end;

function DequeuePQ(var PQ: TPriorityQueue): String;
var
  OldHead : PPQNode;
begin
  if PQ.Head = nil then begin Result := ''; Exit; end;
  OldHead  := PQ.Head;
  Result   := OldHead^.Data;
  PQ.Head  := OldHead^.Next;
  Dispose(OldHead);
  Dec(PQ.Size);
end;

begin
  RestaurantQueueDemo;
  WriteLn;
  
  // Priority Queue Demo
  WriteLn('=== Priority Queue Demo ===');
  var PQ : TPriorityQueue;
  InitPQ(PQ);
  
  EnqueuePQ(PQ, 'งานปกติ', 3);
  EnqueuePQ(PQ, 'งานด่วนมาก', 1);
  EnqueuePQ(PQ, 'งานด่วน', 2);
  EnqueuePQ(PQ, 'งานทั่วไป', 3);
  EnqueuePQ(PQ, 'งาน Emergency', 0);
  
  WriteLn('ประมวลผลตามลำดับ priority:');
  while PQ.Head <> nil do
    WriteLn('  [', PQ.Head^.Priority, '] ', DequeuePQ(PQ));
  
  ReadLn;
end.
```

---

## 14.12 Doubly Linked List

### ตัวอย่างที่ 15: Doubly Linked List

```pascal
program DoublyLinkedList;

{$mode objfpc}{$H+}

type
  PDNode = ^TDNode;
  TDNode = record
    Data : String[50];
    Prev : PDNode;
    Next : PDNode;
  end;
  
  TDList = record
    Head  : PDNode;
    Tail  : PDNode;
    Count : Integer;
  end;

procedure InitDList(var L: TDList);
begin
  L.Head  := nil;
  L.Tail  := nil;
  L.Count := 0;
end;

procedure DPushBack(var L: TDList; Data: String);
var
  Node : PDNode;
begin
  New(Node);
  Node^.Data := Data;
  Node^.Next := nil;
  Node^.Prev := L.Tail;
  
  if L.Tail = nil then
    L.Head := Node
  else
    L.Tail^.Next := Node;
  L.Tail := Node;
  Inc(L.Count);
end;

procedure DPushFront(var L: TDList; Data: String);
var
  Node : PDNode;
begin
  New(Node);
  Node^.Data := Data;
  Node^.Next := L.Head;
  Node^.Prev := nil;
  
  if L.Head = nil then
    L.Tail := Node
  else
    L.Head^.Prev := Node;
  L.Head := Node;
  Inc(L.Count);
end;

function DPopBack(var L: TDList): String;
var
  OldTail : PDNode;
begin
  if L.Tail = nil then begin Result := ''; Exit; end;
  OldTail := L.Tail;
  Result  := OldTail^.Data;
  L.Tail  := OldTail^.Prev;
  if L.Tail = nil then L.Head := nil
  else L.Tail^.Next := nil;
  Dispose(OldTail);
  Dec(L.Count);
end;

procedure PrintForward(L: TDList);
var
  Current : PDNode;
begin
  Write('Forward[', L.Count, ']: ');
  Current := L.Head;
  while Current <> nil do
  begin
    Write('"', Current^.Data, '"');
    if Current^.Next <> nil then Write(' <-> ');
    Current := Current^.Next;
  end;
  WriteLn;
end;

procedure PrintBackward(L: TDList);
var
  Current : PDNode;
begin
  Write('Backward: ');
  Current := L.Tail;
  while Current <> nil do
  begin
    Write('"', Current^.Data, '"');
    if Current^.Prev <> nil then Write(' <-> ');
    Current := Current^.Prev;
  end;
  WriteLn;
end;

procedure FreeDList(var L: TDList);
var
  Current, Next : PDNode;
begin
  Current := L.Head;
  while Current <> nil do
  begin
    Next := Current^.Next;
    Dispose(Current);
    Current := Next;
  end;
  L.Head  := nil;
  L.Tail  := nil;
  L.Count := 0;
end;

var
  DL : TDList;

begin
  InitDList(DL);
  
  WriteLn('=== Doubly Linked List ===');
  
  DPushBack(DL, 'สอง');
  DPushBack(DL, 'สาม');
  DPushBack(DL, 'สี่');
  DPushFront(DL, 'หนึ่ง');
  DPushBack(DL, 'ห้า');
  
  PrintForward(DL);
  PrintBackward(DL);
  WriteLn;
  
  WriteLn('PopBack: "', DPopBack(DL), '"');
  WriteLn('PopBack: "', DPopBack(DL), '"');
  PrintForward(DL);
  
  FreeDList(DL);
  WriteLn;
  WriteLn('หลัง Free: Count = ', DL.Count);
  
  ReadLn;
end.
```

---

## 14.13 Memory Leaks และการป้องกัน

### ตัวอย่างที่ 16: ตรวจสอบ Memory Leaks

```pascal
program MemoryLeakDemo;

{$mode objfpc}{$H+}

uses HeapTrc;  // เปิด heap tracing

type
  PData = ^TData;
  TData = record
    ID   : Integer;
    Name : String[30];
    Next : PData;
  end;

// BAD: Memory leak example (อย่าทำแบบนี้!)
procedure BadFunction;
var
  P : PData;
begin
  New(P);
  P^.ID   := 1;
  P^.Name := 'test';
  // ลืม Dispose! -> Memory Leak
  WriteLn('BadFunction: ลืม Dispose P');
end;

// GOOD: Proper cleanup
procedure GoodFunction;
var
  P : PData;
begin
  New(P);
  try
    P^.ID   := 2;
    P^.Name := 'good';
    WriteLn('GoodFunction: P = ', P^.Name);
    // อาจมี exception ที่นี่
  finally
    Dispose(P);  // รับรองว่าจะ Dispose เสมอ
    P := nil;
  end;
end;

// GOOD: Clean linked list
procedure CleanLinkedList;
var
  Head, Current, Next : PData;
  i : Integer;
begin
  Head := nil;
  
  // สร้าง list
  for i := 1 to 5 do
  begin
    New(Current);
    Current^.ID   := i;
    Current^.Name := 'Node ' + IntToStr(i);
    Current^.Next := Head;
    Head          := Current;
  end;
  
  // ใช้งาน
  Current := Head;
  while Current <> nil do
  begin
    WriteLn('Node: ', Current^.ID, ' - ', Current^.Name);
    Current := Current^.Next;
  end;
  
  // ล้างทั้งหมด
  Current := Head;
  while Current <> nil do
  begin
    Next    := Current^.Next;
    Dispose(Current);
    Current := Next;
  end;
  Head := nil;
  WriteLn('List freed successfully');
end;

begin
  // BadFunction;  // จะแสดง leak ถ้า uncomment
  GoodFunction;
  WriteLn;
  CleanLinkedList;
  
  ReadLn;
end.
```

### ตัวอย่างที่ 17: Smart Pointer Pattern

```pascal
program SmartPointerPattern;

{$mode objfpc}{$H+}

// จำลอง smart pointer ด้วย record + finalization
type
  TResourceType = (rtFile, rtMemory, rtNetwork);
  
  TResource = record
    Name     : String[50];
    ResType  : TResourceType;
    IsOpen   : Boolean;
    Data     : Pointer;
    DataSize : Integer;
  end;
  PResource = ^TResource;

function CreateResource(Name: String; ResType: TResourceType): PResource;
begin
  New(Result);
  Result^.Name     := Name;
  Result^.ResType  := ResType;
  Result^.IsOpen   := True;
  Result^.Data     := nil;
  Result^.DataSize := 0;
  WriteLn('[Resource] Created: "', Name, '"');
end;

procedure AllocData(R: PResource; Size: Integer);
begin
  if R = nil then Exit;
  if R^.Data <> nil then FreeMem(R^.Data);
  GetMem(R^.Data, Size);
  R^.DataSize := Size;
  WriteLn('[Resource] Allocated ', Size, ' bytes for "', R^.Name, '"');
end;

procedure FreeResource(var R: PResource);
begin
  if R = nil then Exit;
  if R^.Data <> nil then
  begin
    FreeMem(R^.Data, R^.DataSize);
    R^.Data     := nil;
    R^.DataSize := 0;
  end;
  R^.IsOpen := False;
  WriteLn('[Resource] Freed: "', R^.Name, '"');
  Dispose(R);
  R := nil;
end;

var
  R1, R2 : PResource;

begin
  WriteLn('=== Smart Pointer Pattern ===');
  WriteLn;
  
  R1 := CreateResource('Database Connection', rtNetwork);
  R2 := CreateResource('Config File', rtFile);
  
  AllocData(R1, 1024);
  AllocData(R2, 256);
  
  WriteLn;
  WriteLn('ใช้งาน resources...');
  WriteLn('R1: ', R1^.Name, ' (open: ', R1^.IsOpen, ')');
  WriteLn('R2: ', R2^.Name, ' (size: ', R2^.DataSize, ' bytes)');
  
  WriteLn;
  WriteLn('คืน resources:');
  FreeResource(R1);
  FreeResource(R2);
  
  WriteLn;
  WriteLn('R1 = nil: ', R1 = nil);
  WriteLn('R2 = nil: ', R2 = nil);
  
  ReadLn;
end.
```

---

## 14.14 Binary Tree

### ตัวอย่างที่ 18: Binary Search Tree

```pascal
program BinarySearchTree;

{$mode objfpc}{$H+}

type
  PBSTNode = ^TBSTNode;
  TBSTNode = record
    Data  : Integer;
    Left  : PBSTNode;
    Right : PBSTNode;
  end;
  
  TBST = record
    Root : PBSTNode;
    Size : Integer;
  end;

procedure InitBST(var T: TBST);
begin
  T.Root := nil;
  T.Size := 0;
end;

procedure Insert(var T: TBST; Value: Integer);

  procedure InsertNode(var Node: PBSTNode; Value: Integer);
  begin
    if Node = nil then
    begin
      New(Node);
      Node^.Data  := Value;
      Node^.Left  := nil;
      Node^.Right := nil;
    end
    else if Value < Node^.Data then
      InsertNode(Node^.Left, Value)
    else if Value > Node^.Data then
      InsertNode(Node^.Right, Value);
    // ถ้า equal ไม่แทรก
  end;
  
begin
  InsertNode(T.Root, Value);
  Inc(T.Size);
end;

function Search(T: TBST; Value: Integer): Boolean;
var
  Current : PBSTNode;
begin
  Current := T.Root;
  while Current <> nil do
  begin
    if Value = Current^.Data then begin Result := True; Exit; end
    else if Value < Current^.Data then Current := Current^.Left
    else Current := Current^.Right;
  end;
  Result := False;
end;

// In-order traversal (Left -> Root -> Right) = sorted order
procedure InOrder(Node: PBSTNode);
begin
  if Node = nil then Exit;
  InOrder(Node^.Left);
  Write(Node^.Data, ' ');
  InOrder(Node^.Right);
end;

// Pre-order traversal (Root -> Left -> Right)
procedure PreOrder(Node: PBSTNode);
begin
  if Node = nil then Exit;
  Write(Node^.Data, ' ');
  PreOrder(Node^.Left);
  PreOrder(Node^.Right);
end;

// Post-order traversal (Left -> Right -> Root)
procedure PostOrder(Node: PBSTNode);
begin
  if Node = nil then Exit;
  PostOrder(Node^.Left);
  PostOrder(Node^.Right);
  Write(Node^.Data, ' ');
end;

function Height(Node: PBSTNode): Integer;
var
  LeftH, RightH : Integer;
begin
  if Node = nil then begin Result := 0; Exit; end;
  LeftH  := Height(Node^.Left);
  RightH := Height(Node^.Right);
  if LeftH > RightH then Result := LeftH + 1
  else Result := RightH + 1;
end;

procedure FreeBST(var T: TBST);

  procedure FreeNode(var Node: PBSTNode);
  begin
    if Node = nil then Exit;
    FreeNode(Node^.Left);
    FreeNode(Node^.Right);
    Dispose(Node);
    Node := nil;
  end;
  
begin
  FreeNode(T.Root);
  T.Size := 0;
end;

var
  T : TBST;

begin
  InitBST(T);
  
  // แทรกข้อมูล
  var Data := [5, 3, 7, 1, 4, 6, 8, 2, 9];
  WriteLn('แทรก: 5 3 7 1 4 6 8 2 9');
  for var v in Data do Insert(T, v);
  
  WriteLn('Size: ', T.Size);
  WriteLn('Height: ', Height(T.Root));
  WriteLn;
  
  Write('In-order (sorted): ');
  InOrder(T.Root);
  WriteLn;
  
  Write('Pre-order: ');
  PreOrder(T.Root);
  WriteLn;
  
  Write('Post-order: ');
  PostOrder(T.Root);
  WriteLn;
  
  WriteLn;
  WriteLn('ค้นหา 4: ', Search(T, 4));
  WriteLn('ค้นหา 10: ', Search(T, 10));
  
  FreeBST(T);
  WriteLn;
  WriteLn('Tree freed. Size = ', T.Size);
  
  ReadLn;
end.
```

---

## 14.15 แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนโปรแกรมสาธิต pointer ชี้ไปที่ตัวแปรชนิดต่างๆ
แสดง address และค่าของตัวแปรทั้งก่อนและหลังแก้ไขผ่าน pointer

### แบบฝึกหัดที่ 2
เขียน function `Swap` ที่ใช้ pointer สลับค่าของตัวแปร 2 ตัว
ทดสอบกับ Integer, Real, String

### แบบฝึกหัดที่ 3
สร้าง dynamic array ด้วย `GetMem/FreeMem`
เขียน function sort (quick sort) สำหรับ array นี้

### แบบฝึกหัดที่ 4
สร้าง Linked List ของ String
เพิ่มฟังก์ชัน:
- Insert at position
- Delete at position
- Reverse the list
- Find nth element

### แบบฝึกหัดที่ 5
สร้าง Stack ที่ใช้ pointer เก็บ String
เขียนโปรแกรมตรวจสอบ HTML tags ว่าสมดุลหรือไม่
เช่น `<html><body></body></html>` = balanced

### แบบฝึกหัดที่ 6
สร้าง Queue ที่ใช้ pointer
จำลองระบบ printer queue:
- เพิ่ม print job
- ประมวลผล job
- ยกเลิก job
- แสดงสถานะ queue

### แบบฝึกหัดที่ 7
สร้าง Circular Linked List
เพิ่ม Insert, Delete, Print, Search

### แบบฝึกหัดที่ 8
สร้าง Doubly Linked List พร้อม:
- Insert sorted
- Delete by value
- Find and update
- Print forward/backward

### แบบฝึกหัดที่ 9
สร้าง Binary Search Tree ที่เก็บ String (ชื่อนักเรียน)
เพิ่มฟังก์ชัน Delete node (3 cases: leaf, one child, two children)

### แบบฝึกหัดที่ 10
เขียน memory pool allocator อย่างง่าย:
- Pre-allocate block ขนาดใหญ่
- จัดสรรจาก pool
- คืน memory กลับสู่ pool

### แบบฝึกหัดที่ 11
สร้าง Hash Table ด้วย chaining (linked list)
เพิ่ม Put, Get, Delete, Contains, Print

### แบบฝึกหัดที่ 12
สร้าง Graph representation ด้วย adjacency list (linked list)
เพิ่ม AddVertex, AddEdge, DFS, BFS

### แบบฝึกหัดที่ 13
เขียน function ที่รับ pointer ไปยัง array ของ integers
เรียงด้วย Merge Sort (in-place)

### แบบฝึกหัดที่ 14
สร้าง LRU Cache ด้วย Doubly Linked List + Hash Map
Put(key, value), Get(key), evict LRU item

### แบบฝึกหัดที่ 15
เขียน mini garbage collector อย่างง่าย:
- Track allocated pointers
- Mark reachable objects
- Sweep (free) unreachable ones

---

## สรุป

| คำสั่ง | ความหมาย |
|--------|---------|
| `^TypeName` | ประกาศ pointer type |
| `@Variable` | หา address ของตัวแปร |
| `P^` | Dereference (เข้าถึงค่าที่ชี้ไป) |
| `New(P)` | จัดสรร memory ใน heap |
| `Dispose(P)` | คืน memory |
| `GetMem(P, Size)` | จัดสรร memory แบบกำหนดขนาด |
| `FreeMem(P, Size)` | คืน memory แบบกำหนดขนาด |
| `nil` | pointer ไม่ชี้ที่ไหน |
| `Inc(P)` | เลื่อน pointer ไปข้างหน้า |
| `Dec(P)` | เลื่อน pointer ถอยหลัง |

**กฎทอง:**
1. `New` ทุกครั้งต้องมี `Dispose`
2. ตรวจสอบ `nil` ก่อน dereference เสมอ
3. หลัง `Dispose` ให้ set `P := nil`
4. ใช้ `try...finally` ป้องกัน memory leak
5. อย่าใช้ dangling pointers (pointer ที่ชี้ไปยัง memory ที่ถูก free แล้ว)
