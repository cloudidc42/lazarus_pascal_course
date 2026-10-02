# Part 27 - Generics (เจนเนอริก)

## บทนำ

Generics คือการเขียนโค้ดที่ทำงานกับข้อมูลหลายชนิดได้โดยไม่ต้องเขียนโค้ดซ้ำ เช่น Stack ที่เก็บ Integer, String, หรือ Object ก็ใช้โค้ดเดียวกัน ใน Lazarus/FPC เราสามารถสร้าง Generic ได้ตั้งแต่เวอร์ชัน 2.2

### ทำไมต้องใช้ Generics?

```pascal
// ปัญหาโดยไม่ใช้ Generics: ต้องเขียนซ้ำสำหรับแต่ละชนิดข้อมูล
type
  TIntegerStack = class ... end;   // Stack สำหรับ Integer
  TStringStack = class ... end;    // Stack สำหรับ String
  TDoubleStack = class ... end;    // Stack สำหรับ Double
  // ... ต้องเขียนใหม่ทุกครั้ง!

// วิธีแก้: ใช้ Generics
type
  TStack<T> = class ... end;       // Stack สำหรับทุกชนิด!
  
var
  IntStack: TStack<Integer>;
  StrStack: TStack<String>;
  DblStack: TStack<Double>;
```

---

## 27.1 Generic Procedures and Functions

```pascal
program GenericProcedures;
{$mode objfpc}{$H+}

uses
  SysUtils;

// Generic procedure สำหรับ swap ค่าสองค่า
generic procedure Swap<T>(var A, B: T);
var
  Temp: T;
begin
  Temp := A;
  A := B;
  B := Temp;
end;

// Generic function สำหรับหาค่าสูงสุด
generic function Max<T>(A, B: T): T;
begin
  if A > B then
    Result := A
  else
    Result := B;
end;

// Generic function สำหรับหาค่าต่ำสุด
generic function Min<T>(A, B: T): T;
begin
  if A < B then
    Result := A
  else
    Result := B;
end;

// Generic procedure แสดงค่าใน array
generic procedure PrintArray<T>(const Arr: array of T);
var
  I: Integer;
begin
  Write('[');
  for I := 0 to High(Arr) do
  begin
    if I > 0 then Write(', ');
    Write(Arr[I]);
  end;
  WriteLn(']');
end;

// Specialize generic procedures
procedure SwapInt(var A, B: Integer); specialize Swap<Integer>;
procedure SwapStr(var A, B: String); specialize Swap<String>;
function MaxInt(A, B: Integer): Integer; specialize Max<Integer>;
function MaxStr(A, B: String): String; specialize Max<String>;
function MaxDouble(A, B: Double): Double; specialize Max<Double>;
procedure PrintIntArray(const Arr: array of Integer); specialize PrintArray<Integer>;
procedure PrintStrArray(const Arr: array of String); specialize PrintArray<String>;

var
  X, Y: Integer;
  S1, S2: String;
  D1, D2: Double;
  IntArr: array[0..4] of Integer = (5, 3, 8, 1, 9);
  StrArr: array[0..3] of String = ('วิชัย', 'สมชาย', 'อนันต์', 'บุญมี');

begin
  WriteLn('=== Generic Procedures และ Functions ===');
  
  // Swap Integer
  WriteLn(#10'1. Swap Integer:');
  X := 10; Y := 20;
  WriteLn('ก่อน Swap: X = ', X, ', Y = ', Y);
  SwapInt(X, Y);
  WriteLn('หลัง Swap: X = ', X, ', Y = ', Y);
  
  // Swap String
  WriteLn(#10'2. Swap String:');
  S1 := 'สวัสดี'; S2 := 'ลาก่อน';
  WriteLn('ก่อน Swap: S1 = ', S1, ', S2 = ', S2);
  SwapStr(S1, S2);
  WriteLn('หลัง Swap: S1 = ', S1, ', S2 = ', S2);
  
  // Max
  WriteLn(#10'3. Max:');
  WriteLn('Max(10, 20) = ', MaxInt(10, 20));
  WriteLn('Max("Apple", "Banana") = ', MaxStr('Apple', 'Banana'));
  WriteLn('Max(3.14, 2.71) = ', MaxDouble(3.14, 2.71):0:2);
  
  // Print Arrays
  WriteLn(#10'4. Print Arrays:');
  Write('Integer Array: ');
  PrintIntArray(IntArr);
  Write('String Array: ');
  PrintStrArray(StrArr);
  
  ReadLn;
end.
```

---

## 27.2 Generic Classes

