# ตอนที่ 83: Domain-Driven Design (DDD) กับ Pascal/Lazarus

## บทนำ: Domain-Driven Design คืออะไร?

Domain-Driven Design (DDD) คือวิธีการพัฒนาซอฟต์แวร์ที่เน้นการทำความเข้าใจ Business Domain (โดเมนธุรกิจ) ก่อน แล้วจึงออกแบบซอฟต์แวร์ให้สะท้อน Domain นั้น

## 1. Building Blocks of DDD

### Bounded Contexts (บริบทที่จำกัด)

```
┌─────────────────────────────────────────────────────────┐
│                     E-Commerce System                   │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │  Sales       │  │  Inventory   │  │  Shipping    │  │
│  │  Context     │  │  Context     │  │  Context     │  │
│  │              │  │              │  │              │  │
│  │  Customer    │  │  Product     │  │  Shipment    │  │
│  │  Order       │  │  StockItem   │  │  Carrier     │  │
│  │  OrderLine   │  │  Warehouse   │  │  Route       │  │
│  └──────────────┘  └──────────────┘  └──────────────┘  │
│                                                         │
│  ┌──────────────┐  ┌──────────────┐                     │
│  │  Payment     │  │  Identity    │                     │
│  │  Context     │  │  Context     │                     │
│  │              │  │              │                     │
│  │  Invoice     │  │  User        │                     │
│  │  Payment     │  │  Role        │                     │
│  │  Refund      │  │  Permission  │                     │
│  └──────────────┘  └──────────────┘                     │
└─────────────────────────────────────────────────────────┘
```

## 2. Entities และ Value Objects

```pascal
// Domain/ValueObjects/uEmail.pas
unit uEmail;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  // Email Value Object - Immutable
  TEmail = class sealed
  private
    FValue: string;

    class function Validate(const AEmail: string): Boolean; static;

  public
    constructor Create(const AEmail: string);

    function Equals(const AOther: TEmail): Boolean;
    function ToString: string; override;

    property Value: string read FValue;

    class function TryCreate(const AEmail: string;
      out AResult: TEmail): Boolean; static;
  end;

implementation

constructor TEmail.Create(const AEmail: string);
begin
  inherited Create;
  if not Validate(AEmail) then
    raise EDomainException.CreateFmt('อีเมล "%s" ไม่ถูกต้อง', [AEmail]);
  FValue := LowerCase(Trim(AEmail));
end;

class function TEmail.Validate(const AEmail: string): Boolean;
var
  AtPos, DotPos: Integer;
begin
  Result := False;
  AtPos := Pos('@', AEmail);
  if (AtPos <= 1) or (AtPos = Length(AEmail)) then Exit;
  DotPos := LastDelimiter('.', AEmail);
  if (DotPos <= AtPos + 1) or (DotPos = Length(AEmail)) then Exit;
  Result := True;
end;

class function TEmail.TryCreate(const AEmail: string;
  out AResult: TEmail): Boolean;
begin
  try
    AResult := TEmail.Create(AEmail);
    Result := True;
  except
    AResult := nil;
    Result := False;
  end;
end;

function TEmail.Equals(const AOther: TEmail): Boolean;
begin
  Result := FValue = AOther.Value;
end;

function TEmail.ToString: string;
begin
  Result := FValue;
end;

end.
```

```pascal
// Domain/ValueObjects/uPersonName.pas
unit uPersonName;

{$mode objfpc}{$H+}

interface

type
  TPersonName = class sealed
  private
    FFirstName: string;
    FLastName: string;
    FMiddleName: string;

    procedure Validate;

  public
    constructor Create(const AFirstName, ALastName: string;
      const AMiddleName: string = '');

    function GetFullName: string;
    function GetFormalName: string;   // นามสกุล ชื่อ
    function GetInitials: string;     // ตัวอักษรย่อ
    function Equals(const AOther: TPersonName): Boolean;

    property FirstName: string read FFirstName;
    property LastName: string read FLastName;
    property MiddleName: string read FMiddleName;
    property FullName: string read GetFullName;
  end;

implementation

constructor TPersonName.Create(const AFirstName, ALastName: string;
  const AMiddleName: string);
begin
  inherited Create;
  FFirstName := Trim(AFirstName);
  FLastName := Trim(ALastName);
  FMiddleName := Trim(AMiddleName);
  Validate;
end;

procedure TPersonName.Validate;
begin
  if FFirstName = '' then
    raise EDomainException.Create('ชื่อต้องไม่ว่าง');
  if FLastName = '' then
    raise EDomainException.Create('นามสกุลต้องไม่ว่าง');
  if Length(FFirstName) > 100 then
    raise EDomainException.Create('ชื่อยาวเกินไป (สูงสุด 100 ตัวอักษร)');
  if Length(FLastName) > 100 then
    raise EDomainException.Create('นามสกุลยาวเกินไป (สูงสุด 100 ตัวอักษร)');
end;

function TPersonName.GetFullName: string;
begin
  if FMiddleName <> '' then
    Result := FFirstName + ' ' + FMiddleName + ' ' + FLastName
  else
    Result := FFirstName + ' ' + FLastName;
end;

function TPersonName.GetFormalName: string;
begin
  Result := FLastName + ', ' + FFirstName;
end;

function TPersonName.GetInitials: string;
begin
  Result := UpperCase(FFirstName[1]) + '.' + UpperCase(FLastName[1]) + '.';
end;

end.
```

## 3. Aggregate Root

Aggregate Root คือ Entity หลักที่ควบคุม Aggregate ทั้งหมด:

