# Part 58 - Performance Optimization ใน Pascal/Lazarus

## บทนำ

Performance Optimization คือกระบวนการทำให้โปรแกรมทำงานเร็วขึ้นหรือใช้ทรัพยากรน้อยลง หลักสำคัญคือ:
1. **วัดก่อน** - อย่า Optimize โดยไม่รู้ว่าส่วนไหนช้า
2. **ระบุ Bottleneck** - หาส่วนที่ใช้เวลามากที่สุด
3. **Optimize อย่างมีเป้าหมาย** - แก้ไขส่วนที่สำคัญที่สุดก่อน

---

## 1. Benchmark Framework

```pascal
unit BenchmarkUtils;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, DateUtils;

type
  TBenchmarkResult = record
    Name: string;
    Iterations: Integer;
    TotalTime: Double;    { วินาที }
    MinTime: Double;
    MaxTime: Double;
    AvgTime: Double;
    OpsPerSecond: Double;
    MemoryUsed: Int64;    { bytes }
  end;

  TBenchmarkProc = procedure of object;
  TBenchmarkProcAnon = reference to procedure;

  TBenchmark = class
  private
    FResults: array of TBenchmarkResult;
    FWarmupRuns: Integer;
    FMeasureRuns: Integer;
    
    function GetHighResTime: Double;
    function GetMemoryUsage: Int64;
  public
    constructor Create(AWarmupRuns: Integer = 3; AMeasureRuns: Integer = 10);
    
    function Run(const AName: string; AProc: TBenchmarkProcAnon; 
                  AIterations: Integer = 1000000): TBenchmarkResult;
    
    procedure CompareTwo(const AName1, AName2: string;
                          AProc1, AProc2: TBenchmarkProcAnon);
    
    procedure PrintResult(const AResult: TBenchmarkResult);
    procedure PrintAllResults;
    procedure SaveToCSV(const AFileName: string);
    
    property WarmupRuns: Integer read FWarmupRuns write FWarmupRuns;
    property MeasureRuns: Integer read FMeasureRuns write FMeasureRuns;
  end;

  { Timer สำหรับวัดเวลาแบบ High Resolution }
  THighResTimer = class
  private
    FStartTime: Int64;
    FFrequency: Int64;
    FRunning: Boolean;
  public
    constructor Create;
    procedure Start;
    procedure Stop;
    procedure Reset;
    function ElapsedMilliseconds: Double;
    function ElapsedMicroseconds: Double;
    function ElapsedSeconds: Double;
    property Running: Boolean read FRunning;
  end;

implementation

uses
  {$IFDEF WINDOWS}Windows{$ENDIF}
  {$IFDEF LINUX}unixtype, linux{$ENDIF};

{ THighResTimer }
constructor THighResTimer.Create;
begin
  FRunning := False;
  FStartTime := 0;
  {$IFDEF WINDOWS}
  QueryPerformanceFrequency(FFrequency);
  {$ELSE}
  FFrequency := 1000000000; { nanoseconds }
  {$ENDIF}
end;

procedure THighResTimer.Start;
begin
  {$IFDEF WINDOWS}
  QueryPerformanceCounter(FStartTime);
  {$ELSE}
  var TS: Ttimespec;
  clock_gettime(CLOCK_MONOTONIC, @TS);
  FStartTime := TS.tv_sec * 1000000000 + TS.tv_nsec;
  {$ENDIF}
  FRunning := True;
end;

procedure THighResTimer.Stop;
begin
  FRunning := False;
end;

procedure THighResTimer.Reset;
begin
  FStartTime := 0;
  FRunning := False;
end;

function THighResTimer.ElapsedSeconds: Double;
var
  CurrentTime: Int64;
begin
  {$IFDEF WINDOWS}
  QueryPerformanceCounter(CurrentTime);
  {$ELSE}
  var TS: Ttimespec;
  clock_gettime(CLOCK_MONOTONIC, @TS);
  CurrentTime := TS.tv_sec * 1000000000 + TS.tv_nsec;
  {$ENDIF}
  Result := (CurrentTime - FStartTime) / FFrequency;
end;

function THighResTimer.ElapsedMilliseconds: Double;
begin
  Result := ElapsedSeconds * 1000;
end;

function THighResTimer.ElapsedMicroseconds: Double;
begin
  Result := ElapsedSeconds * 1000000;
end;

{ TBenchmark }
constructor TBenchmark.Create(AWarmupRuns, AMeasureRuns: Integer);
begin
  FWarmupRuns := AWarmupRuns;
  FMeasureRuns := AMeasureRuns;
  SetLength(FResults, 0);
end;

function TBenchmark.GetHighResTime: Double;
var
  Timer: THighResTimer;
begin
  Timer := THighResTimer.Create;
  Timer.Start;
  Result := Timer.ElapsedSeconds;
  Timer.Free;
end;

function TBenchmark.GetMemoryUsage: Int64;
begin
  { TODO: Platform-specific memory usage }
  Result := 0;
end;

function TBenchmark.Run(const AName: string; AProc: TBenchmarkProcAnon;
  AIterations: Integer): TBenchmarkResult;
var
  Timer: THighResTimer;
  I, J: Integer;
  Times: array of Double;
  StartMem, EndMem: Int64;
begin
  Result.Name := AName;
  Result.Iterations := AIterations;
  
  Timer := THighResTimer.Create;
  try
    { Warmup }
    for I := 1 to FWarmupRuns do
    begin
      Timer.Start;
      for J := 1 to AIterations do
        AProc();
      Timer.Stop;
    end;
    
    { Measure }
    SetLength(Times, FMeasureRuns);
    StartMem := GetMemoryUsage;
    
    for I := 0 to FMeasureRuns - 1 do
    begin
      Timer.Start;
      for J := 1 to AIterations do
        AProc();
      Times[I] := Timer.ElapsedSeconds;
    end;
    
    EndMem := GetMemoryUsage;
    Result.MemoryUsed := EndMem - StartMem;
    
    { คำนวณสถิติ }
    Result.TotalTime := 0;
    Result.MinTime := Times[0];
    Result.MaxTime := Times[0];
    
    for I := 0 to FMeasureRuns - 1 do
    begin
      Result.TotalTime := Result.TotalTime + Times[I];
      if Times[I] < Result.MinTime then Result.MinTime := Times[I];
      if Times[I] > Result.MaxTime then Result.MaxTime := Times[I];
    end;
    
    Result.AvgTime := Result.TotalTime / FMeasureRuns;
    Result.OpsPerSecond := AIterations / Result.AvgTime;
    
    { บันทึกผลลัพธ์ }
    SetLength(FResults, Length(FResults) + 1);
    FResults[High(FResults)] := Result;
  finally
    Timer.Free;
  end;
end;

procedure TBenchmark.CompareTwo(const AName1, AName2: string;
  AProc1, AProc2: TBenchmarkProcAnon);
var
  R1, R2: TBenchmarkResult;
  Ratio: Double;
begin
  WriteLn('=== เปรียบเทียบ Performance ===');
  
  R1 := Run(AName1, AProc1);
  R2 := Run(AName2, AProc2);
  
  PrintResult(R1);
  PrintResult(R2);
  
  if R2.AvgTime > 0 then
    Ratio := R1.AvgTime / R2.AvgTime
  else
    Ratio := 0;
  
  WriteLn('---');
  if Ratio < 1 then
    WriteLn(Format('%s เร็วกว่า %s ประมาณ %.1fx', [AName1, AName2, 1/Ratio]))
  else
    WriteLn(Format('%s เร็วกว่า %s ประมาณ %.1fx', [AName2, AName1, Ratio]));
end;

procedure TBenchmark.PrintResult(const AResult: TBenchmarkResult);
begin
  WriteLn(Format('[%s]', [AResult.Name]));
  WriteLn(Format('  Iterations: %d', [AResult.Iterations]));
  WriteLn(Format('  Avg: %.3f ms', [AResult.AvgTime * 1000]));
  WriteLn(Format('  Min: %.3f ms', [AResult.MinTime * 1000]));
  WriteLn(Format('  Max: %.3f ms', [AResult.MaxTime * 1000]));
  WriteLn(Format('  Ops/sec: %.0f', [AResult.OpsPerSecond]));
  if AResult.MemoryUsed > 0 then
    WriteLn(Format('  Memory: %d bytes', [AResult.MemoryUsed]));
end;

procedure TBenchmark.PrintAllResults;
var
  R: TBenchmarkResult;
begin
  WriteLn('=== ผลการทดสอบทั้งหมด ===');
  for R in FResults do
    PrintResult(R);
end;

procedure TBenchmark.SaveToCSV(const AFileName: string);
var
  Lines: TStringList;
  R: TBenchmarkResult;
begin
  Lines := TStringList.Create;
  try
    Lines.Add('Name,Iterations,Avg(ms),Min(ms),Max(ms),Ops/sec');
    for R in FResults do
      Lines.Add(Format('"%s",%d,%.4f,%.4f,%.4f,%.0f',
        [R.Name, R.Iterations, 
         R.AvgTime * 1000, R.MinTime * 1000, R.MaxTime * 1000,
         R.OpsPerSecond]));
    Lines.SaveToFile(AFileName);
  finally
    Lines.Free;
  end;
end;

end.
```

