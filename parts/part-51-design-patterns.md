# Part 51 - Design Patterns ใน Pascal/Lazarus

## บทนำ

Design Patterns คือแนวทางการแก้ปัญหาที่พิสูจน์แล้วว่าใช้ได้ดีในสถานการณ์ที่พบบ่อยในการพัฒนาซอฟต์แวร์ แนวคิดนี้มาจากหนังสือ "Design Patterns: Elements of Reusable Object-Oriented Software" โดย Gang of Four (GoF)

Pattern แบ่งออกเป็น 3 กลุ่มหลัก:
1. **Creational Patterns** - เกี่ยวกับการสร้าง Object
2. **Structural Patterns** - เกี่ยวกับโครงสร้างของ Class และ Object
3. **Behavioral Patterns** - เกี่ยวกับพฤติกรรมและความสัมพันธ์ระหว่าง Object

---

## Creational Patterns

### 1. Singleton Pattern

**แนวคิด**: รับประกันว่า Class จะมีเพียง Instance เดียว และให้ Global Access Point

**เมื่อใช้**: เมื่อต้องการควบคุมทรัพยากรที่ใช้ร่วมกัน เช่น Connection Pool, Logger, Configuration

```pascal
unit SingletonPattern;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs;

type
  { TDatabaseConnection - Singleton สำหรับการเชื่อมต่อฐานข้อมูล }
  TDatabaseConnection = class
  private
    class var FInstance: TDatabaseConnection;
    class var FLock: TCriticalSection;
    FConnectionString: string;
    FIsConnected: Boolean;
    constructor CreatePrivate;
  public
    class function GetInstance: TDatabaseConnection;
    class procedure DestroyInstance;
    procedure Connect(const AConnectionString: string);
    procedure Disconnect;
    function ExecuteQuery(const ASQL: string): string;
    property ConnectionString: string read FConnectionString;
    property IsConnected: Boolean read FIsConnected;
    destructor Destroy; override;
  end;

implementation

constructor TDatabaseConnection.CreatePrivate;
begin
  inherited Create;
  FIsConnected := False;
  FConnectionString := '';
  WriteLn('TDatabaseConnection: สร้าง Instance ใหม่');
end;

class function TDatabaseConnection.GetInstance: TDatabaseConnection;
begin
  if FInstance = nil then
  begin
    FLock.Enter;
    try
      if FInstance = nil then
        FInstance := TDatabaseConnection.CreatePrivate;
    finally
      FLock.Leave;
    end;
  end;
  Result := FInstance;
end;

class procedure TDatabaseConnection.DestroyInstance;
begin
  FLock.Enter;
  try
    FreeAndNil(FInstance);
  finally
    FLock.Leave;
  end;
end;

procedure TDatabaseConnection.Connect(const AConnectionString: string);
begin
  FConnectionString := AConnectionString;
  FIsConnected := True;
  WriteLn('เชื่อมต่อกับ: ', AConnectionString);
end;

procedure TDatabaseConnection.Disconnect;
begin
  FIsConnected := False;
  WriteLn('ตัดการเชื่อมต่อแล้ว');
end;

function TDatabaseConnection.ExecuteQuery(const ASQL: string): string;
begin
  if not FIsConnected then
    raise Exception.Create('ยังไม่ได้เชื่อมต่อกับฐานข้อมูล');
  Result := 'ผลลัพธ์จาก: ' + ASQL;
end;

destructor TDatabaseConnection.Destroy;
begin
  Disconnect;
  inherited Destroy;
end;

initialization
  TDatabaseConnection.FLock := TCriticalSection.Create;

finalization
  TDatabaseConnection.DestroyInstance;
  TDatabaseConnection.FLock.Free;

end.

{ ตัวอย่างการใช้งาน }
program UseSingleton;
uses SingletonPattern;

var
  DB1, DB2: TDatabaseConnection;
begin
  DB1 := TDatabaseConnection.GetInstance;
  DB2 := TDatabaseConnection.GetInstance;
  
  WriteLn('DB1 = DB2? ', DB1 = DB2);  { จะได้ True เสมอ }
  
  DB1.Connect('Server=localhost;Database=mydb');
  WriteLn(DB2.ExecuteQuery('SELECT * FROM users'));
  { DB2 ใช้การเชื่อมต่อเดียวกับ DB1 }
end.
```

---

### 2. Factory Method Pattern

**แนวคิด**: กำหนด Interface สำหรับการสร้าง Object แต่ปล่อยให้ Subclass ตัดสินใจว่าจะสร้าง Class ใด

**เมื่อใช้**: เมื่อต้องการแยกโค้ดการสร้าง Object ออกจากโค้ดที่ใช้งาน

```pascal
unit FactoryPattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Interface สำหรับ Payment }
  IPayment = interface
    ['{A1B2C3D4-E5F6-7890-ABCD-EF1234567890}']
    function ProcessPayment(Amount: Double): Boolean;
    function GetPaymentType: string;
  end;

  { Credit Card Payment }
  TCreditCardPayment = class(TInterfacedObject, IPayment)
  private
    FCardNumber: string;
  public
    constructor Create(const ACardNumber: string);
    function ProcessPayment(Amount: Double): Boolean;
    function GetPaymentType: string;
  end;

  { PayPal Payment }
  TPayPalPayment = class(TInterfacedObject, IPayment)
  private
    FEmail: string;
  public
    constructor Create(const AEmail: string);
    function ProcessPayment(Amount: Double): Boolean;
    function GetPaymentType: string;
  end;

  { Bitcoin Payment }
  TBitcoinPayment = class(TInterfacedObject, IPayment)
  private
    FWalletAddress: string;
  public
    constructor Create(const AWalletAddress: string);
    function ProcessPayment(Amount: Double): Boolean;
    function GetPaymentType: string;
  end;

  { Payment Type Enum }
  TPaymentType = (ptCreditCard, ptPayPal, ptBitcoin);

  { Payment Factory }
  TPaymentFactory = class
  public
    class function CreatePayment(
      PaymentType: TPaymentType;
      const Credential: string): IPayment;
  end;

implementation

{ TCreditCardPayment }
constructor TCreditCardPayment.Create(const ACardNumber: string);
begin
  FCardNumber := ACardNumber;
end;

function TCreditCardPayment.ProcessPayment(Amount: Double): Boolean;
begin
  WriteLn(Format('ชำระด้วยบัตรเครดิต %s จำนวน %.2f บาท', 
    [Copy(FCardNumber, 1, 4) + '****', Amount]));
  Result := True;
end;

function TCreditCardPayment.GetPaymentType: string;
begin
  Result := 'Credit Card';
end;

{ TPayPalPayment }
constructor TPayPalPayment.Create(const AEmail: string);
begin
  FEmail := AEmail;
end;

function TPayPalPayment.ProcessPayment(Amount: Double): Boolean;
begin
  WriteLn(Format('ชำระผ่าน PayPal (%s) จำนวน %.2f บาท', [FEmail, Amount]));
  Result := True;
end;

function TPayPalPayment.GetPaymentType: string;
begin
  Result := 'PayPal';
end;

{ TBitcoinPayment }
constructor TBitcoinPayment.Create(const AWalletAddress: string);
begin
  FWalletAddress := AWalletAddress;
end;

function TBitcoinPayment.ProcessPayment(Amount: Double): Boolean;
begin
  WriteLn(Format('ชำระด้วย Bitcoin ไปยัง %s จำนวน %.2f บาท', 
    [FWalletAddress, Amount]));
  Result := True;
end;

function TBitcoinPayment.GetPaymentType: string;
begin
  Result := 'Bitcoin';
end;

{ TPaymentFactory }
class function TPaymentFactory.CreatePayment(
  PaymentType: TPaymentType;
  const Credential: string): IPayment;
begin
  case PaymentType of
    ptCreditCard: Result := TCreditCardPayment.Create(Credential);
    ptPayPal:     Result := TPayPalPayment.Create(Credential);
    ptBitcoin:    Result := TBitcoinPayment.Create(Credential);
  else
    raise Exception.Create('ประเภทการชำระเงินไม่ถูกต้อง');
  end;
end;

end.

{ การใช้งาน }
program UseFactory;
uses FactoryPattern;

var
  Payment: IPayment;
begin
  Payment := TPaymentFactory.CreatePayment(ptCreditCard, '4111111111111111');
  Payment.ProcessPayment(1500.00);
  
  Payment := TPaymentFactory.CreatePayment(ptPayPal, 'user@example.com');
  Payment.ProcessPayment(2000.00);
  
  Payment := TPaymentFactory.CreatePayment(ptBitcoin, '1A2B3C4D5E6F');
  Payment.ProcessPayment(500.00);
end.
```

