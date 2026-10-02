# Part 59 - Profiling และ Debugging ใน Lazarus/Pascal

## บทนำ

Profiling คือกระบวนการวิเคราะห์โปรแกรมเพื่อหา Bottleneck และ Memory Leaks Lazarus มีเครื่องมือ Debug และ Profile ที่ครบครัน

---

## 1. Lazarus Debugger

### การตั้ง Breakpoints

```pascal
program DebugDemo;

{$mode objfpc}{$H+}

uses Classes, SysUtils;

type
  TCalculator = class
  private
    FHistory: TStringList;
  public
    constructor Create;
    destructor Destroy; override;
    
    function Add(A, B: Double): Double;
    function Divide(A, B: Double): Double;
    function ProcessExpression(const AExpr: string): Double;
    procedure PrintHistory;
  end;

constructor TCalculator.Create;
begin
  FHistory := TStringList.Create;
end;

destructor TCalculator.Destroy;
begin
  FHistory.Free;
  inherited;
end;

function TCalculator.Add(A, B: Double): Double;
begin
  Result := A + B;  { ← ตั้ง Breakpoint ที่นี่: F5 ใน Lazarus }
  FHistory.Add(Format('%.2f + %.2f = %.2f', [A, B, Result]));
end;

function TCalculator.Divide(A, B: Double): Double;
begin
  { ตัวอย่าง Debug: ถ้า B = 0 จะเกิด Exception }
  if B = 0 then
    raise EDivByZero.Create('ไม่สามารถหารด้วยศูนย์ได้');
  
  Result := A / B;
  FHistory.Add(Format('%.2f / %.2f = %.2f', [A, B, Result]));
end;

function TCalculator.ProcessExpression(const AExpr: string): Double;
{ 
  Debug Tips:
  1. ตั้ง Breakpoint ที่บรรทัดแรกของฟังก์ชัน
  2. Step Over (F8) เพื่อดูทีละบรรทัด
  3. Step Into (F7) เพื่อเข้าไปใน Function ที่เรียก
  4. Watch Variables: เพิ่ม AExpr ใน Watch Window
  5. Evaluate: Ctrl+F7 เพื่อคำนวณค่า Expression
}
var
  Parts: TStringArray;
  Op: Char;
  Left, Right: Double;
begin
  Result := 0;
  
  { แยก Expression }
  if Pos('+', AExpr) > 0 then
  begin
    Parts := AExpr.Split(['+']);
    if Length(Parts) = 2 then
      Result := Add(StrToFloat(Trim(Parts[0])), StrToFloat(Trim(Parts[1])));
  end
  else if Pos('/', AExpr) > 0 then
  begin
    Parts := AExpr.Split(['/']);
    if Length(Parts) = 2 then
      Result := Divide(StrToFloat(Trim(Parts[0])), StrToFloat(Trim(Parts[1])));
  end;
end;

procedure TCalculator.PrintHistory;
var
  I: Integer;
begin
  WriteLn('=== ประวัติการคำนวณ ===');
  for I := 0 to FHistory.Count - 1 do
    WriteLn(FHistory[I]);
end;

var
  Calc: TCalculator;
begin
  Calc := TCalculator.Create;
  try
    WriteLn(Calc.Add(10, 20):0:2);
    WriteLn(Calc.Divide(100, 5):0:2);
    
    { ทดสอบ Exception }
    try
      WriteLn(Calc.Divide(100, 0):0:2);
    except
      on E: EDivByZero do
        WriteLn('ข้อผิดพลาด: ' + E.Message);
    end;
    
    WriteLn(Calc.ProcessExpression('15 + 25'):0:2);
    Calc.PrintHistory;
  finally
    Calc.Free;
  end;
end.
```

---

## 2. HeapTrc - Memory Leak Detection