```pascal
// Domain/Aggregates/uOrderAggregate.pas
unit uOrderAggregate;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections,
  uEntity, uDomainEvents,
  uMoney, uCustomerId, uProductId;

type
  TOrderStatus = (
    osCreated,        // เพิ่งสร้าง
    osConfirmed,      // ยืนยันแล้ว
    osPaid,           // ชำระแล้ว
    osShipped,        // จัดส่งแล้ว
    osDelivered,      // ส่งถึงแล้ว
    osCancelled,      // ยกเลิก
    osRefunded        // คืนเงินแล้ว
  );

  TOrderLine = class
  private
    FOrderLineId: Integer;
    FProductId: TProductId;
    FProductName: string;
    FQuantity: Integer;
    FUnitPrice: TMoney;
    FDiscount: TMoney;

    function GetTotal: TMoney;

  public
    constructor Create(AOrderLineId: Integer; AProductId: TProductId;
      const AProductName: string; AQuantity: Integer;
      AUnitPrice: TMoney; ADiscount: TMoney = nil);
    destructor Destroy; override;

    procedure UpdateQuantity(ANewQuantity: Integer);

    property OrderLineId: Integer read FOrderLineId;
    property ProductId: TProductId read FProductId;
    property ProductName: string read FProductName;
    property Quantity: Integer read FQuantity;
    property UnitPrice: TMoney read FUnitPrice;
    property Discount: TMoney read FDiscount;
    property Total: TMoney read GetTotal;
  end;

  TShippingAddress = class
  private
    FStreet: string;
    FCity: string;
    FProvince: string;
    FPostalCode: string;
    FCountry: string;

  public
    constructor Create(const AStreet, ACity, AProvince,
      APostalCode, ACountry: string);

    function ToString: string; override;
    function Equals(const AOther: TShippingAddress): Boolean;

    property Street: string read FStreet;
    property City: string read FCity;
    property Province: string read FProvince;
    property PostalCode: string read FPostalCode;
    property Country: string read FCountry;
  end;

  // Aggregate Root
  TOrder = class(TEntity)
  private
    FOrderNumber: string;
    FCustomerId: TCustomerId;
    FStatus: TOrderStatus;
    FOrderLines: TObjectList<TOrderLine>;
    FShippingAddress: TShippingAddress;
    FNotes: string;
    FCreatedAt: TDateTime;
    FConfirmedAt: TDateTime;
    FPaidAt: TDateTime;
    FShippedAt: TDateTime;
    FDeliveredAt: TDateTime;
    FCancelledAt: TDateTime;
    FCancelReason: string;
    FDomainEvents: TList<IDomainEvent>;
    FNextLineId: Integer;

    function GenerateOrderNumber: string;
    procedure ValidateStatusTransition(ANewStatus: TOrderStatus);
    procedure AddDomainEvent(const AEvent: IDomainEvent);
    function GetSubTotal: TMoney;
    function GetTotalDiscount: TMoney;
    function GetTotalTax: TMoney;
    function GetGrandTotal: TMoney;

  public
    constructor Create(const ACustomerId: TCustomerId;
      const AShippingAddress: TShippingAddress);
    destructor Destroy; override;

    // Business Methods (Domain Logic)
    procedure AddOrderLine(const AProductId: TProductId;
      const AProductName: string; AQuantity: Integer;
      AUnitPrice: TMoney; ADiscount: TMoney = nil);
    procedure RemoveOrderLine(AOrderLineId: Integer);
    procedure UpdateOrderLineQuantity(AOrderLineId: Integer;
      ANewQuantity: Integer);
    procedure UpdateShippingAddress(const ANewAddress: TShippingAddress);

    procedure Confirm;
    procedure Pay(const APaymentReference: string);
    procedure Ship(const ATrackingNumber, ACarrier: string);
    procedure Deliver;
    procedure Cancel(const AReason: string);
    procedure Refund(const ARefundReason: string);

    function CanBeCancelled: Boolean;
    function CanBeShipped: Boolean;
    function HasOrderLine(const AProductId: TProductId): Boolean;

    // Domain Events (Collected Events)
    function GetAndClearDomainEvents: TList<IDomainEvent>;

    property OrderNumber: string read FOrderNumber;
    property CustomerId: TCustomerId read FCustomerId;
    property Status: TOrderStatus read FStatus;
    property OrderLines: TObjectList<TOrderLine> read FOrderLines;
    property ShippingAddress: TShippingAddress read FShippingAddress;
    property Notes: string read FNotes write FNotes;
    property CreatedAt: TDateTime read FCreatedAt;
    property SubTotal: TMoney read GetSubTotal;
    property TotalDiscount: TMoney read GetTotalDiscount;
    property TotalTax: TMoney read GetTotalTax;
    property GrandTotal: TMoney read GetGrandTotal;
  end;

implementation

// TOrderLine
constructor TOrderLine.Create(AOrderLineId: Integer;
  AProductId: TProductId; const AProductName: string;
  AQuantity: Integer; AUnitPrice: TMoney; ADiscount: TMoney);
begin
  inherited Create;
  if AQuantity <= 0 then
    raise EDomainException.Create('จำนวนสินค้าต้องมากกว่า 0');
  if AUnitPrice.IsNegative then
    raise EDomainException.Create('ราคาสินค้าต้องไม่ติดลบ');

  FOrderLineId := AOrderLineId;
  FProductId := AProductId;
  FProductName := AProductName;
  FQuantity := AQuantity;
  FUnitPrice := AUnitPrice;
  FDiscount := ADiscount;
  if not Assigned(FDiscount) then
    FDiscount := TMoney.Create(0, FUnitPrice.Currency);
end;

function TOrderLine.GetTotal: TMoney;
var
  Subtotal: TMoney;
begin
  Subtotal := FUnitPrice.Multiply(FQuantity);
  try
    Result := Subtotal.Subtract(FDiscount);
  finally
    Subtotal.Free;
  end;
end;

procedure TOrderLine.UpdateQuantity(ANewQuantity: Integer);
begin
  if ANewQuantity <= 0 then
    raise EDomainException.Create('จำนวนสินค้าต้องมากกว่า 0');
  FQuantity := ANewQuantity;
end;

// TOrder Aggregate Root
constructor TOrder.Create(const ACustomerId: TCustomerId;
  const AShippingAddress: TShippingAddress);
begin
  inherited Create(0); // ID will be assigned by repository
  if not Assigned(ACustomerId) then
    raise EDomainException.Create('CustomerId ต้องไม่เป็น nil');
  if not Assigned(AShippingAddress) then
    raise EDomainException.Create('ShippingAddress ต้องไม่เป็น nil');

  FCustomerId := ACustomerId;
  FShippingAddress := AShippingAddress;
  FStatus := osCreated;
  FOrderNumber := GenerateOrderNumber;
  FOrderLines := TObjectList<TOrderLine>.Create(True);
  FDomainEvents := TList<IDomainEvent>.Create;
  FCreatedAt := Now;
  FNextLineId := 1;

  // Raise Domain Event
  AddDomainEvent(TOrderCreatedEvent.Create(Self));
end;

destructor TOrder.Destroy;
begin
  FDomainEvents.Free;
  FOrderLines.Free;
  FShippingAddress.Free;
  inherited Destroy;
end;

function TOrder.GenerateOrderNumber: string;
begin
  Result := Format('ORD-%s-%d',
    [FormatDateTime('yyyymmdd', Now),
     Random(100000)]);
end;

procedure TOrder.AddOrderLine(const AProductId: TProductId;
  const AProductName: string; AQuantity: Integer;
  AUnitPrice: TMoney; ADiscount: TMoney);
var
  Line: TOrderLine;
begin
  if FStatus <> osCreated then
    raise EDomainException.Create(
      'ไม่สามารถเพิ่มรายการสินค้าได้เนื่องจาก Order ไม่ได้อยู่ในสถานะ Created');

  // Check if product already exists
  if HasOrderLine(AProductId) then
    raise EDomainException.CreateFmt(
      'สินค้า %s มีในรายการแล้ว', [AProductId.Value]);

  Line := TOrderLine.Create(FNextLineId, AProductId, AProductName,
    AQuantity, AUnitPrice, ADiscount);
  Inc(FNextLineId);
  FOrderLines.Add(Line);

  AddDomainEvent(TOrderLineAddedEvent.Create(Id, Line.OrderLineId));
end;

procedure TOrder.RemoveOrderLine(AOrderLineId: Integer);
var
  I: Integer;
begin
  if FStatus <> osCreated then
    raise EDomainException.Create(
      'ไม่สามารถลบรายการสินค้าได้เนื่องจาก Order ไม่ได้อยู่ในสถานะ Created');

  if FOrderLines.Count <= 1 then
    raise EDomainException.Create('Order ต้องมีรายการสินค้าอย่างน้อย 1 รายการ');

  for I := 0 to FOrderLines.Count - 1 do
    if FOrderLines[I].OrderLineId = AOrderLineId then
    begin
      FOrderLines.Delete(I);
      Exit;
    end;

  raise EDomainException.CreateFmt(
    'ไม่พบรายการสินค้า ID: %d', [AOrderLineId]);
end;

procedure TOrder.Confirm;
begin
  ValidateStatusTransition(osConfirmed);
  if FOrderLines.Count = 0 then
    raise EDomainException.Create('ไม่สามารถ Confirm Order ที่ไม่มีสินค้าได้');

  FStatus := osConfirmed;
  FConfirmedAt := Now;
  AddDomainEvent(TOrderConfirmedEvent.Create(Self));
end;

procedure TOrder.Pay(const APaymentReference: string);
begin
  ValidateStatusTransition(osPaid);
  if APaymentReference = '' then
    raise EDomainException.Create('ต้องระบุ Payment Reference');

  FStatus := osPaid;
  FPaidAt := Now;
  AddDomainEvent(TOrderPaidEvent.Create(Id, APaymentReference));
end;

procedure TOrder.Ship(const ATrackingNumber, ACarrier: string);
begin
  ValidateStatusTransition(osShipped);
  if ATrackingNumber = '' then
    raise EDomainException.Create('ต้องระบุ Tracking Number');

  FStatus := osShipped;
  FShippedAt := Now;
  AddDomainEvent(TOrderShippedEvent.Create(Id, ATrackingNumber, ACarrier));
end;

procedure TOrder.Deliver;
begin
  ValidateStatusTransition(osDelivered);
  FStatus := osDelivered;
  FDeliveredAt := Now;
  AddDomainEvent(TOrderDeliveredEvent.Create(Id));
end;

procedure TOrder.Cancel(const AReason: string);
begin
  if not CanBeCancelled then
    raise EDomainException.CreateFmt(
      'ไม่สามารถยกเลิก Order ที่อยู่ในสถานะ %s ได้', [StatusNames[FStatus]]);

  FStatus := osCancelled;
  FCancelledAt := Now;
  FCancelReason := AReason;
  AddDomainEvent(TOrderCancelledEvent.Create(Id, AReason));
end;

function TOrder.CanBeCancelled: Boolean;
begin
  Result := FStatus in [osCreated, osConfirmed];
end;

procedure TOrder.ValidateStatusTransition(ANewStatus: TOrderStatus);
const
  ValidTransitions: array[TOrderStatus] of set of TOrderStatus = (
    {osCreated}    [osConfirmed, osCancelled],
    {osConfirmed}  [osPaid, osCancelled],
    {osPaid}       [osShipped],
    {osShipped}    [osDelivered],
    {osDelivered}  [osRefunded],
    {osCancelled}  [],
    {osRefunded}   []
  );
begin
  if not (ANewStatus in ValidTransitions[FStatus]) then
    raise EDomainException.CreateFmt(
      'ไม่สามารถเปลี่ยนสถานะจาก %s เป็น %s ได้',
      [StatusNames[FStatus], StatusNames[ANewStatus]]);
end;

function TOrder.GetSubTotal: TMoney;
var
  Line: TOrderLine;
  LineTotal: TMoney;
begin
  Result := TMoney.Create(0, 'THB');
  for Line in FOrderLines do
  begin
    LineTotal := Line.Total;
    try
      var NewTotal := Result.Add(LineTotal);
      Result.Free;
      Result := NewTotal;
    finally
      LineTotal.Free;
    end;
  end;
end;

function TOrder.GetGrandTotal: TMoney;
var
  Sub, Tax: TMoney;
begin
  Sub := GetSubTotal;
  Tax := GetTotalTax;
  try
    Result := Sub.Add(Tax);
  finally
    Sub.Free;
    Tax.Free;
  end;
end;

function TOrder.HasOrderLine(const AProductId: TProductId): Boolean;
var
  Line: TOrderLine;
begin
  Result := False;
  for Line in FOrderLines do
    if Line.ProductId.Equals(AProductId) then
    begin
      Result := True;
      Exit;
    end;
end;

procedure TOrder.AddDomainEvent(const AEvent: IDomainEvent);
begin
  FDomainEvents.Add(AEvent);
end;

function TOrder.GetAndClearDomainEvents: TList<IDomainEvent>;
begin
  Result := TList<IDomainEvent>.Create;
  Result.AddRange(FDomainEvents);
  FDomainEvents.Clear;
end;

end.
```