---

### 3. Abstract Factory Pattern

**แนวคิด**: สร้าง Family ของ Object ที่เกี่ยวข้องกันโดยไม่ระบุ Concrete Class

**เมื่อใช้**: เมื่อต้องการสร้างชุดของ Object ที่ต้องทำงานร่วมกัน

```pascal
unit AbstractFactoryPattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Abstract Products }
  IButton = interface
    ['{11111111-1111-1111-1111-111111111111}']
    procedure Render;
    function GetStyle: string;
  end;

  ICheckBox = interface
    ['{22222222-2222-2222-2222-222222222222}']
    procedure Render;
    function IsChecked: Boolean;
  end;

  { Windows UI Elements }
  TWindowsButton = class(TInterfacedObject, IButton)
  public
    procedure Render;
    function GetStyle: string;
  end;

  TWindowsCheckBox = class(TInterfacedObject, ICheckBox)
  private
    FChecked: Boolean;
  public
    procedure Render;
    function IsChecked: Boolean;
  end;

  { macOS UI Elements }
  TMacButton = class(TInterfacedObject, IButton)
  public
    procedure Render;
    function GetStyle: string;
  end;

  TMacCheckBox = class(TInterfacedObject, ICheckBox)
  private
    FChecked: Boolean;
  public
    procedure Render;
    function IsChecked: Boolean;
  end;

  { Abstract Factory Interface }
  IUIFactory = interface
    ['{33333333-3333-3333-3333-333333333333}']
    function CreateButton: IButton;
    function CreateCheckBox: ICheckBox;
  end;

  { Concrete Factories }
  TWindowsFactory = class(TInterfacedObject, IUIFactory)
  public
    function CreateButton: IButton;
    function CreateCheckBox: ICheckBox;
  end;

  TMacFactory = class(TInterfacedObject, IUIFactory)
  public
    function CreateButton: IButton;
    function CreateCheckBox: ICheckBox;
  end;

  { Application ที่ใช้ Factory }
  TApplication = class
  private
    FFactory: IUIFactory;
    FButton: IButton;
    FCheckBox: ICheckBox;
  public
    constructor Create(AFactory: IUIFactory);
    procedure BuildUI;
    procedure Paint;
  end;

implementation

{ Windows UI }
procedure TWindowsButton.Render;
begin
  WriteLn('[Windows Button] แสดงปุ่มแบบ Windows Flat Design');
end;

function TWindowsButton.GetStyle: string;
begin
  Result := 'Windows Flat';
end;

procedure TWindowsCheckBox.Render;
begin
  WriteLn('[Windows CheckBox] แสดง Checkbox แบบ Windows');
end;

function TWindowsCheckBox.IsChecked: Boolean;
begin
  Result := FChecked;
end;

{ macOS UI }
procedure TMacButton.Render;
begin
  WriteLn('[Mac Button] แสดงปุ่มแบบ macOS Aqua');
end;

function TMacButton.GetStyle: string;
begin
  Result := 'macOS Aqua';
end;

procedure TMacCheckBox.Render;
begin
  WriteLn('[Mac CheckBox] แสดง Checkbox แบบ macOS');
end;

function TMacCheckBox.IsChecked: Boolean;
begin
  Result := FChecked;
end;

{ Windows Factory }
function TWindowsFactory.CreateButton: IButton;
begin
  Result := TWindowsButton.Create;
end;

function TWindowsFactory.CreateCheckBox: ICheckBox;
begin
  Result := TWindowsCheckBox.Create;
end;

{ Mac Factory }
function TMacFactory.CreateButton: IButton;
begin
  Result := TMacButton.Create;
end;

function TMacFactory.CreateCheckBox: ICheckBox;
begin
  Result := TMacCheckBox.Create;
end;

{ Application }
constructor TApplication.Create(AFactory: IUIFactory);
begin
  FFactory := AFactory;
end;

procedure TApplication.BuildUI;
begin
  FButton := FFactory.CreateButton;
  FCheckBox := FFactory.CreateCheckBox;
end;

procedure TApplication.Paint;
begin
  FButton.Render;
  FCheckBox.Render;
end;

end.
```

---

### 4. Builder Pattern

**แนวคิด**: แยกกระบวนการสร้าง Object ที่ซับซ้อนออกจากการแสดงผล

**เมื่อใช้**: เมื่อต้องการสร้าง Object ที่มีขั้นตอนการสร้างหลายขั้นตอน