```pascal
{
  การใช้งาน HeapTrc:
  1. เพิ่ม 'heaptrc' ใน uses section (ต้องเป็นรายการแรก)
  2. Compile ด้วย -gh flag
  3. Run โปรแกรม
  4. รายงานจะแสดงที่ท้าย Output
}

program HeapTrcDemo;

{$mode objfpc}{$H+}

uses
  heaptrc,  { ← ต้องอยู่บรรทัดแรก }
  Classes, SysUtils;

type
  TNode = class
  private
    FValue: Integer;
    FNext: TNode;
  public
    constructor Create(AValue: Integer);
    property Value: Integer read FValue;
    property Next: TNode read FNext write FNext;
  end;

  TLeakyList = class
  private
    FHead: TNode;
    FCount: Integer;
  public
    procedure Add(AValue: Integer);
    procedure PrintAll;
    { ตัวอย่าง Memory Leak: ลืม Free Nodes }
    destructor Destroy; override;
  end;

  TProperList = class
  private
    FHead: TNode;
    FCount: Integer;
  public
    procedure Add(AValue: Integer);
    procedure PrintAll;
    { ถูกต้อง: Free ทุก Node }
    destructor Destroy; override;
  end;

constructor TNode.Create(AValue: Integer);
begin
  FValue := AValue;
  FNext := nil;
end;

{ TLeakyList - มี Memory Leak }
procedure TLeakyList.Add(AValue: Integer);
var
  NewNode: TNode;
begin
  NewNode := TNode.Create(AValue);
  NewNode.Next := FHead;
  FHead := NewNode;
  Inc(FCount);
end;

procedure TLeakyList.PrintAll;
var
  Current: TNode;
begin
  Current := FHead;
  while Current <> nil do
  begin
    Write(Current.Value, ' ');
    Current := Current.Next;
  end;
  WriteLn;
end;

destructor TLeakyList.Destroy;
begin
  { BUG: ลืม Free Nodes! }
  FHead := nil;  { แค่ล้าง Reference แต่ไม่ Free Memory }
  inherited;
end;

{ TProperList - ถูกต้อง }
procedure TProperList.Add(AValue: Integer);
var
  NewNode: TNode;
begin
  NewNode := TNode.Create(AValue);
  NewNode.Next := FHead;
  FHead := NewNode;
  Inc(FCount);
end;

procedure TProperList.PrintAll;
var
  Current: TNode;
begin
  Current := FHead;
  while Current <> nil do
  begin
    Write(Current.Value, ' ');
    Current := Current.Next;
  end;
  WriteLn;
end;

destructor TProperList.Destroy;
var
  Current, Next: TNode;
begin
  Current := FHead;
  while Current <> nil do
  begin
    Next := Current.Next;
    Current.Free;  { ← Free ทุก Node }
    Current := Next;
  end;
  inherited;
end;

var
  Leaky: TLeakyList;
  Proper: TProperList;
  I: Integer;
begin
  { ตัวอย่าง Memory Leak }
  Leaky := TLeakyList.Create;
  try
    for I := 1 to 5 do
      Leaky.Add(I);
    Leaky.PrintAll;
  finally
    Leaky.Free;  { ← Destructor ไม่ได้ Free Nodes จึงเกิด Leak }
  end;
  
  { ตัวอย่างถูกต้อง }
  Proper := TProperList.Create;
  try
    for I := 1 to 5 do
      Proper.Add(I);
    Proper.PrintAll;
  finally
    Proper.Free;
  end;
  
  {
    Output จาก heaptrc จะแสดง:
    Heap dump by heaptrc unit of ...
    5 memory blocks allocated : 5/80
    5 unfreed memory blocks : 5
    True heap size : ...
    ...
    allocated at line X of ...
  }
end.
```

---

## 3. Custom Logger สำหรับ Debugging

