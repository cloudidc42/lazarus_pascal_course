# Part 22 - Classes และ Objects (เชิงลึก)

## บทนำ

ในบทที่ 21 เราเรียนพื้นฐาน OOP ไปแล้ว บทนี้จะเจาะลึกเรื่อง Classes และ Objects อย่างละเอียด ครอบคลุม Properties ขั้นสูง, Method Overloading, Class References, RTTI, Type Checking และการจัดการ Memory

---

## 22.1 Class Declarations ทุกรูปแบบ

### รูปแบบที่ 1: Class พื้นฐาน

```pascal
type
  TSimpleClass = class
    Field: Integer;
    procedure Method;
  end;
```

### รูปแบบที่ 2: Class สืบทอด

```pascal
type
  TParent = class
    procedure ParentMethod; virtual;
  end;

  TChild = class(TParent)
    procedure ParentMethod; override;
    procedure ChildMethod;
  end;
```

### รูปแบบที่ 3: Forward Declaration

```pascal
type
  TClassA = class;  // Forward declaration

  TClassB = class
    FRefToA: TClassA;  // ใช้ A ก่อน declare A ได้
  end;

  TClassA = class     // Declaration จริง
    FRefToB: TClassB;
  end;
```

### รูปแบบที่ 4: Class with Interfaces

```pascal
type
  ISerializable = interface
    function Serialize: string;
    procedure Deserialize(const Data: string);
  end;

  TDocument = class(TObject, ISerializable)
    function Serialize: string;
    procedure Deserialize(const Data: string);
  end;
```

### รูปแบบที่ 5: Class ที่สมบูรณ์

```pascal
type
  TFullClass = class(TParentClass, IInterface1, IInterface2)
  strict private
    FStrictPrivate: Integer;
  private
    FPrivate: string;
    class var FClassVar: Integer;
  protected
    FProtected: Boolean;
    procedure ProtectedMethod; virtual;
  public
    class var PublicClassVar: string;

    constructor Create; overload;
    constructor Create(AValue: Integer); overload;
    destructor Destroy; override;

    class function ClassFactory: TFullClass;
    class procedure ClassMethod;

    procedure InstanceMethod;
    function ComputeValue: Double; virtual;

    property PrivateVal: Integer read FPrivate write FPrivate;
    property ProtectedVal: Boolean read FProtected;

  published
    property PublishedProp: string read FPrivate write FPrivate;
  end;
```

---

## 22.2 Properties ขั้นสูง

### Indexed Properties

```pascal
type
  TMatrix = class
  private
    FData: array of array of Double;
    FRows, FCols: Integer;

    function GetElement(Row, Col: Integer): Double;
    procedure SetElement(Row, Col: Integer; Value: Double);
    function GetRow(Index: Integer): string;
    function GetRowCount: Integer;
    function GetColCount: Integer;

  public
    constructor Create(Rows, Cols: Integer);

    // Indexed property สำหรับเข้าถึง element
    property Elements[Row, Col: Integer]: Double
      read GetElement write SetElement; default;

    // Indexed property แบบ 1D
    property Rows[Index: Integer]: string read GetRow;

    property RowCount: Integer read GetRowCount;
    property ColCount: Integer read GetColCount;
  end;

constructor TMatrix.Create(Rows, Cols: Integer);
var
  I: Integer;
begin
  inherited Create;
  FRows := Rows;
  FCols := Cols;
  SetLength(FData, Rows);
  for I := 0 to Rows - 1 do
    SetLength(FData[I], Cols);
end;

function TMatrix.GetElement(Row, Col: Integer): Double;
begin
  if (Row < 0) or (Row >= FRows) or (Col < 0) or (Col >= FCols) then
    raise Exception.CreateFmt('Index (%d,%d) ออกนอกขอบเขต', [Row, Col]);
  Result := FData[Row][Col];
end;

procedure TMatrix.SetElement(Row, Col: Integer; Value: Double);
begin
  if (Row < 0) or (Row >= FRows) or (Col < 0) or (Col >= FCols) then
    raise Exception.CreateFmt('Index (%d,%d) ออกนอกขอบเขต', [Row, Col]);
  FData[Row][Col] := Value;
end;

function TMatrix.GetRow(Index: Integer): string;
var
  J: Integer;
  S: string;
begin
  S := '[';
  for J := 0 to FCols - 1 do
  begin
    if J > 0 then S := S + ', ';
    S := S + FloatToStr(FData[Index][J]);
  end;
  Result := S + ']';
end;

function TMatrix.GetRowCount: Integer;
begin
  Result := FRows;
end;

function TMatrix.GetColCount: Integer;
begin
  Result := FCols;
end;

// ใช้งาน
var
  M: TMatrix;
begin
  M := TMatrix.Create(3, 3);

  // ใช้ default property
  M[0, 0] := 1;  M[0, 1] := 2;  M[0, 2] := 3;
  M[1, 0] := 4;  M[1, 1] := 5;  M[1, 2] := 6;
  M[2, 0] := 7;  M[2, 1] := 8;  M[2, 2] := 9;

  WriteLn('Element [1,1]: ', M[1, 1]);    // 5
  WriteLn('Row 0: ', M.Rows[0]);          // [1, 2, 3]

  M.Free;
end;
```

### Array Property

```pascal
type
  TStringList = class
  private
    FItems: array of string;
    FCount: Integer;

    function GetItem(Index: Integer): string;
    procedure SetItem(Index: Integer; const Value: string);
    function GetCount: Integer;

  public
    constructor Create;
    procedure Add(const Item: string);
    procedure Delete(Index: Integer);
    procedure Clear;

    property Items[Index: Integer]: string
      read GetItem write SetItem; default;
    property Count: Integer read GetCount;
  end;

function TStringList.GetItem(Index: Integer): string;
begin
  if (Index < 0) or (Index >= FCount) then
    raise ERangeError.CreateFmt('Index %d ออกนอกขอบเขต (0..%d)', [Index, FCount-1]);
  Result := FItems[Index];
end;

procedure TStringList.SetItem(Index: Integer; const Value: string);
begin
  if (Index < 0) or (Index >= FCount) then
    raise ERangeError.CreateFmt('Index %d ออกนอกขอบเขต', [Index]);
  FItems[Index] := Value;
end;

procedure TStringList.Add(const Item: string);
begin
  if FCount >= Length(FItems) then
    SetLength(FItems, Length(FItems) + 16);
  FItems[FCount] := Item;
  Inc(FCount);
end;

// ใช้งาน
var
  List: TStringList;
begin
  List := TStringList.Create;
  List.Add('สมชาย');
  List.Add('สมหญิง');
  List.Add('สมศักดิ์');

  WriteLn(List[0]);   // สมชาย
  WriteLn(List[1]);   // สมหญิง
  List[2] := 'สมบัติ'; // แก้ไขด้วย indexed property
  WriteLn(List[2]);   // สมบัติ

  List.Free;
end;
```

### Stored Property

```pascal
type
  TConfig = class
  private
    FFontSize: Integer;
    FDefaultFontSize: Integer;

    function GetFontSize: Integer;
    procedure SetFontSize(Value: Integer);
    function IsFontSizeStored: Boolean;

  public
    constructor Create;

    // Stored property - บอกว่าควร save ค่านี้หรือเปล่า
    property FontSize: Integer
      read GetFontSize write SetFontSize stored IsFontSizeStored;
  end;

function TConfig.IsFontSizeStored: Boolean;
begin
  // บันทึกเฉพาะเมื่อค่าต่างจาก default
  Result := FFontSize <> FDefaultFontSize;
end;
```

---

## 22.3 Method Overloading

Method overloading คือการมี methods หลายตัวที่ชื่อเดียวกันแต่ parameter ต่างกัน

