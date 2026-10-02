# Part 29 - Database Programming พื้นฐาน

## บทนำ

การเขียนโปรแกรมฐานข้อมูลเป็นทักษะสำคัญในการพัฒนาแอปพลิเคชันจริง Lazarus มีสถาปัตยกรรมที่แข็งแกร่งสำหรับการทำงานกับฐานข้อมูล โดยใช้ component-based approach ที่แยกส่วน UI ออกจากส่วนข้อมูล

---

## 29.1 แนวคิดฐานข้อมูลเชิงสัมพันธ์

### โครงสร้างพื้นฐาน

```
ฐานข้อมูล (Database)
├── ตาราง (Table) - Students
│   ├── คอลัมน์ (Column/Field) - StudentID, Name, Age, GPA
│   └── แถว (Row/Record) - ข้อมูลนักเรียนแต่ละคน
├── ตาราง - Courses
│   └── CourseID, CourseName, Credits
└── ตาราง - Enrollments
    └── StudentID, CourseID, Grade (ตารางความสัมพันธ์)
```

### ชนิดของ Keys

```sql
-- Primary Key: ระบุ record ได้ไม่ซ้ำกัน
CREATE TABLE Students (
    StudentID INTEGER PRIMARY KEY,
    Name TEXT NOT NULL,
    Age INTEGER,
    GPA REAL
);

-- Foreign Key: อ้างอิง Primary Key ของตารางอื่น
CREATE TABLE Enrollments (
    EnrollID INTEGER PRIMARY KEY,
    StudentID INTEGER REFERENCES Students(StudentID),
    CourseID INTEGER REFERENCES Courses(CourseID),
    Grade TEXT
);

-- Composite Key: ใช้หลาย field เป็น primary key
CREATE TABLE CourseSchedule (
    CourseID INTEGER,
    RoomID INTEGER,
    TimeSlot TEXT,
    PRIMARY KEY (CourseID, TimeSlot)
);
```

---

## 29.2 SQL พื้นฐาน

### CREATE TABLE

```sql
-- สร้างตาราง Products
CREATE TABLE IF NOT EXISTS Products (
    ProductID   INTEGER PRIMARY KEY AUTOINCREMENT,
    Name        TEXT NOT NULL,
    Description TEXT,
    Price       REAL NOT NULL DEFAULT 0.0,
    Stock       INTEGER NOT NULL DEFAULT 0,
    CategoryID  INTEGER,
    CreatedAt   TEXT DEFAULT (datetime('now')),
    IsActive    INTEGER DEFAULT 1,
    FOREIGN KEY (CategoryID) REFERENCES Categories(CategoryID)
);

-- สร้างตาราง Categories
CREATE TABLE IF NOT EXISTS Categories (
    CategoryID  INTEGER PRIMARY KEY AUTOINCREMENT,
    Name        TEXT NOT NULL UNIQUE,
    Description TEXT
);

-- สร้าง Index
CREATE INDEX idx_products_category ON Products(CategoryID);
CREATE INDEX idx_products_name ON Products(Name);
```

### SELECT

```sql
-- เลือกทุก column
SELECT * FROM Products;

-- เลือก columns ที่ต้องการ
SELECT ProductID, Name, Price FROM Products;

-- กรองด้วย WHERE
SELECT * FROM Products WHERE Price > 100;
SELECT * FROM Products WHERE Stock = 0 AND IsActive = 1;
SELECT * FROM Products WHERE Name LIKE '%กาแฟ%';

-- เรียงลำดับ
SELECT * FROM Products ORDER BY Price ASC;
SELECT * FROM Products ORDER BY Price DESC, Name ASC;

-- จำกัดจำนวน
SELECT * FROM Products LIMIT 10;
SELECT * FROM Products LIMIT 10 OFFSET 20;  -- หน้า 3

-- Aggregate functions
SELECT COUNT(*) FROM Products;
SELECT AVG(Price) FROM Products;
SELECT SUM(Stock * Price) AS TotalValue FROM Products;
SELECT MIN(Price), MAX(Price) FROM Products;

-- GROUP BY
SELECT CategoryID, COUNT(*) AS Count, AVG(Price) AS AvgPrice
FROM Products
GROUP BY CategoryID
HAVING COUNT(*) > 2;

-- JOIN
SELECT p.Name, c.Name AS Category
FROM Products p
JOIN Categories c ON p.CategoryID = c.CategoryID;

-- LEFT JOIN
SELECT p.Name, c.Name AS Category
FROM Products p
LEFT JOIN Categories c ON p.CategoryID = c.CategoryID;

-- Subquery
SELECT * FROM Products 
WHERE Price > (SELECT AVG(Price) FROM Products);
```

### INSERT

```sql
-- เพิ่มแถวเดียว
INSERT INTO Products (Name, Price, Stock, CategoryID)
VALUES ('กาแฟดำ', 50.00, 100, 1);

-- เพิ่มหลายแถว
INSERT INTO Products (Name, Price, Stock)
VALUES 
    ('ชาเขียว', 45.00, 80),
    ('น้ำส้ม', 35.00, 120),
    ('นม', 25.00, 200);

-- INSERT OR IGNORE (ถ้าซ้ำให้ข้าม)
INSERT OR IGNORE INTO Categories (Name) VALUES ('เครื่องดื่ม');

-- INSERT OR REPLACE (ถ้าซ้ำให้แทนที่)
INSERT OR REPLACE INTO Products (ProductID, Name, Price, Stock)
VALUES (1, 'กาแฟดำพิเศษ', 60.00, 90);
```

### UPDATE

```sql
-- อัปเดตทุก row (ระวัง!)
UPDATE Products SET IsActive = 0;

-- อัปเดตด้วย WHERE
UPDATE Products SET Price = Price * 1.07 WHERE CategoryID = 1;
UPDATE Products SET Stock = Stock - 1 WHERE ProductID = 5;

-- อัปเดตหลาย columns
UPDATE Products 
SET Name = 'กาแฟดำพรีเมียม', 
    Price = 65.00, 
    Description = 'กาแฟอาราบิก้า 100%'
WHERE ProductID = 1;
```

### DELETE

```sql
-- ลบทุก row (ระวังมาก!)
DELETE FROM Products;

-- ลบด้วย WHERE
DELETE FROM Products WHERE IsActive = 0;
DELETE FROM Products WHERE Stock = 0 AND Price < 10;

-- ลบ row เดียว
DELETE FROM Products WHERE ProductID = 5;
```

---

## 29.3 Database Architecture ใน Lazarus

```
สถาปัตยกรรม Database ใน Lazarus:

┌─────────────────────────────────┐
│           UI Layer              │
│  TDBGrid, TDBEdit, TDBLabel...  │
└─────────────┬───────────────────┘
              │ (DB-aware controls)
              ▼
┌─────────────────────────────────┐
│         TDataSource             │
│  เชื่อม Dataset กับ DB Controls  │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│         TDataSet                │
│  (TSQLQuery, TTable, TQuery...) │
│  - Navigate records             │
│  - Edit data                    │
│  - Filter/Sort                  │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│    Database Connection          │
│  (TSQLite3Connection,           │
│   TMySQLConnection...)          │
└─────────────┬───────────────────┘
              │
              ▼
┌─────────────────────────────────┐
│        Database                 │
│  (SQLite, MySQL, PostgreSQL...) │
└─────────────────────────────────┘
```

---

## 29.4 TDataSource

