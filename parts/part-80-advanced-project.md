# ตอนที่ 80: โปรเจ็กต์ขั้นสูง - ระบบ POS (Point of Sale)

## บทนำ

บทนี้สร้างระบบ POS (Point of Sale) สมบูรณ์ที่ใช้ SQLite ประกอบด้วยการจัดการสินค้า, การขาย, รายงาน และการจัดการผู้ใช้

---

## 80.1 Database Schema

```sql
-- database/pos_schema.sql

-- ประเภทสินค้า
CREATE TABLE IF NOT EXISTS categories (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    parent_id INTEGER REFERENCES categories(id),
    sort_order INTEGER DEFAULT 0,
    is_active INTEGER DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- สินค้า
CREATE TABLE IF NOT EXISTS products (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    barcode TEXT UNIQUE,
    name TEXT NOT NULL,
    description TEXT,
    category_id INTEGER REFERENCES categories(id),
    unit TEXT DEFAULT 'piece',
    cost_price REAL DEFAULT 0,
    selling_price REAL NOT NULL,
    stock_quantity REAL DEFAULT 0,
    min_stock REAL DEFAULT 0,
    max_stock REAL DEFAULT 0,
    is_active INTEGER DEFAULT 1,
    image_path TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ผู้ใช้ระบบ
CREATE TABLE IF NOT EXISTS users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username TEXT NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    full_name TEXT NOT NULL,
    role TEXT NOT NULL CHECK (role IN ('admin', 'manager', 'cashier')),
    email TEXT,
    phone TEXT,
    is_active INTEGER DEFAULT 1,
    last_login DATETIME,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- ลูกค้า
CREATE TABLE IF NOT EXISTS customers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    code TEXT UNIQUE,
    name TEXT NOT NULL,
    phone TEXT,
    email TEXT,
    address TEXT,
    tax_id TEXT,
    discount_percent REAL DEFAULT 0,
    credit_limit REAL DEFAULT 0,
    total_purchases REAL DEFAULT 0,
    points INTEGER DEFAULT 0,
    is_active INTEGER DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- การขาย (หัวบิล)
CREATE TABLE IF NOT EXISTS sales (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    bill_number TEXT NOT NULL UNIQUE,
    sale_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    customer_id INTEGER REFERENCES customers(id),
    user_id INTEGER REFERENCES users(id),
    subtotal REAL DEFAULT 0,
    discount_amount REAL DEFAULT 0,
    tax_amount REAL DEFAULT 0,
    total_amount REAL DEFAULT 0,
    payment_method TEXT DEFAULT 'cash',
    amount_paid REAL DEFAULT 0,
    change_amount REAL DEFAULT 0,
    notes TEXT,
    status TEXT DEFAULT 'completed' CHECK (status IN ('pending', 'completed', 'cancelled')),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- รายการขาย (รายการย่อย)
CREATE TABLE IF NOT EXISTS sale_items (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sale_id INTEGER NOT NULL REFERENCES sales(id),
    product_id INTEGER NOT NULL REFERENCES products(id),
    quantity REAL NOT NULL,
    unit_price REAL NOT NULL,
    discount_percent REAL DEFAULT 0,
    discount_amount REAL DEFAULT 0,
    total_price REAL NOT NULL,
    cost_price REAL DEFAULT 0
);

-- การปรับสต็อก
CREATE TABLE IF NOT EXISTS stock_adjustments (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    product_id INTEGER NOT NULL REFERENCES products(id),
    adjustment_type TEXT NOT NULL,
    quantity REAL NOT NULL,
    reason TEXT,
    reference TEXT,
    user_id INTEGER REFERENCES users(id),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- Indexes
CREATE INDEX IF NOT EXISTS idx_products_barcode ON products(barcode);
CREATE INDEX IF NOT EXISTS idx_products_category ON products(category_id);
CREATE INDEX IF NOT EXISTS idx_sales_date ON sales(sale_date);
CREATE INDEX IF NOT EXISTS idx_sales_customer ON sales(customer_id);
CREATE INDEX IF NOT EXISTS idx_sale_items_sale ON sale_items(sale_id);
```

---

## 80.2 Database Layer

