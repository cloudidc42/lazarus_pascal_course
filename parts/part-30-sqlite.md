# Part 30 - SQLite Database

## บทนำ

SQLite เป็นฐานข้อมูลเชิงสัมพันธ์ที่ใช้ไฟล์เดียวในการเก็บข้อมูลทั้งหมด ไม่ต้องการ server ทำให้เหมาะสำหรับแอปพลิเคชัน desktop, mobile และ embedded systems ใน Lazarus เราสามารถใช้ SQLite ได้ผ่าน SQLDB components

### ข้อดีของ SQLite:
- ไม่ต้องติดตั้ง server
- ไฟล์เดียว = ฐานข้อมูลทั้งหมด
- ทำงานบน Windows, Linux, macOS, Android, iOS
- ACID compliant
- ฟรีและ open source
- เร็วมากสำหรับ read operations

---

## 30.1 SQLite Overview และ Setup

### การติดตั้ง SQLite Components ใน Lazarus

```
1. ติดตั้ง SQLite library:
   - Windows: sqlite3.dll (ดาวน์โหลดจาก sqlite.org)
   - Linux: sudo apt install libsqlite3-dev
   - macOS: มาพร้อมระบบ

2. ใน Lazarus IDE:
   - ไปที่ Project > Project Options > Compiler Options > Libraries
   - เพิ่ม path ของ sqlite library
   
3. Units ที่ต้องใช้:
   - sqlite3conn (TSQLite3Connection)
   - sqldb (TSQLQuery, TSQLTransaction)
   - db (TDataSource, TDataSet, TField)

4. ตรวจสอบว่า sqlite3.dll อยู่ใน:
   - Windows: ใน folder ของโปรแกรม หรือ System32
   - Linux: /usr/lib/ หรือ /usr/local/lib/
```

### โครงสร้าง Project

```
MyProject/
├── main.pas          - โปรแกรมหลัก
├── database.pas      - Database layer
├── models.pas        - Data models
├── forms/
│   ├── mainform.pas  - Form หลัก
│   └── editform.pas  - Form แก้ไข
└── data/
    └── myapp.db      - SQLite database file
```

---

## 30.2 TSQLite3Connection

```pascal
program SQLite3ConnectionDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, SQLite3Conn, SQLDB;

procedure DemoConnection;
var
  Conn: TSQLite3Connection;
begin
  WriteLn('=== TSQLite3Connection Demo ===');
  
  Conn := TSQLite3Connection.Create(nil);
  try
    // ตั้งค่า connection
    Conn.DatabaseName := '/tmp/test.db';
    // Conn.UserName := '';     // SQLite ไม่ต้องการ
    // Conn.Password := '';     // SQLite ไม่ต้องการ
    
    // เปิดการเชื่อมต่อ
    try
      Conn.Open;
      WriteLn('เชื่อมต่อสำเร็จ');
      WriteLn('DatabaseName: ', Conn.DatabaseName);
      WriteLn('Connected: ', Conn.Connected);
      
      // ปิด connection
      Conn.Close;
      WriteLn('ปิดการเชื่อมต่อแล้ว');
      
    except
      on E: Exception do
        WriteLn('ข้อผิดพลาด: ', E.Message);
    end;
    
  finally
    Conn.Free;
  end;
end;

procedure DemoConnectionOptions;
var
  Conn: TSQLite3Connection;
begin
  WriteLn(#10'=== Connection Options ===');
  
  Conn := TSQLite3Connection.Create(nil);
  try
    Conn.DatabaseName := '/tmp/test_options.db';
    
    // Params สำหรับ SQLite
    // Conn.Params.Add('timeout=5000');  // timeout ใน ms
    
    // Journal Mode (WAL = Write-Ahead Logging - เร็วกว่า)
    Conn.Open;
    Conn.ExecuteDirect('PRAGMA journal_mode=WAL');
    Conn.ExecuteDirect('PRAGMA foreign_keys=ON');  // Enable FK constraints
    Conn.ExecuteDirect('PRAGMA synchronous=NORMAL');  // เร็วขึ้นแต่ความปลอดภัยลดลงเล็กน้อย
    
    WriteLn('ตั้งค่า SQLite options สำเร็จ');
    
    Conn.Close;
  finally
    Conn.Free;
  end;
end;

begin
  DemoConnection;
  DemoConnectionOptions;
  ReadLn;
end.
```

---

## 30.3 TSQLTransaction

```pascal
program SQLTransactionDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, SQLite3Conn, SQLDB, DB;

procedure DemoTransaction;
var
  Conn: TSQLite3Connection;
  Trans: TSQLTransaction;
begin
  WriteLn('=== TSQLTransaction Demo ===');
  
  Conn := TSQLite3Connection.Create(nil);
  Trans := TSQLTransaction.Create(nil);
  try
    Trans.DataBase := Conn;
    Conn.DatabaseName := '/tmp/test_trans.db';
    Conn.Open;
    
    // สร้างตาราง
    Trans.StartTransaction;
    try
      Conn.ExecuteDirect('CREATE TABLE IF NOT EXISTS accounts (' +
                         '  id INTEGER PRIMARY KEY AUTOINCREMENT,' +
                         '  name TEXT NOT NULL,' +
                         '  balance REAL DEFAULT 0' +
                         ')');
      
      // เพิ่มข้อมูล
      Conn.ExecuteDirect('INSERT INTO accounts (name, balance) VALUES ("บัญชี A", 1000)');
      Conn.ExecuteDirect('INSERT INTO accounts (name, balance) VALUES ("บัญชี B", 500)');
      
      Trans.Commit;
      WriteLn('Commit สำเร็จ: สร้างตารางและเพิ่มข้อมูล');
    except
      Trans.Rollback;
      raise;
    end;
    
    // Transaction ที่ล้มเหลว (Rollback)
    WriteLn(#10'ทดสอบ Rollback:');
    Trans.StartTransaction;
    try
      Conn.ExecuteDirect('UPDATE accounts SET balance = balance - 200 WHERE name = "บัญชี A"');
      WriteLn('หักเงินจาก บัญชี A');
      
      // จำลองข้อผิดพลาด
      raise Exception.Create('การโอนเงินล้มเหลว');
      
      Conn.ExecuteDirect('UPDATE accounts SET balance = balance + 200 WHERE name = "บัญชี B"');
      Trans.Commit;
    except
      on E: Exception do
      begin
        Trans.Rollback;
        WriteLn('Rollback เพราะ: ', E.Message);
      end;
    end;
    
    // Transaction ที่สำเร็จ
    WriteLn(#10'ทดสอบ Transaction สำเร็จ:');
    Trans.StartTransaction;
    try
      Conn.ExecuteDirect('UPDATE accounts SET balance = balance - 200 WHERE name = "บัญชี A"');
      Conn.ExecuteDirect('UPDATE accounts SET balance = balance + 200 WHERE name = "บัญชี B"');
      Trans.Commit;
      WriteLn('โอนเงิน 200 จาก A ไป B สำเร็จ');
    except
      Trans.Rollback;
      raise;
    end;
    
    Conn.Close;
  finally
    Trans.Free;
    Conn.Free;
  end;
end;

begin
  DemoTransaction;
  ReadLn;
end.
```

---

## 30.4 TSQLQuery

```pascal
program SQLQueryDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, SQLite3Conn, SQLDB, DB;

procedure SetupDatabase(Conn: TSQLite3Connection; Trans: TSQLTransaction);
begin
  Trans.StartTransaction;
  try
    Conn.ExecuteDirect('DROP TABLE IF EXISTS students');
    Conn.ExecuteDirect(
      'CREATE TABLE students (' +
      '  id INTEGER PRIMARY KEY AUTOINCREMENT,' +
      '  name TEXT NOT NULL,' +
      '  age INTEGER,' +
      '  gpa REAL,' +
      '  major TEXT,' +
      '  enrolled_date TEXT' +
      ')'
    );
    
    // เพิ่มข้อมูลตัวอย่าง
    Conn.ExecuteDirect('INSERT INTO students (name, age, gpa, major, enrolled_date) VALUES ' +
                       '("สมชาย ใจดี", 20, 3.5, "วิศวกรรมคอมพิวเตอร์", "2022-06-01")');
    Conn.ExecuteDirect('INSERT INTO students (name, age, gpa, major, enrolled_date) VALUES ' +
                       '("สมหญิง งาม", 21, 3.8, "วิทยาการคอมพิวเตอร์", "2021-06-01")');
    Conn.ExecuteDirect('INSERT INTO students (name, age, gpa, major, enrolled_date) VALUES ' +
                       '("วิชัย เก่ง", 22, 3.2, "เทคโนโลยีสารสนเทศ", "2020-06-01")');
    Conn.ExecuteDirect('INSERT INTO students (name, age, gpa, major, enrolled_date) VALUES ' +
                       '("อนันต์ สุข", 20, 3.9, "วิศวกรรมคอมพิวเตอร์", "2022-06-01")');
    Conn.ExecuteDirect('INSERT INTO students (name, age, gpa, major, enrolled_date) VALUES ' +
                       '("บุญมี ดี", 23, 2.8, "วิทยาการคอมพิวเตอร์", "2019-06-01")');
    
    Trans.Commit;
    WriteLn('สร้างฐานข้อมูลตัวอย่างสำเร็จ');
  except
    Trans.Rollback;
    raise;
  end;
end;

procedure DemoSQLQuery;
var
  Conn: TSQLite3Connection;
  Trans: TSQLTransaction;
  Query: TSQLQuery;
begin
  WriteLn('=== TSQLQuery Demo ===');
  
  Conn := TSQLite3Connection.Create(nil);
  Trans := TSQLTransaction.Create(nil);
  Query := TSQLQuery.Create(nil);
  try
    Trans.DataBase := Conn;
    Query.DataBase := Conn;
    Query.Transaction := Trans;
    
    Conn.DatabaseName := '/tmp/students.db';
    Conn.Open;
    
    SetupDatabase(Conn, Trans);
    
    // SELECT ทั้งหมด
    WriteLn(#10'1. SELECT ทั้งหมด:');
    Trans.StartTransaction;
    Query.SQL.Text := 'SELECT * FROM students ORDER BY name';
    Query.Open;
    while not Query.EOF do
    begin
      WriteLn(Format('  ID: %-3d | %-20s | อายุ: %-3d | GPA: %.1f | %s',
                     [Query.FieldByName('id').AsInteger,
                      Query.FieldByName('name').AsString,
                      Query.FieldByName('age').AsInteger,
                      Query.FieldByName('gpa').AsFloat,
                      Query.FieldByName('major').AsString]));
      Query.Next;
    end;
    Query.Close;
    Trans.Commit;
    
    // SELECT พร้อม WHERE
    WriteLn(#10'2. SELECT WHERE GPA >= 3.5:');
    Trans.StartTransaction;
    Query.SQL.Text := 'SELECT name, gpa FROM students WHERE gpa >= 3.5 ORDER BY gpa DESC';
    Query.Open;
    while not Query.EOF do
    begin
      WriteLn(Format('  %-20s: %.1f', [Query.FieldByName('name').AsString,
                                        Query.FieldByName('gpa').AsFloat]));
      Query.Next;
    end;
    Query.Close;
    Trans.Commit;
    
    // Aggregate
    WriteLn(#10'3. Aggregate:');
    Trans.StartTransaction;
    Query.SQL.Text := 'SELECT COUNT(*) AS total, AVG(gpa) AS avg_gpa, ' +
                      'MAX(gpa) AS max_gpa, MIN(gpa) AS min_gpa FROM students';
    Query.Open;
    WriteLn(Format('  จำนวน: %d, เฉลี่ย: %.2f, สูงสุด: %.1f, ต่ำสุด: %.1f',
                   [Query.FieldByName('total').AsInteger,
                    Query.FieldByName('avg_gpa').AsFloat,
                    Query.FieldByName('max_gpa').AsFloat,
                    Query.FieldByName('min_gpa').AsFloat]));
    Query.Close;
    Trans.Commit;
    
    // GROUP BY
    WriteLn(#10'4. GROUP BY major:');
    Trans.StartTransaction;
    Query.SQL.Text := 'SELECT major, COUNT(*) AS count, AVG(gpa) AS avg_gpa ' +
                      'FROM students GROUP BY major ORDER BY major';
    Query.Open;
    while not Query.EOF do
    begin
      WriteLn(Format('  %-35s: %d คน, GPA เฉลี่ย %.2f',
                     [Query.FieldByName('major').AsString,
                      Query.FieldByName('count').AsInteger,
                      Query.FieldByName('avg_gpa').AsFloat]));
      Query.Next;
    end;
    Query.Close;
    Trans.Commit;
    
    Conn.Close;
  finally
    Query.Free;
    Trans.Free;
    Conn.Free;
  end;
end;

begin
  DemoSQLQuery;
  ReadLn;
end.
```

