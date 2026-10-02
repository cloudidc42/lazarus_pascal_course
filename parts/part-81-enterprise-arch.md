# ตอนที่ 81: Enterprise Architecture กับ Pascal/Lazarus

## บทนำ: สถาปัตยกรรมระดับองค์กร

Enterprise Architecture (สถาปัตยกรรมระดับองค์กร) คือการออกแบบระบบซอฟต์แวร์ขนาดใหญ่ที่ต้องรองรับผู้ใช้งานหลายพันคน มีความซับซ้อนสูง และต้องการความน่าเชื่อถือสูง ในบทนี้เราจะเรียนรู้วิธีการออกแบบและสร้างระบบระดับองค์กรด้วย Pascal/Lazarus

## 1. N-Tier Architecture (สถาปัตยกรรมแบบหลายชั้น)

### แนวคิดพื้นฐาน

N-Tier Architecture แบ่งระบบออกเป็นชั้น (Layer) ต่างๆ ที่มีหน้าที่เฉพาะ:

```
┌─────────────────────────────────┐
│       Presentation Layer        │  ← UI, Forms, API Controllers
├─────────────────────────────────┤
│       Application Layer         │  ← Use Cases, Application Services
├─────────────────────────────────┤
│         Domain Layer            │  ← Business Logic, Entities
├─────────────────────────────────┤
│      Infrastructure Layer       │  ← Database, External Services
└─────────────────────────────────┘
```

### โครงสร้างโปรเจ็กต์ Enterprise Pascal

```
EnterpriseApp/
├── src/
│   ├── Domain/
│   │   ├── Entities/
│   │   │   ├── uCustomer.pas
│   │   │   ├── uOrder.pas
│   │   │   └── uProduct.pas
│   │   ├── ValueObjects/
│   │   │   ├── uMoney.pas
│   │   │   └── uAddress.pas
│   │   ├── Interfaces/
│   │   │   ├── IRepository.pas
│   │   │   └── IUnitOfWork.pas
│   │   └── Services/
│   │       └── uOrderDomainService.pas
│   ├── Application/
│   │   ├── Commands/
│   │   │   └── uCreateOrderCommand.pas
│   │   ├── Queries/
│   │   │   └── uGetOrderQuery.pas
│   │   └── Services/
│   │       └── uOrderAppService.pas
│   ├── Infrastructure/
│   │   ├── Repositories/
│   │   │   ├── uCustomerRepository.pas
│   │   │   └── uOrderRepository.pas
│   │   ├── Database/
│   │   │   └── uDbContext.pas
│   │   └── DI/
│   │       └── uDIContainer.pas
│   └── Presentation/
│       ├── API/
│       │   └── uOrderController.pas
│       └── Forms/
│           └── frmMain.pas
├── tests/
│   ├── Domain/
│   └── Application/
└── EnterpriseApp.lpi
```

## 2. Repository Pattern

Repository Pattern แยกการเข้าถึงข้อมูลออกจาก Business Logic:

```pascal
// IRepository.pas - Interface สำหรับ Repository
unit IRepository;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections;

type
  // Generic Repository Interface
  IRepository<T> = interface
    ['{A1B2C3D4-E5F6-7890-ABCD-EF1234567890}']
    function GetById(const AId: Integer): T;
    function GetAll: TList<T>;
    procedure Add(const AEntity: T);
    procedure Update(const AEntity: T);
    procedure Delete(const AId: Integer);
    function Exists(const AId: Integer): Boolean;
  end;

  // Specification Pattern สำหรับ Query
  ISpecification<T> = interface
    ['{B2C3D4E5-F6A7-8901-BCDE-F12345678901}']
    function IsSatisfiedBy(const AEntity: T): Boolean;
    function ToExpression: string;
  end;

  // Extended Repository with Specification
  IQueryableRepository<T> = interface(IRepository<T>)
    ['{C3D4E5F6-A7B8-9012-CDEF-123456789012}']
    function Find(const ASpecification: ISpecification<T>): TList<T>;
    function Count(const ASpecification: ISpecification<T>): Integer;
    function FindPaged(const ASpecification: ISpecification<T>;
      const APage, APageSize: Integer): TList<T>;
  end;

implementation

end.
```

