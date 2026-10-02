# ตอนที่ 73: Logging System ใน Lazarus/Pascal

## บทนำ

บทนี้ครอบคลุมการสร้างระบบ logging ที่มีประสิทธิภาพ รองรับ log levels, file rotation, structured logging และการส่ง logs ไปยัง remote services

---

## 73.1 Log Levels และ Logger พื้นฐาน

```pascal
unit logging;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs, DateUtils;

type
  TLogLevel = (
    llTrace,    // รายละเอียดสูงสุด
    llDebug,    // Debug information
    llInfo,     // ข้อมูลทั่วไป
    llWarning,  // คำเตือน
    llError,    // ข้อผิดพลาด
    llFatal,    // ข้อผิดพลาดร้ายแรง
    llNone      // ปิด logging
  );

  TLogEntry = record
    Timestamp: TDateTime;
    Level: TLogLevel;
    Logger: string;
    Message: string;
    StackTrace: string;
    Fields: TStringList;  // Structured fields
    ThreadID: TThreadID;
  end;

  ILogHandler = interface
    procedure Handle(const AEntry: TLogEntry);
    procedure Flush;
    procedure Close;
  end;

  TLogger = class
  private
    FName: string;
    FMinLevel: TLogLevel;
    FHandlers: TList;
    FLock: TCriticalSection;
    FEnabled: Boolean;
    
    procedure DoLog(ALevel: TLogLevel; const AMessage: string; 
      AFields: TStringList = nil);
    function CreateEntry(ALevel: TLogLevel; const AMessage: string;
      AFields: TStringList): TLogEntry;
    
  public
    constructor Create(const AName: string; AMinLevel: TLogLevel = llInfo);
    destructor Destroy; override;
    
    procedure AddHandler(AHandler: ILogHandler);
    procedure RemoveHandler(AHandler: ILogHandler);
    
    procedure Trace(const AMessage: string); overload;
    procedure Debug(const AMessage: string); overload;
    procedure Info(const AMessage: string); overload;
    procedure Warning(const AMessage: string); overload;
    procedure Error(const AMessage: string); overload;
    procedure Fatal(const AMessage: string); overload;
    
    // Formatted versions
    procedure Tracef(const AFormat: string; const AArgs: array of const);
    procedure Debugf(const AFormat: string; const AArgs: array of const);
    procedure Infof(const AFormat: string; const AArgs: array of const);
    procedure Warningf(const AFormat: string; const AArgs: array of const);
    procedure Errorf(const AFormat: string; const AArgs: array of const);
    procedure Fatalf(const AFormat: string; const AArgs: array of const);
    
    // With fields (structured logging)
    procedure InfoWith(const AMessage: string; AFields: TStringList);
    procedure ErrorWith(const AMessage: string; AFields: TStringList);
    
    // Log with exception
    procedure LogException(ALevel: TLogLevel; E: Exception; const AContext: string = '');
    
    procedure Flush;
    
    property Name: string read FName;
    property MinLevel: TLogLevel read FMinLevel write FMinLevel;
    property Enabled: Boolean read FEnabled write FEnabled;
  end;

  // Global logger registry
  TLoggerRegistry = class
  private
    FLoggers: TStringList;
    FDefaultLevel: TLogLevel;
    FDefaultHandlers: TList;
    class var FInstance: TLoggerRegistry;
    
    constructor CreatePrivate;
    
  public
    constructor Create;
    destructor Destroy; override;
    
    class function Instance: TLoggerRegistry;
    class procedure FreeInstance;
    
    function GetLogger(const AName: string): TLogger;
    procedure AddDefaultHandler(AHandler: ILogHandler);
    procedure SetDefaultLevel(ALevel: TLogLevel);
    procedure FlushAll;
    procedure CloseAll;
  end;

// Convenience function
function Logger(const AName: string): TLogger;
function RootLogger: TLogger;

// Log level to string
function LogLevelToStr(ALevel: TLogLevel): string;
function StrToLogLevel(const AStr: string): TLogLevel;

implementation

function LogLevelToStr(ALevel: TLogLevel): string;
begin
  case ALevel of
    llTrace:   Result := 'TRACE';
    llDebug:   Result := 'DEBUG';
    llInfo:    Result := 'INFO ';
    llWarning: Result := 'WARN ';
    llError:   Result := 'ERROR';
    llFatal:   Result := 'FATAL';
    else       Result := '?????';
  end;
end;

function StrToLogLevel(const AStr: string): TLogLevel;
var
  Upper: string;
begin
  Upper := UpperCase(Trim(AStr));
  if Upper = 'TRACE' then Result := llTrace
  else if Upper = 'DEBUG' then Result := llDebug
  else if Upper = 'INFO' then Result := llInfo
  else if Upper = 'WARN' then Result := llWarning
  else if Upper = 'WARNING' then Result := llWarning
  else if Upper = 'ERROR' then Result := llError
  else if Upper = 'FATAL' then Result := llFatal
  else Result := llInfo;
end;

{ TLogger }

constructor TLogger.Create(const AName: string; AMinLevel: TLogLevel);
begin
  inherited Create;
  FName := AName;
  FMinLevel := AMinLevel;
  FHandlers := TList.Create;
  FLock := TCriticalSection.Create;
  FEnabled := True;
end;

destructor TLogger.Destroy;
begin
  Flush;
  FHandlers.Free;
  FLock.Free;
  inherited Destroy;
end;

procedure TLogger.AddHandler(AHandler: ILogHandler);
begin
  FLock.Enter;
  try
    FHandlers.Add(Pointer(AHandler));
  finally
    FLock.Leave;
  end;
end;

function TLogger.CreateEntry(ALevel: TLogLevel; const AMessage: string;
  AFields: TStringList): TLogEntry;
begin
  Result.Timestamp := Now;
  Result.Level := ALevel;
  Result.Logger := FName;
  Result.Message := AMessage;
  Result.StackTrace := '';
  Result.ThreadID := GetCurrentThreadId;
  
  if Assigned(AFields) then
  begin
    Result.Fields := TStringList.Create;
    Result.Fields.Assign(AFields);
  end
  else
    Result.Fields := nil;
end;

procedure TLogger.DoLog(ALevel: TLogLevel; const AMessage: string; 
  AFields: TStringList);
var
  Entry: TLogEntry;
  i: Integer;
  Handler: ILogHandler;
begin
  if not FEnabled then Exit;
  if ALevel < FMinLevel then Exit;
  
  Entry := CreateEntry(ALevel, AMessage, AFields);
  
  FLock.Enter;
  try
    for i := 0 to FHandlers.Count - 1 do
    begin
      Handler := ILogHandler(FHandlers[i]);
      Handler.Handle(Entry);
    end;
  finally
    FLock.Leave;
  end;
  
  if Assigned(Entry.Fields) then
    Entry.Fields.Free;
end;

procedure TLogger.Trace(const AMessage: string);
begin
  DoLog(llTrace, AMessage);
end;

procedure TLogger.Debug(const AMessage: string);
begin
  DoLog(llDebug, AMessage);
end;

procedure TLogger.Info(const AMessage: string);
begin
  DoLog(llInfo, AMessage);
end;

procedure TLogger.Warning(const AMessage: string);
begin
  DoLog(llWarning, AMessage);
end;

procedure TLogger.Error(const AMessage: string);
begin
  DoLog(llError, AMessage);
end;

procedure TLogger.Fatal(const AMessage: string);
begin
  DoLog(llFatal, AMessage);
end;

procedure TLogger.Tracef(const AFormat: string; const AArgs: array of const);
begin
  DoLog(llTrace, Format(AFormat, AArgs));
end;

procedure TLogger.Debugf(const AFormat: string; const AArgs: array of const);
begin
  DoLog(llDebug, Format(AFormat, AArgs));
end;

procedure TLogger.Infof(const AFormat: string; const AArgs: array of const);
begin
  DoLog(llInfo, Format(AFormat, AArgs));
end;

procedure TLogger.Warningf(const AFormat: string; const AArgs: array of const);
begin
  DoLog(llWarning, Format(AFormat, AArgs));
end;

procedure TLogger.Errorf(const AFormat: string; const AArgs: array of const);
begin
  DoLog(llError, Format(AFormat, AArgs));
end;

procedure TLogger.Fatalf(const AFormat: string; const AArgs: array of const);
begin
  DoLog(llFatal, Format(AFormat, AArgs));
end;

procedure TLogger.InfoWith(const AMessage: string; AFields: TStringList);
begin
  DoLog(llInfo, AMessage, AFields);
end;

procedure TLogger.ErrorWith(const AMessage: string; AFields: TStringList);
begin
  DoLog(llError, AMessage, AFields);
end;

procedure TLogger.LogException(ALevel: TLogLevel; E: Exception; 
  const AContext: string);
var
  Msg: string;
  Fields: TStringList;
begin
  Msg := AContext;
  if Msg <> '' then Msg := Msg + ': ';
  Msg := Msg + E.ClassName + ': ' + E.Message;
  
  Fields := TStringList.Create;
  try
    Fields.Values['exception_type'] := E.ClassName;
    Fields.Values['exception_message'] := E.Message;
    if AContext <> '' then
      Fields.Values['context'] := AContext;
    
    DoLog(ALevel, Msg, Fields);
  finally
    Fields.Free;
  end;
end;

procedure TLogger.Flush;
var
  i: Integer;
  Handler: ILogHandler;
begin
  FLock.Enter;
  try
    for i := 0 to FHandlers.Count - 1 do
    begin
      Handler := ILogHandler(FHandlers[i]);
      Handler.Flush;
    end;
  finally
    FLock.Leave;
  end;
end;

end.
```