---

## 30.5 CRUD Operations

```pascal
program CRUDOperations;
{$mode objfpc}{$H+}

uses
  SysUtils, SQLite3Conn, SQLDB, DB;

type
  TStudentDB = class
  private
    FConn: TSQLite3Connection;
    FTrans: TSQLTransaction;
    FQuery: TSQLQuery;
    
    procedure InitDatabase;
  public
    constructor Create(const DBFile: String);
    destructor Destroy; override;
    
    // CREATE
    function InsertStudent(const Name: String; Age: Integer; 
                           GPA: Double; const Major: String): Integer;
    // READ
    procedure GetAllStudents;
    function GetStudentByID(ID: Integer): Boolean;
    procedure SearchStudents(const Keyword: String);
    // UPDATE
    function UpdateStudent(ID: Integer; const Name: String; Age: Integer;
                           GPA: Double; const Major: String): Boolean;
    function UpdateGPA(ID: Integer; NewGPA: Double): Boolean;
    // DELETE
    function DeleteStudent(ID: Integer): Boolean;
    procedure DeleteByGPA(MaxGPA: Double);
    // COUNT
    function GetCount: Integer;
  end;

constructor TStudentDB.Create(const DBFile: String);
begin
  inherited Create;
  
  FConn := TSQLite3Connection.Create(nil);
  FTrans := TSQLTransaction.Create(nil);
  FQuery := TSQLQuery.Create(nil);
  
  FTrans.DataBase := FConn;
  FQuery.DataBase := FConn;
  FQuery.Transaction := FTrans;
  
  FConn.DatabaseName := DBFile;
  FConn.Open;
  
  InitDatabase;
end;

destructor TStudentDB.Destroy;
begin
  if FConn.Connected then
    FConn.Close;
  FQuery.Free;
  FTrans.Free;
  FConn.Free;
  inherited Destroy;
end;

procedure TStudentDB.InitDatabase;
begin
  FTrans.StartTransaction;
  try
    FConn.ExecuteDirect(
      'CREATE TABLE IF NOT EXISTS students (' +
      '  id INTEGER PRIMARY KEY AUTOINCREMENT,' +
      '  name TEXT NOT NULL,' +
      '  age INTEGER NOT NULL,' +
      '  gpa REAL NOT NULL DEFAULT 0.0,' +
      '  major TEXT,' +
      '  created_at TEXT DEFAULT (datetime("now","localtime")),' +
      '  updated_at TEXT DEFAULT (datetime("now","localtime"))' +
      ')'
    );
    
    // Pragma settings
    FConn.ExecuteDirect('PRAGMA foreign_keys = ON');
    FConn.ExecuteDirect('PRAGMA journal_mode = WAL');
    
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

// CREATE
function TStudentDB.InsertStudent(const Name: String; Age: Integer;
                                   GPA: Double; const Major: String): Integer;
begin
  Result := -1;
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text :=
      'INSERT INTO students (name, age, gpa, major) VALUES (:name, :age, :gpa, :major)';
    FQuery.ParamByName('name').AsString := Name;
    FQuery.ParamByName('age').AsInteger := Age;
    FQuery.ParamByName('gpa').AsFloat := GPA;
    FQuery.ParamByName('major').AsString := Major;
    FQuery.ExecSQL;
    
    // ได้ ID ที่เพิ่งสร้าง
    FQuery.SQL.Text := 'SELECT last_insert_rowid() AS id';
    FQuery.Open;
    Result := FQuery.FieldByName('id').AsInteger;
    FQuery.Close;
    
    FTrans.Commit;
    WriteLn(Format('เพิ่มนักเรียน "%s" สำเร็จ (ID: %d)', [Name, Result]));
  except
    on E: Exception do
    begin
      FTrans.Rollback;
      WriteLn('เพิ่มไม่ได้: ', E.Message);
    end;
  end;
end;

// READ - แสดงทั้งหมด
procedure TStudentDB.GetAllStudents;
begin
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'SELECT * FROM students ORDER BY name';
    FQuery.Open;
    
    WriteLn(StringOfChar('=', 75));
    WriteLn(Format('%-5s | %-20s | %-4s | %-5s | %-25s',
                   ['ID', 'ชื่อ', 'อายุ', 'GPA', 'สาขา']));
    WriteLn(StringOfChar('-', 75));
    
    while not FQuery.EOF do
    begin
      WriteLn(Format('%-5d | %-20s | %-4d | %-5.2f | %-25s',
                     [FQuery.FieldByName('id').AsInteger,
                      FQuery.FieldByName('name').AsString,
                      FQuery.FieldByName('age').AsInteger,
                      FQuery.FieldByName('gpa').AsFloat,
                      FQuery.FieldByName('major').AsString]));
      FQuery.Next;
    end;
    
    WriteLn(StringOfChar('=', 75));
    WriteLn('จำนวน: ', FQuery.RecordCount, ' รายการ (อาจไม่ถูกต้องสำหรับ SQL query)');
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

// READ - ค้นหาด้วย ID
function TStudentDB.GetStudentByID(ID: Integer): Boolean;
begin
  Result := False;
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'SELECT * FROM students WHERE id = :id';
    FQuery.ParamByName('id').AsInteger := ID;
    FQuery.Open;
    
    if not FQuery.EOF then
    begin
      WriteLn(Format('พบ: ID=%d, %s, อายุ=%d, GPA=%.2f, สาขา=%s',
                     [FQuery.FieldByName('id').AsInteger,
                      FQuery.FieldByName('name').AsString,
                      FQuery.FieldByName('age').AsInteger,
                      FQuery.FieldByName('gpa').AsFloat,
                      FQuery.FieldByName('major').AsString]));
      Result := True;
    end
    else
      WriteLn('ไม่พบ ID: ', ID);
    
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

// READ - ค้นหาด้วยคำ
procedure TStudentDB.SearchStudents(const Keyword: String);
begin
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'SELECT * FROM students WHERE name LIKE :kw OR major LIKE :kw ORDER BY name';
    FQuery.ParamByName('kw').AsString := '%' + Keyword + '%';
    FQuery.Open;
    
    WriteLn('ค้นหา "', Keyword, '":');
    while not FQuery.EOF do
    begin
      WriteLn(Format('  %s (GPA: %.2f, สาขา: %s)',
                     [FQuery.FieldByName('name').AsString,
                      FQuery.FieldByName('gpa').AsFloat,
                      FQuery.FieldByName('major').AsString]));
      FQuery.Next;
    end;
    
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

// UPDATE - แก้ไขทั้งหมด
function TStudentDB.UpdateStudent(ID: Integer; const Name: String; Age: Integer;
                                   GPA: Double; const Major: String): Boolean;
begin
  Result := False;
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text :=
      'UPDATE students SET name=:name, age=:age, gpa=:gpa, major=:major, ' +
      'updated_at=datetime("now","localtime") WHERE id=:id';
    FQuery.ParamByName('name').AsString := Name;
    FQuery.ParamByName('age').AsInteger := Age;
    FQuery.ParamByName('gpa').AsFloat := GPA;
    FQuery.ParamByName('major').AsString := Major;
    FQuery.ParamByName('id').AsInteger := ID;
    FQuery.ExecSQL;
    
    Result := FQuery.RowsAffected > 0;
    FTrans.Commit;
    
    if Result then
      WriteLn(Format('อัปเดต ID %d สำเร็จ', [ID]))
    else
      WriteLn('ไม่พบ ID: ', ID);
  except
    on E: Exception do
    begin
      FTrans.Rollback;
      WriteLn('อัปเดตไม่ได้: ', E.Message);
    end;
  end;
end;

// UPDATE - แก้ไข GPA เท่านั้น
function TStudentDB.UpdateGPA(ID: Integer; NewGPA: Double): Boolean;
begin
  Result := False;
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'UPDATE students SET gpa=:gpa, updated_at=datetime("now","localtime") WHERE id=:id';
    FQuery.ParamByName('gpa').AsFloat := NewGPA;
    FQuery.ParamByName('id').AsInteger := ID;
    FQuery.ExecSQL;
    
    Result := FQuery.RowsAffected > 0;
    FTrans.Commit;
    
    if Result then
      WriteLn(Format('อัปเดต GPA ID %d เป็น %.2f', [ID, NewGPA]))
    else
      WriteLn('ไม่พบ ID: ', ID);
  except
    on E: Exception do
    begin
      FTrans.Rollback;
      WriteLn('อัปเดตไม่ได้: ', E.Message);
    end;
  end;
end;

// DELETE - ลบด้วย ID
function TStudentDB.DeleteStudent(ID: Integer): Boolean;
begin
  Result := False;
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'DELETE FROM students WHERE id = :id';
    FQuery.ParamByName('id').AsInteger := ID;
    FQuery.ExecSQL;
    
    Result := FQuery.RowsAffected > 0;
    FTrans.Commit;
    
    if Result then
      WriteLn('ลบ ID ', ID, ' สำเร็จ')
    else
      WriteLn('ไม่พบ ID: ', ID);
  except
    on E: Exception do
    begin
      FTrans.Rollback;
      WriteLn('ลบไม่ได้: ', E.Message);
    end;
  end;
end;

// DELETE - ลบตาม GPA
procedure TStudentDB.DeleteByGPA(MaxGPA: Double);
begin
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'DELETE FROM students WHERE gpa < :max_gpa';
    FQuery.ParamByName('max_gpa').AsFloat := MaxGPA;
    FQuery.ExecSQL;
    
    WriteLn(Format('ลบนักเรียน GPA < %.1f: %d รายการ', [MaxGPA, FQuery.RowsAffected]));
    FTrans.Commit;
  except
    on E: Exception do
    begin
      FTrans.Rollback;
      WriteLn('ลบไม่ได้: ', E.Message);
    end;
  end;
end;

// COUNT
function TStudentDB.GetCount: Integer;
begin
  Result := 0;
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'SELECT COUNT(*) AS cnt FROM students';
    FQuery.Open;
    Result := FQuery.FieldByName('cnt').AsInteger;
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

begin
  WriteLn('=== CRUD Operations Demo ===');
  
  var DB := TStudentDB.Create('/tmp/crud_demo.db');
  try
    // CREATE
    WriteLn(#10'--- CREATE ---');
    DB.InsertStudent('สมชาย ใจดี', 20, 3.5, 'วิศวกรรมคอมพิวเตอร์');
    DB.InsertStudent('สมหญิง งาม', 21, 3.8, 'วิทยาการคอมพิวเตอร์');
    DB.InsertStudent('วิชัย เก่ง', 22, 3.2, 'เทคโนโลยีสารสนเทศ');
    DB.InsertStudent('อนันต์ สุข', 20, 3.9, 'วิศวกรรมคอมพิวเตอร์');
    DB.InsertStudent('บุญมี ดี', 23, 2.8, 'วิทยาการคอมพิวเตอร์');
    
    // READ
    WriteLn(#10'--- READ ---');
    DB.GetAllStudents;
    
    WriteLn(#10'ค้นหา ID 2:');
    DB.GetStudentByID(2);
    
    WriteLn(#10'ค้นหา "คอมพิวเตอร์":');
    DB.SearchStudents('คอมพิวเตอร์');
    
    // UPDATE
    WriteLn(#10'--- UPDATE ---');
    DB.UpdateGPA(1, 3.7);
    DB.UpdateStudent(3, 'วิชัย เก่งมาก', 23, 3.5, 'เทคโนโลยีสารสนเทศ');
    
    WriteLn(#10'หลัง UPDATE:');
    DB.GetAllStudents;
    
    // DELETE
    WriteLn(#10'--- DELETE ---');
    DB.DeleteStudent(5);
    DB.GetAllStudents;
    
    WriteLn(#10'จำนวนทั้งหมด: ', DB.GetCount);
    
  finally
    DB.Free;
  end;
  
  ReadLn;
end.
```

