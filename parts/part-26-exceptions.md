# Part 26 - Exception Handling (การจัดการข้อผิดพลาด)

## บทนำ

การจัดการข้อผิดพลาด (Exception Handling) เป็นหนึ่งในกลไกที่สำคัญที่สุดในการเขียนโปรแกรมที่มีคุณภาพ ใน Pascal/Lazarus เราใช้ระบบ Exception เพื่อจัดการกับสถานการณ์ที่ผิดปกติที่อาจเกิดขึ้นระหว่างการทำงานของโปรแกรม

### ทำไมต้องใช้ Exception Handling?

1. **ป้องกันโปรแกรมล่ม** - เมื่อเกิดข้อผิดพลาด โปรแกรมจะไม่หยุดทำงานทันที
2. **แยกโค้ดหลักออกจากการจัดการข้อผิดพลาด** - ทำให้โค้ดอ่านง่ายขึ้น
3. **ส่งต่อข้อมูลข้อผิดพลาด** - สามารถบอกผู้ใช้ว่าเกิดอะไรขึ้น
4. **ทำความสะอาดทรัพยากร** - ปิดไฟล์, คืน memory, ปิดการเชื่อมต่อ

---

## 26.1 แนวคิดพื้นฐาน (Exception Concepts)

Exception คือวัตถุ (Object) ที่ถูกสร้างขึ้นเมื่อเกิดข้อผิดพลาด มันจะ "throw" หรือ "raise" ขึ้นมา และโปรแกรมจะหยุดการทำงานปกติแล้ว "catch" หรือ "handle" exception นั้น

```pascal
program ExceptionBasics;
{$mode objfpc}{$H+}

uses
  SysUtils;

begin
  // ตัวอย่างที่ไม่มีการจัดการ exception - โปรแกรมจะล่ม!
  // var x: Integer = 10 div 0;  // EDivByZero exception
  
  WriteLn('โปรแกรมเริ่มต้น');
  
  try
    WriteLn('กำลังทำงาน...');
    raise Exception.Create('นี่คือ exception ทดสอบ');
    WriteLn('บรรทัดนี้จะไม่ถูกรัน');
  except
    on E: Exception do
      WriteLn('จับ exception ได้: ', E.Message);
  end;
  
  WriteLn('โปรแกรมยังคงทำงานต่อ');
  ReadLn;
end.
```

**ผลลัพธ์:**
```
โปรแกรมเริ่มต้น
กำลังทำงาน...
จับ exception ได้: นี่คือ exception ทดสอบ
โปรแกรมยังคงทำงานต่อ
```

---

## 26.2 try...except Block

```pascal
program TryExceptDemo;
{$mode objfpc}{$H+}

uses
  SysUtils;

var
  Num1, Num2, Result: Integer;

begin
  WriteLn('=== ตัวอย่าง try...except ===');
  
  // ตัวอย่าง 1: หารด้วยศูนย์
  WriteLn(#10'ตัวอย่างที่ 1: หารด้วยศูนย์');
  try
    Num1 := 10;
    Num2 := 0;
    Result := Num1 div Num2;
    WriteLn('ผลลัพธ์: ', Result);
  except
    on E: EDivByZero do
      WriteLn('ข้อผิดพลาด: หารด้วยศูนย์ไม่ได้ - ', E.Message);
  end;
  
  // ตัวอย่าง 2: แปลงค่าผิดพลาด
  WriteLn(#10'ตัวอย่างที่ 2: แปลงค่าผิดพลาด');
  try
    Num1 := StrToInt('ไม่ใช่ตัวเลข');
    WriteLn('ค่า: ', Num1);
  except
    on E: EConvertError do
      WriteLn('ข้อผิดพลาด: แปลงค่าไม่ได้ - ', E.Message);
  end;
  
  // ตัวอย่าง 3: จัดการหลาย exception
  WriteLn(#10'ตัวอย่างที่ 3: จัดการหลาย exception');
  try
    // ลองเปลี่ยนค่าเพื่อทดสอบ
    Num1 := 100;
    Num2 := 0;
    
    if Num2 = 0 then
      raise EDivByZero.Create('ตัวหารเป็นศูนย์');
      
    Result := Num1 div Num2;
  except
    on E: EDivByZero do
      WriteLn('หารด้วยศูนย์: ', E.Message);
    on E: EConvertError do
      WriteLn('แปลงค่าผิดพลาด: ', E.Message);
    on E: Exception do
      WriteLn('ข้อผิดพลาดทั่วไป: ', E.Message);
  end;
  
  ReadLn;
end.
```

### การจัดลำดับ Exception Handlers

```pascal
program ExceptionOrder;
{$mode objfpc}{$H+}

uses
  SysUtils;

begin
  // สำคัญ: ต้องเรียง exception จาก specific ไป general
  // ถ้าเรียงผิด compiler จะเตือน
  
  try
    raise EInvalidArgument.Create('อาร์กิวเมนต์ไม่ถูกต้อง');
  except
    // ต้องอยู่ก่อน Exception เพราะ EInvalidArgument เป็นลูกของ Exception
    on E: EInvalidArgument do
      WriteLn('จับ EInvalidArgument: ', E.Message);
    on E: Exception do
      WriteLn('จับ Exception ทั่วไป: ', E.Message);
  end;
  
  ReadLn;
end.
```

---

## 26.3 try...finally Block

`finally` block จะถูกรันเสมอ ไม่ว่าจะเกิด exception หรือไม่ก็ตาม มีประโยชน์มากสำหรับการทำความสะอาดทรัพยากร

```pascal
program TryFinallyDemo;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

procedure ProcessFile(const FileName: string);
var
  F: TextFile;
  Line: string;
begin
  WriteLn('เปิดไฟล์: ', FileName);
  AssignFile(F, FileName);
  
  try
    Reset(F);  // เปิดไฟล์
    try
      while not EOF(F) do
      begin
        ReadLn(F, Line);
        WriteLn('อ่านได้: ', Line);
      end;
    finally
      // ส่วน finally จะรันเสมอ แม้จะเกิดข้อผิดพลาด
      CloseFile(F);
      WriteLn('ปิดไฟล์เรียบร้อย');
    end;
  except
    on E: EInOutError do
      WriteLn('ข้อผิดพลาด I/O: ', E.Message);
  end;
end;

procedure DemoFinally;
var
  Counter: Integer;
begin
  WriteLn(#10'=== ตัวอย่าง try...finally ===');
  Counter := 0;
  
  try
    WriteLn('ก่อน exception');
    Inc(Counter);
    raise Exception.Create('ทดสอบ exception');
    Inc(Counter);  // บรรทัดนี้จะไม่รัน
  finally
    // รันเสมอ แม้จะเกิด exception
    WriteLn('ใน finally: Counter = ', Counter);
    WriteLn('ส่วน finally ถูกรันเสมอ');
  end;
end;

begin
  // ทดสอบ finally กับ exception
  try
    DemoFinally;
  except
    on E: Exception do
      WriteLn('จับ exception ใน main: ', E.Message);
  end;
  
  ReadLn;
end.
```

### ตัวอย่างการจัดการหน่วยความจำด้วย finally