```pascal
program TDataSourceDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes, DB;

// TDataSource เชื่อม Dataset กับ DB-aware controls
// คุณสมบัติหลัก:
// - DataSet: dataset ที่เชื่อมอยู่
// - Enabled: เปิด/ปิดการเชื่อม
// - AutoEdit: เข้า edit mode อัตโนมัติ

// Events:
// - OnDataChange: เมื่อข้อมูลเปลี่ยน
// - OnStateChange: เมื่อ state ของ dataset เปลี่ยน
// - OnUpdateData: ก่อนที่ dataset จะ post

procedure ShowDataSourceUsage;
begin
  WriteLn('TDataSource - เชื่อม Dataset กับ Controls');
  WriteLn;
  WriteLn('การใช้งาน:');
  WriteLn('1. วาง TDataSource บน Form');
  WriteLn('2. ตั้งค่า DataSource.DataSet := MyQuery (หรือ Table)');
  WriteLn('3. ตั้งค่า DB controls: DBEdit.DataSource := DataSource1');
  WriteLn('                         DBEdit.DataField := "Name"');
  WriteLn;
  WriteLn('Events:');
  WriteLn('- OnDataChange: รับรู้เมื่อข้อมูล field เปลี่ยน');
  WriteLn('- OnStateChange: รับรู้เมื่อ dataset state เปลี่ยน');
  WriteLn('- OnUpdateData: รับรู้ก่อน Post');
end;

// ตัวอย่างการใช้ TDataSource ใน code
procedure DataSourceCodeExample;
var
  DS: TDataSource;
begin
  DS := TDataSource.Create(nil);
  try
    // เชื่อม dataset (ในตัวอย่างนี้ไม่มี actual dataset)
    // DS.DataSet := MyQuery;
    
    // Events
    DS.OnDataChange := @(procedure(Sender: TObject; Field: TField)
    begin
      if Field <> nil then
        WriteLn('ข้อมูลเปลี่ยน: ', Field.FieldName)
      else
        WriteLn('เปลี่ยนไปยัง record ใหม่');
    end);
    
    DS.OnStateChange := @(procedure(Sender: TObject)
    begin
      WriteLn('State เปลี่ยน');
    end);
    
    WriteLn('TDataSource สร้างสำเร็จ');
  finally
    DS.Free;
  end;
end;

begin
  ShowDataSourceUsage;
  DataSourceCodeExample;
  ReadLn;
end.
```

---

## 29.5 TDataSet Hierarchy

```pascal
program TDataSetHierarchy;
{$mode objfpc}{$H+}

uses
  SysUtils, DB;

procedure ShowHierarchy;
begin
  WriteLn('=== TDataSet Hierarchy ===');
  WriteLn;
  WriteLn('TDataSet (DB.pas) - Base class');
  WriteLn('  |-- TBufDataset - In-memory dataset');
  WriteLn('  |     (rửAn be filled with data in code)');
  WriteLn('  |');
  WriteLn('  |-- TCustomSQLQuery (SQLdb.pas)');
  WriteLn('  |     |-- TSQLQuery - Standard SQL query');
  WriteLn('  |');
  WriteLn('  |-- TMSSQLQuery (MySQL, PostgreSQL, etc.)');
  WriteLn('  |');
  WriteLn('  |-- TDBFDataset (dBASE files)');
  WriteLn('  |');
  WriteLn('  └-- TMemDataset (In-memory)');
end;

procedure DemoBufDataset;
var
  DS: TBufDataset;
begin
  WriteLn(#10'=== TBufDataset Demo (In-Memory) ===');
  
  DS := TBufDataset.Create(nil);
  try
    // กำหนด fields
    DS.FieldDefs.Add('ID', ftInteger);
    DS.FieldDefs.Add('Name', ftString, 100);
    DS.FieldDefs.Add('Age', ftInteger);
    DS.FieldDefs.Add('Score', ftFloat);
    DS.FieldDefs.Add('Active', ftBoolean);
    
    // เปิด dataset
    DS.CreateDataset;
    DS.Open;
    
    WriteLn('สร้าง Dataset สำเร็จ');
    WriteLn('Fields:');
    for var I := 0 to DS.Fields.Count - 1 do
      WriteLn(Format('  %d. %-15s %s', 
                     [I + 1, DS.Fields[I].FieldName, 
                      GetEnumName(TypeInfo(TFieldType), Ord(DS.Fields[I].DataType))]));
    
    // เพิ่มข้อมูล
    WriteLn(#10'เพิ่มข้อมูล:');
    
    DS.Append;
    DS.FieldByName('ID').AsInteger := 1;
    DS.FieldByName('Name').AsString := 'สมชาย';
    DS.FieldByName('Age').AsInteger := 25;
    DS.FieldByName('Score').AsFloat := 85.5;
    DS.FieldByName('Active').AsBoolean := True;
    DS.Post;
    
    DS.Append;
    DS.FieldByName('ID').AsInteger := 2;
    DS.FieldByName('Name').AsString := 'สมหญิง';
    DS.FieldByName('Age').AsInteger := 23;
    DS.FieldByName('Score').AsFloat := 92.0;
    DS.FieldByName('Active').AsBoolean := True;
    DS.Post;
    
    DS.Append;
    DS.FieldByName('ID').AsInteger := 3;
    DS.FieldByName('Name').AsString := 'วิชัย';
    DS.FieldByName('Age').AsInteger := 28;
    DS.FieldByName('Score').AsFloat := 78.3;
    DS.FieldByName('Active').AsBoolean := False;
    DS.Post;
    
    WriteLn('Count: ', DS.RecordCount);
    
    // วน loop อ่านข้อมูล
    WriteLn(#10'ข้อมูลทั้งหมด:');
    DS.First;
    while not DS.EOF do
    begin
      WriteLn(Format('  ID: %-3d | Name: %-10s | Age: %-3d | Score: %-6.1f | Active: %s',
                     [DS.FieldByName('ID').AsInteger,
                      DS.FieldByName('Name').AsString,
                      DS.FieldByName('Age').AsInteger,
                      DS.FieldByName('Score').AsFloat,
                      BoolToStr(DS.FieldByName('Active').AsBoolean, 'ใช่', 'ไม่')]));
      DS.Next;
    end;
    
    DS.Close;
  finally
    DS.Free;
  end;
end;

begin
  ShowHierarchy;
  DemoBufDataset;
  ReadLn;
end.
```

---

## 29.6 TField และ TFields

