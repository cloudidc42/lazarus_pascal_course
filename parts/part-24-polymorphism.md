# Part 24 - Polymorphism (พหุสัณฐาน)

## บทนำ

Polymorphism (พหุสัณฐาน) หมายถึงความสามารถของ objects ประเภทต่างๆ ในการตอบสนองต่อ interface เดียวกัน แต่ในลักษณะที่แตกต่างกัน คำว่า "Polymorphism" มาจากภาษากรีก "poly" (หลาย) + "morph" (รูปแบบ)

ใน Pascal/Free Pascal มี polymorphism สองแบบหลัก:
1. **Compile-time Polymorphism** - Method Overloading, Operator Overloading
2. **Runtime Polymorphism** - Virtual Methods, Interface-based

---

## 24.1 Virtual Dispatch

Virtual dispatch คือกลไกที่ Pascal ใช้ตัดสินว่าจะเรียก method ไหนขณะ runtime

### Virtual Dispatch Table (VMT)

ทุก class ที่มี virtual methods จะมี Virtual Method Table (VMT) ที่เก็บ pointers ไปยัง method implementations

```pascal
type
  TAnimal = class
    procedure Sound; virtual;  // VMT entry สำหรับ Sound
    procedure Move; virtual;   // VMT entry สำหรับ Move
  end;

  TDog = class(TAnimal)
    procedure Sound; override;  // แทนที่ VMT entry ของ Sound
    // Move ไม่ได้ override ใช้ pointer ของ TAnimal
  end;

  TCat = class(TAnimal)
    procedure Sound; override;
    procedure Move; override;
  end;
```

### ตัวอย่าง Virtual Dispatch

```pascal
program VirtualDispatch;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TShape = class
  private
    FX, FY: Double;
  public
    constructor Create(AX, AY: Double);
    function Area: Double; virtual; abstract;
    function Name: string; virtual; abstract;
    procedure Describe;  // Non-virtual - เรียก virtual methods ภายใน
    procedure Draw; virtual;
    property X: Double read FX;
    property Y: Double read FY;
  end;

  TCircle = class(TShape)
  private
    FRadius: Double;
  public
    constructor Create(AX, AY, ARadius: Double);
    function Area: Double; override;
    function Name: string; override;
    procedure Draw; override;
    property Radius: Double read FRadius;
  end;

  TRectangle = class(TShape)
  private
    FWidth, FHeight: Double;
  public
    constructor Create(AX, AY, AWidth, AHeight: Double);
    function Area: Double; override;
    function Name: string; override;
    procedure Draw; override;
  end;

  TTriangle = class(TShape)
  private
    FB, FH: Double;  // Base, Height
  public
    constructor Create(AX, AY, ABase, AHeight: Double);
    function Area: Double; override;
    function Name: string; override;
    procedure Draw; override;
  end;

constructor TShape.Create(AX, AY: Double);
begin
  inherited Create;
  FX := AX;
  FY := AY;
end;

procedure TShape.Describe;
begin
  // เรียก virtual methods - จะ dispatch ตาม type จริง
  WriteLn(Name, ' ที่ (', FX:0:1, ',', FY:0:1, '): พื้นที่=', Area:0:4);
end;

procedure TShape.Draw;
begin
  WriteLn('วาด ', Name, ' ที่ (', FX:0:1, ',', FY:0:1, ')');
end;

constructor TCircle.Create(AX, AY, ARadius: Double);
begin
  inherited Create(AX, AY);
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

procedure TCircle.Draw;
begin
  WriteLn('⭕ วาดวงกลม รัศมี=', FRadius:0:1, ' ที่ (', X:0:1, ',', Y:0:1, ')');
end;

constructor TRectangle.Create(AX, AY, AWidth, AHeight: Double);
begin
  inherited Create(AX, AY);
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

procedure TRectangle.Draw;
begin
  WriteLn('▭ วาดสี่เหลี่ยม ', FWidth:0:1, 'x', FHeight:0:1, ' ที่ (', X:0:1, ',', Y:0:1, ')');
end;

constructor TTriangle.Create(AX, AY, ABase, AHeight: Double);
begin
  inherited Create(AX, AY);
  FB := ABase;
  FH := AHeight;
end;

function TTriangle.Area: Double;
begin
  Result := 0.5 * FB * FH;
end;

function TTriangle.Name: string;
begin
  Result := 'สามเหลี่ยม';
end;

procedure TTriangle.Draw;
begin
  WriteLn('△ วาดสามเหลี่ยม ฐาน=', FB:0:1, ' สูง=', FH:0:1, ' ที่ (', X:0:1, ',', Y:0:1, ')');
end;

// Demo polymorphism
procedure DrawAllShapes(Shapes: array of TShape);
var
  S: TShape;
  TotalArea: Double;
begin
  TotalArea := 0;
  for S in Shapes do
  begin
    S.Describe;    // virtual dispatch ผ่าน TShape reference
    S.Draw;        // virtual dispatch
    TotalArea := TotalArea + S.Area;
  end;
  WriteLn('พื้นที่รวม: ', TotalArea:0:4);
end;

var
  Shapes: array[0..4] of TShape;
begin
  Shapes[0] := TCircle.Create(0, 0, 5);
  Shapes[1] := TRectangle.Create(10, 0, 8, 6);
  Shapes[2] := TTriangle.Create(20, 0, 10, 8);
  Shapes[3] := TCircle.Create(0, 20, 3);
  Shapes[4] := TRectangle.Create(10, 20, 5, 5);

  WriteLn('=== ทดสอบ Virtual Dispatch ===');
  DrawAllShapes(Shapes);

  var I: Integer;
  for I := 0 to 4 do Shapes[I].Free;
  ReadLn;
end.
```

---

## 24.2 Dynamic Binding vs Static Binding

### Static Binding
ตัดสินใจตอน compile time ตาม declared type ของ variable

```pascal
type
  TBase = class
    procedure Method;         // static
    procedure VMethod; virtual;  // virtual
  end;
  TChild = class(TBase)
    procedure Method;         // hides, static
    procedure VMethod; override;  // dynamic
  end;

var
  B: TBase;
  C: TChild;
begin
  C := TChild.Create;
  B := C;

  B.Method;   // เรียก TBase.Method (static - based on declared type TBase)
  B.VMethod;  // เรียก TChild.VMethod (dynamic - based on actual type TChild)

  C.Free;
end;
```

### เปรียบเทียบ Static vs Dynamic

```pascal
program StaticVsDynamic;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TGreeter = class
    // Static method
    procedure Greet;
    // Dynamic method
    procedure VGreet; virtual;
  end;

  TFormalGreeter = class(TGreeter)
    procedure Greet;       // Static - hides parent
    procedure VGreet; override;  // Dynamic - overrides parent
  end;

  TInformalGreeter = class(TGreeter)
    procedure Greet;       // Static
    procedure VGreet; override;  // Dynamic
  end;

procedure TGreeter.Greet;
begin
  WriteLn('สวัสดี (TGreeter.Greet)');
end;

procedure TGreeter.VGreet;
begin
  WriteLn('สวัสดี (TGreeter.VGreet)');
end;

procedure TFormalGreeter.Greet;
begin
  WriteLn('สวัสดีครับ/ค่ะ ท่านที่เคารพ (TFormalGreeter.Greet)');
end;

procedure TFormalGreeter.VGreet;
begin
  WriteLn('สวัสดีครับ/ค่ะ ท่านที่เคารพ (TFormalGreeter.VGreet)');
end;

procedure TInformalGreeter.Greet;
begin
  WriteLn('หวัดดี! (TInformalGreeter.Greet)');
end;

procedure TInformalGreeter.VGreet;
begin
  WriteLn('หวัดดี! (TInformalGreeter.VGreet)');
end;

procedure TestGreeting(G: TGreeter);
begin
  G.Greet;   // Static - always calls TGreeter.Greet!
  G.VGreet;  // Dynamic - calls actual implementation
end;

var
  Formal: TFormalGreeter;
  Informal: TInformalGreeter;
begin
  Formal := TFormalGreeter.Create;
  Informal := TInformalGreeter.Create;

  WriteLn('=== ทดสอบผ่าน TGreeter reference ===');
  WriteLn('Formal:');
  TestGreeting(Formal);
  WriteLn('Informal:');
  TestGreeting(Informal);
  WriteLn;

  WriteLn('=== ทดสอบผ่าน specific type ===');
  Formal.Greet;    // Static - TFormalGreeter.Greet
  Formal.VGreet;   // Dynamic - TFormalGreeter.VGreet
  WriteLn;
  Informal.Greet;
  Informal.VGreet;

  Formal.Free;
  Informal.Free;
  ReadLn;
end.
```