```pascal
program MemoryManagement;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  TDataProcessor = class
  private
    FData: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    procedure ProcessData;
  end;

constructor TDataProcessor.Create;
begin
  inherited Create;
  FData := TStringList.Create;
  WriteLn('สร้าง TDataProcessor');
end;

destructor TDataProcessor.Destroy;
begin
  FData.Free;
  WriteLn('ทำลาย TDataProcessor');
  inherited Destroy;
end;

procedure TDataProcessor.ProcessData;
begin
  FData.Add('รายการที่ 1');
  FData.Add('รายการที่ 2');
  
  // จำลองข้อผิดพลาด
  raise Exception.Create('ข้อผิดพลาดในการประมวลผล');
end;

procedure RunWithoutFinally;
var
  Processor: TDataProcessor;
begin
  Processor := TDataProcessor.Create;
  // ถ้าไม่ใช้ finally และเกิด exception
  // Processor จะไม่ถูก Free - Memory Leak!
  Processor.ProcessData;  // Exception เกิดที่นี่
  Processor.Free;  // บรรทัดนี้จะไม่รัน!
end;

procedure RunWithFinally;
var
  Processor: TDataProcessor;
begin
  Processor := TDataProcessor.Create;
  try
    Processor.ProcessData;  // Exception เกิดที่นี่
  finally
    Processor.Free;  // รันเสมอ ไม่มี memory leak
  end;
end;

begin
  WriteLn('=== ตัวอย่างการจัดการหน่วยความจำ ===');
  
  WriteLn(#10'1. ไม่ใช้ finally (อาจเกิด memory leak):');
  try
    RunWithoutFinally;
  except
    on E: Exception do
      WriteLn('Exception: ', E.Message);
  end;
  
  WriteLn(#10'2. ใช้ finally (ปลอดภัย):');
  try
    RunWithFinally;
  except
    on E: Exception do
      WriteLn('Exception: ', E.Message);
  end;
  
  ReadLn;
end.
```

---

## 26.4 try...except...finally Combined

```pascal
program TryExceptFinally;
{$mode objfpc}{$H+}

uses
  SysUtils;

function Divide(A, B: Integer): Integer;
begin
  if B = 0 then
    raise EDivByZero.Create('ไม่สามารถหารด้วยศูนย์ได้');
  Result := A div B;
end;

procedure ProcessNumbers;
var
  Resource: string;
begin
  Resource := 'ทรัพยากร';
  WriteLn('จองทรัพยากร: ', Resource);
  
  try
    try
      WriteLn('กำลังประมวลผล...');
      WriteLn('10 / 2 = ', Divide(10, 2));
      WriteLn('10 / 0 = ', Divide(10, 0));  // เกิด exception
      WriteLn('บรรทัดนี้จะไม่รัน');
    except
      on E: EDivByZero do
      begin
        WriteLn('จัดการ EDivByZero: ', E.Message);
        // ส่งต่อ exception หรือไม่ก็ได้
      end;
      on E: Exception do
      begin
        WriteLn('จัดการ Exception: ', E.Message);
        raise;  // ส่งต่อ exception ขึ้นไป
      end;
    end;
  finally
    WriteLn('คืนทรัพยากร: ', Resource);
  end;
end;

begin
  WriteLn('=== try...except...finally ===');
  
  try
    ProcessNumbers;
  except
    on E: Exception do
      WriteLn('Exception ในระดับบน: ', E.Message);
  end;
  
  WriteLn('โปรแกรมทำงานเสร็จสิ้น');
  ReadLn;
end.
```

---

## 26.5 Exception Classes Hierarchy (ลำดับชั้นของ Exception)

```pascal
program ExceptionHierarchy;
{$mode objfpc}{$H+}

uses
  SysUtils;

procedure ShowHierarchy;
begin
  WriteLn('=== ลำดับชั้นของ Exception Classes ===');
  WriteLn;
  WriteLn('Exception (SysUtils)');
  WriteLn('  |-- EAbort');
  WriteLn('  |-- EHeapException');
  WriteLn('  |     |-- EOutOfMemory');
  WriteLn('  |     |-- EInvalidPointer');
  WriteLn('  |-- EArithmetic');
  WriteLn('  |     |-- EDivByZero');
  WriteLn('  |     |-- ERangeError');
  WriteLn('  |     |-- EOverflow');
  WriteLn('  |     |-- EUnderflow');
  WriteLn('  |-- EConvertError');
  WriteLn('  |-- EAccessViolation');
  WriteLn('  |-- EInOutError');
  WriteLn('  |-- EExternal');
  WriteLn('  |-- EPrivilege');
  WriteLn('  |-- EStackOverflow');
  WriteLn('  |-- EInvalidArgument');
end;

procedure TestEachException;
begin
  // EDivByZero
  WriteLn(#10'1. EDivByZero:');
  try
    var X: Integer := 10 div 0;
  except
    on E: EDivByZero do
      WriteLn('  จับได้: ', E.ClassName, ' - ', E.Message);
  end;
  
  // ERangeError (ต้องเปิด {$R+})
  WriteLn(#10'2. EConvertError:');
  try
    var N: Integer := StrToInt('abc');
  except
    on E: EConvertError do
      WriteLn('  จับได้: ', E.ClassName, ' - ', E.Message);
  end;
  
  // EInvalidArgument
  WriteLn(#10'3. EInvalidArgument:');
  try
    raise EInvalidArgument.Create('อาร์กิวเมนต์ไม่ถูกต้อง');
  except
    on E: EInvalidArgument do
      WriteLn('  จับได้: ', E.ClassName, ' - ', E.Message);
  end;
end;

begin
  ShowHierarchy;
  TestEachException;
  ReadLn;
end.
```

---

## 26.6 การสร้าง Custom Exceptions (Exception ที่กำหนดเอง)