```pascal
unit DebugLogger;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, SyncObjs;

type
  TLogLevel = (llDebug, llInfo, llWarning, llError, llCritical);
  TLogOutput = (loConsole, loFile, loBoth);

  TLogEntry = record
    Level: TLogLevel;
    Timestamp: TDateTime;
    ThreadID: TThreadID;
    FileName: string;
    LineNumber: Integer;
    FunctionName: string;
    Message: string;
    Exception: string;
  end;

  TLogFormatter = class
  public
    class function Format(const AEntry: TLogEntry): string; virtual;
    class function LevelToString(ALevel: TLogLevel): string;
    class function LevelToColor(ALevel: TLogLevel): string;
  end;

  TLogger = class
  private
    class var FInstance: TLogger;
    
    FLevel: TLogLevel;
    FOutput: TLogOutput;
    FLogFile: TextFile;
    FLogFileName: string;
    FLock: TCriticalSection;
    FIsFileOpen: Boolean;
    FBuffered: Boolean;
    FBuffer: TStringList;
    FMaxBufferSize: Integer;
    
    constructor CreateInternal;
    procedure WriteToOutput(const AEntry: TLogEntry);
    procedure FlushBuffer;
    
  public
    class function GetInstance: TLogger;
    class procedure FreeInstance;
    
    procedure Configure(ALevel: TLogLevel; AOutput: TLogOutput; 
                         const AFileName: string = '');
    
    procedure Log(ALevel: TLogLevel; const AMessage: string;
                  const AFile: string = ''; ALine: Integer = 0;
                  const AFunc: string = ''; AException: Exception = nil);
    
    procedure Debug(const AMessage: string; const AFile: string = '';
                    ALine: Integer = 0; const AFunc: string = '');
    procedure Info(const AMessage: string; const AFile: string = '';
                   ALine: Integer = 0; const AFunc: string = '');
    procedure Warning(const AMessage: string; const AFile: string = '';
                      ALine: Integer = 0; const AFunc: string = '');
    procedure Error(const AMessage: string; AException: Exception = nil;
                    const AFile: string = ''; ALine: Integer = 0;
                    const AFunc: string = '');
    procedure Critical(const AMessage: string; AException: Exception = nil;
                       const AFile: string = ''; ALine: Integer = 0;
                       const AFunc: string = '');
    
    procedure Flush;
    procedure SetBuffered(ABuffered: Boolean; ABufferSize: Integer = 100);
    
    property Level: TLogLevel read FLevel write FLevel;
    property Output: TLogOutput read FOutput;
    
    destructor Destroy; override;
  end;

{ Macros สำหรับการใช้งาน }
procedure LogDebug(const AMessage: string);
procedure LogInfo(const AMessage: string);
procedure LogWarning(const AMessage: string);
procedure LogError(const AMessage: string; AException: Exception = nil);

implementation

{ TLogFormatter }
class function TLogFormatter.LevelToString(ALevel: TLogLevel): string;
begin
  case ALevel of
    llDebug:    Result := 'DEBUG';
    llInfo:     Result := 'INFO ';
    llWarning:  Result := 'WARN ';
    llError:    Result := 'ERROR';
    llCritical: Result := 'CRIT ';
  end;
end;

class function TLogFormatter.LevelToColor(ALevel: TLogLevel): string;
begin
  case ALevel of
    llDebug:    Result := #27'[36m';  { Cyan }
    llInfo:     Result := #27'[32m';  { Green }
    llWarning:  Result := #27'[33m';  { Yellow }
    llError:    Result := #27'[31m';  { Red }
    llCritical: Result := #27'[35m';  { Magenta }
  else
    Result := #27'[0m';
  end;
end;

class function TLogFormatter.Format(const AEntry: TLogEntry): string;
var
  TimeStr: string;
begin
  TimeStr := FormatDateTime('yyyy-mm-dd hh:nn:ss.zzz', AEntry.Timestamp);
  
  Result := Format('[%s] [%s] [TID:%d]', 
    [TimeStr, LevelToString(AEntry.Level), AEntry.ThreadID]);
  
  if AEntry.FunctionName <> '' then
    Result := Result + Format(' [%s', [AEntry.FunctionName]);
  
  if AEntry.LineNumber > 0 then
    Result := Result + Format(':%d', [AEntry.LineNumber]);
  
  if AEntry.FunctionName <> '' then
    Result := Result + ']';
  
  Result := Result + ' ' + AEntry.Message;
  
  if AEntry.Exception <> '' then
    Result := Result + #13#10'  Exception: ' + AEntry.Exception;
end;

{ TLogger }
constructor TLogger.CreateInternal;
begin
  FLevel := llDebug;
  FOutput := loConsole;
  FLogFileName := '';
  FIsFileOpen := False;
  FLock := TCriticalSection.Create;
  FBuffered := False;
  FBuffer := TStringList.Create;
  FMaxBufferSize := 100;
end;

destructor TLogger.Destroy;
begin
  Flush;
  if FIsFileOpen then
  begin
    CloseFile(FLogFile);
    FIsFileOpen := False;
  end;
  FBuffer.Free;
  FLock.Free;
  inherited;
end;

class function TLogger.GetInstance: TLogger;
begin
  if FInstance = nil then
    FInstance := TLogger.CreateInternal;
  Result := FInstance;
end;

class procedure TLogger.FreeInstance;
begin
  FreeAndNil(FInstance);
end;

procedure TLogger.Configure(ALevel: TLogLevel; AOutput: TLogOutput;
  const AFileName: string);
begin
  FLock.Enter;
  try
    FLevel := ALevel;
    FOutput := AOutput;
    
    if FIsFileOpen then
    begin
      CloseFile(FLogFile);
      FIsFileOpen := False;
    end;
    
    if (AOutput in [loFile, loBoth]) and (AFileName <> '') then
    begin
      FLogFileName := AFileName;
      AssignFile(FLogFile, AFileName);
      if FileExists(AFileName) then
        Append(FLogFile)
      else
        Rewrite(FLogFile);
      FIsFileOpen := True;
    end;
  finally
    FLock.Leave;
  end;
end;

procedure TLogger.WriteToOutput(const AEntry: TLogEntry);
var
  Line: string;
begin
  Line := TLogFormatter.Format(AEntry);
  
  if FBuffered then
  begin
    FBuffer.Add(Line);
    if FBuffer.Count >= FMaxBufferSize then
      FlushBuffer;
  end
  else
  begin
    if FOutput in [loConsole, loBoth] then
    begin
      Write(TLogFormatter.LevelToColor(AEntry.Level));
      WriteLn(Line);
      Write(#27'[0m');
    end;
    
    if (FOutput in [loFile, loBoth]) and FIsFileOpen then
      WriteLn(FLogFile, Line);
  end;
end;

procedure TLogger.FlushBuffer;
var
  I: Integer;
  Line: string;
begin
  for Line in FBuffer do
  begin
    if FOutput in [loConsole, loBoth] then
      WriteLn(Line);
    if (FOutput in [loFile, loBoth]) and FIsFileOpen then
      WriteLn(FLogFile, Line);
  end;
  FBuffer.Clear;
end;

procedure TLogger.Log(ALevel: TLogLevel; const AMessage: string;
  const AFile: string; ALine: Integer; const AFunc: string;
  AException: Exception);
var
  Entry: TLogEntry;
begin
  if ALevel < FLevel then Exit;
  
  FLock.Enter;
  try
    Entry.Level := ALevel;
    Entry.Timestamp := Now;
    Entry.ThreadID := GetCurrentThreadID;
    Entry.FileName := AFile;
    Entry.LineNumber := ALine;
    Entry.FunctionName := AFunc;
    Entry.Message := AMessage;
    if AException <> nil then
      Entry.Exception := AException.ClassName + ': ' + AException.Message
    else
      Entry.Exception := '';
    
    WriteToOutput(Entry);
  finally
    FLock.Leave;
  end;
end;

procedure TLogger.Debug(const AMessage: string; const AFile: string;
  ALine: Integer; const AFunc: string);
begin
  Log(llDebug, AMessage, AFile, ALine, AFunc);
end;

procedure TLogger.Info(const AMessage: string; const AFile: string;
  ALine: Integer; const AFunc: string);
begin
  Log(llInfo, AMessage, AFile, ALine, AFunc);
end;

procedure TLogger.Warning(const AMessage: string; const AFile: string;
  ALine: Integer; const AFunc: string);
begin
  Log(llWarning, AMessage, AFile, ALine, AFunc);
end;

procedure TLogger.Error(const AMessage: string; AException: Exception;
  const AFile: string; ALine: Integer; const AFunc: string);
begin
  Log(llError, AMessage, AFile, ALine, AFunc, AException);
end;

procedure TLogger.Critical(const AMessage: string; AException: Exception;
  const AFile: string; ALine: Integer; const AFunc: string);
begin
  Log(llCritical, AMessage, AFile, ALine, AFunc, AException);
end;

procedure TLogger.Flush;
begin
  FLock.Enter;
  try
    if FBuffered then FlushBuffer;
  finally
    FLock.Leave;
  end;
end;

procedure TLogger.SetBuffered(ABuffered: Boolean; ABufferSize: Integer);
begin
  FBuffered := ABuffered;
  FMaxBufferSize := ABufferSize;
end;

{ Helper Functions }
procedure LogDebug(const AMessage: string);
begin
  TLogger.GetInstance.Debug(AMessage);
end;

procedure LogInfo(const AMessage: string);
begin
  TLogger.GetInstance.Info(AMessage);
end;

procedure LogWarning(const AMessage: string);
begin
  TLogger.GetInstance.Warning(AMessage);
end;

procedure LogError(const AMessage: string; AException: Exception);
begin
  TLogger.GetInstance.Error(AMessage, AException);
end;

initialization
  { ไม่ต้องสร้างที่นี่ - ใช้ Lazy Initialization }

finalization
  TLogger.FreeInstance;

end.
```

