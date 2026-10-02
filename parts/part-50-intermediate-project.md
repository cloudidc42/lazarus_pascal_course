# Part 50 - โปรเจกต์กลาง: ERP Mini System ใน Lazarus/Pascal

## บทนำ

โปรเจกต์นี้สร้างระบบ ERP (Enterprise Resource Planning) ขนาดเล็กที่มีคุณสมบัติสำคัญ:
- จัดการลูกค้า สินค้า และคำสั่งซื้อ
- ระบบ authentication
- SQLite database
- รายงานพื้นฐาน

---

## โครงสร้างโปรเจกต์

```
erp_mini/
├── erp_mini.lpi
├── erp_mini.lpr
├── src/
│   ├── database/
│   │   ├── db_connection.pas
│   │   └── db_schema.pas
│   ├── models/
│   │   ├── model_user.pas
│   │   ├── model_customer.pas
│   │   ├── model_product.pas
│   │   └── model_order.pas
│   ├── forms/
│   │   ├── frm_main.pas
│   │   ├── frm_login.pas
│   │   ├── frm_customers.pas
│   │   ├── frm_products.pas
│   │   ├── frm_orders.pas
│   │   └── frm_reports.pas
│   └── utils/
│       ├── auth_manager.pas
│       └── report_generator.pas
└── data/
    └── erp.db (สร้างอัตโนมัติ)
```

---

## 50.1 Database Schema

```pascal
{$mode objfpc}{$H+}

// db_schema.pas
unit DBSchema;

interface

const
  SQL_CREATE_USERS = '''
    CREATE TABLE IF NOT EXISTS users (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      username TEXT UNIQUE NOT NULL,
      password_hash TEXT NOT NULL,
      full_name TEXT NOT NULL,
      role TEXT NOT NULL DEFAULT ''user'',
      is_active INTEGER NOT NULL DEFAULT 1,
      last_login DATETIME,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  ''';

  SQL_CREATE_CUSTOMERS = '''
    CREATE TABLE IF NOT EXISTS customers (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      code TEXT UNIQUE NOT NULL,
      name TEXT NOT NULL,
      email TEXT,
      phone TEXT,
      address TEXT,
      tax_id TEXT,
      credit_limit REAL DEFAULT 0,
      balance REAL DEFAULT 0,
      is_active INTEGER DEFAULT 1,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  ''';

  SQL_CREATE_PRODUCTS = '''
    CREATE TABLE IF NOT EXISTS products (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      code TEXT UNIQUE NOT NULL,
      name TEXT NOT NULL,
      description TEXT,
      category TEXT,
      unit TEXT DEFAULT ''pcs'',
      price REAL NOT NULL DEFAULT 0,
      cost REAL NOT NULL DEFAULT 0,
      stock_qty REAL DEFAULT 0,
      min_stock REAL DEFAULT 0,
      is_active INTEGER DEFAULT 1,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  ''';

  SQL_CREATE_ORDERS = '''
    CREATE TABLE IF NOT EXISTS orders (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      order_no TEXT UNIQUE NOT NULL,
      customer_id INTEGER NOT NULL,
      order_date DATE NOT NULL,
      due_date DATE,
      status TEXT DEFAULT ''draft'',
      subtotal REAL DEFAULT 0,
      discount REAL DEFAULT 0,
      tax REAL DEFAULT 0,
      total REAL DEFAULT 0,
      notes TEXT,
      created_by INTEGER,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
      FOREIGN KEY (customer_id) REFERENCES customers(id),
      FOREIGN KEY (created_by) REFERENCES users(id)
    )
  ''';

  SQL_CREATE_ORDER_ITEMS = '''
    CREATE TABLE IF NOT EXISTS order_items (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      order_id INTEGER NOT NULL,
      product_id INTEGER NOT NULL,
      quantity REAL NOT NULL DEFAULT 1,
      unit_price REAL NOT NULL DEFAULT 0,
      discount REAL DEFAULT 0,
      subtotal REAL DEFAULT 0,
      FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
      FOREIGN KEY (product_id) REFERENCES products(id)
    )
  ''';

  SQL_CREATE_AUDIT_LOG = '''
    CREATE TABLE IF NOT EXISTS audit_log (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      user_id INTEGER,
      action TEXT NOT NULL,
      table_name TEXT,
      record_id INTEGER,
      old_data TEXT,
      new_data TEXT,
      ip_address TEXT,
      created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )
  ''';

  // Default admin user (password: admin123)
  SQL_INSERT_DEFAULT_ADMIN = '''
    INSERT OR IGNORE INTO users (username, password_hash, full_name, role)
    VALUES (''admin'', ''$2a$10$placeholder_hash'', ''Administrator'', ''admin'')
  ''';

implementation

end.
```

---

## 50.2 Database Connection