```pascal
unit BuilderPattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Product }
  TComputerSpec = record
    CPU: string;
    RAM: string;
    Storage: string;
    GPU: string;
    OS: string;
    Price: Double;
  end;

  TComputer = class
  private
    FSpec: TComputerSpec;
  public
    constructor Create(const ASpec: TComputerSpec);
    procedure ShowSpec;
    property Spec: TComputerSpec read FSpec;
  end;

  { Abstract Builder }
  TComputerBuilder = class abstract
  protected
    FSpec: TComputerSpec;
  public
    procedure Reset; virtual;
    procedure SetCPU(const ACPU: string); virtual; abstract;
    procedure SetRAM(const ARAM: string); virtual; abstract;
    procedure SetStorage(const AStorage: string); virtual; abstract;
    procedure SetGPU(const AGPU: string); virtual;
    procedure SetOS(const AOS: string); virtual;
    function Build: TComputer;
  end;

  { Gaming PC Builder }
  TGamingComputerBuilder = class(TComputerBuilder)
  public
    procedure SetCPU(const ACPU: string); override;
    procedure SetRAM(const ARAM: string); override;
    procedure SetStorage(const AStorage: string); override;
  end;

  { Office PC Builder }
  TOfficeComputerBuilder = class(TComputerBuilder)
  public
    procedure SetCPU(const ACPU: string); override;
    procedure SetRAM(const ARAM: string); override;
    procedure SetStorage(const AStorage: string); override;
  end;

  { Director }
  TComputerDirector = class
  private
    FBuilder: TComputerBuilder;
  public
    constructor Create(ABuilder: TComputerBuilder);
    procedure ChangeBuilder(ABuilder: TComputerBuilder);
    function BuildGamingPC: TComputer;
    function BuildOfficePC: TComputer;
    function BuildMinimalPC: TComputer;
  end;

implementation

{ TComputer }
constructor TComputer.Create(const ASpec: TComputerSpec);
begin
  FSpec := ASpec;
end;

procedure TComputer.ShowSpec;
begin
  WriteLn('=== ข้อมูลคอมพิวเตอร์ ===');
  WriteLn('CPU: ', FSpec.CPU);
  WriteLn('RAM: ', FSpec.RAM);
  WriteLn('Storage: ', FSpec.Storage);
  if FSpec.GPU <> '' then
    WriteLn('GPU: ', FSpec.GPU);
  if FSpec.OS <> '' then
    WriteLn('OS: ', FSpec.OS);
  WriteLn(Format('ราคา: %.2f บาท', [FSpec.Price]));
  WriteLn('========================');
end;

{ TComputerBuilder }
procedure TComputerBuilder.Reset;
begin
  FSpec.CPU := '';
  FSpec.RAM := '';
  FSpec.Storage := '';
  FSpec.GPU := '';
  FSpec.OS := '';
  FSpec.Price := 0;
end;

procedure TComputerBuilder.SetGPU(const AGPU: string);
begin
  FSpec.GPU := AGPU;
  FSpec.Price := FSpec.Price + 15000;
end;

procedure TComputerBuilder.SetOS(const AOS: string);
begin
  FSpec.OS := AOS;
  FSpec.Price := FSpec.Price + 4000;
end;

function TComputerBuilder.Build: TComputer;
begin
  Result := TComputer.Create(FSpec);
  Reset;
end;

{ TGamingComputerBuilder }
procedure TGamingComputerBuilder.SetCPU(const ACPU: string);
begin
  FSpec.CPU := ACPU;
  FSpec.Price := FSpec.Price + 12000;
end;

procedure TGamingComputerBuilder.SetRAM(const ARAM: string);
begin
  FSpec.RAM := ARAM;
  FSpec.Price := FSpec.Price + 6000;
end;

procedure TGamingComputerBuilder.SetStorage(const AStorage: string);
begin
  FSpec.Storage := AStorage;
  FSpec.Price := FSpec.Price + 5000;
end;

{ TOfficeComputerBuilder }
procedure TOfficeComputerBuilder.SetCPU(const ACPU: string);
begin
  FSpec.CPU := ACPU;
  FSpec.Price := FSpec.Price + 8000;
end;

procedure TOfficeComputerBuilder.SetRAM(const ARAM: string);
begin
  FSpec.RAM := ARAM;
  FSpec.Price := FSpec.Price + 3000;
end;

procedure TOfficeComputerBuilder.SetStorage(const AStorage: string);
begin
  FSpec.Storage := AStorage;
  FSpec.Price := FSpec.Price + 2000;
end;

{ TComputerDirector }
constructor TComputerDirector.Create(ABuilder: TComputerBuilder);
begin
  FBuilder := ABuilder;
end;

procedure TComputerDirector.ChangeBuilder(ABuilder: TComputerBuilder);
begin
  FBuilder := ABuilder;
end;

function TComputerDirector.BuildGamingPC: TComputer;
begin
  FBuilder.SetCPU('Intel Core i9-13900K');
  FBuilder.SetRAM('DDR5 64GB');
  FBuilder.SetStorage('NVMe SSD 2TB');
  FBuilder.SetGPU('RTX 4090 24GB');
  FBuilder.SetOS('Windows 11 Pro');
  Result := FBuilder.Build;
end;

function TComputerDirector.BuildOfficePC: TComputer;
begin
  FBuilder.SetCPU('Intel Core i5-13400');
  FBuilder.SetRAM('DDR4 16GB');
  FBuilder.SetStorage('SSD 512GB');
  FBuilder.SetOS('Windows 11 Home');
  Result := FBuilder.Build;
end;

function TComputerDirector.BuildMinimalPC: TComputer;
begin
  FBuilder.SetCPU('Intel Core i3-13100');
  FBuilder.SetRAM('DDR4 8GB');
  FBuilder.SetStorage('HDD 1TB');
  Result := FBuilder.Build;
end;

end.
```

---

### 5. Prototype Pattern

**แนวคิด**: สร้าง Object ใหม่โดยการ Copy จาก Object ที่มีอยู่

**เมื่อใช้**: เมื่อการสร้าง Object ใหม่มีต้นทุนสูง หรือต้องการ Clone Object

```pascal
unit PrototypePattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Interface สำหรับ Prototype }
  ICloneable = interface
    ['{AAAAAAAA-BBBB-CCCC-DDDD-EEEEEEEEEEEE}']
    function Clone: TObject;
  end;

  { Shape Base }
  TShape = class abstract(TInterfacedObject, ICloneable)
  protected
    FColor: string;
    FX, FY: Integer;
  public
    constructor Create(const AColor: string; AX, AY: Integer);
    function Clone: TObject; virtual; abstract;
    procedure Draw; virtual; abstract;
    property Color: string read FColor write FColor;
    property X: Integer read FX write FX;
    property Y: Integer read FY write FY;
  end;

  { Circle }
  TCircle = class(TShape)
  private
    FRadius: Integer;
  public
    constructor Create(const AColor: string; AX, AY, ARadius: Integer);
    function Clone: TObject; override;
    procedure Draw; override;
    property Radius: Integer read FRadius write FRadius;
  end;

  { Rectangle }
  TRectangle = class(TShape)
  private
    FWidth, FHeight: Integer;
  public
    constructor Create(const AColor: string; AX, AY, AWidth, AHeight: Integer);
    function Clone: TObject; override;
    procedure Draw; override;
    property Width: Integer read FWidth write FWidth;
    property Height: Integer read FHeight write FHeight;
  end;

  { Shape Registry - เก็บ Prototype }
  TShapeRegistry = class
  private
    FPrototypes: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    procedure RegisterShape(const AName: string; AShape: TShape);
    function GetShape(const AName: string): TShape;
  end;

implementation

constructor TShape.Create(const AColor: string; AX, AY: Integer);
begin
  FColor := AColor;
  FX := AX;
  FY := AY;
end;

{ TCircle }
constructor TCircle.Create(const AColor: string; AX, AY, ARadius: Integer);
begin
  inherited Create(AColor, AX, AY);
  FRadius := ARadius;
end;

function TCircle.Clone: TObject;
begin
  Result := TCircle.Create(FColor, FX, FY, FRadius);
end;

procedure TCircle.Draw;
begin
  WriteLn(Format('วาดวงกลม: สี=%s, ที่ (%d,%d), รัศมี=%d', 
    [FColor, FX, FY, FRadius]));
end;

{ TRectangle }
constructor TRectangle.Create(const AColor: string; AX, AY, AWidth, AHeight: Integer);
begin
  inherited Create(AColor, AX, AY);
  FWidth := AWidth;
  FHeight := AHeight;
end;

function TRectangle.Clone: TObject;
begin
  Result := TRectangle.Create(FColor, FX, FY, FWidth, FHeight);
end;

procedure TRectangle.Draw;
begin
  WriteLn(Format('วาดสี่เหลี่ยม: สี=%s, ที่ (%d,%d), ขนาด=%dx%d',
    [FColor, FX, FY, FWidth, FHeight]));
end;

{ TShapeRegistry }
constructor TShapeRegistry.Create;
begin
  FPrototypes := TStringList.Create;
  FPrototypes.OwnsObjects := True;
end;

destructor TShapeRegistry.Destroy;
begin
  FPrototypes.Free;
  inherited;
end;

procedure TShapeRegistry.RegisterShape(const AName: string; AShape: TShape);
begin
  FPrototypes.AddObject(AName, AShape);
end;

function TShapeRegistry.GetShape(const AName: string): TShape;
var
  Idx: Integer;
  Proto: TShape;
begin
  Idx := FPrototypes.IndexOf(AName);
  if Idx < 0 then
    raise Exception.CreateFmt('ไม่พบ Shape: %s', [AName]);
  Proto := TShape(FPrototypes.Objects[Idx]);
  Result := TShape(Proto.Clone);
end;

end.
```

---

## Structural Patterns

### 6. Adapter Pattern

**แนวคิด**: แปลง Interface ของ Class หนึ่งให้เป็น Interface ที่ Client ต้องการ

**เมื่อใช้**: เมื่อต้องการใช้งาน Class เก่าที่มี Interface ไม่ตรงกับที่ต้องการ

