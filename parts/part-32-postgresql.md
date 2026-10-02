# ตอนที่ 32: การเชื่อมต่อ PostgreSQL

## บทนำ

PostgreSQL เป็นระบบจัดการฐานข้อมูลเชิงสัมพันธ์แบบ Object-Relational (ORDBMS) ที่มีประสิทธิภาพสูงและรองรับมาตรฐาน SQL อย่างครบครัน PostgreSQL มีคุณสมบัติขั้นสูงหลายอย่างที่ MySQL ไม่มี เช่น JSON data type, Array types, Window functions, และ Full-text search ที่ทรงพลัง

## ลักษณะเด่นของ PostgreSQL

### ความสามารถหลัก
- **ACID Compliance**: รองรับ transactions อย่างสมบูรณ์
- **JSON/JSONB**: เก็บและค้นหาข้อมูล JSON ได้อย่างมีประสิทธิภาพ
- **Array Types**: รองรับ array ในคอลัมน์
- **Advanced Indexing**: B-tree, Hash, GiST, SP-GiST, GIN, BRIN
- **Full-text Search**: ค้นหาข้อความขั้นสูง
- **Table Inheritance**: รองรับการสืบทอดตาราง
- **Extensibility**: สร้าง custom types, functions, operators
- **MVCC**: Multi-Version Concurrency Control

### เปรียบเทียบกับ MySQL
```
Feature             PostgreSQL          MySQL
-----------         ----------          -----
JSON support        JSONB (fast)        JSON (basic)
Array types         ✓                   ✗
Window functions    ✓ (advanced)        ✓ (basic)
Full-text search    ✓ (built-in)        ✓ (MyISAM/InnoDB)
CHECK constraints   ✓ (enforced)        ✓ (MySQL 8.0.16+)
Partial indexes     ✓                   ✗
Materialized views  ✓                   ✗
Extensions          ✓ (PostGIS, etc.)   ✗
Replication         ✓ (logical+physical) ✓
```

---

## การติดตั้ง PostgreSQL Connector สำหรับ Lazarus

### ใช้ pqconnection (SQLdb)

pqconnection มาพร้อมกับ Lazarus ในแพ็คเกจ SQLdb

**การเพิ่ม units ที่จำเป็น:**
```pascal
uses
  pqconnection,  // PostgreSQL connection
  sqldb,         // TQuery, TTransaction
  db;            // TDataSource
```

**การติดตั้ง libpq:**

Linux:
```bash
# Ubuntu/Debian
sudo apt-get install libpq-dev libpq5

# Fedora
sudo dnf install postgresql-devel

# ตรวจสอบ
find /usr -name "libpq.so*"
```

Windows:
```
1. ดาวน์โหลด PostgreSQL จาก postgresql.org
2. คัดลอกไฟล์ libpq.dll และ dependencies ไปไว้ในโฟลเดอร์โปรแกรม
   Files: libpq.dll, libeay32.dll, ssleay32.dll, libiconv-2.dll, libintl-8.dll
```

### ใช้ ZEOS (แนะนำสำหรับ cross-database)

```pascal
uses
  ZConnection,
  ZDataSet,
  ZDbcIntfs;

// กำหนด Protocol
FConnection.Protocol := 'postgresql';
// หรือ
FConnection.Protocol := 'postgresql-9';  // สำหรับ PostgreSQL 9.x
```

---

## Connection Parameters

```pascal
unit PostgreSQLConnect;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, pqconnection, sqldb, db;

type
  TPgConnectionConfig = record
    Host: string;
    Port: Integer;
    Database: string;
    User: string;
    Password: string;
    Schema: string;
    SSLMode: string;
    AppName: string;
    ConnectTimeout: Integer;
  end;

  TPgDatabaseManager = class
  private
    FConnection: TPQConnection;
    FTransaction: TSQLTransaction;
    FConfig: TPgConnectionConfig;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    function Connect(const Config: TPgConnectionConfig): Boolean;
    procedure Disconnect;
    function IsConnected: Boolean;
    function GetServerVersion: string;
    
    // Execute queries
    function ExecuteQuery(const SQL: string): TSQLQuery;
    function ExecuteNonQuery(const SQL: string): Integer;
    
    property Connection: TPQConnection read FConnection;
    property Transaction: TSQLTransaction read FTransaction;
  end;

implementation

constructor TPgDatabaseManager.Create;
begin
  inherited Create;
  FConnection := TPQConnection.Create(nil);
  FTransaction := TSQLTransaction.Create(nil);
  FTransaction.DataBase := FConnection;
end;

destructor TPgDatabaseManager.Destroy;
begin
  Disconnect;
  FTransaction.Free;
  FConnection.Free;
  inherited Destroy;
end;

function TPgDatabaseManager.Connect(const Config: TPgConnectionConfig): Boolean;
begin
  Result := False;
  FConfig := Config;
  
  try
    FConnection.HostName := Config.Host;
    FConnection.DatabaseName := Config.Database;
    FConnection.UserName := Config.User;
    FConnection.Password := Config.Password;
    FConnection.CharSet := 'UTF8';
    
    // PostgreSQL specific params
    if Config.Port <> 0 then
      FConnection.Params.Values['port'] := IntToStr(Config.Port);
      
    if Config.Schema <> '' then
      FConnection.Params.Values['options'] := '-c search_path=' + Config.Schema;
      
    if Config.SSLMode <> '' then
      FConnection.Params.Values['sslmode'] := Config.SSLMode;
      
    if Config.AppName <> '' then
      FConnection.Params.Values['application_name'] := Config.AppName;
      
    if Config.ConnectTimeout > 0 then
      FConnection.Params.Values['connect_timeout'] := IntToStr(Config.ConnectTimeout);
    
    FConnection.Connected := True;
    
    // ตั้ง search_path ถ้ามี schema
    if Config.Schema <> '' then
    begin
      var Q := TSQLQuery.Create(nil);
      try
        Q.DataBase := FConnection;
        Q.Transaction := FTransaction;
        Q.SQL.Text := 'SET search_path TO ' + Config.Schema + ', public';
        FTransaction.StartTransaction;
        Q.ExecSQL;
        FTransaction.Commit;
      finally
        Q.Free;
      end;
    end;
    
    Result := True;
    WriteLn('Connected to PostgreSQL: ' + GetServerVersion);
    
  except
    on E: Exception do
    begin
      WriteLn('Connection failed: ' + E.Message);
      Result := False;
    end;
  end;
end;

procedure TPgDatabaseManager.Disconnect;
begin
  if Assigned(FTransaction) and FTransaction.Active then
    FTransaction.Rollback;
  if Assigned(FConnection) and FConnection.Connected then
    FConnection.Connected := False;
end;

function TPgDatabaseManager.IsConnected: Boolean;
begin
  Result := Assigned(FConnection) and FConnection.Connected;
end;

function TPgDatabaseManager.GetServerVersion: string;
var
  Q: TSQLQuery;
begin
  Result := 'Unknown';
  Q := TSQLQuery.Create(nil);
  try
    Q.DataBase := FConnection;
    Q.Transaction := FTransaction;
    Q.SQL.Text := 'SELECT version()';
    FTransaction.StartTransaction;
    Q.Open;
    if not Q.EOF then
      Result := Q.Fields[0].AsString;
    Q.Close;
    FTransaction.Commit;
  except
    FTransaction.Rollback;
  end;
  Q.Free;
end;

function TPgDatabaseManager.ExecuteQuery(const SQL: string): TSQLQuery;
var
  Q: TSQLQuery;
begin
  Q := TSQLQuery.Create(nil);
  Q.DataBase := FConnection;
  Q.Transaction := FTransaction;
  Q.SQL.Text := SQL;
  
  try
    FTransaction.StartTransaction;
    Q.Open;
    FTransaction.Commit;
    Result := Q;
  except
    on E: Exception do
    begin
      FTransaction.Rollback;
      Q.Free;
      raise;
    end;
  end;
end;

function TPgDatabaseManager.ExecuteNonQuery(const SQL: string): Integer;
var
  Q: TSQLQuery;
begin
  Result := -1;
  Q := TSQLQuery.Create(nil);
  try
    Q.DataBase := FConnection;
    Q.Transaction := FTransaction;
    Q.SQL.Text := SQL;
    
    FTransaction.StartTransaction;
    Q.ExecSQL;
    Result := Q.RowsAffected;
    FTransaction.Commit;
    
  except
    on E: Exception do
    begin
      FTransaction.Rollback;
      raise;
    end;
  finally
    Q.Free;
  end;
end;

end.
```