```pascal
program GenericClasses;
{$mode objfpc}{$H+}

uses
  SysUtils;

type
  // Generic Stack class
  generic TStack<T> = class
  private
    FItems: array of T;
    FCount: Integer;
    FCapacity: Integer;
    procedure Grow;
  public
    constructor Create(ACapacity: Integer = 16);
    destructor Destroy; override;
    procedure Push(const Item: T);
    function Pop: T;
    function Peek: T;
    function IsEmpty: Boolean;
    property Count: Integer read FCount;
    procedure Print; // ต้องระวัง: ใช้ Write ได้เฉพาะบางชนิด
  end;

  // Specialize สำหรับแต่ละชนิด
  TIntStack = specialize TStack<Integer>;
  TStrStack = specialize TStack<String>;
  TDoubleStack = specialize TStack<Double>;

constructor TStack.Create(ACapacity: Integer);
begin
  inherited Create;
  FCapacity := ACapacity;
  FCount := 0;
  SetLength(FItems, FCapacity);
end;

destructor TStack.Destroy;
begin
  SetLength(FItems, 0);
  inherited Destroy;
end;

procedure TStack.Grow;
begin
  FCapacity := FCapacity * 2;
  SetLength(FItems, FCapacity);
end;

procedure TStack.Push(const Item: T);
begin
  if FCount >= FCapacity then
    Grow;
  FItems[FCount] := Item;
  Inc(FCount);
end;

function TStack.Pop: T;
begin
  if FCount = 0 then
    raise Exception.Create('Stack ว่างเปล่า - ไม่สามารถ Pop ได้');
  Dec(FCount);
  Result := FItems[FCount];
end;

function TStack.Peek: T;
begin
  if FCount = 0 then
    raise Exception.Create('Stack ว่างเปล่า');
  Result := FItems[FCount - 1];
end;

function TStack.IsEmpty: Boolean;
begin
  Result := FCount = 0;
end;

procedure TStack.Print;
var
  I: Integer;
begin
  Write('Stack [');
  for I := FCount - 1 downto 0 do
  begin
    if I < FCount - 1 then Write(', ');
    Write(FItems[I]);
  end;
  WriteLn('] (top -> bottom)');
end;

begin
  WriteLn('=== Generic Stack Class ===');
  
  // Integer Stack
  WriteLn(#10'1. Integer Stack:');
  var IntStack := TIntStack.Create;
  try
    IntStack.Push(10);
    IntStack.Push(20);
    IntStack.Push(30);
    IntStack.Print;
    WriteLn('Peek: ', IntStack.Peek);
    WriteLn('Pop: ', IntStack.Pop);
    WriteLn('หลัง Pop: ');
    IntStack.Print;
    WriteLn('Count: ', IntStack.Count);
  finally
    IntStack.Free;
  end;
  
  // String Stack
  WriteLn(#10'2. String Stack:');
  var StrStack := TStrStack.Create;
  try
    StrStack.Push('ก');
    StrStack.Push('ข');
    StrStack.Push('ค');
    StrStack.Print;
    WriteLn('Pop: ', StrStack.Pop);
    WriteLn('Pop: ', StrStack.Pop);
    StrStack.Print;
  finally
    StrStack.Free;
  end;
  
  // ทดสอบ Empty Stack
  WriteLn(#10'3. ทดสอบ Empty Stack:');
  var EmptyStack := TIntStack.Create;
  try
    try
      EmptyStack.Pop;
    except
      on E: Exception do
        WriteLn('จับได้: ', E.Message);
    end;
  finally
    EmptyStack.Free;
  end;
  
  ReadLn;
end.
```

---

## 27.3 Generic Queue

```pascal
program GenericQueue;
{$mode objfpc}{$H+}

uses
  SysUtils;

type
  generic TQueue<T> = class
  private
    FItems: array of T;
    FHead: Integer;
    FTail: Integer;
    FCount: Integer;
    FCapacity: Integer;
    procedure Grow;
  public
    constructor Create(ACapacity: Integer = 16);
    procedure Enqueue(const Item: T);
    function Dequeue: T;
    function Peek: T;
    function IsEmpty: Boolean;
    property Count: Integer read FCount;
    procedure Print;
  end;

  TIntQueue = specialize TQueue<Integer>;
  TStrQueue = specialize TQueue<String>;

constructor TQueue.Create(ACapacity: Integer);
begin
  inherited Create;
  FCapacity := ACapacity;
  FHead := 0;
  FTail := 0;
  FCount := 0;
  SetLength(FItems, FCapacity);
end;

procedure TQueue.Grow;
var
  NewItems: array of T;
  I: Integer;
begin
  SetLength(NewItems, FCapacity * 2);
  for I := 0 to FCount - 1 do
    NewItems[I] := FItems[(FHead + I) mod FCapacity];
  FItems := NewItems;
  FHead := 0;
  FTail := FCount;
  FCapacity := FCapacity * 2;
end;

procedure TQueue.Enqueue(const Item: T);
begin
  if FCount >= FCapacity then
    Grow;
  FItems[FTail] := Item;
  FTail := (FTail + 1) mod FCapacity;
  Inc(FCount);
end;

function TQueue.Dequeue: T;
begin
  if FCount = 0 then
    raise Exception.Create('Queue ว่างเปล่า');
  Result := FItems[FHead];
  FHead := (FHead + 1) mod FCapacity;
  Dec(FCount);
end;

function TQueue.Peek: T;
begin
  if FCount = 0 then
    raise Exception.Create('Queue ว่างเปล่า');
  Result := FItems[FHead];
end;

function TQueue.IsEmpty: Boolean;
begin
  Result := FCount = 0;
end;

procedure TQueue.Print;
var
  I: Integer;
begin
  Write('Queue (front -> back): [');
  for I := 0 to FCount - 1 do
  begin
    if I > 0 then Write(', ');
    Write(FItems[(FHead + I) mod FCapacity]);
  end;
  WriteLn(']');
end;

begin
  WriteLn('=== Generic Queue ===');
  
  // Integer Queue
  WriteLn(#10'Integer Queue:');
  var Q := TIntQueue.Create;
  try
    Q.Enqueue(1);
    Q.Enqueue(2);
    Q.Enqueue(3);
    Q.Enqueue(4);
    Q.Enqueue(5);
    Q.Print;
    
    WriteLn('Dequeue: ', Q.Dequeue);
    WriteLn('Dequeue: ', Q.Dequeue);
    Q.Print;
    
    WriteLn('Count: ', Q.Count);
  finally
    Q.Free;
  end;
  
  // String Queue (จำลองระบบคิว)
  WriteLn(#10'String Queue (ระบบคิวลูกค้า):');
  var CustomerQueue := TStrQueue.Create;
  try
    CustomerQueue.Enqueue('คุณสมชาย');
    CustomerQueue.Enqueue('คุณสมหญิง');
    CustomerQueue.Enqueue('คุณวิชัย');
    CustomerQueue.Print;
    
    WriteLn('รับลูกค้า: ', CustomerQueue.Dequeue);
    WriteLn('ลูกค้าถัดไป: ', CustomerQueue.Peek);
    CustomerQueue.Print;
  finally
    CustomerQueue.Free;
  end;
  
  ReadLn;
end.
```