---

## 30.6 Parameterized Queries (ป้องกัน SQL Injection)

```pascal
program ParameterizedQueries;
{$mode objfpc}{$H+}

uses
  SysUtils, SQLite3Conn, SQLDB, DB;

procedure DemoSQLInjection;
begin
  WriteLn('=== SQL Injection ===');
  WriteLn;
  WriteLn('ตัวอย่าง SQL Injection ที่ไม่ปลอดภัย:');
  WriteLn;
  
  var UserInput := "'; DROP TABLE users; --";
  var SQL := 'SELECT * FROM users WHERE name = ''' + UserInput + '''';
  WriteLn('SQL ที่อันตราย:');
  WriteLn(SQL);
  WriteLn('SQL ดังกล่าวจะลบตาราง users!');
  
  WriteLn;
  WriteLn('วิธีป้องกัน: ใช้ Parameterized Queries เสมอ');
end;

procedure DemoParameters;
var
  Conn: TSQLite3Connection;
  Trans: TSQLTransaction;
  Query: TSQLQuery;
begin
  WriteLn(#10'=== Parameterized Queries ===');
  
  Conn := TSQLite3Connection.Create(nil);
  Trans := TSQLTransaction.Create(nil);
  Query := TSQLQuery.Create(nil);
  try
    Trans.DataBase := Conn;
    Query.DataBase := Conn;
    Query.Transaction := Trans;
    
    Conn.DatabaseName := '/tmp/params_demo.db';
    Conn.Open;
    
    // สร้างตาราง
    Trans.StartTransaction;
    Conn.ExecuteDirect(
      'CREATE TABLE IF NOT EXISTS users (' +
      '  id INTEGER PRIMARY KEY,' +
      '  username TEXT,' +
      '  email TEXT,' +
      '  age INTEGER,' +
      '  balance REAL' +
      ')'
    );
    Conn.ExecuteDirect('DELETE FROM users');
    Trans.Commit;
    
    // INSERT พร้อม parameters
    WriteLn(#10'1. INSERT พร้อม Parameters:');
    Trans.StartTransaction;
    Query.SQL.Text := 'INSERT INTO users (username, email, age, balance) VALUES (:name, :email, :age, :balance)';
    
    // ใช้ parameters แทน string concatenation
    Query.ParamByName('name').AsString := 'สมชาย';
    Query.ParamByName('email').AsString := 'somchai@example.com';
    Query.ParamByName('age').AsInteger := 25;
    Query.ParamByName('balance').AsFloat := 1000.50;
    Query.ExecSQL;
    
    // แม้ input มี SQL injection ก็ปลอดภัย
    Query.ParamByName('name').AsString := "'; DROP TABLE users; --";
    Query.ParamByName('email').AsString := 'hacker@evil.com';
    Query.ParamByName('age').AsInteger := 0;
    Query.ParamByName('balance').AsFloat := 0;
    Query.ExecSQL;
    
    Query.ParamByName('name').AsString := 'สมหญิง';
    Query.ParamByName('email').AsString := 'somying@example.com';
    Query.ParamByName('age').AsInteger := 23;
    Query.ParamByName('balance').AsFloat := 2500.00;
    Query.ExecSQL;
    
    Trans.Commit;
    WriteLn('INSERT สำเร็จ (แม้มี SQL injection input)');
    
    // SELECT พร้อม parameters
    WriteLn(#10'2. SELECT พร้อม Parameters:');
    Trans.StartTransaction;
    Query.SQL.Text := 'SELECT * FROM users WHERE age > :min_age ORDER BY age';
    Query.ParamByName('min_age').AsInteger := 20;
    Query.Open;
    
    while not Query.EOF do
    begin
      WriteLn(Format('  %s (%d ปี) - ยอด: ฿%.2f',
                     [Query.FieldByName('username').AsString,
                      Query.FieldByName('age').AsInteger,
                      Query.FieldByName('balance').AsFloat]));
      Query.Next;
    end;
    Query.Close;
    Trans.Commit;
    
    // UPDATE พร้อม parameters
    WriteLn(#10'3. UPDATE พร้อม Parameters:');
    Trans.StartTransaction;
    Query.SQL.Text := 'UPDATE users SET balance = balance + :amount WHERE username = :name';
    Query.ParamByName('amount').AsFloat := 500.0;
    Query.ParamByName('name').AsString := 'สมชาย';
    Query.ExecSQL;
    WriteLn('อัปเดต ', Query.RowsAffected, ' รายการ');
    Trans.Commit;
    
    // ตรวจสอบชนิด parameter ต่างๆ
    WriteLn(#10'4. ชนิด Parameters ต่างๆ:');
    Trans.StartTransaction;
    Conn.ExecuteDirect(
      'CREATE TABLE IF NOT EXISTS test_params (' +
      '  int_val INTEGER, str_val TEXT, float_val REAL,' +
      '  bool_val INTEGER, date_val TEXT, null_val TEXT' +
      ')'
    );
    
    Query.SQL.Text :=
      'INSERT INTO test_params (int_val, str_val, float_val, bool_val, date_val, null_val) ' +
      'VALUES (:i, :s, :f, :b, :d, :n)';
    Query.ParamByName('i').AsInteger := 42;
    Query.ParamByName('s').AsString := 'ทดสอบ';
    Query.ParamByName('f').AsFloat := 3.14;
    Query.ParamByName('b').AsBoolean := True;
    Query.ParamByName('d').AsDateTime := Now;
    Query.ParamByName('n').Clear;  // NULL
    Query.ExecSQL;
    
    Trans.Commit;
    WriteLn('INSERT ด้วย parameter ชนิดต่างๆ สำเร็จ');
    
    Conn.Close;
  finally
    Query.Free;
    Trans.Free;
    Conn.Free;
  end;
end;

begin
  DemoSQLInjection;
  DemoParameters;
  ReadLn;
end.
```

---

## 30.7 Database Migration