```pascal
unit AdapterPattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Target Interface ที่ Client ต้องการ }
  IModernLogger = interface
    ['{11111111-2222-3333-4444-555555555555}']
    procedure LogInfo(const AMessage: string);
    procedure LogError(const AMessage: string);
    procedure LogDebug(const AMessage: string);
  end;

  { Legacy Logger - ระบบเก่า }
  TLegacyLogger = class
  public
    procedure WriteToFile(const ALevel, AMessage: string);
    procedure WriteError(const AMessage: string);
  end;

  { Adapter - แปลง LegacyLogger เป็น ModernLogger }
  TLoggerAdapter = class(TInterfacedObject, IModernLogger)
  private
    FLegacyLogger: TLegacyLogger;
  public
    constructor Create(ALegacyLogger: TLegacyLogger);
    destructor Destroy; override;
    procedure LogInfo(const AMessage: string);
    procedure LogError(const AMessage: string);
    procedure LogDebug(const AMessage: string);
  end;

  { ระบบใหม่ที่ใช้ IModernLogger }
  TApplication = class
  private
    FLogger: IModernLogger;
  public
    constructor Create(ALogger: IModernLogger);
    procedure Run;
  end;

implementation

procedure TLegacyLogger.WriteToFile(const ALevel, AMessage: string);
begin
  WriteLn(Format('[%s][%s] %s', 
    [FormatDateTime('yyyy-mm-dd hh:nn:ss', Now), ALevel, AMessage]));
end;

procedure TLegacyLogger.WriteError(const AMessage: string);
begin
  WriteLn('!ERROR! ' + AMessage);
end;

{ TLoggerAdapter }
constructor TLoggerAdapter.Create(ALegacyLogger: TLegacyLogger);
begin
  FLegacyLogger := ALegacyLogger;
end;

destructor TLoggerAdapter.Destroy;
begin
  FLegacyLogger.Free;
  inherited;
end;

procedure TLoggerAdapter.LogInfo(const AMessage: string);
begin
  FLegacyLogger.WriteToFile('INFO', AMessage);
end;

procedure TLoggerAdapter.LogError(const AMessage: string);
begin
  FLegacyLogger.WriteError(AMessage);
end;

procedure TLoggerAdapter.LogDebug(const AMessage: string);
begin
  FLegacyLogger.WriteToFile('DEBUG', AMessage);
end;

{ TApplication }
constructor TApplication.Create(ALogger: IModernLogger);
begin
  FLogger := ALogger;
end;

procedure TApplication.Run;
begin
  FLogger.LogInfo('แอปพลิเคชันเริ่มต้น');
  FLogger.LogDebug('โหลดการตั้งค่า...');
  FLogger.LogError('ไม่พบไฟล์ config.ini');
  FLogger.LogInfo('แอปพลิเคชันสิ้นสุด');
end;

end.
```

---

### 7. Decorator Pattern

**แนวคิด**: เพิ่ม Behavior ให้กับ Object แบบ Dynamic โดยไม่ต้องแก้ไข Class เดิม

**เมื่อใช้**: เมื่อต้องการเพิ่มฟังก์ชันให้กับ Object โดยไม่ต้องสร้าง Subclass จำนวนมาก

```pascal
unit DecoratorPattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Interface สำหรับ Coffee }
  ICoffee = interface
    ['{AABBCCDD-1122-3344-5566-778899AABBCC}']
    function GetDescription: string;
    function GetCost: Double;
  end;

  { Concrete Coffee }
  TSimpleCoffee = class(TInterfacedObject, ICoffee)
  public
    function GetDescription: string;
    function GetCost: Double;
  end;

  { Base Decorator }
  TCoffeeDecorator = class abstract(TInterfacedObject, ICoffee)
  protected
    FCoffee: ICoffee;
  public
    constructor Create(ACoffee: ICoffee);
    function GetDescription: string; virtual;
    function GetCost: Double; virtual;
  end;

  { นม }
  TMilkDecorator = class(TCoffeeDecorator)
  public
    function GetDescription: string; override;
    function GetCost: Double; override;
  end;

  { น้ำตาล }
  TSugarDecorator = class(TCoffeeDecorator)
  public
    function GetDescription: string; override;
    function GetCost: Double; override;
  end;

  { วานิลลา }
  TVanillaDecorator = class(TCoffeeDecorator)
  public
    function GetDescription: string; override;
    function GetCost: Double; override;
  end;

  { ช็อคโกแลต }
  TChocolateDecorator = class(TCoffeeDecorator)
  public
    function GetDescription: string; override;
    function GetCost: Double; override;
  end;

implementation

{ TSimpleCoffee }
function TSimpleCoffee.GetDescription: string;
begin
  Result := 'กาแฟดำ';
end;

function TSimpleCoffee.GetCost: Double;
begin
  Result := 25.0;
end;

{ TCoffeeDecorator }
constructor TCoffeeDecorator.Create(ACoffee: ICoffee);
begin
  FCoffee := ACoffee;
end;

function TCoffeeDecorator.GetDescription: string;
begin
  Result := FCoffee.GetDescription;
end;

function TCoffeeDecorator.GetCost: Double;
begin
  Result := FCoffee.GetCost;
end;

{ TMilkDecorator }
function TMilkDecorator.GetDescription: string;
begin
  Result := FCoffee.GetDescription + ', นม';
end;

function TMilkDecorator.GetCost: Double;
begin
  Result := FCoffee.GetCost + 10.0;
end;

{ TSugarDecorator }
function TSugarDecorator.GetDescription: string;
begin
  Result := FCoffee.GetDescription + ', น้ำตาล';
end;

function TSugarDecorator.GetCost: Double;
begin
  Result := FCoffee.GetCost + 5.0;
end;

{ TVanillaDecorator }
function TVanillaDecorator.GetDescription: string;
begin
  Result := FCoffee.GetDescription + ', วานิลลา';
end;

function TVanillaDecorator.GetCost: Double;
begin
  Result := FCoffee.GetCost + 15.0;
end;

{ TChocolateDecorator }
function TChocolateDecorator.GetDescription: string;
begin
  Result := FCoffee.GetDescription + ', ช็อคโกแลต';
end;

function TChocolateDecorator.GetCost: Double;
begin
  Result := FCoffee.GetCost + 20.0;
end;

end.

{ การใช้งาน }
program UseCoffeeDecorator;
uses DecoratorPattern;

var
  MyCoffee: ICoffee;
begin
  { สั่งกาแฟดำธรรมดา }
  MyCoffee := TSimpleCoffee.Create;
  WriteLn(MyCoffee.GetDescription + ' - ' + FormatFloat('0.00', MyCoffee.GetCost) + ' บาท');

  { เพิ่มนม }
  MyCoffee := TMilkDecorator.Create(MyCoffee);
  WriteLn(MyCoffee.GetDescription + ' - ' + FormatFloat('0.00', MyCoffee.GetCost) + ' บาท');

  { เพิ่มน้ำตาล }
  MyCoffee := TSugarDecorator.Create(MyCoffee);
  WriteLn(MyCoffee.GetDescription + ' - ' + FormatFloat('0.00', MyCoffee.GetCost) + ' บาท');

  { เพิ่มวานิลลา }
  MyCoffee := TVanillaDecorator.Create(MyCoffee);
  WriteLn(MyCoffee.GetDescription + ' - ' + FormatFloat('0.00', MyCoffee.GetCost) + ' บาท');
end.
```

---

## Behavioral Patterns

### 8. Observer Pattern

**แนวคิด**: กำหนดความสัมพันธ์แบบ One-to-Many ระหว่าง Object เมื่อ State เปลี่ยน ผู้ติดตามทั้งหมดจะได้รับการแจ้งเตือน

**เมื่อใช้**: Event System, UI Event Handling, ระบบ Notification