---

## 27.4 Generic Pair

```pascal
program GenericPair;
{$mode objfpc}{$H+}

uses
  SysUtils;

type
  // Generic Pair
  generic TPair<TKey, TValue> = class
  private
    FKey: TKey;
    FValue: TValue;
  public
    constructor Create(const AKey: TKey; const AValue: TValue);
    property Key: TKey read FKey write FKey;
    property Value: TValue read FValue write FValue;
    function ToString: string; override;
  end;
  
  // Specialize
  TIntStrPair = specialize TPair<Integer, String>;
  TStrStrPair = specialize TPair<String, String>;
  TStrIntPair = specialize TPair<String, Integer>;
  
  // Generic Tuple (3 ค่า)
  generic TTuple3<T1, T2, T3> = record
    First: T1;
    Second: T2;
    Third: T3;
  end;
  
  TPersonRecord = specialize TTuple3<String, Integer, String>;
  // Name, Age, Email

constructor TPair.Create(const AKey: TKey; const AValue: TValue);
begin
  inherited Create;
  FKey := AKey;
  FValue := AValue;
end;

function TPair.ToString: string;
begin
  Result := Format('(%s, %s)', [string(FKey), string(FValue)]);
end;

begin
  WriteLn('=== Generic Pair ===');
  
  // Integer-String Pair
  var P1 := TIntStrPair.Create(1, 'หนึ่ง');
  try
    WriteLn('Pair 1: Key = ', P1.Key, ', Value = ', P1.Value);
  finally
    P1.Free;
  end;
  
  // String-String Pair
  var P2 := TStrStrPair.Create('th', 'ภาษาไทย');
  try
    WriteLn('Pair 2: Key = ', P2.Key, ', Value = ', P2.Value);
  finally
    P2.Free;
  end;
  
  // String-Integer Pair
  var P3 := TStrIntPair.Create('คะแนน', 95);
  try
    WriteLn('Pair 3: Key = ', P3.Key, ', Value = ', P3.Value);
  finally
    P3.Free;
  end;
  
  // Tuple
  WriteLn(#10'=== Generic Tuple ===');
  var Person: TPersonRecord;
  Person.First := 'สมชาย';
  Person.Second := 25;
  Person.Third := 'somchai@example.com';
  WriteLn(Format('ชื่อ: %s, อายุ: %d, Email: %s', 
                 [Person.First, Person.Second, Person.Third]));
  
  ReadLn;
end.
```

---

## 27.5 Generic Collections