```pascal
program DatabaseMigration;
{$mode objfpc}{$H+}

uses
  SysUtils, SQLite3Conn, SQLDB, DB;

type
  TMigration = class
  private
    FConn: TSQLite3Connection;
    FTrans: TSQLTransaction;
    FQuery: TSQLQuery;
    
    procedure EnsureMigrationTable;
    function GetCurrentVersion: Integer;
    procedure SetVersion(Version: Integer);
    function MigrationExists(Version: Integer): Boolean;
    procedure RunMigration(Version: Integer; const SQL: String; const Description: String);
  public
    constructor Create(Conn: TSQLite3Connection; Trans: TSQLTransaction);
    destructor Destroy; override;
    procedure MigrateToLatest;
    function GetVersion: Integer;
  end;

constructor TMigration.Create(Conn: TSQLite3Connection; Trans: TSQLTransaction);
begin
  inherited Create;
  FConn := Conn;
  FTrans := Trans;
  FQuery := TSQLQuery.Create(nil);
  FQuery.DataBase := FConn;
  FQuery.Transaction := FTrans;
  EnsureMigrationTable;
end;

destructor TMigration.Destroy;
begin
  FQuery.Free;
  inherited Destroy;
end;

procedure TMigration.EnsureMigrationTable;
begin
  FTrans.StartTransaction;
  try
    FConn.ExecuteDirect(
      'CREATE TABLE IF NOT EXISTS schema_migrations (' +
      '  version INTEGER PRIMARY KEY,' +
      '  description TEXT,' +
      '  applied_at TEXT DEFAULT (datetime("now","localtime"))' +
      ')'
    );
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

function TMigration.GetCurrentVersion: Integer;
begin
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'SELECT COALESCE(MAX(version), 0) AS ver FROM schema_migrations';
    FQuery.Open;
    Result := FQuery.FieldByName('ver').AsInteger;
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

function TMigration.MigrationExists(Version: Integer): Boolean;
begin
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'SELECT COUNT(*) AS cnt FROM schema_migrations WHERE version = :ver';
    FQuery.ParamByName('ver').AsInteger := Version;
    FQuery.Open;
    Result := FQuery.FieldByName('cnt').AsInteger > 0;
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

procedure TMigration.SetVersion(Version: Integer);
begin
  // จะถูกเรียกใน RunMigration
end;

procedure TMigration.RunMigration(Version: Integer; const SQL: String; const Description: String);
begin
  if MigrationExists(Version) then
  begin
    WriteLn(Format('  Migration v%d "%s" - ข้ามแล้ว', [Version, Description]));
    Exit;
  end;
  
  FTrans.StartTransaction;
  try
    // รัน migration SQL
    FConn.ExecuteDirect(SQL);
    
    // บันทึก version
    FQuery.SQL.Text := 'INSERT INTO schema_migrations (version, description) VALUES (:ver, :desc)';
    FQuery.ParamByName('ver').AsInteger := Version;
    FQuery.ParamByName('desc').AsString := Description;
    FQuery.ExecSQL;
    
    FTrans.Commit;
    WriteLn(Format('  Migration v%d "%s" - สำเร็จ', [Version, Description]));
  except
    on E: Exception do
    begin
      FTrans.Rollback;
      WriteLn(Format('  Migration v%d "%s" - ล้มเหลว: %s', [Version, Description, E.Message]));
      raise;
    end;
  end;
end;

procedure TMigration.MigrateToLatest;
begin
  WriteLn('เริ่ม Migration...');
  
  // Migration 1: สร้างตาราง users
  RunMigration(1,
    'CREATE TABLE IF NOT EXISTS users (' +
    '  id INTEGER PRIMARY KEY AUTOINCREMENT,' +
    '  username TEXT NOT NULL UNIQUE,' +
    '  email TEXT NOT NULL,' +
    '  created_at TEXT DEFAULT (datetime("now","localtime"))' +
    ')',
    'Create users table'
  );
  
  // Migration 2: เพิ่มคอลัมน์ age
  RunMigration(2,
    'ALTER TABLE users ADD COLUMN age INTEGER DEFAULT 0',
    'Add age column to users'
  );
  
  // Migration 3: สร้างตาราง products
  RunMigration(3,
    'CREATE TABLE IF NOT EXISTS products (' +
    '  id INTEGER PRIMARY KEY AUTOINCREMENT,' +
    '  name TEXT NOT NULL,' +
    '  price REAL NOT NULL DEFAULT 0,' +
    '  stock INTEGER NOT NULL DEFAULT 0,' +
    '  created_at TEXT DEFAULT (datetime("now","localtime"))' +
    ')',
    'Create products table'
  );
  
  // Migration 4: เพิ่ม index
  RunMigration(4,
    'CREATE INDEX IF NOT EXISTS idx_users_email ON users(email)',
    'Add index on users.email'
  );
  
  // Migration 5: เพิ่มคอลัมน์ discount ใน products
  RunMigration(5,
    'ALTER TABLE products ADD COLUMN discount REAL DEFAULT 0',
    'Add discount column to products'
  );
  
  WriteLn('Migration เสร็จสิ้น');
end;

function TMigration.GetVersion: Integer;
begin
  Result := GetCurrentVersion;
end;

begin
  WriteLn('=== Database Migration ===');
  
  var Conn := TSQLite3Connection.Create(nil);
  var Trans := TSQLTransaction.Create(nil);
  try
    Trans.DataBase := Conn;
    Conn.DatabaseName := '/tmp/migration_demo.db';
    Conn.Open;
    
    var Migration := TMigration.Create(Conn, Trans);
    try
      // รัน migration
      Migration.MigrateToLatest;
      WriteLn(#10'Schema Version: ', Migration.GetVersion);
      
      // รัน migration อีกครั้ง (ควรข้ามทั้งหมด)
      WriteLn(#10'รัน Migration อีกครั้ง (ควรข้ามทั้งหมด):');
      Migration.MigrateToLatest;
      
    finally
      Migration.Free;
    end;
    
    Conn.Close;
  finally
    Trans.Free;
    Conn.Free;
  end;
  
  ReadLn;
end.
```

---

## 30.8 Export to CSV

```pascal
program ExportToCSV;
{$mode objfpc}{$H+}

uses
  SysUtils, SQLite3Conn, SQLDB, DB, Classes;

procedure ExportQueryToCSV(Query: TSQLQuery; const FileName: String);
var
  F: TextFile;
  I: Integer;
  Row: String;
begin
  AssignFile(F, FileName);
  Rewrite(F);
  try
    // Header row
    Row := '';
    for I := 0 to Query.Fields.Count - 1 do
    begin
      if I > 0 then Row := Row + ',';
      Row := Row + '"' + Query.Fields[I].FieldName + '"';
    end;
    WriteLn(F, Row);
    
    // Data rows
    Query.First;
    while not Query.EOF do
    begin
      Row := '';
      for I := 0 to Query.Fields.Count - 1 do
      begin
        if I > 0 then Row := Row + ',';
        var Value := Query.Fields[I].AsString;
        // escape quotes
        Value := StringReplace(Value, '"', '""', [rfReplaceAll]);
        Row := Row + '"' + Value + '"';
      end;
      WriteLn(F, Row);
      Query.Next;
    end;
    
    WriteLn('Export เสร็จ: ', FileName);
  finally
    CloseFile(F);
  end;
end;

procedure DemoExport;
var
  Conn: TSQLite3Connection;
  Trans: TSQLTransaction;
  Query: TSQLQuery;
begin
  WriteLn('=== Export to CSV ===');
  
  Conn := TSQLite3Connection.Create(nil);
  Trans := TSQLTransaction.Create(nil);
  Query := TSQLQuery.Create(nil);
  try
    Trans.DataBase := Conn;
    Query.DataBase := Conn;
    Query.Transaction := Trans;
    
    Conn.DatabaseName := '/tmp/export_demo.db';
    Conn.Open;
    
    // สร้างข้อมูลตัวอย่าง
    Trans.StartTransaction;
    Conn.ExecuteDirect('CREATE TABLE IF NOT EXISTS sales (' +
                       '  id INTEGER PRIMARY KEY, product TEXT, qty INTEGER, price REAL, date TEXT)');
    Conn.ExecuteDirect('DELETE FROM sales');
    Conn.ExecuteDirect('INSERT INTO sales VALUES (1,"กาแฟ",10,50.0,"2024-01-01")');
    Conn.ExecuteDirect('INSERT INTO sales VALUES (2,"ชา",5,30.0,"2024-01-01")');
    Conn.ExecuteDirect('INSERT INTO sales VALUES (3,"น้ำส้ม",8,40.0,"2024-01-02")');
    Conn.ExecuteDirect('INSERT INTO sales VALUES (4,"กาแฟ",15,50.0,"2024-01-02")');
    Conn.ExecuteDirect('INSERT INTO sales VALUES (5,"ชา",12,30.0,"2024-01-03")');
    Trans.Commit;
    
    // Export ทั้งหมด
    Trans.StartTransaction;
    Query.SQL.Text := 'SELECT * FROM sales ORDER BY id';
    Query.Open;
    ExportQueryToCSV(Query, '/tmp/sales_all.csv');
    Query.Close;
    Trans.Commit;
    
    // Export แบบ summary
    Trans.StartTransaction;
    Query.SQL.Text := 'SELECT product, SUM(qty) AS total_qty, SUM(qty*price) AS total_sales ' +
                      'FROM sales GROUP BY product ORDER BY total_sales DESC';
    Query.Open;
    ExportQueryToCSV(Query, '/tmp/sales_summary.csv');
    
    WriteLn(#10'Summary:');
    while not Query.EOF do
    begin
      WriteLn(Format('  %-10s: %d ชิ้น, ฿%.2f',
                     [Query.FieldByName('product').AsString,
                      Query.FieldByName('total_qty').AsInteger,
                      Query.FieldByName('total_sales').AsFloat]));
      Query.Next;
    end;
    Query.Close;
    Trans.Commit;
    
    Conn.Close;
  finally
    Query.Free;
    Trans.Free;
    Conn.Free;
  end;
end;

begin
  DemoExport;
  ReadLn;
end.
```

---

## 30.9 โปรแกรมตัวอย่างสมบูรณ์: Student Database App