---

## Basic Queries

```pascal
unit PostgreSQLQueries;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sqldb, db;

type
  TPostgreSQLQueryExamples = class
  private
    FDB: TPgDatabaseManager;
  public
    constructor Create(ADB: TPgDatabaseManager);
    
    procedure BasicCRUD;
    procedure UsingParameters;
    procedure WorkingWithNULL;
    procedure DateTimeHandling;
    procedure BulkInsert;
  end;

implementation

constructor TPostgreSQLQueryExamples.Create(ADB: TPgDatabaseManager);
begin
  FDB := ADB;
end;

procedure TPostgreSQLQueryExamples.BasicCRUD;
var
  Q: TSQLQuery;
begin
  // CREATE TABLE
  FDB.ExecuteNonQuery(
    'CREATE TABLE IF NOT EXISTS products (' +
    '  product_id SERIAL PRIMARY KEY,' +
    '  sku VARCHAR(50) UNIQUE NOT NULL,' +
    '  name VARCHAR(200) NOT NULL,' +
    '  description TEXT,' +
    '  price NUMERIC(10,2) NOT NULL CHECK (price >= 0),' +
    '  stock_qty INTEGER DEFAULT 0,' +
    '  category_id INTEGER,' +
    '  is_active BOOLEAN DEFAULT TRUE,' +
    '  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP' +
    ')'
  );
  
  // INSERT
  FDB.ExecuteNonQuery(
    'INSERT INTO products (sku, name, price, stock_qty) ' +
    'VALUES (''LAPTOP-001'', ''โน้ตบุ๊ค Asus VivoBook'', 25990.00, 50)'
  );
  
  // SELECT
  Q := FDB.ExecuteQuery(
    'SELECT product_id, sku, name, price, stock_qty ' +
    'FROM products ' +
    'ORDER BY name'
  );
  
  WriteLn('รายการสินค้า:');
  while not Q.EOF do
  begin
    WriteLn(Format('  [%s] %s - ราคา %.2f บาท (คงเหลือ %d ชิ้น)',
      [Q.FieldByName('sku').AsString,
       Q.FieldByName('name').AsString,
       Q.FieldByName('price').AsFloat,
       Q.FieldByName('stock_qty').AsInteger]));
    Q.Next;
  end;
  Q.Free;
  
  // UPDATE
  FDB.ExecuteNonQuery(
    'UPDATE products SET price = 24990.00 WHERE sku = ''LAPTOP-001'''
  );
  
  // DELETE
  FDB.ExecuteNonQuery(
    'DELETE FROM products WHERE is_active = FALSE'
  );
end;

procedure TPostgreSQLQueryExamples.UsingParameters;
var
  Q: TSQLQuery;
begin
  Q := TSQLQuery.Create(nil);
  try
    Q.DataBase := FDB.Connection;
    Q.Transaction := FDB.Transaction;
    
    // PostgreSQL ใช้ $1, $2 สำหรับ parameters (SQLdb)
    Q.SQL.Text := 
      'SELECT p.product_id, p.name, p.price, c.name AS category_name ' +
      'FROM products p ' +
      'LEFT JOIN categories c ON p.category_id = c.category_id ' +
      'WHERE p.price BETWEEN :min_price AND :max_price ' +
      '  AND p.is_active = :is_active ' +
      'ORDER BY p.price';
    
    Q.ParamByName('min_price').AsFloat := 1000;
    Q.ParamByName('max_price').AsFloat := 50000;
    Q.ParamByName('is_active').AsBoolean := True;
    
    FDB.Transaction.StartTransaction;
    Q.Open;
    
    while not Q.EOF do
    begin
      WriteLn(Format('%s - %.2f บาท (%s)',
        [Q.FieldByName('name').AsString,
         Q.FieldByName('price').AsFloat,
         Q.FieldByName('category_name').AsString]));
      Q.Next;
    end;
    
    Q.Close;
    FDB.Transaction.Commit;
    
  finally
    Q.Free;
  end;
end;

procedure TPostgreSQLQueryExamples.DateTimeHandling;
var
  Q: TSQLQuery;
begin
  Q := TSQLQuery.Create(nil);
  try
    Q.DataBase := FDB.Connection;
    Q.Transaction := FDB.Transaction;
    
    // PostgreSQL date/time functions
    Q.SQL.Text := 
      'SELECT ' +
      '  order_id, ' +
      '  customer_name, ' +
      '  order_date, ' +
      '  order_date::DATE AS date_only, ' +
      '  EXTRACT(YEAR FROM order_date) AS order_year, ' +
      '  EXTRACT(MONTH FROM order_date) AS order_month, ' +
      '  AGE(CURRENT_DATE, order_date::DATE) AS days_ago, ' +
      '  TO_CHAR(order_date, ''DD Mon YYYY HH24:MI'') AS formatted_date ' +
      'FROM orders ' +
      'WHERE order_date >= CURRENT_DATE - INTERVAL ''30 days'' ' +
      'ORDER BY order_date DESC';
    
    FDB.Transaction.StartTransaction;
    Q.Open;
    
    while not Q.EOF do
    begin
      WriteLn(Format('Order #%d - %s - %s',
        [Q.FieldByName('order_id').AsInteger,
         Q.FieldByName('customer_name').AsString,
         Q.FieldByName('formatted_date').AsString]));
      Q.Next;
    end;
    
    Q.Close;
    FDB.Transaction.Commit;
    
  finally
    Q.Free;
  end;
end;

procedure TPostgreSQLQueryExamples.BulkInsert;
var
  Q: TSQLQuery;
  i: Integer;
  InsertValues: TStringList;
begin
  // Bulk insert ด้วย VALUES list
  InsertValues := TStringList.Create;
  
  try
    for i := 1 to 100 do
      InsertValues.Add(Format('(''PROD-%04d'', ''สินค้า %d'', %.2f)',
        [i, i, Random * 10000]));
    
    var SQL := 
      'INSERT INTO products (sku, name, price) VALUES ' +
      InsertValues.CommaText;
    
    var Affected := FDB.ExecuteNonQuery(SQL);
    WriteLn(Format('Insert %d รายการสำเร็จ', [Affected]));
    
  finally
    InsertValues.Free;
  end;
  
  // หรือใช้ COPY command (เร็วกว่ามาก)
  // เหมาะสำหรับข้อมูลปริมาณมาก
end;

end.
```

---

## Advanced SQL: Joins, Subqueries, Window Functions

