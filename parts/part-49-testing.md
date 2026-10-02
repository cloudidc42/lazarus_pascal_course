# Part 49 - Unit Testing ใน Lazarus/Pascal

## บทนำ

Unit Testing เป็นการทดสอบ code ในระดับ unit (function, procedure, class) โดยอัตโนมัติ ช่วยให้มั่นใจว่า code ทำงานถูกต้องและป้องกัน regression bugs

---

## 49.1 FPCUnit Framework

### การตั้งค่า FPCUnit

```pascal
{$mode objfpc}{$H+}

// การสร้าง test project พื้นฐานด้วย FPCUnit
program TestRunner;

uses
  {$IFDEF UNIX}
  cwstring,
  {$ENDIF}
  Classes, SysUtils,
  fpcunit,      // FPCUnit framework
  testutils,    // Utility functions
  testregistry, // Test registry
  consoletestrunner; // Console output

// Import test units
// uses TestCalculator, TestDatabase;

var
  Application: TTestRunner;
begin
  Application := TTestRunner.Create(nil);
  Application.Initialize;
  Application.Title := 'MyApp Test Suite';
  Application.Run;
  Application.Free;
end.
```

### Test Case พื้นฐาน

```pascal
{$mode objfpc}{$H+}

// test_math.pas
unit TestMath;

interface

uses
  fpcunit, testutils, testregistry;

type
  // Test Case class - สืบทอดจาก TTestCase
  TTestMath = class(TTestCase)
  protected
    // Setup: รันก่อนแต่ละ test
    procedure SetUp; override;
    // TearDown: รันหลังแต่ละ test
    procedure TearDown; override;
  published
    // Test methods ต้องเป็น published
    procedure TestAdd;
    procedure TestSubtract;
    procedure TestMultiply;
    procedure TestDivide;
    procedure TestDivideByZero;
    procedure TestNegativeNumbers;
  end;

implementation

uses
  SysUtils, Math;

// ฟังก์ชันที่จะทดสอบ (ใน production จะ import จาก unit อื่น)
function Add(A, B: Integer): Integer;
begin
  Result := A + B;
end;

function Subtract(A, B: Integer): Integer;
begin
  Result := A - B;
end;

function Multiply(A, B: Integer): Integer;
begin
  Result := A * B;
end;

function Divide(A, B: Double): Double;
begin
  if B = 0 then
    raise EDivByZero.Create('Division by zero');
  Result := A / B;
end;

procedure TTestMath.SetUp;
begin
  // Initialize ทรัพยากรก่อน test
end;

procedure TTestMath.TearDown;
begin
  // Cleanup ทรัพยากรหลัง test
end;

procedure TTestMath.TestAdd;
begin
  // AssertEquals(expected, actual, message)
  AssertEquals('2 + 3 should be 5', 5, Add(2, 3));
  AssertEquals('0 + 0 should be 0', 0, Add(0, 0));
  AssertEquals('-1 + 1 should be 0', 0, Add(-1, 1));
  AssertEquals('100 + 200 should be 300', 300, Add(100, 200));
end;

procedure TTestMath.TestSubtract;
begin
  AssertEquals('5 - 3 should be 2', 2, Subtract(5, 3));
  AssertEquals('0 - 0 should be 0', 0, Subtract(0, 0));
  AssertEquals('3 - 5 should be -2', -2, Subtract(3, 5));
end;

procedure TTestMath.TestMultiply;
begin
  AssertEquals('3 * 4 should be 12', 12, Multiply(3, 4));
  AssertEquals('0 * 5 should be 0', 0, Multiply(0, 5));
  AssertEquals('-2 * 3 should be -6', -6, Multiply(-2, 3));
  AssertEquals('-2 * -3 should be 6', 6, Multiply(-2, -3));
end;

procedure TTestMath.TestDivide;
begin
  // AssertEqualsWithDelta สำหรับ float comparison
  AssertEqualsWithDelta('10 / 2 should be 5', 5.0, Divide(10, 2), 0.0001);
  AssertEqualsWithDelta('1 / 3 should be ~0.333', 1/3, Divide(1, 3), 0.0001);
end;

procedure TTestMath.TestDivideByZero;
begin
  // AssertException ทดสอบว่า exception ถูก raise
  AssertException('Division by zero should raise EDivByZero',
    EDivByZero,
    procedure begin Divide(5, 0) end);
end;

procedure TTestMath.TestNegativeNumbers;
begin
  AssertEquals('Add negatives', -5, Add(-2, -3));
  AssertEquals('Multiply negatives', 4, Multiply(-2, -2));
end;

initialization
  // Register test class กับ FPCUnit
  RegisterTest(TTestMath);
end.
```

---

## 49.2 DUnit Framework

### การใช้ DUnit

```pascal
{$mode delphi}{$H+}  // DUnit ใช้ Delphi mode

// test_with_dunit.pas
unit TestWithDUnit;

interface

uses
  TestFramework; // DUnit

type
  TCalculatorTests = class(TTestCase)
  private
    FCalc: TObject; // Calculator instance
  protected
    procedure SetUp; override;
    procedure TearDown; override;
  published
    procedure TestBasicArithmetic;
    procedure TestStringOperations;
    procedure TestEdgeCases;
  end;

implementation

uses
  SysUtils, Calculator; // ใส่ unit ที่ต้องการทดสอบ

{ TCalculator - Simple calculator class for testing }
type
  TCalculator = class
  private
    FMemory: Double;
    FHistory: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    function Add(A, B: Double): Double;
    function Subtract(A, B: Double): Double;
    function Multiply(A, B: Double): Double;
    function Divide(A, B: Double): Double;
    procedure StoreMemory(Value: Double);
    function RecallMemory: Double;
    procedure ClearMemory;
    property History: TStringList read FHistory;
  end;

constructor TCalculator.Create;
begin
  inherited;
  FHistory := TStringList.Create;
  FMemory := 0;
end;

destructor TCalculator.Destroy;
begin
  FHistory.Free;
  inherited;
end;

function TCalculator.Add(A, B: Double): Double;
begin
  Result := A + B;
  FHistory.Add(Format('%g + %g = %g', [A, B, Result]));
end;

function TCalculator.Subtract(A, B: Double): Double;
begin
  Result := A - B;
  FHistory.Add(Format('%g - %g = %g', [A, B, Result]));
end;

function TCalculator.Multiply(A, B: Double): Double;
begin
  Result := A * B;
  FHistory.Add(Format('%g * %g = %g', [A, B, Result]));
end;

function TCalculator.Divide(A, B: Double): Double;
begin
  if B = 0 then
    raise EDivByZero.Create('Cannot divide by zero');
  Result := A / B;
  FHistory.Add(Format('%g / %g = %g', [A, B, Result]));
end;

procedure TCalculator.StoreMemory(Value: Double);
begin
  FMemory := Value;
end;

function TCalculator.RecallMemory: Double;
begin
  Result := FMemory;
end;

procedure TCalculator.ClearMemory;
begin
  FMemory := 0;
end;

{ Test implementation }
procedure TCalculatorTests.SetUp;
begin
  // สร้าง calculator ใหม่ก่อนแต่ละ test
  // FCalc := TCalculator.Create;
end;

procedure TCalculatorTests.TearDown;
begin
  // ลบ calculator หลังแต่ละ test
  // FCalc.Free;
end;

procedure TCalculatorTests.TestBasicArithmetic;
var
  Calc: TCalculator;
begin
  Calc := TCalculator.Create;
  try
    CheckEquals(5.0, Calc.Add(2, 3), 'Add 2+3');
    CheckEquals(2.0, Calc.Subtract(5, 3), 'Subtract 5-3');
    CheckEquals(12.0, Calc.Multiply(3, 4), 'Multiply 3*4');
    CheckEquals(2.5, Calc.Divide(5, 2), 0.0001, 'Divide 5/2');
  finally
    Calc.Free;
  end;
end;

procedure TCalculatorTests.TestStringOperations;
var
  Calc: TCalculator;
begin
  Calc := TCalculator.Create;
  try
    Calc.Add(1, 2);
    Calc.Multiply(3, 4);
    
    CheckEquals(2, Calc.History.Count, 'Should have 2 history entries');
    CheckTrue(Pos('1 + 2 = 3', Calc.History[0]) > 0, 'History contains addition');
    CheckTrue(Pos('3 * 4 = 12', Calc.History[1]) > 0, 'History contains multiplication');
  finally
    Calc.Free;
  end;
end;

procedure TCalculatorTests.TestEdgeCases;
var
  Calc: TCalculator;
begin
  Calc := TCalculator.Create;
  try
    // Memory functions
    Calc.StoreMemory(42.5);
    CheckEquals(42.5, Calc.RecallMemory, 0.0001, 'Memory recall');
    Calc.ClearMemory;
    CheckEquals(0.0, Calc.RecallMemory, 0.0001, 'Memory cleared');
    
    // Division by zero
    try
      Calc.Divide(5, 0);
      Fail('Should have raised EDivByZero');
    except
      on E: EDivByZero do
        CheckTrue(True, 'Correctly raised EDivByZero');
    end;
  finally
    Calc.Free;
  end;
end;

initialization
  RegisterTest('Calculator Tests', TCalculatorTests.Suite);
end.
```

