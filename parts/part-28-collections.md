# Part 28 - Collections (คอลเลกชัน)

## บทนำ

Collections คือโครงสร้างข้อมูลที่ใช้เก็บกลุ่มของวัตถุหรือค่า Lazarus/FPC มีหลาย collection classes ใน unit `Classes`, `Contnrs`, และ `Generics.Collections` แต่ละชนิดเหมาะกับการใช้งานที่แตกต่างกัน

### เปรียบเทียบ Collections หลัก

| Collection | ใช้เมื่อ | ข้อดี | ข้อเสีย |
|-----------|---------|-------|--------|
| TList | ข้อมูลแบบลำดับ | เข้าถึงด้วย index เร็ว | Insert ตรงกลางช้า |
| TStringList | จัดการ string | มี Sort, Find built-in | เฉพาะ string เท่านั้น |
| TObjectList | object list | จัดการ lifetime | overhead กว่า TList |
| TDictionary | key-value | ค้นหาเร็วมาก | ไม่เรียงลำดับ |
| TStack | LIFO | Push/Pop | เข้าถึง random ไม่ได้ |
| TQueue | FIFO | Enqueue/Dequeue | เข้าถึง random ไม่ได้ |

---

## 28.1 TList

```pascal
program TListDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

// TList เก็บ Pointer ดังนั้นต้องระวัง type safety
// ใช้ร่วมกับ Integer เก็บแบบ Pointer casting

type
  PInteger = ^Integer;

procedure DemoTList;
var
  List: TList;
  P: PInteger;
  I: Integer;
begin
  WriteLn('=== TList Demo ===');
  List := TList.Create;
  try
    // Add integers (ต้อง cast)
    for I := 1 to 5 do
    begin
      New(P);
      P^ := I * 10;
      List.Add(P);
    end;
    
    WriteLn('Count: ', List.Count);
    
    // Access items
    Write('รายการ: ');
    for I := 0 to List.Count - 1 do
      Write(PInteger(List[I])^, ' ');
    WriteLn;
    
    // First and Last
    WriteLn('First: ', PInteger(List.First)^);
    WriteLn('Last: ', PInteger(List.Last)^);
    
    // IndexOf
    New(P);
    P^ := 30;
    var Idx := List.IndexOf(List[2]);
    WriteLn('IndexOf(30 pointer): ', Idx);
    Dispose(P);
    
    // Delete by index
    Dispose(PInteger(List[2]));  // ต้อง free ก่อน
    List.Delete(2);
    WriteLn('หลังลบ index 2, Count: ', List.Count);
    
    // Clear ต้อง free pointers ก่อน
    for I := 0 to List.Count - 1 do
      Dispose(PInteger(List[I]));
    List.Clear;
    WriteLn('หลัง Clear, Count: ', List.Count);
    
  finally
    List.Free;
  end;
end;

// ใช้ TList กับ Record
type
  TPoint = record
    X, Y: Integer;
  end;
  PPoint = ^TPoint;

procedure DemoTListWithRecords;
var
  List: TList;
  P: PPoint;
  I: Integer;
begin
  WriteLn(#10'=== TList กับ Records ===');
  List := TList.Create;
  try
    // Add points
    for I := 0 to 4 do
    begin
      New(P);
      P^.X := I * 10;
      P^.Y := I * 5;
      List.Add(P);
    end;
    
    WriteLn('Points:');
    for I := 0 to List.Count - 1 do
    begin
      P := PPoint(List[I]);
      WriteLn(Format('  (%d, %d)', [P^.X, P^.Y]));
    end;
    
    // Sort by X coordinate
    List.Sort(@(function(A, B: Pointer): Integer
    begin
      Result := PPoint(A)^.X - PPoint(B)^.X;
    end));
    
    // Cleanup
    for I := 0 to List.Count - 1 do
      Dispose(PPoint(List[I]));
  finally
    List.Free;
  end;
end;

begin
  DemoTList;
  DemoTListWithRecords;
  ReadLn;
end.
```

---

## 28.2 TStringList

```pascal
program TStringListDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

procedure DemoBasicOperations;
var
  SL: TStringList;
  I: Integer;
begin
  WriteLn('=== TStringList - การทำงานพื้นฐาน ===');
  SL := TStringList.Create;
  try
    // Add
    SL.Add('กล้วย');
    SL.Add('แอปเปิ้ล');
    SL.Add('ส้ม');
    SL.Add('มะม่วง');
    SL.Add('สับปะรด');
    
    WriteLn('Count: ', SL.Count);
    WriteLn('รายการทั้งหมด:');
    for I := 0 to SL.Count - 1 do
      WriteLn('  [', I, '] ', SL[I]);
    
    // Insert
    SL.Insert(2, 'ฝรั่ง');
    WriteLn(#10'หลัง Insert "ฝรั่ง" ที่ index 2:');
    for I := 0 to SL.Count - 1 do
      WriteLn('  [', I, '] ', SL[I]);
    
    // Delete
    SL.Delete(0);
    WriteLn(#10'หลังลบ index 0 (กล้วย):');
    for I := 0 to SL.Count - 1 do
      WriteLn('  [', I, '] ', SL[I]);
    
    // IndexOf
    var Idx := SL.IndexOf('ส้ม');
    WriteLn(#10'IndexOf("ส้ม") = ', Idx);
    
    // First and Last
    WriteLn('First: ', SL[0]);
    WriteLn('Last: ', SL[SL.Count - 1]);
    
    // Sort
    SL.Sort;
    WriteLn(#10'หลัง Sort:');
    for I := 0 to SL.Count - 1 do
      WriteLn('  ', SL[I]);
    
  finally
    SL.Free;
  end;
end;

procedure DemoTextAndLines;
var
  SL: TStringList;
begin
  WriteLn(#10'=== TStringList - Text และ Lines ===');
  SL := TStringList.Create;
  try
    // ใช้ Text property
    SL.Text := 'บรรทัดที่ 1' + #13#10 + 'บรรทัดที่ 2' + #13#10 + 'บรรทัดที่ 3';
    WriteLn('จาก Text, Count: ', SL.Count);
    WriteLn('Text property:');
    WriteLn(SL.Text);
    
    // SaveToFile และ LoadFromFile
    SL.SaveToFile('/tmp/test_stringlist.txt');
    WriteLn('บันทึกไฟล์แล้ว');
    
    SL.Clear;
    SL.LoadFromFile('/tmp/test_stringlist.txt');
    WriteLn('โหลดไฟล์แล้ว, Count: ', SL.Count);
    for var I := 0 to SL.Count - 1 do
      WriteLn('  ', SL[I]);
    
  finally
    SL.Free;
  end;
end;

procedure DemoNameValuePairs;
var
  SL: TStringList;
begin
  WriteLn(#10'=== TStringList - Name=Value Pairs ===');
  SL := TStringList.Create;
  try
    SL.NameValueSeparator := '=';
    SL.Add('ชื่อ=สมชาย');
    SL.Add('อายุ=25');
    SL.Add('เมือง=กรุงเทพ');
    SL.Add('อีเมล=somchai@example.com');
    
    WriteLn('ชื่อ: ', SL.Values['ชื่อ']);
    WriteLn('อายุ: ', SL.Values['อายุ']);
    WriteLn('เมือง: ', SL.Values['เมือง']);
    
    // แก้ไขค่า
    SL.Values['อายุ'] := '26';
    WriteLn('อายุ (หลังแก้): ', SL.Values['อายุ']);
    
    // Names
    WriteLn('Names:');
    for var I := 0 to SL.Count - 1 do
      WriteLn('  ', SL.Names[I], ' = ', SL.Values[SL.Names[I]]);
    
  finally
    SL.Free;
  end;
end;

procedure DemoFind;
var
  SL: TStringList;
  Idx: Integer;
begin
  WriteLn(#10'=== TStringList - Find (ต้อง Sorted) ===');
  SL := TStringList.Create;
  try
    SL.Sorted := True;  // ต้องเป็น sorted สำหรับ Find
    SL.Add('วิชัย');
    SL.Add('สมชาย');
    SL.Add('อนันต์');
    SL.Add('บุญมี');
    SL.Add('ประยุทธ์');
    
    WriteLn('รายการ (sorted):');
    for var I := 0 to SL.Count - 1 do
      WriteLn('  ', SL[I]);
    
    if SL.Find('อนันต์', Idx) then
      WriteLn('พบ "อนันต์" ที่ index: ', Idx)
    else
      WriteLn('ไม่พบ "อนันต์"');
    
    if SL.Find('ไม่มีชื่อนี้', Idx) then
      WriteLn('พบ')
    else
      WriteLn('ไม่พบ "ไม่มีชื่อนี้"');
    
    // Duplicates
    SL.Duplicates := dupIgnore;  // dupError, dupIgnore, dupAccept
    SL.Add('สมชาย');  // จะถูกข้ามเพราะ dupIgnore
    WriteLn('Count หลังเพิ่ม duplicate: ', SL.Count);
    
  finally
    SL.Free;
  end;
end;

procedure DemoDelimitedText;
var
  SL: TStringList;
begin
  WriteLn(#10'=== TStringList - DelimitedText ===');
  SL := TStringList.Create;
  try
    // Parse CSV
    SL.Delimiter := ',';
    SL.StrictDelimiter := True;
    SL.DelimitedText := 'หนึ่ง,สอง,สาม,สี่,ห้า';
    
    WriteLn('Parse CSV:');
    for var I := 0 to SL.Count - 1 do
      WriteLn('  [', I, '] "', SL[I], '"');
    
    // Create CSV
    SL.Clear;
    SL.Add('ชื่อ');
    SL.Add('อายุ');
    SL.Add('เมือง');
    WriteLn('Create CSV: ', SL.DelimitedText);
    
    // เปลี่ยน delimiter เป็น |
    SL.Delimiter := '|';
    WriteLn('Pipe separated: ', SL.DelimitedText);
    
  finally
    SL.Free;
  end;
end;

begin
  DemoBasicOperations;
  DemoTextAndLines;
  DemoNameValuePairs;
  DemoFind;
  DemoDelimitedText;
  ReadLn;
end.
```