## 4. Domain Events

```pascal
// Domain/Events/uDomainEvents.pas
unit uDomainEvents;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  IDomainEvent = interface
    function GetEventId: string;
    function GetOccurredOn: TDateTime;
    function GetEventType: string;
  end;

  TDomainEvent = class(TInterfacedObject, IDomainEvent)
  private
    FEventId: string;
    FOccurredOn: TDateTime;
    FEventType: string;
  public
    constructor Create(const AEventType: string);
    function GetEventId: string;
    function GetOccurredOn: TDateTime;
    function GetEventType: string;
  end;

  // Order Events
  TOrderCreatedEvent = class(TDomainEvent)
  private
    FOrderId: Integer;
    FOrderNumber: string;
    FCustomerId: Integer;
    FTotal: Double;
  public
    constructor Create(const AOrder: TObject); // TOrder ใช้ TObject เพื่อหลีกเลี่ยง Circular Reference
    property OrderId: Integer read FOrderId;
    property OrderNumber: string read FOrderNumber;
    property CustomerId: Integer read FCustomerId;
    property Total: Double read FTotal;
  end;

  TOrderConfirmedEvent = class(TDomainEvent)
  private
    FOrderId: Integer;
  public
    constructor Create(const AOrder: TObject);
    property OrderId: Integer read FOrderId;
  end;

  TOrderPaidEvent = class(TDomainEvent)
  private
    FOrderId: Integer;
    FPaymentReference: string;
  public
    constructor Create(AOrderId: Integer; const APaymentRef: string);
    property OrderId: Integer read FOrderId;
    property PaymentReference: string read FPaymentReference;
  end;

  TOrderCancelledEvent = class(TDomainEvent)
  private
    FOrderId: Integer;
    FReason: string;
  public
    constructor Create(AOrderId: Integer; const AReason: string);
    property OrderId: Integer read FOrderId;
    property Reason: string read FReason;
  end;

  TOrderShippedEvent = class(TDomainEvent)
  private
    FOrderId: Integer;
    FTrackingNumber: string;
    FCarrier: string;
  public
    constructor Create(AOrderId: Integer;
      const ATrackingNumber, ACarrier: string);
    property OrderId: Integer read FOrderId;
    property TrackingNumber: string read FTrackingNumber;
    property Carrier: string read FCarrier;
  end;

  // Domain Event Handler Interface
  IDomainEventHandler<T: IDomainEvent> = interface
    procedure Handle(const AEvent: T);
  end;

  // Domain Event Dispatcher
  IDomainEventDispatcher = interface
    procedure Dispatch(const AEvent: IDomainEvent);
    procedure Register<T: IDomainEvent>(
      const AHandler: IDomainEventHandler<T>);
  end;

implementation

constructor TDomainEvent.Create(const AEventType: string);
begin
  inherited Create;
  FEventId := TGuid.NewGuid.ToString;
  FOccurredOn := Now;
  FEventType := AEventType;
end;

constructor TOrderCreatedEvent.Create(const AOrder: TObject);
begin
  inherited Create('order.created');
  // Cast AOrder to access Order properties
  // (In real code, use proper typing)
  FOrderId := 0; // Set from AOrder
  FOrderNumber := '';
end;

constructor TOrderPaidEvent.Create(AOrderId: Integer;
  const APaymentRef: string);
begin
  inherited Create('order.paid');
  FOrderId := AOrderId;
  FPaymentReference := APaymentRef;
end;

constructor TOrderCancelledEvent.Create(AOrderId: Integer;
  const AReason: string);
begin
  inherited Create('order.cancelled');
  FOrderId := AOrderId;
  FReason := AReason;
end;

constructor TOrderShippedEvent.Create(AOrderId: Integer;
  const ATrackingNumber, ACarrier: string);
begin
  inherited Create('order.shipped');
  FOrderId := AOrderId;
  FTrackingNumber := ATrackingNumber;
  FCarrier := ACarrier;
end;

end.
```