```pascal
// ไฟล์หลัก: StudentDB.pas
// โปรแกรมบริหารข้อมูลนักเรียนสมบูรณ์
// รองรับ: Add, Edit, Delete, Search, Sort, Export CSV

program StudentDatabaseApp;
{$mode objfpc}{$H+}

uses
  SysUtils, SQLite3Conn, SQLDB, DB, Classes, StrUtils;

const
  DB_FILE = '/tmp/student_app.db';
  APP_VERSION = '1.0.0';
  MIN_GPA = 0.0;
  MAX_GPA = 4.0;
  MIN_AGE = 15;
  MAX_AGE = 100;

type
  TStudentRecord = record
    ID: Integer;
    Name: String;
    Age: Integer;
    GPA: Double;
    Major: String;
    Phone: String;
    Email: String;
    EnrollYear: Integer;
    IsActive: Boolean;
  end;
  
  TSortOrder = (soName, soAge, soGPA, soMajor, soEnrollYear);
  
  TStudentApp = class
  private
    FConn: TSQLite3Connection;
    FTrans: TSQLTransaction;
    FQuery: TSQLQuery;
    FCurrentSort: TSortOrder;
    FSortDesc: Boolean;
    
    procedure InitDatabase;
    procedure AddSampleData;
    function RecordToStudent(Q: TSQLQuery): TStudentRecord;
    function ValidateStudent(const S: TStudentRecord; out Error: String): Boolean;
    function GetSortSQL: String;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    // CRUD Operations
    function AddStudent(const S: TStudentRecord): Integer;
    function UpdateStudent(const S: TStudentRecord): Boolean;
    function DeleteStudent(ID: Integer): Boolean;
    function GetStudent(ID: Integer; out S: TStudentRecord): Boolean;
    
    // Query Operations
    procedure ListStudents(const Filter: String = '');
    procedure ListByMajor(const Major: String);
    procedure ListTopStudents(N: Integer);
    procedure SearchStudents(const Keyword: String);
    
    // Statistics
    procedure ShowStats;
    procedure ShowMajorReport;
    
    // Sort
    procedure SetSort(Order: TSortOrder; Desc: Boolean = False);
    
    // Export
    procedure ExportToCSV(const FileName: String);
    
    // Batch Operations
    procedure UpdateGraduationStatus;
    procedure ArchiveOldStudents(Year: Integer);
    
    // Menu
    procedure RunMenu;
    procedure ShowHelp;
  end;

constructor TStudentApp.Create;
begin
  inherited Create;
  
  FConn := TSQLite3Connection.Create(nil);
  FTrans := TSQLTransaction.Create(nil);
  FQuery := TSQLQuery.Create(nil);
  
  FTrans.DataBase := FConn;
  FQuery.DataBase := FConn;
  FQuery.Transaction := FTrans;
  
  FCurrentSort := soName;
  FSortDesc := False;
  
  FConn.DatabaseName := DB_FILE;
  FConn.Open;
  FConn.ExecuteDirect('PRAGMA journal_mode=WAL');
  FConn.ExecuteDirect('PRAGMA foreign_keys=ON');
  
  InitDatabase;
end;

destructor TStudentApp.Destroy;
begin
  if FConn.Connected then
    FConn.Close;
  FQuery.Free;
  FTrans.Free;
  FConn.Free;
  inherited Destroy;
end;

procedure TStudentApp.InitDatabase;
begin
  FTrans.StartTransaction;
  try
    FConn.ExecuteDirect(
      'CREATE TABLE IF NOT EXISTS majors (' +
      '  id INTEGER PRIMARY KEY AUTOINCREMENT,' +
      '  name TEXT NOT NULL UNIQUE' +
      ')'
    );
    
    FConn.ExecuteDirect(
      'CREATE TABLE IF NOT EXISTS students (' +
      '  id INTEGER PRIMARY KEY AUTOINCREMENT,' +
      '  name TEXT NOT NULL,' +
      '  age INTEGER NOT NULL CHECK(age BETWEEN 15 AND 100),' +
      '  gpa REAL NOT NULL DEFAULT 0.0 CHECK(gpa BETWEEN 0 AND 4.0),' +
      '  major_id INTEGER REFERENCES majors(id),' +
      '  phone TEXT,' +
      '  email TEXT,' +
      '  enroll_year INTEGER NOT NULL DEFAULT ' + IntToStr(CurrentYear) + ',' +
      '  is_active INTEGER NOT NULL DEFAULT 1,' +
      '  created_at TEXT DEFAULT (datetime("now","localtime")),' +
      '  updated_at TEXT DEFAULT (datetime("now","localtime"))' +
      ')'
    );
    
    FConn.ExecuteDirect('CREATE INDEX IF NOT EXISTS idx_students_name ON students(name)');
    FConn.ExecuteDirect('CREATE INDEX IF NOT EXISTS idx_students_major ON students(major_id)');
    FConn.ExecuteDirect('CREATE INDEX IF NOT EXISTS idx_students_gpa ON students(gpa)');
    
    // เพิ่ม majors เริ่มต้น
    FConn.ExecuteDirect('INSERT OR IGNORE INTO majors(name) VALUES ("วิศวกรรมคอมพิวเตอร์")');
    FConn.ExecuteDirect('INSERT OR IGNORE INTO majors(name) VALUES ("วิทยาการคอมพิวเตอร์")');
    FConn.ExecuteDirect('INSERT OR IGNORE INTO majors(name) VALUES ("เทคโนโลยีสารสนเทศ")');
    FConn.ExecuteDirect('INSERT OR IGNORE INTO majors(name) VALUES ("วิศวกรรมซอฟต์แวร์")');
    FConn.ExecuteDirect('INSERT OR IGNORE INTO majors(name) VALUES ("ปัญญาประดิษฐ์")');
    
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

procedure TStudentApp.AddSampleData;
type
  TData = record
    Name: String[60];
    Age: Integer;
    GPA: Double;
    Major: String[60];
    Phone: String[20];
    Email: String[60];
    Year: Integer;
  end;
const
  SampleData: array[0..9] of TData = (
    (Name:'สมชาย ใจดี';    Age:20; GPA:3.50; Major:'วิศวกรรมคอมพิวเตอร์'; Phone:'081-111-1111'; Email:'somchai@email.com';    Year:2022),
    (Name:'สมหญิง งาม';    Age:21; GPA:3.80; Major:'วิทยาการคอมพิวเตอร์';  Phone:'082-222-2222'; Email:'somying@email.com';    Year:2021),
    (Name:'วิชัย เก่ง';    Age:22; GPA:3.20; Major:'เทคโนโลยีสารสนเทศ';    Phone:'083-333-3333'; Email:'wichai@email.com';     Year:2020),
    (Name:'อนันต์ สุข';    Age:20; GPA:3.90; Major:'วิศวกรรมคอมพิวเตอร์';  Phone:'084-444-4444'; Email:'anant@email.com';      Year:2022),
    (Name:'บุญมี ดี';      Age:23; GPA:2.80; Major:'วิทยาการคอมพิวเตอร์';  Phone:'085-555-5555'; Email:'boonmee@email.com';    Year:2019),
    (Name:'ประยุทธ์ แกร่ง'; Age:24; GPA:3.65; Major:'วิศวกรรมซอฟต์แวร์';  Phone:'086-666-6666'; Email:'prayuth@email.com';    Year:2018),
    (Name:'สุวรรณ ทอง';    Age:21; GPA:3.40; Major:'ปัญญาประดิษฐ์';        Phone:'087-777-7777'; Email:'suwan@email.com';      Year:2021),
    (Name:'มณี สวย';      Age:19; GPA:3.95; Major:'วิทยาการคอมพิวเตอร์';  Phone:'088-888-8888'; Email:'manee@email.com';      Year:2023),
    (Name:'ชัยวัฒน์ กล้า'; Age:22; GPA:3.10; Major:'เทคโนโลยีสารสนเทศ';  Phone:'089-999-9999'; Email:'chaiwat@email.com';    Year:2020),
    (Name:'นฤมล นิ่ม';     Age:20; GPA:3.75; Major:'วิศวกรรมซอฟต์แวร์';   Phone:'090-000-0000'; Email:'narumon@email.com';    Year:2022)
  );
var
  I: Integer;
  S: TStudentRecord;
  Err: String;
begin
  FTrans.StartTransaction;
  FQuery.SQL.Text := 'SELECT COUNT(*) AS cnt FROM students';
  FQuery.Open;
  var Count := FQuery.FieldByName('cnt').AsInteger;
  FQuery.Close;
  FTrans.Commit;
  
  if Count > 0 then Exit;  // มีข้อมูลแล้ว
  
  WriteLn('เพิ่มข้อมูลตัวอย่าง...');
  for I := 0 to High(SampleData) do
  begin
    S.Name := SampleData[I].Name;
    S.Age := SampleData[I].Age;
    S.GPA := SampleData[I].GPA;
    S.Major := SampleData[I].Major;
    S.Phone := SampleData[I].Phone;
    S.Email := SampleData[I].Email;
    S.EnrollYear := SampleData[I].Year;
    S.IsActive := True;
    
    if ValidateStudent(S, Err) then
      AddStudent(S)
    else
      WriteLn('ข้อมูลผิดพลาด: ', Err);
  end;
  WriteLn('เพิ่มข้อมูลตัวอย่างเสร็จ');
end;

function TStudentApp.RecordToStudent(Q: TSQLQuery): TStudentRecord;
begin
  Result.ID := Q.FieldByName('id').AsInteger;
  Result.Name := Q.FieldByName('name').AsString;
  Result.Age := Q.FieldByName('age').AsInteger;
  Result.GPA := Q.FieldByName('gpa').AsFloat;
  Result.Major := Q.FieldByName('major_name').AsString;
  Result.Phone := Q.FieldByName('phone').AsString;
  Result.Email := Q.FieldByName('email').AsString;
  Result.EnrollYear := Q.FieldByName('enroll_year').AsInteger;
  Result.IsActive := Q.FieldByName('is_active').AsBoolean;
end;

function TStudentApp.ValidateStudent(const S: TStudentRecord; out Error: String): Boolean;
begin
  Result := False;
  
  if Trim(S.Name) = '' then begin Error := 'ชื่อต้องไม่ว่างเปล่า'; Exit; end;
  if Length(S.Name) > 100 then begin Error := 'ชื่อยาวเกินไป (สูงสุด 100 ตัวอักษร)'; Exit; end;
  if (S.Age < MIN_AGE) or (S.Age > MAX_AGE) then 
    begin Error := Format('อายุต้องอยู่ระหว่าง %d-%d', [MIN_AGE, MAX_AGE]); Exit; end;
  if (S.GPA < MIN_GPA) or (S.GPA > MAX_GPA) then 
    begin Error := Format('GPA ต้องอยู่ระหว่าง %.1f-%.1f', [MIN_GPA, MAX_GPA]); Exit; end;
  if Trim(S.Major) = '' then begin Error := 'ต้องระบุสาขา'; Exit; end;
  if (S.EnrollYear < 2000) or (S.EnrollYear > CurrentYear) then 
    begin Error := 'ปีที่เข้าเรียนไม่ถูกต้อง'; Exit; end;
  
  Result := True;
  Error := '';
end;

function TStudentApp.GetSortSQL: String;
var
  SortCol: String;
begin
  case FCurrentSort of
    soName: SortCol := 's.name';
    soAge: SortCol := 's.age';
    soGPA: SortCol := 's.gpa';
    soMajor: SortCol := 'm.name';
    soEnrollYear: SortCol := 's.enroll_year';
  end;
  
  Result := ' ORDER BY ' + SortCol;
  if FSortDesc then Result := Result + ' DESC';
  Result := Result + ', s.name ASC';
end;

function TStudentApp.AddStudent(const S: TStudentRecord): Integer;
var
  MajorID: Integer;
begin
  Result := -1;
  
  FTrans.StartTransaction;
  try
    // หรือสร้าง major ถ้าไม่มี
    FQuery.SQL.Text := 'SELECT id FROM majors WHERE name = :major';
    FQuery.ParamByName('major').AsString := S.Major;
    FQuery.Open;
    
    if FQuery.EOF then
    begin
      FQuery.Close;
      FQuery.SQL.Text := 'INSERT INTO majors (name) VALUES (:major)';
      FQuery.ParamByName('major').AsString := S.Major;
      FQuery.ExecSQL;
      
      FQuery.SQL.Text := 'SELECT last_insert_rowid() AS id';
      FQuery.Open;
      MajorID := FQuery.FieldByName('id').AsInteger;
      FQuery.Close;
    end
    else
    begin
      MajorID := FQuery.FieldByName('id').AsInteger;
      FQuery.Close;
    end;
    
    // Insert student
    FQuery.SQL.Text :=
      'INSERT INTO students (name, age, gpa, major_id, phone, email, enroll_year, is_active) ' +
      'VALUES (:name, :age, :gpa, :major_id, :phone, :email, :year, :active)';
    FQuery.ParamByName('name').AsString := S.Name;
    FQuery.ParamByName('age').AsInteger := S.Age;
    FQuery.ParamByName('gpa').AsFloat := S.GPA;
    FQuery.ParamByName('major_id').AsInteger := MajorID;
    FQuery.ParamByName('phone').AsString := S.Phone;
    FQuery.ParamByName('email').AsString := S.Email;
    FQuery.ParamByName('year').AsInteger := S.EnrollYear;
    FQuery.ParamByName('active').AsBoolean := S.IsActive;
    FQuery.ExecSQL;
    
    FQuery.SQL.Text := 'SELECT last_insert_rowid() AS id';
    FQuery.Open;
    Result := FQuery.FieldByName('id').AsInteger;
    FQuery.Close;
    
    FTrans.Commit;
  except
    on E: Exception do
    begin
      FTrans.Rollback;
      WriteLn('เพิ่มนักเรียนไม่ได้: ', E.Message);
    end;
  end;
end;

function TStudentApp.UpdateStudent(const S: TStudentRecord): Boolean;
var
  MajorID: Integer;
begin
  Result := False;
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'SELECT id FROM majors WHERE name = :major';
    FQuery.ParamByName('major').AsString := S.Major;
    FQuery.Open;
    
    if FQuery.EOF then
    begin
      FQuery.Close;
      FQuery.SQL.Text := 'INSERT INTO majors (name) VALUES (:major)';
      FQuery.ParamByName('major').AsString := S.Major;
      FQuery.ExecSQL;
      FQuery.SQL.Text := 'SELECT last_insert_rowid() AS id';
      FQuery.Open;
      MajorID := FQuery.FieldByName('id').AsInteger;
      FQuery.Close;
    end
    else
    begin
      MajorID := FQuery.FieldByName('id').AsInteger;
      FQuery.Close;
    end;
    
    FQuery.SQL.Text :=
      'UPDATE students SET name=:name, age=:age, gpa=:gpa, major_id=:major_id, ' +
      'phone=:phone, email=:email, enroll_year=:year, is_active=:active, ' +
      'updated_at=datetime("now","localtime") WHERE id=:id';
    FQuery.ParamByName('name').AsString := S.Name;
    FQuery.ParamByName('age').AsInteger := S.Age;
    FQuery.ParamByName('gpa').AsFloat := S.GPA;
    FQuery.ParamByName('major_id').AsInteger := MajorID;
    FQuery.ParamByName('phone').AsString := S.Phone;
    FQuery.ParamByName('email').AsString := S.Email;
    FQuery.ParamByName('year').AsInteger := S.EnrollYear;
    FQuery.ParamByName('active').AsBoolean := S.IsActive;
    FQuery.ParamByName('id').AsInteger := S.ID;
    FQuery.ExecSQL;
    
    Result := FQuery.RowsAffected > 0;
    FTrans.Commit;
  except
    on E: Exception do
    begin
      FTrans.Rollback;
      WriteLn('อัปเดตไม่ได้: ', E.Message);
    end;
  end;
end;

function TStudentApp.DeleteStudent(ID: Integer): Boolean;
begin
  Result := False;
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text := 'DELETE FROM students WHERE id = :id';
    FQuery.ParamByName('id').AsInteger := ID;
    FQuery.ExecSQL;
    Result := FQuery.RowsAffected > 0;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

function TStudentApp.GetStudent(ID: Integer; out S: TStudentRecord): Boolean;
begin
  Result := False;
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text :=
      'SELECT s.*, m.name AS major_name FROM students s ' +
      'LEFT JOIN majors m ON s.major_id = m.id WHERE s.id = :id';
    FQuery.ParamByName('id').AsInteger := ID;
    FQuery.Open;
    
    if not FQuery.EOF then
    begin
      S := RecordToStudent(FQuery);
      Result := True;
    end;
    
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

procedure TStudentApp.ListStudents(const Filter: String);
var
  SQL: String;
begin
  FTrans.StartTransaction;
  try
    SQL := 'SELECT s.*, m.name AS major_name FROM students s LEFT JOIN majors m ON s.major_id = m.id';
    
    if Filter <> '' then
      SQL := SQL + ' WHERE (' + Filter + ')';
    
    SQL := SQL + GetSortSQL;
    
    FQuery.SQL.Text := SQL;
    FQuery.Open;
    
    WriteLn(StringOfChar('=', 90));
    WriteLn(Format('%-5s | %-22s | %-4s | %-5s | %-28s | %-6s',
                   ['ID', 'ชื่อ', 'อายุ', 'GPA', 'สาขา', 'ปีที่เข้า']));
    WriteLn(StringOfChar('-', 90));
    
    var Count := 0;
    while not FQuery.EOF do
    begin
      var IsActive := FQuery.FieldByName('is_active').AsBoolean;
      var Status := IfThen(IsActive, '', ' [ไม่ใช้งาน]');
      WriteLn(Format('%-5d | %-22s | %-4d | %-5.2f | %-28s | %-6d%s',
                     [FQuery.FieldByName('id').AsInteger,
                      FQuery.FieldByName('name').AsString,
                      FQuery.FieldByName('age').AsInteger,
                      FQuery.FieldByName('gpa').AsFloat,
                      FQuery.FieldByName('major_name').AsString,
                      FQuery.FieldByName('enroll_year').AsInteger,
                      Status]));
      Inc(Count);
      FQuery.Next;
    end;
    
    WriteLn(StringOfChar('=', 90));
    WriteLn('รวม: ', Count, ' รายการ');
    
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

procedure TStudentApp.ListByMajor(const Major: String);
begin
  ListStudents('m.name LIKE "%' + Major + '%"');
end;

procedure TStudentApp.ListTopStudents(N: Integer);
var
  SQL: String;
begin
  WriteLn(Format(#10'Top %d นักเรียนที่มี GPA สูงสุด:', [N]));
  FTrans.StartTransaction;
  try
    SQL := 'SELECT s.*, m.name AS major_name FROM students s ' +
           'LEFT JOIN majors m ON s.major_id = m.id ' +
           'WHERE s.is_active = 1 ORDER BY s.gpa DESC LIMIT :n';
    FQuery.SQL.Text := SQL;
    FQuery.ParamByName('n').AsInteger := N;
    FQuery.Open;
    
    WriteLn(StringOfChar('-', 55));
    var Rank := 1;
    while not FQuery.EOF do
    begin
      WriteLn(Format('  %2d. %-20s GPA: %.2f (%s)',
                     [Rank,
                      FQuery.FieldByName('name').AsString,
                      FQuery.FieldByName('gpa').AsFloat,
                      FQuery.FieldByName('major_name').AsString]));
      Inc(Rank);
      FQuery.Next;
    end;
    WriteLn(StringOfChar('-', 55));
    
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

procedure TStudentApp.SearchStudents(const Keyword: String);
begin
  WriteLn(Format(#10'ผลการค้นหา "%s":', [Keyword]));
  ListStudents(Format('s.name LIKE "%%%s%%" OR m.name LIKE "%%%s%%"', 
                       [Keyword, Keyword]));
end;

procedure TStudentApp.ShowStats;
begin
  WriteLn(#10'=== สถิติ ===');
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text :=
      'SELECT ' +
      '  COUNT(*) AS total,' +
      '  SUM(CASE WHEN is_active=1 THEN 1 ELSE 0 END) AS active,' +
      '  SUM(CASE WHEN is_active=0 THEN 1 ELSE 0 END) AS inactive,' +
      '  ROUND(AVG(gpa),3) AS avg_gpa,' +
      '  ROUND(MAX(gpa),2) AS max_gpa,' +
      '  ROUND(MIN(gpa),2) AS min_gpa,' +
      '  ROUND(AVG(age),1) AS avg_age,' +
      '  MIN(enroll_year) AS oldest_year,' +
      '  MAX(enroll_year) AS newest_year ' +
      'FROM students';
    FQuery.Open;
    
    WriteLn('จำนวนนักเรียนทั้งหมด : ', FQuery.FieldByName('total').AsInteger);
    WriteLn('  - ยังใช้งานอยู่      : ', FQuery.FieldByName('active').AsInteger);
    WriteLn('  - ไม่ใช้งานแล้ว     : ', FQuery.FieldByName('inactive').AsInteger);
    WriteLn('GPA เฉลี่ย            : ', FQuery.FieldByName('avg_gpa').AsFloat:0:3);
    WriteLn('GPA สูงสุด            : ', FQuery.FieldByName('max_gpa').AsFloat:0:2);
    WriteLn('GPA ต่ำสุด            : ', FQuery.FieldByName('min_gpa').AsFloat:0:2);
    WriteLn('อายุเฉลี่ย            : ', FQuery.FieldByName('avg_age').AsFloat:0:1, ' ปี');
    WriteLn('ปีที่เข้าเรียนเก่าสุด : ', FQuery.FieldByName('oldest_year').AsInteger);
    WriteLn('ปีที่เข้าเรียนใหม่สุด : ', FQuery.FieldByName('newest_year').AsInteger);
    
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

procedure TStudentApp.ShowMajorReport;
begin
  WriteLn(#10'=== รายงานตามสาขา ===');
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text :=
      'SELECT m.name AS major, COUNT(s.id) AS count, ' +
      'ROUND(AVG(s.gpa),3) AS avg_gpa, ' +
      'ROUND(MAX(s.gpa),2) AS max_gpa, ' +
      'ROUND(MIN(s.gpa),2) AS min_gpa ' +
      'FROM students s ' +
      'JOIN majors m ON s.major_id = m.id ' +
      'WHERE s.is_active = 1 ' +
      'GROUP BY m.name ' +
      'ORDER BY count DESC, avg_gpa DESC';
    FQuery.Open;
    
    WriteLn(StringOfChar('-', 70));
    WriteLn(Format('%-35s | %-5s | %-7s | %-7s | %-7s',
                   ['สาขา', 'จำนวน', 'GPA เฉลี่ย', 'GPA สูง', 'GPA ต่ำ']));
    WriteLn(StringOfChar('-', 70));
    
    while not FQuery.EOF do
    begin
      WriteLn(Format('%-35s | %-5d | %-9.3f | %-7.2f | %-7.2f',
                     [FQuery.FieldByName('major').AsString,
                      FQuery.FieldByName('count').AsInteger,
                      FQuery.FieldByName('avg_gpa').AsFloat,
                      FQuery.FieldByName('max_gpa').AsFloat,
                      FQuery.FieldByName('min_gpa').AsFloat]));
      FQuery.Next;
    end;
    
    WriteLn(StringOfChar('-', 70));
    FQuery.Close;
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

procedure TStudentApp.SetSort(Order: TSortOrder; Desc: Boolean);
begin
  FCurrentSort := Order;
  FSortDesc := Desc;
end;

procedure TStudentApp.ExportToCSV(const FileName: String);
var
  F: TextFile;
begin
  WriteLn('Export ไปที่ ', FileName, '...');
  
  AssignFile(F, FileName);
  Rewrite(F);
  
  FTrans.StartTransaction;
  try
    FQuery.SQL.Text :=
      'SELECT s.id, s.name, s.age, s.gpa, m.name AS major, ' +
      's.phone, s.email, s.enroll_year, s.is_active, s.created_at ' +
      'FROM students s LEFT JOIN majors m ON s.major_id = m.id ' +
      'ORDER BY s.name';
    FQuery.Open;
    
    WriteLn(F, '"ID","ชื่อ","อายุ","GPA","สาขา","โทรศัพท์","อีเมล","ปีที่เข้า","สถานะ","วันที่สร้าง"');
    
    var Count := 0;
    while not FQuery.EOF do
    begin
      WriteLn(F, Format('"%d","%s","%d","%.2f","%s","%s","%s","%d","%s","%s"',
                        [FQuery.FieldByName('id').AsInteger,
                         FQuery.FieldByName('name').AsString,
                         FQuery.FieldByName('age').AsInteger,
                         FQuery.FieldByName('gpa').AsFloat,
                         FQuery.FieldByName('major').AsString,
                         FQuery.FieldByName('phone').AsString,
                         FQuery.FieldByName('email').AsString,
                         FQuery.FieldByName('enroll_year').AsInteger,
                         IfThen(FQuery.FieldByName('is_active').AsBoolean, 'active', 'inactive'),
                         FQuery.FieldByName('created_at').AsString]));
      Inc(Count);
      FQuery.Next;
    end;
    
    FQuery.Close;
    FTrans.Commit;
    
    CloseFile(F);
    WriteLn('Export เสร็จ: ', Count, ' รายการ');
    
  except
    on E: Exception do
    begin
      FTrans.Rollback;
      CloseFile(F);
      WriteLn('Export ล้มเหลว: ', E.Message);
    end;
  end;
end;

procedure TStudentApp.UpdateGraduationStatus;
begin
  WriteLn('อัปเดตสถานะการจบการศึกษา...');
  FTrans.StartTransaction;
  try
    // นักเรียนที่เรียน 4 ปีขึ้นไป ถือว่าจบ
    FQuery.SQL.Text :=
      'UPDATE students SET is_active = 0 ' +
      'WHERE enroll_year <= :year AND is_active = 1';
    FQuery.ParamByName('year').AsInteger := CurrentYear - 4;
    FQuery.ExecSQL;
    
    WriteLn('อัปเดต ', FQuery.RowsAffected, ' รายการ');
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

procedure TStudentApp.ArchiveOldStudents(Year: Integer);
begin
  WriteLn(Format('Archive นักเรียนที่เข้าปี %d และก่อนหน้า...', [Year]));
  FTrans.StartTransaction;
  try
    // ในโปรแกรมจริงควร copy ไปตาราง archive ก่อน แล้วค่อยลบ
    FQuery.SQL.Text :=
      'DELETE FROM students WHERE enroll_year <= :year AND is_active = 0';
    FQuery.ParamByName('year').AsInteger := Year;
    FQuery.ExecSQL;
    
    WriteLn('Archive ', FQuery.RowsAffected, ' รายการ');
    FTrans.Commit;
  except
    FTrans.Rollback;
    raise;
  end;
end;

procedure TStudentApp.ShowHelp;
begin
  WriteLn('=== คำสั่งที่รองรับ ===');
  WriteLn('  l         - แสดงรายการทั้งหมด');
  WriteLn('  a         - เพิ่มนักเรียนใหม่');
  WriteLn('  e <id>    - แก้ไขนักเรียน');
  WriteLn('  d <id>    - ลบนักเรียน');
  WriteLn('  g <id>    - ดูข้อมูลนักเรียน');
  WriteLn('  s <คำ>   - ค้นหา');
  WriteLn('  top <n>   - Top N นักเรียน GPA สูงสุด');
  WriteLn('  stat      - สถิติ');
  WriteLn('  report    - รายงานตามสาขา');
  WriteLn('  sort      - เปลี่ยนการเรียงลำดับ');
  WriteLn('  exp       - Export CSV');
  WriteLn('  grad      - อัปเดตสถานะจบการศึกษา');
  WriteLn('  q         - ออก');
  WriteLn('  ?         - ช่วยเหลือ');
end;

procedure TStudentApp.RunMenu;
var
  Cmd: String;
  S: TStudentRecord;
  Err: String;
  ID: Integer;
begin
  WriteLn('=================================================');
  WriteLn('  Student Database App v', APP_VERSION);
  WriteLn('=================================================');
  WriteLn('พิมพ์ ? เพื่อดูคำสั่ง');
  WriteLn;
  
  AddSampleData;
  
  while True do
  begin
    Write(#10'คำสั่ง> ');
    ReadLn(Cmd);
    Cmd := Trim(Cmd);
    
    if Cmd = '' then Continue;
    
    var CmdLower := LowerCase(Cmd);
    var Parts := Cmd.Split([' '], 2);
    
    try
      if CmdLower = 'q' then Break
      
      else if CmdLower = 'l' then
        ListStudents
      
      else if CmdLower = 'stat' then
        ShowStats
      
      else if CmdLower = 'report' then
        ShowMajorReport
      
      else if CmdLower = 'grad' then
        UpdateGraduationStatus
      
      else if CmdLower = 'exp' then
        ExportToCSV('/tmp/students_export.csv')
      
      else if CmdLower = '?' then
        ShowHelp
      
      else if (Length(Parts) >= 1) and (LowerCase(Parts[0]) = 'top') then
      begin
        var N := 5;
        if Length(Parts) >= 2 then
          N := StrToIntDef(Parts[1], 5);
        ListTopStudents(N);
      end
      
      else if (Length(Parts) >= 2) and (LowerCase(Parts[0]) = 's') then
        SearchStudents(Parts[1])
      
      else if (Length(Parts) >= 2) and (LowerCase(Parts[0]) = 'g') then
      begin
        ID := StrToIntDef(Parts[1], 0);
        if ID > 0 then
        begin
          if GetStudent(ID, S) then
          begin
            WriteLn(Format('ID: %d', [S.ID]));
            WriteLn(Format('ชื่อ: %s', [S.Name]));
            WriteLn(Format('อายุ: %d', [S.Age]));
            WriteLn(Format('GPA: %.2f', [S.GPA]));
            WriteLn(Format('สาขา: %s', [S.Major]));
            WriteLn(Format('โทรศัพท์: %s', [S.Phone]));
            WriteLn(Format('อีเมล: %s', [S.Email]));
            WriteLn(Format('ปีที่เข้า: %d', [S.EnrollYear]));
            WriteLn(Format('สถานะ: %s', [IfThen(S.IsActive, 'ใช้งาน', 'ไม่ใช้งาน')]));
          end;
        end;
      end
      
      else if (Length(Parts) >= 2) and (LowerCase(Parts[0]) = 'd') then
      begin
        ID := StrToIntDef(Parts[1], 0);
        if ID > 0 then
        begin
          Write('ยืนยันการลบ ID ', ID, ' (y/n)? ');
          var Confirm: String;
          ReadLn(Confirm);
          if LowerCase(Trim(Confirm)) = 'y' then
          begin
            if DeleteStudent(ID) then
              WriteLn('ลบสำเร็จ')
            else
              WriteLn('ไม่พบ ID: ', ID);
          end;
        end;
      end
      
      else if LowerCase(Parts[0]) = 'a' then
      begin
        WriteLn('--- เพิ่มนักเรียนใหม่ ---');
        Write('ชื่อ: '); ReadLn(S.Name);
        Write('อายุ: '); ReadLn(Cmd); S.Age := StrToIntDef(Cmd, 0);
        Write('GPA (0-4.0): '); ReadLn(Cmd); S.GPA := StrToFloatDef(Cmd, 0);
        Write('สาขา: '); ReadLn(S.Major);
        Write('โทรศัพท์: '); ReadLn(S.Phone);
        Write('อีเมล: '); ReadLn(S.Email);
        Write('ปีที่เข้าเรียน: '); ReadLn(Cmd); S.EnrollYear := StrToIntDef(Cmd, CurrentYear);
        S.IsActive := True;
        
        if ValidateStudent(S, Err) then
        begin
          var NewID := AddStudent(S);
          if NewID > 0 then
            WriteLn('เพิ่มสำเร็จ ID: ', NewID)
          else
            WriteLn('เพิ่มไม่ได้');
        end
        else
          WriteLn('ข้อผิดพลาด: ', Err);
      end
      
      else if (Length(Parts) >= 2) and (LowerCase(Parts[0]) = 'e') then
      begin
        ID := StrToIntDef(Parts[1], 0);
        if (ID > 0) and GetStudent(ID, S) then
        begin
          WriteLn(Format('แก้ไข ID %d (%s)', [ID, S.Name]));
          WriteLn('(Enter เพื่อคงค่าเดิม)');
          
          Write(Format('ชื่อ [%s]: ', [S.Name])); ReadLn(Cmd);
          if Trim(Cmd) <> '' then S.Name := Cmd;
          
          Write(Format('อายุ [%d]: ', [S.Age])); ReadLn(Cmd);
          if Trim(Cmd) <> '' then S.Age := StrToIntDef(Cmd, S.Age);
          
          Write(Format('GPA [%.2f]: ', [S.GPA])); ReadLn(Cmd);
          if Trim(Cmd) <> '' then S.GPA := StrToFloatDef(Cmd, S.GPA);
          
          Write(Format('สาขา [%s]: ', [S.Major])); ReadLn(Cmd);
          if Trim(Cmd) <> '' then S.Major := Cmd;
          
          if ValidateStudent(S, Err) then
          begin
            if UpdateStudent(S) then
              WriteLn('อัปเดตสำเร็จ')
            else
              WriteLn('อัปเดตไม่ได้');
          end
          else
            WriteLn('ข้อผิดพลาด: ', Err);
        end
        else
          WriteLn('ไม่พบ ID: ', Parts[1]);
      end
      
      else
        WriteLn('คำสั่งไม่รู้จัก: ', Cmd, ' (พิมพ์ ? เพื่อดูความช่วยเหลือ)');
        
    except
      on E: Exception do
        WriteLn('ข้อผิดพลาด: ', E.Message);
    end;
  end;
  
  WriteLn('ขอบคุณที่ใช้งาน Student Database App!');
end;

begin
  var App := TStudentApp.Create;
  try
    App.RunMenu;
  except
    on E: Exception do
    begin
      WriteLn('Fatal Error: ', E.Message);
      ReadLn;
    end;
  end;
  App.Free;
end.
```

