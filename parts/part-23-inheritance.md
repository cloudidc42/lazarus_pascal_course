# Part 23 - Inheritance (การสืบทอด)

## บทนำ

Inheritance หรือการสืบทอด คือหนึ่งในหลัก OOP ที่สำคัญที่สุด ช่วยให้เราสร้าง class ใหม่โดยต่อยอดจาก class ที่มีอยู่แล้ว ลดการเขียนโค้ดซ้ำและสร้างความสัมพันธ์ระหว่าง class ในลักษณะ "IS-A"

---

## 23.1 Single Inheritance

Free Pascal รองรับเฉพาะ single inheritance คือ class หนึ่งสามารถมี parent ได้เพียง class เดียว

### รูปแบบ Single Inheritance

```pascal
type
  TAnimal = class
    FName: string;
    procedure Breathe;
    procedure Eat;
  end;

  // TDog IS-A TAnimal
  TDog = class(TAnimal)
    FBreed: string;
    procedure Fetch;
    procedure Bark;
  end;

  // TGoldenRetriever IS-A TDog IS-A TAnimal
  TGoldenRetriever = class(TDog)
    procedure GuideBlind;
  end;
```

### ตัวอย่างสมบูรณ์ Single Inheritance

```pascal
program SingleInheritance;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TVehicle = class
  private
    FBrand: string;
    FModel: string;
    FYear: Integer;
    FMileage: Double;
  public
    constructor Create(const ABrand, AModel: string; AYear: Integer);
    procedure StartEngine; virtual;
    procedure StopEngine; virtual;
    procedure Drive(Distance: Double); virtual;
    procedure ShowInfo; virtual;

    property Brand: string read FBrand;
    property Model: string read FModel;
    property Year: Integer read FYear;
    property Mileage: Double read FMileage;
  end;

  TCar = class(TVehicle)   // สืบทอดจาก TVehicle
  private
    FDoors: Integer;
    FAirCondition: Boolean;
  public
    constructor Create(const ABrand, AModel: string; AYear, ADoors: Integer);
    procedure ShowInfo; override;
    procedure ToggleAC;

    property Doors: Integer read FDoors;
    property AirCondition: Boolean read FAirCondition;
  end;

  TElectricCar = class(TCar)  // สืบทอดจาก TCar
  private
    FBatteryLevel: Double;
    FRange: Double;
  public
    constructor Create(const ABrand, AModel: string; AYear, ADoors: Integer; ARange: Double);
    procedure StartEngine; override;
    procedure Drive(Distance: Double); override;
    procedure Charge(Amount: Double);
    procedure ShowInfo; override;

    property BatteryLevel: Double read FBatteryLevel;
    property Range: Double read FRange;
  end;

// ============= TVehicle =============
constructor TVehicle.Create(const ABrand, AModel: string; AYear: Integer);
begin
  inherited Create;
  FBrand := ABrand;
  FModel := AModel;
  FYear := AYear;
  FMileage := 0;
end;

procedure TVehicle.StartEngine;
begin
  WriteLn(FBrand, ' ', FModel, ' สตาร์ทเครื่อง: วรร...');
end;

procedure TVehicle.StopEngine;
begin
  WriteLn(FBrand, ' ', FModel, ' ดับเครื่อง');
end;

procedure TVehicle.Drive(Distance: Double);
begin
  FMileage := FMileage + Distance;
  WriteLn(FBrand, ' ', FModel, ' ขับ ', Distance:0:1, ' กม. | ระยะรวม: ', FMileage:0:1, ' กม.');
end;

procedure TVehicle.ShowInfo;
begin
  WriteLn('=== ข้อมูลยานพาหนะ ===');
  WriteLn('ยี่ห้อ/รุ่น: ', FBrand, ' ', FModel, ' (', FYear, ')');
  WriteLn('ระยะทาง: ', FMileage:0:1, ' กม.');
end;

// ============= TCar =============
constructor TCar.Create(const ABrand, AModel: string; AYear, ADoors: Integer);
begin
  inherited Create(ABrand, AModel, AYear);
  FDoors := ADoors;
  FAirCondition := False;
end;

procedure TCar.ShowInfo;
begin
  inherited ShowInfo;  // เรียก parent method
  WriteLn('จำนวนประตู: ', FDoors);
  WriteLn('แอร์: ', IfThen(FAirCondition, 'เปิด', 'ปิด'));
end;

procedure TCar.ToggleAC;
begin
  FAirCondition := not FAirCondition;
  WriteLn('แอร์: ', IfThen(FAirCondition, 'เปิด', 'ปิด'));
end;

// ============= TElectricCar =============
constructor TElectricCar.Create(const ABrand, AModel: string; AYear, ADoors: Integer; ARange: Double);
begin
  inherited Create(ABrand, AModel, AYear, ADoors);
  FBatteryLevel := 100;
  FRange := ARange;
end;

procedure TElectricCar.StartEngine;
begin
  WriteLn(Brand, ' ', Model, ' พร้อมวิ่ง (ไฟฟ้า - ไม่มีเสียง)');
  WriteLn('แบตเตอรี่: ', FBatteryLevel:0:1, '%');
end;

procedure TElectricCar.Drive(Distance: Double);
var
  BatteryUsed: Double;
begin
  BatteryUsed := (Distance / FRange) * 100;
  if BatteryUsed > FBatteryLevel then
  begin
    WriteLn('แบตเตอรี่ไม่พอ! ชาร์จก่อน');
    Exit;
  end;
  FBatteryLevel := FBatteryLevel - BatteryUsed;
  inherited Drive(Distance);  // เรียก parent Drive
  WriteLn('แบตเตอรี่เหลือ: ', FBatteryLevel:0:1, '%');
end;

procedure TElectricCar.Charge(Amount: Double);
begin
  FBatteryLevel := Min(100, FBatteryLevel + Amount);
  WriteLn('ชาร์จ +', Amount:0:1, '% | แบตเตอรี่: ', FBatteryLevel:0:1, '%');
end;

procedure TElectricCar.ShowInfo;
begin
  inherited ShowInfo;
  WriteLn('ประเภท: รถไฟฟ้า');
  WriteLn('ระยะทางต่อชาร์จ: ', FRange:0:0, ' กม.');
  WriteLn('แบตเตอรี่ปัจจุบัน: ', FBatteryLevel:0:1, '%');
end;

var
  Car: TCar;
  EV: TElectricCar;
begin
  Car := TCar.Create('Toyota', 'Camry', 2020, 4);
  Car.StartEngine;
  Car.Drive(150);
  Car.ToggleAC;
  Car.ShowInfo;
  WriteLn;

  EV := TElectricCar.Create('Tesla', 'Model 3', 2023, 4, 550);
  EV.StartEngine;
  EV.Drive(200);
  EV.Drive(200);
  EV.Drive(200);  // แบตอาจไม่พอ
  EV.Charge(50);
  EV.Drive(100);
  EV.ShowInfo;
  WriteLn;

  // Polymorphism ผ่าน base class reference
  WriteLn('--- Test via TVehicle reference ---');
  var V: TVehicle := EV;
  V.StartEngine;  // เรียก TElectricCar.StartEngine (virtual dispatch)
  V.Drive(50);

  Car.Free;
  EV.Free;

  ReadLn;
end.
```

---

## 23.2 Multi-Level Inheritance