```pascal
unit AdvancedSQL;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sqldb;

procedure DemonstrateJoins(DB: TPgDatabaseManager);
procedure DemonstrateSubqueries(DB: TPgDatabaseManager);
procedure DemonstrateWindowFunctions(DB: TPgDatabaseManager);
procedure DemonstrateCTE(DB: TPgDatabaseManager);

implementation

procedure DemonstrateJoins(DB: TPgDatabaseManager);
var
  Q: TSQLQuery;
begin
  Q := DB.ExecuteQuery(
    '-- INNER JOIN: เฉพาะที่มีคู่กัน' + #13#10 +
    'SELECT ' +
    '  o.order_id, ' +
    '  c.customer_name, ' +
    '  c.email, ' +
    '  o.total_amount, ' +
    '  o.status, ' +
    '  s.company_name AS shipping_company, ' +
    '  o.tracking_number ' +
    'FROM orders o ' +
    'INNER JOIN customers c ON o.customer_id = c.customer_id ' +
    'LEFT JOIN shipments s ON o.order_id = s.order_id ' +  -- LEFT JOIN: รวมคำสั่งที่ยังไม่จัดส่ง
    'WHERE o.status IN (''confirmed'', ''shipped'') ' +
    'ORDER BY o.order_id DESC ' +
    'LIMIT 50'
  );
  
  WriteLn('คำสั่งซื้อล่าสุด:');
  while not Q.EOF do
  begin
    var ShipInfo := Q.FieldByName('shipping_company').AsString;
    if ShipInfo = '' then
      ShipInfo := 'ยังไม่จัดส่ง';
    
    WriteLn(Format('#%d - %s - %.2f บาท - %s',
      [Q.FieldByName('order_id').AsInteger,
       Q.FieldByName('customer_name').AsString,
       Q.FieldByName('total_amount').AsFloat,
       ShipInfo]));
    Q.Next;
  end;
  Q.Free;
end;

procedure DemonstrateSubqueries(DB: TPgDatabaseManager);
var
  Q: TSQLQuery;
begin
  // Correlated subquery - หาลูกค้าที่มีคำสั่งซื้อมากกว่าค่าเฉลี่ย
  Q := DB.ExecuteQuery(
    'SELECT ' +
    '  c.customer_id, ' +
    '  c.customer_name, ' +
    '  (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id) AS order_count, ' +
    '  (SELECT SUM(total_amount) FROM orders o WHERE o.customer_id = c.customer_id) AS total_spent ' +
    'FROM customers c ' +
    'WHERE ( ' +
    '  SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.customer_id ' +
    ') > ( ' +
    '  SELECT AVG(order_count) FROM (' +
    '    SELECT COUNT(*) AS order_count FROM orders GROUP BY customer_id' +
    '  ) sub ' +
    ') ' +
    'ORDER BY total_spent DESC'
  );
  
  WriteLn('ลูกค้าที่มียอดสั่งซื้อมากกว่าค่าเฉลี่ย:');
  while not Q.EOF do
  begin
    WriteLn(Format('%s - %d คำสั่ง - รวม %.2f บาท',
      [Q.FieldByName('customer_name').AsString,
       Q.FieldByName('order_count').AsInteger,
       Q.FieldByName('total_spent').AsFloat]));
    Q.Next;
  end;
  Q.Free;
  
  // IN subquery
  Q := DB.ExecuteQuery(
    'SELECT product_id, name, price ' +
    'FROM products ' +
    'WHERE product_id IN (' +
    '  SELECT DISTINCT oi.product_id ' +
    '  FROM order_items oi ' +
    '  INNER JOIN orders o ON oi.order_id = o.order_id ' +
    '  WHERE o.order_date >= CURRENT_DATE - INTERVAL ''7 days''' +
    ') ' +
    'ORDER BY name'
  );
  
  WriteLn('สินค้าที่ขายใน 7 วันที่ผ่านมา:');
  while not Q.EOF do
  begin
    WriteLn(Format('  %s (%.2f บาท)', 
      [Q.FieldByName('name').AsString,
       Q.FieldByName('price').AsFloat]));
    Q.Next;
  end;
  Q.Free;
end;

procedure DemonstrateWindowFunctions(DB: TPgDatabaseManager);
var
  Q: TSQLQuery;
begin
  // Window Functions - PostgreSQL's powerful feature
  Q := DB.ExecuteQuery(
    'SELECT ' +
    '  product_id, ' +
    '  name, ' +
    '  category_name, ' +
    '  price, ' +
    '  -- Ranking within category' +
    '  RANK() OVER (PARTITION BY category_id ORDER BY price DESC) AS price_rank, ' +
    '  -- Running total' +
    '  SUM(price) OVER (PARTITION BY category_id ORDER BY price ' +
    '    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total, ' +
    '  -- Moving average (last 3 products by price)' +
    '  AVG(price) OVER (PARTITION BY category_id ORDER BY price ' +
    '    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW) AS moving_avg, ' +
    '  -- Percentile' +
    '  PERCENT_RANK() OVER (PARTITION BY category_id ORDER BY price) AS percentile, ' +
    '  -- Lead/Lag' +
    '  LAG(price) OVER (PARTITION BY category_id ORDER BY price) AS prev_price, ' +
    '  LEAD(price) OVER (PARTITION BY category_id ORDER BY price) AS next_price ' +
    'FROM products p ' +
    'JOIN categories c ON p.category_id = c.category_id ' +
    'ORDER BY category_name, price DESC'
  );
  
  WriteLn('Window Function Analysis:');
  WriteLn(Format('%-30s %-15s %10s %6s %10s',
    ['ชื่อสินค้า', 'หมวดหมู่', 'ราคา', 'อันดับ', 'เปอร์เซ็นต์ไทล์']));
  WriteLn(StringOfChar('-', 80));
  
  while not Q.EOF do
  begin
    WriteLn(Format('%-30s %-15s %10.2f %6d %10.1f%%',
      [Q.FieldByName('name').AsString,
       Q.FieldByName('category_name').AsString,
       Q.FieldByName('price').AsFloat,
       Q.FieldByName('price_rank').AsInteger,
       Q.FieldByName('percentile').AsFloat * 100]));
    Q.Next;
  end;
  Q.Free;
end;

procedure DemonstrateCTE(DB: TPgDatabaseManager);
var
  Q: TSQLQuery;
begin
  // Common Table Expressions (CTE) - อ่านง่ายและ reusable
  Q := DB.ExecuteQuery(
    'WITH monthly_sales AS (' +
    '  SELECT ' +
    '    DATE_TRUNC(''month'', order_date) AS month, ' +
    '    SUM(total_amount) AS total_sales, ' +
    '    COUNT(*) AS order_count ' +
    '  FROM orders ' +
    '  WHERE status = ''completed'' ' +
    '  GROUP BY DATE_TRUNC(''month'', order_date) ' +
    '), ' +
    'sales_growth AS (' +
    '  SELECT ' +
    '    month, ' +
    '    total_sales, ' +
    '    order_count, ' +
    '    LAG(total_sales) OVER (ORDER BY month) AS prev_month_sales, ' +
    '    (total_sales - LAG(total_sales) OVER (ORDER BY month)) / ' +
    '      NULLIF(LAG(total_sales) OVER (ORDER BY month), 0) * 100 AS growth_pct ' +
    '  FROM monthly_sales ' +
    ') ' +
    'SELECT ' +
    '  TO_CHAR(month, ''Mon YYYY'') AS period, ' +
    '  total_sales, ' +
    '  order_count, ' +
    '  COALESCE(growth_pct, 0) AS growth_percentage ' +
    'FROM sales_growth ' +
    'ORDER BY month DESC ' +
    'LIMIT 12'
  );
  
  WriteLn('ยอดขายรายเดือน (12 เดือนล่าสุด):');
  WriteLn(Format('%-12s %15s %10s %12s', ['เดือน', 'ยอดขาย', 'คำสั่ง', 'เติบโต%']));
  WriteLn(StringOfChar('-', 55));
  
  while not Q.EOF do
  begin
    var GrowthStr := Format('%+.1f%%', [Q.FieldByName('growth_percentage').AsFloat]);
    WriteLn(Format('%-12s %15.2f %10d %12s',
      [Q.FieldByName('period').AsString,
       Q.FieldByName('total_sales').AsFloat,
       Q.FieldByName('order_count').AsInteger,
       GrowthStr]));
    Q.Next;
  end;
  Q.Free;
  
  // Recursive CTE - สำหรับ hierarchical data
  Q := DB.ExecuteQuery(
    'WITH RECURSIVE category_tree AS (' +
    '  -- Base case: root categories' +
    '  SELECT ' +
    '    category_id, parent_id, name, ' +
    '    0 AS depth, ' +
    '    name AS full_path ' +
    '  FROM categories ' +
    '  WHERE parent_id IS NULL ' +
    '  UNION ALL ' +
    '  -- Recursive case' +
    '  SELECT ' +
    '    c.category_id, c.parent_id, c.name, ' +
    '    ct.depth + 1, ' +
    '    ct.full_path || '' > '' || c.name ' +
    '  FROM categories c ' +
    '  INNER JOIN category_tree ct ON c.parent_id = ct.category_id ' +
    ') ' +
    'SELECT ' +
    '  category_id, depth, ' +
    '  REPEAT(''  '', depth) || name AS indented_name, ' +
    '  full_path ' +
    'FROM category_tree ' +
    'ORDER BY full_path'
  );
  
  WriteLn('โครงสร้างหมวดหมู่สินค้า:');
  while not Q.EOF do
  begin
    WriteLn(Q.FieldByName('indented_name').AsString);
    Q.Next;
  end;
  Q.Free;
end;

end.
```

---

## PostgreSQL-Specific Features: Sequences, Schemas, Roles

