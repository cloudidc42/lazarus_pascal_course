# ตอนที่ 31: การเชื่อมต่อ MySQL/MariaDB

## บทนำ

MySQL และ MariaDB เป็นระบบจัดการฐานข้อมูลเชิงสัมพันธ์ (RDBMS) ที่ได้รับความนิยมสูงสุดในโลก โดย MySQL พัฒนาโดย Oracle Corporation ส่วน MariaDB เป็น fork ของ MySQL ที่พัฒนาโดยชุมชนและยังคงเข้ากันได้กับ MySQL ในบทนี้เราจะเรียนรู้การเชื่อมต่อและใช้งานฐานข้อมูลเหล่านี้ใน Lazarus/Pascal

## ภาพรวม MySQL/MariaDB

### ลักษณะเด่น
- **Open Source**: ใช้งานได้ฟรีสำหรับงาน open source
- **ประสิทธิภาพสูง**: รองรับ transaction จำนวนมากพร้อมกัน
- **หลายแพลตฟอร์ม**: ทำงานได้บน Windows, Linux, macOS
- **รองรับ SQL มาตรฐาน**: ใช้ SQL ตามมาตรฐาน ANSI
- **Stored Procedures**: รองรับการเขียนโปรแกรมในฐานข้อมูล
- **Replication**: รองรับการทำ Master-Slave replication

### Storage Engines
```
InnoDB  - รองรับ ACID transactions, foreign keys (แนะนำ)
MyISAM  - ประสิทธิภาพสูง แต่ไม่รองรับ transactions
MEMORY  - เก็บข้อมูลใน RAM เร็วมากแต่หายเมื่อปิดเครื่อง
Archive - บีบอัดข้อมูล เหมาะสำหรับเก็บ log
```

---

## การติดตั้ง MySQL Connector สำหรับ Lazarus

### วิธีที่ 1: ใช้ ZEOS Library (แนะนำ)

ZEOS เป็น library ที่รองรับฐานข้อมูลหลายประเภทรวมถึง MySQL, PostgreSQL, SQLite และอื่นๆ

**ขั้นตอนการติดตั้ง ZEOS:**

1. ดาวน์โหลด ZEOS จาก https://sourceforge.net/projects/zeoslib/
2. แตกไฟล์ไปยังโฟลเดอร์ที่ต้องการ เช่น `C:\lazarus\components\zeos`
3. เปิด Lazarus IDE
4. ไปที่ Package > Open Package File (.lpk)
5. เลือกไฟล์ `zcomponent.lpk` ในโฟลเดอร์ packages ของ ZEOS
6. คลิก Compile แล้ว Install
7. รีสตาร์ท Lazarus

**ขั้นตอนการติดตั้ง MySQL Client Library:**

สำหรับ Windows:
```
1. ดาวน์โหลด MySQL Connector/C จาก dev.mysql.com
2. ติดตั้งและคัดลอกไฟล์ libmysql.dll ไปไว้ในโฟลเดอร์โปรแกรม
   หรือ C:\Windows\System32
```

สำหรับ Linux:
```bash
# Ubuntu/Debian
sudo apt-get install libmysqlclient-dev

# Fedora/RHEL
sudo dnf install mysql-devel

# ตรวจสอบการติดตั้ง
find /usr -name "libmysqlclient*"
```

### วิธีที่ 2: ใช้ SQLdb (มากับ Lazarus)

SQLdb เป็น component ที่มาพร้อมกับ Lazarus โดยไม่ต้องติดตั้งเพิ่ม

**การเพิ่ม units:**
```pascal
uses
  mysql57conn,  // สำหรับ MySQL 5.7
  sqldb,        // สำหรับ TQuery, TTransaction
  db;           // สำหรับ TDataSource, TField
```

---

## การสร้างการเชื่อมต่อ

### การเชื่อมต่อด้วย ZConnection (ZEOS)

```pascal
unit Unit1;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  ZConnection, ZDataSet, ZDbcIntfs;

type
  TForm1 = class(TForm)
  private
    FConnection: TZConnection;
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
    function Connect: Boolean;
    procedure Disconnect;
  end;

implementation

constructor TForm1.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  
  // สร้าง ZConnection
  FConnection := TZConnection.Create(Self);
  FConnection.Protocol := 'mysql';         // ระบุประเภทฐานข้อมูล
  FConnection.HostName := 'localhost';     // ที่อยู่เซิร์ฟเวอร์
  FConnection.Port := 3306;               // พอร์ต (ค่าเริ่มต้นของ MySQL)
  FConnection.Database := 'mydb';         // ชื่อฐานข้อมูล
  FConnection.User := 'root';             // ชื่อผู้ใช้
  FConnection.Password := 'password';     // รหัสผ่าน
  FConnection.Properties.Values['codepage'] := 'UTF8';  // encoding
end;

destructor TForm1.Destroy;
begin
  if FConnection.Connected then
    FConnection.Disconnect;
  FConnection.Free;
  inherited Destroy;
end;

function TForm1.Connect: Boolean;
begin
  Result := False;
  try
    FConnection.Connect;
    Result := True;
    ShowMessage('เชื่อมต่อฐานข้อมูลสำเร็จ!');
  except
    on E: Exception do
      ShowMessage('ไม่สามารถเชื่อมต่อได้: ' + E.Message);
  end;
end;

procedure TForm1.Disconnect;
begin
  if FConnection.Connected then
  begin
    FConnection.Disconnect;
    ShowMessage('ตัดการเชื่อมต่อแล้ว');
  end;
end;

end.
```

### การเชื่อมต่อด้วย TMySQL57Connection (SQLdb)

```pascal
unit MySQLConnection;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, mysql57conn, sqldb, db;

type
  TDatabaseManager = class
  private
    FConnection: TMySQL57Connection;
    FTransaction: TSQLTransaction;
    FQuery: TSQLQuery;
  public
    constructor Create;
    destructor Destroy; override;
    
    function Connect(const Host, Database, User, Password: string;
                     Port: Integer = 3306): Boolean;
    procedure Disconnect;
    function IsConnected: Boolean;
    
    property Connection: TMySQL57Connection read FConnection;
    property Transaction: TSQLTransaction read FTransaction;
    property Query: TSQLQuery read FQuery;
  end;

implementation

constructor TDatabaseManager.Create;
begin
  inherited Create;
  
  // สร้าง Connection object
  FConnection := TMySQL57Connection.Create(nil);
  
  // สร้าง Transaction object
  FTransaction := TSQLTransaction.Create(nil);
  FTransaction.DataBase := FConnection;
  
  // สร้าง Query object
  FQuery := TSQLQuery.Create(nil);
  FQuery.DataBase := FConnection;
  FQuery.Transaction := FTransaction;
end;

destructor TDatabaseManager.Destroy;
begin
  if IsConnected then
    Disconnect;
  FQuery.Free;
  FTransaction.Free;
  FConnection.Free;
  inherited Destroy;
end;

function TDatabaseManager.Connect(const Host, Database, User, Password: string;
                                   Port: Integer = 3306): Boolean;
begin
  Result := False;
  try
    FConnection.HostName := Host;
    FConnection.DatabaseName := Database;
    FConnection.UserName := User;
    FConnection.Password := Password;
    FConnection.CharSet := 'utf8mb4';  // รองรับอักษรไทยและ emoji
    
    FConnection.Connected := True;
    Result := True;
  except
    on E: Exception do
    begin
      WriteLn('Connection Error: ' + E.Message);
      Result := False;
    end;
  end;
end;

procedure TDatabaseManager.Disconnect;
begin
  if FTransaction.Active then
    FTransaction.Rollback;
  if FConnection.Connected then
    FConnection.Connected := False;
end;

function TDatabaseManager.IsConnected: Boolean;
begin
  Result := Assigned(FConnection) and FConnection.Connected;
end;

end.
```

---

## รูปแบบ Connection String

### MySQL Connection String Format

```
mysql://[user]:[password]@[host]:[port]/[database]?[options]
```

### ตัวอย่าง Connection Strings

```pascal
// แบบพื้นฐาน
FConnection.Protocol := 'mysql';
FConnection.HostName := 'localhost';
FConnection.Port := 3306;
FConnection.Database := 'hospital_db';
FConnection.User := 'admin';
FConnection.Password := 'secure_password';

// แบบใช้ connection string เต็ม
// mysql://admin:secure_password@localhost:3306/hospital_db?charset=utf8mb4

// สำหรับ SSL connection
FConnection.Properties.Values['MYSQL_SSL'] := '1';
FConnection.Properties.Values['MYSQL_SSL_CA'] := '/path/to/ca-cert.pem';
FConnection.Properties.Values['MYSQL_SSL_CERT'] := '/path/to/client-cert.pem';
FConnection.Properties.Values['MYSQL_SSL_KEY'] := '/path/to/client-key.pem';

// Timeout settings
FConnection.Properties.Values['timeout'] := '30';
FConnection.Properties.Values['connectTimeout'] := '10';
```

---

## การ Query ข้อมูล

### การ SELECT ข้อมูล

