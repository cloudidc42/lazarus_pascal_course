# Part 25 - Interfaces

## บทนำ

Interface คือ "สัญญา" ที่กำหนดว่า class ต้องมี methods อะไรบ้าง โดยไม่กำหนด implementation ทำให้ class ที่ไม่เกี่ยวข้องกันสามารถใช้งานร่วมกันได้ผ่าน interface เดียวกัน

Interface ใน Free Pascal สืบทอดมาจาก Delphi และ compatible กับ COM (Component Object Model) ของ Windows

---

## 25.1 Interface Declaration

### รูปแบบ Interface

```pascal
type
  IMyInterface = interface
    ['{GUID-STRING}']  // Optional แต่แนะนำ
    procedure Method1;
    function Method2(Param: Integer): string;
    property Prop1: Integer read GetProp1;
  end;
```

### Interface พื้นฐาน

```pascal
program InterfaceBasics;

{$mode delphi}  // หรือ {$mode objfpc}

uses SysUtils;

type
  // Interface สำหรับสิ่งที่ทักทายได้
  IGreetable = interface
    ['{A1B2C3D4-E5F6-7890-ABCD-EF1234567890}']
    function Greet: string;
    function GetName: string;
    property Name: string read GetName;
  end;

  // Interface สำหรับสิ่งที่คำนวณได้
  ICalculable = interface
    ['{11223344-5566-7788-99AA-BBCCDDEEFF00}']
    function Calculate(A, B: Double): Double;
    function GetOperationName: string;
  end;

  // Class ที่ implement IGreetable
  TFriendlyPerson = class(TInterfacedObject, IGreetable)
  private
    FName: string;
    FLanguage: string;
  public
    constructor Create(const AName, ALanguage: string);
    function Greet: string;
    function GetName: string;
  end;

  // Class ที่ implement ทั้ง IGreetable และ ICalculable
  TSmartPerson = class(TInterfacedObject, IGreetable, ICalculable)
  private
    FName: string;
    FSpecialty: string;
  public
    constructor Create(const AName, ASpecialty: string);
    // IGreetable
    function Greet: string;
    function GetName: string;
    // ICalculable
    function Calculate(A, B: Double): Double;
    function GetOperationName: string;
  end;

constructor TFriendlyPerson.Create(const AName, ALanguage: string);
begin
  inherited Create;
  FName := AName;
  FLanguage := ALanguage;
end;

function TFriendlyPerson.Greet: string;
begin
  case FLanguage of
    'Thai': Result := 'สวัสดี! ฉันชื่อ ' + FName;
    'English': Result := 'Hello! I am ' + FName;
    'Japanese': Result := 'こんにちは! ' + FName + 'です';
    else Result := 'Hi, ' + FName;
  end;
end;

function TFriendlyPerson.GetName: string;
begin
  Result := FName;
end;

constructor TSmartPerson.Create(const AName, ASpecialty: string);
begin
  inherited Create;
  FName := AName;
  FSpecialty := ASpecialty;
end;

function TSmartPerson.Greet: string;
begin
  Result := Format('สวัสดี! ฉัน%s เชี่ยวชาญด้าน%s', [FName, FSpecialty]);
end;

function TSmartPerson.GetName: string;
begin
  Result := FName;
end;

function TSmartPerson.Calculate(A, B: Double): Double;
begin
  Result := A + B;  // Default: บวก
end;

function TSmartPerson.GetOperationName: string;
begin
  Result := 'Addition';
end;

// Function ที่รับ IGreetable - polymorphic
procedure GreetAll(Greeters: array of IGreetable);
var
  G: IGreetable;
begin
  for G in Greeters do
    WriteLn(G.Greet);
end;

var
  P1: TFriendlyPerson;
  P2: TSmartPerson;
  Greeters: array of IGreetable;
  Calc: ICalculable;
begin
  WriteLn('=== Interface Basics ===');
  WriteLn;

  P1 := TFriendlyPerson.Create('สมชาย', 'Thai');
  P2 := TSmartPerson.Create('นาทา', 'วิทยาศาสตร์');

  SetLength(Greeters, 2);
  Greeters[0] := P1;
  Greeters[1] := P2;

  WriteLn('--- ทักทายทุกคน ---');
  GreetAll(Greeters);
  WriteLn;

  // ใช้ ICalculable
  Calc := P2;  // TSmartPerson implement ICalculable
  WriteLn('การคำนวณ: ', Calc.GetOperationName);
  WriteLn('ผลลัพธ์: ', Calc.Calculate(10, 25):0:2);

  // TInterfacedObject จัดการ memory ด้วย reference counting
  // เมื่อ P1, P2 ออกจาก scope จะถูกลบอัตโนมัติถ้าไม่มี reference แล้ว

  ReadLn;
end.
```

---

## 25.2 Interface Implementation

### การ Implement Interface

Class ต้อง implement ทุก methods ที่ interface ประกาศ

```pascal
type
  IShape = interface
    function Area: Double;
    function Perimeter: Double;
    procedure Draw;
  end;

  // ต้อง implement Area, Perimeter, Draw ทุกตัว
  TCircle = class(TInterfacedObject, IShape)
  private
    FRadius: Double;
  public
    constructor Create(ARadius: Double);
    function Area: Double;          // จาก IShape
    function Perimeter: Double;     // จาก IShape
    procedure Draw;                 // จาก IShape
    // Method เพิ่มเติมของ class เอง
    property Radius: Double read FRadius;
  end;
```

### Multiple Interface Implementation

```pascal
type
  ISerializable = interface
    function Serialize: string;
    procedure Deserialize(const Data: string);
  end;

  IComparable = interface
    function CompareTo(Other: TObject): Integer;
    function Equals(Other: TObject): Boolean;
  end;

  ICloneable = interface
    function Clone: TObject;
  end;

  // Implement หลาย interfaces
  TRecord = class(TInterfacedObject, ISerializable, IComparable, ICloneable)
  private
    FId: Integer;
    FName: string;
  public
    constructor Create(AId: Integer; const AName: string);
    // ISerializable
    function Serialize: string;
    procedure Deserialize(const Data: string);
    // IComparable
    function CompareTo(Other: TObject): Integer;
    function Equals(Other: TObject): Boolean;
    // ICloneable
    function Clone: TObject;

    property Id: Integer read FId;
    property Name: string read FName;
  end;
```

---

## 25.3 Interface Reference Counting

เมื่อ class สืบทอดจาก `TInterfacedObject` จะมี reference counting อัตโนมัติ

```pascal
program ReferenceCountingDemo;

{$mode objfpc}{$H+}

uses SysUtils;

type
  ITracked = interface
    function GetId: Integer;
    property Id: Integer read GetId;
  end;

  TTrackedObject = class(TInterfacedObject, ITracked)
  private
    FId: Integer;
    class var FCount: Integer;
  public
    constructor Create;
    destructor Destroy; override;
    function GetId: Integer;
    class function GetObjectCount: Integer;
  end;

constructor TTrackedObject.Create;
begin
  inherited Create;
  Inc(FCount);
  FId := FCount;
  WriteLn('สร้าง TTrackedObject #', FId, ' | จำนวน: ', FCount);
end;

destructor TTrackedObject.Destroy;
begin
  Dec(FCount);
  WriteLn('ลบ TTrackedObject #', FId, ' | จำนวน: ', FCount);
  inherited;
end;

function TTrackedObject.GetId: Integer;
begin
  Result := FId;
end;

class function TTrackedObject.GetObjectCount: Integer;
begin
  Result := FCount;
end;

var
  Obj1, Obj2: ITracked;  // Interface references - มี reference counting
  DirectObj: TTrackedObject;
begin
  WriteLn('=== Reference Counting ===');
  WriteLn('จำนวน objects: ', TTrackedObject.GetObjectCount);
  WriteLn;

  WriteLn('--- สร้างผ่าน interface reference ---');
  Obj1 := TTrackedObject.Create;  // RefCount = 1
  WriteLn('Obj1.Id = ', Obj1.Id);
  WriteLn;

  WriteLn('--- Assign ให้ Obj2 ---');
  Obj2 := Obj1;  // RefCount = 2 (ทั้ง Obj1 และ Obj2 ชี้ที่ object เดิม)
  WriteLn('Obj1 = Obj2 ? ', Obj1 = Obj2);
  WriteLn;

  WriteLn('--- ล้าง Obj1 ---');
  Obj1 := nil;  // RefCount = 1 (เหลือแค่ Obj2)
  WriteLn;

  WriteLn('--- ล้าง Obj2 ---');
  Obj2 := nil;  // RefCount = 0 -> ลบอัตโนมัติ!
  WriteLn;

  WriteLn('จำนวน objects หลังล้าง: ', TTrackedObject.GetObjectCount);
  WriteLn;

  WriteLn('--- Manual object (ไม่ใช้ reference counting) ---');
  DirectObj := TTrackedObject.Create;  // ต้อง Free เอง
  WriteLn('DirectObj.Id = ', DirectObj.Id);
  DirectObj.Free;  // ต้อง Free เอง!
  WriteLn;

  ReadLn;
end.
```

---

## 25.4 IInterface และ IUnknown

`IInterface` เป็น base interface ที่ทุก interface สืบทอด มี 3 methods:

```pascal
type
  IInterface = interface
    ['{00000000-0000-0000-C000-000000000046}']
    function QueryInterface(constref IID: TGUID; out Obj): HResult; stdcall;
    function _AddRef: Integer; stdcall;
    function _Release: Integer; stdcall;
  end;

  // IUnknown เป็น alias ของ IInterface ใน Free Pascal
  IUnknown = IInterface;
```

### ตัวอย่างการใช้ QueryInterface