---

## 28.3 TObjectList

```pascal
program TObjectListDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, Contnrs;

type
  TAnimal = class
  private
    FName: String;
    FType: String;
    FAge: Integer;
  public
    constructor Create(const AName, AType: String; AAge: Integer);
    destructor Destroy; override;
    function ToString: String; override;
    property Name: String read FName;
    property AnimalType: String read FType;
    property Age: Integer read FAge;
  end;

constructor TAnimal.Create(const AName, AType: String; AAge: Integer);
begin
  inherited Create;
  FName := AName;
  FType := AType;
  FAge := AAge;
end;

destructor TAnimal.Destroy;
begin
  WriteLn('  [Destroy] ', FName);
  inherited Destroy;
end;

function TAnimal.ToString: String;
begin
  Result := Format('%s (%s, %d ปี)', [FName, FType, FAge]);
end;

procedure DemoObjectList;
var
  List: TObjectList;
  Animal: TAnimal;
  I: Integer;
begin
  WriteLn('=== TObjectList Demo ===');
  
  // OwnsObjects = True (default) - จะ Free objects อัตโนมัติ
  List := TObjectList.Create(True);
  try
    // Add animals
    List.Add(TAnimal.Create('บอล', 'สุนัข', 3));
    List.Add(TAnimal.Create('วิสกี้', 'แมว', 2));
    List.Add(TAnimal.Create('บัน', 'กระต่าย', 1));
    List.Add(TAnimal.Create('นิโม่', 'ปลา', 1));
    List.Add(TAnimal.Create('เหยี่ยว', 'นก', 5));
    
    WriteLn('สัตว์ทั้งหมด (', List.Count, ' ตัว):');
    for I := 0 to List.Count - 1 do
      WriteLn('  ', TAnimal(List[I]).ToString);
    
    // Access by index
    Animal := TAnimal(List[0]);
    WriteLn(#10'สัตว์แรก: ', Animal.ToString);
    
    // Remove (object จะถูก Free ถ้า OwnsObjects = True)
    WriteLn(#10'ลบสัตว์ index 1:');
    List.Delete(1);
    WriteLn('Count หลังลบ: ', List.Count);
    
    // Extract (เอา object ออกโดยไม่ Free)
    WriteLn(#10'Extract สัตว์ index 0 (ไม่ Free):');
    var Extracted := TAnimal(List.Extract(List[0]));
    WriteLn('Extracted: ', Extracted.ToString);
    WriteLn('Count หลัง Extract: ', List.Count);
    Extracted.Free;  // ต้อง Free เอง
    
    // Find
    WriteLn(#10'สัตว์ที่เหลือ:');
    for I := 0 to List.Count - 1 do
      WriteLn('  ', TAnimal(List[I]).ToString);
    
    // Sort (ใช้ CompareFunc)
    WriteLn(#10'Sort โดยอายุ:');
    List.Sort(@(function(A, B: Pointer): Integer
    begin
      Result := TAnimal(A).Age - TAnimal(B).Age;
    end));
    for I := 0 to List.Count - 1 do
      WriteLn('  ', TAnimal(List[I]).ToString);
    
  finally
    WriteLn(#10'Free TObjectList (จะ Free objects อัตโนมัติ):');
    List.Free;
  end;
end;

procedure DemoObjectListNotOwning;
var
  List: TObjectList;
  Animals: array[0..2] of TAnimal;
  I: Integer;
begin
  WriteLn(#10'=== TObjectList ไม่ owns objects ===');
  
  // สร้าง animals แยกต่างหาก
  Animals[0] := TAnimal.Create('บอล', 'สุนัข', 3);
  Animals[1] := TAnimal.Create('วิสกี้', 'แมว', 2);
  Animals[2] := TAnimal.Create('บัน', 'กระต่าย', 1);
  
  // OwnsObjects = False
  List := TObjectList.Create(False);
  try
    for I := 0 to 2 do
      List.Add(Animals[I]);
    
    WriteLn('รายการ:');
    for I := 0 to List.Count - 1 do
      WriteLn('  ', TAnimal(List[I]).ToString);
    
    List.Delete(0);  // ไม่ Free object
    WriteLn('หลังลบ index 0 (object ยังมีอยู่): ', Animals[0].ToString);
    
  finally
    List.Free;  // List ถูก Free แต่ objects ไม่ถูก Free
  end;
  
  // ต้อง Free objects เอง
  WriteLn('Free animals:');
  for I := 0 to 2 do
    Animals[I].Free;
end;

begin
  DemoObjectList;
  DemoObjectListNotOwning;
  ReadLn;
end.
```

---

## 28.4 TCollection and TCollectionItem