```pascal
program MultiLevelInheritance;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TLivingThing = class
  protected
    FAge: Integer;
  public
    constructor Create;
    procedure Grow;
    procedure Die; virtual;
    function IsAlive: Boolean; virtual;
    property Age: Integer read FAge;
  end;

  TAnimal = class(TLivingThing)  // Level 2
  protected
    FName: string;
    FWeight: Double;
  public
    constructor Create(const AName: string; AWeight: Double);
    procedure Breathe;
    procedure Eat(Food: string); virtual;
    procedure Move; virtual;
    procedure ShowInfo; virtual;
    property Name: string read FName;
  end;

  TMammal = class(TAnimal)  // Level 3
  protected
    FBodyTemp: Double;
  public
    constructor Create(const AName: string; AWeight: Double);
    procedure RegulateTemp; virtual;
    procedure ShowInfo; override;
  end;

  TDog = class(TMammal)  // Level 4
  private
    FBreed: string;
    FIsTrained: Boolean;
  public
    constructor Create(const AName, ABreed: string; AWeight: Double);
    procedure Bark;
    procedure Fetch;
    procedure ShowInfo; override;
    property Breed: string read FBreed;
    property IsTrained: Boolean read FIsTrained write FIsTrained;
  end;

  TGuideDog = class(TDog)  // Level 5
  private
    FOwnerName: string;
    FIsOnDuty: Boolean;
  public
    constructor Create(const AName, ABreed: string; AWeight: Double;
                      const AOwnerName: string);
    procedure Guide;
    procedure ShowInfo; override;
    property OwnerName: string read FOwnerName;
  end;

// ===== TLivingThing =====
constructor TLivingThing.Create;
begin
  inherited Create;
  FAge := 0;
end;

procedure TLivingThing.Grow;
begin
  Inc(FAge);
end;

procedure TLivingThing.Die;
begin
  WriteLn('สิ่งมีชีวิตสิ้นอายุขัย');
end;

function TLivingThing.IsAlive: Boolean;
begin
  Result := True;
end;

// ===== TAnimal =====
constructor TAnimal.Create(const AName: string; AWeight: Double);
begin
  inherited Create;
  FName := AName;
  FWeight := AWeight;
end;

procedure TAnimal.Breathe;
begin
  WriteLn(FName, ' หายใจ');
end;

procedure TAnimal.Eat(Food: string);
begin
  WriteLn(FName, ' กิน ', Food);
end;

procedure TAnimal.Move;
begin
  WriteLn(FName, ' เคลื่อนที่');
end;

procedure TAnimal.ShowInfo;
begin
  WriteLn('=== สัตว์ ===');
  WriteLn('ชื่อ: ', FName, ' | อายุ: ', FAge, ' | น้ำหนัก: ', FWeight:0:1, ' กก.');
end;

// ===== TMammal =====
constructor TMammal.Create(const AName: string; AWeight: Double);
begin
  inherited Create(AName, AWeight);
  FBodyTemp := 37.0;
end;

procedure TMammal.RegulateTemp;
begin
  WriteLn(FName, ' รักษาอุณหภูมิร่างกาย: ', FBodyTemp:0:1, '°C');
end;

procedure TMammal.ShowInfo;
begin
  inherited ShowInfo;
  WriteLn('อุณหภูมิ: ', FBodyTemp:0:1, '°C (สัตว์เลี้ยงลูกด้วยนม)');
end;

// ===== TDog =====
constructor TDog.Create(const AName, ABreed: string; AWeight: Double);
begin
  inherited Create(AName, AWeight);
  FBreed := ABreed;
  FIsTrained := False;
end;

procedure TDog.Bark;
begin
  WriteLn(FName, ' (', FBreed, '): โฮ่ง! โฮ่ง!');
end;

procedure TDog.Fetch;
begin
  if FIsTrained then
    WriteLn(FName, ' วิ่งเอาลูกบอลมา')
  else
    WriteLn(FName, ' ยังไม่ได้ฝึก ไม่เอาลูกบอล');
end;

procedure TDog.ShowInfo;
begin
  inherited ShowInfo;
  WriteLn('สายพันธุ์: ', FBreed);
  WriteLn('ฝึกแล้ว: ', IfThen(FIsTrained, 'ใช่', 'ยัง'));
end;

// ===== TGuideDog =====
constructor TGuideDog.Create(const AName, ABreed: string; AWeight: Double;
                            const AOwnerName: string);
begin
  inherited Create(AName, ABreed, AWeight);
  FOwnerName := AOwnerName;
  FIsTrained := True;  // Guide dog ฝึกแล้วเสมอ
  FIsOnDuty := False;
end;

procedure TGuideDog.Guide;
begin
  if FIsOnDuty then
    WriteLn(FName, ' นำทาง ', FOwnerName, ' อย่างปลอดภัย')
  else
    WriteLn(FName, ' ยังไม่ได้ปฏิบัติหน้าที่');
end;

procedure TGuideDog.ShowInfo;
begin
  inherited ShowInfo;
  WriteLn('เจ้าของ: ', FOwnerName);
  WriteLn('กำลังทำงาน: ', IfThen(FIsOnDuty, 'ใช่', 'ไม่'));
end;

var
  GD: TGuideDog;
  Animal: TAnimal;
begin
  GD := TGuideDog.Create('แมกซ์', 'Labrador', 28.5, 'นาย สมชาย (ตาบอด)');
  GD.FIsOnDuty := True;

  GD.ShowInfo;
  WriteLn;

  // เรียก methods จาก levels ต่างๆ
  GD.Breathe;    // TAnimal
  GD.RegulateTemp;  // TMammal
  GD.Bark;       // TDog
  GD.Guide;      // TGuideDog
  WriteLn;

  // Polymorphism ผ่าน hierarchy
  Animal := GD;
  Animal.ShowInfo;  // เรียก TGuideDog.ShowInfo (virtual dispatch)

  GD.Free;
  ReadLn;
end.
```

---

## 23.3 Virtual Methods

Virtual methods ทำให้ method dispatch ทำงานตาม type จริงของ object ขณะ runtime

### Static vs Dynamic Binding

```pascal
type
  TBase = class
    procedure StaticMethod;          // Static binding - ใช้ type ของ variable
    procedure VirtualMethod; virtual; // Dynamic binding - ใช้ type จริงของ object
  end;

  TChild = class(TBase)
    procedure StaticMethod;          // Hides parent (ไม่แนะนำ)
    procedure VirtualMethod; override; // Overrides parent
  end;

var
  B: TBase;
  C: TChild;
begin
  C := TChild.Create;
  B := C;  // upcast

  B.StaticMethod;   // เรียก TBase.StaticMethod (static binding ตาม type ของ B)
  B.VirtualMethod;  // เรียก TChild.VirtualMethod (dynamic binding ตาม type จริง)

  B.Free; // C ไม่ต้อง Free แยกเพราะเป็น object เดียวกัน
end;
```

### ตัวอย่าง Virtual Methods

```pascal
program VirtualMethods;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TRenderer = class
  public
    procedure BeginRender; virtual;
    procedure RenderBackground; virtual;
    procedure RenderObjects; virtual;
    procedure EndRender; virtual;
    // Template method pattern
    procedure Render; // ไม่ virtual - ลำดับคงที่
  end;

  TConsoleRenderer = class(TRenderer)
  public
    procedure BeginRender; override;
    procedure RenderBackground; override;
    procedure RenderObjects; override;
    procedure EndRender; override;
  end;

  THTMLRenderer = class(TRenderer)
  public
    procedure BeginRender; override;
    procedure RenderBackground; override;
    procedure RenderObjects; override;
    procedure EndRender; override;
  end;

procedure TRenderer.BeginRender;
begin
  WriteLn('[Base] เริ่ม render');
end;

procedure TRenderer.RenderBackground;
begin
  WriteLn('[Base] Render background ขาว');
end;

procedure TRenderer.RenderObjects;
begin
  WriteLn('[Base] Render objects');
end;

procedure TRenderer.EndRender;
begin
  WriteLn('[Base] จบ render');
end;

// Template method - กำหนดขั้นตอน ให้ virtual methods ทำงานจริง
procedure TRenderer.Render;
begin
  BeginRender;
  RenderBackground;
  RenderObjects;
  EndRender;
end;

procedure TConsoleRenderer.BeginRender;
begin
  WriteLn('[Console] === เริ่ม Console Render ===');
end;

procedure TConsoleRenderer.RenderBackground;
begin
  WriteLn('[Console] Background: .........');
end;

procedure TConsoleRenderer.RenderObjects;
begin
  WriteLn('[Console] Objects: [O][X][*]');
end;

procedure TConsoleRenderer.EndRender;
begin
  WriteLn('[Console] === จบ Console Render ===');
end;

procedure THTMLRenderer.BeginRender;
begin
  WriteLn('[HTML] <html><body>');
end;

procedure THTMLRenderer.RenderBackground;
begin
  WriteLn('[HTML] <div style="background:white">');
end;

procedure THTMLRenderer.RenderObjects;
begin
  WriteLn('[HTML] <div class="objects">content</div>');
end;

procedure THTMLRenderer.EndRender;
begin
  WriteLn('[HTML] </div></body></html>');
end;

var
  Renderers: array[0..1] of TRenderer;
  R: TRenderer;
begin
  Renderers[0] := TConsoleRenderer.Create;
  Renderers[1] := THTMLRenderer.Create;

  for R in Renderers do
  begin
    R.Render;  // เรียก template method
    WriteLn;
  end;

  for R in Renderers do R.Free;
  ReadLn;
end.
```