```pascal
program GenericCollections;
{$mode objfpc}{$H+}

uses
  SysUtils, Generics.Collections;

type
  TStudent = record
    Name: string;
    Score: Integer;
    Grade: Char;
  end;

procedure DemoTList;
var
  List: specialize TList<Integer>;
  I: Integer;
begin
  WriteLn('=== TList<Integer> ===');
  List := specialize TList<Integer>.Create;
  try
    // Add items
    for I := 1 to 10 do
      List.Add(I * I);  // เพิ่มเลขยกกำลังสอง
    
    // Print
    Write('รายการ: ');
    for I := 0 to List.Count - 1 do
    begin
      if I > 0 then Write(', ');
      Write(List[I]);
    end;
    WriteLn;
    
    // Find
    var Idx := List.IndexOf(25);
    if Idx >= 0 then
      WriteLn('พบ 25 ที่ index: ', Idx)
    else
      WriteLn('ไม่พบ 25');
    
    // Delete
    List.Delete(0);
    WriteLn('หลังลบ index 0, Count = ', List.Count);
    
    // Sort (ascending)
    List.Sort;
    Write('หลัง Sort: ');
    for I := 0 to List.Count - 1 do
    begin
      if I > 0 then Write(', ');
      Write(List[I]);
    end;
    WriteLn;
  finally
    List.Free;
  end;
end;

procedure DemoTDictionary;
var
  Dict: specialize TDictionary<String, Integer>;
  Key: String;
  Value: Integer;
  Pair: specialize TPair<String, Integer>;
begin
  WriteLn(#10'=== TDictionary<String, Integer> ===');
  Dict := specialize TDictionary<String, Integer>.Create;
  try
    // Add
    Dict.Add('หนึ่ง', 1);
    Dict.Add('สอง', 2);
    Dict.Add('สาม', 3);
    Dict.Add('สี่', 4);
    Dict.Add('ห้า', 5);
    
    WriteLn('Count: ', Dict.Count);
    
    // Access
    WriteLn('Dict["สาม"] = ', Dict['สาม']);
    
    // ContainsKey
    if Dict.ContainsKey('สอง') then
      WriteLn('มี key "สอง"')
    else
      WriteLn('ไม่มี key "สอง"');
    
    // TryGetValue
    if Dict.TryGetValue('ห้า', Value) then
      WriteLn('TryGetValue("ห้า") = ', Value)
    else
      WriteLn('ไม่พบ "ห้า"');
    
    // วน loop
    WriteLn('รายการทั้งหมด:');
    for Pair in Dict do
      WriteLn('  ', Pair.Key, ' = ', Pair.Value);
    
    // Remove
    Dict.Remove('สาม');
    WriteLn('หลังลบ "สาม", Count = ', Dict.Count);
    
    // AddOrSetValue (update if exists)
    Dict.AddOrSetValue('หนึ่ง', 100);
    WriteLn('หลัง update "หนึ่ง", Dict["หนึ่ง"] = ', Dict['หนึ่ง']);
    
  finally
    Dict.Free;
  end;
end;

procedure DemoTObjectList;
type
  TStudentObj = class
  public
    Name: String;
    Score: Integer;
    constructor Create(const AName: String; AScore: Integer);
    destructor Destroy; override;
  end;

var
  StudentList: specialize TObjectList<TStudentObj>;
  S: TStudentObj;

  constructor TStudentObj.Create(const AName: String; AScore: Integer);
  begin
    inherited Create;
    Name := AName;
    Score := AScore;
  end;
  
  destructor TStudentObj.Destroy;
  begin
    WriteLn('  ทำลาย: ', Name);
    inherited Destroy;
  end;

begin
  WriteLn(#10'=== TObjectList<TStudentObj> ===');
  StudentList := specialize TObjectList<TStudentObj>.Create(True);  // OwnsObjects = True
  try
    StudentList.Add(TStudentObj.Create('สมชาย', 85));
    StudentList.Add(TStudentObj.Create('สมหญิง', 92));
    StudentList.Add(TStudentObj.Create('วิชัย', 78));
    
    WriteLn('นักเรียนทั้งหมด:');
    for S in StudentList do
      WriteLn('  ', S.Name, ': ', S.Score, ' คะแนน');
    
    WriteLn('Count: ', StudentList.Count);
    
    // Sort by score (ต้องใช้ comparer)
    WriteLn('ลบ index 1...');
    StudentList.Delete(1);
    WriteLn('Count หลังลบ: ', StudentList.Count);
    // Objects จะถูก free อัตโนมัติเมื่อ OwnsObjects = True
  finally
    StudentList.Free;  // Free objects อัตโนมัติ
  end;
end;

procedure DemoTQueue;
var
  Q: specialize TQueue<String>;
  Item: String;
begin
  WriteLn(#10'=== TQueue<String> ===');
  Q := specialize TQueue<String>.Create;
  try
    Q.Enqueue('งานที่ 1');
    Q.Enqueue('งานที่ 2');
    Q.Enqueue('งานที่ 3');
    
    WriteLn('Count: ', Q.Count);
    WriteLn('Peek: ', Q.Peek);
    
    while Q.Count > 0 do
    begin
      Item := Q.Dequeue;
      WriteLn('ประมวลผล: ', Item);
    end;
  finally
    Q.Free;
  end;
end;

procedure DemoTStack;
var
  S: specialize TStack<Integer>;
begin
  WriteLn(#10'=== TStack<Integer> ===');
  S := specialize TStack<Integer>.Create;
  try
    S.Push(10);
    S.Push(20);
    S.Push(30);
    
    WriteLn('Count: ', S.Count);
    WriteLn('Peek: ', S.Peek);
    
    while S.Count > 0 do
      WriteLn('Pop: ', S.Pop);
  finally
    S.Free;
  end;
end;

begin
  DemoTList;
  DemoTDictionary;
  DemoTObjectList;
  DemoTQueue;
  DemoTStack;
  ReadLn;
end.
```

---

## 27.6 Generic Repository Pattern