---

## 24.3 Override และ Virtual Keywords

```pascal
type
  TLevel1 = class
    procedure Method1; virtual;           // virtual ครั้งแรก
    procedure Method2; virtual;
    procedure Method3; virtual;
  end;

  TLevel2 = class(TLevel1)
    procedure Method1; override;          // override
    procedure Method2; virtual; override; // virtual + override (ทำให้ override ได้ต่อ)
    // Method3 ไม่ได้ override
  end;

  TLevel3 = class(TLevel2)
    procedure Method1; override;          // override TLevel2 (ซึ่ง override TLevel1)
    procedure Method2; override;          // override TLevel2
    // Method3 ไม่ได้ override (ยังใช้ TLevel1.Method3)
  end;
```

### reintroduce keyword
ใช้เมื่อต้องการ "ซ่อน" virtual method ของ parent โดยไม่ override

```pascal
type
  TBase = class
    procedure Method; virtual;
  end;

  TChild = class(TBase)
    // ซ่อน TBase.Method โดยไม่ใช้ virtual dispatch
    procedure Method; reintroduce;
  end;
```

---

## 24.4 Abstract Classes และ Methods

### Abstract Method
ประกาศว่ามี method แต่ไม่มี implementation กำหนดให้ subclass implement

```pascal
type
  TAbstractProcessor = class
  public
    // Abstract methods - ต้อง override
    function Process(const Input: string): string; virtual; abstract;
    function Validate(const Input: string): Boolean; virtual; abstract;
    function GetName: string; virtual; abstract;

    // Template method - กำหนดขั้นตอน
    function Execute(const Input: string): string;
  end;

function TAbstractProcessor.Execute(const Input: string): string;
begin
  WriteLn('=== ', GetName, ' ===');
  if not Validate(Input) then
    raise Exception.Create('Input ไม่ถูกต้อง');
  Result := Process(Input);
  WriteLn('ผลลัพธ์: ', Result);
end;

// ต้อง implement abstract methods ทั้งหมดจึงจะ instantiate ได้
type
  TUpperCaseProcessor = class(TAbstractProcessor)
  public
    function Process(const Input: string): string; override;
    function Validate(const Input: string): Boolean; override;
    function GetName: string; override;
  end;

function TUpperCaseProcessor.Process(const Input: string): string;
begin
  Result := UpperCase(Input);
end;

function TUpperCaseProcessor.Validate(const Input: string): Boolean;
begin
  Result := Trim(Input) <> '';
end;

function TUpperCaseProcessor.GetName: string;
begin
  Result := 'Upper Case Processor';
end;
```

---

## 24.5 Interface-Based Polymorphism

Interfaces ให้ polymorphism โดยไม่ต้องมี inheritance hierarchy

```pascal
program InterfacePolymorphism;

{$mode objfpc}{$H+}

uses SysUtils;

type
  // Interface สำหรับ drawable objects
  IDrawable = interface
    ['{11111111-1111-1111-1111-111111111111}']
    procedure Draw;
    function GetBounds: string;
  end;

  // Interface สำหรับ resizable objects
  IResizable = interface
    ['{22222222-2222-2222-2222-222222222222}']
    procedure Resize(Factor: Double);
    function GetSize: Double;
  end;

  // Interface สำหรับ movable objects
  IMovable = interface
    ['{33333333-3333-3333-3333-333333333333}']
    procedure Move(DX, DY: Double);
    function GetPosition: string;
  end;

  // Class ที่ implement ทุก interfaces
  TUIElement = class(TInterfacedObject, IDrawable, IResizable, IMovable)
  private
    FX, FY: Double;
    FWidth, FHeight: Double;
    FName: string;
  public
    constructor Create(const AName: string; X, Y, W, H: Double);

    // IDrawable
    procedure Draw;
    function GetBounds: string;

    // IResizable
    procedure Resize(Factor: Double);
    function GetSize: Double;

    // IMovable
    procedure Move(DX, DY: Double);
    function GetPosition: string;

    property Name: string read FName;
  end;

  // Class ที่ implement เฉพาะบาง interfaces
  TBackgroundImage = class(TInterfacedObject, IDrawable)
  private
    FImageFile: string;
  public
    constructor Create(const AImageFile: string);
    procedure Draw;
    function GetBounds: string;
  end;

  TAnimation = class(TInterfacedObject, IDrawable, IMovable)
  private
    FX, FY: Double;
    FFrameCount: Integer;
    FCurrentFrame: Integer;
  public
    constructor Create(X, Y: Double; Frames: Integer);
    procedure Draw;
    function GetBounds: string;
    procedure Move(DX, DY: Double);
    function GetPosition: string;
    procedure NextFrame;
  end;

constructor TUIElement.Create(const AName: string; X, Y, W, H: Double);
begin
  inherited Create;
  FName := AName;
  FX := X; FY := Y;
  FWidth := W; FHeight := H;
end;

procedure TUIElement.Draw;
begin
  WriteLn('[UIElement] วาด "', FName, '" ที่ (', FX:0:1, ',', FY:0:1,
          ') ขนาด ', FWidth:0:1, 'x', FHeight:0:1);
end;

function TUIElement.GetBounds: string;
begin
  Result := Format('(%.1f,%.1f,%.1f,%.1f)', [FX, FY, FWidth, FHeight]);
end;

procedure TUIElement.Resize(Factor: Double);
begin
  FWidth := FWidth * Factor;
  FHeight := FHeight * Factor;
  WriteLn('[UIElement] ขยาย "', FName, '" x', Factor:0:2, ' -> ', FWidth:0:1, 'x', FHeight:0:1);
end;

function TUIElement.GetSize: Double;
begin
  Result := FWidth * FHeight;
end;

procedure TUIElement.Move(DX, DY: Double);
begin
  FX := FX + DX;
  FY := FY + DY;
  WriteLn('[UIElement] ย้าย "', FName, '" ไป (', FX:0:1, ',', FY:0:1, ')');
end;

function TUIElement.GetPosition: string;
begin
  Result := Format('(%.1f,%.1f)', [FX, FY]);
end;

constructor TBackgroundImage.Create(const AImageFile: string);
begin
  inherited Create;
  FImageFile := AImageFile;
end;

procedure TBackgroundImage.Draw;
begin
  WriteLn('[Background] วาดรูป: ', FImageFile);
end;

function TBackgroundImage.GetBounds: string;
begin
  Result := '(0,0,1920,1080)';
end;

constructor TAnimation.Create(X, Y: Double; Frames: Integer);
begin
  inherited Create;
  FX := X; FY := Y;
  FFrameCount := Frames;
  FCurrentFrame := 0;
end;

procedure TAnimation.Draw;
begin
  WriteLn('[Animation] Frame ', FCurrentFrame + 1, '/', FFrameCount,
          ' ที่ (', FX:0:1, ',', FY:0:1, ')');
end;

function TAnimation.GetBounds: string;
begin
  Result := Format('(%.1f,%.1f)', [FX, FY]);
end;

procedure TAnimation.Move(DX, DY: Double);
begin
  FX := FX + DX;
  FY := FY + DY;
  WriteLn('[Animation] ย้ายไป (', FX:0:1, ',', FY:0:1, ')');
end;

function TAnimation.GetPosition: string;
begin
  Result := Format('(%.1f,%.1f)', [FX, FY]);
end;

procedure TAnimation.NextFrame;
begin
  FCurrentFrame := (FCurrentFrame + 1) mod FFrameCount;
end;

// Functions ที่รับ interface (polymorphic)
procedure DrawAll(Drawables: array of IDrawable);
var
  D: IDrawable;
begin
  WriteLn('=== วาดทุกอย่าง ===');
  for D in Drawables do
    D.Draw;
end;

procedure MoveAll(Movables: array of IMovable; DX, DY: Double);
var
  M: IMovable;
begin
  WriteLn('=== ย้ายทุกอย่าง ===');
  for M in Movables do
    M.Move(DX, DY);
end;

var
  Btn: TUIElement;
  Panel: TUIElement;
  BG: TBackgroundImage;
  Anim: TAnimation;
  Drawables: array of IDrawable;
  Movables: array of IMovable;
begin
  Btn := TUIElement.Create('ปุ่มตกลง', 100, 200, 120, 40);
  Panel := TUIElement.Create('Panel หลัก', 0, 0, 800, 600);
  BG := TBackgroundImage.Create('background.png');
  Anim := TAnimation.Create(300, 300, 5);

  // Polymorphism ผ่าน IDrawable
  SetLength(Drawables, 4);
  Drawables[0] := BG;   // TBackgroundImage
  Drawables[1] := Panel; // TUIElement
  Drawables[2] := Btn;   // TUIElement
  Drawables[3] := Anim;  // TAnimation

  DrawAll(Drawables);
  WriteLn;

  // Polymorphism ผ่าน IMovable
  SetLength(Movables, 3);
  Movables[0] := Btn;   // TUIElement (implements IMovable)
  Movables[1] := Panel; // TUIElement
  Movables[2] := Anim;  // TAnimation

  MoveAll(Movables, 10, 5);
  WriteLn;

  // Resize ผ่าน IResizable
  WriteLn('=== Resize ===');
  var Resizable: IResizable := Btn;
  Resizable.Resize(1.5);

  Anim.NextFrame;
  Anim.Draw;

  // TInterfacedObject จัดการ memory ให้อัตโนมัติ
  // ไม่จำเป็นต้อง Free ถ้าใช้ผ่าน interface reference
  // แต่ถ้าต้อง Free ให้ใช้ FreeAndNil หรือ Free ตรงๆ
  Btn.Free;
  Panel.Free;
  BG.Free;
  Anim.Free;

  ReadLn;
end.
```