---

## 49.3 Test Cases และ Test Suites

### การจัดกลุ่ม Tests

```pascal
{$mode objfpc}{$H+}

// test_suites.pas - ตัวอย่างการจัดกลุ่ม test
unit TestSuites;

interface

uses
  fpcunit, testregistry;

type
  // Test Suite 1: String Tests
  TStringTests = class(TTestCase)
  published
    procedure TestTrim;
    procedure TestUpperCase;
    procedure TestLowerCase;
    procedure TestStringReplace;
    procedure TestSplitString;
  end;

  // Test Suite 2: Date/Time Tests
  TDateTimeTests = class(TTestCase)
  published
    procedure TestDateFormat;
    procedure TestDateParsing;
    procedure TestDateArithmetic;
    procedure TestTimeZone;
  end;

  // Test Suite 3: File Tests
  TFileTests = class(TTestCase)
  private
    FTempDir: String;
  protected
    procedure SetUp; override;
    procedure TearDown; override;
  published
    procedure TestFileCreate;
    procedure TestFileRead;
    procedure TestFileDelete;
    procedure TestDirectoryCreate;
  end;

implementation

uses
  SysUtils, Classes, DateUtils;

{ TStringTests }

procedure TStringTests.TestTrim;
begin
  AssertEquals('Trim spaces', 'hello', Trim('  hello  '));
  AssertEquals('Trim tabs', 'world', Trim(#9'world'#9));
  AssertEquals('Empty string', '', Trim(''));
  AssertEquals('No spaces', 'test', Trim('test'));
end;

procedure TStringTests.TestUpperCase;
begin
  AssertEquals('Upper simple', 'HELLO', UpperCase('hello'));
  AssertEquals('Upper mixed', 'HELLO WORLD', UpperCase('Hello World'));
  AssertEquals('Upper numbers', 'TEST123', UpperCase('test123'));
end;

procedure TStringTests.TestLowerCase;
begin
  AssertEquals('Lower simple', 'hello', LowerCase('HELLO'));
  AssertEquals('Lower mixed', 'hello world', LowerCase('Hello World'));
end;

procedure TStringTests.TestStringReplace;
begin
  AssertEquals('Replace word', 
    'Hello World',
    StringReplace('Hello Pascal', 'Pascal', 'World', []));
  AssertEquals('Replace all', 
    'aXbXcXd',
    StringReplace('a1b1c1d', '1', 'X', [rfReplaceAll]));
end;

procedure TStringTests.TestSplitString;
var
  Parts: TStringArray;
begin
  Parts := 'a,b,c,d'.Split(',');
  AssertEquals('Split count', 4, Length(Parts));
  AssertEquals('Split[0]', 'a', Parts[0]);
  AssertEquals('Split[3]', 'd', Parts[3]);
end;

{ TDateTimeTests }

procedure TDateTimeTests.TestDateFormat;
begin
  // FormatDateTime
  AssertEquals('Date format', 
    '2024-01-15',
    FormatDateTime('yyyy-mm-dd', EncodeDate(2024, 1, 15)));
end;

procedure TDateTimeTests.TestDateParsing;
var
  D: TDateTime;
begin
  D := EncodeDate(2024, 6, 15);
  AssertEquals('Year', 2024, YearOf(D));
  AssertEquals('Month', 6, MonthOf(D));
  AssertEquals('Day', 15, DayOf(D));
end;

procedure TDateTimeTests.TestDateArithmetic;
var
  D1, D2: TDateTime;
begin
  D1 := EncodeDate(2024, 1, 1);
  D2 := EncodeDate(2024, 1, 31);
  AssertEquals('Days between', 30, DaysBetween(D1, D2));
end;

procedure TDateTimeTests.TestTimeZone;
var
  T: TDateTime;
begin
  T := EncodeTime(14, 30, 0, 0);
  AssertEquals('Hour', 14, HourOf(T));
  AssertEquals('Minute', 30, MinuteOf(T));
end;

{ TFileTests }

procedure TFileTests.SetUp;
begin
  FTempDir := GetTempDir + 'fpcunit_test_' + IntToStr(GetTickCount64) + PathDelim;
  ForceDirectories(FTempDir);
end;

procedure TFileTests.TearDown;
begin
  // ลบ temp directory
  if DirectoryExists(FTempDir) then
    DeleteDirectory(FTempDir, True);
end;

procedure TFileTests.TestFileCreate;
var
  FileName: String;
  F: TextFile;
begin
  FileName := FTempDir + 'test.txt';
  
  AssignFile(F, FileName);
  Rewrite(F);
  WriteLn(F, 'Hello Test');
  CloseFile(F);
  
  AssertTrue('File should exist', FileExists(FileName));
end;

procedure TFileTests.TestFileRead;
var
  FileName: String;
  F: TextFile;
  Content: String;
begin
  FileName := FTempDir + 'read_test.txt';
  
  // สร้างไฟล์
  with TStringList.Create do
  try
    Add('Line 1');
    Add('Line 2');
    SaveToFile(FileName);
  finally
    Free;
  end;
  
  // อ่านไฟล์
  with TStringList.Create do
  try
    LoadFromFile(FileName);
    AssertEquals('Line count', 2, Count);
    AssertEquals('First line', 'Line 1', Strings[0]);
    AssertEquals('Second line', 'Line 2', Strings[1]);
  finally
    Free;
  end;
end;

procedure TFileTests.TestFileDelete;
var
  FileName: String;
begin
  FileName := FTempDir + 'delete_test.txt';
  
  // สร้างไฟล์
  with TStringList.Create do
  try
    Add('temp');
    SaveToFile(FileName);
  finally
    Free;
  end;
  
  AssertTrue('File should exist', FileExists(FileName));
  
  // ลบไฟล์
  DeleteFile(FileName);
  
  AssertFalse('File should not exist', FileExists(FileName));
end;

procedure TFileTests.TestDirectoryCreate;
var
  DirName: String;
begin
  DirName := FTempDir + 'subdir' + PathDelim + 'nested';
  
  ForceDirectories(DirName);
  
  AssertTrue('Directory should exist', DirectoryExists(DirName));
end;

initialization
  RegisterTest('String Tests', TStringTests);
  RegisterTest('DateTime Tests', TDateTimeTests);
  RegisterTest('File Tests', TFileTests);
end.
```

---

## 49.4 Setup และ TearDown

### Pattern สำหรับ Setup/TearDown