```pascal
// uCustomer.pas - Customer Entity
unit uCustomer;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, uMoney, uAddress;

type
  TCustomerStatus = (csActive, csInactive, csSuspended);

  TCustomer = class
  private
    FId: Integer;
    FFirstName: string;
    FLastName: string;
    FEmail: string;
    FPhone: string;
    FAddress: TAddress;
    FStatus: TCustomerStatus;
    FCreditLimit: TMoney;
    FCreatedAt: TDateTime;
    FUpdatedAt: TDateTime;

    function GetFullName: string;
    procedure Validate;

  public
    constructor Create(const AId: Integer; const AFirstName, ALastName, AEmail: string);
    destructor Destroy; override;

    // Business Methods
    procedure Activate;
    procedure Deactivate;
    procedure Suspend(const AReason: string);
    function CanPlaceOrder(const AAmount: TMoney): Boolean;
    procedure UpdateCreditLimit(const ANewLimit: TMoney);
    procedure UpdateAddress(const ANewAddress: TAddress);

    // Properties
    property Id: Integer read FId;
    property FirstName: string read FFirstName write FFirstName;
    property LastName: string read FLastName write FLastName;
    property FullName: string read GetFullName;
    property Email: string read FEmail write FEmail;
    property Phone: string read FPhone write FPhone;
    property Address: TAddress read FAddress;
    property Status: TCustomerStatus read FStatus;
    property CreditLimit: TMoney read FCreditLimit;
    property CreatedAt: TDateTime read FCreatedAt;
    property UpdatedAt: TDateTime read FUpdatedAt;
  end;

implementation

constructor TCustomer.Create(const AId: Integer;
  const AFirstName, ALastName, AEmail: string);
begin
  inherited Create;
  FId := AId;
  FFirstName := AFirstName;
  FLastName := ALastName;
  FEmail := AEmail;
  FStatus := csActive;
  FCreditLimit := TMoney.Create(0, 'THB');
  FCreatedAt := Now;
  FUpdatedAt := Now;
  Validate;
end;

destructor TCustomer.Destroy;
begin
  FCreditLimit.Free;
  if Assigned(FAddress) then
    FAddress.Free;
  inherited Destroy;
end;

function TCustomer.GetFullName: string;
begin
  Result := FFirstName + ' ' + FLastName;
end;

procedure TCustomer.Validate;
begin
  if Trim(FFirstName) = '' then
    raise Exception.Create('ชื่อลูกค้าต้องไม่ว่าง');
  if Trim(FLastName) = '' then
    raise Exception.Create('นามสกุลลูกค้าต้องไม่ว่าง');
  if not FEmail.Contains('@') then
    raise Exception.Create('อีเมลไม่ถูกต้อง');
end;

procedure TCustomer.Activate;
begin
  if FStatus = csSuspended then
    raise Exception.Create('ไม่สามารถ Activate ลูกค้าที่ถูก Suspend ได้โดยตรง');
  FStatus := csActive;
  FUpdatedAt := Now;
end;

procedure TCustomer.Deactivate;
begin
  FStatus := csInactive;
  FUpdatedAt := Now;
end;

procedure TCustomer.Suspend(const AReason: string);
begin
  if AReason = '' then
    raise Exception.Create('ต้องระบุเหตุผลในการ Suspend');
  FStatus := csSuspended;
  FUpdatedAt := Now;
end;

function TCustomer.CanPlaceOrder(const AAmount: TMoney): Boolean;
begin
  Result := (FStatus = csActive) and (AAmount.Amount <= FCreditLimit.Amount);
end;

procedure TCustomer.UpdateCreditLimit(const ANewLimit: TMoney);
begin
  if ANewLimit.Amount < 0 then
    raise Exception.Create('วงเงินเครดิตต้องไม่ติดลบ');
  FCreditLimit.Free;
  FCreditLimit := ANewLimit;
  FUpdatedAt := Now;
end;

procedure TCustomer.UpdateAddress(const ANewAddress: TAddress);
begin
  if Assigned(FAddress) then
    FAddress.Free;
  FAddress := ANewAddress;
  FUpdatedAt := Now;
end;

end.
```

## 3. Unit of Work Pattern

Unit of Work จัดการ Transaction ให้กับหลาย Repository:

```pascal
// IUnitOfWork.pas
unit IUnitOfWork;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, IRepository, uCustomer, uOrder, uProduct;

type
  IUnitOfWork = interface
    ['{D4E5F6A7-B8C9-0123-DEF0-234567890123}']
    // Repositories
    function GetCustomers: IQueryableRepository<TCustomer>;
    function GetOrders: IQueryableRepository<TOrder>;
    function GetProducts: IQueryableRepository<TProduct>;

    // Transaction Management
    procedure BeginTransaction;
    procedure Commit;
    procedure Rollback;

    // Save Changes
    function SaveChanges: Integer;
  end;

// uDbUnitOfWork.pas - Implementation
type
  TDbUnitOfWork = class(TInterfacedObject, IUnitOfWork)
  private
    FConnection: TSQLConnection;
    FTransaction: TSQLTransaction;
    FCustomerRepository: IQueryableRepository<TCustomer>;
    FOrderRepository: IQueryableRepository<TOrder>;
    FProductRepository: IQueryableRepository<TProduct>;
    FIsInTransaction: Boolean;

  public
    constructor Create(const AConnectionString: string);
    destructor Destroy; override;

    function GetCustomers: IQueryableRepository<TCustomer>;
    function GetOrders: IQueryableRepository<TOrder>;
    function GetProducts: IQueryableRepository<TProduct>;

    procedure BeginTransaction;
    procedure Commit;
    procedure Rollback;
    function SaveChanges: Integer;
  end;

implementation

constructor TDbUnitOfWork.Create(const AConnectionString: string);
begin
  inherited Create;
  // เชื่อมต่อฐานข้อมูล
  FConnection := TSQLConnection.Create(nil);
  FConnection.DatabaseName := AConnectionString;
  FConnection.Open;

  FTransaction := TSQLTransaction.Create(nil);
  FTransaction.DataBase := FConnection;
  FIsInTransaction := False;
end;

destructor TDbUnitOfWork.Destroy;
begin
  if FIsInTransaction then
    FTransaction.Rollback;
  FTransaction.Free;
  FConnection.Free;
  inherited Destroy;
end;

procedure TDbUnitOfWork.BeginTransaction;
begin
  if FIsInTransaction then
    raise Exception.Create('มี Transaction ที่ยังไม่ได้ปิดอยู่');
  FTransaction.StartTransaction;
  FIsInTransaction := True;
end;

procedure TDbUnitOfWork.Commit;
begin
  if not FIsInTransaction then
    raise Exception.Create('ไม่มี Transaction ที่ Active อยู่');
  FTransaction.Commit;
  FIsInTransaction := False;
end;

procedure TDbUnitOfWork.Rollback;
begin
  if FIsInTransaction then
  begin
    FTransaction.Rollback;
    FIsInTransaction := False;
  end;
end;

function TDbUnitOfWork.SaveChanges: Integer;
begin
  // บันทึกการเปลี่ยนแปลงทั้งหมด
  Result := 0;
  // Implementation ขึ้นอยู่กับ Change Tracking
end;

end.
```