```pascal
program GenericRepository;
{$mode objfpc}{$H+}

uses
  SysUtils, Generics.Collections;

type
  // Interface สำหรับ Entity ที่มี ID
  IEntity = interface
    function GetID: Integer;
    property ID: Integer read GetID;
  end;
  
  // Generic Repository Interface
  generic IRepository<T: IEntity> = interface
    procedure Add(const Entity: T);
    procedure Update(const Entity: T);
    procedure Delete(ID: Integer);
    function FindByID(ID: Integer): T;
    function FindAll: specialize TList<T>;
    function Count: Integer;
  end;
  
  // Entity classes
  TProduct = class(TInterfacedObject, IEntity)
  private
    FID: Integer;
    FName: String;
    FPrice: Double;
    FStock: Integer;
    function GetID: Integer;
  public
    constructor Create(AID: Integer; const AName: String; APrice: Double; AStock: Integer);
    property ID: Integer read GetID;
    property Name: String read FName write FName;
    property Price: Double read FPrice write FPrice;
    property Stock: Integer read FStock write FStock;
    function ToString: String; override;
  end;
  
  // Generic In-Memory Repository
  generic TInMemoryRepository<T: IEntity> = class(TInterfacedObject,
                                                   specialize IRepository<T>)
  private
    FItems: specialize TList<T>;
  public
    constructor Create;
    destructor Destroy; override;
    procedure Add(const Entity: T);
    procedure Update(const Entity: T);
    procedure Delete(ID: Integer);
    function FindByID(ID: Integer): T;
    function FindAll: specialize TList<T>;
    function Count: Integer;
  end;
  
  TProductRepository = specialize TInMemoryRepository<TProduct>;
  IProductRepository = specialize IRepository<TProduct>;

function TProduct.GetID: Integer;
begin
  Result := FID;
end;

constructor TProduct.Create(AID: Integer; const AName: String; APrice: Double; AStock: Integer);
begin
  inherited Create;
  FID := AID;
  FName := AName;
  FPrice := APrice;
  FStock := AStock;
end;

function TProduct.ToString: String;
begin
  Result := Format('Product[%d] %s - ฿%.2f (คงเหลือ: %d)', 
                   [FID, FName, FPrice, FStock]);
end;

constructor TInMemoryRepository.Create;
begin
  inherited Create;
  FItems := specialize TList<T>.Create;
end;

destructor TInMemoryRepository.Destroy;
begin
  FItems.Free;
  inherited Destroy;
end;

procedure TInMemoryRepository.Add(const Entity: T);
begin
  FItems.Add(Entity);
end;

procedure TInMemoryRepository.Update(const Entity: T);
var
  I: Integer;
begin
  for I := 0 to FItems.Count - 1 do
    if FItems[I].ID = Entity.ID then
    begin
      FItems[I] := Entity;
      Exit;
    end;
  raise Exception.CreateFmt('ไม่พบ Entity ID: %d', [Entity.ID]);
end;

procedure TInMemoryRepository.Delete(ID: Integer);
var
  I: Integer;
begin
  for I := 0 to FItems.Count - 1 do
    if FItems[I].ID = ID then
    begin
      FItems.Delete(I);
      Exit;
    end;
  raise Exception.CreateFmt('ไม่พบ Entity ID: %d', [ID]);
end;

function TInMemoryRepository.FindByID(ID: Integer): T;
var
  I: Integer;
begin
  for I := 0 to FItems.Count - 1 do
    if FItems[I].ID = ID then
    begin
      Result := FItems[I];
      Exit;
    end;
  Result := Default(T);
end;

function TInMemoryRepository.FindAll: specialize TList<T>;
begin
  Result := FItems;
end;

function TInMemoryRepository.Count: Integer;
begin
  Result := FItems.Count;
end;

begin
  WriteLn('=== Generic Repository Pattern ===');
  
  var Repo := TProductRepository.Create;
  try
    // Add products
    Repo.Add(TProduct.Create(1, 'แป้งสาลี', 35.00, 100));
    Repo.Add(TProduct.Create(2, 'น้ำตาล', 25.00, 200));
    Repo.Add(TProduct.Create(3, 'เกลือ', 10.00, 150));
    Repo.Add(TProduct.Create(4, 'น้ำมันพืช', 60.00, 80));
    
    WriteLn('สินค้าทั้งหมด (', Repo.Count, ' รายการ):');
    var All := Repo.FindAll;
    for var P in All do
      WriteLn('  ', P.ToString);
    
    // Find by ID
    WriteLn(#10'ค้นหา ID 2:');
    var Found := Repo.FindByID(2);
    if Found <> nil then
      WriteLn('  พบ: ', Found.ToString)
    else
      WriteLn('  ไม่พบ');
    
    // Update
    WriteLn(#10'อัปเดตสินค้า ID 1:');
    var Updated := TProduct.Create(1, 'แป้งสาลีเกรด A', 45.00, 90);
    Repo.Update(Updated);
    WriteLn('  ', Repo.FindByID(1).ToString);
    
    // Delete
    WriteLn(#10'ลบสินค้า ID 3:');
    Repo.Delete(3);
    WriteLn('Count หลังลบ: ', Repo.Count);
    
    // Print all again
    WriteLn(#10'สินค้าที่เหลือ:');
    for var P in Repo.FindAll do
      WriteLn('  ', P.ToString);
    
  finally
    Repo.Free;
  end;
  
  ReadLn;
end.
```

---

## 27.7 Type Constraints

