# ตอนที่ 84: CQRS Pattern กับ Pascal/Lazarus

## บทนำ: Command Query Responsibility Segregation

CQRS คือ Pattern ที่แยก Operation ออกเป็น 2 ส่วน:
- **Command** - เปลี่ยนแปลงข้อมูล (Write)
- **Query** - อ่านข้อมูล (Read)

## 1. ทำไมต้องใช้ CQRS?

**ปัญหาของ Traditional CRUD:**
```
┌──────────────────────────────────────┐
│    Single Model for Read & Write     │
│                                      │
│  Read: SELECT * FROM orders          │
│  Write: INSERT/UPDATE/DELETE orders  │
│                                      │
│  ปัญหา: Read/Write มี requirements  │
│  ต่างกัน มักต้อง compromise กัน     │
└──────────────────────────────────────┘
```

**ด้วย CQRS:**
```
┌──────────────────┐      ┌──────────────────┐
│   Command Side   │      │    Query Side     │
│   (Write Model)  │      │   (Read Model)    │
│                  │      │                  │
│  Normalized DB   │      │  Denormalized    │
│  Business Logic  │      │  Optimized Views │
│  Validation      │      │  Fast Queries    │
│  Domain Events   │      │  DTOs            │
└────────┬─────────┘      └──────────────────┘
         │                        ↑
         └────── Domain Events ───┘
                (Sync Read Model)
```

## 2. Command Infrastructure