```pascal
unit QueryExamples;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, ZConnection, ZDataSet;

type
  TQueryHelper = class
  private
    FConnection: TZConnection;
  public
    constructor Create(AConnection: TZConnection);
    
    // ตัวอย่างการ Query
    procedure SelectExample;
    procedure SelectWithCondition;
    procedure SelectWithJoin;
    procedure SelectWithAggregate;
  end;

implementation

constructor TQueryHelper.Create(AConnection: TZConnection);
begin
  FConnection := AConnection;
end;

// ตัวอย่างที่ 1: SELECT ง่ายๆ
procedure TQueryHelper.SelectExample;
var
  Query: TZQuery;
begin
  Query := TZQuery.Create(nil);
  try
    Query.Connection := FConnection;
    Query.SQL.Text := 'SELECT patient_id, first_name, last_name, birth_date ' +
                      'FROM patients ' +
                      'ORDER BY last_name, first_name';
    Query.Open;
    
    while not Query.EOF do
    begin
      WriteLn(Format('ID: %d, ชื่อ: %s %s, วันเกิด: %s',
        [Query.FieldByName('patient_id').AsInteger,
         Query.FieldByName('first_name').AsString,
         Query.FieldByName('last_name').AsString,
         FormatDateTime('dd/mm/yyyy', Query.FieldByName('birth_date').AsDateTime)]));
      Query.Next;
    end;
    
    WriteLn('จำนวนผู้ป่วยทั้งหมด: ' + IntToStr(Query.RecordCount));
    Query.Close;
  finally
    Query.Free;
  end;
end;

// ตัวอย่างที่ 2: SELECT พร้อม WHERE condition
procedure TQueryHelper.SelectWithCondition;
var
  Query: TZQuery;
begin
  Query := TZQuery.Create(nil);
  try
    Query.Connection := FConnection;
    
    // ค้นหาผู้ป่วยที่อายุมากกว่า 60 ปี
    Query.SQL.Text := 
      'SELECT p.patient_id, p.first_name, p.last_name, ' +
      '       TIMESTAMPDIFF(YEAR, p.birth_date, CURDATE()) AS age ' +
      'FROM patients p ' +
      'WHERE TIMESTAMPDIFF(YEAR, p.birth_date, CURDATE()) > 60 ' +
      'ORDER BY age DESC';
    
    Query.Open;
    
    WriteLn('ผู้ป่วยอายุมากกว่า 60 ปี:');
    WriteLn(StringOfChar('-', 50));
    
    while not Query.EOF do
    begin
      WriteLn(Format('  %s %s - อายุ %d ปี',
        [Query.FieldByName('first_name').AsString,
         Query.FieldByName('last_name').AsString,
         Query.FieldByName('age').AsInteger]));
      Query.Next;
    end;
    
    Query.Close;
  finally
    Query.Free;
  end;
end;

// ตัวอย่างที่ 3: SELECT พร้อม JOIN
procedure TQueryHelper.SelectWithJoin;
var
  Query: TZQuery;
begin
  Query := TZQuery.Create(nil);
  try
    Query.Connection := FConnection;
    
    // JOIN ระหว่างตาราง appointments, patients, doctors
    Query.SQL.Text := 
      'SELECT a.appointment_id, ' +
      '       CONCAT(p.first_name, " ", p.last_name) AS patient_name, ' +
      '       CONCAT("นพ./พญ.", d.first_name, " ", d.last_name) AS doctor_name, ' +
      '       a.appointment_date, a.appointment_time, ' +
      '       a.reason, a.status ' +
      'FROM appointments a ' +
      'INNER JOIN patients p ON a.patient_id = p.patient_id ' +
      'INNER JOIN doctors d ON a.doctor_id = d.doctor_id ' +
      'WHERE a.appointment_date = CURDATE() ' +
      'ORDER BY a.appointment_time';
    
    Query.Open;
    
    WriteLn('นัดหมายวันนี้:');
    WriteLn(StringOfChar('=', 80));
    
    while not Query.EOF do
    begin
      WriteLn(Format('เวลา %s - %s พบ %s (%s)',
        [FormatDateTime('hh:nn', Query.FieldByName('appointment_time').AsDateTime),
         Query.FieldByName('patient_name').AsString,
         Query.FieldByName('doctor_name').AsString,
         Query.FieldByName('reason').AsString]));
      Query.Next;
    end;
    
    Query.Close;
  finally
    Query.Free;
  end;
end;

// ตัวอย่างที่ 4: SELECT พร้อม Aggregate Functions
procedure TQueryHelper.SelectWithAggregate;
var
  Query: TZQuery;
begin
  Query := TZQuery.Create(nil);
  try
    Query.Connection := FConnection;
    
    // สถิติการนัดหมายแต่ละแพทย์
    Query.SQL.Text := 
      'SELECT CONCAT(d.first_name, " ", d.last_name) AS doctor_name, ' +
      '       d.specialty, ' +
      '       COUNT(a.appointment_id) AS total_appointments, ' +
      '       SUM(CASE WHEN a.status = "completed" THEN 1 ELSE 0 END) AS completed, ' +
      '       SUM(CASE WHEN a.status = "cancelled" THEN 1 ELSE 0 END) AS cancelled, ' +
      '       AVG(TIMESTAMPDIFF(MINUTE, a.appointment_time, a.end_time)) AS avg_duration_min ' +
      'FROM doctors d ' +
      'LEFT JOIN appointments a ON d.doctor_id = a.doctor_id ' +
      'GROUP BY d.doctor_id, d.first_name, d.last_name, d.specialty ' +
      'HAVING COUNT(a.appointment_id) > 0 ' +
      'ORDER BY total_appointments DESC';
    
    Query.Open;
    
    WriteLn('สถิติแพทย์:');
    WriteLn(Format('%-30s %-20s %8s %8s %8s %10s',
      ['ชื่อแพทย์', 'ความเชี่ยวชาญ', 'นัดทั้งหมด', 'สำเร็จ', 'ยกเลิก', 'เวลาเฉลี่ย']));
    WriteLn(StringOfChar('-', 90));
    
    while not Query.EOF do
    begin
      WriteLn(Format('%-30s %-20s %8d %8d %8d %10.1f',
        [Query.FieldByName('doctor_name').AsString,
         Query.FieldByName('specialty').AsString,
         Query.FieldByName('total_appointments').AsInteger,
         Query.FieldByName('completed').AsInteger,
         Query.FieldByName('cancelled').AsInteger,
         Query.FieldByName('avg_duration_min').AsFloat]));
      Query.Next;
    end;
    
    Query.Close;
  finally
    Query.Free;
  end;
end;

end.
```

---

## Prepared Statements (Parameterized Queries)

Prepared statements ป้องกัน SQL Injection และเพิ่มประสิทธิภาพ

```pascal
unit PreparedStatements;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, ZConnection, ZDataSet;

type
  TPatientDAO = class
  private
    FConnection: TZConnection;
    FInsertQuery: TZQuery;
    FUpdateQuery: TZQuery;
    FDeleteQuery: TZQuery;
    FSelectQuery: TZQuery;
    
    procedure PrepareStatements;
  public
    constructor Create(AConnection: TZConnection);
    destructor Destroy; override;
    
    function InsertPatient(const FirstName, LastName, IDCard: string;
                          BirthDate: TDateTime; Gender: Char;
                          const Phone, Address: string): Integer;
    function UpdatePatient(PatientID: Integer; const Phone, Address: string): Boolean;
    function DeletePatient(PatientID: Integer): Boolean;
    function FindPatientByID(PatientID: Integer): Boolean;
    function FindPatientByName(const LastName: string): TZQuery;
  end;

implementation

constructor TPatientDAO.Create(AConnection: TZConnection);
begin
  FConnection := AConnection;
  
  FInsertQuery := TZQuery.Create(nil);
  FUpdateQuery := TZQuery.Create(nil);
  FDeleteQuery := TZQuery.Create(nil);
  FSelectQuery := TZQuery.Create(nil);
  
  FInsertQuery.Connection := FConnection;
  FUpdateQuery.Connection := FConnection;
  FDeleteQuery.Connection := FConnection;
  FSelectQuery.Connection := FConnection;
  
  PrepareStatements;
end;

destructor TPatientDAO.Destroy;
begin
  FInsertQuery.Free;
  FUpdateQuery.Free;
  FDeleteQuery.Free;
  FSelectQuery.Free;
  inherited Destroy;
end;

procedure TPatientDAO.PrepareStatements;
begin
  // Prepare INSERT
  FInsertQuery.SQL.Text := 
    'INSERT INTO patients (first_name, last_name, id_card, birth_date, gender, phone, address) ' +
    'VALUES (:first_name, :last_name, :id_card, :birth_date, :gender, :phone, :address)';
  FInsertQuery.Prepare;
  
  // Prepare UPDATE
  FUpdateQuery.SQL.Text := 
    'UPDATE patients SET phone = :phone, address = :address ' +
    'WHERE patient_id = :patient_id';
  FUpdateQuery.Prepare;
  
  // Prepare DELETE
  FDeleteQuery.SQL.Text := 
    'DELETE FROM patients WHERE patient_id = :patient_id';
  FDeleteQuery.Prepare;
  
  // Prepare SELECT
  FSelectQuery.SQL.Text := 
    'SELECT * FROM patients WHERE patient_id = :patient_id';
  FSelectQuery.Prepare;
end;

function TPatientDAO.InsertPatient(const FirstName, LastName, IDCard: string;
                                   BirthDate: TDateTime; Gender: Char;
                                   const Phone, Address: string): Integer;
begin
  Result := -1;
  try
    FInsertQuery.ParamByName('first_name').AsString := FirstName;
    FInsertQuery.ParamByName('last_name').AsString := LastName;
    FInsertQuery.ParamByName('id_card').AsString := IDCard;
    FInsertQuery.ParamByName('birth_date').AsDate := BirthDate;
    FInsertQuery.ParamByName('gender').AsString := Gender;
    FInsertQuery.ParamByName('phone').AsString := Phone;
    FInsertQuery.ParamByName('address').AsString := Address;
    
    FInsertQuery.ExecSQL;
    
    // ดึง ID ที่เพิ่งสร้าง
    with TZQuery.Create(nil) do
    try
      Connection := FConnection;
      SQL.Text := 'SELECT LAST_INSERT_ID()';
      Open;
      Result := Fields[0].AsInteger;
      Close;
    finally
      Free;
    end;
    
  except
    on E: Exception do
    begin
      WriteLn('Error inserting patient: ' + E.Message);
      Result := -1;
    end;
  end;
end;

function TPatientDAO.UpdatePatient(PatientID: Integer; 
                                   const Phone, Address: string): Boolean;
begin
  Result := False;
  try
    FUpdateQuery.ParamByName('patient_id').AsInteger := PatientID;
    FUpdateQuery.ParamByName('phone').AsString := Phone;
    FUpdateQuery.ParamByName('address').AsString := Address;
    FUpdateQuery.ExecSQL;
    
    Result := FUpdateQuery.RowsAffected > 0;
  except
    on E: Exception do
    begin
      WriteLn('Error updating patient: ' + E.Message);
      Result := False;
    end;
  end;
end;

function TPatientDAO.DeletePatient(PatientID: Integer): Boolean;
begin
  Result := False;
  try
    FDeleteQuery.ParamByName('patient_id').AsInteger := PatientID;
    FDeleteQuery.ExecSQL;
    Result := FDeleteQuery.RowsAffected > 0;
  except
    on E: Exception do
    begin
      WriteLn('Error deleting patient: ' + E.Message);
      Result := False;
    end;
  end;
end;

function TPatientDAO.FindPatientByID(PatientID: Integer): Boolean;
begin
  if FSelectQuery.Active then
    FSelectQuery.Close;
    
  FSelectQuery.ParamByName('patient_id').AsInteger := PatientID;
  FSelectQuery.Open;
  
  Result := not FSelectQuery.EOF;
end;

function TPatientDAO.FindPatientByName(const LastName: string): TZQuery;
var
  DynamicQuery: TZQuery;
begin
  DynamicQuery := TZQuery.Create(nil);
  DynamicQuery.Connection := FConnection;
  
  // ใช้ LIKE สำหรับการค้นหา
  DynamicQuery.SQL.Text := 
    'SELECT * FROM patients ' +
    'WHERE last_name LIKE :search_term ' +
    'ORDER BY last_name, first_name';
  
  DynamicQuery.ParamByName('search_term').AsString := '%' + LastName + '%';
  DynamicQuery.Open;
  
  Result := DynamicQuery;
end;

end.
```