## 5. Domain Services

Domain Services ประกอบด้วย Business Logic ที่ไม่สังกัดใน Entity เดี่ยว:

```pascal
// Domain/Services/uPricingDomainService.pas
unit uPricingDomainService;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections,
  uCustomer, uProduct, uOrder, uMoney, uDiscount;

type
  TPricingContext = record
    Customer: TCustomer;
    OrderDate: TDateTime;
    IsFirstOrder: Boolean;
    OrderCount: Integer;
    TotalPurchaseAmount: TMoney;
  end;

  TPricingDomainService = class
  private
    FDiscountRepository: IDiscountRepository;

    function CalculateCustomerDiscount(const AContext: TPricingContext;
      const AProduct: TProduct; AQuantity: Integer): TMoney;
    function CalculateVolumeDiscount(const AProduct: TProduct;
      AQuantity: Integer): TMoney;
    function CalculateSeasonalDiscount(const AProduct: TProduct;
      AOrderDate: TDateTime): TMoney;
    function ApplyPromotionCodes(const AProduct: TProduct;
      const APromoCodes: TArray<string>): TMoney;

  public
    constructor Create(const ADiscountRepository: IDiscountRepository);

    function CalculatePrice(const AContext: TPricingContext;
      const AProduct: TProduct; AQuantity: Integer;
      const APromoCodes: TArray<string> = nil): TPriceResult;

    function CalculateOrderTotal(const AOrder: TOrder): TOrderPriceBreakdown;
  end;

  TPriceResult = record
    BasePrice: TMoney;
    CustomerDiscount: TMoney;
    VolumeDiscount: TMoney;
    SeasonalDiscount: TMoney;
    PromoDiscount: TMoney;
    FinalPrice: TMoney;
    DiscountPercentage: Double;
  end;

  TOrderPriceBreakdown = record
    SubTotal: TMoney;
    TotalDiscount: TMoney;
    TaxableAmount: TMoney;
    TaxAmount: TMoney;
    ShippingCost: TMoney;
    GrandTotal: TMoney;
  end;

implementation

function TPricingDomainService.CalculatePrice(
  const AContext: TPricingContext;
  const AProduct: TProduct; AQuantity: Integer;
  const APromoCodes: TArray<string>): TPriceResult;
var
  BaseAmount: TMoney;
  CustomerDisc, VolumeDisc, SeasonalDisc, PromoDisc: TMoney;
  TotalDiscount: TMoney;
begin
  BaseAmount := AProduct.Price.Multiply(AQuantity);

  CustomerDisc := CalculateCustomerDiscount(AContext, AProduct, AQuantity);
  VolumeDisc := CalculateVolumeDiscount(AProduct, AQuantity);
  SeasonalDisc := CalculateSeasonalDiscount(AProduct, AContext.OrderDate);
  PromoDisc := TMoney.Create(0, AProduct.Price.Currency);

  if Assigned(APromoCodes) and (Length(APromoCodes) > 0) then
  begin
    PromoDisc.Free;
    PromoDisc := ApplyPromotionCodes(AProduct, APromoCodes);
  end;

  // ใช้ Discount สูงสุด (ไม่สะสม)
  TotalDiscount := CustomerDisc;
  if VolumeDisc.IsGreaterThan(TotalDiscount) then
  begin
    TotalDiscount.Free;
    TotalDiscount := VolumeDisc;
  end;
  if SeasonalDisc.IsGreaterThan(TotalDiscount) then
  begin
    TotalDiscount.Free;
    TotalDiscount := SeasonalDisc;
  end;

  // เพิ่ม Promo Code Discount
  var FinalDiscount := TotalDiscount.Add(PromoDisc);

  Result.BasePrice := BaseAmount;
  Result.CustomerDiscount := CustomerDisc;
  Result.VolumeDiscount := VolumeDisc;
  Result.SeasonalDiscount := SeasonalDisc;
  Result.PromoDiscount := PromoDisc;

  if FinalDiscount.IsGreaterThan(BaseAmount) then
    Result.FinalPrice := TMoney.Create(0, AProduct.Price.Currency)
  else
    Result.FinalPrice := BaseAmount.Subtract(FinalDiscount);

  if BaseAmount.Amount > 0 then
    Result.DiscountPercentage :=
      (FinalDiscount.Amount / BaseAmount.Amount) * 100
  else
    Result.DiscountPercentage := 0;
end;

function TPricingDomainService.CalculateVolumeDiscount(
  const AProduct: TProduct; AQuantity: Integer): TMoney;
var
  DiscountPct: Double;
  BasePrice: TMoney;
begin
  // Volume Discount Tiers
  if AQuantity >= 100 then DiscountPct := 0.15      // 15% สำหรับ 100+ ชิ้น
  else if AQuantity >= 50 then DiscountPct := 0.10   // 10% สำหรับ 50+ ชิ้น
  else if AQuantity >= 20 then DiscountPct := 0.05   // 5% สำหรับ 20+ ชิ้น
  else if AQuantity >= 10 then DiscountPct := 0.02   // 2% สำหรับ 10+ ชิ้น
  else DiscountPct := 0;

  BasePrice := AProduct.Price.Multiply(AQuantity);
  try
    Result := BasePrice.Multiply(DiscountPct);
  finally
    BasePrice.Free;
  end;
end;

end.
```