```pascal
program MethodOverloading;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TPrinter = class
  public
    // Overloaded methods
    procedure Print(Value: Integer); overload;
    procedure Print(Value: Double); overload;
    procedure Print(const Value: string); overload;
    procedure Print(Value: Boolean); overload;
    procedure Print(Values: array of Integer); overload;

    // Overloaded constructor
    constructor Create; overload;
    constructor Create(const DeviceName: string); overload;
  end;

procedure TPrinter.Print(Value: Integer);
begin
  WriteLn('Integer: ', Value);
end;

procedure TPrinter.Print(Value: Double);
begin
  WriteLn('Double: ', Value:0:4);
end;

procedure TPrinter.Print(const Value: string);
begin
  WriteLn('String: ', Value);
end;

procedure TPrinter.Print(Value: Boolean);
begin
  WriteLn('Boolean: ', IfThen(Value, 'True', 'False'));
end;

procedure TPrinter.Print(Values: array of Integer);
var
  I: Integer;
begin
  Write('Array: [');
  for I := 0 to High(Values) do
  begin
    if I > 0 then Write(', ');
    Write(Values[I]);
  end;
  WriteLn(']');
end;

constructor TPrinter.Create;
begin
  inherited Create;
  WriteLn('Printer สร้างแบบ default');
end;

constructor TPrinter.Create(const DeviceName: string);
begin
  inherited Create;
  WriteLn('Printer สร้างสำหรับ: ', DeviceName);
end;

var
  P: TPrinter;
begin
  P := TPrinter.Create('HP LaserJet');

  P.Print(42);
  P.Print(3.14159);
  P.Print('สวัสดี');
  P.Print(True);
  P.Print([1, 2, 3, 4, 5]);

  P.Free;

  ReadLn;
end.
```

---

## 22.4 Class References (Metaclasses)

Class references ให้เราเก็บ reference ถึง class เอง (ไม่ใช่ instance) และสร้าง objects ได้ dynamic

```pascal
program ClassReferences;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TShape = class
  public
    function Area: Double; virtual; abstract;
    function Name: string; virtual; abstract;
  end;

  TCircle = class(TShape)
  private
    FRadius: Double;
  public
    constructor Create(ARadius: Double);
    function Area: Double; override;
    function Name: string; override;
  end;

  TRectangle = class(TShape)
  private
    FWidth, FHeight: Double;
  public
    constructor Create(AWidth, AHeight: Double);
    function Area: Double; override;
    function Name: string; override;
  end;

  TTriangle = class(TShape)
  private
    FBase, FHeight: Double;
  public
    constructor Create(ABase, AHeight: Double);
    function Area: Double; override;
    function Name: string; override;
  end;

  // Class reference type
  TShapeClass = class of TShape;

  // Factory ที่ใช้ class reference
  TShapeFactory = class
  public
    // สร้าง shape จาก class name
    class function CreateByName(const ClassName: string): TShape;
    // Register ทำไม่ได้ง่ายๆ แต่ demo ด้วย array
    class procedure ListShapes(ShapeClasses: array of TShapeClass);
  end;

constructor TCircle.Create(ARadius: Double);
begin
  inherited Create;
  FRadius := ARadius;
end;

function TCircle.Area: Double;
begin
  Result := Pi * FRadius * FRadius;
end;

function TCircle.Name: string;
begin
  Result := 'วงกลม';
end;

constructor TRectangle.Create(AWidth, AHeight: Double);
begin
  inherited Create;
  FWidth := AWidth;
  FHeight := AHeight;
end;

function TRectangle.Area: Double;
begin
  Result := FWidth * FHeight;
end;

function TRectangle.Name: string;
begin
  Result := 'สี่เหลี่ยม';
end;

constructor TTriangle.Create(ABase, AHeight: Double);
begin
  inherited Create;
  FBase := ABase;
  FHeight := AHeight;
end;

function TTriangle.Area: Double;
begin
  Result := 0.5 * FBase * FHeight;
end;

function TTriangle.Name: string;
begin
  Result := 'สามเหลี่ยม';
end;

class function TShapeFactory.CreateByName(const ClassName: string): TShape;
begin
  if ClassName = 'Circle' then
    Result := TCircle.Create(5.0)
  else if ClassName = 'Rectangle' then
    Result := TRectangle.Create(4.0, 6.0)
  else if ClassName = 'Triangle' then
    Result := TTriangle.Create(3.0, 8.0)
  else
    raise Exception.CreateFmt('ไม่รู้จัก class "%s"', [ClassName]);
end;

class procedure TShapeFactory.ListShapes(ShapeClasses: array of TShapeClass);
var
  ShapeClass: TShapeClass;
  Shape: TShape;
begin
  WriteLn('รายการรูปทรงที่รองรับ:');
  for ShapeClass in ShapeClasses do
  begin
    WriteLn('  ', ShapeClass.ClassName);
  end;
end;

// ใช้งาน Class References
var
  ShapeClass: TShapeClass;
  Shape: TShape;
  ShapeNames: array[0..2] of string = ('Circle', 'Rectangle', 'Triangle');
  Name: string;
begin
  // ใช้ class reference ตรงๆ
  ShapeClass := TCircle;
  WriteLn('Class: ', ShapeClass.ClassName);

  // สร้าง object ผ่าน class reference
  // (ต้องมี virtual constructor หรือใช้ factory pattern)

  // ใช้ factory
  for Name in ShapeNames do
  begin
    Shape := TShapeFactory.CreateByName(Name);
    WriteLn(Shape.Name, ': พื้นที่ = ', Shape.Area:0:4);
    Shape.Free;
  end;

  WriteLn;
  // List shape classes
  TShapeFactory.ListShapes([TCircle, TRectangle, TTriangle]);

  ReadLn;
end.
```

---

## 22.5 Object Creation Patterns

### Simple Factory

```pascal
type
  TAnimal = class
    function Speak: string; virtual; abstract;
  end;

  TDog = class(TAnimal)
    function Speak: string; override;
  end;

  TCat = class(TAnimal)
    function Speak: string; override;
  end;

  TAnimalFactory = class
    class function CreateAnimal(const Kind: string): TAnimal;
  end;

class function TAnimalFactory.CreateAnimal(const Kind: string): TAnimal;
begin
  case LowerCase(Kind) of
    'dog': Result := TDog.Create;
    'cat': Result := TCat.Create;
    else raise Exception.CreateFmt('ไม่รู้จักสัตว์ "%s"', [Kind]);
  end;
end;
```

### Builder Pattern

```pascal
type
  TPizza = class
    FSize: string;
    FCrust: string;
    FSauce: string;
    FToppings: TStringList;

    constructor Create;
    destructor Destroy; override;
    procedure Show;
  end;

  TPizzaBuilder = class
  private
    FPizza: TPizza;
  public
    constructor Create;
    destructor Destroy; override;

    function SetSize(const Size: string): TPizzaBuilder;
    function SetCrust(const Crust: string): TPizzaBuilder;
    function SetSauce(const Sauce: string): TPizzaBuilder;
    function AddTopping(const Topping: string): TPizzaBuilder;
    function Build: TPizza;
  end;

constructor TPizza.Create;
begin
  inherited;
  FToppings := TStringList.Create;
end;

destructor TPizza.Destroy;
begin
  FToppings.Free;
  inherited;
end;

procedure TPizza.Show;
var
  I: Integer;
begin
  WriteLn('=== พิซซ่า ===');
  WriteLn('ขนาด: ', FSize);
  WriteLn('ขอบ: ', FCrust);
  WriteLn('ซอส: ', FSauce);
  Write('Toppings: ');
  for I := 0 to FToppings.Count - 1 do
  begin
    if I > 0 then Write(', ');
    Write(FToppings[I]);
  end;
  WriteLn;
end;

constructor TPizzaBuilder.Create;
begin
  inherited;
  FPizza := TPizza.Create;
  // ค่า default
  FPizza.FSize := 'Medium';
  FPizza.FCrust := 'บาง';
  FPizza.FSauce := 'มะเขือเทศ';
end;

destructor TPizzaBuilder.Destroy;
begin
  // ไม่ Free FPizza ที่นี่ เพราะ Build คืนไปแล้ว
  inherited;
end;

function TPizzaBuilder.SetSize(const Size: string): TPizzaBuilder;
begin
  FPizza.FSize := Size;
  Result := Self;
end;

function TPizzaBuilder.SetCrust(const Crust: string): TPizzaBuilder;
begin
  FPizza.FCrust := Crust;
  Result := Self;
end;

function TPizzaBuilder.SetSauce(const Sauce: string): TPizzaBuilder;
begin
  FPizza.FSauce := Sauce;
  Result := Self;
end;

function TPizzaBuilder.AddTopping(const Topping: string): TPizzaBuilder;
begin
  FPizza.FToppings.Add(Topping);
  Result := Self;
end;

function TPizzaBuilder.Build: TPizza;
begin
  Result := FPizza;
  FPizza := TPizza.Create;  // สร้าง pizza ใหม่เผื่อสร้างต่อ
end;

// ใช้งาน
var
  Builder: TPizzaBuilder;
  MyPizza: TPizza;
begin
  Builder := TPizzaBuilder.Create;
  MyPizza := Builder
    .SetSize('Large')
    .SetCrust('หนา')
    .SetSauce('ครีม')
    .AddTopping('ชีส')
    .AddTopping('เห็ด')
    .AddTopping('พริก')
    .Build;

  MyPizza.Show;
  MyPizza.Free;
  Builder.Free;
end;
```