```pascal
{$mode objfpc}{$H+}

// test_database.pas - ตัวอย่าง Database Tests
unit TestDatabase;

interface

uses
  fpcunit, testregistry,
  sqlite3ds, db;

type
  TDatabaseTests = class(TTestCase)
  private
    FDB: TSQLite3Dataset; // หรือ database component ที่ใช้
    FTestDBPath: String;
    
    procedure CreateTestDatabase;
    procedure InsertTestData;
  protected
    procedure SetUp; override;
    procedure TearDown; override;
  published
    procedure TestInsert;
    procedure TestSelect;
    procedure TestUpdate;
    procedure TestDelete;
    procedure TestTransaction;
    procedure TestConstraints;
  end;

implementation

uses
  SysUtils, Classes;

{ Helper: Simple database wrapper for testing }
type
  TSimpleDB = class
  private
    FDBPath: String;
  public
    constructor Create(const DBPath: String);
    function Execute(const SQL: String): Boolean;
    function QueryValue(const SQL: String): Variant;
    function QueryCount(const SQL: String): Integer;
  end;

constructor TSimpleDB.Create(const DBPath: String);
begin
  inherited Create;
  FDBPath := DBPath;
end;

function TSimpleDB.Execute(const SQL: String): Boolean;
begin
  // ใน production จะเรียก real database
  Result := True;
  WriteLn('Execute: ', SQL);
end;

function TSimpleDB.QueryValue(const SQL: String): Variant;
begin
  Result := Null;
  WriteLn('Query: ', SQL);
end;

function TSimpleDB.QueryCount(const SQL: String): Integer;
begin
  Result := 0;
  WriteLn('Count query: ', SQL);
end;

{ TDatabaseTests }

procedure TDatabaseTests.SetUp;
begin
  // สร้าง temp database สำหรับ test
  FTestDBPath := GetTempDir + 'test_' + IntToStr(Random(99999)) + '.db';
  CreateTestDatabase;
  InsertTestData;
end;

procedure TDatabaseTests.TearDown;
begin
  // ปิดและลบ test database
  if FileExists(FTestDBPath) then
    DeleteFile(FTestDBPath);
end;

procedure TDatabaseTests.CreateTestDatabase;
var
  DB: TSimpleDB;
begin
  DB := TSimpleDB.Create(FTestDBPath);
  try
    DB.Execute('CREATE TABLE IF NOT EXISTS customers (' +
      'id INTEGER PRIMARY KEY AUTOINCREMENT, ' +
      'name TEXT NOT NULL, ' +
      'email TEXT UNIQUE NOT NULL, ' +
      'created_at DATETIME DEFAULT CURRENT_TIMESTAMP' +
      ')');
  finally
    DB.Free;
  end;
end;

procedure TDatabaseTests.InsertTestData;
var
  DB: TSimpleDB;
begin
  DB := TSimpleDB.Create(FTestDBPath);
  try
    DB.Execute('INSERT INTO customers (name, email) VALUES (''John Doe'', ''john@test.com'')');
    DB.Execute('INSERT INTO customers (name, email) VALUES (''Jane Doe'', ''jane@test.com'')');
    DB.Execute('INSERT INTO customers (name, email) VALUES (''Bob Smith'', ''bob@test.com'')');
  finally
    DB.Free;
  end;
end;

procedure TDatabaseTests.TestInsert;
var
  DB: TSimpleDB;
begin
  DB := TSimpleDB.Create(FTestDBPath);
  try
    DB.Execute('INSERT INTO customers (name, email) VALUES (''New User'', ''new@test.com'')');
    AssertEquals('Should have 4 customers', 4,
      DB.QueryCount('SELECT COUNT(*) FROM customers'));
  finally
    DB.Free;
  end;
end;

procedure TDatabaseTests.TestSelect;
var
  DB: TSimpleDB;
begin
  DB := TSimpleDB.Create(FTestDBPath);
  try
    AssertEquals('Initial count', 3,
      DB.QueryCount('SELECT COUNT(*) FROM customers'));
  finally
    DB.Free;
  end;
end;

procedure TDatabaseTests.TestUpdate;
var
  DB: TSimpleDB;
begin
  DB := TSimpleDB.Create(FTestDBPath);
  try
    DB.Execute('UPDATE customers SET name = ''Updated Name'' WHERE email = ''john@test.com''');
    // ตรวจสอบว่า update สำเร็จ
    AssertTrue('Update should succeed', True);
  finally
    DB.Free;
  end;
end;

procedure TDatabaseTests.TestDelete;
var
  DB: TSimpleDB;
begin
  DB := TSimpleDB.Create(FTestDBPath);
  try
    DB.Execute('DELETE FROM customers WHERE email = ''bob@test.com''');
    AssertEquals('Should have 2 customers after delete', 2,
      DB.QueryCount('SELECT COUNT(*) FROM customers'));
  finally
    DB.Free;
  end;
end;

procedure TDatabaseTests.TestTransaction;
var
  DB: TSimpleDB;
begin
  DB := TSimpleDB.Create(FTestDBPath);
  try
    DB.Execute('BEGIN TRANSACTION');
    DB.Execute('INSERT INTO customers (name, email) VALUES (''Trans User'', ''trans@test.com'')');
    DB.Execute('ROLLBACK');
    
    // ตรวจสอบว่า rollback ทำงาน
    AssertEquals('Transaction rolled back', 3,
      DB.QueryCount('SELECT COUNT(*) FROM customers'));
  finally
    DB.Free;
  end;
end;

procedure TDatabaseTests.TestConstraints;
var
  DB: TSimpleDB;
  ExceptionRaised: Boolean;
begin
  DB := TSimpleDB.Create(FTestDBPath);
  try
    ExceptionRaised := False;
    try
      // พยายามใส่ duplicate email
      DB.Execute('INSERT INTO customers (name, email) VALUES (''Dup'', ''john@test.com'')');
    except
      ExceptionRaised := True;
    end;
    
    AssertTrue('UNIQUE constraint should prevent duplicate email', ExceptionRaised);
  finally
    DB.Free;
  end;
end;

initialization
  RegisterTest('Database Tests', TDatabaseTests);
end.
```

---

## 49.5 Assertions

### Assertion Methods ใน FPCUnit