---

## 2. String Performance

```pascal
program StringPerformance;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, BenchmarkUtils;

{ การต่อ String แบบต่างๆ }

{ แบบที่ 1: ต่อ String ธรรมดา (ช้าที่สุด) }
function ConcatSimple(ACount: Integer): string;
var
  I: Integer;
begin
  Result := '';
  for I := 1 to ACount do
    Result := Result + IntToStr(I);
end;

{ แบบที่ 2: ใช้ TStringBuilder }
function ConcatStringBuilder(ACount: Integer): string;
var
  SB: TStringBuilder;
  I: Integer;
begin
  SB := TStringBuilder.Create;
  try
    for I := 1 to ACount do
      SB.Append(IntToStr(I));
    Result := SB.ToString;
  finally
    SB.Free;
  end;
end;

{ แบบที่ 3: Pre-allocate ด้วย SetLength }
function ConcatPreAlloc(ACount: Integer): string;
var
  Pos, I: Integer;
  NumStr: string;
begin
  { ประมาณขนาดที่ต้องการ }
  SetLength(Result, ACount * 5);
  Pos := 1;
  for I := 1 to ACount do
  begin
    NumStr := IntToStr(I);
    Move(NumStr[1], Result[Pos], Length(NumStr));
    Inc(Pos, Length(NumStr));
  end;
  SetLength(Result, Pos - 1);
end;

{ แบบที่ 4: ใช้ TStringList }
function ConcatStringList(ACount: Integer): string;
var
  SL: TStringList;
  I: Integer;
begin
  SL := TStringList.Create;
  try
    for I := 1 to ACount do
      SL.Add(IntToStr(I));
    Result := SL.Text;
  finally
    SL.Free;
  end;
end;

{ การ Search String แบบต่างๆ }
procedure DemoStringSearch;
var
  LongString: string;
  I: Integer;
  Found: Boolean;
  SearchFor: string;
begin
  { สร้าง String ยาวๆ }
  LongString := '';
  for I := 1 to 10000 do
    LongString := LongString + 'Hello World ' + IntToStr(I) + ' ';
  
  SearchFor := 'World 9999';
  
  { Pos - เร็วสำหรับการค้นหาธรรมดา }
  Found := Pos(SearchFor, LongString) > 0;
  WriteLn('Pos: Found = ', Found);
  
  { PosEx - เร็วกว่าเมื่อต้องการ Start Position }
  Found := PosEx(SearchFor, LongString, 1) > 0;
  WriteLn('PosEx: Found = ', Found);
end;

{ การเปรียบเทียบ String }
procedure DemoStringCompare;
const
  N = 100000;
var
  S1, S2: string;
  Bench: TBenchmark;
begin
  S1 := 'Hello World Test String';
  S2 := 'hello world test string';
  
  Bench := TBenchmark.Create(3, 5);
  try
    Bench.CompareTwo(
      'CompareText (case insensitive)',
      'LowerCase + Compare',
      procedure begin CompareText(S1, S2); end,
      procedure begin LowerCase(S1) = LowerCase(S2); end
    );
  finally
    Bench.Free;
  end;
end;

var
  Bench: TBenchmark;
  R1, R2, R3, R4: TBenchmarkResult;
const
  STR_COUNT = 1000;
  ITERATIONS = 100;
begin
  WriteLn('=== String Performance Test ===');
  WriteLn;
  
  Bench := TBenchmark.Create(2, 5);
  try
    R1 := Bench.Run('Simple Concat (1000 items)',
      procedure begin ConcatSimple(STR_COUNT); end, ITERATIONS);
    
    R2 := Bench.Run('TStringBuilder (1000 items)',
      procedure begin ConcatStringBuilder(STR_COUNT); end, ITERATIONS);
    
    R3 := Bench.Run('PreAlloc (1000 items)',
      procedure begin ConcatPreAlloc(STR_COUNT); end, ITERATIONS);
    
    R4 := Bench.Run('TStringList (1000 items)',
      procedure begin ConcatStringList(STR_COUNT); end, ITERATIONS);
    
    Bench.PrintAllResults;
    
    WriteLn;
    WriteLn('=== String Search Test ===');
    DemoStringSearch;
    
    WriteLn;
    WriteLn('=== String Compare Test ===');
    DemoStringCompare;
  finally
    Bench.Free;
  end;
end.
```