```pascal
program QueryInterfaceDemo;

{$mode objfpc}{$H+}

uses SysUtils;

type
  IFlyable = interface
    ['{11111111-1111-1111-1111-111111111111}']
    procedure Fly;
    function GetAltitude: Double;
  end;

  ISwimmable = interface
    ['{22222222-2222-2222-2222-222222222222}']
    procedure Swim;
    function GetDepth: Double;
  end;

  TDuck = class(TInterfacedObject, IFlyable, ISwimmable)
  public
    procedure Fly;
    function GetAltitude: Double;
    procedure Swim;
    function GetDepth: Double;
  end;

  TFish = class(TInterfacedObject, ISwimmable)
  public
    procedure Swim;
    function GetDepth: Double;
  end;

procedure TDuck.Fly;
begin
  WriteLn('เป็ดบินขึ้น!');
end;

function TDuck.GetAltitude: Double;
begin
  Result := 50.0;
end;

procedure TDuck.Swim;
begin
  WriteLn('เป็ดว่ายน้ำ!');
end;

function TDuck.GetDepth: Double;
begin
  Result := 0.5;
end;

procedure TFish.Swim;
begin
  WriteLn('ปลาว่ายน้ำ!');
end;

function TFish.GetDepth: Double;
begin
  Result := 10.0;
end;

procedure TryFlyAndSwim(Obj: IInterface);
var
  Flyable: IFlyable;
  Swimmable: ISwimmable;
begin
  // QueryInterface ตรวจสอบว่า implement interface ไหม
  if Obj.QueryInterface(IFlyable, Flyable) = S_OK then
  begin
    WriteLn('สามารถบินได้!');
    Flyable.Fly;
    WriteLn('ความสูง: ', Flyable.GetAltitude:0:1, ' ม.');
  end
  else
    WriteLn('บินไม่ได้');

  if Obj.QueryInterface(ISwimmable, Swimmable) = S_OK then
  begin
    WriteLn('สามารถว่ายน้ำได้!');
    Swimmable.Swim;
    WriteLn('ความลึก: ', Swimmable.GetDepth:0:1, ' ม.');
  end
  else
    WriteLn('ว่ายน้ำไม่ได้');
end;

var
  Duck: IInterface;
  Fish: IInterface;
begin
  WriteLn('=== Duck ===');
  Duck := TDuck.Create;
  TryFlyAndSwim(Duck);
  WriteLn;

  WriteLn('=== Fish ===');
  Fish := TFish.Create;
  TryFlyAndSwim(Fish);

  ReadLn;
end.
```

---

## 25.5 Interface Casting

### Supports Function
ตรวจสอบและ cast interface อย่างปลอดภัย

```pascal
uses SysUtils;

// Supports function ตรวจสอบว่า object implement interface ไหม
if Supports(Obj, IMyInterface) then
  WriteLn('implement IMyInterface');

// Supports พร้อม output parameter
var Intf: IMyInterface;
if Supports(Obj, IMyInterface, Intf) then
  Intf.DoSomething;
```

### ตัวอย่าง Interface Casting

```pascal
program InterfaceCasting;

{$mode objfpc}{$H+}

uses SysUtils;

type
  IAnimal = interface
    function MakeSound: string;
    function GetName: string;
  end;

  ILandAnimal = interface(IAnimal)
    function GetLegs: Integer;
  end;

  IWaterAnimal = interface(IAnimal)
    function GetSwimmingSpeed: Double;
  end;

  IAirAnimal = interface(IAnimal)
    function GetFlyingSpeed: Double;
  end;

  TDuck = class(TInterfacedObject, ILandAnimal, IWaterAnimal, IAirAnimal)
  public
    function MakeSound: string;
    function GetName: string;
    function GetLegs: Integer;
    function GetSwimmingSpeed: Double;
    function GetFlyingSpeed: Double;
  end;

  TDog = class(TInterfacedObject, ILandAnimal)
  public
    function MakeSound: string;
    function GetName: string;
    function GetLegs: Integer;
  end;

  TCarp = class(TInterfacedObject, IWaterAnimal)
  public
    function MakeSound: string;
    function GetName: string;
    function GetSwimmingSpeed: Double;
  end;

function TDuck.MakeSound: string; begin Result := 'กแ็ กแ็!'; end;
function TDuck.GetName: string; begin Result := 'เป็ด'; end;
function TDuck.GetLegs: Integer; begin Result := 2; end;
function TDuck.GetSwimmingSpeed: Double; begin Result := 5.0; end;
function TDuck.GetFlyingSpeed: Double; begin Result := 40.0; end;

function TDog.MakeSound: string; begin Result := 'โฮ่ง!'; end;
function TDog.GetName: string; begin Result := 'สุนัข'; end;
function TDog.GetLegs: Integer; begin Result := 4; end;

function TCarp.MakeSound: string; begin Result := '...'; end;
function TCarp.GetName: string; begin Result := 'ปลาคาร์พ'; end;
function TCarp.GetSwimmingSpeed: Double; begin Result := 2.0; end;

procedure DescribeAnimal(Animal: IAnimal);
var
  Land: ILandAnimal;
  Water: IWaterAnimal;
  Air: IAirAnimal;
begin
  WriteLn('=== ', Animal.GetName, ' ===');
  WriteLn('เสียงร้อง: ', Animal.MakeSound);

  // ตรวจสอบด้วย Supports
  if Supports(Animal, ILandAnimal, Land) then
    WriteLn('จำนวนขา: ', Land.GetLegs);

  if Supports(Animal, IWaterAnimal, Water) then
    WriteLn('ความเร็วว่ายน้ำ: ', Water.GetSwimmingSpeed:0:1, ' กม./ชม.');

  if Supports(Animal, IAirAnimal, Air) then
    WriteLn('ความเร็วบิน: ', Air.GetFlyingSpeed:0:1, ' กม./ชม.');
  WriteLn;
end;

var
  Animals: array of IAnimal;
  A: IAnimal;
begin
  SetLength(Animals, 3);
  Animals[0] := TDuck.Create;
  Animals[1] := TDog.Create;
  Animals[2] := TCarp.Create;

  for A in Animals do
    DescribeAnimal(A);

  ReadLn;
end.
```

---

## 25.6 Interface Inheritance

Interfaces สามารถสืบทอดจาก interface อื่นได้

```pascal
type
  IBase = interface
    procedure BaseMethod;
  end;

  IExtended = interface(IBase)
    procedure ExtendedMethod;
  end;

  // ต้อง implement ทั้ง BaseMethod และ ExtendedMethod
  TImpl = class(TInterfacedObject, IExtended)
    procedure BaseMethod;
    procedure ExtendedMethod;
  end;
```

### ตัวอย่าง Interface Hierarchy