---

## 24.6 Collections ของ Polymorphic Objects

```pascal
program PolymorphicCollections;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TDataProcessor = class
  public
    function ProcessData(const Data: string): string; virtual; abstract;
    function GetProcessorName: string; virtual; abstract;
  end;

  TUpperCaseProcessor = class(TDataProcessor)
  public
    function ProcessData(const Data: string): string; override;
    function GetProcessorName: string; override;
  end;

  TLowerCaseProcessor = class(TDataProcessor)
  public
    function ProcessData(const Data: string): string; override;
    function GetProcessorName: string; override;
  end;

  TTrimProcessor = class(TDataProcessor)
  public
    function ProcessData(const Data: string): string; override;
    function GetProcessorName: string; override;
  end;

  TReverseProcessor = class(TDataProcessor)
  public
    function ProcessData(const Data: string): string; override;
    function GetProcessorName: string; override;
  end;

  TProcessorPipeline = class
  private
    FProcessors: array of TDataProcessor;
    FCount: Integer;
    FOwnProcessors: Boolean;
  public
    constructor Create(AOwnProcessors: Boolean = True);
    destructor Destroy; override;
    procedure Add(Processor: TDataProcessor);
    function Execute(const Input: string): string;
    procedure ShowPipeline;
  end;

function TUpperCaseProcessor.ProcessData(const Data: string): string;
begin
  Result := UpperCase(Data);
end;

function TUpperCaseProcessor.GetProcessorName: string;
begin
  Result := 'UpperCase';
end;

function TLowerCaseProcessor.ProcessData(const Data: string): string;
begin
  Result := LowerCase(Data);
end;

function TLowerCaseProcessor.GetProcessorName: string;
begin
  Result := 'LowerCase';
end;

function TTrimProcessor.ProcessData(const Data: string): string;
begin
  Result := Trim(Data);
end;

function TTrimProcessor.GetProcessorName: string;
begin
  Result := 'Trim';
end;

function TReverseProcessor.ProcessData(const Data: string): string;
var
  I: Integer;
begin
  Result := '';
  for I := Length(Data) downto 1 do
    Result := Result + Data[I];
end;

function TReverseProcessor.GetProcessorName: string;
begin
  Result := 'Reverse';
end;

constructor TProcessorPipeline.Create(AOwnProcessors: Boolean);
begin
  inherited Create;
  FOwnProcessors := AOwnProcessors;
  FCount := 0;
  SetLength(FProcessors, 8);
end;

destructor TProcessorPipeline.Destroy;
var
  I: Integer;
begin
  if FOwnProcessors then
    for I := 0 to FCount - 1 do
      FProcessors[I].Free;
  inherited;
end;

procedure TProcessorPipeline.Add(Processor: TDataProcessor);
begin
  if FCount >= Length(FProcessors) then
    SetLength(FProcessors, Length(FProcessors) * 2);
  FProcessors[FCount] := Processor;
  Inc(FCount);
end;

function TProcessorPipeline.Execute(const Input: string): string;
var
  I: Integer;
begin
  Result := Input;
  for I := 0 to FCount - 1 do
  begin
    Result := FProcessors[I].ProcessData(Result);  // Polymorphic call!
    WriteLn('  หลัง ', FProcessors[I].GetProcessorName, ': "', Result, '"');
  end;
end;

procedure TProcessorPipeline.ShowPipeline;
var
  I: Integer;
  Pipeline: string;
begin
  Pipeline := '';
  for I := 0 to FCount - 1 do
  begin
    if I > 0 then Pipeline := Pipeline + ' -> ';
    Pipeline := Pipeline + FProcessors[I].GetProcessorName;
  end;
  WriteLn('Pipeline: ', Pipeline);
end;

var
  Pipeline: TProcessorPipeline;
  Input, Output: string;
begin
  WriteLn('=== Processor Pipeline ===');

  Pipeline := TProcessorPipeline.Create;
  Pipeline.Add(TTrimProcessor.Create);
  Pipeline.Add(TLowerCaseProcessor.Create);
  Pipeline.Add(TReverseProcessor.Create);
  Pipeline.Add(TUpperCaseProcessor.Create);

  Pipeline.ShowPipeline;
  WriteLn;

  Input := '  Hello World  ';
  WriteLn('Input: "', Input, '"');
  Output := Pipeline.Execute(Input);
  WriteLn('Output: "', Output, '"');
  WriteLn;

  Pipeline.Free;
  ReadLn;
end.
```