---

## Stored Procedures

### การสร้าง Stored Procedure ใน MySQL

```sql
-- สร้าง Procedure สำหรับสร้างนัดหมาย
DELIMITER //

CREATE PROCEDURE create_appointment(
    IN p_patient_id INT,
    IN p_doctor_id INT,
    IN p_appointment_date DATE,
    IN p_appointment_time TIME,
    IN p_reason VARCHAR(500),
    OUT p_appointment_id INT,
    OUT p_status_message VARCHAR(200)
)
BEGIN
    DECLARE v_doctor_available INT;
    DECLARE v_slot_count INT;
    
    -- ตรวจสอบว่าแพทย์ว่างหรือไม่
    SELECT COUNT(*) INTO v_doctor_available
    FROM doctor_schedule ds
    WHERE ds.doctor_id = p_doctor_id
      AND ds.schedule_date = p_appointment_date
      AND ds.start_time <= p_appointment_time
      AND ds.end_time > p_appointment_time
      AND ds.is_available = 1;
    
    IF v_doctor_available = 0 THEN
        SET p_appointment_id = -1;
        SET p_status_message = 'แพทย์ไม่ว่างในช่วงเวลาที่ระบุ';
        LEAVE create_appointment;
    END IF;
    
    -- ตรวจสอบจำนวน slot ที่เหลือ
    SELECT (ds.max_patients - COUNT(a.appointment_id)) INTO v_slot_count
    FROM doctor_schedule ds
    LEFT JOIN appointments a ON a.doctor_id = ds.doctor_id
        AND a.appointment_date = ds.schedule_date
        AND a.appointment_time = p_appointment_time
        AND a.status != 'cancelled'
    WHERE ds.doctor_id = p_doctor_id
      AND ds.schedule_date = p_appointment_date
      AND ds.start_time <= p_appointment_time
    GROUP BY ds.max_patients;
    
    IF v_slot_count <= 0 THEN
        SET p_appointment_id = -2;
        SET p_status_message = 'ช่วงเวลานี้เต็มแล้ว';
        LEAVE create_appointment;
    END IF;
    
    -- สร้างนัดหมาย
    INSERT INTO appointments 
        (patient_id, doctor_id, appointment_date, appointment_time, reason, status)
    VALUES 
        (p_patient_id, p_doctor_id, p_appointment_date, p_appointment_time, p_reason, 'pending');
    
    SET p_appointment_id = LAST_INSERT_ID();
    SET p_status_message = 'สร้างนัดหมายสำเร็จ';
END //

DELIMITER ;
```

### การเรียก Stored Procedure จาก Lazarus

```pascal
unit StoredProcExample;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, ZConnection, ZStoredProcedure;

type
  TAppointmentService = class
  private
    FConnection: TZConnection;
  public
    constructor Create(AConnection: TZConnection);
    
    function CreateAppointment(PatientID, DoctorID: Integer;
                               AppDate: TDateTime; AppTime: TDateTime;
                               const Reason: string;
                               out AppointmentID: Integer;
                               out StatusMessage: string): Boolean;
  end;

implementation

constructor TAppointmentService.Create(AConnection: TZConnection);
begin
  FConnection := AConnection;
end;

function TAppointmentService.CreateAppointment(PatientID, DoctorID: Integer;
                                               AppDate: TDateTime; AppTime: TDateTime;
                                               const Reason: string;
                                               out AppointmentID: Integer;
                                               out StatusMessage: string): Boolean;
var
  SP: TZStoredProcedure;
begin
  Result := False;
  AppointmentID := -1;
  StatusMessage := '';
  
  SP := TZStoredProcedure.Create(nil);
  try
    SP.Connection := FConnection;
    SP.StoredProcName := 'create_appointment';
    
    // กำหนดค่า parameters
    SP.Prepare;
    
    SP.ParamByName('p_patient_id').AsInteger := PatientID;
    SP.ParamByName('p_doctor_id').AsInteger := DoctorID;
    SP.ParamByName('p_appointment_date').AsDate := AppDate;
    SP.ParamByName('p_appointment_time').AsTime := AppTime;
    SP.ParamByName('p_reason').AsString := Reason;
    
    // เรียก procedure
    SP.ExecProc;
    
    // รับค่า output parameters
    AppointmentID := SP.ParamByName('p_appointment_id').AsInteger;
    StatusMessage := SP.ParamByName('p_status_message').AsString;
    
    Result := AppointmentID > 0;
  except
    on E: Exception do
    begin
      StatusMessage := 'เกิดข้อผิดพลาด: ' + E.Message;
      Result := False;
    end;
  end;
  SP.Free;
end;

end.
```

---

## Transaction Management