```pascal
program TCollectionDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  // TCollectionItem subclass
  TColorItem = class(TCollectionItem)
  private
    FName: String;
    FRed: Byte;
    FGreen: Byte;
    FBlue: Byte;
  protected
    function GetDisplayName: String; override;
  public
    procedure Assign(Source: TPersistent); override;
    function ToHex: String;
    property Name: String read FName write FName;
    property Red: Byte read FRed write FRed;
    property Green: Byte read FGreen write FGreen;
    property Blue: Byte read FBlue write FBlue;
  end;
  
  // TCollection subclass
  TColorCollection = class(TCollection)
  private
    function GetItem(Index: Integer): TColorItem;
    procedure SetItem(Index: Integer; Value: TColorItem);
  public
    constructor Create;
    function Add: TColorItem;
    function FindByName(const AName: String): TColorItem;
    property Items[Index: Integer]: TColorItem read GetItem write SetItem;
  end;

function TColorItem.GetDisplayName: String;
begin
  Result := Format('%s (#%s)', [FName, ToHex]);
end;

procedure TColorItem.Assign(Source: TPersistent);
begin
  if Source is TColorItem then
  begin
    FName := TColorItem(Source).FName;
    FRed := TColorItem(Source).FRed;
    FGreen := TColorItem(Source).FGreen;
    FBlue := TColorItem(Source).FBlue;
  end
  else
    inherited Assign(Source);
end;

function TColorItem.ToHex: String;
begin
  Result := Format('%02X%02X%02X', [FRed, FGreen, FBlue]);
end;

constructor TColorCollection.Create;
begin
  inherited Create(TColorItem);
end;

function TColorCollection.Add: TColorItem;
begin
  Result := TColorItem(inherited Add);
end;

function TColorCollection.GetItem(Index: Integer): TColorItem;
begin
  Result := TColorItem(inherited GetItem(Index));
end;

procedure TColorCollection.SetItem(Index: Integer; Value: TColorItem);
begin
  inherited SetItem(Index, Value);
end;

function TColorCollection.FindByName(const AName: String): TColorItem;
var
  I: Integer;
begin
  Result := nil;
  for I := 0 to Count - 1 do
    if SameText(Items[I].Name, AName) then
    begin
      Result := Items[I];
      Exit;
    end;
end;

procedure AddColor(Colors: TColorCollection; const Name: String; R, G, B: Byte);
var
  Item: TColorItem;
begin
  Item := Colors.Add;
  Item.Name := Name;
  Item.Red := R;
  Item.Green := G;
  Item.Blue := B;
end;

begin
  WriteLn('=== TCollection Demo ===');
  
  var Colors := TColorCollection.Create;
  try
    AddColor(Colors, 'แดง', 255, 0, 0);
    AddColor(Colors, 'เขียว', 0, 255, 0);
    AddColor(Colors, 'น้ำเงิน', 0, 0, 255);
    AddColor(Colors, 'เหลือง', 255, 255, 0);
    AddColor(Colors, 'ขาว', 255, 255, 255);
    AddColor(Colors, 'ดำ', 0, 0, 0);
    
    WriteLn('สีทั้งหมด (', Colors.Count, ' สี):');
    for var I := 0 to Colors.Count - 1 do
    begin
      var C := Colors.Items[I];
      WriteLn(Format('  %d. %s - R:%d G:%d B:%d (#%s)',
                     [I + 1, C.Name, C.Red, C.Green, C.Blue, C.ToHex]));
    end;
    
    // Find
    WriteLn(#10'ค้นหา "น้ำเงิน":');
    var Found := Colors.FindByName('น้ำเงิน');
    if Found <> nil then
      WriteLn('พบ: ', Found.GetDisplayName)
    else
      WriteLn('ไม่พบ');
    
    // Delete
    Colors.Items[1].Free;  // ลบ "เขียว"
    WriteLn(#10'หลังลบ index 1, Count: ', Colors.Count);
    
    // Clone item
    WriteLn(#10'Clone item:');
    var NewColor := Colors.Add;
    NewColor.Assign(Colors.Items[0]);
    NewColor.Name := 'แดงสำเนา';
    WriteLn('Clone: ', NewColor.GetDisplayName);
    
  finally
    Colors.Free;
  end;
  
  ReadLn;
end.
```

---

## 28.5 TDictionary

```pascal
program TDictionaryDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, Generics.Collections;

type
  TWordCount = specialize TDictionary<String, Integer>;
  TStudentGrades = specialize TDictionary<String, TArray<Integer>>;

procedure DemoBasicDictionary;
var
  Dict: specialize TDictionary<String, String>;
  Pair: specialize TPair<String, String>;
begin
  WriteLn('=== Basic TDictionary<String, String> ===');
  Dict := specialize TDictionary<String, String>.Create;
  try
    // Add
    Dict.Add('en', 'English');
    Dict.Add('th', 'ภาษาไทย');
    Dict.Add('ja', 'Japanese');
    Dict.Add('zh', 'Chinese');
    Dict.Add('ko', 'Korean');
    
    WriteLn('Count: ', Dict.Count);
    
    // Access
    WriteLn('th = ', Dict['th']);
    
    // ContainsKey
    WriteLn('ContainsKey("en"): ', Dict.ContainsKey('en'));
    WriteLn('ContainsKey("fr"): ', Dict.ContainsKey('fr'));
    
    // ContainsValue
    WriteLn('ContainsValue("ภาษาไทย"): ', Dict.ContainsValue('ภาษาไทย'));
    
    // TryGetValue
    var Value: String;
    if Dict.TryGetValue('ja', Value) then
      WriteLn('TryGetValue("ja") = ', Value)
    else
      WriteLn('ไม่พบ "ja"');
    
    if Dict.TryGetValue('fr', Value) then
      WriteLn('TryGetValue("fr") = ', Value)
    else
      WriteLn('ไม่พบ "fr"');
    
    // Update (AddOrSetValue)
    Dict.AddOrSetValue('en', 'English Language');
    WriteLn('หลัง update "en": ', Dict['en']);
    
    // Remove
    Dict.Remove('ko');
    WriteLn('หลัง Remove "ko", Count: ', Dict.Count);
    
    // Enumerate
    WriteLn('รายการทั้งหมด:');
    for Pair in Dict do
      WriteLn('  ', Pair.Key, ' -> ', Pair.Value);
    
    // Keys and Values
    Write('Keys: ');
    for var K in Dict.Keys do
      Write(K, ' ');
    WriteLn;
    
    Write('Values: ');
    for var V in Dict.Values do
      Write(V, ' ');
    WriteLn;
    
  finally
    Dict.Free;
  end;
end;

procedure DemoWordCounter;
var
  Words: array of String;
  Counter: TWordCount;
  Pair: specialize TPair<String, Integer>;
begin
  WriteLn(#10'=== Word Counter ด้วย TDictionary ===');
  
  Words := ['สวัสดี', 'โลก', 'สวัสดี', 'ไทย', 'โลก', 'สวัสดี', 'ไทย', 'ไทย'];
  Counter := TWordCount.Create;
  try
    // Count words
    for var Word in Words do
    begin
      var Count: Integer;
      if Counter.TryGetValue(Word, Count) then
        Counter[Word] := Count + 1
      else
        Counter.Add(Word, 1);
    end;
    
    // Display
    WriteLn('จำนวนคำ:');
    for Pair in Counter do
      WriteLn('  "', Pair.Key, '": ', Pair.Value, ' ครั้ง');
    
    // หาคำที่เจอมากที่สุด
    var MaxWord := '';
    var MaxCount := 0;
    for Pair in Counter do
      if Pair.Value > MaxCount then
      begin
        MaxCount := Pair.Value;
        MaxWord := Pair.Key;
      end;
    WriteLn('คำที่เจอมากที่สุด: "', MaxWord, '" (', MaxCount, ' ครั้ง)');
    
  finally
    Counter.Free;
  end;
end;

procedure DemoNestedDictionary;
type
  TInnerDict = specialize TDictionary<String, Integer>;
  TOuterDict = specialize TDictionary<String, TInnerDict>;
  
var
  SchoolData: TOuterDict;
  ClassData: TInnerDict;
begin
  WriteLn(#10'=== Nested Dictionary ===');
  
  SchoolData := TOuterDict.Create;
  try
    // ม.1
    ClassData := TInnerDict.Create;
    ClassData.Add('สมชาย', 85);
    ClassData.Add('สมหญิง', 92);
    ClassData.Add('วิชัย', 78);
    SchoolData.Add('ม.1/1', ClassData);
    
    // ม.2
    ClassData := TInnerDict.Create;
    ClassData.Add('อนันต์', 88);
    ClassData.Add('บุญมี', 75);
    SchoolData.Add('ม.2/1', ClassData);
    
    // Display
    for var ClassPair in SchoolData do
    begin
      WriteLn('ห้อง: ', ClassPair.Key);
      for var StudentPair in ClassPair.Value do
        WriteLn(Format('  %s: %d คะแนน', [StudentPair.Key, StudentPair.Value]));
    end;
    
    // Cleanup
    for var ClassPair2 in SchoolData do
      ClassPair2.Value.Free;
  finally
    SchoolData.Free;
  end;
end;

begin
  DemoBasicDictionary;
  DemoWordCounter;
  DemoNestedDictionary;
  ReadLn;
end.
```

---

## 28.6 Sorting Custom Objects