---

## 73.2 Log Handlers

```pascal
unit log_handlers;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, logging, SyncObjs;

// Console handler ที่มี colors
type
  TConsoleHandler = class(TInterfacedObject, ILogHandler)
  private
    FMinLevel: TLogLevel;
    FColorized: Boolean;
    FFormat: string;
    
    function FormatEntry(const AEntry: TLogEntry): string;
    procedure SetConsoleColor(ALevel: TLogLevel);
    procedure ResetConsoleColor;
    
  public
    constructor Create(AMinLevel: TLogLevel = llDebug; AColorized: Boolean = True);
    
    procedure Handle(const AEntry: TLogEntry);
    procedure Flush;
    procedure Close;
    
    property Format: string read FFormat write FFormat;
  end;

// File handler พร้อม rotation
type
  TFileHandler = class(TInterfacedObject, ILogHandler)
  private
    FFilePath: string;
    FMinLevel: TLogLevel;
    FFileStream: TFileStream;
    FLock: TCriticalSection;
    FMaxFileSizeBytes: Int64;
    FMaxBackupFiles: Integer;
    FCurrentSize: Int64;
    
    procedure OpenFile;
    procedure CloseFile;
    procedure RotateFile;
    function FormatEntry(const AEntry: TLogEntry): string;
    
  public
    constructor Create(const AFilePath: string; AMinLevel: TLogLevel = llInfo;
      AMaxSizeMB: Integer = 10; AMaxBackups: Integer = 5);
    destructor Destroy; override;
    
    procedure Handle(const AEntry: TLogEntry);
    procedure Flush;
    procedure Close;
    
    property FilePath: string read FFilePath;
  end;

// JSON file handler สำหรับ structured logging
type
  TJSONFileHandler = class(TInterfacedObject, ILogHandler)
  private
    FFilePath: string;
    FMinLevel: TLogLevel;
    FFile: TextFile;
    FLock: TCriticalSection;
    FIsOpen: Boolean;
    
    function EntryToJSON(const AEntry: TLogEntry): string;
    
  public
    constructor Create(const AFilePath: string; AMinLevel: TLogLevel = llInfo);
    destructor Destroy; override;
    
    procedure Handle(const AEntry: TLogEntry);
    procedure Flush;
    procedure Close;
  end;

// Memory handler สำหรับ testing
type
  TMemoryHandler = class(TInterfacedObject, ILogHandler)
  private
    FEntries: TList;
    FMaxEntries: Integer;
    
  public
    constructor Create(AMaxEntries: Integer = 1000);
    destructor Destroy; override;
    
    procedure Handle(const AEntry: TLogEntry);
    procedure Flush;
    procedure Close;
    
    function GetEntry(AIndex: Integer): TLogEntry;
    function Count: Integer;
    procedure Clear;
    function ContainsMessage(const AMessage: string): Boolean;
    function GetEntriesByLevel(ALevel: TLogLevel): TList;
  end;

implementation

{ TConsoleHandler }

constructor TConsoleHandler.Create(AMinLevel: TLogLevel; AColorized: Boolean);
begin
  inherited Create;
  FMinLevel := AMinLevel;
  FColorized := AColorized;
  FFormat := '{TIMESTAMP} [{LEVEL}] [{LOGGER}] {MESSAGE}';
end;

function TConsoleHandler.FormatEntry(const AEntry: TLogEntry): string;
var
  i: Integer;
begin
  Result := FFormat;
  Result := StringReplace(Result, '{TIMESTAMP}', 
    FormatDateTime('yyyy-mm-dd hh:nn:ss.zzz', AEntry.Timestamp), [rfReplaceAll]);
  Result := StringReplace(Result, '{LEVEL}', 
    LogLevelToStr(AEntry.Level), [rfReplaceAll]);
  Result := StringReplace(Result, '{LOGGER}', AEntry.Logger, [rfReplaceAll]);
  Result := StringReplace(Result, '{MESSAGE}', AEntry.Message, [rfReplaceAll]);
  Result := StringReplace(Result, '{THREAD}', 
    IntToStr(AEntry.ThreadID), [rfReplaceAll]);
  
  // Add structured fields
  if Assigned(AEntry.Fields) and (AEntry.Fields.Count > 0) then
  begin
    Result := Result + ' [';
    for i := 0 to AEntry.Fields.Count - 1 do
    begin
      if i > 0 then Result := Result + ', ';
      Result := Result + AEntry.Fields.Names[i] + '=' + 
        AEntry.Fields.ValueFromIndex[i];
    end;
    Result := Result + ']';
  end;
end;

procedure TConsoleHandler.SetConsoleColor(ALevel: TLogLevel);
begin
  {$IFDEF UNIX}
  case ALevel of
    llTrace:   Write(#27'[90m');   // Dark gray
    llDebug:   Write(#27'[36m');   // Cyan
    llInfo:    Write(#27'[32m');   // Green
    llWarning: Write(#27'[33m');   // Yellow
    llError:   Write(#27'[31m');   // Red
    llFatal:   Write(#27'[35m');   // Magenta
  end;
  {$ENDIF}
end;

procedure TConsoleHandler.ResetConsoleColor;
begin
  {$IFDEF UNIX}
  Write(#27'[0m');
  {$ENDIF}
end;

procedure TConsoleHandler.Handle(const AEntry: TLogEntry);
begin
  if AEntry.Level < FMinLevel then Exit;
  
  if FColorized then SetConsoleColor(AEntry.Level);
  WriteLn(FormatEntry(AEntry));
  if FColorized then ResetConsoleColor;
end;

procedure TConsoleHandler.Flush;
begin
  // Console flushes automatically
end;

procedure TConsoleHandler.Close;
begin
  // Nothing to close
end;

{ TFileHandler }

constructor TFileHandler.Create(const AFilePath: string; AMinLevel: TLogLevel;
  AMaxSizeMB: Integer; AMaxBackups: Integer);
begin
  inherited Create;
  FFilePath := AFilePath;
  FMinLevel := AMinLevel;
  FMaxFileSizeBytes := AMaxSizeMB * 1024 * 1024;
  FMaxBackupFiles := AMaxBackups;
  FLock := TCriticalSection.Create;
  FCurrentSize := 0;
  
  OpenFile;
end;

destructor TFileHandler.Destroy;
begin
  Close;
  FLock.Free;
  inherited Destroy;
end;

procedure TFileHandler.OpenFile;
var
  Dir: string;
begin
  Dir := ExtractFileDir(FFilePath);
  if (Dir <> '') and not DirectoryExists(Dir) then
    ForceDirectories(Dir);
  
  if FileExists(FFilePath) then
  begin
    FFileStream := TFileStream.Create(FFilePath, fmOpenReadWrite or fmShareDenyWrite);
    FFileStream.Seek(0, soEnd);
    FCurrentSize := FFileStream.Size;
  end
  else
  begin
    FFileStream := TFileStream.Create(FFilePath, fmCreate or fmShareDenyWrite);
    FCurrentSize := 0;
  end;
end;

procedure TFileHandler.CloseFile;
begin
  if Assigned(FFileStream) then
  begin
    FFileStream.Free;
    FFileStream := nil;
  end;
end;

procedure TFileHandler.RotateFile;
var
  i: Integer;
  BackupPath, OldPath: string;
begin
  CloseFile;
  
  // Delete oldest backup
  BackupPath := Format('%s.%d', [FFilePath, FMaxBackupFiles]);
  if FileExists(BackupPath) then
    DeleteFile(BackupPath);
  
  // Shift backups
  for i := FMaxBackupFiles - 1 downto 1 do
  begin
    OldPath := Format('%s.%d', [FFilePath, i]);
    BackupPath := Format('%s.%d', [FFilePath, i + 1]);
    
    if FileExists(OldPath) then
      RenameFile(OldPath, BackupPath);
  end;
  
  // Rename current file to .1
  RenameFile(FFilePath, FFilePath + '.1');
  
  // Open new file
  OpenFile;
  WriteLn('Log rotated: ', FFilePath);
end;

function TFileHandler.FormatEntry(const AEntry: TLogEntry): string;
begin
  Result := Format('%s [%s] [%s] %s',
    [FormatDateTime('yyyy-mm-dd hh:nn:ss.zzz', AEntry.Timestamp),
     LogLevelToStr(AEntry.Level),
     AEntry.Logger,
     AEntry.Message]);
     
  if Assigned(AEntry.Fields) and (AEntry.Fields.Count > 0) then
    Result := Result + ' ' + AEntry.Fields.Text;
    
  Result := Result + LineEnding;
end;

procedure TFileHandler.Handle(const AEntry: TLogEntry);
var
  Line: string;
  LineBytes: TBytes;
begin
  if AEntry.Level < FMinLevel then Exit;
  
  Line := FormatEntry(AEntry);
  LineBytes := TEncoding.UTF8.GetBytes(Line);
  
  FLock.Enter;
  try
    // Check if rotation needed
    if FCurrentSize + Length(LineBytes) > FMaxFileSizeBytes then
      RotateFile;
      
    if Assigned(FFileStream) then
    begin
      FFileStream.Write(LineBytes[0], Length(LineBytes));
      Inc(FCurrentSize, Length(LineBytes));
    end;
  finally
    FLock.Leave;
  end;
end;

procedure TFileHandler.Flush;
begin
  // TFileStream flushes on Write
end;

procedure TFileHandler.Close;
begin
  FLock.Enter;
  try
    CloseFile;
  finally
    FLock.Leave;
  end;
end;

{ TJSONFileHandler }

function TJSONFileHandler.EntryToJSON(const AEntry: TLogEntry): string;
var
  i: Integer;
begin
  Result := Format(
    '{"timestamp":"%s","level":"%s","logger":"%s","message":"%s","thread":%d',
    [FormatDateTime('yyyy-mm-dd"T"hh:nn:ss.zzz', AEntry.Timestamp),
     LowerCase(Trim(LogLevelToStr(AEntry.Level))),
     StringReplace(AEntry.Logger, '"', '\"', [rfReplaceAll]),
     StringReplace(AEntry.Message, '"', '\"', [rfReplaceAll]),
     AEntry.ThreadID]);
  
  if Assigned(AEntry.Fields) then
    for i := 0 to AEntry.Fields.Count - 1 do
      Result := Result + Format(',%s":"%s"', [
        '"' + AEntry.Fields.Names[i],
        StringReplace(AEntry.Fields.ValueFromIndex[i], '"', '\"', [rfReplaceAll])
      ]);
  
  Result := Result + '}';
end;

constructor TJSONFileHandler.Create(const AFilePath: string; AMinLevel: TLogLevel);
begin
  inherited Create;
  FFilePath := AFilePath;
  FMinLevel := AMinLevel;
  FLock := TCriticalSection.Create;
  FIsOpen := False;
  
  var Dir := ExtractFileDir(AFilePath);
  if (Dir <> '') and not DirectoryExists(Dir) then
    ForceDirectories(Dir);
  
  AssignFile(FFile, AFilePath);
  if FileExists(AFilePath) then
    Append(FFile)
  else
    Rewrite(FFile);
  FIsOpen := True;
end;

destructor TJSONFileHandler.Destroy;
begin
  Close;
  FLock.Free;
  inherited Destroy;
end;

procedure TJSONFileHandler.Handle(const AEntry: TLogEntry);
begin
  if AEntry.Level < FMinLevel then Exit;
  
  FLock.Enter;
  try
    if FIsOpen then
      WriteLn(FFile, EntryToJSON(AEntry));
  finally
    FLock.Leave;
  end;
end;

procedure TJSONFileHandler.Flush;
begin
  FLock.Enter;
  try
    if FIsOpen then
      System.Flush(FFile);
  finally
    FLock.Leave;
  end;
end;

procedure TJSONFileHandler.Close;
begin
  FLock.Enter;
  try
    if FIsOpen then
    begin
      CloseFile(FFile);
      FIsOpen := False;
    end;
  finally
    FLock.Leave;
  end;
end;

{ TMemoryHandler }

constructor TMemoryHandler.Create(AMaxEntries: Integer);
begin
  inherited Create;
  FEntries := TList.Create;
  FMaxEntries := AMaxEntries;
end;

destructor TMemoryHandler.Destroy;
begin
  Clear;
  FEntries.Free;
  inherited Destroy;
end;

procedure TMemoryHandler.Handle(const AEntry: TLogEntry);
var
  Entry: ^TLogEntry;
begin
  if FEntries.Count >= FMaxEntries then
  begin
    Dispose(^TLogEntry(FEntries[0]));
    FEntries.Delete(0);
  end;
  
  New(Entry);
  Entry^ := AEntry;
  if Assigned(AEntry.Fields) then
  begin
    Entry^.Fields := TStringList.Create;
    Entry^.Fields.Assign(AEntry.Fields);
  end;
  FEntries.Add(Entry);
end;

procedure TMemoryHandler.Flush;
begin
  // Memory handler doesn't need flush
end;

procedure TMemoryHandler.Close;
begin
  Clear;
end;

function TMemoryHandler.GetEntry(AIndex: Integer): TLogEntry;
begin
  Result := ^TLogEntry(FEntries[AIndex])^;
end;

function TMemoryHandler.Count: Integer;
begin
  Result := FEntries.Count;
end;

procedure TMemoryHandler.Clear;
var
  i: Integer;
  Entry: ^TLogEntry;
begin
  for i := 0 to FEntries.Count - 1 do
  begin
    Entry := FEntries[i];
    if Assigned(Entry^.Fields) then
      Entry^.Fields.Free;
    Dispose(Entry);
  end;
  FEntries.Clear;
end;

function TMemoryHandler.ContainsMessage(const AMessage: string): Boolean;
var
  i: Integer;
begin
  Result := False;
  for i := 0 to FEntries.Count - 1 do
    if Pos(AMessage, ^TLogEntry(FEntries[i])^.Message) > 0 then
    begin
      Result := True;
      Exit;
    end;
end;

end.
```