---

## 4. Call Stack Tracing

```pascal
unit CallStackUtils;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils;

type
  TCallEntry = record
    FunctionName: string;
    FileName: string;
    LineNumber: Integer;
    ElapsedMs: Double;
  end;

  { Function Call Tracer }
  TCallTracer = class
  private
    class var FInstance: TCallTracer;
    FStack: array of TCallEntry;
    FStackDepth: Integer;
    FEnabled: Boolean;
    FOutput: TStringList;
    
    function GetIndent: string;
  public
    class function GetInstance: TCallTracer;
    
    procedure Enter(const AFuncName, AFile: string; ALine: Integer);
    procedure Leave(const AFuncName: string);
    
    procedure Enable;
    procedure Disable;
    procedure Clear;
    procedure PrintTrace;
    procedure SaveTrace(const AFileName: string);
    
    property Enabled: Boolean read FEnabled;
  end;

  { Auto-trace helper - ใช้ try/finally }
  TAutoTrace = class
  private
    FFuncName: string;
    FStartTime: TDateTime;
  public
    constructor Create(const AFuncName, AFile: string; ALine: Integer);
    destructor Destroy; override;
  end;

implementation

uses DateUtils;

{ TCallTracer }
class function TCallTracer.GetInstance: TCallTracer;
begin
  if FInstance = nil then
  begin
    FInstance := TCallTracer.Create;
    FInstance.FEnabled := False;
    FInstance.FStackDepth := 0;
    FInstance.FOutput := TStringList.Create;
    SetLength(FInstance.FStack, 100);
  end;
  Result := FInstance;
end;

function TCallTracer.GetIndent: string;
var I: Integer;
begin
  Result := '';
  for I := 1 to FStackDepth do
    Result := Result + '  ';
end;

procedure TCallTracer.Enter(const AFuncName, AFile: string; ALine: Integer);
var
  Entry: string;
begin
  if not FEnabled then Exit;
  
  if FStackDepth < Length(FStack) then
  begin
    FStack[FStackDepth].FunctionName := AFuncName;
    FStack[FStackDepth].FileName := AFile;
    FStack[FStackDepth].LineNumber := ALine;
    FStack[FStackDepth].ElapsedMs := MilliSecondsBetween(Now, 0);
  end;
  
  Entry := GetIndent + '→ ' + AFuncName;
  if ALine > 0 then
    Entry := Entry + Format(' (%s:%d)', [ExtractFileName(AFile), ALine]);
  
  FOutput.Add(Entry);
  Inc(FStackDepth);
end;

procedure TCallTracer.Leave(const AFuncName: string);
var
  Elapsed: Double;
  Entry: string;
begin
  if not FEnabled then Exit;
  if FStackDepth = 0 then Exit;
  
  Dec(FStackDepth);
  
  Elapsed := MilliSecondsBetween(Now, 0) - FStack[FStackDepth].ElapsedMs;
  Entry := GetIndent + '← ' + AFuncName + Format(' (%.2f ms)', [Elapsed]);
  FOutput.Add(Entry);
end;

procedure TCallTracer.Enable;
begin
  FEnabled := True;
end;

procedure TCallTracer.Disable;
begin
  FEnabled := False;
end;

procedure TCallTracer.Clear;
begin
  FOutput.Clear;
  FStackDepth := 0;
end;

procedure TCallTracer.PrintTrace;
var
  Line: string;
begin
  WriteLn('=== Call Stack Trace ===');
  for Line in FOutput do
    WriteLn(Line);
end;

procedure TCallTracer.SaveTrace(const AFileName: string);
begin
  FOutput.SaveToFile(AFileName);
end;

{ TAutoTrace }
constructor TAutoTrace.Create(const AFuncName, AFile: string; ALine: Integer);
begin
  FFuncName := AFuncName;
  FStartTime := Now;
  TCallTracer.GetInstance.Enter(AFuncName, AFile, ALine);
end;

destructor TAutoTrace.Destroy;
begin
  TCallTracer.GetInstance.Leave(FFuncName);
  inherited;
end;

end.
```