```pascal
program SortingCustomObjects;
{$mode objfpc}{$H+}

uses
  SysUtils, Generics.Collections, Generics.Defaults;

type
  TEmployee = class
  public
    Name: String;
    Department: String;
    Salary: Double;
    Experience: Integer;
    constructor Create(const AName, ADept: String; ASalary: Double; AExp: Integer);
    function ToString: String; override;
  end;
  
  TEmployeeList = specialize TObjectList<TEmployee>;

constructor TEmployee.Create(const AName, ADept: String; ASalary: Double; AExp: Integer);
begin
  inherited Create;
  Name := AName;
  Department := ADept;
  Salary := ASalary;
  Experience := AExp;
end;

function TEmployee.ToString: String;
begin
  Result := Format('%-15s | %-12s | ฿%10.2f | %d ปี', 
                   [Name, Department, Salary, Experience]);
end;

procedure PrintEmployees(const List: TEmployeeList; const Title: String);
begin
  WriteLn(#10, Title, ':');
  WriteLn(StringOfChar('-', 60));
  for var E in List do
    WriteLn('  ', E.ToString);
  WriteLn(StringOfChar('-', 60));
end;

begin
  WriteLn('=== Sorting Custom Objects ===');
  
  var Employees := TEmployeeList.Create(True);
  try
    Employees.Add(TEmployee.Create('สมชาย', 'IT', 45000, 5));
    Employees.Add(TEmployee.Create('สมหญิง', 'HR', 35000, 3));
    Employees.Add(TEmployee.Create('วิชัย', 'IT', 55000, 8));
    Employees.Add(TEmployee.Create('อนันต์', 'Finance', 40000, 4));
    Employees.Add(TEmployee.Create('บุญมี', 'HR', 32000, 2));
    Employees.Add(TEmployee.Create('ประยุทธ์', 'IT', 60000, 10));
    Employees.Add(TEmployee.Create('สุวรรณ', 'Finance', 48000, 6));
    
    PrintEmployees(Employees, 'ลำดับเดิม');
    
    // Sort by Salary (ascending)
    Employees.Sort(specialize TComparer<TEmployee>.Construct(
      function(const A, B: TEmployee): Integer
      begin
        Result := Round(A.Salary - B.Salary);
      end));
    PrintEmployees(Employees, 'เรียงตามเงินเดือน (น้อย -> มาก)');
    
    // Sort by Salary (descending)
    Employees.Sort(specialize TComparer<TEmployee>.Construct(
      function(const A, B: TEmployee): Integer
      begin
        Result := Round(B.Salary - A.Salary);
      end));
    PrintEmployees(Employees, 'เรียงตามเงินเดือน (มาก -> น้อย)');
    
    // Sort by Department then Name
    Employees.Sort(specialize TComparer<TEmployee>.Construct(
      function(const A, B: TEmployee): Integer
      begin
        Result := CompareStr(A.Department, B.Department);
        if Result = 0 then
          Result := CompareStr(A.Name, B.Name);
      end));
    PrintEmployees(Employees, 'เรียงตามแผนก แล้วตามชื่อ');
    
    // Sort by Experience (descending)
    Employees.Sort(specialize TComparer<TEmployee>.Construct(
      function(const A, B: TEmployee): Integer
      begin
        Result := B.Experience - A.Experience;
      end));
    PrintEmployees(Employees, 'เรียงตามประสบการณ์ (มาก -> น้อย)');
    
  finally
    Employees.Free;
  end;
  
  ReadLn;
end.
```

---

## 28.7 Enumerators and for..in Loop

```pascal
program EnumeratorsDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  // Custom collection ที่รองรับ for..in
  TNumberRange = class
  private
    FStart: Integer;
    FEnd: Integer;
    FStep: Integer;
  public
    constructor Create(AStart, AEnd: Integer; AStep: Integer = 1);
    function GetEnumerator: TNumberRangeEnumerator;
  end;
  
  TNumberRangeEnumerator = class
  private
    FRange: TNumberRange;
    FCurrent: Integer;
  public
    constructor Create(ARange: TNumberRange);
    function MoveNext: Boolean;
    property Current: Integer read FCurrent;
  end;

constructor TNumberRange.Create(AStart, AEnd: Integer; AStep: Integer);
begin
  inherited Create;
  FStart := AStart;
  FEnd := AEnd;
  FStep := AStep;
end;

function TNumberRange.GetEnumerator: TNumberRangeEnumerator;
begin
  Result := TNumberRangeEnumerator.Create(Self);
end;

constructor TNumberRangeEnumerator.Create(ARange: TNumberRange);
begin
  inherited Create;
  FRange := ARange;
  FCurrent := ARange.FStart - ARange.FStep;
end;

function TNumberRangeEnumerator.MoveNext: Boolean;
begin
  Inc(FCurrent, FRange.FStep);
  Result := FCurrent <= FRange.FEnd;
end;

// Custom filtered collection
type
  TFilteredList = class
  private
    FItems: TStringList;
    FFilter: String;
  public
    constructor Create;
    destructor Destroy; override;
    procedure Add(const S: String);
    function GetEnumerator: TFilteredEnumerator;
    property Filter: String read FFilter write FFilter;
  end;
  
  TFilteredEnumerator = class
  private
    FList: TFilteredList;
    FIndex: Integer;
    FCurrent: String;
  public
    constructor Create(AList: TFilteredList);
    destructor Destroy; override;
    function MoveNext: Boolean;
    property Current: String read FCurrent;
  end;

constructor TFilteredList.Create;
begin
  inherited Create;
  FItems := TStringList.Create;
end;

destructor TFilteredList.Destroy;
begin
  FItems.Free;
  inherited Destroy;
end;

procedure TFilteredList.Add(const S: String);
begin
  FItems.Add(S);
end;

function TFilteredList.GetEnumerator: TFilteredEnumerator;
begin
  Result := TFilteredEnumerator.Create(Self);
end;

constructor TFilteredEnumerator.Create(AList: TFilteredList);
begin
  inherited Create;
  FList := AList;
  FIndex := -1;
end;

destructor TFilteredEnumerator.Destroy;
begin
  inherited Destroy;
end;

function TFilteredEnumerator.MoveNext: Boolean;
begin
  Result := False;
  Inc(FIndex);
  while FIndex < FList.FItems.Count do
  begin
    FCurrent := FList.FItems[FIndex];
    if (FList.FFilter = '') or 
       (Pos(FList.FFilter, FCurrent) > 0) then
    begin
      Result := True;
      Exit;
    end;
    Inc(FIndex);
  end;
end;

begin
  WriteLn('=== Enumerators และ for..in ===');
  
  // ใช้ TStringList กับ for..in
  WriteLn(#10'1. TStringList for..in:');
  var SL := TStringList.Create;
  try
    SL.Add('แดง');
    SL.Add('เขียว');
    SL.Add('น้ำเงิน');
    for var S in SL do
      WriteLn('  ', S);
  finally
    SL.Free;
  end;
  
  // Custom Range Enumerator
  WriteLn(#10'2. Custom Range Enumerator:');
  var Range := TNumberRange.Create(1, 10, 2);  // 1, 3, 5, 7, 9
  try
    Write('เลขคี่ 1-10: ');
    for var N in Range do
      Write(N, ' ');
    WriteLn;
  finally
    Range.Free;
  end;
  
  var Range2 := TNumberRange.Create(0, 20, 5);  // 0, 5, 10, 15, 20
  try
    Write('ทวีคูณของ 5: ');
    for var N in Range2 do
      Write(N, ' ');
    WriteLn;
  finally
    Range2.Free;
  end;
  
  // Filtered List Enumerator
  WriteLn(#10'3. Filtered List Enumerator:');
  var FilteredList := TFilteredList.Create;
  try
    FilteredList.Add('สวัสดีกรุงเทพ');
    FilteredList.Add('สวัสดีเชียงใหม่');
    FilteredList.Add('ลาก่อนกรุงเทพ');
    FilteredList.Add('ยินดีต้อนรับกรุงเทพ');
    FilteredList.Add('สวัสดีภูเก็ต');
    
    // ไม่กรอง
    FilteredList.Filter := '';
    WriteLn('ไม่กรอง:');
    for var S in FilteredList do
      WriteLn('  ', S);
    
    // กรองเฉพาะที่มี "สวัสดี"
    FilteredList.Filter := 'สวัสดี';
    WriteLn('กรอง "สวัสดี":');
    for var S in FilteredList do
      WriteLn('  ', S);
    
    // กรองเฉพาะที่มี "กรุงเทพ"
    FilteredList.Filter := 'กรุงเทพ';
    WriteLn('กรอง "กรุงเทพ":');
    for var S in FilteredList do
      WriteLn('  ', S);
    
  finally
    FilteredList.Free;
  end;
  
  ReadLn;
end.
```