---

## 23.4 Abstract Methods และ Abstract Classes

### Abstract Method
Method ที่ต้อง override ใน subclass ไม่มี implementation ใน parent

```pascal
type
  TShape = class
  public
    // Abstract - ต้อง override
    function Area: Double; virtual; abstract;
    function Perimeter: Double; virtual; abstract;
    procedure Draw; virtual; abstract;

    // Non-abstract - มี default implementation
    procedure ShowStats;
  end;

procedure TShape.ShowStats;
begin
  WriteLn('พื้นที่: ', Area:0:4);      // เรียก abstract method
  WriteLn('เส้นรอบรูป: ', Perimeter:0:4);
end;

// TShape เป็น abstract ไม่สามารถสร้าง instance ตรงๆ ได้
// var S: TShape := TShape.Create;  // ERROR!
```

### Abstract Class
Class ที่มี abstract methods อย่างน้อยหนึ่งตัว ไม่สามารถสร้าง instance ได้

```pascal
program AbstractClasses;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TPaymentMethod = class  // Abstract payment
  private
    FAmount: Double;
    FCurrency: string;
  public
    constructor Create(AAmount: Double; const ACurrency: string = 'THB');

    // Abstract methods - ต้อง implement ใน subclass
    function Process: Boolean; virtual; abstract;
    function Verify: Boolean; virtual; abstract;
    function GetDetails: string; virtual; abstract;

    // Concrete method - ใช้ abstract methods
    procedure Execute;
    procedure PrintReceipt;

    property Amount: Double read FAmount;
    property Currency: string read FCurrency;
  end;

  TCreditCard = class(TPaymentMethod)
  private
    FCardNumber: string;
    FCardHolder: string;
    FCVV: string;
    FExpiry: string;
  public
    constructor Create(AAmount: Double; const ACardNumber, AHolder, ACVV, AExpiry: string);

    function Process: Boolean; override;
    function Verify: Boolean; override;
    function GetDetails: string; override;
  end;

  TBankTransfer = class(TPaymentMethod)
  private
    FFromAccount: string;
    FToAccount: string;
    FBankCode: string;
  public
    constructor Create(AAmount: Double; const AFrom, ATo, ABankCode: string);

    function Process: Boolean; override;
    function Verify: Boolean; override;
    function GetDetails: string; override;
  end;

  TCryptoPay = class(TPaymentMethod)
  private
    FWalletAddress: string;
    FCoinType: string;
    FCoinAmount: Double;
    FExchangeRate: Double;
  public
    constructor Create(AAmount: Double; const AWallet, ACoin: string; ARate: Double);

    function Process: Boolean; override;
    function Verify: Boolean; override;
    function GetDetails: string; override;
  end;

// ===== TPaymentMethod =====
constructor TPaymentMethod.Create(AAmount: Double; const ACurrency: string);
begin
  inherited Create;
  FAmount := AAmount;
  FCurrency := ACurrency;
end;

procedure TPaymentMethod.Execute;
begin
  WriteLn('--- เริ่มชำระเงิน ---');
  WriteLn('จำนวน: ', FAmount:0:2, ' ', FCurrency);

  if Verify then
  begin
    WriteLn('ตรวจสอบสำเร็จ');
    if Process then
      WriteLn('ชำระเงินสำเร็จ!')
    else
      WriteLn('ชำระเงินล้มเหลว!');
  end
  else
    WriteLn('ตรวจสอบล้มเหลว!');
end;

procedure TPaymentMethod.PrintReceipt;
begin
  WriteLn('=== ใบเสร็จ ===');
  WriteLn('วิธีชำระ: ', ClassName);
  WriteLn('จำนวน: ', FAmount:0:2, ' ', FCurrency);
  WriteLn('รายละเอียด: ', GetDetails);
  WriteLn('===============');
end;

// ===== TCreditCard =====
constructor TCreditCard.Create(AAmount: Double; const ACardNumber, AHolder, ACVV, AExpiry: string);
begin
  inherited Create(AAmount);
  FCardNumber := ACardNumber;
  FCardHolder := AHolder;
  FCVV := ACVV;
  FExpiry := AExpiry;
end;

function TCreditCard.Verify: Boolean;
begin
  // ตรวจสอบข้อมูลบัตร (simplified)
  Result := (Length(FCardNumber) = 16) and (Length(FCVV) = 3);
  if not Result then
    WriteLn('ข้อมูลบัตรไม่ถูกต้อง');
end;

function TCreditCard.Process: Boolean;
begin
  WriteLn('กำลังติดต่อธนาคาร...');
  WriteLn('ชาร์จบัตร ', Copy(FCardNumber, 1, 4), '****', Copy(FCardNumber, 13, 4));
  Result := True;  // สมมติว่าสำเร็จ
end;

function TCreditCard.GetDetails: string;
begin
  Result := Format('บัตร %s****%s (ชื่อ: %s)',
    [Copy(FCardNumber, 1, 4), Copy(FCardNumber, 13, 4), FCardHolder]);
end;

// ===== TBankTransfer =====
constructor TBankTransfer.Create(AAmount: Double; const AFrom, ATo, ABankCode: string);
begin
  inherited Create(AAmount);
  FFromAccount := AFrom;
  FToAccount := ATo;
  FBankCode := ABankCode;
end;

function TBankTransfer.Verify: Boolean;
begin
  Result := (FFromAccount <> '') and (FToAccount <> '') and (FBankCode <> '');
end;

function TBankTransfer.Process: Boolean;
begin
  WriteLn('โอนเงินจาก ', FFromAccount, ' ไป ', FToAccount);
  WriteLn('ผ่านธนาคาร: ', FBankCode);
  Result := True;
end;

function TBankTransfer.GetDetails: string;
begin
  Result := Format('โอนจาก %s ไป %s (%s)',
    [FFromAccount, FToAccount, FBankCode]);
end;

// ===== TCryptoPay =====
constructor TCryptoPay.Create(AAmount: Double; const AWallet, ACoin: string; ARate: Double);
begin
  inherited Create(AAmount);
  FWalletAddress := AWallet;
  FCoinType := ACoin;
  FExchangeRate := ARate;
  FCoinAmount := AAmount / ARate;
end;

function TCryptoPay.Verify: Boolean;
begin
  Result := (Length(FWalletAddress) >= 26) and (FExchangeRate > 0);
end;

function TCryptoPay.Process: Boolean;
begin
  WriteLn('ส่ง ', FCoinAmount:0:8, ' ', FCoinType);
  WriteLn('ไปยัง wallet: ', Copy(FWalletAddress, 1, 8), '...');
  Result := True;
end;

function TCryptoPay.GetDetails: string;
begin
  Result := Format('%.8f %s (1%s = %.2f THB)',
    [FCoinAmount, FCoinType, FCoinType, FExchangeRate]);
end;

var
  Payments: array of TPaymentMethod;
  P: TPaymentMethod;
begin
  SetLength(Payments, 3);
  Payments[0] := TCreditCard.Create(1500, '4111111111111111', 'SOMCHAI J', '123', '12/25');
  Payments[1] := TBankTransfer.Create(5000, '1234567890', '0987654321', 'KBANK');
  Payments[2] := TCryptoPay.Create(100, '1A2b3C4d5E6f7G8h9I0jKLMNOP', 'BTC', 1000000);

  for P in Payments do
  begin
    P.Execute;
    P.PrintReceipt;
    WriteLn;
  end;

  for P in Payments do P.Free;
  ReadLn;
end.
```

---

## 23.5 Inherited Keyword

`inherited` ใช้เรียก method หรือ constructor/destructor ของ parent class

### การใช้ inherited