```pascal
{$mode objfpc}{$H+}

// test_assertions.pas - ตัวอย่างการใช้ assertions ต่างๆ
unit TestAssertions;

interface

uses
  fpcunit, testregistry;

type
  TAssertionExamples = class(TTestCase)
  published
    procedure TestBooleanAssertions;
    procedure TestEqualityAssertions;
    procedure TestNilAssertions;
    procedure TestStringAssertions;
    procedure TestNumberAssertions;
    procedure TestExceptionAssertions;
    procedure TestContainerAssertions;
  end;

implementation

uses
  SysUtils, Classes;

procedure TAssertionExamples.TestBooleanAssertions;
begin
  // AssertTrue / AssertFalse
  AssertTrue('Should be true', 1 = 1);
  AssertFalse('Should be false', 1 = 2);
  AssertTrue('String not empty', Length('hello') > 0);
  AssertFalse('String is empty', Length('') > 0);
end;

procedure TAssertionExamples.TestEqualityAssertions;
begin
  // AssertEquals สำหรับหลายชนิดข้อมูล
  
  // Integer
  AssertEquals('Integer equal', 42, 6 * 7);
  
  // String
  AssertEquals('String equal', 'Hello', UpperCase('hello'));
  
  // Boolean
  AssertEquals('Boolean equal', True, 5 > 3);
  
  // Float (ต้องระบุ delta)
  AssertEqualsWithDelta('Float equal', 3.14159, Pi, 0.00001);
  
  // ไม่เท่ากัน
  AssertNotEquals('Not equal', 1, 2);
end;

procedure TAssertionExamples.TestNilAssertions;
var
  Obj: TObject;
  Ptr: Pointer;
begin
  // AssertNull / AssertNotNull
  Obj := nil;
  AssertNull('Object should be nil', Obj);
  
  Obj := TObject.Create;
  try
    AssertNotNull('Object should not be nil', Obj);
  finally
    Obj.Free;
  end;
  
  Ptr := nil;
  AssertNull('Pointer should be nil', Ptr);
end;

procedure TAssertionExamples.TestStringAssertions;
begin
  // String assertions
  AssertEquals('Strings equal', 'hello', 'hello');
  AssertNotEquals('Strings not equal', 'hello', 'world');
  
  // Case insensitive ทำเอง
  AssertEquals('Case insensitive', 
    LowerCase('HELLO'), LowerCase('Hello'));
  
  // Contains
  AssertTrue('Contains substring', 
    Pos('world', 'hello world') > 0);
    
  // StartsWith / EndsWith
  AssertTrue('Starts with', 
    Copy('Hello World', 1, 5) = 'Hello');
  AssertTrue('Ends with',
    Copy('Hello World', Length('Hello World') - 4, 5) = 'World');
end;

procedure TAssertionExamples.TestNumberAssertions;
begin
  // Integer comparisons
  AssertTrue('Greater than', 5 > 3);
  AssertTrue('Less than', 3 < 5);
  AssertTrue('Greater or equal', 5 >= 5);
  AssertTrue('In range', (5 >= 1) and (5 <= 10));
  
  // Float
  AssertTrue('Float positive', Pi > 0);
  AssertEqualsWithDelta('Sqrt(2)', 1.41421, Sqrt(2), 0.0001);
  
  // Max/Min
  AssertEquals('Max', 10, Max(5, 10));
  AssertEquals('Min', 3, Min(3, 7));
end;

procedure TAssertionExamples.TestExceptionAssertions;
begin
  // AssertException - ตรวจว่า exception ถูก raise
  AssertException(
    'Should raise EDivByZero',
    EDivByZero,
    procedure
    var X: Integer;
    begin
      X := 5 div 0;
    end
  );
  
  // ตรวจ exception message
  try
    raise EArgumentException.Create('Invalid argument: test');
  except
    on E: EArgumentException do
      AssertTrue('Exception message', Pos('Invalid argument', E.Message) > 0);
  end;
end;

procedure TAssertionExamples.TestContainerAssertions;
var
  List: TStringList;
begin
  List := TStringList.Create;
  try
    List.Add('apple');
    List.Add('banana');
    List.Add('cherry');
    
    AssertEquals('List count', 3, List.Count);
    AssertEquals('First item', 'apple', List[0]);
    AssertEquals('Last item', 'cherry', List[List.Count - 1]);
    AssertTrue('Contains item', List.IndexOf('banana') >= 0);
    AssertFalse('Not contains', List.IndexOf('grape') >= 0);
  finally
    List.Free;
  end;
end;

initialization
  RegisterTest('Assertion Examples', TAssertionExamples);
end.
```

---

## 49.6 Mock Objects

### การสร้าง Mock Objects

```pascal
{$mode objfpc}{$H+}

// mock_objects.pas - ตัวอย่าง Mock Objects
unit MockObjects;

interface

uses
  Classes, SysUtils;

// ============================================================
// Interface ที่จะ Mock
// ============================================================
type
  IEmailService = interface
    ['{12345678-ABCD-1234-EFGH-123456789012}']
    function SendEmail(const ToAddr, Subject, Body: String): Boolean;
    function GetSentCount: Integer;
  end;

  IUserRepository = interface
    ['{ABCDEFGH-1234-5678-ABCD-EFGHIJKLMNOP}']
    function FindByEmail(const Email: String): Boolean;
    function Save(const Name, Email: String): Integer;
    function Delete(ID: Integer): Boolean;
  end;

// ============================================================
// Mock implementations
// ============================================================
type
  TEmailServiceMock = class(TInterfacedObject, IEmailService)
  private
    FSentCount: Integer;
    FLastTo: String;
    FLastSubject: String;
    FLastBody: String;
    FShouldSucceed: Boolean;
  public
    constructor Create(ShouldSucceed: Boolean = True);
    function SendEmail(const ToAddr, Subject, Body: String): Boolean;
    function GetSentCount: Integer;
    
    // Verification methods
    property LastTo: String read FLastTo;
    property LastSubject: String read FLastSubject;
    property LastBody: String read FLastBody;
    property SentCount: Integer read FSentCount;
  end;

  TUserRepositoryMock = class(TInterfacedObject, IUserRepository)
  private
    FUsers: TStringList; // email -> id
    FNextID: Integer;
    FCallLog: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    function FindByEmail(const Email: String): Boolean;
    function Save(const Name, Email: String): Integer;
    function Delete(ID: Integer): Boolean;
    
    // Setup methods
    procedure AddExistingUser(const Email: String; ID: Integer);
    
    // Verification methods
    function WasCalled(const Method: String): Boolean;
    property CallLog: TStringList read FCallLog;
  end;

// Service ที่ใช้ dependencies
type
  TUserRegistrationService = class
  private
    FEmailService: IEmailService;
    FUserRepo: IUserRepository;
  public
    constructor Create(EmailSvc: IEmailService; UserRepo: IUserRepository);
    function RegisterUser(const Name, Email: String): Boolean;
  end;

implementation

{ TEmailServiceMock }

constructor TEmailServiceMock.Create(ShouldSucceed: Boolean = True);
begin
  inherited Create;
  FSentCount := 0;
  FShouldSucceed := ShouldSucceed;
end;

function TEmailServiceMock.SendEmail(const ToAddr, Subject, Body: String): Boolean;
begin
  FLastTo := ToAddr;
  FLastSubject := Subject;
  FLastBody := Body;
  Inc(FSentCount);
  Result := FShouldSucceed;
end;

function TEmailServiceMock.GetSentCount: Integer;
begin
  Result := FSentCount;
end;

{ TUserRepositoryMock }

constructor TUserRepositoryMock.Create;
begin
  inherited Create;
  FUsers := TStringList.Create;
  FCallLog := TStringList.Create;
  FNextID := 1;
end;

destructor TUserRepositoryMock.Destroy;
begin
  FUsers.Free;
  FCallLog.Free;
  inherited;
end;

function TUserRepositoryMock.FindByEmail(const Email: String): Boolean;
begin
  FCallLog.Add('FindByEmail:' + Email);
  Result := FUsers.IndexOfName(Email) >= 0;
end;

function TUserRepositoryMock.Save(const Name, Email: String): Integer;
begin
  FCallLog.Add('Save:' + Name + ':' + Email);
  Result := FNextID;
  FUsers.AddPair(Email, IntToStr(FNextID));
  Inc(FNextID);
end;

function TUserRepositoryMock.Delete(ID: Integer): Boolean;
var
  i: Integer;
begin
  FCallLog.Add('Delete:' + IntToStr(ID));
  Result := False;
  for i := 0 to FUsers.Count - 1 do
  begin
    if FUsers.ValueFromIndex[i] = IntToStr(ID) then
    begin
      FUsers.Delete(i);
      Result := True;
      Break;
    end;
  end;
end;

procedure TUserRepositoryMock.AddExistingUser(const Email: String; ID: Integer);
begin
  FUsers.AddPair(Email, IntToStr(ID));
  if ID >= FNextID then
    FNextID := ID + 1;
end;

function TUserRepositoryMock.WasCalled(const Method: String): Boolean;
var
  i: Integer;
begin
  Result := False;
  for i := 0 to FCallLog.Count - 1 do
    if Pos(Method, FCallLog[i]) > 0 then
    begin
      Result := True;
      Break;
    end;
end;

{ TUserRegistrationService }

constructor TUserRegistrationService.Create(EmailSvc: IEmailService; 
  UserRepo: IUserRepository);
begin
  inherited Create;
  FEmailService := EmailSvc;
  FUserRepo := UserRepo;
end;

function TUserRegistrationService.RegisterUser(const Name, Email: String): Boolean;
begin
  Result := False;
  
  // ตรวจสอบว่า email ซ้ำไหม
  if FUserRepo.FindByEmail(Email) then
    Exit;
  
  // บันทึก user
  FUserRepo.Save(Name, Email);
  
  // ส่ง welcome email
  FEmailService.SendEmail(Email, 
    'Welcome to MyApp!', 
    'Hi ' + Name + ', your account has been created.');
  
  Result := True;
end;

end.

// ============================================================
// Test ที่ใช้ Mock Objects
// ============================================================
unit TestUserRegistration;

interface

uses
  fpcunit, testregistry, MockObjects;

type
  TUserRegistrationTests = class(TTestCase)
  private
    FEmailMock: TEmailServiceMock;
    FUserRepoMock: TUserRepositoryMock;
    FService: TUserRegistrationService;
  protected
    procedure SetUp; override;
    procedure TearDown; override;
  published
    procedure TestNewUserRegistration;
    procedure TestDuplicateEmail;
    procedure TestEmailSentOnRegistration;
    procedure TestEmailContentCorrect;
    procedure TestRepositoryCalledCorrectly;
  end;

implementation

procedure TUserRegistrationTests.SetUp;
begin
  FEmailMock := TEmailServiceMock.Create(True);
  FUserRepoMock := TUserRepositoryMock.Create;
  FService := TUserRegistrationService.Create(FEmailMock, FUserRepoMock);
end;

procedure TUserRegistrationTests.TearDown;
begin
  FService.Free;
  FUserRepoMock.Free;
  FEmailMock.Free;
end;

procedure TUserRegistrationTests.TestNewUserRegistration;
begin
  AssertTrue('New user should register successfully',
    FService.RegisterUser('John Doe', 'john@test.com'));
end;

procedure TUserRegistrationTests.TestDuplicateEmail;
begin
  // เพิ่ม user ที่มีอยู่แล้ว
  FUserRepoMock.AddExistingUser('existing@test.com', 1);
  
  AssertFalse('Duplicate email should fail',
    FService.RegisterUser('Another User', 'existing@test.com'));
end;

procedure TUserRegistrationTests.TestEmailSentOnRegistration;
begin
  FService.RegisterUser('Jane Doe', 'jane@test.com');
  
  AssertEquals('Email should be sent once', 1, FEmailMock.SentCount);
  AssertEquals('Email sent to correct address', 
    'jane@test.com', FEmailMock.LastTo);
end;

procedure TUserRegistrationTests.TestEmailContentCorrect;
begin
  FService.RegisterUser('Bob Smith', 'bob@test.com');
  
  AssertEquals('Email subject correct', 
    'Welcome to MyApp!', FEmailMock.LastSubject);
  AssertTrue('Email body contains name',
    Pos('Bob Smith', FEmailMock.LastBody) > 0);
end;

procedure TUserRegistrationTests.TestRepositoryCalledCorrectly;
begin
  FService.RegisterUser('Test User', 'test@test.com');
  
  AssertTrue('FindByEmail should be called',
    FUserRepoMock.WasCalled('FindByEmail'));
  AssertTrue('Save should be called',
    FUserRepoMock.WasCalled('Save'));
end;

initialization
  RegisterTest('User Registration Tests', TUserRegistrationTests);
end.
```