```pascal
program TFieldDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, DB;

procedure DemoFields;
var
  DS: TBufDataset;
begin
  WriteLn('=== TField และ TFields Demo ===');
  
  DS := TBufDataset.Create(nil);
  try
    // กำหนด fields หลายชนิด
    DS.FieldDefs.Add('IntField', ftInteger);
    DS.FieldDefs.Add('StrField', ftString, 50);
    DS.FieldDefs.Add('FloatField', ftFloat);
    DS.FieldDefs.Add('DateField', ftDate);
    DS.FieldDefs.Add('DateTimeField', ftDateTime);
    DS.FieldDefs.Add('BoolField', ftBoolean);
    DS.FieldDefs.Add('CurrField', ftCurrency);
    DS.FieldDefs.Add('LargeStrField', ftMemo);
    
    DS.CreateDataset;
    DS.Open;
    
    // เพิ่มข้อมูล
    DS.Append;
    DS.FieldByName('IntField').AsInteger := 42;
    DS.FieldByName('StrField').AsString := 'ทดสอบ';
    DS.FieldByName('FloatField').AsFloat := 3.14159;
    DS.FieldByName('DateField').AsDateTime := Date;
    DS.FieldByName('DateTimeField').AsDateTime := Now;
    DS.FieldByName('BoolField').AsBoolean := True;
    DS.FieldByName('CurrField').AsCurrency := 1234.56;
    DS.FieldByName('LargeStrField').AsString := 'ข้อความยาวๆ สำหรับทดสอบ Memo field';
    DS.Post;
    
    DS.First;
    
    // Properties ของ TField
    WriteLn('Fields Properties:');
    var F: TField;
    for var I := 0 to DS.Fields.Count - 1 do
    begin
      F := DS.Fields[I];
      WriteLn(Format('  Field: %-20s | Type: %-15s | Value: %s',
                     [F.FieldName,
                      GetEnumName(TypeInfo(TFieldType), Ord(F.DataType)),
                      F.AsString]));
    end;
    
    // TField Properties
    WriteLn(#10'คุณสมบัติของ TField:');
    F := DS.FieldByName('IntField');
    WriteLn('  FieldName: ', F.FieldName);
    WriteLn('  DataType: ', GetEnumName(TypeInfo(TFieldType), Ord(F.DataType)));
    WriteLn('  AsInteger: ', F.AsInteger);
    WriteLn('  AsString: ', F.AsString);
    WriteLn('  IsNull: ', F.IsNull);
    WriteLn('  ReadOnly: ', F.ReadOnly);
    WriteLn('  Required: ', F.Required);
    WriteLn('  DisplayLabel: ', F.DisplayLabel);
    WriteLn('  FieldNo: ', F.FieldNo);
    
    // AsVariant
    WriteLn(#10'AsVariant:');
    for var I := 0 to DS.Fields.Count - 1 do
    begin
      var V := DS.Fields[I].AsVariant;
      WriteLn(Format('  %s = %s', [DS.Fields[I].FieldName, VarToStr(V)]));
    end;
    
    // NULL handling
    WriteLn(#10'NULL Handling:');
    DS.Append;
    DS.FieldByName('IntField').Clear;  // Set to NULL
    DS.FieldByName('StrField').AsString := 'partial data';
    DS.Post;
    
    DS.Last;  // ไปยัง record ที่เพิ่มล่าสุด
    WriteLn('IntField is NULL: ', DS.FieldByName('IntField').IsNull);
    WriteLn('IntField AsInteger (when NULL): ', DS.FieldByName('IntField').AsInteger);
    // จะได้ค่าเริ่มต้น (0 สำหรับ integer)
    
    DS.Close;
  finally
    DS.Free;
  end;
end;

begin
  DemoFields;
  ReadLn;
end.
```

---

## 29.7 TDataSet Navigation

```pascal
program DataSetNavigation;
{$mode objfpc}{$H+}

uses
  SysUtils, DB;

procedure CreateSampleData(DS: TBufDataset);
var
  I: Integer;
begin
  for I := 1 to 10 do
  begin
    DS.Append;
    DS.FieldByName('ID').AsInteger := I;
    DS.FieldByName('Name').AsString := Format('รายการที่ %d', [I]);
    DS.FieldByName('Value').AsFloat := I * 10.5;
    DS.Post;
  end;
end;

procedure ShowCurrentRecord(DS: TDataSet);
begin
  WriteLn(Format('  ID: %d | Name: %s | Value: %.1f | BOF: %s | EOF: %s',
                 [DS.FieldByName('ID').AsInteger,
                  DS.FieldByName('Name').AsString,
                  DS.FieldByName('Value').AsFloat,
                  BoolToStr(DS.BOF, 'True', 'False'),
                  BoolToStr(DS.EOF, 'True', 'False')]));
end;

procedure DemoNavigation;
var
  DS: TBufDataset;
begin
  WriteLn('=== Dataset Navigation ===');
  
  DS := TBufDataset.Create(nil);
  try
    DS.FieldDefs.Add('ID', ftInteger);
    DS.FieldDefs.Add('Name', ftString, 50);
    DS.FieldDefs.Add('Value', ftFloat);
    DS.CreateDataset;
    DS.Open;
    
    CreateSampleData(DS);
    WriteLn('สร้างข้อมูล ', DS.RecordCount, ' records');
    
    // First
    WriteLn(#10'1. First:');
    DS.First;
    ShowCurrentRecord(DS);
    
    // Last
    WriteLn(#10'2. Last:');
    DS.Last;
    ShowCurrentRecord(DS);
    
    // Prior
    WriteLn(#10'3. Prior (จาก Last):');
    DS.Prior;
    ShowCurrentRecord(DS);
    
    // Next
    WriteLn(#10'4. Next:');
    DS.Next;
    ShowCurrentRecord(DS);
    
    // MoveBy (เลื่อนหลาย records)
    WriteLn(#10'5. MoveBy(-3) จาก record ปัจจุบัน:');
    DS.MoveBy(-3);
    ShowCurrentRecord(DS);
    
    WriteLn(#10'6. MoveBy(5):');
    DS.MoveBy(5);
    ShowCurrentRecord(DS);
    
    // BOF/EOF
    WriteLn(#10'7. Navigation ด้วย BOF/EOF:');
    DS.First;
    while not DS.EOF do
    begin
      WriteLn('  ', DS.FieldByName('Name').AsString);
      DS.Next;
    end;
    
    // RecNo
    WriteLn(#10'8. RecNo (record number):');
    DS.First;
    DS.Next;  // ไปที่ record 2
    DS.Next;  // ไปที่ record 3
    WriteLn('RecNo: ', DS.RecNo);
    WriteLn('RecordCount: ', DS.RecordCount);
    
    // Bookmark
    WriteLn(#10'9. Bookmark:');
    DS.First;
    DS.MoveBy(4);  // ไปที่ record 5
    var BM := DS.Bookmark;
    WriteLn('บันทึก Bookmark ที่ record: ', DS.FieldByName('ID').AsInteger);
    
    DS.Last;  // ไปที่ record สุดท้าย
    WriteLn('ไปที่ Last: ', DS.FieldByName('ID').AsInteger);
    
    DS.Bookmark := BM;  // กลับไปที่ bookmark
    WriteLn('กลับที่ Bookmark: ', DS.FieldByName('ID').AsInteger);
    
    DS.Close;
  finally
    DS.Free;
  end;
end;

procedure DemoRecordCount;
var
  DS: TBufDataset;
begin
  WriteLn(#10'=== RecordCount และ Active ===');
  
  DS := TBufDataset.Create(nil);
  try
    DS.FieldDefs.Add('ID', ftInteger);
    DS.FieldDefs.Add('Name', ftString, 50);
    DS.FieldDefs.Add('Value', ftFloat);
    DS.CreateDataset;
    
    WriteLn('Active ก่อน Open: ', DS.Active);
    
    DS.Open;
    WriteLn('Active หลัง Open: ', DS.Active);
    WriteLn('RecordCount (ว่าง): ', DS.RecordCount);
    
    CreateSampleData(DS);
    WriteLn('RecordCount (มีข้อมูล): ', DS.RecordCount);
    
    DS.Close;
    WriteLn('Active หลัง Close: ', DS.Active);
    
    // พยายาม access เมื่อ Close
    try
      var R := DS.RecordCount;  // อาจเกิด exception
      WriteLn('RecordCount หลัง Close: ', R);
    except
      on E: Exception do
        WriteLn('Error: ', E.Message);
    end;
    
  finally
    DS.Free;
  end;
end;

begin
  DemoNavigation;
  DemoRecordCount;
  ReadLn;
end.
```

---

## 29.8 TDataSet States