---

## 22.6 Memory Management

### Manual Memory Management

```pascal
// ต้อง Free เองเสมอ
var
  Obj: TMyClass;
begin
  Obj := TMyClass.Create;
  try
    Obj.DoSomething;
  finally
    Obj.Free;  // ต้องเรียกเสมอ แม้มี exception
  end;
end;
```

### FreeAndNil

```pascal
// FreeAndNil ลบและตั้งค่าเป็น nil ป้องกัน dangling pointer
var
  Obj: TMyClass;
begin
  Obj := TMyClass.Create;
  // ...
  FreeAndNil(Obj);  // เทียบเท่า Obj.Free; Obj := nil;
  // ตอนนี้ Obj = nil ปลอดภัยกว่า
end;
```

### Reference Counting (Interface-based)

เมื่อ class implement `IInterface` Pascal จะ reference count อัตโนมัติ

```pascal
type
  IMyInterface = interface
    ['{A1B2C3D4-E5F6-7890-ABCD-EF1234567890}']
    procedure DoWork;
  end;

  TMyClass = class(TInterfacedObject, IMyInterface)
    procedure DoWork;
  end;

// ไม่ต้อง Free! reference counting จัดการเอง
var
  Obj: IMyInterface;
begin
  Obj := TMyClass.Create;   // RefCount = 1
  Obj.DoWork;
  // เมื่อ Obj ออกนอก scope RefCount = 0 -> ลบอัตโนมัติ
end;
```

### Memory Leak Detection

```pascal
program MemoryLeakDemo;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  TLeakyClass = class
  private
    FData: PByte;
    FSize: Integer;
  public
    constructor Create(Size: Integer);
    destructor Destroy; override;
  end;

constructor TLeakyClass.Create(Size: Integer);
begin
  inherited Create;
  FSize := Size;
  GetMem(FData, Size);
  FillByte(FData^, Size, 0);
  WriteLn('จัดสรร ', Size, ' bytes');
end;

destructor TLeakyClass.Destroy;
begin
  if Assigned(FData) then
  begin
    FreeMem(FData);
    FData := nil;
    WriteLn('คืน memory ', FSize, ' bytes');
  end;
  inherited;
end;

var
  Obj1, Obj2: TLeakyClass;
begin
  Obj1 := TLeakyClass.Create(1024);  // จะ free ด้วย try/finally
  Obj2 := TLeakyClass.Create(2048);  // จะ leak ถ้าไม่ระวัง

  try
    // ทำงานกับ Obj1
    WriteLn('ใช้งาน Obj1');
  finally
    Obj1.Free;  // ปลอดภัย
  end;

  // Obj2 ต้อง free ด้วย
  Obj2.Free;

  ReadLn;
end.
```

---

## 22.7 TObject Hierarchy

```
TObject
├── TInterfacedObject    (มี reference counting)
├── TComponent           (มี Owner, Name, serialization)
│   ├── TControl         (มี visual properties)
│   │   ├── TWinControl  (มี handle บน Windows)
│   │   │   ├── TButton
│   │   │   ├── TEdit
│   │   │   └── TPanel
│   │   └── TGraphicControl (ไม่มี handle)
│   │       ├── TLabel
│   │       └── TImage
│   └── TNonVisualComponent
│       ├── TTimer
│       └── TDataSet
├── TPersistent          (มี serialization)
│   ├── TCollection
│   └── TStrings
└── TStream
    ├── TFileStream
    ├── TMemoryStream
    └── TStringStream
```

### ตัวอย่าง TObject methods

```pascal
program TObjectMethods;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TBase = class
  public
    function ToString: string; override;
    function Equals(Obj: TObject): Boolean; override;
    function GetHashCode: PtrInt; override;
  end;

  TChild = class(TBase)
  private
    FValue: Integer;
  public
    constructor Create(AValue: Integer);
    function ToString: string; override;
  end;

function TBase.ToString: string;
begin
  Result := Format('TBase@%p', [Pointer(Self)]);
end;

function TBase.Equals(Obj: TObject): Boolean;
begin
  Result := (Obj <> nil) and (Obj.ClassType = ClassType);
end;

function TBase.GetHashCode: PtrInt;
begin
  Result := PtrInt(Self);  // ใช้ address เป็น hash
end;

constructor TChild.Create(AValue: Integer);
begin
  inherited Create;
  FValue := AValue;
end;

function TChild.ToString: string;
begin
  Result := Format('TChild(Value=%d)', [FValue]);
end;

var
  B: TBase;
  C1, C2: TChild;
begin
  B := TBase.Create;
  C1 := TChild.Create(42);
  C2 := TChild.Create(100);

  // ClassName และ type info
  WriteLn('B.ClassName: ', B.ClassName);
  WriteLn('C1.ClassName: ', C1.ClassName);
  WriteLn('C1.ClassParent.ClassName: ', C1.ClassParent.ClassName);

  // InheritsFrom
  WriteLn;
  WriteLn('C1 สืบทอดจาก TBase: ', C1.InheritsFrom(TBase));
  WriteLn('C1 สืบทอดจาก TObject: ', C1.InheritsFrom(TObject));
  WriteLn('B สืบทอดจาก TChild: ', B.InheritsFrom(TChild));

  // ToString
  WriteLn;
  WriteLn('B.ToString: ', B.ToString);
  WriteLn('C1.ToString: ', C1.ToString);
  WriteLn('C2.ToString: ', C2.ToString);

  // Type checking
  WriteLn;
  var Obj: TObject := C1;
  WriteLn('Obj is TBase: ', Obj is TBase);
  WriteLn('Obj is TChild: ', Obj is TChild);
  WriteLn('Obj is TObject: ', Obj is TObject);

  B.Free;
  C1.Free;
  C2.Free;

  ReadLn;
end.
```

---

## 22.8 RTTI (Run-Time Type Information)

RTTI ช่วยให้เราดึงข้อมูลเกี่ยวกับ type ขณะ runtime