```pascal
program InterfaceHierarchy;

{$mode objfpc}{$H+}

uses SysUtils;

type
  // Interface hierarchy สำหรับ streams
  IReadable = interface
    ['{10000001-0000-0000-0000-000000000001}']
    function Read(var Buffer; Count: Integer): Integer;
    function ReadByte: Byte;
    function ReadInt32: Integer;
    function ReadString: string;
    function GetPosition: Int64;
    function GetSize: Int64;
    property Position: Int64 read GetPosition;
    property Size: Int64 read GetSize;
  end;

  IWritable = interface
    ['{10000001-0000-0000-0000-000000000002}']
    function Write(const Buffer; Count: Integer): Integer;
    procedure WriteByte(Value: Byte);
    procedure WriteInt32(Value: Integer);
    procedure WriteString(const Value: string);
    procedure Flush;
  end;

  ISeekable = interface
    ['{10000001-0000-0000-0000-000000000003}']
    function Seek(Offset: Int64; Origin: Integer): Int64;
    procedure SeekToBegin;
    procedure SeekToEnd;
  end;

  // Stream ที่ read ได้และ seek ได้
  IInputStream = interface(IReadable)
    ['{10000002-0000-0000-0000-000000000001}']
    procedure Close;
  end;

  // Stream ที่ read-write ได้และ seek ได้
  IStream = interface(IReadable)
    ['{10000003-0000-0000-0000-000000000001}']
    function Write(const Buffer; Count: Integer): Integer;
    procedure Flush;
    procedure Close;
    function Seek(Offset: Int64; Origin: Integer): Int64;
  end;

  // Memory Stream implementation
  TMemStream = class(TInterfacedObject, IReadable, IWritable, ISeekable)
  private
    FBuffer: array of Byte;
    FSize: Int64;
    FPosition: Int64;
    FCapacity: Int64;
    procedure EnsureCapacity(Required: Int64);
  public
    constructor Create(InitCapacity: Int64 = 1024);

    // IReadable
    function Read(var Buffer; Count: Integer): Integer;
    function ReadByte: Byte;
    function ReadInt32: Integer;
    function ReadString: string;
    function GetPosition: Int64;
    function GetSize: Int64;

    // IWritable
    function Write(const Buffer; Count: Integer): Integer;
    procedure WriteByte(Value: Byte);
    procedure WriteInt32(Value: Integer);
    procedure WriteString(const Value: string);
    procedure Flush;

    // ISeekable
    function Seek(Offset: Int64; Origin: Integer): Int64;
    procedure SeekToBegin;
    procedure SeekToEnd;

    procedure Clear;
    function ToArray: TBytes;
  end;

procedure TMemStream.EnsureCapacity(Required: Int64);
begin
  if Required > FCapacity then
  begin
    FCapacity := Max(Required, FCapacity * 2);
    SetLength(FBuffer, FCapacity);
  end;
end;

constructor TMemStream.Create(InitCapacity: Int64);
begin
  inherited Create;
  FCapacity := InitCapacity;
  FSize := 0;
  FPosition := 0;
  SetLength(FBuffer, FCapacity);
end;

function TMemStream.Read(var Buffer; Count: Integer): Integer;
var
  Available: Int64;
begin
  Available := FSize - FPosition;
  if Available <= 0 then
  begin
    Result := 0;
    Exit;
  end;
  Result := Min(Count, Available);
  Move(FBuffer[FPosition], Buffer, Result);
  Inc(FPosition, Result);
end;

function TMemStream.ReadByte: Byte;
begin
  Read(Result, 1);
end;

function TMemStream.ReadInt32: Integer;
begin
  Read(Result, SizeOf(Integer));
end;

function TMemStream.ReadString: string;
var
  Len: Integer;
begin
  ReadInt32;  // อ่านความยาวก่อน (แต่ใช้ variable แทน)
  Len := ReadInt32;
  SetLength(Result, Len);
  if Len > 0 then
    Read(Result[1], Len);
end;

function TMemStream.GetPosition: Int64;
begin
  Result := FPosition;
end;

function TMemStream.GetSize: Int64;
begin
  Result := FSize;
end;

function TMemStream.Write(const Buffer; Count: Integer): Integer;
begin
  EnsureCapacity(FPosition + Count);
  Move(Buffer, FBuffer[FPosition], Count);
  Inc(FPosition, Count);
  if FPosition > FSize then FSize := FPosition;
  Result := Count;
end;

procedure TMemStream.WriteByte(Value: Byte);
begin
  Write(Value, 1);
end;

procedure TMemStream.WriteInt32(Value: Integer);
begin
  Write(Value, SizeOf(Integer));
end;

procedure TMemStream.WriteString(const Value: string);
var
  Len: Integer;
begin
  Len := Length(Value);
  WriteInt32(Len);
  if Len > 0 then
    Write(Value[1], Len);
end;

procedure TMemStream.Flush;
begin
  // Memory stream ไม่ต้อง flush
end;

function TMemStream.Seek(Offset: Int64; Origin: Integer): Int64;
begin
  case Origin of
    0: FPosition := Offset;           // จากต้น
    1: FPosition := FPosition + Offset; // จากตำแหน่งปัจจุบัน
    2: FPosition := FSize + Offset;    // จากท้าย
  end;
  FPosition := Max(0, Min(FPosition, FSize));
  Result := FPosition;
end;

procedure TMemStream.SeekToBegin;
begin
  FPosition := 0;
end;

procedure TMemStream.SeekToEnd;
begin
  FPosition := FSize;
end;

procedure TMemStream.Clear;
begin
  FSize := 0;
  FPosition := 0;
end;

function TMemStream.ToArray: TBytes;
begin
  SetLength(Result, FSize);
  if FSize > 0 then
    Move(FBuffer[0], Result[0], FSize);
end;

// Demo
procedure WriteData(Writer: IWritable);
begin
  Writer.WriteString('สวัสดี');
  Writer.WriteInt32(12345);
  Writer.WriteByte(255);
  Writer.WriteString('World');
  WriteLn('เขียนข้อมูลสำเร็จ');
end;

procedure ReadData(Reader: IReadable);
begin
  WriteLn('อ่านข้อมูล:');
  WriteLn('String: ', Reader.ReadString);
  WriteLn('Int32: ', Reader.ReadInt32);
  WriteLn('Byte: ', Reader.ReadByte);
  WriteLn('String: ', Reader.ReadString);
end;

var
  Stream: TMemStream;
  Writer: IWritable;
  Reader: IReadable;
  Seeker: ISeekable;
begin
  Stream := TMemStream.Create;

  // ใช้ผ่าน interfaces
  Writer := Stream;
  WriteData(Writer);

  WriteLn('ขนาดข้อมูล: ', Stream.GetSize, ' bytes');
  WriteLn('ตำแหน่งปัจจุบัน: ', Stream.GetPosition);
  WriteLn;

  // Seek กลับไปต้น
  Seeker := Stream;
  Seeker.SeekToBegin;
  WriteLn('หลัง SeekToBegin ตำแหน่ง: ', Stream.GetPosition);
  WriteLn;

  // อ่านข้อมูล
  Reader := Stream;
  ReadData(Reader);

  Stream.Free;
  ReadLn;
end.
```

---

## 25.7 implements Keyword

`implements` ให้ class มอบหมาย interface implementation ไปให้ field อื่น

```pascal
program ImplementsDelegation;

{$mode objfpc}{$H+}

uses SysUtils;

type
  ILogger = interface
    procedure Log(const Message: string);
    procedure LogError(const Message: string);
  end;

  // Simple logger implementation
  TSimpleLogger = class(TInterfacedObject, ILogger)
  private
    FPrefix: string;
  public
    constructor Create(const APrefix: string);
    procedure Log(const Message: string);
    procedure LogError(const Message: string);
  end;

  // Class ที่ implement ILogger ผ่าน delegation
  TService = class(TInterfacedObject, ILogger)
  private
    FLogger: ILogger;  // Field ที่รับ delegation
    FServiceName: string;
  public
    constructor Create(const AName: string);
    destructor Destroy; override;

    procedure DoWork(const Task: string);

    // implements keyword - มอบหมาย ILogger ให้ FLogger จัดการ
    property Logger: ILogger read FLogger implements ILogger;
  end;

constructor TSimpleLogger.Create(const APrefix: string);
begin
  inherited Create;
  FPrefix := APrefix;
end;

procedure TSimpleLogger.Log(const Message: string);
begin
  WriteLn('[', FPrefix, '][INFO] ', Message);
end;

procedure TSimpleLogger.LogError(const Message: string);
begin
  WriteLn('[', FPrefix, '][ERROR] ', Message);
end;

constructor TService.Create(const AName: string);
begin
  inherited Create;
  FServiceName := AName;
  FLogger := TSimpleLogger.Create(AName);
end;

destructor TService.Destroy;
begin
  FLogger := nil;  // ลด reference count
  inherited;
end;

procedure TService.DoWork(const Task: string);
begin
  FLogger.Log('เริ่ม task: ' + Task);
  // ทำงาน...
  FLogger.Log('จบ task: ' + Task);
end;

var
  Svc: TService;
  Logger: ILogger;
begin
  WriteLn('=== implements Delegation ===');
  WriteLn;

  Svc := TService.Create('UserService');

  Svc.DoWork('สร้างผู้ใช้');
  Svc.DoWork('อัปเดตโปรไฟล์');
  WriteLn;

  // เรียก ILogger ผ่าน TService (delegation)
  Logger := Svc;  // TService มี ILogger ผ่าน implements
  Logger.Log('Log โดยตรงจาก service');
  Logger.LogError('เกิดข้อผิดพลาด!');

  Svc.Free;
  ReadLn;
end.
```

---

## 25.8 GUID สำหรับ Interfaces

GUID (Globally Unique Identifier) ใช้ระบุ interface อย่างไม่ซ้ำกันทั่วโลก

```pascal
// สร้าง GUID ใน Lazarus: Tools -> Generate GUID (หรือ Ctrl+Shift+G)
// รูปแบบ: {XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX}

type
  IMyService = interface
    ['{A1234567-89AB-CDEF-0123-456789ABCDEF}']
    procedure DoSomething;
  end;

// ใช้ GUID ตรวจสอบ
const
  IID_IMyService: TGUID = '{A1234567-89AB-CDEF-0123-456789ABCDEF}';

// เปรียบเทียบ GUID
var
  IID: TGUID;
begin
  IID := GetTypeData(TypeInfo(IMyService))^.GUID;
  WriteLn(GUIDToString(IID));  // แสดง GUID
end;
```

---

## 25.9 COM-Compatible Interfaces

```pascal
program COMCompatible;

{$mode objfpc}{$H+}

uses SysUtils, ComObj;

type
  // COM-compatible interface (สืบทอดจาก IUnknown)
  IMyComObject = interface(IUnknown)
    ['{B2345678-9ABC-DEF0-1234-56789ABCDEF0}']
    function GetVersion: string; stdcall;
    procedure Process(const Data: WideString); stdcall;
  end;

  TMyComImpl = class(TInterfacedObject, IMyComObject)
  private
    FVersion: string;
  public
    constructor Create;
    function GetVersion: string; stdcall;
    procedure Process(const Data: WideString); stdcall;
  end;

constructor TMyComImpl.Create;
begin
  inherited Create;
  FVersion := '1.0.0';
end;

function TMyComImpl.GetVersion: string;
begin
  Result := FVersion;
end;

procedure TMyComImpl.Process(const Data: WideString);
begin
  WriteLn('Processing: ', Data);
end;

var
  Obj: IMyComObject;
begin
  Obj := TMyComImpl.Create;
  WriteLn('Version: ', Obj.GetVersion);
  Obj.Process('ข้อมูลสำหรับประมวลผล');

  ReadLn;
end.
```

---

## 25.10 IInterface vs Abstract Classes

### เปรียบเทียบ

| คุณสมบัติ | Interface | Abstract Class |
|-----------|-----------|----------------|
| Implementation | ไม่มี | มีได้บางส่วน |
| Multiple inheritance | ได้ | ไม่ได้ (single only) |
| Fields | ไม่ได้ | ได้ |
| Constructor | ไม่มี | มีได้ |
| Reference counting | ได้ (TInterfacedObject) | ต้อง Free เอง |
| COM compatible | ได้ | ไม่ |
| เมื่อใช้ | ต้องการ contract | ต้องการ shared behavior |