```pascal
program DataSetStates;
{$mode objfpc}{$H+}

uses
  SysUtils, DB;

function StateToString(State: TDataSetState): String;
begin
  case State of
    dsInactive:      Result := 'dsInactive';
    dsBrowse:        Result := 'dsBrowse';
    dsEdit:          Result := 'dsEdit';
    dsInsert:        Result := 'dsInsert';
    dsSetKey:        Result := 'dsSetKey';
    dsCalcFields:    Result := 'dsCalcFields';
    dsFilter:        Result := 'dsFilter';
    dsNewValue:      Result := 'dsNewValue';
    dsOldValue:      Result := 'dsOldValue';
    dsCurValue:      Result := 'dsCurValue';
    dsBlockRead:     Result := 'dsBlockRead';
    dsInternalCalc:  Result := 'dsInternalCalc';
    dsOpening:       Result := 'dsOpening';
    else             Result := 'Unknown';
  end;
end;

procedure DemoStates;
var
  DS: TBufDataset;
begin
  WriteLn('=== TDataSet States ===');
  WriteLn;
  WriteLn('States หลัก:');
  WriteLn('  dsInactive - ปิดอยู่');
  WriteLn('  dsBrowse   - กำลัง browse (อ่านข้อมูล)');
  WriteLn('  dsEdit     - กำลัง edit');
  WriteLn('  dsInsert   - กำลัง insert record ใหม่');
  
  DS := TBufDataset.Create(nil);
  try
    DS.FieldDefs.Add('ID', ftInteger);
    DS.FieldDefs.Add('Name', ftString, 50);
    DS.CreateDataset;
    
    // dsInactive
    WriteLn(#10'State ก่อน Open: ', StateToString(DS.State));
    
    DS.Open;
    // dsBrowse (ว่าง)
    WriteLn('State หลัง Open: ', StateToString(DS.State));
    
    // Insert state
    WriteLn(#10'Insert:');
    DS.Insert;
    WriteLn('State หลัง Insert: ', StateToString(DS.State));
    
    DS.FieldByName('ID').AsInteger := 1;
    DS.FieldByName('Name').AsString := 'ทดสอบ 1';
    
    // ยังอยู่ใน Insert state จนกว่าจะ Post หรือ Cancel
    WriteLn('State ระหว่าง Fill: ', StateToString(DS.State));
    
    DS.Post;
    WriteLn('State หลัง Post: ', StateToString(DS.State));  // กลับเป็น dsBrowse
    
    // Append state (เหมือน Insert แต่เพิ่มท้าย)
    WriteLn(#10'Append:');
    DS.Append;
    WriteLn('State หลัง Append: ', StateToString(DS.State));  // dsInsert
    DS.FieldByName('ID').AsInteger := 2;
    DS.FieldByName('Name').AsString := 'ทดสอบ 2';
    DS.Post;
    
    // Edit state
    WriteLn(#10'Edit:');
    DS.First;
    DS.Edit;
    WriteLn('State หลัง Edit: ', StateToString(DS.State));
    DS.FieldByName('Name').AsString := 'แก้ไขแล้ว';
    
    // Cancel (ยกเลิกการแก้ไข)
    DS.Cancel;
    WriteLn('State หลัง Cancel: ', StateToString(DS.State));
    WriteLn('ค่าหลัง Cancel: ', DS.FieldByName('Name').AsString);  // ค่าเดิม
    
    // Edit แล้ว Post
    DS.Edit;
    DS.FieldByName('Name').AsString := 'แก้ไขจริง';
    DS.Post;
    WriteLn('ค่าหลัง Post: ', DS.FieldByName('Name').AsString);
    
    // ตรวจสอบ state ก่อนทำงาน
    WriteLn(#10'ตรวจสอบ State:');
    WriteLn('กำลัง Browse: ', DS.State = dsBrowse);
    WriteLn('กำลัง Edit/Insert: ', DS.State in [dsEdit, dsInsert]);
    
    // CanModify
    WriteLn('CanModify: ', DS.CanModify);
    
    DS.Close;
    WriteLn(#10'State หลัง Close: ', StateToString(DS.State));
    
  finally
    DS.Free;
  end;
end;

begin
  DemoStates;
  ReadLn;
end.
```

---

## 29.9 Edit, Insert, Append, Post, Cancel, Delete

```pascal
program DataSetEditing;
{$mode objfpc}{$H+}

uses
  SysUtils, DB;

procedure DemoEditing;
var
  DS: TBufDataset;
begin
  WriteLn('=== Dataset Editing Operations ===');
  
  DS := TBufDataset.Create(nil);
  try
    DS.FieldDefs.Add('ID', ftInteger);
    DS.FieldDefs.Add('Name', ftString, 50);
    DS.FieldDefs.Add('Score', ftFloat);
    DS.CreateDataset;
    DS.Open;
    
    // INSERT - เพิ่มก่อน record ปัจจุบัน
    WriteLn(#10'1. Insert:');
    DS.Insert;
    DS.FieldByName('ID').AsInteger := 3;
    DS.FieldByName('Name').AsString := 'วิชัย';
    DS.FieldByName('Score').AsFloat := 78.5;
    DS.Post;
    WriteLn('Insert สำเร็จ: ', DS.FieldByName('Name').AsString);
    
    // APPEND - เพิ่มท้าย
    WriteLn(#10'2. Append:');
    DS.Append;
    DS.FieldByName('ID').AsInteger := 1;
    DS.FieldByName('Name').AsString := 'สมชาย';
    DS.FieldByName('Score').AsFloat := 85.0;
    DS.Post;
    
    DS.Append;
    DS.FieldByName('ID').AsInteger := 2;
    DS.FieldByName('Name').AsString := 'สมหญิง';
    DS.FieldByName('Score').AsFloat := 92.5;
    DS.Post;
    
    WriteLn('Append 2 records สำเร็จ, Total: ', DS.RecordCount);
    
    // แสดงข้อมูลทั้งหมด
    WriteLn(#10'ข้อมูลปัจจุบัน:');
    DS.First;
    while not DS.EOF do
    begin
      WriteLn(Format('  ID: %d, Name: %s, Score: %.1f',
                     [DS.FieldByName('ID').AsInteger,
                      DS.FieldByName('Name').AsString,
                      DS.FieldByName('Score').AsFloat]));
      DS.Next;
    end;
    
    // EDIT - แก้ไข
    WriteLn(#10'3. Edit:');
    DS.First;
    DS.Edit;
    DS.FieldByName('Score').AsFloat := 90.0;
    DS.Post;
    WriteLn('Edit สำเร็จ: ', DS.FieldByName('Name').AsString,
            ' Score = ', DS.FieldByName('Score').AsFloat:0:1);
    
    // CANCEL - ยกเลิกการแก้ไข
    WriteLn(#10'4. Cancel:');
    DS.First;
    var OldScore := DS.FieldByName('Score').AsFloat;
    DS.Edit;
    DS.FieldByName('Score').AsFloat := 50.0;
    WriteLn('ระหว่าง Edit: Score = ', DS.FieldByName('Score').AsFloat:0:1);
    DS.Cancel;
    WriteLn('หลัง Cancel: Score = ', DS.FieldByName('Score').AsFloat:0:1);
    WriteLn('ค่าเดิม ', OldScore:0:1, ' = ค่าหลัง Cancel ', DS.FieldByName('Score').AsFloat:0:1, ': ',
            OldScore = DS.FieldByName('Score').AsFloat);
    
    // DELETE - ลบ
    WriteLn(#10'5. Delete:');
    DS.First;  // ไปที่ record แรก
    var DeletedName := DS.FieldByName('Name').AsString;
    DS.Delete;
    WriteLn('ลบ "', DeletedName, '" สำเร็จ');
    WriteLn('RecordCount หลัง Delete: ', DS.RecordCount);
    
    // ลบด้วยเงื่อนไข
    WriteLn(#10'6. Delete ด้วยเงื่อนไข:');
    DS.First;
    while not DS.EOF do
    begin
      if DS.FieldByName('Score').AsFloat < 80 then
      begin
        WriteLn('ลบ: ', DS.FieldByName('Name').AsString,
                ' (Score: ', DS.FieldByName('Score').AsFloat:0:1, ')');
        DS.Delete;
        // ไม่ต้อง Next เพราะ Delete จะไปที่ record ถัดไปอัตโนมัติ
      end
      else
        DS.Next;
    end;
    
    WriteLn('RecordCount หลัง Delete ตามเงื่อนไข: ', DS.RecordCount);
    
    // แสดงข้อมูลที่เหลือ
    WriteLn(#10'ข้อมูลที่เหลือ:');
    DS.First;
    while not DS.EOF do
    begin
      WriteLn(Format('  ID: %d, Name: %s, Score: %.1f',
                     [DS.FieldByName('ID').AsInteger,
                      DS.FieldByName('Name').AsString,
                      DS.FieldByName('Score').AsFloat]));
      DS.Next;
    end;
    
    DS.Close;
  finally
    DS.Free;
  end;
end;

begin
  DemoEditing;
  ReadLn;
end.
```