```pascal
program RTTIExample;

{$mode objfpc}{$H+}

uses
  SysUtils, TypInfo;

type
  TPersonKind = (pkStudent, pkTeacher, pkAdmin);

  TPerson = class
  private
    FName: string;
    FAge: Integer;
    FKind: TPersonKind;
  published
    property Name: string read FName write FName;
    property Age: Integer read FAge write FAge;
    property Kind: TPersonKind read FKind write FKind;
  end;

// แสดง RTTI ของ class
procedure ShowTypeInfo(AClass: TClass);
var
  TypeInfo: PTypeInfo;
  TypeData: PTypeData;
  PropList: PPropList;
  PropCount, I: Integer;
  PropInfo: PPropInfo;
begin
  TypeInfo := PTypeInfo(AClass.ClassInfo);
  if TypeInfo = nil then
  begin
    WriteLn('ไม่มี TypeInfo');
    Exit;
  end;

  TypeData := GetTypeData(TypeInfo);

  WriteLn('=== RTTI ของ ', AClass.ClassName, ' ===');
  WriteLn('ชนิด: ', TypeInfo^.Name);

  // นับ properties
  PropCount := GetPropList(TypeInfo, tkProperties, nil);
  WriteLn('จำนวน properties: ', PropCount);

  // แสดง properties ทั้งหมด
  GetMem(PropList, PropCount * SizeOf(PPropInfo));
  try
    GetPropList(TypeInfo, tkProperties, PropList);
    for I := 0 to PropCount - 1 do
    begin
      PropInfo := PropList^[I];
      WriteLn('  Property: ', PropInfo^.Name,
              ' ชนิด: ', PropInfo^.PropType^.Name);
    end;
  finally
    FreeMem(PropList);
  end;
end;

// Get/Set property ด้วย RTTI
procedure SetPropertyValue(Obj: TObject; const PropName: string; const Value: string);
var
  PropInfo: PPropInfo;
begin
  PropInfo := GetPropInfo(Obj.ClassInfo, PropName);
  if PropInfo = nil then
  begin
    WriteLn('ไม่พบ property: ', PropName);
    Exit;
  end;

  case PropInfo^.PropType^.Kind of
    tkString, tkLString, tkUString:
      SetStrProp(Obj, PropInfo, Value);
    tkInteger:
      SetOrdProp(Obj, PropInfo, StrToInt(Value));
    else
      WriteLn('ไม่รองรับชนิด: ', PropInfo^.PropType^.Name);
  end;
end;

function GetPropertyValue(Obj: TObject; const PropName: string): string;
var
  PropInfo: PPropInfo;
begin
  PropInfo := GetPropInfo(Obj.ClassInfo, PropName);
  if PropInfo = nil then
  begin
    Result := '(ไม่พบ)';
    Exit;
  end;

  case PropInfo^.PropType^.Kind of
    tkString, tkLString, tkUString:
      Result := GetStrProp(Obj, PropInfo);
    tkInteger:
      Result := IntToStr(GetOrdProp(Obj, PropInfo));
    else
      Result := '(ไม่รองรับ)';
  end;
end;

var
  P: TPerson;
begin
  ShowTypeInfo(TPerson);
  WriteLn;

  P := TPerson.Create;

  // ใช้ RTTI set properties
  SetPropertyValue(P, 'Name', 'สมชาย');
  SetPropertyValue(P, 'Age', '25');

  // ใช้ RTTI get properties
  WriteLn('Name: ', GetPropertyValue(P, 'Name'));
  WriteLn('Age: ', GetPropertyValue(P, 'Age'));

  P.Free;

  ReadLn;
end.
```

---

## 22.9 is Operator และ as Operator

### is Operator - ตรวจสอบ type

```pascal
var
  Animal: TAnimal;
  Dog: TDog;
begin
  Dog := TDog.Create('บุ๋ม', 'Labrador', 3, 10.0);
  Animal := Dog;  // upcast - ปลอดภัย

  // ตรวจสอบด้วย is
  if Animal is TDog then
    WriteLn('เป็น TDog');

  if Animal is TAnimal then
    WriteLn('เป็น TAnimal ด้วย (เพราะสืบทอด)');

  if not (Animal is TCat) then
    WriteLn('ไม่ใช่ TCat');

  Dog.Free;
end;
```

### as Operator - Safe Typecast

```pascal
var
  Animal: TAnimal;
  ActualDog: TDog;
begin
  Animal := TDog.Create('บุ๋ม', 'Poodle', 2, 5.0);

  // as จะ raise exception ถ้า cast ไม่ได้
  try
    ActualDog := Animal as TDog;   // OK
    ActualDog.Fetch;

    var Cat := Animal as TCat;      // จะ raise EInvalidCast
  except
    on E: EInvalidCast do
      WriteLn('Cast ไม่ได้: ', E.Message);
  end;

  // ตรวจก่อน cast - pattern ที่ดีกว่า
  if Animal is TDog then
  begin
    ActualDog := TDog(Animal);  // ปลอดภัยเพราะตรวจแล้ว
    ActualDog.Fetch;
  end;

  Animal.Free;
end;
```

---

## 22.10 ClassType, ClassName, InheritsFrom, ClassParent

```pascal
program TypeCheckingDemo;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TLevel1 = class
  end;

  TLevel2 = class(TLevel1)
  end;

  TLevel3 = class(TLevel2)
  end;

procedure ExploreTypeHierarchy(Obj: TObject);
begin
  WriteLn('=== ข้อมูล Type ===');
  WriteLn('ClassName: ', Obj.ClassName);
  WriteLn('ClassType: ', Obj.ClassType.ClassName);

  var Parent := Obj.ClassParent;
  Write('Hierarchy: ', Obj.ClassName);
  while Parent <> nil do
  begin
    Write(' -> ', Parent.ClassName);
    Parent := Parent.ClassParent;
  end;
  WriteLn;

  WriteLn('InheritsFrom TLevel1: ', Obj.InheritsFrom(TLevel1));
  WriteLn('InheritsFrom TLevel2: ', Obj.InheritsFrom(TLevel2));
  WriteLn('InheritsFrom TObject: ', Obj.InheritsFrom(TObject));
  WriteLn;
end;

procedure ProcessObject(Obj: TObject);
begin
  // Dispatch ตาม type
  if Obj is TLevel3 then
    WriteLn('จัดการ TLevel3')
  else if Obj is TLevel2 then
    WriteLn('จัดการ TLevel2')
  else if Obj is TLevel1 then
    WriteLn('จัดการ TLevel1')
  else
    WriteLn('จัดการ TObject');
end;

var
  L1: TLevel1;
  L2: TLevel2;
  L3: TLevel3;
begin
  L1 := TLevel1.Create;
  L2 := TLevel2.Create;
  L3 := TLevel3.Create;

  ExploreTypeHierarchy(L1);
  ExploreTypeHierarchy(L3);

  ProcessObject(L1);
  ProcessObject(L2);
  ProcessObject(L3);

  L1.Free;
  L2.Free;
  L3.Free;

  ReadLn;
end.
```

---

## 22.11 Object Serialization แนวคิด

```pascal
program SerializationConcept;

{$mode objfpc}{$H+}

uses SysUtils, Classes;

type
  // Interface สำหรับ serialization
  ISerializable = interface
    ['{12345678-1234-1234-1234-123456789ABC}']
    procedure SaveToStream(Stream: TStream);
    procedure LoadFromStream(Stream: TStream);
    function ToJSON: string;
    procedure FromJSON(const JSON: string);
  end;

  TPoint2D = class(TInterfacedObject, ISerializable)
  private
    FX, FY: Double;
  public
    constructor Create(AX, AY: Double);

    procedure SaveToStream(Stream: TStream);
    procedure LoadFromStream(Stream: TStream);
    function ToJSON: string;
    procedure FromJSON(const JSON: string);

    property X: Double read FX write FX;
    property Y: Double read FY write FY;

    function ToString: string; override;
  end;

  TLine = class(TInterfacedObject, ISerializable)
  private
    FStart, FEnd: TPoint2D;
  public
    constructor Create(X1, Y1, X2, Y2: Double);
    destructor Destroy; override;

    procedure SaveToStream(Stream: TStream);
    procedure LoadFromStream(Stream: TStream);
    function ToJSON: string;
    procedure FromJSON(const JSON: string);

    function Length: Double;
    function ToString: string; override;
  end;

constructor TPoint2D.Create(AX, AY: Double);
begin
  inherited Create;
  FX := AX;
  FY := AY;
end;

procedure TPoint2D.SaveToStream(Stream: TStream);
begin
  Stream.Write(FX, SizeOf(FX));
  Stream.Write(FY, SizeOf(FY));
end;

procedure TPoint2D.LoadFromStream(Stream: TStream);
begin
  Stream.Read(FX, SizeOf(FX));
  Stream.Read(FY, SizeOf(FY));
end;

function TPoint2D.ToJSON: string;
begin
  Result := Format('{"x":%.4f,"y":%.4f}', [FX, FY]);
end;

procedure TPoint2D.FromJSON(const JSON: string);
// ตัวอย่างง่ายๆ (ใน production ควรใช้ JSON parser จริงๆ)
var
  XStr, YStr: string;
begin
  // Parse "{\"x\":1.0,\"y\":2.0}"
  // simplified parsing
  FX := 0;
  FY := 0;
end;

function TPoint2D.ToString: string;
begin
  Result := Format('(%.2f, %.2f)', [FX, FY]);
end;

constructor TLine.Create(X1, Y1, X2, Y2: Double);
begin
  inherited Create;
  FStart := TPoint2D.Create(X1, Y1);
  FEnd := TPoint2D.Create(X2, Y2);
end;

destructor TLine.Destroy;
begin
  FStart.Free;
  FEnd.Free;
  inherited;
end;

procedure TLine.SaveToStream(Stream: TStream);
begin
  FStart.SaveToStream(Stream);
  FEnd.SaveToStream(Stream);
end;

procedure TLine.LoadFromStream(Stream: TStream);
begin
  FStart.LoadFromStream(Stream);
  FEnd.LoadFromStream(Stream);
end;

function TLine.ToJSON: string;
begin
  Result := Format('{"start":%s,"end":%s}', [FStart.ToJSON, FEnd.ToJSON]);
end;

procedure TLine.FromJSON(const JSON: string);
begin
  // Implement JSON parsing
end;

function TLine.Length: Double;
var
  DX, DY: Double;
begin
  DX := FEnd.X - FStart.X;
  DY := FEnd.Y - FStart.Y;
  Result := Sqrt(DX*DX + DY*DY);
end;

function TLine.ToString: string;
begin
  Result := Format('Line[%s -> %s]', [FStart.ToString, FEnd.ToString]);
end;

var
  P: TPoint2D;
  L: TLine;
  Stream: TMemoryStream;
  P2: TPoint2D;
begin
  P := TPoint2D.Create(3.0, 4.0);
  L := TLine.Create(0, 0, 3, 4);

  WriteLn('Point: ', P.ToString);
  WriteLn('Line: ', L.ToString);
  WriteLn('Line Length: ', L.Length:0:4);
  WriteLn;

  // JSON
  WriteLn('Point JSON: ', P.ToJSON);
  WriteLn('Line JSON: ', L.ToJSON);
  WriteLn;

  // Binary serialization
  Stream := TMemoryStream.Create;
  try
    P.SaveToStream(Stream);
    WriteLn('Saved to stream, size: ', Stream.Size, ' bytes');

    Stream.Position := 0;
    P2 := TPoint2D.Create(0, 0);
    try
      P2.LoadFromStream(Stream);
      WriteLn('Loaded from stream: ', P2.ToString);
    finally
      P2.Free;
    end;
  finally
    Stream.Free;
  end;

  P.Free;
  L.Free;

  ReadLn;
end.
```