## 4. Dependency Injection Container

DI Container จัดการการสร้าง Object และ Dependencies:

```pascal
// uDIContainer.pas
unit uDIContainer;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, TypInfo, RTTI;

type
  TLifetime = (ltTransient, ltSingleton, ltScoped);

  TRegistration = class
  private
    FServiceType: TClass;
    FImplementationType: TClass;
    FLifetime: TLifetime;
    FInstance: TObject;
    FFactory: TFunc<TObject>;
  public
    property ServiceType: TClass read FServiceType write FServiceType;
    property ImplementationType: TClass read FImplementationType write FImplementationType;
    property Lifetime: TLifetime read FLifetime write FLifetime;
    property Instance: TObject read FInstance write FInstance;
    property Factory: TFunc<TObject> read FFactory write FFactory;
  end;

  TDIContainer = class
  private
    FRegistrations: TObjectDictionary<string, TRegistration>;
    FScopedInstances: TObjectDictionary<string, TObject>;

    function GetKey(const AType: TClass): string;
    function CreateInstance(const AReg: TRegistration): TObject;
    function ResolveConstructorParams(const AClass: TClass): TArray<TObject>;

  public
    constructor Create;
    destructor Destroy; override;

    // การลงทะเบียน Services
    procedure Register<TService, TImpl: class>(
      ALifetime: TLifetime = ltTransient);
    procedure RegisterSingleton<TService, TImpl: class>;
    procedure RegisterInstance<TService: class>(const AInstance: TService);
    procedure RegisterFactory<TService: class>(
      const AFactory: TFunc<TService>);

    // การ Resolve
    function Resolve<TService: class>: TService;
    function TryResolve<TService: class>(out AService: TService): Boolean;

    // Scope Management
    function CreateScope: TDIContainer;

    class var Instance: TDIContainer;
    class constructor ClassCreate;
    class destructor ClassDestroy;
  end;

implementation

constructor TDIContainer.Create;
begin
  inherited Create;
  FRegistrations := TObjectDictionary<string, TRegistration>.Create([doOwnsValues]);
  FScopedInstances := TObjectDictionary<string, TObject>.Create([doOwnsValues]);
end;

destructor TDIContainer.Destroy;
begin
  FScopedInstances.Free;
  FRegistrations.Free;
  inherited Destroy;
end;

function TDIContainer.GetKey(const AType: TClass): string;
begin
  Result := AType.ClassName;
end;

procedure TDIContainer.Register<TService, TImpl>(ALifetime: TLifetime);
var
  Reg: TRegistration;
  Key: string;
begin
  Reg := TRegistration.Create;
  Reg.ServiceType := TService;
  Reg.ImplementationType := TImpl;
  Reg.Lifetime := ALifetime;

  Key := GetKey(TService);
  FRegistrations.AddOrSetValue(Key, Reg);
end;

procedure TDIContainer.RegisterSingleton<TService, TImpl>;
begin
  Register<TService, TImpl>(ltSingleton);
end;

procedure TDIContainer.RegisterInstance<TService>(const AInstance: TService);
var
  Reg: TRegistration;
  Key: string;
begin
  Reg := TRegistration.Create;
  Reg.ServiceType := TService;
  Reg.Lifetime := ltSingleton;
  Reg.Instance := AInstance;

  Key := GetKey(TService);
  FRegistrations.AddOrSetValue(Key, Reg);
end;

function TDIContainer.Resolve<TService>: TService;
var
  Key: string;
  Reg: TRegistration;
begin
  Key := GetKey(TService);
  if not FRegistrations.TryGetValue(Key, Reg) then
    raise Exception.CreateFmt('ไม่พบการลงทะเบียนสำหรับ %s', [TService.ClassName]);

  case Reg.Lifetime of
    ltTransient:
      Result := TService(CreateInstance(Reg));

    ltSingleton:
    begin
      if not Assigned(Reg.Instance) then
        Reg.Instance := CreateInstance(Reg);
      Result := TService(Reg.Instance);
    end;

    ltScoped:
    begin
      if not FScopedInstances.TryGetValue(Key, TObject(Result)) then
      begin
        Result := TService(CreateInstance(Reg));
        FScopedInstances.Add(Key, Result);
      end;
    end;
  end;
end;

function TDIContainer.CreateInstance(const AReg: TRegistration): TObject;
begin
  if Assigned(AReg.Factory) then
    Result := AReg.Factory()
  else if Assigned(AReg.ImplementationType) then
    Result := AReg.ImplementationType.Create
  else
    raise Exception.Create('ไม่สามารถสร้าง Instance ได้');
end;

class constructor TDIContainer.ClassCreate;
begin
  Instance := TDIContainer.Create;
end;

class destructor TDIContainer.ClassDestroy;
begin
  Instance.Free;
end;

end.
```