```pascal
unit TransactionExample;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, ZConnection, ZDataSet;

type
  TPatientService = class
  private
    FConnection: TZConnection;
  public
    constructor Create(AConnection: TZConnection);
    
    // โอนย้ายผู้ป่วยระหว่างแผนก (ต้องทำเป็น transaction)
    function TransferPatient(PatientID, FromDepartment, ToDepartment: Integer;
                             const Reason: string): Boolean;
    
    // บันทึกผลการตรวจ (ต้องทำเป็น transaction)
    function SaveExaminationResult(AppointmentID: Integer;
                                   const Diagnosis, Treatment: string;
                                   Prescriptions: TStringList): Boolean;
  end;

implementation

constructor TPatientService.Create(AConnection: TZConnection);
begin
  FConnection := AConnection;
end;

function TPatientService.TransferPatient(PatientID, FromDepartment, 
                                         ToDepartment: Integer;
                                         const Reason: string): Boolean;
var
  Query: TZQuery;
begin
  Result := False;
  Query := TZQuery.Create(nil);
  
  try
    Query.Connection := FConnection;
    
    // เริ่ม Transaction
    FConnection.StartTransaction;
    
    try
      // อัปเดตแผนกของผู้ป่วย
      Query.SQL.Text := 
        'UPDATE patients SET department_id = :new_dept ' +
        'WHERE patient_id = :patient_id AND department_id = :old_dept';
      Query.ParamByName('new_dept').AsInteger := ToDepartment;
      Query.ParamByName('patient_id').AsInteger := PatientID;
      Query.ParamByName('old_dept').AsInteger := FromDepartment;
      Query.ExecSQL;
      
      if Query.RowsAffected = 0 then
        raise Exception.Create('ไม่พบผู้ป่วยหรือผู้ป่วยไม่ได้อยู่ในแผนกต้นทาง');
      
      // บันทึก log การโอนย้าย
      Query.SQL.Text := 
        'INSERT INTO transfer_log (patient_id, from_dept, to_dept, reason, transfer_date) ' +
        'VALUES (:patient_id, :from_dept, :to_dept, :reason, NOW())';
      Query.ParamByName('patient_id').AsInteger := PatientID;
      Query.ParamByName('from_dept').AsInteger := FromDepartment;
      Query.ParamByName('to_dept').AsInteger := ToDepartment;
      Query.ParamByName('reason').AsString := Reason;
      Query.ExecSQL;
      
      // Commit ถ้าทุกอย่างสำเร็จ
      FConnection.Commit;
      Result := True;
      WriteLn('โอนย้ายผู้ป่วยสำเร็จ');
      
    except
      on E: Exception do
      begin
        // Rollback เมื่อเกิดข้อผิดพลาด
        FConnection.Rollback;
        WriteLn('โอนย้ายผู้ป่วยล้มเหลว: ' + E.Message);
        WriteLn('ยกเลิกการเปลี่ยนแปลงทั้งหมด');
      end;
    end;
    
  finally
    Query.Free;
  end;
end;

function TPatientService.SaveExaminationResult(AppointmentID: Integer;
                                               const Diagnosis, Treatment: string;
                                               Prescriptions: TStringList): Boolean;
var
  Query: TZQuery;
  i: Integer;
  ExamID: Integer;
begin
  Result := False;
  Query := TZQuery.Create(nil);
  
  try
    Query.Connection := FConnection;
    
    // เริ่ม Transaction
    FConnection.StartTransaction;
    
    try
      // บันทึกผลการตรวจ
      Query.SQL.Text := 
        'INSERT INTO examinations (appointment_id, diagnosis, treatment, exam_date) ' +
        'VALUES (:appointment_id, :diagnosis, :treatment, NOW())';
      Query.ParamByName('appointment_id').AsInteger := AppointmentID;
      Query.ParamByName('diagnosis').AsString := Diagnosis;
      Query.ParamByName('treatment').AsString := Treatment;
      Query.ExecSQL;
      
      // ดึง ID ที่เพิ่งสร้าง
      Query.SQL.Text := 'SELECT LAST_INSERT_ID()';
      Query.Open;
      ExamID := Query.Fields[0].AsInteger;
      Query.Close;
      
      // บันทึกใบสั่งยา
      for i := 0 to Prescriptions.Count - 1 do
      begin
        Query.SQL.Text := 
          'INSERT INTO prescriptions (examination_id, medicine_code, dosage) ' +
          'VALUES (:exam_id, :medicine_code, :dosage)';
        
        // สมมติว่า Prescriptions เก็บในรูปแบบ "medicine_code|dosage"
        var Parts := Prescriptions[i].Split(['|']);
        if Length(Parts) >= 2 then
        begin
          Query.ParamByName('exam_id').AsInteger := ExamID;
          Query.ParamByName('medicine_code').AsString := Parts[0];
          Query.ParamByName('dosage').AsString := Parts[1];
          Query.ExecSQL;
        end;
      end;
      
      // อัปเดตสถานะนัดหมาย
      Query.SQL.Text := 
        'UPDATE appointments SET status = "completed" ' +
        'WHERE appointment_id = :appointment_id';
      Query.ParamByName('appointment_id').AsInteger := AppointmentID;
      Query.ExecSQL;
      
      // Commit
      FConnection.Commit;
      Result := True;
      
    except
      on E: Exception do
      begin
        FConnection.Rollback;
        WriteLn('บันทึกผลการตรวจล้มเหลว: ' + E.Message);
      end;
    end;
    
  finally
    Query.Free;
  end;
end;

end.
```

---

## Connection Pooling

```pascal
unit ConnectionPool;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, ZConnection;

const
  MAX_POOL_SIZE = 10;
  MIN_POOL_SIZE = 2;

type
  TPooledConnection = class
  public
    Connection: TZConnection;
    InUse: Boolean;
    LastUsed: TDateTime;
    
    constructor Create;
    destructor Destroy; override;
  end;

  TConnectionPool = class
  private
    FPool: TList;
    FLock: TCriticalSection;
    FHost, FDatabase, FUser, FPassword: string;
    FPort: Integer;
    
    procedure CreateConnection(var Conn: TPooledConnection);
    procedure CleanupIdleConnections;
  public
    constructor Create(const Host, Database, User, Password: string;
                      Port: Integer = 3306);
    destructor Destroy; override;
    
    function AcquireConnection: TZConnection;
    procedure ReleaseConnection(AConnection: TZConnection);
    function GetPoolStats: string;
  end;

implementation

constructor TPooledConnection.Create;
begin
  Connection := TZConnection.Create(nil);
  InUse := False;
  LastUsed := Now;
end;

destructor TPooledConnection.Destroy;
begin
  if Connection.Connected then
    Connection.Disconnect;
  Connection.Free;
  inherited Destroy;
end;

constructor TConnectionPool.Create(const Host, Database, User, Password: string;
                                   Port: Integer = 3306);
var
  i: Integer;
  PC: TPooledConnection;
begin
  FPool := TList.Create;
  FLock := TCriticalSection.Create;
  
  FHost := Host;
  FDatabase := Database;
  FUser := User;
  FPassword := Password;
  FPort := Port;
  
  // สร้าง connection เริ่มต้น
  for i := 1 to MIN_POOL_SIZE do
  begin
    PC := TPooledConnection.Create;
    CreateConnection(PC);
    FPool.Add(PC);
  end;
end;

destructor TConnectionPool.Destroy;
var
  i: Integer;
begin
  FLock.Acquire;
  try
    for i := 0 to FPool.Count - 1 do
      TPooledConnection(FPool[i]).Free;
    FPool.Free;
  finally
    FLock.Release;
  end;
  FLock.Free;
  inherited Destroy;
end;

procedure TConnectionPool.CreateConnection(var Conn: TPooledConnection);
begin
  Conn.Connection.Protocol := 'mysql';
  Conn.Connection.HostName := FHost;
  Conn.Connection.Port := FPort;
  Conn.Connection.Database := FDatabase;
  Conn.Connection.User := FUser;
  Conn.Connection.Password := FPassword;
  Conn.Connection.Properties.Values['codepage'] := 'UTF8';
  
  try
    Conn.Connection.Connect;
  except
    on E: Exception do
      WriteLn('Cannot create pool connection: ' + E.Message);
  end;
end;

function TConnectionPool.AcquireConnection: TZConnection;
var
  i: Integer;
  PC: TPooledConnection;
begin
  Result := nil;
  
  FLock.Acquire;
  try
    // ค้นหา connection ที่ว่าง
    for i := 0 to FPool.Count - 1 do
    begin
      PC := TPooledConnection(FPool[i]);
      if not PC.InUse then
      begin
        // ตรวจสอบว่า connection ยังใช้งานได้
        if not PC.Connection.Connected then
          PC.Connection.Connect;
          
        PC.InUse := True;
        PC.LastUsed := Now;
        Result := PC.Connection;
        Exit;
      end;
    end;
    
    // ถ้าไม่มี connection ว่าง และยังไม่เต็ม ให้สร้างใหม่
    if FPool.Count < MAX_POOL_SIZE then
    begin
      PC := TPooledConnection.Create;
      CreateConnection(PC);
      PC.InUse := True;
      PC.LastUsed := Now;
      FPool.Add(PC);
      Result := PC.Connection;
    end
    else
      raise Exception.Create('Connection pool เต็ม ไม่สามารถรับ connection ได้');
      
  finally
    FLock.Release;
  end;
end;

procedure TConnectionPool.ReleaseConnection(AConnection: TZConnection);
var
  i: Integer;
  PC: TPooledConnection;
begin
  FLock.Acquire;
  try
    for i := 0 to FPool.Count - 1 do
    begin
      PC := TPooledConnection(FPool[i]);
      if PC.Connection = AConnection then
      begin
        PC.InUse := False;
        PC.LastUsed := Now;
        Break;
      end;
    end;
  finally
    FLock.Release;
  end;
end;

function TConnectionPool.GetPoolStats: string;
var
  i, InUseCount, FreeCount: Integer;
begin
  InUseCount := 0;
  FreeCount := 0;
  
  FLock.Acquire;
  try
    for i := 0 to FPool.Count - 1 do
    begin
      if TPooledConnection(FPool[i]).InUse then
        Inc(InUseCount)
      else
        Inc(FreeCount);
    end;
  finally
    FLock.Release;
  end;
  
  Result := Format('Pool: %d connections (%d in use, %d free)',
    [FPool.Count, InUseCount, FreeCount]);
end;

procedure TConnectionPool.CleanupIdleConnections;
var
  i: Integer;
  PC: TPooledConnection;
  IdleTime: Double;
begin
  FLock.Acquire;
  try
    i := FPool.Count - 1;
    while (i >= MIN_POOL_SIZE) and (FPool.Count > MIN_POOL_SIZE) do
    begin
      PC := TPooledConnection(FPool[i]);
      if not PC.InUse then
      begin
        IdleTime := (Now - PC.LastUsed) * 24 * 60;  // นาที
        if IdleTime > 30 then  // idle มากกว่า 30 นาที
        begin
          FPool.Delete(i);
          PC.Free;
        end;
      end;
      Dec(i);
    end;
  finally
    FLock.Release;
  end;
end;

end.
```

---

## Error Handling