```pascal
{$mode objfpc}{$H+}

// db_connection.pas
unit DBConnection;

interface

uses
  Classes, SysUtils, sqlite3conn, sqldb, db;

type
  TDatabase = class
  private
    class var FInstance: TDatabase;
    FConnection: TSQLite3Connection;
    FTransaction: TSQLTransaction;
    FConnected: Boolean;
    FDBPath: String;

    procedure CreateTables;
    procedure InsertDefaultData;
  public
    constructor Create(const DBPath: String);
    destructor Destroy; override;

    class function GetInstance: TDatabase;
    class procedure SetDBPath(const Path: String);
    class procedure FreeInstance;

    function Connect: Boolean;
    procedure Disconnect;

    // Query execution
    function ExecuteSQL(const SQL: String): Boolean;
    function ExecuteQuery(const SQL: String): TSQLQuery;
    function GetScalar(const SQL: String): Variant;
    function GetInt(const SQL: String): Integer;

    // Transaction
    procedure BeginTransaction;
    procedure Commit;
    procedure Rollback;

    property Connection: TSQLite3Connection read FConnection;
    property Transaction: TSQLTransaction read FTransaction;
    property Connected: Boolean read FConnected;
  end;

// Helper function
function DB: TDatabase;

implementation

uses
  DBSchema;

var
  GDBPath: String = '';

class function TDatabase.GetInstance: TDatabase;
begin
  if FInstance = nil then
    FInstance := TDatabase.Create(GDBPath);
  Result := FInstance;
end;

class procedure TDatabase.SetDBPath(const Path: String);
begin
  GDBPath := Path;
end;

class procedure TDatabase.FreeInstance;
begin
  FreeAndNil(FInstance);
end;

constructor TDatabase.Create(const DBPath: String);
begin
  inherited Create;
  FDBPath := DBPath;
  FConnected := False;

  FTransaction := TSQLTransaction.Create(nil);
  FConnection := TSQLite3Connection.Create(nil);
  FConnection.Transaction := FTransaction;
  FTransaction.DataBase := FConnection;

  Connect;
end;

destructor TDatabase.Destroy;
begin
  Disconnect;
  FConnection.Free;
  FTransaction.Free;
  inherited;
end;

function TDatabase.Connect: Boolean;
begin
  Result := False;
  try
    if FDBPath = '' then
      FDBPath := ExtractFilePath(ParamStr(0)) + 'data' + PathDelim + 'erp.db';

    ForceDirectories(ExtractFilePath(FDBPath));

    FConnection.DatabaseName := FDBPath;
    FConnection.Open;
    FTransaction.Active := True;

    CreateTables;
    InsertDefaultData;
    Commit;

    FConnected := True;
    Result := True;
  except
    on E: Exception do
    begin
      WriteLn('Database connection error: ', E.Message);
      FConnected := False;
    end;
  end;
end;

procedure TDatabase.Disconnect;
begin
  if FConnected then
  begin
    try
      if FTransaction.Active then
        FTransaction.Rollback;
    except
    end;
    FConnection.Close;
    FConnected := False;
  end;
end;

procedure TDatabase.CreateTables;
begin
  ExecuteSQL(SQL_CREATE_USERS);
  ExecuteSQL(SQL_CREATE_CUSTOMERS);
  ExecuteSQL(SQL_CREATE_PRODUCTS);
  ExecuteSQL(SQL_CREATE_ORDERS);
  ExecuteSQL(SQL_CREATE_ORDER_ITEMS);
  ExecuteSQL(SQL_CREATE_AUDIT_LOG);
end;

procedure TDatabase.InsertDefaultData;
begin
  // ตรวจสอบว่ามี admin user แล้วหรือยัง
  if GetInt('SELECT COUNT(*) FROM users WHERE username = ''admin''') = 0 then
  begin
    // สร้าง default admin (password: admin123)
    ExecuteSQL(
      'INSERT INTO users (username, password_hash, full_name, role) ' +
      'VALUES (''admin'', ''8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918'', ' +
      '''Administrator'', ''admin'')'
    );
  end;
end;

function TDatabase.ExecuteSQL(const SQL: String): Boolean;
begin
  Result := False;
  try
    FConnection.ExecuteDirect(SQL);
    Result := True;
  except
    on E: Exception do
      WriteLn('SQL Error: ', E.Message, #13#10'SQL: ', SQL);
  end;
end;

function TDatabase.ExecuteQuery(const SQL: String): TSQLQuery;
begin
  Result := TSQLQuery.Create(nil);
  try
    Result.DataBase := FConnection;
    Result.Transaction := FTransaction;
    Result.SQL.Text := SQL;
    Result.Open;
  except
    on E: Exception do
    begin
      FreeAndNil(Result);
      WriteLn('Query Error: ', E.Message);
    end;
  end;
end;

function TDatabase.GetScalar(const SQL: String): Variant;
var
  Q: TSQLQuery;
begin
  Result := Null;
  Q := ExecuteQuery(SQL);
  if Assigned(Q) then
  try
    if not Q.IsEmpty then
      Result := Q.Fields[0].AsVariant;
  finally
    Q.Free;
  end;
end;

function TDatabase.GetInt(const SQL: String): Integer;
var
  V: Variant;
begin
  V := GetScalar(SQL);
  if VarIsNull(V) then
    Result := 0
  else
    Result := Integer(V);
end;

procedure TDatabase.BeginTransaction;
begin
  if not FTransaction.Active then
    FTransaction.StartTransaction;
end;

procedure TDatabase.Commit;
begin
  if FTransaction.Active then
    FTransaction.Commit;
  FTransaction.StartTransaction;
end;

procedure TDatabase.Rollback;
begin
  if FTransaction.Active then
    FTransaction.Rollback;
  FTransaction.StartTransaction;
end;

function DB: TDatabase;
begin
  Result := TDatabase.GetInstance;
end;

end.
```

---

## 50.3 Authentication Manager

```pascal
{$mode objfpc}{$H+}

// auth_manager.pas
unit AuthManager;

interface

uses
  Classes, SysUtils;

type
  TUserRole = (urGuest, urUser, urManager, urAdmin);

  TCurrentUser = record
    ID: Integer;
    Username: String;
    FullName: String;
    Role: TUserRole;
    IsLoggedIn: Boolean;
  end;

  TAuthManager = class
  private
    class var FInstance: TAuthManager;
    FCurrentUser: TCurrentUser;

    function HashPassword(const Password: String): String;
    function RoleFromString(const S: String): TUserRole;
    function StringFromRole(Role: TUserRole): String;
  public
    class function GetInstance: TAuthManager;
    class procedure FreeInstance;

    function Login(const Username, Password: String): Boolean;
    procedure Logout;
    function ChangePassword(const OldPass, NewPass: String): Boolean;

    function HasPermission(const Permission: String): Boolean;
    function IsAdmin: Boolean;
    function IsManager: Boolean;

    property CurrentUser: TCurrentUser read FCurrentUser;
    property IsLoggedIn: Boolean read FCurrentUser.IsLoggedIn;
  end;

function Auth: TAuthManager;

implementation

uses
  sha256, DBConnection;

class function TAuthManager.GetInstance: TAuthManager;
begin
  if FInstance = nil then
    FInstance := TAuthManager.Create;
  Result := FInstance;
end;

class procedure TAuthManager.FreeInstance;
begin
  FreeAndNil(FInstance);
end;

function TAuthManager.HashPassword(const Password: String): String;
begin
  // SHA256 hash (ใน production ควรใช้ bcrypt หรือ Argon2)
  Result := SHA256Print(SHA256String(Password));
end;

function TAuthManager.RoleFromString(const S: String): TUserRole;
begin
  case LowerCase(S) of
    'admin': Result := urAdmin;
    'manager': Result := urManager;
    'user': Result := urUser;
    else Result := urGuest;
  end;
end;

function TAuthManager.StringFromRole(Role: TUserRole): String;
begin
  case Role of
    urAdmin: Result := 'admin';
    urManager: Result := 'manager';
    urUser: Result := 'user';
    else Result := 'guest';
  end;
end;

function TAuthManager.Login(const Username, Password: String): Boolean;
var
  Q: TSQLQuery;
  HashedPwd: String;
begin
  Result := False;
  FCurrentUser.IsLoggedIn := False;

  HashedPwd := HashPassword(Password);

  Q := DB.ExecuteQuery(
    'SELECT id, username, full_name, role FROM users ' +
    'WHERE username = ''' + Username + ''' ' +
    'AND password_hash = ''' + HashedPwd + ''' ' +
    'AND is_active = 1'
  );

  if Assigned(Q) then
  try
    if not Q.IsEmpty then
    begin
      FCurrentUser.ID := Q.FieldByName('id').AsInteger;
      FCurrentUser.Username := Q.FieldByName('username').AsString;
      FCurrentUser.FullName := Q.FieldByName('full_name').AsString;
      FCurrentUser.Role := RoleFromString(Q.FieldByName('role').AsString);
      FCurrentUser.IsLoggedIn := True;

      // อัพเดท last_login
      DB.ExecuteSQL(Format(
        'UPDATE users SET last_login = CURRENT_TIMESTAMP WHERE id = %d',
        [FCurrentUser.ID]
      ));
      DB.Commit;

      Result := True;
    end;
  finally
    Q.Free;
  end;
end;

procedure TAuthManager.Logout;
begin
  FCurrentUser.IsLoggedIn := False;
  FCurrentUser.ID := 0;
  FCurrentUser.Username := '';
  FCurrentUser.FullName := '';
  FCurrentUser.Role := urGuest;
end;

function TAuthManager.ChangePassword(const OldPass, NewPass: String): Boolean;
var
  OldHash, NewHash: String;
begin
  Result := False;
  if not FCurrentUser.IsLoggedIn then Exit;

  OldHash := HashPassword(OldPass);
  NewHash := HashPassword(NewPass);

  // ตรวจสอบ old password
  if DB.GetInt(Format(
    'SELECT COUNT(*) FROM users WHERE id = %d AND password_hash = ''%s''',
    [FCurrentUser.ID, OldHash])) = 0 then
    Exit;

  // เปลี่ยน password
  if DB.ExecuteSQL(Format(
    'UPDATE users SET password_hash = ''%s'' WHERE id = %d',
    [NewHash, FCurrentUser.ID])) then
  begin
    DB.Commit;
    Result := True;
  end;
end;

function TAuthManager.HasPermission(const Permission: String): Boolean;
begin
  if not FCurrentUser.IsLoggedIn then Exit(False);

  case FCurrentUser.Role of
    urAdmin: Result := True; // Admin มีทุก permission
    urManager:
      Result := (Permission <> 'manage_users') and
                (Permission <> 'system_settings');
    urUser:
      Result := Permission in ['view_customers', 'create_order',
                               'view_products', 'view_reports'];
    else Result := False;
  end;
end;

function TAuthManager.IsAdmin: Boolean;
begin
  Result := FCurrentUser.Role = urAdmin;
end;

function TAuthManager.IsManager: Boolean;
begin
  Result := FCurrentUser.Role in [urAdmin, urManager];
end;

function Auth: TAuthManager;
begin
  Result := TAuthManager.GetInstance;
end;

end.
```