---

## 24.7 Strategy Pattern ด้วย Polymorphism

```pascal
program StrategyPattern;

{$mode objfpc}{$H+}

uses SysUtils;

type
  // Strategy Interface
  TSortStrategy = class
  public
    procedure Sort(var Arr: array of Integer; N: Integer); virtual; abstract;
    function GetName: string; virtual; abstract;
  end;

  // Concrete Strategies
  TBubbleSort = class(TSortStrategy)
  public
    procedure Sort(var Arr: array of Integer; N: Integer); override;
    function GetName: string; override;
  end;

  TSelectionSort = class(TSortStrategy)
  public
    procedure Sort(var Arr: array of Integer; N: Integer); override;
    function GetName: string; override;
  end;

  TInsertionSort = class(TSortStrategy)
  public
    procedure Sort(var Arr: array of Integer; N: Integer); override;
    function GetName: string; override;
  end;

  TQuickSort = class(TSortStrategy)
  private
    procedure QuickSortHelper(var Arr: array of Integer; Low, High: Integer);
    function Partition(var Arr: array of Integer; Low, High: Integer): Integer;
  public
    procedure Sort(var Arr: array of Integer; N: Integer); override;
    function GetName: string; override;
  end;

  // Context ที่ใช้ Strategy
  TSorter = class
  private
    FStrategy: TSortStrategy;
    FOwnStrategy: Boolean;
  public
    constructor Create(AStrategy: TSortStrategy; AOwnIt: Boolean = False);
    destructor Destroy; override;
    procedure SetStrategy(AStrategy: TSortStrategy; AOwnIt: Boolean = False);
    procedure Sort(var Arr: array of Integer; N: Integer);
    function StrategyName: string;
  end;

// ===== Bubble Sort =====
procedure TBubbleSort.Sort(var Arr: array of Integer; N: Integer);
var
  I, J, Temp: Integer;
  Swapped: Boolean;
begin
  for I := 0 to N - 2 do
  begin
    Swapped := False;
    for J := 0 to N - 2 - I do
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

function TBubbleSort.GetName: string;
begin
  Result := 'Bubble Sort';
end;

// ===== Selection Sort =====
procedure TSelectionSort.Sort(var Arr: array of Integer; N: Integer);
var
  I, J, MinIdx, Temp: Integer;
begin
  for I := 0 to N - 2 do
  begin
    MinIdx := I;
    for J := I + 1 to N - 1 do
      if Arr[J] < Arr[MinIdx] then MinIdx := J;
    if MinIdx <> I then
    begin
      Temp := Arr[I];
      Arr[I] := Arr[MinIdx];
      Arr[MinIdx] := Temp;
    end;
  end;
end;

function TSelectionSort.GetName: string;
begin
  Result := 'Selection Sort';
end;

// ===== Insertion Sort =====
procedure TInsertionSort.Sort(var Arr: array of Integer; N: Integer);
var
  I, J, Key: Integer;
begin
  for I := 1 to N - 1 do
  begin
    Key := Arr[I];
    J := I - 1;
    while (J >= 0) and (Arr[J] > Key) do
    begin
      Arr[J + 1] := Arr[J];
      Dec(J);
    end;
    Arr[J + 1] := Key;
  end;
end;

function TInsertionSort.GetName: string;
begin
  Result := 'Insertion Sort';
end;

// ===== Quick Sort =====
function TQuickSort.Partition(var Arr: array of Integer; Low, High: Integer): Integer;
var
  Pivot, Temp, I, J: Integer;
begin
  Pivot := Arr[High];
  I := Low - 1;
  for J := Low to High - 1 do
  begin
    if Arr[J] <= Pivot then
    begin
      Inc(I);
      Temp := Arr[I]; Arr[I] := Arr[J]; Arr[J] := Temp;
    end;
  end;
  Temp := Arr[I + 1]; Arr[I + 1] := Arr[High]; Arr[High] := Temp;
  Result := I + 1;
end;

procedure TQuickSort.QuickSortHelper(var Arr: array of Integer; Low, High: Integer);
var
  Pi: Integer;
begin
  if Low < High then
  begin
    Pi := Partition(Arr, Low, High);
    QuickSortHelper(Arr, Low, Pi - 1);
    QuickSortHelper(Arr, Pi + 1, High);
  end;
end;

procedure TQuickSort.Sort(var Arr: array of Integer; N: Integer);
begin
  if N > 1 then QuickSortHelper(Arr, 0, N - 1);
end;

function TQuickSort.GetName: string;
begin
  Result := 'Quick Sort';
end;

// ===== TSorter Context =====
constructor TSorter.Create(AStrategy: TSortStrategy; AOwnIt: Boolean);
begin
  inherited Create;
  FStrategy := AStrategy;
  FOwnStrategy := AOwnIt;
end;

destructor TSorter.Destroy;
begin
  if FOwnStrategy then
    FStrategy.Free;
  inherited;
end;

procedure TSorter.SetStrategy(AStrategy: TSortStrategy; AOwnIt: Boolean);
begin
  if FOwnStrategy then FStrategy.Free;
  FStrategy := AStrategy;
  FOwnStrategy := AOwnIt;
end;

procedure TSorter.Sort(var Arr: array of Integer; N: Integer);
begin
  FStrategy.Sort(Arr, N);
end;

function TSorter.StrategyName: string;
begin
  Result := FStrategy.GetName;
end;

// ===== Main =====
procedure PrintArray(const Arr: array of Integer; N: Integer);
var
  I: Integer;
begin
  Write('[');
  for I := 0 to N - 1 do
  begin
    if I > 0 then Write(', ');
    Write(Arr[I]);
  end;
  WriteLn(']');
end;

const
  OrigData: array[0..9] of Integer = (64, 34, 25, 12, 22, 11, 90, 45, 78, 3);

var
  Data: array[0..9] of Integer;
  Sorter: TSorter;
  Strategies: array[0..3] of TSortStrategy;
  I: Integer;
begin
  Strategies[0] := TBubbleSort.Create;
  Strategies[1] := TSelectionSort.Create;
  Strategies[2] := TInsertionSort.Create;
  Strategies[3] := TQuickSort.Create;

  Sorter := TSorter.Create(Strategies[0]);  // เริ่มด้วย Bubble Sort

  for I := 0 to 3 do
  begin
    // รีเซ็ต data
    Move(OrigData, Data, SizeOf(OrigData));

    // เปลี่ยน strategy
    Sorter.SetStrategy(Strategies[I]);

    Write('Input: ');
    PrintArray(Data, 10);

    Sorter.Sort(Data, 10);  // Polymorphic sort

    Write(Sorter.StrategyName, ': ');
    PrintArray(Data, 10);
    WriteLn;
  end;

  Sorter.Free;
  for I := 0 to 3 do Strategies[I].Free;

  ReadLn;
end.
```

---

## 24.8 Factory Pattern