```pascal
unit MySQLErrorHandling;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, ZConnection, ZDataSet, ZDbcIntfs;

type
  // Custom exception สำหรับ MySQL errors
  EMySQL_Error = class(Exception)
  private
    FErrorCode: Integer;
    FSQLState: string;
  public
    constructor Create(const Msg: string; ErrorCode: Integer; const SQLState: string);
    property ErrorCode: Integer read FErrorCode;
    property SQLState: string read FSQLState;
  end;

  TDatabaseErrorHandler = class
  public
    class procedure HandleError(const Operation: string; E: Exception);
    class function GetFriendlyMessage(E: Exception): string;
    class function IsDuplicateKeyError(E: Exception): Boolean;
    class function IsForeignKeyError(E: Exception): Boolean;
    class function IsConnectionError(E: Exception): Boolean;
  end;

  TRobustQueryHelper = class
  private
    FConnection: TZConnection;
    FMaxRetries: Integer;
  public
    constructor Create(AConnection: TZConnection; MaxRetries: Integer = 3);
    
    function ExecuteWithRetry(const SQL: string; Params: array of Variant): Boolean;
    function QueryWithRetry(const SQL: string): TZQuery;
  end;

implementation

// MySQL Error Codes
const
  ER_DUP_ENTRY = 1062;      // Duplicate entry
  ER_NO_REFERENCED_ROW = 1216;  // Foreign key violation
  ER_ROW_IS_REFERENCED = 1217;  // Foreign key referenced
  ER_LOCK_DEADLOCK = 1213;  // Deadlock
  ER_LOCK_WAIT_TIMEOUT = 1205;  // Lock wait timeout

constructor EMySQL_Error.Create(const Msg: string; ErrorCode: Integer; 
                                const SQLState: string);
begin
  inherited Create(Msg);
  FErrorCode := ErrorCode;
  FSQLState := SQLState;
end;

class procedure TDatabaseErrorHandler.HandleError(const Operation: string; E: Exception);
begin
  if IsDuplicateKeyError(E) then
    WriteLn(Format('[%s] ข้อมูลซ้ำ: %s', [Operation, E.Message]))
  else if IsForeignKeyError(E) then
    WriteLn(Format('[%s] ข้อผิดพลาด Foreign Key: %s', [Operation, E.Message]))
  else if IsConnectionError(E) then
    WriteLn(Format('[%s] ไม่สามารถเชื่อมต่อฐานข้อมูล: %s', [Operation, E.Message]))
  else
    WriteLn(Format('[%s] ข้อผิดพลาดทั่วไป: %s', [Operation, E.Message]));
end;

class function TDatabaseErrorHandler.GetFriendlyMessage(E: Exception): string;
begin
  if IsDuplicateKeyError(E) then
    Result := 'ข้อมูลนี้มีอยู่ในระบบแล้ว กรุณาตรวจสอบและลองใหม่'
  else if IsForeignKeyError(E) then
    Result := 'ไม่สามารถดำเนินการได้เนื่องจากข้อมูลอ้างอิงกัน'
  else if IsConnectionError(E) then
    Result := 'ไม่สามารถเชื่อมต่อฐานข้อมูลได้ กรุณาตรวจสอบการเชื่อมต่อ'
  else
    Result := 'เกิดข้อผิดพลาด: ' + E.Message;
end;

class function TDatabaseErrorHandler.IsDuplicateKeyError(E: Exception): Boolean;
begin
  Result := (E is EMySQL_Error) and (EMySQL_Error(E).ErrorCode = ER_DUP_ENTRY);
  
  // หรือตรวจสอบจาก message
  if not Result then
    Result := Pos('Duplicate entry', E.Message) > 0;
end;

class function TDatabaseErrorHandler.IsForeignKeyError(E: Exception): Boolean;
begin
  Result := (E is EMySQL_Error) and 
            ((EMySQL_Error(E).ErrorCode = ER_NO_REFERENCED_ROW) or
             (EMySQL_Error(E).ErrorCode = ER_ROW_IS_REFERENCED));
             
  if not Result then
    Result := Pos('foreign key constraint', LowerCase(E.Message)) > 0;
end;

class function TDatabaseErrorHandler.IsConnectionError(E: Exception): Boolean;
begin
  Result := Pos('connection', LowerCase(E.Message)) > 0;
end;

constructor TRobustQueryHelper.Create(AConnection: TZConnection; MaxRetries: Integer = 3);
begin
  FConnection := AConnection;
  FMaxRetries := MaxRetries;
end;

function TRobustQueryHelper.ExecuteWithRetry(const SQL: string; 
                                             Params: array of Variant): Boolean;
var
  RetryCount: Integer;
  Query: TZQuery;
  Success: Boolean;
begin
  Result := False;
  RetryCount := 0;
  
  while RetryCount < FMaxRetries do
  begin
    Query := TZQuery.Create(nil);
    Success := False;
    
    try
      Query.Connection := FConnection;
      Query.SQL.Text := SQL;
      
      // กำหนดค่า parameters
      var i: Integer;
      for i := 0 to High(Params) do
        Query.Params[i].Value := Params[i];
      
      Query.ExecSQL;
      Success := True;
      
    except
      on E: Exception do
      begin
        Inc(RetryCount);
        
        // ถ้าเป็น deadlock ให้ลองใหม่
        if (Pos('deadlock', LowerCase(E.Message)) > 0) and 
           (RetryCount < FMaxRetries) then
        begin
          WriteLn(Format('Deadlock พบในครั้งที่ %d ลองใหม่...', [RetryCount]));
          Sleep(100 * RetryCount);  // รอเพิ่มขึ้นทุกครั้ง
        end
        else
        begin
          TDatabaseErrorHandler.HandleError('ExecuteWithRetry', E);
          Query.Free;
          Exit;
        end;
      end;
    end;
    
    Query.Free;
    
    if Success then
    begin
      Result := True;
      Exit;
    end;
  end;
end;

function TRobustQueryHelper.QueryWithRetry(const SQL: string): TZQuery;
var
  RetryCount: Integer;
  Query: TZQuery;
begin
  Result := nil;
  RetryCount := 0;
  
  while RetryCount < FMaxRetries do
  begin
    Query := TZQuery.Create(nil);
    try
      Query.Connection := FConnection;
      Query.SQL.Text := SQL;
      Query.Open;
      Result := Query;
      Exit;
    except
      on E: Exception do
      begin
        Query.Free;
        Inc(RetryCount);
        
        if IsConnectionLost(E) and (RetryCount < FMaxRetries) then
        begin
          WriteLn('Connection lost, reconnecting...');
          try
            FConnection.Disconnect;
            FConnection.Connect;
          except
            // ignore reconnect error
          end;
          Sleep(500);
        end
        else
          raise;
      end;
    end;
  end;
end;

function TRobustQueryHelper.IsConnectionLost(E: Exception): Boolean;
begin
  Result := (Pos('MySQL server has gone away', E.Message) > 0) or
            (Pos('Lost connection', E.Message) > 0) or
            (Pos('Connection reset', E.Message) > 0);
end;

end.
```

---

## ตัวอย่างระบบโรงพยาบาล - CRUD สมบูรณ์

### โครงสร้างฐานข้อมูล

```sql
-- สร้างฐานข้อมูล
CREATE DATABASE hospital_db
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE hospital_db;

-- ตารางแผนก
CREATE TABLE departments (
    department_id INT PRIMARY KEY AUTO_INCREMENT,
    department_name VARCHAR(100) NOT NULL,
    department_code VARCHAR(20) UNIQUE NOT NULL,
    location VARCHAR(200),
    phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- ตารางแพทย์
CREATE TABLE doctors (
    doctor_id INT PRIMARY KEY AUTO_INCREMENT,
    employee_id VARCHAR(20) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    specialty VARCHAR(100),
    department_id INT,
    phone VARCHAR(20),
    email VARCHAR(100),
    license_number VARCHAR(50) UNIQUE,
    hire_date DATE,
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

-- ตารางผู้ป่วย
CREATE TABLE patients (
    patient_id INT PRIMARY KEY AUTO_INCREMENT,
    patient_code VARCHAR(20) UNIQUE NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    id_card VARCHAR(13) UNIQUE,
    birth_date DATE NOT NULL,
    gender ENUM('M', 'F', 'Other') NOT NULL,
    blood_type ENUM('A', 'B', 'AB', 'O') NOT NULL,
    phone VARCHAR(20),
    email VARCHAR(100),
    address TEXT,
    allergies TEXT,
    emergency_contact VARCHAR(100),
    emergency_phone VARCHAR(20),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
);

-- ตารางนัดหมาย
CREATE TABLE appointments (
    appointment_id INT PRIMARY KEY AUTO_INCREMENT,
    patient_id INT NOT NULL,
    doctor_id INT NOT NULL,
    department_id INT NOT NULL,
    appointment_date DATE NOT NULL,
    appointment_time TIME NOT NULL,
    end_time TIME,
    reason VARCHAR(500),
    status ENUM('pending', 'confirmed', 'in_progress', 'completed', 'cancelled', 'no_show') DEFAULT 'pending',
    notes TEXT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (patient_id) REFERENCES patients(patient_id),
    FOREIGN KEY (doctor_id) REFERENCES doctors(doctor_id),
    FOREIGN KEY (department_id) REFERENCES departments(department_id)
);

-- ตารางการตรวจ
CREATE TABLE examinations (
    examination_id INT PRIMARY KEY AUTO_INCREMENT,
    appointment_id INT NOT NULL,
    diagnosis TEXT,
    treatment TEXT,
    vital_signs JSON,
    exam_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    follow_up_date DATE,
    FOREIGN KEY (appointment_id) REFERENCES appointments(appointment_id)
);

-- ตารางใบสั่งยา
CREATE TABLE prescriptions (
    prescription_id INT PRIMARY KEY AUTO_INCREMENT,
    examination_id INT NOT NULL,
    medicine_code VARCHAR(20) NOT NULL,
    medicine_name VARCHAR(200),
    dosage VARCHAR(100),
    frequency VARCHAR(100),
    duration_days INT,
    instructions TEXT,
    FOREIGN KEY (examination_id) REFERENCES examinations(examination_id)
);

-- Index สำหรับประสิทธิภาพ
CREATE INDEX idx_patients_name ON patients(last_name, first_name);
CREATE INDEX idx_patients_idcard ON patients(id_card);
CREATE INDEX idx_appointments_date ON appointments(appointment_date, doctor_id);
CREATE INDEX idx_appointments_patient ON appointments(patient_id, appointment_date);
```