```pascal
type
  TBase = class
  public
    constructor Create;
    destructor Destroy; override;
    procedure DoWork; virtual;
  end;

  TChild = class(TBase)
  public
    constructor Create;
    destructor Destroy; override;
    procedure DoWork; override;
  end;

constructor TBase.Create;
begin
  inherited Create;  // เรียก TObject.Create
  WriteLn('TBase.Create');
end;

destructor TBase.Destroy;
begin
  WriteLn('TBase.Destroy');
  inherited Destroy;  // เรียก TObject.Destroy
end;

procedure TBase.DoWork;
begin
  WriteLn('TBase.DoWork');
end;

constructor TChild.Create;
begin
  inherited Create;  // เรียก TBase.Create
  WriteLn('TChild.Create');
end;

destructor TChild.Destroy;
begin
  WriteLn('TChild.Destroy');
  inherited Destroy;  // เรียก TBase.Destroy
end;

procedure TChild.DoWork;
begin
  inherited DoWork;  // เรียก TBase.DoWork ก่อน
  WriteLn('TChild.DoWork เพิ่มเติม');
end;
```

### Constructor Chaining

```pascal
program ConstructorChaining;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TBaseEntity = class
  private
    FId: Integer;
    FCreatedAt: TDateTime;
    class var FNextId: Integer;
  public
    constructor Create;
    destructor Destroy; override;
    property Id: Integer read FId;
    property CreatedAt: TDateTime read FCreatedAt;
    procedure ShowBase;
  end;

  TNamedEntity = class(TBaseEntity)
  private
    FName: string;
    FDescription: string;
  public
    constructor Create(const AName: string; const ADesc: string = '');
    procedure ShowNamed;
    property Name: string read FName write FName;
    property Description: string read FDescription write FDescription;
  end;

  TVersionedEntity = class(TNamedEntity)
  private
    FVersion: Integer;
    FLastModified: TDateTime;
  public
    constructor Create(const AName: string; AVersion: Integer = 1);
    procedure IncrementVersion;
    procedure ShowVersioned;
    property Version: Integer read FVersion;
    property LastModified: TDateTime read FLastModified;
  end;

  TActiveEntity = class(TVersionedEntity)
  private
    FIsActive: Boolean;
    FActivatedAt: TDateTime;
  public
    constructor Create(const AName: string; AActive: Boolean = True);
    procedure Activate;
    procedure Deactivate;
    procedure ShowAll;
    property IsActive: Boolean read FIsActive;
  end;

constructor TBaseEntity.Create;
begin
  inherited Create;  // TObject.Create
  Inc(FNextId);
  FId := FNextId;
  FCreatedAt := Now;
  WriteLn('  TBaseEntity.Create -> ID=', FId);
end;

destructor TBaseEntity.Destroy;
begin
  WriteLn('  TBaseEntity.Destroy ID=', FId);
  inherited;
end;

procedure TBaseEntity.ShowBase;
begin
  WriteLn('ID: ', FId, ' | สร้างเมื่อ: ', FormatDateTime('dd/mm/yy hh:nn', FCreatedAt));
end;

constructor TNamedEntity.Create(const AName: string; const ADesc: string);
begin
  inherited Create;  // TBaseEntity.Create
  FName := AName;
  FDescription := ADesc;
  WriteLn('  TNamedEntity.Create -> Name=', FName);
end;

procedure TNamedEntity.ShowNamed;
begin
  ShowBase;
  WriteLn('ชื่อ: ', FName);
  if FDescription <> '' then WriteLn('รายละเอียด: ', FDescription);
end;

constructor TVersionedEntity.Create(const AName: string; AVersion: Integer);
begin
  inherited Create(AName);  // TNamedEntity.Create
  FVersion := AVersion;
  FLastModified := Now;
  WriteLn('  TVersionedEntity.Create -> Version=', FVersion);
end;

procedure TVersionedEntity.IncrementVersion;
begin
  Inc(FVersion);
  FLastModified := Now;
end;

procedure TVersionedEntity.ShowVersioned;
begin
  ShowNamed;
  WriteLn('Version: ', FVersion, ' | แก้ไขล่าสุด: ', FormatDateTime('dd/mm/yy hh:nn', FLastModified));
end;

constructor TActiveEntity.Create(const AName: string; AActive: Boolean);
begin
  inherited Create(AName);  // TVersionedEntity.Create
  FIsActive := AActive;
  if AActive then FActivatedAt := Now;
  WriteLn('  TActiveEntity.Create -> Active=', AActive);
end;

procedure TActiveEntity.Activate;
begin
  FIsActive := True;
  FActivatedAt := Now;
  WriteLn(Name, ' ถูกเปิดใช้งาน');
end;

procedure TActiveEntity.Deactivate;
begin
  FIsActive := False;
  WriteLn(Name, ' ถูกปิดใช้งาน');
end;

procedure TActiveEntity.ShowAll;
begin
  ShowVersioned;
  WriteLn('สถานะ: ', IfThen(FIsActive, 'เปิดใช้งาน', 'ปิดใช้งาน'));
  if FIsActive then
    WriteLn('เปิดใช้เมื่อ: ', FormatDateTime('dd/mm/yy hh:nn', FActivatedAt));
end;

var
  E: TActiveEntity;
begin
  WriteLn('=== สร้าง TActiveEntity ===');
  E := TActiveEntity.Create('เอกสาร A');
  WriteLn;

  WriteLn('=== ข้อมูล ===');
  E.ShowAll;
  WriteLn;

  E.IncrementVersion;
  E.Deactivate;
  E.ShowAll;
  WriteLn;

  WriteLn('=== ลบ TActiveEntity ===');
  E.Free;

  ReadLn;
end.
```

---

## 23.6 Destructor Chaining

```pascal
program DestructorChaining;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TResourceHolder = class
  private
    FName: string;
    FResource: PByte;
    FResourceSize: Integer;
  public
    constructor Create(const AName: string; ResourceSize: Integer);
    destructor Destroy; override;
    property Name: string read FName;
  end;

  TComplexObject = class(TResourceHolder)
  private
    FSubObject: TResourceHolder;
    FData: TStringList;  // สมมติว่า TStringList คล้าย array of string
  public
    constructor Create(const AName: string);
    destructor Destroy; override;
  end;

// ============= Custom TStringList เพื่อ demo =============
type
  TStringList = class
  private
    FItems: array of string;
    FCount: Integer;
  public
    constructor Create;
    destructor Destroy; override;
    procedure Add(const S: string);
    property Count: Integer read FCount;
    property Items[I: Integer]: string read GetItem; default;
  private
    function GetItem(I: Integer): string;
  end;

constructor TStringList.Create;
begin
  inherited;
  FCount := 0;
  WriteLn('  TStringList.Create');
end;

destructor TStringList.Destroy;
begin
  WriteLn('  TStringList.Destroy');
  inherited;
end;

procedure TStringList.Add(const S: string);
begin
  if FCount >= Length(FItems) then SetLength(FItems, FCount + 8);
  FItems[FCount] := S;
  Inc(FCount);
end;

function TStringList.GetItem(I: Integer): string;
begin
  Result := FItems[I];
end;

// ============= TResourceHolder =============
constructor TResourceHolder.Create(const AName: string; ResourceSize: Integer);
begin
  inherited Create;
  FName := AName;
  FResourceSize := ResourceSize;
  GetMem(FResource, ResourceSize);
  FillByte(FResource^, ResourceSize, 0);
  WriteLn('TResourceHolder.Create "', FName, '" จัดสรร ', ResourceSize, ' bytes');
end;

destructor TResourceHolder.Destroy;
begin
  WriteLn('TResourceHolder.Destroy "', FName, '" คืน memory');
  if Assigned(FResource) then
  begin
    FreeMem(FResource);
    FResource := nil;
  end;
  inherited Destroy;  // สำคัญ! เรียก parent destructor เสมอ
end;

// ============= TComplexObject =============
constructor TComplexObject.Create(const AName: string);
begin
  inherited Create(AName, 1024);  // เรียก parent constructor
  FSubObject := TResourceHolder.Create(AName + '_Sub', 512);
  FData := TStringList.Create;
  FData.Add('ข้อมูล1');
  FData.Add('ข้อมูล2');
  WriteLn('TComplexObject.Create "', AName, '"');
end;

destructor TComplexObject.Destroy;
begin
  WriteLn('TComplexObject.Destroy "', Name, '"');
  // ลบ sub-objects ก่อน
  FreeAndNil(FData);
  FreeAndNil(FSubObject);
  // เรียก parent destructor สุดท้าย
  inherited Destroy;
end;

var
  Obj: TComplexObject;
begin
  WriteLn('=== Creating ===');
  Obj := TComplexObject.Create('หลัก');
  WriteLn;

  WriteLn('=== Destroying ===');
  Obj.Free;  // จะเรียก destructor chain

  ReadLn;
end.
```