```pascal
unit ObserverPattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Event Data }
  TStockEvent = record
    Symbol: string;
    Price: Double;
    Change: Double;
  end;

  { Observer Interface }
  IStockObserver = interface
    ['{OBSERVER-1111-2222-3333-444444444444}']
    procedure OnStockUpdate(const AEvent: TStockEvent);
    function GetName: string;
  end;

  { Subject Interface }
  IStockSubject = interface
    ['{SUBJECT-1111-2222-3333-444444444444}']
    procedure Subscribe(AObserver: IStockObserver);
    procedure Unsubscribe(AObserver: IStockObserver);
    procedure NotifyObservers(const AEvent: TStockEvent);
  end;

  { Stock Market - Subject }
  TStockMarket = class(TInterfacedObject, IStockSubject)
  private
    FObservers: TInterfaceList;
    FPrices: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    procedure Subscribe(AObserver: IStockObserver);
    procedure Unsubscribe(AObserver: IStockObserver);
    procedure NotifyObservers(const AEvent: TStockEvent);
    procedure UpdatePrice(const ASymbol: string; ANewPrice: Double);
  end;

  { Mobile App Observer }
  TMobileApp = class(TInterfacedObject, IStockObserver)
  private
    FName: string;
    FWatchList: TStringList;
  public
    constructor Create(const AName: string);
    destructor Destroy; override;
    procedure OnStockUpdate(const AEvent: TStockEvent);
    function GetName: string;
    procedure AddToWatchList(const ASymbol: string);
  end;

  { Email Alert Observer }
  TEmailAlert = class(TInterfacedObject, IStockObserver)
  private
    FEmail: string;
    FThreshold: Double;
  public
    constructor Create(const AEmail: string; AThreshold: Double);
    procedure OnStockUpdate(const AEvent: TStockEvent);
    function GetName: string;
  end;

implementation

{ TStockMarket }
constructor TStockMarket.Create;
begin
  FObservers := TInterfaceList.Create;
  FPrices := TStringList.Create;
end;

destructor TStockMarket.Destroy;
begin
  FObservers.Free;
  FPrices.Free;
  inherited;
end;

procedure TStockMarket.Subscribe(AObserver: IStockObserver);
begin
  FObservers.Add(AObserver);
  WriteLn(AObserver.GetName + ' สมัครรับข้อมูล');
end;

procedure TStockMarket.Unsubscribe(AObserver: IStockObserver);
begin
  FObservers.Remove(AObserver);
  WriteLn(AObserver.GetName + ' ยกเลิกการรับข้อมูล');
end;

procedure TStockMarket.NotifyObservers(const AEvent: TStockEvent);
var
  I: Integer;
  Observer: IStockObserver;
begin
  for I := 0 to FObservers.Count - 1 do
  begin
    Observer := IStockObserver(FObservers[I]);
    Observer.OnStockUpdate(AEvent);
  end;
end;

procedure TStockMarket.UpdatePrice(const ASymbol: string; ANewPrice: Double);
var
  Event: TStockEvent;
  OldPrice: Double;
  Idx: Integer;
begin
  Idx := FPrices.IndexOf(ASymbol);
  if Idx >= 0 then
    OldPrice := Double(FPrices.Objects[Idx]^)
  else
    OldPrice := ANewPrice;

  Event.Symbol := ASymbol;
  Event.Price := ANewPrice;
  Event.Change := ((ANewPrice - OldPrice) / OldPrice) * 100;

  { อัปเดตราคา }
  if Idx >= 0 then
    FPrices.Objects[Idx] := TObject(PDouble(@ANewPrice)^)
  else
    FPrices.AddObject(ASymbol, nil);

  NotifyObservers(Event);
end;

{ TMobileApp }
constructor TMobileApp.Create(const AName: string);
begin
  FName := AName;
  FWatchList := TStringList.Create;
end;

destructor TMobileApp.Destroy;
begin
  FWatchList.Free;
  inherited;
end;

procedure TMobileApp.OnStockUpdate(const AEvent: TStockEvent);
var
  Direction: string;
begin
  if FWatchList.IndexOf(AEvent.Symbol) >= 0 then
  begin
    if AEvent.Change >= 0 then Direction := '▲' else Direction := '▼';
    WriteLn(Format('[%s] %s: %.2f %s (%.2f%%)',
      [FName, AEvent.Symbol, AEvent.Price, Direction, AEvent.Change]));
  end;
end;

function TMobileApp.GetName: string;
begin
  Result := 'มือถือ: ' + FName;
end;

procedure TMobileApp.AddToWatchList(const ASymbol: string);
begin
  FWatchList.Add(ASymbol);
end;

{ TEmailAlert }
constructor TEmailAlert.Create(const AEmail: string; AThreshold: Double);
begin
  FEmail := AEmail;
  FThreshold := AThreshold;
end;

procedure TEmailAlert.OnStockUpdate(const AEvent: TStockEvent);
begin
  if Abs(AEvent.Change) >= FThreshold then
    WriteLn(Format('[อีเมล -> %s] แจ้งเตือน! %s เปลี่ยนแปลง %.2f%%',
      [FEmail, AEvent.Symbol, AEvent.Change]));
end;

function TEmailAlert.GetName: string;
begin
  Result := 'Email Alert: ' + FEmail;
end;

end.
```

---

### 9. Strategy Pattern

**แนวคิด**: กำหนด Family ของ Algorithm, Encapsulate แต่ละตัว และทำให้สามารถสลับกันได้

**เมื่อใช้**: เมื่อต้องการเปลี่ยน Algorithm ในขณะ Runtime