```pascal
unit PostgreSQLFeatures;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sqldb;

type
  TPostgreSQLAdmin = class
  private
    FDB: TPgDatabaseManager;
  public
    constructor Create(ADB: TPgDatabaseManager);
    
    procedure ManageSequences;
    procedure ManageSchemas;
    procedure ManageRoles;
    procedure DemonstrateTablespaces;
  end;

implementation

constructor TPostgreSQLAdmin.Create(ADB: TPgDatabaseManager);
begin
  FDB := ADB;
end;

procedure TPostgreSQLAdmin.ManageSequences;
begin
  // สร้าง sequence
  FDB.ExecuteNonQuery(
    'CREATE SEQUENCE IF NOT EXISTS order_number_seq ' +
    '  START WITH 10000 ' +
    '  INCREMENT BY 1 ' +
    '  MINVALUE 10000 ' +
    '  MAXVALUE 9999999 ' +
    '  CACHE 100'
  );
  
  // ใช้งาน sequence ใน query
  var Q := FDB.ExecuteQuery(
    'SELECT ' +
    '  NEXTVAL(''order_number_seq'') AS next_order_num, ' +
    '  CURRVAL(''order_number_seq'') AS current_val, ' +
    '  LASTVAL() AS last_generated'
  );
  WriteLn('Next Order Number: ' + Q.FieldByName('next_order_num').AsString);
  Q.Free;
  
  // ดู sequence ทั้งหมด
  Q := FDB.ExecuteQuery(
    'SELECT sequence_name, start_value, increment_by, max_value, ' +
    '       last_value ' +
    'FROM information_schema.sequences s ' +
    'LEFT JOIN pg_sequences ps ON s.sequence_name = ps.sequencename ' +
    'WHERE sequence_schema = current_schema()'
  );
  
  WriteLn('Sequences:');
  while not Q.EOF do
  begin
    WriteLn(Format('  %s: current=%s',
      [Q.FieldByName('sequence_name').AsString,
       Q.FieldByName('last_value').AsString]));
    Q.Next;
  end;
  Q.Free;
end;

procedure TPostgreSQLAdmin.ManageSchemas;
begin
  // สร้าง schema สำหรับแต่ละ module
  FDB.ExecuteNonQuery('CREATE SCHEMA IF NOT EXISTS ecommerce');
  FDB.ExecuteNonQuery('CREATE SCHEMA IF NOT EXISTS inventory');
  FDB.ExecuteNonQuery('CREATE SCHEMA IF NOT EXISTS hr');
  
  // สร้างตารางใน schema
  FDB.ExecuteNonQuery(
    'CREATE TABLE IF NOT EXISTS ecommerce.customers (' +
    '  customer_id SERIAL PRIMARY KEY,' +
    '  customer_code VARCHAR(20) UNIQUE NOT NULL,' +
    '  full_name VARCHAR(100) NOT NULL,' +
    '  email VARCHAR(100) UNIQUE,' +
    '  phone VARCHAR(20),' +
    '  tier VARCHAR(20) DEFAULT ''bronze''' +
    ')'
  );
  
  // ค้นหาตารางใน schema
  var Q := FDB.ExecuteQuery(
    'SELECT table_schema, table_name, ' +
    '       pg_size_pretty(pg_total_relation_size(' +
    '         quote_ident(table_schema) || ''.'' || quote_ident(table_name)' +
    '       )) AS table_size ' +
    'FROM information_schema.tables ' +
    'WHERE table_schema NOT IN (''information_schema'', ''pg_catalog'') ' +
    '  AND table_type = ''BASE TABLE'' ' +
    'ORDER BY table_schema, table_name'
  );
  
  WriteLn('Tables by Schema:');
  var CurrentSchema := '';
  while not Q.EOF do
  begin
    var Schema := Q.FieldByName('table_schema').AsString;
    if Schema <> CurrentSchema then
    begin
      WriteLn('Schema: ' + Schema);
      CurrentSchema := Schema;
    end;
    WriteLn(Format('  %s (%s)',
      [Q.FieldByName('table_name').AsString,
       Q.FieldByName('table_size').AsString]));
    Q.Next;
  end;
  Q.Free;
end;

procedure TPostgreSQLAdmin.ManageRoles;
begin
  // สร้าง role สำหรับแต่ละ access level
  FDB.ExecuteNonQuery(
    'DO $$ ' +
    'BEGIN ' +
    '  IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = ''app_readonly'') THEN ' +
    '    CREATE ROLE app_readonly; ' +
    '  END IF; ' +
    '  IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = ''app_readwrite'') THEN ' +
    '    CREATE ROLE app_readwrite; ' +
    '  END IF; ' +
    'END $$'
  );
  
  // กำหนด privileges
  FDB.ExecuteNonQuery('GRANT CONNECT ON DATABASE ecommerce_db TO app_readonly');
  FDB.ExecuteNonQuery('GRANT USAGE ON SCHEMA public TO app_readonly');
  FDB.ExecuteNonQuery('GRANT SELECT ON ALL TABLES IN SCHEMA public TO app_readonly');
  
  FDB.ExecuteNonQuery('GRANT CONNECT ON DATABASE ecommerce_db TO app_readwrite');
  FDB.ExecuteNonQuery('GRANT USAGE ON SCHEMA public TO app_readwrite');
  FDB.ExecuteNonQuery('GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO app_readwrite');
  FDB.ExecuteNonQuery('GRANT USAGE ON ALL SEQUENCES IN SCHEMA public TO app_readwrite');
  
  // สร้างผู้ใช้และกำหนด role
  FDB.ExecuteNonQuery(
    'DO $$ ' +
    'BEGIN ' +
    '  IF NOT EXISTS (SELECT FROM pg_catalog.pg_roles WHERE rolname = ''app_user'') THEN ' +
    '    CREATE USER app_user WITH PASSWORD ''secure_password123'' IN ROLE app_readwrite; ' +
    '  END IF; ' +
    'END $$'
  );
  
  // ดู roles และ privileges
  var Q := FDB.ExecuteQuery(
    'SELECT r.rolname, r.rolsuper, r.rolinherit, r.rolcreaterole, ' +
    '       r.rolcreatedb, r.rolcanlogin, r.rolconnlimit, ' +
    '       ARRAY(SELECT b.rolname FROM pg_catalog.pg_auth_members m ' +
    '             JOIN pg_catalog.pg_roles b ON m.roleid = b.oid ' +
    '             WHERE m.member = r.oid) AS member_of ' +
    'FROM pg_catalog.pg_roles r ' +
    'WHERE r.rolname NOT LIKE ''pg_%'' ' +
    'ORDER BY r.rolname'
  );
  
  WriteLn('Database Roles:');
  while not Q.EOF do
  begin
    WriteLn(Format('  %s (login=%s, super=%s)',
      [Q.FieldByName('rolname').AsString,
       BoolToStr(Q.FieldByName('rolcanlogin').AsBoolean, 'yes', 'no'),
       BoolToStr(Q.FieldByName('rolsuper').AsBoolean, 'yes', 'no')]));
    Q.Next;
  end;
  Q.Free;
end;

end.
```

---

## JSON Data Type