```pascal
program FactoryPattern;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TLogLevel = (llDebug, llInfo, llWarning, llError, llCritical);
  TOutputType = (otConsole, otFile, otDatabase, otNetwork);

  TLogger = class
  private
    FPrefix: string;
    FMinLevel: TLogLevel;
  public
    constructor Create(const APrefix: string; AMinLevel: TLogLevel = llDebug);
    procedure Log(Level: TLogLevel; const Message: string); virtual; abstract;
    function GetLevelName(Level: TLogLevel): string;
    function ShouldLog(Level: TLogLevel): Boolean;
    property Prefix: string read FPrefix;
    property MinLevel: TLogLevel read FMinLevel write FMinLevel;
  end;

  TConsoleLogger = class(TLogger)
  public
    procedure Log(Level: TLogLevel; const Message: string); override;
  end;

  TFileLogger = class(TLogger)
  private
    FFilePath: string;
    FFileHandle: TextFile;
    FIsOpen: Boolean;
  public
    constructor Create(const APrefix, AFilePath: string; AMinLevel: TLogLevel = llInfo);
    destructor Destroy; override;
    procedure Log(Level: TLogLevel; const Message: string); override;
  end;

  TMemoryLogger = class(TLogger)
  private
    FLogs: array of string;
    FCount: Integer;
  public
    constructor Create(const APrefix: string; ACapacity: Integer = 100);
    procedure Log(Level: TLogLevel; const Message: string); override;
    procedure DumpLogs;
    function GetLogCount: Integer;
  end;

  TCompositeLogger = class(TLogger)
  private
    FLoggers: array of TLogger;
    FCount: Integer;
    FOwnLoggers: Boolean;
  public
    constructor Create(AOwnLoggers: Boolean = False);
    destructor Destroy; override;
    procedure AddLogger(ALogger: TLogger);
    procedure Log(Level: TLogLevel; const Message: string); override;
  end;

  // Factory
  TLoggerFactory = class
  public
    class function CreateLogger(OutputType: TOutputType;
                               const Prefix: string;
                               const Params: string = ''): TLogger;
    class function CreateComposite: TCompositeLogger;
  end;

// ===== TLogger =====
const
  LevelNames: array[TLogLevel] of string = ('DEBUG', 'INFO ', 'WARN ', 'ERROR', 'CRIT!');
  LevelColors: array[TLogLevel] of string = ('', '', '[!]', '[E]', '[!!!]');

constructor TLogger.Create(const APrefix: string; AMinLevel: TLogLevel);
begin
  inherited Create;
  FPrefix := APrefix;
  FMinLevel := AMinLevel;
end;

function TLogger.GetLevelName(Level: TLogLevel): string;
begin
  Result := LevelNames[Level];
end;

function TLogger.ShouldLog(Level: TLogLevel): Boolean;
begin
  Result := Level >= FMinLevel;
end;

// ===== TConsoleLogger =====
procedure TConsoleLogger.Log(Level: TLogLevel; const Message: string);
begin
  if not ShouldLog(Level) then Exit;
  WriteLn(FormatDateTime('[hh:nn:ss]', Now), ' [', GetLevelName(Level), '] ',
          LevelColors[Level], ' [', Prefix, '] ', Message);
end;

// ===== TFileLogger =====
constructor TFileLogger.Create(const APrefix, AFilePath: string; AMinLevel: TLogLevel);
begin
  inherited Create(APrefix, AMinLevel);
  FFilePath := AFilePath;
  FIsOpen := False;
  try
    AssignFile(FFileHandle, FFilePath);
    if FileExists(FFilePath) then
      Append(FFileHandle)
    else
      Rewrite(FFileHandle);
    FIsOpen := True;
  except
    on E: Exception do
      WriteLn('ไม่สามารถเปิดไฟล์ log: ', E.Message);
  end;
end;

destructor TFileLogger.Destroy;
begin
  if FIsOpen then CloseFile(FFileHandle);
  inherited;
end;

procedure TFileLogger.Log(Level: TLogLevel; const Message: string);
begin
  if not ShouldLog(Level) then Exit;
  if FIsOpen then
    WriteLn(FFileHandle,
            FormatDateTime('[dd/mm/yy hh:nn:ss]', Now), ' [', GetLevelName(Level), ']',
            ' [', Prefix, '] ', Message);
end;

// ===== TMemoryLogger =====
constructor TMemoryLogger.Create(const APrefix: string; ACapacity: Integer);
begin
  inherited Create(APrefix);
  FCount := 0;
  SetLength(FLogs, ACapacity);
end;

procedure TMemoryLogger.Log(Level: TLogLevel; const Message: string);
begin
  if not ShouldLog(Level) then Exit;
  if FCount < Length(FLogs) then
  begin
    FLogs[FCount] := Format('[%s][%s] %s',
      [GetLevelName(Level), Prefix, Message]);
    Inc(FCount);
  end;
end;

procedure TMemoryLogger.DumpLogs;
var
  I: Integer;
begin
  WriteLn('=== Memory Logs (', FCount, ' entries) ===');
  for I := 0 to FCount - 1 do
    WriteLn(FLogs[I]);
end;

function TMemoryLogger.GetLogCount: Integer;
begin
  Result := FCount;
end;

// ===== TCompositeLogger =====
constructor TCompositeLogger.Create(AOwnLoggers: Boolean);
begin
  inherited Create('Composite');
  FOwnLoggers := AOwnLoggers;
  FCount := 0;
  SetLength(FLoggers, 8);
end;

destructor TCompositeLogger.Destroy;
var
  I: Integer;
begin
  if FOwnLoggers then
    for I := 0 to FCount - 1 do FLoggers[I].Free;
  inherited;
end;

procedure TCompositeLogger.AddLogger(ALogger: TLogger);
begin
  if FCount >= Length(FLoggers) then
    SetLength(FLoggers, Length(FLoggers) * 2);
  FLoggers[FCount] := ALogger;
  Inc(FCount);
end;

procedure TCompositeLogger.Log(Level: TLogLevel; const Message: string);
var
  I: Integer;
begin
  for I := 0 to FCount - 1 do
    FLoggers[I].Log(Level, Message);  // Polymorphic call!
end;

// ===== TLoggerFactory =====
class function TLoggerFactory.CreateLogger(OutputType: TOutputType;
                                          const Prefix: string;
                                          const Params: string): TLogger;
begin
  case OutputType of
    otConsole: Result := TConsoleLogger.Create(Prefix);
    otFile: Result := TFileLogger.Create(Prefix,
              IfThen(Params = '', 'app.log', Params));
    otDatabase: begin
      // ในตัวอย่างนี้ใช้ Memory logger แทน
      Result := TMemoryLogger.Create(Prefix + '_DB');
    end;
    otNetwork: begin
      Result := TMemoryLogger.Create(Prefix + '_NET');
    end;
    else raise Exception.Create('ไม่รู้จัก output type');
  end;
end;

class function TLoggerFactory.CreateComposite: TCompositeLogger;
begin
  Result := TCompositeLogger.Create(True);
end;

// ===== Main =====
var
  ConsoleLog: TConsoleLogger;
  MemLog: TMemoryLogger;
  Composite: TCompositeLogger;
  Logger: TLogger;
begin
  WriteLn('=== Logger Factory Pattern ===');
  WriteLn;

  // สร้างด้วย factory
  ConsoleLog := TLoggerFactory.CreateLogger(otConsole, 'App') as TConsoleLogger;
  MemLog := TLoggerFactory.CreateLogger(otDatabase, 'DB') as TMemoryLogger;

  // Composite logger
  Composite := TLoggerFactory.CreateComposite;
  Composite.AddLogger(ConsoleLog);
  Composite.AddLogger(MemLog);

  // Log ผ่าน composite - ทุก logger จะได้รับ message
  Composite.Log(llDebug, 'โปรแกรมเริ่มทำงาน');
  Composite.Log(llInfo, 'เชื่อมต่อ database สำเร็จ');
  Composite.Log(llWarning, 'Connection pool เหลือน้อย');
  Composite.Log(llError, 'ไม่สามารถเขียนไฟล์ได้');
  Composite.Log(llCritical, 'System memory ต่ำวิกฤต!');
  WriteLn;

  // แสดง memory logs
  MemLog.DumpLogs;
  WriteLn;
  WriteLn('จำนวน logs ใน memory: ', MemLog.GetLogCount);

  Composite.Free;
  ConsoleLog.Free;
  MemLog.Free;

  ReadLn;
end.
```