```pascal
program InterfaceVsAbstract;

{$mode objfpc}{$H+}

uses SysUtils;

// ===== Abstract Class approach =====
type
  TAbstractAnimal = class
  private
    FName: string;
    FAge: Integer;
  public
    constructor Create(const AName: string; AAge: Integer);
    // Abstract methods
    function MakeSound: string; virtual; abstract;
    function CanFly: Boolean; virtual; abstract;
    // Concrete methods (shared behavior)
    procedure Breathe;
    procedure ShowInfo; virtual;
    property Name: string read FName;
    property Age: Integer read FAge;
  end;

  TAbstractDog = class(TAbstractAnimal)
    function MakeSound: string; override;
    function CanFly: Boolean; override;
  end;

  TAbstractBird = class(TAbstractAnimal)
    function MakeSound: string; override;
    function CanFly: Boolean; override;
  end;

// ===== Interface approach =====
type
  IAnimal = interface
    function MakeSound: string;
    function CanFly: Boolean;
    function GetName: string;
    property Name: string read GetName;
  end;

  ISpecialAbility = interface
    function GetAbilityName: string;
    procedure UseAbility;
  end;

  TIntfDog = class(TInterfacedObject, IAnimal)
    function MakeSound: string;
    function CanFly: Boolean;
    function GetName: string;
  end;

  // Robot ที่ implement IAnimal แต่ไม่ได้เป็น animal จริงๆ
  TRobotAnimal = class(TInterfacedObject, IAnimal, ISpecialAbility)
    function MakeSound: string;
    function CanFly: Boolean;
    function GetName: string;
    function GetAbilityName: string;
    procedure UseAbility;
  end;

// Abstract implementations
constructor TAbstractAnimal.Create(const AName: string; AAge: Integer);
begin inherited Create; FName := AName; FAge := AAge; end;

procedure TAbstractAnimal.Breathe;
begin WriteLn(FName, ' หายใจ'); end;

procedure TAbstractAnimal.ShowInfo;
begin
  WriteLn('ชื่อ: ', FName, ' อายุ: ', FAge);
  WriteLn('เสียง: ', MakeSound);
  WriteLn('บินได้: ', CanFly);
end;

function TAbstractDog.MakeSound: string; begin Result := 'โฮ่ง!'; end;
function TAbstractDog.CanFly: Boolean; begin Result := False; end;
function TAbstractBird.MakeSound: string; begin Result := 'จ๊วก จ๊วก'; end;
function TAbstractBird.CanFly: Boolean; begin Result := True; end;

// Interface implementations
function TIntfDog.MakeSound: string; begin Result := 'โฮ่ง!'; end;
function TIntfDog.CanFly: Boolean; begin Result := False; end;
function TIntfDog.GetName: string; begin Result := 'สุนัข'; end;

function TRobotAnimal.MakeSound: string; begin Result := 'บีบ โบ๊บ!'; end;
function TRobotAnimal.CanFly: Boolean; begin Result := True; end;
function TRobotAnimal.GetName: string; begin Result := 'หุ่นยนต์สัตว์'; end;
function TRobotAnimal.GetAbilityName: string; begin Result := 'ยิงเลเซอร์'; end;
procedure TRobotAnimal.UseAbility; begin WriteLn('ยิงเลเซอร์! zzap!'); end;

var
  // Abstract class - สามารถใช้ shared methods
  Animals: array of TAbstractAnimal;
  // Interface - polymorphism ข้าม hierarchy
  IAnimals: array of IAnimal;
  A: TAbstractAnimal;
  IA: IAnimal;
begin
  WriteLn('=== Abstract Class ===');
  SetLength(Animals, 2);
  Animals[0] := TAbstractDog.Create('บุ๋ม', 3);
  Animals[1] := TAbstractBird.Create('ทวีต', 1);
  for A in Animals do
  begin
    A.ShowInfo;
    A.Breathe;  // Shared behavior จาก abstract class
    WriteLn;
  end;
  for A in Animals do A.Free;

  WriteLn('=== Interface ===');
  SetLength(IAnimals, 3);
  IAnimals[0] := TIntfDog.Create;
  IAnimals[1] := TAbstractBird.Create('นก', 2);  // อ่านผ่าน interface ด้วย
  IAnimals[2] := TRobotAnimal.Create;
  for IA in IAnimals do
  begin
    WriteLn(IA.GetName, ': ', IA.MakeSound, ' (บินได้: ', IA.CanFly, ')');
    var Special: ISpecialAbility;
    if Supports(IA, ISpecialAbility, Special) then
      Special.UseAbility;
  end;

  // Note: IAnimals[1] เป็น TAbstractBird ต้อง Free แยก
  (IAnimals[1] as TAbstractBird).Free;

  ReadLn;
end.
```

---

## 25.11 โปรแกรมตัวอย่าง: Serializable Interface