---

## 73.3 ตัวอย่างการใช้งาน

```pascal
program logging_demo;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, DateUtils,
  logging, log_handlers;

// สร้าง application logger
procedure SetupLogging;
var
  Console: TConsoleHandler;
  FileH: TFileHandler;
  JSONH: TJSONFileHandler;
  Log: TLogger;
begin
  Log := TLoggerRegistry.Instance.GetLogger('app');
  Log.MinLevel := llDebug;
  
  // Console handler
  Console := TConsoleHandler.Create(llDebug, True);
  Log.AddHandler(Console);
  
  // File handler (rotate at 10MB, keep 5 backups)
  FileH := TFileHandler.Create('/var/log/myapp/app.log', llInfo, 10, 5);
  Log.AddHandler(FileH);
  
  // JSON for machine-readable logs
  JSONH := TJSONFileHandler.Create('/var/log/myapp/app.json', llWarning);
  Log.AddHandler(JSONH);
end;

type
  TOrderService = class
  private
    FLog: TLogger;
    
  public
    constructor Create;
    
    function CreateOrder(AUserID: Integer; AAmount: Double): Integer;
    procedure ProcessPayment(AOrderID: Integer; APaymentMethod: string);
    procedure CancelOrder(AOrderID: Integer; AReason: string);
  end;

constructor TOrderService.Create;
begin
  inherited Create;
  FLog := TLoggerRegistry.Instance.GetLogger('order_service');
end;

function TOrderService.CreateOrder(AUserID: Integer; AAmount: Double): Integer;
var
  OrderID: Integer;
  Fields: TStringList;
begin
  OrderID := Random(10000) + 1;
  
  Fields := TStringList.Create;
  try
    Fields.Values['user_id'] := IntToStr(AUserID);
    Fields.Values['amount'] := Format('%.2f', [AAmount]);
    Fields.Values['order_id'] := IntToStr(OrderID);
    
    FLog.InfoWith('Order created', Fields);
  finally
    Fields.Free;
  end;
  
  Result := OrderID;
end;

procedure TOrderService.ProcessPayment(AOrderID: Integer; APaymentMethod: string);
var
  Fields: TStringList;
begin
  FLog.Debugf('Processing payment for order %d via %s', 
    [AOrderID, APaymentMethod]);
    
  // Simulate payment processing
  try
    if APaymentMethod = 'invalid' then
      raise Exception.Create('Invalid payment method');
      
    Fields := TStringList.Create;
    try
      Fields.Values['order_id'] := IntToStr(AOrderID);
      Fields.Values['payment_method'] := APaymentMethod;
      Fields.Values['status'] := 'success';
      
      FLog.InfoWith('Payment processed successfully', Fields);
    finally
      Fields.Free;
    end;
    
  except
    on E: Exception do
    begin
      FLog.LogException(llError, E, 
        Format('Payment failed for order %d', [AOrderID]));
    end;
  end;
end;

procedure TOrderService.CancelOrder(AOrderID: Integer; AReason: string);
begin
  FLog.Warningf('Order %d cancelled: %s', [AOrderID, AReason]);
end;

// Performance logging
type
  TTimer = class
  private
    FName: string;
    FStartTime: TDateTime;
    FLog: TLogger;
  public
    constructor Create(const AName: string; ALog: TLogger);
    destructor Destroy; override;
    procedure Stop(const AContext: string = '');
  end;

constructor TTimer.Create(const AName: string; ALog: TLogger);
begin
  inherited Create;
  FName := AName;
  FStartTime := Now;
  FLog := ALog;
end;

destructor TTimer.Destroy;
begin
  Stop;
  inherited Destroy;
end;

procedure TTimer.Stop(const AContext: string);
var
  Elapsed: Double;
  Fields: TStringList;
begin
  Elapsed := MilliSecondsBetween(Now, FStartTime);
  
  Fields := TStringList.Create;
  try
    Fields.Values['operation'] := FName;
    Fields.Values['elapsed_ms'] := Format('%.1f', [Elapsed]);
    if AContext <> '' then
      Fields.Values['context'] := AContext;
    
    FLog.InfoWith(Format('Operation completed: %s (%.1fms)', [FName, Elapsed]), Fields);
  finally
    Fields.Free;
  end;
end;

var
  Log: TLogger;
  OrderSvc: TOrderService;
  Timer: TTimer;
  OrderID: Integer;
begin
  Randomize;
  
  // ตั้งค่า logging
  SetupLogging;
  Log := TLoggerRegistry.Instance.GetLogger('main');
  
  Log.Info('Application starting...');
  Log.Debugf('Process ID: %d', [GetProcessID]);
  
  // สร้าง services
  OrderSvc := TOrderService.Create;
  try
    // ทำงาน
    Timer := TTimer.Create('create_order', Log);
    try
      OrderID := OrderSvc.CreateOrder(42, 1250.00);
    finally
      Timer.Free;
    end;
    
    // Process payment
    OrderSvc.ProcessPayment(OrderID, 'credit_card');
    OrderSvc.ProcessPayment(OrderID + 1, 'invalid');
    
    // Cancel
    OrderSvc.CancelOrder(OrderID, 'Customer requested');
    
  finally
    OrderSvc.Free;
  end;
  
  Log.Info('Application shutting down');
  TLoggerRegistry.Instance.FlushAll;
  TLoggerRegistry.Instance.FreeInstance;
end.
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:
1. **Log Levels** - TRACE, DEBUG, INFO, WARNING, ERROR, FATAL
2. **Logger** - Object-oriented logger ด้วย interface
3. **Console Handler** - แสดง logs ด้วยสี
4. **File Handler** - บันทึก log ไฟล์พร้อม rotation
5. **JSON Handler** - Structured logging สำหรับ machine processing
6. **Memory Handler** - Testing และ in-memory logs
7. **Performance Logging** - วัดเวลาการทำงาน

ระบบ logging ที่ดีช่วยให้เราหาข้อผิดพลาดได้รวดเร็ว และเข้าใจพฤติกรรมของระบบในสภาพ production