---

## 29.10 Filtering and Searching

```pascal
program FilteringSearching;
{$mode objfpc}{$H+}

uses
  SysUtils, DB;

procedure CreateTestData(DS: TBufDataset);
type
  TData = record
    ID: Integer;
    Name: String;
    Dept: String;
    Salary: Double;
    Age: Integer;
  end;
const
  Data: array[0..6] of TData = (
    (ID: 1; Name: 'สมชาย'; Dept: 'IT';      Salary: 45000; Age: 30),
    (ID: 2; Name: 'สมหญิง'; Dept: 'HR';      Salary: 35000; Age: 25),
    (ID: 3; Name: 'วิชัย';  Dept: 'IT';      Salary: 55000; Age: 35),
    (ID: 4; Name: 'อนันต์'; Dept: 'Finance'; Salary: 40000; Age: 28),
    (ID: 5; Name: 'บุญมี';  Dept: 'HR';      Salary: 32000; Age: 23),
    (ID: 6; Name: 'ประยุทธ์'; Dept: 'IT';   Salary: 60000; Age: 40),
    (ID: 7; Name: 'สุวรรณ'; Dept: 'Finance'; Salary: 48000; Age: 33)
  );
var
  I: Integer;
begin
  for I := 0 to High(Data) do
  begin
    DS.Append;
    DS.FieldByName('ID').AsInteger := Data[I].ID;
    DS.FieldByName('Name').AsString := Data[I].Name;
    DS.FieldByName('Dept').AsString := Data[I].Dept;
    DS.FieldByName('Salary').AsFloat := Data[I].Salary;
    DS.FieldByName('Age').AsInteger := Data[I].Age;
    DS.Post;
  end;
end;

procedure ShowAll(DS: TDataSet; const Title: String);
begin
  WriteLn(Title, ':');
  DS.First;
  while not DS.EOF do
  begin
    WriteLn(Format('  %-12s | %-10s | ฿%8.0f | อายุ %d',
                   [DS.FieldByName('Name').AsString,
                    DS.FieldByName('Dept').AsString,
                    DS.FieldByName('Salary').AsFloat,
                    DS.FieldByName('Age').AsInteger]));
    DS.Next;
  end;
  WriteLn('  (', DS.RecordCount, ' records)');
end;

procedure DemoFilter;
var
  DS: TBufDataset;
begin
  WriteLn('=== Filtering ===');
  
  DS := TBufDataset.Create(nil);
  try
    DS.FieldDefs.Add('ID', ftInteger);
    DS.FieldDefs.Add('Name', ftString, 50);
    DS.FieldDefs.Add('Dept', ftString, 30);
    DS.FieldDefs.Add('Salary', ftFloat);
    DS.FieldDefs.Add('Age', ftInteger);
    DS.CreateDataset;
    DS.Open;
    
    CreateTestData(DS);
    ShowAll(DS, 'ข้อมูลทั้งหมด');
    
    // Filter 1: กรองโดย Dept = 'IT'
    WriteLn(#10'Filter: Dept = "IT"');
    DS.Filter := 'Dept = "IT"';
    DS.Filtered := True;
    ShowAll(DS, 'ผลลัพธ์');
    
    // Filter 2: Salary > 40000
    WriteLn(#10'Filter: Salary > 40000');
    DS.Filter := 'Salary > 40000';
    ShowAll(DS, 'ผลลัพธ์');
    
    // Filter 3: รวม conditions
    WriteLn(#10'Filter: Dept = "IT" AND Salary > 50000');
    DS.Filter := 'Dept = "IT" AND Salary > 50000';
    ShowAll(DS, 'ผลลัพธ์');
    
    // ปิด Filter
    WriteLn(#10'ปิด Filter:');
    DS.Filtered := False;
    ShowAll(DS, 'ข้อมูลทั้งหมด');
    
    DS.Close;
  finally
    DS.Free;
  end;
end;

procedure DemoLocate;
var
  DS: TBufDataset;
begin
  WriteLn(#10'=== Locate (ค้นหา) ===');
  
  DS := TBufDataset.Create(nil);
  try
    DS.FieldDefs.Add('ID', ftInteger);
    DS.FieldDefs.Add('Name', ftString, 50);
    DS.FieldDefs.Add('Dept', ftString, 30);
    DS.FieldDefs.Add('Salary', ftFloat);
    DS.FieldDefs.Add('Age', ftInteger);
    DS.CreateDataset;
    DS.Open;
    
    CreateTestData(DS);
    
    // Locate ด้วย single field
    WriteLn('1. Locate Name = "วิชัย":');
    if DS.Locate('Name', 'วิชัย', []) then
      WriteLn('  พบที่ record: ', DS.FieldByName('ID').AsInteger,
              ' - ', DS.FieldByName('Dept').AsString)
    else
      WriteLn('  ไม่พบ');
    
    // Locate ด้วย partial match
    WriteLn(#10'2. Locate Name = "สม" (partial):');
    if DS.Locate('Name', 'สม', [loPartialKey]) then
    begin
      WriteLn('  พบ: ', DS.FieldByName('Name').AsString);
      // ค้นหาต่อ (ต้องใช้ FindNext หรือ loop)
    end;
    
    // Locate ด้วยหลาย fields
    WriteLn(#10'3. Locate Dept="HR" AND Age=25:');
    var LocateValues: Variant;
    LocateValues := VarArrayOf(['HR', 25]);
    if DS.Locate('Dept;Age', LocateValues, []) then
      WriteLn('  พบ: ', DS.FieldByName('Name').AsString)
    else
      WriteLn('  ไม่พบ');
    
    // หลังจาก Locate cursor จะอยู่ที่ record ที่พบ
    WriteLn(#10'หลัง Locate, อยู่ที่: ', DS.FieldByName('Name').AsString);
    
    DS.Close;
  finally
    DS.Free;
  end;
end;

begin
  DemoFilter;
  DemoLocate;
  ReadLn;
end.
```

---

## 29.11 Dataset Events