```pascal
program SerializableInterface;

{$mode objfpc}{$H+}

uses SysUtils, Classes;

type
  ISerializable = interface
    ['{SERIAL01-0000-0000-0000-000000000001}']
    function Serialize: string;
    procedure Deserialize(const Data: string);
    function GetTypeName: string;
  end;

  IValidatable = interface
    ['{VALID001-0000-0000-0000-000000000001}']
    function Validate: Boolean;
    function GetValidationErrors: string;
  end;

  // Helper class สำหรับ simple JSON-like serialization
  TSimpleJSON = class
  private
    class function EscapeString(const S: string): string;
    class function UnescapeString(const S: string): string;
    class function ExtractValue(const JSON, Key: string): string;
  public
    class function ObjectStart: string;
    class function ObjectEnd: string;
    class function Field(const Name, Value: string): string;
    class function FieldInt(const Name: string; Value: Integer): string;
    class function FieldFloat(const Name: string; Value: Double): string;
    class function FieldBool(const Name: string; Value: Boolean): string;
    class function GetString(const JSON, Key: string): string;
    class function GetInt(const JSON, Key: string): Integer;
    class function GetFloat(const JSON, Key: string): Double;
    class function GetBool(const JSON, Key: string): Boolean;
  end;

  TAddress = class(TInterfacedObject, ISerializable, IValidatable)
  private
    FStreet: string;
    FCity: string;
    FProvince: string;
    FZipCode: string;
    FCountry: string;
  public
    constructor Create; overload;
    constructor Create(const AStreet, ACity, AProvince, AZip: string;
                      const ACountry: string = 'Thailand'); overload;
    // ISerializable
    function Serialize: string;
    procedure Deserialize(const Data: string);
    function GetTypeName: string;
    // IValidatable
    function Validate: Boolean;
    function GetValidationErrors: string;
    // Properties
    property Street: string read FStreet write FStreet;
    property City: string read FCity write FCity;
    property Province: string read FProvince write FProvince;
    property ZipCode: string read FZipCode write FZipCode;
    property Country: string read FCountry write FCountry;
    // Display
    function ToString: string; override;
  end;

  TPerson = class(TInterfacedObject, ISerializable, IValidatable)
  private
    FFirstName: string;
    FLastName: string;
    FAge: Integer;
    FEmail: string;
    FAddress: TAddress;
  public
    constructor Create; overload;
    constructor Create(const AFirst, ALast: string; AAge: Integer;
                      const AEmail: string); overload;
    destructor Destroy; override;
    // ISerializable
    function Serialize: string;
    procedure Deserialize(const Data: string);
    function GetTypeName: string;
    // IValidatable
    function Validate: Boolean;
    function GetValidationErrors: string;
    // Properties
    property FirstName: string read FFirstName write FFirstName;
    property LastName: string read FLastName write FLastName;
    property Age: Integer read FAge write FAge;
    property Email: string read FEmail write FEmail;
    property Address: TAddress read FAddress;
    // Display
    function ToString: string; override;
    function FullName: string;
  end;

  // Repository ที่ serialize/deserialize objects
  TSimpleRepository = class
  private
    FItems: array of ISerializable;
    FCount: Integer;
  public
    constructor Create;
    procedure Add(Item: ISerializable);
    procedure SaveToFile(const FileName: string);
    procedure LoadFromFile(const FileName: string);
    procedure ShowAll;
    function GetCount: Integer;
  end;

// ===== TSimpleJSON =====
class function TSimpleJSON.EscapeString(const S: string): string;
begin
  Result := StringReplace(S, '"', '\"', [rfReplaceAll]);
  Result := StringReplace(Result, #10, '\n', [rfReplaceAll]);
  Result := StringReplace(Result, #13, '\r', [rfReplaceAll]);
end;

class function TSimpleJSON.UnescapeString(const S: string): string;
begin
  Result := StringReplace(S, '\"', '"', [rfReplaceAll]);
  Result := StringReplace(Result, '\n', #10, [rfReplaceAll]);
  Result := StringReplace(Result, '\r', #13, [rfReplaceAll]);
end;

class function TSimpleJSON.ExtractValue(const JSON, Key: string): string;
var
  SearchKey, StartPos: string;
  P, Q: Integer;
begin
  Result := '';
  SearchKey := '"' + Key + '":';
  P := Pos(SearchKey, JSON);
  if P = 0 then Exit;
  P := P + Length(SearchKey);
  // Skip spaces
  while (P <= Length(JSON)) and (JSON[P] = ' ') do Inc(P);
  if P > Length(JSON) then Exit;
  if JSON[P] = '"' then
  begin
    // String value
    Inc(P);
    Q := P;
    while (Q <= Length(JSON)) and not ((JSON[Q] = '"') and (Q > P) and (JSON[Q-1] <> '\')) do
      Inc(Q);
    Result := UnescapeString(Copy(JSON, P, Q - P));
  end
  else
  begin
    // Number/Boolean/null
    Q := P;
    while (Q <= Length(JSON)) and not (JSON[Q] in [',', '}', ']']) do Inc(Q);
    Result := Trim(Copy(JSON, P, Q - P));
  end;
end;

class function TSimpleJSON.ObjectStart: string;
begin Result := '{'; end;
class function TSimpleJSON.ObjectEnd: string;
begin Result := '}'; end;

class function TSimpleJSON.Field(const Name, Value: string): string;
begin
  Result := Format('"%s":"%s"', [Name, EscapeString(Value)]);
end;

class function TSimpleJSON.FieldInt(const Name: string; Value: Integer): string;
begin
  Result := Format('"%s":%d', [Name, Value]);
end;

class function TSimpleJSON.FieldFloat(const Name: string; Value: Double): string;
begin
  Result := Format('"%s":%.4f', [Name, Value]);
end;

class function TSimpleJSON.FieldBool(const Name: string; Value: Boolean): string;
begin
  Result := Format('"%s":%s', [Name, IfThen(Value, 'true', 'false')]);
end;

class function TSimpleJSON.GetString(const JSON, Key: string): string;
begin
  Result := ExtractValue(JSON, Key);
end;

class function TSimpleJSON.GetInt(const JSON, Key: string): Integer;
begin
  Result := StrToIntDef(ExtractValue(JSON, Key), 0);
end;

class function TSimpleJSON.GetFloat(const JSON, Key: string): Double;
begin
  Result := StrToFloatDef(ExtractValue(JSON, Key), 0);
end;

class function TSimpleJSON.GetBool(const JSON, Key: string): Boolean;
begin
  Result := LowerCase(ExtractValue(JSON, Key)) = 'true';
end;

// ===== TAddress =====
constructor TAddress.Create;
begin
  inherited Create;
  FCountry := 'Thailand';
end;

constructor TAddress.Create(const AStreet, ACity, AProvince, AZip: string;
                           const ACountry: string);
begin
  inherited Create;
  FStreet := AStreet;
  FCity := ACity;
  FProvince := AProvince;
  FZipCode := AZip;
  FCountry := ACountry;
end;

function TAddress.Serialize: string;
begin
  Result := TSimpleJSON.ObjectStart +
    TSimpleJSON.Field('street', FStreet) + ',' +
    TSimpleJSON.Field('city', FCity) + ',' +
    TSimpleJSON.Field('province', FProvince) + ',' +
    TSimpleJSON.Field('zip', FZipCode) + ',' +
    TSimpleJSON.Field('country', FCountry) +
    TSimpleJSON.ObjectEnd;
end;

procedure TAddress.Deserialize(const Data: string);
begin
  FStreet := TSimpleJSON.GetString(Data, 'street');
  FCity := TSimpleJSON.GetString(Data, 'city');
  FProvince := TSimpleJSON.GetString(Data, 'province');
  FZipCode := TSimpleJSON.GetString(Data, 'zip');
  FCountry := TSimpleJSON.GetString(Data, 'country');
end;

function TAddress.GetTypeName: string;
begin
  Result := 'TAddress';
end;

function TAddress.Validate: Boolean;
begin
  Result := (Trim(FCity) <> '') and (Trim(FCountry) <> '');
end;

function TAddress.GetValidationErrors: string;
begin
  Result := '';
  if Trim(FCity) = '' then Result := Result + 'City ต้องไม่ว่าง; ';
  if Trim(FCountry) = '' then Result := Result + 'Country ต้องไม่ว่าง; ';
end;

function TAddress.ToString: string;
begin
  Result := FStreet + ', ' + FCity + ', ' + FProvince + ' ' + FZipCode + ', ' + FCountry;
end;

// ===== TPerson =====
constructor TPerson.Create;
begin
  inherited Create;
  FAddress := TAddress.Create;
end;

constructor TPerson.Create(const AFirst, ALast: string; AAge: Integer;
                          const AEmail: string);
begin
  Create;
  FFirstName := AFirst;
  FLastName := ALast;
  FAge := AAge;
  FEmail := AEmail;
end;

destructor TPerson.Destroy;
begin
  FAddress.Free;
  inherited;
end;

function TPerson.Serialize: string;
begin
  Result := TSimpleJSON.ObjectStart +
    TSimpleJSON.Field('type', 'TPerson') + ',' +
    TSimpleJSON.Field('firstName', FFirstName) + ',' +
    TSimpleJSON.Field('lastName', FLastName) + ',' +
    TSimpleJSON.FieldInt('age', FAge) + ',' +
    TSimpleJSON.Field('email', FEmail) + ',' +
    '"address":' + FAddress.Serialize +
    TSimpleJSON.ObjectEnd;
end;

procedure TPerson.Deserialize(const Data: string);
var
  AddrStart, AddrEnd: Integer;
  AddrJSON: string;
begin
  FFirstName := TSimpleJSON.GetString(Data, 'firstName');
  FLastName := TSimpleJSON.GetString(Data, 'lastName');
  FAge := TSimpleJSON.GetInt(Data, 'age');
  FEmail := TSimpleJSON.GetString(Data, 'email');
  // Extract address object (simplified)
  AddrStart := Pos('"address":{', Data);
  if AddrStart > 0 then
  begin
    AddrStart := AddrStart + Length('"address":');
    AddrEnd := Length(Data);  // Simplified
    AddrJSON := Copy(Data, AddrStart, AddrEnd - AddrStart);
    FAddress.Deserialize(AddrJSON);
  end;
end;

function TPerson.GetTypeName: string;
begin
  Result := 'TPerson';
end;

function TPerson.Validate: Boolean;
begin
  Result := (Trim(FFirstName) <> '') and (Trim(FLastName) <> '') and
            (FAge > 0) and (Pos('@', FEmail) > 0) and FAddress.Validate;
end;

function TPerson.GetValidationErrors: string;
begin
  Result := '';
  if Trim(FFirstName) = '' then Result := Result + 'FirstName ต้องไม่ว่าง; ';
  if Trim(FLastName) = '' then Result := Result + 'LastName ต้องไม่ว่าง; ';
  if FAge <= 0 then Result := Result + 'Age ต้องมากกว่า 0; ';
  if Pos('@', FEmail) = 0 then Result := Result + 'Email ไม่ถูกต้อง; ';
  Result := Result + FAddress.GetValidationErrors;
end;

function TPerson.ToString: string;
begin
  Result := Format('%s (อายุ %d) - %s', [FullName, FAge, FEmail]);
end;

function TPerson.FullName: string;
begin
  Result := FFirstName + ' ' + FLastName;
end;

// ===== TSimpleRepository =====
constructor TSimpleRepository.Create;
begin
  inherited Create;
  FCount := 0;
  SetLength(FItems, 10);
end;

procedure TSimpleRepository.Add(Item: ISerializable);
begin
  if FCount >= Length(FItems) then
    SetLength(FItems, Length(FItems) * 2);
  FItems[FCount] := Item;
  Inc(FCount);
end;

procedure TSimpleRepository.SaveToFile(const FileName: string);
var
  F: TextFile;
  I: Integer;
begin
  AssignFile(F, FileName);
  Rewrite(F);
  try
    WriteLn(F, FCount);
    for I := 0 to FCount - 1 do
    begin
      WriteLn(F, FItems[I].GetTypeName);
      WriteLn(F, FItems[I].Serialize);
    end;
    WriteLn('บันทึกข้อมูล ', FCount, ' รายการไปยัง ', FileName);
  finally
    CloseFile(F);
  end;
end;

procedure TSimpleRepository.LoadFromFile(const FileName: string);
var
  F: TextFile;
  Count: Integer;
  TypeName, Data: string;
  I: Integer;
  Item: ISerializable;
begin
  if not FileExists(FileName) then
  begin
    WriteLn('ไม่พบไฟล์: ', FileName);
    Exit;
  end;
  AssignFile(F, FileName);
  Reset(F);
  try
    ReadLn(F, Count);
    FCount := 0;
    for I := 1 to Count do
    begin
      ReadLn(F, TypeName);
      ReadLn(F, Data);
      // สร้าง object ตาม type name
      if TypeName = 'TPerson' then
      begin
        var P := TPerson.Create;
        P.Deserialize(Data);
        Item := P;
      end
      else
        Continue;
      Add(Item);
    end;
    WriteLn('โหลดข้อมูล ', FCount, ' รายการจาก ', FileName);
  finally
    CloseFile(F);
  end;
end;

procedure TSimpleRepository.ShowAll;
var
  I: Integer;
begin
  WriteLn('=== Repository (', FCount, ' รายการ) ===');
  for I := 0 to FCount - 1 do
  begin
    WriteLn(I + 1, '. [', FItems[I].GetTypeName, ']');
    var Valid: IValidatable;
    if Supports(FItems[I], IValidatable, Valid) then
    begin
      if Valid.Validate then
        WriteLn('   ✓ ข้อมูลถูกต้อง')
      else
        WriteLn('   ✗ ข้อผิดพลาด: ', Valid.GetValidationErrors);
    end;
    WriteLn('   JSON: ', Copy(FItems[I].Serialize, 1, 80), '...');
  end;
end;

function TSimpleRepository.GetCount: Integer;
begin
  Result := FCount;
end;

// ===== Main =====
var
  Repo: TSimpleRepository;
  P1, P2, P3: TPerson;
begin
  WriteLn('===== Serializable Interface Demo =====');
  WriteLn;

  Repo := TSimpleRepository.Create;

  // สร้างข้อมูล
  P1 := TPerson.Create('สมชาย', 'ใจดี', 30, 'somchai@example.com');
  P1.Address.FStreet := '123 ถนนสุขุมวิท';
  P1.Address.FCity := 'กรุงเทพมหานคร';
  P1.Address.FProvince := 'กรุงเทพมหานคร';
  P1.Address.FZipCode := '10110';

  P2 := TPerson.Create('สมหญิง', 'สวยงาม', 25, 'somying@example.com');
  P2.Address.FCity := 'เชียงใหม่';
  P2.Address.FProvince := 'เชียงใหม่';

  P3 := TPerson.Create('', '', -1, 'invalid');  // ข้อมูลไม่ถูกต้อง

  Repo.Add(P1);
  Repo.Add(P2);
  Repo.Add(P3);

  Repo.ShowAll;
  WriteLn;

  // บันทึกและโหลด
  Repo.SaveToFile('/tmp/persons.dat');
  WriteLn;

  var Repo2 := TSimpleRepository.Create;
  Repo2.LoadFromFile('/tmp/persons.dat');
  Repo2.ShowAll;

  Repo2.Free;
  Repo.Free;
  P1.Free;
  P2.Free;
  P3.Free;

  ReadLn;
end.
```