```pascal
program CustomExceptions;
{$mode objfpc}{$H+}

uses
  SysUtils;

type
  // Exception พื้นฐานสำหรับแอปพลิเคชัน
  EAppException = class(Exception)
  private
    FErrorCode: Integer;
  public
    constructor Create(const AMessage: string; AErrorCode: Integer);
    property ErrorCode: Integer read FErrorCode;
  end;
  
  // Exception สำหรับการตรวจสอบข้อมูล
  EValidationError = class(EAppException)
  private
    FFieldName: string;
  public
    constructor Create(const AField, AMessage: string; AErrorCode: Integer);
    property FieldName: string read FFieldName;
  end;
  
  // Exception สำหรับการเชื่อมต่อ
  EConnectionError = class(EAppException)
  private
    FHost: string;
    FPort: Integer;
  public
    constructor Create(const AHost: string; APort: Integer; 
                       const AMessage: string);
    property Host: string read FHost;
    property Port: Integer read FPort;
  end;
  
  // Exception สำหรับสิทธิ์การเข้าถึง
  EPermissionDenied = class(EAppException)
  private
    FRequiredRole: string;
  public
    constructor Create(const ARequiredRole: string);
    property RequiredRole: string read FRequiredRole;
  end;

constructor EAppException.Create(const AMessage: string; AErrorCode: Integer);
begin
  inherited Create(AMessage);
  FErrorCode := AErrorCode;
end;

constructor EValidationError.Create(const AField, AMessage: string; AErrorCode: Integer);
begin
  inherited Create(Format('ฟิลด์ "%s": %s', [AField, AMessage]), AErrorCode);
  FFieldName := AField;
end;

constructor EConnectionError.Create(const AHost: string; APort: Integer;
                                     const AMessage: string);
begin
  inherited Create(Format('เชื่อมต่อ %s:%d ล้มเหลว: %s', [AHost, APort, AMessage]), 1001);
  FHost := AHost;
  FPort := APort;
end;

constructor EPermissionDenied.Create(const ARequiredRole: string);
begin
  inherited Create(Format('ต้องมีสิทธิ์ "%s" เพื่อดำเนินการนี้', [ARequiredRole]), 403);
  FRequiredRole := ARequiredRole;
end;

// ตัวอย่างการใช้ custom exceptions
procedure ValidateAge(Age: Integer);
begin
  if Age < 0 then
    raise EValidationError.Create('อายุ', 'ต้องไม่เป็นค่าลบ', 1001);
  if Age > 150 then
    raise EValidationError.Create('อายุ', 'ค่าเกินขอบเขตที่เป็นไปได้', 1002);
end;

procedure ConnectToServer(const Host: string; Port: Integer);
begin
  // จำลองการเชื่อมต่อล้มเหลว
  if Port = 0 then
    raise EConnectionError.Create(Host, Port, 'พอร์ตไม่ถูกต้อง');
  WriteLn(Format('เชื่อมต่อ %s:%d สำเร็จ', [Host, Port]));
end;

procedure PerformAdminAction(const UserRole: string);
begin
  if UserRole <> 'admin' then
    raise EPermissionDenied.Create('admin');
  WriteLn('ดำเนินการสำเร็จในฐานะ admin');
end;

begin
  WriteLn('=== Custom Exceptions ===');
  
  // ทดสอบ EValidationError
  WriteLn(#10'1. ทดสอบ EValidationError:');
  try
    ValidateAge(-5);
  except
    on E: EValidationError do
      WriteLn(Format('  ValidationError [%d]: %s (ฟิลด์: %s)', 
                     [E.ErrorCode, E.Message, E.FieldName]));
  end;
  
  try
    ValidateAge(200);
  except
    on E: EValidationError do
      WriteLn(Format('  ValidationError [%d]: %s (ฟิลด์: %s)', 
                     [E.ErrorCode, E.Message, E.FieldName]));
  end;
  
  // ทดสอบ EConnectionError
  WriteLn(#10'2. ทดสอบ EConnectionError:');
  try
    ConnectToServer('localhost', 0);
  except
    on E: EConnectionError do
      WriteLn(Format('  ConnectionError: %s (Host: %s, Port: %d)', 
                     [E.Message, E.Host, E.Port]));
  end;
  
  ConnectToServer('localhost', 8080);
  
  // ทดสอบ EPermissionDenied
  WriteLn(#10'3. ทดสอบ EPermissionDenied:');
  try
    PerformAdminAction('user');
  except
    on E: EPermissionDenied do
      WriteLn(Format('  PermissionDenied [%d]: %s (ต้องการ: %s)', 
                     [E.ErrorCode, E.Message, E.RequiredRole]));
  end;
  
  PerformAdminAction('admin');
  
  ReadLn;
end.
```

---

## 26.7 การ Raise Exception (การยกข้อผิดพลาด)

```pascal
program RaiseExceptions;
{$mode objfpc}{$H+}

uses
  SysUtils;

// 1. raise ธรรมดา
procedure RaiseSimple;
begin
  raise Exception.Create('ข้อผิดพลาดธรรมดา');
end;

// 2. raise พร้อม HelpContext
procedure RaiseWithHelp;
var
  E: Exception;
begin
  E := Exception.Create('ข้อผิดพลาดพร้อม Help Context');
  E.HelpContext := 1234;
  raise E;
end;

// 3. re-raise (ส่งต่อ exception เดิม)
procedure ProcessData;
begin
  try
    raise Exception.Create('ข้อผิดพลาดในการประมวลผล');
  except
    on E: Exception do
    begin
      WriteLn('บันทึก exception: ', E.Message);
      raise;  // ส่งต่อ exception เดิม
    end;
  end;
end;

// 4. raise exception ใหม่จาก exception เดิม
procedure ProcessWithNewException;
begin
  try
    raise EDivByZero.Create('หารด้วยศูนย์');
  except
    on E: EDivByZero do
    begin
      // ห่อ exception เดิมด้วย exception ใหม่
      raise Exception.CreateFmt('ข้อผิดพลาดในการคำนวณ: %s', [E.Message]);
    end;
  end;
end;

// 5. Conditional raise
procedure ValidateInput(Value: Integer);
begin
  if Value < 0 then
    raise EInvalidArgument.CreateFmt('ค่า %d ต้องไม่เป็นลบ', [Value]);
  if Value > 100 then
    raise EInvalidArgument.CreateFmt('ค่า %d ต้องไม่เกิน 100', [Value]);
  WriteLn('ค่าถูกต้อง: ', Value);
end;

begin
  WriteLn('=== การ Raise Exception ===');
  
  // ทดสอบแต่ละแบบ
  WriteLn(#10'1. Raise ธรรมดา:');
  try
    RaiseSimple;
  except
    on E: Exception do WriteLn('จับได้: ', E.Message);
  end;
  
  WriteLn(#10'2. Raise พร้อม HelpContext:');
  try
    RaiseWithHelp;
  except
    on E: Exception do 
      WriteLn('จับได้: ', E.Message, ' (Help: ', E.HelpContext, ')');
  end;
  
  WriteLn(#10'3. Re-raise:');
  try
    ProcessData;
  except
    on E: Exception do WriteLn('จับ re-raised exception: ', E.Message);
  end;
  
  WriteLn(#10'4. Raise exception ใหม่:');
  try
    ProcessWithNewException;
  except
    on E: Exception do WriteLn('จับได้: ', E.Message);
  end;
  
  WriteLn(#10'5. Conditional raise:');
  try
    ValidateInput(-10);
  except
    on E: EInvalidArgument do WriteLn('ข้อผิดพลาด: ', E.Message);
  end;
  
  try
    ValidateInput(150);
  except
    on E: EInvalidArgument do WriteLn('ข้อผิดพลาด: ', E.Message);
  end;
  
  ValidateInput(50);
  
  ReadLn;
end.
```

---

## 26.8 Exception Properties

```pascal
program ExceptionProperties;
{$mode objfpc}{$H+}

uses
  SysUtils;

type
  TDetailedException = class(Exception)
  private
    FErrorCode: Integer;
    FTimestamp: TDateTime;
    FLocation: string;
  public
    constructor Create(const AMessage, ALocation: string; AErrorCode: Integer);
    property ErrorCode: Integer read FErrorCode;
    property Timestamp: TDateTime read FTimestamp;
    property Location: string read FLocation;
    function ToString: string; override;
  end;

constructor TDetailedException.Create(const AMessage, ALocation: string; AErrorCode: Integer);
begin
  inherited Create(AMessage);
  FErrorCode := AErrorCode;
  FTimestamp := Now;
  FLocation := ALocation;
end;

function TDetailedException.ToString: string;
begin
  Result := Format('[%s] Error %d at %s: %s', 
                   [FormatDateTime('dd/mm/yyyy hh:nn:ss', FTimestamp),
                    FErrorCode, FLocation, Message]);
end;

procedure DemonstrateProperties;
var
  E: TDetailedException;
begin
  WriteLn('=== Exception Properties ===');
  
  // Message property
  WriteLn(#10'1. Message property:');
  try
    raise Exception.Create('ข้อความข้อผิดพลาด');
  except
    on E: Exception do
    begin
      WriteLn('  ClassName: ', E.ClassName);
      WriteLn('  Message: ', E.Message);
      WriteLn('  HelpContext: ', E.HelpContext);
    end;
  end;
  
  // Custom properties
  WriteLn(#10'2. Custom properties:');
  try
    raise TDetailedException.Create(
      'ข้อผิดพลาดการประมวลผล',
      'ProcessData',
      500
    );
  except
    on E: TDetailedException do
    begin
      WriteLn('  ErrorCode: ', E.ErrorCode);
      WriteLn('  Location: ', E.Location);
      WriteLn('  Timestamp: ', FormatDateTime('dd/mm/yyyy hh:nn:ss', E.Timestamp));
      WriteLn('  ToString: ', E.ToString);
    end;
  end;
  
  // HelpContext
  WriteLn(#10'3. HelpContext:');
  try
    var Ex := Exception.Create('ข้อผิดพลาดพร้อม Help Context');
    Ex.HelpContext := 9999;
    raise Ex;
  except
    on E: Exception do
    begin
      WriteLn('  Message: ', E.Message);
      WriteLn('  HelpContext: ', E.HelpContext);
    end;
  end;
end;

begin
  DemonstrateProperties;
  ReadLn;
end.
```