```pascal
unit pos_database;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SQLite3, SQLite3Conn, SQLDB, DB;

type
  TPOSDatabase = class
  private
    FConnection: TSQLite3Connection;
    FTransaction: TSQLTransaction;
    FQuery: TSQLQuery;
    FDBPath: string;
    
    class var FInstance: TPOSDatabase;
    
    procedure Initialize;
    procedure CreateSchema;
    procedure InsertSampleData;
    function GetLastInsertID: Int64;
    
  public
    constructor Create(const ADBPath: string);
    destructor Destroy; override;
    
    class function Instance: TPOSDatabase;
    class procedure SetDBPath(const APath: string);
    
    procedure BeginTransaction;
    procedure CommitTransaction;
    procedure RollbackTransaction;
    
    function ExecuteQuery(const ASQL: string): TSQLQuery;
    function ExecuteNonQuery(const ASQL: string): Integer;
    function ExecuteScalar(const ASQL: string): Variant;
    
    function QueryTable(const ATable: string; 
      const AWhere: string = '';
      const AOrderBy: string = '';
      ALimit: Integer = 0): TSQLQuery;
      
    function InsertRecord(const ATable: string; AFields: TStringList): Int64;
    function UpdateRecord(const ATable: string; AFields: TStringList; 
      const AWhere: string): Integer;
    function DeleteRecord(const ATable: string; const AWhere: string): Integer;
    
    property Connection: TSQLite3Connection read FConnection;
    property DBPath: string read FDBPath;
  end;

implementation

class var
  FDefaultDBPath: string = 'pos.db';

class function TPOSDatabase.Instance: TPOSDatabase;
begin
  if FInstance = nil then
    FInstance := TPOSDatabase.Create(FDefaultDBPath);
  Result := FInstance;
end;

class procedure TPOSDatabase.SetDBPath(const APath: string);
begin
  FDefaultDBPath := APath;
end;

constructor TPOSDatabase.Create(const ADBPath: string);
begin
  inherited Create;
  FDBPath := ADBPath;
  
  FConnection := TSQLite3Connection.Create(nil);
  FConnection.DatabaseName := FDBPath;
  FConnection.Open;
  
  FTransaction := TSQLTransaction.Create(nil);
  FTransaction.DataBase := FConnection;
  
  FQuery := TSQLQuery.Create(nil);
  FQuery.DataBase := FConnection;
  FQuery.Transaction := FTransaction;
  
  Initialize;
end;

destructor TPOSDatabase.Destroy;
begin
  FQuery.Free;
  FTransaction.Free;
  FConnection.Free;
  inherited Destroy;
end;

procedure TPOSDatabase.Initialize;
begin
  // Enable WAL mode for better performance
  ExecuteNonQuery('PRAGMA journal_mode=WAL');
  ExecuteNonQuery('PRAGMA foreign_keys=ON');
  ExecuteNonQuery('PRAGMA synchronous=NORMAL');
  
  CreateSchema;
end;

procedure TPOSDatabase.CreateSchema;
var
  SchemaFile: string;
  Schema: TStringList;
begin
  SchemaFile := ExtractFilePath(ParamStr(0)) + 'database/pos_schema.sql';
  
  if FileExists(SchemaFile) then
  begin
    Schema := TStringList.Create;
    try
      Schema.LoadFromFile(SchemaFile);
      
      // Execute each statement
      var SQL := Schema.Text;
      var Statements := SQL.Split([';']);
      
      for var Stmt in Statements do
      begin
        var Trimmed := Trim(Stmt);
        if Trimmed <> '' then
          ExecuteNonQuery(Trimmed);
      end;
      
    finally
      Schema.Free;
    end;
  end
  else
    // Inline schema
    ExecuteNonQuery('CREATE TABLE IF NOT EXISTS categories (id INTEGER PRIMARY KEY, name TEXT NOT NULL)');
    
  WriteLn('Database schema initialized');
end;

procedure TPOSDatabase.BeginTransaction;
begin
  FTransaction.StartTransaction;
end;

procedure TPOSDatabase.CommitTransaction;
begin
  FTransaction.Commit;
end;

procedure TPOSDatabase.RollbackTransaction;
begin
  FTransaction.Rollback;
end;

function TPOSDatabase.ExecuteQuery(const ASQL: string): TSQLQuery;
var
  Q: TSQLQuery;
begin
  Q := TSQLQuery.Create(nil);
  Q.DataBase := FConnection;
  Q.Transaction := FTransaction;
  Q.SQL.Text := ASQL;
  
  if not FTransaction.Active then
    FTransaction.StartTransaction;
    
  Q.Open;
  Result := Q;
end;

function TPOSDatabase.ExecuteNonQuery(const ASQL: string): Integer;
begin
  Result := 0;
  try
    if not FTransaction.Active then
      FTransaction.StartTransaction;
      
    FConnection.ExecuteDirect(ASQL);
    FTransaction.CommitRetaining;
    Result := 1;
  except
    on E: Exception do
    begin
      WriteLn('SQL Error: ', E.Message);
      WriteLn('SQL: ', ASQL);
      FTransaction.RollbackRetaining;
    end;
  end;
end;

function TPOSDatabase.ExecuteScalar(const ASQL: string): Variant;
var
  Q: TSQLQuery;
begin
  Result := Null;
  Q := ExecuteQuery(ASQL);
  try
    if not Q.IsEmpty then
      Result := Q.Fields[0].Value;
  finally
    Q.Free;
  end;
end;

function TPOSDatabase.InsertRecord(const ATable: string; AFields: TStringList): Int64;
var
  Cols, Vals: string;
  i: Integer;
begin
  Result := -1;
  Cols := '';
  Vals := '';
  
  for i := 0 to AFields.Count - 1 do
  begin
    if i > 0 then
    begin
      Cols := Cols + ',';
      Vals := Vals + ',';
    end;
    Cols := Cols + AFields.Names[i];
    Vals := Vals + '''' + StringReplace(AFields.ValueFromIndex[i], '''', '''''', [rfReplaceAll]) + '''';
  end;
  
  var SQL := Format('INSERT INTO %s (%s) VALUES (%s)', [ATable, Cols, Vals]);
  
  if ExecuteNonQuery(SQL) > 0 then
    Result := Int64(ExecuteScalar('SELECT last_insert_rowid()'));
end;

function TPOSDatabase.UpdateRecord(const ATable: string; AFields: TStringList;
  const AWhere: string): Integer;
var
  SetClause: string;
  i: Integer;
begin
  SetClause := '';
  for i := 0 to AFields.Count - 1 do
  begin
    if i > 0 then SetClause := SetClause + ',';
    SetClause := SetClause + AFields.Names[i] + '=''' +
      StringReplace(AFields.ValueFromIndex[i], '''', '''''', [rfReplaceAll]) + '''';
  end;
  
  var SQL := Format('UPDATE %s SET %s WHERE %s', [ATable, SetClause, AWhere]);
  Result := ExecuteNonQuery(SQL);
end;

function TPOSDatabase.DeleteRecord(const ATable: string; const AWhere: string): Integer;
begin
  Result := ExecuteNonQuery(Format('DELETE FROM %s WHERE %s', [ATable, AWhere]));
end;

end.
```

---

## 80.3 Business Logic Layer