## 6. Application Services (Use Cases)

```pascal
// Application/Services/uPlaceOrderUseCase.pas
unit uPlaceOrderUseCase;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections,
  uOrderAggregate, uCustomer, uProduct,
  IRepository, IUnitOfWork,
  uPricingDomainService,
  uPlaceOrderCommand, uOrderDto;

type
  // Command สำหรับ Place Order
  TPlaceOrderItemCommand = record
    ProductId: Integer;
    Quantity: Integer;
    PromoCodes: TArray<string>;
  end;

  TPlaceOrderCommand = class
  public
    CustomerId: Integer;
    Items: TList<TPlaceOrderItemCommand>;
    ShippingAddress: TShippingAddressDto;
    Notes: string;
    RequestedDeliveryDate: TDateTime;

    constructor Create;
    destructor Destroy; override;
  end;

  // Use Case Output
  TPlaceOrderResult = class
  public
    Success: Boolean;
    OrderId: Integer;
    OrderNumber: string;
    GrandTotal: Double;
    EstimatedDeliveryDate: TDateTime;
    ErrorMessage: string;
  end;

  // Use Case
  TPlaceOrderUseCase = class
  private
    FUnitOfWork: IUnitOfWork;
    FPricingService: TPricingDomainService;
    FEventDispatcher: IDomainEventDispatcher;
    FNotificationService: INotificationService;

    function GetCustomerOrThrow(ACustomerId: Integer): TCustomer;
    function GetProductOrThrow(AProductId: Integer): TProduct;
    function BuildShippingAddress(
      const ADto: TShippingAddressDto): TShippingAddress;
    procedure DispatchDomainEvents(const AOrder: TOrder);

  public
    constructor Create(
      const AUnitOfWork: IUnitOfWork;
      const APricingService: TPricingDomainService;
      const AEventDispatcher: IDomainEventDispatcher;
      const ANotificationService: INotificationService);

    function Execute(const ACommand: TPlaceOrderCommand): TPlaceOrderResult;
  end;

  // Domain Event Handlers
  TOrderCreatedHandler = class(TInterfacedObject,
    IDomainEventHandler<TOrderCreatedEvent>)
  private
    FEmailService: IEmailService;
    FInventoryService: IInventoryService;
  public
    procedure Handle(const AEvent: TOrderCreatedEvent);
  end;

  TOrderPaidHandler = class(TInterfacedObject,
    IDomainEventHandler<TOrderPaidEvent>)
  private
    FWarehouseService: IWarehouseService;
  public
    procedure Handle(const AEvent: TOrderPaidEvent);
  end;

implementation

constructor TPlaceOrderCommand.Create;
begin
  inherited Create;
  Items := TList<TPlaceOrderItemCommand>.Create;
end;

destructor TPlaceOrderCommand.Destroy;
begin
  Items.Free;
  inherited Destroy;
end;

function TPlaceOrderUseCase.Execute(
  const ACommand: TPlaceOrderCommand): TPlaceOrderResult;
var
  Customer: TCustomer;
  Product: TProduct;
  ShippingAddress: TShippingAddress;
  Order: TOrder;
  PricingCtx: TPricingContext;
  Item: TPlaceOrderItemCommand;
  PriceResult: TPriceResult;
begin
  Result := TPlaceOrderResult.Create;
  Result.Success := False;

  FUnitOfWork.BeginTransaction;
  try
    // 1. Load Customer (with Domain Validation)
    Customer := GetCustomerOrThrow(ACommand.CustomerId);

    // 2. Verify Customer can place order
    if Customer.Status <> csActive then
    begin
      Result.ErrorMessage := 'ลูกค้าไม่สามารถสั่งซื้อได้ในขณะนี้';
      FUnitOfWork.Rollback;
      Exit;
    end;

    // 3. Build Shipping Address (Value Object)
    ShippingAddress := BuildShippingAddress(ACommand.ShippingAddress);

    // 4. Create Order Aggregate
    Order := TOrder.Create(
      TCustomerId.Create(ACommand.CustomerId),
      ShippingAddress
    );
    try
      // 5. Setup Pricing Context
      PricingCtx.Customer := Customer;
      PricingCtx.OrderDate := Now;
      PricingCtx.IsFirstOrder := Customer.OrderCount = 0;
      PricingCtx.OrderCount := Customer.OrderCount;

      // 6. Add Order Lines with Pricing
      for Item in ACommand.Items do
      begin
        Product := GetProductOrThrow(Item.ProductId);

        // Check Stock Availability
        if not Product.IsInStock(Item.Quantity) then
          raise EDomainException.CreateFmt(
            'สินค้า %s มีสต็อกไม่เพียงพอ', [Product.Name]);

        // Calculate Price
        PriceResult := FPricingService.CalculatePrice(
          PricingCtx, Product, Item.Quantity, Item.PromoCodes);

        // Add Line to Order
        Order.AddOrderLine(
          TProductId.Create(Item.ProductId),
          Product.Name,
          Item.Quantity,
          PriceResult.BasePrice,
          PriceResult.FinalPrice  // Discounted price
        );

        // Reserve Stock
        Product.ReserveStock(Item.Quantity);
        FUnitOfWork.GetProducts.Update(Product);
      end;

      // 7. Set Order Notes
      Order.Notes := ACommand.Notes;

      // 8. Auto-Confirm Order
      Order.Confirm;

      // 9. Save Order
      FUnitOfWork.GetOrders.Add(Order);

      // 10. Update Customer Statistics
      Customer.RecordOrderPlaced;
      FUnitOfWork.GetCustomers.Update(Customer);

      FUnitOfWork.Commit;

      // 11. Dispatch Domain Events (after commit)
      DispatchDomainEvents(Order);

      // 12. Build Result
      Result.Success := True;
      Result.OrderId := Order.Id;
      Result.OrderNumber := Order.OrderNumber;
      Result.GrandTotal := Order.GrandTotal.Amount;
      Result.EstimatedDeliveryDate := Now + 3; // 3 วันทำการ

    finally
      Order.Free;
    end;

  except
    on E: EDomainException do
    begin
      FUnitOfWork.Rollback;
      Result.ErrorMessage := E.Message;
    end;
    on E: Exception do
    begin
      FUnitOfWork.Rollback;
      raise;
    end;
  end;
end;

procedure TPlaceOrderUseCase.DispatchDomainEvents(const AOrder: TOrder);
var
  Events: TList<IDomainEvent>;
  Event: IDomainEvent;
begin
  Events := AOrder.GetAndClearDomainEvents;
  try
    for Event in Events do
      FEventDispatcher.Dispatch(Event);
  finally
    Events.Free;
  end;
end;

// Order Created Handler
procedure TOrderCreatedHandler.Handle(const AEvent: TOrderCreatedEvent);
begin
  // ส่ง Email ยืนยัน
  FEmailService.SendOrderConfirmation(
    AEvent.OrderId,
    AEvent.CustomerId
  );

  // แจ้ง Inventory
  FInventoryService.ProcessNewOrder(AEvent.OrderId);
end;

end.
```