---

## 5. Performance Profiler

```pascal
unit PerformanceProfiler;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, DateUtils;

type
  TProfileData = class
  private
    FName: string;
    FCallCount: Int64;
    FTotalTime: Double;
    FMinTime: Double;
    FMaxTime: Double;
    FLastCallTime: TDateTime;
  public
    constructor Create(const AName: string);
    procedure RecordCall(AElapsed: Double);
    
    property Name: string read FName;
    property CallCount: Int64 read FCallCount;
    property TotalTime: Double read FTotalTime;
    property MinTime: Double read FMinTime;
    property MaxTime: Double read FMaxTime;
    property AvgTime: Double read GetAvgTime;
    function GetAvgTime: Double;
    function ToCSVLine: string;
  end;

  TProfilerReport = class
  public
    class procedure PrintReport(AData: specialize TObjectDictionary<string, TProfileData>);
    class procedure SaveCSV(const AFileName: string; 
                             AData: specialize TObjectDictionary<string, TProfileData>);
    class procedure SaveHTML(const AFileName: string;
                              AData: specialize TObjectDictionary<string, TProfileData>);
  end;

  TProfiler = class
  private
    class var FInstance: TProfiler;
    
    FData: specialize TObjectDictionary<string, TProfileData>;
    FEnabled: Boolean;
    FCallStack: TStringList;
    FCallTimes: array of Double;
    FCallDepth: Integer;
    
    function GetHighResTime: Double;
  public
    class function GetInstance: TProfiler;
    
    procedure BeginProfile(const AFuncName: string);
    procedure EndProfile(const AFuncName: string);
    
    procedure Enable;
    procedure Disable;
    procedure Reset;
    
    procedure PrintReport;
    procedure SaveReport(const AFileName: string);
    
    property Enabled: Boolean read FEnabled;
    
    destructor Destroy; override;
  end;

  { Scoped Profiler - Auto begin/end }
  TScopedProfiler = class
  private
    FFuncName: string;
  public
    constructor Create(const AFuncName: string);
    destructor Destroy; override;
  end;

implementation

{ TProfileData }
constructor TProfileData.Create(const AName: string);
begin
  FName := AName;
  FCallCount := 0;
  FTotalTime := 0;
  FMinTime := MaxDouble;
  FMaxTime := 0;
end;

procedure TProfileData.RecordCall(AElapsed: Double);
begin
  Inc(FCallCount);
  FTotalTime := FTotalTime + AElapsed;
  if AElapsed < FMinTime then FMinTime := AElapsed;
  if AElapsed > FMaxTime then FMaxTime := AElapsed;
  FLastCallTime := Now;
end;

function TProfileData.GetAvgTime: Double;
begin
  if FCallCount > 0 then
    Result := FTotalTime / FCallCount
  else
    Result := 0;
end;

function TProfileData.ToCSVLine: string;
begin
  Result := Format('"%s",%d,%.6f,%.6f,%.6f,%.6f',
    [FName, FCallCount, FTotalTime * 1000, GetAvgTime * 1000,
     FMinTime * 1000, FMaxTime * 1000]);
end;

{ TProfilerReport }
class procedure TProfilerReport.PrintReport(
  AData: specialize TObjectDictionary<string, TProfileData>);
var
  Data: TProfileData;
  SortedList: specialize TObjectList<TProfileData>;
  I: Integer;
begin
  WriteLn('=== Performance Profile Report ===');
  WriteLn(Format('%-40s %8s %10s %10s %10s %10s',
    ['Function', 'Calls', 'Total(ms)', 'Avg(ms)', 'Min(ms)', 'Max(ms)']));
  WriteLn(StringOfChar('-', 100));
  
  { Sort by Total Time (Descending) }
  SortedList := specialize TObjectList<TProfileData>.Create(False);
  try
    for Data in AData.Values do
      SortedList.Add(Data);
    
    SortedList.Sort(
      specialize TComparer<TProfileData>.Construct(
        function(const A, B: TProfileData): Integer
        begin
          if A.TotalTime > B.TotalTime then Result := -1
          else if A.TotalTime < B.TotalTime then Result := 1
          else Result := 0;
        end
      )
    );
    
    for I := 0 to SortedList.Count - 1 do
    begin
      Data := SortedList[I];
      WriteLn(Format('%-40s %8d %10.3f %10.3f %10.3f %10.3f',
        [Data.Name, Data.CallCount,
         Data.TotalTime * 1000, Data.GetAvgTime * 1000,
         Data.MinTime * 1000, Data.MaxTime * 1000]));
    end;
  finally
    SortedList.Free;
  end;
end;

class procedure TProfilerReport.SaveCSV(const AFileName: string;
  AData: specialize TObjectDictionary<string, TProfileData>);
var
  Lines: TStringList;
  Data: TProfileData;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('Function,Calls,Total(ms),Avg(ms),Min(ms),Max(ms)');
    for Data in AData.Values do
      Lines.Add(Data.ToCSVLine);
    Lines.SaveToFile(AFileName);
  finally
    Lines.Free;
  end;
end;

class procedure TProfilerReport.SaveHTML(const AFileName: string;
  AData: specialize TObjectDictionary<string, TProfileData>);
var
  F: TextFile;
  Data: TProfileData;
begin
  AssignFile(F, AFileName);
  Rewrite(F);
  try
    WriteLn(F, '<!DOCTYPE html><html><head>');
    WriteLn(F, '<style>');
    WriteLn(F, 'body{font-family:monospace;margin:20px}');
    WriteLn(F, 'table{border-collapse:collapse;width:100%}');
    WriteLn(F, 'th,td{border:1px solid #ccc;padding:8px;text-align:right}');
    WriteLn(F, 'th{background:#333;color:white}');
    WriteLn(F, 'tr:nth-child(even){background:#f0f0f0}');
    WriteLn(F, '.func{text-align:left}');
    WriteLn(F, '</style></head><body>');
    WriteLn(F, '<h1>Performance Profile</h1>');
    WriteLn(F, '<p>สร้างเมื่อ: ' + FormatDateTime('dd/mm/yyyy hh:nn:ss', Now) + '</p>');
    WriteLn(F, '<table>');
    WriteLn(F, '<tr><th>Function</th><th>Calls</th><th>Total(ms)</th>');
    WriteLn(F, '<th>Avg(ms)</th><th>Min(ms)</th><th>Max(ms)</th></tr>');
    
    for Data in AData.Values do
      WriteLn(F, Format(
        '<tr><td class="func">%s</td><td>%d</td><td>%.3f</td>' +
        '<td>%.3f</td><td>%.3f</td><td>%.3f</td></tr>',
        [Data.Name, Data.CallCount,
         Data.TotalTime * 1000, Data.GetAvgTime * 1000,
         Data.MinTime * 1000, Data.MaxTime * 1000]));
    
    WriteLn(F, '</table></body></html>');
  finally
    CloseFile(F);
  end;
end;

{ TProfiler }
class function TProfiler.GetInstance: TProfiler;
begin
  if FInstance = nil then
  begin
    FInstance := TProfiler.Create;
    FInstance.FData := specialize TObjectDictionary<string, TProfileData>.Create([doOwnsValues]);
    FInstance.FEnabled := True;
    FInstance.FCallStack := TStringList.Create;
    FInstance.FCallDepth := 0;
    SetLength(FInstance.FCallTimes, 1000);
  end;
  Result := FInstance;
end;

destructor TProfiler.Destroy;
begin
  FData.Free;
  FCallStack.Free;
  inherited;
end;

function TProfiler.GetHighResTime: Double;
begin
  Result := Now * 86400.0;  { แปลงเป็น seconds }
end;

procedure TProfiler.BeginProfile(const AFuncName: string);
begin
  if not FEnabled then Exit;
  
  FCallStack.Add(AFuncName);
  if FCallDepth < Length(FCallTimes) then
    FCallTimes[FCallDepth] := GetHighResTime;
  Inc(FCallDepth);
end;

procedure TProfiler.EndProfile(const AFuncName: string);
var
  Elapsed: Double;
  Data: TProfileData;
begin
  if not FEnabled then Exit;
  if FCallDepth = 0 then Exit;
  
  Dec(FCallDepth);
  Elapsed := GetHighResTime - FCallTimes[FCallDepth];
  
  if not FData.TryGetValue(AFuncName, Data) then
  begin
    Data := TProfileData.Create(AFuncName);
    FData.Add(AFuncName, Data);
  end;
  
  Data.RecordCall(Elapsed);
  
  if FCallStack.Count > 0 then
    FCallStack.Delete(FCallStack.Count - 1);
end;

procedure TProfiler.Enable;
begin FEnabled := True; end;

procedure TProfiler.Disable;
begin FEnabled := False; end;

procedure TProfiler.Reset;
begin
  FData.Clear;
  FCallStack.Clear;
  FCallDepth := 0;
end;

procedure TProfiler.PrintReport;
begin
  TProfilerReport.PrintReport(FData);
end;

procedure TProfiler.SaveReport(const AFileName: string);
begin
  case LowerCase(ExtractFileExt(AFileName)) of
    '.csv': TProfilerReport.SaveCSV(AFileName, FData);
    '.html', '.htm': TProfilerReport.SaveHTML(AFileName, FData);
  else
    TProfilerReport.SaveCSV(AFileName, FData);
  end;
end;

{ TScopedProfiler }
constructor TScopedProfiler.Create(const AFuncName: string);
begin
  FFuncName := AFuncName;
  TProfiler.GetInstance.BeginProfile(AFuncName);
end;

destructor TScopedProfiler.Destroy;
begin
  TProfiler.GetInstance.EndProfile(FFuncName);
  inherited;
end;

end.
```