```pascal
unit pos_services;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, DB, DateUtils,
  pos_database;

type
  TProduct = record
    ID: Integer;
    Barcode: string;
    Name: string;
    CategoryID: Integer;
    Unit: string;
    CostPrice: Double;
    SellingPrice: Double;
    StockQty: Double;
    IsActive: Boolean;
  end;

  TSaleItem = record
    ProductID: Integer;
    ProductName: string;
    Quantity: Double;
    UnitPrice: Double;
    DiscountPercent: Double;
    TotalPrice: Double;
    CostPrice: Double;
  end;

  TSale = record
    ID: Integer;
    BillNumber: string;
    CustomerID: Integer;
    Items: array of TSaleItem;
    ItemCount: Integer;
    Subtotal: Double;
    DiscountAmount: Double;
    TaxAmount: Double;
    TotalAmount: Double;
    PaymentMethod: string;
    AmountPaid: Double;
    ChangeAmount: Double;
  end;

  TProductService = class
  private
    FDB: TPOSDatabase;
    
    function BuildProduct(Q: TDataset): TProduct;
    
  public
    constructor Create;
    
    function FindByBarcode(const ABarcode: string; out AProduct: TProduct): Boolean;
    function FindByID(AID: Integer; out AProduct: TProduct): Boolean;
    function SearchProducts(const AKeyword: string): TList;
    function GetAllActive: TList;
    
    function AddProduct(const AProduct: TProduct): Integer;
    function UpdateProduct(const AProduct: TProduct): Boolean;
    function UpdateStock(AProductID: Integer; ADelta: Double; 
      const AReason: string): Boolean;
    
    function GetLowStockProducts: TList;
    function GetStockValue: Double;
  end;

  TSaleService = class
  private
    FDB: TPOSDatabase;
    FCurrentSale: TSale;
    
    function GenerateBillNumber: string;
    procedure AddSaleItem(const AItem: TSaleItem);
    procedure RecalculateTotals;
    
  public
    constructor Create;
    
    // ขั้นตอนการขาย
    procedure StartNewSale(ACustomerID: Integer = 0);
    function AddItem(AProductID: Integer; AQty: Double; 
      ADiscount: Double = 0): Boolean;
    function RemoveItem(AIndex: Integer): Boolean;
    procedure UpdateItemQty(AIndex: Integer; AQty: Double);
    procedure SetDiscount(AAmount: Double);
    procedure SetTaxRate(ARate: Double);
    
    // ชำระเงิน
    function CompleteSale(const APaymentMethod: string; 
      AAmountPaid: Double): Boolean;
    procedure CancelSale;
    
    // ดูข้อมูล
    function GetCurrentSale: TSale;
    function GetSaleByBill(const ABillNumber: string): TSale;
    function GetSalesByDate(ADate: TDateTime): TList;
    function GetSalesByDateRange(AFrom, ATo: TDateTime): TList;
    
    // รายงาน
    function GetDailySummary(ADate: TDateTime): string;
    function GetTopProducts(ATopN: Integer): TList;
  end;

  TUserService = class
  private
    FDB: TPOSDatabase;
    FCurrentUser: Integer;
    
    function HashPassword(const APassword: string): string;
    
  public
    constructor Create;
    
    function Login(const AUsername, APassword: string): Boolean;
    procedure Logout;
    
    function IsLoggedIn: Boolean;
    function GetCurrentUserID: Integer;
    function HasPermission(const APermission: string): Boolean;
    
    function AddUser(const AUsername, APassword, AFullName, ARole: string): Integer;
    function ChangePassword(AUserID: Integer; const ANewPassword: string): Boolean;
  end;

implementation

uses
  MD5;  // For password hashing

{ TProductService }

constructor TProductService.Create;
begin
  inherited Create;
  FDB := TPOSDatabase.Instance;
end;

function TProductService.BuildProduct(Q: TDataset): TProduct;
begin
  Result.ID := Q.FieldByName('id').AsInteger;
  Result.Barcode := Q.FieldByName('barcode').AsString;
  Result.Name := Q.FieldByName('name').AsString;
  Result.CategoryID := Q.FieldByName('category_id').AsInteger;
  Result.Unit := Q.FieldByName('unit').AsString;
  Result.CostPrice := Q.FieldByName('cost_price').AsFloat;
  Result.SellingPrice := Q.FieldByName('selling_price').AsFloat;
  Result.StockQty := Q.FieldByName('stock_quantity').AsFloat;
  Result.IsActive := Q.FieldByName('is_active').AsInteger = 1;
end;

function TProductService.FindByBarcode(const ABarcode: string; 
  out AProduct: TProduct): Boolean;
var
  Q: TSQLQuery;
begin
  Result := False;
  Q := FDB.ExecuteQuery(
    Format('SELECT * FROM products WHERE barcode=''%s'' AND is_active=1', 
      [StringReplace(ABarcode, '''', '''''', [rfReplaceAll])]));
  try
    if not Q.IsEmpty then
    begin
      AProduct := BuildProduct(Q);
      Result := True;
    end;
  finally
    Q.Free;
  end;
end;

function TProductService.FindByID(AID: Integer; out AProduct: TProduct): Boolean;
var
  Q: TSQLQuery;
begin
  Result := False;
  Q := FDB.ExecuteQuery(
    Format('SELECT * FROM products WHERE id=%d', [AID]));
  try
    if not Q.IsEmpty then
    begin
      AProduct := BuildProduct(Q);
      Result := True;
    end;
  finally
    Q.Free;
  end;
end;

function TProductService.SearchProducts(const AKeyword: string): TList;
var
  Q: TSQLQuery;
  P: ^TProduct;
begin
  Result := TList.Create;
  Q := FDB.ExecuteQuery(
    Format('SELECT * FROM products WHERE is_active=1 AND (name LIKE ''%%%s%%'' OR barcode LIKE ''%%%s%%'') ORDER BY name LIMIT 50',
      [AKeyword, AKeyword]));
  try
    while not Q.EOF do
    begin
      New(P);
      P^ := BuildProduct(Q);
      Result.Add(P);
      Q.Next;
    end;
  finally
    Q.Free;
  end;
end;

function TProductService.UpdateStock(AProductID: Integer; ADelta: Double;
  const AReason: string): Boolean;
var
  Fields: TStringList;
begin
  Result := False;
  
  // Update product stock
  if FDB.ExecuteNonQuery(
    Format('UPDATE products SET stock_quantity = stock_quantity + %f WHERE id=%d',
      [ADelta, AProductID])) > 0 then
  begin
    // Log adjustment
    Fields := TStringList.Create;
    try
      Fields.Values['product_id'] := IntToStr(AProductID);
      Fields.Values['adjustment_type'] := IfThen(ADelta > 0, 'in', 'out');
      Fields.Values['quantity'] := FloatToStr(Abs(ADelta));
      Fields.Values['reason'] := AReason;
      
      FDB.InsertRecord('stock_adjustments', Fields);
    finally
      Fields.Free;
    end;
    
    Result := True;
  end;
end;

{ TSaleService }

constructor TSaleService.Create;
begin
  inherited Create;
  FDB := TPOSDatabase.Instance;
  FillChar(FCurrentSale, SizeOf(FCurrentSale), 0);
end;

function TSaleService.GenerateBillNumber: string;
begin
  Result := 'B' + FormatDateTime('yyyymmddhhnnss', Now);
end;

procedure TSaleService.StartNewSale(ACustomerID: Integer);
begin
  FillChar(FCurrentSale, SizeOf(FCurrentSale), 0);
  FCurrentSale.BillNumber := GenerateBillNumber;
  FCurrentSale.CustomerID := ACustomerID;
  FCurrentSale.PaymentMethod := 'cash';
  SetLength(FCurrentSale.Items, 0);
  FCurrentSale.ItemCount := 0;
  
  WriteLn('New sale started: ', FCurrentSale.BillNumber);
end;

function TSaleService.AddItem(AProductID: Integer; AQty: Double; 
  ADiscount: Double): Boolean;
var
  ProductSvc: TProductService;
  Product: TProduct;
  Item: TSaleItem;
  i: Integer;
begin
  Result := False;
  
  ProductSvc := TProductService.Create;
  try
    if not ProductSvc.FindByID(AProductID, Product) then
    begin
      WriteLn('Product not found: ', AProductID);
      Exit;
    end;
    
    if Product.StockQty < AQty then
    begin
      WriteLn('Insufficient stock. Available: ', Product.StockQty);
      Exit;
    end;
    
    // Check if product already in sale
    for i := 0 to FCurrentSale.ItemCount - 1 do
      if FCurrentSale.Items[i].ProductID = AProductID then
      begin
        FCurrentSale.Items[i].Quantity := FCurrentSale.Items[i].Quantity + AQty;
        FCurrentSale.Items[i].TotalPrice := 
          FCurrentSale.Items[i].Quantity * FCurrentSale.Items[i].UnitPrice *
          (1 - FCurrentSale.Items[i].DiscountPercent / 100);
        RecalculateTotals;
        Result := True;
        Exit;
      end;
    
    // New item
    Item.ProductID := AProductID;
    Item.ProductName := Product.Name;
    Item.Quantity := AQty;
    Item.UnitPrice := Product.SellingPrice;
    Item.DiscountPercent := ADiscount;
    Item.TotalPrice := AQty * Product.SellingPrice * (1 - ADiscount / 100);
    Item.CostPrice := Product.CostPrice;
    
    Inc(FCurrentSale.ItemCount);
    SetLength(FCurrentSale.Items, FCurrentSale.ItemCount);
    FCurrentSale.Items[FCurrentSale.ItemCount - 1] := Item;
    
    RecalculateTotals;
    Result := True;
    
  finally
    ProductSvc.Free;
  end;
end;

procedure TSaleService.RecalculateTotals;
var
  i: Integer;
  Subtotal: Double;
begin
  Subtotal := 0;
  
  for i := 0 to FCurrentSale.ItemCount - 1 do
    Subtotal := Subtotal + FCurrentSale.Items[i].TotalPrice;
    
  FCurrentSale.Subtotal := Subtotal;
  FCurrentSale.TotalAmount := Subtotal - FCurrentSale.DiscountAmount + FCurrentSale.TaxAmount;
end;

function TSaleService.CompleteSale(const APaymentMethod: string; 
  AAmountPaid: Double): Boolean;
var
  Fields: TStringList;
  SaleID: Int64;
  i: Integer;
  ItemFields: TStringList;
  ProductSvc: TProductService;
begin
  Result := False;
  
  if FCurrentSale.ItemCount = 0 then
  begin
    WriteLn('No items in sale');
    Exit;
  end;
  
  FCurrentSale.PaymentMethod := APaymentMethod;
  FCurrentSale.AmountPaid := AAmountPaid;
  FCurrentSale.ChangeAmount := AAmountPaid - FCurrentSale.TotalAmount;
  
  FDB.BeginTransaction;
  try
    // Insert sale header
    Fields := TStringList.Create;
    try
      Fields.Values['bill_number'] := FCurrentSale.BillNumber;
      Fields.Values['customer_id'] := IntToStr(FCurrentSale.CustomerID);
      Fields.Values['subtotal'] := Format('%.2f', [FCurrentSale.Subtotal]);
      Fields.Values['discount_amount'] := Format('%.2f', [FCurrentSale.DiscountAmount]);
      Fields.Values['tax_amount'] := Format('%.2f', [FCurrentSale.TaxAmount]);
      Fields.Values['total_amount'] := Format('%.2f', [FCurrentSale.TotalAmount]);
      Fields.Values['payment_method'] := APaymentMethod;
      Fields.Values['amount_paid'] := Format('%.2f', [AAmountPaid]);
      Fields.Values['change_amount'] := Format('%.2f', [FCurrentSale.ChangeAmount]);
      Fields.Values['status'] := 'completed';
      
      SaleID := FDB.InsertRecord('sales', Fields);
    finally
      Fields.Free;
    end;
    
    if SaleID < 0 then
    begin
      FDB.RollbackTransaction;
      Exit;
    end;
    
    // Insert items and update stock
    ProductSvc := TProductService.Create;
    try
      for i := 0 to FCurrentSale.ItemCount - 1 do
      begin
        ItemFields := TStringList.Create;
        try
          ItemFields.Values['sale_id'] := IntToStr(SaleID);
          ItemFields.Values['product_id'] := IntToStr(FCurrentSale.Items[i].ProductID);
          ItemFields.Values['quantity'] := Format('%.4f', [FCurrentSale.Items[i].Quantity]);
          ItemFields.Values['unit_price'] := Format('%.2f', [FCurrentSale.Items[i].UnitPrice]);
          ItemFields.Values['discount_percent'] := Format('%.2f', [FCurrentSale.Items[i].DiscountPercent]);
          ItemFields.Values['total_price'] := Format('%.2f', [FCurrentSale.Items[i].TotalPrice]);
          ItemFields.Values['cost_price'] := Format('%.2f', [FCurrentSale.Items[i].CostPrice]);
          
          FDB.InsertRecord('sale_items', ItemFields);
        finally
          ItemFields.Free;
        end;
        
        // Deduct stock
        ProductSvc.UpdateStock(FCurrentSale.Items[i].ProductID,
          -FCurrentSale.Items[i].Quantity, 'sale:' + FCurrentSale.BillNumber);
      end;
    finally
      ProductSvc.Free;
    end;
    
    FDB.CommitTransaction;
    FCurrentSale.ID := SaleID;
    
    WriteLn(Format('Sale completed: %s, Total: %.2f, Change: %.2f',
      [FCurrentSale.BillNumber, FCurrentSale.TotalAmount, FCurrentSale.ChangeAmount]));
      
    Result := True;
    
  except
    on E: Exception do
    begin
      FDB.RollbackTransaction;
      WriteLn('Sale failed: ', E.Message);
    end;
  end;
end;

function TSaleService.GetDailySummary(ADate: TDateTime): string;
var
  DateStr: string;
  Q: TSQLQuery;
begin
  DateStr := FormatDateTime('yyyy-mm-dd', ADate);
  
  Q := FDB.ExecuteQuery(Format(
    'SELECT COUNT(*) as bill_count, SUM(total_amount) as total_sales, ' +
    'SUM(discount_amount) as total_discount ' +
    'FROM sales WHERE date(sale_date)=''%s'' AND status=''completed''',
    [DateStr]));
  try
    if not Q.IsEmpty then
      Result := Format(
        'วันที่: %s | บิล: %d | ยอดขาย: %.2f | ส่วนลด: %.2f',
        [DateStr,
         Q.FieldByName('bill_count').AsInteger,
         Q.FieldByName('total_sales').AsFloat,
         Q.FieldByName('total_discount').AsFloat])
    else
      Result := 'ไม่มีข้อมูล';
  finally
    Q.Free;
  end;
end;

{ TUserService }

constructor TUserService.Create;
begin
  inherited Create;
  FDB := TPOSDatabase.Instance;
  FCurrentUser := 0;
end;

function TUserService.HashPassword(const APassword: string): string;
begin
  Result := MD5Print(MD5String(APassword + 'pos_salt_2024'));
end;

function TUserService.Login(const AUsername, APassword: string): Boolean;
var
  Q: TSQLQuery;
  Hash: string;
begin
  Result := False;
  Hash := HashPassword(APassword);
  
  Q := FDB.ExecuteQuery(
    Format('SELECT id FROM users WHERE username=''%s'' AND password_hash=''%s'' AND is_active=1',
      [StringReplace(AUsername, '''', '''''', [rfReplaceAll]), Hash]));
  try
    if not Q.IsEmpty then
    begin
      FCurrentUser := Q.FieldByName('id').AsInteger;
      
      // Update last login
      FDB.ExecuteNonQuery(Format(
        'UPDATE users SET last_login=datetime(''now'') WHERE id=%d',
        [FCurrentUser]));
        
      WriteLn('Login success: ', AUsername);
      Result := True;
    end
    else
    begin
      WriteLn('Login failed: ', AUsername);
      FCurrentUser := 0;
    end;
  finally
    Q.Free;
  end;
end;

procedure TUserService.Logout;
begin
  WriteLn('User logged out: ', FCurrentUser);
  FCurrentUser := 0;
end;

function TUserService.IsLoggedIn: Boolean;
begin
  Result := FCurrentUser > 0;
end;

function TUserService.GetCurrentUserID: Integer;
begin
  Result := FCurrentUser;
end;

function TUserService.HasPermission(const APermission: string): Boolean;
var
  Q: TSQLQuery;
  Role: string;
begin
  Result := False;
  if not IsLoggedIn then Exit;
  
  Q := FDB.ExecuteQuery(Format(
    'SELECT role FROM users WHERE id=%d', [FCurrentUser]));
  try
    if not Q.IsEmpty then
    begin
      Role := Q.FieldByName('role').AsString;
      
      // Admin can do everything
      if Role = 'admin' then Result := True
      else if Role = 'manager' then
        Result := APermission in ['sale', 'report', 'product', 'customer', 'discount']
      else if Role = 'cashier' then
        Result := APermission in ['sale', 'product_view']
      else
        Result := False;
    end;
  finally
    Q.Free;
  end;
end;

function TUserService.AddUser(const AUsername, APassword, AFullName, ARole: string): Integer;
var
  Fields: TStringList;
begin
  Fields := TStringList.Create;
  try
    Fields.Values['username'] := AUsername;
    Fields.Values['password_hash'] := HashPassword(APassword);
    Fields.Values['full_name'] := AFullName;
    Fields.Values['role'] := ARole;
    
    Result := FDB.InsertRecord('users', Fields);
  finally
    Fields.Free;
  end;
end;

end.
```