## 5. Service Layer Pattern

Application Services ประสานงานระหว่าง Domain Layer และ Infrastructure:

```pascal
// uOrderAppService.pas - Application Service
unit uOrderAppService;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections,
  uCustomer, uOrder, uProduct,
  IRepository, IUnitOfWork,
  uCreateOrderCommand, uOrderDto;

type
  TOrderAppService = class
  private
    FUnitOfWork: IUnitOfWork;
    FEmailService: IEmailService;
    FEventBus: IEventBus;

    procedure ValidateOrder(const ACommand: TCreateOrderCommand);
    function BuildOrderFromCommand(
      const ACommand: TCreateOrderCommand;
      const ACustomer: TCustomer): TOrder;
    procedure SendOrderConfirmation(const AOrder: TOrder);

  public
    constructor Create(
      const AUnitOfWork: IUnitOfWork;
      const AEmailService: IEmailService;
      const AEventBus: IEventBus);

    // Commands (Write Operations)
    function CreateOrder(const ACommand: TCreateOrderCommand): TOrderDto;
    procedure CancelOrder(const AOrderId: Integer; const AReason: string);
    procedure ShipOrder(const AOrderId: Integer;
      const ATrackingNumber: string);
    procedure CompleteOrder(const AOrderId: Integer);

    // Queries (Read Operations)
    function GetOrder(const AOrderId: Integer): TOrderDto;
    function GetCustomerOrders(const ACustomerId: Integer): TList<TOrderDto>;
    function GetPendingOrders: TList<TOrderDto>;
  end;

implementation

constructor TOrderAppService.Create(
  const AUnitOfWork: IUnitOfWork;
  const AEmailService: IEmailService;
  const AEventBus: IEventBus);
begin
  inherited Create;
  FUnitOfWork := AUnitOfWork;
  FEmailService := AEmailService;
  FEventBus := AEventBus;
end;

function TOrderAppService.CreateOrder(
  const ACommand: TCreateOrderCommand): TOrderDto;
var
  Customer: TCustomer;
  Order: TOrder;
  Item: TCreateOrderItemCommand;
  Product: TProduct;
begin
  // 1. Validate Command
  ValidateOrder(ACommand);

  FUnitOfWork.BeginTransaction;
  try
    // 2. Load Customer
    Customer := FUnitOfWork.GetCustomers.GetById(ACommand.CustomerId);
    if not Assigned(Customer) then
      raise Exception.CreateFmt('ไม่พบลูกค้า ID: %d', [ACommand.CustomerId]);

    // 3. Build Order (Domain Logic)
    Order := BuildOrderFromCommand(ACommand, Customer);
    try
      // 4. Check Credit Limit
      if not Customer.CanPlaceOrder(Order.TotalAmount) then
        raise Exception.Create('วงเงินเครดิตไม่เพียงพอ');

      // 5. Reserve Stock
      for Item in ACommand.Items do
      begin
        Product := FUnitOfWork.GetProducts.GetById(Item.ProductId);
        Product.ReserveStock(Item.Quantity);
        FUnitOfWork.GetProducts.Update(Product);
      end;

      // 6. Save Order
      FUnitOfWork.GetOrders.Add(Order);

      // 7. Update Customer Stats
      Customer.RecordOrder(Order.TotalAmount);
      FUnitOfWork.GetCustomers.Update(Customer);

      // 8. Commit Transaction
      FUnitOfWork.Commit;

      // 9. Send Confirmation (Outside Transaction)
      SendOrderConfirmation(Order);

      // 10. Publish Domain Event
      FEventBus.Publish(TOrderCreatedEvent.Create(Order.Id));

      // 11. Map to DTO
      Result := TOrderMapper.ToDto(Order);

    finally
      Order.Free;
    end;

  except
    FUnitOfWork.Rollback;
    raise;
  end;
end;

procedure TOrderAppService.CancelOrder(
  const AOrderId: Integer; const AReason: string);
var
  Order: TOrder;
begin
  FUnitOfWork.BeginTransaction;
  try
    Order := FUnitOfWork.GetOrders.GetById(AOrderId);
    if not Assigned(Order) then
      raise Exception.CreateFmt('ไม่พบ Order ID: %d', [AOrderId]);

    Order.Cancel(AReason);
    FUnitOfWork.GetOrders.Update(Order);
    FUnitOfWork.Commit;

    FEventBus.Publish(TOrderCancelledEvent.Create(AOrderId, AReason));

  except
    FUnitOfWork.Rollback;
    raise;
  end;
end;

procedure TOrderAppService.ValidateOrder(
  const ACommand: TCreateOrderCommand);
begin
  if ACommand.CustomerId <= 0 then
    raise Exception.Create('CustomerId ไม่ถูกต้อง');
  if ACommand.Items.Count = 0 then
    raise Exception.Create('ต้องมีรายการสินค้าอย่างน้อย 1 รายการ');
  // เพิ่ม Validation อื่นๆ ตามต้องการ
end;

end.
```

## 6. Value Objects

Value Objects คือ Object ที่ไม่มี Identity แต่มีค่า (Immutable):