```pascal
unit StrategyPattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils, Math;

type
  { Strategy Interface }
  ISortStrategy = interface
    ['{SORT-1111-2222-3333-444444444444}']
    procedure Sort(var AArray: array of Integer);
    function GetName: string;
  end;

  { Bubble Sort }
  TBubbleSort = class(TInterfacedObject, ISortStrategy)
  public
    procedure Sort(var AArray: array of Integer);
    function GetName: string;
  end;

  { Quick Sort }
  TQuickSort = class(TInterfacedObject, ISortStrategy)
  private
    procedure QuickSortHelper(var A: array of Integer; Low, High: Integer);
    function Partition(var A: array of Integer; Low, High: Integer): Integer;
  public
    procedure Sort(var AArray: array of Integer);
    function GetName: string;
  end;

  { Merge Sort }
  TMergeSort = class(TInterfacedObject, ISortStrategy)
  private
    procedure MergeSortHelper(var A: array of Integer; Left, Right: Integer);
    procedure Merge(var A: array of Integer; Left, Mid, Right: Integer);
  public
    procedure Sort(var AArray: array of Integer);
    function GetName: string;
  end;

  { Context - ใช้ Strategy }
  TSorter = class
  private
    FStrategy: ISortStrategy;
  public
    constructor Create(AStrategy: ISortStrategy);
    procedure SetStrategy(AStrategy: ISortStrategy);
    procedure SortArray(var AArray: array of Integer);
    procedure PrintArray(const AArray: array of Integer);
  end;

implementation

{ TBubbleSort }
procedure TBubbleSort.Sort(var AArray: array of Integer);
var
  I, J, Temp: Integer;
  Swapped: Boolean;
begin
  for I := High(AArray) downto 1 do
  begin
    Swapped := False;
    for J := 0 to I - 1 do
    begin
      if AArray[J] > AArray[J + 1] then
      begin
        Temp := AArray[J];
        AArray[J] := AArray[J + 1];
        AArray[J + 1] := Temp;
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

{ TQuickSort }
function TQuickSort.Partition(var A: array of Integer; Low, High: Integer): Integer;
var
  Pivot, Temp, I, J: Integer;
begin
  Pivot := A[High];
  I := Low - 1;
  for J := Low to High - 1 do
  begin
    if A[J] <= Pivot then
    begin
      Inc(I);
      Temp := A[I]; A[I] := A[J]; A[J] := Temp;
    end;
  end;
  Temp := A[I + 1]; A[I + 1] := A[High]; A[High] := Temp;
  Result := I + 1;
end;

procedure TQuickSort.QuickSortHelper(var A: array of Integer; Low, High: Integer);
var
  PI: Integer;
begin
  if Low < High then
  begin
    PI := Partition(A, Low, High);
    QuickSortHelper(A, Low, PI - 1);
    QuickSortHelper(A, PI + 1, High);
  end;
end;

procedure TQuickSort.Sort(var AArray: array of Integer);
begin
  if Length(AArray) > 1 then
    QuickSortHelper(AArray, 0, High(AArray));
end;

function TQuickSort.GetName: string;
begin
  Result := 'Quick Sort';
end;

{ TMergeSort }
procedure TMergeSort.Merge(var A: array of Integer; Left, Mid, Right: Integer);
var
  N1, N2, I, J, K: Integer;
  L, R: array of Integer;
begin
  N1 := Mid - Left + 1;
  N2 := Right - Mid;
  SetLength(L, N1);
  SetLength(R, N2);
  for I := 0 to N1 - 1 do L[I] := A[Left + I];
  for J := 0 to N2 - 1 do R[J] := A[Mid + 1 + J];
  I := 0; J := 0; K := Left;
  while (I < N1) and (J < N2) do
  begin
    if L[I] <= R[J] then begin A[K] := L[I]; Inc(I); end
    else begin A[K] := R[J]; Inc(J); end;
    Inc(K);
  end;
  while I < N1 do begin A[K] := L[I]; Inc(I); Inc(K); end;
  while J < N2 do begin A[K] := R[J]; Inc(J); Inc(K); end;
end;

procedure TMergeSort.MergeSortHelper(var A: array of Integer; Left, Right: Integer);
var
  Mid: Integer;
begin
  if Left < Right then
  begin
    Mid := (Left + Right) div 2;
    MergeSortHelper(A, Left, Mid);
    MergeSortHelper(A, Mid + 1, Right);
    Merge(A, Left, Mid, Right);
  end;
end;

procedure TMergeSort.Sort(var AArray: array of Integer);
begin
  if Length(AArray) > 1 then
    MergeSortHelper(AArray, 0, High(AArray));
end;

function TMergeSort.GetName: string;
begin
  Result := 'Merge Sort';
end;

{ TSorter }
constructor TSorter.Create(AStrategy: ISortStrategy);
begin
  FStrategy := AStrategy;
end;

procedure TSorter.SetStrategy(AStrategy: ISortStrategy);
begin
  FStrategy := AStrategy;
end;

procedure TSorter.SortArray(var AArray: array of Integer);
var
  StartTime: TDateTime;
begin
  StartTime := Now;
  WriteLn('กำลังเรียงลำดับด้วย ' + FStrategy.GetName + '...');
  FStrategy.Sort(AArray);
  WriteLn(Format('เสร็จสิ้น ใช้เวลา %.3f วินาที', 
    [(Now - StartTime) * 86400]));
end;

procedure TSorter.PrintArray(const AArray: array of Integer);
var
  I: Integer;
  S: string;
begin
  S := '[';
  for I := 0 to Min(High(AArray), 9) do
  begin
    if I > 0 then S := S + ', ';
    S := S + IntToStr(AArray[I]);
  end;
  if High(AArray) > 9 then S := S + '...';
  S := S + ']';
  WriteLn(S);
end;

end.
```

---

### 10. Command Pattern

**แนวคิด**: Encapsulate คำขอเป็น Object ทำให้สามารถ Undo/Redo ได้

**เมื่อใช้**: เมื่อต้องการรองรับ Undo/Redo, Queue Commands, Transaction

```pascal
unit CommandPattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Command Interface }
  ICommand = interface
    ['{CMD-1111-2222-3333-444444444444}']
    procedure Execute;
    procedure Undo;
    function GetDescription: string;
  end;

  { Text Editor State }
  TTextEditor = class
  private
    FContent: TStringList;
    FCursorPos: Integer;
  public
    constructor Create;
    destructor Destroy; override;
    procedure InsertText(const AText: string; APosition: Integer);
    procedure DeleteText(APosition, ALength: Integer);
    function GetContent: string;
    property CursorPos: Integer read FCursorPos write FCursorPos;
    procedure ShowContent;
  end;

  { Insert Text Command }
  TInsertCommand = class(TInterfacedObject, ICommand)
  private
    FEditor: TTextEditor;
    FText: string;
    FPosition: Integer;
  public
    constructor Create(AEditor: TTextEditor; const AText: string; APosition: Integer);
    procedure Execute;
    procedure Undo;
    function GetDescription: string;
  end;

  { Delete Text Command }
  TDeleteCommand = class(TInterfacedObject, ICommand)
  private
    FEditor: TTextEditor;
    FDeletedText: string;
    FPosition: Integer;
    FLength: Integer;
  public
    constructor Create(AEditor: TTextEditor; APosition, ALength: Integer);
    procedure Execute;
    procedure Undo;
    function GetDescription: string;
  end;

  { Command History (สำหรับ Undo/Redo) }
  TCommandHistory = class
  private
    FUndoStack: TInterfaceList;
    FRedoStack: TInterfaceList;
  public
    constructor Create;
    destructor Destroy; override;
    procedure ExecuteCommand(ACommand: ICommand);
    procedure Undo;
    procedure Redo;
    function CanUndo: Boolean;
    function CanRedo: Boolean;
  end;

implementation

{ TTextEditor }
constructor TTextEditor.Create;
begin
  FContent := TStringList.Create;
  FContent.Add('');
  FCursorPos := 0;
end;

destructor TTextEditor.Destroy;
begin
  FContent.Free;
  inherited;
end;

procedure TTextEditor.InsertText(const AText: string; APosition: Integer);
var
  S: string;
begin
  S := FContent.Text;
  Insert(AText, S, APosition + 1);
  FContent.Text := S;
end;

procedure TTextEditor.DeleteText(APosition, ALength: Integer);
var
  S: string;
begin
  S := FContent.Text;
  Delete(S, APosition + 1, ALength);
  FContent.Text := S;
end;

function TTextEditor.GetContent: string;
begin
  Result := FContent.Text;
end;

procedure TTextEditor.ShowContent;
begin
  WriteLn('--- เนื้อหา Editor ---');
  WriteLn(FContent.Text);
  WriteLn('--------------------');
end;

{ TInsertCommand }
constructor TInsertCommand.Create(AEditor: TTextEditor; const AText: string; APosition: Integer);
begin
  FEditor := AEditor;
  FText := AText;
  FPosition := APosition;
end;

procedure TInsertCommand.Execute;
begin
  FEditor.InsertText(FText, FPosition);
end;

procedure TInsertCommand.Undo;
begin
  FEditor.DeleteText(FPosition, Length(FText));
end;

function TInsertCommand.GetDescription: string;
begin
  Result := Format('แทรกข้อความ "%s" ที่ตำแหน่ง %d', [FText, FPosition]);
end;

{ TDeleteCommand }
constructor TDeleteCommand.Create(AEditor: TTextEditor; APosition, ALength: Integer);
var
  S: string;
begin
  FEditor := AEditor;
  FPosition := APosition;
  FLength := ALength;
  S := AEditor.GetContent;
  FDeletedText := Copy(S, APosition + 1, ALength);
end;

procedure TDeleteCommand.Execute;
begin
  FEditor.DeleteText(FPosition, FLength);
end;

procedure TDeleteCommand.Undo;
begin
  FEditor.InsertText(FDeletedText, FPosition);
end;

function TDeleteCommand.GetDescription: string;
begin
  Result := Format('ลบข้อความ "%s" ที่ตำแหน่ง %d', [FDeletedText, FPosition]);
end;

{ TCommandHistory }
constructor TCommandHistory.Create;
begin
  FUndoStack := TInterfaceList.Create;
  FRedoStack := TInterfaceList.Create;
end;

destructor TCommandHistory.Destroy;
begin
  FUndoStack.Free;
  FRedoStack.Free;
  inherited;
end;

procedure TCommandHistory.ExecuteCommand(ACommand: ICommand);
begin
  ACommand.Execute;
  FUndoStack.Add(ACommand);
  FRedoStack.Clear; { Clear redo stack เมื่อมี command ใหม่ }
  WriteLn('ดำเนินการ: ' + ACommand.GetDescription);
end;

procedure TCommandHistory.Undo;
var
  Cmd: ICommand;
begin
  if not CanUndo then
  begin
    WriteLn('ไม่มีคำสั่งให้ Undo');
    Exit;
  end;
  Cmd := ICommand(FUndoStack[FUndoStack.Count - 1]);
  FUndoStack.Delete(FUndoStack.Count - 1);
  Cmd.Undo;
  FRedoStack.Add(Cmd);
  WriteLn('Undo: ' + Cmd.GetDescription);
end;

procedure TCommandHistory.Redo;
var
  Cmd: ICommand;
begin
  if not CanRedo then
  begin
    WriteLn('ไม่มีคำสั่งให้ Redo');
    Exit;
  end;
  Cmd := ICommand(FRedoStack[FRedoStack.Count - 1]);
  FRedoStack.Delete(FRedoStack.Count - 1);
  Cmd.Execute;
  FUndoStack.Add(Cmd);
  WriteLn('Redo: ' + Cmd.GetDescription);
end;

function TCommandHistory.CanUndo: Boolean;
begin
  Result := FUndoStack.Count > 0;
end;

function TCommandHistory.CanRedo: Boolean;
begin
  Result := FRedoStack.Count > 0;
end;

end.
```