```pascal
// CQRS/Commands/uCommand.pas
unit uCommand;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  // Base Command
  TCommand = class abstract
  private
    FCommandId: string;
    FTimestamp: TDateTime;
    FUserId: Integer;
    FCorrelationId: string;

  public
    constructor Create;

    property CommandId: string read FCommandId;
    property Timestamp: TDateTime read FTimestamp;
    property UserId: Integer read FUserId write FUserId;
    property CorrelationId: string read FCorrelationId write FCorrelationId;
  end;

  // Command Result
  TCommandResult = class
  private
    FSuccess: Boolean;
    FErrorMessage: string;
    FErrorCode: string;
    FData: TObject;

  public
    constructor CreateSuccess(AData: TObject = nil);
    constructor CreateFailure(const AMessage: string;
      const ACode: string = '');
    destructor Destroy; override;

    property Success: Boolean read FSuccess;
    property ErrorMessage: string read FErrorMessage;
    property ErrorCode: string read FErrorCode;
    property Data: TObject read FData;

    class function Ok(AData: TObject = nil): TCommandResult; static;
    class function Fail(const AMessage, ACode: string = ''): TCommandResult; static;
  end;

  // Command Handler Interface
  ICommandHandler<TCmd: TCommand> = interface
    function Handle(const ACommand: TCmd): TCommandResult;
  end;

  // Command Dispatcher
  ICommandDispatcher = interface
    function Dispatch(const ACommand: TCommand): TCommandResult;
    procedure Register<TCmd: TCommand>(
      const AHandler: ICommandHandler<TCmd>);
  end;

  TCommandDispatcher = class(TInterfacedObject, ICommandDispatcher)
  private
    FHandlers: TDictionary<string, TObject>;
    FMiddlewares: TList<TObject>;

    function GetHandlerKey(const ACommandType: TClass): string;

  public
    constructor Create;
    destructor Destroy; override;

    function Dispatch(const ACommand: TCommand): TCommandResult;
    procedure Register<TCmd: TCommand>(
      const AHandler: ICommandHandler<TCmd>);
    procedure AddMiddleware(const AMiddleware: TCommandMiddleware);
  end;

  // Command Pipeline Middleware
  TCommandMiddleware = class abstract
  public
    function Execute(const ACommand: TCommand;
      const ANext: TFunc<TCommandResult>): TCommandResult; virtual; abstract;
  end;

  // Validation Middleware
  TValidationMiddleware = class(TCommandMiddleware)
  private
    FValidators: TDictionary<string, TObject>;
  public
    function Execute(const ACommand: TCommand;
      const ANext: TFunc<TCommandResult>): TCommandResult; override;
  end;

  // Logging Middleware
  TLoggingMiddleware = class(TCommandMiddleware)
  private
    FLogger: ILogger;
  public
    constructor Create(const ALogger: ILogger);
    function Execute(const ACommand: TCommand;
      const ANext: TFunc<TCommandResult>): TCommandResult; override;
  end;

  // Transaction Middleware
  TTransactionMiddleware = class(TCommandMiddleware)
  private
    FUnitOfWork: IUnitOfWork;
  public
    constructor Create(const AUnitOfWork: IUnitOfWork);
    function Execute(const ACommand: TCommand;
      const ANext: TFunc<TCommandResult>): TCommandResult; override;
  end;

implementation

constructor TCommand.Create;
begin
  inherited Create;
  FCommandId := TGuid.NewGuid.ToString;
  FTimestamp := Now;
end;

constructor TCommandResult.CreateSuccess(AData: TObject);
begin
  inherited Create;
  FSuccess := True;
  FData := AData;
end;

constructor TCommandResult.CreateFailure(const AMessage, ACode: string);
begin
  inherited Create;
  FSuccess := False;
  FErrorMessage := AMessage;
  FErrorCode := ACode;
end;

class function TCommandResult.Ok(AData: TObject): TCommandResult;
begin
  Result := TCommandResult.CreateSuccess(AData);
end;

class function TCommandResult.Fail(const AMessage, ACode: string): TCommandResult;
begin
  Result := TCommandResult.CreateFailure(AMessage, ACode);
end;

// Command Dispatcher
constructor TCommandDispatcher.Create;
begin
  inherited Create;
  FHandlers := TDictionary<string, TObject>.Create;
  FMiddlewares := TList<TObject>.Create;
end;

destructor TCommandDispatcher.Destroy;
begin
  FMiddlewares.Free;
  FHandlers.Free;
  inherited Destroy;
end;

function TCommandDispatcher.Dispatch(const ACommand: TCommand): TCommandResult;
var
  HandlerKey: string;
  Handler: TObject;
  Pipeline: TFunc<TCommandResult>;
  I: Integer;
begin
  HandlerKey := GetHandlerKey(ACommand.ClassType);

  if not FHandlers.TryGetValue(HandlerKey, Handler) then
    raise Exception.CreateFmt(
      'No handler registered for command: %s', [ACommand.ClassName]);

  // Build Pipeline (Middleware Chain)
  Pipeline :=
    function: TCommandResult
    begin
      // Final handler
      Result := nil; // Actual handler call here
    end;

  // Wrap with middlewares (in reverse order)
  for I := FMiddlewares.Count - 1 downto 0 do
  begin
    var Middleware := TCommandMiddleware(FMiddlewares[I]);
    var NextPipeline := Pipeline;
    Pipeline :=
      function: TCommandResult
      begin
        Result := Middleware.Execute(ACommand, NextPipeline);
      end;
  end;

  Result := Pipeline();
end;

// Logging Middleware
function TLoggingMiddleware.Execute(const ACommand: TCommand;
  const ANext: TFunc<TCommandResult>): TCommandResult;
var
  StartTime: TDateTime;
  ElapsedMs: Int64;
begin
  StartTime := Now;
  FLogger.Info(Format('Executing command: %s [%s]',
    [ACommand.ClassName, ACommand.CommandId]));
  try
    Result := ANext();
    ElapsedMs := Round((Now - StartTime) * MSecsPerDay);
    if Result.Success then
      FLogger.Info(Format('Command succeeded: %s (%dms)',
        [ACommand.ClassName, ElapsedMs]))
    else
      FLogger.Warning(Format('Command failed: %s - %s (%dms)',
        [ACommand.ClassName, Result.ErrorMessage, ElapsedMs]));
  except
    on E: Exception do
    begin
      ElapsedMs := Round((Now - StartTime) * MSecsPerDay);
      FLogger.Error(Format('Command exception: %s (%dms)',
        [ACommand.ClassName, ElapsedMs]), E);
      raise;
    end;
  end;
end;

// Transaction Middleware
function TTransactionMiddleware.Execute(const ACommand: TCommand;
  const ANext: TFunc<TCommandResult>): TCommandResult;
begin
  FUnitOfWork.BeginTransaction;
  try
    Result := ANext();
    if Result.Success then
      FUnitOfWork.Commit
    else
      FUnitOfWork.Rollback;
  except
    FUnitOfWork.Rollback;
    raise;
  end;
end;

end.
```

## 3. Specific Commands