```pascal
// uMoney.pas - Money Value Object
unit uMoney;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  TMoney = class sealed
  private
    FAmount: Currency;
    FCurrency: string;

    procedure ValidateCurrency(const ACurrency: string);

  public
    constructor Create(const AAmount: Currency; const ACurrency: string);

    // Arithmetic Operations (Returns New Instance - Immutable)
    function Add(const AOther: TMoney): TMoney;
    function Subtract(const AOther: TMoney): TMoney;
    function Multiply(const AFactor: Double): TMoney;
    function Divide(const ADivisor: Double): TMoney;

    // Comparison
    function Equals(const AOther: TMoney): Boolean;
    function IsGreaterThan(const AOther: TMoney): Boolean;
    function IsLessThan(const AOther: TMoney): Boolean;
    function IsZero: Boolean;
    function IsNegative: Boolean;

    // Formatting
    function ToString: string; override;
    function ToFormattedString(const AFormat: string = '#,##0.00'): string;

    property Amount: Currency read FAmount;
    property Currency: string read FCurrency;
  end;

implementation

constructor TMoney.Create(const AAmount: Currency; const ACurrency: string);
begin
  inherited Create;
  ValidateCurrency(ACurrency);
  FAmount := AAmount;
  FCurrency := UpperCase(ACurrency);
end;

procedure TMoney.ValidateCurrency(const ACurrency: string);
const
  ValidCurrencies: array[0..4] of string = ('THB', 'USD', 'EUR', 'GBP', 'JPY');
var
  Valid: Boolean;
  I: Integer;
  UpperCurr: string;
begin
  UpperCurr := UpperCase(ACurrency);
  Valid := False;
  for I := 0 to High(ValidCurrencies) do
    if ValidCurrencies[I] = UpperCurr then
    begin
      Valid := True;
      Break;
    end;
  if not Valid then
    raise Exception.CreateFmt('สกุลเงิน %s ไม่ถูกต้อง', [ACurrency]);
end;

function TMoney.Add(const AOther: TMoney): TMoney;
begin
  if FCurrency <> AOther.Currency then
    raise Exception.Create('ไม่สามารถบวกเงินต่างสกุลได้');
  Result := TMoney.Create(FAmount + AOther.Amount, FCurrency);
end;

function TMoney.Subtract(const AOther: TMoney): TMoney;
begin
  if FCurrency <> AOther.Currency then
    raise Exception.Create('ไม่สามารถลบเงินต่างสกุลได้');
  Result := TMoney.Create(FAmount - AOther.Amount, FCurrency);
end;

function TMoney.Multiply(const AFactor: Double): TMoney;
begin
  Result := TMoney.Create(FAmount * AFactor, FCurrency);
end;

function TMoney.Divide(const ADivisor: Double): TMoney;
begin
  if ADivisor = 0 then
    raise Exception.Create('ไม่สามารถหารด้วยศูนย์ได้');
  Result := TMoney.Create(FAmount / ADivisor, FCurrency);
end;

function TMoney.Equals(const AOther: TMoney): Boolean;
begin
  Result := (FCurrency = AOther.Currency) and (FAmount = AOther.Amount);
end;

function TMoney.IsGreaterThan(const AOther: TMoney): Boolean;
begin
  if FCurrency <> AOther.Currency then
    raise Exception.Create('ไม่สามารถเปรียบเทียบเงินต่างสกุลได้');
  Result := FAmount > AOther.Amount;
end;

function TMoney.IsZero: Boolean;
begin
  Result := FAmount = 0;
end;

function TMoney.IsNegative: Boolean;
begin
  Result := FAmount < 0;
end;

function TMoney.ToString: string;
begin
  Result := FormatFloat('#,##0.00', FAmount) + ' ' + FCurrency;
end;

end.
```

## 7. Complete Enterprise Project Setup

### การตั้งค่า DI Container ใน Application Startup

```pascal
// uAppStartup.pas - Application Bootstrap
unit uAppStartup;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, uDIContainer,
  IUnitOfWork, uDbUnitOfWork,
  IEmailService, uSmtpEmailService,
  IEventBus, uEventBus,
  uOrderAppService, uCustomerAppService,
  uProductAppService;

type
  TAppStartup = class
  public
    class procedure ConfigureServices(const AContainer: TDIContainer);
    class procedure Configure(const AContainer: TDIContainer);
  end;

implementation

class procedure TAppStartup.ConfigureServices(
  const AContainer: TDIContainer);
begin
  // Infrastructure Services
  AContainer.RegisterSingleton<IUnitOfWork, TDbUnitOfWork>;
  AContainer.RegisterSingleton<IEmailService, TSmtpEmailService>;
  AContainer.RegisterSingleton<IEventBus, TEventBus>;

  // Application Services (Transient - สร้างใหม่ทุกครั้ง)
  AContainer.Register<TOrderAppService, TOrderAppService>(ltTransient);
  AContainer.Register<TCustomerAppService, TCustomerAppService>(ltTransient);
  AContainer.Register<TProductAppService, TProductAppService>(ltTransient);
end;

class procedure TAppStartup.Configure(const AContainer: TDIContainer);
var
  EventBus: IEventBus;
begin
  // ตั้งค่า Event Handlers
  EventBus := AContainer.Resolve<IEventBus>;
  EventBus.Subscribe<TOrderCreatedEvent>(
    procedure(AEvent: TOrderCreatedEvent)
    begin
      // Handle order created event
      WriteLn('Order created: ', AEvent.OrderId);
    end
  );
end;

end.
```