```pascal
unit PostgreSQLJSON;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sqldb, fpjson, jsonparser;

type
  TJSONExamples = class
  private
    FDB: TPgDatabaseManager;
  public
    constructor Create(ADB: TPgDatabaseManager);
    
    procedure CreateJSONTable;
    procedure InsertJSONData;
    procedure QueryJSONData;
    procedure UpdateJSONData;
    procedure JSONAggregation;
  end;

implementation

constructor TJSONExamples.Create(ADB: TPgDatabaseManager);
begin
  FDB := ADB;
end;

procedure TJSONExamples.CreateJSONTable;
begin
  FDB.ExecuteNonQuery(
    'CREATE TABLE IF NOT EXISTS products_json (' +
    '  product_id SERIAL PRIMARY KEY,' +
    '  name VARCHAR(200) NOT NULL,' +
    '  price NUMERIC(10,2),' +
    '  attributes JSONB,' +  // JSONB เร็วกว่า JSON สำหรับการค้นหา
    '  tags TEXT[]' +
    ')'
  );
  
  // สร้าง index บน JSONB
  FDB.ExecuteNonQuery(
    'CREATE INDEX IF NOT EXISTS idx_products_attributes ' +
    'ON products_json USING GIN (attributes)'
  );
end;

procedure TJSONExamples.InsertJSONData;
begin
  // Insert ด้วย JSON
  FDB.ExecuteNonQuery(
    'INSERT INTO products_json (name, price, attributes, tags) VALUES ' +
    '(''โน้ตบุ๊ค Asus'', 25990.00, ' +
    ' ''{"brand": "Asus", "specs": {"ram": "16GB", "storage": "512GB SSD", ' +
    '   "display": "15.6 inch FHD"}, "warranty_years": 2}'', ' +
    ' ARRAY[''laptop'', ''asus'', ''gaming'']),' +
    
    '(''แป้นพิมพ์ Mechanical'', 2990.00, ' +
    ' ''{"brand": "Keychron", "type": "mechanical", ' +
    '   "switch": "Cherry MX Red", "backlight": "RGB"}'', ' +
    ' ARRAY[''keyboard'', ''mechanical'', ''gaming''])'
  );
end;

procedure TJSONExamples.QueryJSONData;
var
  Q: TSQLQuery;
begin
  // ค้นหาด้วย JSON operators
  Q := FDB.ExecuteQuery(
    'SELECT ' +
    '  product_id, ' +
    '  name, ' +
    '  price, ' +
    '  attributes->>''brand'' AS brand, ' +          // ดึงค่าเป็น text
    '  attributes->''specs'' AS specs_json, ' +       // ดึงค่าเป็น JSON
    '  attributes->''specs''->>''ram'' AS ram, ' +    // nested access
    '  (attributes->>''warranty_years'')::INTEGER AS warranty ' +
    'FROM products_json ' +
    'WHERE attributes->>''brand'' = ''Asus''' +       // filter by JSON value
  );
  
  WriteLn('Asus Products:');
  while not Q.EOF do
  begin
    WriteLn(Format('%s (RAM: %s, Warranty: %d years)',
      [Q.FieldByName('name').AsString,
       Q.FieldByName('ram').AsString,
       Q.FieldByName('warranty').AsInteger]));
    Q.Next;
  end;
  Q.Free;
  
  // ค้นหาด้วย @> (contains)
  Q := FDB.ExecuteQuery(
    'SELECT name, attributes ' +
    'FROM products_json ' +
    'WHERE attributes @> ''{"type": "mechanical"}'''
  );
  
  WriteLn('Mechanical Products:');
  while not Q.EOF do
  begin
    WriteLn(Q.FieldByName('name').AsString);
    Q.Next;
  end;
  Q.Free;
  
  // ค้นหาด้วย Array
  Q := FDB.ExecuteQuery(
    'SELECT name, tags ' +
    'FROM products_json ' +
    'WHERE tags && ARRAY[''gaming'']'  // array overlap
  );
  
  WriteLn('Gaming Products:');
  while not Q.EOF do
  begin
    WriteLn(Q.FieldByName('name').AsString);
    Q.Next;
  end;
  Q.Free;
end;

procedure TJSONExamples.UpdateJSONData;
begin
  // อัปเดต JSON field
  FDB.ExecuteNonQuery(
    'UPDATE products_json ' +
    'SET attributes = attributes || ''{"on_sale": true, "discount_pct": 10}'' ' +
    'WHERE product_id = 1'
  );
  
  // ลบ key จาก JSON
  FDB.ExecuteNonQuery(
    'UPDATE products_json ' +
    'SET attributes = attributes - ''on_sale'' ' +
    'WHERE product_id = 1'
  );
  
  // อัปเดต nested JSON
  FDB.ExecuteNonQuery(
    'UPDATE products_json ' +
    'SET attributes = jsonb_set(attributes, ''{specs, ram}'', ''"32GB"'') ' +
    'WHERE product_id = 1'
  );
end;

procedure TJSONExamples.JSONAggregation;
var
  Q: TSQLQuery;
begin
  // Aggregate JSON
  Q := FDB.ExecuteQuery(
    'SELECT ' +
    '  attributes->>''brand'' AS brand, ' +
    '  COUNT(*) AS product_count, ' +
    '  AVG(price) AS avg_price, ' +
    '  json_agg(json_build_object(' +
    '    ''id'', product_id, ' +
    '    ''name'', name, ' +
    '    ''price'', price' +
    '  )) AS products_json ' +
    'FROM products_json ' +
    'GROUP BY attributes->>''brand'''
  );
  
  while not Q.EOF do
  begin
    WriteLn(Format('Brand: %s, Products: %d, Avg Price: %.2f',
      [Q.FieldByName('brand').AsString,
       Q.FieldByName('product_count').AsInteger,
       Q.FieldByName('avg_price').AsFloat]));
    
    // Parse JSON result
    var JSONStr := Q.FieldByName('products_json').AsString;
    var JData := GetJSON(JSONStr) as TJSONArray;
    if Assigned(JData) then
    begin
      for var i := 0 to JData.Count - 1 do
      begin
        var JObj := JData[i] as TJSONObject;
        WriteLn(Format('  - %s: %.2f',
          [JObj.Get('name', ''),
           JObj.Get('price', 0.0)]));
      end;
      JData.Free;
    end;
    
    Q.Next;
  end;
  Q.Free;
end;

end.
```

---

## Array Data Type

```pascal
unit PostgreSQLArrays;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sqldb;

procedure ArrayExamples(DB: TPgDatabaseManager);

implementation

procedure ArrayExamples(DB: TPgDatabaseManager);
var
  Q: TSQLQuery;
begin
  // สร้างตารางที่มี array columns
  DB.ExecuteNonQuery(
    'CREATE TABLE IF NOT EXISTS employees (' +
    '  emp_id SERIAL PRIMARY KEY,' +
    '  name VARCHAR(100) NOT NULL,' +
    '  skills TEXT[],' +         // array ของ text
    '  scores INTEGER[],' +      // array ของ integer
    '  schedule BOOLEAN[7]' +   // array 7 วันสำหรับตาราง
    ')'
  );
  
  // Insert ด้วย array
  DB.ExecuteNonQuery(
    'INSERT INTO employees (name, skills, scores, schedule) VALUES ' +
    '(''สมชาย ใจดี'', ARRAY[''Pascal'', ''Python'', ''SQL''], ' +
    ' ARRAY[95, 88, 92], ' +
    ' ARRAY[true, true, true, true, true, false, false])' +  // จันทร์-ศุกร์ทำงาน
  );
  
  // Query arrays
  Q := DB.ExecuteQuery(
    'SELECT ' +
    '  name, ' +
    '  skills, ' +
    '  skills[1] AS first_skill, ' +   // index เริ่มที่ 1
    '  array_length(skills, 1) AS skill_count, ' +
    '  scores, ' +
    '  (SELECT AVG(s) FROM UNNEST(scores) s) AS avg_score ' +
    'FROM employees'
  );
  
  while not Q.EOF do
  begin
    WriteLn(Format('%s - Skills: %s, Avg Score: %.1f',
      [Q.FieldByName('name').AsString,
       Q.FieldByName('skills').AsString,
       Q.FieldByName('avg_score').AsFloat]));
    Q.Next;
  end;
  Q.Free;
  
  // ค้นหาด้วย array operators
  Q := DB.ExecuteQuery(
    'SELECT name, skills ' +
    'FROM employees ' +
    'WHERE skills @> ARRAY[''Pascal'']'  // contains Pascal
  );
  WriteLn('Pascal developers:');
  while not Q.EOF do
  begin
    WriteLn('  ' + Q.FieldByName('name').AsString);
    Q.Next;
  end;
  Q.Free;
  
  // UNNEST - แปลง array เป็น rows
  Q := DB.ExecuteQuery(
    'SELECT name, UNNEST(skills) AS skill ' +
    'FROM employees ' +
    'ORDER BY name, skill'
  );
  WriteLn('All skills (unnested):');
  while not Q.EOF do
  begin
    WriteLn(Format('  %s: %s',
      [Q.FieldByName('name').AsString,
       Q.FieldByName('skill').AsString]));
    Q.Next;
  end;
  Q.Free;
  
  // UPDATE array
  DB.ExecuteNonQuery(
    'UPDATE employees ' +
    'SET skills = skills || ARRAY[''Lazarus''] ' +  // append
    'WHERE name = ''สมชาย ใจดี'''
  );
  
  DB.ExecuteNonQuery(
    'UPDATE employees ' +
    'SET skills = ARRAY_REMOVE(skills, ''Python'') ' +  // remove element
    'WHERE name = ''สมชาย ใจดี'''
  );
end;

end.
```

---

## Full-text Search