---

## 26.9 Exception Types ที่สำคัญ

```pascal
program ImportantExceptionTypes;
{$mode objfpc}{$H+}

uses
  SysUtils;

procedure TestEAccessViolation;
begin
  WriteLn('--- EAccessViolation ---');
  WriteLn('เกิดเมื่อพยายามเข้าถึงหน่วยความจำที่ไม่มีสิทธิ์');
  WriteLn('ตัวอย่าง: เข้าถึง nil pointer');
  
  // ตัวอย่างที่ปลอดภัย
  var P: PInteger := nil;
  if P = nil then
    WriteLn('Pointer เป็น nil - ไม่สามารถเข้าถึงได้')
  else
    WriteLn('ค่า: ', P^);
end;

procedure TestEDivByZero;
begin
  WriteLn(#10'--- EDivByZero ---');
  WriteLn('เกิดเมื่อหารด้วยศูนย์ (Integer)');
  
  try
    var X: Integer := 10;
    var Y: Integer := 0;
    var Z: Integer := X div Y;
    WriteLn('ผลลัพธ์: ', Z);
  except
    on E: EDivByZero do
      WriteLn('จับได้ EDivByZero: ', E.Message);
  end;
  
  // หมายเหตุ: Float ไม่เกิด exception แต่ให้ค่า Infinity
  try
    var A: Double := 10.0;
    var B: Double := 0.0;
    var C: Double := A / B;
    WriteLn('10.0 / 0.0 = ', C);  // Infinity
  except
    on E: Exception do
      WriteLn('Exception: ', E.Message);
  end;
end;

procedure TestEConvertError;
begin
  WriteLn(#10'--- EConvertError ---');
  WriteLn('เกิดเมื่อแปลงค่าไม่สำเร็จ');
  
  // StrToInt
  try
    var N := StrToInt('123abc');
  except
    on E: EConvertError do
      WriteLn('StrToInt ล้มเหลว: ', E.Message);
  end;
  
  // StrToFloat
  try
    var F := StrToFloat('12.34.56');
  except
    on E: EConvertError do
      WriteLn('StrToFloat ล้มเหลว: ', E.Message);
  end;
  
  // StrToDateTime
  try
    var D := StrToDateTime('32/13/2024');
  except
    on E: EConvertError do
      WriteLn('StrToDateTime ล้มเหลว: ', E.Message);
  end;
  
  // วิธีที่ปลอดภัย: ใช้ TryStrToInt
  var N: Integer;
  if TryStrToInt('123abc', N) then
    WriteLn('แปลงได้: ', N)
  else
    WriteLn('TryStrToInt: แปลงไม่ได้ - ใช้ค่าเริ่มต้นแทน');
end;

procedure TestERangeError;
begin
  WriteLn(#10'--- ERangeError ---');
  WriteLn('เกิดเมื่อค่าเกินขอบเขต (ต้องเปิด {$R+})');
  
  {$R+}  // เปิดการตรวจสอบ range
  try
    var A: array[0..4] of Integer;
    // การเข้าถึงเกิน index
    A[10] := 100;  // Index เกินขอบเขต
  except
    on E: ERangeError do
      WriteLn('จับได้ ERangeError: ', E.Message);
  end;
  {$R-}  // ปิดการตรวจสอบ range
end;

procedure TestEInOutError;
begin
  WriteLn(#10'--- EInOutError ---');
  WriteLn('เกิดเมื่อ Input/Output ล้มเหลว');
  
  {$I-}  // ปิด I/O checking อัตโนมัติ
  try
    {$I+}  // เปิด I/O checking
    var F: TextFile;
    AssignFile(F, '/nonexistent/path/file.txt');
    Reset(F);  // จะเกิด exception ถ้าไม่มีไฟล์
    CloseFile(F);
  except
    on E: EInOutError do
      WriteLn('จับได้ EInOutError: ', E.Message, ' (ErrorCode: ', E.ErrorCode, ')');
    on E: Exception do
      WriteLn('จับได้ Exception: ', E.ClassName, ': ', E.Message);
  end;
end;

procedure TestEOutOfMemory;
begin
  WriteLn(#10'--- EOutOfMemory ---');
  WriteLn('เกิดเมื่อจัดสรรหน่วยความจำไม่ได้');
  WriteLn('(ไม่ทดสอบจริงเพราะอาจทำให้ระบบค้าง)');
  WriteLn('วิธีป้องกัน: ตรวจสอบขนาดก่อนจัดสรร');
end;

begin
  WriteLn('=== Exception Types ที่สำคัญ ===');
  
  TestEAccessViolation;
  TestEDivByZero;
  TestEConvertError;
  TestERangeError;
  TestEInOutError;
  TestEOutOfMemory;
  
  ReadLn;
end.
```

---

## 26.10 Re-raising Exceptions

```pascal
program ReRaising;
{$mode objfpc}{$H+}

uses
  SysUtils;

procedure LevelThree;
begin
  WriteLn('  [Level 3] กำลัง raise exception');
  raise Exception.Create('ข้อผิดพลาดจาก Level 3');
end;

procedure LevelTwo;
begin
  WriteLn('  [Level 2] เรียก LevelThree');
  try
    LevelThree;
  except
    on E: Exception do
    begin
      WriteLn('  [Level 2] จับ exception: ', E.Message);
      WriteLn('  [Level 2] บันทึก log...');
      raise;  // ส่งต่อ exception เดิม
    end;
  end;
end;

procedure LevelOne;
begin
  WriteLn('  [Level 1] เรียก LevelTwo');
  try
    LevelTwo;
  except
    on E: Exception do
    begin
      WriteLn('  [Level 1] จับ exception: ', E.Message);
      // ส่งต่อในรูปแบบ exception ใหม่
      raise Exception.CreateFmt('[Level 1 Wrapper] %s', [E.Message]);
    end;
  end;
end;

// ตัวอย่างการ re-raise อย่างมีเงื่อนไข
procedure ConditionalReRaise(const Input: string);
var
  N: Integer;
begin
  try
    N := StrToInt(Input);
    WriteLn('แปลงได้: ', N);
  except
    on E: EConvertError do
    begin
      if Input = '' then
      begin
        WriteLn('Input ว่างเปล่า - ใช้ค่าเริ่มต้น 0');
        // ไม่ re-raise ถ้า input ว่างเปล่า
      end
      else
      begin
        WriteLn('Input ไม่ถูกต้อง: ', Input, ' - re-raise');
        raise;  // re-raise ถ้า input ไม่ถูกต้อง
      end;
    end;
  end;
end;

begin
  WriteLn('=== Re-raising Exceptions ===');
  
  WriteLn(#10'1. Re-raise ผ่านหลาย level:');
  try
    LevelOne;
  except
    on E: Exception do
      WriteLn('[Main] จับ exception สุดท้าย: ', E.Message);
  end;
  
  WriteLn(#10'2. Conditional Re-raise:');
  
  WriteLn('ทดสอบด้วย "":');
  try
    ConditionalReRaise('');
  except
    on E: Exception do WriteLn('Exception: ', E.Message);
  end;
  
  WriteLn('ทดสอบด้วย "abc":');
  try
    ConditionalReRaise('abc');
  except
    on E: Exception do WriteLn('Exception: ', E.Message);
  end;
  
  WriteLn('ทดสอบด้วย "123":');
  try
    ConditionalReRaise('123');
  except
    on E: Exception do WriteLn('Exception: ', E.Message);
  end;
  
  ReadLn;
end.
```