```pascal
// CQRS/Commands/Order/uCreateOrderCommand.pas
unit uCreateOrderCommand;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, uCommand;

type
  TOrderItemDto = record
    ProductId: Integer;
    Quantity: Integer;
    PromoCodes: TArray<string>;
  end;

  TShippingAddressDto = record
    Street: string;
    City: string;
    Province: string;
    PostalCode: string;
    Country: string;
  end;

  TCreateOrderCommand = class(TCommand)
  public
    CustomerId: Integer;
    Items: TList<TOrderItemDto>;
    ShippingAddress: TShippingAddressDto;
    Notes: string;
    RequestedDeliveryDate: TDateTime;

    constructor Create;
    destructor Destroy; override;
  end;

  TCreateOrderCommandHandler = class(TInterfacedObject,
    ICommandHandler<TCreateOrderCommand>)
  private
    FOrderRepository: IOrderRepository;
    FCustomerRepository: ICustomerRepository;
    FProductRepository: IProductRepository;
    FPricingService: TPricingDomainService;
    FEventDispatcher: IDomainEventDispatcher;

    procedure ValidateCommand(const ACommand: TCreateOrderCommand);

  public
    constructor Create(
      const AOrderRepo: IOrderRepository;
      const ACustomerRepo: ICustomerRepository;
      const AProductRepo: IProductRepository;
      const APricingService: TPricingDomainService;
      const AEventDispatcher: IDomainEventDispatcher);

    function Handle(const ACommand: TCreateOrderCommand): TCommandResult;
  end;

implementation

constructor TCreateOrderCommand.Create;
begin
  inherited Create;
  Items := TList<TOrderItemDto>.Create;
end;

destructor TCreateOrderCommand.Destroy;
begin
  Items.Free;
  inherited Destroy;
end;

function TCreateOrderCommandHandler.Handle(
  const ACommand: TCreateOrderCommand): TCommandResult;
var
  Customer: TCustomer;
  Order: TOrder;
  Item: TOrderItemDto;
  Product: TProduct;
  PriceResult: TPriceResult;
begin
  try
    ValidateCommand(ACommand);

    Customer := FCustomerRepository.GetById(ACommand.CustomerId);
    if not Assigned(Customer) then
      Exit(TCommandResult.Fail('ไม่พบลูกค้า', 'CUSTOMER_NOT_FOUND'));

    if Customer.Status <> csActive then
      Exit(TCommandResult.Fail('ลูกค้าไม่ Active', 'CUSTOMER_INACTIVE'));

    var ShippingAddr := TShippingAddress.Create(
      ACommand.ShippingAddress.Street,
      ACommand.ShippingAddress.City,
      ACommand.ShippingAddress.Province,
      ACommand.ShippingAddress.PostalCode,
      ACommand.ShippingAddress.Country
    );

    Order := TOrder.Create(TCustomerId.Create(ACommand.CustomerId), ShippingAddr);
    try
      var PricingCtx: TPricingContext;
      PricingCtx.Customer := Customer;
      PricingCtx.OrderDate := Now;

      for Item in ACommand.Items do
      begin
        Product := FProductRepository.GetById(Item.ProductId);
        if not Assigned(Product) then
          raise EDomainException.CreateFmt('ไม่พบสินค้า ID: %d', [Item.ProductId]);

        PriceResult := FPricingService.CalculatePrice(
          PricingCtx, Product, Item.Quantity, Item.PromoCodes);

        Order.AddOrderLine(
          TProductId.Create(Item.ProductId),
          Product.Name,
          Item.Quantity,
          PriceResult.BasePrice,
          PriceResult.PromoDiscount
        );
      end;

      Order.Confirm;
      FOrderRepository.Add(Order);

      // Dispatch Domain Events
      var Events := Order.GetAndClearDomainEvents;
      try
        for var Event in Events do
          FEventDispatcher.Dispatch(Event);
      finally
        Events.Free;
      end;

      var ResultData := TOrderCreatedResultData.Create;
      ResultData.OrderId := Order.Id;
      ResultData.OrderNumber := Order.OrderNumber;
      ResultData.GrandTotal := Order.GrandTotal.Amount;

      Result := TCommandResult.Ok(ResultData);

    except
      Order.Free;
      raise;
    end;

  except
    on E: EDomainException do
      Result := TCommandResult.Fail(E.Message, 'DOMAIN_ERROR');
    on E: Exception do
    begin
      Result := TCommandResult.Fail('เกิดข้อผิดพลาดภายใน', 'INTERNAL_ERROR');
      // Log the exception
    end;
  end;
end;

procedure TCreateOrderCommandHandler.ValidateCommand(
  const ACommand: TCreateOrderCommand);
begin
  if ACommand.CustomerId <= 0 then
    raise EValidationException.Create('CustomerId ไม่ถูกต้อง');
  if ACommand.Items.Count = 0 then
    raise EValidationException.Create('ต้องมีสินค้าอย่างน้อย 1 รายการ');

  for var Item in ACommand.Items do
  begin
    if Item.ProductId <= 0 then
      raise EValidationException.Create('ProductId ไม่ถูกต้อง');
    if Item.Quantity <= 0 then
      raise EValidationException.Create('จำนวนต้องมากกว่า 0');
  end;
end;

end.
```

## 4. Query Infrastructure