---

## 3. Memory Allocation Patterns

```pascal
unit MemoryPatterns;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, BenchmarkUtils;

type
  { Object Pool Pattern - ลด Memory Allocation }
  TObjectPool<T: class, constructor> = class
  private
    FPool: TList;
    FMaxSize: Integer;
    FCreated: Integer;
    FReused: Integer;
    
  public
    constructor Create(AMaxSize: Integer = 100);
    destructor Destroy; override;
    
    function Acquire: T;
    procedure Release(AObj: T);
    
    function GetPoolSize: Integer;
    function GetCreatedCount: Integer;
    function GetReusedCount: Integer;
    procedure PrintStats;
  end;

  { Memory Pool สำหรับ Fixed-size Blocks }
  TMemoryPool = class
  private
    FBlocks: array of Pointer;
    FBlockSize: Integer;
    FFreeBlocks: TList;
    FAllocated: Integer;
    FPoolSize: Integer;
    
    procedure GrowPool;
  public
    constructor Create(ABlockSize: Integer; AInitialBlocks: Integer = 100);
    destructor Destroy; override;
    
    function Alloc: Pointer;
    procedure Free(APtr: Pointer);
    function GetAllocated: Integer;
    function GetFree: Integer;
  end;

  { Arena Allocator - สำหรับ Short-lived Objects }
  TArenaAllocator = class
  private
    FArenas: array of TBytes;
    FCurrentArena: Integer;
    FCurrentPos: Integer;
    FArenaSize: Integer;
    FTotalAllocated: Int64;
    
    procedure GrowArena;
  public
    constructor Create(AArenaSize: Integer = 65536);
    destructor Destroy; override;
    
    function Alloc(ASize: Integer): Pointer;
    procedure Reset;
    function GetTotalAllocated: Int64;
  end;

implementation

{ TObjectPool }
constructor TObjectPool<T>.Create(AMaxSize: Integer);
begin
  FPool := TList.Create;
  FMaxSize := AMaxSize;
  FCreated := 0;
  FReused := 0;
end;

destructor TObjectPool<T>.Destroy;
var
  I: Integer;
begin
  for I := 0 to FPool.Count - 1 do
    T(FPool[I]).Free;
  FPool.Free;
  inherited;
end;

function TObjectPool<T>.Acquire: T;
begin
  if FPool.Count > 0 then
  begin
    Result := T(FPool.Last);
    FPool.Delete(FPool.Count - 1);
    Inc(FReused);
  end
  else
  begin
    Result := T.Create;
    Inc(FCreated);
  end;
end;

procedure TObjectPool<T>.Release(AObj: T);
begin
  if FPool.Count < FMaxSize then
    FPool.Add(AObj)
  else
    AObj.Free;
end;

function TObjectPool<T>.GetPoolSize: Integer;
begin Result := FPool.Count; end;

function TObjectPool<T>.GetCreatedCount: Integer;
begin Result := FCreated; end;

function TObjectPool<T>.GetReusedCount: Integer;
begin Result := FReused; end;

procedure TObjectPool<T>.PrintStats;
begin
  WriteLn(Format('Object Pool Stats:'));
  WriteLn(Format('  Pool size: %d', [FPool.Count]));
  WriteLn(Format('  Created: %d', [FCreated]));
  WriteLn(Format('  Reused: %d', [FReused]));
  if FCreated + FReused > 0 then
    WriteLn(Format('  Reuse rate: %.1f%%', 
      [FReused / (FCreated + FReused) * 100]));
end;

{ TMemoryPool }
constructor TMemoryPool.Create(ABlockSize: Integer; AInitialBlocks: Integer);
begin
  FBlockSize := ABlockSize;
  FFreeBlocks := TList.Create;
  FAllocated := 0;
  FPoolSize := 0;
  GrowPool;
  while FPoolSize < AInitialBlocks do
    GrowPool;
end;

destructor TMemoryPool.Destroy;
var
  I: Integer;
begin
  for I := 0 to High(FBlocks) do
    FreeMem(FBlocks[I]);
  FFreeBlocks.Free;
  inherited;
end;

procedure TMemoryPool.GrowPool;
var
  NewBlock: Pointer;
  I: Integer;
  BatchSize: Integer;
begin
  BatchSize := 64;
  SetLength(FBlocks, Length(FBlocks) + BatchSize);
  
  for I := Length(FBlocks) - BatchSize to High(FBlocks) do
  begin
    GetMem(NewBlock, FBlockSize);
    FBlocks[I] := NewBlock;
    FFreeBlocks.Add(NewBlock);
    Inc(FPoolSize);
  end;
end;

function TMemoryPool.Alloc: Pointer;
begin
  if FFreeBlocks.Count = 0 then
    GrowPool;
  
  Result := FFreeBlocks.Last;
  FFreeBlocks.Delete(FFreeBlocks.Count - 1);
  Inc(FAllocated);
end;

procedure TMemoryPool.Free(APtr: Pointer);
begin
  FFreeBlocks.Add(APtr);
  Dec(FAllocated);
end;

function TMemoryPool.GetAllocated: Integer;
begin Result := FAllocated; end;

function TMemoryPool.GetFree: Integer;
begin Result := FFreeBlocks.Count; end;

{ TArenaAllocator }
constructor TArenaAllocator.Create(AArenaSize: Integer);
begin
  FArenaSize := AArenaSize;
  FCurrentArena := -1;
  FCurrentPos := 0;
  FTotalAllocated := 0;
  GrowArena;
end;

destructor TArenaAllocator.Destroy;
begin
  SetLength(FArenas, 0);
  inherited;
end;

procedure TArenaAllocator.GrowArena;
begin
  SetLength(FArenas, Length(FArenas) + 1);
  SetLength(FArenas[High(FArenas)], FArenaSize);
  FCurrentArena := High(FArenas);
  FCurrentPos := 0;
end;

function TArenaAllocator.Alloc(ASize: Integer): Pointer;
var
  AlignedSize: Integer;
begin
  { Align to 8 bytes }
  AlignedSize := (ASize + 7) and not 7;
  
  if FCurrentPos + AlignedSize > FArenaSize then
    GrowArena;
  
  Result := @FArenas[FCurrentArena][FCurrentPos];
  Inc(FCurrentPos, AlignedSize);
  Inc(FTotalAllocated, AlignedSize);
end;

procedure TArenaAllocator.Reset;
begin
  SetLength(FArenas, 1);
  FCurrentArena := 0;
  FCurrentPos := 0;
  FTotalAllocated := 0;
end;

function TArenaAllocator.GetTotalAllocated: Int64;
begin Result := FTotalAllocated; end;

end.
```