---

## 26.11 Exceptions ใน Constructors/Destructors

```pascal
program ConstructorDestructorExceptions;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  TResourceA = class
  public
    constructor Create;
    destructor Destroy; override;
    procedure DoWork;
  end;
  
  TResourceB = class
  public
    constructor Create;
    destructor Destroy; override;
  end;
  
  TComplexObject = class
  private
    FResA: TResourceA;
    FResB: TResourceB;
  public
    constructor Create;
    destructor Destroy; override;
  end;

constructor TResourceA.Create;
begin
  inherited Create;
  WriteLn('  TResourceA: สร้างสำเร็จ');
end;

destructor TResourceA.Destroy;
begin
  WriteLn('  TResourceA: ทำลายสำเร็จ');
  inherited Destroy;
end;

procedure TResourceA.DoWork;
begin
  WriteLn('  TResourceA: กำลังทำงาน...');
  raise Exception.Create('ข้อผิดพลาดใน DoWork');
end;

constructor TResourceB.Create;
begin
  inherited Create;
  WriteLn('  TResourceB: เริ่ม Create...');
  raise Exception.Create('ไม่สามารถสร้าง TResourceB ได้');
  WriteLn('  TResourceB: สร้างสำเร็จ');  // ไม่รัน
end;

destructor TResourceB.Destroy;
begin
  WriteLn('  TResourceB: ทำลาย');
  inherited Destroy;
end;

constructor TComplexObject.Create;
begin
  inherited Create;
  WriteLn('TComplexObject: เริ่ม Create');
  
  FResA := TResourceA.Create;  // สำเร็จ
  
  try
    FResB := TResourceB.Create;  // เกิด exception
  except
    on E: Exception do
    begin
      WriteLn('TComplexObject: จัดการ exception ใน constructor: ', E.Message);
      // ต้องทำความสะอาดส่วนที่สร้างไปแล้ว
      FResA.Free;
      FResA := nil;
      raise;  // re-raise เพื่อบอกว่า create ล้มเหลว
    end;
  end;
end;

destructor TComplexObject.Destroy;
begin
  WriteLn('TComplexObject: เริ่ม Destroy');
  FResA.Free;
  FResB.Free;
  inherited Destroy;
end;

begin
  WriteLn('=== Exceptions ใน Constructor/Destructor ===');
  
  WriteLn(#10'1. Exception ใน Constructor:');
  var Obj: TComplexObject := nil;
  try
    Obj := TComplexObject.Create;
    WriteLn('สร้างสำเร็จ');
  except
    on E: Exception do
    begin
      WriteLn('สร้างล้มเหลว: ', E.Message);
      // Obj ไม่ได้ถูกสร้าง ไม่ต้อง Free
    end;
  end;
  
  if Obj = nil then
    WriteLn('Object เป็น nil - ไม่ต้อง Free')
  else
  begin
    WriteLn('Object ถูกสร้าง - ต้อง Free');
    Obj.Free;
  end;
  
  WriteLn(#10'2. Exception ใน Method พร้อม finally:');
  var ResA: TResourceA;
  ResA := TResourceA.Create;
  try
    ResA.DoWork;
  except
    on E: Exception do
      WriteLn('จับ exception จาก DoWork: ', E.Message);
  end;
  ResA.Free;
  
  ReadLn;
end.
```

---

## 26.12 Logging Exceptions

```pascal
program LoggingExceptions;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  TLogLevel = (llDebug, llInfo, llWarning, llError, llCritical);
  
  TExceptionLogger = class
  private
    FLogFile: string;
    FLogList: TStringList;
    function LevelToString(Level: TLogLevel): string;
  public
    constructor Create(const ALogFile: string);
    destructor Destroy; override;
    procedure Log(Level: TLogLevel; const Message: string); overload;
    procedure LogException(E: Exception; const Context: string = ''); overload;
    procedure SaveToFile;
    procedure PrintAll;
  end;

constructor TExceptionLogger.Create(const ALogFile: string);
begin
  inherited Create;
  FLogFile := ALogFile;
  FLogList := TStringList.Create;
end;

destructor TExceptionLogger.Destroy;
begin
  FLogList.Free;
  inherited Destroy;
end;

function TExceptionLogger.LevelToString(Level: TLogLevel): string;
begin
  case Level of
    llDebug:    Result := 'DEBUG';
    llInfo:     Result := 'INFO';
    llWarning:  Result := 'WARNING';
    llError:    Result := 'ERROR';
    llCritical: Result := 'CRITICAL';
  end;
end;

procedure TExceptionLogger.Log(Level: TLogLevel; const Message: string);
var
  LogEntry: string;
begin
  LogEntry := Format('[%s] [%s] %s',
                     [FormatDateTime('dd/mm/yyyy hh:nn:ss', Now),
                      LevelToString(Level),
                      Message]);
  FLogList.Add(LogEntry);
  WriteLn(LogEntry);
end;

procedure TExceptionLogger.LogException(E: Exception; const Context: string);
var
  Message: string;
begin
  if Context <> '' then
    Message := Format('Exception in %s: [%s] %s', [Context, E.ClassName, E.Message])
  else
    Message := Format('Exception: [%s] %s', [E.ClassName, E.Message]);
  Log(llError, Message);
end;

procedure TExceptionLogger.SaveToFile;
begin
  try
    FLogList.SaveToFile(FLogFile);
    WriteLn('บันทึก log ไปที่: ', FLogFile);
  except
    on E: Exception do
      WriteLn('ไม่สามารถบันทึก log: ', E.Message);
  end;
end;

procedure TExceptionLogger.PrintAll;
var
  I: Integer;
begin
  WriteLn(#10'=== Log ทั้งหมด ===');
  for I := 0 to FLogList.Count - 1 do
    WriteLn(FLogList[I]);
end;

// ตัวอย่างฟังก์ชันที่ใช้ logger
function SafeDivide(A, B: Integer; Logger: TExceptionLogger): Integer;
begin
  try
    if B = 0 then
      raise EDivByZero.Create('ตัวหารเป็นศูนย์');
    Result := A div B;
    Logger.Log(llInfo, Format('คำนวณ %d / %d = %d', [A, B, Result]));
  except
    on E: EDivByZero do
    begin
      Logger.LogException(E, 'SafeDivide');
      Result := 0;
    end;
  end;
end;

begin
  WriteLn('=== Logging Exceptions ===');
  
  var Logger := TExceptionLogger.Create('/tmp/app.log');
  try
    Logger.Log(llInfo, 'โปรแกรมเริ่มต้น');
    
    // ทดสอบ logging
    var R1 := SafeDivide(10, 2, Logger);
    WriteLn('ผลลัพธ์: ', R1);
    
    var R2 := SafeDivide(10, 0, Logger);
    WriteLn('ผลลัพธ์: ', R2);
    
    // Log exception โดยตรง
    try
      raise EInvalidArgument.Create('พารามิเตอร์ไม่ถูกต้อง');
    except
      on E: Exception do
        Logger.LogException(E, 'MainProgram');
    end;
    
    Logger.Log(llInfo, 'โปรแกรมสิ้นสุด');
    
    Logger.PrintAll;
    // Logger.SaveToFile;  // uncomment เพื่อบันทึกไฟล์
  finally
    Logger.Free;
  end;
  
  ReadLn;
end.
```