### แอปพลิเคชัน GUI สำหรับจัดการผู้ป่วย

```pascal
unit HospitalMainForm;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Forms, Controls, Graphics, Dialogs,
  StdCtrls, ExtCtrls, ComCtrls, DBGrids, DBCtrls, DB,
  ZConnection, ZDataSet, ZDbcIntfs,
  DateUtils;

type
  TfrmHospital = class(TForm)
    // Panels
    pnlTop: TPanel;
    pnlLeft: TPanel;
    pnlMain: TPanel;
    pnlBottom: TPanel;
    
    // Navigation
    btnPatients: TButton;
    btnDoctors: TButton;
    btnAppointments: TButton;
    btnReports: TButton;
    
    // Status
    lblStatus: TLabel;
    lblRecordCount: TLabel;
    
    // Patient Tab
    pnlPatientSearch: TPanel;
    edtSearch: TEdit;
    btnSearch: TButton;
    btnClearSearch: TButton;
    
    // DBGrid
    dbgPatients: TDBGrid;
    
    // Patient Form
    pnlPatientForm: TPanel;
    edtFirstName: TEdit;
    edtLastName: TEdit;
    edtIDCard: TEdit;
    dtpBirthDate: TDateTimePicker;
    cmbGender: TComboBox;
    cmbBloodType: TComboBox;
    edtPhone: TEdit;
    edtEmail: TEdit;
    memAddress: TMemo;
    memAllergies: TMemo;
    edtEmergencyContact: TEdit;
    edtEmergencyPhone: TEdit;
    
    // Buttons
    btnNew: TButton;
    btnSave: TButton;
    btnDelete: TButton;
    btnCancel: TButton;
    
    // Data
    dsPatients: TDataSource;
    qPatients: TZQuery;
    ZConn: TZConnection;
    
  private
    FCurrentPatientID: Integer;
    FIsNewRecord: Boolean;
    FIsEditing: Boolean;
    
    procedure InitializeDatabase;
    procedure LoadPatients(const SearchTerm: string = '');
    procedure LoadPatientDetails(PatientID: Integer);
    procedure ClearPatientForm;
    procedure EnableEditing(Enable: Boolean);
    
    procedure btnNewClick(Sender: TObject);
    procedure btnSaveClick(Sender: TObject);
    procedure btnDeleteClick(Sender: TObject);
    procedure btnCancelClick(Sender: TObject);
    procedure btnSearchClick(Sender: TObject);
    procedure btnClearSearchClick(Sender: TObject);
    procedure dbgPatientsSelectionChange(Sender: TObject);
    
    function ValidatePatientForm: Boolean;
    function GeneratePatientCode: string;
    
  public
    constructor Create(AOwner: TComponent); override;
    destructor Destroy; override;
  end;

var
  frmHospital: TfrmHospital;

implementation

{$R *.lfm}

constructor TfrmHospital.Create(AOwner: TComponent);
begin
  inherited Create(AOwner);
  
  Caption := 'ระบบบริหารโรงพยาบาล';
  Width := 1200;
  Height := 700;
  
  FCurrentPatientID := -1;
  FIsNewRecord := False;
  FIsEditing := False;
  
  InitializeDatabase;
  LoadPatients;
  EnableEditing(False);
end;

destructor TfrmHospital.Destroy;
begin
  if ZConn.Connected then
    ZConn.Disconnect;
  inherited Destroy;
end;

procedure TfrmHospital.InitializeDatabase;
begin
  try
    ZConn.Protocol := 'mysql';
    ZConn.HostName := 'localhost';
    ZConn.Port := 3306;
    ZConn.Database := 'hospital_db';
    ZConn.User := 'hospital_user';
    ZConn.Password := 'hospital_pass';
    ZConn.Properties.Values['codepage'] := 'UTF8';
    ZConn.Connect;
    
    // Setup DataSource
    qPatients.Connection := ZConn;
    dsPatients.DataSet := qPatients;
    dbgPatients.DataSource := dsPatients;
    
    lblStatus.Caption := 'เชื่อมต่อฐานข้อมูลสำเร็จ';
    
  except
    on E: Exception do
    begin
      ShowMessage('ไม่สามารถเชื่อมต่อฐานข้อมูล: ' + E.Message);
      lblStatus.Caption := 'ไม่สามารถเชื่อมต่อ';
    end;
  end;
end;

procedure TfrmHospital.LoadPatients(const SearchTerm: string = '');
begin
  if qPatients.Active then
    qPatients.Close;
    
  if SearchTerm <> '' then
    qPatients.SQL.Text := 
      'SELECT patient_id, patient_code, first_name, last_name, ' +
      '       TIMESTAMPDIFF(YEAR, birth_date, CURDATE()) AS age, ' +
      '       gender, blood_type, phone, ' +
      '       DATE_FORMAT(created_at, "%d/%m/%Y") AS registered_date ' +
      'FROM patients ' +
      'WHERE last_name LIKE :search OR first_name LIKE :search ' +
      '   OR patient_code LIKE :search OR id_card LIKE :search ' +
      'ORDER BY last_name, first_name'
  else
    qPatients.SQL.Text := 
      'SELECT patient_id, patient_code, first_name, last_name, ' +
      '       TIMESTAMPDIFF(YEAR, birth_date, CURDATE()) AS age, ' +
      '       gender, blood_type, phone, ' +
      '       DATE_FORMAT(created_at, "%d/%m/%Y") AS registered_date ' +
      'FROM patients ' +
      'ORDER BY last_name, first_name ' +
      'LIMIT 1000';
      
  if SearchTerm <> '' then
    qPatients.ParamByName('search').AsString := '%' + SearchTerm + '%';
    
  qPatients.Open;
  
  lblRecordCount.Caption := Format('จำนวน: %d ราย', [qPatients.RecordCount]);
end;

procedure TfrmHospital.LoadPatientDetails(PatientID: Integer);
var
  Q: TZQuery;
begin
  Q := TZQuery.Create(nil);
  try
    Q.Connection := ZConn;
    Q.SQL.Text := 'SELECT * FROM patients WHERE patient_id = :id';
    Q.ParamByName('id').AsInteger := PatientID;
    Q.Open;
    
    if not Q.EOF then
    begin
      edtFirstName.Text := Q.FieldByName('first_name').AsString;
      edtLastName.Text := Q.FieldByName('last_name').AsString;
      edtIDCard.Text := Q.FieldByName('id_card').AsString;
      dtpBirthDate.Date := Q.FieldByName('birth_date').AsDateTime;
      
      case Q.FieldByName('gender').AsString of
        'M': cmbGender.ItemIndex := 0;
        'F': cmbGender.ItemIndex := 1;
        else cmbGender.ItemIndex := 2;
      end;
      
      cmbBloodType.ItemIndex := cmbBloodType.Items.IndexOf(
        Q.FieldByName('blood_type').AsString);
      
      edtPhone.Text := Q.FieldByName('phone').AsString;
      edtEmail.Text := Q.FieldByName('email').AsString;
      memAddress.Text := Q.FieldByName('address').AsString;
      memAllergies.Text := Q.FieldByName('allergies').AsString;
      edtEmergencyContact.Text := Q.FieldByName('emergency_contact').AsString;
      edtEmergencyPhone.Text := Q.FieldByName('emergency_phone').AsString;
      
      FCurrentPatientID := PatientID;
    end;
    
    Q.Close;
  finally
    Q.Free;
  end;
end;

procedure TfrmHospital.ClearPatientForm;
begin
  edtFirstName.Text := '';
  edtLastName.Text := '';
  edtIDCard.Text := '';
  dtpBirthDate.Date := Now;
  cmbGender.ItemIndex := 0;
  cmbBloodType.ItemIndex := 0;
  edtPhone.Text := '';
  edtEmail.Text := '';
  memAddress.Text := '';
  memAllergies.Text := '';
  edtEmergencyContact.Text := '';
  edtEmergencyPhone.Text := '';
  FCurrentPatientID := -1;
end;

procedure TfrmHospital.EnableEditing(Enable: Boolean);
begin
  edtFirstName.Enabled := Enable;
  edtLastName.Enabled := Enable;
  edtIDCard.Enabled := Enable;
  dtpBirthDate.Enabled := Enable;
  cmbGender.Enabled := Enable;
  cmbBloodType.Enabled := Enable;
  edtPhone.Enabled := Enable;
  edtEmail.Enabled := Enable;
  memAddress.Enabled := Enable;
  memAllergies.Enabled := Enable;
  edtEmergencyContact.Enabled := Enable;
  edtEmergencyPhone.Enabled := Enable;
  
  btnSave.Enabled := Enable;
  btnCancel.Enabled := Enable;
  btnNew.Enabled := not Enable;
  btnDelete.Enabled := not Enable and (FCurrentPatientID > 0);
end;

function TfrmHospital.ValidatePatientForm: Boolean;
begin
  Result := False;
  
  if Trim(edtFirstName.Text) = '' then
  begin
    ShowMessage('กรุณาระบุชื่อ');
    edtFirstName.SetFocus;
    Exit;
  end;
  
  if Trim(edtLastName.Text) = '' then
  begin
    ShowMessage('กรุณาระบุนามสกุล');
    edtLastName.SetFocus;
    Exit;
  end;
  
  if Trim(edtIDCard.Text) <> '' then
  begin
    if Length(Trim(edtIDCard.Text)) <> 13 then
    begin
      ShowMessage('เลขบัตรประชาชนต้องมี 13 หลัก');
      edtIDCard.SetFocus;
      Exit;
    end;
  end;
  
  Result := True;
end;

function TfrmHospital.GeneratePatientCode: string;
var
  Q: TZQuery;
  MaxCode: Integer;
begin
  Q := TZQuery.Create(nil);
  try
    Q.Connection := ZConn;
    Q.SQL.Text := 'SELECT MAX(CAST(SUBSTRING(patient_code, 4) AS UNSIGNED)) FROM patients';
    Q.Open;
    MaxCode := Q.Fields[0].AsInteger + 1;
    Q.Close;
  finally
    Q.Free;
  end;
  
  Result := Format('HN-%06d', [MaxCode]);
end;

procedure TfrmHospital.btnNewClick(Sender: TObject);
begin
  ClearPatientForm;
  FIsNewRecord := True;
  FIsEditing := True;
  EnableEditing(True);
  edtFirstName.SetFocus;
end;

procedure TfrmHospital.btnSaveClick(Sender: TObject);
var
  Q: TZQuery;
begin
  if not ValidatePatientForm then
    Exit;
    
  Q := TZQuery.Create(nil);
  try
    Q.Connection := ZConn;
    
    try
      ZConn.StartTransaction;
      
      if FIsNewRecord then
      begin
        Q.SQL.Text := 
          'INSERT INTO patients (patient_code, first_name, last_name, id_card, ' +
          '  birth_date, gender, blood_type, phone, email, address, allergies, ' +
          '  emergency_contact, emergency_phone) ' +
          'VALUES (:patient_code, :first_name, :last_name, :id_card, ' +
          '  :birth_date, :gender, :blood_type, :phone, :email, :address, :allergies, ' +
          '  :emergency_contact, :emergency_phone)';
        
        Q.ParamByName('patient_code').AsString := GeneratePatientCode;
      end
      else
      begin
        Q.SQL.Text := 
          'UPDATE patients SET first_name = :first_name, last_name = :last_name, ' +
          '  id_card = :id_card, birth_date = :birth_date, gender = :gender, ' +
          '  blood_type = :blood_type, phone = :phone, email = :email, ' +
          '  address = :address, allergies = :allergies, ' +
          '  emergency_contact = :emergency_contact, emergency_phone = :emergency_phone ' +
          'WHERE patient_id = :patient_id';
        
        Q.ParamByName('patient_id').AsInteger := FCurrentPatientID;
      end;
      
      Q.ParamByName('first_name').AsString := Trim(edtFirstName.Text);
      Q.ParamByName('last_name').AsString := Trim(edtLastName.Text);
      
      if Trim(edtIDCard.Text) = '' then
        Q.ParamByName('id_card').Clear
      else
        Q.ParamByName('id_card').AsString := Trim(edtIDCard.Text);
        
      Q.ParamByName('birth_date').AsDate := dtpBirthDate.Date;
      
      case cmbGender.ItemIndex of
        0: Q.ParamByName('gender').AsString := 'M';
        1: Q.ParamByName('gender').AsString := 'F';
        else Q.ParamByName('gender').AsString := 'Other';
      end;
      
      Q.ParamByName('blood_type').AsString := cmbBloodType.Text;
      Q.ParamByName('phone').AsString := Trim(edtPhone.Text);
      Q.ParamByName('email').AsString := Trim(edtEmail.Text);
      Q.ParamByName('address').AsString := Trim(memAddress.Text);
      Q.ParamByName('allergies').AsString := Trim(memAllergies.Text);
      Q.ParamByName('emergency_contact').AsString := Trim(edtEmergencyContact.Text);
      Q.ParamByName('emergency_phone').AsString := Trim(edtEmergencyPhone.Text);
      
      Q.ExecSQL;
      ZConn.Commit;
      
      if FIsNewRecord then
        ShowMessage('เพิ่มข้อมูลผู้ป่วยสำเร็จ')
      else
        ShowMessage('อัปเดตข้อมูลผู้ป่วยสำเร็จ');
        
      FIsNewRecord := False;
      FIsEditing := False;
      EnableEditing(False);
      LoadPatients(edtSearch.Text);
      
    except
      on E: Exception do
      begin
        ZConn.Rollback;
        if Pos('Duplicate entry', E.Message) > 0 then
          ShowMessage('ข้อมูลซ้ำ: เลขบัตรประชาชนนี้มีในระบบแล้ว')
        else
          ShowMessage('ไม่สามารถบันทึกข้อมูลได้: ' + E.Message);
      end;
    end;
    
  finally
    Q.Free;
  end;
end;

procedure TfrmHospital.btnDeleteClick(Sender: TObject);
begin
  if FCurrentPatientID < 0 then
  begin
    ShowMessage('กรุณาเลือกผู้ป่วยที่ต้องการลบ');
    Exit;
  end;
  
  if MessageDlg('ยืนยันการลบ', 
                'คุณต้องการลบข้อมูลผู้ป่วยนี้หรือไม่?',
                mtConfirmation, [mbYes, mbNo], 0) = mrYes then
  begin
    var Q := TZQuery.Create(nil);
    try
      Q.Connection := ZConn;
      
      try
        ZConn.StartTransaction;
        
        // ตรวจสอบว่ามีนัดหมายอยู่หรือไม่
        Q.SQL.Text := 'SELECT COUNT(*) FROM appointments WHERE patient_id = :id AND status NOT IN ("completed", "cancelled")';
        Q.ParamByName('id').AsInteger := FCurrentPatientID;
        Q.Open;
        
        if Q.Fields[0].AsInteger > 0 then
        begin
          Q.Close;
          ZConn.Rollback;
          ShowMessage('ไม่สามารถลบได้: ผู้ป่วยมีนัดหมายที่ยังไม่เสร็จสิ้น');
          Exit;
        end;
        Q.Close;
        
        Q.SQL.Text := 'DELETE FROM patients WHERE patient_id = :id';
        Q.ParamByName('id').AsInteger := FCurrentPatientID;
        Q.ExecSQL;
        
        ZConn.Commit;
        ShowMessage('ลบข้อมูลผู้ป่วยสำเร็จ');
        
        ClearPatientForm;
        LoadPatients;
        
      except
        on E: Exception do
        begin
          ZConn.Rollback;
          ShowMessage('ไม่สามารถลบข้อมูลได้: ' + E.Message);
        end;
      end;
      
    finally
      Q.Free;
    end;
  end;
end;

procedure TfrmHospital.btnCancelClick(Sender: TObject);
begin
  if FIsNewRecord then
    ClearPatientForm
  else if FCurrentPatientID > 0 then
    LoadPatientDetails(FCurrentPatientID);
    
  FIsNewRecord := False;
  FIsEditing := False;
  EnableEditing(False);
end;

procedure TfrmHospital.btnSearchClick(Sender: TObject);
begin
  LoadPatients(Trim(edtSearch.Text));
end;

procedure TfrmHospital.btnClearSearchClick(Sender: TObject);
begin
  edtSearch.Text := '';
  LoadPatients;
end;

procedure TfrmHospital.dbgPatientsSelectionChange(Sender: TObject);
begin
  if not qPatients.EOF and not qPatients.BOF then
  begin
    FCurrentPatientID := qPatients.FieldByName('patient_id').AsInteger;
    LoadPatientDetails(FCurrentPatientID);
    btnDelete.Enabled := True;
  end;
end;

end.
```