---

## 50.4 Customer Model

```pascal
{$mode objfpc}{$H+}

// model_customer.pas
unit ModelCustomer;

interface

uses
  Classes, SysUtils, db;

type
  TCustomer = record
    ID: Integer;
    Code: String;
    Name: String;
    Email: String;
    Phone: String;
    Address: String;
    TaxID: String;
    CreditLimit: Double;
    Balance: Double;
    IsActive: Boolean;
  end;

  TCustomerList = array of TCustomer;

  TCustomerModel = class
  private
    function CustomerFromQuery(Q: TSQLQuery): TCustomer;
    function EscapeSQL(const S: String): String;
  public
    function GetAll(ActiveOnly: Boolean = True): TCustomerList;
    function GetByID(ID: Integer): TCustomer;
    function GetByCode(const Code: String): TCustomer;
    function Search(const Keyword: String): TCustomerList;

    function Save(var Customer: TCustomer): Boolean; // Insert or Update
    function Delete(ID: Integer): Boolean;
    function SetActive(ID: Integer; Active: Boolean): Boolean;

    function GenerateCode: String;
    function ValidateEmail(const Email: String): Boolean;
    function CodeExists(const Code: String; ExcludeID: Integer = 0): Boolean;
  end;

implementation

uses
  DBConnection, AuthManager, DateUtils;

function TCustomerModel.EscapeSQL(const S: String): String;
begin
  Result := StringReplace(S, '''', '''''', [rfReplaceAll]);
end;

function TCustomerModel.CustomerFromQuery(Q: TSQLQuery): TCustomer;
begin
  Result.ID := Q.FieldByName('id').AsInteger;
  Result.Code := Q.FieldByName('code').AsString;
  Result.Name := Q.FieldByName('name').AsString;
  Result.Email := Q.FieldByName('email').AsString;
  Result.Phone := Q.FieldByName('phone').AsString;
  Result.Address := Q.FieldByName('address').AsString;
  Result.TaxID := Q.FieldByName('tax_id').AsString;
  Result.CreditLimit := Q.FieldByName('credit_limit').AsFloat;
  Result.Balance := Q.FieldByName('balance').AsFloat;
  Result.IsActive := Q.FieldByName('is_active').AsInteger = 1;
end;

function TCustomerModel.GetAll(ActiveOnly: Boolean = True): TCustomerList;
var
  Q: TSQLQuery;
  SQL: String;
  Count: Integer;
begin
  SetLength(Result, 0);

  SQL := 'SELECT * FROM customers';
  if ActiveOnly then
    SQL := SQL + ' WHERE is_active = 1';
  SQL := SQL + ' ORDER BY name';

  Q := DB.ExecuteQuery(SQL);
  if not Assigned(Q) then Exit;

  try
    Count := 0;
    while not Q.EOF do
    begin
      Inc(Count);
      SetLength(Result, Count);
      Result[Count - 1] := CustomerFromQuery(Q);
      Q.Next;
    end;
  finally
    Q.Free;
  end;
end;

function TCustomerModel.GetByID(ID: Integer): TCustomer;
var
  Q: TSQLQuery;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.ID := -1;

  Q := DB.ExecuteQuery(Format('SELECT * FROM customers WHERE id = %d', [ID]));
  if Assigned(Q) then
  try
    if not Q.IsEmpty then
      Result := CustomerFromQuery(Q);
  finally
    Q.Free;
  end;
end;

function TCustomerModel.GetByCode(const Code: String): TCustomer;
var
  Q: TSQLQuery;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.ID := -1;

  Q := DB.ExecuteQuery(Format(
    'SELECT * FROM customers WHERE code = ''%s''', [EscapeSQL(Code)]));
  if Assigned(Q) then
  try
    if not Q.IsEmpty then
      Result := CustomerFromQuery(Q);
  finally
    Q.Free;
  end;
end;

function TCustomerModel.Search(const Keyword: String): TCustomerList;
var
  Q: TSQLQuery;
  KW: String;
  Count: Integer;
begin
  SetLength(Result, 0);
  KW := '%' + EscapeSQL(Keyword) + '%';

  Q := DB.ExecuteQuery(Format(
    'SELECT * FROM customers WHERE ' +
    'name LIKE ''%s'' OR code LIKE ''%s'' OR ' +
    'email LIKE ''%s'' OR phone LIKE ''%s'' ' +
    'ORDER BY name',
    [KW, KW, KW, KW]));

  if not Assigned(Q) then Exit;

  try
    Count := 0;
    while not Q.EOF do
    begin
      Inc(Count);
      SetLength(Result, Count);
      Result[Count - 1] := CustomerFromQuery(Q);
      Q.Next;
    end;
  finally
    Q.Free;
  end;
end;

function TCustomerModel.Save(var Customer: TCustomer): Boolean;
var
  SQL: String;
begin
  Result := False;

  if Customer.ID = 0 then
  begin
    // INSERT
    if Customer.Code = '' then
      Customer.Code := GenerateCode;

    SQL := Format(
      'INSERT INTO customers (code, name, email, phone, address, tax_id, credit_limit) ' +
      'VALUES (''%s'', ''%s'', ''%s'', ''%s'', ''%s'', ''%s'', %f)',
      [EscapeSQL(Customer.Code), EscapeSQL(Customer.Name),
       EscapeSQL(Customer.Email), EscapeSQL(Customer.Phone),
       EscapeSQL(Customer.Address), EscapeSQL(Customer.TaxID),
       Customer.CreditLimit]);

    if DB.ExecuteSQL(SQL) then
    begin
      Customer.ID := DB.GetInt('SELECT last_insert_rowid()');
      DB.Commit;
      Result := Customer.ID > 0;
    end;
  end
  else
  begin
    // UPDATE
    SQL := Format(
      'UPDATE customers SET ' +
      'name = ''%s'', email = ''%s'', phone = ''%s'', ' +
      'address = ''%s'', tax_id = ''%s'', credit_limit = %f, ' +
      'updated_at = CURRENT_TIMESTAMP ' +
      'WHERE id = %d',
      [EscapeSQL(Customer.Name), EscapeSQL(Customer.Email),
       EscapeSQL(Customer.Phone), EscapeSQL(Customer.Address),
       EscapeSQL(Customer.TaxID), Customer.CreditLimit, Customer.ID]);

    if DB.ExecuteSQL(SQL) then
    begin
      DB.Commit;
      Result := True;
    end;
  end;
end;

function TCustomerModel.Delete(ID: Integer): Boolean;
begin
  // ตรวจสอบว่ามี orders ที่ reference customer นี้อยู่
  if DB.GetInt(Format(
    'SELECT COUNT(*) FROM orders WHERE customer_id = %d', [ID])) > 0 then
  begin
    // Soft delete ถ้ามี orders
    Result := SetActive(ID, False);
  end
  else
  begin
    // Hard delete ถ้าไม่มี orders
    Result := DB.ExecuteSQL(Format('DELETE FROM customers WHERE id = %d', [ID]));
    if Result then DB.Commit;
  end;
end;

function TCustomerModel.SetActive(ID: Integer; Active: Boolean): Boolean;
begin
  Result := DB.ExecuteSQL(Format(
    'UPDATE customers SET is_active = %d WHERE id = %d',
    [Ord(Active), ID]));
  if Result then DB.Commit;
end;

function TCustomerModel.GenerateCode: String;
var
  MaxCode: String;
  Num: Integer;
begin
  MaxCode := DB.GetScalar(
    'SELECT MAX(code) FROM customers WHERE code LIKE ''C%''');

  if VarIsNull(MaxCode) or (MaxCode = '') then
    Num := 1
  else
  begin
    Num := StrToIntDef(Copy(MaxCode, 2, Length(MaxCode) - 1), 0) + 1;
  end;

  Result := Format('C%05d', [Num]);
end;

function TCustomerModel.ValidateEmail(const Email: String): Boolean;
begin
  Result := (Pos('@', Email) > 1) and
            (Pos('.', Email) > Pos('@', Email) + 1);
end;

function TCustomerModel.CodeExists(const Code: String; ExcludeID: Integer = 0): Boolean;
var
  SQL: String;
begin
  SQL := Format('SELECT COUNT(*) FROM customers WHERE code = ''%s''',
    [EscapeSQL(Code)]);
  if ExcludeID > 0 then
    SQL := SQL + Format(' AND id <> %d', [ExcludeID]);
  Result := DB.GetInt(SQL) > 0;
end;

end.
```