---

## 22.12 โปรแกรมตัวอย่าง: Shape Hierarchy

```pascal
program ShapeHierarchy;

{$mode objfpc}{$H+}

uses SysUtils, Math;

type
  TColor = record
    R, G, B: Byte;
    function ToString: string;
    class function Red: TColor; static;
    class function Green: TColor; static;
    class function Blue: TColor; static;
    class function Black: TColor; static;
  end;

  TPoint = record
    X, Y: Double;
    function ToString: string;
  end;

  TShape = class
  private
    FColor: TColor;
    FName: string;
    FVisible: Boolean;
    class var FShapeCount: Integer;
  protected
    FId: Integer;
  public
    constructor Create(const AName: string; AColor: TColor);
    destructor Destroy; override;

    function Area: Double; virtual; abstract;
    function Perimeter: Double; virtual; abstract;
    procedure Draw; virtual;
    procedure Move(DX, DY: Double); virtual; abstract;
    function Clone: TShape; virtual; abstract;
    function Describe: string; virtual;

    class function GetShapeCount: Integer;

    property Color: TColor read FColor write FColor;
    property Name: string read FName;
    property Visible: Boolean read FVisible write FVisible;
    property Id: Integer read FId;
  end;

  TCircle = class(TShape)
  private
    FCenter: TPoint;
    FRadius: Double;
  public
    constructor Create(CX, CY, ARadius: Double; AColor: TColor);

    function Area: Double; override;
    function Perimeter: Double; override;
    procedure Move(DX, DY: Double); override;
    function Clone: TShape; override;
    function Describe: string; override;

    function ContainsPoint(PX, PY: Double): Boolean;

    property Center: TPoint read FCenter;
    property Radius: Double read FRadius write FRadius;
  end;

  TRectangle = class(TShape)
  private
    FTopLeft: TPoint;
    FWidth, FHeight: Double;
  public
    constructor Create(X, Y, AWidth, AHeight: Double; AColor: TColor);

    function Area: Double; override;
    function Perimeter: Double; override;
    procedure Move(DX, DY: Double); override;
    function Clone: TShape; override;
    function Describe: string; override;

    function IsSquare: Boolean;

    property TopLeft: TPoint read FTopLeft;
    property Width: Double read FWidth write FWidth;
    property Height: Double read FHeight write FHeight;
  end;

  TTriangle = class(TShape)
  private
    FA, FB, FC: TPoint;
  public
    constructor Create(AX, AY, BX, BY, CX, CY: Double; AColor: TColor);

    function Area: Double; override;
    function Perimeter: Double; override;
    procedure Move(DX, DY: Double); override;
    function Clone: TShape; override;
    function Describe: string; override;

    function IsEquilateral: Boolean;
  end;

  // Collection ของ shapes
  TShapeCollection = class
  private
    FShapes: array of TShape;
    FCount: Integer;
    FOwnShapes: Boolean;
  public
    constructor Create(AOwnShapes: Boolean = True);
    destructor Destroy; override;

    procedure Add(Shape: TShape);
    procedure Remove(Index: Integer);
    procedure Clear;
    function TotalArea: Double;
    function TotalPerimeter: Double;
    procedure DrawAll;
    procedure ShowStats;

    property Count: Integer read FCount;
    property Shapes[Index: Integer]: TShape read GetShape; default;
  private
    function GetShape(Index: Integer): TShape;
  end;

// ============= TColor =============
function TColor.ToString: string;
begin
  Result := Format('RGB(%d,%d,%d)', [R, G, B]);
end;

class function TColor.Red: TColor;
begin
  Result.R := 255; Result.G := 0; Result.B := 0;
end;

class function TColor.Green: TColor;
begin
  Result.R := 0; Result.G := 255; Result.B := 0;
end;

class function TColor.Blue: TColor;
begin
  Result.R := 0; Result.G := 0; Result.B := 255;
end;

class function TColor.Black: TColor;
begin
  Result.R := 0; Result.G := 0; Result.B := 0;
end;

// ============= TPoint =============
function TPoint.ToString: string;
begin
  Result := Format('(%.1f,%.1f)', [X, Y]);
end;

// ============= TShape =============
constructor TShape.Create(const AName: string; AColor: TColor);
begin
  inherited Create;
  FName := AName;
  FColor := AColor;
  FVisible := True;
  Inc(FShapeCount);
  FId := FShapeCount;
end;

destructor TShape.Destroy;
begin
  Dec(FShapeCount);
  inherited;
end;

procedure TShape.Draw;
begin
  if FVisible then
    WriteLn('วาด ', FName, ' #', FId, ' สี ', FColor.ToString, ' พื้นที่=', Area:0:2);
end;

function TShape.Describe: string;
begin
  Result := Format('%s #%d: Area=%.2f, Perimeter=%.2f',
    [FName, FId, Area, Perimeter]);
end;

class function TShape.GetShapeCount: Integer;
begin
  Result := FShapeCount;
end;

// ============= TCircle =============
constructor TCircle.Create(CX, CY, ARadius: Double; AColor: TColor);
begin
  inherited Create('วงกลม', AColor);
  FCenter.X := CX;
  FCenter.Y := CY;
  FRadius := ARadius;
end;

function TCircle.Area: Double;
begin
  Result := Pi * FRadius * FRadius;
end;

function TCircle.Perimeter: Double;
begin
  Result := 2 * Pi * FRadius;
end;

procedure TCircle.Move(DX, DY: Double);
begin
  FCenter.X := FCenter.X + DX;
  FCenter.Y := FCenter.Y + DY;
end;

function TCircle.Clone: TShape;
begin
  Result := TCircle.Create(FCenter.X, FCenter.Y, FRadius, FColor);
end;

function TCircle.Describe: string;
begin
  Result := Format('วงกลม #%d: ศูนย์กลาง=%s, รัศมี=%.2f, พื้นที่=%.2f, เส้นรอบวง=%.2f',
    [FId, FCenter.ToString, FRadius, Area, Perimeter]);
end;

function TCircle.ContainsPoint(PX, PY: Double): Boolean;
var
  DX, DY: Double;
begin
  DX := PX - FCenter.X;
  DY := PY - FCenter.Y;
  Result := Sqrt(DX*DX + DY*DY) <= FRadius;
end;

// ============= TRectangle =============
constructor TRectangle.Create(X, Y, AWidth, AHeight: Double; AColor: TColor);
begin
  inherited Create('สี่เหลี่ยม', AColor);
  FTopLeft.X := X;
  FTopLeft.Y := Y;
  FWidth := AWidth;
  FHeight := AHeight;
end;

function TRectangle.Area: Double;
begin
  Result := FWidth * FHeight;
end;

function TRectangle.Perimeter: Double;
begin
  Result := 2 * (FWidth + FHeight);
end;

procedure TRectangle.Move(DX, DY: Double);
begin
  FTopLeft.X := FTopLeft.X + DX;
  FTopLeft.Y := FTopLeft.Y + DY;
end;

function TRectangle.Clone: TShape;
begin
  Result := TRectangle.Create(FTopLeft.X, FTopLeft.Y, FWidth, FHeight, FColor);
end;

function TRectangle.Describe: string;
begin
  Result := Format('สี่เหลี่ยม #%d: ตำแหน่ง=%s, กว้าง=%.2f, สูง=%.2f, พื้นที่=%.2f',
    [FId, FTopLeft.ToString, FWidth, FHeight, Area]);
end;

function TRectangle.IsSquare: Boolean;
begin
  Result := Abs(FWidth - FHeight) < 0.001;
end;

// ============= TTriangle =============
constructor TTriangle.Create(AX, AY, BX, BY, CX, CY: Double; AColor: TColor);
begin
  inherited Create('สามเหลี่ยม', AColor);
  FA.X := AX; FA.Y := AY;
  FB.X := BX; FB.Y := BY;
  FC.X := CX; FC.Y := CY;
end;

function TTriangle.Area: Double;
begin
  // Shoelace formula
  Result := Abs((FB.X - FA.X) * (FC.Y - FA.Y) -
                (FC.X - FA.X) * (FB.Y - FA.Y)) / 2;
end;

function TTriangle.Perimeter: Double;
var
  AB, BC, CA: Double;
begin
  AB := Sqrt(Sqr(FB.X-FA.X) + Sqr(FB.Y-FA.Y));
  BC := Sqrt(Sqr(FC.X-FB.X) + Sqr(FC.Y-FB.Y));
  CA := Sqrt(Sqr(FA.X-FC.X) + Sqr(FA.Y-FC.Y));
  Result := AB + BC + CA;
end;

procedure TTriangle.Move(DX, DY: Double);
begin
  FA.X := FA.X + DX; FA.Y := FA.Y + DY;
  FB.X := FB.X + DX; FB.Y := FB.Y + DY;
  FC.X := FC.X + DX; FC.Y := FC.Y + DY;
end;

function TTriangle.Clone: TShape;
begin
  Result := TTriangle.Create(FA.X, FA.Y, FB.X, FB.Y, FC.X, FC.Y, FColor);
end;

function TTriangle.Describe: string;
begin
  Result := Format('สามเหลี่ยม #%d: A=%s, B=%s, C=%s, พื้นที่=%.2f',
    [FId, FA.ToString, FB.ToString, FC.ToString, Area]);
end;

function TTriangle.IsEquilateral: Boolean;
var
  AB, BC, CA: Double;
begin
  AB := Sqrt(Sqr(FB.X-FA.X) + Sqr(FB.Y-FA.Y));
  BC := Sqrt(Sqr(FC.X-FB.X) + Sqr(FC.Y-FB.Y));
  CA := Sqrt(Sqr(FA.X-FC.X) + Sqr(FA.Y-FC.Y));
  Result := (Abs(AB-BC) < 0.001) and (Abs(BC-CA) < 0.001);
end;

// ============= TShapeCollection =============
constructor TShapeCollection.Create(AOwnShapes: Boolean);
begin
  inherited Create;
  FOwnShapes := AOwnShapes;
  FCount := 0;
  SetLength(FShapes, 16);
end;

destructor TShapeCollection.Destroy;
begin
  if FOwnShapes then
    Clear;
  inherited;
end;

procedure TShapeCollection.Add(Shape: TShape);
begin
  if FCount >= Length(FShapes) then
    SetLength(FShapes, Length(FShapes) * 2);
  FShapes[FCount] := Shape;
  Inc(FCount);
end;

procedure TShapeCollection.Remove(Index: Integer);
var
  I: Integer;
begin
  if (Index < 0) or (Index >= FCount) then Exit;
  if FOwnShapes then
    FShapes[Index].Free;
  for I := Index to FCount - 2 do
    FShapes[I] := FShapes[I + 1];
  Dec(FCount);
end;

procedure TShapeCollection.Clear;
var
  I: Integer;
begin
  if FOwnShapes then
    for I := 0 to FCount - 1 do
      FShapes[I].Free;
  FCount := 0;
end;

function TShapeCollection.TotalArea: Double;
var
  I: Integer;
begin
  Result := 0;
  for I := 0 to FCount - 1 do
    Result := Result + FShapes[I].Area;
end;

function TShapeCollection.TotalPerimeter: Double;
var
  I: Integer;
begin
  Result := 0;
  for I := 0 to FCount - 1 do
    Result := Result + FShapes[I].Perimeter;
end;

procedure TShapeCollection.DrawAll;
var
  I: Integer;
begin
  for I := 0 to FCount - 1 do
    FShapes[I].Draw;
end;

procedure TShapeCollection.ShowStats;
var
  I: Integer;
begin
  WriteLn('=== สถิติรูปทรงทั้งหมด ===');
  WriteLn('จำนวน: ', FCount, ' รูป');
  WriteLn('พื้นที่รวม: ', TotalArea:0:4);
  WriteLn('เส้นรอบวงรวม: ', TotalPerimeter:0:4);
  WriteLn;
  for I := 0 to FCount - 1 do
    WriteLn(FShapes[I].Describe);
end;

function TShapeCollection.GetShape(Index: Integer): TShape;
begin
  if (Index < 0) or (Index >= FCount) then
    raise ERangeError.CreateFmt('Index %d ออกนอกขอบเขต', [Index]);
  Result := FShapes[Index];
end;

// ============= Main Program =============
var
  Collection: TShapeCollection;
  Circle: TCircle;
  Rect: TRectangle;
  Tri: TTriangle;
begin
  WriteLn('===== Shape Hierarchy System =====');
  WriteLn;

  Collection := TShapeCollection.Create;
  try
    Circle := TCircle.Create(5, 5, 3, TColor.Red);
    Rect := TRectangle.Create(0, 0, 6, 4, TColor.Blue);
    Tri := TTriangle.Create(0, 0, 5, 0, 2.5, 4, TColor.Green);

    Collection.Add(Circle);
    Collection.Add(Rect);
    Collection.Add(Tri);

    // เพิ่มอีก
    Collection.Add(TCircle.Create(10, 10, 2, TColor.Green));
    Collection.Add(TRectangle.Create(1, 1, 5, 5, TColor.Red));

    Collection.DrawAll;
    WriteLn;
    Collection.ShowStats;
    WriteLn;

    // ตรวจสอบ type
    WriteLn('--- Type Checking ---');
    WriteLn('Collection[0] เป็น TCircle: ', Collection[0] is TCircle);
    WriteLn('Collection[1] เป็น TRectangle: ', Collection[1] is TRectangle);

    // Clone
    WriteLn;
    WriteLn('--- Cloning ---');
    var ClonedCircle := Circle.Clone;
    WriteLn('Clone: ', ClonedCircle.Describe);
    ClonedCircle.Move(10, 10);
    WriteLn('หลัง Move: ', ClonedCircle.Describe);
    ClonedCircle.Free;

    // Special properties
    WriteLn;
    WriteLn('--- Special Properties ---');
    WriteLn('วงกลมมีจุด (5,5): ', Circle.ContainsPoint(5, 5));
    WriteLn('วงกลมมีจุด (9,9): ', Circle.ContainsPoint(9, 9));
    WriteLn('สี่เหลี่ยมจัตุรัส: ', Rect.IsSquare);
    WriteLn('สามเหลี่ยมด้านเท่า: ', Tri.IsEquilateral);

    WriteLn;
    WriteLn('จำนวน shapes ทั้งหมดที่สร้าง: ', TShape.GetShapeCount);
  finally
    Collection.Free;
  end;

  ReadLn;
end.
```