---

## 80.4 Main POS Form

```pascal
unit pos_main_form;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, StdCtrls, ExtCtrls,
  ComCtrls, Grids, Graphics, Dialogs,
  pos_database, pos_services;

type
  TPOSMainForm = class(TForm)
  private
    // Services
    FProductSvc: TProductService;
    FSaleSvc: TSaleService;
    FUserSvc: TUserService;
    
    // UI Components
    pnlTop: TPanel;
    pnlLeft: TPanel;
    pnlRight: TPanel;
    pnlBottom: TPanel;
    
    // Barcode input
    lblBarcode: TLabel;
    edtBarcode: TEdit;
    
    // Cart
    CartGrid: TStringGrid;
    lblTotal: TLabel;
    lblSubtotal: TLabel;
    lblDiscount: TLabel;
    lblTax: TLabel;
    
    // Payment
    edtPaid: TEdit;
    lblChange: TLabel;
    cmbPayment: TComboBox;
    
    // Buttons
    btnAddItem: TButton;
    btnRemoveItem: TButton;
    btnDiscount: TButton;
    btnNewSale: TButton;
    btnCheckout: TButton;
    btnCancelSale: TButton;
    btnReports: TButton;
    
    procedure SetupUI;
    procedure RefreshCart;
    procedure UpdateTotals;
    procedure SetupCartColumns;
    
    procedure edtBarcodeKeyPress(Sender: TObject; var Key: Char);
    procedure btnAddItemClick(Sender: TObject);
    procedure btnRemoveItemClick(Sender: TObject);
    procedure btnNewSaleClick(Sender: TObject);
    procedure btnCheckoutClick(Sender: TObject);
    procedure btnCancelSaleClick(Sender: TObject);
    procedure edtPaidChange(Sender: TObject);
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    
    procedure StartNewSale;
    procedure ProcessBarcode(const ABarcode: string);
  end;

implementation

constructor TPOSMainForm.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  
  Caption := 'POS System v1.0';
  WindowState := wsMaximized;
  Position := poScreenCenter;
  
  FProductSvc := TProductService.Create;
  FSaleSvc := TSaleService.Create;
  FUserSvc := TUserService.Create;
  
  SetupUI;
  SetupCartColumns;
  StartNewSale;
end;

destructor TPOSMainForm.Destroy;
begin
  FProductSvc.Free;
  FSaleSvc.Free;
  FUserSvc.Free;
  inherited Destroy;
end;

procedure TPOSMainForm.SetupUI;
begin
  // Top panel
  pnlTop := TPanel.Create(Self);
  pnlTop.Parent := Self;
  pnlTop.Align := alTop;
  pnlTop.Height := 60;
  pnlTop.Color := $002D2D2D;
  pnlTop.BevelOuter := bvNone;
  
  var lblLogo := TLabel.Create(Self);
  lblLogo.Parent := pnlTop;
  lblLogo.Caption := 'POS SYSTEM';
  lblLogo.Font.Color := clWhite;
  lblLogo.Font.Size := 16;
  lblLogo.Font.Style := [fsBold];
  lblLogo.Left := 20;
  lblLogo.Top := 15;
  
  // Left panel - Barcode & buttons
  pnlLeft := TPanel.Create(Self);
  pnlLeft.Parent := Self;
  pnlLeft.Align := alLeft;
  pnlLeft.Width := 250;
  pnlLeft.Color := $00F5F5F5;
  pnlLeft.BevelOuter := bvNone;
  
  lblBarcode := TLabel.Create(Self);
  lblBarcode.Parent := pnlLeft;
  lblBarcode.Caption := 'สแกน Barcode:';
  lblBarcode.Left := 12;
  lblBarcode.Top := 20;
  
  edtBarcode := TEdit.Create(Self);
  edtBarcode.Parent := pnlLeft;
  edtBarcode.Left := 12;
  edtBarcode.Top := 40;
  edtBarcode.Width := 220;
  edtBarcode.Height := 32;
  edtBarcode.Font.Size := 12;
  edtBarcode.OnKeyPress := @edtBarcodeKeyPress;
  
  btnAddItem := TButton.Create(Self);
  btnAddItem.Parent := pnlLeft;
  btnAddItem.Caption := 'เพิ่มสินค้า (F5)';
  btnAddItem.Left := 12; btnAddItem.Top := 90;
  btnAddItem.Width := 220; btnAddItem.Height := 36;
  btnAddItem.OnClick := @btnAddItemClick;
  
  btnRemoveItem := TButton.Create(Self);
  btnRemoveItem.Parent := pnlLeft;
  btnRemoveItem.Caption := 'ลบรายการ (Del)';
  btnRemoveItem.Left := 12; btnRemoveItem.Top := 135;
  btnRemoveItem.Width := 220; btnRemoveItem.Height := 36;
  btnRemoveItem.OnClick := @btnRemoveItemClick;
  
  btnNewSale := TButton.Create(Self);
  btnNewSale.Parent := pnlLeft;
  btnNewSale.Caption := 'บิลใหม่ (F1)';
  btnNewSale.Left := 12; btnNewSale.Top := 220;
  btnNewSale.Width := 220; btnNewSale.Height := 36;
  btnNewSale.OnClick := @btnNewSaleClick;
  
  btnCancelSale := TButton.Create(Self);
  btnCancelSale.Parent := pnlLeft;
  btnCancelSale.Caption := 'ยกเลิกบิล (F2)';
  btnCancelSale.Left := 12; btnCancelSale.Top := 265;
  btnCancelSale.Width := 220; btnCancelSale.Height := 36;
  btnCancelSale.OnClick := @btnCancelSaleClick;
  
  // Right panel - Cart
  pnlRight := TPanel.Create(Self);
  pnlRight.Parent := Self;
  pnlRight.Align := alClient;
  pnlRight.BevelOuter := bvNone;
  
  CartGrid := TStringGrid.Create(Self);
  CartGrid.Parent := pnlRight;
  CartGrid.Left := 0; CartGrid.Top := 0;
  CartGrid.Align := alClient;
  CartGrid.RowCount := 1;
  CartGrid.ColCount := 6;
  CartGrid.Options := [goColSizing, goRowSelect, goFixedVertLine, goFixedHorzLine, goVertLine, goHorzLine];
  CartGrid.DefaultRowHeight := 30;
  
  // Bottom panel - totals & payment
  pnlBottom := TPanel.Create(Self);
  pnlBottom.Parent := Self;
  pnlBottom.Align := alBottom;
  pnlBottom.Height := 120;
  pnlBottom.Color := $002D2D2D;
  pnlBottom.BevelOuter := bvNone;
  
  lblSubtotal := TLabel.Create(Self);
  lblSubtotal.Parent := pnlBottom;
  lblSubtotal.Caption := 'ยอดรวม: 0.00';
  lblSubtotal.Font.Color := clWhite;
  lblSubtotal.Left := 20; lblSubtotal.Top := 15;
  
  lblDiscount := TLabel.Create(Self);
  lblDiscount.Parent := pnlBottom;
  lblDiscount.Caption := 'ส่วนลด: 0.00';
  lblDiscount.Font.Color := $00AAAAAA;
  lblDiscount.Left := 20; lblDiscount.Top := 35;
  
  lblTotal := TLabel.Create(Self);
  lblTotal.Parent := pnlBottom;
  lblTotal.Caption := 'ยอดสุทธิ: 0.00';
  lblTotal.Font.Color := $00F4A740;
  lblTotal.Font.Size := 18;
  lblTotal.Font.Style := [fsBold];
  lblTotal.Left := 20; lblTotal.Top := 60;
  
  var lblPaid := TLabel.Create(Self);
  lblPaid.Parent := pnlBottom;
  lblPaid.Caption := 'รับเงิน:';
  lblPaid.Font.Color := clWhite;
  lblPaid.Left := 400; lblPaid.Top := 15;
  
  edtPaid := TEdit.Create(Self);
  edtPaid.Parent := pnlBottom;
  edtPaid.Left := 460; edtPaid.Top := 10;
  edtPaid.Width := 120; edtPaid.Height := 28;
  edtPaid.Font.Size := 12;
  edtPaid.Text := '0';
  edtPaid.OnChange := @edtPaidChange;
  
  lblChange := TLabel.Create(Self);
  lblChange.Parent := pnlBottom;
  lblChange.Caption := 'เงินทอน: 0.00';
  lblChange.Font.Color := $0000FF80;
  lblChange.Font.Size := 12;
  lblChange.Left := 400; lblChange.Top := 50;
  
  cmbPayment := TComboBox.Create(Self);
  cmbPayment.Parent := pnlBottom;
  cmbPayment.Left := 400; cmbPayment.Top := 80;
  cmbPayment.Width := 180;
  cmbPayment.Style := csDropDownList;
  cmbPayment.Items.AddStrings(['เงินสด', 'บัตรเครดิต', 'โอนเงิน', 'QR Code']);
  cmbPayment.ItemIndex := 0;
  
  btnCheckout := TButton.Create(Self);
  btnCheckout.Parent := pnlBottom;
  btnCheckout.Caption := 'ชำระเงิน (F12)';
  btnCheckout.Left := 600; btnCheckout.Top := 40;
  btnCheckout.Width := 180; btnCheckout.Height := 60;
  btnCheckout.Font.Size := 14;
  btnCheckout.Font.Style := [fsBold];
  btnCheckout.Color := $00E87722;
  btnCheckout.Font.Color := clWhite;
  btnCheckout.OnClick := @btnCheckoutClick;
end;

procedure TPOSMainForm.SetupCartColumns;
begin
  CartGrid.Cells[0, 0] := '#';
  CartGrid.Cells[1, 0] := 'สินค้า';
  CartGrid.Cells[2, 0] := 'จำนวน';
  CartGrid.Cells[3, 0] := 'ราคา/หน่วย';
  CartGrid.Cells[4, 0] := 'ส่วนลด%';
  CartGrid.Cells[5, 0] := 'รวม';
  
  CartGrid.ColWidths[0] := 40;
  CartGrid.ColWidths[1] := 200;
  CartGrid.ColWidths[2] := 80;
  CartGrid.ColWidths[3] := 100;
  CartGrid.ColWidths[4] := 80;
  CartGrid.ColWidths[5] := 100;
end;

procedure TPOSMainForm.RefreshCart;
var
  Sale: TSale;
  i: Integer;
begin
  Sale := FSaleSvc.GetCurrentSale;
  
  CartGrid.RowCount := Max(2, Sale.ItemCount + 1);
  
  for i := 0 to Sale.ItemCount - 1 do
  begin
    CartGrid.Cells[0, i + 1] := IntToStr(i + 1);
    CartGrid.Cells[1, i + 1] := Sale.Items[i].ProductName;
    CartGrid.Cells[2, i + 1] := Format('%.2f', [Sale.Items[i].Quantity]);
    CartGrid.Cells[3, i + 1] := Format('%.2f', [Sale.Items[i].UnitPrice]);
    CartGrid.Cells[4, i + 1] := Format('%.0f%%', [Sale.Items[i].DiscountPercent]);
    CartGrid.Cells[5, i + 1] := Format('%.2f', [Sale.Items[i].TotalPrice]);
  end;
  
  UpdateTotals;
end;

procedure TPOSMainForm.UpdateTotals;
var
  Sale: TSale;
begin
  Sale := FSaleSvc.GetCurrentSale;
  
  lblSubtotal.Caption := Format('ยอดรวม: %.2f', [Sale.Subtotal]);
  lblDiscount.Caption := Format('ส่วนลด: %.2f', [Sale.DiscountAmount]);
  lblTotal.Caption := Format('ยอดสุทธิ: %.2f', [Sale.TotalAmount]);
  
  edtPaidChange(nil);
end;

procedure TPOSMainForm.StartNewSale;
begin
  FSaleSvc.StartNewSale;
  RefreshCart;
  edtBarcode.SetFocus;
end;

procedure TPOSMainForm.ProcessBarcode(const ABarcode: string);
var
  Product: TProduct;
begin
  if ABarcode = '' then Exit;
  
  if FProductSvc.FindByBarcode(ABarcode, Product) then
  begin
    if FSaleSvc.AddItem(Product.ID, 1) then
    begin
      RefreshCart;
      edtBarcode.Clear;
    end
    else
      ShowMessage('ไม่สามารถเพิ่มสินค้า: ' + Product.Name);
  end
  else
    ShowMessage('ไม่พบสินค้า barcode: ' + ABarcode);
end;

procedure TPOSMainForm.edtBarcodeKeyPress(Sender: TObject; var Key: Char);
begin
  if Key = #13 then  // Enter key
  begin
    ProcessBarcode(Trim(edtBarcode.Text));
    Key := #0;
  end;
end;

procedure TPOSMainForm.btnAddItemClick(Sender: TObject);
begin
  ProcessBarcode(Trim(edtBarcode.Text));
end;

procedure TPOSMainForm.btnRemoveItemClick(Sender: TObject);
begin
  if CartGrid.Row > 0 then
  begin
    FSaleSvc.RemoveItem(CartGrid.Row - 1);
    RefreshCart;
  end;
end;

procedure TPOSMainForm.btnNewSaleClick(Sender: TObject);
begin
  if MessageDlg('เริ่มบิลใหม่', 'ต้องการเริ่มบิลใหม่ใช่ไหม?', 
    mtConfirmation, [mbYes, mbNo], 0) = mrYes then
    StartNewSale;
end;

procedure TPOSMainForm.btnCancelSaleClick(Sender: TObject);
begin
  if MessageDlg('ยกเลิกบิล', 'ต้องการยกเลิกบิลนี้ใช่ไหม?', 
    mtConfirmation, [mbYes, mbNo], 0) = mrYes then
  begin
    FSaleSvc.CancelSale;
    StartNewSale;
  end;
end;

procedure TPOSMainForm.edtPaidChange(Sender: TObject);
var
  Sale: TSale;
  Paid: Double;
  Change: Double;
begin
  Sale := FSaleSvc.GetCurrentSale;
  Paid := StrToFloatDef(edtPaid.Text, 0);
  Change := Paid - Sale.TotalAmount;
  
  if Change >= 0 then
  begin
    lblChange.Caption := Format('เงินทอน: %.2f', [Change]);
    lblChange.Font.Color := $0000FF80;
  end
  else
  begin
    lblChange.Caption := Format('ยังขาด: %.2f', [Abs(Change)]);
    lblChange.Font.Color := $000000FF;
  end;
end;

procedure TPOSMainForm.btnCheckoutClick(Sender: TObject);
var
  Sale: TSale;
  Paid: Double;
  PaymentMethods: array[0..3] of string;
begin
  Sale := FSaleSvc.GetCurrentSale;
  
  if Sale.ItemCount = 0 then
  begin
    ShowMessage('กรุณาเพิ่มสินค้าก่อน');
    Exit;
  end;
  
  Paid := StrToFloatDef(edtPaid.Text, 0);
  
  if Paid < Sale.TotalAmount then
  begin
    ShowMessage(Format('รับเงินไม่พอ ขาด %.2f บาท', [Sale.TotalAmount - Paid]));
    Exit;
  end;
  
  PaymentMethods[0] := 'cash';
  PaymentMethods[1] := 'credit_card';
  PaymentMethods[2] := 'transfer';
  PaymentMethods[3] := 'qr_code';
  
  var PayMethod := PaymentMethods[cmbPayment.ItemIndex];
  
  if FSaleSvc.CompleteSale(PayMethod, Paid) then
  begin
    Sale := FSaleSvc.GetCurrentSale;
    
    ShowMessage(Format(
      'ชำระเงินสำเร็จ!' + #13#10 +
      'บิล: %s' + #13#10 +
      'ยอดรวม: %.2f' + #13#10 +
      'รับเงิน: %.2f' + #13#10 +
      'เงินทอน: %.2f',
      [Sale.BillNumber, Sale.TotalAmount, 
       Sale.AmountPaid, Sale.ChangeAmount]));
       
    StartNewSale;
  end
  else
    ShowMessage('เกิดข้อผิดพลาดในการบันทึกการขาย');
end;

end.
```

---

## สรุป

ในโปรเจ็กต์นี้เราได้สร้าง:

1. **Database Schema** - Tables สำหรับ products, users, customers, sales
2. **Database Layer** - TPOSDatabase ด้วย SQLite3
3. **Service Layer** - TProductService, TSaleService, TUserService
4. **UI Layer** - TPOSMainForm สำหรับ cashier
5. **Business Logic** - การคำนวณ, stock management, user roles

ระบบ POS สมบูรณ์นี้ประกอบด้วย:
- การจัดการสินค้าและ stock
- ระบบขายที่รองรับ barcode scanning
- หลายวิธีชำระเงิน
- รายงานยอดขายประจำวัน
- ระบบ user roles (admin/manager/cashier)
- Transaction safety ด้วย SQLite transactions

เป็นตัวอย่างที่ดีของการใช้ Pascal/Lazarus ในระบบธุรกิจจริง โดยรวมเทคนิคต่างๆ ที่เรียนมาตลอดคอร์สนี้