---

### 11. Template Method Pattern

**แนวคิด**: กำหนดโครงสร้างของ Algorithm ใน Base Class และให้ Subclass Override บาง Step

```pascal
unit TemplatePattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Abstract Data Processor }
  TDataProcessor = class abstract
  protected
    FData: TStringList;
    { Hook Methods - Subclass สามารถ Override ได้ }
    procedure BeforeProcess; virtual;
    procedure AfterProcess; virtual;
    { Abstract Methods - Subclass ต้อง Override }
    procedure ReadData; virtual; abstract;
    procedure ProcessData; virtual; abstract;
    procedure WriteOutput; virtual; abstract;
  public
    constructor Create;
    destructor Destroy; override;
    { Template Method - กำหนด Algorithm }
    procedure Run;
  end;

  { CSV Processor }
  TCSVProcessor = class(TDataProcessor)
  private
    FFileName: string;
    FOutputFile: string;
  public
    constructor Create(const AInputFile, AOutputFile: string);
    procedure ReadData; override;
    procedure ProcessData; override;
    procedure WriteOutput; override;
    procedure BeforeProcess; override;
  end;

  { JSON Processor }
  TJSONProcessor = class(TDataProcessor)
  private
    FJsonData: string;
  public
    constructor Create(const AJsonData: string);
    procedure ReadData; override;
    procedure ProcessData; override;
    procedure WriteOutput; override;
  end;

implementation

constructor TDataProcessor.Create;
begin
  FData := TStringList.Create;
end;

destructor TDataProcessor.Destroy;
begin
  FData.Free;
  inherited;
end;

procedure TDataProcessor.BeforeProcess;
begin
  WriteLn('เริ่มต้นประมวลผลข้อมูล...');
end;

procedure TDataProcessor.AfterProcess;
begin
  WriteLn('ประมวลผลข้อมูลเสร็จสิ้น');
end;

{ Template Method }
procedure TDataProcessor.Run;
begin
  BeforeProcess;    { Hook }
  ReadData;         { Abstract }
  ProcessData;      { Abstract }
  WriteOutput;      { Abstract }
  AfterProcess;     { Hook }
end;

{ TCSVProcessor }
constructor TCSVProcessor.Create(const AInputFile, AOutputFile: string);
begin
  inherited Create;
  FFileName := AInputFile;
  FOutputFile := AOutputFile;
end;

procedure TCSVProcessor.BeforeProcess;
begin
  WriteLn('CSV Processor: ตรวจสอบไฟล์อินพุต ' + FFileName);
  inherited BeforeProcess;
end;

procedure TCSVProcessor.ReadData;
begin
  WriteLn('อ่านไฟล์ CSV: ' + FFileName);
  { จำลองการอ่านข้อมูล }
  FData.Add('ชื่อ,นามสกุล,อีเมล');
  FData.Add('สมชาย,ใจดี,somchai@example.com');
  FData.Add('สมหญิง,สวยงาม,somying@example.com');
  WriteLn(Format('อ่านข้อมูลได้ %d แถว', [FData.Count]));
end;

procedure TCSVProcessor.ProcessData;
var
  I: Integer;
  Parts: TStringList;
begin
  WriteLn('ประมวลผล CSV...');
  Parts := TStringList.Create;
  Parts.Delimiter := ',';
  try
    for I := 1 to FData.Count - 1 do  { ข้าม Header }
    begin
      Parts.DelimitedText := FData[I];
      WriteLn(Format('  ประมวลผล: %s %s (%s)', 
        [Parts[0], Parts[1], Parts[2]]));
    end;
  finally
    Parts.Free;
  end;
end;

procedure TCSVProcessor.WriteOutput;
begin
  WriteLn('เขียนผลลัพธ์ไปยัง: ' + FOutputFile);
end;

{ TJSONProcessor }
constructor TJSONProcessor.Create(const AJsonData: string);
begin
  inherited Create;
  FJsonData := AJsonData;
end;

procedure TJSONProcessor.ReadData;
begin
  WriteLn('แปลงข้อมูล JSON...');
  FData.Add(FJsonData);
end;

procedure TJSONProcessor.ProcessData;
begin
  WriteLn('ประมวลผล JSON: ' + FData.Text);
end;

procedure TJSONProcessor.WriteOutput;
begin
  WriteLn('ส่งออกผลลัพธ์ JSON');
end;

end.
```

---

### 12. Facade Pattern

**แนวคิด**: ให้ Interface ที่เรียบง่ายสำหรับระบบที่ซับซ้อน