---

## 26.13 โปรแกรมตัวอย่าง: File Processor with Error Handling

```pascal
program FileProcessor;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  EFileProcessorError = class(Exception)
  private
    FFileName: string;
  public
    constructor Create(const AFileName, AMessage: string);
    property FileName: string read FFileName;
  end;
  
  TFileProcessor = class
  private
    FInputFile: string;
    FOutputFile: string;
    FProcessedLines: Integer;
    FErrorLines: Integer;
    FLog: TStringList;
    
    procedure LogMessage(const Msg: string);
    function ProcessLine(const Line: string; LineNum: Integer): string;
  public
    constructor Create(const AInputFile, AOutputFile: string);
    destructor Destroy; override;
    procedure Process;
    procedure PrintStats;
    property ProcessedLines: Integer read FProcessedLines;
    property ErrorLines: Integer read FErrorLines;
  end;

constructor EFileProcessorError.Create(const AFileName, AMessage: string);
begin
  inherited CreateFmt('ข้อผิดพลาดในไฟล์ "%s": %s', [AFileName, AMessage]);
  FFileName := AFileName;
end;

constructor TFileProcessor.Create(const AInputFile, AOutputFile: string);
begin
  inherited Create;
  FInputFile := AInputFile;
  FOutputFile := AOutputFile;
  FProcessedLines := 0;
  FErrorLines := 0;
  FLog := TStringList.Create;
end;

destructor TFileProcessor.Destroy;
begin
  FLog.Free;
  inherited Destroy;
end;

procedure TFileProcessor.LogMessage(const Msg: string);
begin
  FLog.Add(Format('[%s] %s', [FormatDateTime('hh:nn:ss', Now), Msg]));
end;

function TFileProcessor.ProcessLine(const Line: string; LineNum: Integer): string;
var
  Parts: TStringList;
  Value: Integer;
begin
  Parts := TStringList.Create;
  try
    Parts.Delimiter := ',';
    Parts.StrictDelimiter := True;
    Parts.DelimitedText := Line;
    
    if Parts.Count < 2 then
      raise EFileProcessorError.Create(FInputFile,
        Format('บรรทัดที่ %d: รูปแบบไม่ถูกต้อง (ต้องมีอย่างน้อย 2 คอลัมน์)', [LineNum]));
    
    // แปลงคอลัมน์ที่ 2 เป็นตัวเลข
    Value := StrToInt(Trim(Parts[1]));
    
    Result := Format('%s,%.2f', [Trim(Parts[0]), Value * 1.07]);  // บวก VAT 7%
  finally
    Parts.Free;
  end;
end;

procedure TFileProcessor.Process;
var
  InputF, OutputF: TextFile;
  Line, ProcessedLine: string;
  LineNum: Integer;
begin
  LogMessage(Format('เริ่มประมวลผล: %s -> %s', [FInputFile, FOutputFile]));
  
  // เปิดไฟล์ input
  AssignFile(InputF, FInputFile);
  try
    Reset(InputF);
  except
    on E: EInOutError do
      raise EFileProcessorError.Create(FInputFile,
        Format('ไม่สามารถเปิดไฟล์ได้: %s', [E.Message]));
  end;
  
  // เปิดไฟล์ output
  AssignFile(OutputF, FOutputFile);
  try
    Rewrite(OutputF);
  except
    on E: Exception do
    begin
      CloseFile(InputF);
      raise EFileProcessorError.Create(FOutputFile,
        Format('ไม่สามารถสร้างไฟล์ output: %s', [E.Message]));
    end;
  end;
  
  // ประมวลผลแต่ละบรรทัด
  LineNum := 0;
  try
    while not EOF(InputF) do
    begin
      Inc(LineNum);
      ReadLn(InputF, Line);
      
      if Trim(Line) = '' then Continue;  // ข้ามบรรทัดว่าง
      
      try
        ProcessedLine := ProcessLine(Line, LineNum);
        WriteLn(OutputF, ProcessedLine);
        Inc(FProcessedLines);
        LogMessage(Format('บรรทัดที่ %d: ประมวลผลสำเร็จ', [LineNum]));
      except
        on E: EConvertError do
        begin
          Inc(FErrorLines);
          LogMessage(Format('บรรทัดที่ %d: ข้อผิดพลาดการแปลงค่า - %s', [LineNum, E.Message]));
          // ข้ามบรรทัดที่มีข้อผิดพลาดและดำเนินการต่อ
        end;
        on E: EFileProcessorError do
        begin
          Inc(FErrorLines);
          LogMessage(Format('บรรทัดที่ %d: %s', [LineNum, E.Message]));
          // ข้ามบรรทัดที่มีข้อผิดพลาดและดำเนินการต่อ
        end;
      end;
    end;
  finally
    CloseFile(InputF);
    CloseFile(OutputF);
  end;
  
  LogMessage(Format('ประมวลผลเสร็จสิ้น: %d บรรทัดสำเร็จ, %d บรรทัดผิดพลาด',
                    [FProcessedLines, FErrorLines]));
end;

procedure TFileProcessor.PrintStats;
var
  I: Integer;
begin
  WriteLn(#10'=== สถิติการประมวลผล ===');
  WriteLn('บรรทัดที่ประมวลผลสำเร็จ: ', FProcessedLines);
  WriteLn('บรรทัดที่มีข้อผิดพลาด: ', FErrorLines);
  WriteLn(#10'=== Log ===');
  for I := 0 to FLog.Count - 1 do
    WriteLn(FLog[I]);
end;

procedure CreateTestFile;
var
  F: TextFile;
begin
  AssignFile(F, '/tmp/input.csv');
  Rewrite(F);
  WriteLn(F, 'สินค้า A,100');
  WriteLn(F, 'สินค้า B,200');
  WriteLn(F, 'สินค้า C,abc');  // ข้อมูลผิดพลาด
  WriteLn(F, 'สินค้า D');      // ข้อมูลไม่ครบ
  WriteLn(F, 'สินค้า E,500');
  WriteLn(F, '');               // บรรทัดว่าง
  WriteLn(F, 'สินค้า F,300');
  CloseFile(F);
end;

begin
  WriteLn('=== File Processor with Error Handling ===');
  
  // สร้างไฟล์ทดสอบ
  CreateTestFile;
  WriteLn('สร้างไฟล์ทดสอบแล้ว: /tmp/input.csv');
  
  // ประมวลผล
  var Processor := TFileProcessor.Create('/tmp/input.csv', '/tmp/output.csv');
  try
    try
      Processor.Process;
      Processor.PrintStats;
    except
      on E: EFileProcessorError do
        WriteLn('ข้อผิดพลาดร้ายแรง: ', E.Message, ' (ไฟล์: ', E.FileName, ')');
      on E: Exception do
        WriteLn('ข้อผิดพลาดที่ไม่คาดหมาย: ', E.Message);
    end;
  finally
    Processor.Free;
  end;
  
  ReadLn;
end.
```