---

## 28.8 LINQ-style Operations

```pascal
program LINQStyleDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, Generics.Collections;

type
  TIntList = specialize TList<Integer>;
  TStrList = specialize TList<String>;

// LINQ-style extensions
function Where(const Source: TIntList; 
               Predicate: specialize TFunc<Integer, Boolean>): TIntList;
var
  Item: Integer;
begin
  Result := TIntList.Create;
  for Item in Source do
    if Predicate(Item) then
      Result.Add(Item);
end;

function Select(const Source: TIntList;
                Selector: specialize TFunc<Integer, Integer>): TIntList;
var
  Item: Integer;
begin
  Result := TIntList.Create;
  for Item in Source do
    Result.Add(Selector(Item));
end;

function SelectStr(const Source: TIntList;
                   Selector: specialize TFunc<Integer, String>): TStrList;
var
  Item: Integer;
begin
  Result := TStrList.Create;
  for Item in Source do
    Result.Add(Selector(Item));
end;

function Sum(const Source: TIntList): Integer;
var
  Item: Integer;
begin
  Result := 0;
  for Item in Source do
    Inc(Result, Item);
end;

function Average(const Source: TIntList): Double;
begin
  if Source.Count = 0 then
    Result := 0
  else
    Result := Sum(Source) / Source.Count;
end;

function MaxValue(const Source: TIntList): Integer;
var
  Item: Integer;
begin
  if Source.Count = 0 then
    raise Exception.Create('List ว่างเปล่า');
  Result := Source[0];
  for Item in Source do
    if Item > Result then Result := Item;
end;

function MinValue(const Source: TIntList): Integer;
var
  Item: Integer;
begin
  if Source.Count = 0 then
    raise Exception.Create('List ว่างเปล่า');
  Result := Source[0];
  for Item in Source do
    if Item < Result then Result := Item;
end;

function Any(const Source: TIntList;
             Predicate: specialize TFunc<Integer, Boolean>): Boolean;
var
  Item: Integer;
begin
  Result := False;
  for Item in Source do
    if Predicate(Item) then
    begin
      Result := True;
      Exit;
    end;
end;

function All(const Source: TIntList;
             Predicate: specialize TFunc<Integer, Boolean>): Boolean;
var
  Item: Integer;
begin
  Result := True;
  for Item in Source do
    if not Predicate(Item) then
    begin
      Result := False;
      Exit;
    end;
end;

function CountWhere(const Source: TIntList;
                    Predicate: specialize TFunc<Integer, Boolean>): Integer;
var
  Item: Integer;
begin
  Result := 0;
  for Item in Source do
    if Predicate(Item) then
      Inc(Result);
end;

procedure PrintList(const List: TIntList; const Title: String);
var
  I: Integer;
begin
  Write(Title, ': [');
  for I := 0 to List.Count - 1 do
  begin
    if I > 0 then Write(', ');
    Write(List[I]);
  end;
  WriteLn(']');
end;

procedure PrintStrList(const List: TStrList; const Title: String);
var
  I: Integer;
begin
  Write(Title, ': [');
  for I := 0 to List.Count - 1 do
  begin
    if I > 0 then Write(', ');
    Write('"', List[I], '"');
  end;
  WriteLn(']');
end;

var
  Numbers: TIntList;
  Filtered, Mapped: TIntList;
  MappedStr: TStrList;

begin
  WriteLn('=== LINQ-style Operations ===');
  
  // สร้างข้อมูล
  Numbers := TIntList.Create;
  try
    for var I := 1 to 10 do
      Numbers.Add(I);
    
    PrintList(Numbers, 'ต้นฉบับ');
    
    // Where (กรอง)
    WriteLn(#10'--- Where ---');
    Filtered := Where(Numbers, function(N: Integer): Boolean
    begin
      Result := N mod 2 = 0;  // เลขคู่
    end);
    try
      PrintList(Filtered, 'เลขคู่');
    finally
      Filtered.Free;
    end;
    
    Filtered := Where(Numbers, function(N: Integer): Boolean
    begin
      Result := N > 5;  // มากกว่า 5
    end);
    try
      PrintList(Filtered, 'มากกว่า 5');
    finally
      Filtered.Free;
    end;
    
    // Select (แปลง)
    WriteLn(#10'--- Select ---');
    Mapped := Select(Numbers, function(N: Integer): Integer
    begin
      Result := N * N;  // ยกกำลังสอง
    end);
    try
      PrintList(Mapped, 'ยกกำลังสอง');
    finally
      Mapped.Free;
    end;
    
    MappedStr := SelectStr(Numbers, function(N: Integer): String
    begin
      if N mod 2 = 0 then
        Result := IntToStr(N) + ' (คู่)'
      else
        Result := IntToStr(N) + ' (คี่)';
    end);
    try
      PrintStrList(MappedStr, 'แปลงเป็น String');
    finally
      MappedStr.Free;
    end;
    
    // Aggregates
    WriteLn(#10'--- Aggregates ---');
    WriteLn('Sum: ', Sum(Numbers));
    WriteLn(Format('Average: %.1f', [Average(Numbers)]));
    WriteLn('Max: ', MaxValue(Numbers));
    WriteLn('Min: ', MinValue(Numbers));
    WriteLn('Count: ', Numbers.Count);
    WriteLn('Count (เลขคู่): ', CountWhere(Numbers, 
      function(N: Integer): Boolean begin Result := N mod 2 = 0; end));
    
    // Any/All
    WriteLn(#10'--- Any/All ---');
    WriteLn('Any > 8: ', Any(Numbers, function(N: Integer): Boolean begin Result := N > 8; end));
    WriteLn('Any > 15: ', Any(Numbers, function(N: Integer): Boolean begin Result := N > 15; end));
    WriteLn('All > 0: ', All(Numbers, function(N: Integer): Boolean begin Result := N > 0; end));
    WriteLn('All > 5: ', All(Numbers, function(N: Integer): Boolean begin Result := N > 5; end));
    
    // Method Chaining (ใช้หลาย operations)
    WriteLn(#10'--- Method Chaining ---');
    WriteLn('Sum ของ เลขคู่ ยกกำลังสอง:');
    var Step1 := Where(Numbers, function(N: Integer): Boolean begin Result := N mod 2 = 0; end);
    try
      var Step2 := Select(Step1, function(N: Integer): Integer begin Result := N * N; end);
      try
        PrintList(Step2, 'เลขคู่ ยกกำลังสอง');
        WriteLn('Sum: ', Sum(Step2));
      finally
        Step2.Free;
      end;
    finally
      Step1.Free;
    end;
    
  finally
    Numbers.Free;
  end;
  
  ReadLn;
end.
```

---

## 28.9 โปรแกรมตัวอย่าง: Contact Book using TStringList