### Main Program - การประกอบทุกอย่างเข้าด้วยกัน

```pascal
// EnterpriseApp.pas - Main Program
program EnterpriseApp;

{$mode objfpc}{$H+}

uses
  SysUtils,
  uDIContainer,
  uAppStartup,
  uOrderAppService,
  uCreateOrderCommand;

var
  Container: TDIContainer;
  OrderService: TOrderAppService;
  Command: TCreateOrderCommand;
  OrderDto: TOrderDto;

begin
  WriteLn('=== Enterprise Application Starting ===');

  // 1. Setup DI Container
  Container := TDIContainer.Instance;
  TAppStartup.ConfigureServices(Container);
  TAppStartup.Configure(Container);

  WriteLn('Services configured successfully');

  // 2. Use Application Services
  OrderService := Container.Resolve<TOrderAppService>;

  // 3. Create an Order
  Command := TCreateOrderCommand.Create;
  try
    Command.CustomerId := 1001;
    Command.Items.Add(TCreateOrderItemCommand.Create(2001, 5)); // Product 2001, Qty 5
    Command.Items.Add(TCreateOrderItemCommand.Create(2002, 2)); // Product 2002, Qty 2
    Command.ShippingAddressId := 3001;
    Command.Notes := 'กรุณาส่งด่วน';

    OrderDto := OrderService.CreateOrder(Command);
    WriteLn('Order created successfully: ', OrderDto.OrderNumber);
    WriteLn('Total Amount: ', OrderDto.TotalAmount.ToString);

  finally
    Command.Free;
  end;

  WriteLn('=== Application Finished ===');

  ReadLn;
end.
```

## 8. Configuration Management

```pascal
// uConfiguration.pas - Application Configuration
unit uConfiguration;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, IniFiles, jsonparser, fpjson;

type
  TDatabaseConfig = record
    Host: string;
    Port: Integer;
    DatabaseName: string;
    Username: string;
    Password: string;
    MaxConnections: Integer;
    ConnectionTimeout: Integer;
  end;

  TEmailConfig = record
    SmtpHost: string;
    SmtpPort: Integer;
    Username: string;
    Password: string;
    UseSsl: Boolean;
    FromAddress: string;
    FromName: string;
  end;

  TCacheConfig = record
    RedisHost: string;
    RedisPort: Integer;
    DefaultExpiry: Integer; // seconds
    MaxMemory: string;
  end;

  TAppConfiguration = class
  private
    FDatabase: TDatabaseConfig;
    FEmail: TEmailConfig;
    FCache: TCacheConfig;
    FEnvironment: string;
    FLogLevel: string;
    FApiBaseUrl: string;

    procedure LoadFromJson(const AJsonFile: string);
    procedure LoadFromIni(const AIniFile: string);
    procedure LoadFromEnvironment;

    class var FInstance: TAppConfiguration;

  public
    constructor Create;

    class function GetInstance: TAppConfiguration;
    class procedure Initialize(const AConfigFile: string = '');

    property Database: TDatabaseConfig read FDatabase;
    property Email: TEmailConfig read FEmail;
    property Cache: TCacheConfig read FCache;
    property Environment: string read FEnvironment;
    property LogLevel: string read FLogLevel;
    property ApiBaseUrl: string read FApiBaseUrl;

    function IsProduction: Boolean;
    function IsDevelopment: Boolean;
    function GetConnectionString: string;
  end;

function Config: TAppConfiguration;

implementation

function Config: TAppConfiguration;
begin
  Result := TAppConfiguration.GetInstance;
end;

constructor TAppConfiguration.Create;
begin
  inherited Create;
  // Default values
  FDatabase.MaxConnections := 10;
  FDatabase.ConnectionTimeout := 30;
  FEmail.SmtpPort := 587;
  FEmail.UseSsl := True;
  FCache.DefaultExpiry := 3600;
  FEnvironment := 'development';
  FLogLevel := 'info';
end;

class function TAppConfiguration.GetInstance: TAppConfiguration;
begin
  if not Assigned(FInstance) then
    FInstance := TAppConfiguration.Create;
  Result := FInstance;
end;

class procedure TAppConfiguration.Initialize(const AConfigFile: string);
var
  ConfigFile: string;
begin
  ConfigFile := AConfigFile;
  if ConfigFile = '' then
  begin
    // ค้นหาไฟล์ config ตามลำดับ
    if FileExists('appsettings.local.json') then
      ConfigFile := 'appsettings.local.json'
    else if FileExists('appsettings.json') then
      ConfigFile := 'appsettings.json'
    else
      ConfigFile := 'app.ini';
  end;

  GetInstance.LoadFromEnvironment;

  if FileExists(ConfigFile) then
  begin
    if ConfigFile.EndsWith('.json') then
      GetInstance.LoadFromJson(ConfigFile)
    else
      GetInstance.LoadFromIni(ConfigFile);
  end;
end;

procedure TAppConfiguration.LoadFromEnvironment;
begin
  // โหลดจาก Environment Variables (สำหรับ Docker/Kubernetes)
  FEnvironment := GetEnvironmentVariable('APP_ENV');
  if FEnvironment = '' then FEnvironment := 'development';

  FDatabase.Host := GetEnvironmentVariable('DB_HOST');
  if FDatabase.Host = '' then FDatabase.Host := 'localhost';

  FDatabase.Port := StrToIntDef(GetEnvironmentVariable('DB_PORT'), 5432);
  FDatabase.DatabaseName := GetEnvironmentVariable('DB_NAME');
  FDatabase.Username := GetEnvironmentVariable('DB_USER');
  FDatabase.Password := GetEnvironmentVariable('DB_PASSWORD');
end;

procedure TAppConfiguration.LoadFromJson(const AJsonFile: string);
var
  JsonStr: TStringList;
  Json: TJSONData;
  DbObj, EmailObj: TJSONObject;
begin
  JsonStr := TStringList.Create;
  try
    JsonStr.LoadFromFile(AJsonFile);
    Json := GetJSON(JsonStr.Text);
    try
      if Json is TJSONObject then
      begin
        // Database Config
        DbObj := TJSONObject(TJSONObject(Json).Find('database'));
        if Assigned(DbObj) then
        begin
          if FDatabase.Host = '' then
            FDatabase.Host := DbObj.Get('host', 'localhost');
          FDatabase.Port := DbObj.Get('port', 5432);
          if FDatabase.DatabaseName = '' then
            FDatabase.DatabaseName := DbObj.Get('database', '');
          if FDatabase.Username = '' then
            FDatabase.Username := DbObj.Get('username', '');
          if FDatabase.Password = '' then
            FDatabase.Password := DbObj.Get('password', '');
          FDatabase.MaxConnections := DbObj.Get('maxConnections', 10);
        end;

        // Email Config
        EmailObj := TJSONObject(TJSONObject(Json).Find('email'));
        if Assigned(EmailObj) then
        begin
          FEmail.SmtpHost := EmailObj.Get('smtpHost', '');
          FEmail.SmtpPort := EmailObj.Get('smtpPort', 587);
          FEmail.Username := EmailObj.Get('username', '');
          FEmail.FromAddress := EmailObj.Get('fromAddress', '');
          FEmail.FromName := EmailObj.Get('fromName', 'System');
        end;
      end;
    finally
      Json.Free;
    end;
  finally
    JsonStr.Free;
  end;
end;

function TAppConfiguration.IsProduction: Boolean;
begin
  Result := LowerCase(FEnvironment) = 'production';
end;

function TAppConfiguration.IsDevelopment: Boolean;
begin
  Result := LowerCase(FEnvironment) = 'development';
end;

function TAppConfiguration.GetConnectionString: string;
begin
  Result := Format('host=%s port=%d dbname=%s user=%s password=%s',
    [FDatabase.Host, FDatabase.Port, FDatabase.DatabaseName,
     FDatabase.Username, FDatabase.Password]);
end;

end.
```