---

## 50.5 Order Model

```pascal
{$mode objfpc}{$H+}

// model_order.pas
unit ModelOrder;

interface

uses
  Classes, SysUtils, db;

type
  TOrderStatus = (osDraft, osConfirmed, osShipped, osCompleted, osCancelled);

  TOrderItem = record
    ID: Integer;
    OrderID: Integer;
    ProductID: Integer;
    ProductCode: String;
    ProductName: String;
    Quantity: Double;
    UnitPrice: Double;
    Discount: Double;
    Subtotal: Double;
  end;

  TOrder = record
    ID: Integer;
    OrderNo: String;
    CustomerID: Integer;
    CustomerName: String;
    OrderDate: TDateTime;
    DueDate: TDateTime;
    Status: TOrderStatus;
    Subtotal: Double;
    Discount: Double;
    Tax: Double;
    Total: Double;
    Notes: String;
    Items: array of TOrderItem;
    ItemCount: Integer;
  end;

  TOrderModel = class
  private
    function StatusFromString(const S: String): TOrderStatus;
    function StringFromStatus(Status: TOrderStatus): String;
    function EscapeSQL(const S: String): String;
    function OrderFromQuery(Q: TSQLQuery): TOrder;
    procedure LoadOrderItems(var Order: TOrder);
    procedure RecalcOrder(var Order: TOrder);
  public
    function GetAll(const FilterStatus: String = ''): array of TOrder;
    function GetByID(ID: Integer): TOrder;
    function GetByOrderNo(const OrderNo: String): TOrder;
    function GetByCustomer(CustomerID: Integer): array of TOrder;

    function CreateOrder(CustomerID: Integer): TOrder;
    function SaveOrder(var Order: TOrder): Boolean;
    function AddItem(var Order: TOrder; ProductID: Integer;
      Qty, Price, Discount: Double): Boolean;
    function RemoveItem(var Order: TOrder; ItemID: Integer): Boolean;
    function UpdateItemQty(var Order: TOrder; ItemID: Integer; Qty: Double): Boolean;

    function ConfirmOrder(ID: Integer): Boolean;
    function CancelOrder(ID: Integer): Boolean;
    function CompleteOrder(ID: Integer): Boolean;

    function GenerateOrderNo: String;
    function GetTotal(ID: Integer): Double;
  end;

implementation

uses
  DBConnection, AuthManager, DateUtils;

function TOrderModel.EscapeSQL(const S: String): String;
begin
  Result := StringReplace(S, '''', '''''', [rfReplaceAll]);
end;

function TOrderModel.StatusFromString(const S: String): TOrderStatus;
begin
  case LowerCase(S) of
    'confirmed': Result := osConfirmed;
    'shipped': Result := osShipped;
    'completed': Result := osCompleted;
    'cancelled': Result := osCancelled;
    else Result := osDraft;
  end;
end;

function TOrderModel.StringFromStatus(Status: TOrderStatus): String;
begin
  case Status of
    osConfirmed: Result := 'confirmed';
    osShipped: Result := 'shipped';
    osCompleted: Result := 'completed';
    osCancelled: Result := 'cancelled';
    else Result := 'draft';
  end;
end;

function TOrderModel.OrderFromQuery(Q: TSQLQuery): TOrder;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.ID := Q.FieldByName('id').AsInteger;
  Result.OrderNo := Q.FieldByName('order_no').AsString;
  Result.CustomerID := Q.FieldByName('customer_id').AsInteger;
  Result.CustomerName := Q.FieldByName('customer_name').AsString;
  Result.OrderDate := Q.FieldByName('order_date').AsDateTime;
  Result.DueDate := Q.FieldByName('due_date').AsDateTime;
  Result.Status := StatusFromString(Q.FieldByName('status').AsString);
  Result.Subtotal := Q.FieldByName('subtotal').AsFloat;
  Result.Discount := Q.FieldByName('discount').AsFloat;
  Result.Tax := Q.FieldByName('tax').AsFloat;
  Result.Total := Q.FieldByName('total').AsFloat;
  Result.Notes := Q.FieldByName('notes').AsString;
end;

procedure TOrderModel.LoadOrderItems(var Order: TOrder);
var
  Q: TSQLQuery;
begin
  Order.ItemCount := 0;
  SetLength(Order.Items, 0);

  Q := DB.ExecuteQuery(Format(
    'SELECT oi.*, p.code AS product_code, p.name AS product_name ' +
    'FROM order_items oi ' +
    'JOIN products p ON p.id = oi.product_id ' +
    'WHERE oi.order_id = %d ORDER BY oi.id', [Order.ID]));

  if not Assigned(Q) then Exit;

  try
    while not Q.EOF do
    begin
      Inc(Order.ItemCount);
      SetLength(Order.Items, Order.ItemCount);

      with Order.Items[Order.ItemCount - 1] do
      begin
        ID := Q.FieldByName('id').AsInteger;
        OrderID := Order.ID;
        ProductID := Q.FieldByName('product_id').AsInteger;
        ProductCode := Q.FieldByName('product_code').AsString;
        ProductName := Q.FieldByName('product_name').AsString;
        Quantity := Q.FieldByName('quantity').AsFloat;
        UnitPrice := Q.FieldByName('unit_price').AsFloat;
        Discount := Q.FieldByName('discount').AsFloat;
        Subtotal := Q.FieldByName('subtotal').AsFloat;
      end;

      Q.Next;
    end;
  finally
    Q.Free;
  end;
end;

procedure TOrderModel.RecalcOrder(var Order: TOrder);
var
  i: Integer;
begin
  Order.Subtotal := 0;
  for i := 0 to Order.ItemCount - 1 do
    Order.Subtotal := Order.Subtotal + Order.Items[i].Subtotal;

  Order.Total := Order.Subtotal - Order.Discount + Order.Tax;
end;

function TOrderModel.GetByID(ID: Integer): TOrder;
var
  Q: TSQLQuery;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.ID := -1;

  Q := DB.ExecuteQuery(Format(
    'SELECT o.*, c.name AS customer_name ' +
    'FROM orders o JOIN customers c ON c.id = o.customer_id ' +
    'WHERE o.id = %d', [ID]));

  if Assigned(Q) then
  try
    if not Q.IsEmpty then
    begin
      Result := OrderFromQuery(Q);
      LoadOrderItems(Result);
    end;
  finally
    Q.Free;
  end;
end;

function TOrderModel.CreateOrder(CustomerID: Integer): TOrder;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.ID := 0;
  Result.OrderNo := GenerateOrderNo;
  Result.CustomerID := CustomerID;
  Result.OrderDate := Date;
  Result.DueDate := Date + 30;
  Result.Status := osdraft;
  Result.ItemCount := 0;
end;

function TOrderModel.SaveOrder(var Order: TOrder): Boolean;
var
  SQL: String;
  i: Integer;
begin
  Result := False;

  RecalcOrder(Order);

  DB.BeginTransaction;
  try
    if Order.ID = 0 then
    begin
      // INSERT
      SQL := Format(
        'INSERT INTO orders (order_no, customer_id, order_date, due_date, ' +
        'status, subtotal, discount, tax, total, notes, created_by) ' +
        'VALUES (''%s'', %d, ''%s'', ''%s'', ''%s'', %f, %f, %f, %f, ''%s'', %d)',
        [EscapeSQL(Order.OrderNo), Order.CustomerID,
         FormatDateTime('yyyy-mm-dd', Order.OrderDate),
         FormatDateTime('yyyy-mm-dd', Order.DueDate),
         StringFromStatus(Order.Status),
         Order.Subtotal, Order.Discount, Order.Tax, Order.Total,
         EscapeSQL(Order.Notes), Auth.CurrentUser.ID]);

      if not DB.ExecuteSQL(SQL) then Exit;
      Order.ID := DB.GetInt('SELECT last_insert_rowid()');
    end
    else
    begin
      // UPDATE
      SQL := Format(
        'UPDATE orders SET subtotal = %f, discount = %f, tax = %f, ' +
        'total = %f, notes = ''%s'', updated_at = CURRENT_TIMESTAMP ' +
        'WHERE id = %d',
        [Order.Subtotal, Order.Discount, Order.Tax, Order.Total,
         EscapeSQL(Order.Notes), Order.ID]);

      if not DB.ExecuteSQL(SQL) then Exit;

      // ลบ items เก่า
      DB.ExecuteSQL(Format('DELETE FROM order_items WHERE order_id = %d', [Order.ID]));
    end;

    // INSERT items
    for i := 0 to Order.ItemCount - 1 do
    begin
      with Order.Items[i] do
      begin
        Subtotal := Quantity * UnitPrice * (1 - Discount / 100);
        SQL := Format(
          'INSERT INTO order_items (order_id, product_id, quantity, unit_price, discount, subtotal) ' +
          'VALUES (%d, %d, %f, %f, %f, %f)',
          [Order.ID, ProductID, Quantity, UnitPrice, Discount, Subtotal]);
        if not DB.ExecuteSQL(SQL) then Exit;
      end;
    end;

    DB.Commit;
    Result := True;
  except
    DB.Rollback;
    raise;
  end;
end;

function TOrderModel.ConfirmOrder(ID: Integer): Boolean;
begin
  Result := DB.ExecuteSQL(Format(
    'UPDATE orders SET status = ''confirmed'', updated_at = CURRENT_TIMESTAMP ' +
    'WHERE id = %d AND status = ''draft''', [ID]));
  if Result then DB.Commit;
end;

function TOrderModel.CancelOrder(ID: Integer): Boolean;
begin
  Result := DB.ExecuteSQL(Format(
    'UPDATE orders SET status = ''cancelled'', updated_at = CURRENT_TIMESTAMP ' +
    'WHERE id = %d AND status IN (''draft'', ''confirmed'')', [ID]));
  if Result then DB.Commit;
end;

function TOrderModel.GenerateOrderNo: String;
var
  YM: String;
  Seq: Integer;
begin
  YM := FormatDateTime('YYYYMM', Date);
  Seq := DB.GetInt(Format(
    'SELECT COUNT(*) + 1 FROM orders WHERE order_no LIKE ''ORD-%s-%%''', [YM]));
  Result := Format('ORD-%s-%04d', [YM, Seq]);
end;

function TOrderModel.GetTotal(ID: Integer): Double;
begin
  Result := DB.GetScalar(Format(
    'SELECT total FROM orders WHERE id = %d', [ID]));
end;

function TOrderModel.AddItem(var Order: TOrder; ProductID: Integer;
  Qty, Price, Discount: Double): Boolean;
var
  Idx: Integer;
begin
  Result := True;
  Idx := Order.ItemCount;
  Inc(Order.ItemCount);
  SetLength(Order.Items, Order.ItemCount);

  with Order.Items[Idx] do
  begin
    ID := 0;
    OrderID := Order.ID;
    ProductID := ProductID;
    Quantity := Qty;
    UnitPrice := Price;
    Discount := Discount;
    Subtotal := Qty * Price * (1 - Discount / 100);
  end;

  RecalcOrder(Order);
end;

function TOrderModel.RemoveItem(var Order: TOrder; ItemID: Integer): Boolean;
var
  i, NewCount: Integer;
  NewItems: array of TOrderItem;
begin
  Result := False;
  NewCount := 0;
  SetLength(NewItems, Order.ItemCount);

  for i := 0 to Order.ItemCount - 1 do
  begin
    if Order.Items[i].ID <> ItemID then
    begin
      NewItems[NewCount] := Order.Items[i];
      Inc(NewCount);
      Result := True;
    end;
  end;

  Order.Items := NewItems;
  Order.ItemCount := NewCount;
  RecalcOrder(Order);
end;

function TOrderModel.UpdateItemQty(var Order: TOrder; ItemID: Integer; Qty: Double): Boolean;
var
  i: Integer;
begin
  Result := False;
  for i := 0 to Order.ItemCount - 1 do
  begin
    if Order.Items[i].ID = ItemID then
    begin
      Order.Items[i].Quantity := Qty;
      Order.Items[i].Subtotal := Qty * Order.Items[i].UnitPrice *
        (1 - Order.Items[i].Discount / 100);
      RecalcOrder(Order);
      Result := True;
      Break;
    end;
  end;
end;

function TOrderModel.GetByOrderNo(const OrderNo: String): TOrder;
var
  Q: TSQLQuery;
begin
  FillChar(Result, SizeOf(Result), 0);
  Result.ID := -1;

  Q := DB.ExecuteQuery(Format(
    'SELECT o.*, c.name AS customer_name ' +
    'FROM orders o JOIN customers c ON c.id = o.customer_id ' +
    'WHERE o.order_no = ''%s''', [EscapeSQL(OrderNo)]));

  if Assigned(Q) then
  try
    if not Q.IsEmpty then
    begin
      Result := OrderFromQuery(Q);
      LoadOrderItems(Result);
    end;
  finally
    Q.Free;
  end;
end;

function TOrderModel.GetByCustomer(CustomerID: Integer): array of TOrder;
var
  Q: TSQLQuery;
  Count: Integer;
begin
  SetLength(Result, 0);

  Q := DB.ExecuteQuery(Format(
    'SELECT o.*, c.name AS customer_name ' +
    'FROM orders o JOIN customers c ON c.id = o.customer_id ' +
    'WHERE o.customer_id = %d ORDER BY o.order_date DESC', [CustomerID]));

  if not Assigned(Q) then Exit;

  try
    Count := 0;
    while not Q.EOF do
    begin
      Inc(Count);
      SetLength(Result, Count);
      Result[Count - 1] := OrderFromQuery(Q);
      Q.Next;
    end;
  finally
    Q.Free;
  end;
end;

function TOrderModel.CompleteOrder(ID: Integer): Boolean;
begin
  Result := DB.ExecuteSQL(Format(
    'UPDATE orders SET status = ''completed'', updated_at = CURRENT_TIMESTAMP ' +
    'WHERE id = %d AND status IN (''confirmed'', ''shipped'')', [ID]));
  if Result then DB.Commit;
end;

end.
```