```pascal
program TypeConstraints;
{$mode objfpc}{$H+}

uses
  SysUtils;

// ตัวอย่างการใช้ type constraints ใน FPC

// constraint: T ต้องเป็น class
generic function CloneObject<T: class>(Obj: T): T;
begin
  // ใน FPC ยังไม่รองรับ constraint อย่างสมบูรณ์
  // แต่สามารถทำได้โดยใช้ RTTI
  Result := Obj;  // simplified example
end;

// Generic Comparer
generic function Compare<T>(const A, B: T): Integer;
begin
  if A < B then
    Result := -1
  else if A > B then
    Result := 1
  else
    Result := 0;
end;

// Generic MinMax
generic procedure MinMax<T>(const Values: array of T; out MinVal, MaxVal: T);
var
  I: Integer;
begin
  if Length(Values) = 0 then
    raise Exception.Create('Array ว่างเปล่า');
  MinVal := Values[0];
  MaxVal := Values[0];
  for I := 1 to High(Values) do
  begin
    if Values[I] < MinVal then MinVal := Values[I];
    if Values[I] > MaxVal then MaxVal := Values[I];
  end;
end;

// Specialized versions
function CompareInt(const A, B: Integer): Integer; specialize Compare<Integer>;
function CompareStr(const A, B: String): Integer; specialize Compare<String>;
procedure MinMaxInt(const Values: array of Integer; out MinVal, MaxVal: Integer); 
  specialize MinMax<Integer>;
procedure MinMaxDouble(const Values: array of Double; out MinVal, MaxVal: Double); 
  specialize MinMax<Double>;

var
  IntVals: array[0..5] of Integer = (5, 3, 8, 1, 9, 4);
  DblVals: array[0..4] of Double = (1.1, 5.5, 2.2, 4.4, 3.3);
  MinI, MaxI: Integer;
  MinD, MaxD: Double;

begin
  WriteLn('=== Type Constraints และ Generic Functions ===');
  
  WriteLn(#10'1. Compare:');
  WriteLn('Compare(1, 2) = ', CompareInt(1, 2));
  WriteLn('Compare(2, 2) = ', CompareInt(2, 2));
  WriteLn('Compare(3, 2) = ', CompareInt(3, 2));
  WriteLn('Compare("Apple", "Banana") = ', CompareStr('Apple', 'Banana'));
  
  WriteLn(#10'2. MinMax Integer:');
  MinMaxInt(IntVals, MinI, MaxI);
  WriteLn('Min = ', MinI, ', Max = ', MaxI);
  
  WriteLn(#10'3. MinMax Double:');
  MinMaxDouble(DblVals, MinD, MaxD);
  WriteLn(Format('Min = %.1f, Max = %.1f', [MinD, MaxD]));
  
  ReadLn;
end.
```

---

## 27.8 Generic Sorting

```pascal
program GenericSorting;
{$mode objfpc}{$H+}

uses
  SysUtils;

// Generic Bubble Sort
generic procedure BubbleSort<T>(var Arr: array of T);
var
  I, J: Integer;
  Temp: T;
  Swapped: Boolean;
begin
  for I := High(Arr) downto 1 do
  begin
    Swapped := False;
    for J := 0 to I - 1 do
    begin
      if Arr[J] > Arr[J + 1] then
      begin
        Temp := Arr[J];
        Arr[J] := Arr[J + 1];
        Arr[J + 1] := Temp;
        Swapped := True;
      end;
    end;
    if not Swapped then Break;
  end;
end;

// Generic Quick Sort
generic procedure QuickSort<T>(var Arr: array of T; Low, High: Integer);
var
  I, J: Integer;
  Pivot, Temp: T;
begin
  I := Low;
  J := High;
  Pivot := Arr[(Low + High) div 2];
  
  while I <= J do
  begin
    while Arr[I] < Pivot do Inc(I);
    while Arr[J] > Pivot do Dec(J);
    if I <= J then
    begin
      Temp := Arr[I];
      Arr[I] := Arr[J];
      Arr[J] := Temp;
      Inc(I);
      Dec(J);
    end;
  end;
  
  if Low < J then
    specialize QuickSort<T>(Arr, Low, J);
  if I < High then
    specialize QuickSort<T>(Arr, I, High);
end;

// Specialized sorting
procedure BubbleSortInt(var Arr: array of Integer); specialize BubbleSort<Integer>;
procedure BubbleSortStr(var Arr: array of String); specialize BubbleSort<String>;
procedure QuickSortInt(var Arr: array of Integer; Low, High: Integer); 
  specialize QuickSort<Integer>;

// Generic Binary Search
generic function BinarySearch<T>(const Arr: array of T; const Value: T): Integer;
var
  Low, High, Mid: Integer;
begin
  Low := 0;
  High := Length(Arr) - 1;
  Result := -1;
  
  while Low <= High do
  begin
    Mid := (Low + High) div 2;
    if Arr[Mid] = Value then
    begin
      Result := Mid;
      Exit;
    end
    else if Arr[Mid] < Value then
      Low := Mid + 1
    else
      High := Mid - 1;
  end;
end;

function BinarySearchInt(const Arr: array of Integer; const Value: Integer): Integer;
  specialize BinarySearch<Integer>;

procedure PrintIntArray(const Arr: array of Integer; const Label: String);
var
  I: Integer;
begin
  Write(Label, ': [');
  for I := 0 to High(Arr) do
  begin
    if I > 0 then Write(', ');
    Write(Arr[I]);
  end;
  WriteLn(']');
end;

var
  IntArr: array[0..9] of Integer = (64, 34, 25, 12, 22, 11, 90, 45, 67, 3);
  StrArr: array[0..4] of String = ('กล้วย', 'แอปเปิ้ล', 'ส้ม', 'มะม่วง', 'สับปะรด');
  I: Integer;

begin
  WriteLn('=== Generic Sorting ===');
  
  // Bubble Sort Integer
  WriteLn(#10'1. Bubble Sort (Integer):');
  PrintIntArray(IntArr, 'ก่อน Sort');
  BubbleSortInt(IntArr);
  PrintIntArray(IntArr, 'หลัง Sort');
  
  // Bubble Sort String
  WriteLn(#10'2. Bubble Sort (String):');
  Write('ก่อน Sort: [');
  for I := 0 to High(StrArr) do
  begin
    if I > 0 then Write(', ');
    Write(StrArr[I]);
  end;
  WriteLn(']');
  BubbleSortStr(StrArr);
  Write('หลัง Sort: [');
  for I := 0 to High(StrArr) do
  begin
    if I > 0 then Write(', ');
    Write(StrArr[I]);
  end;
  WriteLn(']');
  
  // Quick Sort
  WriteLn(#10'3. Quick Sort (Integer):');
  var QArr: array[0..7] of Integer = (8, 3, 5, 1, 9, 2, 7, 4);
  PrintIntArray(QArr, 'ก่อน Quick Sort');
  QuickSortInt(QArr, 0, High(QArr));
  PrintIntArray(QArr, 'หลัง Quick Sort');
  
  // Binary Search
  WriteLn(#10'4. Binary Search:');
  var SortedArr: array[0..9] of Integer = (1, 3, 5, 7, 9, 11, 13, 15, 17, 19);
  PrintIntArray(SortedArr, 'Array');
  
  var Idx := BinarySearchInt(SortedArr, 7);
  if Idx >= 0 then
    WriteLn('พบ 7 ที่ index: ', Idx)
  else
    WriteLn('ไม่พบ 7');
  
  Idx := BinarySearchInt(SortedArr, 6);
  if Idx >= 0 then
    WriteLn('พบ 6 ที่ index: ', Idx)
  else
    WriteLn('ไม่พบ 6 ใน array');
  
  ReadLn;
end.
```