---

## 23.7 Override vs Overload

### Override
Override method ที่ virtual จาก parent

```pascal
type
  TBase = class
    procedure VirtualMethod; virtual;
  end;

  TChild = class(TBase)
    procedure VirtualMethod; override;  // Override ต้อง virtual ใน parent
  end;
```

### Overload (ใน Inheritance)
Method ชื่อเดียวกันแต่ parameter ต่างกัน - ใช้ `overload` keyword

```pascal
type
  TBase = class
    procedure Print(Value: Integer); virtual;
    procedure Print(const Value: string); virtual; overload;
  end;

  TChild = class(TBase)
    procedure Print(Value: Integer); override;  // Override
    procedure Print(const Value: string); override; overload;  // Override อีกตัว
    procedure Print(Value: Double); overload;  // เพิ่มใหม่
  end;
```

---

## 23.8 Liskov Substitution Principle

LSP กล่าวว่า objects ของ subclass ต้องใช้แทน objects ของ superclass ได้โดยไม่ทำให้โปรแกรมผิดพลาด

### ตัวอย่างที่ดี (ถูกต้องตาม LSP)

```pascal
type
  TRectangle = class
  private
    FWidth, FHeight: Double;
  public
    constructor Create(W, H: Double); virtual;
    procedure SetWidth(W: Double); virtual;
    procedure SetHeight(H: Double); virtual;
    function Area: Double; virtual;
    property Width: Double read FWidth write SetWidth;
    property Height: Double read FHeight write SetHeight;
  end;

  // ถ้า Square extends Rectangle จะ violate LSP!
  // เพราะ SetWidth บน Square ต้องเซ็ต Height ด้วย
  // ทำให้พฤติกรรมไม่ consistent

  // วิธีที่ดีกว่า: ให้ทั้งคู่ implement TShape
  TShape = class
    function Area: Double; virtual; abstract;
    function Perimeter: Double; virtual; abstract;
  end;

  TGoodRectangle = class(TShape)
  private
    FWidth, FHeight: Double;
  public
    constructor Create(W, H: Double);
    function Area: Double; override;
    function Perimeter: Double; override;
  end;

  TGoodSquare = class(TShape)
  private
    FSide: Double;
  public
    constructor Create(Side: Double);
    function Area: Double; override;
    function Perimeter: Double; override;
  end;
```

---

## 23.9 Method Resolution

เมื่อมีหลาย levels ของ inheritance Pascal ใช้ virtual dispatch table

```pascal
program MethodResolution;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TA = class
    procedure M1; virtual;
    procedure M2;
    procedure M3; virtual;
  end;

  TB = class(TA)
    procedure M1; override;
    procedure M2;  // Hides TA.M2 (ไม่ได้ override)
    // M3 ไม่ได้ override - ใช้ TA.M3
  end;

  TC = class(TB)
    procedure M1; override;
    procedure M3; override;
  end;

procedure TA.M1; begin WriteLn('TA.M1'); end;
procedure TA.M2; begin WriteLn('TA.M2'); end;
procedure TA.M3; begin WriteLn('TA.M3'); end;

procedure TB.M1; begin WriteLn('TB.M1'); end;
procedure TB.M2; begin WriteLn('TB.M2'); end;

procedure TC.M1; begin WriteLn('TC.M1'); end;
procedure TC.M3; begin WriteLn('TC.M3'); end;

var
  A: TA;
  B: TB;
  C: TC;
begin
  C := TC.Create;
  B := C;  // upcast TB
  A := C;  // upcast TA

  WriteLn('--- เรียกผ่าน A (TA reference) ---');
  A.M1;  // TC.M1 (virtual dispatch)
  A.M2;  // TA.M2 (static dispatch - M2 ไม่ virtual)
  A.M3;  // TC.M3 (virtual dispatch)
  WriteLn;

  WriteLn('--- เรียกผ่าน B (TB reference) ---');
  B.M1;  // TC.M1 (virtual dispatch)
  B.M2;  // TB.M2 (static dispatch)
  B.M3;  // TC.M3 (virtual dispatch)
  WriteLn;

  WriteLn('--- เรียกผ่าน C (TC reference) ---');
  C.M1;  // TC.M1
  // C.M2 - TB.M2 (hidden TA.M2)
  C.M3;  // TC.M3
  WriteLn;

  C.Free;
  ReadLn;
end.
```

---

## 23.10 โปรแกรมตัวอย่าง: Vehicle Hierarchy