```pascal
program ContactBook;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  TContactBook = class
  private
    FContacts: TStringList;
    FDataFile: String;
    
    function FormatContact(const Name, Phone, Email: String): String;
    procedure ParseContact(const Line: String; out Name, Phone, Email: String);
  public
    constructor Create(const ADataFile: String);
    destructor Destroy; override;
    procedure Add(const Name, Phone, Email: String);
    procedure Update(const Name, NewPhone, NewEmail: String);
    procedure Delete(const Name: String);
    function Search(const Query: String): TStringList;
    procedure ListAll;
    procedure Save;
    procedure Load;
    function Count: Integer;
  end;

constructor TContactBook.Create(const ADataFile: String);
begin
  inherited Create;
  FDataFile := ADataFile;
  FContacts := TStringList.Create;
  FContacts.Sorted := True;
  FContacts.NameValueSeparator := '|';
  Load;
end;

destructor TContactBook.Destroy;
begin
  Save;
  FContacts.Free;
  inherited Destroy;
end;

function TContactBook.FormatContact(const Name, Phone, Email: String): String;
begin
  Result := Format('%s|%s;%s', [Name, Phone, Email]);
end;

procedure TContactBook.ParseContact(const Line: String; out Name, Phone, Email: String);
var
  Pos1, Pos2: Integer;
  ValuePart: String;
begin
  Pos1 := Pos('|', Line);
  if Pos1 = 0 then
  begin
    Name := Line;
    Phone := '';
    Email := '';
    Exit;
  end;
  
  Name := Copy(Line, 1, Pos1 - 1);
  ValuePart := Copy(Line, Pos1 + 1, MaxInt);
  
  Pos2 := Pos(';', ValuePart);
  if Pos2 = 0 then
  begin
    Phone := ValuePart;
    Email := '';
  end
  else
  begin
    Phone := Copy(ValuePart, 1, Pos2 - 1);
    Email := Copy(ValuePart, Pos2 + 1, MaxInt);
  end;
end;

procedure TContactBook.Add(const Name, Phone, Email: String);
var
  Idx: Integer;
begin
  if FContacts.Find(Name + '|', Idx) then
    raise Exception.CreateFmt('ติดต่อ "%s" มีอยู่แล้ว', [Name]);
  FContacts.Add(FormatContact(Name, Phone, Email));
  WriteLn('เพิ่มติดต่อ: ', Name);
end;

procedure TContactBook.Update(const Name, NewPhone, NewEmail: String);
var
  Idx: Integer;
  I: Integer;
begin
  for I := 0 to FContacts.Count - 1 do
  begin
    var n, p, e: String;
    ParseContact(FContacts[I], n, p, e);
    if SameText(n, Name) then
    begin
      FContacts[I] := FormatContact(Name, NewPhone, NewEmail);
      WriteLn('อัปเดตติดต่อ: ', Name);
      Exit;
    end;
  end;
  raise Exception.CreateFmt('ไม่พบติดต่อ "%s"', [Name]);
end;

procedure TContactBook.Delete(const Name: String);
var
  I: Integer;
begin
  for I := 0 to FContacts.Count - 1 do
  begin
    var n, p, e: String;
    ParseContact(FContacts[I], n, p, e);
    if SameText(n, Name) then
    begin
      FContacts.Delete(I);
      WriteLn('ลบติดต่อ: ', Name);
      Exit;
    end;
  end;
  raise Exception.CreateFmt('ไม่พบติดต่อ "%s"', [Name]);
end;

function TContactBook.Search(const Query: String): TStringList;
begin
  Result := TStringList.Create;
  for var I := 0 to FContacts.Count - 1 do
  begin
    if (Query = '') or (Pos(LowerCase(Query), LowerCase(FContacts[I])) > 0) then
      Result.Add(FContacts[I]);
  end;
end;

procedure TContactBook.ListAll;
var
  I: Integer;
  Name, Phone, Email: String;
begin
  WriteLn(StringOfChar('=', 50));
  WriteLn('สมุดโทรศัพท์ (', FContacts.Count, ' ติดต่อ)');
  WriteLn(StringOfChar('-', 50));
  
  if FContacts.Count = 0 then
    WriteLn('ไม่มีติดต่อ')
  else
    for I := 0 to FContacts.Count - 1 do
    begin
      ParseContact(FContacts[I], Name, Phone, Email);
      WriteLn(Format('%-20s | %-15s | %s', [Name, Phone, Email]));
    end;
  
  WriteLn(StringOfChar('=', 50));
end;

procedure TContactBook.Save;
begin
  try
    FContacts.SaveToFile(FDataFile);
  except
    on E: Exception do
      WriteLn('ไม่สามารถบันทึก: ', E.Message);
  end;
end;

procedure TContactBook.Load;
begin
  if FileExists(FDataFile) then
    try
      FContacts.LoadFromFile(FDataFile);
    except
      on E: Exception do
        WriteLn('ไม่สามารถโหลด: ', E.Message);
    end;
end;

function TContactBook.Count: Integer;
begin
  Result := FContacts.Count;
end;

begin
  WriteLn('=== สมุดโทรศัพท์ ===');
  
  var Book := TContactBook.Create('/tmp/contacts.txt');
  try
    // เพิ่มติดต่อ
    WriteLn(#10'เพิ่มติดต่อ:');
    Book.Add('สมชาย ใจดี', '081-234-5678', 'somchai@email.com');
    Book.Add('สมหญิง สวยงาม', '089-876-5432', 'somying@email.com');
    Book.Add('วิชัย เก่งกาจ', '062-111-2233', 'wichai@email.com');
    Book.Add('อนันต์ มีสุข', '095-444-5566', 'anant@email.com');
    Book.Add('บุญมี ร่ำรวย', '086-777-8899', 'boonmee@email.com');
    
    // แสดงทั้งหมด
    WriteLn(#10'รายการทั้งหมด:');
    Book.ListAll;
    
    // ค้นหา
    WriteLn(#10'ค้นหา "สมชาย":');
    var Results := Book.Search('สมชาย');
    try
      if Results.Count = 0 then
        WriteLn('ไม่พบ')
      else
        for var S in Results do
          WriteLn('  ', S);
    finally
      Results.Free;
    end;
    
    // อัปเดต
    WriteLn(#10'อัปเดตวิชัย:');
    Book.Update('วิชัย เก่งกาจ', '099-000-1111', 'wichai_new@email.com');
    
    // ลบ
    WriteLn(#10'ลบบุญมี:');
    Book.Delete('บุญมี ร่ำรวย');
    
    // แสดงอีกครั้ง
    WriteLn(#10'รายการหลังแก้ไข:');
    Book.ListAll;
    
    WriteLn(#10'จำนวนติดต่อทั้งหมด: ', Book.Count);
    
  finally
    Book.Free;
  end;
  
  ReadLn;
end.
```

---

## 28.10 โปรแกรมตัวอย่าง: Task Manager using TObjectList