---

## 25.12 โปรแกรมตัวอย่าง: Plugin Interface System

```pascal
program PluginSystem;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TPluginInfo = record
    Name: string;
    Version: string;
    Author: string;
    Description: string;
  end;

  // Interface หลักสำหรับทุก plugin
  IPlugin = interface
    ['{PLUGIN01-0000-0000-0000-000000000001}']
    function GetInfo: TPluginInfo;
    function Initialize: Boolean;
    procedure Shutdown;
    function IsInitialized: Boolean;
  end;

  // Interface สำหรับ text processor plugins
  ITextPlugin = interface(IPlugin)
    ['{PLUGIN02-0000-0000-0000-000000000001}']
    function ProcessText(const Input: string): string;
    function GetMimeType: string;
  end;

  // Interface สำหรับ image plugins
  IImagePlugin = interface(IPlugin)
    ['{PLUGIN03-0000-0000-0000-000000000001}']
    function ProcessImage(const ImageData: TBytes): TBytes;
    function GetSupportedFormats: TStringArray;
  end;

  // Interface สำหรับ data format plugins
  IDataFormatPlugin = interface(IPlugin)
    ['{PLUGIN04-0000-0000-0000-000000000001}']
    function Encode(const Data: Variant): string;
    function Decode(const Encoded: string): Variant;
    function GetFormat: string;
  end;

  // Abstract base class สำหรับ plugins ทุกตัว
  TBasePlugin = class(TInterfacedObject, IPlugin)
  private
    FInitialized: Boolean;
  protected
    function GetPluginInfo: TPluginInfo; virtual; abstract;
    function DoInitialize: Boolean; virtual;
    procedure DoShutdown; virtual;
  public
    // IPlugin
    function GetInfo: TPluginInfo;
    function Initialize: Boolean;
    procedure Shutdown;
    function IsInitialized: Boolean;
    property Initialized: Boolean read FInitialized;
  end;

  // Concrete text plugins
  TEncryptPlugin = class(TBasePlugin, ITextPlugin)
  private
    FKey: Byte;
  protected
    function GetPluginInfo: TPluginInfo; override;
    function DoInitialize: Boolean; override;
  public
    constructor Create(AKey: Byte = 42);
    function ProcessText(const Input: string): string;
    function GetMimeType: string;
  end;

  TMarkdownPlugin = class(TBasePlugin, ITextPlugin)
  protected
    function GetPluginInfo: TPluginInfo; override;
  public
    function ProcessText(const Input: string): string;
    function GetMimeType: string;
  end;

  TWordCountPlugin = class(TBasePlugin, ITextPlugin)
  protected
    function GetPluginInfo: TPluginInfo; override;
  public
    function ProcessText(const Input: string): string;
    function GetMimeType: string;
  end;

  // Plugin Manager
  TPluginManager = class
  private
    FPlugins: array of IPlugin;
    FCount: Integer;
    procedure RegisterPlugin(Plugin: IPlugin);
  public
    constructor Create;
    procedure LoadPlugin(Plugin: IPlugin);
    procedure UnloadAll;
    function GetPlugin(const Name: string): IPlugin;
    function GetTextPlugins: TArray<ITextPlugin>;
    procedure InitializeAll;
    procedure ShowLoadedPlugins;
    procedure ProcessText(const Input: string);
    function GetCount: Integer;
  end;

// ===== TBasePlugin =====
function TBasePlugin.GetInfo: TPluginInfo;
begin
  Result := GetPluginInfo;
end;

function TBasePlugin.Initialize: Boolean;
begin
  if FInitialized then
  begin
    Result := True;
    Exit;
  end;
  Result := DoInitialize;
  FInitialized := Result;
  if Result then
    WriteLn('Plugin "', GetInfo.Name, '" initialized')
  else
    WriteLn('Plugin "', GetInfo.Name, '" failed to initialize');
end;

procedure TBasePlugin.Shutdown;
begin
  if not FInitialized then Exit;
  DoShutdown;
  FInitialized := False;
  WriteLn('Plugin "', GetInfo.Name, '" shut down');
end;

function TBasePlugin.IsInitialized: Boolean;
begin
  Result := FInitialized;
end;

function TBasePlugin.DoInitialize: Boolean;
begin
  Result := True;  // Default: สำเร็จ
end;

procedure TBasePlugin.DoShutdown;
begin
  // Default: ไม่ต้องทำอะไร
end;

// ===== TEncryptPlugin =====
constructor TEncryptPlugin.Create(AKey: Byte);
begin
  inherited Create;
  FKey := AKey;
end;

function TEncryptPlugin.GetPluginInfo: TPluginInfo;
begin
  Result.Name := 'XOR Encrypt';
  Result.Version := '1.0';
  Result.Author := 'Claude';
  Result.Description := 'เข้ารหัส text ด้วย XOR';
end;

function TEncryptPlugin.DoInitialize: Boolean;
begin
  Result := FKey > 0;
  if not Result then
    WriteLn('XOR key ต้องมากกว่า 0');
end;

function TEncryptPlugin.ProcessText(const Input: string): string;
var
  I: Integer;
begin
  SetLength(Result, Length(Input));
  for I := 1 to Length(Input) do
    Result[I] := Chr(Ord(Input[I]) xor FKey);
end;

function TEncryptPlugin.GetMimeType: string;
begin
  Result := 'application/encrypted';
end;

// ===== TMarkdownPlugin =====
function TMarkdownPlugin.GetPluginInfo: TPluginInfo;
begin
  Result.Name := 'Markdown to HTML';
  Result.Version := '2.0';
  Result.Author := 'Claude';
  Result.Description := 'แปลง Markdown เป็น HTML อย่างง่าย';
end;

function TMarkdownPlugin.ProcessText(const Input: string): string;
begin
  // Simple Markdown conversion
  Result := Input;
  // Bold: **text** -> <b>text</b>
  // (simplified - ใช้ regex จริงๆ ควรใช้ library)
  Result := StringReplace(Result, '**', '<b>', [rfReplaceAll]);
  // Headers: # text -> <h1>text</h1>
  if (Length(Result) > 0) and (Result[1] = '#') then
    Result := '<h1>' + Trim(Copy(Result, 2, MaxInt)) + '</h1>';
end;

function TMarkdownPlugin.GetMimeType: string;
begin
  Result := 'text/html';
end;

// ===== TWordCountPlugin =====
function TWordCountPlugin.GetPluginInfo: TPluginInfo;
begin
  Result.Name := 'Word Count';
  Result.Version := '1.0';
  Result.Author := 'Claude';
  Result.Description := 'นับจำนวนคำใน text';
end;

function TWordCountPlugin.ProcessText(const Input: string): string;
var
  I, WordCount: Integer;
  InWord: Boolean;
begin
  WordCount := 0;
  InWord := False;
  for I := 1 to Length(Input) do
  begin
    if Input[I] in [' ', #9, #10, #13] then
      InWord := False
    else if not InWord then
    begin
      InWord := True;
      Inc(WordCount);
    end;
  end;
  Result := Format('จำนวนคำ: %d | จำนวนตัวอักษร: %d', [WordCount, Length(Input)]);
end;

function TWordCountPlugin.GetMimeType: string;
begin
  Result := 'text/plain';
end;

// ===== TPluginManager =====
constructor TPluginManager.Create;
begin
  inherited Create;
  FCount := 0;
  SetLength(FPlugins, 10);
end;

procedure TPluginManager.RegisterPlugin(Plugin: IPlugin);
begin
  if FCount >= Length(FPlugins) then
    SetLength(FPlugins, Length(FPlugins) * 2);
  FPlugins[FCount] := Plugin;
  Inc(FCount);
end;

procedure TPluginManager.LoadPlugin(Plugin: IPlugin);
begin
  RegisterPlugin(Plugin);
  WriteLn('โหลด plugin: ', Plugin.GetInfo.Name, ' v', Plugin.GetInfo.Version);
end;

procedure TPluginManager.UnloadAll;
var
  I: Integer;
begin
  for I := 0 to FCount - 1 do
    FPlugins[I].Shutdown;
  FCount := 0;
end;

function TPluginManager.GetPlugin(const Name: string): IPlugin;
var
  I: Integer;
begin
  Result := nil;
  for I := 0 to FCount - 1 do
    if SameText(FPlugins[I].GetInfo.Name, Name) then
    begin
      Result := FPlugins[I];
      Exit;
    end;
end;

function TPluginManager.GetTextPlugins: TArray<ITextPlugin>;
var
  I, Count: Integer;
  TextPlugin: ITextPlugin;
begin
  Count := 0;
  SetLength(Result, FCount);
  for I := 0 to FCount - 1 do
    if Supports(FPlugins[I], ITextPlugin, TextPlugin) then
    begin
      Result[Count] := TextPlugin;
      Inc(Count);
    end;
  SetLength(Result, Count);
end;

procedure TPluginManager.InitializeAll;
var
  I: Integer;
begin
  WriteLn('=== Initializing Plugins ===');
  for I := 0 to FCount - 1 do
    FPlugins[I].Initialize;
  WriteLn;
end;

procedure TPluginManager.ShowLoadedPlugins;
var
  I: Integer;
  Info: TPluginInfo;
begin
  WriteLn('=== Loaded Plugins (', FCount, ') ===');
  for I := 0 to FCount - 1 do
  begin
    Info := FPlugins[I].GetInfo;
    WriteLn('  [', IfThen(FPlugins[I].IsInitialized, '✓', '✗'), '] ',
            Info.Name, ' v', Info.Version,
            ' by ', Info.Author);
    WriteLn('      ', Info.Description);
  end;
  WriteLn;
end;

procedure TPluginManager.ProcessText(const Input: string);
var
  TextPlugins: TArray<ITextPlugin>;
  Plugin: ITextPlugin;
begin
  WriteLn('=== Process Text ===');
  WriteLn('Input: "', Input, '"');
  WriteLn;
  TextPlugins := GetTextPlugins;
  for Plugin in TextPlugins do
  begin
    if Plugin.IsInitialized then
    begin
      var Output := Plugin.ProcessText(Input);
      WriteLn('[', (Plugin as IPlugin).GetInfo.Name, '] -> "', Output, '"');
    end;
  end;
end;

function TPluginManager.GetCount: Integer;
begin
  Result := FCount;
end;

// ===== Main =====
var
  Manager: TPluginManager;
begin
  WriteLn('===== Plugin System =====');
  WriteLn;

  Manager := TPluginManager.Create;

  // โหลด plugins
  Manager.LoadPlugin(TEncryptPlugin.Create(99));
  Manager.LoadPlugin(TMarkdownPlugin.Create);
  Manager.LoadPlugin(TWordCountPlugin.Create);
  WriteLn;

  // Initialize
  Manager.InitializeAll;

  // แสดงรายการ
  Manager.ShowLoadedPlugins;

  // ประมวลผล text
  Manager.ProcessText('**Hello World** สวัสดีชาวโลก ทดสอบ Plugin System');
  WriteLn;

  // ใช้ plugin เฉพาะ
  var Plugin := Manager.GetPlugin('Word Count');
  if Plugin <> nil then
  begin
    var TextPlugin := Plugin as ITextPlugin;
    WriteLn('=== Word Count Plugin ===');
    WriteLn(TextPlugin.ProcessText('the quick brown fox jumps over the lazy dog'));
  end;

  Manager.UnloadAll;
  Manager.Free;

  ReadLn;
end.
```