---

## 50.6 Report Generator

```pascal
{$mode objfpc}{$H+}

// report_generator.pas
unit ReportGenerator;

interface

uses
  Classes, SysUtils;

type
  TReportType = (rtSalesSummary, rtCustomerStatement, rtInventory, rtTopProducts);

  TReportGenerator = class
  private
    function EscapeHTML(const S: String): String;
    function FormatMoney(Value: Double): String;
    function GenerateHTMLHeader(const Title: String): String;
    function GenerateHTMLFooter: String;
  public
    function GenerateSalesSummary(FromDate, ToDate: TDateTime): String;
    function GenerateCustomerStatement(CustomerID: Integer;
      FromDate, ToDate: TDateTime): String;
    function GenerateInventoryReport: String;
    function GenerateTopProducts(TopN: Integer = 10): String;

    function SaveReportToFile(const HTML, FileName: String): Boolean;
    procedure PrintReport(const HTML: String);
  end;

implementation

uses
  DBConnection, DateUtils;

function TReportGenerator.EscapeHTML(const S: String): String;
begin
  Result := S;
  Result := StringReplace(Result, '&', '&amp;', [rfReplaceAll]);
  Result := StringReplace(Result, '<', '&lt;', [rfReplaceAll]);
  Result := StringReplace(Result, '>', '&gt;', [rfReplaceAll]);
  Result := StringReplace(Result, '"', '&quot;', [rfReplaceAll]);
end;

function TReportGenerator.FormatMoney(Value: Double): String;
begin
  Result := Format('%,.2f', [Value]);
end;

function TReportGenerator.GenerateHTMLHeader(const Title: String): String;
begin
  Result :=
    '<!DOCTYPE html>' + #13#10 +
    '<html lang="th">' + #13#10 +
    '<head>' + #13#10 +
    '<meta charset="UTF-8">' + #13#10 +
    '<meta name="viewport" content="width=device-width, initial-scale=1.0">' + #13#10 +
    '<title>' + EscapeHTML(Title) + '</title>' + #13#10 +
    '<style>' + #13#10 +
    'body { font-family: Arial, sans-serif; margin: 20px; }' + #13#10 +
    'h1 { color: #333; border-bottom: 2px solid #007bff; padding-bottom: 10px; }' + #13#10 +
    'table { width: 100%; border-collapse: collapse; margin-top: 20px; }' + #13#10 +
    'th { background: #007bff; color: white; padding: 10px; text-align: left; }' + #13#10 +
    'td { padding: 8px 10px; border-bottom: 1px solid #ddd; }' + #13#10 +
    'tr:nth-child(even) { background: #f8f9fa; }' + #13#10 +
    '.total-row { font-weight: bold; background: #e8f0fe !important; }' + #13#10 +
    '.text-right { text-align: right; }' + #13#10 +
    '.report-info { margin-bottom: 20px; color: #666; }' + #13#10 +
    '.summary-box { display: inline-block; border: 1px solid #ddd; ' +
    '  padding: 15px; margin: 10px; border-radius: 5px; min-width: 150px; ' +
    '  text-align: center; }' + #13#10 +
    '.summary-value { font-size: 24px; font-weight: bold; color: #007bff; }' + #13#10 +
    '</style>' + #13#10 +
    '</head>' + #13#10 +
    '<body>' + #13#10 +
    '<h1>' + EscapeHTML(Title) + '</h1>' + #13#10;
end;

function TReportGenerator.GenerateHTMLFooter: String;
begin
  Result :=
    '<div style="margin-top:30px; color:#999; font-size:12px;">' + #13#10 +
    'Generated: ' + FormatDateTime('yyyy-mm-dd hh:nn:ss', Now) + #13#10 +
    '</div>' + #13#10 +
    '</body>' + #13#10 +
    '</html>';
end;

function TReportGenerator.GenerateSalesSummary(
  FromDate, ToDate: TDateTime): String;
var
  Q: TSQLQuery;
  HTML: TStringList;
  TotalOrders, TotalRevenue: Double;
  FromStr, ToStr: String;
begin
  FromStr := FormatDateTime('yyyy-mm-dd', FromDate);
  ToStr := FormatDateTime('yyyy-mm-dd', ToDate);

  HTML := TStringList.Create;
  try
    HTML.Add(GenerateHTMLHeader('Sales Summary Report'));
    HTML.Add(Format(
      '<div class="report-info">Period: %s to %s</div>',
      [FormatDateTime('dd/mm/yyyy', FromDate),
       FormatDateTime('dd/mm/yyyy', ToDate)]));

    // Summary boxes
    TotalOrders := DB.GetInt(Format(
      'SELECT COUNT(*) FROM orders WHERE order_date BETWEEN ''%s'' AND ''%s'' ' +
      'AND status <> ''cancelled''', [FromStr, ToStr]));

    TotalRevenue := DB.GetScalar(Format(
      'SELECT COALESCE(SUM(total), 0) FROM orders WHERE order_date BETWEEN ''%s'' AND ''%s'' ' +
      'AND status <> ''cancelled''', [FromStr, ToStr]));

    HTML.Add('<div>');
    HTML.Add(Format(
      '<div class="summary-box"><div>Total Orders</div>' +
      '<div class="summary-value">%d</div></div>',
      [Round(TotalOrders)]));
    HTML.Add(Format(
      '<div class="summary-box"><div>Total Revenue</div>' +
      '<div class="summary-value">%s</div></div>',
      [FormatMoney(TotalRevenue)]));
    HTML.Add('</div>');

    // Order detail table
    HTML.Add('<table>');
    HTML.Add('<tr><th>Order No</th><th>Date</th><th>Customer</th>' +
             '<th>Status</th><th class="text-right">Amount</th></tr>');

    Q := DB.ExecuteQuery(Format(
      'SELECT o.order_no, o.order_date, c.name AS customer_name, ' +
      'o.status, o.total ' +
      'FROM orders o JOIN customers c ON c.id = o.customer_id ' +
      'WHERE o.order_date BETWEEN ''%s'' AND ''%s'' ' +
      'AND o.status <> ''cancelled'' ORDER BY o.order_date',
      [FromStr, ToStr]));

    if Assigned(Q) then
    try
      while not Q.EOF do
      begin
        HTML.Add(Format(
          '<tr><td>%s</td><td>%s</td><td>%s</td><td>%s</td>' +
          '<td class="text-right">%s</td></tr>',
          [EscapeHTML(Q.FieldByName('order_no').AsString),
           FormatDateTime('dd/mm/yyyy', Q.FieldByName('order_date').AsDateTime),
           EscapeHTML(Q.FieldByName('customer_name').AsString),
           EscapeHTML(Q.FieldByName('status').AsString),
           FormatMoney(Q.FieldByName('total').AsFloat)]));
        Q.Next;
      end;
    finally
      Q.Free;
    end;

    HTML.Add(Format(
      '<tr class="total-row"><td colspan="4">Total</td>' +
      '<td class="text-right">%s</td></tr>',
      [FormatMoney(TotalRevenue)]));
    HTML.Add('</table>');
    HTML.Add(GenerateHTMLFooter);

    Result := HTML.Text;
  finally
    HTML.Free;
  end;
end;

function TReportGenerator.GenerateInventoryReport: String;
var
  Q: TSQLQuery;
  HTML: TStringList;
begin
  HTML := TStringList.Create;
  try
    HTML.Add(GenerateHTMLHeader('Inventory Report'));

    HTML.Add('<table>');
    HTML.Add('<tr><th>Code</th><th>Name</th><th>Category</th><th>Unit</th>' +
             '<th class="text-right">Stock</th><th class="text-right">Min Stock</th>' +
             '<th class="text-right">Price</th><th>Status</th></tr>');

    Q := DB.ExecuteQuery(
      'SELECT * FROM products WHERE is_active = 1 ORDER BY category, name');

    if Assigned(Q) then
    try
      while not Q.EOF do
      begin
        var RowClass := '';
        if Q.FieldByName('stock_qty').AsFloat <= Q.FieldByName('min_stock').AsFloat then
          RowClass := ' style="background:#fff3cd"'; // ไฮไลท์ low stock

        HTML.Add(Format(
          '<tr%s><td>%s</td><td>%s</td><td>%s</td><td>%s</td>' +
          '<td class="text-right">%s</td><td class="text-right">%s</td>' +
          '<td class="text-right">%s</td><td>%s</td></tr>',
          [RowClass,
           EscapeHTML(Q.FieldByName('code').AsString),
           EscapeHTML(Q.FieldByName('name').AsString),
           EscapeHTML(Q.FieldByName('category').AsString),
           EscapeHTML(Q.FieldByName('unit').AsString),
           FormatMoney(Q.FieldByName('stock_qty').AsFloat),
           FormatMoney(Q.FieldByName('min_stock').AsFloat),
           FormatMoney(Q.FieldByName('price').AsFloat),
           IfThen(Q.FieldByName('stock_qty').AsFloat <= Q.FieldByName('min_stock').AsFloat,
             'Low Stock', 'OK')]));
        Q.Next;
      end;
    finally
      Q.Free;
    end;

    HTML.Add('</table>');
    HTML.Add(GenerateHTMLFooter);

    Result := HTML.Text;
  finally
    HTML.Free;
  end;
end;

function TReportGenerator.GenerateTopProducts(TopN: Integer = 10): String;
var
  Q: TSQLQuery;
  HTML: TStringList;
begin
  HTML := TStringList.Create;
  try
    HTML.Add(GenerateHTMLHeader(Format('Top %d Products by Sales', [TopN])));

    HTML.Add('<table>');
    HTML.Add('<tr><th>#</th><th>Product</th><th class="text-right">Qty Sold</th>' +
             '<th class="text-right">Revenue</th></tr>');

    Q := DB.ExecuteQuery(Format(
      'SELECT p.code, p.name, ' +
      'SUM(oi.quantity) AS total_qty, ' +
      'SUM(oi.subtotal) AS total_revenue ' +
      'FROM order_items oi ' +
      'JOIN products p ON p.id = oi.product_id ' +
      'JOIN orders o ON o.id = oi.order_id ' +
      'WHERE o.status <> ''cancelled'' ' +
      'GROUP BY p.id ORDER BY total_revenue DESC LIMIT %d', [TopN]));

    if Assigned(Q) then
    try
      var Rank := 1;
      while not Q.EOF do
      begin
        HTML.Add(Format(
          '<tr><td>%d</td><td>%s - %s</td>' +
          '<td class="text-right">%s</td>' +
          '<td class="text-right">%s</td></tr>',
          [Rank,
           EscapeHTML(Q.FieldByName('code').AsString),
           EscapeHTML(Q.FieldByName('name').AsString),
           FormatMoney(Q.FieldByName('total_qty').AsFloat),
           FormatMoney(Q.FieldByName('total_revenue').AsFloat)]));
        Q.Next;
        Inc(Rank);
      end;
    finally
      Q.Free;
    end;

    HTML.Add('</table>');
    HTML.Add(GenerateHTMLFooter);

    Result := HTML.Text;
  finally
    HTML.Free;
  end;
end;

function TReportGenerator.GenerateCustomerStatement(CustomerID: Integer;
  FromDate, ToDate: TDateTime): String;
var
  Q: TSQLQuery;
  HTML: TStringList;
  FromStr, ToStr: String;
begin
  FromStr := FormatDateTime('yyyy-mm-dd', FromDate);
  ToStr := FormatDateTime('yyyy-mm-dd', ToDate);

  HTML := TStringList.Create;
  try
    HTML.Add(GenerateHTMLHeader('Customer Statement'));

    Q := DB.ExecuteQuery(Format(
      'SELECT o.order_no, o.order_date, o.status, o.total ' +
      'FROM orders o WHERE o.customer_id = %d ' +
      'AND o.order_date BETWEEN ''%s'' AND ''%s'' ' +
      'ORDER BY o.order_date', [CustomerID, FromStr, ToStr]));

    HTML.Add('<table>');
    HTML.Add('<tr><th>Order No</th><th>Date</th><th>Status</th>' +
             '<th class="text-right">Amount</th></tr>');

    var RunningTotal: Double := 0;
    if Assigned(Q) then
    try
      while not Q.EOF do
      begin
        RunningTotal := RunningTotal + Q.FieldByName('total').AsFloat;
        HTML.Add(Format(
          '<tr><td>%s</td><td>%s</td><td>%s</td><td class="text-right">%s</td></tr>',
          [EscapeHTML(Q.FieldByName('order_no').AsString),
           FormatDateTime('dd/mm/yyyy', Q.FieldByName('order_date').AsDateTime),
           EscapeHTML(Q.FieldByName('status').AsString),
           FormatMoney(Q.FieldByName('total').AsFloat)]));
        Q.Next;
      end;
    finally
      Q.Free;
    end;

    HTML.Add(Format(
      '<tr class="total-row"><td colspan="3">Total</td>' +
      '<td class="text-right">%s</td></tr>',
      [FormatMoney(RunningTotal)]));
    HTML.Add('</table>');
    HTML.Add(GenerateHTMLFooter);

    Result := HTML.Text;
  finally
    HTML.Free;
  end;
end;

function TReportGenerator.SaveReportToFile(const HTML, FileName: String): Boolean;
var
  F: TStringList;
begin
  Result := False;
  F := TStringList.Create;
  try
    F.Text := HTML;
    ForceDirectories(ExtractFilePath(FileName));
    F.SaveToFile(FileName, TEncoding.UTF8);
    Result := True;
  finally
    F.Free;
  end;
end;

procedure TReportGenerator.PrintReport(const HTML: String);
var
  TmpFile: String;
begin
  TmpFile := GetTempDir + 'erp_report_' + IntToStr(GetTickCount64) + '.html';
  SaveReportToFile(HTML, TmpFile);

  {$IFDEF WINDOWS}
  ShellExecute(0, 'open', PChar(TmpFile), nil, nil, SW_SHOW);
  {$ELSE}
  if FileExists('/usr/bin/xdg-open') then
    ExecuteProcess('/usr/bin/xdg-open', [TmpFile], [])
  else if FileExists('/usr/bin/open') then
    ExecuteProcess('/usr/bin/open', [TmpFile], []);
  {$ENDIF}
end;

end.
```