---

## Backup และ Restore

```pascal
unit DatabaseBackup;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Process;

type
  TDatabaseBackup = class
  private
    FHost, FUser, FPassword, FDatabase: string;
    FPort: Integer;
    FMySQLBinPath: string;
    
  public
    constructor Create(const Host, User, Password, Database: string;
                      Port: Integer = 3306);
    
    function Backup(const OutputFile: string): Boolean;
    function Restore(const InputFile: string): Boolean;
    function BackupTable(const TableName, OutputFile: string): Boolean;
    function GetBackupInfo(const BackupFile: string): string;
  end;

implementation

constructor TDatabaseBackup.Create(const Host, User, Password, Database: string;
                                   Port: Integer = 3306);
begin
  FHost := Host;
  FUser := User;
  FPassword := Password;
  FDatabase := Database;
  FPort := Port;
  
  // ค้นหา mysqldump
  {$IFDEF WINDOWS}
  FMySQLBinPath := 'C:\Program Files\MySQL\MySQL Server 8.0\bin\';
  {$ELSE}
  FMySQLBinPath := '/usr/bin/';
  {$ENDIF}
end;

function TDatabaseBackup.Backup(const OutputFile: string): Boolean;
var
  Process: TProcess;
  OutputSL: TStringList;
begin
  Result := False;
  
  Process := TProcess.Create(nil);
  try
    Process.Executable := FMySQLBinPath + 'mysqldump';
    Process.Parameters.Add('--host=' + FHost);
    Process.Parameters.Add('--port=' + IntToStr(FPort));
    Process.Parameters.Add('--user=' + FUser);
    Process.Parameters.Add('--password=' + FPassword);
    Process.Parameters.Add('--single-transaction');  // สำหรับ InnoDB
    Process.Parameters.Add('--routines');             // รวม stored procedures
    Process.Parameters.Add('--triggers');             // รวม triggers
    Process.Parameters.Add('--result-file=' + OutputFile);
    Process.Parameters.Add(FDatabase);
    
    Process.Options := [poWaitOnExit, poUsePipes];
    Process.Execute;
    
    Result := Process.ExitCode = 0;
    
    if Result then
      WriteLn('Backup สำเร็จ: ' + OutputFile)
    else
    begin
      // อ่าน error output
      OutputSL := TStringList.Create;
      try
        OutputSL.LoadFromStream(Process.Stderr);
        WriteLn('Backup ล้มเหลว: ' + OutputSL.Text);
      finally
        OutputSL.Free;
      end;
    end;
    
  finally
    Process.Free;
  end;
end;

function TDatabaseBackup.Restore(const InputFile: string): Boolean;
var
  Process: TProcess;
begin
  Result := False;
  
  if not FileExists(InputFile) then
  begin
    WriteLn('ไม่พบไฟล์ backup: ' + InputFile);
    Exit;
  end;
  
  Process := TProcess.Create(nil);
  try
    Process.Executable := FMySQLBinPath + 'mysql';
    Process.Parameters.Add('--host=' + FHost);
    Process.Parameters.Add('--port=' + IntToStr(FPort));
    Process.Parameters.Add('--user=' + FUser);
    Process.Parameters.Add('--password=' + FPassword);
    Process.Parameters.Add(FDatabase);
    Process.Parameters.Add('-e');
    Process.Parameters.Add('source ' + InputFile);
    
    Process.Options := [poWaitOnExit];
    Process.Execute;
    
    Result := Process.ExitCode = 0;
    
    if Result then
      WriteLn('Restore สำเร็จจากไฟล์: ' + InputFile)
    else
      WriteLn('Restore ล้มเหลว');
      
  finally
    Process.Free;
  end;
end;

function TDatabaseBackup.BackupTable(const TableName, OutputFile: string): Boolean;
var
  Process: TProcess;
begin
  Result := False;
  
  Process := TProcess.Create(nil);
  try
    Process.Executable := FMySQLBinPath + 'mysqldump';
    Process.Parameters.Add('--host=' + FHost);
    Process.Parameters.Add('--user=' + FUser);
    Process.Parameters.Add('--password=' + FPassword);
    Process.Parameters.Add('--result-file=' + OutputFile);
    Process.Parameters.Add(FDatabase);
    Process.Parameters.Add(TableName);
    
    Process.Options := [poWaitOnExit];
    Process.Execute;
    
    Result := Process.ExitCode = 0;
  finally
    Process.Free;
  end;
end;

function TDatabaseBackup.GetBackupInfo(const BackupFile: string): string;
var
  SL: TStringList;
  i: Integer;
  FileSize: Int64;
begin
  Result := '';
  
  if not FileExists(BackupFile) then
    Exit;
    
  FileSize := FileSize(BackupFile);
  
  SL := TStringList.Create;
  try
    SL.LoadFromFile(BackupFile);
    
    var ServerVersion := '';
    var BackupDate := '';
    
    for i := 0 to Min(20, SL.Count - 1) do
    begin
      if Pos('Server version:', SL[i]) > 0 then
        ServerVersion := Trim(Copy(SL[i], Pos('Server version:', SL[i]) + 15, 100));
      if Pos('Dump completed on', SL[i]) > 0 then
        BackupDate := Trim(Copy(SL[i], Pos('--', SL[i]) + 3, 100));
    end;
    
    Result := Format('ไฟล์: %s'#13#10 +
                    'ขนาด: %s'#13#10 +
                    'Server: %s'#13#10 +
                    'วันที่: %s',
      [ExtractFileName(BackupFile),
       FormatFloat('#,##0', FileSize) + ' bytes',
       ServerVersion,
       BackupDate]);
  finally
    SL.Free;
  end;
end;

end.
```