```pascal
unit FacadePattern;

{$mode objfpc}{$H+}

interface

uses Classes, SysUtils;

type
  { Subsystems ที่ซับซ้อน }
  TAudioSystem = class
  public
    procedure Initialize;
    procedure SetVolume(AVolume: Integer);
    procedure PlaySound(const AFile: string);
    procedure Shutdown;
  end;

  TVideoSystem = class
  public
    procedure Initialize;
    procedure SetResolution(AWidth, AHeight: Integer);
    procedure PlayVideo(const AFile: string);
    procedure Shutdown;
  end;

  TSubtitleSystem = class
  public
    procedure LoadSubtitles(const AFile: string);
    procedure SetLanguage(const ALang: string);
    procedure Enable;
    procedure Disable;
  end;

  TStreamingService = class
  public
    function Connect(const AURL: string): Boolean;
    function GetStreamURL(const AContentID: string): string;
    procedure Disconnect;
  end;

  { Facade - ให้ Interface เรียบง่าย }
  TMediaPlayerFacade = class
  private
    FAudio: TAudioSystem;
    FVideo: TVideoSystem;
    FSubtitles: TSubtitleSystem;
    FStreaming: TStreamingService;
  public
    constructor Create;
    destructor Destroy; override;
    { เมธอดที่เรียบง่าย ซ่อนความซับซ้อนไว้ข้างใน }
    procedure PlayMovie(const AContentID: string; AWithSubtitles: Boolean = True);
    procedure StopMovie;
    procedure SetVolume(APercent: Integer);
  end;

implementation

procedure TAudioSystem.Initialize;
begin WriteLn('  [Audio] เริ่มต้นระบบเสียง'); end;

procedure TAudioSystem.SetVolume(AVolume: Integer);
begin WriteLn(Format('  [Audio] ตั้งระดับเสียง %d%%', [AVolume])); end;

procedure TAudioSystem.PlaySound(const AFile: string);
begin WriteLn('  [Audio] เล่นเสียง: ' + AFile); end;

procedure TAudioSystem.Shutdown;
begin WriteLn('  [Audio] ปิดระบบเสียง'); end;

procedure TVideoSystem.Initialize;
begin WriteLn('  [Video] เริ่มต้นระบบวิดีโอ'); end;

procedure TVideoSystem.SetResolution(AWidth, AHeight: Integer);
begin WriteLn(Format('  [Video] ตั้งความละเอียด %dx%d', [AWidth, AHeight])); end;

procedure TVideoSystem.PlayVideo(const AFile: string);
begin WriteLn('  [Video] เล่นวิดีโอ: ' + AFile); end;

procedure TVideoSystem.Shutdown;
begin WriteLn('  [Video] ปิดระบบวิดีโอ'); end;

procedure TSubtitleSystem.LoadSubtitles(const AFile: string);
begin WriteLn('  [Sub] โหลดคำบรรยาย: ' + AFile); end;

procedure TSubtitleSystem.SetLanguage(const ALang: string);
begin WriteLn('  [Sub] ตั้งภาษา: ' + ALang); end;

procedure TSubtitleSystem.Enable;
begin WriteLn('  [Sub] เปิดคำบรรยาย'); end;

procedure TSubtitleSystem.Disable;
begin WriteLn('  [Sub] ปิดคำบรรยาย'); end;

function TStreamingService.Connect(const AURL: string): Boolean;
begin
  WriteLn('  [Stream] เชื่อมต่อกับ: ' + AURL);
  Result := True;
end;

function TStreamingService.GetStreamURL(const AContentID: string): string;
begin
  Result := 'https://cdn.streaming.com/' + AContentID + '.m3u8';
  WriteLn('  [Stream] URL: ' + Result);
end;

procedure TStreamingService.Disconnect;
begin WriteLn('  [Stream] ตัดการเชื่อมต่อ'); end;

{ TMediaPlayerFacade }
constructor TMediaPlayerFacade.Create;
begin
  FAudio := TAudioSystem.Create;
  FVideo := TVideoSystem.Create;
  FSubtitles := TSubtitleSystem.Create;
  FStreaming := TStreamingService.Create;
end;

destructor TMediaPlayerFacade.Destroy;
begin
  FAudio.Free;
  FVideo.Free;
  FSubtitles.Free;
  FStreaming.Free;
  inherited;
end;

procedure TMediaPlayerFacade.PlayMovie(const AContentID: string; AWithSubtitles: Boolean);
var
  StreamURL: string;
begin
  WriteLn('=== เริ่มเล่นภาพยนตร์ ===');
  { เริ่มต้นระบบต่างๆ }
  FAudio.Initialize;
  FVideo.Initialize;
  FAudio.SetVolume(80);
  FVideo.SetResolution(1920, 1080);
  
  { เชื่อมต่อ Streaming }
  FStreaming.Connect('https://api.streaming.com');
  StreamURL := FStreaming.GetStreamURL(AContentID);
  
  { เล่นเนื้อหา }
  FVideo.PlayVideo(StreamURL);
  FAudio.PlaySound(StreamURL);
  
  { คำบรรยาย }
  if AWithSubtitles then
  begin
    FSubtitles.LoadSubtitles(AContentID + '_th.srt');
    FSubtitles.SetLanguage('Thai');
    FSubtitles.Enable;
  end;
  WriteLn('กำลังเล่น...');
end;

procedure TMediaPlayerFacade.StopMovie;
begin
  WriteLn('=== หยุดเล่นภาพยนตร์ ===');
  FSubtitles.Disable;
  FStreaming.Disconnect;
  FAudio.Shutdown;
  FVideo.Shutdown;
end;

procedure TMediaPlayerFacade.SetVolume(APercent: Integer);
begin
  FAudio.SetVolume(APercent);
end;

end.

{ การใช้งาน - เรียบง่ายมาก! }
program UseMediaFacade;
uses FacadePattern;

var
  Player: TMediaPlayerFacade;
begin
  Player := TMediaPlayerFacade.Create;
  try
    Player.PlayMovie('avengers-2024', True);
    ReadLn;
    Player.SetVolume(50);
    Player.StopMovie;
  finally
    Player.Free;
  end;
end.
```

---

## สรุป Design Patterns

| Pattern | ประเภท | ใช้เมื่อ |
|---------|--------|---------|
| Singleton | Creational | ต้องการ Instance เดียว |
| Factory | Creational | ซ่อนการสร้าง Object |
| Abstract Factory | Creational | สร้างกลุ่ม Object |
| Builder | Creational | สร้าง Object ซับซ้อน |
| Prototype | Creational | Clone Object |
| Adapter | Structural | แปลง Interface |
| Decorator | Structural | เพิ่ม Behavior ไม่ต้อง Inherit |
| Facade | Structural | ซ่อนความซับซ้อน |
| Observer | Behavioral | Event/Notification System |
| Strategy | Behavioral | สลับ Algorithm |
| Command | Behavioral | Undo/Redo |
| Template Method | Behavioral | กำหนดโครงสร้าง Algorithm |

---

## แบบฝึกหัด

### ข้อ 1 - Singleton Logger
สร้าง Singleton Logger ที่รองรับ:
- Log levels: DEBUG, INFO, WARN, ERROR
- บันทึกไปยังไฟล์และ Console พร้อมกัน
- Thread-safe

### ข้อ 2 - Factory สำหรับ Shape
สร้าง Factory ที่สร้าง Shape ต่างๆ (Circle, Triangle, Rectangle) โดยแต่ละ Shape มีเมธอด `Area()` และ `Perimeter()`

### ข้อ 3 - Builder สำหรับ SQL Query
สร้าง SQL Query Builder ที่รองรับ:
- SELECT, WHERE, ORDER BY, LIMIT
- Method Chaining เช่น `builder.Select('*').From('users').Where('age > 18').OrderBy('name').Build`

### ข้อ 4 - Decorator สำหรับ File I/O
สร้าง Decorator สำหรับอ่านไฟล์ที่รองรับ:
- Encryption Decorator
- Compression Decorator
- Logging Decorator

### ข้อ 5 - Observer สำหรับระบบ Stock
ขยาย Observer Pattern ที่แสดงด้านบนให้รองรับ:
- Alert เมื่อราคาแตะ Target Price
- Stop Loss Alert
- Volume Alert

### ข้อ 6 - Strategy สำหรับ Payment
สร้าง Strategy Pattern สำหรับระบบคิดค่าจัดส่ง:
- Standard: 3-5 วัน, ราคาตามน้ำหนัก
- Express: 1-2 วัน, ราคา 2x
- Same Day: วันเดียวกัน, ราคา 5x

### ข้อ 7 - Command สำหรับ Calculator
สร้าง Calculator ที่รองรับ Undo/Redo สำหรับการคำนวณ:
- Add, Subtract, Multiply, Divide
- History ของการคำนวณ

### ข้อ 8 - Template Method สำหรับ Report
สร้าง Report Generator ด้วย Template Method:
- HTMLReport
- PDFReport
- CSVReport
แต่ละตัวมีขั้นตอน: Header, Body, Footer

### ข้อ 9 - Composite Pattern
สร้าง Composite Pattern สำหรับโครงสร้างแฟ้มเอกสาร (File/Folder) ที่สามารถ:
- คำนวณขนาดรวม
- แสดงโครงสร้างแบบ Tree

### ข้อ 10 - Chain of Responsibility
สร้าง Chain of Responsibility สำหรับระบบ Approval:
- Manager: อนุมัติได้ถึง 10,000 บาท
- Director: อนุมัติได้ถึง 50,000 บาท
- CEO: อนุมัติได้ทุกจำนวน