---

## 24.9 โปรแกรมตัวอย่าง: Shape Drawing System

```pascal
program ShapeDrawingSystem;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TPoint = record
    X, Y: Double;
    class function Create(AX, AY: Double): TPoint; static;
    function ToString: string;
  end;

  TColor = record
    R, G, B, A: Byte;
    class function RGBA(AR, AG, AB: Byte; AA: Byte = 255): TColor; static;
    class function Red: TColor; static;
    class function Green: TColor; static;
    class function Blue: TColor; static;
    class function Yellow: TColor; static;
    function ToString: string;
  end;

  TDrawContext = class
  private
    FWidth, FHeight: Integer;
    FCanvas: array of array of string;
  public
    constructor Create(AWidth, AHeight: Integer);
    procedure Clear;
    procedure DrawPixel(X, Y: Integer; const Char: string);
    procedure DrawText(X, Y: Integer; const Text: string);
    procedure Render;
    property Width: Integer read FWidth;
    property Height: Integer read FHeight;
  end;

  TShape = class
  private
    FPosition: TPoint;
    FFillColor: TColor;
    FStrokeColor: TColor;
    FLineWidth: Double;
    FVisible: Boolean;
    FZOrder: Integer;
    FName: string;
    class var FShapeCount: Integer;
    FId: Integer;
  public
    constructor Create(const AName: string; X, Y: Double; AColor: TColor);

    // Core virtual methods
    procedure Draw(Context: TDrawContext); virtual; abstract;
    function HitTest(X, Y: Double): Boolean; virtual; abstract;
    function Area: Double; virtual; abstract;
    procedure Scale(Factor: Double); virtual; abstract;
    procedure Rotate(Angle: Double); virtual;
    procedure Translate(DX, DY: Double); virtual;
    function Describe: string; virtual;

    class function GetShapeCount: Integer;

    property Position: TPoint read FPosition write FPosition;
    property FillColor: TColor read FFillColor write FFillColor;
    property StrokeColor: TColor read FStrokeColor write FStrokeColor;
    property LineWidth: Double read FLineWidth write FLineWidth;
    property Visible: Boolean read FVisible write FVisible;
    property ZOrder: Integer read FZOrder write FZOrder;
    property Name: string read FName write FName;
    property Id: Integer read FId;
  end;

  TCircleShape = class(TShape)
  private
    FRadius: Double;
  public
    constructor Create(X, Y, ARadius: Double; AColor: TColor);
    procedure Draw(Context: TDrawContext); override;
    function HitTest(X, Y: Double): Boolean; override;
    function Area: Double; override;
    procedure Scale(Factor: Double); override;
    function Describe: string; override;
    property Radius: Double read FRadius write FRadius;
  end;

  TRectShape = class(TShape)
  private
    FWidth, FHeight: Double;
  public
    constructor Create(X, Y, AWidth, AHeight: Double; AColor: TColor);
    procedure Draw(Context: TDrawContext); override;
    function HitTest(X, Y: Double): Boolean; override;
    function Area: Double; override;
    procedure Scale(Factor: Double); override;
    function Describe: string; override;
  end;

  TLineShape = class(TShape)
  private
    FEndPoint: TPoint;
  public
    constructor Create(X1, Y1, X2, Y2: Double; AColor: TColor);
    procedure Draw(Context: TDrawContext); override;
    function HitTest(X, Y: Double): Boolean; override;
    function Area: Double; override;
    procedure Scale(Factor: Double); override;
    function Describe: string; override;
    property EndPoint: TPoint read FEndPoint write FEndPoint;
  end;

  TTextShape = class(TShape)
  private
    FText: string;
    FFontSize: Integer;
  public
    constructor Create(X, Y: Double; const AText: string; AColor: TColor; AFontSize: Integer = 12);
    procedure Draw(Context: TDrawContext); override;
    function HitTest(X, Y: Double): Boolean; override;
    function Area: Double; override;
    procedure Scale(Factor: Double); override;
    function Describe: string; override;
    property Text: string read FText write FText;
    property FontSize: Integer read FFontSize write FFontSize;
  end;

  TDrawingCanvas = class
  private
    FShapes: array of TShape;
    FCount: Integer;
    FContext: TDrawContext;
    FOwnShapes: Boolean;
    FSelectedShape: TShape;
  public
    constructor Create(Width, Height: Integer; AOwnShapes: Boolean = True);
    destructor Destroy; override;

    procedure AddShape(Shape: TShape);
    procedure RemoveShape(Index: Integer);
    function HitTestAll(X, Y: Double): TShape;
    procedure DrawAll;
    procedure SelectShape(X, Y: Double);
    procedure MoveSelected(DX, DY: Double);
    procedure ScaleSelected(Factor: Double);
    procedure ShowShapeList;

    function FindByName(const AName: string): TShape;

    property ShapeCount: Integer read FCount;
    property Selected: TShape read FSelectedShape;
  end;

// ===== TPoint =====
class function TPoint.Create(AX, AY: Double): TPoint;
begin
  Result.X := AX;
  Result.Y := AY;
end;

function TPoint.ToString: string;
begin
  Result := Format('(%.1f,%.1f)', [X, Y]);
end;

// ===== TColor =====
class function TColor.RGBA(AR, AG, AB: Byte; AA: Byte): TColor;
begin
  Result.R := AR; Result.G := AG; Result.B := AB; Result.A := AA;
end;

class function TColor.Red: TColor;
begin Result := RGBA(255, 0, 0); end;
class function TColor.Green: TColor;
begin Result := RGBA(0, 200, 0); end;
class function TColor.Blue: TColor;
begin Result := RGBA(0, 0, 255); end;
class function TColor.Yellow: TColor;
begin Result := RGBA(255, 220, 0); end;

function TColor.ToString: string;
begin
  Result := Format('rgba(%d,%d,%d,%d)', [R, G, B, A]);
end;

// ===== TDrawContext =====
constructor TDrawContext.Create(AWidth, AHeight: Integer);
var
  I: Integer;
begin
  inherited Create;
  FWidth := AWidth;
  FHeight := AHeight;
  SetLength(FCanvas, AHeight);
  for I := 0 to AHeight - 1 do
    SetLength(FCanvas[I], AWidth);
  Clear;
end;

procedure TDrawContext.Clear;
var
  I, J: Integer;
begin
  for I := 0 to FHeight - 1 do
    for J := 0 to FWidth - 1 do
      FCanvas[I][J] := '.';
end;

procedure TDrawContext.DrawPixel(X, Y: Integer; const Char: string);
begin
  if (X >= 0) and (X < FWidth) and (Y >= 0) and (Y < FHeight) then
    FCanvas[Y][X] := Char;
end;

procedure TDrawContext.DrawText(X, Y: Integer; const Text: string);
var
  I: Integer;
begin
  for I := 1 to Length(Text) do
    DrawPixel(X + I - 1, Y, Text[I]);
end;

procedure TDrawContext.Render;
var
  I, J: Integer;
begin
  for I := 0 to FHeight - 1 do
  begin
    for J := 0 to FWidth - 1 do
      Write(FCanvas[I][J]);
    WriteLn;
  end;
end;

// ===== TShape =====
constructor TShape.Create(const AName: string; X, Y: Double; AColor: TColor);
begin
  inherited Create;
  FName := AName;
  FPosition.X := X;
  FPosition.Y := Y;
  FFillColor := AColor;
  FStrokeColor := AColor;
  FLineWidth := 1;
  FVisible := True;
  Inc(FShapeCount);
  FId := FShapeCount;
  FZOrder := FId;
end;

procedure TShape.Rotate(Angle: Double);
begin
  WriteLn('หมุน ', FName, ' ', Angle:0:1, ' องศา');
end;

procedure TShape.Translate(DX, DY: Double);
begin
  FPosition.X := FPosition.X + DX;
  FPosition.Y := FPosition.Y + DY;
end;

function TShape.Describe: string;
begin
  Result := Format('[%s] #%d ที่ %s พื้นที่=%.2f', [FName, FId, FPosition.ToString, Area]);
end;

class function TShape.GetShapeCount: Integer;
begin
  Result := FShapeCount;
end;

// ===== TCircleShape =====
constructor TCircleShape.Create(X, Y, ARadius: Double; AColor: TColor);
begin
  inherited Create('Circle', X, Y, AColor);
  FRadius := ARadius;
end;

procedure TCircleShape.Draw(Context: TDrawContext);
var
  CX, CY, R, X, Y: Integer;
  Angle: Double;
begin
  CX := Round(FPosition.X);
  CY := Round(FPosition.Y);
  R := Round(FRadius);
  // วาดวงกลมแบบ ASCII
  Angle := 0;
  while Angle < 2 * Pi do
  begin
    X := CX + Round(R * Cos(Angle));
    Y := CY + Round(R * Sin(Angle) / 2);  // หาร 2 เพราะ console ตัวสูงกว่ากว้าง
    Context.DrawPixel(X, Y, 'O');
    Angle := Angle + 0.1;
  end;
  Context.DrawPixel(CX, CY, '+');
end;

function TCircleShape.HitTest(X, Y: Double): Boolean;
var
  DX, DY: Double;
begin
  DX := X - FPosition.X;
  DY := Y - FPosition.Y;
  Result := Sqrt(DX * DX + DY * DY) <= FRadius;
end;

function TCircleShape.Area: Double;
begin
  Result := Pi * FRadius * FRadius;
end;

procedure TCircleShape.Scale(Factor: Double);
begin
  FRadius := FRadius * Factor;
end;

function TCircleShape.Describe: string;
begin
  Result := Format('[Circle] #%d ที่ %s รัศมี=%.1f พื้นที่=%.2f',
    [Id, Position.ToString, FRadius, Area]);
end;

// ===== TRectShape =====
constructor TRectShape.Create(X, Y, AWidth, AHeight: Double; AColor: TColor);
begin
  inherited Create('Rect', X, Y, AColor);
  FWidth := AWidth;
  FHeight := AHeight;
end;

procedure TRectShape.Draw(Context: TDrawContext);
var
  X1, Y1, X2, Y2, X, Y: Integer;
begin
  X1 := Round(FPosition.X);
  Y1 := Round(FPosition.Y);
  X2 := X1 + Round(FWidth) - 1;
  Y2 := Y1 + Round(FHeight / 2) - 1;
  // วาดกรอบ
  for X := X1 to X2 do
  begin
    Context.DrawPixel(X, Y1, '-');
    Context.DrawPixel(X, Y2, '-');
  end;
  for Y := Y1 to Y2 do
  begin
    Context.DrawPixel(X1, Y, '|');
    Context.DrawPixel(X2, Y, '|');
  end;
  Context.DrawPixel(X1, Y1, '+');
  Context.DrawPixel(X2, Y1, '+');
  Context.DrawPixel(X1, Y2, '+');
  Context.DrawPixel(X2, Y2, '+');
end;

function TRectShape.HitTest(X, Y: Double): Boolean;
begin
  Result := (X >= FPosition.X) and (X <= FPosition.X + FWidth) and
            (Y >= FPosition.Y) and (Y <= FPosition.Y + FHeight);
end;

function TRectShape.Area: Double;
begin
  Result := FWidth * FHeight;
end;

procedure TRectShape.Scale(Factor: Double);
begin
  FWidth := FWidth * Factor;
  FHeight := FHeight * Factor;
end;

function TRectShape.Describe: string;
begin
  Result := Format('[Rect] #%d ที่ %s %dx%d',
    [Id, Position.ToString, Round(FWidth), Round(FHeight)]);
end;

// ===== TLineShape =====
constructor TLineShape.Create(X1, Y1, X2, Y2: Double; AColor: TColor);
begin
  inherited Create('Line', X1, Y1, AColor);
  FEndPoint.X := X2;
  FEndPoint.Y := Y2;
end;

procedure TLineShape.Draw(Context: TDrawContext);
var
  X1, Y1, X2, Y2, DX, DY, Steps, I: Integer;
  StepX, StepY, CurX, CurY: Double;
begin
  X1 := Round(FPosition.X);
  Y1 := Round(FPosition.Y);
  X2 := Round(FEndPoint.X);
  Y2 := Round(FEndPoint.Y);
  DX := Abs(X2 - X1);
  DY := Abs(Y2 - Y1);
  Steps := Max(DX, DY);
  if Steps = 0 then Exit;
  StepX := (X2 - X1) / Steps;
  StepY := (Y2 - Y1) / Steps;
  CurX := X1;
  CurY := Y1;
  for I := 0 to Steps do
  begin
    Context.DrawPixel(Round(CurX), Round(CurY div 2), '*');
    CurX := CurX + StepX;
    CurY := CurY + StepY;
  end;
end;

function TLineShape.HitTest(X, Y: Double): Boolean;
begin
  Result := False; // Simplified
end;

function TLineShape.Area: Double;
begin
  Result := 0;
end;

procedure TLineShape.Scale(Factor: Double);
begin
  FEndPoint.X := FPosition.X + (FEndPoint.X - FPosition.X) * Factor;
  FEndPoint.Y := FPosition.Y + (FEndPoint.Y - FPosition.Y) * Factor;
end;

function TLineShape.Describe: string;
begin
  Result := Format('[Line] #%d จาก %s ถึง %s',
    [Id, Position.ToString, FEndPoint.ToString]);
end;

// ===== TTextShape =====
constructor TTextShape.Create(X, Y: Double; const AText: string; AColor: TColor; AFontSize: Integer);
begin
  inherited Create('Text', X, Y, AColor);
  FText := AText;
  FFontSize := AFontSize;
end;

procedure TTextShape.Draw(Context: TDrawContext);
begin
  Context.DrawText(Round(FPosition.X), Round(FPosition.Y / 2), FText);
end;

function TTextShape.HitTest(X, Y: Double): Boolean;
begin
  Result := (X >= FPosition.X) and (X <= FPosition.X + Length(FText) * FFontSize / 2) and
            (Y >= FPosition.Y) and (Y <= FPosition.Y + FFontSize);
end;

function TTextShape.Area: Double;
begin
  Result := Length(FText) * FFontSize * FFontSize / 2;
end;

procedure TTextShape.Scale(Factor: Double);
begin
  FFontSize := Round(FFontSize * Factor);
end;

function TTextShape.Describe: string;
begin
  Result := Format('[Text] #%d "%s" ที่ %s ขนาด %d',
    [Id, FText, Position.ToString, FFontSize]);
end;

// ===== TDrawingCanvas =====
constructor TDrawingCanvas.Create(Width, Height: Integer; AOwnShapes: Boolean);
begin
  inherited Create;
  FOwnShapes := AOwnShapes;
  FCount := 0;
  FSelectedShape := nil;
  SetLength(FShapes, 16);
  FContext := TDrawContext.Create(Width, Height);
end;

destructor TDrawingCanvas.Destroy;
var
  I: Integer;
begin
  if FOwnShapes then
    for I := 0 to FCount - 1 do FShapes[I].Free;
  FContext.Free;
  inherited;
end;

procedure TDrawingCanvas.AddShape(Shape: TShape);
begin
  if FCount >= Length(FShapes) then
    SetLength(FShapes, Length(FShapes) * 2);
  FShapes[FCount] := Shape;
  Inc(FCount);
end;

procedure TDrawingCanvas.RemoveShape(Index: Integer);
var
  I: Integer;
begin
  if (Index < 0) or (Index >= FCount) then Exit;
  if FOwnShapes then FShapes[Index].Free;
  for I := Index to FCount - 2 do FShapes[I] := FShapes[I + 1];
  Dec(FCount);
end;

function TDrawingCanvas.HitTestAll(X, Y: Double): TShape;
var
  I: Integer;
begin
  Result := nil;
  for I := FCount - 1 downto 0 do
    if FShapes[I].HitTest(X, Y) then
    begin
      Result := FShapes[I];
      Exit;
    end;
end;

procedure TDrawingCanvas.DrawAll;
var
  I: Integer;
begin
  FContext.Clear;
  for I := 0 to FCount - 1 do
    if FShapes[I].Visible then
      FShapes[I].Draw(FContext);  // Polymorphic Draw!
  FContext.Render;
end;

procedure TDrawingCanvas.SelectShape(X, Y: Double);
begin
  FSelectedShape := HitTestAll(X, Y);
  if FSelectedShape <> nil then
    WriteLn('เลือก: ', FSelectedShape.Describe)
  else
    WriteLn('ไม่มี shape ที่ตำแหน่ง (', X:0:0, ',', Y:0:0, ')');
end;

procedure TDrawingCanvas.MoveSelected(DX, DY: Double);
begin
  if FSelectedShape <> nil then
  begin
    FSelectedShape.Translate(DX, DY);
    WriteLn('ย้าย "', FSelectedShape.Name, '" ไป ', DX:0:0, ',', DY:0:0);
  end;
end;

procedure TDrawingCanvas.ScaleSelected(Factor: Double);
begin
  if FSelectedShape <> nil then
  begin
    FSelectedShape.Scale(Factor);
    WriteLn('ขยาย "', FSelectedShape.Name, '" x', Factor:0:2);
  end;
end;

procedure TDrawingCanvas.ShowShapeList;
var
  I: Integer;
begin
  WriteLn('=== รายการ Shapes (', FCount, ' รูป) ===');
  for I := 0 to FCount - 1 do
    WriteLn('  ', I, ': ', FShapes[I].Describe);
end;

function TDrawingCanvas.FindByName(const AName: string): TShape;
var
  I: Integer;
begin
  Result := nil;
  for I := 0 to FCount - 1 do
    if SameText(FShapes[I].Name, AName) then
    begin
      Result := FShapes[I];
      Exit;
    end;
end;

// ===== Main =====
var
  Canvas: TDrawingCanvas;
begin
  WriteLn('===== Shape Drawing System =====');
  WriteLn;

  Canvas := TDrawingCanvas.Create(60, 25);

  Canvas.AddShape(TCircleShape.Create(30, 12, 8, TColor.Red));
  Canvas.AddShape(TRectShape.Create(2, 2, 20, 10, TColor.Blue));
  Canvas.AddShape(TLineShape.Create(0, 0, 59, 24, TColor.Green));
  Canvas.AddShape(TTextShape.Create(20, 2, 'Hello!', TColor.Yellow));
  Canvas.AddShape(TCircleShape.Create(50, 20, 4, TColor.Green));

  WriteLn('--- Drawing ---');
  Canvas.DrawAll;
  WriteLn;

  Canvas.ShowShapeList;
  WriteLn;

  // Hit testing
  Canvas.SelectShape(30, 12);
  Canvas.MoveSelected(5, -3);
  Canvas.ScaleSelected(1.5);

  WriteLn;
  Canvas.ShowShapeList;

  Canvas.Free;
  ReadLn;
end.
```