---

## แบบฝึกหัด 15 ข้อ

**ข้อ 1:** เขียนโปรแกรมเชื่อมต่อ MySQL และแสดงรายชื่อฐานข้อมูลทั้งหมด
```pascal
// แนวทาง: ใช้ SHOW DATABASES; query
// ลองใช้ ZConnection และ ZQuery
// แสดงผลใน ListBox
```

**ข้อ 2:** สร้างฟังก์ชันตรวจสอบความถูกต้องของบัตรประชาชน 13 หลัก
```pascal
function ValidateThaiIDCard(const IDCard: string): Boolean;
// ตรวจสอบ:
// 1. ความยาว 13 หลัก
// 2. เป็นตัวเลขทั้งหมด
// 3. Check digit ถูกต้อง (ใช้ algorithm มาตรฐาน)
```

**ข้อ 3:** เขียนโปรแกรมค้นหาผู้ป่วยแบบ fuzzy search
```pascal
// ใช้ SOUNDEX() หรือ LIKE เพื่อค้นหาชื่อที่ออกเสียงคล้ายกัน
// แสดงผลในตาราง
```

**ข้อ 4:** สร้างระบบ log การเข้าถึงข้อมูล
```pascal
// บันทึกทุกครั้งที่มีการ INSERT/UPDATE/DELETE
// เก็บ: ผู้ใช้, วันเวลา, ตาราง, action, ข้อมูลก่อน/หลัง
```

**ข้อ 5:** เขียน stored procedure สำหรับสร้างรายงานสรุปประจำวัน
```sql
-- Procedure ที่รับ date parameter
-- คืนค่า: จำนวนนัดหมาย, ผู้ป่วยใหม่, การตรวจที่เสร็จ
```

**ข้อ 6:** สร้างระบบ pagination สำหรับ DBGrid
```pascal
// แสดงข้อมูล 50 รายการต่อหน้า
// มีปุ่ม First, Previous, Next, Last
// แสดง "หน้า X จาก Y"
```

**ข้อ 7:** เขียนโปรแกรม export ข้อมูลผู้ป่วยเป็น CSV
```pascal
// ส่งออกทุก field
// รองรับ encoding UTF-8 กับ TIS-620
// มี header row
```

**ข้อ 8:** สร้างระบบ full-text search สำหรับค้นหาผล diagnosis
```sql
-- ใช้ MySQL FULLTEXT index
-- สร้าง index บนคอลัมน์ diagnosis
-- ใช้ MATCH...AGAINST syntax
```

**ข้อ 9:** เขียนโปรแกรมตรวจสอบ duplicate ก่อน insert
```pascal
// ก่อน insert ผู้ป่วยใหม่
// ตรวจสอบ: ชื่อ+นามสกุล+วันเกิดซ้ำ หรือ บัตรประชาชนซ้ำ
// แสดง dialog เพื่อถามผู้ใช้
```

**ข้อ 10:** สร้างระบบ connection health check
```pascal
// ตรวจสอบว่า connection ยังใช้งานได้ทุก 5 นาที
// reconnect อัตโนมัติหากหลุด
// แสดงสถานะใน status bar
```

**ข้อ 11:** เขียน trigger ใน MySQL สำหรับ audit trail
```sql
-- BEFORE UPDATE trigger บน patients table
-- บันทึกค่าเก่าไว้ใน audit_log table
```

**ข้อ 12:** สร้างรายงานนัดหมายรายสัปดาห์
```pascal
// Query นัดหมายใน 7 วันข้างหน้า
// จัดกลุ่มตามแพทย์
// แสดงใน grid พร้อม color coding ตามสถานะ
```

**ข้อ 13:** เขียนระบบ scheduled backup
```pascal
// ใช้ TTimer เพื่อ backup อัตโนมัติ
// Backup ทุกวันเที่ยงคืน
// เก็บ backup ไม่เกิน 30 วัน
// ส่ง email แจ้งเตือนเมื่อ backup สำเร็จ/ล้มเหลว
```

**ข้อ 14:** สร้างระบบ import ข้อมูลจาก Excel/CSV
```pascal
// อ่านไฟล์ CSV ที่มีข้อมูลผู้ป่วย
// Validate ข้อมูลก่อน import
// แสดง progress bar ระหว่าง import
// รายงานสรุปหลัง import (สำเร็จ/ล้มเหลว)
```

**ข้อ 15:** โปรเจกต์สุดท้าย - ระบบจองนัดหมายออนไลน์
```pascal
// สร้าง form สำหรับ:
// 1. เลือกแผนกและแพทย์
// 2. ดูตาราง slot ที่ว่าง (calendar view)
// 3. จองนัดหมาย
// 4. ส่ง email ยืนยัน
// 5. แสดงประวัตินัดหมาย
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
- การเชื่อมต่อ MySQL/MariaDB ด้วย ZEOS และ SQLdb
- การใช้ Prepared Statements ป้องกัน SQL Injection
- การจัดการ Transactions
- Stored Procedures และการเรียกใช้งาน
- Connection Pooling สำหรับระบบที่มีผู้ใช้จำนวนมาก
- Error Handling ที่เหมาะสม
- การ Backup และ Restore ฐานข้อมูล
- ระบบโรงพยาบาลตัวอย่างแบบสมบูรณ์

บทถัดไปจะเรียนรู้การใช้งาน PostgreSQL ซึ่งมีความสามารถขั้นสูงกว่า MySQL ในบางด้าน