```pascal
program VehicleHierarchy;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TFuelType = (ftGasoline, ftDiesel, ftElectric, ftHybrid, ftHydrogen);

  TVehicle = class
  private
    FMake: string;
    FModel: string;
    FYear: Integer;
    FTopSpeed: Double;
    FFuelType: TFuelType;
  public
    constructor Create(const AMake, AModel: string; AYear: Integer;
                      ATopSpeed: Double; AFuel: TFuelType);

    function GetFuelName: string;
    procedure Describe; virtual;
    function MaxRange: Double; virtual; abstract;
    procedure Move(Distance: Double); virtual;

    property Make: string read FMake;
    property Model: string read FModel;
    property Year: Integer read FYear;
    property TopSpeed: Double read FTopSpeed;
    property FuelType: TFuelType read FFuelType;
  end;

  TLandVehicle = class(TVehicle)
  private
    FWheels: Integer;
    FHasAWD: Boolean;
  public
    constructor Create(const AMake, AModel: string; AYear: Integer;
                      ATopSpeed: Double; AFuel: TFuelType;
                      AWheels: Integer; AHasAWD: Boolean = False);
    procedure Describe; override;
    function MaxRange: Double; override;
    property Wheels: Integer read FWheels;
    property HasAWD: Boolean read FHasAWD;
  end;

  TWaterVehicle = class(TVehicle)
  private
    FHullType: string;
    FDisplacement: Double;
  public
    constructor Create(const AMake, AModel: string; AYear: Integer;
                      ATopSpeed: Double; AFuel: TFuelType;
                      const AHullType: string; ADisplacement: Double);
    procedure Describe; override;
    function MaxRange: Double; override;
    procedure Dock(const Location: string);
  end;

  TAirVehicle = class(TVehicle)
  private
    FCruisingAltitude: Double;
    FPassengerCapacity: Integer;
  public
    constructor Create(const AMake, AModel: string; AYear: Integer;
                      ATopSpeed: Double; AFuel: TFuelType;
                      ACruisingAlt: Double; APassengers: Integer);
    procedure Describe; override;
    function MaxRange: Double; override;
    procedure TakeOff;
    procedure Land(const Airport: string);
  end;

  TCar = class(TLandVehicle)
  private
    FDoors: Integer;
    FHorsepower: Integer;
  public
    constructor Create(const AMake, AModel: string; AYear: Integer;
                      ATopSpeed: Double; AFuel: TFuelType; ADoors, AHP: Integer);
    procedure Describe; override;
    function MaxRange: Double; override;
  end;

  TMotorcycle = class(TLandVehicle)
  private
    FHasSidecar: Boolean;
    FEngineCC: Integer;
  public
    constructor Create(const AMake, AModel: string; AYear: Integer;
                      ATopSpeed: Double; AFuel: TFuelType; AEngineCC: Integer);
    procedure Describe; override;
    function MaxRange: Double; override;
  end;

  TElectricCar = class(TCar)
  private
    FBatteryKWh: Double;
    FChargeLevel: Double;  // 0-100%
  public
    constructor Create(const AMake, AModel: string; AYear: Integer;
                      ATopSpeed: Double; ADoors: Integer; ABatteryKWh: Double);
    procedure Describe; override;
    function MaxRange: Double; override;
    procedure Charge(Percent: Double);
    procedure Move(Distance: Double); override;
  end;

const
  FuelNames: array[TFuelType] of string = (
    'น้ำมันเบนซิน', 'น้ำมันดีเซล', 'ไฟฟ้า', 'ไฮบริด', 'ไฮโดรเจน'
  );

// ====== TVehicle ======
constructor TVehicle.Create(const AMake, AModel: string; AYear: Integer;
                           ATopSpeed: Double; AFuel: TFuelType);
begin
  inherited Create;
  FMake := AMake;
  FModel := AModel;
  FYear := AYear;
  FTopSpeed := ATopSpeed;
  FFuelType := AFuel;
end;

function TVehicle.GetFuelName: string;
begin
  Result := FuelNames[FFuelType];
end;

procedure TVehicle.Describe;
begin
  WriteLn('=== ยานพาหนะ ===');
  WriteLn(FYear, ' ', FMake, ' ', FModel);
  WriteLn('ความเร็วสูงสุด: ', FTopSpeed:0:0, ' กม./ชม.');
  WriteLn('เชื้อเพลิง: ', GetFuelName);
  WriteLn('ระยะทางสูงสุด: ', MaxRange:0:0, ' กม.');
end;

procedure TVehicle.Move(Distance: Double);
begin
  WriteLn(FMake, ' ', FModel, ' เดินทาง ', Distance:0:0, ' กม.');
end;

// ====== TLandVehicle ======
constructor TLandVehicle.Create(const AMake, AModel: string; AYear: Integer;
                               ATopSpeed: Double; AFuel: TFuelType;
                               AWheels: Integer; AHasAWD: Boolean);
begin
  inherited Create(AMake, AModel, AYear, ATopSpeed, AFuel);
  FWheels := AWheels;
  FHasAWD := AHasAWD;
end;

procedure TLandVehicle.Describe;
begin
  inherited Describe;
  WriteLn('จำนวนล้อ: ', FWheels);
  WriteLn('ขับเคลื่อน 4 ล้อ: ', IfThen(FHasAWD, 'ใช่', 'ไม่'));
end;

function TLandVehicle.MaxRange: Double;
begin
  Result := 500;  // Default
end;

// ====== TWaterVehicle ======
constructor TWaterVehicle.Create(const AMake, AModel: string; AYear: Integer;
                                ATopSpeed: Double; AFuel: TFuelType;
                                const AHullType: string; ADisplacement: Double);
begin
  inherited Create(AMake, AModel, AYear, ATopSpeed, AFuel);
  FHullType := AHullType;
  FDisplacement := ADisplacement;
end;

procedure TWaterVehicle.Describe;
begin
  inherited Describe;
  WriteLn('ตัวเรือ: ', FHullType);
  WriteLn('การขับน้ำ: ', FDisplacement:0:0, ' ตัน');
end;

function TWaterVehicle.MaxRange: Double;
begin
  Result := 2000;
end;

procedure TWaterVehicle.Dock(const Location: string);
begin
  WriteLn(Make, ' ', Model, ' จอดที่ ', Location);
end;

// ====== TAirVehicle ======
constructor TAirVehicle.Create(const AMake, AModel: string; AYear: Integer;
                              ATopSpeed: Double; AFuel: TFuelType;
                              ACruisingAlt: Double; APassengers: Integer);
begin
  inherited Create(AMake, AModel, AYear, ATopSpeed, AFuel);
  FCruisingAltitude := ACruisingAlt;
  FPassengerCapacity := APassengers;
end;

procedure TAirVehicle.Describe;
begin
  inherited Describe;
  WriteLn('ความสูงบิน: ', FCruisingAltitude:0:0, ' เมตร');
  WriteLn('ความจุผู้โดยสาร: ', FPassengerCapacity, ' คน');
end;

function TAirVehicle.MaxRange: Double;
begin
  Result := 10000;
end;

procedure TAirVehicle.TakeOff;
begin
  WriteLn(Make, ' ', Model, ' กำลังขึ้นบิน');
end;

procedure TAirVehicle.Land(const Airport: string);
begin
  WriteLn(Make, ' ', Model, ' ลงจอดที่ ', Airport);
end;

// ====== TCar ======
constructor TCar.Create(const AMake, AModel: string; AYear: Integer;
                       ATopSpeed: Double; AFuel: TFuelType; ADoors, AHP: Integer);
begin
  inherited Create(AMake, AModel, AYear, ATopSpeed, AFuel, 4);
  FDoors := ADoors;
  FHorsepower := AHP;
end;

procedure TCar.Describe;
begin
  inherited Describe;
  WriteLn('ประตู: ', FDoors);
  WriteLn('แรงม้า: ', FHorsepower, ' HP');
end;

function TCar.MaxRange: Double;
begin
  case FuelType of
    ftGasoline: Result := 600;
    ftDiesel: Result := 800;
    ftHybrid: Result := 900;
    else Result := 500;
  end;
end;

// ====== TMotorcycle ======
constructor TMotorcycle.Create(const AMake, AModel: string; AYear: Integer;
                              ATopSpeed: Double; AFuel: TFuelType; AEngineCC: Integer);
begin
  inherited Create(AMake, AModel, AYear, ATopSpeed, AFuel, 2);
  FHasSidecar := False;
  FEngineCC := AEngineCC;
end;

procedure TMotorcycle.Describe;
begin
  inherited Describe;
  WriteLn('ขนาดเครื่องยนต์: ', FEngineCC, ' CC');
  WriteLn('มีพ่วงข้าง: ', IfThen(FHasSidecar, 'ใช่', 'ไม่'));
end;

function TMotorcycle.MaxRange: Double;
begin
  Result := 400;
end;

// ====== TElectricCar ======
constructor TElectricCar.Create(const AMake, AModel: string; AYear: Integer;
                               ATopSpeed: Double; ADoors: Integer; ABatteryKWh: Double);
begin
  inherited Create(AMake, AModel, AYear, ATopSpeed, ftElectric, ADoors, 300);
  FBatteryKWh := ABatteryKWh;
  FChargeLevel := 100;
end;

procedure TElectricCar.Describe;
begin
  inherited Describe;
  WriteLn('แบตเตอรี่: ', FBatteryKWh:0:1, ' kWh');
  WriteLn('ระดับชาร์จ: ', FChargeLevel:0:1, '%');
end;

function TElectricCar.MaxRange: Double;
begin
  Result := (FBatteryKWh * 6) * (FChargeLevel / 100);
end;

procedure TElectricCar.Charge(Percent: Double);
begin
  FChargeLevel := Min(100, FChargeLevel + Percent);
  WriteLn('ชาร์จแล้ว: ', FChargeLevel:0:1, '% | ระยะทาง: ', MaxRange:0:0, ' กม.');
end;

procedure TElectricCar.Move(Distance: Double);
var
  BatteryUsed: Double;
begin
  if MaxRange < Distance then
  begin
    WriteLn('แบตเตอรี่ไม่พอ!');
    Exit;
  end;
  BatteryUsed := (Distance / (FBatteryKWh * 6)) * 100;
  FChargeLevel := Max(0, FChargeLevel - BatteryUsed);
  inherited Move(Distance);
  WriteLn('แบตเตอรี่เหลือ: ', FChargeLevel:0:1, '%');
end;

// ====== Main ======
var
  Vehicles: array of TVehicle;
  V: TVehicle;
begin
  SetLength(Vehicles, 5);
  Vehicles[0] := TCar.Create('Toyota', 'Camry', 2022, 200, ftGasoline, 4, 180);
  Vehicles[1] := TElectricCar.Create('Tesla', 'Model S', 2023, 250, 4, 100);
  Vehicles[2] := TMotorcycle.Create('Honda', 'CBR600RR', 2021, 250, ftGasoline, 600);
  Vehicles[3] := TWaterVehicle.Create('Princess', 'V55', 2020, 60, ftDiesel, 'Planing', 15);
  Vehicles[4] := TAirVehicle.Create('Airbus', 'A320', 2019, 830, ftGasoline, 12000, 180);

  for V in Vehicles do
  begin
    V.Describe;
    WriteLn;
  end;

  // Specific actions
  (Vehicles[3] as TWaterVehicle).Dock('ท่าเรือกรุงเทพ');
  (Vehicles[4] as TAirVehicle).TakeOff;
  (Vehicles[4] as TAirVehicle).Land('สนามบินสุวรรณภูมิ');

  WriteLn;
  WriteLn('--- Electric Car Charging ---');
  var EV := Vehicles[1] as TElectricCar;
  EV.Move(200);
  EV.Charge(30);
  EV.Move(150);

  for V in Vehicles do V.Free;
  ReadLn;
end.
```