---

## 24.10 แบบฝึกหัด 20 ข้อ

**ข้อ 1:** สร้าง polymorphic `TCalculator` hierarchy ที่ implement operations ต่างๆ (Add, Subtract, Multiply, Divide, Power, Sqrt) ด้วย command objects

**ข้อ 2:** สร้าง `TFilterChain` ที่ chain polymorphic filters สำหรับ process text

**ข้อ 3:** ออกแบบ Event System ด้วย polymorphic `TEvent` hierarchy และ `IEventHandler` interface

**ข้อ 4:** สร้าง polymorphic `TSerializer` ที่ serialize objects เป็น JSON, XML, CSV

**ข้อ 5:** ออกแบบ Game physics ด้วย polymorphic `TCollider` hierarchy (AABB, Circle, Polygon)

**ข้อ 6:** สร้าง `TValidationRule` hierarchy สำหรับ validate data (Required, MinLength, MaxLength, Regex, Range)

**ข้อ 7:** สร้าง `TDataSource` hierarchy ที่ read data จาก CSV, JSON, XML, Database

**ข้อ 8:** สร้าง polymorphic `THashFunction` (MD5, SHA1, SHA256, CRC32) สำหรับ compute hash

**ข้อ 9:** ออกแบบ `TCompressionAlgorithm` hierarchy (RLE, LZ77, Huffman) ด้วย Compress/Decompress