```pascal
// CQRS/Queries/uQuery.pas
unit uQuery;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections;

type
  TQuery = class abstract
  private
    FQueryId: string;
    FTimestamp: TDateTime;
    FRequestedBy: Integer;

  public
    constructor Create;

    property QueryId: string read FQueryId;
    property Timestamp: TDateTime read FTimestamp;
    property RequestedBy: Integer read FRequestedBy write FRequestedBy;
  end;

  // Paged Query Support
  TPagedQuery = class(TQuery)
  public
    Page: Integer;
    PageSize: Integer;
    SortBy: string;
    SortDescending: Boolean;

    constructor Create(APage: Integer = 1; APageSize: Integer = 20);
  end;

  TPagedResult<T> = class
  public
    Items: TList<T>;
    TotalCount: Integer;
    Page: Integer;
    PageSize: Integer;
    TotalPages: Integer;

    constructor Create;
    destructor Destroy; override;

    function HasNextPage: Boolean;
    function HasPreviousPage: Boolean;
  end;

  // Query Handler Interface
  IQueryHandler<TQry: TQuery; TResult> = interface
    function Handle(const AQuery: TQry): TResult;
  end;

  // Query Dispatcher
  IQueryDispatcher = interface
    function Dispatch<TResult>(const AQuery: TQuery): TResult;
    procedure Register<TQry: TQuery; TResult>(
      const AHandler: IQueryHandler<TQry, TResult>);
  end;

  // Cached Query Decorator
  TCachedQueryHandler<TQry: TQuery; TResult> = class(
    TInterfacedObject, IQueryHandler<TQry, TResult>)
  private
    FInnerHandler: IQueryHandler<TQry, TResult>;
    FCache: ICache;
    FCacheKeyPrefix: string;
    FCacheDuration: Integer; // seconds

  public
    constructor Create(
      const AInnerHandler: IQueryHandler<TQry, TResult>;
      const ACache: ICache;
      const ACacheKeyPrefix: string;
      ACacheDuration: Integer = 300);

    function Handle(const AQuery: TQry): TResult;
    function BuildCacheKey(const AQuery: TQry): string; virtual;
  end;

implementation

constructor TQuery.Create;
begin
  inherited Create;
  FQueryId := TGuid.NewGuid.ToString;
  FTimestamp := Now;
end;

constructor TPagedQuery.Create(APage, APageSize: Integer);
begin
  inherited Create;
  Page := APage;
  PageSize := APageSize;
  SortDescending := True;
end;

constructor TPagedResult<T>.Create;
begin
  inherited Create;
  Items := TList<T>.Create;
end;

destructor TPagedResult<T>.Destroy;
begin
  Items.Free;
  inherited Destroy;
end;

function TPagedResult<T>.HasNextPage: Boolean;
begin
  Result := Page < TotalPages;
end;

function TPagedResult<T>.HasPreviousPage: Boolean;
begin
  Result := Page > 1;
end;

// Cached Query Handler
function TCachedQueryHandler<TQry, TResult>.Handle(
  const AQuery: TQry): TResult;
var
  CacheKey: string;
  CachedValue: TResult;
begin
  CacheKey := BuildCacheKey(AQuery);

  if FCache.TryGet<TResult>(CacheKey, CachedValue) then
  begin
    Result := CachedValue;
    Exit;
  end;

  Result := FInnerHandler.Handle(AQuery);
  FCache.Set(CacheKey, Result, FCacheDuration);
end;

end.
```

## 5. Specific Queries