---

## 22.13 โปรแกรมตัวอย่าง: Employee Management System

```pascal
program EmployeeManagement;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TDepartment = (dHR, dIT, dFinance, dMarketing, dOperations);
  TEmployeeStatus = (esActive, esOnLeave, esTerminated);

  TEmployee = class
  private
    FId: Integer;
    FName: string;
    FDepartment: TDepartment;
    FBaseSalary: Double;
    FStatus: TEmployeeStatus;
    class var FNextId: Integer;

    function GetDepartmentName: string;
    function GetStatusName: string;

  public
    constructor Create(const AName: string; ADept: TDepartment; ABaseSalary: Double);

    function CalculateSalary: Double; virtual;
    procedure ShowInfo; virtual;
    function ToString: string; override;

    property Id: Integer read FId;
    property Name: string read FName write FName;
    property Department: TDepartment read FDepartment write FDepartment;
    property BaseSalary: Double read FBaseSalary write FBaseSalary;
    property Status: TEmployeeStatus read FStatus write FStatus;
    property DepartmentName: string read GetDepartmentName;
    property StatusName: string read GetStatusName;
  end;

  TFullTimeEmployee = class(TEmployee)
  private
    FHoursWorked: Double;
    FOvertimeRate: Double;
    FBenefits: Double;
  public
    constructor Create(const AName: string; ADept: TDepartment;
                      ABaseSalary: Double; ABenefits: Double);

    function CalculateSalary: Double; override;
    procedure ShowInfo; override;
    procedure RecordOvertime(Hours: Double);

    property HoursWorked: Double read FHoursWorked write FHoursWorked;
    property Benefits: Double read FBenefits write FBenefits;
  end;

  TPartTimeEmployee = class(TEmployee)
  private
    FHoursWorked: Double;
    FHourlyRate: Double;
  public
    constructor Create(const AName: string; ADept: TDepartment;
                      AHourlyRate: Double);

    function CalculateSalary: Double; override;
    procedure ShowInfo; override;
    procedure RecordHours(Hours: Double);

    property HoursWorked: Double read FHoursWorked write FHoursWorked;
    property HourlyRate: Double read FHourlyRate write FHourlyRate;
  end;

  TManager = class(TFullTimeEmployee)
  private
    FBonus: Double;
    FTeamSize: Integer;
    FTeamPerformance: Double;
  public
    constructor Create(const AName: string; ADept: TDepartment;
                      ABaseSalary, ABenefits: Double);

    function CalculateSalary: Double; override;
    procedure ShowInfo; override;
    procedure SetPerformance(ATeamSize: Integer; APerformance: Double);

    property Bonus: Double read FBonus write FBonus;
    property TeamSize: Integer read FTeamSize;
    property TeamPerformance: Double read FTeamPerformance;
  end;

  TEmployeeList = class
  private
    FEmployees: array of TEmployee;
    FCount: Integer;
  public
    constructor Create;
    destructor Destroy; override;
    procedure Add(Emp: TEmployee);
    function FindById(AId: Integer): TEmployee;
    function FindByName(const AName: string): TEmployee;
    procedure ShowPayroll;
    procedure ShowByDepartment(ADept: TDepartment);
    function GetTotalSalaryCost: Double;
    property Count: Integer read FCount;
  end;

// ============= TEmployee =============
const
  DeptNames: array[TDepartment] of string = ('HR', 'IT', 'การเงิน', 'การตลาด', 'ปฏิบัติการ');
  StatusNames: array[TEmployeeStatus] of string = ('ปฏิบัติงาน', 'ลา', 'สิ้นสุดสัญญา');

constructor TEmployee.Create(const AName: string; ADept: TDepartment; ABaseSalary: Double);
begin
  inherited Create;
  Inc(FNextId);
  FId := FNextId;
  FName := AName;
  FDepartment := ADept;
  FBaseSalary := ABaseSalary;
  FStatus := esActive;
end;

function TEmployee.GetDepartmentName: string;
begin
  Result := DeptNames[FDepartment];
end;

function TEmployee.GetStatusName: string;
begin
  Result := StatusNames[FStatus];
end;

function TEmployee.CalculateSalary: Double;
begin
  Result := FBaseSalary;
end;

procedure TEmployee.ShowInfo;
begin
  WriteLn('รหัส: ', FId, ' | ชื่อ: ', FName);
  WriteLn('แผนก: ', DepartmentName, ' | สถานะ: ', StatusName);
  WriteLn('เงินเดือนพื้นฐาน: ', FBaseSalary:0:2, ' บาท');
  WriteLn('เงินเดือนสุทธิ: ', CalculateSalary:0:2, ' บาท');
end;

function TEmployee.ToString: string;
begin
  Result := Format('[%d] %s (%s)', [FId, FName, DepartmentName]);
end;

// ============= TFullTimeEmployee =============
constructor TFullTimeEmployee.Create(const AName: string; ADept: TDepartment;
                                    ABaseSalary: Double; ABenefits: Double);
begin
  inherited Create(AName, ADept, ABaseSalary);
  FBenefits := ABenefits;
  FHoursWorked := 160;  // ชั่วโมงมาตรฐาน/เดือน
  FOvertimeRate := 1.5;
end;

function TFullTimeEmployee.CalculateSalary: Double;
var
  OvertimeHours, OvertimePay, HourlyRate: Double;
begin
  HourlyRate := FBaseSalary / 160;
  OvertimeHours := Max(0, FHoursWorked - 160);
  OvertimePay := OvertimeHours * HourlyRate * FOvertimeRate;
  Result := FBaseSalary + OvertimePay + FBenefits;
end;

procedure TFullTimeEmployee.ShowInfo;
begin
  inherited ShowInfo;
  WriteLn('ชั่วโมงทำงาน: ', FHoursWorked:0:1, ' ชม.');
  WriteLn('สวัสดิการ: ', FBenefits:0:2, ' บาท');
end;

procedure TFullTimeEmployee.RecordOvertime(Hours: Double);
begin
  FHoursWorked := FHoursWorked + Hours;
end;

// ============= TPartTimeEmployee =============
constructor TPartTimeEmployee.Create(const AName: string; ADept: TDepartment;
                                    AHourlyRate: Double);
begin
  inherited Create(AName, ADept, 0);
  FHourlyRate := AHourlyRate;
  FHoursWorked := 0;
end;

function TPartTimeEmployee.CalculateSalary: Double;
begin
  Result := FHoursWorked * FHourlyRate;
end;

procedure TPartTimeEmployee.ShowInfo;
begin
  WriteLn('รหัส: ', Id, ' | ชื่อ: ', Name, ' (พนักงานพาร์ทไทม์)');
  WriteLn('แผนก: ', DepartmentName);
  WriteLn('อัตรา: ', FHourlyRate:0:2, ' บาท/ชม. | ชั่วโมงทำงาน: ', FHoursWorked:0:1);
  WriteLn('เงินได้: ', CalculateSalary:0:2, ' บาท');
end;

procedure TPartTimeEmployee.RecordHours(Hours: Double);
begin
  FHoursWorked := FHoursWorked + Hours;
end;

// ============= TManager =============
constructor TManager.Create(const AName: string; ADept: TDepartment;
                           ABaseSalary, ABenefits: Double);
begin
  inherited Create(AName, ADept, ABaseSalary, ABenefits);
  FBonus := 0;
  FTeamSize := 0;
  FTeamPerformance := 0;
end;

procedure TManager.SetPerformance(ATeamSize: Integer; APerformance: Double);
begin
  FTeamSize := ATeamSize;
  FTeamPerformance := APerformance;
  // คำนวณ bonus ตาม performance
  FBonus := FBaseSalary * (APerformance / 100) * 0.5;
end;

function TManager.CalculateSalary: Double;
begin
  Result := inherited CalculateSalary + FBonus;
end;

procedure TManager.ShowInfo;
begin
  inherited ShowInfo;
  WriteLn('โบนัส: ', FBonus:0:2, ' บาท');
  WriteLn('ทีมงาน: ', FTeamSize, ' คน | ผลงานทีม: ', FTeamPerformance:0:1, '%');
end;

// ============= TEmployeeList =============
constructor TEmployeeList.Create;
begin
  inherited;
  FCount := 0;
  SetLength(FEmployees, 10);
end;

destructor TEmployeeList.Destroy;
var
  I: Integer;
begin
  for I := 0 to FCount - 1 do
    FEmployees[I].Free;
  inherited;
end;

procedure TEmployeeList.Add(Emp: TEmployee);
begin
  if FCount >= Length(FEmployees) then
    SetLength(FEmployees, Length(FEmployees) * 2);
  FEmployees[FCount] := Emp;
  Inc(FCount);
end;

function TEmployeeList.FindById(AId: Integer): TEmployee;
var
  I: Integer;
begin
  Result := nil;
  for I := 0 to FCount - 1 do
    if FEmployees[I].Id = AId then
    begin
      Result := FEmployees[I];
      Exit;
    end;
end;

function TEmployeeList.FindByName(const AName: string): TEmployee;
var
  I: Integer;
begin
  Result := nil;
  for I := 0 to FCount - 1 do
    if SameText(FEmployees[I].Name, AName) then
    begin
      Result := FEmployees[I];
      Exit;
    end;
end;

procedure TEmployeeList.ShowPayroll;
var
  I: Integer;
  Total: Double;
begin
  WriteLn('========================================');
  WriteLn('           สรุปการจ่ายเงินเดือน         ');
  WriteLn('========================================');
  Total := 0;
  for I := 0 to FCount - 1 do
  begin
    var Salary := FEmployees[I].CalculateSalary;
    WriteLn(FEmployees[I].ToString, ': ', Salary:0:2, ' บาท');
    Total := Total + Salary;
  end;
  WriteLn('----------------------------------------');
  WriteLn('รวม: ', Total:0:2, ' บาท');
  WriteLn('========================================');
end;

procedure TEmployeeList.ShowByDepartment(ADept: TDepartment);
var
  I: Integer;
begin
  WriteLn('--- แผนก ', DeptNames[ADept], ' ---');
  for I := 0 to FCount - 1 do
    if FEmployees[I].Department = ADept then
      WriteLn('  ', FEmployees[I].ToString);
end;

function TEmployeeList.GetTotalSalaryCost: Double;
var
  I: Integer;
begin
  Result := 0;
  for I := 0 to FCount - 1 do
    Result := Result + FEmployees[I].CalculateSalary;
end;

// ============= Main Program =============
var
  Employees: TEmployeeList;
  FT1, FT2: TFullTimeEmployee;
  PT1: TPartTimeEmployee;
  Mgr1, Mgr2: TManager;
begin
  WriteLn('===== ระบบจัดการพนักงาน =====');
  WriteLn;

  Employees := TEmployeeList.Create;

  // สร้างพนักงาน
  Mgr1 := TManager.Create('นาย ก. ผู้จัดการ', dIT, 80000, 5000);
  Mgr1.SetPerformance(5, 95);
  Mgr1.RecordOvertime(20);

  FT1 := TFullTimeEmployee.Create('นาย ข. โปรแกรมเมอร์', dIT, 45000, 3000);
  FT1.RecordOvertime(15);

  FT2 := TFullTimeEmployee.Create('นางสาว ค. นักบัญชี', dFinance, 40000, 2500);

  PT1 := TPartTimeEmployee.Create('นาย ง. ฟรีแลนซ์', dIT, 350);
  PT1.RecordHours(80);

  Mgr2 := TManager.Create('นางสาว จ. ผู้จัดการ HR', dHR, 70000, 4500);
  Mgr2.SetPerformance(3, 88);

  Employees.Add(Mgr1);
  Employees.Add(FT1);
  Employees.Add(FT2);
  Employees.Add(PT1);
  Employees.Add(Mgr2);

  // แสดงรายละเอียดพนักงาน
  WriteLn('--- รายละเอียดพนักงาน ---');
  WriteLn;
  Mgr1.ShowInfo;
  WriteLn;
  FT1.ShowInfo;
  WriteLn;
  PT1.ShowInfo;
  WriteLn;

  // แสดงเงินเดือน
  Employees.ShowPayroll;
  WriteLn;

  // แสดงตามแผนก
  Employees.ShowByDepartment(dIT);
  WriteLn;

  // ค้นหา
  var Found := Employees.FindByName('นาย ข. โปรแกรมเมอร์');
  if Found <> nil then
    WriteLn('พบพนักงาน: ', Found.ToString);

  Employees.Free;
  ReadLn;
end.
```