---

## 23.11 โปรแกรมตัวอย่าง: GUI Control Hierarchy

```pascal
program GUIControlHierarchy;

{$mode objfpc}{$H+}

uses SysUtils;

type
  TRect = record
    X, Y, Width, Height: Integer;
    function Right: Integer;
    function Bottom: Integer;
    function Contains(PX, PY: Integer): Boolean;
  end;

  TControl = class
  private
    FBounds: TRect;
    FVisible: Boolean;
    FEnabled: Boolean;
    FParent: TControl;
    FChildren: array of TControl;
    FChildCount: Integer;
    FName: string;
    FTabOrder: Integer;
    class var FControlCount: Integer;
  protected
    FId: Integer;
    procedure DoPaint; virtual;
    procedure DoClick; virtual;
    procedure DoKeyDown(Key: Char); virtual;
  public
    constructor Create(AParent: TControl; X, Y, W, H: Integer);
    destructor Destroy; override;

    procedure Paint;
    procedure Click;
    procedure KeyDown(Key: Char);
    procedure Show;
    procedure Hide;
    procedure Enable;
    procedure Disable;
    procedure AddChild(Child: TControl);

    property Bounds: TRect read FBounds write FBounds;
    property Visible: Boolean read FVisible write FVisible;
    property Enabled: Boolean read FEnabled write FEnabled;
    property Name: string read FName write FName;
    property Id: Integer read FId;
  end;

  TButton = class(TControl)
  private
    FCaption: string;
    FIsPressed: Boolean;
    FOnClick: TNotifyEvent;
  public
    constructor Create(AParent: TControl; X, Y, W, H: Integer;
                      const ACaption: string);
    procedure DoPaint; override;
    procedure DoClick; override;
    property Caption: string read FCaption write FCaption;
    property OnClick: TNotifyEvent read FOnClick write FOnClick;
  end;

  TLabel = class(TControl)
  private
    FCaption: string;
    FAlignment: string;
  public
    constructor Create(AParent: TControl; X, Y, W, H: Integer;
                      const ACaption: string);
    procedure DoPaint; override;
    property Caption: string read FCaption write FCaption;
    property Alignment: string read FAlignment write FAlignment;
  end;

  TTextBox = class(TControl)
  private
    FText: string;
    FMaxLength: Integer;
    FIsPassword: Boolean;
    FCursorPos: Integer;
  public
    constructor Create(AParent: TControl; X, Y, W, H: Integer);
    procedure DoPaint; override;
    procedure DoKeyDown(Key: Char); override;
    procedure Clear;
    property Text: string read FText write FText;
    property MaxLength: Integer read FMaxLength write FMaxLength;
    property IsPassword: Boolean read FIsPassword write FIsPassword;
  end;

  TPanel = class(TControl)
  private
    FBorderStyle: string;
    FBackColor: string;
  public
    constructor Create(AParent: TControl; X, Y, W, H: Integer);
    procedure DoPaint; override;
    property BorderStyle: string read FBorderStyle write FBorderStyle;
    property BackColor: string read FBackColor write FBackColor;
  end;

  TCheckBox = class(TControl)
  private
    FCaption: string;
    FChecked: Boolean;
  public
    constructor Create(AParent: TControl; X, Y, W, H: Integer;
                      const ACaption: string);
    procedure DoPaint; override;
    procedure DoClick; override;
    property Caption: string read FCaption write FCaption;
    property Checked: Boolean read FChecked write FChecked;
  end;

// TNotifyEvent type
type
  TNotifyEvent = procedure(Sender: TObject) of object;

// ===== TRect =====
function TRect.Right: Integer;
begin
  Result := X + Width;
end;

function TRect.Bottom: Integer;
begin
  Result := Y + Height;
end;

function TRect.Contains(PX, PY: Integer): Boolean;
begin
  Result := (PX >= X) and (PX <= Right) and (PY >= Y) and (PY <= Bottom);
end;

// ===== TControl =====
constructor TControl.Create(AParent: TControl; X, Y, W, H: Integer);
begin
  inherited Create;
  Inc(FControlCount);
  FId := FControlCount;
  FBounds.X := X;
  FBounds.Y := Y;
  FBounds.Width := W;
  FBounds.Height := H;
  FVisible := True;
  FEnabled := True;
  FParent := AParent;
  FChildCount := 0;
  FName := ClassName + IntToStr(FId);

  if AParent <> nil then
    AParent.AddChild(Self);
end;

destructor TControl.Destroy;
var
  I: Integer;
begin
  for I := 0 to FChildCount - 1 do
    FChildren[I].Free;
  inherited;
end;

procedure TControl.DoPaint;
begin
  WriteLn('[', FName, '] paint at (', FBounds.X, ',', FBounds.Y, ') ',
          FBounds.Width, 'x', FBounds.Height);
end;

procedure TControl.DoClick;
begin
  WriteLn('[', FName, '] clicked');
end;

procedure TControl.DoKeyDown(Key: Char);
begin
  WriteLn('[', FName, '] key: ', Key);
end;

procedure TControl.Paint;
begin
  if not FVisible then Exit;
  DoPaint;
  var I: Integer;
  for I := 0 to FChildCount - 1 do
    FChildren[I].Paint;
end;

procedure TControl.Click;
begin
  if not FEnabled then Exit;
  DoClick;
end;

procedure TControl.KeyDown(Key: Char);
begin
  if not FEnabled then Exit;
  DoKeyDown(Key);
end;

procedure TControl.Show;
begin
  FVisible := True;
end;

procedure TControl.Hide;
begin
  FVisible := False;
end;

procedure TControl.Enable;
begin
  FEnabled := True;
end;

procedure TControl.Disable;
begin
  FEnabled := False;
end;

procedure TControl.AddChild(Child: TControl);
begin
  if FChildCount >= Length(FChildren) then
    SetLength(FChildren, Length(FChildren) + 8);
  FChildren[FChildCount] := Child;
  Inc(FChildCount);
end;

// ===== TButton =====
constructor TButton.Create(AParent: TControl; X, Y, W, H: Integer;
                           const ACaption: string);
begin
  inherited Create(AParent, X, Y, W, H);
  FCaption := ACaption;
  FIsPressed := False;
end;

procedure TButton.DoPaint;
begin
  if FEnabled then
    WriteLn('[Button] [', FCaption, '] at (', Bounds.X, ',', Bounds.Y, ')')
  else
    WriteLn('[Button] [', FCaption, '] (disabled) at (', Bounds.X, ',', Bounds.Y, ')');
end;

procedure TButton.DoClick;
begin
  WriteLn('[Button] "', FCaption, '" pressed!');
  if Assigned(FOnClick) then
    FOnClick(Self);
end;

// ===== TLabel =====
constructor TLabel.Create(AParent: TControl; X, Y, W, H: Integer;
                         const ACaption: string);
begin
  inherited Create(AParent, X, Y, W, H);
  FCaption := ACaption;
  FAlignment := 'left';
end;

procedure TLabel.DoPaint;
begin
  WriteLn('[Label] "', FCaption, '" (', FAlignment, ') at (', Bounds.X, ',', Bounds.Y, ')');
end;

// ===== TTextBox =====
constructor TTextBox.Create(AParent: TControl; X, Y, W, H: Integer);
begin
  inherited Create(AParent, X, Y, W, H);
  FText := '';
  FMaxLength := 255;
  FIsPassword := False;
  FCursorPos := 0;
end;

procedure TTextBox.DoPaint;
var
  Display: string;
begin
  if FIsPassword then
    Display := StringOfChar('*', Length(FText))
  else
    Display := FText;
  WriteLn('[TextBox] "', Display, '" at (', Bounds.X, ',', Bounds.Y, ')');
end;

procedure TTextBox.DoKeyDown(Key: Char);
begin
  if Key = #8 then  // Backspace
  begin
    if Length(FText) > 0 then
      Delete(FText, Length(FText), 1);
  end
  else if (Key >= #32) and (Length(FText) < FMaxLength) then
    FText := FText + Key;

  WriteLn('[TextBox] Text: "', FText, '"');
end;

procedure TTextBox.Clear;
begin
  FText := '';
  FCursorPos := 0;
end;

// ===== TPanel =====
constructor TPanel.Create(AParent: TControl; X, Y, W, H: Integer);
begin
  inherited Create(AParent, X, Y, W, H);
  FBorderStyle := 'single';
  FBackColor := 'white';
end;

procedure TPanel.DoPaint;
begin
  WriteLn('[Panel] border=', FBorderStyle, ' bg=', FBackColor,
          ' at (', Bounds.X, ',', Bounds.Y, ') ',
          Bounds.Width, 'x', Bounds.Height);
end;

// ===== TCheckBox =====
constructor TCheckBox.Create(AParent: TControl; X, Y, W, H: Integer;
                            const ACaption: string);
begin
  inherited Create(AParent, X, Y, W, H);
  FCaption := ACaption;
  FChecked := False;
end;

procedure TCheckBox.DoPaint;
begin
  WriteLn('[CheckBox] [', IfThen(FChecked, 'X', ' '), '] ', FCaption);
end;

procedure TCheckBox.DoClick;
begin
  FChecked := not FChecked;
  WriteLn('[CheckBox] "', FCaption, '" = ', FChecked);
end;

// ===== Main =====
var
  Form: TPanel;
  Panel1: TPanel;
  Lbl1: TLabel;
  TextBox1: TTextBox;
  Btn1, Btn2: TButton;
  ChkBox: TCheckBox;
begin
  WriteLn('===== GUI Control Demo =====');
  WriteLn;

  // สร้าง form (root panel)
  Form := TPanel.Create(nil, 0, 0, 800, 600);
  Form.Name := 'MainForm';
  Form.BackColor := 'lightgray';

  // ใส่ controls ใน form
  Lbl1 := TLabel.Create(Form, 20, 20, 200, 30, 'ชื่อผู้ใช้:');
  TextBox1 := TTextBox.Create(Form, 20, 50, 200, 30);
  Btn1 := TButton.Create(Form, 20, 100, 100, 35, 'ตกลง');
  Btn2 := TButton.Create(Form, 130, 100, 100, 35, 'ยกเลิก');
  ChkBox := TCheckBox.Create(Form, 20, 150, 200, 25, 'จำรหัสผ่าน');

  // Panel ใน form
  Panel1 := TPanel.Create(Form, 250, 20, 500, 200);
  Panel1.BackColor := 'white';
  Panel1.BorderStyle := 'raised';

  // Controls ใน Panel1
  var InnerLabel := TLabel.Create(Panel1, 10, 10, 200, 30, 'ข้อมูลเพิ่มเติม');
  var InnerText := TTextBox.Create(Panel1, 10, 50, 300, 30);

  WriteLn('--- Paint Tree ---');
  Form.Paint;
  WriteLn;

  WriteLn('--- Interactions ---');
  TextBox1.KeyDown('S');
  TextBox1.KeyDown('m');
  TextBox1.KeyDown('i');
  TextBox1.KeyDown('t');
  TextBox1.KeyDown('h');
  WriteLn;

  Btn1.Click;
  ChkBox.Click;
  ChkBox.Click;
  WriteLn;

  Btn2.Disable;
  Btn2.Click;  // ไม่มีผล เพราะ disabled
  WriteLn;

  Form.Free;  // จะลบ children ทั้งหมดด้วย
  ReadLn;
end.
```