---

## 26.14 โปรแกรมตัวอย่าง: Database Error Handling

```pascal
program DatabaseErrorHandling;
{$mode objfpc}{$H+}

uses
  SysUtils, Classes;

type
  EDatabaseError = class(Exception)
  private
    FErrorCode: Integer;
    FSQL: string;
  public
    constructor Create(const ASQL, AMessage: string; AErrorCode: Integer);
    property ErrorCode: Integer read FErrorCode;
    property SQL: string read FSQL;
  end;
  
  EConnectionError = class(EDatabaseError)
  private
    FConnectionString: string;
  public
    constructor Create(const AConnectionStr, AMessage: string);
    property ConnectionString: string read FConnectionString;
  end;
  
  EQueryError = class(EDatabaseError);
  ETransactionError = class(EDatabaseError);
  EConstraintViolation = class(EDatabaseError)
  private
    FConstraintName: string;
  public
    constructor Create(const AConstraintName, AMessage: string);
    property ConstraintName: string read FConstraintName;
  end;
  
  // จำลอง Database Connection
  TSimpleDB = class
  private
    FConnected: Boolean;
    FInTransaction: Boolean;
    FData: TStringList;
    FLog: TStringList;
    
    procedure LogSQL(const SQL: string);
    procedure SimulateError(const SQL: string);
  public
    constructor Create;
    destructor Destroy; override;
    procedure Connect(const ConnectionStr: string);
    procedure Disconnect;
    procedure BeginTransaction;
    procedure Commit;
    procedure Rollback;
    procedure Execute(const SQL: string);
    function Query(const SQL: string): TStringList;
    procedure PrintData;
    property Connected: Boolean read FConnected;
    property InTransaction: Boolean read FInTransaction;
  end;

constructor EDatabaseError.Create(const ASQL, AMessage: string; AErrorCode: Integer);
begin
  inherited CreateFmt('[SQL Error %d] %s (SQL: %s)', [AErrorCode, AMessage, ASQL]);
  FErrorCode := AErrorCode;
  FSQL := ASQL;
end;

constructor EConnectionError.Create(const AConnectionStr, AMessage: string);
begin
  inherited Create('', AMessage, 1000);
  FConnectionString := AConnectionStr;
end;

constructor EConstraintViolation.Create(const AConstraintName, AMessage: string);
begin
  inherited Create('', AMessage, 2627);
  FConstraintName := AConstraintName;
end;

constructor TSimpleDB.Create;
begin
  inherited Create;
  FConnected := False;
  FInTransaction := False;
  FData := TStringList.Create;
  FLog := TStringList.Create;
end;

destructor TSimpleDB.Destroy;
begin
  if FConnected then Disconnect;
  FData.Free;
  FLog.Free;
  inherited Destroy;
end;

procedure TSimpleDB.LogSQL(const SQL: string);
begin
  FLog.Add(Format('[%s] %s', [FormatDateTime('hh:nn:ss', Now), SQL]));
  WriteLn('SQL: ', SQL);
end;

procedure TSimpleDB.SimulateError(const SQL: string);
begin
  // จำลองข้อผิดพลาดบางประเภท
  if Pos('DUPLICATE', UpperCase(SQL)) > 0 then
    raise EConstraintViolation.Create('PK_Users', 'Primary key ซ้ำกัน');
  
  if Pos('ERROR', UpperCase(SQL)) > 0 then
    raise EQueryError.Create(SQL, 'SQL syntax error', 1064);
end;

procedure TSimpleDB.Connect(const ConnectionStr: string);
begin
  if FConnected then
    raise EConnectionError.Create(ConnectionStr, 'เชื่อมต่ออยู่แล้ว');
  
  // จำลองการเชื่อมต่อ
  if ConnectionStr = '' then
    raise EConnectionError.Create(ConnectionStr, 'Connection string ว่างเปล่า');
  
  FConnected := True;
  LogSQL(Format('CONNECT: %s', [ConnectionStr]));
  WriteLn('เชื่อมต่อฐานข้อมูลสำเร็จ');
end;

procedure TSimpleDB.Disconnect;
begin
  if not FConnected then Exit;
  
  if FInTransaction then
  begin
    WriteLn('คำเตือน: ยกเลิก transaction ที่ค้างอยู่');
    Rollback;
  end;
  
  FConnected := False;
  LogSQL('DISCONNECT');
  WriteLn('ตัดการเชื่อมต่อฐานข้อมูล');
end;

procedure TSimpleDB.BeginTransaction;
begin
  if not FConnected then
    raise EConnectionError.Create('', 'ยังไม่ได้เชื่อมต่อ');
  if FInTransaction then
    raise ETransactionError.Create('', 'มี transaction อยู่แล้ว', 1);
  FInTransaction := True;
  LogSQL('BEGIN TRANSACTION');
end;

procedure TSimpleDB.Commit;
begin
  if not FInTransaction then
    raise ETransactionError.Create('', 'ไม่มี transaction ที่กำลังทำงาน', 2);
  FInTransaction := False;
  LogSQL('COMMIT');
  WriteLn('Commit สำเร็จ');
end;

procedure TSimpleDB.Rollback;
begin
  if not FInTransaction then Exit;
  FInTransaction := False;
  LogSQL('ROLLBACK');
  WriteLn('Rollback สำเร็จ');
end;

procedure TSimpleDB.Execute(const SQL: string);
begin
  if not FConnected then
    raise EConnectionError.Create('', 'ยังไม่ได้เชื่อมต่อ');
  
  LogSQL(SQL);
  SimulateError(SQL);
  
  // จำลองการเพิ่มข้อมูล
  if Pos('INSERT', UpperCase(SQL)) > 0 then
    FData.Add(SQL);
  
  WriteLn('Execute สำเร็จ: ', SQL);
end;

function TSimpleDB.Query(const SQL: string): TStringList;
begin
  if not FConnected then
    raise EConnectionError.Create('', 'ยังไม่ได้เชื่อมต่อ');
  
  LogSQL(SQL);
  SimulateError(SQL);
  
  Result := TStringList.Create;
  Result.Add('ID,Name,Email');
  Result.Add('1,สมชาย,somchai@example.com');
  Result.Add('2,สมหญิง,somying@example.com');
end;

procedure TSimpleDB.PrintData;
var
  I: Integer;
begin
  WriteLn(#10'=== ข้อมูลในฐานข้อมูล ===');
  for I := 0 to FData.Count - 1 do
    WriteLn(I + 1, ': ', FData[I]);
end;

// ฟังก์ชันสำหรับ CRUD ที่มีการจัดการ exception
procedure SafeInsertUser(DB: TSimpleDB; const Name, Email: string);
begin
  try
    DB.BeginTransaction;
    try
      DB.Execute(Format('INSERT INTO users (name, email) VALUES ("%s", "%s")', 
                        [Name, Email]));
      DB.Commit;
      WriteLn('เพิ่มผู้ใช้สำเร็จ: ', Name);
    except
      on E: EConstraintViolation do
      begin
        DB.Rollback;
        WriteLn('ไม่สามารถเพิ่มได้: ข้อมูลซ้ำ (Constraint: ', E.ConstraintName, ')');
      end;
      on E: EDatabaseError do
      begin
        DB.Rollback;
        WriteLn('ไม่สามารถเพิ่มได้: ', E.Message);
        raise;
      end;
    end;
  except
    on E: ETransactionError do
      WriteLn('Transaction Error: ', E.Message);
  end;
end;

begin
  WriteLn('=== Database Error Handling ===');
  
  var DB := TSimpleDB.Create;
  try
    // 1. ทดสอบการเชื่อมต่อ
    WriteLn(#10'1. ทดสอบการเชื่อมต่อ:');
    try
      DB.Connect('');  // Connection string ว่าง
    except
      on E: EConnectionError do
        WriteLn('ข้อผิดพลาด: ', E.Message);
    end;
    
    DB.Connect('localhost:5432/mydb');
    
    // 2. ทดสอบ Query
    WriteLn(#10'2. ทดสอบ Query:');
    var Results := DB.Query('SELECT * FROM users');
    try
      var I: Integer;
      for I := 0 to Results.Count - 1 do
        WriteLn(Results[I]);
    finally
      Results.Free;
    end;
    
    // 3. ทดสอบ Insert พร้อม Transaction
    WriteLn(#10'3. ทดสอบ Insert:');
    SafeInsertUser(DB, 'วิชัย', 'wichai@example.com');
    SafeInsertUser(DB, 'DUPLICATE User', 'dup@example.com');  // จะเกิด constraint violation
    
    // 4. ทดสอบ SQL Error
    WriteLn(#10'4. ทดสอบ SQL Error:');
    try
      DB.Execute('SELECT * FROM ERROR_TABLE');
    except
      on E: EQueryError do
        WriteLn('Query Error [', E.ErrorCode, ']: ', E.Message);
    end;
    
    DB.PrintData;
    
  finally
    DB.Free;
  end;
  
  ReadLn;
end.
```