```pascal
program TaskManager;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes, Contnrs, Generics.Collections;

type
  TTaskStatus = (tsNew, tsInProgress, tsCompleted, tsCancelled);
  TTaskPriority = (tpLow, tpNormal, tpHigh, tpCritical);
  
  TTask = class
  private
    FID: Integer;
    FTitle: String;
    FDescription: String;
    FStatus: TTaskStatus;
    FPriority: TTaskPriority;
    FCreatedAt: TDateTime;
    FDueDate: TDateTime;
    
    function GetStatusText: String;
    function GetPriorityText: String;
  public
    constructor Create(AID: Integer; const ATitle: String;
                       APriority: TTaskPriority = tpNormal);
    function ToString: String; override;
    property ID: Integer read FID;
    property Title: String read FTitle write FTitle;
    property Description: String read FDescription write FDescription;
    property Status: TTaskStatus read FStatus write FStatus;
    property Priority: TTaskPriority read FPriority write FPriority;
    property CreatedAt: TDateTime read FCreatedAt;
    property DueDate: TDateTime read FDueDate write FDueDate;
    property StatusText: String read GetStatusText;
    property PriorityText: String read GetPriorityText;
  end;
  
  TTaskList = specialize TObjectList<TTask>;
  
  TTaskManager = class
  private
    FTasks: TTaskList;
    FNextID: Integer;
  public
    constructor Create;
    destructor Destroy; override;
    function AddTask(const Title: String; Priority: TTaskPriority = tpNormal): TTask;
    function GetTaskByID(ID: Integer): TTask;
    procedure StartTask(ID: Integer);
    procedure CompleteTask(ID: Integer);
    procedure CancelTask(ID: Integer);
    function GetByStatus(Status: TTaskStatus): TTaskList;
    function GetByPriority(Priority: TTaskPriority): TTaskList;
    procedure PrintAll;
    procedure PrintByStatus(Status: TTaskStatus);
    procedure PrintStats;
    property Count: Integer read (FTasks.Count);
  end;

function TTask.GetStatusText: String;
begin
  case FStatus of
    tsNew: Result := 'ใหม่';
    tsInProgress: Result := 'กำลังทำ';
    tsCompleted: Result := 'เสร็จแล้ว';
    tsCancelled: Result := 'ยกเลิก';
  end;
end;

function TTask.GetPriorityText: String;
begin
  case FPriority of
    tpLow: Result := 'ต่ำ';
    tpNormal: Result := 'ปกติ';
    tpHigh: Result := 'สูง';
    tpCritical: Result := 'วิกฤต';
  end;
end;

constructor TTask.Create(AID: Integer; const ATitle: String; APriority: TTaskPriority);
begin
  inherited Create;
  FID := AID;
  FTitle := ATitle;
  FStatus := tsNew;
  FPriority := APriority;
  FCreatedAt := Now;
  FDueDate := 0;
end;

function TTask.ToString: String;
begin
  Result := Format('[#%d] %s | สถานะ: %-10s | ความสำคัญ: %-8s | สร้าง: %s',
                   [FID, FTitle, StatusText, PriorityText,
                    FormatDateTime('dd/mm/yy', FCreatedAt)]);
end;

constructor TTaskManager.Create;
begin
  inherited Create;
  FTasks := TTaskList.Create(True);
  FNextID := 1;
end;

destructor TTaskManager.Destroy;
begin
  FTasks.Free;
  inherited Destroy;
end;

function TTaskManager.AddTask(const Title: String; Priority: TTaskPriority): TTask;
begin
  Result := TTask.Create(FNextID, Title, Priority);
  FTasks.Add(Result);
  Inc(FNextID);
end;

function TTaskManager.GetTaskByID(ID: Integer): TTask;
begin
  Result := nil;
  for var T in FTasks do
    if T.ID = ID then
    begin
      Result := T;
      Exit;
    end;
end;

procedure TTaskManager.StartTask(ID: Integer);
var
  T: TTask;
begin
  T := GetTaskByID(ID);
  if T = nil then
    raise Exception.CreateFmt('ไม่พบงาน #%d', [ID]);
  if T.Status <> tsNew then
    raise Exception.CreateFmt('งาน #%d ไม่อยู่ในสถานะ "ใหม่"', [ID]);
  T.Status := tsInProgress;
end;

procedure TTaskManager.CompleteTask(ID: Integer);
var
  T: TTask;
begin
  T := GetTaskByID(ID);
  if T = nil then
    raise Exception.CreateFmt('ไม่พบงาน #%d', [ID]);
  T.Status := tsCompleted;
end;

procedure TTaskManager.CancelTask(ID: Integer);
var
  T: TTask;
begin
  T := GetTaskByID(ID);
  if T = nil then
    raise Exception.CreateFmt('ไม่พบงาน #%d', [ID]);
  T.Status := tsCancelled;
end;

function TTaskManager.GetByStatus(Status: TTaskStatus): TTaskList;
begin
  Result := TTaskList.Create(False);  // ไม่ own objects
  for var T in FTasks do
    if T.Status = Status then
      Result.Add(T);
end;

function TTaskManager.GetByPriority(Priority: TTaskPriority): TTaskList;
begin
  Result := TTaskList.Create(False);
  for var T in FTasks do
    if T.Priority = Priority then
      Result.Add(T);
end;

procedure TTaskManager.PrintAll;
begin
  WriteLn(StringOfChar('=', 80));
  WriteLn('งานทั้งหมด (', Count, ' งาน)');
  WriteLn(StringOfChar('-', 80));
  for var T in FTasks do
    WriteLn('  ', T.ToString);
  WriteLn(StringOfChar('=', 80));
end;

procedure TTaskManager.PrintByStatus(Status: TTaskStatus);
var
  StatusText: String;
begin
  case Status of
    tsNew: StatusText := 'ใหม่';
    tsInProgress: StatusText := 'กำลังทำ';
    tsCompleted: StatusText := 'เสร็จแล้ว';
    tsCancelled: StatusText := 'ยกเลิก';
  end;
  
  WriteLn(Format('งานสถานะ "%s":', [StatusText]));
  var List := GetByStatus(Status);
  try
    if List.Count = 0 then
      WriteLn('  ไม่มีงาน')
    else
      for var T in List do
        WriteLn('  ', T.ToString);
  finally
    List.Free;
  end;
end;

procedure TTaskManager.PrintStats;
var
  Counts: array[TTaskStatus] of Integer;
  S: TTaskStatus;
begin
  for S := Low(TTaskStatus) to High(TTaskStatus) do
    Counts[S] := 0;
  
  for var T in FTasks do
    Inc(Counts[T.Status]);
  
  WriteLn(#10'=== สถิติงาน ===');
  WriteLn(Format('  %-12s: %d', ['ใหม่', Counts[tsNew]]));
  WriteLn(Format('  %-12s: %d', ['กำลังทำ', Counts[tsInProgress]]));
  WriteLn(Format('  %-12s: %d', ['เสร็จแล้ว', Counts[tsCompleted]]));
  WriteLn(Format('  %-12s: %d', ['ยกเลิก', Counts[tsCancelled]]));
  WriteLn(Format('  %-12s: %d', ['ทั้งหมด', Count]));
end;

begin
  WriteLn('=== ระบบจัดการงาน ===');
  
  var Manager := TTaskManager.Create;
  try
    // เพิ่มงาน
    WriteLn(#10'เพิ่มงาน:');
    Manager.AddTask('ออกแบบ UI', tpHigh);
    Manager.AddTask('เขียน Backend API', tpHigh);
    Manager.AddTask('เขียน Database Schema', tpCritical);
    Manager.AddTask('เขียน Unit Tests', tpNormal);
    Manager.AddTask('จัดทำ Documentation', tpLow);
    Manager.AddTask('Deploy to Production', tpCritical);
    Manager.AddTask('Code Review', tpNormal);
    
    Manager.PrintAll;
    
    // เริ่มงาน
    WriteLn(#10'เริ่มงาน:');
    Manager.StartTask(1);
    Manager.StartTask(3);
    Manager.StartTask(6);
    
    // เสร็จงาน
    WriteLn(#10'เสร็จงาน:');
    Manager.CompleteTask(3);
    Manager.CompleteTask(6);
    
    // ยกเลิกงาน
    WriteLn(#10'ยกเลิกงาน:');
    Manager.CancelTask(5);
    
    // แสดงตามสถานะ
    WriteLn(#10'แสดงตามสถานะ:');
    Manager.PrintByStatus(tsNew);
    Manager.PrintByStatus(tsInProgress);
    Manager.PrintByStatus(tsCompleted);
    Manager.PrintByStatus(tsCancelled);
    
    // สถิติ
    Manager.PrintStats;
    
    // แสดงงาน critical
    WriteLn(#10'งานที่มีความสำคัญวิกฤต:');
    var CriticalTasks := Manager.GetByPriority(tpCritical);
    try
      for var T in CriticalTasks do
        WriteLn('  ', T.ToString);
    finally
      CriticalTasks.Free;
    end;
    
  finally
    Manager.Free;
  end;
  
  ReadLn;
end.
```

---

## 28.11 โปรแกรมตัวอย่าง: Word Frequency Counter using TDictionary