---

## 30.10 Backup and Restore

```pascal
program BackupRestore;
{$mode objfpc}{$H+}

uses
  SysUtils, SQLite3Conn, SQLDB, Classes;

procedure BackupDatabase(const SourceDB, DestDB: String);
var
  Source, Dest: TFileStream;
begin
  WriteLn('Backup: ', SourceDB, ' -> ', DestDB);
  
  if not FileExists(SourceDB) then
    raise Exception.Create('ไม่พบไฟล์ฐานข้อมูล: ' + SourceDB);
  
  Source := TFileStream.Create(SourceDB, fmOpenRead or fmShareDenyWrite);
  try
    Dest := TFileStream.Create(DestDB, fmCreate);
    try
      Dest.CopyFrom(Source, Source.Size);
    finally
      Dest.Free;
    end;
  finally
    Source.Free;
  end;
  
  WriteLn(Format('Backup สำเร็จ: %.2f KB', [FileSize(DestDB) / 1024]));
end;

procedure RestoreDatabase(const BackupDB, TargetDB: String);
begin
  WriteLn('Restore: ', BackupDB, ' -> ', TargetDB);
  
  if not FileExists(BackupDB) then
    raise Exception.Create('ไม่พบไฟล์ backup: ' + BackupDB);
  
  // สำรองของเดิมไว้ก่อน
  if FileExists(TargetDB) then
  begin
    var OldBackup := TargetDB + '.before_restore';
    CopyFile(PChar(TargetDB), PChar(OldBackup), False);
    WriteLn('สำรองของเดิมไว้ที่: ', OldBackup);
  end;
  
  CopyFile(PChar(BackupDB), PChar(TargetDB), False);
  WriteLn('Restore สำเร็จ');
end;

procedure VacuumDatabase(const DBFile: String);
var
  Conn: TSQLite3Connection;
  Trans: TSQLTransaction;
begin
  WriteLn('VACUUM: ', DBFile, ' (ลดขนาดและ optimize)');
  
  var SizeBefore := FileSize(DBFile);
  
  Conn := TSQLite3Connection.Create(nil);
  Trans := TSQLTransaction.Create(nil);
  try
    Trans.DataBase := Conn;
    Conn.DatabaseName := DBFile;
    Conn.Open;
    
    // VACUUM ต้องไม่อยู่ใน transaction
    Conn.ExecuteDirect('VACUUM');
    
    Conn.Close;
  finally
    Trans.Free;
    Conn.Free;
  end;
  
  var SizeAfter := FileSize(DBFile);
  WriteLn(Format('VACUUM เสร็จ: %.1f KB -> %.1f KB (ลด %.1f KB)',
                 [SizeBefore / 1024, SizeAfter / 1024, (SizeBefore - SizeAfter) / 1024]));
end;

begin
  WriteLn('=== Backup and Restore ===');
  
  var DBFile := '/tmp/main.db';
  var BackupFile := '/tmp/backup_' + FormatDateTime('yyyymmdd_hhnnss', Now) + '.db';
  
  // ตรวจสอบว่ามีไฟล์ DB
  if not FileExists(DBFile) then
  begin
    // สร้าง database เปล่า
    var Conn := TSQLite3Connection.Create(nil);
    var Trans := TSQLTransaction.Create(nil);
    Trans.DataBase := Conn;
    Conn.DatabaseName := DBFile;
    Conn.Open;
    Trans.StartTransaction;
    Conn.ExecuteDirect('CREATE TABLE test (id INTEGER, data TEXT)');
    Conn.ExecuteDirect('INSERT INTO test VALUES (1, "ทดสอบ")');
    Trans.Commit;
    Conn.Close;
    Trans.Free;
    Conn.Free;
    WriteLn('สร้าง database เปล่าแล้ว');
  end;
  
  // Backup
  try
    BackupDatabase(DBFile, BackupFile);
  except
    on E: Exception do
      WriteLn('Backup ล้มเหลว: ', E.Message);
  end;
  
  // VACUUM
  try
    VacuumDatabase(DBFile);
  except
    on E: Exception do
      WriteLn('VACUUM ล้มเหลว: ', E.Message);
  end;
  
  // Restore (ถ้าต้องการ)
  // RestoreDatabase(BackupFile, DBFile);
  
  ReadLn;
end.
```