---

## 22.14 แบบฝึกหัด 20 ข้อ

**ข้อ 1:** สร้าง class `TComplexNumber` สำหรับเลขจำนวนเชิงซ้อน พร้อม properties Real/Imaginary และ methods บวก ลบ คูณ หาร

**ข้อ 2:** สร้าง class `TBinaryTree` สำหรับ binary search tree พร้อม Insert, Search, Delete และ InOrderTraversal

**ข้อ 3:** สร้าง class `TPriorityQueue` พร้อม Enqueue (ด้วย priority), Dequeue (item ที่ priority สูงสุด) และ Peek

**ข้อ 4:** ใช้ RTTI สร้าง function `CloneObject` ที่ copy ทุก published property จาก object หนึ่งไปยังอีก object

**ข้อ 5:** สร้าง `TSmartPtr<T>` class ที่ทำ automatic memory management สำหรับ TObject

**ข้อ 6:** สร้าง class `TStatePattern` ตาม State design pattern สำหรับ traffic light (Red/Yellow/Green)

**ข้อ 7:** สร้าง class `TEventBus` ที่ subscribe/publish events ด้วย type-safe

**ข้อ 8:** ออกแบบ class hierarchy สำหรับ Database connections (TDBConnection -> TMySQL, TPostgres, TSQLite)