---

## 25.13 โปรแกรมตัวอย่าง: Event System with Interfaces

```pascal
program EventSystem;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TEventData = class
  private
    FSource: TObject;
    FTimestamp: TDateTime;
    FEventName: string;
  public
    constructor Create(ASource: TObject; const AName: string);
    property Source: TObject read FSource;
    property Timestamp: TDateTime read FTimestamp;
    property EventName: string read FEventName;
  end;

  TClickEventData = class(TEventData)
  private
    FX, FY: Integer;
  public
    constructor Create(ASource: TObject; AX, AY: Integer);
    property X: Integer read FX;
    property Y: Integer read FY;
  end;

  TDataChangedEventData = class(TEventData)
  private
    FOldValue: string;
    FNewValue: string;
    FFieldName: string;
  public
    constructor Create(ASource: TObject; const AField, AOld, ANew: string);
    property OldValue: string read FOldValue;
    property NewValue: string read FNewValue;
    property FieldName: string read FFieldName;
  end;

  // Interface สำหรับ event handlers
  IEventHandler = interface
    ['{EVENT001-0000-0000-0000-000000000001}']
    procedure HandleEvent(EventData: TEventData);
    function GetHandlerName: string;
  end;

  // Interface สำหรับ event source
  IEventSource = interface
    ['{EVENT002-0000-0000-0000-000000000001}']
    procedure Subscribe(const EventName: string; Handler: IEventHandler);
    procedure Unsubscribe(const EventName: string; Handler: IEventHandler);
    procedure Fire(const EventName: string; EventData: TEventData);
    function GetEventNames: TStringArray;
  end;

  // Concrete event handlers
  TLogHandler = class(TInterfacedObject, IEventHandler)
  private
    FName: string;
  public
    constructor Create(const AName: string);
    procedure HandleEvent(EventData: TEventData);
    function GetHandlerName: string;
  end;

  TAlertHandler = class(TInterfacedObject, IEventHandler)
  private
    FThreshold: Integer;
    FAlertCount: Integer;
  public
    constructor Create(AThreshold: Integer);
    procedure HandleEvent(EventData: TEventData);
    function GetHandlerName: string;
    property AlertCount: Integer read FAlertCount;
  end;

  TStatisticsHandler = class(TInterfacedObject, IEventHandler)
  private
    FEventCounts: array[0..99] of record Name: string; Count: Integer; end;
    FHandled: Integer;
  public
    constructor Create;
    procedure HandleEvent(EventData: TEventData);
    function GetHandlerName: string;
    procedure ShowStats;
    property HandledCount: Integer read FHandled;
  end;

  // Event Bus (Central event dispatcher)
  TEventBus = class(TInterfacedObject, IEventSource)
  private
    type
      TSubscription = record
        EventName: string;
        Handler: IEventHandler;
      end;
    var
      FSubscriptions: array of TSubscription;
      FSubCount: Integer;
      FFireDepth: Integer;
      FEventLog: array of string;
      FLogCount: Integer;
  public
    constructor Create;
    procedure Subscribe(const EventName: string; Handler: IEventHandler);
    procedure Unsubscribe(const EventName: string; Handler: IEventHandler);
    procedure Fire(const EventName: string; EventData: TEventData);
    function GetEventNames: TStringArray;
    procedure ShowLog;
    function GetEventCount: Integer;
  end;

  // UI Controls ที่ fire events
  TUIButton = class(TInterfacedObject)
  private
    FCaption: string;
    FEventBus: IEventSource;
    FClickCount: Integer;
  public
    constructor Create(const ACaption: string; ABus: IEventSource);
    procedure Click(X, Y: Integer);
    property Caption: string read FCaption;
    property ClickCount: Integer read FClickCount;
  end;

  TDataModel = class(TInterfacedObject)
  private
    FName: string;
    FValue: string;
    FEventBus: IEventSource;
  public
    constructor Create(const AName: string; ABus: IEventSource);
    procedure SetValue(const NewValue: string);
    property Name: string read FName;
    property Value: string read FValue;
  end;

// ===== Event Data =====
constructor TEventData.Create(ASource: TObject; const AName: string);
begin
  inherited Create;
  FSource := ASource;
  FTimestamp := Now;
  FEventName := AName;
end;

constructor TClickEventData.Create(ASource: TObject; AX, AY: Integer);
begin
  inherited Create(ASource, 'click');
  FX := AX;
  FY := AY;
end;

constructor TDataChangedEventData.Create(ASource: TObject; const AField, AOld, ANew: string);
begin
  inherited Create(ASource, 'dataChanged');
  FFieldName := AField;
  FOldValue := AOld;
  FNewValue := ANew;
end;

// ===== Handlers =====
constructor TLogHandler.Create(const AName: string);
begin
  inherited Create;
  FName := AName;
end;

procedure TLogHandler.HandleEvent(EventData: TEventData);
begin
  Write('[LOG/', FName, '] ', FormatDateTime('hh:nn:ss', EventData.Timestamp));
  Write(' Event: ', EventData.EventName);
  if EventData is TClickEventData then
  begin
    var CE := TClickEventData(EventData);
    WriteLn(' at (', CE.X, ',', CE.Y, ')');
  end
  else if EventData is TDataChangedEventData then
  begin
    var DE := TDataChangedEventData(EventData);
    WriteLn(' ', DE.FieldName, ': "', DE.OldValue, '" -> "', DE.NewValue, '"');
  end
  else
    WriteLn;
end;

function TLogHandler.GetHandlerName: string;
begin
  Result := 'Logger/' + FName;
end;

constructor TAlertHandler.Create(AThreshold: Integer);
begin
  inherited Create;
  FThreshold := AThreshold;
  FAlertCount := 0;
end;

procedure TAlertHandler.HandleEvent(EventData: TEventData);
begin
  if EventData.EventName = 'click' then
  begin
    Inc(FAlertCount);
    if FAlertCount >= FThreshold then
    begin
      WriteLn('[ALERT] คลิกครบ ', FAlertCount, ' ครั้ง! (threshold=', FThreshold, ')');
      FAlertCount := 0;
    end;
  end;
end;

function TAlertHandler.GetHandlerName: string;
begin
  Result := Format('Alert(threshold=%d)', [FThreshold]);
end;

constructor TStatisticsHandler.Create;
begin
  inherited Create;
  FHandled := 0;
end;

procedure TStatisticsHandler.HandleEvent(EventData: TEventData);
var
  I: Integer;
begin
  Inc(FHandled);
  for I := 0 to 99 do
  begin
    if FEventCounts[I].Name = EventData.EventName then
    begin
      Inc(FEventCounts[I].Count);
      Exit;
    end;
    if FEventCounts[I].Name = '' then
    begin
      FEventCounts[I].Name := EventData.EventName;
      FEventCounts[I].Count := 1;
      Exit;
    end;
  end;
end;

function TStatisticsHandler.GetHandlerName: string;
begin
  Result := 'Statistics';
end;

procedure TStatisticsHandler.ShowStats;
var
  I: Integer;
begin
  WriteLn('=== Event Statistics ===');
  WriteLn('Events handled: ', FHandled);
  for I := 0 to 99 do
  begin
    if FEventCounts[I].Name = '' then Break;
    WriteLn('  ', FEventCounts[I].Name, ': ', FEventCounts[I].Count, ' times');
  end;
end;

// ===== TEventBus =====
constructor TEventBus.Create;
begin
  inherited Create;
  FSubCount := 0;
  FFireDepth := 0;
  FLogCount := 0;
  SetLength(FSubscriptions, 50);
  SetLength(FEventLog, 100);
end;

procedure TEventBus.Subscribe(const EventName: string; Handler: IEventHandler);
begin
  if FSubCount >= Length(FSubscriptions) then
    SetLength(FSubscriptions, Length(FSubscriptions) * 2);
  FSubscriptions[FSubCount].EventName := EventName;
  FSubscriptions[FSubCount].Handler := Handler;
  Inc(FSubCount);
  WriteLn('Subscribe: ', Handler.GetHandlerName, ' -> ', EventName);
end;

procedure TEventBus.Unsubscribe(const EventName: string; Handler: IEventHandler);
var
  I, J: Integer;
begin
  for I := 0 to FSubCount - 1 do
    if (FSubscriptions[I].EventName = EventName) and
       (FSubscriptions[I].Handler = Handler) then
    begin
      for J := I to FSubCount - 2 do
        FSubscriptions[J] := FSubscriptions[J + 1];
      Dec(FSubCount);
      WriteLn('Unsubscribe: ', Handler.GetHandlerName, ' <- ', EventName);
      Exit;
    end;
end;

procedure TEventBus.Fire(const EventName: string; EventData: TEventData);
var
  I: Integer;
begin
  Inc(FFireDepth);
  try
    // Log event
    if FLogCount < Length(FEventLog) then
    begin
      FEventLog[FLogCount] := Format('[%s] %s',
        [FormatDateTime('hh:nn:ss', Now), EventName]);
      Inc(FLogCount);
    end;

    // Dispatch ให้ handlers ทุกตัวที่ subscribe
    for I := 0 to FSubCount - 1 do
      if (FSubscriptions[I].EventName = EventName) or
         (FSubscriptions[I].EventName = '*') then
        FSubscriptions[I].Handler.HandleEvent(EventData);
  finally
    Dec(FFireDepth);
  end;
end;

function TEventBus.GetEventNames: TStringArray;
var
  I, Count: Integer;
  Names: array of string;
begin
  SetLength(Names, FSubCount);
  Count := 0;
  for I := 0 to FSubCount - 1 do
  begin
    var Found := False;
    var J: Integer;
    for J := 0 to Count - 1 do
      if Names[J] = FSubscriptions[I].EventName then
      begin
        Found := True;
        Break;
      end;
    if not Found then
    begin
      Names[Count] := FSubscriptions[I].EventName;
      Inc(Count);
    end;
  end;
  SetLength(Result, Count);
  Move(Names[0], Result[0], Count * SizeOf(string));
end;

procedure TEventBus.ShowLog;
var
  I: Integer;
begin
  WriteLn('=== Event Log (', FLogCount, ' events) ===');
  for I := 0 to FLogCount - 1 do
    WriteLn('  ', FEventLog[I]);
end;

function TEventBus.GetEventCount: Integer;
begin
  Result := FLogCount;
end;

// ===== UI Controls =====
constructor TUIButton.Create(const ACaption: string; ABus: IEventSource);
begin
  inherited Create;
  FCaption := ACaption;
  FEventBus := ABus;
  FClickCount := 0;
end;

procedure TUIButton.Click(X, Y: Integer);
var
  EventData: TClickEventData;
begin
  Inc(FClickCount);
  EventData := TClickEventData.Create(Self, X, Y);
  try
    FEventBus.Fire('click', EventData);
    FEventBus.Fire('button.click', EventData);
  finally
    EventData.Free;
  end;
end;

constructor TDataModel.Create(const AName: string; ABus: IEventSource);
begin
  inherited Create;
  FName := AName;
  FEventBus := ABus;
  FValue := '';
end;

procedure TDataModel.SetValue(const NewValue: string);
var
  EventData: TDataChangedEventData;
begin
  if FValue = NewValue then Exit;
  EventData := TDataChangedEventData.Create(Self, FName, FValue, NewValue);
  FValue := NewValue;
  try
    FEventBus.Fire('dataChanged', EventData);
    FEventBus.Fire('model.' + FName + '.changed', EventData);
  finally
    EventData.Free;
  end;
end;

// ===== Main =====
var
  Bus: TEventBus;
  Logger: TLogHandler;
  Alert: TAlertHandler;
  Stats: TStatisticsHandler;
  Button1, Button2: TUIButton;
  Model: TDataModel;
begin
  WriteLn('===== Event System with Interfaces =====');
  WriteLn;

  Bus := TEventBus.Create;

  // สร้าง handlers
  Logger := TLogHandler.Create('Main');
  Alert := TAlertHandler.Create(3);
  Stats := TStatisticsHandler.Create;

  // Subscribe events
  WriteLn('--- Subscribing ---');
  Bus.Subscribe('click', Logger);
  Bus.Subscribe('click', Alert);
  Bus.Subscribe('*', Stats);        // Subscribe ทุก events
  Bus.Subscribe('dataChanged', Logger);
  WriteLn;

  // สร้าง UI controls
  Button1 := TUIButton.Create('ปุ่มตกลง', Bus);
  Button2 := TUIButton.Create('ปุ่มยกเลิก', Bus);
  Model := TDataModel.Create('username', Bus);

  // Simulate interactions
  WriteLn('--- Interactions ---');
  Button1.Click(100, 200);
  Button1.Click(105, 202);
  Button2.Click(200, 200);
  Button1.Click(100, 200);  // Alert เมื่อคลิกครบ 3 ครั้ง
  WriteLn;

  Model.SetValue('สมชาย');
  Model.SetValue('สมชาย ใจดี');
  Model.SetValue('สมชาย ใจดี');  // ค่าเดิม ไม่ fire event
  WriteLn;

  // Unsubscribe
  Bus.Unsubscribe('click', Alert);
  Button1.Click(100, 200);  // Alert ไม่รับแล้ว
  WriteLn;

  // แสดงสถิติ
  Stats.ShowStats;
  WriteLn;
  Bus.ShowLog;
  WriteLn;
  WriteLn('Button1 คลิก ', Button1.ClickCount, ' ครั้ง');
  WriteLn('Button2 คลิก ', Button2.ClickCount, ' ครั้ง');
  WriteLn('Model value: "', Model.Value, '"');

  Button1.Free;
  Button2.Free;
  Model.Free;
  Logger.Free;
  Alert.Free;
  Stats.Free;
  Bus.Free;

  ReadLn;
end.
```