```pascal
// CQRS/Queries/Order/uGetOrderQuery.pas
unit uGetOrderQuery;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, uQuery;

type
  TGetOrderQuery = class(TQuery)
  public
    OrderId: Integer;
  end;

  TOrderDetailDto = class
  public
    OrderId: Integer;
    OrderNumber: string;
    Status: string;
    CustomerName: string;
    CustomerEmail: string;

    // Pricing
    SubTotal: Double;
    TotalDiscount: Double;
    TaxAmount: Double;
    ShippingCost: Double;
    GrandTotal: Double;
    Currency: string;

    // Dates
    CreatedAt: TDateTime;
    ConfirmedAt: TDateTime;
    PaidAt: TDateTime;
    ShippedAt: TDateTime;
    DeliveredAt: TDateTime;

    // Shipping
    ShippingAddress: TShippingAddressDto;
    TrackingNumber: string;
    Carrier: string;

    // Items
    Lines: TList<TOrderLineDto>;

    constructor Create;
    destructor Destroy; override;
  end;

  TOrderLineDto = class
  public
    OrderLineId: Integer;
    ProductId: Integer;
    ProductName: string;
    ProductImageUrl: string;
    Quantity: Integer;
    UnitPrice: Double;
    Discount: Double;
    Total: Double;
  end;

  // Handler - Reads from Optimized Read Model
  TGetOrderQueryHandler = class(TInterfacedObject,
    IQueryHandler<TGetOrderQuery, TOrderDetailDto>)
  private
    FDbContext: TReadDbContext; // Separate Read DB Connection

    function MapToDto(ADataset: TSQLQuery): TOrderDetailDto;
    procedure LoadOrderLines(ADto: TOrderDetailDto);

  public
    constructor Create(const ADbContext: TReadDbContext);

    function Handle(const AQuery: TGetOrderQuery): TOrderDetailDto;
  end;

  // List Orders Query
  TListOrdersQuery = class(TPagedQuery)
  public
    CustomerId: Integer;
    Status: string;
    FromDate: TDateTime;
    ToDate: TDateTime;
    SearchTerm: string;
  end;

  TOrderSummaryDto = class
  public
    OrderId: Integer;
    OrderNumber: string;
    Status: string;
    CustomerName: string;
    GrandTotal: Double;
    Currency: string;
    CreatedAt: TDateTime;
    ItemCount: Integer;
  end;

  TListOrdersQueryHandler = class(TInterfacedObject,
    IQueryHandler<TListOrdersQuery, TPagedResult<TOrderSummaryDto>>)
  private
    FDbContext: TReadDbContext;

    function BuildWhereClause(const AQuery: TListOrdersQuery;
      out AParams: TStrings): string;

  public
    constructor Create(const ADbContext: TReadDbContext);

    function Handle(const AQuery: TListOrdersQuery):
      TPagedResult<TOrderSummaryDto>;
  end;

implementation

constructor TOrderDetailDto.Create;
begin
  inherited Create;
  Lines := TList<TOrderLineDto>.Create;
end;

destructor TOrderDetailDto.Destroy;
var
  Line: TOrderLineDto;
begin
  for Line in Lines do Line.Free;
  Lines.Free;
  inherited Destroy;
end;

function TGetOrderQueryHandler.Handle(
  const AQuery: TGetOrderQuery): TOrderDetailDto;
var
  Query: TSQLQuery;
begin
  Result := nil;
  Query := FDbContext.CreateQuery;
  try
    // Read from optimized view (JOIN between orders, customers, etc.)
    Query.SQL.Text :=
      'SELECT o.id, o.order_number, o.status, ' +
      '       o.sub_total, o.total_discount, o.tax_amount, ' +
      '       o.shipping_cost, o.grand_total, o.currency, ' +
      '       o.created_at, o.confirmed_at, o.paid_at, ' +
      '       o.shipped_at, o.delivered_at, ' +
      '       o.shipping_street, o.shipping_city, o.shipping_province, ' +
      '       o.shipping_postal_code, o.shipping_country, ' +
      '       o.tracking_number, o.carrier, ' +
      '       c.first_name || '' '' || c.last_name AS customer_name, ' +
      '       c.email AS customer_email ' +
      'FROM orders o ' +
      'JOIN customers c ON o.customer_id = c.id ' +
      'WHERE o.id = :order_id AND o.deleted_at IS NULL';

    Query.ParamByName('order_id').AsInteger := AQuery.OrderId;
    Query.Open;

    if not Query.IsEmpty then
    begin
      Result := MapToDto(Query);
      LoadOrderLines(Result);
    end;
  finally
    Query.Free;
  end;
end;

procedure TGetOrderQueryHandler.LoadOrderLines(ADto: TOrderDetailDto);
var
  Query: TSQLQuery;
  Line: TOrderLineDto;
begin
  Query := FDbContext.CreateQuery;
  try
    Query.SQL.Text :=
      'SELECT ol.id, ol.product_id, ol.product_name, ' +
      '       ol.quantity, ol.unit_price, ol.discount, ol.total, ' +
      '       p.image_url ' +
      'FROM order_lines ol ' +
      'LEFT JOIN products p ON ol.product_id = p.id ' +
      'WHERE ol.order_id = :order_id ' +
      'ORDER BY ol.id';

    Query.ParamByName('order_id').AsInteger := ADto.OrderId;
    Query.Open;

    while not Query.EOF do
    begin
      Line := TOrderLineDto.Create;
      Line.OrderLineId := Query.FieldByName('id').AsInteger;
      Line.ProductId := Query.FieldByName('product_id').AsInteger;
      Line.ProductName := Query.FieldByName('product_name').AsString;
      Line.ProductImageUrl := Query.FieldByName('image_url').AsString;
      Line.Quantity := Query.FieldByName('quantity').AsInteger;
      Line.UnitPrice := Query.FieldByName('unit_price').AsCurrency;
      Line.Discount := Query.FieldByName('discount').AsCurrency;
      Line.Total := Query.FieldByName('total').AsCurrency;
      ADto.Lines.Add(Line);
      Query.Next;
    end;
  finally
    Query.Free;
  end;
end;

// List Orders with Filtering
function TListOrdersQueryHandler.Handle(
  const AQuery: TListOrdersQuery): TPagedResult<TOrderSummaryDto>;
var
  Params: TStrings;
  WhereClause: string;
  CountQuery, DataQuery: TSQLQuery;
  Order: TOrderSummaryDto;
  Offset: Integer;
begin
  Result := TPagedResult<TOrderSummaryDto>.Create;
  Params := TStringList.Create;
  try
    WhereClause := BuildWhereClause(AQuery, Params);

    // Count total
    CountQuery := FDbContext.CreateQuery;
    try
      CountQuery.SQL.Text :=
        'SELECT COUNT(*) FROM orders o ' +
        'JOIN customers c ON o.customer_id = c.id ' +
        WhereClause;
      // Set params
      CountQuery.Open;
      Result.TotalCount := CountQuery.Fields[0].AsInteger;
    finally
      CountQuery.Free;
    end;

    // Get paged data
    Offset := (AQuery.Page - 1) * AQuery.PageSize;
    DataQuery := FDbContext.CreateQuery;
    try
      DataQuery.SQL.Text :=
        'SELECT o.id, o.order_number, o.status, ' +
        '  c.first_name || '' '' || c.last_name AS customer_name, ' +
        '  o.grand_total, o.currency, o.created_at, ' +
        '  (SELECT COUNT(*) FROM order_lines WHERE order_id = o.id) AS item_count ' +
        'FROM orders o ' +
        'JOIN customers c ON o.customer_id = c.id ' +
        WhereClause +
        Format(' ORDER BY o.created_at %s',
          [IfThen(AQuery.SortDescending, 'DESC', 'ASC')]) +
        Format(' LIMIT %d OFFSET %d', [AQuery.PageSize, Offset]);

      // Set params
      DataQuery.Open;

      while not DataQuery.EOF do
      begin
        Order := TOrderSummaryDto.Create;
        Order.OrderId := DataQuery.FieldByName('id').AsInteger;
        Order.OrderNumber := DataQuery.FieldByName('order_number').AsString;
        Order.Status := DataQuery.FieldByName('status').AsString;
        Order.CustomerName := DataQuery.FieldByName('customer_name').AsString;
        Order.GrandTotal := DataQuery.FieldByName('grand_total').AsCurrency;
        Order.Currency := DataQuery.FieldByName('currency').AsString;
        Order.CreatedAt := DataQuery.FieldByName('created_at').AsDateTime;
        Order.ItemCount := DataQuery.FieldByName('item_count').AsInteger;
        Result.Items.Add(Order);
        DataQuery.Next;
      end;
    finally
      DataQuery.Free;
    end;

    Result.Page := AQuery.Page;
    Result.PageSize := AQuery.PageSize;
    Result.TotalPages := Ceil(Result.TotalCount / AQuery.PageSize);
  finally
    Params.Free;
  end;
end;

end.
```