## 9. Logging Infrastructure

```pascal
// uLogger.pas - Structured Logging
unit uLogger;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, fpjson;

type
  TLogLevel = (llTrace, llDebug, llInfo, llWarning, llError, llCritical);

  TLogEntry = record
    Timestamp: TDateTime;
    Level: TLogLevel;
    Message: string;
    Source: string;
    CorrelationId: string;
    Extra: TJSONObject;
  end;

  ILogSink = interface
    procedure Write(const AEntry: TLogEntry);
    procedure Flush;
  end;

  ILogger = interface
    procedure Trace(const AMessage: string; const AExtra: TJSONObject = nil);
    procedure Debug(const AMessage: string; const AExtra: TJSONObject = nil);
    procedure Info(const AMessage: string; const AExtra: TJSONObject = nil);
    procedure Warning(const AMessage: string; const AExtra: TJSONObject = nil);
    procedure Error(const AMessage: string; const AException: Exception = nil;
      const AExtra: TJSONObject = nil);
    procedure Critical(const AMessage: string; const AException: Exception = nil);

    function WithContext(const AKey, AValue: string): ILogger;
    function WithCorrelationId(const AId: string): ILogger;
  end;

  TLogger = class(TInterfacedObject, ILogger)
  private
    FSinks: TInterfaceList;
    FSource: string;
    FCorrelationId: string;
    FContext: TJSONObject;
    FMinLevel: TLogLevel;
    FCriticalSection: TCriticalSection;

    procedure WriteToSinks(const ALevel: TLogLevel; const AMessage: string;
      const AException: Exception = nil; const AExtra: TJSONObject = nil);
    function LevelToString(const ALevel: TLogLevel): string;

  public
    constructor Create(const ASource: string; AMinLevel: TLogLevel = llInfo);
    destructor Destroy; override;

    procedure AddSink(const ASink: ILogSink);

    procedure Trace(const AMessage: string; const AExtra: TJSONObject = nil);
    procedure Debug(const AMessage: string; const AExtra: TJSONObject = nil);
    procedure Info(const AMessage: string; const AExtra: TJSONObject = nil);
    procedure Warning(const AMessage: string; const AExtra: TJSONObject = nil);
    procedure Error(const AMessage: string; const AException: Exception = nil;
      const AExtra: TJSONObject = nil);
    procedure Critical(const AMessage: string; const AException: Exception = nil);

    function WithContext(const AKey, AValue: string): ILogger;
    function WithCorrelationId(const AId: string): ILogger;
  end;

  // Console Sink
  TConsoleSink = class(TInterfacedObject, ILogSink)
  public
    procedure Write(const AEntry: TLogEntry);
    procedure Flush;
  end;

  // File Sink with Rotation
  TFileSink = class(TInterfacedObject, ILogSink)
  private
    FFilePath: string;
    FMaxFileSize: Int64;
    FMaxFiles: Integer;
    FCurrentFile: TFileStream;
    FWriter: TStreamWriter;

    procedure RotateIfNeeded;
    procedure OpenFile;

  public
    constructor Create(const AFilePath: string;
      AMaxFileSize: Int64 = 10 * 1024 * 1024; // 10MB
      AMaxFiles: Integer = 5);
    destructor Destroy; override;

    procedure Write(const AEntry: TLogEntry);
    procedure Flush;
  end;

implementation

constructor TLogger.Create(const ASource: string; AMinLevel: TLogLevel);
begin
  inherited Create;
  FSinks := TInterfaceList.Create;
  FSource := ASource;
  FMinLevel := AMinLevel;
  FContext := TJSONObject.Create;
  FCriticalSection := TCriticalSection.Create;
end;

destructor TLogger.Destroy;
begin
  FCriticalSection.Free;
  FContext.Free;
  FSinks.Free;
  inherited Destroy;
end;

procedure TLogger.AddSink(const ASink: ILogSink);
begin
  FSinks.Add(ASink);
end;

procedure TLogger.WriteToSinks(const ALevel: TLogLevel;
  const AMessage: string; const AException: Exception;
  const AExtra: TJSONObject);
var
  Entry: TLogEntry;
  I: Integer;
  Sink: ILogSink;
begin
  if ALevel < FMinLevel then Exit;

  FCriticalSection.Acquire;
  try
    Entry.Timestamp := Now;
    Entry.Level := ALevel;
    Entry.Source := FSource;
    Entry.CorrelationId := FCorrelationId;

    if Assigned(AException) then
      Entry.Message := AMessage + ' | Exception: ' + AException.Message
    else
      Entry.Message := AMessage;

    Entry.Extra := AExtra;

    for I := 0 to FSinks.Count - 1 do
    begin
      Sink := ILogSink(FSinks[I]);
      try
        Sink.Write(Entry);
      except
        // ไม่ให้ Error ใน Logger ทำให้โปรแกรมพัง
      end;
    end;
  finally
    FCriticalSection.Release;
  end;
end;

procedure TLogger.Info(const AMessage: string; const AExtra: TJSONObject);
begin
  WriteToSinks(llInfo, AMessage, nil, AExtra);
end;

procedure TLogger.Error(const AMessage: string; const AException: Exception;
  const AExtra: TJSONObject);
begin
  WriteToSinks(llError, AMessage, AException, AExtra);
end;

function TLogger.LevelToString(const ALevel: TLogLevel): string;
begin
  case ALevel of
    llTrace: Result := 'TRACE';
    llDebug: Result := 'DEBUG';
    llInfo: Result := 'INFO';
    llWarning: Result := 'WARN';
    llError: Result := 'ERROR';
    llCritical: Result := 'CRITICAL';
  end;
end;

// Console Sink Implementation
procedure TConsoleSink.Write(const AEntry: TLogEntry);
const
  Colors: array[TLogLevel] of Byte = (7, 11, 10, 14, 12, 13);
begin
  TextColor(Colors[AEntry.Level]);
  WriteLn(Format('[%s] [%s] %s: %s',
    [FormatDateTime('yyyy-mm-dd hh:nn:ss', AEntry.Timestamp),
     LevelNames[AEntry.Level],
     AEntry.Source,
     AEntry.Message]));
  TextColor(LightGray);
end;

procedure TConsoleSink.Flush;
begin
  // Console ไม่ต้อง Flush
end;

end.
```