```pascal
program DataSetEvents;
{$mode objfpc}{$H+}

uses
  SysUtils, DB;

type
  TDataSetEventDemo = class
  private
    FDS: TBufDataset;
    
    procedure OnBeforeInsert(DataSet: TDataSet);
    procedure OnAfterInsert(DataSet: TDataSet);
    procedure OnBeforeEdit(DataSet: TDataSet);
    procedure OnAfterEdit(DataSet: TDataSet);
    procedure OnBeforePost(DataSet: TDataSet);
    procedure OnAfterPost(DataSet: TDataSet);
    procedure OnBeforeDelete(DataSet: TDataSet);
    procedure OnAfterDelete(DataSet: TDataSet);
    procedure OnBeforeScroll(DataSet: TDataSet);
    procedure OnAfterScroll(DataSet: TDataSet);
    procedure OnCalcFields(DataSet: TDataSet);
    procedure OnFilterRecord(DataSet: TDataSet; var Accept: Boolean);
  public
    constructor Create;
    destructor Destroy; override;
    procedure RunDemo;
  end;

constructor TDataSetEventDemo.Create;
begin
  inherited Create;
  
  FDS := TBufDataset.Create(nil);
  FDS.FieldDefs.Add('ID', ftInteger);
  FDS.FieldDefs.Add('Name', ftString, 50);
  FDS.FieldDefs.Add('Score', ftFloat);
  FDS.FieldDefs.Add('Grade', ftString, 2);  // Calculated field
  
  // Assign events
  FDS.BeforeInsert := @OnBeforeInsert;
  FDS.AfterInsert := @OnAfterInsert;
  FDS.BeforeEdit := @OnBeforeEdit;
  FDS.AfterEdit := @OnAfterEdit;
  FDS.BeforePost := @OnBeforePost;
  FDS.AfterPost := @OnAfterPost;
  FDS.BeforeDelete := @OnBeforeDelete;
  FDS.AfterDelete := @OnAfterDelete;
  FDS.BeforeScroll := @OnBeforeScroll;
  FDS.AfterScroll := @OnAfterScroll;
  FDS.OnCalcFields := @OnCalcFields;
  FDS.OnFilterRecord := @OnFilterRecord;
  
  FDS.CreateDataset;
  FDS.Open;
end;

destructor TDataSetEventDemo.Destroy;
begin
  FDS.Free;
  inherited Destroy;
end;

procedure TDataSetEventDemo.OnBeforeInsert(DataSet: TDataSet);
begin
  WriteLn('  [Event] BeforeInsert');
end;

procedure TDataSetEventDemo.OnAfterInsert(DataSet: TDataSet);
begin
  WriteLn('  [Event] AfterInsert');
end;

procedure TDataSetEventDemo.OnBeforeEdit(DataSet: TDataSet);
begin
  WriteLn('  [Event] BeforeEdit: ', DataSet.FieldByName('Name').AsString);
end;

procedure TDataSetEventDemo.OnAfterEdit(DataSet: TDataSet);
begin
  WriteLn('  [Event] AfterEdit');
end;

procedure TDataSetEventDemo.OnBeforePost(DataSet: TDataSet);
begin
  // ตรวจสอบข้อมูลก่อน post
  var Score := DataSet.FieldByName('Score').AsFloat;
  if (Score < 0) or (Score > 100) then
    raise Exception.CreateFmt('คะแนน %.1f ไม่ถูกต้อง (0-100)', [Score]);
  WriteLn('  [Event] BeforePost: Score = ', Score:0:1);
end;

procedure TDataSetEventDemo.OnAfterPost(DataSet: TDataSet);
begin
  WriteLn('  [Event] AfterPost');
end;

procedure TDataSetEventDemo.OnBeforeDelete(DataSet: TDataSet);
begin
  WriteLn('  [Event] BeforeDelete: ', DataSet.FieldByName('Name').AsString);
end;

procedure TDataSetEventDemo.OnAfterDelete(DataSet: TDataSet);
begin
  WriteLn('  [Event] AfterDelete');
end;

procedure TDataSetEventDemo.OnBeforeScroll(DataSet: TDataSet);
begin
  // WriteLn('  [Event] BeforeScroll');  // commented out - too verbose
end;

procedure TDataSetEventDemo.OnAfterScroll(DataSet: TDataSet);
begin
  // WriteLn('  [Event] AfterScroll');  // commented out - too verbose
end;

procedure TDataSetEventDemo.OnCalcFields(DataSet: TDataSet);
var
  Score: Double;
begin
  // คำนวณ Grade จาก Score
  Score := DataSet.FieldByName('Score').AsFloat;
  if Score >= 80 then
    DataSet.FieldByName('Grade').AsString := 'A'
  else if Score >= 70 then
    DataSet.FieldByName('Grade').AsString := 'B'
  else if Score >= 60 then
    DataSet.FieldByName('Grade').AsString := 'C'
  else
    DataSet.FieldByName('Grade').AsString := 'F';
end;

procedure TDataSetEventDemo.OnFilterRecord(DataSet: TDataSet; var Accept: Boolean);
begin
  // กรองเฉพาะคะแนน >= 70
  Accept := DataSet.FieldByName('Score').AsFloat >= 70;
end;

procedure TDataSetEventDemo.RunDemo;
begin
  WriteLn('=== Dataset Events Demo ===');
  
  // Insert
  WriteLn(#10'1. Insert events:');
  FDS.Insert;
  FDS.FieldByName('ID').AsInteger := 1;
  FDS.FieldByName('Name').AsString := 'สมชาย';
  FDS.FieldByName('Score').AsFloat := 85.0;
  FDS.Post;
  
  FDS.Insert;
  FDS.FieldByName('ID').AsInteger := 2;
  FDS.FieldByName('Name').AsString := 'สมหญิง';
  FDS.FieldByName('Score').AsFloat := 92.0;
  FDS.Post;
  
  FDS.Insert;
  FDS.FieldByName('ID').AsInteger := 3;
  FDS.FieldByName('Name').AsString := 'วิชัย';
  FDS.FieldByName('Score').AsFloat := 65.0;
  FDS.Post;
  
  // Calc Fields
  WriteLn(#10'2. Calc Fields (Grade):');
  FDS.First;
  while not FDS.EOF do
  begin
    WriteLn(Format('  %s: %.0f -> %s',
                   [FDS.FieldByName('Name').AsString,
                    FDS.FieldByName('Score').AsFloat,
                    FDS.FieldByName('Grade').AsString]));
    FDS.Next;
  end;
  
  // Edit events
  WriteLn(#10'3. Edit events:');
  FDS.First;
  FDS.Edit;
  FDS.FieldByName('Score').AsFloat := 88.0;
  FDS.Post;
  
  // Validation ใน BeforePost
  WriteLn(#10'4. Validation ใน BeforePost:');
  try
    FDS.Insert;
    FDS.FieldByName('ID').AsInteger := 4;
    FDS.FieldByName('Name').AsString := 'ทดสอบ';
    FDS.FieldByName('Score').AsFloat := 150.0;  // ผิดพลาด!
    FDS.Post;
  except
    on E: Exception do
    begin
      WriteLn('  Validation ล้มเหลว: ', E.Message);
      FDS.Cancel;
    end;
  end;
  
  // Delete events
  WriteLn(#10'5. Delete events:');
  FDS.Last;
  FDS.Delete;
  
  WriteLn(#10'สุดท้าย Count: ', FDS.RecordCount);
end;

begin
  var Demo := TDataSetEventDemo.Create;
  try
    Demo.RunDemo;
  finally
    Demo.Free;
  end;
  ReadLn;
end.
```

---

## 29.12 DB-Aware Controls Overview