## 6. Read Model Projection (Syncing Read DB)

```pascal
// CQRS/Projections/uOrderProjection.pas
unit uOrderProjection;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, uDomainEvents, uReadDbContext;

type
  // Projection ที่ Sync Read Model จาก Domain Events
  TOrderProjection = class
  private
    FReadDb: TReadDbContext;

    procedure CreateOrderReadModel(const AEvent: TOrderCreatedEvent);
    procedure UpdateOrderStatus(AOrderId: Integer; const AStatus: string);
    procedure AddPaymentInfo(const AEvent: TOrderPaidEvent);
    procedure AddShippingInfo(const AEvent: TOrderShippedEvent);
    procedure MarkAsDelivered(const AEvent: TOrderDeliveredEvent);
    procedure CancelOrder(const AEvent: TOrderCancelledEvent);

  public
    constructor Create(const AReadDb: TReadDbContext);

    // Handle Domain Events to Update Read Model
    procedure OnOrderCreated(const AEvent: TOrderCreatedEvent);
    procedure OnOrderConfirmed(const AEvent: TOrderConfirmedEvent);
    procedure OnOrderPaid(const AEvent: TOrderPaidEvent);
    procedure OnOrderShipped(const AEvent: TOrderShippedEvent);
    procedure OnOrderDelivered(const AEvent: TOrderDeliveredEvent);
    procedure OnOrderCancelled(const AEvent: TOrderCancelledEvent);
  end;

implementation

constructor TOrderProjection.Create(const AReadDb: TReadDbContext);
begin
  inherited Create;
  FReadDb := AReadDb;
end;

procedure TOrderProjection.OnOrderCreated(const AEvent: TOrderCreatedEvent);
var
  Query: TSQLQuery;
begin
  Query := FReadDb.CreateQuery;
  try
    // Create denormalized read model with all order info
    Query.SQL.Text :=
      'INSERT INTO order_read_models (' +
      '  order_id, order_number, customer_id, customer_name, customer_email, ' +
      '  status, sub_total, grand_total, currency, item_count, ' +
      '  shipping_street, shipping_city, shipping_province, ' +
      '  shipping_postal_code, shipping_country, ' +
      '  created_at, updated_at' +
      ') VALUES (' +
      '  :order_id, :order_number, :customer_id, ' +
      '  (SELECT first_name || '' '' || last_name FROM customers WHERE id = :customer_id), ' +
      '  (SELECT email FROM customers WHERE id = :customer_id), ' +
      '  ''created'', :sub_total, :grand_total, :currency, :item_count, ' +
      '  :street, :city, :province, :postal, :country, ' +
      '  :created_at, :created_at' +
      ')';

    Query.ParamByName('order_id').AsInteger := AEvent.OrderId;
    Query.ParamByName('order_number').AsString := AEvent.OrderNumber;
    Query.ParamByName('customer_id').AsInteger := AEvent.CustomerId;
    Query.ParamByName('sub_total').AsCurrency := AEvent.SubTotal;
    Query.ParamByName('grand_total').AsCurrency := AEvent.GrandTotal;
    Query.ParamByName('currency').AsString := AEvent.Currency;
    Query.ParamByName('item_count').AsInteger := AEvent.ItemCount;
    Query.ParamByName('street').AsString := AEvent.ShippingStreet;
    Query.ParamByName('city').AsString := AEvent.ShippingCity;
    Query.ParamByName('province').AsString := AEvent.ShippingProvince;
    Query.ParamByName('postal').AsString := AEvent.ShippingPostalCode;
    Query.ParamByName('country').AsString := AEvent.ShippingCountry;
    Query.ParamByName('created_at').AsDateTime := AEvent.GetOccurredOn;
    Query.ExecSQL;
  finally
    Query.Free;
  end;
end;

procedure TOrderProjection.OnOrderPaid(const AEvent: TOrderPaidEvent);
var
  Query: TSQLQuery;
begin
  Query := FReadDb.CreateQuery;
  try
    Query.SQL.Text :=
      'UPDATE order_read_models SET ' +
      '  status = ''paid'', ' +
      '  payment_reference = :payment_ref, ' +
      '  paid_at = :paid_at, ' +
      '  updated_at = :updated_at ' +
      'WHERE order_id = :order_id';

    Query.ParamByName('payment_ref').AsString := AEvent.PaymentReference;
    Query.ParamByName('paid_at').AsDateTime := AEvent.GetOccurredOn;
    Query.ParamByName('updated_at').AsDateTime := Now;
    Query.ParamByName('order_id').AsInteger := AEvent.OrderId;
    Query.ExecSQL;
  finally
    Query.Free;
  end;
end;

procedure TOrderProjection.OnOrderShipped(const AEvent: TOrderShippedEvent);
var
  Query: TSQLQuery;
begin
  Query := FReadDb.CreateQuery;
  try
    Query.SQL.Text :=
      'UPDATE order_read_models SET ' +
      '  status = ''shipped'', ' +
      '  tracking_number = :tracking, ' +
      '  carrier = :carrier, ' +
      '  shipped_at = :shipped_at, ' +
      '  updated_at = :updated_at ' +
      'WHERE order_id = :order_id';

    Query.ParamByName('tracking').AsString := AEvent.TrackingNumber;
    Query.ParamByName('carrier').AsString := AEvent.Carrier;
    Query.ParamByName('shipped_at').AsDateTime := AEvent.GetOccurredOn;
    Query.ParamByName('updated_at').AsDateTime := Now;
    Query.ParamByName('order_id').AsInteger := AEvent.OrderId;
    Query.ExecSQL;
  finally
    Query.Free;
  end;
end;

end.
```