---

## 27.9 Generic Tree

```pascal
program GenericTree;
{$mode objfpc}{$H+}

uses
  SysUtils;

type
  // Generic Binary Tree Node
  generic TBSTNode<T> = class
  public
    Value: T;
    Left: specialize TBSTNode<T>;
    Right: specialize TBSTNode<T>;
    constructor Create(const AValue: T);
  end;
  
  // Generic Binary Search Tree
  generic TBST<T> = class
  private
    FRoot: specialize TBSTNode<T>;
    FCount: Integer;
    procedure InsertNode(var Node: specialize TBSTNode<T>; const Value: T);
    function SearchNode(Node: specialize TBSTNode<T>; const Value: T): Boolean;
    procedure InOrder(Node: specialize TBSTNode<T>; const Callback: specialize TProc<T>);
    procedure FreeNode(Node: specialize TBSTNode<T>);
  public
    constructor Create;
    destructor Destroy; override;
    procedure Insert(const Value: T);
    function Contains(const Value: T): Boolean;
    procedure TraverseInOrder(const Callback: specialize TProc<T>);
    property Count: Integer read FCount;
  end;
  
  TIntBST = specialize TBST<Integer>;

constructor TBSTNode.Create(const AValue: T);
begin
  inherited Create;
  Value := AValue;
  Left := nil;
  Right := nil;
end;

constructor TBST.Create;
begin
  inherited Create;
  FRoot := nil;
  FCount := 0;
end;

destructor TBST.Destroy;
begin
  FreeNode(FRoot);
  inherited Destroy;
end;

procedure TBST.FreeNode(Node: specialize TBSTNode<T>);
begin
  if Node = nil then Exit;
  FreeNode(Node.Left);
  FreeNode(Node.Right);
  Node.Free;
end;

procedure TBST.InsertNode(var Node: specialize TBSTNode<T>; const Value: T);
begin
  if Node = nil then
  begin
    Node := specialize TBSTNode<T>.Create(Value);
    Inc(FCount);
  end
  else if Value < Node.Value then
    InsertNode(Node.Left, Value)
  else if Value > Node.Value then
    InsertNode(Node.Right, Value);
  // ถ้าเท่ากัน ไม่เพิ่ม (BST ไม่มีค่าซ้ำ)
end;

function TBST.SearchNode(Node: specialize TBSTNode<T>; const Value: T): Boolean;
begin
  if Node = nil then
    Result := False
  else if Value = Node.Value then
    Result := True
  else if Value < Node.Value then
    Result := SearchNode(Node.Left, Value)
  else
    Result := SearchNode(Node.Right, Value);
end;

procedure TBST.InOrder(Node: specialize TBSTNode<T>; const Callback: specialize TProc<T>);
begin
  if Node = nil then Exit;
  InOrder(Node.Left, Callback);
  Callback(Node.Value);
  InOrder(Node.Right, Callback);
end;

procedure TBST.Insert(const Value: T);
begin
  InsertNode(FRoot, Value);
end;

function TBST.Contains(const Value: T): Boolean;
begin
  Result := SearchNode(FRoot, Value);
end;

procedure TBST.TraverseInOrder(const Callback: specialize TProc<T>);
begin
  InOrder(FRoot, Callback);
end;

begin
  WriteLn('=== Generic Binary Search Tree ===');
  
  var Tree := TIntBST.Create;
  try
    // Insert values
    var Values: array of Integer = [5, 3, 7, 1, 4, 6, 8, 2, 9];
    for var V in Values do
    begin
      Tree.Insert(V);
      Write('เพิ่ม ', V, ' -> Count: ', Tree.Count);
      WriteLn;
    end;
    
    // In-order traversal (should be sorted)
    Write(#10'In-order traversal (เรียงลำดับ): ');
    Tree.TraverseInOrder(procedure(const V: Integer)
    begin
      Write(V, ' ');
    end);
    WriteLn;
    
    // Search
    WriteLn(#10'ค้นหา:');
    var SearchVals: array of Integer = [1, 5, 10, 7, 11];
    for var V in SearchVals do
    begin
      if Tree.Contains(V) then
        WriteLn('  ', V, ': พบ')
      else
        WriteLn('  ', V, ': ไม่พบ');
    end;
    
  finally
    Tree.Free;
  end;
  
  ReadLn;
end.
```