```pascal
// หมายเหตุ: ส่วนนี้เป็น GUI controls
// ต้องใช้ใน Lazarus Form Application
// ตัวอย่างนี้แสดงการตั้งค่าผ่านโค้ด

program DBControlsOverview;
{$mode objfpc}{$H+}

// ไม่ต้องคอมไพล์จริง - เป็นแค่ pseudocode เพื่อแสดงแนวคิด

uses
  SysUtils;

begin
  WriteLn('=== DB-Aware Controls Overview ===');
  WriteLn;
  WriteLn('DB-Aware Controls เชื่อม DataSource กับ UI โดยอัตโนมัติ');
  WriteLn;
  WriteLn('Controls หลัก (จาก unit DBCtrls):');
  WriteLn;
  WriteLn('1. TDBEdit');
  WriteLn('   - แสดง/แก้ไข field เดียว');
  WriteLn('   - Properties: DataSource, DataField');
  WriteLn;
  WriteLn('2. TDBLabel');
  WriteLn('   - แสดงค่า field (read-only)');
  WriteLn('   - Properties: DataSource, DataField');
  WriteLn;
  WriteLn('3. TDBMemo');
  WriteLn('   - แสดง/แก้ไข text ยาว (Memo fields)');
  WriteLn;
  WriteLn('4. TDBCheckBox');
  WriteLn('   - สำหรับ Boolean fields');
  WriteLn('   - Properties: ValueChecked, ValueUnchecked');
  WriteLn;
  WriteLn('5. TDBComboBox');
  WriteLn('   - เลือกค่าจาก dropdown');
  WriteLn('   - สามารถกำหนด Items หรือ link กับ dataset อื่น');
  WriteLn;
  WriteLn('6. TDBListBox');
  WriteLn('   - เลือกค่าจาก list');
  WriteLn;
  WriteLn('7. TDBImage');
  WriteLn('   - แสดงรูปจาก Blob field');
  WriteLn;
  WriteLn('8. TDBGrid (จาก unit DBGrids)');
  WriteLn('   - แสดงข้อมูลแบบตาราง');
  WriteLn('   - Properties: DataSource, Columns');
  WriteLn('   - สามารถแก้ไขใน grid ได้');
  WriteLn;
  WriteLn('9. TDBNavigator (จาก unit DBCtrls)');
  WriteLn('   - ปุ่ม Navigate: First, Prior, Next, Last');
  WriteLn('   - ปุ่ม Edit: Insert, Edit, Delete, Post, Cancel');
  WriteLn('   - Properties: DataSource, VisibleButtons');
  WriteLn;
  WriteLn('การตั้งค่าใน Form:');
  WriteLn('1. วาง TSQLite3Connection + TSQLTransaction + TSQLQuery');
  WriteLn('2. ตั้งค่า Connection');
  WriteLn('3. วาง TDataSource และตั้งค่า DataSet = TSQLQuery');
  WriteLn('4. วาง DB Controls และตั้งค่า DataSource + DataField');
  
  ReadLn;
end.
```

---

## 29.13 โปรแกรมตัวอย่าง: Simple Data Browser

```pascal
program SimpleDataBrowser;
{$mode objfpc}{$H+}

uses
  SysUtils, DB;

type
  TDataBrowser = class
  private
    FDS: TBufDataset;
    
    procedure PrintHeader;
    procedure PrintCurrentRecord;
    procedure PrintSeparator;
    function GetInput(const Prompt: String): String;
  public
    constructor Create;
    destructor Destroy; override;
    procedure LoadSampleData;
    procedure Run;
    procedure ShowAll;
    procedure NavigateTo(N: Integer);
    procedure SearchByName(const Name: String);
    procedure AddRecord(const Name, Dept: String; Salary: Double);
    procedure EditCurrentSalary(NewSalary: Double);
    procedure DeleteCurrent;
  end;

constructor TDataBrowser.Create;
begin
  inherited Create;
  
  FDS := TBufDataset.Create(nil);
  FDS.FieldDefs.Add('ID', ftInteger);
  FDS.FieldDefs.Add('Name', ftString, 50);
  FDS.FieldDefs.Add('Department', ftString, 30);
  FDS.FieldDefs.Add('Salary', ftFloat);
  FDS.FieldDefs.Add('HireDate', ftDate);
  FDS.CreateDataset;
  FDS.Open;
end;

destructor TDataBrowser.Destroy;
begin
  FDS.Free;
  inherited Destroy;
end;

procedure TDataBrowser.PrintHeader;
begin
  PrintSeparator;
  WriteLn(Format('%-5s | %-15s | %-12s | %10s | %12s',
                 ['ID', 'ชื่อ', 'แผนก', 'เงินเดือน', 'วันที่เข้า']));
  PrintSeparator;
end;

procedure TDataBrowser.PrintCurrentRecord;
begin
  if FDS.EOF and FDS.BOF then
  begin
    WriteLn('ไม่มีข้อมูล');
    Exit;
  end;
  WriteLn(Format('%-5d | %-15s | %-12s | %10.2f | %12s',
                 [FDS.FieldByName('ID').AsInteger,
                  FDS.FieldByName('Name').AsString,
                  FDS.FieldByName('Department').AsString,
                  FDS.FieldByName('Salary').AsFloat,
                  FormatDateTime('dd/mm/yyyy', FDS.FieldByName('HireDate').AsDateTime)]));
end;

procedure TDataBrowser.PrintSeparator;
begin
  WriteLn(StringOfChar('-', 65));
end;

function TDataBrowser.GetInput(const Prompt: String): String;
begin
  Write(Prompt);
  ReadLn(Result);
end;

procedure TDataBrowser.LoadSampleData;
type
  TEmployee = record
    ID: Integer;
    Name: String[50];
    Dept: String[30];
    Salary: Double;
    HireDate: String[12];
  end;
const
  Employees: array[0..6] of TEmployee = (
    (ID: 1; Name: 'สมชาย ใจดี'; Dept: 'IT'; Salary: 45000; HireDate: '01/01/2020'),
    (ID: 2; Name: 'สมหญิง งาม'; Dept: 'HR'; Salary: 35000; HireDate: '15/03/2019'),
    (ID: 3; Name: 'วิชัย เก่ง'; Dept: 'IT'; Salary: 55000; HireDate: '10/06/2018'),
    (ID: 4; Name: 'อนันต์ สุข'; Dept: 'Finance'; Salary: 40000; HireDate: '20/09/2021'),
    (ID: 5; Name: 'บุญมี ดี'; Dept: 'HR'; Salary: 32000; HireDate: '05/12/2022'),
    (ID: 6; Name: 'ประยุทธ์ แกร่ง'; Dept: 'IT'; Salary: 60000; HireDate: '01/01/2015'),
    (ID: 7; Name: 'สุวรรณ ทอง'; Dept: 'Finance'; Salary: 48000; HireDate: '14/02/2020')
  );
var
  I: Integer;
begin
  for I := 0 to High(Employees) do
  begin
    FDS.Append;
    FDS.FieldByName('ID').AsInteger := Employees[I].ID;
    FDS.FieldByName('Name').AsString := Employees[I].Name;
    FDS.FieldByName('Department').AsString := Employees[I].Dept;
    FDS.FieldByName('Salary').AsFloat := Employees[I].Salary;
    FDS.FieldByName('HireDate').AsDateTime := StrToDate(Employees[I].HireDate);
    FDS.Post;
  end;
  WriteLn('โหลดข้อมูลตัวอย่าง ', FDS.RecordCount, ' รายการ');
end;

procedure TDataBrowser.ShowAll;
begin
  WriteLn(#10'ข้อมูลทั้งหมด (', FDS.RecordCount, ' รายการ):');
  PrintHeader;
  FDS.First;
  while not FDS.EOF do
  begin
    PrintCurrentRecord;
    FDS.Next;
  end;
  PrintSeparator;
end;

procedure TDataBrowser.NavigateTo(N: Integer);
begin
  if (N < 1) or (N > FDS.RecordCount) then
  begin
    WriteLn('หมายเลข record ไม่ถูกต้อง (1-', FDS.RecordCount, ')');
    Exit;
  end;
  FDS.First;
  FDS.MoveBy(N - 1);
  WriteLn('Record ที่ ', N, ':');
  PrintHeader;
  PrintCurrentRecord;
  PrintSeparator;
end;

procedure TDataBrowser.SearchByName(const Name: String);
var
  Found: Boolean;
begin
  WriteLn('ค้นหา "', Name, '":');
  PrintHeader;
  Found := False;
  FDS.First;
  while not FDS.EOF do
  begin
    if Pos(LowerCase(Name), LowerCase(FDS.FieldByName('Name').AsString)) > 0 then
    begin
      PrintCurrentRecord;
      Found := True;
    end;
    FDS.Next;
  end;
  PrintSeparator;
  if not Found then
    WriteLn('ไม่พบข้อมูลที่ตรงกับ "', Name, '"');
end;

procedure TDataBrowser.AddRecord(const Name, Dept: String; Salary: Double);
begin
  var NewID := FDS.RecordCount + 1;
  FDS.Append;
  FDS.FieldByName('ID').AsInteger := NewID;
  FDS.FieldByName('Name').AsString := Name;
  FDS.FieldByName('Department').AsString := Dept;
  FDS.FieldByName('Salary').AsFloat := Salary;
  FDS.FieldByName('HireDate').AsDateTime := Date;
  FDS.Post;
  WriteLn('เพิ่มข้อมูลสำเร็จ: ', Name, ' (ID: ', NewID, ')');
end;

procedure TDataBrowser.EditCurrentSalary(NewSalary: Double);
begin
  if FDS.EOF and FDS.BOF then
  begin
    WriteLn('ไม่มี record ปัจจุบัน');
    Exit;
  end;
  var OldSalary := FDS.FieldByName('Salary').AsFloat;
  FDS.Edit;
  FDS.FieldByName('Salary').AsFloat := NewSalary;
  FDS.Post;
  WriteLn(Format('แก้ไขเงินเดือน %s: ฿%.2f -> ฿%.2f',
                 [FDS.FieldByName('Name').AsString, OldSalary, NewSalary]));
end;

procedure TDataBrowser.DeleteCurrent;
begin
  if FDS.EOF and FDS.BOF then
  begin
    WriteLn('ไม่มี record ปัจจุบัน');
    Exit;
  end;
  var DeletedName := FDS.FieldByName('Name').AsString;
  FDS.Delete;
  WriteLn('ลบ "', DeletedName, '" สำเร็จ');
end;

procedure TDataBrowser.Run;
begin
  WriteLn('=== Simple Data Browser ===');
  
  LoadSampleData;
  ShowAll;
  
  // Navigation demo
  WriteLn(#10'--- Navigation Demo ---');
  NavigateTo(3);
  NavigateTo(1);
  NavigateTo(7);
  
  // Search demo
  WriteLn(#10'--- Search Demo ---');
  SearchByName('สม');
  SearchByName('IT');
  
  // Edit demo
  WriteLn(#10'--- Edit Demo ---');
  NavigateTo(1);
  FDS.First;
  EditCurrentSalary(50000);
  
  // Add record
  WriteLn(#10'--- Add Record Demo ---');
  AddRecord('นักเรียน ใหม่', 'IT', 25000);
  
  // Delete demo
  WriteLn(#10'--- Delete Demo ---');
  FDS.Last;
  DeleteCurrent;
  
  // Show final state
  ShowAll;
end;

begin
  var Browser := TDataBrowser.Create;
  try
    Browser.Run;
  finally
    Browser.Free;
  end;
  ReadLn;
end.
```