---

## 23.12 แบบฝึกหัด 20 ข้อ

**ข้อ 1:** สร้าง class hierarchy สำหรับ Musical Instruments (TInstrument -> TStringed, TPercussion, TWind -> TGuitar, TViolin, TDrum, TFlute, TTrumpet)

**ข้อ 2:** ออกแบบ class hierarchy สำหรับ File System (TFileSystemEntry -> TFile, TDirectory) พร้อม Name, Size, Path และ List contents

**ข้อ 3:** สร้าง class hierarchy สำหรับ UI Dialogs (TDialog -> TMessageBox, TInputDialog, TFileDialog, TColorDialog) พร้อม ShowModal

**ข้อ 4:** สร้าง class hierarchy สำหรับ Sort Algorithms (TSorter -> TBubbleSort, TQuickSort, TMergeSort) พร้อม Sort method

**ข้อ 5:** ออกแบบ class hierarchy สำหรับ Database operations (TDBCommand -> TSelectCommand, TInsertCommand, TUpdateCommand, TDeleteCommand)

**ข้อ 6:** สร้าง class hierarchy สำหรับ Network Protocols (TProtocol -> THTTP, TFTP, TSMTP, TPOP3)

**ข้อ 7:** สร้าง class hierarchy สำหรับ Graph Nodes (TNode -> TTreeNode, TBinaryTreeNode, TGraphNode) พร้อม traversal

**ข้อ 8:** ออกแบบ class hierarchy สำหรับ Encryption (TEncryption -> TAES, TRSA, TDES) ด้วย Encrypt/Decrypt methods

**ข้อ 9:** สร้าง class hierarchy สำหรับ Report formats (TReport -> THTMLReport, TPDFReport, TCSVReport, TJSONReport)

**ข้อ 10:** สร้าง class hierarchy สำหรับ Medical Records (TRecord -> TPatientRecord, TDoctorNote, TLabResult, TPrescription)

**ข้อ 11:** ออกแบบ class hierarchy สำหรับ Game Characters (TCharacter -> TWarrior, TMage, TArcher, THealer) พร้อม combat system

**ข้อ 12:** สร้าง class hierarchy สำหรับ Notification Systems (TNotification -> TEmailNotif, TSMSNotif, TPushNotif, TWebhookNotif)

**ข้อ 13:** สร้าง class hierarchy สำหรับ Payment Gateways พร้อม retry logic และ logging ใน base class

**ข้อ 14:** ออกแบบ class hierarchy สำหรับ Cache implementations (TCache -> TMemoryCache, TRedisCache, TFileCache) พร้อม TTL

**ข้อ 15:** สร้าง class hierarchy สำหรับ Logger (TLogger -> TFileLogger, TConsoleLogger, TDBLogger, TCompositeLogger)

**ข้อ 16:** สร้าง Vehicle hierarchy ที่สมบูรณ์กว่าในตัวอย่าง เพิ่ม TRocket, TSubmarine, THelicopter

**ข้อ 17:** สร้าง class hierarchy สำหรับ Expression evaluator ด้วย Composite pattern (TExpr -> TNumber, TBinaryOp, TUnaryOp)

**ข้อ 18:** สร้าง class hierarchy สำหรับ Data Converters (TConverter -> TJSONConverter, TXMLConverter, TCSVConverter, TBinaryConverter)

**ข้อ 19:** ออกแบบ class hierarchy สำหรับ UI Themes (TTheme -> TLightTheme, TDarkTheme, THighContrastTheme) พร้อม apply colors

**ข้อ 20:** สร้าง class hierarchy สำหรับ Animal Kingdom ที่ละเอียด (TLivingThing -> TPlant, TAnimal -> TMammal, TBird, TReptile, TFish, TInsect) พร้อม complete behavior

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Single Inheritance** - class สืบทอดจาก parent เดียว
2. **Multi-Level Inheritance** - chain ของ inheritance หลายระดับ
3. **Virtual Methods** - dynamic dispatch ตาม type จริง
4. **Abstract Methods/Classes** - บังคับให้ subclass implement
5. **inherited keyword** - เรียก parent class methods
6. **Constructor/Destructor Chaining** - ลำดับการสร้างและทำลาย
7. **Override vs Overload** - ความแตกต่างและการใช้งาน
8. **Liskov Substitution Principle** - subclass ต้องใช้แทน superclass ได้
9. **Method Resolution** - ลำดับการ dispatch methods

บทต่อไปจะเรียนเรื่อง Polymorphism เชิงลึก