---

## 6. ตัวอย่าง Debug Session

```pascal
program DebugSession;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, DebugLogger, PerformanceProfiler;

type
  TDataProcessor = class
  private
    FData: array of Integer;
    FProfiler: TProfiler;
    FLogger: TLogger;
    
    procedure LoadData;
    function FindDuplicates: Integer;
    function SortData: Double;
    function CalculateStats: string;
    
  public
    constructor Create;
    destructor Destroy; override;
    procedure Process;
    procedure PrintReport;
  end;

constructor TDataProcessor.Create;
begin
  FProfiler := TProfiler.GetInstance;
  FLogger := TLogger.GetInstance;
  FLogger.Configure(llDebug, loBoth, 'app.log');
end;

destructor TDataProcessor.Destroy;
begin
  inherited;
end;

procedure TDataProcessor.LoadData;
var
  Profile: TScopedProfiler;
  I: Integer;
begin
  Profile := TScopedProfiler.Create('LoadData');
  try
    FLogger.Debug('เริ่มโหลดข้อมูล...', {$I %FILE%}, {$I %LINE%}, {$I %CURRENTROUTINE%});
    
    SetLength(FData, 10000);
    for I := 0 to High(FData) do
      FData[I] := Random(1000);
    
    FLogger.Info(Format('โหลดข้อมูลแล้ว: %d รายการ', [Length(FData)]),
      {$I %FILE%}, {$I %LINE%}, {$I %CURRENTROUTINE%});
  finally
    Profile.Free;
  end;
end;

function TDataProcessor.FindDuplicates: Integer;
var
  Profile: TScopedProfiler;
  Counts: array[0..999] of Integer;
  I: Integer;
begin
  Profile := TScopedProfiler.Create('FindDuplicates');
  try
    FLogger.Debug('ค้นหา Duplicates...', {$I %FILE%}, {$I %LINE%}, {$I %CURRENTROUTINE%});
    
    FillChar(Counts, SizeOf(Counts), 0);
    Result := 0;
    
    for I := 0 to High(FData) do
    begin
      Inc(Counts[FData[I]]);
      if Counts[FData[I]] = 2 then
        Inc(Result);
    end;
    
    FLogger.Info(Format('พบ Duplicates: %d ค่า', [Result]),
      {$I %FILE%}, {$I %LINE%}, {$I %CURRENTROUTINE%});
  finally
    Profile.Free;
  end;
end;

function TDataProcessor.SortData: Double;
var
  Profile: TScopedProfiler;
  StartTime: TDateTime;
  I, J, Temp: Integer;
begin
  Profile := TScopedProfiler.Create('SortData');
  try
    FLogger.Debug('เริ่มเรียงลำดับ...', {$I %FILE%}, {$I %LINE%}, {$I %CURRENTROUTINE%});
    StartTime := Now;
    
    { QuickSort }
    { ... (ตัวอย่าง Bubble Sort สำหรับความเรียบง่าย) }
    for I := High(FData) downto 1 do
      for J := 0 to I - 1 do
        if FData[J] > FData[J + 1] then
        begin
          Temp := FData[J];
          FData[J] := FData[J + 1];
          FData[J + 1] := Temp;
        end;
    
    Result := MilliSecondsBetween(Now, StartTime);
    FLogger.Info(Format('เรียงลำดับเสร็จ ใช้เวลา %.0f ms', [Result]),
      {$I %FILE%}, {$I %LINE%}, {$I %CURRENTROUTINE%});
  finally
    Profile.Free;
  end;
end;

function TDataProcessor.CalculateStats: string;
var
  Profile: TScopedProfiler;
  Sum: Int64;
  Min, Max, I: Integer;
  Avg: Double;
begin
  Profile := TScopedProfiler.Create('CalculateStats');
  try
    Sum := 0;
    Min := FData[0];
    Max := FData[0];
    
    for I := 0 to High(FData) do
    begin
      Sum := Sum + FData[I];
      if FData[I] < Min then Min := FData[I];
      if FData[I] > Max then Max := FData[I];
    end;
    
    Avg := Sum / Length(FData);
    
    Result := Format('Min=%d, Max=%d, Avg=%.2f, Sum=%d',
      [Min, Max, Avg, Sum]);
  finally
    Profile.Free;
  end;
end;

procedure TDataProcessor.Process;
var
  Duplicates: Integer;
  SortTime: Double;
  Stats: string;
begin
  FLogger.Info('เริ่มต้นประมวลผล', {$I %FILE%}, {$I %LINE%}, {$I %CURRENTROUTINE%});
  
  LoadData;
  Duplicates := FindDuplicates;
  Stats := CalculateStats;
  SortTime := SortData;
  
  FLogger.Info(Format('ประมวลผลเสร็จสิ้น: %s, Duplicates=%d, Sort=%.0fms',
    [Stats, Duplicates, SortTime]),
    {$I %FILE%}, {$I %LINE%}, {$I %CURRENTROUTINE%});
end;

procedure TDataProcessor.PrintReport;
begin
  FProfiler.PrintReport;
  FProfiler.SaveReport('profile_report.html');
  WriteLn('บันทึก HTML Report แล้ว: profile_report.html');
end;

var
  Processor: TDataProcessor;
begin
  Processor := TDataProcessor.Create;
  try
    Processor.Process;
    Processor.PrintReport;
  finally
    Processor.Free;
  end;
end.
```

---

## แบบฝึกหัด

### ข้อ 1 - Memory Leak Hunt
เขียนโปรแกรมที่มี Memory Leak แล้วใช้ HeapTrc ค้นหาและแก้ไข:
- Linked List ที่ไม่ได้ Free
- Event Listener ที่ไม่ได้ Unsubscribe
- Circular Reference

### ข้อ 2 - Logger Enhancement
เพิ่มฟีเจอร์ให้ Logger:
- Log Rotation (แบ่งไฟล์รายวัน)
- Log Level Filter
- Remote Logging (UDP)
- Colored Console Output

### ข้อ 3 - Profiler Web Dashboard
สร้าง Web Dashboard แสดงผล Profile:
- Real-time Update
- Hot Functions Highlight
- Call Graph
- Memory Usage

### ข้อ 4 - Exception Tracker
สร้าง Exception Tracking System:
- Capture Stack Trace
- Group Similar Exceptions
- Alert เมื่อ Exception Rate สูง
- Export Report

### ข้อ 5 - Performance Regression Test
สร้าง Automated Performance Test:
- Run Benchmarks อัตโนมัติ
- เปรียบเทียบกับ Baseline
- Fail ถ้า Performance ลดลงเกิน X%
- บันทึก History