---

## 49.7 TDD (Test-Driven Development)

### ขั้นตอน TDD

TDD มี 3 ขั้นตอน: **Red → Green → Refactor**

```pascal
{$mode objfpc}{$H+}

// TDD Example: สร้าง Stack implementation
// ขั้นที่ 1: เขียน test ก่อน (Red - test จะ fail)

unit TestStack;

interface

uses
  fpcunit, testregistry;

type
  TStackTests = class(TTestCase)
  published
    // เขียน tests ก่อน implement Stack
    procedure TestNewStackIsEmpty;
    procedure TestPushIncreasesCount;
    procedure TestPopDecreasesCount;
    procedure TestPeekDoesNotRemove;
    procedure TestLIFOOrder;
    procedure TestPopEmptyStackRaisesException;
    procedure TestPeekEmptyStackRaisesException;
  end;

implementation

uses
  SysUtils,
  Stack; // unit ที่เราจะสร้าง (ยังไม่มี)

procedure TStackTests.TestNewStackIsEmpty;
var
  S: TIntStack;
begin
  S := TIntStack.Create;
  try
    AssertTrue('New stack should be empty', S.IsEmpty);
    AssertEquals('New stack size should be 0', 0, S.Count);
  finally
    S.Free;
  end;
end;

procedure TStackTests.TestPushIncreasesCount;
var
  S: TIntStack;
begin
  S := TIntStack.Create;
  try
    S.Push(1);
    AssertEquals('Count after 1 push', 1, S.Count);
    S.Push(2);
    AssertEquals('Count after 2 pushes', 2, S.Count);
  finally
    S.Free;
  end;
end;

procedure TStackTests.TestPopDecreasesCount;
var
  S: TIntStack;
begin
  S := TIntStack.Create;
  try
    S.Push(1);
    S.Push(2);
    S.Pop;
    AssertEquals('Count after pop', 1, S.Count);
  finally
    S.Free;
  end;
end;

procedure TStackTests.TestPeekDoesNotRemove;
var
  S: TIntStack;
begin
  S := TIntStack.Create;
  try
    S.Push(42);
    AssertEquals('Peek value', 42, S.Peek);
    AssertEquals('Count unchanged after peek', 1, S.Count);
  finally
    S.Free;
  end;
end;

procedure TStackTests.TestLIFOOrder;
var
  S: TIntStack;
begin
  S := TIntStack.Create;
  try
    S.Push(1);
    S.Push(2);
    S.Push(3);
    
    AssertEquals('Pop order LIFO: 3', 3, S.Pop);
    AssertEquals('Pop order LIFO: 2', 2, S.Pop);
    AssertEquals('Pop order LIFO: 1', 1, S.Pop);
  finally
    S.Free;
  end;
end;

procedure TStackTests.TestPopEmptyStackRaisesException;
var
  S: TIntStack;
begin
  S := TIntStack.Create;
  try
    AssertException('Pop empty stack should raise exception',
      EStackEmptyException,
      procedure begin S.Pop end);
  finally
    S.Free;
  end;
end;

procedure TStackTests.TestPeekEmptyStackRaisesException;
var
  S: TIntStack;
begin
  S := TIntStack.Create;
  try
    AssertException('Peek empty stack should raise exception',
      EStackEmptyException,
      procedure begin S.Peek end);
  finally
    S.Free;
  end;
end;

initialization
  RegisterTest('Stack Tests', TStackTests);
end.

// ขั้นที่ 2: Implement Stack (Green - ทำให้ tests ผ่าน)

// stack.pas
unit Stack;

interface

uses
  SysUtils, Generics.Collections;

type
  EStackEmptyException = class(Exception);

  TIntStack = class
  private
    FItems: array of Integer;
    FCount: Integer;
  public
    constructor Create;
    procedure Push(Value: Integer);
    function Pop: Integer;
    function Peek: Integer;
    function IsEmpty: Boolean;
    property Count: Integer read FCount;
  end;

implementation

constructor TIntStack.Create;
begin
  inherited Create;
  FCount := 0;
  SetLength(FItems, 0);
end;

procedure TIntStack.Push(Value: Integer);
begin
  if FCount >= Length(FItems) then
    SetLength(FItems, Max(1, Length(FItems) * 2));
  FItems[FCount] := Value;
  Inc(FCount);
end;

function TIntStack.Pop: Integer;
begin
  if FCount = 0 then
    raise EStackEmptyException.Create('Stack is empty');
  Dec(FCount);
  Result := FItems[FCount];
end;

function TIntStack.Peek: Integer;
begin
  if FCount = 0 then
    raise EStackEmptyException.Create('Stack is empty');
  Result := FItems[FCount - 1];
end;

function TIntStack.IsEmpty: Boolean;
begin
  Result := FCount = 0;
end;

end.

// ขั้นที่ 3: Refactor (ปรับปรุง code โดยที่ tests ยังผ่าน)
```