## 7. Repository Implementation

```pascal
// Infrastructure/Repositories/uOrderRepository.pas
unit uOrderRepository;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, sqldb,
  uOrderAggregate, IRepository, uDbContext;

type
  TOrderRepository = class(TInterfacedObject, IQueryableRepository<TOrder>)
  private
    FDbContext: TDbContext;

    function MapRowToOrder(ADataset: TSQLQuery): TOrder;
    procedure LoadOrderLines(AOrder: TOrder);
    procedure SaveOrderLines(AOrder: TOrder);
    procedure DeleteOrderLines(AOrderId: Integer);

  public
    constructor Create(const ADbContext: TDbContext);

    function GetById(const AId: Integer): TOrder;
    function GetAll: TList<TOrder>;
    procedure Add(const AEntity: TOrder);
    procedure Update(const AEntity: TOrder);
    procedure Delete(const AId: Integer);
    function Exists(const AId: Integer): Boolean;

    function Find(const ASpecification: ISpecification<TOrder>): TList<TOrder>;
    function Count(const ASpecification: ISpecification<TOrder>): Integer;
    function FindPaged(const ASpecification: ISpecification<TOrder>;
      const APage, APageSize: Integer): TList<TOrder>;

    // Custom Queries
    function FindByCustomer(ACustomerId: Integer): TList<TOrder>;
    function FindByStatus(AStatus: TOrderStatus): TList<TOrder>;
    function FindByDateRange(AFrom, ATo: TDateTime): TList<TOrder>;
  end;

implementation

constructor TOrderRepository.Create(const ADbContext: TDbContext);
begin
  inherited Create;
  FDbContext := ADbContext;
end;

function TOrderRepository.GetById(const AId: Integer): TOrder;
var
  Query: TSQLQuery;
begin
  Result := nil;
  Query := FDbContext.CreateQuery;
  try
    Query.SQL.Text :=
      'SELECT o.*, c.first_name, c.last_name ' +
      'FROM orders o ' +
      'JOIN customers c ON o.customer_id = c.id ' +
      'WHERE o.id = :id AND o.deleted_at IS NULL';
    Query.ParamByName('id').AsInteger := AId;
    Query.Open;

    if not Query.IsEmpty then
    begin
      Result := MapRowToOrder(Query);
      LoadOrderLines(Result);
    end;
  finally
    Query.Free;
  end;
end;

procedure TOrderRepository.Add(const AEntity: TOrder);
var
  Query: TSQLQuery;
begin
  Query := FDbContext.CreateQuery;
  try
    Query.SQL.Text :=
      'INSERT INTO orders (order_number, customer_id, status, notes, ' +
      '  shipping_street, shipping_city, shipping_province, ' +
      '  shipping_postal_code, shipping_country, created_at) ' +
      'VALUES (:order_number, :customer_id, :status, :notes, ' +
      '  :street, :city, :province, :postal, :country, :created_at) ' +
      'RETURNING id';

    Query.ParamByName('order_number').AsString := AEntity.OrderNumber;
    Query.ParamByName('customer_id').AsInteger := AEntity.CustomerId.Value;
    Query.ParamByName('status').AsInteger := Ord(AEntity.Status);
    Query.ParamByName('notes').AsString := AEntity.Notes;
    Query.ParamByName('street').AsString := AEntity.ShippingAddress.Street;
    Query.ParamByName('city').AsString := AEntity.ShippingAddress.City;
    Query.ParamByName('province').AsString := AEntity.ShippingAddress.Province;
    Query.ParamByName('postal').AsString := AEntity.ShippingAddress.PostalCode;
    Query.ParamByName('country').AsString := AEntity.ShippingAddress.Country;
    Query.ParamByName('created_at').AsDateTime := AEntity.CreatedAt;

    Query.Open;
    if not Query.IsEmpty then
      AEntity.SetId(Query.Fields[0].AsInteger);

    // Save Order Lines
    SaveOrderLines(AEntity);
  finally
    Query.Free;
  end;
end;

procedure TOrderRepository.SaveOrderLines(AOrder: TOrder);
var
  Query: TSQLQuery;
  Line: TOrderLine;
begin
  Query := FDbContext.CreateQuery;
  try
    // Delete existing lines first
    Query.SQL.Text := 'DELETE FROM order_lines WHERE order_id = :order_id';
    Query.ParamByName('order_id').AsInteger := AOrder.Id;
    Query.ExecSQL;

    // Insert new lines
    for Line in AOrder.OrderLines do
    begin
      Query.SQL.Text :=
        'INSERT INTO order_lines (order_id, product_id, product_name, ' +
        '  quantity, unit_price, discount, total, currency) ' +
        'VALUES (:order_id, :product_id, :product_name, ' +
        '  :quantity, :unit_price, :discount, :total, :currency)';

      Query.ParamByName('order_id').AsInteger := AOrder.Id;
      Query.ParamByName('product_id').AsInteger := Line.ProductId.Value;
      Query.ParamByName('product_name').AsString := Line.ProductName;
      Query.ParamByName('quantity').AsInteger := Line.Quantity;
      Query.ParamByName('unit_price').AsCurrency := Line.UnitPrice.Amount;
      Query.ParamByName('discount').AsCurrency := Line.Discount.Amount;
      Query.ParamByName('total').AsCurrency := Line.Total.Amount;
      Query.ParamByName('currency').AsString := Line.UnitPrice.Currency;
      Query.ExecSQL;
    end;
  finally
    Query.Free;
  end;
end;

function TOrderRepository.FindByCustomer(ACustomerId: Integer): TList<TOrder>;
var
  Query: TSQLQuery;
  Order: TOrder;
begin
  Result := TList<TOrder>.Create;
  Query := FDbContext.CreateQuery;
  try
    Query.SQL.Text :=
      'SELECT * FROM orders ' +
      'WHERE customer_id = :customer_id AND deleted_at IS NULL ' +
      'ORDER BY created_at DESC';
    Query.ParamByName('customer_id').AsInteger := ACustomerId;
    Query.Open;

    while not Query.EOF do
    begin
      Order := MapRowToOrder(Query);
      LoadOrderLines(Order);
      Result.Add(Order);
      Query.Next;
    end;
  finally
    Query.Free;
  end;
end;

end.
```