---

## 4. Caching Strategies

```pascal
unit CachingStrategies;

{$mode objfpc}{$H+}

interface

uses
  Classes, SysUtils, Generics.Collections, DateUtils;

type
  { LRU Cache }
  TLRUCache<TKey, TValue> = class
  private
    type
      TCacheEntry = record
        Key: TKey;
        Value: TValue;
        LastAccess: TDateTime;
        AccessCount: Integer;
      end;
    
  private
    FCache: specialize TObjectDictionary<TKey, ^TCacheEntry>;
    FMaxSize: Integer;
    FHits: Int64;
    FMisses: Int64;
    
    procedure EvictLRU;
  public
    constructor Create(AMaxSize: Integer = 1000);
    destructor Destroy; override;
    
    procedure Put(const AKey: TKey; const AValue: TValue);
    function TryGet(const AKey: TKey; out AValue: TValue): Boolean;
    function Get(const AKey: TKey): TValue;
    procedure Remove(const AKey: TKey);
    procedure Clear;
    
    function GetHitRate: Double;
    function GetSize: Integer;
    procedure PrintStats;
  end;

  { Memoization Decorator }
  TMemoize<TInput, TOutput> = class
  private
    FCache: specialize TDictionary<string, TOutput>;
    FFunc: specialize TFunc<TInput, TOutput>;
    FKeyFunc: specialize TFunc<TInput, string>;
    FHits, FMisses: Integer;
  public
    constructor Create(AFunc: specialize TFunc<TInput, TOutput>;
                        AKeyFunc: specialize TFunc<TInput, string> = nil);
    destructor Destroy; override;
    
    function Call(const AInput: TInput): TOutput;
    procedure InvalidateAll;
    procedure InvalidateKey(const AKey: string);
    function GetHitRate: Double;
  end;

  { TTL Cache - หมดอายุอัตโนมัติ }
  TTTLCache<TKey, TValue> = class
  private
    type
      TEntry = record
        Value: TValue;
        ExpiresAt: TDateTime;
      end;
    
  private
    FCache: specialize TDictionary<TKey, TEntry>;
    FTTL: Integer;  { seconds }
    FLock: TCriticalSection;
    FCleanupInterval: Integer;
    FLastCleanup: TDateTime;
    
    procedure Cleanup;
  public
    constructor Create(ATTL: Integer = 300; ACleanupInterval: Integer = 60);
    destructor Destroy; override;
    
    procedure Put(const AKey: TKey; const AValue: TValue);
    procedure PutWithTTL(const AKey: TKey; const AValue: TValue; ATTL_Seconds: Integer);
    function TryGet(const AKey: TKey; out AValue: TValue): Boolean;
    procedure Remove(const AKey: TKey);
    procedure Clear;
    function GetSize: Integer;
    property TTL: Integer read FTTL write FTTL;
  end;

implementation

{ TLRUCache }
constructor TLRUCache<TKey, TValue>.Create(AMaxSize: Integer);
begin
  FMaxSize := AMaxSize;
  FCache := specialize TObjectDictionary<TKey, ^TCacheEntry>.Create;
  FHits := 0;
  FMisses := 0;
end;

destructor TLRUCache<TKey, TValue>.Destroy;
begin
  Clear;
  FCache.Free;
  inherited;
end;

procedure TLRUCache<TKey, TValue>.EvictLRU;
var
  OldestKey: TKey;
  OldestTime: TDateTime;
  Entry: ^TCacheEntry;
  HasEntry: Boolean;
begin
  OldestTime := Now + 1;
  HasEntry := False;
  
  for Entry in FCache.Values do
  begin
    if (not HasEntry) or (Entry^.LastAccess < OldestTime) then
    begin
      OldestTime := Entry^.LastAccess;
      OldestKey := Entry^.Key;
      HasEntry := True;
    end;
  end;
  
  if HasEntry then
  begin
    Dispose(FCache[OldestKey]);
    FCache.Remove(OldestKey);
  end;
end;

procedure TLRUCache<TKey, TValue>.Put(const AKey: TKey; const AValue: TValue);
var
  Entry: ^TCacheEntry;
begin
  if FCache.ContainsKey(AKey) then
  begin
    { Update existing }
    FCache[AKey]^.Value := AValue;
    FCache[AKey]^.LastAccess := Now;
  end
  else
  begin
    { Evict if full }
    if FCache.Count >= FMaxSize then
      EvictLRU;
    
    New(Entry);
    Entry^.Key := AKey;
    Entry^.Value := AValue;
    Entry^.LastAccess := Now;
    Entry^.AccessCount := 0;
    FCache.Add(AKey, Entry);
  end;
end;

function TLRUCache<TKey, TValue>.TryGet(const AKey: TKey; out AValue: TValue): Boolean;
var
  Entry: ^TCacheEntry;
begin
  if FCache.TryGetValue(AKey, Entry) then
  begin
    AValue := Entry^.Value;
    Entry^.LastAccess := Now;
    Inc(Entry^.AccessCount);
    Inc(FHits);
    Result := True;
  end
  else
  begin
    Inc(FMisses);
    Result := False;
  end;
end;

function TLRUCache<TKey, TValue>.Get(const AKey: TKey): TValue;
begin
  if not TryGet(AKey, Result) then
    raise Exception.CreateFmt('ไม่พบ Key ใน Cache', []);
end;

procedure TLRUCache<TKey, TValue>.Remove(const AKey: TKey);
var
  Entry: ^TCacheEntry;
begin
  if FCache.TryGetValue(AKey, Entry) then
  begin
    Dispose(Entry);
    FCache.Remove(AKey);
  end;
end;

procedure TLRUCache<TKey, TValue>.Clear;
var
  Entry: ^TCacheEntry;
begin
  for Entry in FCache.Values do
    Dispose(Entry);
  FCache.Clear;
end;

function TLRUCache<TKey, TValue>.GetHitRate: Double;
var
  Total: Int64;
begin
  Total := FHits + FMisses;
  if Total = 0 then Result := 0
  else Result := FHits / Total;
end;

function TLRUCache<TKey, TValue>.GetSize: Integer;
begin
  Result := FCache.Count;
end;

procedure TLRUCache<TKey, TValue>.PrintStats;
begin
  WriteLn(Format('LRU Cache Stats:'));
  WriteLn(Format('  Size: %d/%d', [FCache.Count, FMaxSize]));
  WriteLn(Format('  Hits: %d', [FHits]));
  WriteLn(Format('  Misses: %d', [FMisses]));
  WriteLn(Format('  Hit Rate: %.1f%%', [GetHitRate * 100]));
end;

{ TTTLCache }
constructor TTTLCache<TKey, TValue>.Create(ATTL: Integer; ACleanupInterval: Integer);
begin
  FCache := specialize TDictionary<TKey, TEntry>.Create;
  FTTL := ATTL;
  FLock := TCriticalSection.Create;
  FCleanupInterval := ACleanupInterval;
  FLastCleanup := Now;
end;

destructor TTTLCache<TKey, TValue>.Destroy;
begin
  FCache.Free;
  FLock.Free;
  inherited;
end;

procedure TTTLCache<TKey, TValue>.Cleanup;
var
  ToDelete: specialize TList<TKey>;
  Key: TKey;
  Entry: TEntry;
begin
  ToDelete := specialize TList<TKey>.Create;
  try
    for Key in FCache.Keys do
    begin
      if FCache.TryGetValue(Key, Entry) then
        if Now > Entry.ExpiresAt then
          ToDelete.Add(Key);
    end;
    
    for Key in ToDelete do
      FCache.Remove(Key);
  finally
    ToDelete.Free;
  end;
  
  FLastCleanup := Now;
end;

procedure TTTLCache<TKey, TValue>.Put(const AKey: TKey; const AValue: TValue);
var
  Entry: TEntry;
begin
  FLock.Enter;
  try
    Entry.Value := AValue;
    Entry.ExpiresAt := Now + (FTTL / 86400.0);
    FCache.AddOrSetValue(AKey, Entry);
    
    { Auto Cleanup }
    if SecondsBetween(Now, FLastCleanup) >= FCleanupInterval then
      Cleanup;
  finally
    FLock.Leave;
  end;
end;

procedure TTTLCache<TKey, TValue>.PutWithTTL(const AKey: TKey; const AValue: TValue; ATTL_Seconds: Integer);
var
  Entry: TEntry;
begin
  FLock.Enter;
  try
    Entry.Value := AValue;
    Entry.ExpiresAt := Now + (ATTL_Seconds / 86400.0);
    FCache.AddOrSetValue(AKey, Entry);
  finally
    FLock.Leave;
  end;
end;

function TTTLCache<TKey, TValue>.TryGet(const AKey: TKey; out AValue: TValue): Boolean;
var
  Entry: TEntry;
begin
  FLock.Enter;
  try
    if FCache.TryGetValue(AKey, Entry) then
    begin
      if Now <= Entry.ExpiresAt then
      begin
        AValue := Entry.Value;
        Result := True;
      end
      else
      begin
        { Expired }
        FCache.Remove(AKey);
        Result := False;
      end;
    end
    else
      Result := False;
  finally
    FLock.Leave;
  end;
end;

procedure TTTLCache<TKey, TValue>.Remove(const AKey: TKey);
begin
  FLock.Enter;
  try
    FCache.Remove(AKey);
  finally
    FLock.Leave;
  end;
end;

procedure TTTLCache<TKey, TValue>.Clear;
begin
  FLock.Enter;
  try
    FCache.Clear;
  finally
    FLock.Leave;
  end;
end;

function TTTLCache<TKey, TValue>.GetSize: Integer;
begin
  FLock.Enter;
  try
    Result := FCache.Count;
  finally
    FLock.Leave;
  end;
end;

end.
```