---

## 50.7 Main Program

```pascal
{$mode objfpc}{$H+}

// erp_mini.lpr - Main program
program ERP_Mini;

uses
  {$IFDEF UNIX}
  cwstring,
  {$ENDIF}
  Interfaces, // LCL
  Forms,
  frm_main in 'src/forms/frm_main.pas' {MainForm},
  frm_login in 'src/forms/frm_login.pas' {LoginForm},
  DBConnection in 'src/database/db_connection.pas',
  DBSchema in 'src/database/db_schema.pas',
  AuthManager in 'src/utils/auth_manager.pas',
  ModelCustomer in 'src/models/model_customer.pas',
  ModelProduct in 'src/models/model_product.pas',
  ModelOrder in 'src/models/model_order.pas',
  ReportGenerator in 'src/utils/report_generator.pas';

{$R *.res}

begin
  // ตั้งค่า database path
  TDatabase.SetDBPath(ExtractFilePath(ParamStr(0)) + 'data' + PathDelim + 'erp.db');

  RequireDerivedFormResource := True;
  Application.Scaled := True;
  Application.Initialize;
  Application.Title := 'ERP Mini System';

  Application.CreateForm(TMainForm, MainForm);
  Application.CreateForm(TLoginForm, LoginForm);
  Application.Run;

  // Cleanup
  TDatabase.FreeInstance;
  TAuthManager.FreeInstance;
end.
```