**ข้อ 9:** สร้าง class `TExpressionTree` ที่ represent และ evaluate arithmetic expressions

**ข้อ 10:** สร้าง class `TChessBoard` และ `TChessPiece` hierarchy (King, Queen, Bishop, Knight, Rook, Pawn)

**ข้อ 11:** สร้าง class `TLRUCache` ที่ Least Recently Used cache พร้อม Put/Get/Eviction

**ข้อ 12:** สร้าง class `TDecoratorBase` และ concrete decorators ตาม Decorator pattern สำหรับ text formatting

**ข้อ 13:** สร้าง indexed property `Items` สำหรับ class `THashMap<K,V>` ที่ implement hash table

**ข้อ 14:** ใช้ class reference สร้าง `TPluginManager` ที่ register และ instantiate plugins ด้วย class reference

**ข้อ 15:** สร้าง class `TUndoableList` ที่ support undo/redo operations บน list

**ข้อ 16:** สร้าง class `TThreadSafeQueue` ที่ thread-safe queue ด้วย critical section

**ข้อ 17:** ออกแบบ class hierarchy สำหรับ Financial instruments (TInstrument -> TStock, TBond, TOption)

**ข้อ 18:** สร้าง class `TPatternMatcher` ที่ support wildcard matching (`*` และ `?`)

**ข้อ 19:** สร้าง class `TGraphTraversal` สำหรับ graph DFS/BFS traversal พร้อม path finding

**ข้อ 20:** สร้าง complete mini-ORM ที่ map class properties กับ database columns ผ่าน RTTI

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Class Declarations** ทุกรูปแบบ รวมถึง forward declaration และ interface implementation
2. **Properties ขั้นสูง** - indexed properties, array properties, stored properties
3. **Method Overloading** - หลาย signatures ชื่อเดียวกัน
4. **Class References (Metaclasses)** - เก็บ reference ถึง class เอง
5. **Object Creation Patterns** - Factory, Builder, Singleton
6. **Memory Management** - manual, FreeAndNil, reference counting
7. **TObject hierarchy** และ methods ที่สำคัญ
8. **RTTI** - ดึงข้อมูล type ขณะ runtime
9. **is/as operators** - type checking และ safe casting
10. **Object Serialization** - แนวคิดการบันทึก/โหลด objects

บทต่อไปจะเรียนเรื่อง Inheritance เชิงลึก