```pascal
program WordFrequency;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes, Generics.Collections;

type
  TWordFreq = specialize TDictionary<String, Integer>;
  TWordFreqPair = specialize TPair<String, Integer>;
  TWordFreqList = specialize TList<TWordFreqPair>;

procedure CountWords(const Text: String; FreqMap: TWordFreq);
var
  Words: TStringList;
  Word: String;
  Count: Integer;
begin
  Words := TStringList.Create;
  try
    Words.Delimiter := ' ';
    Words.StrictDelimiter := True;
    Words.DelimitedText := LowerCase(Text);
    
    for Word in Words do
    begin
      // ลบเครื่องหมายวรรคตอน
      var CleanWord := '';
      for var C in Word do
        if C in ['a'..'z', '0'..'9', 'ก'..'ฮ', 'า'..'ๆ'] then
          CleanWord := CleanWord + C;
      
      if CleanWord = '' then Continue;
      
      if FreqMap.TryGetValue(CleanWord, Count) then
        FreqMap[CleanWord] := Count + 1
      else
        FreqMap.Add(CleanWord, 1);
    end;
  finally
    Words.Free;
  end;
end;

function GetTopN(FreqMap: TWordFreq; N: Integer): TWordFreqList;
var
  Pair: TWordFreqPair;
begin
  Result := TWordFreqList.Create;
  for Pair in FreqMap do
    Result.Add(Pair);
  
  // Sort by frequency (descending)
  Result.Sort(specialize TComparer<TWordFreqPair>.Construct(
    function(const A, B: TWordFreqPair): Integer
    begin
      Result := B.Value - A.Value;  // descending
      if Result = 0 then
        Result := CompareStr(A.Key, B.Key);  // alphabetical if same freq
    end));
  
  // Trim to N items
  while Result.Count > N do
    Result.Delete(Result.Count - 1);
end;

begin
  WriteLn('=== Word Frequency Counter ===');
  
  var Text := 'สวัสดี โลก สวัสดี ไทย ไทย ไทย ' +
              'การเขียน โปรแกรม การเขียน โปรแกรม ' +
              'ภาษา ปาสคาล ภาษา ปาสคาล ภาษา ' +
              'ลาซารัส ลาซารัส Lazarus Pascal ' +
              'สวัสดี สวัสดี สวัสดี';
  
  WriteLn('ข้อความ:');
  WriteLn(Text);
  WriteLn;
  
  var FreqMap := TWordFreq.Create;
  try
    CountWords(Text, FreqMap);
    
    WriteLn('จำนวนคำที่ไม่ซ้ำ: ', FreqMap.Count);
    
    // Top 10 คำที่ใช้บ่อยที่สุด
    WriteLn(#10'Top 10 คำที่ใช้บ่อยที่สุด:');
    WriteLn(StringOfChar('-', 30));
    
    var TopWords := GetTopN(FreqMap, 10);
    try
      for var I := 0 to TopWords.Count - 1 do
      begin
        var Pair := TopWords[I];
        var Bar := StringOfChar('█', Pair.Value * 2);
        WriteLn(Format('%-15s: %3d %s', [Pair.Key, Pair.Value, Bar]));
      end;
    finally
      TopWords.Free;
    end;
    
    // ค้นหาคำ
    WriteLn(#10'ค้นหาความถี่คำ:');
    var SearchWords: array of String = ['สวัสดี', 'ไทย', 'Pascal', 'xyz'];
    for var W in SearchWords do
    begin
      var Count: Integer;
      if FreqMap.TryGetValue(LowerCase(W), Count) then
        WriteLn(Format('  "%s": %d ครั้ง', [W, Count]))
      else
        WriteLn(Format('  "%s": ไม่พบ', [W]));
    end;
    
    // คำที่ใช้ครั้งเดียว
    WriteLn(#10'คำที่ใช้ครั้งเดียว:');
    var Once: TStringList := TStringList.Create;
    try
      for var Pair in FreqMap do
        if Pair.Value = 1 then
          Once.Add(Pair.Key);
      Once.Sort;
      for var W in Once do
        Write(W, ' ');
      WriteLn;
    finally
      Once.Free;
    end;
    
  finally
    FreqMap.Free;
  end;
  
  ReadLn;
end.
```

---

## 28.12 แบบฝึกหัด

### แบบฝึกหัดที่ 1: TStringList Utilities
เขียน utility functions สำหรับ TStringList:
- `RemoveDuplicates` - ลบรายการซ้ำ
- `Intersect` - หาส่วนที่เหมือนกันของ 2 lists
- `Difference` - หาส่วนที่แตกต่าง
- `Union` - รวม 2 lists โดยไม่ซ้ำ

### แบบฝึกหัดที่ 2: CSV Parser
เขียน `TCSVParser` ที่ใช้ TStringList:
- อ่านไฟล์ CSV
- แยก header และ data rows
- ให้เข้าถึงข้อมูลด้วย column name
- รองรับ quoted values

### แบบฝึกหัดที่ 3: MultiMap
สร้าง `TMultiMap<TKey, TValue>` ที่ key หนึ่งมีได้หลาย values:
```pascal
MultiMap.Add('fruits', 'apple');
MultiMap.Add('fruits', 'banana');
MultiMap.GetValues('fruits'); // ['apple', 'banana']
```

### แบบฝึกหัดที่ 4: Ordered Dictionary
สร้าง dictionary ที่เก็บลำดับการเพิ่มข้อมูล (ไม่เหมือน TDictionary ที่ไม่มีลำดับ)

### แบบฝึกหัดที่ 5: LRU Cache
สร้าง `TLRUCache<TKey, TValue>` (Least Recently Used) ที่:
- กำหนดขนาดสูงสุด
- เมื่อเต็ม ลบรายการที่ใช้นานที่สุด
- O(1) get และ put

### แบบฝึกหัดที่ 6: Sorted ObjectList
สร้าง `TSortedObjectList<T>` ที่แทรกข้อมูลในตำแหน่งที่ถูกต้องเสมอ (binary insert)

### แบบฝึกหัดที่ 7: Bag (Multiset)
สร้าง `TBag<T>` ที่:
- เก็บได้หลายชุดของค่าเดียวกัน
- Add, Remove, Contains
- Frequency(item) ส่งคืนจำนวนครั้ง

### แบบฝึกหัดที่ 8: Graph
สร้าง `TGraph` ที่ใช้ TDictionary เก็บ adjacency list:
- AddVertex, AddEdge
- HasPath (BFS)
- Neighbors

### แบบฝึกหัดที่ 9: Event Log
สร้าง `TEventLog` ที่ใช้ TObjectList:
- บันทึก events พร้อมเวลา
- Filter by type, date range
- Export to string

### แบบฝึกหัดที่ 10: Inventory System
สร้างระบบ inventory ที่:
- ใช้ TDictionary<String, TProduct> สำหรับ catalog
- ใช้ TObjectList สำหรับ transactions
- คำนวณ stock level
- หา low stock items

### แบบฝึกหัดที่ 11: Student Grade Book
สร้าง grade book ที่:
- ใช้ TDictionary<String, TStringList> (student -> subjects)
- คำนวณ GPA
- หานักเรียนที่ทำคะแนนสูงสุด/ต่ำสุด
- สร้างรายงาน

### แบบฝึกหัดที่ 12: Collection Pipeline
สร้าง fluent API สำหรับ collection operations:
```pascal
Result := TCollection.From(Data)
  .Where(...)
  .Select(...)
  .OrderBy(...)
  .Take(10)
  .ToList;
```

### แบบฝึกหัดที่ 13: In-Memory Cache
สร้าง thread-safe in-memory cache ที่:
- กำหนด TTL
- Auto-expire entries
- Max size with eviction

### แบบฝึกหัดที่ 14: Priority Task Queue
สร้าง `TPriorityTaskQueue` ที่:
- เพิ่มงานพร้อม priority
- Dequeue ตาม priority
- ดู pending tasks

### แบบฝึกหัดที่ 15: Document Store
สร้าง simple document store ที่:
- เก็บ documents เป็น key-value
- Index fields สำหรับค้นหา
- Query ด้วย multiple conditions
- Sort results

---

## สรุป

ในบทนี้เราได้เรียนรู้ Collections หลักๆ ใน Lazarus:

1. **TList** - pointer list สำหรับ raw data
2. **TStringList** - string collection พร้อม utilities
3. **TObjectList** - object collection ที่จัดการ lifetime
4. **TCollection/TCollectionItem** - collection สำหรับ component
5. **TDictionary<K,V>** - hash map สำหรับ key-value
6. **TStack/TQueue** - LIFO/FIFO data structures
7. **Enumerators** - ทำให้ใช้ for..in loop ได้
8. **LINQ-style** - functional operations บน collections
9. **Custom Sorting** - เรียงข้อมูลด้วย comparer

### การเลือกใช้ Collection ที่เหมาะสม:
- **เก็บ strings** -> TStringList
- **เก็บ objects** -> TObjectList หรือ TList<T>
- **ค้นหาด้วย key** -> TDictionary<K,V>
- **LIFO** -> TStack<T>
- **FIFO** -> TQueue<T>
- **Component-based** -> TCollection