---

## 50.8 Build Instructions

### การ Build โปรเจกต์

```bash
#!/bin/bash
# build_erp.sh

echo "Building ERP Mini System..."

# ตรวจสอบ dependencies
check_dep() {
  if ! command -v "$1" &> /dev/null; then
    echo "ERROR: $1 not found"
    exit 1
  fi
}

check_dep lazbuild
check_dep fpc

# สร้าง output directories
mkdir -p dist/linux dist/windows

# Build Linux release
echo "Building Linux release..."
lazbuild --build-mode=Release erp_mini.lpi

if [ $? -eq 0 ]; then
  # Copy to dist
  cp erp_mini dist/linux/
  strip dist/linux/erp_mini
  echo "Linux build: OK ($(du -sh dist/linux/erp_mini | cut -f1))"
else
  echo "Linux build: FAILED"
  exit 1
fi

# Build Windows release (ต้องมี cross-compiler)
if fpc -Twin64 -v 2>&1 | grep -q "Free Pascal"; then
  echo "Building Windows release..."
  lazbuild --build-mode=Release_Win64 erp_mini.lpi
  if [ $? -eq 0 ]; then
    echo "Windows build: OK"
  else
    echo "Windows build: FAILED (skipping)"
  fi
else
  echo "Windows cross-compiler not found, skipping Windows build"
fi

echo ""
echo "Build complete!"
echo "Linux binary: dist/linux/erp_mini"
```