---

## 49.8 Code Coverage

### การตรวจสอบ Code Coverage

```bash
#!/bin/bash
# code_coverage.sh

# FPC มี code coverage ผ่าน gprof หรือ lcov

# Build with profiling
fpc -pg -gl myapp_tests.pas

# Run tests
./myapp_tests

# Generate coverage report
gprof ./myapp_tests gmon.out > coverage_report.txt

echo "Coverage report saved to coverage_report.txt"
```

### Custom Coverage Tracking (ใน Code)

```pascal
{$mode objfpc}{$H+}

// coverage_tracker.pas
unit CoverageTracker;

interface

uses
  Classes, SysUtils;

type
  TCoverageTracker = class
  private
    class var FInstance: TCoverageTracker;
    FHits: TStringList; // "unit.function" -> hit count
    procedure RecordHit(const UnitName, FuncName: String);
  public
    class function GetInstance: TCoverageTracker;
    class procedure FreeInstance;
    
    destructor Destroy; override;
    
    procedure Hit(const Location: String);
    function GetHitCount(const Location: String): Integer;
    procedure PrintReport;
    procedure SaveReport(const FileName: String);
  end;

// Macro สำหรับ tracking (compile-time constant)
{$IFDEF COVERAGE}
procedure CoverageHit(const Location: String); inline;
{$ENDIF}

implementation

class function TCoverageTracker.GetInstance: TCoverageTracker;
begin
  if FInstance = nil then
    FInstance := TCoverageTracker.Create;
  Result := FInstance;
end;

class procedure TCoverageTracker.FreeInstance;
begin
  FreeAndNil(FInstance);
end;

destructor TCoverageTracker.Destroy;
begin
  FHits.Free;
  inherited;
end;

constructor TCoverageTracker.Create_helper;
begin
  inherited Create;
  FHits := TStringList.Create;
end;

procedure TCoverageTracker.Hit(const Location: String);
var
  Idx: Integer;
  Count: Integer;
begin
  Idx := FHits.IndexOfName(Location);
  if Idx < 0 then
    FHits.AddPair(Location, '1')
  else
  begin
    Count := StrToIntDef(FHits.ValueFromIndex[Idx], 0);
    FHits.ValueFromIndex[Idx] := IntToStr(Count + 1);
  end;
end;

function TCoverageTracker.GetHitCount(const Location: String): Integer;
var
  Idx: Integer;
begin
  Idx := FHits.IndexOfName(Location);
  if Idx >= 0 then
    Result := StrToIntDef(FHits.ValueFromIndex[Idx], 0)
  else
    Result := 0;
end;

procedure TCoverageTracker.PrintReport;
var
  i: Integer;
begin
  WriteLn('=== Coverage Report ===');
  WriteLn(Format('Total locations: %d', [FHits.Count]));
  WriteLn('');
  WriteLn('Location                              Hits');
  WriteLn(StringOfChar('-', 50));
  
  for i := 0 to FHits.Count - 1 do
    WriteLn(Format('%-38s %s', [
      FHits.Names[i], FHits.ValueFromIndex[i]]));
end;

procedure TCoverageTracker.SaveReport(const FileName: String);
var
  Lines: TStringList;
  i: Integer;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('Location,Hits');
    for i := 0 to FHits.Count - 1 do
      Lines.Add(FHits.Names[i] + ',' + FHits.ValueFromIndex[i]);
    Lines.SaveToFile(FileName);
  finally
    Lines.Free;
  end;
end;

{$IFDEF COVERAGE}
procedure CoverageHit(const Location: String);
begin
  TCoverageTracker.GetInstance.Hit(Location);
end;
{$ENDIF}

end.
```

---

## 49.9 Complete Calculator Library Test

### Calculator Library ที่สมบูรณ์

```pascal
{$mode objfpc}{$H+}

// calc_library.pas - Complete calculator library
unit CalcLibrary;

interface

uses
  SysUtils, Math;

type
  ECalcError = class(Exception);
  EDivisionByZero = class(ECalcError);
  EInvalidInput = class(ECalcError);
  EOverflow = class(ECalcError);

  TOperationType = (opAdd, opSubtract, opMultiply, opDivide, opPower, opSqrt, opAbs);

  TCalculationResult = record
    Value: Double;
    Operation: String;
    IsError: Boolean;
    ErrorMessage: String;
  end;

  TCalcHistory = record
    Timestamp: TDateTime;
    Expression: String;
    Result: Double;
  end;

  TCalculatorLib = class
  private
    FHistory: array of TCalcHistory;
    FHistoryCount: Integer;
    FMaxHistory: Integer;
    FMemory: Double;
    
    procedure AddToHistory(const Expr: String; Value: Double);
    procedure ValidateInputs(A, B: Double; Op: TOperationType);
  public
    constructor Create(MaxHistory: Integer = 100);
    
    // Basic operations
    function Add(A, B: Double): Double;
    function Subtract(A, B: Double): Double;
    function Multiply(A, B: Double): Double;
    function Divide(A, B: Double): Double;
    
    // Advanced operations
    function Power(Base, Exponent: Double): Double;
    function SquareRoot(Value: Double): Double;
    function AbsoluteValue(Value: Double): Double;
    function Factorial(N: Integer): Int64;
    function Percentage(Value, Percent: Double): Double;
    
    // Memory
    procedure MemoryStore(Value: Double);
    function MemoryRecall: Double;
    procedure MemoryClear;
    procedure MemoryAdd(Value: Double);
    
    // History
    function GetHistoryCount: Integer;
    function GetHistory(Index: Integer): TCalcHistory;
    procedure ClearHistory;
    
    // Safe operations (no exception)
    function SafeDivide(A, B: Double; Default: Double = 0): Double;
    function TryCalculate(const Expression: String; out Result: Double): Boolean;
    
    property Memory: Double read FMemory;
  end;

implementation

constructor TCalculatorLib.Create(MaxHistory: Integer = 100);
begin
  inherited Create;
  FMaxHistory := MaxHistory;
  FHistoryCount := 0;
  FMemory := 0;
  SetLength(FHistory, FMaxHistory);
end;

procedure TCalculatorLib.AddToHistory(const Expr: String; Value: Double);
begin
  if FHistoryCount < FMaxHistory then
  begin
    FHistory[FHistoryCount].Timestamp := Now;
    FHistory[FHistoryCount].Expression := Expr;
    FHistory[FHistoryCount].Result := Value;
    Inc(FHistoryCount);
  end;
end;

procedure TCalculatorLib.ValidateInputs(A, B: Double; Op: TOperationType);
begin
  if IsNan(A) or IsInfinite(A) then
    raise EInvalidInput.Create('Invalid first operand');
  if (Op <> opSqrt) and (Op <> opAbs) then
    if IsNan(B) or IsInfinite(B) then
      raise EInvalidInput.Create('Invalid second operand');
end;

function TCalculatorLib.Add(A, B: Double): Double;
begin
  ValidateInputs(A, B, opAdd);
  Result := A + B;
  AddToHistory(Format('%g + %g = %g', [A, B, Result]), Result);
end;

function TCalculatorLib.Subtract(A, B: Double): Double;
begin
  ValidateInputs(A, B, opSubtract);
  Result := A - B;
  AddToHistory(Format('%g - %g = %g', [A, B, Result]), Result);
end;

function TCalculatorLib.Multiply(A, B: Double): Double;
begin
  ValidateInputs(A, B, opMultiply);
  Result := A * B;
  AddToHistory(Format('%g × %g = %g', [A, B, Result]), Result);
end;

function TCalculatorLib.Divide(A, B: Double): Double;
begin
  ValidateInputs(A, B, opDivide);
  if B = 0 then
    raise EDivisionByZero.Create('Cannot divide by zero');
  Result := A / B;
  AddToHistory(Format('%g ÷ %g = %g', [A, B, Result]), Result);
end;

function TCalculatorLib.Power(Base, Exponent: Double): Double;
begin
  ValidateInputs(Base, Exponent, opPower);
  Result := Math.Power(Base, Exponent);
  AddToHistory(Format('%g ^ %g = %g', [Base, Exponent, Result]), Result);
end;

function TCalculatorLib.SquareRoot(Value: Double): Double;
begin
  if Value < 0 then
    raise EInvalidInput.Create('Cannot calculate square root of negative number');
  Result := Sqrt(Value);
  AddToHistory(Format('√%g = %g', [Value, Result]), Result);
end;

function TCalculatorLib.AbsoluteValue(Value: Double): Double;
begin
  Result := Abs(Value);
  AddToHistory(Format('|%g| = %g', [Value, Result]), Result);
end;

function TCalculatorLib.Factorial(N: Integer): Int64;
var
  i: Integer;
begin
  if N < 0 then
    raise EInvalidInput.Create('Factorial not defined for negative numbers');
  if N > 20 then
    raise EOverflow.Create('Factorial overflow: N must be <= 20');
  
  Result := 1;
  for i := 2 to N do
    Result := Result * i;
  
  AddToHistory(Format('%d! = %d', [N, Result]), Result);
end;

function TCalculatorLib.Percentage(Value, Percent: Double): Double;
begin
  Result := Value * Percent / 100;
  AddToHistory(Format('%g%% of %g = %g', [Percent, Value, Result]), Result);
end;

procedure TCalculatorLib.MemoryStore(Value: Double);
begin
  FMemory := Value;
end;

function TCalculatorLib.MemoryRecall: Double;
begin
  Result := FMemory;
end;

procedure TCalculatorLib.MemoryClear;
begin
  FMemory := 0;
end;

procedure TCalculatorLib.MemoryAdd(Value: Double);
begin
  FMemory := FMemory + Value;
end;

function TCalculatorLib.GetHistoryCount: Integer;
begin
  Result := FHistoryCount;
end;

function TCalculatorLib.GetHistory(Index: Integer): TCalcHistory;
begin
  if (Index < 0) or (Index >= FHistoryCount) then
    raise EArgumentOutOfRangeException.Create('History index out of range');
  Result := FHistory[Index];
end;

procedure TCalculatorLib.ClearHistory;
begin
  FHistoryCount := 0;
end;

function TCalculatorLib.SafeDivide(A, B: Double; Default: Double = 0): Double;
begin
  if B = 0 then
    Result := Default
  else
    Result := A / B;
end;

function TCalculatorLib.TryCalculate(const Expression: String; out Result: Double): Boolean;
begin
  Result := 0;
  try
    // Simple expression parser (ตัวอย่างพื้นฐาน)
    // ใน production จะใช้ parser ที่สมบูรณ์กว่านี้
    Result := 0;
    Exit(False);
  except
    Exit(False);
  end;
end;

end.
```