---

## 26.15 แบบฝึกหัด (Exercises)

### แบบฝึกหัดที่ 1: Basic Exception Handling
เขียนโปรแกรมที่รับตัวเลขจากผู้ใช้และหารด้วยตัวเลขที่สอง จัดการ exception สำหรับ:
- การป้อนค่าที่ไม่ใช่ตัวเลข
- การหารด้วยศูนย์

```pascal
// โครงสร้างที่แนะนำ:
program Exercise1;
uses SysUtils;
var S1, S2: string;
    N1, N2: Integer;
begin
  Write('ป้อนตัวตั้ง: '); ReadLn(S1);
  Write('ป้อนตัวหาร: '); ReadLn(S2);
  try
    N1 := StrToInt(S1);
    N2 := StrToInt(S2);
    // TODO: จัดการ exception
  except
    // TODO: จัดการ exceptions
  end;
end.
```

### แบบฝึกหัดที่ 2: Custom Exception Classes
สร้าง exception class สำหรับระบบธนาคาร:
- `EBankException` (base class)
- `EInsufficientFunds` (ยอดเงินไม่พอ)
- `EAccountNotFound` (ไม่พบบัญชี)
- `ETransactionLimit` (เกินวงเงิน)

### แบบฝึกหัดที่ 3: Finally Block
เขียนโปรแกรมอ่านไฟล์ที่ใช้ finally เพื่อให้แน่ใจว่าไฟล์จะถูกปิดเสมอ แม้จะเกิด exception

### แบบฝึกหัดที่ 4: Exception Chaining
สร้างระบบที่ catch exception จาก low-level แล้ว wrap เป็น high-level exception ก่อน re-raise

### แบบฝึกหัดที่ 5: Exception Logger
สร้าง class `TExceptionLogger` ที่:
- บันทึก exception ลงไฟล์
- มีระดับ log (DEBUG, INFO, WARNING, ERROR)
- แสดงวันเวลาที่เกิด exception

### แบบฝึกหัดที่ 6: Safe Number Parser
สร้างฟังก์ชัน `SafeParseNumbers` ที่รับ string array และแปลงเป็น integer array โดย:
- ข้ามค่าที่แปลงไม่ได้
- บันทึกว่าบรรทัดไหนมีข้อผิดพลาด
- ส่งคืนจำนวน error ทั้งหมด

### แบบฝึกหัดที่ 7: Resource Manager
เขียน class ที่จัดการทรัพยากรหลายอย่างพร้อมกัน (เช่น ไฟล์ + DB connection + network) โดยใช้ finally ให้ถูกต้อง

### แบบฝึกหัดที่ 8: Exception Hierarchy
ออกแบบ exception hierarchy สำหรับแอปพลิเคชัน e-commerce:
- Order errors
- Payment errors
- Inventory errors
- Shipping errors

### แบบฝึกหัดที่ 9: Retry Logic
สร้างฟังก์ชัน `RetryOperation` ที่:
- รับ operation เป็น parameter
- ลองทำซ้ำถ้าเกิด exception ชนิดที่กำหนด
- กำหนดจำนวนครั้งสูงสุดและ delay ระหว่างการลอง

### แบบฝึกหัดที่ 10: Stack Unwinding
เขียนโปรแกรมที่แสดงให้เห็น stack unwinding:
- มีฟังก์ชัน A เรียก B เรียก C
- C raise exception
- แสดง message เมื่อแต่ละฟังก์ชันถูก unwind

### แบบฝึกหัดที่ 11: Exception in Loop
เขียนโปรแกรมที่วน loop ประมวลผลข้อมูล 100 รายการ โดย:
- จัดการ exception แต่ละรายการ
- ข้ามรายการที่มีข้อผิดพลาด
- รายงานสถิติเมื่อจบ

### แบบฝึกหัดที่ 12: Constructor Exception Safety
เขียน class ที่ constructor จัดสรรทรัพยากรหลายอย่าง และจัดการ exception อย่างถูกต้อง ไม่ให้มี memory leak

### แบบฝึกหัดที่ 13: Exception Conversion
สร้างฟังก์ชันที่แปลง OS-level exception เป็น application-level exception ที่มีความหมายมากขึ้น

### แบบฝึกหัดที่ 14: Aggregate Exception
สร้าง `TAggregateException` ที่เก็บ exception หลายตัวพร้อมกัน (เช่น เมื่อ validate form หลายฟิลด์)

### แบบฝึกหัดที่ 15: Exception Report Generator
เขียนโปรแกรมที่:
- จำลองการทำงานที่อาจเกิด exception หลายชนิด
- เก็บ exception ทั้งหมดที่เกิดขึ้น
- สร้างรายงานสรุป (HTML หรือ text) เมื่อโปรแกรมจบ

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **try...except** - จับและจัดการ exception
2. **try...finally** - ทำความสะอาดทรัพยากรเสมอ
3. **try...except...finally** - รวมทั้งสองแบบ
4. **Exception Hierarchy** - ลำดับชั้นของ exception classes
5. **Custom Exceptions** - สร้าง exception ของตัวเอง
6. **Raise** - ยกข้อผิดพลาด
7. **Re-raise** - ส่งต่อ exception
8. **Exception Types** - ชนิดของ exception ที่สำคัญ
9. **Exception Logging** - การบันทึก exception
10. **Best Practices** - วิธีปฏิบัติที่ดีในการจัดการ exception

### หลักการสำคัญ:
- ใช้ `finally` เสมอเมื่อจัดสรรทรัพยากร
- Catch exception เฉพาะที่รู้วิธีจัดการ
- อย่า Catch exception แล้วเงียบ (ต้อง log หรือ re-raise)
- สร้าง Custom Exception ที่มีความหมายชัดเจน
- จัดลำดับ Exception Handlers จาก Specific ไป General