## 7. CQRS Controller (REST API)

```pascal
// Presentation/uOrderController.pas
unit uOrderController;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, fphttpserver, httpdefs, fpjson,
  uBaseMicroservice,
  uCreateOrderCommand, uCancelOrderCommand,
  uGetOrderQuery, uListOrdersQuery,
  uCommandDispatcher, uQueryDispatcher;

type
  TOrderController = class
  private
    FCommandDispatcher: ICommandDispatcher;
    FQueryDispatcher: IQueryDispatcher;

  public
    constructor Create(
      const ACmdDispatcher: ICommandDispatcher;
      const AQryDispatcher: IQueryDispatcher);

    // Command Endpoints (Write)
    procedure CreateOrder(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure CancelOrder(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure ShipOrder(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);

    // Query Endpoints (Read)
    procedure GetOrder(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure ListOrders(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
    procedure GetOrderStats(ARequest: TFPHTTPConnectionRequest;
      AResponse: TFPHTTPConnectionResponse);
  end;

implementation

procedure TOrderController.CreateOrder(ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse);
var
  JsonBody: TJSONObject;
  Command: TCreateOrderCommand;
  Result: TCommandResult;
  Response: TJSONObject;
begin
  JsonBody := ParseJsonBody(ARequest);
  if not Assigned(JsonBody) then
  begin
    WriteError(AResponse, 'Invalid JSON', 400);
    Exit;
  end;

  Command := TCreateOrderCommand.Create;
  try
    // Map JSON to Command
    Command.UserId := GetCurrentUserId(ARequest);
    Command.CustomerId := JsonBody.Get('customerId', 0);
    Command.Notes := JsonBody.Get('notes', '');

    // Map Items
    var ItemsArray := TJSONArray(JsonBody.Find('items'));
    if Assigned(ItemsArray) then
    begin
      for var I := 0 to ItemsArray.Count - 1 do
      begin
        var ItemObj := TJSONObject(ItemsArray[I]);
        var Item: TOrderItemDto;
        Item.ProductId := ItemObj.Get('productId', 0);
        Item.Quantity := ItemObj.Get('quantity', 0);
        Command.Items.Add(Item);
      end;
    end;

    // Dispatch Command
    Result := FCommandDispatcher.Dispatch(Command);
    try
      if Result.Success then
      begin
        Response := TJSONObject.Create;
        try
          var Data := TOrderCreatedResultData(Result.Data);
          Response.Add('orderId', Data.OrderId);
          Response.Add('orderNumber', Data.OrderNumber);
          Response.Add('grandTotal', Data.GrandTotal);
          WriteJson(AResponse, Response.AsJSON, 201);
        finally
          Response.Free;
        end;
      end
      else
        WriteError(AResponse, Result.ErrorMessage, 422);
    finally
      Result.Free;
    end;
  finally
    Command.Free;
    JsonBody.Free;
  end;
end;

procedure TOrderController.GetOrder(ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse);
var
  OrderId: Integer;
  Query: TGetOrderQuery;
  Order: TOrderDetailDto;
begin
  OrderId := StrToIntDef(ExtractId(ARequest.PathInfo), 0);
  if OrderId <= 0 then
  begin
    WriteError(AResponse, 'Invalid order ID', 400);
    Exit;
  end;

  Query := TGetOrderQuery.Create;
  try
    Query.OrderId := OrderId;
    Query.RequestedBy := GetCurrentUserId(ARequest);

    Order := FQueryDispatcher.Dispatch<TOrderDetailDto>(Query);
    try
      if Assigned(Order) then
        WriteJson(AResponse, OrderDtoToJson(Order).AsJSON, 200)
      else
        WriteError(AResponse, 'Order not found', 404);
    finally
      Order.Free;
    end;
  finally
    Query.Free;
  end;
end;

procedure TOrderController.ListOrders(ARequest: TFPHTTPConnectionRequest;
  AResponse: TFPHTTPConnectionResponse);
var
  Query: TListOrdersQuery;
  Result: TPagedResult<TOrderSummaryDto>;
  Response: TJSONObject;
  ItemsArray: TJSONArray;
begin
  Query := TListOrdersQuery.Create;
  try
    Query.Page := StrToIntDef(GetQueryParam(ARequest, 'page', '1'), 1);
    Query.PageSize := StrToIntDef(GetQueryParam(ARequest, 'pageSize', '20'), 20);
    Query.Status := GetQueryParam(ARequest, 'status', '');
    Query.SearchTerm := GetQueryParam(ARequest, 'q', '');
    Query.CustomerId := StrToIntDef(GetQueryParam(ARequest, 'customerId', '0'), 0);
    Query.RequestedBy := GetCurrentUserId(ARequest);

    Result := FQueryDispatcher.Dispatch<TPagedResult<TOrderSummaryDto>>(Query);
    try
      Response := TJSONObject.Create;
      try
        ItemsArray := TJSONArray.Create;
        for var Order in Result.Items do
          ItemsArray.Add(OrderSummaryToJson(Order));

        Response.Add('data', ItemsArray);
        Response.Add('page', Result.Page);
        Response.Add('pageSize', Result.PageSize);
        Response.Add('totalCount', Result.TotalCount);
        Response.Add('totalPages', Result.TotalPages);
        Response.Add('hasNextPage', Result.HasNextPage);
        Response.Add('hasPreviousPage', Result.HasPreviousPage);

        WriteJson(AResponse, Response.AsJSON, 200);
      finally
        Response.Free;
      end;
    finally
      Result.Free;
    end;
  finally
    Query.Free;
  end;
end;

end.
```

## 8. สรุป CQRS Pattern

**ข้อดีของ CQRS:**
1. แยก Read/Write ทำให้ Optimize แต่ละส่วนได้
2. Scale Read/Write ได้อิสระ
3. Read Model ซับซ้อนได้โดยไม่กระทบ Write Model
4. รองรับ Event Sourcing ได้ดี

**เมื่อไหรควรใช้ CQRS:**
- ระบบที่มี Read มากกว่า Write มาก
- ต้องการ Custom Read Models
- มี Complex Business Logic
- ต้องการ Scalability สูง

**ข้อควรระวัง:**
- เพิ่มความซับซ้อน
- Eventual Consistency ระหว่าง Write/Read DB
- ต้องจัดการ Projection Failures