---

## 29.14 แบบฝึกหัด

### แบบฝึกหัดที่ 1: SQL Practice
เขียน SQL queries สำหรับ:
- ตาราง Products (ProductID, Name, Price, Stock, CategoryID)
- ตาราง Categories (CategoryID, Name)
- ตาราง Orders (OrderID, ProductID, Qty, Date)
คำถาม: หาสินค้าที่มีราคาสูงกว่าค่าเฉลี่ย, หา category ที่มีสินค้ามากที่สุด, คำนวณยอดขายรวมแต่ละ category

### แบบฝึกหัดที่ 2: TBufDataset CRUD
สร้างโปรแกรมที่จัดการข้อมูลนักเรียนด้วย TBufDataset:
- Add student
- Edit score
- Delete student
- Search by name
- Sort by score

### แบบฝึกหัดที่ 3: Dataset Events Validation
ใช้ BeforePost event เพื่อ validate:
- ชื่อต้องไม่ว่าง
- อายุต้องอยู่ระหว่าง 18-65
- เงินเดือนต้องมากกว่า minimum wage

### แบบฝึกหัดที่ 4: Filter Builder
สร้างฟังก์ชัน `BuildFilter` ที่รับ criteria และสร้าง filter string:
```pascal
BuildFilter(['Dept=IT', 'Salary>40000']) -> 'Dept = "IT" AND Salary > 40000'
```

### แบบฝึกหัดที่ 5: Dataset to StringList
เขียนฟังก์ชันที่แปลง Dataset เป็น CSV format ด้วย TStringList

### แบบฝึกหัดที่ 6: Aggregate Calculator
คำนวณ aggregate ด้วยการ traverse dataset:
- Count records
- Sum field
- Average field
- Min/Max field

### แบบฝึกหัดที่ 7: Multi-Level Sort
สร้าง sort ที่รองรับหลาย criteria:
```pascal
SortDataSet(DS, ['Dept ASC', 'Salary DESC', 'Name ASC']);
```

### แบบฝึกหัดที่ 8: Dataset Diff
เขียนฟังก์ชันที่เปรียบเทียบ 2 datasets และหาความแตกต่าง (added, modified, deleted records)

### แบบฝึกหัดที่ 9: Lookup Field
จำลอง lookup field: เมื่อแสดง CategoryID ให้แสดงชื่อ Category แทน

### แบบฝึกหัดที่ 10: Dataset Report
สร้าง text report จาก dataset พร้อม:
- Header
- Column headers
- Data rows (formatted)
- Footer (totals, count)
- Page breaks

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **แนวคิดฐานข้อมูล** - Tables, Fields, Records, Keys, Relationships
2. **SQL พื้นฐาน** - CREATE, SELECT, INSERT, UPDATE, DELETE
3. **สถาปัตยกรรม Lazarus** - Connection -> Dataset -> DataSource -> Controls
4. **TDataSource** - เชื่อม dataset กับ DB controls
5. **TDataSet** - base class สำหรับทุก dataset
6. **TField** - จัดการ fields ใน dataset
7. **Navigation** - First, Last, Next, Prior, MoveBy, Bookmark
8. **States** - dsBrowse, dsEdit, dsInsert, dsInactive
9. **Editing** - Insert, Append, Edit, Post, Cancel, Delete
10. **Filtering** - Filter property, OnFilterRecord
11. **Events** - BeforeInsert, AfterPost, OnCalcFields
12. **DB Controls** - DBEdit, DBGrid, DBNavigator

### หลักสำคัญ:
- เสมอตรวจสอบ DS.Active ก่อนทำงาน
- ใช้ try...finally เพื่อ ensure DS.Cancel หรือ DS.Post
- BeforePost เหมาะสำหรับ validation
- OnCalcFields สำหรับ calculated fields