### Lazarus Project File (erp_mini.lpi)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<CONFIG>
  <ProjectOptions>
    <Version Value="12"/>
    <PathDelim Value="/"/>
    <General>
      <Flags>
        <MainUnitHasCreateFormStatements Value="True"/>
        <MainUnitHasTitleStatement Value="True"/>
        <MainUnitHasScaledStatement Value="True"/>
      </Flags>
      <SessionStorage Value="InProjectDir"/>
      <Title Value="ERP Mini System"/>
      <UseAppBundle Value="True"/>
      <ResourceType Value="res"/>
    </General>
    <BuildModes Count="2">
      <Item1 Name="Debug" Default="True"/>
      <Item2 Name="Release"/>
    </BuildModes>
    <Units Count="10">
      <Unit0>
        <Filename Value="erp_mini.lpr"/>
        <IsPartOfProject Value="True"/>
      </Unit0>
      <!-- ... additional units ... -->
    </Units>
  </ProjectOptions>
  <Compiler>
    <Version Value="21"/>
    <CompilerPath Value="$(CompPath)"/>
  </Compiler>
</CONFIG>
```

---

## แบบฝึกหัด 5 ข้อ

### ข้อ 1: เพิ่ม Product Management
สร้าง ProductModel และ form ที่มีฟีเจอร์:
- CRUD สำหรับ products
- Category management
- Stock quantity tracking
- Low stock alerts

### ข้อ 2: ปรับปรุง Order System
เพิ่มฟีเจอร์ให้ Order:
- Print invoice เป็น PDF หรือ HTML
- Email invoice ให้ customer
- Order status workflow ที่สมบูรณ์
- Partial delivery tracking

### ข้อ 3: User Management
สร้างระบบ user management:
- CRUD users
- Role-based access control
- Password reset
- Login audit log

### ข้อ 4: Dashboard
สร้าง dashboard หน้าแรกที่แสดง:
- Today's orders count
- Monthly revenue graph
- Top 5 customers
- Low stock alerts
- Recent activity log

### ข้อ 5: Import/Export
เพิ่มความสามารถ:
- Export customers/products to CSV/Excel
- Import from CSV
- Backup database
- Restore from backup