---

## 30.11 แบบฝึกหัด

### แบบฝึกหัดที่ 1: Library Management
สร้างระบบจัดการห้องสมุด:
- ตาราง: Books, Members, Loans
- CRUD สำหรับ Books และ Members
- ยืม/คืนหนังสือ
- ตรวจสอบสถานะหนังสือ
- รายงาน: หนังสือที่ยืมมากที่สุด

### แบบฝึกหัดที่ 2: Shopping Cart
สร้างระบบตะกร้าสินค้า:
- Products, Customers, Orders, OrderItems
- เพิ่ม/ลบสินค้าในตะกร้า
- คำนวณราคา + ภาษี + ส่วนลด
- สร้าง Invoice
- บันทึก Order history

### แบบฝึกหัดที่ 3: Contact Manager
สร้างระบบจัดการ contacts:
- Contacts, Groups, ContactGroups
- เพิ่ม contact พร้อมหลาย phone numbers
- จัดกลุ่ม contacts
- ค้นหาขั้นสูง
- Export vCard format

### แบบฝึกหัดที่ 4: Expense Tracker
สร้างระบบติดตามค่าใช้จ่าย:
- Categories, Transactions
- เพิ่มรายรับ/รายจ่าย
- รายงานประจำเดือน
- กราฟแท่ง (text-based)
- Budget alerts