```pascal
unit PostgreSQLFullText;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sqldb;

type
  TFullTextSearch = class
  private
    FDB: TPgDatabaseManager;
  public
    constructor Create(ADB: TPgDatabaseManager);
    
    procedure SetupFullTextSearch;
    procedure SearchProducts(const SearchTerm: string);
    procedure SearchWithRanking(const SearchTerm: string);
    procedure SearchThaiText(const SearchTerm: string);
  end;

implementation

constructor TFullTextSearch.Create(ADB: TPgDatabaseManager);
begin
  FDB := ADB;
end;

procedure TFullTextSearch.SetupFullTextSearch;
begin
  // เพิ่ม tsvector column สำหรับ full-text search
  FDB.ExecuteNonQuery(
    'ALTER TABLE products ' +
    'ADD COLUMN IF NOT EXISTS search_vector tsvector'
  );
  
  // สร้าง GIN index สำหรับ full-text search
  FDB.ExecuteNonQuery(
    'CREATE INDEX IF NOT EXISTS idx_products_search ' +
    'ON products USING GIN (search_vector)'
  );
  
  // อัปเดต search_vector
  FDB.ExecuteNonQuery(
    'UPDATE products ' +
    'SET search_vector = ' +
    '  TO_TSVECTOR(''english'', COALESCE(name, '''') || '' '' || ' +
    '              COALESCE(description, '''') || '' '' || ' +
    '              COALESCE(brand, ''''))'
  );
  
  // สร้าง trigger เพื่ออัปเดตอัตโนมัติ
  FDB.ExecuteNonQuery(
    'CREATE OR REPLACE FUNCTION update_search_vector() ' +
    'RETURNS TRIGGER AS $$ ' +
    'BEGIN ' +
    '  NEW.search_vector := ' +
    '    TO_TSVECTOR(''english'', ' +
    '      COALESCE(NEW.name, '''') || '' '' || ' +
    '      COALESCE(NEW.description, '''') || '' '' || ' +
    '      COALESCE(NEW.brand, '''')); ' +
    '  RETURN NEW; ' +
    'END; ' +
    '$$ LANGUAGE plpgsql'
  );
  
  FDB.ExecuteNonQuery(
    'DROP TRIGGER IF EXISTS products_search_update ON products'
  );
  
  FDB.ExecuteNonQuery(
    'CREATE TRIGGER products_search_update ' +
    'BEFORE INSERT OR UPDATE ON products ' +
    'FOR EACH ROW EXECUTE FUNCTION update_search_vector()'
  );
end;

procedure TFullTextSearch.SearchProducts(const SearchTerm: string);
var
  Q: TSQLQuery;
begin
  Q := TSQLQuery.Create(nil);
  try
    Q.DataBase := FDB.Connection;
    Q.Transaction := FDB.Transaction;
    
    Q.SQL.Text := 
      'SELECT product_id, name, description, price ' +
      'FROM products ' +
      'WHERE search_vector @@ PLAINTO_TSQUERY(:search_term) ' +
      'ORDER BY name';
    
    Q.ParamByName('search_term').AsString := SearchTerm;
    
    FDB.Transaction.StartTransaction;
    Q.Open;
    
    WriteLn(Format('ผลการค้นหา "%s":', [SearchTerm]));
    if Q.EOF then
      WriteLn('  ไม่พบสินค้าที่ตรงกับคำค้นหา')
    else
    begin
      while not Q.EOF do
      begin
        WriteLn(Format('  %s - %.2f บาท',
          [Q.FieldByName('name').AsString,
           Q.FieldByName('price').AsFloat]));
        Q.Next;
      end;
    end;
    
    Q.Close;
    FDB.Transaction.Commit;
    
  finally
    Q.Free;
  end;
end;

procedure TFullTextSearch.SearchWithRanking(const SearchTerm: string);
var
  Q: TSQLQuery;
begin
  Q := TSQLQuery.Create(nil);
  try
    Q.DataBase := FDB.Connection;
    Q.Transaction := FDB.Transaction;
    
    // ค้นหาพร้อม ranking
    Q.SQL.Text := 
      'SELECT ' +
      '  product_id, name, price, ' +
      '  TS_RANK(search_vector, query) AS rank, ' +
      '  TS_HEADLINE(''english'', description, query, ' +
      '    ''StartSel = <b>, StopSel = </b>, MaxWords=50'') AS excerpt ' +
      'FROM products, PLAINTO_TSQUERY(:search_term) query ' +
      'WHERE search_vector @@ query ' +
      'ORDER BY rank DESC ' +
      'LIMIT 10';
    
    Q.ParamByName('search_term').AsString := SearchTerm;
    
    FDB.Transaction.StartTransaction;
    Q.Open;
    
    WriteLn(Format('ผลการค้นหา "%s" (เรียงตาม relevance):', [SearchTerm]));
    var Rank := 1;
    while not Q.EOF do
    begin
      WriteLn(Format('%d. %s (score: %.4f)',
        [Rank,
         Q.FieldByName('name').AsString,
         Q.FieldByName('rank').AsFloat]));
      Inc(Rank);
      Q.Next;
    end;
    
    Q.Close;
    FDB.Transaction.Commit;
    
  finally
    Q.Free;
  end;
end;

procedure TFullTextSearch.SearchThaiText(const SearchTerm: string);
var
  Q: TSQLQuery;
begin
  // PostgreSQL ยังไม่รองรับ Thai tokenization แบบ built-in
  // ต้องใช้ LIKE หรือ trigram สำหรับภาษาไทย
  
  // ติดตั้ง pg_trgm extension
  FDB.ExecuteNonQuery('CREATE EXTENSION IF NOT EXISTS pg_trgm');
  
  // สร้าง trigram index
  FDB.ExecuteNonQuery(
    'CREATE INDEX IF NOT EXISTS idx_products_name_trgm ' +
    'ON products USING GIN (name gin_trgm_ops)'
  );
  
  Q := TSQLQuery.Create(nil);
  try
    Q.DataBase := FDB.Connection;
    Q.Transaction := FDB.Transaction;
    
    // ค้นหาด้วย similarity
    Q.SQL.Text := 
      'SELECT product_id, name, price, ' +
      '  SIMILARITY(name, :search_term) AS sim_score ' +
      'FROM products ' +
      'WHERE SIMILARITY(name, :search_term) > 0.2 ' +
      'ORDER BY sim_score DESC';
    
    Q.ParamByName('search_term').AsString := SearchTerm;
    
    FDB.Transaction.StartTransaction;
    Q.Open;
    
    WriteLn(Format('ค้นหา "%s" (Thai fuzzy search):', [SearchTerm]));
    while not Q.EOF do
    begin
      WriteLn(Format('  %s (similarity: %.2f)',
        [Q.FieldByName('name').AsString,
         Q.FieldByName('sim_score').AsFloat]));
      Q.Next;
    end;
    
    Q.Close;
    FDB.Transaction.Commit;
    
  finally
    Q.Free;
  end;
end;

end.
```

---

## ตัวอย่างระบบ E-Commerce

### โครงสร้างฐานข้อมูล