**ข้อ 10:** สร้าง `TSearchAlgorithm` hierarchy (Linear, Binary, Jump, Interpolation Search)

**ข้อ 11:** สร้าง `TImageFilter` hierarchy สำหรับ image processing (Blur, Sharpen, Grayscale, Threshold)

**ข้อ 12:** ออกแบบ polymorphic `TAuthProvider` (Password, JWT, OAuth, APIKey)

**ข้อ 13:** สร้าง `TExporter` hierarchy ที่ export data เป็น formats ต่างๆ

**ข้อ 14:** สร้าง polymorphic `TTransportLayer` (TCP, UDP, WebSocket, gRPC)

**ข้อ 15:** สร้าง `TGraphAlgorithm` hierarchy (DFS, BFS, Dijkstra, A*)

**ข้อ 16:** ออกแบบ `TRenderPipeline` ด้วย polymorphic render passes

**ข้อ 17:** สร้าง `TMessageQueue` hierarchy (InMemory, Redis, RabbitMQ simulation)

**ข้อ 18:** สร้าง polymorphic `TFormatParser` (JSON, YAML, TOML, INI)

**ข้อ 19:** ออกแบบ AI behavior system ด้วย polymorphic `TBehaviorTree` nodes

**ข้อ 20:** สร้าง complete `TWorkflow` system ด้วย polymorphic `TWorkflowStep` และ conditions

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Virtual Dispatch** - กลไก VMT สำหรับ method resolution
2. **Dynamic vs Static Binding** - ความแตกต่างและผลลัพธ์
3. **Override/Virtual** - การ implement polymorphism
4. **Abstract Classes** - กำหนด interface ให้ subclass implement
5. **Interface-based Polymorphism** - polymorphism ข้าม class hierarchy
6. **Collections ของ Polymorphic Objects** - จัดการ objects ต่าง type ร่วมกัน
7. **Strategy Pattern** - เปลี่ยน algorithm ขณะ runtime
8. **Factory Pattern** - สร้าง objects แบบ polymorphic

บทต่อไปจะเรียนเรื่อง Interfaces เชิงลึก