### แบบฝึกหัดที่ 5: Task Management
สร้างระบบ To-Do/Task:
- Tasks, Tags, TaskTags
- Priority levels
- Due dates with reminders
- Sub-tasks
- Progress tracking
- Statistics dashboard

### แบบฝึกหัดที่ 6: Employee Database
สร้างระบบ HR:
- Employees, Departments, Positions
- Salary history
- Leave tracking
- Performance reviews
- Org chart (nested data)

### แบบฝึกหัดที่ 7: Inventory System
สร้างระบบ inventory:
- Products, Categories, Suppliers
- Stock in/out transactions
- Auto reorder point alerts
- Stock valuation (FIFO/LIFO)
- Monthly reports

### แบบฝึกหัดที่ 8: Restaurant POS
สร้างระบบร้านอาหาร:
- Menu items, Categories
- Orders, Order items
- Tables management
- Bill generation
- Daily sales report

### แบบฝึกหัดที่ 9: Grade Calculator
สร้างระบบคะแนนนักเรียน:
- Students, Subjects, Grades
- GPA calculation
- Grade distribution report
- Top/Bottom students
- Trend analysis

### แบบฝึกหัดที่ 10: Blog System
สร้างระบบ blog แบบ offline:
- Posts, Categories, Tags, Comments
- Markdown to HTML (basic)
- Search full-text
- Related posts
- Draft management

### แบบฝึกหัดที่ 11: Password Manager
สร้างระบบ password manager:
- Sites, Credentials
- Encryption สำหรับ passwords
- Password generator
- Last used tracking
- Security audit

### แบบฝึกหัดที่ 12: Quiz App
สร้างแอป Quiz:
- Questions, Answers, Categories
- Multiple choice
- Score tracking
- High scores
- Import/Export questions

### แบบฝึกหัดที่ 13: Time Tracker
สร้างระบบติดตามเวลา:
- Projects, Tasks, TimeEntries
- Start/stop timer
- Daily/weekly reports
- Billable hours
- CSV export

### แบบฝึกหัดที่ 14: Document Manager
สร้างระบบจัดการเอกสาร:
- Documents, Tags, Categories
- Full-text search
- Version history
- Recently accessed
- Favorites

### แบบฝึกหัดที่ 15: School Management System
สร้างระบบโรงเรียนสมบูรณ์:
- Students, Teachers, Classes, Subjects
- Enrollment management
- Attendance tracking
- Grade books
- Parent notifications
- Reports: transcript, attendance, grade statistics
- GUI ด้วย Lazarus forms

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **SQLite Overview** - ข้อดีและการใช้งาน
2. **TSQLite3Connection** - การเชื่อมต่อและตั้งค่า
3. **TSQLTransaction** - Commit, Rollback
4. **TSQLQuery** - SELECT, INSERT, UPDATE, DELETE
5. **CRUD Operations** - Create, Read, Update, Delete พร้อม error handling
6. **Parameterized Queries** - ป้องกัน SQL Injection
7. **Database Migration** - จัดการ schema changes
8. **Export to CSV** - ส่งออกข้อมูล
9. **Student Database App** - โปรแกรมสมบูรณ์พร้อม menu
10. **Backup/Restore/Vacuum** - ดูแลรักษาฐานข้อมูล

### Best Practices:
- ใช้ Parameterized Queries เสมอ (ป้องกัน SQL Injection)
- ใช้ Transaction ครอบ operations ที่ต้องทำพร้อมกัน
- ตรวจสอบ RowsAffected หลัง UPDATE/DELETE
- ใช้ Migration สำหรับ schema changes
- Backup ก่อนทำ destructive operations
- ตั้งค่า PRAGMA foreign_keys = ON เสมอ
- ใช้ PRAGMA journal_mode = WAL สำหรับ performance