---

## 25.14 แบบฝึกหัด 15 ข้อ

**ข้อ 1:** สร้าง `IComparable<T>` interface สำหรับ generic comparison พร้อม class ที่ implement สำหรับ Integer, String, Date

**ข้อ 2:** ออกแบบ `IRepository<T>` interface สำหรับ CRUD operations พร้อม `TInMemoryRepository<T>` implementation

**ข้อ 3:** สร้าง `IObserver` และ `IObservable` interfaces ตาม Observer pattern พร้อม type-safe variant

**ข้อ 4:** สร้าง `IConverter<TFrom, TTo>` interface สำหรับ type conversion พร้อม converters หลายชนิด

**ข้อ 5:** ออกแบบ `IBuilder<T>` interface พร้อม concrete builders สำหรับสร้าง complex objects

**ข้อ 6:** สร้าง `ICache<K, V>` interface พร้อม `TMemoryCache`, `TFileCache` implementations

**ข้อ 7:** ออกแบบ `IValidator<T>` interface พร้อม composite validator ที่ chain validators

**ข้อ 8:** สร้าง `ILogger` interface ที่ structured logging พร้อม multiple sinks (Console, File, Memory)

**ข้อ 9:** สร้าง `ITransactionManager` interface พร้อม `ITransaction` สำหรับ unit of work pattern

**ข้อ 10:** ออกแบบ `IMediator` interface ตาม Mediator pattern สำหรับลด coupling ระหว่าง objects

**ข้อ 11:** สร้าง `ICommand` และ `ICommandHandler` interfaces ตาม CQRS pattern

**ข้อ 12:** ออกแบบ `ISpecification<T>` interface สำหรับ business rules เป็น objects

**ข้อ 13:** สร้าง `ILifecycle` interface สำหรับ manage object lifecycle (Initialize, Start, Stop, Destroy)

**ข้อ 14:** ออกแบบ Plugin system ที่ complete กว่าตัวอย่าง พร้อม version checking และ dependency management

**ข้อ 15:** สร้าง complete Event Sourcing system ด้วย `IEvent`, `IEventStore`, `IEventHandler` และ replay capability

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Interface Declaration** - syntax และ GUID
2. **Interface Implementation** - การ implement single และ multiple interfaces
3. **Reference Counting** - การจัดการ memory อัตโนมัติผ่าน TInterfacedObject
4. **IInterface/IUnknown** - base interface และ QueryInterface
5. **Interface Casting** - Supports function และ as operator
6. **Interface Hierarchy** - interfaces สืบทอดจากกัน
7. **implements Keyword** - delegation pattern
8. **GUID** - ระบุ interface อย่างไม่ซ้ำกัน
9. **COM Compatibility** - interfaces ที่ทำงานกับ COM
10. **Interface vs Abstract Class** - เมื่อใช้อะไร

## ทบทวนบทที่ 21-25

- **Part 21**: OOP พื้นฐาน - Class, Object, Encapsulation, Inheritance, Polymorphism, Abstraction
- **Part 22**: Classes เชิงลึก - Properties ขั้นสูง, RTTI, Memory Management, Patterns
- **Part 23**: Inheritance - Virtual methods, Abstract, Constructor/Destructor chaining, LSP
- **Part 24**: Polymorphism - Virtual dispatch, Strategy Pattern, Factory Pattern, Collections
- **Part 25**: Interfaces - Declaration, Reference Counting, Multiple implementations, COM

เนื้อหาเหล่านี้เป็นพื้นฐานสำคัญของ Object-Oriented Programming ใน Free Pascal/Lazarus บทต่อไปจะเรียนเรื่อง Advanced OOP features เช่น Generics, Operator Overloading และ Design Patterns