```sql
-- PostgreSQL E-Commerce Database

CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gin;

-- Customers
CREATE TABLE customers (
    customer_id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    customer_code VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(100) UNIQUE NOT NULL,
    password_hash VARCHAR(255),
    full_name VARCHAR(100) NOT NULL,
    phone VARCHAR(20),
    birth_date DATE,
    gender VARCHAR(10),
    tier VARCHAR(20) DEFAULT 'bronze' CHECK (tier IN ('bronze', 'silver', 'gold', 'platinum')),
    total_spent NUMERIC(15,2) DEFAULT 0,
    points INTEGER DEFAULT 0,
    addresses JSONB DEFAULT '[]',
    preferences JSONB DEFAULT '{}',
    tags TEXT[],
    is_active BOOLEAN DEFAULT TRUE,
    email_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    last_login TIMESTAMP WITH TIME ZONE
);

-- Categories (Hierarchical)
CREATE TABLE categories (
    category_id SERIAL PRIMARY KEY,
    parent_id INTEGER REFERENCES categories(category_id),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    image_url VARCHAR(500),
    sort_order INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    path ltree  -- ใช้ ltree extension สำหรับ hierarchical path
);

-- Products
CREATE TABLE products (
    product_id SERIAL PRIMARY KEY,
    sku VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    description TEXT,
    category_id INTEGER REFERENCES categories(category_id),
    brand VARCHAR(100),
    price NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    sale_price NUMERIC(10,2) CHECK (sale_price >= 0),
    cost NUMERIC(10,2),
    stock_qty INTEGER DEFAULT 0,
    low_stock_threshold INTEGER DEFAULT 10,
    weight NUMERIC(8,3),
    dimensions JSONB,  -- {length, width, height}
    attributes JSONB DEFAULT '{}',
    images TEXT[],
    tags TEXT[],
    search_vector tsvector,
    rating_avg NUMERIC(3,2) DEFAULT 0,
    rating_count INTEGER DEFAULT 0,
    sold_count INTEGER DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Orders
CREATE TABLE orders (
    order_id SERIAL PRIMARY KEY,
    order_number VARCHAR(20) UNIQUE NOT NULL,
    customer_id UUID REFERENCES customers(customer_id),
    status VARCHAR(30) DEFAULT 'pending' 
        CHECK (status IN ('pending', 'confirmed', 'processing', 
                         'shipped', 'delivered', 'cancelled', 'refunded')),
    subtotal NUMERIC(12,2) NOT NULL,
    discount_amount NUMERIC(12,2) DEFAULT 0,
    shipping_amount NUMERIC(10,2) DEFAULT 0,
    tax_amount NUMERIC(10,2) DEFAULT 0,
    total_amount NUMERIC(12,2) NOT NULL,
    currency CHAR(3) DEFAULT 'THB',
    payment_method VARCHAR(50),
    payment_status VARCHAR(20) DEFAULT 'pending',
    shipping_address JSONB NOT NULL,
    billing_address JSONB,
    notes TEXT,
    metadata JSONB DEFAULT '{}',
    ordered_at TIMESTAMP DEFAULT NOW(),
    confirmed_at TIMESTAMP,
    shipped_at TIMESTAMP,
    delivered_at TIMESTAMP
);

-- Order Items
CREATE TABLE order_items (
    item_id SERIAL PRIMARY KEY,
    order_id INTEGER REFERENCES orders(order_id) ON DELETE CASCADE,
    product_id INTEGER REFERENCES products(product_id),
    sku VARCHAR(50),
    product_name VARCHAR(200),
    quantity INTEGER NOT NULL CHECK (quantity > 0),
    unit_price NUMERIC(10,2) NOT NULL,
    discount_amount NUMERIC(10,2) DEFAULT 0,
    total_price NUMERIC(10,2) NOT NULL,
    attributes JSONB
);

-- Reviews
CREATE TABLE reviews (
    review_id SERIAL PRIMARY KEY,
    product_id INTEGER REFERENCES products(product_id),
    customer_id UUID REFERENCES customers(customer_id),
    order_id INTEGER REFERENCES orders(order_id),
    rating INTEGER CHECK (rating BETWEEN 1 AND 5),
    title VARCHAR(200),
    body TEXT,
    pros TEXT[],
    cons TEXT[],
    images TEXT[],
    helpful_count INTEGER DEFAULT 0,
    is_verified BOOLEAN DEFAULT FALSE,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_brand ON products(brand);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_search ON products USING GIN(search_vector);
CREATE INDEX idx_products_tags ON products USING GIN(tags);
CREATE INDEX idx_products_attributes ON products USING GIN(attributes jsonb_path_ops);
CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_date ON orders(ordered_at DESC);
```

### Pascal Code สำหรับระบบ E-Commerce

```pascal
unit ECommerceService;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, sqldb, fpjson, jsonparser, DateUtils;

type
  TOrderItem = record
    ProductID: Integer;
    Quantity: Integer;
    UnitPrice: Double;
  end;
  TOrderItemArray = array of TOrderItem;

  TECommerceService = class
  private
    FDB: TPgDatabaseManager;
  public
    constructor Create(ADB: TPgDatabaseManager);
    
    // Product operations
    function SearchProducts(const Query: string; 
                           MinPrice, MaxPrice: Double;
                           CategoryID: Integer;
                           Page, PageSize: Integer): TSQLQuery;
    function GetProductDetails(ProductID: Integer): TSQLQuery;
    function GetRelatedProducts(ProductID: Integer; Limit: Integer = 5): TSQLQuery;
    
    // Order operations
    function CreateOrder(CustomerID: string;
                        Items: TOrderItemArray;
                        const ShippingAddress: string;
                        const Coupon: string = ''): Integer;
    function GetOrderHistory(CustomerID: string; 
                            Page, PageSize: Integer): TSQLQuery;
    function UpdateOrderStatus(OrderID: Integer; 
                              const Status: string): Boolean;
    
    // Analytics
    procedure GetSalesSummary(StartDate, EndDate: TDateTime);
    procedure GetTopProducts(Limit: Integer = 10);
    procedure GetCustomerAnalytics;
  end;

implementation

constructor TECommerceService.Create(ADB: TPgDatabaseManager);
begin
  FDB := ADB;
end;

function TECommerceService.SearchProducts(const Query: string;
                                         MinPrice, MaxPrice: Double;
                                         CategoryID: Integer;
                                         Page, PageSize: Integer): TSQLQuery;
var
  SQL: TStringList;
  Q: TSQLQuery;
begin
  SQL := TStringList.Create;
  try
    SQL.Add('SELECT ');
    SQL.Add('  p.product_id, p.sku, p.name, p.price, p.sale_price, ');
    SQL.Add('  p.stock_qty, p.rating_avg, p.rating_count, p.sold_count, ');
    SQL.Add('  p.images[1] AS thumbnail, ');
    SQL.Add('  c.name AS category_name, ');
    SQL.Add('  CASE WHEN p.sale_price IS NOT NULL ');
    SQL.Add('    THEN ROUND((1 - p.sale_price/p.price) * 100) ');
    SQL.Add('    ELSE 0 END AS discount_pct, ');
    SQL.Add('  COUNT(*) OVER() AS total_count ');
    SQL.Add('FROM products p ');
    SQL.Add('LEFT JOIN categories c ON p.category_id = c.category_id ');
    SQL.Add('WHERE p.is_active = TRUE ');
    
    if Query <> '' then
      SQL.Add('  AND p.search_vector @@ PLAINTO_TSQUERY(:search_term) ');
    
    if MinPrice > 0 then
      SQL.Add('  AND p.price >= :min_price ');
    
    if MaxPrice > 0 then
      SQL.Add('  AND p.price <= :max_price ');
    
    if CategoryID > 0 then
      SQL.Add('  AND p.category_id = :category_id ');
    
    if Query <> '' then
      SQL.Add('ORDER BY TS_RANK(p.search_vector, PLAINTO_TSQUERY(:search_term)) DESC, p.sold_count DESC ')
    else
      SQL.Add('ORDER BY p.sold_count DESC ');
    
    SQL.Add(Format('LIMIT %d OFFSET %d', [PageSize, (Page - 1) * PageSize]));
    
    Q := TSQLQuery.Create(nil);
    Q.DataBase := FDB.Connection;
    Q.Transaction := FDB.Transaction;
    Q.SQL.Text := SQL.Text;
    
    if Query <> '' then
      Q.ParamByName('search_term').AsString := Query;
    if MinPrice > 0 then
      Q.ParamByName('min_price').AsFloat := MinPrice;
    if MaxPrice > 0 then
      Q.ParamByName('max_price').AsFloat := MaxPrice;
    if CategoryID > 0 then
      Q.ParamByName('category_id').AsInteger := CategoryID;
    
    FDB.Transaction.StartTransaction;
    Q.Open;
    FDB.Transaction.Commit;
    
    Result := Q;
    
  finally
    SQL.Free;
  end;
end;

function TECommerceService.CreateOrder(CustomerID: string;
                                      Items: TOrderItemArray;
                                      const ShippingAddress: string;
                                      const Coupon: string = ''): Integer;
var
  Q: TSQLQuery;
  Subtotal, TaxAmount, TotalAmount: Double;
  OrderNumber: string;
  i: Integer;
begin
  Result := -1;
  
  Q := TSQLQuery.Create(nil);
  try
    Q.DataBase := FDB.Connection;
    Q.Transaction := FDB.Transaction;
    
    FDB.Transaction.StartTransaction;
    
    try
      // คำนวณยอดรวม
      Subtotal := 0;
      for i := 0 to High(Items) do
        Subtotal := Subtotal + Items[i].Quantity * Items[i].UnitPrice;
      
      TaxAmount := Subtotal * 0.07;  // VAT 7%
      TotalAmount := Subtotal + TaxAmount;
      
      // สร้าง order number
      Q.SQL.Text := 
        'SELECT ''ORD-'' || TO_CHAR(NOW(), ''YYYYMMDD'') || ''-'' || ' +
        '  LPAD(NEXTVAL(''order_seq'')::TEXT, 6, ''0'')';
      Q.Open;
      OrderNumber := Q.Fields[0].AsString;
      Q.Close;
      
      // Insert order
      Q.SQL.Text := 
        'INSERT INTO orders (order_number, customer_id, subtotal, tax_amount, ' +
        '  total_amount, shipping_address) ' +
        'VALUES (:order_number, :customer_id::UUID, :subtotal, :tax_amount, ' +
        '  :total_amount, :shipping_address::JSONB) ' +
        'RETURNING order_id';
      
      Q.ParamByName('order_number').AsString := OrderNumber;
      Q.ParamByName('customer_id').AsString := CustomerID;
      Q.ParamByName('subtotal').AsFloat := Subtotal;
      Q.ParamByName('tax_amount').AsFloat := TaxAmount;
      Q.ParamByName('total_amount').AsFloat := TotalAmount;
      Q.ParamByName('shipping_address').AsString := ShippingAddress;
      
      Q.Open;
      Result := Q.Fields[0].AsInteger;
      Q.Close;
      
      // Insert order items
      for i := 0 to High(Items) do
      begin
        // ตรวจสอบและลด stock
        Q.SQL.Text := 
          'UPDATE products SET stock_qty = stock_qty - :qty, ' +
          '  sold_count = sold_count + :qty ' +
          'WHERE product_id = :product_id AND stock_qty >= :qty ' +
          'RETURNING name';
        
        Q.ParamByName('qty').AsInteger := Items[i].Quantity;
        Q.ParamByName('product_id').AsInteger := Items[i].ProductID;
        
        Q.Open;
        if Q.EOF then
        begin
          Q.Close;
          raise Exception.CreateFmt('สินค้า ID %d ไม่มีในสต็อกเพียงพอ', 
            [Items[i].ProductID]);
        end;
        var ProductName := Q.FieldByName('name').AsString;
        Q.Close;
        
        // Insert item
        Q.SQL.Text := 
          'INSERT INTO order_items (order_id, product_id, product_name, ' +
          '  quantity, unit_price, total_price) ' +
          'VALUES (:order_id, :product_id, :product_name, :qty, :price, :total)';
        
        Q.ParamByName('order_id').AsInteger := Result;
        Q.ParamByName('product_id').AsInteger := Items[i].ProductID;
        Q.ParamByName('product_name').AsString := ProductName;
        Q.ParamByName('qty').AsInteger := Items[i].Quantity;
        Q.ParamByName('price').AsFloat := Items[i].UnitPrice;
        Q.ParamByName('total').AsFloat := Items[i].Quantity * Items[i].UnitPrice;
        
        Q.ExecSQL;
      end;
      
      FDB.Transaction.Commit;
      WriteLn(Format('สร้างคำสั่งซื้อ %s สำเร็จ (ID: %d)', [OrderNumber, Result]));
      
    except
      on E: Exception do
      begin
        FDB.Transaction.Rollback;
        Result := -1;
        WriteLn('ไม่สามารถสร้างคำสั่งซื้อ: ' + E.Message);
      end;
    end;
    
  finally
    Q.Free;
  end;
end;

procedure TECommerceService.GetSalesSummary(StartDate, EndDate: TDateTime);
var
  Q: TSQLQuery;
begin
  Q := TSQLQuery.Create(nil);
  try
    Q.DataBase := FDB.Connection;
    Q.Transaction := FDB.Transaction;
    
    Q.SQL.Text := 
      'WITH daily_stats AS (' +
      '  SELECT ' +
      '    DATE(ordered_at) AS sale_date, ' +
      '    COUNT(*) AS order_count, ' +
      '    SUM(total_amount) AS revenue, ' +
      '    AVG(total_amount) AS avg_order_value, ' +
      '    COUNT(DISTINCT customer_id) AS unique_customers ' +
      '  FROM orders ' +
      '  WHERE status NOT IN (''cancelled'', ''refunded'') ' +
      '    AND ordered_at BETWEEN :start_date AND :end_date ' +
      '  GROUP BY DATE(ordered_at) ' +
      ') ' +
      'SELECT ' +
      '  sale_date, ' +
      '  order_count, ' +
      '  revenue, ' +
      '  avg_order_value, ' +
      '  unique_customers, ' +
      '  SUM(revenue) OVER (ORDER BY sale_date) AS cumulative_revenue ' +
      'FROM daily_stats ' +
      'ORDER BY sale_date';
    
    Q.ParamByName('start_date').AsDateTime := StartDate;
    Q.ParamByName('end_date').AsDateTime := EndDate;
    
    FDB.Transaction.StartTransaction;
    Q.Open;
    
    WriteLn('สรุปยอดขายรายวัน:');
    WriteLn(Format('%-12s %8s %12s %12s %10s %15s',
      ['วันที่', 'คำสั่ง', 'รายได้', 'ค่าเฉลี่ย', 'ลูกค้า', 'สะสม']));
    WriteLn(StringOfChar('-', 75));
    
    while not Q.EOF do
    begin
      WriteLn(Format('%-12s %8d %12.2f %12.2f %10d %15.2f',
        [FormatDateTime('dd/mm/yyyy', Q.FieldByName('sale_date').AsDateTime),
         Q.FieldByName('order_count').AsInteger,
         Q.FieldByName('revenue').AsFloat,
         Q.FieldByName('avg_order_value').AsFloat,
         Q.FieldByName('unique_customers').AsInteger,
         Q.FieldByName('cumulative_revenue').AsFloat]));
      Q.Next;
    end;
    
    Q.Close;
    FDB.Transaction.Commit;
    
  finally
    Q.Free;
  end;
end;

end.
```