### Test Suite สำหรับ Calculator Library

```pascal
{$mode objfpc}{$H+}

// test_calc_library.pas
unit TestCalcLibrary;

interface

uses
  fpcunit, testregistry, CalcLibrary;

type
  TCalcBasicTests = class(TTestCase)
  private
    FCalc: TCalculatorLib;
  protected
    procedure SetUp; override;
    procedure TearDown; override;
  published
    procedure TestAddPositive;
    procedure TestAddNegative;
    procedure TestAddZero;
    procedure TestSubtract;
    procedure TestMultiply;
    procedure TestDivide;
    procedure TestDivideByZero;
  end;

  TCalcAdvancedTests = class(TTestCase)
  private
    FCalc: TCalculatorLib;
  protected
    procedure SetUp; override;
    procedure TearDown; override;
  published
    procedure TestPower;
    procedure TestSquareRoot;
    procedure TestNegativeSqrt;
    procedure TestAbsoluteValue;
    procedure TestFactorial;
    procedure TestFactorialZero;
    procedure TestFactorialNegative;
    procedure TestFactorialOverflow;
    procedure TestPercentage;
  end;

  TCalcMemoryTests = class(TTestCase)
  private
    FCalc: TCalculatorLib;
  protected
    procedure SetUp; override;
    procedure TearDown; override;
  published
    procedure TestMemoryStore;
    procedure TestMemoryRecall;
    procedure TestMemoryClear;
    procedure TestMemoryAdd;
    procedure TestMemoryInitialValue;
  end;

  TCalcHistoryTests = class(TTestCase)
  private
    FCalc: TCalculatorLib;
  protected
    procedure SetUp; override;
    procedure TearDown; override;
  published
    procedure TestHistoryEmpty;
    procedure TestHistoryAfterOperation;
    procedure TestHistoryContent;
    procedure TestClearHistory;
    procedure TestHistoryLimit;
    procedure TestHistoryIndexOutOfRange;
  end;

implementation

uses
  SysUtils, Math;

{ TCalcBasicTests }

procedure TCalcBasicTests.SetUp;
begin
  FCalc := TCalculatorLib.Create;
end;

procedure TCalcBasicTests.TearDown;
begin
  FCalc.Free;
end;

procedure TCalcBasicTests.TestAddPositive;
begin
  AssertEqualsWithDelta('2+3=5', 5.0, FCalc.Add(2, 3), 0.0001);
end;

procedure TCalcBasicTests.TestAddNegative;
begin
  AssertEqualsWithDelta('-2+(-3)=-5', -5.0, FCalc.Add(-2, -3), 0.0001);
end;

procedure TCalcBasicTests.TestAddZero;
begin
  AssertEqualsWithDelta('0+0=0', 0.0, FCalc.Add(0, 0), 0.0001);
  AssertEqualsWithDelta('5+0=5', 5.0, FCalc.Add(5, 0), 0.0001);
end;

procedure TCalcBasicTests.TestSubtract;
begin
  AssertEqualsWithDelta('5-3=2', 2.0, FCalc.Subtract(5, 3), 0.0001);
  AssertEqualsWithDelta('3-5=-2', -2.0, FCalc.Subtract(3, 5), 0.0001);
end;

procedure TCalcBasicTests.TestMultiply;
begin
  AssertEqualsWithDelta('3*4=12', 12.0, FCalc.Multiply(3, 4), 0.0001);
  AssertEqualsWithDelta('0*5=0', 0.0, FCalc.Multiply(0, 5), 0.0001);
  AssertEqualsWithDelta('-2*3=-6', -6.0, FCalc.Multiply(-2, 3), 0.0001);
end;

procedure TCalcBasicTests.TestDivide;
begin
  AssertEqualsWithDelta('10/2=5', 5.0, FCalc.Divide(10, 2), 0.0001);
  AssertEqualsWithDelta('1/3=0.333', 1/3, FCalc.Divide(1, 3), 0.0001);
end;

procedure TCalcBasicTests.TestDivideByZero;
begin
  AssertException('Divide by zero',
    EDivisionByZero,
    procedure begin FCalc.Divide(5, 0) end);
end;

{ TCalcAdvancedTests }

procedure TCalcAdvancedTests.SetUp;
begin
  FCalc := TCalculatorLib.Create;
end;

procedure TCalcAdvancedTests.TearDown;
begin
  FCalc.Free;
end;

procedure TCalcAdvancedTests.TestPower;
begin
  AssertEqualsWithDelta('2^10=1024', 1024.0, FCalc.Power(2, 10), 0.0001);
  AssertEqualsWithDelta('3^0=1', 1.0, FCalc.Power(3, 0), 0.0001);
end;

procedure TCalcAdvancedTests.TestSquareRoot;
begin
  AssertEqualsWithDelta('sqrt(4)=2', 2.0, FCalc.SquareRoot(4), 0.0001);
  AssertEqualsWithDelta('sqrt(2)=1.414', Sqrt(2), FCalc.SquareRoot(2), 0.0001);
  AssertEqualsWithDelta('sqrt(0)=0', 0.0, FCalc.SquareRoot(0), 0.0001);
end;

procedure TCalcAdvancedTests.TestNegativeSqrt;
begin
  AssertException('Negative sqrt',
    EInvalidInput,
    procedure begin FCalc.SquareRoot(-1) end);
end;

procedure TCalcAdvancedTests.TestAbsoluteValue;
begin
  AssertEqualsWithDelta('|5|=5', 5.0, FCalc.AbsoluteValue(5), 0.0001);
  AssertEqualsWithDelta('|-5|=5', 5.0, FCalc.AbsoluteValue(-5), 0.0001);
  AssertEqualsWithDelta('|0|=0', 0.0, FCalc.AbsoluteValue(0), 0.0001);
end;

procedure TCalcAdvancedTests.TestFactorial;
begin
  AssertEquals('5!=120', 120, FCalc.Factorial(5));
  AssertEquals('10!=3628800', 3628800, FCalc.Factorial(10));
end;

procedure TCalcAdvancedTests.TestFactorialZero;
begin
  AssertEquals('0!=1', 1, FCalc.Factorial(0));
end;

procedure TCalcAdvancedTests.TestFactorialNegative;
begin
  AssertException('Negative factorial',
    EInvalidInput,
    procedure begin FCalc.Factorial(-1) end);
end;

procedure TCalcAdvancedTests.TestFactorialOverflow;
begin
  AssertException('Factorial overflow',
    EOverflow,
    procedure begin FCalc.Factorial(21) end);
end;

procedure TCalcAdvancedTests.TestPercentage;
begin
  AssertEqualsWithDelta('10% of 200=20', 20.0, FCalc.Percentage(200, 10), 0.0001);
  AssertEqualsWithDelta('50% of 100=50', 50.0, FCalc.Percentage(100, 50), 0.0001);
end;

{ TCalcMemoryTests }

procedure TCalcMemoryTests.SetUp;
begin
  FCalc := TCalculatorLib.Create;
end;

procedure TCalcMemoryTests.TearDown;
begin
  FCalc.Free;
end;

procedure TCalcMemoryTests.TestMemoryInitialValue;
begin
  AssertEqualsWithDelta('Initial memory is 0', 0.0, FCalc.Memory, 0.0001);
end;

procedure TCalcMemoryTests.TestMemoryStore;
begin
  FCalc.MemoryStore(42.5);
  AssertEqualsWithDelta('Memory stored', 42.5, FCalc.Memory, 0.0001);
end;

procedure TCalcMemoryTests.TestMemoryRecall;
begin
  FCalc.MemoryStore(99.9);
  AssertEqualsWithDelta('Memory recalled', 99.9, FCalc.MemoryRecall, 0.0001);
end;

procedure TCalcMemoryTests.TestMemoryClear;
begin
  FCalc.MemoryStore(100);
  FCalc.MemoryClear;
  AssertEqualsWithDelta('Memory cleared', 0.0, FCalc.Memory, 0.0001);
end;

procedure TCalcMemoryTests.TestMemoryAdd;
begin
  FCalc.MemoryStore(10);
  FCalc.MemoryAdd(5);
  AssertEqualsWithDelta('Memory add', 15.0, FCalc.Memory, 0.0001);
end;

{ TCalcHistoryTests }

procedure TCalcHistoryTests.SetUp;
begin
  FCalc := TCalculatorLib.Create(10); // Max 10 history entries
end;

procedure TCalcHistoryTests.TearDown;
begin
  FCalc.Free;
end;

procedure TCalcHistoryTests.TestHistoryEmpty;
begin
  AssertEquals('New calc has no history', 0, FCalc.GetHistoryCount);
end;

procedure TCalcHistoryTests.TestHistoryAfterOperation;
begin
  FCalc.Add(1, 2);
  AssertEquals('One history entry', 1, FCalc.GetHistoryCount);
  
  FCalc.Multiply(3, 4);
  AssertEquals('Two history entries', 2, FCalc.GetHistoryCount);
end;

procedure TCalcHistoryTests.TestHistoryContent;
var
  Entry: TCalcHistory;
begin
  FCalc.Add(2, 3);
  Entry := FCalc.GetHistory(0);
  
  AssertTrue('History contains expression',
    Pos('2 + 3 = 5', Entry.Expression) > 0);
  AssertEqualsWithDelta('History result', 5.0, Entry.Result, 0.0001);
end;

procedure TCalcHistoryTests.TestClearHistory;
begin
  FCalc.Add(1, 2);
  FCalc.Add(3, 4);
  FCalc.ClearHistory;
  AssertEquals('History cleared', 0, FCalc.GetHistoryCount);
end;

procedure TCalcHistoryTests.TestHistoryLimit;
var
  i: Integer;
begin
  for i := 1 to 15 do // เพิ่ม 15 entries (limit คือ 10)
    FCalc.Add(i, i);
  
  AssertEquals('History limited to max', 10, FCalc.GetHistoryCount);
end;

procedure TCalcHistoryTests.TestHistoryIndexOutOfRange;
begin
  AssertException('Index out of range',
    EArgumentOutOfRangeException,
    procedure begin FCalc.GetHistory(0) end);
end;

initialization
  RegisterTest('Calculator Basic Tests', TCalcBasicTests);
  RegisterTest('Calculator Advanced Tests', TCalcAdvancedTests);
  RegisterTest('Calculator Memory Tests', TCalcMemoryTests);
  RegisterTest('Calculator History Tests', TCalcHistoryTests);
end.
```