## 8. Complete DDD Example - Running the System

```pascal
// Program.pas - Main Entry Point
program DDDExample;

{$mode objfpc}{$H+}

uses
  SysUtils, Classes,
  uDIContainer, uAppStartup,
  uPlaceOrderUseCase,
  uConfiguration;

var
  Container: TDIContainer;
  UseCase: TPlaceOrderUseCase;
  Command: TPlaceOrderCommand;
  Result: TPlaceOrderResult;

begin
  WriteLn('=== DDD Order System ===');

  // Initialize
  TAppConfiguration.Initialize('appsettings.json');
  Container := TDIContainer.Instance;
  TAppStartup.ConfigureServices(Container);

  // Create Use Case (through DI)
  UseCase := Container.Resolve<TPlaceOrderUseCase>;

  // Create Command
  Command := TPlaceOrderCommand.Create;
  try
    Command.CustomerId := 1001;

    var Item: TPlaceOrderItemCommand;
    Item.ProductId := 2001;
    Item.Quantity := 5;
    Item.PromoCodes := ['SUMMER10'];
    Command.Items.Add(Item);

    Item.ProductId := 2002;
    Item.Quantity := 2;
    Item.PromoCodes := nil;
    Command.Items.Add(Item);

    Command.ShippingAddress.Street := '123 ถนนสุขุมวิท';
    Command.ShippingAddress.City := 'กรุงเทพมหานคร';
    Command.ShippingAddress.Province := 'กรุงเทพมหานคร';
    Command.ShippingAddress.PostalCode := '10110';
    Command.ShippingAddress.Country := 'TH';

    Command.Notes := 'กรุณาจัดส่งในช่วงเช้า';

    // Execute Use Case
    Result := UseCase.Execute(Command);
    try
      if Result.Success then
      begin
        WriteLn('สั่งซื้อสำเร็จ!');
        WriteLn('หมายเลข Order: ', Result.OrderNumber);
        WriteLn('ยอดรวม: ', FormatCurr('฿#,##0.00', Result.GrandTotal));
        WriteLn('วันที่คาดว่าจะได้รับ: ',
          FormatDateTime('dd/mm/yyyy', Result.EstimatedDeliveryDate));
      end
      else
        WriteLn('เกิดข้อผิดพลาด: ', Result.ErrorMessage);
    finally
      Result.Free;
    end;

  finally
    Command.Free;
  end;

  ReadLn;
end.
```

## 9. สรุปหลักการ DDD

| หลักการ | คำอธิบาย |
|---------|----------|
| Bounded Context | แบ่ง Domain ออกเป็นส่วนๆ ที่มีความหมายชัดเจน |
| Entity | Object ที่มี Identity (ID) ไม่เปลี่ยน |
| Value Object | Immutable Object ที่แทนค่า ไม่มี ID |
| Aggregate Root | Entity หลักที่ควบคุม Aggregate ทั้งหมด |
| Domain Event | เหตุการณ์สำคัญที่เกิดขึ้นใน Domain |
| Repository | Interface สำหรับเข้าถึงข้อมูล |
| Domain Service | Logic ที่ไม่สังกัด Entity เดี่ยว |
| Application Service | Orchestrate Use Cases |

**กฎสำคัญของ DDD:**
1. Aggregate เข้าถึงกันผ่าน ID เท่านั้น
2. Transaction ต้องอยู่ใน Aggregate เดียว
3. Domain Events ใช้สำหรับ Cross-Aggregate communication
4. Repository หนึ่งต่อ Aggregate Root หนึ่ง
5. Business Rules อยู่ใน Domain Layer เท่านั้น