---

## 5. Algorithm Complexity Demo

```pascal
program AlgorithmComplexity;

{$mode objfpc}{$H+}

uses
  Classes, SysUtils, BenchmarkUtils;

{ O(n) - Linear }
function LinearSearch(const AArray: array of Integer; ATarget: Integer): Integer;
var I: Integer;
begin
  for I := 0 to High(AArray) do
    if AArray[I] = ATarget then Exit(I);
  Result := -1;
end;

{ O(log n) - Logarithmic (Binary Search) }
function BinarySearch(const AArray: array of Integer; ATarget: Integer): Integer;
var
  Left, Right, Mid: Integer;
begin
  Left := 0; Right := High(AArray);
  while Left <= Right do
  begin
    Mid := (Left + Right) div 2;
    if AArray[Mid] = ATarget then Exit(Mid)
    else if AArray[Mid] < ATarget then Left := Mid + 1
    else Right := Mid - 1;
  end;
  Result := -1;
end;

{ O(n^2) - Quadratic }
function HasDuplicates_Naive(const AArray: array of Integer): Boolean;
var I, J: Integer;
begin
  for I := 0 to High(AArray) - 1 do
    for J := I + 1 to High(AArray) do
      if AArray[I] = AArray[J] then Exit(True);
  Result := False;
end;

{ O(n log n) - ด้วย Sort }
function HasDuplicates_Sort(AArray: array of Integer): Boolean;
var
  I: Integer;
  Temp: Integer;
  Swapped: Boolean;
begin
  { Sort ก่อน (Bubble Sort สำหรับตัวอย่าง) }
  repeat
    Swapped := False;
    for I := 0 to High(AArray) - 1 do
      if AArray[I] > AArray[I+1] then
      begin
        Temp := AArray[I]; AArray[I] := AArray[I+1]; AArray[I+1] := Temp;
        Swapped := True;
      end;
  until not Swapped;
  
  { ตรวจ Duplicate }
  for I := 0 to High(AArray) - 1 do
    if AArray[I] = AArray[I+1] then Exit(True);
  Result := False;
end;

var
  SmallArray, LargeArray: array of Integer;
  SortedArray: array of Integer;
  I, N: Integer;
  Bench: TBenchmark;
begin
  WriteLn('=== Algorithm Complexity Comparison ===');
  WriteLn;
  
  { สร้างข้อมูลทดสอบ }
  N := 10000;
  SetLength(SmallArray, N);
  SetLength(SortedArray, N);
  
  for I := 0 to N - 1 do
  begin
    SmallArray[I] := Random(N * 2);
    SortedArray[I] := I;
  end;
  
  Bench := TBenchmark.Create(2, 5);
  try
    { Linear vs Binary Search }
    WriteLn('--- Linear Search vs Binary Search ---');
    Bench.CompareTwo(
      'Linear Search O(n)',
      'Binary Search O(log n)',
      procedure begin LinearSearch(SortedArray, N - 1); end,
      procedure begin BinarySearch(SortedArray, N - 1); end
    );
    
    WriteLn;
    WriteLn('--- Duplicate Detection ---');
    Bench.CompareTwo(
      'Naive O(n^2)',
      'Sort-based O(n log n)',
      procedure begin HasDuplicates_Naive(SmallArray); end,
      procedure begin HasDuplicates_Sort(SmallArray); end
    );
  finally
    Bench.Free;
  end;
end.
```

---

## แบบฝึกหัด

### ข้อ 1 - Benchmark Suite
สร้าง Benchmark Suite ที่ทดสอบ:
- Array operations (Add, Remove, Search)
- String operations (Concat, Split, Replace)
- Database queries (Simple, Join, Aggregate)
- File I/O (Sequential, Random)

### ข้อ 2 - String Pool
สร้าง String Interning Pool:
- เก็บ String Unique ไว้ใน Pool
- Return Reference เดิมถ้า String ซ้ำ
- วัด Memory ที่ประหยัดได้

### ข้อ 3 - Query Cache
สร้าง SQL Query Cache:
- Cache ผลลัพธ์ Query ที่เรียกบ่อย
- Invalidate เมื่อข้อมูลเปลี่ยน
- TTL สำหรับ Cache

### ข้อ 4 - Lazy Loading
สร้าง Lazy Loader สำหรับรูปภาพ:
- โหลดรูปเมื่อจำเป็น
- Cache รูปที่โหลดแล้ว
- วัด Performance

### ข้อ 5 - Object Pool
สร้าง Connection Pool สำหรับ Database:
- Pool ของ Connection Objects
- Borrow/Return Pattern
- Idle Connection Timeout