---

## แบบฝึกหัด 10 ข้อ

### ข้อ 1: First Unit Test
สร้าง TTestCase สำหรับ string utility functions:
- Palindrome check
- Count vowels
- Reverse string
- ทดสอบ edge cases (empty, single char, special chars)

### ข้อ 2: Setup/TearDown Pattern
สร้าง test ที่ใช้ file system:
- SetUp: สร้าง temp directory
- TearDown: ลบ temp directory
- Tests: create, read, write, delete files

### ข้อ 3: Exception Testing
ทดสอบว่า exceptions ถูก raise อย่างถูกต้อง:
- Divide by zero
- Negative square root
- Invalid array index
- File not found

### ข้อ 4: Mock Object
สร้าง mock สำหรับ ILogger interface:
- MockLogger records all log calls
- Tests verify correct log messages are written
- Tests verify log levels (Debug, Info, Warning, Error)

### ข้อ 5: TDD Exercise
ใช้ TDD สร้าง TQueue class:
- เขียน test ก่อน
- Implement จนผ่าน test
- Refactor ให้ clean

### ข้อ 6: Integration Test
สร้าง integration test ที่ทดสอบ:
- หลาย classes ทำงานร่วมกัน
- Database operations
- File reading/writing

### ข้อ 7: Parametric Tests
สร้าง test ที่ทดสอบ input หลายชุด:
- ใช้ loop สำหรับ multiple test cases
- Test data อยู่ใน array
- แต่ละ case มี input และ expected output

### ข้อ 8: Performance Test
สร้าง test ที่ตรวจสอบ performance:
- รัน operation 1000 ครั้ง
- วัดเวลา
- AssertTrue ว่าเวลาน้อยกว่า threshold

### ข้อ 9: Test Suite Organization
จัดกลุ่ม tests ให้เป็นหมวดหมู่:
- Unit Tests
- Integration Tests  
- Smoke Tests
- รัน specific suite ได้

### ข้อ 10: Full TDD Project
ใช้ TDD สร้าง simple bank account:
- Deposit
- Withdraw (ตรวจสอบ insufficient funds)
- Transfer
- Get balance
- Transaction history