## 10. สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:

1. **N-Tier Architecture** - การแบ่งโปรแกรมเป็นชั้นต่างๆ ที่มีหน้าที่ชัดเจน
2. **Repository Pattern** - แยกการเข้าถึงข้อมูลออกจาก Business Logic
3. **Unit of Work** - จัดการ Transaction ให้กับหลาย Repository
4. **Dependency Injection** - ทำให้โค้ดทดสอบได้และยืดหยุ่น
5. **Service Layer** - ประสานงานระหว่าง Domain และ Infrastructure
6. **Value Objects** - Immutable objects ที่แทนค่าแนวคิดทางธุรกิจ
7. **Configuration Management** - จัดการการตั้งค่าสำหรับหลาย Environment
8. **Structured Logging** - การบันทึก Log แบบมีโครงสร้าง

### แนวทางปฏิบัติที่ดี (Best Practices):
- แยก Concern ออกจากกัน (Separation of Concerns)
- ใช้ Interface แทน Concrete Class เพื่อ Testability
- ทุก Service ต้องมี Unit Test
- ใช้ Configuration สำหรับค่าที่เปลี่ยนแปลงได้
- Log ทุก Operation ที่สำคัญ
- จัดการ Error อย่างสม่ำเสมอ

ในบทถัดไปเราจะเรียนรู้เกี่ยวกับ Microservices Architecture ที่นำ Enterprise Patterns เหล่านี้ไปใช้ในสถาปัตยกรรมแบบกระจาย