---

## 27.10 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Generic Clamp
สร้าง generic function `Clamp<T>` ที่จำกัดค่าให้อยู่ในช่วง [Min, Max]

```pascal
// ตัวอย่าง:
// Clamp<Integer>(150, 0, 100) -> 100
// Clamp<Double>(3.14, 0.0, 3.0) -> 3.0
// Clamp<Integer>(-5, 0, 100) -> 0
generic function Clamp<T>(Value, MinVal, MaxVal: T): T;
```

### แบบฝึกหัดที่ 2: Generic Linked List
สร้าง generic doubly linked list `TLinkedList<T>` ที่มี:
- AddFirst, AddLast
- RemoveFirst, RemoveLast
- Contains
- ToArray
- Count

### แบบฝึกหัดที่ 3: Generic Priority Queue
สร้าง `TPriorityQueue<T>` ที่รับ comparer function และ dequeue ค่าที่มี priority สูงสุดก่อน

### แบบฝึกหัดที่ 4: Generic Result Type
สร้าง `TResult<T>` ที่เป็น variant record สำหรับ error handling:
```pascal
TResult<T> = record
  Success: Boolean;
  Value: T;         // ถ้า Success = True
  ErrorMsg: String; // ถ้า Success = False
end;
```

### แบบฝึกหัดที่ 5: Generic Observer Pattern
สร้าง generic observer pattern:
- `TSubject<T>` - เก็บ value และแจ้ง observers
- `TObserver<T>` - interface สำหรับ callback

### แบบฝึกหัดที่ 6: Generic Lazy Initialization
สร้าง `TLazy<T>` ที่สร้าง object เฉพาะเมื่อถูกเรียกใช้ครั้งแรก:
```pascal
TLazy<T: class> = class
  function GetValue: T;  // สร้างถ้ายังไม่มี
end;
```

### แบบฝึกหัดที่ 7: Generic Cache
สร้าง `TCache<TKey, TValue>` ที่:
- มี TTL (Time To Live)
- ลบรายการที่หมดอายุ
- กำหนดขนาดสูงสุด

### แบบฝึกหัดที่ 8: Generic Tuple
สร้าง generic tuple types:
- `TPair<T1, T2>`
- `TTriplet<T1, T2, T3>`
- `TQuad<T1, T2, T3, T4>`

### แบบฝึกหัดที่ 9: Generic Event System
สร้าง generic event system:
- `TEvent<TArgs>` - event ที่ส่ง argument ชนิด T
- สามารถ subscribe/unsubscribe handlers
- Fire event พร้อม arguments

### แบบฝึกหัดที่ 10: Generic Aggregate
สร้าง generic aggregate functions:
- `Sum<T>` - รวมค่าใน array
- `Average<T>` - หาค่าเฉลี่ย
- `Count<T>` - นับรายการที่ตรงเงื่อนไข
- `Filter<T>` - กรองรายการ

### แบบฝึกหัดที่ 11: Generic State Machine
สร้าง generic state machine `TStateMachine<TState, TEvent>`:
- กำหนด transitions
- Handle events
- Callbacks เมื่อเปลี่ยน state

### แบบฝึกหัดที่ 12: Generic Pipeline
สร้าง generic pipeline `TPipeline<TInput, TOutput>`:
- เพิ่ม processing steps
- Execute pipeline
- Handle errors ในแต่ละขั้นตอน

### แบบฝึกหัดที่ 13: Generic Retry
สร้าง generic retry mechanism สำหรับ operations ที่อาจล้มเหลว

### แบบฝึกหัดที่ 14: Generic Object Pool
สร้าง `TObjectPool<T: class>` ที่:
- สร้าง objects ล่วงหน้า
- Borrow และ Return objects
- Auto-expand pool เมื่อจำเป็น

### แบบฝึกหัดที่ 15: Generic Serializer
สร้าง generic serializer ที่แปลง object เป็น JSON string (simplified)

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Generic Procedures/Functions** - เขียนฟังก์ชันที่ทำงานกับหลายชนิดข้อมูล
2. **Generic Classes** - Stack, Queue, Tree ที่ใช้ได้กับทุกชนิด
3. **Specialize** - สร้าง concrete class จาก generic
4. **Generic Collections** - TList<T>, TDictionary<K,V>, TQueue<T>, TStack<T>
5. **Generic Pair/Tuple** - เก็บข้อมูลหลายค่าพร้อมกัน
6. **Repository Pattern** - design pattern ด้วย generics
7. **Generic Sorting/Searching** - อัลกอริทึมที่ใช้ได้ทุกชนิด
8. **Generic Tree** - Binary Search Tree แบบ generic

### ข้อควรระวัง:
- ใน FPC ต้องใช้ `specialize` เมื่อสร้าง concrete type
- Type constraints ยังจำกัดใน FPC เมื่อเทียบกับ C# หรือ Java
- Generic ทำให้โค้ดยืดหยุ่นแต่อาจยากต่อการ debug
- ระวัง circular dependencies ในนิยาม generic