---

## แบบฝึกหัด 10 ข้อ

**ข้อ 1:** สร้างระบบจัดการ category แบบ hierarchical
```pascal
// ใช้ recursive CTE เพื่อแสดงโครงสร้าง category
// เพิ่ม, แก้ไข, ลบ category
// ย้าย category ไปยัง parent อื่น
```

**ข้อ 2:** เขียนฟังก์ชัน search สินค้าขั้นสูง
```pascal
// รองรับ full-text search
// กรองด้วย price range, category, brand
// เรียงลำดับได้หลายแบบ
// Pagination
```

**ข้อ 3:** สร้างระบบ customer tier
```pascal
// คำนวณ tier ตาม total_spent
// bronze: < 1,000
// silver: 1,000-10,000
// gold: 10,000-50,000
// platinum: > 50,000
// อัปเดต tier อัตโนมัติหลัง payment
```

**ข้อ 4:** เขียน window function สำหรับ sales ranking
```pascal
// แสดงอันดับสินค้าขายดีในแต่ละหมวดหมู่
// แสดง % ของยอดขายรวม
// แสดง running total
```

**ข้อ 5:** สร้างระบบ coupon และ discount
```sql
-- ตาราง coupons: code, discount_type (%, fixed), value, min_order, expiry
-- ตรวจสอบ coupon ว่า valid หรือไม่
-- คำนวณยอดหลัง discount
-- บันทึกการใช้ coupon
```

**ข้อ 6:** เขียนระบบ inventory management
```pascal
// ติดตามการเพิ่ม/ลด stock
// แจ้งเตือนเมื่อ stock ต่ำกว่า threshold
// รายงาน stock movement
// Reorder suggestion
```

**ข้อ 7:** สร้างระบบ review และ rating
```pascal
// เพิ่ม review พร้อม rating
// คำนวณ average rating
// ป้องกัน review ซ้ำ (1 order = 1 review ต่อสินค้า)
// แสดง review พร้อม pagination
```

**ข้อ 8:** เขียน stored procedure สำหรับ order processing
```sql
-- รับ order และ process ทีละขั้น
-- Validate stock
-- Apply discount
-- Update inventory
-- Send notification
```

**ข้อ 9:** สร้าง dashboard analytics
```pascal
// ยอดขายวันนี้, สัปดาห์นี้, เดือนนี้
// เปรียบเทียบกับช่วงเดียวกันของปีก่อน
// Top 10 สินค้าขายดี
// Top 10 ลูกค้า
// สินค้าใกล้หมดสต็อก
```

**ข้อ 10:** โปรเจกต์สุดท้าย - ระบบ E-Commerce สมบูรณ์
```pascal
// หน้าค้นหาสินค้า
// รายละเอียดสินค้า + reviews
// ตะกร้าสินค้า (shopping cart)
// Checkout process
// Order tracking
// Admin dashboard
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- การเชื่อมต่อ PostgreSQL ด้วย pqconnection และ ZEOS
- Advanced SQL: Joins, Subqueries, Window Functions, CTE
- PostgreSQL-specific features: Sequences, Schemas, Roles
- JSON/JSONB data type operations
- Array data types
- Full-text search
- ระบบ E-Commerce ตัวอย่างแบบสมบูรณ์

PostgreSQL เป็นฐานข้อมูลที่ทรงพลังและเหมาะสมสำหรับแอปพลิเคชันขนาดใหญ่ที่ต้องการความซับซ้อนของข้อมูล
